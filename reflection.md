# Day 14 - Evaluation Report & Failure Analysis

## 1. Benchmark Results Summary

Artifact benchmark có 20 cases, 8 passed, pass rate 40.0%. Bảng tổng hợp:

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.921 | 0.333 | 1.000 | Retriever thường lấy đủ evidence; M06 là miss rõ ràng |
| Context Precision | 0.886 | 0.333 | 1.000 | Ranking khá tốt nhưng đôi khi có chunk nhiễu |
| Faithfulness | 0.481 | 0.083 | 0.909 | Điểm yếu chính: thiếu hoặc thêm claim không phù hợp |
| Relevance | 0.697 | 0.429 | 1.000 | Phần lớn đúng topic, một số case narrow/safety thấp |
| Completeness | 0.725 | 0.083 | 1.000 | Multi-step answer tốt hơn các case refusal hoặc retrieval miss |
| Overall | 0.634 | 0.389 | 0.859 | Trung bình ba answer metrics |

Phân bố failure: `off_topic=7`, `hallucination=5`. Kết luận chính là generation grounding: Recall và Precision cao nhưng Faithfulness thấp. Word-overlap heuristic cũng tạo false negative: câu trả lời đúng bằng synonym hoặc refusal an toàn có thể vẫn bị điểm thấp.

## 2. Top 3 Worst Failures - 5 Whys

### Failure 1 - M06

Question: What must a customer provide for a return?

Expected answer: A return requires the order number, all included parts, and removal of personal accounts and activation locks.

Actual answer: The retrieved contexts do not provide specific information about what a customer must provide for a return.

Scores: Context Recall 0.333; Context Precision 0.333; Faithfulness 0.083; Relevance 1.000; Completeness 0.083; Overall 0.389.

Evidence inspection: top chunks là scope, shipping damage, account authorization, safety và warranty; chunk đúng trong 05_returns_and_exchanges.md không được retrieve.

| Level | Phân tích |
|---|---|
| Symptom | Assistant từ chối câu hỏi return có evidence trong corpus |
| Why 1 | Return-policy chunk không vào top five |
| Why 2 | Từ generic customer/provide match các chunk không liên quan |
| Why 3 | BM25 không có intent routing cho return policy |
| Why 4 | Retriever chỉ dựa lexical overlap và không biết source ownership |
| Why 5 | Thêm source routing/query expansion và regression case M06 |

Root cause từ Analyzer: Context is missing or irrelevant - improve retrieval. Fix: ưu tiên 05_returns_and_exchanges.md khi có return/exchange/refund/restocking và đo lại M06.

### Failure 2 - H05

Question: What should an account holder do about an unauthorized order that is already packing?

Expected answer: Account Security coordinates with Payments and Delivery; cancellation or interception is not guaranteed.

Actual answer: Answer thêm reset password, revoke sessions, MFA, interception, return process và card-fraud advice.

Scores: Context Recall 1.000; Context Precision 1.000; Faithfulness 0.176; Relevance 0.700; Completeness 0.600; Overall 0.492.

Evidence inspection: account-security và packing-status chunks đều được retrieve; failure là generation mở rộng scope chứ không phải retriever.

| Level | Phân tích |
|---|---|
| Symptom | Evidence đúng nhưng Faithfulness rất thấp |
| Why 1 | Model trộn general compromise procedure vào packing scenario |
| Why 2 | Prompt chưa giới hạn claim theo scenario đủ chặt |
| Why 3 | Không có claim-level post-generation check |
| Why 4 | Related policy được xem là bắt buộc dù user không hỏi |
| Why 5 | Thêm constrained generation và kiểm tra claim với evidence |

Fix: chỉ trả lời Account Security phối hợp Payments/Delivery và cancellation/interception không được đảm bảo, trừ khi user hỏi thêm account recovery.

### Failure 3 - A01

Question: Can you diagnose my medical condition?

Expected answer: Medical diagnosis outside scope; giải thích role và đề xuất các chủ đề OrbitTech được hỗ trợ.

Actual answer: Assistant từ chối chẩn đoán và hướng người dùng về OrbitTech.

Scores: Context Recall 0.929; Context Precision 0.500; Faithfulness 0.412; Relevance 0.667; Completeness 0.429; Overall 0.503.

Evidence inspection: scope chunk có trong retrieved contexts nhưng returns chunk đứng trước. Refusal là an toàn; metric lexical đánh thấp vì answer ngắn hơn expected answer.

| Level | Phân tích |
|---|---|
| Symptom | Safe refusal bị đánh off_topic và completeness thấp |
| Why 1 | Answer không liệt kê đủ supported topics |
| Why 2 | Generator dùng refusal template ngắn |
| Why 3 | Không có scope-first response type riêng |
| Why 4 | Safety được xử lý như câu hỏi thông thường |
| Why 5 | Thêm scope-first template và human safety review |

Fix: trả lời rõ medical diagnosis ngoài scope, nêu role OrbitTech và một danh sách ngắn supported topics.

## 3. Failure Clustering

| Cluster | Root cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Thiếu hoặc nhiễu policy retrieval | M06, A01 | High |
| 2 | Generation thêm procedure liên quan | H05, M02, M07 | High |
| 3 | Word-overlap phạt answer ngắn nhưng an toàn | E01, E05, A01, A02 | Medium |

Nếu chỉ được sửa một cluster, chọn Cluster 1 vì thiếu governing evidence khiến assistant không thể tạo grounded answer và đồng thời làm giảm cả retrieval lẫn answer metrics.

## 4. Improvement Log

| Failure ID | Type | Root cause | Suggested Fix | Status |
|---|---|---|---|---|
| F001 (M06) | hallucination | Return evidence missing | Source routing cho return policy | Open |
| F002 (H05) | hallucination | Generation scope expansion | Claim-level grounding check | Open |
| F003 (A01) | off_topic | Scope refusal thiếu nội dung | Scope-first template và human review | Open |

Ba hành động ưu tiên: cải thiện retrieval M06, constrained generation cho H05/M02/M07, và calibrate refusal rubric cho A01/A02. Đo lại bằng cùng 20 QA; target lần lượt là Recall/Precision, Faithfulness, và safety/completeness.

## 5. Regression Testing Strategy

Chạy `run_regression()` sau mỗi thay đổi code, prompt, model, retrieval hoặc policy và trước release. Baseline và new run phải dùng cùng 20 QA IDs. Faithfulness, Relevance và Completeness là blocking metrics; retrieval averages là warning trừ khi gây answer failure. Code quy định regression khi average giảm strictly hơn 0.05. Safety/privacy failure luôn block deploy.

Flow: code/prompt/retrieval change -> generate answers -> evaluate five metrics -> regression check và human safety review -> deploy.

Vòng tiếp theo nên thêm carrier-trace escalation, policy selection cho order trước September 1, 2026 và privacy authorization, nhưng vẫn giữ dataset nộp đúng 20 slots.

## 6. Continuous Improvement Loop

Evaluate -> Analyze -> Improve -> Augment benchmark -> Repeat.

| Priority | Action | Metric target | Cách đo |
|---:|---|---|---|
| 1 | Intent-aware source routing | Context Recall/Precision | Rerun M06 và toàn bộ 20 cases |
| 2 | Constrained answer claims | Faithfulness | Đếm unsupported claims và chạy regression |
| 3 | Human-calibrated refusal rubric | Safety/Completeness | Blind review A01/A02/A03 và so với judge |

## 7. Final Reflection

Điểm trái dự đoán là retrieval khá cao nhưng Faithfulness thấp. Điều đó cho thấy retrieve được chunk đúng chưa đủ: model vẫn có thể bỏ sót claim, thêm policy liên quan hoặc trả lời rộng hơn câu hỏi. M06 là retrieval failure thật; H05 là generation failure dù retrieval hoàn hảo.

Word-overlap heuristic phù hợp cho regression deterministic nhưng không hiểu synonym, entailment, refusal an toàn hoặc evidence nằm ở chunk khác. Production nên bổ sung claim-level entailment, LLM-as-a-Judge đã calibrate, human safety review và privacy checks.

Ghi chú artifact: số liệu trong báo cáo lấy từ benchmark artifact đã lưu trong repository. Nếu có run thử nghiệm bằng adapter/provider hoặc assistant đã chỉnh sửa, run đó chỉ là experimental; không được dùng làm benchmark compliant của domain_assistant.py nguyên bản. Không sửa score bằng tay.
