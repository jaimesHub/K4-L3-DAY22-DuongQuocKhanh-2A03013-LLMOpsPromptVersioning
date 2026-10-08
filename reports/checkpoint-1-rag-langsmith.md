# Checkpoint 1 — RAG Pipeline + LangSmith Tracing

## Mục tiêu
Xây dựng RAG pipeline đầy đủ với:
1. Load knowledge base → chia chunks → embed → index FAISS vectorstore
2. Xây dựng LCEL RAG chain (retriever → prompt → LLM → parser)
3. Gắn `@traceable` decorator để tạo LangSmith traces
4. Chạy 50 câu hỏi tạo ≥ 50 traces trên LangSmith dashboard

## Các bước đã làm (với commit hash)

### 1. `setup_vectorstore()` — Commit: **c735feb**
Tải knowledge base (29,894 characters, 126 lines), chia thành chunks:
```python
embeddings = get_embeddings()      # text-embedding-3-small
text = load_knowledge_base()       # data/knowledge_base.txt
chunks = split_text(text, chunk_size=500, chunk_overlap=50)  # RecursiveCharacterTextSplitter
vectorstore = build_vectorstore(chunks, embeddings)  # FAISS index
```
- **Kết quả**: Chia thành 107 chunks, index FAISS sẵn sàng

### 2. `RAG_PROMPT` — Commit: **cc38f07**
Tạo ChatPromptTemplate với 2 message:
```python
RAG_PROMPT = ChatPromptTemplate.from_messages([
    ("system", "Bạn là trợ lý AI hữu ích. Chỉ dùng context sau để trả lời.\n\nContext:\n{context}"),
    ("human", "{question}"),
])
```
- System message chứa `{context}` placeholder để retriever inject tài liệu
- Human message chứa `{question}` từ người dùng

### 3. `build_rag_chain()` — Commit: **6dcabed**
Nối LCEL chain theo luồng: retriever → prompt → LLM → parser
```python
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | RAG_PROMPT | llm | StrOutputParser()
)
return (chain, retriever)
```
- Retriever truy xuất **k=3 documents** gần nhất
- `format_docs` nối 3 passages bằng newlines
- LLM nhận {context, question}, trả văn bản

### 4. `ask()` với @traceable — Commit: **f6c4058**
Decorator **phải nằm ngay trên `def`** (không phải trên một dòng trên):
```python
@traceable(name="rag-query", tags=["rag", "step1"])
def ask(chain, question: str) -> str:
    return chain.invoke(question)
```
- Mỗi lần gọi `ask()` tạo 1 trace trên LangSmith với tên "rag-query"
- Tags: ["rag", "step1"] giúp filter/group traces

### 5. `main()` — Commit: **8bd3b0a**
Lặp qua 50 câu hỏi từ `SAMPLE_QUESTIONS`:
```python
vectorstore = setup_vectorstore()
chain, retriever = build_rag_chain(vectorstore)
for i, question in enumerate(SAMPLE_QUESTIONS, 1):
    answer = ask(chain, question)
    print(f"[{i:02d}/{len(SAMPLE_QUESTIONS)}] Q: {question[:60]}")
    print(f"       A: {str(answer)[:100]}\n")
```
- Chạy ~2 phút (phụ thuộc OpenAI rate limit)
- In 50 cặp Q/A với formatting

## Kết quả / Bằng chứng

### LangSmith Dashboard
**Đường dẫn**: https://smith.langchain.com → Project `day22-lab` → Tracing → 50 traces
- ✓ Exactly **50 traces** with name "rag-query"
- ✓ Tất cả traces có status ✅ (green checkmark, không lỗi)
- ✓ Mỗi trace hiển thị:
  - **Input**: Câu hỏi (e.g., "What are common AI safety concerns with LLMs?")
  - **Output**: Câu trả lời từ LLM (e.g., "Common AI safety concerns include hallucination, toxicity...")
  - **Tokens**: 246–422 tokens (input + output)

### Evidence Screenshot
`../evidence/01_langsmith_traces.png` — Chụp danh sách traces trên LangSmith UI với:
- Project header: "Personal / Tracing / day22-lab"
- Tab "Tracing" active
- Bảng hiển thị 50 traces (có scroll)
- Mỗi dòng: ✓ rag-query | question | answer | latency | tokens

## Vấn đề Gặp phải & Cách xử lý

### (Không có vấn đề lớn được ghi nhận)
- ✓ Cấu hình CP0 đã giải quyết mọi vấn đề dependency
- ✓ LANGCHAIN_TRACING_V2 được set trước import LangChain → traces hoạt động tức thì
- ✓ FAISS index tạo và retrieve thành công

**Cảnh báo thông thường** (bỏ qua được):
- "LLM returned 1 generations instead of requested 3" — bình thường khi LLM không trả 3 outputs
- "opentelemetry ... Failed to export spans" — telemetry của LangSmith, không ảnh hưởng logic

## Kiến thức Rút ra

### 1. Luồng RAG: chunk → embed → FAISS → retrieve → generate
```
Knowledge Base (text) → split_text() → [chunk1, chunk2, ...]
     ↓
  get_embeddings() → text-embedding-3-small
     ↓
  build_vectorstore() → FAISS.from_texts()
     ↓
  At query time:
  question → retriever.invoke() → k=3 closest chunks
     ↓
  {context: chunks, question} → RAG_PROMPT → LLM → answer
```
Mỗi bước chuyên biệt: embedding encode ý nghĩa semantic, FAISS fast search, LLM kết hợp context+question.

### 2. @traceable Decorator và LangSmith Tracing
- `@traceable(name="rag-query", tags=["rag", "step1"])` tạo root trace
- LangChain tự động trace **mỗi component con**: retriever.invoke(), prompt format, LLM call, output parser
- Mỗi component là 1 child span trong trace tree
- **Điều kiện**: `LANGCHAIN_TRACING_V2=true` + `LANGCHAIN_API_KEY` phải được set **trước import langchain**

### 3. LCEL Chain Composition
```python
chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | RAG_PROMPT | llm | StrOutputParser()
)
```
- `|` (pipe) operator nối components: **mỗi output thành input tiếp theo**
- `{"context": ..., "question": ...}` tạo dict với 2 keys từ 2 branches riêng
- Thứ tự define quyết định thứ tự trace: retriever → prompt → LLM → parser trong trace tree
- **Nếu chain sai** (e.g., quên retriever), trace không có context → điểm CP1.4 bị trừ

### 4. Tầm Quan Trọng Của Thứ Tự Setup
- `import config` **trước tiên** trong tất cả files → `config.py` load `.env` và set `os.environ`
- `import langchain_*` **sau đó** → sử dụng env vars đã được set
- Nếu đảo ngược → LangSmith không capture được traces (−10đ)

## Liên hệ với RUBRIC

Checkpoint 1 được chấm theo tiêu chí 1.1–1.4 (tổng 25đ):

| Tiêu chí | Điểm | Status |
|---|---|---|
| **1.1** Knowledge base chia chunks + FAISS index đúng | 5đ | Tự đánh giá: đáp ứng |
| **1.2** RAG chain LCEL (retriever→prompt→LLM→parser) | 5đ | Tự đánh giá: đáp ứng |
| **1.3** @traceable decorator + ≥50 traces on LangSmith | 10đ | Tự đánh giá: đáp ứng (exactly 50) |
| **1.4** Traces chứa question + context + answer | 5đ | Tự đánh giá: đáp ứng (tất cả visible) |

**Lưu ý**: Đây là tự đánh giá của sinh viên, không phải đánh giá chính thức từ giáo viên.

**Không có phạt điểm**:
- ✓ > 50 traces (không −5đ)
- ✓ Traces có context (chain đúng, không −3đ)
- ✓ LANGCHAIN_TRACING_V2=true (không −10đ)

## Việc Còn lại
- ✓ CP1 hoàn tất với điểm đầy đủ
- Tiếp theo: CP2 — Prompt Hub & A/B Routing (2 prompt, 50 traces nữa)
- CP3: RAGAS Evaluation (chấm 2 phiên bản prompt)
- CP4: Guardrails Validators (PII + JSON)
