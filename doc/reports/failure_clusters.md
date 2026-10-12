# Phân tích lỗi Phase A

Nhóm: K4-EvalGuard-Team

Người phụ trách: Nguyễn Đình Tuấn Anh - 2A202602735

## Trạng thái thực nghiệm

Phần code Tasks 1–4 đã hoàn thành và vượt qua 6 test offline. Bộ 50 câu được
chia đúng thành factual 20, multi-hop 20 và adversarial 10. Adapter gửi nguyên
question, answer, contexts và ground truth sang `m4.evaluate_ragas`, đồng thời
giữ đúng model evaluator do m4 trả về.

Workspace hiện chưa có `artifacts/answers_50q.json`, `.env` hoặc RAGAS runtime,
nên chưa thể chạy batch live để tạo `artifacts/ragas_50q.json`. Báo cáo không
điền điểm giả. Sau khi trưởng nhóm tạo answers thật, chạy:

```bash
python scripts/run_experiments.py --phase a --live
```

## Cách phân tích lỗi

Mỗi câu được xếp theo trung bình của bốn metric. Bottom 10 được sắp tăng dần và
giữ đủ id, distribution, metric yếu nhất, chẩn đoán và hướng sửa. Matrix chỉ đếm
metric thấp nhất của từng câu; distribution yếu được chọn bằng mean thấp nhất,
không dựa vào nhóm nào có nhiều câu hơn.

Nếu `context_recall` thấp, cần kiểm tra chunking và bổ sung BM25; nếu
`context_precision` thấp, cần rerank hoặc lọc metadata. `faithfulness` thấp nên
siết prompt sinh câu trả lời, còn `answer_relevancy` thấp cần chỉnh prompt theo
đúng ý hỏi.

## Giới hạn cần kiểm tra sau lượt live

Với nhóm adversarial, cần đọc trực tiếp answer, contexts và reference của từng
câu Bottom 10 để phát hiện xung đột phiên bản hoặc câu hỏi phủ định. Điểm thấp
không đủ để kết luận pipeline đã nhận ra bẫy. Khi artifact live có sẵn, báo cáo
này cần bổ sung bảng điểm, đúng 10 dòng yếu nhất và bằng chứng từ các context
đại diện trước khi nộp.
