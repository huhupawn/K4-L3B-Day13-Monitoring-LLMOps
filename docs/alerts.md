# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: P95 latency của `response_sent.latency_ms <= 3000ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu hơn bình thường trước khi nhận phản hồi từ AI, trải nghiệm suy giảm.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel Latency để xác nhận P95/P99 và TTFT, khoanh vùng thời điểm bắt đầu tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh span `retrieval` và `generation` để xác định bước nào bất thường.
- Mitigation tạm thời: Dựa trên evidence để rollback prompt version, khôi phục cấu hình liên quan, hoặc tắt practice scenario.
- Owner: `oncall-engineer`

## Alert 2

- Tên: `HighApiErrorRate`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ lỗi request <= 2%
- Điều kiện và thời gian duy trì: `error_rate_pct > 2%` trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận phản hồi lỗi HTTP 500, dịch vụ chat bị gián đoạn.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel Errors để xem error rate và phân loại lỗi `error_type`.
  2. Lọc `data/logs.jsonl` tìm các event `request_failed`, ghi nhận `error_type` và `correlation_id` đại diện.
  3. Tra cứu `correlation_id` trên Langfuse để xem span gặp lỗi và thông tin chi tiết.
- Mitigation tạm thời: Kiểm tra exception detail, restart pod/service hoặc fallback sang chế độ safe mode nếu phụ thuộc bên ngoài gặp sự cố.
- Owner: `oncall-engineer`

## Alert 3

- Tên: `LowRetrievalSuccessRate`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỉ lệ retrieval thành công >= 90%
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90%` trong 5 phút
- Ảnh hưởng tới người dùng: Bot không lấy được ngữ cảnh chính xác, câu trả lời bị rơi về fallback chung chung hoặc sai lệch.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard panel Errors / Retrieval success để xem tỉ lệ thành công của retrieval.
  2. Lọc `data/logs.jsonl` tìm các log có `tool_name == "retrieval"` và `tool_success == false`, lấy `correlation_id`.
  3. Mở trace tương ứng trên Langfuse để kiểm tra span retriever xem vector store hay retriever timeout.
- Mitigation tạm thời: Khởi động lại service retrieval, kiểm tra kết nối vector store, chuyển sang index tài liệu dự phòng.
- Owner: `rag-ops-team`

