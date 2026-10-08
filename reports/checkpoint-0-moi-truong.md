# Checkpoint 0 — Chuẩn bị Môi trường

## Mục tiêu
Thiết lập đầy đủ môi trường Python với các dependency của lab và xác thực cấu hình LangSmith, LLM provider đã hoạt động chính xác trước khi bắt đầu các bước thực thi.

## Các bước đã làm

### 1. Tạo Virtual Environment
```bash
python -m venv venv
source venv/bin/activate
```
- Sử dụng **Python 3.11.15** (xác minh bằng `python --version`)
- Virtual environment được tạo và kích hoạt thành công

### 2. Cài đặt Dependencies
```bash
pip install -r requirements.txt
pip install "langchain-community<0.4"  # Bắt buộc
```
- Cài đặt tất cả các package chính: `langchain`, `langchain-core`, `langchain-openai`, `langsmith`, `faiss-cpu`, `ragas`, `guardrails-ai`
- **Xử lý vấn đề quan trọng**: Phiên bản `langchain-community` 0.4.2 (trong `requirements.txt`) gây lỗi khi `import ragas` vì thiếu module `langchain_community.chat_models.vertexai`. Giải pháp: Downgrade xuống **0.3.31** (< 0.4)
- Các phiên bản cụ thể: ragas 0.4.3, guardrails-ai 0.11.0, langsmith 0.14.4

### 3. Cấu hình `.env`
```bash
cp .env.example .env
```
Điền các biến môi trường bắt buộc:
- `LANGCHAIN_TRACING_V2=true`
- `LANGCHAIN_API_KEY` (LangSmith API key: `lsv2_...`)
- `LANGCHAIN_PROJECT=day22-lab`
- `PROVIDER=openai`
- `OPENAI_API_KEY` (Provider key)

### 4. Xác thực Cấu hình
```bash
cd src && python config.py
```
**Kết quả**: ✅ Config OK | Provider: OPENAI | Project: day22-lab

## Kết quả / Bằng chứng
- ✓ Virtual environment hoạt động với Python 3.11.15
- ✓ Tất cả dependencies được cài đặt thành công (sau khi downgrade langchain-community)
- ✓ File `.env` được tạo với các biến cần thiết
- ✓ Script xác thực config.py chạy thành công
- ✓ `.env` không bị commit (kiểm tra bằng `git status`)

## Vấn đề Gặp phải & Cách xử lý

### Vấn đề 1: Import RAGAS lỗi với langchain-community 0.4.2
**Lỗi**: `ImportError: No module named langchain_community.chat_models.vertexai`

**Nguyên nhân**: Phiên bản 0.4.2 có sự thay đổi cấu trúc module không tương thích với ragas 0.4.3

**Giải pháp**: Downgrade langchain-community từ 0.4.2 xuống 0.3.31
```bash
pip install "langchain-community<0.4"
```

## Kiến thức Rút ra
1. **Thứ tự import quan trọng**: Biến môi trường LangSmith (`LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`) **phải được set trước khi import bất kỳ module LangChain nào**. Đây là lý do tất cả các file bước đều `import config` đầu tiên — `config.py` tự động tải `.env` và set `os.environ` trước khi các module khác import LangChain.

2. **Dependency version management**: Các phiên bản LLM framework có thể xung đột với nhau. RAGAS 0.4.3 cần langchain-community < 0.4. Phải kiểm tra compatibility matrix giữa các package.

3. **Multi-provider support**: Cấu trúc `config.py` hỗ trợ 5 LLM providers (OpenAI, Gemini, Anthropic, Ollama, OpenRouter) qua cùng một file cấu hình, giảm đặc điểm logic provider vào code chính.

## Liên hệ với RUBRIC
Checkpoint 0 không được chấm điểm riêng nhưng là **điều kiện tiên quyết** cho CP1-4. Nếu cấu hình sai:
- CP1: Không có traces (−10đ vì thiếu `LANGCHAIN_TRACING_V2=true`)
- CP3: Không thể import RAGAS, script crash

## Việc Còn lại
- ✓ CP0 hoàn tất, sẵn sàng cho CP1
- Tiếp theo: Tạo RAG pipeline cơ bản với LangSmith tracing (CP1)
