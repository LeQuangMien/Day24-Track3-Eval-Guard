# Failure Cluster Analysis — Phase A

**Sinh viên:** Lê Quang Miên (2A202600715)
**Ngày:** 30/06/2026

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.775 | 0.437 | 0.767 |
| answer_relevancy | 0.708 | 0.534 | 0.605 |
| context_precision | 0.925 | 0.946 | 0.967 |
| context_recall | 0.808 | 0.708 | 0.617 |
| **avg_score** | **0.802** | **0.656** | **0.739** |

---

## 2. Bottom 10 Questions

| Rank | Distribution | Question | avg_score | worst_metric |
|---|---|---|---|---|
| 1 | multi_hop | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | 0.125 | faithfulness |
| 2 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.333 | faithfulness |
| 3 | factual | Thông tin lương thuộc cấp độ phân loại dữ liệu nào? | 0.333 | faithfulness |
| 4 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.375 | answer_relevancy |
| 5 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.375 | faithfulness |
| 6 | adversarial | Thâm niên bao nhiêu năm thì được cộng thêm ngày phép? | 0.417 | faithfulness |
| 7 | factual | Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu cho nhân viên? | 0.467 | faithfulness |
| 8 | factual | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | 0.500 | faithfulness |
| 9 | multi_hop | Nhân viên tạm ứng 8 triệu, chưa thanh toán sau 30 ngày (quá hạn 15 ngày). Ai phê duyệt khoản này và phí phạt là bao nhiêu? | 0.500 | faithfulness |
| 10 | multi_hop | Nhân viên mới đang trong tháng thứ 3 thử việc có BẮT BUỘC tham dự buổi đào tạo nội bộ của phòng ban không? Tại sao? | 0.500 | faithfulness |

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 6 | 13 | 3 | 22 |
| answer_relevancy | 10 | 4 | 2 | 16 |
| context_precision | 2 | 0 | 0 | 2 |
| context_recall | 2 | 3 | 5 | 10 |

---

## 4. Dominant Failure Analysis

**Dominant distribution:** factual (theo tổng count, vì có 20 câu nên count tự nhiên cao hơn — nhưng đáng chú ý hơn là **multi_hop có avg_score thấp nhất tuyệt đối: 0.656**)
**Dominant metric:** faithfulness (22/50 câu, ~44% tổng số câu có faithfulness là điểm yếu nhất)

**Lý do phân tích:**

> Faithfulness yếu nhất ở multi_hop (13/20 câu, 65%) vì các câu hỏi loại này đòi hỏi tổng hợp thông tin từ nhiều tài liệu khác nhau (ví dụ cộng dồn ngày phép theo thâm niên, tính phụ cấp theo cấp bậc, đối chiếu chính sách giữa nhiều version) — đây là tác vụ suy luận nhiều bước mà LLM dễ "bịa" số liệu trung gian thay vì trích xuất trực tiếp từ context như câu factual. Ngược lại, answer_relevancy là điểm yếu chủ đạo ở factual (10/20 câu) — có thể do câu trả lời đúng về mặt nội dung nhưng lệch trọng tâm câu hỏi (ví dụ trả lời thừa/thiếu chi tiết so với những gì người dùng hỏi), khác hẳn cơ chế lỗi của multi_hop. context_precision gần như không phải vấn đề (chỉ 2/50 câu) cho thấy retrieval pipeline (M2 Search + M3 Rerank) hoạt động khá tốt ở việc lọc nhiễu, vấn đề chính nằm ở khâu generation/reasoning chứ không phải retrieval.

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating | Tighten system prompt, lower temperature; với multi_hop cụ thể: thêm bước decomposition tách câu hỏi thành các sub-query đơn lẻ trước khi retrieve, rồi tổng hợp kết quả bằng một bước tính toán tường minh (không để LLM tự suy luận số liệu) thay vì sinh câu trả lời trực tiếp từ toàn bộ context gộp |
| context_recall | Missing relevant chunks | Improve chunking or add BM25; với adversarial cụ thể (context_recall thấp nhất ở adversarial: 0.617): cần đảm bảo retrieval lấy đủ cả phiên bản cũ lẫn mới của tài liệu khi có version conflict, để model có đủ thông tin phân biệt thay vì chỉ thấy một phần |
| context_precision | Too many irrelevant chunks | Add reranking or metadata filter — hiện tại context_precision đã khá cao (0.92–0.97) nên đây không phải ưu tiên, có thể giữ nguyên M3 Rerank hiện tại |
| answer_relevancy | Answer doesn't match question | Improve prompt template — đặc biệt với factual, có thể do prompt chưa hướng dẫn rõ độ dài/format câu trả lời mong muốn, dẫn đến model trả lời lạc trọng tâm dù context đã đúng |

---

## 6. Nhận xét về Adversarial Distribution

> Adversarial có avg_score 0.739, nằm giữa factual (0.802, cao nhất) và multi_hop (0.656, thấp nhất) — không phải distribution tệ nhất như kỳ vọng ban đầu, nhưng vẫn thấp hơn factual đáng kể (đạt bonus rubric: adversarial < factual, đúng như thiết kế của bộ test). Điểm đáng chú ý nhất là context_recall của adversarial chỉ 0.617 — thấp nhất trong cả 3 distribution — cho thấy pipeline gặp khó khăn thực sự khi cần phân biệt và lấy đúng context giữa các phiên bản chính sách xung đột (v2023 vs v2024), đúng như mục tiêu thiết kế của bộ test này (negation traps và version conflicts). Trong bottom 10, có 2 câu thuộc adversarial: #2 (rank 2, avg=0.333, về VPN cá nhân — model có thể đã không nhận ra chính sách VPN v1.3 cấm dùng VPN cá nhân) và #6 (rank 6, avg=0.417, về điều kiện cộng ngày phép theo thâm niên — khả năng liên quan đến việc lẫn lộn giữa v2023 và v2024 của chính sách nghỉ phép năm). Cả hai đều có faithfulness là worst_metric, củng cố kết luận rằng pipeline có xu hướng hallucinate hoặc trộn lẫn thông tin giữa các version khi không được hướng dẫn rõ ràng để ưu tiên version hiện hành.
