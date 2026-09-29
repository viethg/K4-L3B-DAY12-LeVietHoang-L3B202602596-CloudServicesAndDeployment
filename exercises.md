# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Việt Hoàng  Mã học viên: L3B202602596

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy service lên môi trường cloud/production, nếu người triển khai sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard:
- Nếu code có giá trị mặc định như `"changeme"`, ứng dụng vẫn khởi động bình thường và báo healthy. Kẻ xấu hoặc bot scan Internet có thể dễ dàng đoán ra khóa mặc định phổ biến này và gọi vào `/ask`, đốt sạch tiền API OpenAI/LLM và làm lộ dữ liệu trước khi chúng ta nhận ra.
- Nhờ cơ chế fail-fast (không có default value, pydantic ném `ValidationError`), container crash ngay lập tức khi vừa khởi động, orchestrator báo deploy failed và log ghi rõ `Field required: agent_api_key`. Lập tức ta phát hiện và bổ sung secret đúng cách trước khi service đón bất kỳ request nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T09:30:56.741038+00:00", "user_id": "sv01", "tokens_in": 103, "tokens_out": 52, "cost_usd": 4.665e-05}
```

Hai việc làm được với dòng log JSON này:
1. **Lọc và phân tích tự động trên hệ thống quản lý log tập trung (Datadog, Grafana Loki, CloudWatch)**: Hệ thống ingest log có thể tự động parse các trường `user_id`, `tokens_in`, `tokens_out`, `cost_usd` thành các thuộc tính có thể truy vấn (ví dụ: `SELECT SUM(cost_usd) WHERE user_id = 'sv01'`) mà không phải viết regex phức tạp.
2. **Cảnh báo và tổng hợp metrics theo thời gian thực**: Có thể tạo biểu đồ theo dõi chi phí theo giờ dựa trên trường `cost_usd`, hoặc cấu hình alert tự động gửi về Slack/PagerDuty khi `cost_usd` của một request vượt ngưỡng bất thường. Lệnh `print("đã trả lời xong")` không có cấu trúc và không mang thông tin định lượng để làm việc này.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 329 MB (content size: 77 MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~700 MB) gồm:
1. Base image đầy đủ của Python chứa nhiều package hệ thống, build tools, compiler (gcc, make) và thư viện dev không cần thiết khi chạy ứng dụng.
2. Trong multi-stage build, stage `builder` thực hiện biên dịch và cài đặt thư viện vào `/opt/venv`, còn stage `runtime` sử dụng base image `python:3.11-slim` và chỉ copy đúng thư mục `/opt/venv`. Nhờ đó loại bỏ được toàn bộ pip cache, file tạm, build-time dependencies và apt package thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Những layer được dùng lại từ cache (CACHED)**: Toàn bộ các layer từ đầu đến trước lệnh `COPY . .` (kéo base image `python:3.11-slim`, tạo virtualenv, `COPY requirements.txt .`, `RUN pip install ...`, cài đặt `curl`, tạo user `appuser`, và copy `/opt/venv` sang runtime).
- **Những layer phải chạy lại**: Chỉ từ layer `COPY . .` và `RUN chown -R appuser:appuser /app` trở đi.
- **Nếu đặt `COPY . .` trước `RUN pip install`**: Mỗi lần thay đổi code dù chỉ 1 ký tự, checksum của layer `COPY . .` thay đổi làm mất hiệu lực toàn bộ cache phía sau. Docker sẽ buộc phải chạy lại lệnh `RUN pip install` từ đầu, tải và cài lại toàn bộ thư viện, khiến thời gian build kéo dài từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python có lỗ hổng bảo mật (ví dụ Remote Code Execution - RCE qua deserialization hoặc command injection).
2. Kẻ tấn công khai thác lỗ hổng để thực thi shell command bên trong container. Vì container mặc định chạy dưới user `root` (UID 0), tiến trình của attacker cũng có quyền root bên trong container.
3. Kẻ tấn công tận dụng quyền root để khai thác các lỗ hổng container breakout (lỗ hổng kernel Linux, namespace escape hoặc mount nhạy cảm như `/var/run/docker.sock`). Khi thoát được ra host, do UID trong container là 0 trùng với UID 0 của root host, kẻ tấn công lập tức có toàn quyền kiểm soát máy host.
4. **Lệnh `USER appuser` cắt đứt chuỗi này**: Tiến trình ứng dụng được hạ quyền xuống user thường (UID 1000). Nếu xảy ra RCE, attacker chỉ có quyền của `appuser`, không thể sửa file hệ thống của container, không có Linux capabilities đặc quyền, và việc leo thang đặc quyền để breakout ra máy host bị chặn đứng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Giải thích:
- Giả sử hạn mức là 10 request/phút theo giờ đồng hồ.
- Lúc 10:00:59 (1 giây trước khi hết phút), người dùng gửi 10 request. Vì trong phút 10:00 mới có 10 request nên hệ thống cho qua toàn bộ.
- Ngay lúc 10:01:00 (vừa bước sang phút mới, bộ đếm bị reset về 0), người dùng gửi tiếp 10 request nữa và tiếp tục được cho qua.
- Như vậy, trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ 10:00:59 đến 10:01:01), người dùng đã gửi thành công $10 + 10 = 20$ request (gấp đôi hạn mức quy định), có thể gây sập server. Thuật toán sliding window bằng Redis ZSET giải quyết triệt để lỗi này vì luôn tính tổng request trong 60 giây trượt lùi từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau**:
  - `Rate Limiter`: Bảo vệ **tài nguyên hạ tầng / CPU / Network** bằng cách kiểm soát số lượng request trên đơn vị thời gian (ví dụ 10 req/phút).
  - `Cost Guard`: Bảo vệ **ngân sách tài chính** bằng cách kiểm soát số tiền chi tiêu / lượng token LLM tích lũy trong một chu kỳ (ví dụ 10 USD/tháng).
- **Tình huống Rate limit cho qua nhưng Cost guard chặn**: User chỉ gửi đúng 1 request trong phút (thỏa mãn hạn mức 10 req/phút), nhưng tài khoản của user đó trong tháng đã tiêu hết 9.99 USD trên hạn mức 10.0 USD, và câu hỏi mới ước tính tốn 0.05 USD. Rate limit cho qua nhưng Cost guard chặn với HTTP 402 Payment Required.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn**: Đầu tháng user chưa tiêu đồng nào (ngân sách còn nguyên 10.0 USD), nhưng gửi liên tục 15 request chỉ trong vòng 3 giây. Cost guard cho qua vì chưa hết tiền, nhưng Rate limit chặn từ request thứ 11 với HTTP 429 Too Many Requests để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố mạng hoặc khởi động lại tạm thời trong 30 giây.
2. Orchestrator (Docker/Kubernetes/Render) gửi probe kiểm tra liveness tới cả 3 container agent.
3. Do endpoint kiểm tra liveness lại phụ thuộc vào Redis, cả 3 container đồng loạt trả về thất bại (503 hoặc timeout).
4. Orchestrator hiểu nhầm rằng cả 3 tiến trình agent đã bị treo/hỏng vĩnh viễn, nên lập tức cưỡng chế kill và restart cả 3 container (cascading restart).
5. Trong khi Redis đang hồi phục, các container vừa khởi động lại tiếp tục fail probe và lại bị restart tiếp, tạo thành vòng lặp khởi động vô tận (crash loop). Toàn bộ hệ thống sập hoàn toàn chỉ vì một dependency bên ngoài gián đoạn tạm thời.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python (trong bộ nhớ RAM của tiến trình):
- Khi có 3 instance agent chạy sau load balancer (round-robin), các request liên tiếp của user sẽ lần lượt được gửi tới các instance khác nhau.
- Request 1 vào Instance A: A lưu vào RAM của A, trả về `history_length = 0`.
- Request 2 vào Instance B: B kiểm tra RAM của B thấy chưa có gì, trả về `history_length = 0` (mất ngữ cảnh câu hỏi trước).
- Request 3 vào Instance C: C cũng thấy rỗng, trả về `history_length = 0`.
- Request 4 quay lại Instance A: A đọc RAM của mình và trả về `history_length = 2`.
- Kết quả là `history_length` nhảy thất thường (0, 0, 0, 2, 2...) và bot bị "mất trí nhớ". Khi đưa state sang Redis, cả 3 instance cùng đọc ghi vào một nơi tập trung, `history_length` tăng đều đặn và chính xác (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: Khi deploy lên container, ứng dụng crash ngay lúc khởi động với traceback: `NotImplementedError: TODO (CP4): cài đặt install` tại `app/lifecycle.py`, dẫn đến `Application startup failed. Exiting.` và service bị restart liên tục.
- **Cách tìm ra nguyên nhân**: Kiểm tra log container qua `docker compose logs agent` và tab Logs trên Render Dashboard, phát hiện FastAPI `lifespan()` lúc khởi động gọi `lifecycle.install()`. Do hàm đăng ký signal handler trong `app/lifecycle.py` chưa được implement nên ném `NotImplementedError`.
- **Cách sửa**: Cài đặt hoàn thiện hàm `install()` và `request_shutdown()` trong `app/lifecycle.py`: lưu handler mặc định của Uvicorn vào dictionary `_previous`, đăng ký handler cho `signal.SIGTERM` và `signal.SIGINT`, khi nhận tín hiệu thì bật `shutting_down = True` và gọi lại handler cũ để Uvicorn dừng server một cách graceful. Sau khi push code mới, container khởi động thành công và endpoint `/health` trả về 200 OK.
