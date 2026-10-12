# Phân tích lỗi Phase A

Nhóm: K4-EvalGuard-Team

Người phụ trách: Nguyễn Đình Tuấn Anh - 2A202602735

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T12:46:46.354762+00:00.
Generation mode/fallback và điều kiện chạy: Chạy thực tế qua pipeline HybridSearch kết hợp DenseSearch (BGE-M3) và BM25, reranker MiniLM-L6, sinh câu trả lời bằng LLM gemini-3.8-flash-n qua 9router gateway (100% LLM mode, 0% fallback).
Models generation/evaluator: generation: gemini-3.8-flash-n; evaluator: BAAI/bge-m3, gpt-4o-mini.

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 1.0000 | 0.8042 | 0.9000 |
| answer_relevancy | 0.7020 | 0.5316 | 0.7013 |
| context_precision | 0.5958 | 0.6625 | 0.8333 |
| context_recall | 0.5833 | 0.3833 | 0.4750 |
| avg_score | 0.7203 | 0.5954 | 0.7274 |

## Bottom 10 và nguyên nhân

| Rank | ID | Distribution | Câu hỏi | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 38 | multi_hop | Nhân viên đi công tác nước ngoài 4 ngày tại thành phố lớn, ở khách sạn 200 USD/đêm (3 đêm). Công ty thanh toán bao nhiêu tiền khách sạn? | 0.0000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 40 | multi_hop | Nhân viên cấp Manager có thời hạn thử việc 90 ngày, đang trong tháng thứ 3, phát hiện vi phạm bảo mật. Họ nên và không nên làm gì theo chính sách? | 0.0000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 41 | adversarial | Nhân viên được nghỉ bao nhiêu ngày phép năm? | 0.1250 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 1 | factual | Nhân viên được nghỉ bao nhiêu ngày khi kết hôn? | 0.2500 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 5 | 5 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.2500 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 6 | 8 | factual | Nhân viên được nghỉ bao nhiêu ngày khi cha hoặc mẹ mất? | 0.2500 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 7 | 9 | factual | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | 0.2500 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 8 | 34 | multi_hop | Nhân viên tạm ứng 4 triệu và một nhân viên khác tạm ứng 7 triệu: quy trình phê duyệt khác nhau thế nào? | 0.2500 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 9 | 35 | multi_hop | Nhân viên Junior P1 có lương cơ bản 12 triệu vừa vào thử việc: nhận lương bao nhiêu và phụ cấp gì tháng đầu tiên? | 0.2500 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 10 | 27 | multi_hop | Nhân viên có thâm niên 7 năm theo v2024 được nghỉ bao nhiêu ngày phép? Nếu bị trừ thêm 4 ngày ốm không có giấy y tế thì còn lại bao nhiêu? | 0.3333 | answer_relevancy | Answer doesn't match question | Improve prompt template |

### Phân tích chi tiết các ca đại diện bằng chứng

1. **Câu 38 (multi_hop, Rank 1 - faithfulness = 0.00):**
   - Context: Chỉ tìm được một phần quy định công tác phí, thiếu hạn mức trần khách sạn nước ngoài theo bậc nhân viên.
   - Answer: Pipeline trả về "Không tìm thấy." do system prompt ép buộc không suy đoán khi context thiếu hụt.
   - Reference: Cần tính toán trần chi phí 150 USD/đêm x 3 đêm = 450 USD.
   - Nguyên nhân: Lỗi Retrieval recall (bỏ lỡ chunk bảng định mức công tác phí quốc tế), dẫn tới LLM tuân thủ an toàn từ chối trả lời nhưng bị RAGAS phạt faithfulness/recall.

2. **Câu 41 (adversarial, Rank 3 - faithfulness = 0.00):**
   - Context: Retrieval lấy cả chunk chính sách cũ năm 2023 (12 ngày) và chunk chính sách mới năm 2024 (15 ngày cơ bản).
   - Answer: Mô hình trả về "Không tìm thấy." do phát hiện mâu thuẫn giữa hai văn bản.
   - Reference: Theo chính sách hiện hành năm 2024 là 15 ngày làm việc.
   - Nguyên nhân: Xung đột phiên bản (version conflict) không được giải quyết tại bước retrieval ranking. BGE-M3 kéo cả 2 phiên bản vào context mà không có metadata filter thời hiệu.

3. **Câu 1, 5, 8, 9 (factual - answer_relevancy = 0.00):**
   - Context: Do file PDF scan ảnh không có text layer (như BCTC và Nghị định) khiến kho tri thức thiếu các văn bản gốc tương ứng, RAG pipeline trả về "Không tìm thấy.".
   - Reference: Chứa đáp án số ngày nghỉ cưới (3 ngày), thẩm quyền phê duyệt (Tổng Giám đốc/CEO).
   - Nguyên nhân: Câu trả lời ngắn dạng từ chối làm giảm mạnh điểm answer_relevancy do vector embedding của câu trả lời từ chối lệch xa vector của câu hỏi chuyên biệt.

## Matrix và nhóm yếu nhất

| Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| faithfulness | 0 | 4 | 1 | 5 |
| context_recall | 5 | 7 | 7 | 19 |
| context_precision | 6 | 2 | 1 | 9 |
| answer_relevancy | 9 | 7 | 1 | 17 |

Nhóm yếu theo mean thấp nhất: multi_hop; mean: 0.5954.
Worst metric xuất hiện nhiều nhất: context_recall; số câu: 19.

### Giải thích nhóm yếu nhất và đề xuất khắc phục

Nhóm `multi_hop` có mean thấp nhất (0.5954) do đặc thù yêu cầu kết nối thông tin từ 2 đến 3 tài liệu rời rạc (ví dụ: quy định cấp bậc lương kết hợp chính sách phụ cấp và điều kiện thử việc). Khi một trong các chunk quan trọng bị rớt khỏi top-3 reranker, context_recall tụt sâu xuống mức 0.3833.

Đề xuất khắc phục:
1. **Mở rộng retrieval recall:** Nâng `top_k` của Dense/BM25 từ 20 lên 40 và `RERANK_TOP_K` từ 3 lên 5 để bao quát đủ chuỗi bằng chứng multi-hop.
2. **Metadata filtering theo phiên bản:** Thêm bộ lọc metadata `doc_version=latest` hoặc `valid_year=2024` trước khi rerank để loại bỏ tài liệu hết hiệu lực.
3. **Cách chạy lại kiểm chứng:** Chạy lại `python scripts/setup_answers.py --live` rồi `python scripts/run_experiments.py --phase a --live` và so sánh avg_score của nhóm multi_hop xem có vượt ngưỡng 0.70 hay không.

## Bẫy adversarial và giới hạn

Tập câu hỏi `adversarial` đạt avg_score 0.7274, tuy nhiên điểm số này không chứng minh rằng pipeline đã hoàn toàn miễn nhiễm với bẫy. Trong 10 câu adversarial, các bẫy chia làm hai dạng chính:
- **Xung đột phiên bản (Version Conflict):** Câu hỏi về phép năm cố tình kích hoạt cả tài liệu cũ v2023 và mới v2024 (câu 41).
- **Phủ định và tiền giả định sai:** Câu hỏi hỏi về quyền lợi không tồn tại hoặc giả định sai chính sách bảo mật (câu 50 về VPN cá nhân).

Giới hạn thực nghiệm:
- Bộ câu hỏi adversarial gồm 10 mẫu tập trung vào văn bản nhân sự, chưa phủ hết các vector tấn công prompt injection tinh vi.
- Quá trình OCR PDF scan bị bỏ qua làm giảm độ bao phủ tài liệu của kho tri thức thực tế.
- Đánh giá sử dụng mô hình gemini-3.8-flash-n qua 9router gateway và evaluator BAAI/bge-m3 kết hợp gpt-4o-mini.
