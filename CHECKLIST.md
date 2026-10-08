# CHECKLIST — Day 22: LangSmith + Prompt Versioning

> Tổng hợp từ README, CHECKPOINTS, RUBRIC, SUBMISSION, RULES và khảo sát code. Tick `[x]` khi xong.
> **Bài cá nhân · ~3–4 giờ · Deadline 23:59 ngày 08/10/2026 (GMT+7)** · Chấm theo commit cuối trước deadline.

**Điểm:** 4 nhiệm vụ × 25đ = 100đ, thưởng cộng tối đa +10đ, trừ 10đ nếu lộ API key.

## Trạng thái hiện tại của repo
- [x] `.env` chưa tồn tại (`.env.example` có sẵn, `.gitignore` đã chứa `.env`)
- [x] `evidence/` mới chỉ có `.gitkeep`
- [x] Các file `src/01..04` còn nguyên TODO (số dòng TODO bên dưới là gần đúng)

---

## Checkpoint 0 — Môi trường (~30 phút)
- [x] `python -m venv venv` rồi activate
- [x] `pip install -r requirements.txt`
- [x] `pip install "langchain-community<0.4"` (bắt buộc, bản 0.4 làm `import ragas` lỗi)
- [x] Đăng ký LangSmith, tạo API key (`lsv2_...`)
- [x] `cp .env.example .env`, điền: `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT`, `PROVIDER`, key của provider
  - Anthropic/OpenRouter vẫn cần `OPENAI_API_KEY` cho embeddings
- [ ] Windows: `export PYTHONUTF8=1` (nếu dùng `tee`)
- [x] `cd src && python config.py` in `✅ Config OK`
- [x] `git status` không thấy `.env`

---

## Checkpoint 1 — RAG + LangSmith tracing (25đ) · `src/01_langsmith_rag_pipeline.py`
- [x] `setup_vectorstore()` (~l.42–53): `get_embeddings` → `load_knowledge_base` → `split_text(500, 50)` → `build_vectorstore`
- [x] `RAG_PROMPT` (~l.58–63): system có `{context}` + human `{question}`
- [x] `build_rag_chain()` (~l.79–94): retriever `k=3`, `format_docs`, LCEL chain, **trả về `(chain, retriever)`**
- [x] `ask()` (~l.100–108): `@traceable(name="rag-query", tags=["rag","step1"])` **ngay trên `def`**, `chain.invoke(question)`
- [x] `main()` (~l.120–130): gọi `setup_vectorstore`, `build_rag_chain`, lặp 50 câu hỏi
- [x] Chạy `python 01_langsmith_rag_pipeline.py` (~2 phút), không lỗi, đủ 50 Q/A
- [x] Dashboard LangSmith có **≥ 50 traces** `rag-query`; mở 1 trace thấy câu hỏi + 3 context + câu trả lời
- [x] 📸 `evidence/01_langsmith_traces.png` (che key/email)

## Checkpoint 2 — Prompt Hub & A/B routing (25đ) · `src/02_prompt_hub_ab_routing.py`
- [x] Đổi `PROMPT_V1_NAME` / `PROMPT_V2_NAME` thành tên riêng của bạn (~l.31–33), không dùng mặc định `my-rag-prompt-*`
- [x] Viết `SYSTEM_V1` (~l.37–42, kết thúc bằng `\n\nContext:\n{context}`) và `SYSTEM_V2` (~l.49–53, cũng kết thúc `{context}`), **bắt buộc có `{context}`**
- [x] `push_prompts_to_hub` (~l.69, 76): `client.push_prompt(...)` cho cả 2
- [x] `pull_prompts_from_hub` (~l.96, 104): `client.pull_prompt(...)` cho cả 2
- [x] `get_prompt_version` (~l.125–129): MD5(`request_id`) % 2, trả về **tên prompt**
- [x] `ask_ab()` (~l.133–154): `@traceable(name="ab-rag-query", tags=["ab-test","step2"])`, trả dict `{question, answer, version}`
- [x] `main()` (~l.174–201): `Client(api_key=config.LANGSMITH_API_KEY)`, push, pull, retriever `k=3`, gọi `ask_ab`
- [x] Chạy: `python 02_prompt_hub_ab_routing.py | tee ../evidence/02_ab_routing_log.txt`
- [x] Log có `↓ Đã pull ... từ Hub` cho **cả 2** prompt (không rơi vào fallback local)
- [x] Log có cả nhãn `[prompt-v1]` và `[prompt-v2]`
- [x] Chạy lại: cùng `request_id` → cùng phiên bản (lỗi `409 Nothing to commit` là bình thường)
- [x] 📸 `evidence/02_prompt_hub.png` (2 prompt trên Prompt Hub)
- [x] 📄 `evidence/02_ab_routing_log.txt`

## Checkpoint 3 — RAGAS (25đ) · `src/03_ragas_evaluation.py` ⏱ 15–30 phút, bắt đầu sớm
- [x] Copy `SYSTEM_V1`/`SYSTEM_V2` **giống hệt** bước 2 (~l.40–42, 49–52), **bắt buộc có `{context}`**, không thì faithfulness rất thấp
- [x] `run_rag()` (~l.77–94): `contexts` là `list[str]` (**không ghép chuỗi**), `ctx_str` chỉ để đưa vào prompt; trả `{answer, contexts}`
- [x] `collect_rag_outputs()` (~l.110–119): dùng `qa["question"]`, lấy `answer` và `contexts`
- [x] `build_ragas_dataset()` (~l.137–148): `SingleTurnSample(user_input, response, retrieved_contexts, reference)` → `EvaluationDataset`
- [x] `run_ragas_eval()` (~l.161–181): `evaluate(dataset, metrics=[4 chỉ số], llm=llm_eval, embeddings=emb_eval)`
- [x] `main()` (~l.208–245): `setup_vectorstore()`, ghi `data/ragas_report.json`
- [x] Chạy `python 03_ragas_evaluation.py` (đừng đóng terminal)
- [x] Report có điểm **V1 và V2**, đủ 4 chỉ số, 50 cặp mỗi bản
- [x] **Faithfulness ≥ 0.8** ở ít nhất 1 bản (chưa đạt: kiểm tra `{context}`, giảm `chunk_size`, tăng `k`)
  > **Tuning policy**: Nếu faithfulness < 0.8 ở lần chạy đầu: thử kiểm tra `{context}` trước; sau đó mới cho phép điều chỉnh `chunk_size`/`k`; ghi lại thay đổi và lý do trong reports/checkpoint-3-*.md
- [x] `cp ../data/ragas_report.json ../evidence/03_ragas_report.json`
- [x] `python -m json.tool ../evidence/03_ragas_report.json` hợp lệ
- [x] 📸 `evidence/03_ragas_scores.png` (bảng V1 vs V2)

## Checkpoint 4 — Guardrails (25đ) · `src/04_guardrails_validator.py`
> ⚠️ Dùng `FailResult(error_message=..., fix_value=...)` để thay output. `PassResult(value_override=...)` là cách cũ, không còn hiệu lực.
- [x] `PIIDetector.validate()` (~l.78–94): duyệt `PII_PATTERNS` bằng regex, thay bằng `[TYPE_REDACTED]`; có PII → `FailResult(error_message=..., fix_value=...)`, sạch → `PassResult()`
- [x] `@register_validator` có sẵn trên cả 2 validator (không dùng validator từ Hub)
- [x] `JSONFormatter._repair()` (~l.129–133): nháy đơn → nháy đôi, xóa dấu phẩy thừa (đã có sẵn gỡ fences)
- [x] `JSONFormatter.validate()` (~l.148–166): hợp lệ → `PassResult()`; sửa được → `FailResult(error_message=..., fix_value=json.dumps(parsed, indent=2))`; không sửa được → `FailResult` với JSON dự phòng `{"error": ...}`
- [x] Tạo Guard (~l.177, 205): `Guard().use(PIIDetector(on_fail=OnFailAction.FIX))` — **`on_fail` trong constructor**, không phải `Guard.use()`
- [x] `guard.validate(text)` (~l.190, 217)
- [ ] Chạy: `python 04_guardrails_validator.py | tee ../evidence/04_pii_demo_log.txt ../evidence/04_json_demo_log.txt`
- [ ] PII: ≥ 5 test case; `Output:` chứa `[EMAIL_REDACTED]`, `[PHONE_REDACTED]`, `[SSN_REDACTED]`, `[CREDIT_CARD_REDACTED]`; case sạch giữ nguyên; Output ≠ Input
- [ ] JSON: ≥ 4 test case (hợp lệ, fences, nháy đơn, hỏng hoàn toàn)
- [ ] 📄 `evidence/04_pii_demo_log.txt`, `evidence/04_json_demo_log.txt`

---

## Chạy tổng + chuẩn bị nộp
- [ ] `cd src && python run_all.py` chạy trọn vẹn không cần sửa tay (+2đ thưởng)
- [ ] Tổng traces trên LangSmith project **≥ 100** (50 từ bước 1 + 50 từ bước 2)
- [ ] (Thưởng) `evidence/README.md`: phân tích vì sao V1 hoặc V2 điểm cao hơn (+1đ; thêm +2đ nếu phân tích nằm trong phần bình luận RAGAS)
- [x] (Thưởng) Faithfulness ≥ 0.9 ở cả 2 bản (+3đ)
- [ ] (Thưởng) Đặt LangSmith project ở chế độ chia sẻ công khai (+1đ)
- [ ] (Thưởng) Code sạch, có docstring, có xử lý lỗi/fallback (+2đ, +1đ)
- [ ] Cập nhật README gốc của repo nếu cần (SUBMISSION yêu cầu có `README.md`)

### 7 file evidence bắt buộc
- [x] `01_langsmith_traces.png`
- [x] `02_prompt_hub.png`
- [x] `02_ab_routing_log.txt`
- [x] `03_ragas_scores.png`
- [x] `03_ragas_report.json`
- [ ] `04_pii_demo_log.txt`
- [ ] `04_json_demo_log.txt`

### Kiểm tra trước khi nộp (copy từ SUBMISSION.md)
```bash
git ls-files | grep -E '^\.env$' && echo "XOÁ .env KHỎI GIT NGAY" || echo "OK: .env không bị commit"
git grep -nE 'sk-[A-Za-z0-9_-]{10,}|lsv2_[A-Za-z0-9_]{10,}|AIza[0-9A-Za-z_-]{20,}' -- . ':!.env.example' || echo "OK: không thấy key"
for f in 01_langsmith_traces.png 02_prompt_hub.png 02_ab_routing_log.txt 03_ragas_scores.png \
         03_ragas_report.json 04_pii_demo_log.txt 04_json_demo_log.txt; do
  [ -s "evidence/$f" ] && echo "OK  $f" || echo "THIẾU $f"
done
python -m json.tool evidence/03_ragas_report.json > /dev/null && echo "OK: JSON hợp lệ"
```

### Nộp bài
- [ ] Tạo repo **public** tên `K4-L3-DAY22-HoVaTen-MSSV-LLMOpsPromptVersioning` (không dấu, không khoảng trắng)
- [ ] Push lên GitHub (commit cuối **trước 23:59 08/10/2026**, không force-push)
- [ ] Mở repo bằng cửa sổ ẩn danh: public, thấy đủ file
- [ ] Nộp qua LMS: (1) URL GitHub repo, (2) URL LangSmith project

---

## Quy định cần nhớ (RULES.md)
- Bài cá nhân: có thể trao đổi ý tưởng, **không** dùng chung code/prompt/log/ảnh. Bài trùng bất thường → 0 điểm phần trùng.
- Được dùng AI để giải thích/debug, nhưng phải **hiểu và giải thích được từng dòng** code đã nộp.
- Evidence phải khớp với code và tạo từ máy bạn; evidence giả → 0 điểm cả bài.
- Tên prompt trên Hub phải là tên riêng của bạn.
- **Không commit `.env` hay API key** (−10đ). Lỡ lộ key → revoke ngay, xóa file khỏi commit mới là chưa đủ.
- Blur API key, email cá nhân, org-id trong ảnh chụp màn hình. Chỉ dùng PII **giả** cho test.
- Commit sau deadline không được chấm; nộp muộn bị trừ điểm.

## Bảng trừ điểm cần tránh
| Lỗi | Phạt |
|---|---|
| Lộ API key / commit `.env` | −10 |
| Không bật `LANGCHAIN_TRACING_V2` → không có trace | −10 |
| < 50 traces | −5 |
| Trace không có context (chain sai) | −3 |
| Chỉ 1 prompt trên Hub | −8 |
| Routing ngẫu nhiên | −5 |
| Không pull từ Hub (dùng local) | −4 |
| Log không có nhãn v1/v2 | −3 |
| Chỉ đánh giá 1 phiên bản prompt | −5 |
| Thiếu mỗi chỉ số RAGAS | −2 |
| Faithfulness < 0.8 cả 2 bản | −5 |
| Không lưu `ragas_report.json` | −2 |
| Dùng validator có sẵn từ Hub | −5 |
| `on_fail` đặt trong `Guard.use()` | −3 |
| PII không dùng regex | −3 |

## Lưu ý kỹ thuật nhanh
- Biến môi trường LangSmith phải được đặt **trước khi import LangChain** (mọi file bước đều `import config` đầu tiên).
- RAGAS 0.4: `result[metric]` là list → dùng `numpy.mean()`; cảnh báo deprecated khi import có thể bỏ qua.
- Log `LLM returned 1 generations instead of requested 3` và cảnh báo `opentelemetry ... Failed to export spans` là bình thường.
- Script luôn in thông báo hoàn thành kể cả khi key sai → **dashboard LangSmith mới là bằng chứng**.
