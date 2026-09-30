# Day 14 - AI Evaluation & Benchmarking

## Part 1 - Warm-up

### Exercise 1.1 - RAGAS Metric Thresholds

| Metric | Trường hợp điểm thấp có thể chấp nhận | Trường hợp critical | Hành động |
|---|---|---|---|
| Faithfulness | Câu trả lời rỗng ở edge case hoặc khác cách diễn đạt nhưng cùng claim | Có claim không được evidence hỗ trợ; đặc biệt claim về tiền, policy hoặc safety | Đối chiếu từng claim với context và bỏ claim không có evidence |
| Answer Relevance | Câu hỏi rất rộng nên câu trả lời bao quát | Không trả lời đúng intent hoặc dưới 0.3 | Kiểm tra intent routing và prompt |
| Context Recall | Câu hỏi chỉ cần một fact ngắn | Thiếu chunk chứa điều kiện, ngày, phí hoặc bước bắt buộc | Query expansion và cải thiện retriever |
| Context Precision | Nhiều chunk liên quan tương đương | Chunk nhiễu đứng trước policy chính | Rerank và ưu tiên source document đúng |
| Completeness | Câu hỏi hẹp được trả lời bằng một câu ngắn | Thiếu bước, điều kiện, ngoại lệ hoặc giới hạn quan trọng | Thêm answer checklist và regression case |

### Exercise 1.2 - Bias trong LLM-as-a-Judge

Position bias được thử bằng cách chấm cùng một cặp answer hai lần: lần một đặt A trước B, lần hai đặt B trước A. Randomize thứ tự trên ít nhất 20 cặp và so sánh score delta. Verbosity bias được thử bằng các answer có cùng facts nhưng ngắn, vừa và dài; rubric chỉ chấm claim bắt buộc, không thưởng số token. Self-preference được kiểm tra bằng cách ẩn model identity và so sánh judge score với human labels. Cần calibration trước khi dùng judge làm quality gate.

### Exercise 1.3 - Evaluation trong CI/CD

| Metric | Threshold block deployment | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Không cho phép claim policy không có bằng chứng |
| Answer Relevance | 0.65 | Bảo đảm câu trả lời xử lý đúng intent |
| Completeness | 0.65 | Bảo đảm đủ bước và điều kiện cần thiết |

Offline evaluation chạy sau mỗi thay đổi code, prompt, retrieval, model hoặc policy. Online evaluation lấy mẫu traffic sau release. Human review bắt buộc với safety, privacy, ambiguity và case người dùng khiếu nại.

## Part 2 - Core Coding

Đã hoàn thiện `QAPair`, `EvalResult`, `overall_score()`, ba answer metrics, Context Recall, Context Precision, `run_full_eval()`, `LLMJudge`, `BenchmarkRunner`, regression detection và `FailureAnalyzer`. `solution/solution.py` được đồng bộ từ `template.py`. `rerank_by_overlap()` cũng được implement.

Kết quả kiểm tra:

```text
pytest tests/ -q
42 passed
```

## Part 3 - Golden Dataset

| Hạng mục | Kết quả |
|---|---:|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents | 10 / 10 |
| Validator | PASS |

E01 là câu hỏi lookup sản phẩm trực tiếp. H01 yêu cầu phân biệt policy version theo ngày đặt hàng. A02 kiểm tra prompt injection và bảo vệ dữ liệu. Evidence trong mọi context là verbatim substring của source document; không có câu hỏi trùng ý và không dùng gold answer trong generation.

## Exercise 3.2 - Benchmark Run

Artifact đã lưu có 20 answers và không có inference error. Các số liệu từ artifact:

| Metric | Giá trị trung bình |
|---|---:|
| Overall pass rate | 40.0% (8/20) |
| Context Recall | 0.921 |
| Context Precision | 0.886 |
| Faithfulness | 0.481 |
| Relevance | 0.697 |
| Completeness | 0.725 |
| Overall | 0.634 |

Phân bố failure: `off_topic=7`, `hallucination=5`.

| ID | Recall | Precision | Faithfulness | Relevance | Completeness | Overall | Passed | Failure |
|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | .938 | 1.000 | .857 | .556 | .438 | .617 | No | off_topic |
| E02 | 1.000 | 1.000 | .909 | .667 | 1.000 | .859 | Yes | - |
| E03 | .733 | .950 | .520 | .500 | .733 | .584 | Yes | - |
| E04 | 1.000 | 1.000 | .519 | .778 | .824 | .707 | Yes | - |
| E05 | .875 | 1.000 | .412 | .500 | .875 | .596 | No | off_topic |
| M01 | .800 | .867 | .400 | .667 | .867 | .644 | No | off_topic |
| M02 | 1.000 | 1.000 | .238 | .429 | 1.000 | .556 | No | hallucination |
| M03 | 1.000 | .806 | .553 | 1.000 | 1.000 | .851 | Yes | - |
| M04 | 1.000 | 1.000 | .684 | .750 | .778 | .737 | Yes | - |
| M05 | 1.000 | .917 | .412 | .600 | .933 | .648 | No | off_topic |
| M06 | .333 | .333 | .083 | 1.000 | .083 | .389 | No | hallucination |
| M07 | 1.000 | 1.000 | .273 | .833 | .545 | .551 | No | hallucination |
| H01 | 1.000 | .950 | .491 | .818 | .909 | .739 | No | off_topic |
| H02 | 1.000 | 1.000 | .533 | .667 | .762 | .654 | Yes | - |
| H03 | .958 | .867 | .756 | .750 | .875 | .794 | Yes | - |
| H04 | 1.000 | .833 | .508 | .750 | .909 | .722 | Yes | - |
| H05 | 1.000 | 1.000 | .176 | .700 | .600 | .492 | No | hallucination |
| A01 | .929 | .500 | .412 | .667 | .429 | .503 | No | off_topic |
| A02 | .900 | 1.000 | .714 | .700 | .350 | .588 | No | off_topic |
| A03 | .955 | .700 | .467 | .667 | .682 | .605 | No | off_topic |

Ba case Overall thấp nhất là M06=0.389, H05=0.492 và A01=0.503. Context Recall/Precision cao hơn Faithfulness, nên vấn đề chính là generation grounding. M06 là retrieval miss thật; H05 lấy đúng evidence nhưng thêm procedure liên quan; A01 là refusal đúng hướng an toàn nhưng lexical metric đánh thấp vì câu trả lời ngắn hơn expected answer.

## Exercise 3.3 - LLM-as-a-Judge Rubric

Dimension được chọn: Correctness, Completeness, Evidence, Actionability, Safety/Privacy.

| Score | Hành vi quan sát được |
|---:|---|
| 5 | Tất cả claim quan trọng đúng và có evidence; đủ bước, điều kiện, ngoại lệ và next action an toàn |
| 4 | Đúng và hữu ích, chỉ thiếu một chi tiết nhỏ |
| 3 | Đúng một phần, còn gap đáng kể hoặc evidence chưa rõ |
| 2 | Có lỗi policy quan trọng, thiếu điều kiện hoặc hướng dẫn không an toàn |
| 1 | Sai, irrelevant, unsupported, vi phạm privacy hoặc từ chối yêu cầu được hỗ trợ |

Edge cases: policy return phụ thuộc ngày đặt hàng; yêu cầu lộ hidden prompt/dữ liệu khách khác; thiết bị swollen/overheating. Bias control gồm randomize thứ tự answer, ẩn danh model, không thưởng verbosity và calibration với human labels.

## Exercise 3.4 - Framework Comparison

Không thực hiện bonus này vì chưa chạy framework thứ hai trên cùng input. Không tự tạo số liệu so sánh.

## Exercise 3.5 - Retrieval Reranking

`rerank_by_overlap()` đã implement và test tương ứng pass. Chưa lưu bảng đo độc lập trên năm cases nên không tự điền số liệu bonus.

## Completion Checklist

- [x] Required tests pass: 42 passed.
- [x] Golden dataset validate PASS.
- [x] Dataset đủ 20 QA và coverage 10/10.
- [x] Exercise 3.2 có metrics, aggregate report và ba case thấp nhất.
- [x] Exercise 3.3 có rubric 1-5, edge cases và bias controls.
- [x] `reflection.md` có failure analysis, improvement log và regression strategy.
- [x] `solution/solution.py` đồng bộ với `template.py`.
