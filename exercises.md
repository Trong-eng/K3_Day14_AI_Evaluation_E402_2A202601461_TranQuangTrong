# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Có thể thấp trong câu trả lời từ chối một câu hỏi ngoài scope hoặc câu hỏi mơ hồ cần chuyển đến đúng văn phòng; khi đó phải kiểm tra failure type thay vì kết luận ngay là hallucination. | Câu trả lời chứa policy, ngày, khoản phí hoặc điều kiện không xuất hiện trong context; đặc biệt nghiêm trọng với học phí, học bổng, deadline và quyền riêng tư. | Kiểm tra claim không có evidence, tăng grounding guardrail/citation, và human-review các case rủi ro cao. Nếu retrieval tốt mà score vẫn thấp thì ưu tiên sửa prompt/generator. |
| Answer Relevance | Có thể thấp khi người dùng hỏi quá mơ hồ hoặc ngoài scope và assistant trả lời giới hạn scope một cách an toàn; cũng cần lưu ý heuristic word-overlap có thể phạt paraphrase. | Câu trả lời đi sang chủ đề khác, không giải quyết intent chính, hoặc trả lời một câu hỏi khác dù context đúng. | Kiểm tra intent routing và question-answer overlap; sửa prompt/routing, thêm query paraphrase và test các câu hỏi mơ hồ. |
| Context Recall | Có thể thấp do expected answer dùng từ khác với retrieved chunk nhưng chunk vẫn chứa đủ ý nghĩa; cần kiểm tra thủ công vì đây là heuristic lexical. | Retriever bỏ sót evidence bắt buộc như deadline, amount, exception hoặc điều kiện eligibility, khiến generator không thể trả lời đủ. | So sánh gold contexts với retrieved chunks. Nếu Recall thấp cùng Completeness thấp, ưu tiên sửa query expansion, chunking hoặc retriever. |
| Context Precision | Có thể thấp khi top-k cố ý rộng để tăng recall nhưng answer vẫn grounded và đầy đủ; đây là vấn đề ranking cần monitor, chưa chắc là blocker. | Chunks liên quan đứng rất muộn hoặc bị noise lấn át, làm model chọn nhầm evidence hoặc sinh câu trả lời sai. | Kiểm tra ranking và noise trong top-k; thử reranking, điều chỉnh top-k/chunking, đồng thời theo dõi Faithfulness và Completeness. |
| Completeness | Có thể thấp nếu câu hỏi chỉ yêu cầu một phần thông tin, hoặc case adversarial cần từ chối thay vì cung cấp một expected answer dài; phải đối chiếu rubric và expected answer. | Bỏ sót thông tin bắt buộc như ngày, số tiền, điều kiện, ngoại lệ hoặc bước tiếp theo. | So sánh answer với expected answer theo từng claim. Nếu Context Recall cũng thấp thì sửa retrieval; nếu retrieval đủ mà Completeness thấp thì sửa generation/prompt. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Dùng cùng một tập câu hỏi, cùng rubric và hai câu trả lời A/B có chất lượng đã được human xác định. Tạo hai conditions: (1) judge nhận A trước rồi B, (2) judge nhận B trước rồi A. Randomize thứ tự cho nhiều case và chạy mỗi case nhiều lần nếu có thể. So sánh score của cùng một answer ở hai vị trí, tỷ lệ judge chọn answer đứng trước và mức chênh lệch điểm giữa hai conditions. Nếu answer không đổi chất lượng nhưng thường được điểm cao hơn khi đứng trước, đó là position bias. Có thể giảm bias bằng cách randomize order, chấm cả hai thứ tự rồi lấy trung bình, và dùng nhiều judge độc lập.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric phải chấm theo các tiêu chí độc lập như correctness, coverage của required facts, evidence, relevance và actionability; không chấm độ dài như một proxy cho chất lượng. Ghi rõ rằng câu trả lời ngắn nhưng đủ ý, đúng policy và có evidence có thể đạt điểm 5, còn câu dài nhưng lặp ý, lan man hoặc thêm claim không có evidence phải bị trừ điểm. Có thể quy định answer chỉ cần chứa đủ các điều kiện/ngày/amount/exception liên quan và dùng một test pair gồm bản ngắn và bản dài có cùng nội dung để kiểm tra judge.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Human labels giúp kiểm tra judge có hiểu đúng rubric và có thiên lệch có hệ thống hay không. Calibration cho biết score của judge có tương quan với đánh giá của người thật, giúp phát hiện judge quá dễ, quá khắt khe, ưu tiên answer dài hoặc thích phong cách của chính model. Với các case mơ hồ, adversarial hoặc liên quan đến privacy/safety, human review còn cung cấp chuẩn tham chiếu để chỉnh rubric, threshold và quyết định case nào phải escalated.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Student Services là domain policy; claim không có evidence có thể gây sai deadline, học phí hoặc quyền lợi. Score dưới ngưỡng cần block hoặc human-review. |
| Answer Relevance | 0.80 | Answer phải giải quyết đúng intent của sinh viên, không chỉ chứa các từ trùng với question. Với out-of-scope/ambiguous cases cần kiểm tra failure type và safety behavior. |
| Completeness | 0.80 | Cần giữ đủ ngày, amount, điều kiện, ngoại lệ và bước tiếp theo để câu trả lời có thể hành động được. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Dùng **offline evaluation** trên golden dataset ở mỗi pull request hoặc trước khi deploy khi thay prompt, model, retriever, chunking hay policy version. Nó nhanh, lặp lại được và phù hợp làm quality gate; cần block nếu Faithfulness hoặc các metric quan trọng giảm dưới threshold hay giảm quá 0.05 so với baseline. Dùng **online evaluation** sau khi phát hành để theo dõi traffic thật, drift, câu hỏi mới, latency, cost, user feedback và failure rate; online monitoring có thể alert hoặc rollback khi chất lượng suy giảm. Dùng **human review** cho các case high-stakes, ambiguous, adversarial, privacy/safety và để calibrate LLM judge. Chẩn đoán metric cũng cần phân biệt: Context Recall thấp cùng Completeness thấp thường chỉ ra retriever thiếu evidence; retrieval tốt nhưng Faithfulness thấp thường chỉ ra generator hallucination; Recall cao nhưng Precision thấp gợi ý ranking/noise.

---

## Part 2 — Core Coding (09:45–10:40)

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

**Kết quả checkpoint Task 2:** Đã triển khai đủ 3 answer-side metrics, 2
retrieval-side metrics và wiring trong `run_full_eval()`. Targeted tests đạt
**15 passed** gồm 14 tests bắt buộc và 1 test bonus cho `rerank_by_overlap()`.
Context Recall dùng union của các chunks; Context Precision dùng rank-aware
Average Precision nên chunk liên quan đứng càng sớm thì điểm càng cao.

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

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
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

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
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:* Reranking chỉ thay đổi thứ tự của các chunks đã retrieve, không
> thêm hoặc xóa chunk nào. Context Recall được tính trên union của toàn bộ
> chunks, nên tập token được dùng để đo coverage vẫn giữ nguyên. Vì vậy Recall
> không đổi; chỉ Context Precision có thể tăng khi chunks liên quan được đưa lên
> trước noise.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:* Reranking không đủ khi evidence cần thiết không xuất hiện trong
> tập chunks ban đầu, thể hiện qua Context Recall thấp; đổi thứ tự không thể tạo
> ra evidence bị thiếu. Cũng cần sửa query hoặc query expansion khi câu hỏi dùng
> từ đồng nghĩa/paraphrase khiến retriever không nhận diện đúng intent. Cần sửa
> chunking khi evidence bị chia cắt, chunk quá lớn chứa nhiều noise, hoặc một
> điều kiện/ngoại lệ nằm ở chunk khác không được lấy cùng. Trong các trường hợp
> này nên cải thiện retriever, semantic search, metadata filtering hoặc cách
> chia/chọn top-k trước khi áp dụng reranking.

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
