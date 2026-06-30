# LLM Judge Bias Report — Phase B

**Sinh viên:** Lê Quang Miên (2A202600715)
**Ngày:** 30/06/2026
**Judge model:** gpt-4o-mini

---

## 1. Pairwise Judge Results

*(Demo case chạy trong `phase_b_judge.py` main — so sánh 2 câu trả lời về số ngày phép năm)*

| # | Question (tóm tắt) | Winner | Reasoning tóm tắt |
|---|---|---|---|
| 1 | Nhân viên được nghỉ bao nhiêu ngày phép năm? (A: "15 ngày theo v2024 hiện hành" vs B: "12 ngày") | A | A nêu rõ số liệu đúng kèm căn cứ version hiện hành (v2024), trong khi B chỉ nêu số mà không có ngữ cảnh, dễ gây nhầm với số liệu v2023 đã hết hiệu lực |

*(Ghi chú: Phase B của bài lab này tập trung vào Cohen's κ — 10 câu human-labeled — làm bộ dữ liệu chính cho phân tích, thay vì chạy thêm nhiều cặp pairwise riêng lẻ ngoài demo case trên. Phần 3 dưới đây thể hiện đầy đủ kết quả judge trên 10 câu thực tế.)*

---

## 2. Swap-and-Average Results

*(Chạy swap_and_average() trên cặp câu hỏi ở demo case)*

| # | Pass 1 Winner | Pass 2 Winner | Final | Position Consistent? |
|---|---|---|---|---|
| 1 | A | A | A | Yes |

**Position bias rate:** 0% (0/1 case NOT consistent / tổng 1 case demo)

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu, 6 label=1, 4 label=0)
**Judge labels:** Chấm bằng `judge_correctness()` — LLM tự đánh giá model_answer đúng/sai dựa trên retrieved contexts (lấy từ `answers_50q.json` theo question_id) làm căn cứ, không đoán mò từ kiến thức nội tại.

| Question ID | Human Label | Judge Label | Agree? |
|---|---|---|---|
| 1 | 1 | 1 | Yes |
| 5 | 0 | 0 | Yes |
| 12 | 1 | 1 | Yes |
| 21 | 1 | 0 | No |
| 23 | 1 | 1 | Yes |
| 29 | 0 | 1 | No |
| 33 | 1 | 0 | No |
| 41 | 0 | 0 | Yes |
| 46 | 1 | 1 | Yes |
| 50 | 0 | 0 | Yes |

**Cohen's κ:** 0.40
**Interpretation:** moderate

---

## 4. Verbosity Bias

Trong các case có winner rõ ràng (không phải tie) — dựa trên demo case ở mục 1/2:
- A thắng + A dài hơn B: 1 / 1 cases
- B thắng + B dài hơn A: 0 / 1 cases
- **Verbosity bias rate:** 100% (1/1)

**Kết luận:** Với mẫu duy nhất hiện có (demo case), LLM chọn answer dài hơn — nhưng do n=1, kết quả này không đủ ý nghĩa thống kê để kết luận chắc chắn LLM có thiên vị độ dài hay không; con số 100% phản ánh đúng 1 case, không phải xu hướng tổng quát. Trong demo case cụ thể, answer A (dài hơn, có nêu version v2024) cũng là answer chính xác hơn về nội dung — nên ở đây verbosity và correctness trùng nhau, chưa thể tách bạch được liệu LLM chọn A vì nó dài hơn hay vì nó đúng hơn. Để đánh giá verbosity bias đáng tin cậy, cần chạy `swap_and_average()` trên một tập lớn hơn (ví dụ toàn bộ 10 câu human-labeled hoặc nhiều hơn) thay vì chỉ 1 cặp demo.

---

## 5. Nhận xét chung

> κ = 0.40 chưa đạt ngưỡng "substantial" (>0.6) mà chỉ ở mức "moderate" — LLM judge có độ tin cậy trung bình, không nên dùng làm gate tự động hoàn toàn thay thế con người mà cần kết hợp human-in-the-loop cho các case biên. Đáng chú ý là cả 3 câu judge sai (id 21, 29, 33) đều thuộc loại câu hỏi multi_hop có tính toán (cộng dồn phép năm theo thâm niên, tính phụ cấp, tính phạt trả chậm) — đúng pattern đã thấy ở Phase A, nơi multi_hop cũng là distribution yếu nhất về faithfulness (13/20 câu fail). Điều này cho thấy không chỉ pipeline RAG mà cả LLM judge cũng gặp khó khăn tương tự khi cần verify các phép tính nhiều bước — chính judge cũng có thể bỏ sót lỗi tính toán tinh vi (như case 29: model thiếu người phê duyệt thứ hai và tính sai công thức phạt, nhưng judge vẫn chấm đúng). Vì mẫu demo cho swap-and-average chỉ có 1 case (n=1), chưa đủ để kết luận chắc chắn về position bias hay verbosity bias ở quy mô lớn — cần mở rộng sang chạy trên nhiều cặp câu trả lời thực tế (ví dụ so sánh pipeline hiện tại với baseline naive) để có kết luận đáng tin cậy hơn. Trong môi trường production, khuyến nghị dùng LLM judge như một lớp sàng lọc sơ bộ (flag các case nghi ngờ) thay vì quyết định cuối cùng, đặc biệt ưu tiên human review cho mọi câu trả lời thuộc nhóm multi_hop/tính toán — nơi cả model lẫn judge đều có rủi ro sai cao nhất.
