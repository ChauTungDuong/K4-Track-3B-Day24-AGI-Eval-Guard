# Nhóm thực hiện K4 Track 3B

Tên nhóm: AGI

Repo chung: https://github.com/ChauTungDuong/K4-Track-3B-Day24-Eval-Guard

Trưởng nhóm/người nộp: Châu Tùng Dương - 2A202602822

Ngày lab/deadline áp dụng: 23:59 ngày 12/10/2026 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: HEAD (main branch)

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; mẫu ba rows không ép nhóm phải có ba người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Châu Tùng Dương | 2A202602822 | Trưởng nhóm, Tích hợp chung & Setup pipeline & Task 13 CI/CD Blueprint | Commit integration, doc/reports/blueprint.md |
| 2 | Nguyễn Đình Tuấn Anh | 2A202602735 | Phase A — Đánh giá RAGAS 50 câu & Phân tích Bottom 10 | src/phase_a_ragas.py, doc/reports/failure_clusters.md |
| 3 | Nguyễn Gia Khánh | 2A202602851 | Phase B — Đánh giá LLM-as-a-Judge, Swap A/B, Cohen Kappa & Bias | src/phase_b_judge.py, doc/reports/bias_report.md |
| 4 | Trần Thị Thu Hiền | 2A202602737 | Phase C — Xây dựng Guardrails Presidio PII + NeMo Rails & Latency | src/phase_c_guard.py, tests/test_phase_c.py |

Người tích hợp/chạy batch/nộp link: Châu Tùng Dương - 2A202602822, quản lý repo, điều phối merge, tích hợp chạy pipeline thực nghiệm và nộp bài.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11 64-bit, Python 3.11.6, MSVC v14.0+ (Microsoft C++ Build Tools).
Virtualenv/dependencies/imports: py -3.11 -m venv .venv, cài đặt dependencies và lockfile; 86 unit tests offline pass hoàn toàn.
Models/Qdrant/50 answers đã chuẩn bị: Model embedding BAAI/bge-m3, LLM gpt-4o-mini, Qdrant v1.9.7 container sẵn sàng phục vụ retrieval và indexing 50 câu test set.
Nếu cài full NeMo chưa thành công, ghi lỗi thực tế; không dùng unit test pass hoặc pip resolve để khẳng định runtime đã chạy.

## INDIVIDUAL

Sao chép mẫu cho mỗi thành viên; tiêu đề họ tên không dấu-MSSV.

### ChauTungDuong-2A202602822

- Phần được giao và phần thực tế đã làm: Trưởng nhóm, thiết lập môi trường, tích hợp mã nguồn m1-m5 từ Day 18, xây dựng kiến trúc CI/CD Blueprint (Task 13), điều phối kiểm thử toàn bộ hệ thống.
- Bằng chứng: Lịch sử commit git, file cấu hình config.py, doc/reports/blueprint.md, kết quả pass 86 unit tests offline.
- Cách kiểm tra: Chạy lệnh pytest tests/ -q và python scripts/check_lab.py --with-tests để xác thực toàn bộ hợp đồng.
- Điều đã học: Hiểu rõ kiến trúc guardrail nhiều tầng (Presidio -> NeMo input -> RAG -> NeMo output), cách thiết lập quality gates CI/CD cho hệ thống RAG thực tế.
- Giới hạn/cách xử lý: NeMo Guardrails phụ thuộc API ngoài có thể gây latency; xử lý bằng cách phân tách tầng đo latency input riêng biệt và tối ưu hóa warm-up calls.
- Nguồn/AI hỗ trợ nếu có: Sử dụng Antigravity IDE để phân tích giao diện pipeline và hỗ trợ refactor code adapter.

### NguyenDinhTuanAnh-2A202602735

- Phần được giao và phần thực tế đã làm: Phụ trách Phase A: implement Tasks 1, 2, 3, 4 trong src/phase_a_ragas.py (group_by_distribution, run_ragas_50q, bottom_10, cluster_analysis), phân tích failure clusters.
- Bằng chứng: Mã nguồn src/phase_a_ragas.py, test cases tests/test_phase_a.py, báo cáo doc/reports/failure_clusters.md.
- Cách kiểm tra: Chạy lệnh pytest tests/test_phase_a.py -q đạt 6/6 test cases pass.
- Điều đã học: Nắm vững 4 metrics RAGAS (faithfulness, answer_relevancy, context_precision, context_recall), cách chẩn đoán lỗi retrieval vs generation dựa trên Diagnostic Tree.
- Giới hạn/cách xử lý: Distribution adversarial dễ bị nhầm lẫn giữa version cũ và mới; xử lý bằng cách cải tiến prompt và lọc metadata theo phiên bản tài liệu.
- Nguồn/AI hỗ trợ nếu có: Tham khảo tài liệu RAGAS documentation và starter codelab.

### NguyenGiaKhanh-2A202602851

- Phần được giao và phần thực tế đã làm: Phụ trách Phase B: implement Tasks 5, 6, 7, 8 trong src/phase_b_judge.py (pairwise_judge, swap_and_average, cohen_kappa, bias_report), đánh giá độ tin cậy của LLM Judge.
- Bằng chứng: Mã nguồn src/phase_b_judge.py, test cases tests/test_phase_b.py, báo cáo doc/reports/bias_report.md.
- Cách kiểm tra: Chạy lệnh pytest tests/test_phase_b.py -q đạt 8/8 test cases pass.
- Điều đã học: Hiểu rõ position bias và verbosity bias trong LLM-as-a-Judge; kỹ thuật Swap-and-Average để loại bỏ position bias; công thức Cohen's kappa để đo lường mức độ đồng thuận với nhãn người.
- Giới hạn/cách xử lý: Kích thước mẫu 10 câu nhãn người tương đối nhỏ nên Cohen's kappa nhạy cảm với từng case bất đồng; cần phân tích định tính chi tiết từng reasoning.
- Nguồn/AI hỗ trợ nếu có: Tham khảo lý thuyết Landis-Koch về Cohen's kappa và tài liệu hướng dẫn lab.

### TranThiThuHien-2A202602737

- Phần được giao và phần thực tế đã làm: Phụ trách Phase C: implement Tasks 9a, 9b, 10, 11, 12 trong src/phase_c_guard.py (pii_scan, check_input_rail, check_output_rail, run_adversarial_suite, measure_p95_latency).
- Bằng chứng: Mã nguồn src/phase_c_guard.py, test cases tests/test_phase_c.py.
- Cách kiểm tra: Chạy lệnh pytest tests/test_phase_c.py -q đạt 12/12 test cases pass.
- Điều đã học: Cách xây dựng regex recognizer cho PII Việt Nam (CCCD, SĐT) trong Microsoft Presidio; cấu hình Colang 1.0 và self-check flows trong NeMo Guardrails; đo kiểm P95 latency với warm-up chuẩn.
- Giới hạn/cách xử lý: Tránh false positive với các câu hỏi hợp lệ có chứa từ ngữ xin lỗi/từ chối; kiểm tra kỹ thuật stop rails thay vì lọc từ khóa thô sơ.
- Nguồn/AI hỗ trợ nếu có: Tài liệu hướng dẫn Microsoft Presidio và NeMo Guardrails documentation.

## Cập nhật sau khi nộp

Bản nộp chính thức đúng hạn, không có cập nhật phát sinh sau deadline.
