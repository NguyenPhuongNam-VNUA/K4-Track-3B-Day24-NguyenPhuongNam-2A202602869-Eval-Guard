# Blueprint CI/CD cho RAG và guardrails

Nhóm: Nguyễn Phương Nam

Người phụ trách: Nguyễn Phương Nam - MSSV: 2A202602869

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T16:58:46.462022+00:00; guard models: gpt-4o-mini.
Models generation/evaluator/judge từ A/B: Generation: openai/gpt-4o-mini (artifacts/answers_50q.json); Evaluator: gpt-4o-mini, text-embedding-ada-002 (artifacts/ragas_50q.json); Judge: gpt-4o-mini (artifacts/judge_results.json).

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Data flow và cơ chế bảo vệ qua 4 tầng:
1. **Presidio PII Layer**: Quét pattern/regex cho các thực thể nhạy cảm gồm CCCD 9-12 chữ số, số điện thoại Việt Nam và địa chỉ email. Nếu phát hiện rò rỉ dữ liệu cá nhân vi phạm chính sách, chặn ngay lập tức hoặc thực hiện redact định danh.
2. **NeMo Input Rail**: Kiểm tra câu hỏi người dùng qua flow self_check_input (gpt-4o-mini). Chặn ngay lập tức (short-circuit) các hành vi tấn công jailbreak/DAN, off-topic, authority impersonation, prompt injection, và yêu cầu đánh cắp mật khẩu hệ thống. Khi bị chặn, NeMo ngắt pipeline và trả về câu từ chối chuẩn mực từ rails.co, ngăn chặn triệt để prompt độc hại tiếp cận RAG.
3. **RAG Pipeline**: Truy vấn tìm kiếm ngữ nghĩa trên Qdrant vector database (Day 18 m2_search) kết hợp bộ xếp hạng lại (m3_rerank FlashRank) để trích xuất top-3 context chunks phù hợp nhất, đưa vào LLM generator kèm system prompt chặt chẽ.
4. **NeMo Output Rail**: Rà soát câu trả lời của LLM qua flow self_check_output. Chặn và thay thế toàn bộ phản hồi nếu chứa dữ liệu CCCD bí mật hoặc mật khẩu admin giả lập trước khi phản hồi tới người dùng.

- **Failure action**: Áp dụng nguyên tắc fast-fail (short-circuit). Bất kỳ chốt chặn nào phát hiện vi phạm đều lập tức dừng request và trả về thông điệp an toàn, ghi log cảnh báo.
- **Log và Redact**: Tất cả request log và audit trail tuyệt đối không lưu raw input chứa PII. Dữ liệu CCCD/CMND, số điện thoại và mật khẩu phải được redact thành `<VN_CCCD>`, `<VN_PHONE>`, `<REDACTED>` trước khi ghi vào log database.
- **Quản lý Secrets**: Toàn bộ API keys (OPENAI_API_KEY, OpenRouter key, Qdrant credentials) được lưu trữ an toàn trong Secret Manager (AWS Secrets Manager / Vault), đưa vào container runtime qua biến môi trường (.env), tuyệt đối không commit vào repository.
- **Điều kiện Fallback**: Nếu LLM generator gặp lỗi quá tải (HTTP 429/5xx) hoặc timeout > 5 giây, hệ thống kích hoạt chế độ fallback trích xuất ngữ cảnh trực tiếp (extractive_fallback) từ search chunks kèm thông báo giới hạn. Đối với NeMo rails, nếu service gián đoạn, áp dụng chính sách fail-closed đối với các truy vấn nhạy cảm để đảm bảo an toàn tuyệt đối.

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 11.33 | 13.14 | 13.25 | Đạt ngân sách cục bộ (<50ms), regex CPU siêu nhanh |
| NeMo input | 1385.90 | 1898.63 | 1910.57 | Vượt target 500ms do độ trễ round-trip cloud LLM API |
| Input guard sequential | 1395.86 | 1910.94 | 1922.25 | Vượt budget 500ms (latency_budget_ok: false) |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc Task 12 |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc Task 12 |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms, ngoài phạm vi Task 12 |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: P95 = 1910.94 ms; budget_ok = False.
Bottleneck xác định nằm ở NeMo input rail: Việc thực hiện inference qua Cloud API (gpt-4o-mini qua mạng internet) chịu chi phí kết nối RTT (Round Trip Time) từ 800ms đến 1900ms mỗi lượt. Trong khi đó, Presidio chạy offline trên CPU chỉ tốn 11.33ms (chưa tới 1% tổng thời gian input guard).
Kế hoạch đo lường cho các thành phần ngoài phạm vi:
- Thu thập 50 mẫu request production thực tế với 3 lượt warm-up không tính vào kết quả.
- Đo RAG generation latency độc lập (gồm Qdrant vector retrieval + FlashRank rerank + LLM generation).
- Đo NeMo output rail latency độc lập cho các câu trả lời do LLM sinh ra.
- Đo End-to-End latency của toàn bộ request thực tế khi có short-circuit (chặn tại Presidio hoặc NeMo input).
- Sử dụng timer độ phân giải cao `time.perf_counter()` và phương pháp nội suy tuyến tính `linear`.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.
Đề xuất lệnh job trong CI pipeline (.github/workflows/eval_guard.yml):
```bash
python scripts/run_experiments.py --phase a --phase b --phase c --latency-samples 20
python scripts/check_lab.py --with-tests
```
Cơ chế chặn thất bại: Script kiểm tra `check_lab.py` thực thi assert toàn bộ điều kiện chất lượng (schema hợp lệ, 0 failed checks, zero TODOs, metrics đạt ngưỡng) và trả về exit code 1 khi vi phạm, làm dừng workflow CI ngay lập tức. Các biến môi trường đề xuất như MIN_FAITHFULNESS=0.75 được đưa vào script runner để kiểm tra tự động trước khi cho phép merge code. Lưu ý: Đây là đề xuất kiến trúc CI/CD, workflow này hiện tại được tài liệu hóa trong blueprint và chưa triển khai tự động trên CI runner.

## Monitoring và cải tiến

Giám sát vận hành trong môi trường production:
- **Độ trễ P95**: Cảnh báo Slack/PagerDuty khi P95 input guard > 2500ms hoặc P95 RAG response > 5000ms.
- **Tấn công bảo mật**: Cảnh báo khi tỷ lệ request bị chặn vượt quá 10% trong vòng 15 phút (nghi vấn bị dò quét prompt injection / jailbreak hàng loạt).
- **False Positives**: Cung cấp nút "Báo cáo chặn nhầm" trên giao diện người dùng; cảnh báo nếu tỷ lệ chặn nhầm vượt quá 1% tổng lưu lượng.
- **Drift và Privacy**: Định kỳ hàng tháng lấy mẫu 5% request thực tế (đã qua redact PII hoàn toàn) để đánh giá lại độ trung thực (faithfulness). Người phụ trách xử lý: Nguyễn Phương Nam (MLOps Lead).
- **Giới hạn bộ mẫu**: Bộ dữ liệu 50 câu RAG, 20 attack, 8 clean, 4 output là bộ benchmark kiểm thử thu nhỏ; khi đưa vào thực tế cần mở rộng lên ít nhất 500 ca kiểm thử đa dạng.

RAGAS mean/worst distribution: Factual mean = 0.8976, Multi-hop mean = 0.7692, Adversarial mean = 0.7676 (worst distribution: adversarial với điểm trung bình 0.7676, trong đó faithfulness ở multi-hop thấp nhất đạt 0.5869; overall RAGAS mean đạt 0.8203).
Cohen’s κ: 1.0 (sự nhất trí tuyệt đối giữa LLM Judge và Human Ground Truth trên 10 câu mẫu).
Attacks passed: 19/20; clean false_positive_rate: 0.0 (8/8 clean inputs allowed, FPR = 0.0%).

Các cải tiến cụ thể có thể kiểm chứng gắn với bằng chứng thực nghiệm:
1. Thay thế NeMo Cloud API bằng mô hình Guardrail chuyên dụng on-premise (như Llama-Guard-3-8B hoặc SmolLM phục vụ qua vLLM cục bộ) để giảm độ trễ P95 từ 1910ms xuống dưới ngưỡng mục tiêu 200ms.
2. Áp dụng kỹ thuật truy xuất đa chặng hoặc HyDE cho nhóm câu hỏi multi-hop nhằm nâng cao faithfulness từ 0.5869 lên trên 0.80 (khắc phục lỗi ở câu 33, 38).
3. Siết chặt prompt template sinh câu trả lời với ràng buộc chỉ trả lời dựa trên context được cấp để loại bỏ hoàn toàn hiện tượng hallucination ở nhóm adversarial.
4. Triển khai Semantic Caching (Redis Vector Cache) cho các câu hỏi chính sách thường gặp giúp phản hồi ngay lập tức (<20ms) và tiết kiệm chi phí gọi LLM.
5. Mở rộng Presidio Recognizer với tập dữ liệu tên riêng và số tài khoản ngân hàng Việt Nam để tăng cường độ phủ PII.
