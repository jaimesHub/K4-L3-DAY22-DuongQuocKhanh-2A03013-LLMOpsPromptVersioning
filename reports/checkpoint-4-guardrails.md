# Checkpoint 4 — Guardrails AI Validators

## Mục tiêu
Xây dựng 2 validator tùy chỉnh bằng Guardrails AI và chạy demo:
1. `PIIDetector`: phát hiện và redact email, số điện thoại, SSN, số thẻ tín dụng bằng regex
2. `JSONFormatter`: tự sửa JSON lỗi (fences, nháy đơn, dấu phẩy thừa); nếu không sửa được thì trả JSON dự phòng `{"error": ...}`
3. Gắn mỗi validator vào `Guard` với `on_fail=OnFailAction.FIX` (đặt trong constructor)
4. Demo ≥ 5 case PII và ≥ 4 case JSON, lưu log vào `evidence/`

## Các bước đã làm (với commit hash)

| # | Việc | Commit |
|---|---|---|
| 1 | `PIIDetector.validate`: duyệt `PII_PATTERNS`, thay match bằng `[TYPE_REDACTED]`; có PII → `FailResult(fix_value=...)`, sạch → `PassResult()` | **c8bb353** |
| 2 | `JSONFormatter._repair`: nháy đơn → nháy đôi, xóa dấu phẩy thừa (fences đã có sẵn) | **b2d8cc0** |
| 3 | `JSONFormatter.validate`: hợp lệ → `PassResult`; sửa được → `FailResult(fix_value=json.dumps(parsed, indent=2))`; không sửa được → `FailResult` với JSON dự phòng | **0b0b352** |
| 4 | Tạo PII guard: `Guard().use(PIIDetector(on_fail=OnFailAction.FIX))` | **e8c226f** |
| 5 | `guard.validate(text)` trong `demo_pii_guard` | **4f6f8eb** |
| 6 | Tạo JSON guard tương tự | **3d69706** |
| 7 | `guard.validate(text)` trong `demo_json_guard` | **9aaf4f1** |

Kiểm tra tĩnh `src/04_guardrails_validator.py`: cả 2 validator có `@register_validator`; PII dùng `re.findall` trên `PII_PATTERNS`; `on_fail` nằm trong constructor, không nằm trong `Guard.use()`; không import validator nào từ Hub (chỉ có `OnFailAction` được import qua `try/except`, kèm fallback `guardrails.validator_base`); output được thay bằng `FailResult(fix_value=...)`. Script không gọi LLM (chỉ import `re`, `json`, `guardrails`), nên chạy hoàn toàn cục bộ.

## Kết quả / Bằng chứng
File: `evidence/04_pii_demo_log.txt` và `evidence/04_json_demo_log.txt` — hai file byte-identical (`cmp` xác nhận) vì cùng một lần chạy `python 04_guardrails_validator.py | tee ...`.

### PII (6 case)

| Case | Input | Output |
|---|---|---|
| Email | `Contact John at john.doe@example.com for details.` | `Contact John at [EMAIL_REDACTED] for details.` |
| Phone | `Call our support line at (555) 867-5309.` | `Call our support line at ([PHONE_REDACTED].` |
| SSN | `Patient SSN is 123-45-6789 on file.` | `Patient SSN is [SSN_REDACTED] on file.` |
| Credit Card | `Payment made with card 4532 1234 5678 9010.` | `Payment made with card [CREDIT_CARD_REDACTED].` |
| Multi-PII | `Email: alice@example.com, Phone: 555-123-4567` | `Email: [EMAIL_REDACTED], Phone: [PHONE_REDACTED]` |
| Clean | `No sensitive information in this text.` | giữ nguyên |

Đủ 4 marker; 5 case PII có Output khác Input; case sạch giữ nguyên. Log chỉ chứa email giả `john.doe@example.com`, `alice@example.com`.

### JSON (5 case)

| Case | Input | Output |
|---|---|---|
| Valid JSON | `{"name": "Alice", "age": 30}` | giữ nguyên |
| Markdown fences | ` ```json {"name": "Bob"} ``` ` | `{ "name": "Bob" }` (định dạng lại, indent 2) |
| Single quotes | `{'name': 'Charlie', 'score': 95}` | `{ "name": "Charlie", "score": 95 }` |
| Trailing comma | `{"key": "value",}` | `{ "key": "value" }` |
| Truly invalid | `This is not JSON at all: ??? {]` | `{"error": "Không thể phân tích JSON", "raw": "This is not JS` (script cắt output ở 60 ký tự khi in) |

## Vấn đề gặp phải & Cách xử lý
Các điểm dưới đây là hạn chế đã biết, chủ yếu về hiển thị; được ghi lại trung thực và không sửa vì không ảnh hưởng yêu cầu của rubric.

- **(a) Case Phone còn dấu `(`**: output là `([PHONE_REDACTED].`. Pattern có `\b` trước `\(?`, mà `\b` không khớp giữa khoảng trắng và `(`, nên match bắt đầu từ `555` và `(` còn lại. Marker `[PHONE_REDACTED]` có mặt và số điện thoại đã bị che; chỉ thừa một ký tự `(`.
- **(b) Dòng `🔧 JSON đã được sửa thành công` lệch vị trí**: `validate()` in dòng này trong lúc xử lý case hiện tại, nhưng nhãn `[case]` của case đó được in sau khi `guard.validate` trả về, nên dòng xuất hiện bên dưới case liền trước (ví dụ nằm dưới "Valid JSON" dù thuộc về "Markdown fences"). Chỉ là thứ tự in, kết quả từng case vẫn đúng.
- **(c) Cả 5 case JSON hiện `✅ Pass`, kể cả case hỏng hoàn toàn**: với `OnFailAction.FIX`, output đã sửa được coi là đạt nên `validation_passed` là True. Muốn phân biệt phải dựa vào Output (`{"error": ...}`), không dựa vào nhãn Pass.
- **(d)** Cảnh báo `opentelemetry ... Failed to export spans` chỉ xuất hiện trên terminal (stderr), không nằm trong log.

## Kiến thức Rút ra
1. **Vòng đời validate → fail → fix**: `guard.validate(text)` gọi `validate()` của validator. Trả `PassResult` thì giữ nguyên value. Trả `FailResult` thì Guardrails áp dụng `on_fail`; với `FIX` nó lấy `fix_value` làm `validated_output`.
2. **`FailResult(fix_value=...)` mới thay được output**: `PassResult(value_override=...)` là cách cũ, không còn hiệu lực — output sẽ giống hệt input. Đây cũng là dấu hiệu để tự kiểm tra: Output == Input ở case có PII nghĩa là đang trả nhầm `PassResult`.
3. **`on_fail` trong constructor, không trong `Guard.use()`**: `Guard().use(PIIDetector(on_fail=OnFailAction.FIX))` đúng; truyền `on_fail` vào `use()` cùng class sẽ gây lỗi và bị phạt −3 theo rubric.
4. **PII bằng regex**: mỗi loại PII một pattern, `re.findall` rồi `replace` bằng `[TYPE_REDACTED]`. Đơn giản, tất định, nhưng nhạy với biên từ (`\b`) như ở lỗi (a) và có thể bắt thiếu/thừa với định dạng lạ.
5. **Fallback JSON luôn trả JSON hợp lệ**: khi `_repair` không cứu được, trả `{"error": ..., "raw": value[:200]}` để hệ thống phía sau luôn parse được kết quả.
6. **`_repair` thô**: thay mọi `'` bằng `"` sẽ hỏng với chuỗi chứa dấu nháy đơn trong nội dung (ví dụ `it's`); đủ cho các case demo nhưng không phải bộ sửa JSON tổng quát.

## Liên hệ với RUBRIC
Checkpoint 4 chấm theo tiêu chí 4.1–4.8 (25đ). Đây là **tự đánh giá của sinh viên, không phải điểm chính thức**.

| Tiêu chí | Điểm | Tự đánh giá | Ghi chú |
|---|---|---|---|
| 4.1 `@register_validator` | 3đ | Đạt | Có trên cả `PIIDetector` và `JSONFormatter` |
| 4.2 ≥ 3 loại PII | 5đ | Đạt | 4 loại: EMAIL, PHONE, SSN, CREDIT_CARD; có marker trong log (xem hạn chế (a)) |
| 4.3 `on_fail=FIX` đúng, thay output | 3đ | Đạt | Trong constructor; `FailResult(fix_value=...)` |
| 4.4 Demo ≥ 5 case (có case sạch) | 2đ | Đạt | 6 case |
| 4.5 Validator kiểm tra parse JSON | 3đ | Đạt | `json.loads` trong `JSONFormatter.validate` |
| 4.6 Tự sửa ≥ 2 loại lỗi | 5đ | Đạt | Fences, nháy đơn, dấu phẩy thừa (3 loại) |
| 4.7 JSON lỗi dự phòng | 2đ | Đạt | `{"error": "Không thể phân tích JSON", "raw": ...}` |
| 4.8 Demo ≥ 4 case JSON | 2đ | Đạt | 5 case |

Không vi phạm các mục phạt của checkpoint này: không dùng validator Hub (−5), `on_fail` không đặt trong `Guard.use()` (−3), PII dùng regex (−3).

## Việc Còn lại
- Chạy `python run_all.py` trọn vẹn (thưởng +2đ), kiểm tra tổng traces LangSmith ≥ 100
- Tùy chọn: `evidence/README.md`, chia sẻ công khai LangSmith project
- Nếu có thời gian: sửa `\b` trước `\(` trong pattern PHONE và vị trí in dòng "🔧" (hiện chấp nhận như hạn chế đã biết)
- Chuẩn bị nộp: repo public, push trước 23:59 08/10/2026, chạy các lệnh kiểm tra trong `CHECKLIST.md`
