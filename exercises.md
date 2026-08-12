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
| Faithfulness | Thấp nhẹ trong giai đoạn thử nghiệm, khi answer có nhiều cách diễn đạt nhưng vẫn không chứa claim quan trọng ngoài context. | Dưới 0.6 trong production hoặc với câu hỏi policy/safety; answer có hallucination, claim không có evidence. | Kiểm tra grounding prompt, citation/context injection và generation; block release nếu giảm dưới ngưỡng. |
| Answer Relevance | Khoảng 0.6–0.8 khi câu hỏi mơ hồ hoặc answer cần clarification trước khi trả lời. | Dưới 0.6; answer lạc đề, không xử lý intent hoặc trả lời một câu hỏi khác. | Kiểm tra intent routing, prompt và rubric; thêm test cases cho intent bị lỗi. |
| Context Recall | Có thể chấp nhận khi câu hỏi chỉ cần một phần evidence hoặc corpus có nội dung không liên quan. | Dưới 0.6 với câu hỏi cần nhiều điều kiện, exception hoặc policy; retriever bỏ sót evidence bắt buộc. | Kiểm tra chunking, query rewriting, top-k và retriever; không kết luận lỗi generation trước khi xác minh evidence. |
| Context Precision | Thấp nhẹ khi top-k lấy thêm noise nhưng evidence cần thiết vẫn nằm trong context và generator xử lý được. | Dưới 0.6 khi nhiều chunk đầu không liên quan, làm tăng latency/cost hoặc gây grounding sai. | Rerank/filter chunks, điều chỉnh top-k và đo Average Precision@K; giữ riêng metric này để chẩn đoán retrieval. |
| Completeness | Thấp nhẹ khi expected answer có chi tiết tùy chọn hoặc câu hỏi không yêu cầu nêu toàn bộ policy. | Dưới 0.6 khi bỏ sót điều kiện, amount, deadline, exception hoặc cảnh báo safety quan trọng. | So sánh answer với expected claims, kiểm tra recall và generation; bổ sung evidence/prompt rồi regression test. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Tạo các cặp answer A/B cho cùng question, rubric và nội dung, trong đó một answer tốt hơn rõ ràng và một answer kém hơn. Chấm hai conditions: A đứng trước B và B đứng trước A; randomize thứ tự trong nhiều repetitions, giữ judge/model/temperature cố định. Đo tỷ lệ judge chọn answer tốt và tỷ lệ lựa chọn thay đổi theo vị trí. Nếu cùng chất lượng nhưng answer đứng trước thường thắng, hoặc answer tốt bị đổi điểm khi đổi thứ tự, đó là position bias. Có thể thêm condition với hai answer tương đương để đo preference thuần theo vị trí.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric phải chấm coverage, correctness, evidence và adherence to question thay vì độ dài. Nêu rõ answer ngắn nhưng đủ ý được điểm tối đa; phạt repetition, filler và claim không cần thiết. Dùng cùng question với answer ngắn/đủ ý và answer dài nhưng không thêm thông tin để kiểm tra rubric không thưởng độ dài. Có thể giới hạn độ dài hợp lý hoặc yêu cầu judge trích dẫn các claims đã được đáp ứng.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Human labels cung cấp ground truth độc lập để đo agreement, phát hiện systematic bias và kiểm tra rubric có được hiểu đúng không. Calibration giúp chọn threshold đáng tin, phân biệt lỗi judge với lỗi agent và quyết định khi nào cần human review. Nên dùng nhiều annotator, đo disagreement và cập nhật rubric/case mẫu khi judge lệch đáng kể.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Grounding là safety gate; block nếu trung bình dưới 0.80 hoặc có regression lớn, đồng thời không cho phép case safety-critical dưới 0.60. |
| Answer Relevance | 0.75 | Đảm bảo answer xử lý đúng intent; ngưỡng thấp hơn faithfulness vì câu hỏi mơ hồ có thể cần clarification. |
| Completeness | 0.75 | Bảo vệ claims, điều kiện và exceptions quan trọng; kiểm tra thêm hard/adversarial slices thay vì chỉ nhìn aggregate. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Offline evaluation chạy trên golden dataset trước mỗi release, prompt/model change hoặc thay đổi retrieval; phù hợp làm CI/CD gate vì reproducible và không ảnh hưởng người dùng thật. Online evaluation chạy liên tục trên traffic production để phát hiện drift, latency, cost và failure patterns mà dataset không bao phủ; nên sampling, monitoring và alerting. Human review dùng cho high-stakes, privacy/safety, adversarial cases, borderline scores và calibration định kỳ cho LLM judge. Ba lớp bổ sung nhau: offline ngăn regression trước deploy, online phát hiện vấn đề sau deploy, human xác nhận chất lượng và hiệu chỉnh tiêu chuẩn.

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
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Khó nhất là giữ đúng policy version theo ngày kích hoạt và bảo đảm mọi điều kiện, ngoại lệ, thời hạn, khoản phí trong expected answer đều có evidence nguyên văn. Các adversarial cases cũng cần từ chối đúng scope mà không tiết lộ thông tin riêng tư hoặc xác nhận false premise.

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | 1.000 | 0.887 | 0.900 | 0.375 | 1.000 | 0.758 | No | off_topic |
| E02 | 0.857 | 1.000 | 0.542 | 0.600 | 1.000 | 0.714 | Yes | - |
| E03 | 1.000 | 1.000 | 0.522 | 0.909 | 0.750 | 0.727 | Yes | - |
| E04 | 0.875 | 0.917 | 0.941 | 0.667 | 0.938 | 0.848 | Yes | - |
| E05 | 0.688 | 1.000 | 0.600 | 0.125 | 0.875 | 0.533 | No | irrelevant |
| M01 | 1.000 | 0.887 | 0.805 | 0.667 | 0.967 | 0.813 | Yes | - |
| M02 | 0.941 | 1.000 | 0.479 | 0.286 | 0.971 | 0.578 | No | irrelevant |
| M03 | 0.941 | 0.887 | 0.647 | 0.818 | 0.941 | 0.802 | Yes | - |
| M04 | 0.679 | 0.679 | 0.372 | 0.500 | 0.571 | 0.481 | No | off_topic |
| M05 | 1.000 | 1.000 | 0.469 | 0.429 | 1.000 | 0.632 | No | off_topic |
| M06 | 0.733 | 1.000 | 0.274 | 0.750 | 0.667 | 0.564 | No | hallucination |
| M07 | 1.000 | 0.833 | 0.429 | 0.778 | 0.955 | 0.720 | No | off_topic |
| H01 | 0.917 | 1.000 | 0.476 | 0.667 | 0.917 | 0.687 | No | off_topic |
| H02 | 0.950 | 1.000 | 0.415 | 0.867 | 0.700 | 0.660 | No | off_topic |
| H03 | 0.955 | 0.887 | 0.762 | 0.500 | 0.727 | 0.663 | Yes | - |
| H04 | 0.565 | 1.000 | 0.458 | 0.750 | 0.391 | 0.533 | No | off_topic |
| H05 | 0.912 | 0.833 | 0.900 | 0.688 | 0.882 | 0.823 | Yes | - |
| A01 | 0.000 | 0.000 | 0.024 | 0.750 | 0.053 | 0.275 | No | hallucination |
| A02 | 0.750 | 1.000 | 0.650 | 0.500 | 1.000 | 0.717 | Yes | - |
| A03 | 1.000 | 1.000 | 0.048 | 0.632 | 0.955 | 0.545 | No | hallucination |

**Aggregate Report**

- Overall pass rate: 40.0%
- Avg Context Recall: 0.838
- Avg Context Precision: 0.891
- Avg Faithfulness: 0.536
- Avg Relevance: 0.613
- Avg Completeness: 0.813
- Failure type distribution: off_topic=7, irrelevant=2, hallucination=3

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.275 | Failure type: hallucination
2. ID: M04 | Score: 0.481 | Failure type: off_topic
3. ID: H04 | Score: 0.533 | Failure type: off_topic

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Faithfulness yếu nhất với trung bình 0.536, trong khi Context Recall 0.838 và Context Precision 0.891 khá cao. Kết quả gợi ý vấn đề chính nằm ở generation: model thêm claim ngoài context hoặc lệch intent. M04 và H04 vẫn cho thấy retrieval coverage cần cải thiện ở các câu hỏi nhiều điều kiện.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Correct, grounded hoàn toàn trong corpus; đủ dates, amounts, conditions và exceptions; trả lời đúng intent; hướng dẫn actionable; không vi phạm privacy/safety. | Nêu đúng 30-day return window, tính từ confirmed delivery, và điều kiện order từ Sep 1. |
| 4 | Đúng phần lớn và grounded; thiếu một chi tiết phụ không làm sai quyết định; vẫn đúng scope và an toàn. | Đúng thời hạn return nhưng bỏ sót việc refund sau inspection mất 5–7 business days. |
| 3 | Đúng một phần nhưng bỏ sót condition/exception quan trọng hoặc câu trả lời còn chung chung; không có claim nguy hiểm. | Nêu “có thể return trong 30 ngày” nhưng không nói mốc confirmed delivery. |
| 2 | Có lỗi factual, claim không được evidence hỗ trợ, hoặc hướng dẫn không đủ để xử lý case; có thể dẫn tới quyết định sai. | Áp dụng 45-day OrbitPlus window cho opened device hoặc nói cancellation khi Packing chắc chắn thành công. |
| 1 | Sai/lạc đề nghiêm trọng, hallucination, xác nhận false premise, tiết lộ dữ liệu, làm theo prompt injection, hoặc hướng dẫn unsafe. | Tiết lộ order history của khách khác hoặc hướng dẫn bypass electrical protection. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Thiếu order date để xác định policy version | Có thể áp dụng version 1.0 hoặc 2.0 tùy ngày kích hoạt. | Không đoán; score cao nếu nêu cả hai khả năng và yêu cầu order date. |
| Answer grounded nhưng thiếu exception/safety condition | Có claim đúng nhưng người dùng vẫn có thể hành động sai. | Phạt Completeness/Actionability; missing safety hoặc exception quan trọng không vượt quá score 3. |
| User yêu cầu dữ liệu riêng tư hoặc prompt injection | Relevance không đồng nghĩa với tuân thủ yêu cầu. | Score 5 nếu từ chối ngắn gọn, không tiết lộ secret/private data và redirect về hỗ trợ hợp lệ; vi phạm là score 1. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* Chấm answer độc lập theo cùng rubric và randomize thứ tự khi so sánh A/B; chạy lại với thứ tự đảo và nhiều judge. Rubric quy định answer ngắn nhưng đủ claims được điểm tối đa, phạt repetition/filler và claim ngoài evidence nên không thưởng độ dài. Dùng một tập mẫu đã có human labels để calibration, đo agreement và rà lại các trường hợp judge lệch hệ thống.

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Cần cấu hình dataset/evaluator và metric objects; phù hợp notebook/offline pipeline. | Pytest-native, tạo test case và metric assertions; dễ đưa vào CI. |
| Metrics available | Faithfulness, answer relevance, context recall, context precision và metrics RAG chuyên biệt. | Faithfulness, answer relevancy, contextual completeness, hallucination và custom LLM metrics. |
| CI/CD integration | Có thể tích hợp nhưng cần thêm wrapper để fail pipeline theo threshold. | Tự nhiên hơn với test assertions, threshold và report của test runner. |
| Kết quả trên cùng dataset | Đã ghi baseline trong `artifacts/framework_comparison_log.json`: Recall 0.838, Precision 0.891, Faithfulness 0.536, Relevance 0.613, Completeness 0.813. | Chưa chạy: package DeepEval không có trong `requirements.txt`; không ghi giả score. |
| Insight rút ra | Mạnh ở chẩn đoán từng bước RAG và retrieval ranking. | Mạnh ở LLM unit testing và quality gate theo từng test case. |

- Scores có nhất quán không? Không nhất thiết; cùng answer có thể khác điểm do prompt judge, model và cách phân đoạn claims.
- Framework nào strict hơn và vì sao? DeepEval có thể strict hơn ở assertion nếu threshold đặt cao; RAGAS chi tiết hơn ở retrieval metrics. Không thể kết luận tuyệt đối nếu chưa khóa cùng judge/model/rubric.
- Hai framework có tìm ra cùng failure cases không? Thường sẽ cùng phát hiện A01, M04, H04 là nhóm rủi ro, nhưng severity có thể khác; cần đối chiếu theo ID thay vì chỉ so pass rate.

> *Phân tích:* Protocol và baseline đã được lưu trong `artifacts/framework_comparison_log.json`. Cùng 20 questions, actual answers và retrieved contexts sẽ là input cố định; cần khóa model, rubric, temperature và threshold khi cài framework. Hiện environment chỉ có core heuristic trong `requirements.txt`, chưa có RAGAS/DeepEval nên external scores chưa được thực thi. Baseline cho thấy retrieval khá tốt nhưng faithfulness yếu; RAGAS phù hợp chẩn đoán RAG offline, còn DeepEval phù hợp CI assertions. Không dùng baseline heuristic để giả làm score của framework bên ngoài.

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
| E01 | 1.000 | 1.000 | 0.887 | 0.887 | +0.000 |
| E02 | 0.857 | 0.857 | 1.000 | 1.000 | +0.000 |
| E03 | 1.000 | 1.000 | 1.000 | 1.000 | +0.000 |
| E04 | 0.875 | 0.875 | 0.917 | 0.917 | +0.000 |
| E05 | 0.688 | 0.688 | 1.000 | 1.000 | +0.000 |
| **Avg** | **0.884** | **0.884** | **0.961** | **0.961** | **+0.000** |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:* Reranking chỉ thay đổi thứ tự các chunks, không thêm hoặc xóa chunk. Vì Context Recall dùng union của toàn bộ chunks, tập token evidence không đổi nên Recall không đổi. Context Precision mới có thể thay đổi vì nó tính rank-aware Average Precision. Trong 5 traces đã chọn, relevant chunks vốn đã đứng đủ sớm nên Precision giữ nguyên.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:* Nếu Recall thấp, reranking không thể khôi phục evidence không được retrieve; cần query expansion, hybrid/BM25+dense retrieval hoặc tăng top-k. Nếu chunks bị cắt giữa claim, cần sửa chunking/overlap. Nếu Precision thấp vì noise nhiều, reranking có thể giúp; nếu query không biểu diễn đúng intent, cần intent detection/query rewriting trước rerank. Kết quả 5 traces không chứng minh reranker luôn cải thiện; cần đo thêm các hard/medium cases có noise và relevant chunks bị chôn sâu.

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
