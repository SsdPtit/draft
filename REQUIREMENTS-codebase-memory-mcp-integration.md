# Yêu cầu tích hợp knowledge graph (codebase-memory-mcp + code-review-graph) vào PR Review Tool

## 1. Mục tiêu

Bổ sung vào **bước 3** của luồng review PR hiện có **3 nguồn đánh giá impact chạy song song, độc lập, không gộp ở tầng dữ liệu**:

1. **Dependency graph nội bộ hiện có** (giữ nguyên, không thay thế) → `existing_score`
2. **`code-review-graph`** — đánh giá risk **trong phạm vi 1 service** (blast radius, test gap, security keyword, churn...) → `crg_risk_score`
3. **`codebase-memory-mcp`** — chỉ dùng cho đúng 1 việc: đánh giá **impact xuyên service** (HTTP/gRPC/pub-sub call giữa các service) → `cross_service_impact`

Đồng thời chuẩn hoá cách gọi `kiro-cli` (bước 6-8) theo mô hình headless, zero-tool, deterministic.

### 1.1 Tách thành 2 tính năng riêng trên UI, cùng 1 backend service

Đây là **2 tính năng độc lập mà user có thể chủ động chọn dùng riêng** trên UI — không phải lúc nào cũng đi chung 1 luồng:

- **Tính năng 1 — "Đánh giá ảnh hưởng"**: user bấm để xem ngay `impact_context` (3 điểm số + test gap + escalation test coverage), **không cần chờ LLM review**. Dùng khi chỉ muốn check nhanh trước khi quyết định có review kỹ hay không.
- **Tính năng 2 — "Review PR"**: luồng đầy đủ (baseline mục 2) — vẫn tự động gọi Module A bên trong để làm giàu context cho LLM, y hệt thiết kế hiện tại, user không cần tự bấm "Đánh giá ảnh hưởng" trước.

**Về backend: vẫn 1 service, KHÔNG tách deploy riêng** — chỉ cần **2 API endpoint khác nhau, cùng gọi vào 1 Module A dùng chung**:

```
┌─────────────────────────────┐
│  MODULE A: Impact Assessment │
│  (mục 4 + mục 9)             │
│  - existing_score             │
│  - crg_risk_score              │
│  - cross_service_impact        │
│  - test_gaps + escalation      │
└──────────┬──────────────┬─────┘
           │              │
   gọi trực tiếp   gọi trực tiếp
   (nội bộ)        (nội bộ)
           │              │
┌──────────▼─────┐  ┌─────▼──────────────────────┐
│ API endpoint 1: │  │ API endpoint 2:            │
│ POST /impact-   │  │ POST /pr-review             │
│ assessment      │  │ (baseline mục 2, tự động    │
│ (UI: nút "Đánh  │  │ gọi Module A bên trong ở    │
│ giá ảnh hưởng") │  │ bước 3, rồi mới sang        │
│ → trả thẳng     │  │ Module B/LLM review)         │
│ impact_context, │  │ (UI: nút "Review PR")        │
│ KHÔNG gọi LLM    │  │                              │
└─────────────────┘  └─────────────────────────────┘
```

**Vì sao không cần 2 service riêng để có 2 tính năng UI riêng:** UI chỉ cần gọi đúng API endpoint tương ứng với nút bấm — cả 2 endpoint đều nằm trong cùng backend, cùng gọi vào Module A (không viết 2 lần). Tách deploy chỉ cần thiết khi có lý do khác (scale riêng, team khác quản lý...) — hiện tại chưa có lý do đó nên giữ 1 service cho đơn giản vận hành.

**Endpoint `/impact-assessment` (tính năng 1) trả kết quả theo 2 giai đoạn** — khớp đúng thiết kế bất đồng bộ đã có ở mục 9.3: trả `existing_score`/`crg_risk_score`/`cross_service_impact` ngay (đồng bộ, nhanh), phần escalation test coverage (mục 9, nếu bị trigger) cập nhật sau — UI cần có cơ chế poll/refresh trạng thái cho phần này, không phải 1 lần gọi là có đủ toàn bộ kết quả.

## 2. Luồng review PR hiện tại (baseline)

```
1. Server tìm PR theo thông tin user nhập
2. Server gọi git-mcp get diff để lấy các file thay đổi
3. Service đánh giá ảnh hưởng — HIỆN TẠI: theo dependency graph nội bộ, tính context score
   ────────────────────────────────────────────────────────────────
   THÊM MỚI: chạy code-review-graph (risk trong service) + codebase-memory-mcp
   (impact xuyên service) SONG SONG, độc lập với nhau và với hệ thống cũ (mục 4)
   ────────────────────────────────────────────────────────────────
4. Server đọc các file knowledge và skill, ghép string
   ────────────────────────────────────────────────────────────────
   SỬA: lọc theo domain của PR + score-based depth thay vì ghép nguyên văn tất cả (mục 5)
   ────────────────────────────────────────────────────────────────
5. Server build prompt = skill + knowledge + impact context + text + user context bổ sung
   ────────────────────────────────────────────────────────────────
   SỬA: gộp thành 1 context JSON file duy nhất, nạp qua `resources` (mục 5-6)
   ────────────────────────────────────────────────────────────────
6. Gửi prompt cho kiro-cli
   ────────────────────────────────────────────────────────────────
   SỬA: gọi `kiro-cli chat --no-interactive --agent <config sinh riêng mỗi lần>` (mục 6)
   ────────────────────────────────────────────────────────────────
7. AI agent review
8. Nhận kết quả, parse JSON, lưu session store + history
   ────────────────────────────────────────────────────────────────
   SỬA: parse theo marker cố định + retry khi validate lỗi (mục 7)
   ────────────────────────────────────────────────────────────────
```

## 3. Ràng buộc kiến trúc (đã chốt)

- **Agent (kiro-cli, bước 7) tuyệt đối 0 tool** — `"tools": []` tường minh trong agent config, không chỉ để `allowedTools` trống. Verify bằng `/tools` phải hiện danh sách rỗng (mục 11, acceptance criteria).
- Mọi dữ liệu (diff, impact context từ cả 3 nguồn, knowledge, skill, user context) phải được **Server/Service resolve xong hoàn toàn**, gộp vào **1 file JSON duy nhất**, nạp vào agent qua field `resources` — không dùng tool để agent tự đọc, không dùng stdin.
- Cả `code-review-graph` và `codebase-memory-mcp` đều gọi bằng **CLI mode** (subprocess) trong Service, cùng deterministic layer với `git-mcp get diff` — không dùng MCP/JSON-RPC, vì agent review không được tự gọi tool.
- **Không ghép graph/dữ liệu giữa `code-review-graph` và `codebase-memory-mcp`** — 2 tool có schema SQLite nội bộ hoàn toàn khác nhau, không tương thích, không có API import/export chính thức giữa chúng. Mỗi tool build graph riêng, chạy độc lập, chỉ gộp **kết quả cuối** (số điểm/thông tin) ở tầng Service — không gộp ở tầng dữ liệu.
- **Không thay thế dependency graph nội bộ hiện có** — vẫn giữ `existing_score` như cũ.
- Vì `resources` nạp toàn bộ nội dung file mỗi lần (không có retrieval on-demand) — bài toán "prompt quá dài" giải quyết bằng cách giảm dữ liệu trước khi ghi context file (mục 5), không giao cho agent tự retrieval (loại bỏ phương án Kiro knowledge base — vi phạm rule 0-tool).

### 3.1 Ghi chú riêng cho Node.js (Service/Server hiện tại viết bằng Node)

Toàn bộ lệnh CLI/script trong tài liệu này (bash mẫu, wrapper Python ở mục 10.2) đều gọi qua **subprocess** — không có tool nào trong đây có SDK/binding native cho Node, nên cách gọi thống nhất là spawn process con:

- **Dùng `child_process.execFile`/`spawn` với mảng argument, KHÔNG dùng `exec` với string nối trực tiếp** — `exec` chạy qua shell, dễ dính command injection nếu tham số (tên hàm lấy từ code, tên branch, PR title...) chứa ký tự đặc biệt do PR author tự đặt tên. `execFile`/`spawn` truyền argument dạng mảng, không qua shell, an toàn hơn hẳn cho đúng use case này (dữ liệu đầu vào một phần đến từ nội dung PR — không hoàn toàn tin cậy).
- **Server cần cài sẵn Python runtime** (cho `code-review-graph` + wrapper `crg_get_flows.py` ở mục 10.2) — đây là dependency **ngoài** stack Node bình thường, cần thêm vào yêu cầu hạ tầng deploy (Dockerfile/base image nếu có, hoặc cài trực tiếp trên máy chủ) — không tự có sẵn chỉ vì Server là Node.
- **Concurrency (mục 6.4, 9.5)**: dùng queue/semaphore phía Node (vd thư viện `p-queue`, hoặc tự đếm số child process đang chạy) để giới hạn số lời gọi `kiro-cli`/escalation đồng thời — không có yêu cầu công nghệ cụ thể nào khác ngoài đảm bảo đúng giới hạn đã nêu.
- **Bất đồng bộ (mục 9.3, "Pha 2 chạy async")**: khớp tự nhiên với event loop của Node — dùng Promise/async-await thông thường, không cần thêm hạ tầng queue phức tạp (Redis/RabbitMQ...) trừ khi khối lượng PR thực tế đòi hỏi, không phải yêu cầu bắt buộc của tài liệu này.

## 4. [Module A: Impact Assessment] Chi tiết implement bước 3

### 4.1 Chuẩn bị code local đúng trạng thái PR

`git-mcp get diff` (bước 2) cần trả về: **base SHA, head SHA, danh sách file thay đổi**. Dùng SHA (không dùng tên branch) để checkout — tránh rủi ro merge-base sai nếu base branch chưa fetch mới nhất.

```bash
git fetch origin "$BASE_SHA" "$HEAD_SHA"
git checkout "$HEAD_SHA"
```

### 4.2 `code-review-graph` — risk score trong phạm vi service (thay cho công thức tự viết tay)

**Cài đặt (Server không có mạng — làm 1 lần, không phải mỗi PR):**

`pip install` cần PyPI, không chạy được trên Server offline — quy trình bootstrap tương tự đã làm cho máy Windows offline trước đây:

```bash
# 1. Trên máy CÓ mạng, cùng OS/kiến trúc/version Python với Server thật:
pip download code-review-graph -d ./crg-offline-bundle \
  --platform <manylinux_x86_64 hoặc tương ứng OS Server> \
  --python-version <version Python trên Server> \
  --only-binary=:all:
# Cờ --only-binary=:all: bắt buộc — ép chỉ lấy wheel biên dịch sẵn, tránh trường hợp
# 1 dependency không có wheel sẵn cho platform đó và pip phải build từ source (cần compiler).

# 2. Chuyển thư mục crg-offline-bundle sang Server (không qua mạng — USB/artifact nội bộ/registry riêng)

# 3. Trên Server, cài từ local, không đụng PyPI:
pip install --no-index --find-links=./crg-offline-bundle code-review-graph
```

**Không cài nhóm dependency optional** (`embeddings`, `google-embeddings`, `wiki`) — các nhóm này kéo theo thư viện nặng hoặc cần gọi cloud, không cần thiết cho `detect-changes`/risk score (đã verify: dependency core — `mcp`, `fastmcp`, `tree-sitter`, `tree-sitter-language-pack`, `networkx`, `watchdog` — không có gì cần compiler, chỉ cần đúng wheel platform).

**Bắt buộc tạo `.code-review-graphignore` cho mỗi repo trước khi build lần đầu** — đã verify thực nghiệm: pattern ignore mặc định `**/vendor/**` không khớp các biến thể tên khác (vd `vendored`, `third_party`, `external`...). Nếu repo có thư mục vendor/generated với tên không chuẩn, graph có thể bùng nổ edge (đã gặp thực tế: 24.7 triệu edge từ vài file generated, build mất 11 phút, dùng 17.8GB RAM — so với 203 nghìn edge, 6.5 giây sau khi thêm ignore đúng). **Luôn kiểm tra `code-review-graph status` sau lần build đầu tiên trên mỗi repo mới** — nếu số edge bất thường (vài triệu trở lên cho repo cỡ vừa), rà lại ignore file trước khi đưa vào pipeline chính thức.

**Build/update graph (mỗi lần review, tại đúng HEAD_SHA):**

```bash
code-review-graph update --repo "$REPO_PATH"    # incremental, nhanh — build lần đầu dùng `build`
```

**Lấy risk score + test gap:**

```bash
code-review-graph detect-changes --repo "$REPO_PATH" --base "$BASE_SHA"
```

**Lưu ý quan trọng:** CLI `detect-changes` **không có flag nhận danh sách file trực tiếp** (đã verify: `--help` chỉ có `--base/--brief/--repo/--churn/--verify`) — tool vẫn tự tính `git diff` nội bộ dựa trên `--base`, không phải "nhận file list từ bước 2" như cách hiểu ban đầu. Điều này **vẫn an toàn** vì `--base` ở đây được truyền `$BASE_SHA` (SHA bất biến, không phải tên branch) — rủi ro merge-base sai (đã bàn ở phần git-mcp) chỉ xảy ra với tên branch có thể "trôi", không xảy ra khi so sánh 2 SHA cố định. Luôn truyền `$BASE_SHA`/`$REPO_PATH` tường minh, không dựa vào auto-detect cwd của tool trong môi trường pipeline tự động.

Output mẫu (đã verify thật trên chính repo `codebase-memory-mcp`):
```json
{
  "risk_score": 0.8,
  "changed_functions": [
    {"name": "bind_text", "qualified_name": "...", "is_test": false, "risk_score": 0.65}
  ],
  "test_gaps": [...]
}
```

`risk_score` thang 0.0-1.0 → nhân 100 để cùng thang với `existing_score`.

### 4.3 `codebase-memory-mcp` — CHỈ dùng cho impact xuyên service

Vai trò thu hẹp so với thiết kế trước: không cần tính risk score trong-service nữa (đã có `code-review-graph` làm tốt hơn) — chỉ cần trả lời câu hỏi **"thay đổi này có ảnh hưởng tới service khác qua HTTP/gRPC/pub-sub không, và cụ thể ảnh hưởng tới function/route nào bên service kia"**.

**Sửa quan trọng so với thiết kế trước — phải dùng đúng `CROSS_*` edge, không phải edge nội bộ:**

Theo README `codebase-memory-mcp`, `HTTP_CALLS`/`ASYNC_CALLS`/`EMITS`/`LISTENS_ON` là edge **trong phạm vi 1 project**; để nối được across 2 service (2 repo khác nhau), cần dùng edge riêng `CROSS_HTTP_CALLS`/`CROSS_ASYNC_CALLS`/`CROSS_CHANNEL` — **chỉ sinh ra khi index nhiều project vào CÙNG 1 store và chạy mode `cross-repo-intelligence`**. Nếu chỉ query `HTTP_CALLS` trên 1 project index riêng lẻ như thiết kế trước, sẽ không bắt được lời gọi thật sự đi sang service khác.

**Do đó cần tách thành 2 tiến trình khác nhịp độ:**

**(a) Job nền — chạy định kỳ, KHÔNG phải mỗi PR** (vd mỗi giờ, hoặc trigger khi có merge vào main của bất kỳ service nào trong 11 service):
```bash
# Index toàn bộ 11 service + 4 lib vào CÙNG 1 store (mỗi service = 1 project riêng, cùng CBM_CACHE_DIR)
for repo in "${ALL_SERVICE_PATHS[@]}"; do
  $CBM_EXE_PATH cli index_repository --repo-path "$repo" --mode fast
done

# Sau khi tất cả đã index, chạy riêng 1 lượt để build CROSS_* edges giữa chúng:
$CBM_EXE_PATH cli index_repository --mode cross-repo-intelligence --target-projects '["*"]'
# LƯU Ý: chưa verify chắc chắn --repo-path còn bắt buộc ở mode này hay không (theo mô tả
# tool "skip extraction, chỉ match Route/Channel" — cần Kiro tự chạy --help xác nhận
# trước khi đưa vào production, tránh giả định sai cú pháp).
```

**(b) Per-PR — chỉ re-index đúng service đang review** (tại HEAD_SHA của PR), rồi query dựa trên `CROSS_*` edges đã có sẵn từ job nền (a):

```bash
$CBM_EXE_PATH cli index_repository --repo-path "$REPO_PATH" --mode fast
PROJECT=$($CBM_EXE_PATH cli list_projects | jq -r --arg p "$REPO_PATH" '.projects[] | select(.root_path==$p) | .name')

# Query CROSS_* edges (không phải bản nội bộ) — other.qualified_name có tiền tố
# <project>.<path>.<name>, từ đó suy ra được service/repo đích:
$CBM_EXE_PATH cli query_graph --project "$PROJECT" \
  --query "MATCH (f:Function {name: '$SYMBOL_NAME'})-[:CROSS_HTTP_CALLS|CROSS_ASYNC_CALLS|CROSS_CHANNEL]-(other) RETURN other.name, other.qualified_name, other.file_path"
```

Nếu có kết quả → symbol này tham gia vào 1 lời gọi xuyên service thật sự. `other.qualified_name` cho biết chính xác function/route đích + service nào (mục 9.4 dùng tiếp thông tin này để escalate test coverage sang service đó).

**CHƯA VERIFY — cần Kiro tự kiểm chứng trước khi implement:** câu query trên trả về `other` là node kết nối trực tiếp qua `CROSS_HTTP_CALLS` — chưa xác nhận được `other` là node `Function` (handler) hay node `Route` riêng (theo README, `Route` là 1 label độc lập trong graph). Nếu `other` là `Function`, cần thêm 1 hop nữa (`(other)<-[:HANDLES]-(r:Route)` hay tương tự) mới lấy được path HTTP thật (vd `"POST /charge"`) để điền vào field `route` trong schema (mục 4.4/5.2/9.4). Field `route` trong các ví dụ JSON của tài liệu này là **minh hoạ**, chưa phải câu query đã verify ra đúng giá trị đó — Kiro cần tự chạy thử `get_graph_schema`/`get_architecture` trên 1 cặp service thật có gọi HTTP nhau để xác nhận đúng shape trước khi code cứng câu query này.

**Lưu ý vận hành:** vì job nền (a) không chạy theo từng PR, dữ liệu cross-service có độ trễ nhất định (tối đa bằng chu kỳ chạy job nền) — chấp nhận được vì mục đích là phát hiện xu hướng ảnh hưởng kiến trúc, không phải theo dõi real-time.

**Không cần** lặp `search_graph`/`trace_path` để tính risk score theo từng file như thiết kế cũ — phần đó đã chuyển giao hoàn toàn cho `code-review-graph`.

### 4.4 Kết hợp 3 nguồn thành `final_score`

```
existing_score   = <điểm dependency graph nội bộ hiện tại, 0-100>
crg_risk_score   = code-review-graph risk_score * 100   (0-100)
base_score       = (existing_score + crg_risk_score) / 2

cross_service_impact = {
  affected: true/false/null,   # null = job nền chưa index service đích, KHÔNG được coi là false
  routes: [ {
    service: "...", route: "...", via: "CROSS_HTTP_CALLS|CROSS_ASYNC_CALLS|CROSS_CHANNEL",
    target_function: "...", target_has_test_coverage: true/false, escalated: true/false
  } ]
}

# Cross-service không phải điểm để trung bình cộng — là modifier, vì 1 thay đổi
# breaking API xuyên service luôn phải được coi là rủi ro cao bất kể 2 điểm kia thấp.
# CROSS_SERVICE_FLOOR_SCORE là tham số cấu hình (KHÔNG hardcode) — giá trị mặc định
# đề xuất 70, đội vận hành tự chỉnh theo ngưỡng rủi ro thực tế mà không cần sửa code.
# affected == null (chưa xác định, job nền chưa index service đích) KHÔNG nâng sàn —
# nhưng UI/report PHẢI hiển thị rõ "chưa xác định", không hiển thị giống hệt trường hợp
# affected == false (đã xác định là an toàn) — 2 trạng thái mang ý nghĩa khác hẳn nhau:
final_score = cross_service_impact.affected == true
              ? max(base_score, CROSS_SERVICE_FLOOR_SCORE)
              : base_score
```

Hiển thị cho user xem **cả 3 giá trị riêng biệt** (`existing_score`, `crg_risk_score`, `cross_service_impact`), không gộp ở tầng lưu trữ — chỉ `final_score` là số tổng hợp dùng để đánh giá/gate.

## 5. [Module B: PR Review Orchestration] Context file (nạp qua `resources`) — giải quyết vấn đề prompt dài

### 5.1 Nguyên tắc giảm dữ liệu (làm ở Service, KHÔNG giao cho agent)

1. **Knowledge/skill: lọc theo domain của PR**, không ghép nguyên văn tất cả file. Số file khớp domain **thay đổi theo từng PR** (0, 1, hay nhiều file) — không hardcode số lượng cố định.
2. **Phân luồng theo kích thước file**:

   | Kích thước file | Cách đưa vào context |
   |---|---|
   | Đủ nhỏ, dùng trọn vẹn | Thêm thẳng `file://<path>` vào mảng `resources` của agent config (mục 6.1) |
   | Quá lớn, chỉ cần 1 phần liên quan | Service tự đọc, cắt đúng đoạn liên quan domain, ghi vào field `knowledge_excerpts` trong context JSON (mục 5.2) |

   `resources` cuối cùng là 1 danh sách động: `context_PRxxx.json` (luôn có) + 0..N file knowledge/skill nhỏ (tuỳ PR).

3. **Cache bản tóm tắt/excerpt theo từng file knowledge lớn** (theo file vật lý, không theo tổ hợp service/flow): tính 1 lần, invalidate khi file gốc đổi.
4. **Score-based depth cho diff + code snippet**: độ chi tiết tỉ lệ với `final_score`.

| `final_score` | Diff | Code snippet | Blast radius |
|---|---|---|---|
| Thấp (< 30) | Tên file + số dòng thay đổi | Không | Không |
| Trung bình | Diff đầy đủ | Không | Danh sách caller |
| Cao (> 70) | Diff đầy đủ | Có, cho symbol risk cao nhất | Danh sách caller đầy đủ |

5. Giới hạn top N symbol (10-15, ưu tiên theo `risk_score` từ `code-review-graph`) đưa vào context.

### 5.2 Schema file context (1 file JSON duy nhất mỗi lần gọi)

```json
{
  "pr_meta": {
    "pr_number": "123",
    "base_sha": "...",
    "head_sha": "...",
    "changed_files": ["src/foo.c", "src/bar.c"]
  },
  "diff": "<patch text hoặc tóm tắt, theo score-based depth mục 5.1>",
  "impact_context": {
    "existing_score": 62,
    "crg_risk_score": 80,
    "cross_service_impact": {
      "affected": true,
      "routes": [{"service": "payment-service", "route": "POST /charge", "via": "CROSS_HTTP_CALLS", "target_function": "chargeHandler", "target_has_test_coverage": false, "escalated": true}]
    },
    "final_score": 71,
    "changed_functions": [
      {
        "name": "processOrder",
        "risk_score": 65,
        "is_test": false,
        "inbound_callers": ["handleCheckout", "retryPayment"],
        "code_snippet": "... (chỉ với symbol risk cao)"
      }
    ]
  },
  "knowledge_excerpts": [
    { "source": "architecture-overview.md", "excerpt": "..." }
  ],
  "user_context": "<text bổ sung user nhập, nếu có>"
}
```

`knowledge_excerpts` chỉ dùng khi file knowledge quá lớn cần cắt bớt — để rỗng `[]` nếu mọi knowledge liên quan đều đủ nhỏ để dùng nguyên vẹn qua `resources`.

Ghi ra file riêng theo `pr_number`, không dùng path cố định: `/tmp/pr-review/context_PR123.json`

## 6. Gọi kiro-cli (headless, zero-tool)

### 6.1 Sinh agent config riêng cho mỗi lần gọi

```json
// /tmp/pr-review/agents/pr-reviewer_PR123.json
{
  "name": "pr-reviewer_PR123",
  "description": "Review PR dựa trên context đã resolve, không truy cập file/tool nào",
  "tools": [],
  "allowedTools": [],
  "model": "claude-sonnet-5",
  "resources": [
    "file:///tmp/pr-review/context_PR123.json",
    "file:///knowledge-base/payment-guideline.md",
    "file:///skills/review-skill.md"
  ],
  "prompt": "<mô tả rõ: 'context_PRxxx.json' chứa dữ liệu PR (schema mục 5.2); các file .md khác là tài liệu domain/skill + instruction review + schema output mong muốn>"
}
```

### 6.2 Gọi

```bash
kiro-cli chat --no-interactive --agent /tmp/pr-review/agents/pr-reviewer_PR123.json \
  "Review PR123 theo instructions đã cấu hình trong agent"
```

### 6.3 Dọn dẹp

Xoá `context_PR123.json` + `pr-reviewer_PR123.json` sau khi lấy được output.

### 6.4 Concurrency

Mỗi lời gọi = 1 process riêng, file context + agent config riêng theo `pr_number`. Giới hạn số process chạy đồng thời.

## 7. Parse output + retry

- Agent chỉ in ra đúng 1 khối JSON bọc giữa marker cố định: `<<<RESULT_JSON>>> ... <<<END_RESULT_JSON>>>`.
- Parse/validate thất bại → gọi lại tối đa 2-3 lần, kèm lỗi validate vào lần gọi lại. Thất bại sau N lần → ghi lỗi rõ ràng vào report.

## 8. Edge cases

| Case | Xử lý |
|---|---|
| Repo chưa từng build (`code-review-graph`/`codebase-memory-mcp`) | Tự tạo mới ở bước build/index — không cần check trước |
| Edge count bất thường sau build `code-review-graph` (nghi ngờ thiếu ignore rule) | Cảnh báo trong log, vẫn tiếp tục chạy nhưng flag kết quả `crg_risk_score` là "cần review lại ignore config" |
| File thay đổi hoàn toàn mới | Re-index/update bắt buộc trước khi query, cả 2 tool |
| Không tìm thấy `CROSS_HTTP_CALLS`/`CROSS_ASYNC_CALLS`/`CROSS_CHANNEL` nào liên quan | `cross_service_impact.affected = false`, không coi là lỗi |
| Job nền (mục 4.3a) chưa từng index service đích (service mới thêm, hoặc job vừa lỗi) | `cross_service_impact.affected = null` (KHÔNG phải `false`) — hiển thị rõ "chưa xác định được impact xuyên service, dữ liệu job nền chưa sẵn sàng cho service này", không để hiểu nhầm là an toàn |
| Binary/package của 1 trong 2 tool lỗi | Log lỗi, giá trị tương ứng = `null`, `final_score` tính từ các nguồn còn lại, **không chặn review** |
| `git-mcp get diff` lỗi (PR không tồn tại...) | Chặn cứng — không có diff thì không review được |
| Repo quá lớn, build/update chậm | Timeout riêng cho bước index, tách khỏi timeout gọi kiro-cli |
| PR review tool chạy trên nhiều repo/nhiều user | Set `CBM_ALLOWED_ROOT` cho `codebase-memory-mcp`; giới hạn path tương đương cho `code-review-graph` |
| Parse/validate output kiro-cli thất bại sau retry | Ghi lỗi rõ ràng vào report — lỗi hệ thống, không phải lỗi của PR |

## 9. [Module A: Impact Assessment] Escalate sang "Tool Review Automation Test" khi test coverage yếu

Mỗi service (trong 11 service) có 1 repo component-test riêng, tách biệt khỏi service repo. Câu hỏi "PR này có cần thêm/sửa testcase không" **không dùng chung graph với bước 3** (test coverage cross-repo giữa service code và test-repo không nằm trong phạm vi `cross-repo-intelligence` của `codebase-memory-mcp` — mode đó chỉ match Route/Channel, không match quan hệ test-to-function xuyên repo). Thay vào đó, chạy **nối tiếp 2 pha**, tận dụng lại toàn bộ framework "Tool Review Automation Test" đã spec riêng (xem tài liệu `requirements-fix-review-tool.md`), không viết lại logic đó.

### 9.1 Pha 1 — tín hiệu thô (đã có sẵn từ bước 3, không tốn thêm lời gọi)

`code-review-graph detect-changes` (mục 4.2) đã trả về `test_gaps` — danh sách hàm thay đổi **không có test cover trong chính service repo** (đã verify thật: ví dụ output `Untested: cbm_writer_open, bind_text...`). Dùng ngay tín hiệu này làm điều kiện trigger, không cần tính toán thêm gì mới.

**Điều kiện trigger Pha 2:** `test_gaps` không rỗng CHO các hàm nằm trong `changed_functions` của PR, HOẶC `crg_risk_score >= 70`.

### 9.2 Pha 2 — trigger framework Tool Review Automation Test, scope theo đúng hàm bị PR đụng tới

**Cần thêm 1 config mapping tĩnh** (không tự suy luận được từ graph — là kiến thức tổ chức):
```json
// service-test-repo-map.json
{
  "service-a": { "test_repo": "service-a-component-test", "test_repo_path": "/path/to/service-a-component-test" },
  "service-b": { "test_repo": "service-b-component-test", "test_repo_path": "..." }
}
```

**Gọi lại pipeline Data Resolver + LLM Reviewer của framework kia, nhưng bổ sung 1 tham số scope mới** không có trong bản spec gốc của nó:

```
focus_functions: ["bind_text", "cbm_writer_open"]   # lấy từ test_gaps của Pha 1
```

Data Resolver của framework kia (mục 3 trong `requirements-fix-review-tool.md`) chỉ resolve các row CSV có `prepare_data`/`expectation_data` liên quan tới `focus_functions` (match theo tên hàm/endpoint xuất hiện trong prepare/expectation), thay vì resolve toàn bộ file CSV — giữ đúng tinh thần "chỉ trigger có mục tiêu", không chạy full re-review mỗi lần có PR.

LLM Reviewer của framework kia nhận thêm trong prompt: *"PR vừa sửa các hàm sau chưa có test cover: {focus_functions}. Ưu tiên đánh giá `missing_business_cases` (5.2.6) và `coverage_gap` (5.2.5) cho đúng các hàm này trước, các phần khác của class giữ nguyên phạm vi review bình thường."*

### 9.3 Gộp kết quả vào output PR review

Output của Pha 2 (theo đúng schema `verify_code_issues`/aggregate của framework kia, mục 5.4/6.3 trong tài liệu đó) được đính kèm vào kết quả review PR như 1 mục riêng — **không ép vào cùng schema JSON của bước 8**, vì đây là kết quả từ 1 pipeline khác, chạy async/có thể lâu hơn review PR chính (không nên block PR review chờ Pha 2 xong).

**Khuyến nghị vận hành:** Pha 2 chạy **bất đồng bộ**, không nằm trong đường găng của việc trả kết quả review PR chính (bước 8) — trả review PR trước, đính kèm ghi chú "đang phân tích test coverage sâu, kết quả cập nhật sau" nếu Pha 2 được trigger, cập nhật report khi Pha 2 xong.

### 9.4 Escalate XUYÊN SERVICE — test coverage của service khác bị ảnh hưởng gián tiếp

**Lỗ hổng đã phát hiện và cần vá:** mục 9.1-9.3 chỉ escalate test coverage cho **đúng service đang có PR**. Nếu PR đó phá vỡ hợp đồng API mà service khác đang gọi tới (`cross_service_impact.affected = true`), **không có gì kiểm tra xem test của service kia có bị ảnh hưởng hay không** — đây là khoảng trống rủi ro thật (vd: sửa response của `POST /charge` ở service-a, service payment-service gọi API này có test mock/assert theo format cũ, PR bên service-a merge xong test bên payment-service âm thầm sai mà không ai biết).

**Yêu cầu hạ tầng bổ sung — bắt buộc cho mục này:** `code-review-graph` phải được build/update **cho toàn bộ 11 service liên tục** (không chỉ on-demand cho service đang review) — dùng chung job nền định kỳ đã mô tả ở mục 4.3(a), thêm bước `code-review-graph update --repo <mỗi service>` vào cùng job đó. Không làm on-demand trong luồng PR review vì quá chậm (phải build graph cho service khác ngay trong lúc đang chờ trả kết quả PR).

**Luồng xử lý (nối tiếp mục 9.1-9.3, chạy song song với Pha 2 gốc, không phụ thuộc nhau):**

```
1. Với mỗi entry trong cross_service_impact.routes (mục 4.3, đã có other.qualified_name):
     a. Suy ra service đích + function/handler đích từ qualified_name
        (qualified_name có tiền tố <project>.<path>.<name> — project = tên service)
     b. Tra `code-review-graph` ĐÃ BUILD SẴN của CHÍNH service đích đó (không phải
        service đang có PR) — check function/handler đó có nằm trong test_gaps
        của lần build gần nhất không (dùng `code-review-graph status`/query trực
        tiếp DB của service đích, KHÔNG chạy detect-changes vì không có PR nào
        ở service đích cả — chỉ cần biết trạng thái test coverage HIỆN TẠI của
        đúng function đó)
2. Nếu function/handler đích thiếu test coverage:
     → Tra service-test-repo-map.json (mục 9.2) theo SERVICE ĐÍCH (không phải
       service đang có PR) → trigger Pha 2 cho ĐÚNG test-repo của service đích,
       focus_functions = [tên function/handler đích]
3. Kết quả escalation xuyên service được gắn nhãn rõ "cross-service — thuộc
   service X, không phải service đang review" khi hiển thị (mục 9.3/output),
   tránh gây hiểu nhầm là lỗi của chính PR đang xem
```

### 9.5 Khi PR ảnh hưởng NHIỀU service cùng lúc (vd 3 service)

Bước 1 của mục 9.4 vốn đã lặp qua **từng entry** trong `routes` — nên N service bị ảnh hưởng thì chạy đúng N lần độc lập, không có giới hạn cứng ở 1 service. Làm rõ thêm các điểm quan trọng khi N > 1:

1. **Dedup trước khi trigger**: nếu 2 route khác nhau trong `routes` cùng trỏ tới **cùng 1 `target_function` của cùng 1 service** (vd PR gọi tới cùng 1 API từ 2 chỗ khác nhau trong code), chỉ trigger Pha 2 **1 lần** cho cặp (service, function) đó — dùng `(service, target_function)` làm khoá dedup, không trigger trùng.

2. **Mỗi service escalate độc lập, không chờ nhau**: 3 service bị ảnh hưởng → 3 lời gọi Pha 2 chạy **song song, độc lập** (khác test-repo, khác agent config/context file riêng theo đúng nguyên tắc concurrency mục 6.4) — service này lỗi/chậm không ảnh hưởng tới kết quả của 2 service kia.

3. **Giới hạn số escalation đồng thời**: cùng dùng chung queue/semaphore giới hạn concurrency đã có ở mục 6.4 (gọi kiro-cli) — không tạo thêm cơ chế giới hạn riêng, tránh 1 PR ảnh hưởng nhiều service làm quá tải hệ thống do trigger hàng loạt lời gọi kiro-cli cùng lúc.

4. **Output tổng hợp dạng danh sách, không phải 1 kết quả đơn**: report hiển thị **danh sách các escalation xuyên service**, mỗi phần tử gắn rõ tên service — không gộp chung thành 1 khối text, vì người đọc cần biết chính xác cần báo cho team nào:

```json
{
  "cross_service_escalations": [
    { "service": "payment-service", "target_function": "chargeHandler", "status": "pending|done", "findings": [...] },
    { "service": "notification-service", "target_function": "sendReceipt", "status": "pending|done", "findings": [...] },
    { "service": "inventory-service", "target_function": "reserveStock", "status": "done", "findings": [] }
  ]
}
```

5. **1 service có thể bị ảnh hưởng qua NHIỀU function khác nhau** — không dedup theo service, chỉ dedup theo đúng cặp `(service, target_function)` (điểm 1) — vì mỗi function là 1 rủi ro test-gap độc lập, kể cả khi cùng 1 service.

**Vì sao bước 1b không dùng `detect-changes`:** `detect-changes` cần 1 diff (base vs head) để hoạt động — nhưng ở service đích, không có PR/diff nào cả, ta chỉ cần biết "hiện tại function này có test cover không", tức là đọc thẳng dữ liệu graph/test-coverage đã lưu từ lần build gần nhất của service đích (`code-review-graph status` hoặc 1 câu query trực tiếp trên DB `graph.db` của service đó — không tính lại risk score, chỉ cần trạng thái test coverage boolean).

**Schema mở rộng cho `cross_service_impact`:**
```json
{
  "cross_service_impact": {
    "affected": true,
    "routes": [
      {
        "service": "payment-service",
        "route": "POST /charge",
        "via": "CROSS_HTTP_CALLS",
        "target_function": "chargeHandler",
        "target_has_test_coverage": false,
        "escalated": true
      }
    ]
  }
}
```

## 10. [Tính năng mới — UI riêng] Review sâu theo Flow nghiệp vụ

### 10.1 Mục tiêu

Tính năng UI thứ 3 (độc lập, không gắn với PR cụ thể) — user chọn 1 flow nghiệp vụ trong 1 service, hệ thống trace toàn bộ flow từ entry point tới điểm kết thúc, đưa **toàn bộ code của mọi step trong flow** cho LLM đọc và đánh giá **mọi khía cạnh**: tính nhất quán giữa các bước, đúng đắn logic, rủi ro kỹ thuật, bug tiềm ẩn, performance, nghiệp vụ — không chỉ review phần code thay đổi như 2 tính năng trước.

### 10.2 Nguồn dữ liệu: `code-review-graph` Flow — đã có sẵn, verify thật

Đã verify trực tiếp (gọi `get_affected_flows_func` live trên chính repo `codebase-memory-mcp`, ra kết quả thật):

```python
{
  "name": "cbm_store_checkpoint",
  "criticality": 0.5267,
  "steps": [
    {"name": "cbm_store_checkpoint", "file": ".../store.c", "line_start": 1052, "line_end": 1068},
    {"name": "exec_sql", "file": ".../store.c", "line_start": 153, "line_end": 165},
    {"name": "store_set_error_sqlite", "file": ".../store.c", "line_start": 149, "line_end": 151}
  ]
}
```

Cơ chế nội bộ (`code_review_graph/flows.py`): `detect_entry_points` tự tìm điểm bắt đầu (hàm không ai gọi tới / có decorator framework như `@app.get` / khớp tên quy ước như `main`), `trace_flows` BFS xuôi từ mỗi entry point tới hết các bước gọi nhau (tối đa depth 15), tính `criticality` theo nhiều yếu tố trọng số.

**Vấn đề kỹ thuật cần lưu ý:** `list_flows`/`get_flow`/`get_affected_flows` **chỉ expose qua MCP, không có CLI subcommand** (đã kiểm tra hết danh sách lệnh `code-review-graph --help`: không có `flows`/`affected-flows`). Để giữ đúng kiến trúc CLI-mode/deterministic hiện tại (không dùng MCP), cần viết **1 script Python wrapper mỏng**, import trực tiếp hàm nội bộ và in JSON ra stdout — đã verify chạy được:

```python
# crg_get_flows.py — Service gọi như 1 CLI tool: python crg_get_flows.py --repo <path> [--flow-id N]
from code_review_graph.tools.review import get_affected_flows_func
from code_review_graph.tools.flows_tools import list_flows, get_flow  # liệt kê / lấy chi tiết 1 flow theo id
```

### 10.3 Luồng tính năng

```
1. User mở tính năng "Review Flow", chọn 1 service
2. Server gọi wrapper `list_flows` (mục 10.2) → hiển thị danh sách flow (tên, entry point, criticality) cho user chọn
3. User chọn 1 flow cụ thể
4. Server gọi wrapper `get_flow(flow_id)` → lấy đủ `steps` (qualified_name, file, line_start/line_end) của flow đó
5. Server đọc TOÀN BỘ code từng step (đọc trực tiếp file theo line_start/line_end, không cần gọi thêm tool nào khác —
   dữ liệu file/line đã có sẵn từ bước 4) → build flow_context (mục 10.4)
6. Sinh agent config riêng (giống mục 6.1), resources trỏ vào flow_context.json
7. Gọi kiro-cli --no-interactive, agent 0 tool (giống mục 6.2)
8. Parse output theo marker cố định (giống mục 7), trả kết quả cho user
```

**Trạng thái code được đọc:** tính năng này KHÔNG gắn với PR/commit cụ thể nào — đọc code theo đúng bản `code-review-graph` **đã build sẵn gần nhất** của service đó (cùng dữ liệu do job nền mục 4.3(a)/9.4 duy trì liên tục, thường là nhánh main/master hiện tại). Nếu user cần review flow theo đúng code của 1 PR cụ thể (chưa merge), cần checkout riêng PR đó trước khi build — không nằm trong luồng mặc định của tính năng này (vốn dùng cho code đã merge/đang chạy production).

### 10.4 Giới hạn kích thước — bắt buộc, vì đưa FULL code chứ không phải snippet

Khác với `impact_context` (chỉ code snippet cho symbol risk cao nhất), tính năng này cần **toàn bộ code của MỌI step** — rủi ro phình context rất thật nếu flow có nhiều step/depth lớn.

- **Cap tổng số step đưa vào context** (đề xuất tham số cấu hình `MAX_FLOW_STEPS`, không hardcode): nếu flow có `node_count` vượt ngưỡng, báo cho user biết flow quá lớn để review trọn vẹn trong 1 lần gọi, đề xuất chia nhỏ (review theo từng đoạn của flow) thay vì cắt bớt âm thầm.
- **Cap tổng độ dài code** (ước lượng token trước khi build context, tương tự `estimate_file_tokens` đã có sẵn trong `code-review-graph`) — vượt ngưỡng thì báo lỗi rõ ràng cho user, không tự ý cắt code (cắt code review flow dễ bỏ sót đúng chỗ cần xem).

### 10.5 Schema `flow_context.json`

```json
{
  "flow_name": "cbm_store_checkpoint",
  "entry_point": "...",
  "criticality": 0.5267,
  "steps": [
    {
      "order": 1,
      "name": "cbm_store_checkpoint",
      "file": "src/store/store.c",
      "line_start": 1052,
      "line_end": 1068,
      "code": "... toàn bộ source function này ..."
    }
  ]
}
```

### 10.6 Output — schema khác với PR review, theo từng finding gắn với step cụ thể

```json
{
  "flow_name": "cbm_store_checkpoint",
  "findings": [
    {
      "step_order": 2,
      "location": "exec_sql, store.c:153",
      "category": "bug | performance | consistency | business_logic",
      "severity": "high | medium | low",
      "description": "...",
      "reason": "..."
    }
  ]
}
```

### 10.7 API endpoint

`GET /flows?service=X` (list) + `POST /flow-review` (trigger review theo `flow_id`) — 2 endpoint mới, không đụng tới `/impact-assessment`/`/pr-review` đã có, vẫn trong cùng 1 backend service (đúng nguyên tắc mục 1.1).

## 11. Acceptance criteria

- [ ] Toàn bộ lời gọi subprocess (codebase-memory-mcp, code-review-graph, wrapper Python, git, kiro-cli) dùng `execFile`/`spawn` với argument dạng mảng — không dùng `exec`/nối chuỗi shell, đặc biệt với tham số có nguồn gốc từ nội dung PR (tên hàm, branch, title...)
- [ ] Server đã cài Python runtime như 1 dependency hạ tầng tường minh (không giả định có sẵn vì Server là Node)
- [ ] Module A (Impact Assessment, mục 4+9) và Module B (PR Review Orchestration, mục 5-8) tách rõ ràng ở tầng code — Module B chỉ gọi vào Module A qua đúng interface `impact_context` (mục 1.1), không gọi trực tiếp `code-review-graph`/`codebase-memory-mcp`
- [ ] Có 2 API endpoint riêng biệt (`/impact-assessment` và `/pr-review`) tương ứng 2 nút bấm trên UI — cùng gọi vào Module A, không viết logic trùng lặp giữa 2 endpoint
- [ ] Gọi `/impact-assessment` độc lập KHÔNG kích hoạt kiro-cli/LLM review — chỉ trả `impact_context`
- [ ] `/impact-assessment` trả kết quả 2 giai đoạn: điểm số đồng bộ ngay, escalation test coverage (nếu trigger) cập nhật sau — UI có cơ chế hiển thị/poll trạng thái đang xử lý
- [ ] Bước 3 tính ra 3 giá trị riêng biệt: `existing_score`, `crg_risk_score`, `cross_service_impact` — không gộp ở tầng lưu trữ
- [ ] UI/output hiển thị cả 3 riêng cho user xem
- [ ] `final_score` = trung bình cộng (`existing_score`, `crg_risk_score`), nâng sàn lên `CROSS_SERVICE_FLOOR_SCORE` (tham số cấu hình, không hardcode) nếu `cross_service_impact.affected = true`
- [ ] Khi 1 nguồn lỗi/không tính được → `final_score` vẫn tính từ các nguồn còn lại, không crash toàn bộ pipeline
- [ ] `code-review-graph` cài trên Server qua bundle offline (`pip download --only-binary=:all:` trên máy có mạng, chuyển bundle, `pip install --no-index`) — không cài nhóm optional (embeddings/wiki)
- [ ] Đã tạo `.code-review-graphignore` cho repo trước khi đưa vào pipeline chính thức; đã kiểm tra `status` sau build đầu không có edge count bất thường
- [ ] `code-review-graph detect-changes` nhận `changed_files`/`base` từ git-mcp, không tự tính diff nội bộ lệch với nguồn khác
- [ ] `codebase-memory-mcp` chỉ được gọi cho phần cross-service (không lặp lại việc tính risk score trong-service đã có ở `code-review-graph`)
- [ ] Context file (mục 5.2) chứa đủ field, ghi ra path riêng theo `pr_number`
- [ ] Knowledge/skill lọc theo domain; file nhỏ qua `resources`, file lớn cắt excerpt — không trùng lặp giữa 2 nơi
- [ ] Verify agent 0 tool: `/tools` → danh sách rỗng
- [ ] Verify `resources` nạp được: hỏi lại 1 chi tiết cụ thể trong context file → agent trả lời đúng
- [ ] Output kiro-cli nằm giữa marker cố định, parse/validate + retry đúng
- [ ] File tạm bị xoá sau mỗi lần gọi; chạy nhiều PR song song không đụng nhau
- [ ] Base branch không fetch mới nhất trước khi chạy: không ảnh hưởng, vì file list lấy từ git-mcp (SHA chính xác), không phụ thuộc git diff nội bộ của bất kỳ tool nào
- [ ] Thời gian thêm vào bước 3 (build/update + query cả 2 tool) được đo và log lại, so sánh overhead với luồng cũ
- [ ] Config `service-test-repo-map.json` tồn tại, map đủ 11 service → đúng repo component-test tương ứng
- [ ] Pha 2 (mục 9) chỉ trigger khi `test_gaps` không rỗng cho hàm trong PR HOẶC `crg_risk_score >= 70` — không chạy full review test coverage cho mọi PR
- [ ] Data Resolver của framework Tool Review Automation Test nhận đúng tham số `focus_functions`, chỉ resolve testcase liên quan, không resolve toàn bộ CSV khi được gọi từ luồng PR review
- [ ] Pha 2 chạy bất đồng bộ, không làm chậm thời gian trả kết quả review PR chính (bước 8)
- [ ] Kết quả Pha 2 hiển thị như mục riêng trong report, không ép chung schema với review PR chính
- [ ] Tính năng "Review Flow" (mục 10) có 2 endpoint riêng (`GET /flows`, `POST /flow-review`), không gắn với PR cụ thể, user tự chọn flow
- [ ] Wrapper Python (`crg_get_flows.py` hoặc tương đương) gọi trực tiếp `list_flows`/`get_flow`/`get_affected_flows_func` — không dùng MCP, giữ đúng kiến trúc CLI-mode/deterministic
- [ ] `flow_context.json` chứa full code từng step (không phải snippet) — có cơ chế cap `MAX_FLOW_STEPS`/token trước khi gọi kiro-cli, báo lỗi rõ ràng cho user thay vì âm thầm cắt bớt code khi vượt ngưỡng
- [ ] Agent config của Review Flow cũng `"tools": []`, verify `/tools` rỗng như các tính năng khác
- [ ] Output Review Flow theo schema riêng (mục 10.6, gắn `step_order`/`category`/`severity`), không dùng chung schema với `/pr-review`
- [ ] `codebase-memory-mcp` cross_service_impact dùng đúng edge `CROSS_HTTP_CALLS`/`CROSS_ASYNC_CALLS`/`CROSS_CHANNEL` (không phải bản nội bộ `HTTP_CALLS`/`ASYNC_CALLS`/`EMITS`/`LISTENS_ON`) — có job nền định kỳ index toàn bộ 11 service + chạy `cross-repo-intelligence` riêng biệt khỏi luồng per-PR
- [ ] `code-review-graph` được build/update định kỳ cho TOÀN BỘ 11 service (không chỉ service đang review) qua job nền — phục vụ escalation xuyên service (mục 9.4)
- [ ] Với mỗi route trong `cross_service_impact.routes`, hệ thống tra được `target_function` + `target_has_test_coverage` của đúng service đích
- [ ] Khi `target_has_test_coverage = false` → trigger Pha 2 cho ĐÚNG test-repo của service đích (tra theo `service-test-repo-map.json` bằng service đích, không phải service đang có PR)
- [ ] Kết quả escalation xuyên service hiển thị gắn nhãn rõ thuộc service nào, không gộp lẫn với review của chính PR đang xem
- [ ] `cross_service_impact.affected` phân biệt rõ 3 trạng thái `true`/`false`/`null` (chưa xác định) — UI/report hiển thị khác nhau cho `false` (đã xác nhận an toàn) và `null` (job nền chưa index service liên quan), không gộp chung là "không ảnh hưởng"
- [ ] PR ảnh hưởng nhiều service (N > 1): escalate độc lập cho từng service, không giới hạn cứng ở 1 service, không service nào chờ service khác
- [ ] Dedup escalation theo đúng cặp `(service, target_function)` — không trigger Pha 2 trùng lặp cho cùng 1 cặp dù bị phát hiện qua nhiều route/call-site khác nhau
- [ ] Output hiển thị `cross_service_escalations` dạng danh sách, mỗi phần tử gắn rõ tên service — không gộp chung thành 1 khối text duy nhất
