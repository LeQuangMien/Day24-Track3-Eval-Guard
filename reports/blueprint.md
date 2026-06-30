# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Lê Quang Miên (2A202600715)
**Ngày:** 30/06/2026

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~17ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~7ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 15.28 | 17.27 | 17.27 | <10ms |
| NeMo Input Rail | 6.76 | 7.33 | 7.33 | <300ms |
| RAG Pipeline | (chưa đo trong Phase C — pipeline đo riêng ở Day 18) | — | — | <2000ms |
| NeMo Output Rail | (đo cùng cơ chế NeMo Input Rail ở trên, ~tương đương) | — | — | <300ms |
| **Total Guard** | 20.93 | **24.46** | 24.46 | **<500ms** |

**Budget OK?** [x] Yes / [ ] No
**Comment:** Tổng latency guard stack (Presidio + NeMo input rail) chỉ 24.46ms P95, chưa tới 5% budget 500ms — còn rất nhiều dư địa. Bottleneck duy nhất đáng chú ý là Presidio (17.27ms), vượt mức tham chiếu lý tưởng <10ms trong comment gốc của code — nguyên nhân chủ yếu là chi phí khởi tạo NLP pipeline (spacy `en_core_web_lg`) cho mỗi lần `analyze()`, dù vẫn rất nhỏ so với ngân sách tổng. Nếu cần tối ưu thêm, có thể cache `AnalyzerEngine` instance ở mức singleton thay vì khởi tạo lại, hoặc giảm xuống các recognizer thực sự cần (VN_CCCD, VN_PHONE, EMAIL_ADDRESS) thay vì load toàn bộ registry mặc định.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | factual: 0.802, multi_hop: 0.656, adversarial: 0.739 (overall trung bình ~0.732) |
| Worst metric | faithfulness (đặc biệt yếu ở multi_hop: 13/20 câu fail faithfulness) |
| Dominant failure distribution | factual (theo count tổng: 21 fail-cases), nhưng multi_hop mới là distribution có avg_score thấp nhất (0.656) |
| Cohen's κ | 0.40 (moderate agreement, theo thang Landis-Koch) — judge khớp human label 8/10 câu |
| Adversarial pass rate | 20 / 20 (100%) |
| Guard P95 latency | 24.46 ms |

---

## Nhận xét & Cải tiến

Pipeline RAG hoạt động tốt nhất với câu hỏi factual đơn giản (avg_score 0.802) nhưng giảm rõ rệt ở multi_hop (0.656) — đặc biệt faithfulness yếu khi cần tổng hợp nhiều tài liệu hoặc tính toán (ví dụ cộng dồn phép năm theo thâm niên, tính phụ cấp), cho thấy model có xu hướng hallucinate khi phải suy luận nhiều bước thay vì chỉ trích xuất trực tiếp. Cohen's κ=0.40 giữa LLM judge và human cũng phản ánh đúng pattern này: hai câu judge đánh giá sai lệch với con người đều là multi_hop liên quan tính toán phức tạp, gợi ý rằng cả pipeline lẫn judge đều gặp khó khăn tương tự với loại câu hỏi này. Ngược lại, guardrail stack hoạt động vượt kỳ vọng — Presidio bắt đúng PII tiếng Việt (CCCD, SĐT) sau khi lọc bỏ false-positive từ NLP entity không tin cậy (PERSON dùng model tiếng Anh), và NeMo input rail sau khi cấu hình đúng embedding model multilingual (paraphrase-multilingual-MiniLM-L12-v2) đạt 100% pass rate trên adversarial suite với latency gần như không đáng kể so với budget. Nếu deploy production thực sự, ưu tiên cải tiến đầu tiên sẽ là multi_hop: thêm bước decomposition (tách câu hỏi multi-hop thành các sub-query đơn) trước khi retrieve, hoặc thêm một self-verification step sau khi sinh câu trả lời để kiểm tra lại các con số tính toán trước khi trả về người dùng — vì đây là nơi rủi ro sai chính sách thực tế cao nhất (vd như case tư vấn sai số ngày phép hoặc người phê duyệt).
