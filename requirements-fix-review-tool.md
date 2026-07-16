# Yêu cầu thiết kế lại Tool Review Automation Test

## 1. Kiến trúc bắt buộc

Tách biệt hoàn toàn 2 trách nhiệm, không được gộp:

- **Data Resolver**: code thường (Python/Java/Bash), KHÔNG gọi kiro-cli, KHÔNG dùng LLM dưới bất kỳ hình thức nào. Đọc và resolve toàn bộ dữ liệu cần thiết, đảm bảo 100% đầy đủ hoặc raise lỗi rõ ràng (fail loudly, không skip âm thầm).
- **LLM Reviewer**: gọi `kiro-cli chat --no-interactive` với 1 custom agent cấu hình `"tools": []` (rỗng hoàn toàn, không chỉ để trống `allowedTools`). Toàn bộ dữ liệu đã resolve được đưa vào qua **stdin**, agent không được có bất kỳ quyền đọc file/tool nào.

## 2. Cách gọi Kiro CLI (headless mode)

**Agent config** `.kiro/agents/testcase-reviewer.json`:

```json
{
  "name": "testcase-reviewer",
  "description": "Review testcase data so với source code, không truy cập file/tool nào",
  "tools": [],
  "allowedTools": [],
  "model": "claude-sonnet-5",
  "prompt": "<toàn bộ instruction ở mục 5-6: schema output, quy tắc location+reason, danh sách loại lỗi cần kiểm tra. Chỉ đánh giá dựa trên nội dung nhận qua input, không giả định hay bịa dữ liệu ngoài input.>"
}
```

**Lệnh gọi**:

```bash
cat resolved_context_TC005.json | kiro-cli chat --no-interactive --agent testcase-reviewer "Review testcase TC005 theo instructions đã cấu hình trong agent"
```

**Output có cấu trúc**: agent chỉ được in ra đúng 1 khối JSON bọc giữa 2 marker cố định (`<<<RESULT_JSON>>> ... <<<END_RESULT_JSON>>>`). Script bên ngoài parse stdout, trích phần giữa marker, validate theo schema (mục 5.5/6.3).

**Retry**: nếu parse/validate thất bại → gọi lại `kiro-cli chat --no-interactive` tối đa N lần (2-3 lần), kèm lỗi validate cụ thể vào prompt lần gọi lại. Thất bại sau N lần → ghi lỗi rõ ràng vào report, không trả kết quả rỗng/mặc định.

**Concurrency**: mỗi lời gọi per-testcase là 1 process riêng — giới hạn số process chạy song song (ví dụ `xargs -P` hoặc queue trong script điều phối).

## 3. Data Resolver

### 3.1 Input
- Đường dẫn tới class component test hoặc file CSV.

### 3.2 Xử lý — deterministic 100%
1. Đọc toàn bộ file CSV, parse header + toàn bộ row (trừ row đầu).
2. Với mỗi row: lấy `testcase key`, `testcase name`; lấy path trong cột `prepare data` → đọc file prepare; trong file prepare, resolve toàn bộ reference lồng bên trong (mock file, SQL insert trực tiếp, hoặc path `.sql` khác) — không dừng giữa chừng; lấy toàn bộ cột `expectation data` (schema động, không hardcode tên cột).
3. Fail loudly nếu bất kỳ file/reference nào không tồn tại/không đọc được/parse lỗi — ghi rõ testcase key, row, path lỗi. Không skip âm thầm.
4. Không dùng LLM ở bước này dưới bất kỳ hình thức nào.

### 3.3 Detect cột expectation: path hay JSON trực tiếp
Detect theo từng cell riêng lẻ (không theo tên cột), thứ tự cố định:
1. Trim khoảng trắng.
2. Bắt đầu bằng `{`/`[` → parse như JSON trực tiếp. Thành công → dùng luôn. Thất bại → fail loudly ("giá trị giống JSON nhưng malformed"), không fallback sang path.
3. Không bắt đầu bằng `{`/`[` → coi là path, kiểm tra file tồn tại. Tồn tại → đọc + parse JSON. Không tồn tại → fail loudly ("path không tìm thấy file").
4. Không có nhánh nào được phép âm thầm trả về rỗng.

Sau resolve, chuẩn hoá mọi nguồn (path hay inline) về cùng 1 dạng JSON object trước khi đưa cho LLM.

### 3.4 Output

```json
{
  "testcase_key": "TC005",
  "resolve_status": "OK",
  "prepare_resolved": {
    "prepare_file": "path/to/prepare.yaml",
    "mock_data": { "...": "..." },
    "sql_inserts": ["INSERT INTO ...", "..."]
  },
  "expectation": { "field_a": "...", "field_b": "..." }
}
```

Khi lỗi:

```json
{
  "testcase_key": "TC007",
  "resolve_status": "FAILED",
  "error_detail": {
    "location": "prepare/TC007/prepare.yaml, dòng tham chiếu sql_ref",
    "description": "File SQL tham chiếu không tồn tại",
    "reason": "path 'sql/insert_fee_rule.sql' không tìm thấy trong repo"
  }
}
```

### 3.5 Kiểm thử bắt buộc
Chạy resolver 5 lần liên tiếp trên cùng input → output JSON phải giống hệt nhau byte-by-byte (trừ timestamp nếu có).

## 4. Tối ưu payload gửi cho LLM

Gửi đầy đủ 100% dữ liệu đã resolve — không cắt, không tóm tắt, không lược field nào. Chỉ áp dụng biến đổi KHÔNG làm mất thông tin:

1. **Flatten JSON thành key-path : value** (giữ nguyên toàn bộ field) — giúp `location` khi báo lỗi trỏ đúng field path.
2. **Parse SQL insert thành key-value** (`{table, col1: val1, ...}`) thay vì để nguyên text SQL.

Không áp dụng: prune field theo "không quan trọng", diff theo baseline, filter bớt phần tử mảng — mọi kỹ thuật cắt dữ liệu đều cấm vì rủi ro bỏ sót lỗi thật.

Nếu 1 testcase có quá nhiều cột expectation khiến 1 lời gọi quá lớn: chia theo nhóm cột liên quan tới entity/DTO nào (không cắt data), mỗi nhóm 1 lời gọi riêng, output kèm `testcase_key` + `expectation_group` để merge đúng.

## 5. LLM Reviewer — Review dữ liệu testcase

### 5.1 Input mỗi lời gọi
- JSON đã resolve từ mục 3 (đầy đủ, đã format theo mục 4).
- Source code liên quan, **phải trace xuống tới tầng repository/DAO/JPA/SQL thực thi ghi DB**, không chỉ method entry point (controller/consumer) — thiếu tầng này thì không thể phát hiện lỗi ở mục 5.2.6.

### 5.2 Danh sách loại lỗi bắt buộc phát hiện

Mọi lỗi phát hiện bắt buộc có đủ 3 trường: `location` (vị trí cụ thể), `description` (lỗi là gì), `reason` (vì sao là lỗi, dựa trên đối chiếu code/data thật, không suy đoán).

**5.2.1 weak_assertions** (per-testcase) — assertion tồn tại nhưng lỏng lẻo/sai (ví dụ chỉ check status code mà không check giá trị trả về).

**5.2.2 mock_data_issues** (per-testcase) — đối chiếu `prepare_resolved.mock_data` với field mà code service thực sự đọc từ dependency được mock:
- THIEU: code cần field nào đó nhưng mock không cung cấp/null.
- THUA: mock cung cấp field code không dùng tới.

**5.2.3 name_data_mismatch** (per-testcase) — đối chiếu 3 chiều `testcase_name` ↔ `prepare_resolved` ↔ `expectation`. Báo lỗi nếu 2 trong 3 mâu thuẫn nhau.

**5.2.4 db_verification_gap** (per-testcase) — xác định luồng (dựa trên prepare data của testcase) ghi/update bảng nào, cột nào; đối chiếu với expectation của testcase đó có verify lại đúng bảng/cột đó không. Nếu có ghi mà không verify → báo lỗi.

**5.2.5 coverage_gap** (aggregate, theo cả file) — nhánh/điều kiện trong code chưa testcase nào chạm tới (dựa trên coverage report thật).

**5.2.6 missing_business_cases** (aggregate) — kịch bản nghiệp vụ quan trọng code có xử lý nhưng chưa testcase nào phủ đúng ý nghĩa, kể cả khi dòng code đó vẫn được test khác chạm qua.

**5.2.7 unverified_response_fields** (aggregate) — đối chiếu toàn bộ field response DTO thực sự expose với toàn bộ field mà các cột expectation trong cả file đã từng đề cập — field nào chưa testcase nào verify tới thì báo thiếu.

### 5.3 Granularity lời gọi
- **Per-testcase, song song**: 5.2.1-5.2.4 (chỉ cần data của chính testcase đó).
- **Aggregate, 1 lần cho cả file** (hoặc chia theo nhóm nhánh code nếu quá lớn): 5.2.5-5.2.7 (cần thấy toàn bộ tập testcase để biết cái gì đã được phủ).
- Mọi lời gọi đều phải trả về đúng `testcase_key` (per-testcase) hoặc phạm vi rõ ràng (aggregate) để report merge đúng.

### 5.4 Schema output

**Aggregate:**

```json
{
  "class_name": "FeeCalculationComponentTest",
  "coverage_gap": [
    { "location": "ClassName.methodName - nhánh if (...)", "description": "...", "reason": "..." }
  ],
  "missing_business_cases": [
    { "location": "...", "description": "...", "reason": "..." }
  ],
  "unverified_response_fields": [
    { "location": "FeeCalculationResponse.discountBreakdown", "description": "...", "reason": "..." }
  ]
}
```

**Per-testcase:**

```json
{
  "testcase_key": "TC005",
  "expectation_group": "fee",
  "weak_assertions": [ { "location": "...", "description": "...", "reason": "..." } ],
  "mock_data_issues": [ { "type": "THIEU | THUA", "location": "...", "description": "...", "reason": "..." } ],
  "name_data_mismatch": [ { "location": "...", "description": "...", "reason": "..." } ],
  "db_verification_gap": [ { "location": "bảng fee_transaction, cột status", "description": "...", "reason": "..." } ],
  "verdict": "SUFFICIENT | INSUFFICIENT | NEEDS_REVIEW"
}
```

## 6. Review code verify (assertion logic) của Component Test Framework

Review riêng biệt, khác với mục 5 (mục 5 review data testcase, mục này review chính code Java thực thi assertion trong framework/base class tự viết).

### 6.1 Input
- Toàn bộ source code framework/base class đọc CSV + thực hiện assertion (code dùng chung, không phải 1 class cụ thể).
- Danh sách toàn bộ tên cột đã từng xuất hiện trong các file CSV thực tế.
- Đưa qua stdin, cùng agent `"tools": []` như mục 2.

### 6.2 Danh sách kiểm tra bắt buộc

1. **Cột khai báo nhưng không được verify**: cột tồn tại trong CSV nhưng code không đọc/so sánh tới.
2. **So sánh sai kiểu/logic**: so sánh String bằng `==`, ép kiểu sai, so sánh số dạng String không nhất quán.
3. **Nuốt lỗi assertion (silent pass)**: try-catch bắt `AssertionError`/`Exception` rồi chỉ log thay vì fail test.
4. **Hardcode đè lên data từ CSV**: giá trị hardcode ghi đè giá trị đọc từ CSV.
5. **Verify DB có thực sự chạy**: đoạn code query lại DB có bị comment/disable/return sớm không.
6. **Ignore field trong verify code** (ưu tiên cao nhất): liệt kê toàn bộ field bị ignore (`ignoringFields`, `JSONCompareMode` exclude, `@JsonIgnore`, denylist, regex loại trừ...), với mỗi field:
   - Phân loại hợp lý (timestamp, id tự sinh, UUID) hay đáng ngờ (field nghiệp vụ không có lý do rõ ràng).
   - Đối chiếu với toàn bộ cột expectation trong CSV — field bị ignore nhưng có testcase khai báo giá trị expectation cụ thể → báo mâu thuẫn nghiêm trọng.
   - Ghi rõ phạm vi ignore: GLOBAL (toàn class) hay PER_TESTCASE.

### 6.3 Schema output

```json
{
  "framework_class": "BaseComponentTestRunner",
  "verify_code_issues": [
    { "location": "BaseComponentTestRunner.verifyResponse(), dòng so sánh field 'amount'", "description": "so sánh amount bằng == thay vì equals/compareTo cho BigDecimal", "reason": "..." }
  ],
  "ignored_field_review": [
    {
      "field": "discountRate",
      "location": "BaseComponentTestRunner.verifyResponse(), ignoringFields(...)",
      "scope": "GLOBAL | PER_TESTCASE",
      "justified": false,
      "conflict_with_expectation": ["TC005", "TC012"],
      "description": "...",
      "reason": "..."
    }
  ]
}
```

### 6.4 Tần suất chạy
Chỉ chạy lại khi code framework thay đổi, tách thành script riêng độc lập với pipeline review data ở mục 3-5.

## 7. Report tổng hợp

Phân biệt rõ 4 loại kết quả, không gộp chung:
- Testcase resolve OK + review OK.
- Testcase resolve FAILED — liệt kê riêng, ghi rõ lý do để dev sửa file prepare/CSV.
- Testcase resolve OK nhưng review FAILED (sau retry) — ghi rõ đây là lỗi hệ thống (Kiro CLI/model), không phải lỗi testcase.
- Kết quả review code framework (`verify_code_issues`, `ignored_field_review`) — hiển thị riêng biệt hoàn toàn, không gộp theo từng testcase.

## 8. Acceptance Criteria

1. Chạy toàn bộ pipeline 3 lần liên tiếp trên cùng input → danh sách testcase resolve OK/FAILED giống hệt nhau ở cả 3 lần.
2. Không testcase nào bị bỏ sót khỏi report (dù OK hay FAILED).
3. Output review luôn đúng schema JSON đã định nghĩa, 100% parse được, không cần sửa tay.
4. Log chi tiết từng bước resolve (file nào đã đọc, giá trị gì) để audit lại khi cần.
5. Verify agent `testcase-reviewer` thực sự 0 tool được load:
   ```bash
   kiro-cli agent validate .kiro/agents/testcase-reviewer.json
   kiro-cli chat --agent testcase-reviewer
   # gõ: /tools   → phải hiện danh sách rỗng
   ```
   Kiểm tra lại mỗi khi nâng version kiro-cli.
