# Tài liệu CLI chi tiết — cho hệ thống tích hợp qua CLI

Tài liệu này tập trung vào việc **gọi trực tiếp qua CLI** (không qua MCP
client), dùng cho pipeline/hệ thống tự động hoá gọi `codebase-memory-mcp` như
một binary thông thường. Thông tin lấy trực tiếp từ source (`src/main.c`,
`src/cli/cli.c`, `src/mcp/mcp.c`), không phải đoán.

---

## 0. Cài đặt từ artifact tải trên GitHub Actions

Áp dụng cho artifact tải về từ workflow **"Quick Windows exe build"**
(`build-windows-exe-quick.yml`) hoặc **"Quick Linux binary build"**
(`build-linux-quick.yml`) — tab **Actions** → chọn workflow → chọn lần chạy
đã xong → mục **Artifacts**.

**Lưu ý quan trọng**: GitHub luôn bọc thêm 1 lớp `.zip` ngoài cùng khi tải
artifact, trong khi 2 workflow này TỰ đóng gói kết quả build thành
`.zip`/`.tar.gz` riêng trước khi upload — nên artifact tải về có **2 lớp
nén lồng nhau**, phải giải nén cả 2 lớp mới ra file thực thi.

### Windows

```powershell
# Giải nén lớp ngoài (GitHub) rồi lớp trong (build)
Expand-Archive codebase-memory-mcp-windows-exe.zip -DestinationPath .
Expand-Archive codebase-memory-mcp-windows-amd64.zip -DestinationPath .
```

Ra 4 file: `codebase-memory-mcp.exe`, `LICENSE`, `install.ps1`,
`THIRD_PARTY_NOTICES.md`. Đặt cố định 1 chỗ, ví dụ `D:\tools\codebase-memory-mcp\`.

Kiểm tra chạy được:

```bash
.\codebase-memory-mcp.exe --version
```

Không cần cài Python/Node hay bất kỳ dependency nào khác — binary tự chứa
toàn bộ (frozen/static build).

### Linux

```bash
unzip codebase-memory-mcp-linux-binary.zip
tar -xzf codebase-memory-mcp-linux-amd64-portable.tar.gz
chmod +x codebase-memory-mcp
./codebase-memory-mcp --version
```

Ra 4 file: `codebase-memory-mcp`, `LICENSE`, `install.sh`,
`THIRD_PARTY_NOTICES.md`. Binary build **static hoàn toàn** — chạy được
trên mọi distro Linux, kể cả base image tối giản trong Docker (Debian
slim, Alpine, distroless...), không cần cài glibc version cụ thể.

Ví dụ Dockerfile dùng thẳng binary này:

```dockerfile
FROM debian:bookworm-slim
COPY codebase-memory-mcp /usr/local/bin/codebase-memory-mcp
ENV CBM_CACHE_DIR=/tmp/cbm
ENTRYPOINT ["codebase-memory-mcp"]
```

### Sau khi cài xong

Từ đây trở đi, gọi trực tiếp file thực thi theo đường dẫn đã đặt (không cần
thêm bước nào khác) — chuyển sang mục 2 bên dưới để dùng CLI. Muốn hệ thống
tự quản lý PATH/cấu hình agent thì chạy thêm `install -y` (mục 5), nhưng
với hệ thống tích hợp qua CLI thuần tuý (không qua agent) thì **không bắt
buộc** — chỉ cần trỏ đúng đường dẫn binary khi gọi lệnh.

**Alias ngắn `cbm`**: chạy `install` (hoặc `update`) sẽ tự tạo thêm 1 alias
tên `cbm` (`cbm.exe` trên Windows) nằm CẠNH `codebase-memory-mcp` trong
`~/.local/bin/` — gõ `cbm` thay vì gõ đầy đủ `codebase-memory-mcp` cho đỡ
dài, hành vi giống hệt nhau (POSIX: symlink; Windows: bản copy, tự đồng bộ
lại mỗi lần `install`/`update`). Từ mục 1 trở đi, tài liệu này dùng `cbm`
trong mọi ví dụ lệnh — thay bằng `codebase-memory-mcp` (hoặc đường dẫn đầy
đủ tới binary) nếu bạn chưa chạy `install` để tạo alias.

---

## 1. Cấu trúc lệnh gốc

```
cbm                       Chạy MCP server qua stdio (mặc định, không kèm subcommand)
cbm cli <tool> [args]     Gọi 1 MCP tool trực tiếp — DÙNG CÁI NÀY CHO AUTOMATION
cbm install [-y|-n] [--force] [--dry-run] [--reset-indexes]
cbm uninstall [-y|-n] [--dry-run]
cbm update [-y|-n]
cbm config <list|get|set|reset> [key] [value]
cbm hook-augment
cbm --version
cbm --help
```

(`cbm` = alias của `codebase-memory-mcp`, xem mục 0 — dùng tên nào cũng
được, hành vi giống hệt.)

Cờ toàn cục:
- `--ui=true|false` — bật/tắt HTTP graph visualization (được lưu lại)
- `--port=N` — port cho UI (mặc định 9749, được lưu lại)
- `--profile` — bật CPU profiling

---

## 2. `cli <tool_name>` — cách gọi 1 tool trực tiếp (quan trọng nhất cho automation)

### 2.1. 4 cách truyền tham số (theo thứ tự ưu tiên)

```bash
# 1) --args-file: đọc JSON từ file
cbm cli index_repository --args-file args.json

# 2) --flag value (khuyến nghị cho script đơn giản)
cbm cli index_repository --repo-path /path/to/repo --mode full

# 3) Piped stdin (JSON thuần, không cần escape qua shell) — TỐT NHẤT cho automation
echo '{"repo_path":"/path/to/repo","mode":"full"}' | cbm cli index_repository

# 4) Raw JSON làm argument — ĐÃ DEPRECATED, vẫn hoạt động nhưng in warning ra stderr
cbm cli index_repository '{"repo_path":"/path/to/repo"}'
```

Tên field trong `--flag` tự động chuyển `snake_case` (JSON schema) sang
`kebab-case` (CLI flag): field `repo_path` → cờ `--repo-path`.

### 2.2. Xem schema đầy đủ của 1 tool

```bash
cbm cli <tool_name> --help
```

In ra toàn bộ flag, type, required/optional, description — luôn đúng với
bản build hiện tại (tự sinh từ JSON schema nội bộ, không lệch so với code).

### 2.3. Output format — **PHẢI ĐỌC KỸ PHẦN NÀY**

| Cờ | Hành vi |
|---|---|
| (mặc định, không cờ) | Bóc `content[0].text` từ JSON response MCP, in ra `stdout` (nếu OK) hoặc `stderr` (nếu lỗi). Exit code: **0 nếu OK, 1 nếu `isError:true`**. |
| `--json` | In **nguyên JSON MCP response đầy đủ** ra `stdout`. **Exit code LUÔN LÀ 0** bất kể tool có lỗi hay không — phải tự parse field `isError` trong JSON để biết thành/bại. |

**Khuyến nghị cho hệ thống automation**: dùng `--json`, tự parse response,
kiểm tra `isError` — đừng dựa vào exit code khi dùng `--json`.

```bash
result=$(echo '{"repo_path":"/path/to/repo"}' | cbm cli index_repository --json)
is_error=$(echo "$result" | jq -r '.isError // false')
if [ "$is_error" = "true" ]; then
  echo "FAILED: $(echo "$result" | jq -r '.content[0].text')" >&2
  exit 1
fi
```

Cờ khác: `--progress` (in tiến độ ra stderr, hữu ích cho `index_repository` trên repo lớn).

---

## 3. Danh sách đầy đủ các tool (`cli <tool_name>`)

| Tool | Mục đích | Required params |
|---|---|---|
| `index_repository` | Index (hoặc re-index) 1 repo vào graph. Có mode `full`/`moderate`/`fast`/`cross-repo-intelligence` | `repo_path` |
| `search_graph` | Tìm function/class/route/variable — dùng thay grep | `project` |
| `query_graph` | Chạy Cypher query tuỳ ý trên graph | `query`, `project` |
| `trace_path` | Trace callers/callees, data flow, hoặc cross-service (HTTP/async/gRPC...) | `function_name`, `project` |
| `get_code_snippet` | Lấy source code của 1 symbol theo qualified_name | `qualified_name`, `project` |
| `get_graph_schema` | Lấy danh sách node label + edge type | `project` |
| `get_architecture` | Tổng quan kiến trúc (structure/dependencies/routes/hotspots/clusters...) | `project` |
| `search_code` | Grep + enrich bằng graph (dedup theo function, rank theo mức quan trọng) | `pattern`, `project` |
| `list_projects` | Liệt kê project đã index | — |
| `delete_project` | Xoá 1 project khỏi index | `project` |
| `index_status` | Trạng thái index: số node/edge, coverage report (file bị skip/parse-partial) | `project` |
| `detect_changes` | Phân tích tác động thay đổi code (risk-scored) | `project` |
| `manage_adr` | Đọc/ghi Architecture Decision Record | `project` |
| `ingest_traces` | Nạp runtime trace để bổ sung graph | `traces`, `project` |

`project` = tên project được suy ra từ `repo_path` khi index (thường là tên
thư mục, có thể override qua `name` khi gọi `index_repository`). Dùng
`list_projects` để lấy tên chính xác nếu không chắc.

---

## 4. Quy trình tự động hoá điển hình

```bash
# 1. Index lần đầu (hoặc update — tool tự phát hiện incremental vs full)
echo '{"repo_path":"/repo","mode":"full"}' | cbm cli index_repository --json

# 2. Kiểm tra trạng thái / coverage
echo '{"project":"repo"}' | cbm cli index_status --json

# 3. Truy vấn trong pipeline (ví dụ: review PR)
echo '{"project":"repo","scope":"changed","base_branch":"main"}' \
  | cbm cli detect_changes --json

# 4. Trace impact / tìm caller
echo '{"function_name":"processPayment","project":"repo","mode":"calls"}' \
  | cbm cli trace_path --json
```

Với repo lớn, thêm biến môi trường `CBM_INDEX_SUPERVISOR=1` (mặc định đã
bật khi chạy binary thật — chỉ tắt khi debug) để `index_repository` chạy
trong subprocess con, tránh out-of-memory làm chết cả process CLI.

---

## 5. Biến môi trường liên quan tới CLI/automation

| Biến | Ý nghĩa |
|---|---|
| `CBM_IGNORE_FILE` | Đường dẫn tới ignore file dùng thay `.cbmignore` mặc định — có thể đặt ngoài repo, path tương đối resolve theo `repo_path` |
| `CBM_CACHE_DIR` | Thư mục cache/data (mặc định `~/.cache/codebase-memory-mcp`) — hữu ích khi chạy trong container không có `$HOME` ghi được |
| `CBM_LOG_LEVEL` | Mức log (áp dụng trước dòng log đầu tiên) |
| `CBM_PROFILE` | Bật CPU profiling |
| `CBM_INDEX_SUPERVISOR` | `0` để tắt cơ chế supervisor subprocess khi index (mặc định bật) — chỉ tắt lúc debug |

---

## 6. Exit code tổng hợp

| Lệnh | Exit code |
|---|---|
| `cli <tool>` (mặc định, không `--json`) | 0 = OK, 1 = `isError:true` trong response |
| `cli <tool> --json` | Luôn 0 nếu process chạy được tới cuối — **phải tự check `isError` trong JSON** |
| `cli <tool>` với tool name không tồn tại | 1, in `error: unknown tool '<name>'` ra stderr |
| `install`/`uninstall`/`update` | 0 = thành công; khác 0 khi có lỗi (ví dụ user từ chối prompt khi không có `-y`) |

---

## 7. Lưu ý về config runtime (`config` subcommand)

```bash
cbm config list
cbm config get <key>
cbm config set <key> <value>
cbm config reset <key>
```

Dùng để đọc/sửa các setting đã lưu lại (persisted) như `ui_enabled`, `ui_port`.
