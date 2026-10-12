# CẨM NANG HỌC TẬP VÀ TOÀN TẬP KIẾN TRÚC: RAG EVALUATION & GUARDRAILS (DAY 24)

> **Tác giả:** Nguyễn Phương Nam (MSSV: 2A202602869)  
> **Chủ đề:** Đánh giá RAG toàn diện (RAGAS + LLM-as-a-Judge) và Xây dựng Hệ thống Phòng thủ An toàn (Guardrails) cho Production.

---

## MỤC LỤC
1. [Bức tranh tổng thể: Tại sao cần Eval và Guardrails?](#1-bức-tranh-tổng-thể)
2. [Kiến trúc luồng xử lý End-to-End](#2-kiến-trúc-luồng-xử-lý-end-to-end)
3. [Phase A: Đánh giá RAG tự động bằng RAGAS (4 Metrics Cốt lõi)](#3-phase-a-đánh-giá-rag-bằng-ragas)
4. [Phase B: Đánh giá bằng LLM-as-a-Judge & Khử Định kiến (Bias Mitigation)](#4-phase-b-llm-as-a-judge--khử-định-kiến)
5. [Phase C: Phòng thủ Đa tầng với Presidio PII & NeMo Guardrails](#5-phase-c-phòng-thủ-đa-tầng-guardrails)
6. [Đo lường Độ trễ P50/P95/P99 & Bản thiết kế CI/CD (Blueprint)](#6-đo-lường-latency--bản-thiết-kế-cicd)
7. [Các cạm bẫy thực tế & Bài học kinh nghiệm xương máu](#7-các-cạm-bẫy-thực-tế--kinh-nghiệm)
8. [Hướng dẫn chạy lại thực nghiệm (Reproducibility Guide)](#8-hướng-dẫn-chạy-lại-thực-nghiệm)

---

## 1. BỨC TRANH TỔNG THỂ

Trong các ứng dụng RAG (Retrieval-Augmented Generation) thực tế tại doanh nghiệp, việc chỉ dựng được pipeline tìm kiếm và sinh câu trả lời mới chỉ hoàn thành 30% chặng đường. 70% còn lại quyết định hệ thống có thể lên Production hay không nằm ở hai trụ cột:

1. **Evaluation (Đo lường & Đánh giá):**
   - Làm sao biết pipeline trả lời đúng sự thật hay bị ảo giác (hallucination)?
   - Làm sao định lượng được chất lượng của Retriever (Vector DB + Reranker) và Generator (LLM)?
   - Khi tinh chỉnh prompt, chunk size hay đổi model, làm sao biết hệ thống tốt lên hay tệ đi?
2. **Guardrails (Phòng thủ & An toàn thông tin):**
   - Làm sao ngăn chặn người dùng gửi thông tin nhạy cảm (PII: CCCD, Số điện thoại, Email cá nhân)?
   - Làm sao chống lại các đòn tấn công Prompt Injection, Jailbreak (như DAN - Do Anything Now), mạo danh HR/Admin để trộm dữ liệu lương, mật khẩu nội bộ?
   - Làm sao đảm bảo câu trả lời của AI trước khi đến tay người dùng không bị rò rỉ bí mật công ty?
   - Tất cả các lớp phòng thủ trên phải chạy trong bao lâu để không làm người dùng cảm thấy ứng dụng bị "đơ" (Latency Budget)?

Repository này là một hệ thống hoàn chỉnh từ A đến Z giải quyết toàn bộ các bài toán trên.

---

## 2. KIẾN TRÚC LUỒNG XỬ LÝ END-TO-END

Luồng xử lý của một yêu cầu trong hệ thống phòng thủ hoàn chỉnh:

```mermaid
flowchart TD
    User([Người dùng / Client]) --> In[Query Input]
    
    subgraph Input_Guard [TẦNG BẢO VỆ ĐẦU VÀO - Input Guard]
        In --> PII{Presidio PII Scan}
        PII -- "Phát hiện CCCD/SĐT trái phép" --> BlockPII[Chặn hoặc Redact ngay]
        PII -- "Hợp lệ" --> NeMoIn{NeMo Input Rail}
        NeMoIn -- "Jailbreak / Off-topic / Hack" --> BlockNeMo[Ngắt ngay: Short-circuit]
    end
    
    BlockPII --> Refusal[Trả lời từ chối an toàn]
    BlockNeMo --> Refusal
    
    subgraph Core_RAG [PIPELINE RAG CỐT LÕI]
        NeMoIn -- "An toàn" --> Search[Qdrant Dense Vector Search]
        Search --> Chunks[(Top Chunks)]
        Chunks --> Rerank[Cross-Encoder Reranker]
        Rerank --> TopK[(Top-3 Chunks cô đọng)]
        TopK --> Gen[LLM Generator: gpt-4o-mini]
    end
    
    subgraph Output_Guard [TẦNG BẢO VỆ ĐẦU RA - Output Guard]
        Gen --> RawAns[Raw Answer]
        RawAns --> NeMoOut{NeMo Output Rail}
        NeMoOut -- "Lộ PII / Mật khẩu Admin" --> RedactOut[Thay bằng thông báo từ chối]
        NeMoOut -- "An toàn" --> FinalAns[Giữ nguyên phản hồi sạch]
    end
    
    RedactOut --> User
    FinalAns --> User
    Refusal --> User
```

**Nguyên lý cốt lõi:**
- **Fast-fail (Short-circuit):** Chặn ngay từ tầng ngoài cùng. Nếu input chứa prompt độc hại hoặc PII vi phạm, ngắt pipeline ngay lập tức mà **KHÔNG** gọi vào RAG hay LLM Generator, giúp tiết kiệm 100% token cost và triệt tiêu nguy cơ injection.
- **Defense-in-depth (Phòng thủ chiều sâu):** Presidio xử lý regex/pattern cực nhanh trên CPU; NeMo giải quyết ngữ nghĩa sâu (semantic reasoning); Output rail bọc lót tầng cuối cùng nếu LLM sinh ra nội dung nguy hiểm.

---

## 3. PHASE A: ĐÁNH GIÁ RAG BẰNG RAGAS

RAGAS (Retrieval Augmented Generation Assessment) là framework đánh giá RAG tiêu chuẩn công nghiệp không cần con người chấm điểm thủ công (Reference-free & Reference-based).

### 3.1. Bản chất toán học của 4 Metrics

Hệ thống đánh giá trên 4 chỉ số chia thành 2 nhóm:

| Metric | Nhóm đánh giá | Công thức / Bản chất hoạt động | Ý nghĩa |
|---|---|---|---|
| **Faithfulness** | Generator | $\frac{\text{Số claims được Context suy diễn được}}{\text{Tổng số claims trong Answer}}$ | Đo độ **trung thực**, chống ảo giác (hallucination). Nếu câu trả lời "bịa" ra thông tin không có trong tài liệu tra cứu, điểm sẽ bị phạt nặng. |
| **Answer Relevancy** | Generator | $\frac{1}{N}\sum_{i=1}^N \cos(\mathbf{E}(q), \mathbf{E}(q_i^*))$ | Đo độ **tập trung vào câu hỏi**. RAGAS dùng LLM sinh ngược $N$ câu hỏi $q_i^*$ từ câu trả lời, sau đó tính độ tương đồng cosine embedding với câu hỏi gốc $q$. Nếu trả lời vòng vo, lạc đề, điểm sẽ thấp. |
| **Context Precision** | Retriever | $\text{mAP} = \sum_{k=1}^K \frac{P@k \times \mathbb{I}(k \text{ is relevant})}{\text{Tổng số chunks liên quan}}$ | Đo **chất lượng xếp hạng (ranking)**. Các đoạn văn bản chứa câu trả lời đúng có nằm ở top đầu (rank 1, 2) hay bị trôi xuống dưới cùng? |
| **Context Recall** | Retriever | $\frac{\text{Số câu trong Ground Truth quy chiếu được từ Context}}{\text{Tổng số câu trong Ground Truth}}$ | Đo **độ bao phủ**. Context được lấy về có chứa đủ tất cả các ý cần thiết để trả lời câu hỏi so với đáp án chuẩn hay không? |

### 3.2. Ba phân phối câu hỏi kiểm thử (Test Distributions)
Trong tệp `fixtures/test_set_50q.json`, 50 câu hỏi được chia làm 3 dạng có chủ đích:
1. **Factual (20 câu):** Câu hỏi tra cứu trực tiếp 1 tài liệu (ví dụ: *"Nhân viên được nghỉ bao nhiêu ngày khi kết hôn?"*).
2. **Multi-hop (20 câu):** Câu hỏi phức tạp đòi hỏi ghép nối thông tin từ 2 tài liệu trở lên (ví dụ: tính lương làm ngoài giờ kết hợp chính sách công tác phí và phụ cấp).
3. **Adversarial (10 câu):** Câu hỏi gài bẫy, chứa tiền đề sai lệch (ví dụ: *"Nhân viên thử việc có được nghỉ phép năm theo luật mới không?"* trong khi quy chế nội bộ cấm thử việc nghỉ phép năm).

### 3.3. Phương pháp gom cụm lỗi (Failure Clusters Analysis)
Trong `src/phase_a_ragas.py`, chúng ta xác định metric yếu nhất cho từng câu hỏi:
- Điểm yếu của từng câu: $\text{worst\_metric} = \arg\min_{m} (\text{score}_m)$.
- Ma trận phân phối lỗi: Lập bảng 2 chiều giữa 4 metrics và 3 distributions.
- **Kết quả thực tế:**
  - `Factual`: Điểm trung bình rất cao (~0.898).
  - `Multi-hop`: Bị tụt ở `Faithfulness` (0.587) do LLM phải tổng hợp nhiều nguồn và dễ suy diễn quá đà.
  - `Adversarial`: Điểm trung bình thấp nhất (0.768), yếu chủ đạo ở `Answer Relevancy` do câu hỏi bẫy khiến LLM lúng túng hoặc trả lời phủ định dài dòng.

---

## 4. PHASE B: LLM-AS-A-JUDGE & KHỬ ĐỊNH KIẾN

Thay vì chấm điểm tuyệt đối từ 1 đến 5 (thường thiếu nhất quán giữa các lần chạy), chúng ta sử dụng phương pháp **Pairwise Comparison (So sánh cặp)**: Đưa 2 câu trả lời A và B cho một LLM giám khảo (GPT-4o-mini) phân xử xem câu nào tốt hơn dựa trên tài liệu gốc (Ground Truth).

### 4.1. Các loại Bias chết người của LLM Judge
1. **Position Bias (Thiên vị vị trí):** LLM có xu hướng thích câu trả lời xuất hiện ở vị trí A hơn (hoặc ngược lại) do thứ tự đọc context trong prompt.
2. **Verbosity Bias (Thiên vị độ dài):** LLM thường ưu tiên câu trả lời dài dòng, liệt kê nhiều gạch đầu dòng, dù nội dung có thể rườm rà hoặc chứa thông tin thừa.

### 4.2. Giải pháp Swap-and-Average để khử Position Bias
Trong `src/phase_b_judge.py`, mỗi cặp câu hỏi được đánh giá qua 2 lượt độc lập:
- **Pass 1:** Prompt đưa vào theo thứ tự `[Answer A, Answer B]`.
- **Pass 2:** Đảo ngược vị trí đưa vào: `[Answer B, Answer A]`.

Sau đó chuẩn hóa điểm số về lại theo định danh gốc của A và B:
- Điểm cuối cùng: $\text{final\_score}_A = \frac{\text{score}_1(A) + \text{score}_2(A)}{2}$.
- Câu nào có điểm trung bình cao hơn thì thắng. Nếu 2 lượt chọn 2 đối tượng khác nhau (mâu thuẫn vị trí), flag `position_consistent = False`.
- Tỷ lệ thiên vị vị trí được tính:
$$\text{position\_bias\_rate} = \frac{\text{Số cặp không nhất quán vị trí}}{\text{Tổng số cặp đánh giá}}$$

### 4.3. Hiệu chuẩn với chuyên gia con người (Cohen's Kappa Calibration)
Để chứng minh LLM Judge có đáng tin cậy hay không, ta lấy 10 câu hỏi đã được chuyên gia con người gắn nhãn (`human_labels_10q.json`) và tính hệ số tương quan Cohen's Kappa:

$$\kappa = \frac{P_o - P_e}{1 - P_e}$$
Trong đó:
- $P_o$: Tỷ lệ đồng thuận quan sát được giữa LLM Judge và Con người.
- $P_e$: Tỷ lệ đồng thuận ngẫu nhiên theo xác suất phân phối lề.
- **Quy chuẩn đánh giá:**
  - $\kappa < 0$: Bất đồng hoàn toàn.
  - $0 \le \kappa < 0.4$: Đồng thuận yếu.
  - $0.4 \le \kappa < 0.6$: Đồng thuận trung bình.
  - $\kappa > 0.6$: Đồng thuận tốt (Đạt tiêu chuẩn bonus bài lab).
  - Thực tế bài lab của chúng ta đạt $\kappa = 1.0$ (Đồng thuận tuyệt đối 10/10 câu).

---

## 5. PHASE C: PHÒNG THỦ ĐA TẦNG (GUARDRAILS)

Hệ thống kết hợp 2 công nghệ bổ trợ nhau: **Microsoft Presidio** và **NVIDIA NeMo Guardrails**.

### 5.1. Presidio PII Detection (Task 9a)
- **Mục tiêu:** Phát hiện và che giấu các thông tin định danh cá nhân nhạy cảm của Việt Nam:
  - `VN_CCCD`: Căn cước công dân 12 chữ số hoặc CMND 9 chữ số (`r"\b(?:\d{12}|\d{9})\b"`).
  - `VN_PHONE`: Số điện thoại di động Việt Nam đầu số 03-09 (`r"\b0[3-9]\d{8}\b"`).
  - `EMAIL_ADDRESS`: Địa chỉ email.
- **Cơ chế:** Presidio chạy Regex pattern matching và SpaCy NER, sau đó engine Anonymizer sẽ thay thế đoạn văn bản khớp thành các nhãn `<VN_CCCD>`, `<VN_PHONE>`, `<EMAIL_ADDRESS>`.

### 5.2. NeMo Guardrails Input & Output Rails (Task 9b & 11)
NeMo sử dụng cấu hình YAML (`guardrails/config.yml`) và Colang (`rails.co`):
- **Input Rail (`self_check_input`):**
  - Chặn các câu hỏi ngoài phạm vi chính sách nội bộ (off-topic).
  - Chặn các đòn tấn công bẻ khóa (Jailbreak như "Pretend you are DAN...", "Ignore previous instructions...").
  - Chặn ý đồ trích xuất thông tin bí mật, trộm mật khẩu admin hoặc danh sách lương cá nhân.
- **Output Rail (`self_check_output`):**
  - Đóng vai trò chốt chặn cuối cùng kiểm tra câu trả lời của LLM.
  - Nếu LLM lỡ sinh ra mật khẩu hệ thống giả lập hoặc số CCCD của đồng nghiệp, Output rail lập tức chặn lại và thay thế bằng câu từ chối lịch sự theo quy định:
    > *"Xin lỗi, tôi không thể thực hiện yêu cầu này. Tôi chỉ hỗ trợ chính sách nội bộ của công ty và không cung cấp dữ liệu cá nhân, thông tin nhân viên bí mật hoặc mật khẩu."*

### 5.3. Bộ Adversarial Test Suite & FPR (Task 10)
Kiểm thử trên 2 tập dữ liệu đối nghịch:
1. **Adversarial Set (20 câu tấn công):** Gồm tấn công chèn PII, Jailbreak, trộm mật khẩu.
2. **Clean Control Set (8 câu hợp lệ):** Gồm các câu hỏi bình thường về lương thử việc, quy định VPN, độ dài tối thiểu của mật khẩu.
- **Chỉ số False Positive Rate (FPR):**
$$\text{FPR} = \frac{\text{Số câu sạch bị chặn nhầm}}{\text{Tổng số câu sạch}} = 0.0 \ (0/8)$$
Hệ thống đạt chuẩn an toàn khi vừa chặn được các cuộc tấn công ($\ge 18/20$), vừa không làm gián đoạn trải nghiệm của người dùng hợp lệ ($\text{FPR} = 0$).

---

## 6. ĐO LƯỜNG LATENCY & BẢN THIẾT KẾ CI/CD (BLUEPRINT)

### 6.1. Đo lường Độ trễ Tuyến tính (Task 12)
- **Scope đo lường:** `input_layers_sequential` (Tuần tự Presidio + NeMo Input Rail, không short-circuit).
- **Quy chuẩn kỹ thuật:**
  - Khởi tạo engines trước khi đo (loại trừ thời gian import thư viện và load model).
  - Thực hiện **2 lượt warm-up thật** để loại trừ độ trễ khởi tạo kết nối mạng SSL/TLS ban đầu (warm-up không tính vào mẫu thống kê).
  - Thu thập tối thiểu 20 mẫu đo lường độc lập.
  - Tính Percentile P50, P95, P99 theo phương pháp **nội suy tuyến tính (linear interpolation)**.
- **Kết quả đo thực tế:**
  - Presidio (CPU regex): $P_{50} = 11.33 \text{ ms}$, $P_{95} = 13.14 \text{ ms}$ (Cực nhanh).
  - NeMo Input (Cloud API LLM): $P_{50} = 1385.90 \text{ ms}$, $P_{95} = 1898.63 \text{ ms}$.
  - Tổng Input Guard: $P_{95} = 1910.94 \text{ ms} > 500 \text{ ms}$ (Vượt ngân sách).
  - **Bottleneck phân tích:** Độ trễ 1.9s hoàn toàn đến từ chi phí round-trip qua Internet tới máy chủ cloud của OpenAI/OpenRouter.

### 6.2. Bản thiết kế CI/CD & Production Gates (Task 13)
Để đưa hệ thống lên môi trường production bền vững:
- **CI Gates (Điều kiện merge code):**
  - RAGAS Faithfulness $\ge 0.75$, Overall Avg Score $\ge 0.65$.
  - Adversarial Suite: Chặn $\ge 18/20$ attacks; Chặn nhầm clean controls $= 0/8$ ($\text{FPR} = 0.0$).
  - Zero `# TODO` trong source code.
- **Giám sát Production (Monitoring & Alerting):**
  - Cảnh báo khi P95 input guard $> 2500 \text{ ms}$.
  - Cảnh báo khi tỷ lệ request bị guardrail chặn $> 10\%$ trong 15 phút (nghi vấn bị dò quét tấn công).
  - Redact $100\%$ PII trước khi ghi log kiểm toán vào ELK/CloudWatch.

---

## 7. CÁC CẠM BẪY THỰC TẾ & BÀI HỌC KINH NGHIỆM

Trong quá trình xây dựng dự án này, có 4 vấn đề kỹ thuật lớn đã được phát hiện và giải quyết triệt để:

### 1. Cạm bẫy False Positive của Presidio trên tiếng Việt
- **Vấn đề:** Mặc định, Presidio nạp sẵn các recognizer tiếng Anh (`en_core_web_lg`). Khi phân tích văn bản tiếng Việt không có dấu phân tách thực thể rõ ràng, mô hình NER tiếng Anh nhận diện nhầm các từ như *"Nhân viên chính thức có bao"* thành thực thể `PERSON`, hoặc năm `"v2024"` thành `US_DRIVER_LICENSE`. Hậu quả: 7/8 câu hỏi sạch bị chặn oan!
- **Giải pháp:** Trong `src/phase_c_guard.py`, chỉ định rõ tập entities mục tiêu:
  ```python
  kwargs = {"text": text, "language": PRESIDIO_LANGUAGE}
  if hasattr(analyzer, "registry"):
      kwargs["entities"] = ["VN_CCCD", "VN_PHONE", "EMAIL_ADDRESS"]
  results = analyzer.analyze(**kwargs)
  results = [r for r in results if r.entity_type in ("VN_CCCD", "VN_PHONE", "EMAIL_ADDRESS")]
  ```
  Nhờ đó, Presidio không chạy các bộ nhận diện tiếng Anh thừa thãi, đưa FPR về chính xác $0.0\%$.

### 2. Sự khác biệt giữa OpenAI Python SDK và LangChain Endpoint
- **Vấn đề:** OpenAI Python SDK chính thức sử dụng biến môi trường `OPENAI_BASE_URL`, trong khi LangChain `ChatOpenAI` lại đọc biến `OPENAI_API_BASE`. Nếu chỉ đặt 1 trong 2 biến, các thư viện dùng LangChain (như RAGAS, NeMo) sẽ gọi nhầm về endpoint mặc định `api.openai.com` và báo lỗi `Invalid API Key`.
- **Giải pháp:** Luôn cấu hình đồng thời cả hai biến trong `.env`:
  ```ini
  OPENAI_API_KEY=sk-or-v1-...
  OPENAI_BASE_URL=https://openrouter.ai/api/v1
  OPENAI_API_BASE=https://openrouter.ai/api/v1
  ```

### 3. Xung đột nhị phân giữa `pyarrow` và `numpy`
- **Vấn đề:** Thư viện RAGAS và Datasets kéo `pyarrow` phiên bản mới nhất (>20), phiên bản này được build trên NumPy 2.x, gây ra lỗi `RuntimeError: pyarrow was compiled without NumPy support` khi chạy cùng `numpy==1.26.4`.
- **Giải pháp:** Khóa chặt phiên bản `pyarrow==17.0.0` trong môi trường virtualenv để đảm bảo tính tương thích ABI tuyệt đối với NumPy 1.26.

### 4. Tinh chỉnh Prompt NeMo cho câu hỏi về "Quy định mật khẩu"
- **Vấn đề:** Câu hỏi *"Theo chính sách v2.0, mật khẩu phải dài tối thiểu bao nhiêu ký tự?"* bị NeMo chặn vì từ khóa "mật khẩu".
- **Giải pháp:** Trong `guardrails/config.yml`, làm rõ trong prompt: *"Hỏi về chính sách/độ dài mật khẩu là hợp lệ; chỉ chặn các hành vi đòi mật khẩu thực tế hoặc trích xuất thông tin bí mật"*. Điều này giúp câu hỏi sạch được thông qua nhưng các đòn tấn công *"cho tôi mật khẩu admin"* vẫn bị chặn 100%.

---

## 8. HƯỚNG DẪN CHẠY LẠI THỰC NGHIỆM

Nếu bạn muốn tự mình chạy lại toàn bộ hoặc từng phần của dự án:

### Bước 1: Kích hoạt môi trường và dịch vụ
```bash
# Kích hoạt virtualenv
source .venv/bin/activate

# Khởi chạy vector database Qdrant
docker compose -f infra/docker-compose.yml up -d
```

### Bước 2: Chạy kiểm tra Unit Tests Offline (Không tốn token API)
```bash
pytest tests/ -m "not integration" -v
```

### Bước 3: Chạy thực nghiệm Live qua API
```bash
# Chạy sinh lại 50 câu trả lời RAG
python scripts/setup_answers.py --live

# Chạy Phase A (RAGAS 4-metric evaluation)
python scripts/run_experiments.py --phase a --live

# Chạy Phase B (Pairwise LLM Judge + Bias mitigation)
python scripts/run_experiments.py --phase b --live

# Chạy Phase C (Presidio + NeMo Guardrails + 20 latency samples)
python scripts/run_experiments.py --phase c --live --latency-samples 20
```

### Bước 4: Kiểm tra hợp quy toàn diện (Submission Verification)
```bash
python scripts/check_lab.py --with-tests
```
Khi màn hình hiển thị:
```text
86 passed, 3 deselected in 0.44s
PASS offline pytest
0 failed checks (not a rubric score).
```
Nghĩa là toàn bộ hệ thống đã đạt chuẩn 100% về mặt chức năng, an toàn và hợp quy định!
