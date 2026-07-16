# gRPC & Kafka Performance Tuning Checklist

Checklist cải thiện performance cho luồng gRPC (BB → TO → ASM, Java + Istio) và
Kafka producer (produce mỗi insert/update DB), đúc kết từ POC thực nghiệm trong repo
này (`grpc-channel-poc/` và `kafka-produce-poc/`). Mỗi mục ghi rõ **đã đo được gì** để
không tối ưu mù.

> **Nguyên tắc xuyên suốt:** các đòn bẩy gRPC/connection bên dưới chỉ đáng **~35%
> throughput** (đã đo). Chúng **không** thể tạo ra hay xóa bỏ latency cỡ **giây**. Nếu
> APM thấy spike cỡ giây → nguyên nhân nằm ở P0, không phải ở channel/connection.

---

## Bằng chứng thực nghiệm (POC này, Docker Desktop k8s + Istio, JDK)

| Đo được | Kết quả | Ý nghĩa |
|---|---|---|
| Single channel vs Pool x4-8, tải cao (delay=0, 500 conc) | +~35% throughput, p50 ~2x tốt hơn | Pool giúp thật nhưng **modest** |
| Sweep pool size 1→16 | Đỉnh ở **pool=4**, sau đó đi ngang | Sweet spot ~4; lớn hơn vô ích vì sidecar→server coalesce về `cx_active=2` |
| Request đầu (JVM nguội) vs steady-state | **~80ms vs <1ms** (~80x) | Cold-start là vấn đề thật cho luồng critical |
| `maxConcurrentCallsPerConnection` = 100 vs không giới hạn | Không khác biệt | Giả thuyết "nghẽn do chạm trần stream" **SAI** |
| `http2MaxRequests: 50` (DestinationRule) | 1650/2000 request lỗi `overflow` | Nếu ai lỡ set thấp → bug thật, gây lỗi ngay |
| "4x" ban đầu (p99 57ms vs 246ms) | **Không tái lập được** | Artifact JIT warmup, không phải tín hiệu thật |

---

## P0 — Đo trước khi tối ưu (BẮT BUỘC làm đầu tiên)

- [ ] Tách latency trong APM: thời gian gọi RPC ở client vs xử lý ở server (TO/ASM) vs network
- [ ] Đối chiếu timestamp spike "vài giây" với **GC log** của app JVM (producer/TO/ASM)
- [ ] Đối chiếu timestamp spike với thời điểm **deploy / scale-up** (nghi cold-start)
- [ ] Đo **RTT thực tế giữa các AZ** nếu deploy multi-AZ (không tin số lý thuyết của cloud)
- [ ] Kết luận: nếu spike cỡ giây → xử lý ở P0 (GC / server chậm / network), **KHÔNG** phải P2-P4

## P1 — Warmup lúc startup (ROI cao nhất; đã đo request đầu ~80ms)

- [ ] Gọi thật vài trăm-nghìn request nội bộ lúc startup (warm connection + JIT)
  - [ ] Ép mở connection sớm: `channel.getState(true)`
  - [ ] Nếu dùng ChannelPool: warmup **từng channel** trong pool (mỗi cái 1 connection riêng)
- [ ] **Chặn readiness probe** cho tới khi warmup xong (nếu không, traffic vào lúc còn nguội)
  - [ ] Spring: publish `ReadinessState.ACCEPTING_TRAFFIC` sau khi warmup xong
  - [ ] k8s readinessProbe trỏ `/actuator/health/readiness`
- [ ] Istio annotation `proxy.istio.io/config: '{"holdApplicationUntilProxyStarts": true}'`
      (để app không warmup vào lúc sidecar chưa sẵn sàng)

## P2 — ChannelPool cho luồng tải cao (đã đo: +~35%, sweet spot ~4)

- [ ] Xác định luồng nào có **request rate cao** thật sự (không cần pool cho luồng thưa)
- [ ] Thêm ChannelPool cho luồng đó — dùng `gax-java` `ChannelPool` hoặc mẫu `ChannelPool.java` trong repo
- [ ] Đặt pool size **~4** (điểm bão hòa; lớn hơn không thêm throughput vì `cx_active` sidecar→server = 2)
- [ ] Mỗi channel phải có **channel arg khác nhau** để gRPC không dedupe về chung 1 connection
- [ ] Xác nhận qua JMX: mỗi channel có `client.id` riêng, metric buffer riêng

## P3 — Keepalive tránh cold-start lặp lại

- [ ] Cấu hình `NettyChannelBuilder.keepAliveTime(...)` để connection không bị đóng lúc idle
- [ ] Tránh mỗi đợt traffic thưa lại trả chi phí setup connection ~80ms
- [ ] Kiểm tra pool ở tầng app không đóng-mở connection theo từng request

## P4 — Tune Istio DestinationRule (chỉ khi cần phá trần cx=2)

- [ ] Kiểm tra `http2MaxRequests` **KHÔNG** bị set quá thấp so với traffic thật
      (đã chứng minh: set thấp → lỗi `overflow` hàng loạt — nếu có, sửa ngay)
- [ ] `maxRequestsPerConnection` để ép sidecar mở thêm connection tới server
      (**đã test: giúp ít** — chỉ đáng làm nếu kết hợp ChannelPool và đang chạm trần throughput thật)
- [ ] Kiểm tra metric `upstream_rq_pending_overflow` trên sidecar TO/ASM để xác nhận có nghẽn ở Envoy không
      (cần bật `proxyStatsMatcher` mới thấy stat này trong `/stats`)

## Việc KHÔNG nên làm

- [ ] ❌ Đừng thay gRPC bằng Aeron/SBE... trừ khi đã vắt kiệt P0-P4 (overhead là của kiến trúc quanh gRPC, không phải bản thân gRPC)
- [ ] ❌ Đừng tin 1 lần benchmark — **luôn warmup + đo lặp nhiều lần** (chính POC này suýt kết luận sai 4x do nhiễu warmup)
- [ ] ❌ Đừng cố tăng `cx_active` như mục tiêu tự thân — nó chỉ là hệ quả, không phải nguyên nhân

---

# Phần 2: Kafka Producer Tuning

> **Nguyên tắc xuyên suốt (khác gRPC):** "chục giây" async produce **KHÔNG** xảy ra chỉ vì
> volume cao. Đã đo thực nghiệm: cần **CẢ HAI** — buffer.memory cạn **VÀ** broker đang
> chậm/quá tải — cùng lúc. Thiếu 1 trong 2, latency chỉ tăng nhẹ (~10-20ms), không đủ ra
> chục giây.

## Bằng chứng thực nghiệm (POC `kafka-produce-poc/`, broker throttle 0.1 CPU để mô phỏng quá tải)

| Đo được | Kết quả | Ý nghĩa |
|---|---|---|
| Buffer cạn, broker khỏe | max ~130ms | Không đủ gây "chục giây" một mình |
| Buffer cạn + broker throttle CPU 0.1 core | p99=993ms, max=1406ms | **Tái hiện đúng** hiện tượng — cần cả 2 điều kiện |
| `acks=1` vs `acks=all` (broker throttle) | acks=1 **tệ hơn** (p99=1697ms vs 993ms) | acks không giúp khi bottleneck là CPU broker, chỉ giúp khi bottleneck là chờ replica ack |
| `compression=lz4` (broker throttle, payload random) | tệ hơn (p99=1798ms) | Payload test không nén được — **không kết luận được** cho payload JSON thật |
| `linger.ms=20` + `batch.size=64KB` (broker throttle) | **Tốt nhất**: p99=902ms, max=999ms, throughput cao nhất | Batching giảm số request → giảm tải CPU broker |

---

## KP0 — Xác nhận broker có đang chậm/quá tải không (BẮT BUỘC làm đầu tiên)

- [ ] Kiểm tra CPU/disk I/O broker tại đúng timestamp xảy ra "chục giây"
- [ ] Kiểm tra `UnderReplicatedPartitions` (phải = 0), `IsrShrinksPerSec`
- [ ] Kiểm tra GC log của **broker** JVM
- [ ] Nếu broker hoàn toàn khỏe → quay lại nghi vấn **GC pause của chính app JVM (producer)**,
      không phải Kafka
- [ ] Xem 2 metric producer qua JMX: `buffer-available-bytes` (gần 0 lúc slow = buffer cạn),
      `bufferpool-wait-time-total` (tăng vọt = bằng chứng trực tiếp)

## KP1 — Xử lý gốc rễ: vì sao broker chậm (ROI cao nhất, mọi thứ khác chỉ giảm nhẹ triệu chứng)

- [ ] Broker có đang thiếu tài nguyên (CPU/disk/memory) so với traffic thật không
- [ ] Có hot partition / key skew dồn tải vào 1 broker cụ thể không
- [ ] Có đang trong quá trình rebalance/leader election khi xảy ra spike không

## KP2 — Tune `linger.ms` + `batch.size` (đã đo: đòn bẩy hiệu quả nhất)

- [ ] Tăng `linger.ms` lên 10-20ms (đánh đổi latency nhỏ lấy giảm số request tới broker)
- [ ] Tăng `batch.size` (VD 64KB) tương ứng để chứa được batch lớn hơn
- [ ] Đo lại: giảm p99 lẫn tăng throughput cùng lúc là dấu hiệu đúng hướng

## KP3 — `acks` — CHỈ đổi khi xác nhận đúng bottleneck (đã đính chính, không phải mặc định làm)

- [ ] **Không** tự động đổi `acks=1` — chỉ giúp khi bottleneck là **chờ replica ack** trong
      cluster nhiều broker, đã đo: **không giúp** khi bottleneck là broker CPU-bound
- [ ] Đo `request-latency-avg` trước/sau đổi acks để xác nhận có thật sự cải thiện không,
      đừng đổi theo cảm tính

## KP4 — Compression — đo lại bằng payload thật, đừng tin theo POC

- [ ] Test `compression.type=lz4/zstd` với **payload JSON/protobuf thật** của bạn (không phải
      random bytes như POC) — độ nén ảnh hưởng quyết định tới việc có lợi hay không
- [ ] Nếu payload nén tốt (60-80%+) → giảm cả bytes truyền lẫn tải cho broker → nên dùng
- [ ] Nếu payload đã compact/binary sẵn (VD đã protobuf) → khả năng cao không đáng compress thêm

## KP5 — Giảm volume tận gốc (nếu produce mỗi insert/update là quá nhiều)

- [ ] Cân nhắc **outbox pattern**: ghi message vào 1 bảng DB trong cùng transaction, 1 tiến
      trình riêng đọc bảng đó và produce theo batch — tách hẳn Kafka khỏi đường xử lý nghiệp vụ
- [ ] Xác nhận produce đang nằm **ngoài** transaction DB (đã xác nhận đúng với hệ thống này)

## Việc KHÔNG nên làm (Kafka)

- [ ] ❌ Đừng đổi `acks=1` chỉ vì nghĩ "luôn nhanh hơn" — đã đo ngược lại khi bottleneck là broker CPU
- [ ] ❌ Đừng tin compression tốt/xấu từ POC này — payload test không đại diện, tự đo lại
- [ ] ❌ Đừng tối ưu producer config trước khi xác nhận broker có vấn đề gì không (KP0)
- [ ] ❌ Đừng tin 1 lần benchmark — cùng bài học như gRPC POC, warmup + đo lặp nhiều lần

---

## Snippet tham khảo (Kafka)

### Producer config đề xuất (sau khi xác nhận broker khỏe, chỉ cần giảm request overhead)

```properties
linger.ms=20
batch.size=65536
# acks giữ nguyên theo yêu cầu durability nghiệp vụ, KHÔNG đổi chỉ để "cho nhanh"
acks=all
```

### Metric JMX cần theo dõi

```
buffer-available-bytes        # gần 0 lúc slow = buffer cạn
bufferpool-wait-time-total    # tăng vọt = bằng chứng trực tiếp thread đang chờ buffer
record-queue-time-max         # record nằm chờ trong buffer bao lâu
request-latency-avg           # broker ack nhanh/chậm — tăng cao = broker đang chậm
record-retry-rate             # >0 = đang retry ngầm do broker
```

---

## Snippet tham khảo (gRPC)

### Warmup + readiness gating (Spring Boot)

```java
@Component
public class WarmupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        channel.getState(true); // ép mở connection
        var stub = EchoServiceGrpc.newBlockingStub(channel);
        for (int i = 0; i < 2000; i++) {
            try { stub.echo(EchoRequest.newBuilder().setMessage("warmup").build()); }
            catch (Exception ignored) {}
        }
        AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
    }
}
```

### DestinationRule (chỉ khi cần)

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: to-service
spec:
  host: to-service.<ns>.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http2MaxRequests: 1000          # đủ cao cho peak, KHÔNG set thấp
        maxRequestsPerConnection: 0     # 0 = giữ connection lâu cho gRPC
```

---

## Tóm tắt 1 dòng — cả 2 phần

**gRPC**: Đo trước (P0) → warmup + readiness gating (P1) → ChannelPool ~4 cho luồng nóng (P2) → keepalive (P3).
**Kafka**: Xác nhận broker có chậm không (KP0) → xử lý gốc rễ broker (KP1) → tune linger/batch (KP2), **không** phải acks.
Cả 2: nếu latency cỡ giây, nguyên nhân gần như luôn nằm ở bước đo trước (P0/KP0) — GC pause,
broker/server chậm thật, hoặc network — không phải channel pool hay producer config.
