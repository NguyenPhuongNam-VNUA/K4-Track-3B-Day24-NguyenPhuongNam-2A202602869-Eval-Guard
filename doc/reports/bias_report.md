# Đánh giá LLM Judge Phase B

Nhóm: NguyenPhuongNam-2A202602869

Người phụ trách: Nguyễn Phương Nam - 2A202602869

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T16:51:28.587680+00:00.
Models generation/judge: Generation model: `openai/gpt-4o-mini`; Judge model: `gpt-4o-mini`.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ `artifacts/answers_50q.json:answer`; B từ `fixtures/human_labels_10q.json:model_answer`; reference từ `fixtures/test_set_50q.json:ground_truth`.
Ids: 1, 5, 12, 21, 41.

Bảng kết quả pairwise qua 2 lượt hoán đổi vị trí (swap-and-average):

| ID | Candidate A (Pipeline) | Candidate B (Human Dataset) | Winner Pass 1 | Winner Pass 2 | Final Winner | Pos. Consistent | Scores Pass 1 | Scores Pass 2 |
|---|---|---|---|---|---|---|---|---|
| 1 | Được nghỉ 3 ngày có hưởng lương... | Được nghỉ 3 ngày hưởng nguyên lương khi kết hôn... | B | B | B | True | A: 0.8, B: 1.0 | A: 0.8, B: 1.0 |
| 5 | Cần Giám đốc khối hoặc cấp quản lý tương đương... | Cần Giám đốc khối phê duyệt là đủ. | tie | A | tie | False | A: 0.3, B: 0.3 | A: 0.6, B: 0.0 |
| 12 | Thưởng Tết căn cứ vào kết quả đánh giá... | Thưởng Tết được tính theo công thức: Lương cơ bản... | A | A | A | True | A: 1.0, B: 0.8 | A: 1.0, B: 0.8 |
| 21 | Số ngày phép năm là 13 ngày... Khung lương... | Được nghỉ 13 ngày phép năm... Khung lương Senior... | B | B | B | True | A: 0.8, B: 1.0 | A: 0.7, B: 1.0 |
| 41 | Theo quy định v2024, nhân viên được 14 ngày... | Được nghỉ 15 ngày phép năm theo quy định thâm niên... | A | A | A | True | A: 1.0, B: 0.0 | A: 1.0, B: 0.3 |

Position bias count/rate: 1 / 5 cặp (tỷ lệ 0.200 / 20.0%).
Verbosity win rate: 0.750 (75.0%); decisive cases: 4 / 5 (A thắng dài hơn: 2 case; B thắng dài hơn: 1 case).

### Diễn giải chi tiết về Position Bias và Verbosity

1. **Phân tích Position Inconsistency:**
   - Trong 5 cặp câu hỏi, có 4/5 cặp đạt sự đồng thuận tuyệt đối giữa hai lượt đổi thứ tự (`position_consistent = True`), bao gồm câu 1 (B thắng), câu 12 (A thắng), câu 21 (B thắng) và câu 41 (A thắng).
   - Duy nhất câu 5 có sự không nhất quán (`position_consistent = False`): Ở Pass 1 (thứ tự A/B), Judge đánh giá cả hai câu trả lời đều chưa chuẩn xác về thẩm quyền (cần Tổng Giám đốc thay vì Giám đốc khối) nên đưa ra kết quả `tie` với điểm số cân bằng 0.3 - 0.3. Ở Pass 2 (thứ tự B/A), Judge lại chấm điểm cho candidate A cao hơn (0.6 vs 0.0) vì A có giải thích chi tiết hơn về quy trình đề xuất. Nhờ cơ chế `swap-and-average`, khi hai lượt không đạt consensus, kết quả cuối cùng được hạ về `tie`, giúp loại bỏ hoàn toàn quyết định thiên vị do vị trí đứng trước mang lại.

2. **Phân tích Verbosity Bias:**
   - Tỷ lệ Verbosity đạt 0.75 (3 trên 4 trường hợp có người chiến thắng rõ ràng thuộc về câu trả lời dài hơn).
   - Tuy nhiên, qua đối chiếu ngữ nghĩa, sự vượt trội về độ dài ở đây đi liền với tính đầy đủ (completeness) của thông tin:
     - Ở câu 41: Candidate A dài hơn vì trích dẫn tường minh căn cứ chính sách v2024 (12 ngày cơ bản + 2 ngày cho 10 năm thâm niên = 14 ngày), trong khi B ngắn hơn nhưng dùng công thức cũ sai lệch (mỗi 3 năm tăng 1 ngày). Judge chọn A vì tính đúng đắn chứ không thuần túy vì độ dài.
     - Ở câu 12: Candidate A mô tả đầy đủ các điều kiện xét thưởng thay vì chỉ đưa ra công thức thô.
   - Do đó, tỷ lệ 75% phản ánh tương quan giữa độ dài hợp lý và độ sâu thông tin hơn là thiên vị hình thức.

## Calibration đúng model_answer

Quy trình calibration đánh giá độc lập 10 câu hỏi từ `fixtures/human_labels_10q.json` với chính `model_answer` đi kèm, sử dụng reference từ `test_set_50q.json:ground_truth`. Prompt chỉ gồm câu hỏi, câu trả lời cần thẩm định và văn bản tham chiếu; tuyệt đối không đưa `human_label` hay `human_note` vào prompt.

Bảng đối chiếu 10 nhãn nhị phân giữa LLM Judge và chuyên gia người:

| ID | Model Answer Gốc | Nhãn Judge | Reasoning của Judge | Nhãn Human | Đồng Thuận |
|---|---|---|---|---|---|
| 1 | Được nghỉ 3 ngày hưởng nguyên lương khi kết hôn. Nếu kết hôn lần 2 vẫn được hưởng. | 1 | Câu trả lời chính xác theo tài liệu tham khảo, nêu rõ quyền lợi 3 ngày và quy định kết hôn. | 1 | True |
| 5 | Cần Giám đốc khối phê duyệt là đủ. | 0 | Câu trả lời sai vì mua sắm trên 50 triệu bắt buộc phải có phê duyệt từ Tổng Giám đốc. | 0 | True |
| 12 | Thưởng Tết được tính theo công thức: Lương cơ bản x Hệ số hiệu suất x Tỷ lệ hoàn thành KPI. | 1 | Câu trả lời chính xác theo reference, nêu đúng công thức và các tham số tính thưởng. | 1 | True |
| 21 | Được nghỉ 13 ngày phép năm (12 ngày cơ bản + 1 ngày thâm niên). Khung lương Senior: 25 - 45 triệu VNĐ. | 1 | Câu trả lời chính xác cả hai vế về số ngày phép và khung lương bậc Senior theo chính sách. | 1 | True |
| 23 | Cần thông báo cho quản lý trước 3 ngày làm việc và hoàn tất bàn giao công việc. | 1 | Câu trả lời chính xác theo quy trình nghỉ việc và thời hạn thông báo theo hợp đồng. | 1 | True |
| 29 | Được mang laptop cá nhân về nhà mà không cần đăng ký hay phê duyệt. | 0 | Câu trả lời sai vì chính sách IT bắt buộc phải đăng ký tài sản và có phê duyệt của quản lý. | 0 | True |
| 33 | Phụ cấp Manager: 3 triệu/tháng. Phép năm v2024: 14 ngày (12 cơ bản + 2 ngày cho 12 năm thâm niên). | 1 | Câu trả lời chính xác, tính đúng số ngày phép theo v2024 và mức phụ cấp vị trí. | 1 | True |
| 41 | Được nghỉ 15 ngày phép năm theo quy định thâm niên (cứ 3 năm thêm 1 ngày). | 0 | Câu trả lời sai vì căn cứ trên chính sách cũ v2023 thay vì chính sách mới v2024 (cứ 5 năm). | 0 | True |
| 46 | Nhân viên chính thức ký HĐLĐ từ 12 tháng trở lên được công ty mua bảo hiểm sức khỏe PVI. | 1 | Câu trả lời chính xác theo reference về đối tượng hưởng bảo hiểm sức khỏe tự nguyện. | 1 | True |
| 50 | Có thể sử dụng VPN cá nhân để truy cập tài nguyên nội bộ công ty khi làm việc từ xa. | 0 | Câu trả lời sai vì chính sách bảo mật nghiêm cấm sử dụng VPN thương mại ngoài luồng. | 0 | True |

Cohen’s κ: **1.000**.
Observed/chance agreement và diễn giải:
- Observed Agreement: $P_o = \frac{10}{10} = 1.000$ (100% các nhãn trùng khớp).
- Expected Chance Agreement: Với phân bố 6 nhãn 1 và 4 nhãn 0:
  $$P_e = (0.6 \times 0.6) + (0.4 \times 0.4) = 0.36 + 0.16 = 0.520$$
- Hệ số Cohen's Kappa:
  $$\kappa = \frac{P_o - P_e}{1 - P_e} = \frac{1.000 - 0.520}{1 - 0.520} = \frac{0.480}{0.480} = 1.000$$
- Theo thang đo tiêu chuẩn Landis & Koch (1977), $\kappa = 1.000$ đại diện cho mức đồng thuận tuyệt đối (Almost Perfect / Perfect Agreement). LLM Judge phân biệt chuẩn xác 100% các trường hợp đúng và sai (phát hiện cả 4 trường hợp sai phạm về chính sách: câu 5, 29, 41, 50).

## Giới hạn và cách dùng judge

1. **Quy mô tập mẫu:** Tập kiểm thử calibration gồm 10 mẫu (6 nhãn tốt, 4 nhãn xấu). Mặc dù đạt $\kappa = 1.000$, trong môi trường production cần mở rộng tập calibration lên tối thiểu 100–200 mẫu được gán nhãn đa tầng bởi ít nhất 2 annotators độc lập để đánh giá inter-annotator agreement.
2. **Kỹ thuật hạn chế rủi ro khi dùng LLM Judge:**
   - Luôn sử dụng kỹ thuật **Swap-and-Average** cho các tác vụ pairwise so sánh A/B để triệt tiêu position bias.
   - Luôn cung cấp ground truth / reference policy rõ ràng trong prompt; không để LLM Judge tự suy diễn dựa trên kiến thức tham số tự do.
   - Thiết lập ngưỡng Confidence / Temperature = 0 để đảm bảo tính tất định (deterministic output).
   - Trong production, các quyết định quan trọng (như phê duyệt hoàn tiền, xử phạt nhân sự) không bao giờ giao toàn quyền cho LLM Judge mà phải kết hợp Human-in-the-Loop (HITL) để rà soát các trường hợp điểm số ở ranh giới (borderline cases, $0.4 \le score \le 0.6$).
