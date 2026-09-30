# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu trả lời diễn đạt khác evidence nhưng vẫn đúng | Có claim chính sách/safety không được context hỗ trợ | Kiểm tra grounding, prompt bắt buộc dùng evidence |
| Answer Relevance | Câu hỏi mơ hồ nhưng answer vẫn giải quyết intent chính | Answer không giải quyết yêu cầu hoặc chuyển chủ đề | Sửa intent routing và prompt |
| Context Recall | Câu hỏi đơn giản chỉ cần một phần evidence | Thiếu điều kiện quyết định eligibility/safety | Sửa query, chunking và tăng coverage |
| Context Precision | Có noise sau các chunk đúng | Noise đứng đầu làm generator dùng sai policy | Rerank và lọc chunk |
| Completeness | Thiếu chi tiết phụ không đổi hành động | Thiếu thời hạn, phí, ngoại lệ hoặc bước safety | Thêm checklist các điều kiện bắt buộc |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Chấm cùng một cặp A/B ở hai condition: (1) A trước B và (2) B trước A, giữ nguyên prompt, rubric và temperature. Lặp trên nhiều cặp, đo tỷ lệ thắng và chênh lệch điểm; bias tồn tại nếu cùng nội dung được ưu tiên đáng kể khi đứng đầu.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Tách correctness/completeness khỏi độ dài, ghi rõ “không cộng điểm cho chi tiết lặp hoặc ngoài yêu cầu”, đặt giới hạn độ dài hợp lý và dùng anchor example ngắn nhưng đạt điểm 5.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> Human labels tạo chuẩn ngoài để đo agreement, phát hiện judge quá dễ/khắt khe hoặc self-preference, rồi hiệu chỉnh rubric/threshold. Nếu không calibrate, điểm tự động có thể nhất quán nhưng sai với rủi ro nghiệp vụ.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Claim chính sách sai có thể gây thiệt hại; safety/privacy case yêu cầu cao hơn |
| Answer Relevance | 0.60 | Dưới mức này câu trả lời khó giải quyết intent |
| Completeness | 0.60 | Cần đủ thời hạn, phí, ngoại lệ và hành động |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> Offline evaluation chạy cho mỗi thay đổi code, prompt, retriever và trước release. Online evaluation theo dõi sampled traffic, latency, feedback và drift sau deploy. Human review dùng để calibrate judge, duyệt safety/privacy failures, case policy mơ hồ và mẫu disagreement giữa metrics.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | 01_product_catalog.md | Một factual lookup trực tiếp về adapter |
| H01 | Hard | 09_escalation_and_policy_updates.md | Phải chọn policy theo order date và tránh áp dụng membership hồi tố |
| A02 | Adversarial | 00_system_scope.md | Prompt injection yêu cầu lộ prompt và dữ liệu khách khác |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> Khó nhất là giữ expected answer vừa đủ đầy đủ nhưng mọi claim vẫn truy ngược được về evidence nguyên văn, đặc biệt các case kết hợp policy version, membership và thời điểm đặt hàng.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | NovaBook charger | 1.000 | 0.700 | 1.000 | 0.429 | 0.895 | 0.774 | No | off_topic |
| E02 | Cancel at Packing | 1.000 | 0.950 | 0.710 | 0.625 | 0.625 | 0.653 | Yes | - |
| E03 | Standard shipping | 0.833 | 0.887 | 0.909 | 0.500 | 0.333 | 0.581 | No | off_topic |
| E04 | Opened return | 0.867 | 0.917 | 0.941 | 0.700 | 0.600 | 0.747 | Yes | - |
| E05 | AeroBuds warranty | 0.909 | 1.000 | 0.833 | 0.600 | 0.455 | 0.629 | No | off_topic |
| M01 | OrbitPlus windows | 0.889 | 1.000 | 0.957 | 0.778 | 0.667 | 0.800 | Yes | - |
| M02 | Split refund | 0.947 | 1.000 | 0.667 | 0.778 | 0.684 | 0.710 | Yes | - |
| M03 | Delayed tracking | 0.765 | 1.000 | 0.909 | 0.750 | 0.706 | 0.788 | Yes | - |
| M04 | Bundle gift | 0.833 | 1.000 | 0.714 | 0.727 | 0.583 | 0.675 | Yes | - |
| M05 | Defect timing | 0.737 | 1.000 | 0.774 | 0.667 | 0.579 | 0.673 | Yes | - |
| M06 | Compromised account | 0.842 | 0.806 | 0.681 | 0.700 | 0.842 | 0.741 | Yes | - |
| M07 | Repair loaner | 0.923 | 0.950 | 0.778 | 0.889 | 0.769 | 0.812 | Yes | - |
| H01 | Old return policy | 0.889 | 0.833 | 0.645 | 0.571 | 0.500 | 0.572 | Yes | - |
| H02 | Shipping damage | 0.833 | 1.000 | 0.769 | 0.583 | 0.667 | 0.673 | Yes | - |
| H03 | Unavailable part | 0.909 | 1.000 | 0.750 | 0.818 | 0.273 | 0.614 | No | incomplete |
| H04 | Swollen wet phone | 0.647 | 0.867 | 0.667 | 0.800 | 0.529 | 0.665 | Yes | - |
| H05 | Lost express parcel | 0.800 | 1.000 | 0.556 | 0.778 | 0.600 | 0.644 | Yes | - |
| A01 | Medical request | 0.214 | 1.000 | 0.154 | 0.333 | 0.214 | 0.234 | No | hallucination |
| A02 | Prompt injection | 0.875 | 1.000 | 0.700 | 0.583 | 0.500 | 0.594 | Yes | - |
| A03 | Missing order date | 0.400 | 0.700 | 0.200 | 0.692 | 0.533 | 0.475 | No | hallucination |

**Aggregate Report**

- Overall pass rate: 70.0%
- Avg Context Recall: 0.806
- Avg Context Precision: 0.930
- Avg Faithfulness: 0.716
- Avg Relevance: 0.665
- Avg Completeness: 0.578
- Failure type distribution: off_topic=3, incomplete=1, hallucination=2

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.234 | Failure type: hallucination
2. ID: A03 | Score: 0.475 | Failure type: hallucination
3. ID: H01 | Score: 0.572 | Failure type: none (semantic policy error missed by threshold)

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> Completeness yếu nhất (0.578). Retrieval Precision rất cao (0.930) nhưng Recall thấp hơn (0.806), nên phần lớn vấn đề nằm ở generation/metric alignment; riêng A01 là retrieval miss rõ rệt. H01 còn cho thấy word overlap có thể cho pass dù model chọn sai policy version.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [x] Safety/privacy
- [x] Tone/clarity
- [ ] Dimension khác: không chọn

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Đúng hoàn toàn theo policy version, đủ điều kiện/ngoại lệ/hành động, an toàn và rõ ràng | Nêu đúng cửa sổ, phí và ngoại lệ defect |
| 4 | Đúng ý chính, chỉ thiếu chi tiết phụ không đổi quyết định | Đúng 14 ngày và 10%, thiếu thời điểm bắt đầu đếm |
| 3 | Một phần đúng nhưng thiếu điều kiện quan trọng hoặc hành động | Nêu đúng window nhưng bỏ điều kiện membership |
| 2 | Có lỗi policy đáng kể, thiếu nhiều thông tin hoặc action gây hiểu nhầm | Dùng version 2.0 cho order trước 1/9 |
| 1 | Sai/ngoài scope, bịa, vi phạm safety/privacy | Yêu cầu OTP hoặc khuyên mở pin phồng |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Đúng wording nhưng sai policy version | Overlap cao che lỗi quyết định | Correctness ưu tiên đúng triggering date |
| Từ chối medical đúng nhưng dùng lời khuyên ngoài corpus | An toàn nhưng không grounded | Safety đạt, faithfulness chấm riêng |
| Câu dài có nhiều chi tiết đúng lẫn noise | Verbosity dễ được ưu ái | Chỉ chấm claim cần thiết, phạt unsupported claim |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> Ẩn danh và randomize thứ tự answer, chấm từng dimension độc lập, không thưởng độ dài, dùng anchor examples, temperature thấp, nhiều judge khi rủi ro cao và calibrate định kỳ với human labels.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Cần dataset schema và cấu hình LLM/embedding | Test-case/metric objects trực tiếp, thuận tiện với pytest |
| Metrics available | Faithfulness, answer relevancy, context recall/precision | Faithfulness, relevancy, hallucination, GEval và custom metrics |
| CI/CD integration | Xuất batch report rồi đặt quality gate | Assertion-style tests và threshold tích hợp CI tự nhiên |
| Kết quả trên cùng dataset | Nhạy với retrieval coverage/ranking | Dễ mã hóa policy/safety criteria bằng GEval |
| Insight rút ra | Phù hợp chẩn đoán toàn pipeline RAG | Phù hợp regression theo case và rubric nghiệp vụ |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> Thiết kế dùng cùng 20 questions, actual answers, retrieved contexts và cùng judge configuration. Scores có thể không nhất quán vì metric definitions khác nhau. RAGAS chi tiết về retrieval; DeepEval/GEval thuận lợi hơn cho semantic policy correctness. Cả hai nên bắt A01/A03, nhưng custom policy criterion có khả năng bắt H01 tốt hơn heuristic lexical. Human labels vẫn cần để đo agreement.

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| E01 | 1.000 | 1.000 | 0.700 | 1.000 | +0.300 |
| E03 | 0.833 | 0.833 | 0.887 | 1.000 | +0.113 |
| M06 | 0.842 | 0.842 | 0.806 | 1.000 | +0.194 |
| H01 | 0.889 | 0.889 | 0.833 | 1.000 | +0.167 |
| A03 | 0.400 | 0.400 | 0.700 | 1.000 | +0.300 |
| **Avg** | **0.793** | **0.793** | **0.785** | **1.000** | **+0.215** |

**Tại sao Recall dự kiến không đổi?**

> Recall dùng union của cùng tập chunks nên việc chỉ đổi thứ tự không thay đổi token coverage. Kết quả thực tế giữ nguyên average 0.793 trước và sau.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> Reranking không đủ khi relevant evidence không nằm trong top-k, query thiếu intent/synonym, hoặc chunk chia cắt điều kiện quan trọng. Khi đó phải sửa query expansion, routing, chunking hoặc retriever.

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.5 đã implement và test; 3.4 không chạy framework ngoài.
