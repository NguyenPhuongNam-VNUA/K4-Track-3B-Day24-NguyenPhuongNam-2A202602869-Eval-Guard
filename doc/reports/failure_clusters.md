# Phân tích lỗi Phase A

Nhóm: NguyenPhuongNam-2A202602869

Người phụ trách: Nguyễn Phương Nam - 2A202602869

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T16:50:04.927517+00:00.
Generation mode/fallback và điều kiện chạy: Toàn bộ 50 câu hỏi được thực thi ở chế độ live API với `generation_mode: llm`, không có câu hỏi nào rơi vào `extractive_fallback`. Pipeline sử dụng Hybrid Search (BM25 Underthesea + Dense Qdrant BAAI/bge-m3) kết hợp Cross-Encoder Reranker (BAAI/bge-reranker-v2-m3) top-3 context.
Models generation/evaluator: Generation model: `openai/gpt-4o-mini`; Evaluator models: `gpt-4o-mini`, `text-embedding-ada-002` (RAGAS 0.1.22).

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.9750 | 0.5869 | 0.7750 |
| answer_relevancy | 0.8322 | 0.7025 | 0.5953 |
| context_precision | 0.9833 | 0.9833 | 0.9833 |
| context_recall | 0.8000 | 0.8042 | 0.7167 |
| avg_score | 0.8976 | 0.7692 | 0.7676 |

## Bottom 10 và nguyên nhân

Bảng 10 câu hỏi có avg_score thấp nhất trích xuất trực tiếp từ Task 3:

| Rank | ID | Distribution | Câu hỏi | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|---|---|---|---|---|---|---|---|
| 1 | 33 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 48 | adversarial | Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không? | 0.4167 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 50 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.4167 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 22 | multi_hop | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT? | 0.4583 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 5 | 21 | multi_hop | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | 0.5625 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 6 | 24 | multi_hop | Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu? | 0.6924 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 7 | 7 | factual | Nghỉ phép không lương 20 ngày cần ai phê duyệt? | 0.6959 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 8 | 5 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.7014 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 9 | 38 | multi_hop | Nhân viên đi công tác nước ngoài 4 ngày tại thành phố lớn, ở khách sạn 200 USD/đêm (3 đêm). Công ty thanh toán bao nhiêu tiền khách sạn? | 0.7035 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 10 | 31 | multi_hop | Nhân viên đi công tác trong nước, ở khách sạn 2 đêm với giá 1.500.000 VNĐ/đêm. Công ty thanh toán tối đa bao nhiêu cho tiền khách sạn? | 0.7049 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |

### Đối chiếu chi tiết nguyên nhân lỗi

1. **Lỗi Multi-hop và phép tính số học (Câu 33, 24, 38, 31):**
   - *Câu 33 (avg_score 0.3333, faithfulness 0.0)*: Đòi hỏi kết hợp hai văn bản chính sách độc lập: `phu_cap.md` (phụ cấp Manager) và `nghi_phep_nam_v2024.md` (phép cơ bản 12 ngày + 2 ngày cho 10 năm thâm niên = 14 ngày). Do retrieval lấy chunk từ cả v2023 và v2024, LLM bị phân vân và sinh đáp án không khớp trọn vẹn context dẫn tới điểm faithfulness = 0.
   - *Câu 24 (avg_score 0.6924)*: Câu hỏi yêu cầu tính mức phạt khi tạm ứng quá hạn. Mặc dù retrieval lấy đúng tài liệu `tam_ung.md`, LLM gặp khó khăn trong việc suy luận logic điều kiện mốc ngày hoàn ứng và mức lãi suất quá hạn, dẫn đến việc diễn giải không hoàn toàn trung thực với context.
   - *Câu 38 & 31*: Liên quan đến trần chi phí công tác phí khách sạn trong và ngoài nước. LLM tự động suy diễn nhân tiền tệ và áp trần không chuẩn xác theo từng hạng thành phố.

2. **Lỗi Adversarial - Bẫy phủ định và điều kiện loại trừ (Câu 48, 50):**
   - *Câu 48 (avg_score 0.4167)*: Hỏi về quyền lợi bảo hiểm PVI cho nhân viên thử việc. Context từ `bao_hiem_suc_khoe.md` nêu rõ PVI chỉ áp dụng cho nhân viên ký hợp đồng lao động chính thức, trong khi `thu_viec.md` nêu quyền lợi thử việc không gồm bảo hiểm tự nguyện. LLM có xu hướng trả lời chung chung "được hỗ trợ theo quy định" thay vì khẳng định dứt khoát "KHÔNG ĐƯỢC".
   - *Câu 50 (avg_score 0.4167)*: Hỏi về việc dùng VPN cá nhân (NordVPN). Context `vpn_truy_cap.md` nghiêm cấm mọi kết nối không thuộc hạ tầng bảo mật nội bộ được phê duyệt. LLM hallucinate rằng việc bật thêm VPN cá nhân "có thể chấp nhận được nếu báo trước IT", đi ngược lại chính sách cấm tuyệt đối.

3. **Lỗi Retrieval & Relevancy (Câu 22, 21, 5):**
   - *Câu 22 & 21*: Câu hỏi chứa hai vế độc lập ("ai duyệt + cần gì từ CNTT", "phép năm + khung lương"). Reranker chọn context thiên về một vế (ví dụ chỉ có quy định mua sắm hoặc chỉ có khung lương), khiến LLM trả lời thiếu vế còn lại, làm giảm mạnh `answer_relevancy`.
   - *Câu 5*: Bị ảnh hưởng bởi phân ngưỡng thẩm quyền phê duyệt mua sắm tài sản (trên 50 triệu cần Tổng Giám đốc thay vì Giám đốc Khối). Chữ số 55 triệu nằm ở ranh giới phân cấp thẩm quyền trong bảng biểu `mua_sam.md`.

## Matrix và nhóm yếu nhất

Matrix đếm phân bố worst_metric theo từng distribution (mỗi câu tính 1 lần theo metric thấp nhất của câu đó):

| Worst Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng |
|---|---|---|---|---|
| answer_relevancy | 13 | 3 | 1 | 17 |
| context_precision | 0 | 0 | 1 | 1 |
| context_recall | 6 | 4 | 6 | 16 |
| faithfulness | 1 | 13 | 2 | 16 |
| **Tổng số câu** | **20** | **20** | **10** | **50** |

Nhóm yếu theo mean thấp nhất: `adversarial`; mean: 0.7676 (so với `multi_hop`: 0.7692 và `factual`: 0.8976).
Worst metric xuất hiện nhiều nhất: `answer_relevancy`; số câu: 17 câu.

### Phân tích và đề xuất giải pháp

- **Vì sao adversarial là nhóm yếu nhất:** Điểm trung bình của nhóm adversarial (0.7676) thấp hơn đáng kể so với factual (0.8976). Lý do cốt lõi là các câu adversarial được thiết kế chứa bẫy: câu hỏi có tiền giả định sai, phủ định kép hoặc xung đột phiên bản chính sách. Điểm `answer_relevancy` của adversarial chỉ đạt 0.5953 và `context_recall` chỉ đạt 0.7167. Khi retrieval trích xuất các chunk chứa từ khóa trùng khớp (lexical match), LLM bị bẫy bởi văn bản trích dẫn và không phát hiện ra điều kiện loại trừ hoặc giả định sai trong câu hỏi.
- **Vì sao faithfulness là điểm nghẽn chính của multi-hop:** Trong nhóm multi-hop, có tới 13/20 câu có worst_metric là `faithfulness` (điểm trung bình chỉ đạt 0.5869). Khi cần tổng hợp thông tin từ 2–3 tài liệu khác nhau, LLM thường tự động lấp đầy khoảng trống thông tin bằng tri thức nền (parametric memory) thay vì hoàn toàn bám sát context, dẫn đến trượt tiêu chí faithfulness.
- **Đề xuất cải tiến cụ thể:**
  1. *Prompt engineering*: Bổ sung chỉ dẫn tường minh trong system prompt yêu cầu LLM kiểm tra tính hợp lệ của tiền giả định trước khi trả lời ("Nếu câu hỏi đưa ra giả định sai so với chính sách, hãy chỉ rõ điểm sai đó thay vì trả lời theo giả định").
  2. *Retrieval enrichment*: Đưa siêu dữ liệu `effective_year` / `version` vào bộ lọc metadata filter để loại bỏ hoàn toàn các tài liệu cũ (như `nghi_phep_nam_v2023.md`, `mat_khau_v1.md`) khi câu hỏi truy vấn quy định hiện hành.
  3. *Query Decomposition*: Áp dụng kỹ thuật tách câu hỏi multi-hop thành các sub-queries đơn lẻ trước khi truy xuất nhằm tối ưu hóa `context_recall` cho từng vế câu hỏi.
  4. *Phương pháp kiểm chứng*: Chạy lại `python scripts/run_experiments.py --phase a --live` sau khi cập nhật prompt/metadata filter; mục tiêu là nâng faithfulness của multi-hop lên ≥0.75 và answer_relevancy của adversarial lên ≥0.70.

## Bẫy adversarial và giới hạn

### So sánh Factual vs Adversarial

- Điểm trung bình nhóm Factual đạt **0.8976** (Faithfulness: 0.9750, Precision: 0.9833, Relevancy: 0.8322, Recall: 0.8000). Đây là nhóm câu hỏi đơn giản, một nguồn tài liệu duy nhất và không có bẫy.
- Điểm trung bình nhóm Adversarial đạt **0.7676** (thấp hơn Factual **0.1300** điểm). Điểm Relevancy giảm mạnh từ 0.8322 xuống 0.5953 (-28.5%), Recall giảm từ 0.8000 xuống 0.7167 (-10.4%).
- Sự sụt giảm điểm số này phản ánh đúng bản chất: bộ câu hỏi adversarial thành công trong việc thử thách năng lực phân định logic của pipeline.

### Các dạng bẫy cụ thể trong dữ liệu

1. **Bẫy xung đột phiên bản (Version Conflicts):**
   - Điển hình là cặp văn bản `nghi_phep_nam_v2023.md` (chính sách cũ: 12 ngày phép, mỗi 3 năm tăng 1 ngày) và `nghi_phep_nam_v2024.md` (chính sách mới: 12 ngày phép, mỗi 5 năm tăng 1 ngày). Khi người dùng hỏi quy định thâm niên, dense search trả về cả hai phiên bản vì độ tương đồng ngữ nghĩa cực cao; LLM không phân biệt được thời gian hiệu lực và lấy nhầm mốc tính của năm 2023.
   - Cặp văn bản `mat_khau_v1.md` (độ dài tối thiểu 8 ký tự, đổi mỗi 90 ngày) và `mat_khau_v2.md` (độ dài tối thiểu 12 ký tự, không bắt buộc đổi định kỳ trừ khi lộ lọt). RAG pipeline nếu không có temporal filtering sẽ tạo ra câu trả lời pha trộn cả hai chính sách.

2. **Bẫy phủ định và đối tượng ngoại lệ (Negation & Ineligibility):**
   - Các câu hỏi giả định quyền lợi dành cho đối tượng không đủ tiêu chuẩn: nhân viên thử việc đòi quyền lợi của nhân viên chính thức (câu 48); nhân viên part-time đòi bảo hiểm sức khỏe VIP; nhân viên tự ý dùng phần mềm ngoài luồng (câu 50). LLM thường có bias "đồng thuận" (sycophancy), cố gắng giải thích cách tiếp cận thay vì dứt khoát từ chối.

### Giới hạn thực nghiệm

- Kích thước tập dữ liệu 50 câu (20 factual, 20 multi-hop, 10 adversarial) tuy đủ bao quát các trường hợp kiểm thử chính của lab nhưng còn nhỏ để đại diện cho toàn bộ phân phối truy vấn thực tế trong doanh nghiệp.
- Việc RAGAS evaluator sử dụng model `gpt-4o-mini` cho cả generation và evaluation có thể tiềm ẩn judge alignment bias (model tự đánh giá cao phong cách hành văn của chính nó).
- Toàn bộ kết quả dựa trên 100% live API execution đã được ghi nhận đầy đủ, không can thiệp số liệu thực nghiệm.
