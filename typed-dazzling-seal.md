# Production plan: Thay HikariCP bằng Oracle UCP cho service Spring Boot + TimesTen Scaleout 22.1

## Context

Service production đang dùng Spring Boot + HikariCP + Spring Data JPA/Hibernate để kết nối TimesTen Scaleout (4 data instance, K=2/R=2). Root cause đã xác định qua trace source code HikariCP + tài liệu Oracle: HikariCP không có khái niệm về grid/FAN/FCF, nên khi grid rebalance/failover giữa các data instance, HikariCP phát hiện "dead connection" một cách bị động (qua `isValid()` mỗi lần borrow nếu idle > 500ms) và phải đóng + tạo mới liên tục — đúng như alert đang gặp. Hướng khắc phục đã thống nhất: chuyển pool sang **Oracle UCP**, vốn là pool được TimesTen chính thức document (driver TimesTen không tự implement pooling).

Đã có bản demo tham khảo tại `/Users/hoanvt/Project/ucp-timesten-demo` minh hoạ cách wiring (`TimesTenUcpConfig`, `TimesTenUcpProperties`, layer `repository/service/web` dùng `JdbcTemplate`).

**2 quyết định quan trọng đã chốt với user, ảnh hưởng trực tiếp tới cấu trúc plan:**
1. Production hiện dùng **JPA/Hibernate** → việc bỏ JPA là một thay đổi rủi ro cao, riêng biệt, KHÔNG được bundle chung với việc đổi pool. Đổi pool (Hikari→UCP) không yêu cầu đổi JPA — `DataSource` là interface chung cho cả hai. Plan này tách thành 2 workstream độc lập, ưu tiên workstream A trước vì nó giải quyết trực tiếp sự cố đang gặp.
2. Có môi trường staging/UAT TimesTen Scaleout tách biệt production → có thể validate đầy đủ (load test, failover test) trước khi động vào prod.
3. Rollout: **cutover toàn bộ một lần, có rollback plan** (không canary/dual-pool) → plan cần cơ chế rollback nhanh không phụ thuộc tốc độ CI/CD redeploy.

## Ngoài phạm vi (Non-goals) của lần cutover này

- **Không** migrate JPA → JdbcTemplate trong lần này. Đây là Workstream B, làm sau, có profiling data dẫn đường (xem mục riêng).
- **Không** implement TimesTen Scaleout grid-aware routing (`TimesTenConnectionBuilder`/`TimesTenDistributionKey`) — để tối ưu sau khi pool đã ổn định.
- **Không** tự upgrade patch TimesTen lên bản mới nhất mà không hỏi Oracle Support trước — release notes 22.1.1.29.0 (13/07/2026, tức ~1 tuần trước) đã **deprecate TimesTen Scaleout**, chỉ còn hỗ trợ TimesTen Classic. Cần xác nhận patch level hiện tại của cụm production và **đóng băng** version cho tới khi có quyết định roadmap rõ ràng với Oracle. Đây là rủi ro chiến lược cần escalate cho DBA/architecture team, độc lập với việc đổi pool.

---

## Workstream A — Đổi pool HikariCP → UCP (ưu tiên, giải quyết sự cố hiện tại)

### A0. Discovery & baseline (làm trước khi đổi bất cứ gì)

- Chụp lại baseline hiện tại để so sánh sau cutover: log HikariCP ở DEBUG trong ít nhất 1 chu kỳ tải cao/thấp, ghi nhận: tần suất "(connection is dead)", tần suất tạo connection mới/phút, pool size hiện tại (`maximum-pool-size`, `minimum-idle`, `idle-timeout`, `max-lifetime`, `keepalive-time`).
- Xác nhận patch level TimesTen Scaleout hiện tại (`ttGridAdmin dbStatus` hoặc tương đương) và driver `ttjdbc*.jar` version đang dùng.
- Kiểm tra `ulimit -n` trên app host và trên từng data instance host — vì release notes 22.1 ghi nhận TimesTen Scaleout giữ ít nhất 1 file descriptor/connection, cộng thêm khi commit/rollback (BugDB #25815090). Nếu pool đang churn connection nhiều, đây có thể là yếu tố cộng hưởng.

### A1. Code changes

Tham khảo trực tiếp cấu trúc trong `ucp-timesten-demo`:

- Thêm dependency `com.oracle.database.jdbc:ucp`, loại trừ `com.zaxxer:HikariCP` khỏi `spring-boot-starter-jdbc` (như `pom.xml` trong demo).
- Tạo `DataSource` bean build từ `PoolDataSourceFactory.getPoolDataSource()` với `connectionFactoryClassName = com.timesten.jdbc.TimesTenDataSource` (theo mẫu `TimesTenUcpConfig.java`), **giữ nguyên toàn bộ JPA/Hibernate/Repository hiện có** — chỉ thay bean `DataSource` phía dưới, code tầng trên không đổi.
- **Cơ chế rollback nhanh** (do đã chọn cutover toàn bộ, không canary): định nghĩa 2 `@Configuration` loại trừ nhau qua Spring Profile — `hikari` (giữ code cũ, có thể phục hồi bằng đúng 1 dependency Hikari vẫn còn trong classpath ở lần deploy đầu) và `ucp` (config mới). Switch giữa 2 profile chỉ là đổi 1 biến môi trường (`SPRING_PROFILES_ACTIVE`) + restart, **không cần redeploy artifact mới** — quan trọng để rollback trong vài phút nếu UCP có sự cố ở production, thay vì phải chờ CI/CD build lại bản cũ.
- Cấu hình timeout: áp dụng đúng khuyến nghị đã rút ra — **không** dùng `TimeToLiveConnectionTimeout`/`AbandonedConnectionTimeout` làm cơ chế chính (có thể đóng cưỡng bức connection đang dùng dở, gây `SQLException` giữa transaction); chỉ dựa vào `InactiveConnectionTimeout` + `MaxConnectionReuseTime` (an toàn, chỉ tác động connection rảnh/đã trả về pool). Đặt `time-to-live-timeout = 0` (disable) trừ khi có nhu cầu cụ thể chặn leak.
- Pool sizing: tính lại theo traffic thật (không copy nguyên số từ Hikari) — dùng công thức `connections = ((core_count * 2) + effective_spindle_count)` làm điểm khởi đầu, rồi điều chỉnh theo `getBorrowedConnectionsCount()`/`getAvailableConnectionsCount()` quan sát được trong A2. Với in-memory DB như TimesTen, số connection cần có thể thấp hơn nhiều so với DB disk-based vì latency mỗi query rất thấp.

### A2. Validate ở staging (bắt buộc trước khi động prod)

- Load test staging với traffic pattern giống production nhất có thể (replay traffic thật nếu có công cụ, hoặc traffic tổng hợp theo QPS/burst pattern thực tế).
- **Chaos test bắt buộc**: chủ động gây transient failure ở 1 trong các data instance của grid staging (dừng 1 element, hoặc trigger rebalance/failover thủ công) trong lúc đang load test, quan sát:
  - Pool có phục hồi về đúng min-pool-size sau khi grid ổn định lại không.
  - Ứng dụng có nhận `SQLSTATE TT005`/`ORA-57005` (transient error) và có retry logic phù hợp không (đây là hành vi TimesTen kỳ vọng, không phải bug).
  - So sánh số liệu với baseline A0: tần suất tạo connection mới phải giảm rõ rệt so với HikariCP.
- Test rollback thật (không chỉ trên giấy): switch profile `ucp` → `hikari` trên staging, xác nhận app phục hồi bình thường trong khung thời gian chấp nhận được.
- Sign-off từ DBA/team vận hành grid trước khi lên lịch cutover production.

### A3. Cutover production

- Chọn khung giờ thấp điểm.
- Checklist trước khi bắt đầu: baseline A0 đã chụp, rollback đã test ở A2, alerting đã set (xem A4), người trực sẵn sàng theo dõi trong khung giờ cutover + ít nhất vài giờ sau.
- Deploy artifact có cả 2 profile (`hikari` cũ + `ucp` mới), active profile = `ucp`.
- Theo dõi sát ngay sau cutover: pool stats (`getBorrowedConnectionsCount`/`getAvailableConnectionsCount` tương tự endpoint `/db/pool-stats` trong demo), tỷ lệ lỗi request, latency p99.
- Nếu bất thường vượt ngưỡng đã định trước (xem A4) → rollback ngay bằng đổi profile, không cố sửa tại chỗ trong giờ cutover.
- Sau khi ổn định qua ít nhất 1 chu kỳ tải cao điểm đầy đủ (thường là 1 ngày làm việc) → gỡ profile `hikari`/dependency Hikari ở lần deploy tiếp theo để dọn dẹp.

### A4. Monitoring & success criteria

- Metric cần thêm/theo dõi: số connection borrowed/available theo thời gian thực (expose qua actuator/metrics tương tự `PingController` trong demo), tần suất connection bị đóng và lý do (map log UCP tương đương log DEBUG HikariCP trước đó), error rate theo `SQLSTATE`, latency p50/p99 tầng DB.
- Alert: đặt ngưỡng cho "borrowed connections gần chạm max-pool-size kéo dài" và "connection creation rate bất thường" — đây chính là triệu chứng ban đầu cần theo dõi để xác nhận đã hết.
- Success criteria: tần suất tạo connection mới giảm về mức tương đương "steady state" (không còn liên tục tạo mới ngoài lúc pool warm-up hoặc grid thật sự có sự cố), không tăng latency/error rate so với baseline.

---

## Workstream B — Đánh giá lại tầng JPA/Hibernate (làm sau, có profiling dẫn đường)

Không làm cùng lúc với Workstream A. Sau khi pool ổn định:

1. Profiling thực tế (không đoán): bật SQL logging/APM để tìm N+1 query, transaction giữ connection lâu do OSIV, endpoint nào có latency cao bất thường so với kỳ vọng của TimesTen (in-memory nên latency phải rất thấp).
2. Chỉ migrate sang `JdbcTemplate`/MyBatis cho **các query/endpoint hot-path** xác định được ở bước 1, không rewrite toàn bộ ứng dụng — giữ nguyên phần còn lại trên JPA nếu nó không phải bottleneck.
3. Xác nhận với DBA/Oracle liệu Hibernate dialect đang dùng (nếu có custom dialect nào đó đang chạy) có thật sự sinh đúng SQL tương thích TimesTen 22.1, hay đang "may mắn chạy được" — nếu không chắc, đây là rủi ro cần biết trước khi mở rộng thêm tính năng dựa trên JPA.

---

## Files/artifacts liên quan

- Reference implementation: `/Users/hoanvt/Project/ucp-timesten-demo` (toàn bộ cấu trúc `TimesTenUcpConfig`, `TimesTenUcpProperties`, `application.yml` mẫu, bảng mapping tham số Hikari↔UCP trong `README.md`).
- Production repo thật: chưa có trong môi trường làm việc hiện tại — khi bắt đầu implement cần xác định đúng file đang cấu hình Hikari (thường là `application.yml`/`application.properties` + có thể 1 `@Configuration` class nếu đã custom) để áp dụng theo mẫu tương ứng.

## Verification tổng thể

- Workstream A coi là xong khi: staging pass chaos test (A2), production đã qua ít nhất 1 chu kỳ tải cao điểm không rollback, tần suất tạo connection mới về steady state theo A4.
- Workstream B coi là xong khi: các hot-path đã xác định qua profiling được migrate và đo lại latency cải thiện rõ so với baseline JPA.
