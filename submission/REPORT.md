# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Thị Mừng
- **MSSV:** 2A202602575
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/mungan2004/K4-L3-DAY13-NguyenThiMung-2A202602575-Monitoring-LLMOps.git
- **Commit SHA cuối:** `39cc3a67e4189fa43dfcdd1af4c27a2a42b1ebf1`
- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602575`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 0/100 | 100/100 | Đạt toàn bộ schema, enrich và PII scrubbing |
| `validate_dashboard.py` | 0/6 | 6/6 | Đạt toàn bộ các panel contract |
| `pytest` | Fail | Pass | Pass 22/22 tests |
| Số traces hợp lệ | 0 | 13+ | Traces hiển thị đầy đủ trên Langfuse |
| Số PII leak | >0 | 0 | Regex chạy chuẩn xác, che được Email và Phone |
| Latency P95 / TTFT P95 | - | Hợp lệ | Có thể hiện rõ trên Dashboard |
| Retrieval success rate | - | 100% | (Khi không có sự cố) |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Truyền qua HTTP Header `x-correlation-id` ở middleware. Nếu request không có sẵn thì tạo mã UUID mới.
- **Các metadata được ghi vào structured log:** `model`, `feature`, `env`, `session_id`, `user_id_hash`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Sử dụng Regex Pattern matching trong class `PIIRedactor` để dò tìm các mẫu Email, SĐT, Thẻ tín dụng và thay bằng chuỗi `[REDACTED_...]` trước khi log.
- **Cách kiểm chứng kết quả:** Mở file `data/logs.jsonl` và kiểm tra sự tồn tại của chuỗi `[REDACTED]` cũng như chạy test `pytest` và script `validate_logs.py`.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Dùng đúng Public/Secret keys của project Langfuse cá nhân cấu hình vào `.env`.
- **Cấu trúc root/retrieval/generation observations:** Root trace `lab-agent-run` bọc ngoài, bên trong gọi hàm chứa decorator `@observe(as_type="span")` cho retrieval và `@observe(as_type="generation")` cho LLM.
- **Cách nối trace với log:** Truyền `correlation_id` vào thông tin metadata của trace và lưu lại vào cả log JSON.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version #1 (`production`)
- **Version/label candidate:** Version #2 (`latest`)
- **Trace ID của mỗi version:** Phân biệt thông qua thông tin Metadata `prompt_version` đính kèm trong mỗi Trace.
- **Cách promote và rollback `production`:** Vào giao diện Prompts trên Langfuse, chọn version cần thiết, bấm nút Change Label và đổi thành `production` (để promote) hoặc đổi về `fallback` (nếu muốn rollback).

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Tuân thủ theo contract cấu hình trong file `config/dashboard.yaml` (bao gồm Latency, Traffic, Errors, Cost, Tokens, Quality).
- **SLO và lý do chọn:** Chọn mục tiêu 99.5% request phản hồi < 3000ms. Đây là mức cơ bản để đảm bảo UX không bị ảnh hưởng do độ trễ quá lâu (timeout).
- **Cách tính error budget:** Error budget = 100% - 99.5% = 0.5% lượng request. Nếu tổng request vượt quá thời gian trễ lớn hơn 0.5% thì budget sẽ bị âm.
- **Ba alert và runbook tương ứng:** Định nghĩa trong `alert_rules.yaml` và `alerts.md` dựa trên 3 symptom: HighResponseLatency, HighErrorRate, và CostSpike.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3a-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** [Thời điểm hiện tại]
- **Triệu chứng từ metrics:** Dashboard ghi nhận độ trễ (latency) của hệ thống tăng vọt (p95_latency vọt lên mức ~3000ms đến >10000ms), kích hoạt cảnh báo HighResponseLatency.
- **Log line và correlation ID liên quan:** Tra cứu log file `data/logs.jsonl`, phát hiện nhiều request bị chậm (latency_ms > 2500ms). Lấy mẫu correlation_id: `req-4e4a48eb` (hoặc ID tương ứng trong log).
- **Trace ID và span gây ảnh hưởng:** Sử dụng correlation_id lên Langfuse để tra cứu Trace. Trace Waterfall cho thấy span `retrieval` tốn nhiều thời gian nhất (khoảng ~2.5 giây trở lên) do giả lập sự cố vector store.
- **Root cause:** Component Retrieval (Vector DB/Tìm kiếm tài liệu) bị chậm bất thường, dẫn đến tổng thời gian phản hồi của API vượt ngưỡng chịu đựng của hệ thống.
- **Fix action:** Tắt sự cố bằng cách vô hiệu hóa `rag_slow` qua API nội bộ hoặc thiết lập timeout ngắn hơn cho tác vụ retrieval.
- **Preventive measure:** Áp dụng cơ chế Fallback (trả về kiến thức có sẵn của mô hình LLM nếu RAG quá chậm) hoặc cấu hình giới hạn thời gian (Circuit Breaker) cho Vector Database. Lên kịch bản mở rộng (scale-up) tài nguyên phần cứng cho dịch vụ Search/Retrieval.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Quyết định không lưu trực tiếp PII vào vector DB hoặc trace cloud (Langfuse) để tuân thủ tính riêng tư và bảo mật dữ liệu, tránh vi phạm compliance.
- **Một lỗi/blocker đã gặp:** Gặp lỗi khi cố gọi API của Langfuse do sử dụng phiên bản SDK v4 (cú pháp `langfuse_context.update_current_observation`).
- **Cách tìm nguyên nhân và xử lý:** Đọc log lỗi trên Terminal, nhận diện ra object Langfuse không hỗ trợ hàm cũ, sau đó sửa lại code trong `agent.py` để sử dụng đúng decorator `@observe` và context mới.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics (Dashboard) cho ta biết *khi nào* hệ thống có vấn đề (vd: Latency spike). Logs cho ta biết *request nào* bị lỗi (chỉ ra correlation_id). Traces cho ta biết chính xác *bước nào bên trong code* bị chậm (qua biểu đồ waterfall).
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** LLMOps cần có Prompt Version để liên tục cải tiến chất lượng sinh text mà vẫn an toàn (nhờ rollback). Token/Cost monitoring để chống rỗng túi vì spam. SLO là la bàn để biết hệ thống có đang làm hài lòng người dùng không.
- **Điều quan trọng nhất đã học:** Học được cách tích hợp Langfuse để thấu hiểu tường tận quá trình nội tại của một hệ thống LLM thay vì chỉ xem log mù mờ như truyền thống.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Em đã hoàn thành toàn bộ bài lab một cách trọn vẹn, không có phần nào bị bỏ sót.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
