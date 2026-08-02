# Tài liệu CLI chi tiết — cho hệ thống tích hợp qua CLI

Tài liệu này tập trung vào việc **gọi trực tiếp qua CLI** (không qua MCP
client), dùng cho pipeline/hệ thống tự động hoá gọi `crg`/`code-review-graph`
như một binary/CLI thông thường. Thông tin lấy trực tiếp từ source
(`code_review_graph/cli.py`), không phải đoán.

---

## 0. Cài đặt từ artifact tải trên GitHub Actions

Áp dụng cho artifact tải về từ workflow **"Build standalone Windows exe"**
(`build-exe.yml`) hoặc **"Build standalone Linux binary"**
(`build-linux-exe.yml`) — tab **Actions** → chọn workflow → chọn lần chạy
đã xong → mục **Artifacts**.

Khác với `codebase-memory-mcp`, 2 workflow này upload thẳng file thực thi
(không tự đóng gói `.zip`/`.tar.gz` trước) — nên artifact tải về **chỉ có 1
lớp `.zip`** (do GitHub tự bọc), giải nén 1 lần là ra ngay file chạy được.

### Windows

```powershell
Expand-Archive crg-windows-exe.zip -DestinationPath .
```

Ra file `crg.exe` duy nhất (không kèm LICENSE/notice — khác với artifact
của `codebase-memory-mcp`). Đặt cố định 1 chỗ, ví dụ `D:\tools\crg\`.

```bash
.\crg.exe --version
```

### Linux

```bash
unzip crg-linux-exe.zip
chmod +x crg
./crg --version
```

Ra file `crg` duy nhất. Đây là binary PyInstaller frozen (tự chứa Python
runtime bên trong) — không cần cài Python trên máy chạy, nhưng **không
static** theo nghĩa glibc như `codebase-memory-mcp` — build trên
`ubuntu-latest`, nên máy chạy cần glibc tương đương hoặc mới hơn (Ubuntu
22.04+/Debian 12+ trở lên là an toàn; distro cũ hơn hoặc Alpine/musl có thể
không chạy được — trường hợp đó cân nhắc dùng `pip install crg` thay vì
binary frozen).

### Thay thế: cài qua pip (không cần artifact)

Nếu môi trường có sẵn Python 3.10+, không bắt buộc phải dùng file exe/binary
— cài thẳng qua PyPI cũng cho CLI tương đương:

```bash
pip install crg
crg --version
```

### Sau khi cài xong

Gọi trực tiếp file thực thi (hoặc lệnh `crg` nếu cài qua pip) theo đường dẫn
đã đặt — chuyển sang mục 1 bên dưới để dùng CLI.

---

## 1. Nhóm lệnh

`crg` có 2 nhóm lệnh khác biệt quan trọng cho automation:

- **Nhóm build/quản lý** (`build`, `update`, `status`, `postprocess`, `watch`,
  `forget`, `register`/`unregister`/`repos`, `daemon`...) — hầu hết in **text
  thường** ra stdout, trừ khi có cờ `--json` riêng.
- **Nhóm graph-tool trực tiếp** (`query`, `search`, `impact`, `flows`, `flow`,
  `communities`, `community`, `architecture`, `large-functions`, `refactor`)
  — **LUÔN in đúng 1 JSON object ra stdout, không cần cờ gì thêm**. Đây là
  nhóm lệnh phù hợp nhất để tích hợp CLI vào pipeline tự động, vì output
  parse được ngay không cần đoán format.

---

## 2. Build & quản lý graph

```bash
crg build [--repo PATH] [-q] [--skip-flows] [--skip-postprocess] [--data-dir PATH]
crg update [--repo PATH] [--base REF] [-q] [--brief] [--verify] [--skip-flows] [--skip-postprocess] [--data-dir PATH]
crg postprocess [--repo PATH] [--no-flows] [--no-communities] [--no-fts] [--data-dir PATH]
crg watch [--repo PATH] [--data-dir PATH]
crg forget PATH [PATH...] [--repo PATH] [--dry-run] [--data-dir PATH]
```

- `build`: full rebuild (parse lại toàn bộ). Output mặc định (không `-q`):
  `Full build: <N> files, <N> nodes, <N> edges (postprocess=full)`.
- `update`: incremental (chỉ file thay đổi theo git diff). `--brief` in thêm
  risk summary + Token Savings panel.
- Không có `--json` cho `build`/`update` — sau khi build/update, gọi
  `crg status --json` để lấy số liệu dạng structured.
- Exit code: `0` = thành công, `1` = lỗi (raise SystemExit(1)/sys.exit(1) rải
  rác trong toàn bộ CLI — quy ước nhất quán).

### `status` — trạng thái graph, có sẵn `--json`

```bash
crg status --repo PATH --json
```

```json
{
  "nodes": 12345, "edges": 45678, "files": 890,
  "languages": ["python", "go"],
  "last_updated": "...",
  "vcs": "git",
  "built_on_branch": "main", "built_at_commit": "<sha>",
  "current_branch": "main", "current_sha": "<sha>"
}
```

---

## 3. `detect-changes` — phân tích tác động thay đổi (read-only, không re-parse)

```bash
crg detect-changes [--base HEAD~1] [--repo PATH] [--churn] [--verify] [--brief]
```

- **Không `--brief`**: in **JSON đầy đủ** ra stdout (`json.dumps(result, indent=2)`)
  — dùng cái này cho automation.
- **`--brief`**: in risk summary dạng text (Token Savings panel) — cho người đọc.
- Không re-parse code — chỉ phân tích trên graph đã có sẵn. Muốn re-parse
  + phân tích cùng lúc, dùng `update --brief`.
- `--churn`: cộng thêm điểm risk theo tần suất commit 90 ngày gần nhất
  (chỉnh qua `CRG_CHURN_WINDOW_DAYS`).

```bash
crg detect-changes --base main --repo /path/to/repo | jq '.risk_summary'
```

---

## 4. Nhóm lệnh graph-tool trực tiếp — luôn JSON, không cần cờ

Tất cả các lệnh dưới đây gọi thẳng hàm tool nội bộ và
**in `json.dumps(result, indent=2, default=str)` — không có option nào khác**.

```bash
crg query <pattern> <target> [--repo PATH]
  # pattern: callers_of | callees_of | imports_of | importers_of |
  #          children_of | tests_for | inheritors_of | file_summary

crg impact [--files F1 F2...] [--depth 2] [--max-results 500] [--base HEAD~1] [--repo PATH]

crg search <query> [--kind File|Class|Function|Type|Test] [--limit 20] [--repo PATH]

crg flows [--sort criticality|depth|node_count|file_count|name] [--limit 50] [--kind K] [--repo PATH]
crg flow (--id N | --name NAME) [--source] [--repo PATH]

crg communities [--sort size|cohesion|name] [--min-size 0] [--repo PATH]
crg community (--id N | --name NAME) [--members] [--repo PATH]

crg architecture [--detail-level minimal|standard] [--repo PATH]

crg large-functions [--min-lines 50] [--kind Function|Class|File|Test] [--path SUBSTR] [--limit 50] [--repo PATH]

crg refactor <rename|dead_code|suggest> [--old-name X] [--new-name Y] [--kind Function|Class] [--path SUBSTR] [--repo PATH]
```

Ví dụ pipeline thực tế:

```bash
# Ai gọi hàm này? (dùng cho impact analysis khi review PR)
crg query callers_of processPayment --repo /repo | jq '.results'

# Blast radius của các file vừa đổi trong PR
crg impact --files src/payment.py src/order.py --depth 2 --repo /repo

# Tổng quan kiến trúc (cho báo cáo tự động)
crg architecture --detail-level standard --repo /repo
```

---

## 5. Cài đặt / cấu hình agent (`install`)

```bash
crg install [--platform NAME] [--dry-run] [--yes] [--repo PATH]
```

`--platform`: `codex`, `claude` (= Claude Code CLI, KHÔNG phải Claude Desktop
app), `cursor`, `windsurf`, `zed`, `continue`, `opencode`, `antigravity`,
`gemini-cli`, `qwen`, `kiro`, `qoder`, `copilot`, `copilot-cli`, `codebuddy`,
`all` (mặc định).

Ghi `.mcp.json` vào **thư mục repo** (không phải config global) khi target
là `claude`/`claude-code`.

---

## 6. Multi-repo registry (`register`/`unregister`/`repos`)

```bash
crg register <path> [--alias NAME]
crg unregister <path_or_alias>
crg repos
```

`repos` in danh sách text (`  <path>  (<alias>)`), không có `--json` riêng.

---

## 7. Biến môi trường liên quan CLI/automation

| Biến | Ý nghĩa |
|---|---|
| `CRG_IGNORE_FILE` | Path tới ignore file thay `.crgignore` mặc định — absolute path dùng thẳng, relative resolve theo repo root. Đặt được ở ngoài repo, dùng chung cho nhiều repo. |
| `CRG_CHURN_WINDOW_DAYS` | Số ngày tính change-frequency cho `detect-changes --churn` |
| `CRG_TOOLS` | Danh sách tool expose khi chạy `serve`/`mcp` (comma-separated), tương đương cờ `--tools` |
| `CRG_RECURSE_SUBMODULES` | Fallback mặc định cho tham số `recurse_submodules` khi build |

---

## 8. Đầu ra khi PIPE / redirect (không phải terminal tương tác)

Từ bản build gần nhất, `crg` tự ép `stdout`/`stderr` sang UTF-8
(`errors="replace"`) ngay từ đầu `main()` — an toàn khi gọi qua pipe hoặc
subprocess (không còn crash `UnicodeEncodeError` trên Windows console non-UTF8).

---

## 9. Exit code tổng hợp

| Trường hợp | Exit code |
|---|---|
| Lệnh chạy thành công | 0 |
| Lỗi runtime (repo không tồn tại, git lỗi, tool exception...) | 1 (qua `sys.exit(1)`/`SystemExit(1)`) |
| `--version` | 0, in version rồi thoát ngay |
