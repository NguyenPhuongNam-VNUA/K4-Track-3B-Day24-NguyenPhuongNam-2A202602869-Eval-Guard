# Nhóm thực hiện K4 Track 3B

Tên nhóm: NguyenPhuongNam-2A202602869

Repo chung: https://github.com/NguyenPhuongNam-VNUA/K4-Track-3B-Day24-NguyenPhuongNam-2A202602869-Eval-Guard.git

Trưởng nhóm/người nộp: Nguyễn Phương Nam - 2A202602869

Ngày lab/deadline áp dụng: 11:59 AM (13/10/2026) — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: 5bba7c3bff4af99082f35d8c880850ed5db348c8

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Nguyễn Phương Nam | 2A202602869 | Thực hiện toàn bộ Pipeline Setup, Phase A (RAGAS), Phase B (LLM Judge), Phase C (Guardrails), Task 13 (Blueprint) | Toàn bộ commits git, artifacts/*.json, doc/reports/*.md |

Người tích hợp/chạy batch/nộp link: Nguyễn Phương Nam (2A202602869), thực hiện toàn bộ các phase từ môi trường, chạy batch thực nghiệm thật, viết báo cáo kỹ thuật và nộp repo.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: macOS Sequoia 15.0 (Apple Silicon arm64), Python 3.11.16, Apple Clang 16.0 (Xcode CLI Tools).
Virtualenv/dependencies/imports: Quản lý qua uv venv (.venv), cài đặt đầy đủ theo requirements.txt gồm ragas 0.1.22, langchain 0.2.17, nemoguardrails 0.10.0, presidio 2.2.364, pyarrow 17.0.0, spacy 3.8.16 cùng cả hai model en_core_web_lg và en_core_web_sm.
Models/Qdrant/50 answers đã chuẩn bị: Khởi chạy Qdrant v1.9.7 qua docker compose tại localhost:6333, dense embedding BAAI/bge-m3, cross-encoder reranker BAAI/bge-reranker-v2-m3, LLM gpt-4o-mini qua OpenRouter API. Đã hoàn thành chạy scripts/setup_answers.py --live sinh đủ 50 answers (20 factual, 20 multi_hop, 10 adversarial) lưu tại artifacts/answers_50q.json.

## INDIVIDUAL

### NguyenPhuongNam-2A202602869

- Phần được giao và phần thực tế đã làm:
  1. Pipeline setup: Tích hợp 5 module RAG m1-m5, cấu hình Qdrant Docker, sinh 50 answers cho 3 distributions (factual, multi_hop, adversarial).
  2. Phase A: Implement các hàm group_by_distribution, run_ragas_50q, bottom_10, cluster_analysis trong src/phase_a_ragas.py; chạy thực nghiệm đánh giá 4 metrics RAGAS; phân tích failure clusters.
  3. Phase B: Implement pairwise_judge, swap_and_average (khử position bias), cohen_kappa, bias_report trong src/phase_b_judge.py; chạy pairwise LLM Judge trên 5 cặp câu hỏi và calibration 10 nhãn nhị phân đối chiếu với human annotations.
  4. Phase C: Implement Presidio PII scan (CCCD/CMND, SĐT VN, Email), NeMo input rail, adversarial suite (20 attacks + 8 clean controls), NeMo output rail, và đo lường latency P95 của sequential input layers (warmup 2 lượt, 20 timed samples).
  5. Task 13: Xây dựng blueprint kiến trúc CI/CD, failure action, quality gates và monitoring đề xuất cho production.
- Bằng chứng:
  - Các commit tương ứng từng phase trên branch main.
  - Các artifact lưu trữ kết quả thực nghiệm: artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/judge_results.json, artifacts/guard_results.json.
  - Ba báo cáo kỹ thuật hoàn chỉnh: doc/reports/failure_clusters.md, doc/reports/bias_report.md, doc/reports/blueprint.md.
- Cách kiểm tra:
  - Unit tests offline: python -m pytest tests/ -q (86/86 passed).
  - Live experiments: chạy các script thực nghiệm với cờ --live trên từng phase.
  - Submission checker: python scripts/check_lab.py --with-tests (đạt PASS mọi bài kiểm tra).
- Điều đã học:
  - Nắm vững quy trình đánh giá RAG toàn diện bằng 4 metrics của RAGAS (faithfulness, answer_relevancy, context_precision, context_recall).
  - Hiểu sâu về các loại bias trong LLM-as-a-Judge (position bias, verbosity bias) và kỹ thuật swap-and-average kết hợp hệ số đồng thuận Cohen's Kappa để hiệu chuẩn judge.
  - Triển khai hệ thống defense-in-depth guardrails đa tầng kết hợp Presidio PII regex/NER và NeMo Guardrails (Colang 1.0) cho cả input và output, cân bằng giữa khả năng chặn tấn công và tỷ lệ false positive trên clean inputs, đồng thời kiểm soát độ trễ P95.
- Giới hạn/cách xử lý:
  - Xử lý vấn đề tương thích giữa pyarrow và NumPy 1.26 bằng cách pin pyarrow==17.0.0.
  - Cấu hình OpenRouter endpoint tương thích cho cả OpenAI client (OPENAI_BASE_URL) và LangChain/NeMo (OPENAI_API_BASE).
- Nguồn/AI hỗ trợ nếu có:
  - Tài liệu chính thức của RAGAS, Presidio, NeMo Guardrails, LangChain.
  - Công cụ Antigravity AI pair programming hỗ trợ kiểm tra mã nguồn, đối chiếu schema và phân tích kết quả thực nghiệm.

## Cập nhật sau khi nộp

Không có cập nhật sau deadline. Nộp bài đúng hạn trước deadline quy định.
