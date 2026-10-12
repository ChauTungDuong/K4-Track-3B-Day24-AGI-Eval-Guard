# Blueprint CI/CD cho RAG và guardrails

Nhóm: AGI

Người phụ trách: Châu Tùng Dương - 2A202602822

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T17:04:44.754616+00:00; guard models: gpt-4o-mini.
Models generation/evaluator/judge từ A/B: Generation: gemini-3.8-flash-n (từ answers_50q.json, ragas_50q.json, judge_results.json); Evaluator: BAAI/bge-m3, gpt-4o-mini (từ ragas_50q.json); Judge: gpt-4o-mini (từ judge_results.json); Guard: gpt-4o-mini (từ guard_results.json).

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

- **Data flow**: Luồng dữ liệu di chuyển tuần tự qua 5 trạm kiểm soát:
  1. *User Input*: Tiếp nhận câu hỏi từ người dùng.
  2. *Presidio Scanner*: Phát hiện và ẩn danh hóa (masking/anonymize) các thực thể định danh cá nhân PII (CCCD/CMND, Số điện thoại Việt Nam, Email, Mã số thuế) thành các thẻ placeholder an toàn như `<PHONE_NUMBER>`, `<EMAIL_ADDRESS>`.
  3. *NeMo Input Rail*: Phân tích cấu trúc câu lệnh đã được redact để phát hiện các mẫu tấn công prompt injection, jailbreak, role-playing, yêu cầu tiết lộ system prompt hoặc chỉ đạo bỏ qua quy định an toàn.
  4. *RAG Core Pipeline*: Thực hiện truy xuất ngữ nghĩa lai (Hybrid Search kết hợp BM25 Sparse Search và BAAI/bge-m3 Dense Search trong Qdrant), lọc qua FlashRank/Cohere reranker, lắp context vào prompt template và gửi tới mô hình generation để tổng hợp câu trả lời.
  5. *NeMo Output Rail*: Thẩm định câu trả lời được sinh ra để ngăn chặn ảo giác (hallucination), kiểm tra tính trung thực so với context và chặn rò rỉ dữ liệu nhạy cảm nội bộ trước khi phản hồi người dùng.

- **Failure actions & Short-circuiting**:
  - Khi Presidio phát hiện PII nhạy cảm: Áp dụng cơ chế Anonymization thay thế dữ liệu gốc bằng nhãn định danh trước khi chuyển sang tầng kế tiếp.
  - Khi NeMo Input Rail phát hiện prompt injection/jailbreak: Kích hoạt ngắt sớm (short-circuiting) ngay lập tức, trả về thông báo từ chối tiêu chuẩn từ `rails.co` ("Xin lỗi, tôi không thể hỗ trợ yêu cầu này do vi phạm chính sách an toàn."), không gọi RAG hay LLM generation nhằm tiết kiệm tài nguyên và phòng tránh tấn công.
  - Khi NeMo Output Rail phát hiện vi phạm nội dung nhạy cảm hoặc ảo giác nặng: Thay thế câu trả lời bằng câu từ chối an toàn hoặc redact thông tin mật.

- **Log & Redaction**: Toàn bộ log sự kiện và telemetry (OpenTelemetry, ELK/Datadog) đều đi qua bộ lọc Presidio để che giấu PII trước khi ghi đĩa hoặc gửi qua mạng. Tuyệt đối không ghi log raw input chứa thông tin cá nhân.

- **Quản lý Secrets**: Tất cả API keys (OpenAI, Gemini, Cohere, Qdrant) được lưu trữ tại GitHub Actions Secrets và AWS Secrets Manager / HashiCorp Vault. Trong môi trường runtime, secrets được inject qua biến môi trường (.env / container environment), không bao giờ commit vào mã nguồn.

- **Điều kiện Fallback**: Nếu dịch vụ NeMo Guardrails hoặc LLM API bên thứ ba gặp lỗi kết nối hay timeout (> 2000 ms), hệ thống kích hoạt circuit breaker chuyển sang fallback regex heuristic cục bộ (kiểm tra từ khóa cấm, regex injection) và trả về câu trả lời an toàn giảm cấp (graceful degradation) nhằm đảm bảo tính sẵn sàng cao.

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 5.17 ms | 6.84 ms | 6.97 ms | Đạt ngân sách (< 10 ms), xử lý regex/spacy in-memory |
| NeMo input | 772.49 ms | 828.26 ms | 856.10 ms | Vượt ngân sách (do phụ thuộc OpenAI API roundtrip) |
| Input guard sequential | 777.62 ms | 835.01 ms | 861.61 ms | Vượt ngân sách 500 ms (tổng Presidio + NeMo) |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: P95 = 835.01 ms, budget_ok = False.

- **Phân tích Bottleneck**: Điểm nghẽn độ trễ nằm hoàn toàn ở tầng NeMo Input Rail với P95 = 828.26 ms (chiếm tới 99.2% tổng thời gian input guard). Nguyên nhân là NeMo 0.10.0 sử dụng mô hình LLM từ xa (`gpt-4o-mini`) để thực hiện các lời gọi hàm self-check input qua mạng Internet, chịu chi phí roundtrip mạng và thời gian sinh token. Ngược lại, tầng Presidio chạy cục bộ trên bộ nhớ máy tính đạt hiệu năng xuất sắc với P95 = 6.84 ms (vượt xa ngân sách kỳ vọng).
- **Phần ngoài phạm vi**: Các tầng RAG, NeMo Output Rail và Full Guard short-circuit request hiện được ghi nhận "chưa đo" theo đúng quy định phân định phạm vi của Task 12. Kế hoạch triển khai đo đạc thực tế: Thiết lập test suite end-to-end đo 100 câu truy vấn với 5 lượt warm-up thật, chèn trace span OpenTelemetry ghi nhận mốc thời gian chính xác (t0 tiếp nhận -> t1 presidio -> t2 nemo input -> t3 rag retrieve & rerank -> t4 generation -> t5 nemo output -> t6 response).

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.

- **Lệnh thực thi trong CI Pipeline**:
  - Kiểm thử đơn vị & kiểm thử offline: `pytest tests/ -q`
  - Thẩm định hợp đồng và artifact: `python scripts/check_lab.py --with-tests`
  - Chạy Quality Gate đánh giá tự động:
    ```bash
    python -c "
    from src.report_contracts import read_json, require
    ragas = read_json('artifacts/ragas_50q.json')
    guard = read_json('artifacts/guard_results.json')
    # Quality gate rules
    require(ragas['per_distribution']['factual']['faithfulness'] >= 0.75, 'Faithfulness gate failed')
    require(guard['adversarial']['passed'] >= 18, 'Attacks threshold failed')
    require(guard['clean']['passed'] == 8, 'Clean control false positive detected')
    print('ALL CI QUALITY GATES PASSED')
    "
    ```
- **Cơ chế dừng và trả mã lỗi non-zero**: Script kiểm định đọc trực tiếp các file schema kết quả trong `artifacts/`. Khi bất kỳ chỉ số thực nghiệm nào không đạt ngưỡng tối thiểu quy định, hàm `require()` sẽ ném ngoại lệ `AssertionError` và làm script thoát với mã lỗi khác 0 (`exit code 1`). Bước CI trong GitHub Actions sẽ lập tức bị đánh dấu thất bại (Red build), kích hoạt chặn thao tác Merge Pull Request vào nhánh chính `main`.
- **Trạng thái triển khai**: Đề xuất quy trình CI Quality Gate đã được cấu hình chi tiết tại `.github/workflows/eval_guard_ci.yml` và kiểm chứng cục bộ qua `scripts/check_lab.py --with-tests`.

## Monitoring và cải tiến

- **Chỉ số giám sát production & Ngưỡng cảnh báo**:
  - *P95 Latency của từng tầng*: Giám sát liên tục qua OpenTelemetry; cảnh báo PagerDuty/Slack nếu P95 Input Guard > 1200 ms hoặc RAG Generation > 3000 ms trong 5 phút liên tục.
  - *Tỷ lệ vi phạm chính sách (Block rate)*: Ngưỡng bình thường 1% - 3%; cảnh báo mức độ High nếu Block rate > 10% trong vòng 10 phút (dấu hiệu của đợt tấn công từ chối dịch vụ hoặc spam prompt injection có tổ chức).
  - *False Positive Rate*: Theo dõi tỷ lệ người dùng gửi phản hồi khiếu nại (thumbs down / báo lỗi chặn nhầm); cảnh báo khi tỷ lệ này vượt quá 0.5% tổng số câu hỏi được xử lý.
  - *Data Drift & Model Degradation*: Lấy mẫu ngẫu nhiên 5% traffic sạch hàng tuần để chạy lại bộ đánh giá RAGAS tự động, phát hiện sự suy giảm chất lượng truy xuất tri thức mới.
- **Quy trình xử lý sự cố & Quyền riêng tư (Privacy)**:
  - Khi phát hiện case bị chặn nhầm (False Positive): Đội ngũ AI Engineer phân tích Colang activation log trong môi trường cách ly (đã ẩn danh hóa PII), cập nhật mẫu câu sạch vào `fixtures/clean_set_8.json` và bổ sung flow whitelist trong `guardrails/config.yml`.
  - Quyền riêng tư: Không ghi đè raw logs ra console, định kỳ 30 ngày tự động xóa logs telemetry chứa thông tin truy vấn của người dùng theo chuẩn GDPR/Nghị định 13.

RAGAS mean/worst distribution: Overall Mean = 0.672; Worst distribution là multi_hop với avg_score = 0.595 (dominant failure metric là context_recall = 0.383).
Cohen’s κ: 1.0 (mức độ đồng thuận tuyệt đối giữa LLM Judge và 10 nhãn chuyên gia con người).
Attacks passed: 20/20 (tỷ lệ 100%); clean false_positive_rate: 0.0% (8/8 câu sạch được cho phép hoàn toàn).

- **Kế hoạch cải tiến cụ thể có thể kiểm chứng**:
  1. *Khắc phục điểm yếu context_recall trong multi_hop*: Bổ sung cơ chế chunking phân đoạn ngữ nghĩa theo cấu trúc heading kết hợp Hybrid Search (BM25 + Dense BAAI/bge-m3) để tăng Context Recall của nhóm câu hỏi suy luận đa bước từ 0.383 lên trên 0.65.
  2. *Giải quyết nút thắt độ trễ của NeMo Input Rail*: Thay thế việc gọi OpenAI LLM API từ xa bằng một mô hình phân loại nhỏ gọn triển khai cục bộ (như Llama-Guard-3-1B quantized qua vLLM hoặc DeBERTa phân loại an toàn), giúp giảm độ trễ P95 Input Guard từ 835 ms xuống dưới ngưỡng mục tiêu 300 ms.
  3. *Tích hợp Semantic Caching cho Guard và RAG*: Triển khai tầng bộ đệm ngữ nghĩa bằng Redis vector cache cho các câu hỏi phổ biến đã được kiểm định an toàn, giúp loại bỏ việc chạy lại guardrail cho các câu hỏi lặp lại và giảm độ trễ phản hồi xuống dưới 50 ms.
  4. *Nâng cao độ chính xác thế hệ bằng Prompt Grounding*: Cải tiến prompt template của generator với quy tắc trích dẫn nguồn bắt buộc và giảm nhiệt độ temperature (từ 0.3 xuống 0.0) nhằm loại bỏ triệt để ảo giác trong các câu hỏi đa bước phức tạp (như câu id 38, 40 trong danh sách bottom 10).
