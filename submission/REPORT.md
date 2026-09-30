# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Chỉ cần 3 output text và 5 ảnh runtime; dùng đường dẫn tương đối, ví dụ `evidence/03-incident-trace.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Anh Hoàng
- **MSSV:** 2A202602816
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/anhho/K4-L3-DAY13-NguyenAnhHoang-2A202602816-Monitoring-LLMOps
- **Commit SHA cuối:** 9ed13fa
- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602816`

## 2. Evidence index

Giữ đúng ba output text và năm ảnh dưới đây. Không tách thêm ảnh; nếu cần giải thích, ghi bằng chữ trong các mục sau.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/pytest.txt` |
| Log validator | `evidence/log-validator.txt` |
| Dashboard validator | `evidence/dashboard-validator.txt` |
| Structured log + incident log | `evidence/01-incident-log.png` |
| Trace list | `evidence/02-trace-list.png` |
| Trace waterfall + metadata + incident trace | `evidence/03-incident-trace.png` |
| Prompt versions + promote/rollback | `evidence/04-prompt-versioning.png` |
| Dashboard + incident metric | `evidence/05-dashboard-incident.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | 100/100 | Đạt toàn bộ 4/4 tiêu chí: schema, correlation ID, enrichment, PII scrubbing |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Hợp lệ toàn bộ 6 panel theo chuẩn contract của config/dashboard.yaml |
| `pytest` | 22 passed | 24 passed | Bổ sung test kiểm thử che giấu CCCD và Credit Card, 100% tests pass |
| Số traces hợp lệ | 0 | 64 traces (10 trên prompt v2) | Đầy đủ cây phân cấp: root agent, child retriever và child generation; 53 trace v1 + 10 trace v2 |
| Số PII leak | 0 | 0 | Không còn rò rỉ dữ liệu nhạy cảm (email, SĐT, CCCD, thẻ) trong logs.jsonl |
| Latency P95 / TTFT P95 | 734.5ms / 50ms | 157.1ms / 50ms | Độ trễ ổn định ở trạng thái bình thường; TTFT duy trì ở mức ~50ms |
| Retrieval success rate | 100% | 100% | Retrieval tìm kiếm ngữ cảnh thành công cho mọi câu hỏi trong corpus |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:**
  - Trong `CorrelationIdMiddleware` (`app/middleware.py`), trước khi xử lý request, middleware gọi `clear_contextvars()` để xóa context của request trước đó, tránh rò rỉ dữ liệu giữa các luồng.
  - Middleware kiểm tra header `x-request-id`. Nếu client có gửi thì tái sử dụng; nếu không, tự sinh mã định danh theo đúng quy ước `req-<8-hex>` thông qua `f"req-{uuid.uuid4().hex[:8]}"`.
  - Gắn correlation ID vào structlog contextvars bằng `bind_contextvars(correlation_id=correlation_id)` và lưu vào `request.state.correlation_id`.
  - Khi trả response về client, middleware gắn `x-request-id` và `x-response-time-ms` (thời gian tính bằng `perf_counter` nhân 1000) vào response headers.
- **Các metadata được ghi vào structured log:**
  - `ts`: Thời gian ghi log theo chuẩn ISO 8601 UTC.
  - `level`: Mức độ log (`info`, `warning`, `error`).
  - `service`: Tên dịch vụ (`api`, `control`, hoặc app name).
  - `event`: Tên sự kiện (`request_received`, `response_sent`, `request_failed`).
  - `correlation_id`: Định danh duy nhất theo dõi request.
  - Context enrichment: `user_id_hash` (băm SHA256 12 ký tự), `session_id`, `feature`, `model` (`claude-sonnet-4-5`), `env` (`dev`).
  - Operational metrics: `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`, và `payload` chứa preview đã được lọc sạch PII.
- **Cách bảo đảm PII được scrub trước khi ghi:**
  - Xây dựng module `app/pii.py` với từ điển regex `PII_PATTERNS` bao gồm: email, số điện thoại Việt Nam (đầu số 0 hoặc +84 và 9 chữ số theo sau), CCCD (12 chữ số), thẻ tín dụng/ghi nợ (16 chữ số phân cách bằng khoảng trắng hoặc gạch nối), hộ chiếu (1 chữ cái in hoa kèm 7-8 chữ số).
  - Đăng ký hàm processor `scrub_event` trong chuỗi xử lý của `structlog.configure()` ở `app/logging_config.py`.
  - Hàm `_scrub_value` duyệt đệ quy qua toàn bộ dictionary, list và string trong event log dictionary, áp dụng `scrub_text` để thay thế thông tin nhạy cảm thành `[REDACTED_EMAIL]`, `[REDACTED_PHONE_VN]`, `[REDACTED_CCCD]`, `[REDACTED_CREDIT_CARD]`. Vì processor này nằm trước `JsonlFileProcessor()` và `JSONRenderer()`, dữ liệu thô không bao giờ bị ghi xuống đĩa hay serialize.
- **Cách kiểm chứng kết quả:**
  - Chạy script độc lập `python scripts/validate_logs.py`: kiểm tra cấu trúc log JSON, kiểm tra sự tồn tại và tính duy nhất của correlation ID, kiểm tra enrichment fields, và quét regex toàn bộ file `data/logs.jsonl` để đảm bảo 0 leak PII. Đạt điểm tuyệt đối **100/100**.
  - Bổ sung unit test trong `tests/test_pii.py` và chạy `pytest` xác thực các trường hợp biên của số điện thoại, thẻ tín dụng, CCCD.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
  - Project Langfuse Cloud cá nhân được đặt tên `day13-k4-l3b-2A202602816`.
  - Traces được cấu hình gửi trực tiếp từ ứng dụng thông qua API keys cá nhân và base URL `https://jp.cloud.langfuse.com`.
  - Traces được gắn các tags: `["lab", feature, self.model]`, user ID băm `hash_user_id(user_id)`, trace name `day13-agent-request`, và metadata chứa đúng `correlation_id` khớp với log file của ứng dụng.
- **Cấu trúc root/retrieval/generation observations:**
  - **Root observation:** `@observe(name="lab-agent-run", as_type="agent")` bao bọc hàm `LabAgent.run()`, đóng vai trò là span gốc của agent.
  - **Child observation 1:** `@observe(name="retrieval", as_type="retriever")` theo dõi bước truy xuất ngữ cảnh `retrieve(message)`, ghi nhận thời gian thực thi của vector store / corpus search.
  - **Child observation 2:** `@observe(name="generation", as_type="generation")` theo dõi bước gọi mô hình ngôn ngữ `FakeLLM.generate()`, ghi nhận thông tin mô hình (`claude-sonnet-4-5`), prompt liên kết, `usage_details` (`input`, `output`, `total`), và `cost_details` (`cost_usd`).
- **Cách nối trace với log:**
  - Trong hàm `LabAgent.run()`, correlation ID được truyền từ `request.state.correlation_id` vào `propagate_attributes(metadata={"correlation_id": correlation_id})`.
  - Khi xem một dòng log trong `data/logs.jsonl`, kỹ sư vận hành lấy giá trị `correlation_id` (ví dụ `req-e3abe91f`), sau đó tìm kiếm trong ô filter/search metadata trên giao diện Langfuse để mở chính xác trace tương ứng.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1, labels: `baseline`, `production`
- **Version/label candidate:** Version 2, labels: `candidate`
- **Trace ID của mỗi version:**
  - Baseline, label `baseline` → v1: `00e3338a910c0f3459cc4a60a084fef0`
  - Candidate, label `candidate` → v2: `6122bff2fbaa2990c9a7fa122012e449`
  - **Promote**: đổi label `production` sang v2, chạy workload lúc **05:20:10 UTC** → 10 trace với `prompt_version=2`. Trace đại diện: `27b4902401bd690e477273432fa1eb6f`.
  - **Rollback**: đổi label `production` về v1, chạy lại workload lúc **05:23:58 UTC** → 10 trace với `prompt_version=1`. Trace đại diện: `ff58c6fa51101aa41c8c94df5942e592`.
  - Bằng chứng đo được của vòng promote/rollback: `tokens_in` trung bình **33 (v1) → 56 (v2) → 34 (sau rollback)**, vì v2 thêm tiền tố "Answer in no more than three concise bullet points." vào prompt.
- **Cách promote và rollback `production`:**
  - **Promote:** Trên Langfuse UI (hoặc qua SDK method `client.update_prompt(name='day13-chat', version=2, new_labels=['candidate', 'production'])`), ta chuyển nhãn `production` từ Version 1 sang Version 2. Ứng dụng production gọi `client.get_prompt("day13-chat", label="production")` sẽ tự động nhận prompt Version 2 mà không cần sửa đổi mã nguồn.
  - **Rollback:** Khi phát hiện Version 2 có hiện tượng suy giảm chất lượng, tăng đột biến chi phí hoặc độ trễ, ta thực hiện rollback tức thì bằng cách chuyển nhãn `production` quay trở về Version 1 (`client.update_prompt(name='day13-chat', version=1, new_labels=['baseline', 'production'])`). Ứng dụng lập tức phục hồi phiên bản prompt an toàn.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
  - Được xây dựng và xác thực theo đặc tả trong `config/dashboard.yaml` với nguồn dữ liệu từ `data/logs.jsonl`:
    1. **Panel Latency & TTFT:** Hiển thị P50, P95, P99 độ trễ và P95 TTFT, có đường ngưỡng cảnh báo 3000ms.
    2. **Panel Traffic:** Thống kê lưu lượng request theo thời gian, có ngưỡng tối thiểu 1 req/phút.
    3. **Panel Errors:** Thể hiện tỷ lệ lỗi hệ thống (ngưỡng tối đa 2%) và tỷ lệ truy xuất retrieval thành công (ngưỡng tối thiểu 90%).
    4. **Panel Cost:** Biểu đồ chi phí tích lũy theo thời gian tính bằng USD, có đường budget tối đa $2.50.
    5. **Panel Tokens:** Tổng lượng token đầu vào (input) và token đầu ra (output), giới hạn ngưỡng 50,000 tokens.
    6. **Panel Quality:** Điểm chất lượng trung bình (quality proxy từ 0 đến 1), đường ngưỡng chất lượng tối thiểu 0.75.
- **SLO và lý do chọn:**
  - **Primary SLO:** `fast_successful_requests` với mục tiêu 99.5% trong cửa sổ 28 ngày (`target_percent: 99.5`).
  - **SLI:** Tỷ lệ giữa good events (`event == "response_sent" and latency_ms <= 3000`) trên tổng số request nhận vào (`event == "request_received"`).
  - **Lý do chọn:** Đối với dịch vụ hội thoại AI trực tiếp, người dùng mong muốn phản hồi nhanh chóng dưới 3 giây và không gặp lỗi máy chủ 500. Ngưỡng 99.5% là tiêu chuẩn phù hợp cho dịch vụ cấp độ doanh nghiệp, vừa đảm bảo chất lượng trải nghiệm vừa tạo không gian an toàn cho việc cập nhật tính năng.
- **Cách tính error budget:**
  - Với SLO target là 99.5%, Error Budget cho phép là `100% - 99.5% = 0.5%`.
  - Nếu trong chu kỳ 28 ngày hệ thống nhận được 10,000 requests, số lượng request được phép thất bại hoặc có độ trễ vượt quá 3000ms là:
    $$10,000 \times 0.5\% = 50 \text{ requests}$$
  - Khi số lượng request vi phạm vượt quá 50, Error Budget bị cạn kiệt (burn rate > 100%), kích hoạt chính sách đóng băng triển khai (feature freeze) để ưu tiên ổn định hệ thống.
- **Ba alert và runbook tương ứng:**
  1. **`HighLatencyP95` (Warning):**
     - Điều kiện: `p95(latency_ms) > 3000` duy trì liên tục trong 5 phút. Kênh: Slack `#k4-l3b-alerts`.
     - Ảnh hưởng: Người dùng phải chờ quá lâu trước khi nhận được phản hồi.
     - Runbook: Mở dashboard panel Latency xác định thời điểm tăng; lọc `data/logs.jsonl` tìm correlation ID có `latency_ms > 3000`; mở trace trên Langfuse so sánh span `retrieval` vs `generation`; nếu do LLM dài thì rollback prompt, nếu do vector store thì kích hoạt fallback index.
  2. **`HighApiErrorRate` (Critical):**
     - Điều kiện: `error_rate_pct > 2` duy trì trong 3 phút. Kênh: Slack `#k4-l3b-alerts`.
     - Ảnh hưởng: Người dùng nhận mã lỗi HTTP 500, tính năng AI chat bị gián đoạn.
     - Runbook: Mở panel Errors kiểm tra `error_type`; lọc log tìm các event `request_failed` và trích xuất `correlation_id`; mở trace Langfuse xem lỗi ngoại lệ tại span nào; khởi động lại service hoặc kích hoạt safe mode.
  3. **`LowRetrievalSuccessRate` (Warning):**
     - Điều kiện: `retrieval_success_rate_pct < 90` duy trì trong 5 phút. Kênh: Slack `#k4-l3b-alerts`.
     - Ảnh hưởng: Mô hình không nhận được tài liệu ngữ cảnh chính xác, dẫn đến câu trả lời fallback hoặc giảm độ tin cậy.
     - Runbook: Mở panel Retrieval Success; lọc log các bản ghi có `tool_name == "retrieval"` và `tool_success == false`; mở trace Langfuse để kiểm tra lỗi kết nối hoặc timeout với Vector Database; chuyển sang snapshot dữ liệu dự phòng.

## 7. Điều tra challenge

- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Khoảng thời gian điều tra:** 2026-09-30 04:35:00 UTC đến 04:40:00 UTC
- **Triệu chứng từ metrics:**
  - Panel Latency trên Dashboard ghi nhận độ trễ P95 tăng vọt từ mức bình thường (~157ms) lên trên 2650ms khi chịu tải đồng thời (vượt ngưỡng threshold cảnh báo).
  - Panel Traffic tăng theo luồng challenge, Panel Errors giữ ở mức bình thường (không có lỗi 500 do service vẫn trả 200), nhưng thời gian phản hồi toàn hệ thống bị chậm rõ rệt ở tính năng `monitoring`.
- **Log line và correlation ID liên quan:**
  - Log event `response_sent`:
    ```json
    {"service": "api", "latency_ms": 2651, "ttft_ms": 50, "tokens_in": 35, "tokens_out": 162, "cost_usd": 0.002535, "quality_score": 0.8, "tool_name": "retrieval", "tool_success": true, "event": "response_sent", "session_id": "k4-l3b-challenge-s05", "model": "claude-sonnet-4-5", "feature": "monitoring", "env": "dev", "user_id_hash": "68e37dc7cb5e", "correlation_id": "req-4193d0a4", "level": "info", "ts": "2026-09-30T04:37:35.372839Z"}
    ```
  - Correlation ID đại diện: `req-4193d0a4`
- **Trace ID và span gây ảnh hưởng:**
  - Trace ID: `262bfa77df3682bd4bacb85ac7bb2098` (nối với log bằng `correlation_id: req-4193d0a4`, session `k4-l3b-challenge-s05`).
  - Phân tích Waterfall trên Langfuse:
    - Span `lab-agent-run`: Tổng thời gian 2,651ms (2.651s).
    - Span con `retrieval`: Chiếm **2,500ms (2.5s)** (tương đương 94.3% tổng thời gian).
    - Span con `generation`: Chỉ mất **151ms (0.151s)** (TTFT 50ms).
  - Span gây chậm: `retrieval`.
- **Root cause:**
  - Bước truy xuất tài liệu `retrieve()` bị tắc nghẽn do vector database phản hồi chậm (mô phỏng bởi cờ sự cố `STATE["rag_slow"] = True` trong official challenge `day13-k4-l3b-monitoring-llmops-v1`, làm trễ 2.5s khi tìm kiếm tài liệu).
- **Fix action:**
  - Vô hiệu hóa sự cố hoặc khôi phục kết nối vector store (`python scripts/inject_incident.py --disable`).
  - Thêm timeout ngắn (ví dụ 1.5s) cho bước retrieval và áp dụng fallback sang corpus bộ nhớ đệm (cached context) khi vector store quá tải.
- **Preventive measure:**
  - Bổ sung cảnh báo `RetrievalLatencyP95 > 1500ms` để phát hiện suy giảm hiệu năng vector store trước khi ảnh hưởng đến người dùng cuối.
  - Cấu hình Redis caching cho các câu truy vấn phổ biến để giảm tải trực tiếp lên vector database.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
  - Quyết định thực hiện PII scrubbing ở tầng processor của structlog (`scrub_event`) đệ quy qua toàn bộ payload và metadata trước khi serialize JSON và ghi file đĩa.
  - Lý do: Đảm bảo nguyên tắc "Zero Leakage" ngay từ gốc. Dù bất kỳ đoạn mã nào trong tương lai vô tình in message thô hoặc log context, dữ liệu nhạy cảm vẫn được lọc sạch triệt để trước khi ra khỏi tiến trình ứng dụng.
- **Một lỗi/blocker đã gặp:**
  - Khi bắt đầu cài đặt thư viện trên Python 3.14, gói `pydantic-core` không có sẵn pre-built binary wheel trên Windows và phải biên dịch Rust từ nguồn dẫn đến nghẽn quá trình cài đặt. Đồng thời, các file mã nguồn mở trong editor bị overwrite về bản ban đầu.
- **Cách tìm nguyên nhân và xử lý:**
  - Kiểm tra `py --list` và phát hiện máy tính đã có sẵn Python 3.12 (`py -3.12`). Tạo lại virtual environment trên Python 3.12 để tải trực tiếp các wheel đã biên dịch sẵn, hoàn tất cài đặt trong chưa đầy 30 giây.
  - Với mã nguồn, tiến hành kiểm tra git diff, viết lại chuẩn xác từng module và commit ngay vào git (`git commit`) để bảo toàn trạng thái code an toàn.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - **Metrics:** Đóng vai trò như bảng đồng hồ táp-lô trên xe hơi, cung cấp cái nhìn tổng quan ở mức vĩ mô (High-level symptoms) để nhận biết "Hệ thống có đang gặp vấn đề gì không và từ lúc nào?".
  - **Logs:** Đóng vai trò là nhật ký ghi nhận các sự kiện cụ thể (Contextual events), cho phép lọc và xác định "Request cụ thể nào bị ảnh hưởng?" thông qua `correlation_id`.
  - **Traces:** Cung cấp độ phóng chiếu chi tiết vào từng bước bên trong của request đó (Deep inspection), bóc tách từng span con để chỉ ra "Bước nào (Retrieval hay LLM Generation) là nguyên nhân gây chậm hoặc lỗi?".
  - Kết hợp cả ba tạo nên chuỗi bằng chứng xác đáng để kết luận Root cause mà không cần phỏng đoán.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
  - Prompt trong LLMOps cũng là mã nguồn logic. Khi cập nhật prompt, hành vi mô hình thay đổi, có thể dẫn đến sinh câu trả lời quá dài làm tăng đột biến token và chi phí (Cost spike), hoặc làm tăng độ trễ (Tail latency) vi phạm SLO.
  - Quản lý phiên bản prompt (v1/v2) kết hợp nhãn (`baseline`, `candidate`, `production`) cho phép kỹ sư vận hành thử nghiệm an toàn và thực hiện **Rollback tức thì trong vài giây** chỉ bằng thao tác đổi nhãn trên Langfuse mà không cần sửa code hay redeploy toàn bộ hệ thống.
- **Điều quan trọng nhất đã học:**
  - Học được tư duy vận hành thực chiến của một kỹ sư LLMOps: Từ việc bảo vệ dữ liệu nhạy cảm của người dùng (PII scrubbing), quản lý chuỗi truy vết phân tán (distributed tracing với correlation ID), đến cách xây dựng hệ thống cảnh báo và xử lý sự cố có căn cứ dữ liệu rõ ràng.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**
  - Ba ảnh evidence `02-trace-list.png`, `03-incident-trace.png` và `04-prompt-versioning.png` phải được chụp trực tiếp từ giao diện Langfuse Cloud của project cá nhân `day13-k4-l3b-2A202602816` (trang Traces, trang Trace waterfall và trang Prompt versions). Vì các ảnh này phải phản ánh đúng màn hình hiển thị thật nên không thể sinh tự động bằng script, và tuyệt đối không được dùng ảnh từ project dùng chung hoặc của học viên khác.
  - Dashboard 6 panel được dựng và kiểm chứng bằng `scripts/validate_dashboard.py` (6/6 hợp lệ theo contract `config/dashboard.yaml`); ảnh `05-dashboard-incident.png` chụp dashboard sau khi chạy challenge.
  - Số liệu P50/P95/P99 trong báo cáo được tính trên tập log của lần chạy workload cá nhân, không phải trên dữ liệu production thật.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Có đúng 3 file text và 5 ảnh runtime theo hướng dẫn.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
