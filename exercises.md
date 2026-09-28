# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền nội dung phân tích chi tiết vào bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hải Long  Mã học viên: L3A202602471

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy lên Cloud (Render/Railway), người cấu hình quên thêm biến môi trường `AGENT_API_KEY` trong dashboard. Nếu app đặt giá trị mặc định là `"changeme"`, container vẫn khởi động bình thường và liveness probe báo 200 OK. Khi đó, bot hoặc kẻ tấn công có thể dùng API key mặc định `"changeme"` để gọi API, đốt sạch ngân sách LLM; hoặc người dùng thật gửi key thật lên thì bị từ chối 401 mà không ai phát hiện hệ thống đang chạy với key giả cho đến khi kiểm tra hóa đơn. Cơ chế fail-fast (không có giá trị mặc định) khiến container crash ngay lập tức với `ValidationError`, orchestrator cảnh báo deploy thất bại và lập trình viên sửa ngay trước khi bất kỳ request nào được tiếp nhận.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:00:15.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.000114}
```

Hai việc làm được với log JSON mà `print()` không làm được:
1. **Lọc và truy vấn log có cấu trúc:** Các hệ thống phân tích log tập trung (Datadog, CloudWatch, Elasticsearch) tự động bóc tách các trường JSON để lọc chính xác: ví dụ tìm mọi request của user cụ thể `user_id = "sv-test"` hoặc tìm các request tốn kém `cost_usd > 0.001`. Lệnh `print()` thông thường chỉ ra chuỗi text phi cấu trúc nên không thể lọc chính xác theo trường.
2. **Thống kê số liệu và cảnh báo tự động:** Có thể tự động vẽ biểu đồ tính tổng chi phí token tiêu thụ theo thời gian thực (`SUM(cost_usd)`), hoặc thiết lập cảnh báo (alert) khi tỷ lệ lỗi `level = "error"` vượt ngưỡng 5% trong 5 phút. `print("đã trả lời xong")` hoàn toàn không có dữ liệu định lượng để tổng hợp số liệu.

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
| 1 stage (bản đầu) | 1010 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~739 MB) bao gồm:
1. Toàn bộ bộ công cụ biên dịch (compilers, build-essential, gcc, g++, make) và các header files hệ thống (`python3-dev`) cần để build thư viện C-extension ở stage builder, nhưng hoàn toàn không cần cho runtime.
2. Bộ nhớ đệm của package manager (`pip cache`, wheel cache) và các file cài đặt tạm thời phát sinh trong quá trình cài đặt.
3. Sự khác biệt giữa base image gốc `python:3.11` (chứa đầy đủ các công cụ hệ điều hành, debug packages, man pages) và `python:3.11-slim` (chỉ giữ lại những gì tối thiểu nhất để chạy Python runtime).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer từ đầu cho đến `COPY requirements.txt .`, `RUN pip install ...`, và việc copy các gói từ stage builder sang runtime đều được tái sử dụng hoàn toàn từ cache (`CACHED`). Chỉ layer `COPY app ./app` và các bước kế tiếp sau nó mới phải chạy lại. Thời gian build lại chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` trước `RUN pip install`: Docker tính toán cache theo từng layer tuần tự. Khi sửa một ký tự trong `app/main.py`, checksum của layer `COPY . .` bị thay đổi, kéo theo toàn bộ cache của các layer phía sau bị vô hiệu hóa (cache invalidation). Kết quả là Docker buộc phải chạy lại lệnh `RUN pip install` từ đầu, tải và cài đặt lại toàn bộ thư viện mỗi khi sửa code, làm chậm quá trình deploy rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công phát hiện và khai thác một lỗ hổng trong mã nguồn Python (như Remote Code Execution - RCE qua command injection hoặc deserialize dữ liệu không an toàn).
2. Kẻ tấn công mở được một reverse shell bên trong container. Do container mặc định chạy bằng root (UID 0), kẻ tấn công sở hữu toàn quyền root bên trong container (có thể đọc mọi file, cài công cụ xâm nhập).
3. Từ quyền root container, kẻ tấn công khai thác lỗ hổng bảo mật của nhân Linux (kernel exploit) hoặc truy cập các socket được mount vào container (như `/var/run/docker.sock`) để thoát khỏi container (Container Escape) và chiếm quyền root trực tiếp trên máy host vật lý.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại bước 2: Khi tiến trình Python chạy dưới quyền user thông thường không có đặc quyền (UID 10001), kẻ tấn công dù có RCE cũng chỉ mang quyền hạn chế, không thể sửa đổi file hệ thống, không thể load kernel module hay can thiệp vào các tài nguyên nhạy cảm của host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích:
- Với cơ chế đếm theo phút đồng hồ (fixed window reset tại giây 00):
  - Người dùng gửi 10 request vào giây cuối cùng của phút thứ $T$ (ví dụ: 10:00:59). Vì trong khung phút $T$ số lượng chưa vượt quá 10, toàn bộ 10 request đều được chấp nhận.
  - Ngay ở giây tiếp theo (10:01:00), đồng hồ bước sang phút mới $T+1$ và bộ đếm tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa. Cả 10 request này tiếp tục được chấp nhận vì thuộc hạn mức của phút mới.
  - Tổng cộng có 20 request được gửi thành công chỉ trong vòng 2 giây (từ 10:00:59 đến 10:01:00), gấp đôi ngưỡng cho phép và có thể làm sập hệ thống. Sliding window 60s ngăn chặn được điều này vì nó luôn tính tổng request trong 60 giây liên tục lùi từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Điểm khác nhau:
- **Rate Limit**: Giới hạn **tần suất / số lượng request** trong một khoảng thời gian ngắn (ví dụ: 10 request / 60 giây) nhằm bảo vệ hạ tầng máy chủ khỏi nghẽn mạng và quá tải CPU/RAM (chống spam, DDoS).
- **Cost Guard**: Giới hạn **tổng chi phí tài chính (USD hoặc tokens)** trong một chu kỳ dài hạn (ví dụ: 10.0 USD / tháng) nhằm bảo vệ ví tiền của chủ hệ thống trước các request tốn kém tài nguyên LLM.

Tình huống Rate Limit cho qua nhưng Cost Guard chặn:
- Người dùng chỉ gửi 1 request duy nhất trong 10 phút (tần suất rất thấp, rate limit cho qua). Tuy nhiên, người dùng này đã tiêu hết 10.0 USD ngân sách tháng trước đó -> Cost Guard kiểm tra ngân sách và lập tức chặn lại với mã lỗi 402 Payment Required trước khi gọi LLM.

Tình huống Cost Guard cho qua nhưng Rate Limit chặn:
- Vào đầu tháng, người dùng mới tiêu 0.05 USD / 10.0 USD (ngân sách còn rất nhiều). Nhưng người dùng chạy script gửi dồn dập 15 request trong vòng 3 giây -> Rate Limit lập tức chặn từ request thứ 11 với mã lỗi 429 Too Many Requests để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis mất kết nối mạng hoặc restart trong 30 giây.
2. Endpoint kiểm tra liveness chung gọi tới Redis bị lỗi và trả về mã lỗi 503 trên cả 3 container agent.
3. Bộ điều phối (Container Orchestrator như Docker Swarm / Kubernetes) thấy liveness probe thất bại, suy đoán rằng cả 3 container đã bị treo hoàn toàn và lập tức phát lệnh **restart (kill & start lại)** đồng loạt cả 3 container.
4. Khi cả 3 container đang trong quá trình khởi động lại từ đầu, hệ thống rơi vào trạng thái "chết trắng" (100% downtime), không còn instance nào để phục vụ người dùng.
5. Khi Redis vừa hồi phục sau 30 giây, các container vẫn đang chật vật khởi động lại và có thể rơi vào vòng lặp crashloop back-off nếu chịu tải dồn ứ.
*(Nếu tách riêng: `/health` vẫn trả 200 process còn sống nên không bị restart vô ích; còn `/ready` trả 503 để load balancer tạm thời không đẩy request vào cho đến khi Redis kết nối lại).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi dùng Redis (Stateless): `history_length` tăng đều đặn và liên tục theo số lượt hội thoại: 0 -> 2 -> 4 -> 6 -> 8... bất kể request rơi vào container nào.
- Nếu lưu trong một dict Python (Stateful in-memory): Mỗi container sở hữu một bộ nhớ RAM riêng biệt không chia sẻ với nhau. Khi load balancer phân phối các request luân phiên (round-robin) qua 3 container A, B, C:
  - Lượt 1 vào container A: `history_length` = 0 (A lưu 2 lượt).
  - Lượt 2 vào container B: `history_length` = 0 (vì RAM của B chưa có lịch sử của user này!).
  - Lượt 3 vào container C: `history_length` = 0.
  - Lượt 4 lại vào container A: `history_length` = 2.
  - Kết quả là `history_length` nhảy lộn xộn không thể dự đoán (0, 0, 0, 2, 2, 4...), khiến agent trả lời mất ngữ cảnh và người dùng thấy AI bị "mất trí nhớ" bất thường.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Health check timeout / Port binding fail khi deploy lên nền tảng Cloud (Render / Railway).
- **Thông báo lỗi:** `Deployment failed: Container failed to respond to HTTP pings on port 8000. Port binding timeout`.
- **Cách tìm ra nguyên nhân:** Xem runtime log trên dashboard của platform, phát hiện ra platform tự động gán một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000` hoặc `PORT=35421`), trong khi Dockerfile ban đầu hardcode `--port 8000`. Do đó proxy biên của platform không thể kết nối tới app.
- **Cách sửa:** Sửa chỉ thị `CMD` trong `Dockerfile` sang dạng shell wrapper để nhận biến môi trường động: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Khi đó app sẽ ưu tiên đọc `$PORT` của cloud platform và fallback về 8000 khi chạy local.
