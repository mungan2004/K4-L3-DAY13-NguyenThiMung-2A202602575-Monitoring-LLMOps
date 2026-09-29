# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: HighResponseLatency
- Severity: page
- Duration: 5m
- Kênh thông báo: Slack #slack-oncall
- SLI/SLO liên quan: fast_successful_requests (p95_latency <= 3000ms)
- Điều kiện và thời gian duy trì: p95_latency > 3000ms kéo dài trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ quá lâu để nhận được câu trả lời từ AI (trải nghiệm kém, timeout).
- Ba bước kiểm tra đầu tiên:
  1. Xem dashboard để xác định độ trễ cao nằm ở component nào (Retrieval hay Generation).
  2. Dùng `correlation_id` của request chậm lọc trên Langfuse Trace để xem span bị chậm.
  3. Kiểm tra logs để xem có lỗi kết nối đến Vector DB hoặc API LLM Provider không.
- Mitigation tạm thời: Giảm số lượng tài liệu retrieve (top-k) hoặc chuyển sang dùng mô hình LLM nhỏ/nhanh hơn nếu LLM chính đang quá tải.
- Owner: @team-platform

## Alert 2

- Tên: HighErrorRate
- Severity: page
- Duration: 3m
- Kênh thông báo: Slack #slack-oncall
- SLI/SLO liên quan: fast_successful_requests (error rate <= 2%)
- Điều kiện và thời gian duy trì: tỷ lệ lỗi > 5% duy trì trong 3 phút.
- Ảnh hưởng tới người dùng: Người dùng không nhận được kết quả (nhận lỗi 500 liên tục), tính năng bị gián đoạn.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra log có lỗi `RuntimeError` hay `AttributeError` nào xuất hiện nhiều không.
  2. Xác nhận xem lỗi phát sinh từ Retrieval (Vector Store timeout) hay LLM (API Error).
  3. Lấy 1 request lỗi kiểm tra trực tiếp qua Langfuse trace.
- Mitigation tạm thời: Kích hoạt chế độ degraded mode (trả về câu trả lời cache hoặc thông báo bảo trì thân thiện) hoặc tắt tool bị lỗi để LLM dùng kiến thức nội tại.
- Owner: @team-ai

## Alert 3

- Tên: CostSpike
- Severity: ticket
- Duration: 10m
- Kênh thông báo: Slack #slack-finops
- SLI/SLO liên quan: daily_cost_usd_max <= 2.5
- Điều kiện và thời gian duy trì: Chi phí LLM tăng vọt > 2.5 USD mỗi phút duy trì trong 10 phút.
- Ảnh hưởng tới người dùng: Không ảnh hưởng tới người dùng trực tiếp, nhưng rủi ro cháy túi của tổ chức.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra Dashboard xem số lượng token_in/out có tăng đột biến không.
  2. Xem Logs xem có cuộc tấn công brute-force hoặc loop vô hạn nào không.
  3. Kiểm tra model name đang dùng có bị trỏ nhầm sang model đắt tiền (vd: GPT-4 thay vì GPT-4o-mini) không.
- Mitigation tạm thời: Áp dụng Rate Limit tạm thời cho các user/IP có lượng request quá cao; rollback lại phiên bản prompt cũ nếu prompt mới sinh ra output quá dài.
- Owner: @team-finops
