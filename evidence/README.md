# Phân tích kết quả A/B Testing: V1 vs V2

## Bảng so sánh 4 chỉ số RAGAS

| Chỉ số | V1 | V2 | Chênh lệch |
|--------|-----|-----|----------|
| **Faithfulness** | 0.9355 | 0.9341 | -0.0014 |
| **Answer Relevancy** | 0.9083 | 0.8920 | -0.0163 |
| **Context Recall** | 1.0000 | 1.0000 | 0.0000 |
| **Context Precision** | 0.9450 | 0.9517 | +0.0067 |

*Mỗi phiên bản: n=50 cặp Q&A*

## Phong cách Prompt: So sánh V1 vs V2

| Tiêu chí | V1 | V2 |
|---------|----|----|
| **Độ dài** | 2-4 câu | 3-5 câu |
| **Tính cấu trúc** | Ngắn gọn, trực tiếp | Có tổ chức, phân tích |
| **Prompt text** | "Bạn là trợ lý AI thân thiện. Trả lời ngắn gọn (2-4 câu), chỉ dựa trên context. Nếu không có thông tin, hãy nói thẳng là không biết." | "Bạn là chuyên gia phân tích thông tin. Đọc kỹ context, xác định các facts liên quan, rồi viết câu trả lời rõ ràng, có tổ chức (3-5 câu). Không suy đoán ngoài context." |

## Phân tích kết quả

### 1. Faithfulness (0.9355 vs 0.9341)
- **Kết quả**: Gần như ngang nhau, chênh lệch chỉ ~0.0014
- **Nhận xét**: Cả 2 phiên bản đều ≥0.9, cho thấy LLM tuân thủ tốt context
- **Giả thuyết** (chưa kiểm chứng): V1 ngắn gọn hơn nên ít có sai lệch logic, nhưng chênh lệch quá nhỏ để kết luận

### 2. Answer Relevancy (0.9083 vs 0.8920)
- **Kết quả**: V1 cao hơn V2 với chênh lệch 0.0163
- **Giả thuyết** (chưa kiểm chứng): Prompt V1 tập trung vào "trả lời ngắn gọn", giúp LLM tập trung trực tiếp vào câu hỏi; V2 có hướng dẫn "phân tích" có thể làm câu trả lời dài hơn, kém focused

### 3. Context Precision (0.9450 vs 0.9517)
- **Kết quả**: V2 cao hơn V1 với chênh lệch 0.0067
- **Giả thuyết** (chưa kiểm chứng): Prompt V2 yêu cầu "xác định các facts liên quan" có thể giúp LLM lọc context tốt hơn

### 4. Context Recall (1.0 vs 1.0)
- **Kết quả**: Hoàn toàn giống nhau
- **Lý do**: Cả 2 phiên bản dùng cùng retriever (k=3, chunk_size=500) nên không ảnh hưởng

## Kết luận
- **Giá trị thống kê**: n=50 mỗi phiên bản, chênh lệch nhỏ (<0.02) chưa chắc có ý nghĩa thống kê
- **Xu hướng**: V1 tốt hơn về relevancy; V2 tốt hơn về precision; faithfulness gần bằng
- **Khuyến nghị**: Cần thêm dữ liệu (n>500) hoặc user study để kết luận chắc chắn hơn

## Danh sách file evidence

- **01_langsmith_traces.png**: Dashboard LangSmith hiển thị 50+ traces từ RAG pipeline (Checkpoint 1)
- **01_langsmith_traces_2.png**: Dashboard LangSmith chi tiết trace (Checkpoint 1)
- **02_ab_routing_log.txt**: Log output khi chạy A/B routing (Checkpoint 2)
- **02_prompt_hub.png**: Screenshot Prompt Hub trên LangSmith hiển thị cả 2 phiên bản (Checkpoint 2)
- **03_ragas_scores.png**: Bảng kết quả RAGAS với V1 vs V2 (Checkpoint 3)
- **03_ragas_report.json**: JSON report với điểm chi tiết: V1 và V2 (Checkpoint 3)
- **04_pii_demo_log.txt**: Log demo PII detector với các test case (Checkpoint 4)
- **04_json_demo_log.txt**: Log demo JSON formatter validator (Checkpoint 4)
