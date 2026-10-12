# Đánh giá LLM Judge Phase B

Nhóm: K4-EvalGuard-Team

Người phụ trách: Nguyễn Gia Khánh - 2A202602851

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T12:49:05.175237+00:00.
Models generation/judge: generation: gemini-3.8-flash-n; judge: gpt-4o-mini.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ artifacts/answers_50q.json:answer; B từ fixtures/human_labels_10q.json:model_answer; reference từ fixtures/test_set_50q.json:ground_truth.
Ids: 1, 5, 12, 21, 41. Không đổi nguồn sau khi xem kết quả judge.

| ID | Candidate A | Candidate B | Winner Pass 1 | Winner Pass 2 | Final Winner | Position Consistent | Scores Pass 1 (A/B) | Scores Pass 2 (A/B) | Reasoning Tóm tắt |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Không tìm thấy. | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | B | B | B | true | 0.0 / 0.95 | 0.0 / 0.95 | B trả lời chính xác và súc tích theo reference; A không tìm thấy thông tin dù dữ liệu có sẵn. |
| 5 | Không tìm thấy. | Cần Giám đốc phòng ban phê duyệt. | A | A | A | true | 0.3 / 0.0 | 0.2 / 0.0 | B sai lệch nghiêm trọng (quy định trên 50 triệu cần Tổng Giám đốc); A an toàn hơn vì không bịa đặt hallucination. |
| 12 | Thưởng Tết tối thiểu cho nhân viên chính thức có thời gian làm việc từ 6 tháng trở lên là **1 tháng lương**. | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | A | A | A | true | 1.0 / 0.6 | 1.0 / 0.7 | A nêu đầy đủ ngữ cảnh điều kiện áp dụng (từ 6 tháng trở lên); B khái quát hóa quá mức thiếu điều kiện. |
| 21 | Theo context: Nhân viên có 9 năm thâm niên được 18 ngày phép; thông tin Senior và lương không tìm thấy. | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | B | B | B | true | 0.5 / 1.0 | 0.5 / 1.0 | B trả lời đầy đủ cả hai vế (ngày phép và dải lương Senior); A bị khuyết vế dải lương. |
| 41 | Không tìm thấy. | Nhân viên được nghỉ 12 ngày phép năm. | A | A | A | true | 0.2 / 0.0 | 0.3 / 0.1 | B dùng chính sách cũ v2023 đã hết hiệu lực; A an toàn hơn khi từ chối thay vì cung cấp thông tin sai lệch. |

Position bias count/rate: 0 và 0.0 (tỷ lệ 0%).
Verbosity win rate: 0.4 (40%); decisive cases: 5.

### Phân tích hiện tượng Bias

- **Position Bias:** Đạt 0% vị trí (cả 5/5 cặp đều giữ nguyên quyết định thắng cuộc giữa 2 lượt hoán đổi trật tự prompt A-B và B-A). Điều này chứng minh prompt pairwise kết hợp kỹ thuật hoán đổi vị trí (swap-and-average) triệt tiêu hoàn toàn thiên vị thứ tự xuất hiện của candidate.
- **Verbosity Bias:** Tỷ lệ thắng của câu dài hơn là 40% (2/5 cases: case 12 ứng viên A dài hơn thắng, case 21 ứng viên B dài hơn thắng). Trong 3 cases còn lại (case 1, 5, 41), câu trả lời ngắn hơn hoặc câu từ chối đã giành chiến thắng hoặc câu dài bị phạt vì ảo giác. Do đó, việc câu dài chiến thắng ở case 12 và 21 xuất phát thuần túy từ tính đầy đủ và tính chính xác của dữ kiện ngữ cảnh chứ không phải do LLM thiên vị độ dài đơn thuần.

## Calibration đúng model_answer

Helper đánh giá đúng question/model_answer đang có trong fixtures/human_labels_10q.json; không lấy answer mới của pipeline. Prompt chỉ nhận question, answer, reference. Không gửi human_label, human_note hoặc phân bố nhãn human vào prompt.

| Question ID | Model Answer Gốc | Judge Label | Human Label | Agree | Reasoning của Judge |
|---|---|---|---|---|---|
| 1 | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | 1 | true | Câu trả lời hoàn toàn chính xác và trả lời đầy đủ ý hỏi của câu hỏi dựa trên thông tin trong tài liệu tham khảo. |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | 0 | true | Câu trả lời sai so với tài liệu tham chiếu. Thiết bị trị giá 55 triệu (trên 50.000.000 VNĐ) cần Tổng Giám đốc (CEO) phê duyệt, không phải Giám đốc phòng ban. |
| 12 | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | 1 | true | Câu trả lời nêu chính xác mức thưởng Tết tối thiểu là 1 tháng lương theo đúng thông tin trong tài liệu tham khảo. |
| 21 | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | 1 | true | Câu trả lời hoàn toàn chính xác và đầy đủ các thông tin theo tham chiếu (18 ngày phép và mức lương 20 - 35 triệu). |
| 23 | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | 1 | true | Câu trả lời chính xác số tiền cần hoàn trả (25 triệu) và lý do (chưa đủ 1 năm cam kết) theo đúng thông tin trong tài liệu tham khảo. |
| 29 | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | 0 | true | Câu trả lời thiếu người phê duyệt là Kế toán trưởng (cần cả Trưởng phòng và Kế toán trưởng) và chưa nêu cụ thể số tiền phạt theo quy định trong tài liệu tham khảo. |
| 33 | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | 1 | true | Câu trả lời cung cấp chính xác và đầy đủ cả hai ý về tổng phụ cấp hàng tháng (1.500.000 VNĐ/tháng) và số ngày phép năm (19 ngày) theo đúng thông tin từ reference. |
| 41 | Nhân viên được nghỉ 12 ngày phép năm. | 0 | 0 | true | Câu trả lời sử dụng số liệu từ chính sách cũ (v2023) đã bị thay thế thay vì chính sách hiện hành (v2024) là 15 ngày. |
| 46 | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | 1 | true | Câu trả lời trả lời chính xác ý cốt lõi của câu hỏi dựa trên reference: nhân viên thử việc không được nghỉ phép năm và có thể xin nghỉ không lương nếu cần. |
| 50 | Được, miễn là đảm bảo kết nối an toàn. | 0 | 0 | true | Câu trả lời sai hoàn toàn so với tài liệu tham khảo (cho phép trong khi chính sách cấm sử dụng VPN cá nhân như NordVPN). |

Cohen’s κ: 1.0 (1.0000).
Observed/chance agreement và diễn giải:
- Observed agreement (Po): 10/10 = 1.0 (100% các ca thẩm định đều trùng khớp nhãn).
- Phân bố nhãn thực tế: 6 nhãn 1 và 4 nhãn 0 cho cả judge và human.
- Chance agreement (Pe): (6/10 x 6/10) + (4/10 x 4/10) = 0.36 + 0.16 = 0.52.
- Cohen’s κ = (Po - Pe) / (1 - Pe) = (1.0 - 0.52) / (1.0 - 0.52) = 1.0.
- Diễn giải: Mức độ tương đồng hoàn hảo (Perfect Agreement). Judge đã phân biệt chính xác cả các lỗi tinh vi như thiếu người đồng phê duyệt (câu 29) và thông tin hết hiệu lực (câu 41).

## Giới hạn và cách dùng judge

1. **Giới hạn số mẫu:** Thử nghiệm pairwise chỉ thực hiện trên 5 cặp câu hỏi và calibration trên 10 câu nhãn chuyên gia. Mặc dù κ đạt 1.0, kết quả trên tập mẫu nhỏ này không đảm bảo judge luôn đồng thuận 100% trên toàn bộ các miền nghiệp vụ khác.
2. **Khung nguyên tắc ứng dụng:**
   - Không coi quyết định của LLM Judge là chân lý tuyệt đối mà phải coi là bộ lọc sơ bộ (first-pass filter).
   - Luôn áp dụng kỹ thuật hoán đổi vị trí (swap-and-average) để triệt tiêu position bias.
   - Cần định kỳ lấy mẫu ngẫu nhiên (spot-check) để con người phúc tra các ca nhạy cảm (biên điểm chênh lệch thấp hoặc ca có rủi ro pháp lý cao).
