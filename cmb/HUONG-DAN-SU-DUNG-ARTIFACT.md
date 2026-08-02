# Hướng dẫn sử dụng sau khi tải artifact từ GitHub Actions

Áp dụng cho artifact tải về từ workflow **"Quick Windows exe build"**
(`build-windows-exe-quick.yml`) và **"Quick Linux binary build"**
(`build-linux-quick.yml`).

## 1. Tải artifact

1. Vào tab **Actions** trên GitHub → chọn đúng workflow (Windows hoặc Linux) → chọn
   lần chạy đã hoàn tất → kéo xuống mục **Artifacts** → bấm tải.
2. GitHub luôn đóng gói artifact thành 1 file `.zip` bên ngoài, nên bạn sẽ có
   **2 lớp nén lồng nhau**:
   - Windows: `codebase-memory-mcp-windows-exe.zip` (lớp ngoài, do GitHub tạo) → bên
     trong là `codebase-memory-mcp-windows-amd64.zip` (lớp trong, do build tạo).
   - Linux: `codebase-memory-mcp-linux-binary.zip` (lớp ngoài) → bên trong là
     `codebase-memory-mcp-linux-amd64-portable.tar.gz` (lớp trong).

   Phải giải nén **cả 2 lớp** mới ra được file thực thi.

---

## 2. Windows

### 2.1. Giải nén

Giải nén `codebase-memory-mcp-windows-exe.zip`, rồi giải nén tiếp
`codebase-memory-mcp-windows-amd64.zip` bên trong. Bạn sẽ có 4 file:

```
codebase-memory-mcp.exe
LICENSE
install.ps1
THIRD_PARTY_NOTICES.md
```

Đặt cả thư mục này ở một vị trí cố định, ví dụ `D:\tools\codebase-memory-mcp\`.

### 2.2. Kiểm tra chạy được

Mở PowerShell hoặc cmd tại đúng thư mục đó:

```bash
.\codebase-memory-mcp.exe --version
```

Nếu in ra số phiên bản là file chạy được bình thường (không cần cài Python/Node
gì thêm — binary tự chứa toàn bộ).

### 2.3. Cấu hình cho AI coding agent (khuyến nghị — tự động)

```bash
.\codebase-memory-mcp.exe install -y
```

Lệnh này tự động:
- Copy chính nó vào `%USERPROFILE%\.local\bin\codebase-memory-mcp.exe`
- Tạo thêm alias ngắn `cbm.exe` cạnh đó — từ giờ có thể gõ `cbm` thay vì gõ
  đầy đủ `codebase-memory-mcp.exe`
- Dò và cấu hình MCP config cho các agent đang cài trên máy: **Claude Code,
  Gemini CLI, OpenCode, VS Code, Zed, Cursor, Kiro**
- Dừng các instance server cũ đang chạy (nếu có) để agent nhận config mới —
  **lưu ý**: bước này tìm process theo tên trên toàn hệ thống, không giới
  hạn theo thư mục cài, nên nếu máy đang có 1 server khác đang chạy (kể cả
  của agent khác) cũng sẽ bị dừng theo.

Sau khi chạy xong, **khởi động lại hoàn toàn** agent bạn dùng (đóng hẳn app,
không chỉ đóng cửa sổ) để nó nhận MCP server mới.

Các flag hữu ích khác của `install`:
| Flag | Ý nghĩa |
|---|---|
| `-y` / `--yes` | Không hỏi xác nhận, tự động đồng ý |
| `--dry-run` | Xem trước sẽ làm gì, không thay đổi gì thật |
| `--force` | Ghi đè binary cũ nếu đã tồn tại ở vị trí cài |
| `--reset-indexes` | Xoá luôn index cũ (mặc định giữ nguyên) |

### 2.4. Cấu hình thủ công (nếu `install` không tự nhận đúng agent)

Ví dụ với **Claude Desktop** (app GUI, khác với Claude Code CLI —
`install` ở trên KHÔNG tự cấu hình app này) — sửa file:

```
%APPDATA%\Claude\claude_desktop_config.json
```

Thêm vào:

```json
{
  "mcpServers": {
    "codebase-memory-mcp": {
      "command": "D:\\tools\\codebase-memory-mcp\\codebase-memory-mcp.exe",
      "args": []
    }
  }
}
```

Không cần thêm subcommand nào — chạy không kèm tham số là binary tự khởi
động MCP server qua stdio.

### 2.5. Nếu gặp lỗi liên quan mã hoá / ký tự lạ

Nếu MCP client chạy binary theo kiểu pipe (không phải terminal tương tác) và
gặp lỗi encoding, kiểm tra bạn đang dùng đúng bản build mới nhất (đã có fix
UTF-8 console). Có thể test thủ công bằng cách redirect output ra file:

```bash
codebase-memory-mcp.exe --version > out.log 2>&1
```

---

## 3. Linux

### 3.1. Giải nén

```bash
unzip codebase-memory-mcp-linux-binary.zip
tar -xzf codebase-memory-mcp-linux-amd64-portable.tar.gz
```

Sẽ ra 4 file:

```
codebase-memory-mcp
LICENSE
install.sh
THIRD_PARTY_NOTICES.md
```

Binary này build **static hoàn toàn** (không phụ thuộc glibc version của hệ
thống) — chạy được trên bất kỳ distro Linux nào, kể cả base image tối giản
trong Docker (Alpine, distroless...).

### 3.2. Cấp quyền chạy + kiểm tra

```bash
chmod +x codebase-memory-mcp
./codebase-memory-mcp --version
```

### 3.3. Cấu hình cho AI coding agent (tự động)

```bash
./codebase-memory-mcp install -y
```

Giống hệt Windows: copy vào `~/.local/bin/codebase-memory-mcp`, tạo thêm
symlink alias `cbm` cạnh đó (gõ `cbm` thay vì gõ đầy đủ), tự dò và cấu hình
Claude Code / Gemini CLI / OpenCode / VS Code / Zed / Cursor / Kiro.
Nếu `~/.local/bin` chưa có trong `PATH`, thêm vào shell config:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc   # hoặc ~/.zshrc
```

### 3.4. Dùng trong Docker

Vì binary static, chỉ cần copy thẳng vào image, không cần cài thêm gì:

```dockerfile
FROM debian:bookworm-slim
COPY codebase-memory-mcp /usr/local/bin/codebase-memory-mcp
ENV CBM_CACHE_DIR=/tmp/cbm
ENTRYPOINT ["codebase-memory-mcp"]
```

Build và chạy thử:

```bash
docker build -t codebase-memory-mcp .
docker run --rm -i -v "$(pwd)/my-repo:/repo" codebase-memory-mcp
```

MCP client kết nối qua stdio (`-i` để giữ stdin mở) — client sẽ cần được cấu
hình để `docker run` container này làm subprocess (giống cách trỏ tới file
`.exe`/binary thông thường, chỉ thay `command` bằng `docker` và `args` là các
tham số run tương ứng).

### 3.5. Cấu hình thủ công MCP client (nếu cần)

Tương tự Windows, chỉ khác `command` trỏ vào đường dẫn Linux:

```json
{
  "mcpServers": {
    "codebase-memory-mcp": {
      "command": "/home/<user>/.local/bin/codebase-memory-mcp",
      "args": []
    }
  }
}
```

---

## 4. Kiểm tra MCP server chạy thành công (áp dụng cả 2 hệ)

- Chạy trực tiếp không kèm tham số: nếu **không lỗi và treo chờ input** (không
  thoát ra ngay) — nghĩa là server khởi động đúng, đang chờ JSON-RPC qua
  stdin. Nhấn `Ctrl+C` để thoát sau khi test.
- Cách chắc chắn nhất: cấu hình vào agent thật, mở agent lên, kiểm tra agent
  có liệt kê được danh sách tool của server không (thường có UI hiện trạng
  thái kết nối MCP).
- Nếu lỗi, log chi tiết (traceback thật) thường nằm trong log riêng của MCP
  client (ví dụ Claude Desktop: `%APPDATA%\Claude\logs\` trên Windows).

## 5. Lệnh CLI khác có sẵn

| Lệnh | Ý nghĩa |
|---|---|
| `codebase-memory-mcp --version` | In phiên bản |
| `codebase-memory-mcp --help` | Danh sách lệnh |
| `codebase-memory-mcp install [-y]` | Cài + cấu hình agent tự động |
| `codebase-memory-mcp uninstall` | Gỡ cấu hình đã cài |
| `codebase-memory-mcp update` | Cập nhật binary |
| `codebase-memory-mcp config get/set/reset <key>` | Đọc/sửa config runtime |
| `codebase-memory-mcp cli <tool_name> --flag value` | Gọi trực tiếp 1 MCP tool qua CLI (không cần MCP client), ví dụ `index_repository` |
