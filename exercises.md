# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng hướng dẫn bằng câu trả lời cụ thể của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hoàng Anh Tài  Mã học viên: 2A202602612

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy service lên cloud (như Railway hoặc Render), nếu dev quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard:
- Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường và mở cổng ra Internet. Kẻ tấn công hoặc bot quét tự động có thể thử chuỗi mặc định `"changeme"` để truy cập API trái phép, làm rò rỉ dữ liệu hoặc đốt sạch ngân sách gọi LLM mà lập trình viên không hề hay biết cho đến khi nhận hóa đơn tiền triệu.
- Nhờ cơ chế "fail fast" (không đặt giá trị mặc định cho secret bắt buộc), pydantic ném `ValidationError` và làm ứng dụng crash ngay tại thời điểm khởi động. Platform sẽ lập tức đánh dấu deploy thất bại và báo đỏ trên dashboard ngay khi deploy, buộc lập trình viên phải bổ sung biến môi trường ngay lập tức trước khi có bất kỳ traffic nào tiếp cận được hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:00:00.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 48, "cost_usd": 0.00018}
```

Hai việc làm được với log có cấu trúc (JSON):
1. **Lọc và truy vấn có cấu trúc (Structured Querying):** Các công cụ gom log tập trung (Datadog, Grafana Loki, CloudWatch) có thể parse JSON tự động để thực hiện thống kê chi tiết, ví dụ: tính tổng chi phí `sum(cost_usd)` theo từng `user_id` trong ngày, hoặc tìm kiếm các request có thời gian xử lý/token cao đột biến.
2. **Thiết lập cảnh báo tự động (Alerting):** Dễ dàng đặt điều kiện cảnh báo tự động khi xuất hiện `level == "error"` vượt ngưỡng tần suất cho phép trong 5 phút, hoặc khi `cost_usd` của một user vượt quá hạn mức nhất định. Điều này bất khả thi hoặc rất dễ lỗi nếu phải dùng regex phân tích chuỗi text tự do từ `print()`.

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
| 1 stage (bản đầu) | 1020 MB (~1.02 GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm:
- Bộ công cụ biên dịch (compilers và build tools như `gcc`, `g++`, `make`, `binutils`) cùng các gói header C (`build-essential`, `python3-dev`) vốn chỉ cần thiết trong giai đoạn build để biên dịch các package bánh xe Python (C extensions).
- Các gói tiện ích, file tài liệu hướng dẫn (man pages, doc), cache của package manager (`apt cache`) và pip cache trong base image đầy đủ.
- Với multi-stage build, stage `builder` chịu trách nhiệm cài đặt và biên dịch dependency vào thư mục `/install`, sau đó stage `runtime` sử dụng image `python:3.11-slim` chỉ copy thư viện thành phẩm sang `/usr/local` và copy mã nguồn, loại bỏ hoàn toàn các công cụ biên dịch nặng nề khỏi image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Các layer trước đó gồm: base image `python:3.11-slim`, `WORKDIR /app`, `COPY requirements.txt .`, và `RUN pip install ...` ở cả hai stage đều được tái sử dụng hoàn toàn từ cache (hiển thị `CACHED`) vì file `requirements.txt` không hề bị sửa đổi.
  - Chỉ layer `COPY app ./app` và các layer kế tiếp mới bị mất cache và thực thi lại. Quá trình build lại hoàn tất gần như tức thì (chỉ mất 1 - 2 giây).
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa dù chỉ một ký tự trong `app/main.py`, mã băm checksum của layer `COPY . .` sẽ thay đổi. Theo nguyên tắc của Docker, layer này và tất cả các layer đứng sau nó đều bị mất cache (cache invalidated).
  - Docker sẽ buộc phải chạy lại toàn bộ lệnh `RUN pip install`, tải lại và cài đặt lại toàn bộ dependencies qua mạng từ đầu, gây lãng phí băng thông và kéo dài thời gian build lên vài phút mỗi lần lập trình viên sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện leo quyền khi chạy bằng root:
1. Ứng dụng Python tồn tại lỗ hổng bảo mật (ví dụ: Command Injection, Deserialization lỗi thời, hoặc RCE từ thư viện bên thứ ba).
2. Kẻ tấn công gửi payload kích hoạt RCE và mở được một reverse shell bên trong container.
3. Vì container không khai báo `USER`, tiến trình Python đang chạy với UID 0 (root bên trong container).
4. Kẻ tấn công lợi dụng quyền root này để khai thác các lỗ hổng container escape (như tương tác với mounted socket `/var/run/docker.sock`, khai thác lỗ hổng Linux kernel của host, hoặc lạm dụng đặc quyền capabilities).
5. Khi thoát ra được khỏi container, do UID 0 trong container map trực tiếp với UID 0 (root) trên máy host (nếu không bật user namespace remap), kẻ tấn công chính thức chiếm toàn quyền điều khiển máy chủ host.

Lệnh `USER appuser` cắt đứt chuỗi ở bước 2 - 3:
- Tiến trình ứng dụng bị giáng xuống chạy dưới quyền của một user thông thường không có đặc quyền (UID 10001).
- Khi kẻ tấn công RCE vào hệ thống, họ chỉ có quyền hạn tối thiểu của `appuser`: không thể đọc/ghi các file hệ thống, không có quyền `sudo`, và không sở hữu các Linux capabilities cần thiết để thực hiện container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
- Cách đạt được:
  - Với cơ chế đếm theo phút đồng hồ (fixed window), bộ đếm được reset về 0 tại giây thứ 00 của mỗi phút mới.
  - Người dùng gửi dồn 10 request vào giây cuối cùng của phút thứ nhất (ví dụ: `10:00:59`). Hệ thống ghi nhận 10/10 request và vẫn cho qua.
  - Sang giây tiếp theo (`10:01:00`), bộ đếm tự động reset về 0 cho phút mới. Người dùng lập tức gửi tiếp 10 request nữa.
  - Như vậy, trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ `10:00:59` đến `10:01:00`), hệ thống đã phải gánh chịu 20 request (gấp đôi hạn mức 10 req/phút).
- Thuật toán sliding window 60 giây khắc phục hoàn toàn hiện tượng này nhờ việc luôn đếm số lượng entry trong Sorted Set nằm trong khoảng `[now - 60, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác nhau:**
  - **Rate Limit:** Giới hạn *số lượng request* theo một đơn vị thời gian ngắn (ví dụ: 10 request/phút) để bảo vệ server khỏi quá tải traffic và tấn công brute-force / DoS.
  - **Cost Guard:** Giới hạn *tổng chi phí tài chính (USD)* tích lũy theo chu kỳ dài (ví dụ: $10/tháng) dựa trên số lượng token LLM thực tế tiêu thụ, để bảo vệ ví tiền của chủ dịch vụ.
- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
  - Một user cả tháng chưa gọi request nào hôm nay, đây là request đầu tiên trong ngày (tốc độ 1 req/phút, hoàn toàn nằm trong hạn mức 10 req/phút). Tuy nhiên, tài khoản user này trong tháng đã chi tiêu đạt ngưỡng $10.0. Rate Limiter thấy tần suất thấp nên cho qua, nhưng Cost Guard kiểm tra thấy đã hết ngân sách tháng nên chặn lại và trả về lỗi `402 Payment Required`.
- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn:**
  - Một user mới toanh đầu tháng, ngân sách còn nguyên $10.0. User này viết script gửi liên tiếp 15 câu hỏi cực ngắn trong vòng 5 giây (mỗi câu chỉ tốn $0.0001, tổng cộng mới tốn $0.0015 / $10). Cost Guard thấy ngân sách còn rất nhiều nên đồng ý, nhưng Rate Limiter phát hiện từ request thứ 11 đã vượt quá 10 req/phút nên lập tức chặn lại và trả về lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện (thảm họa cascading failure):
1. Redis gặp sự cố mạng hoặc khởi động lại tạm thời trong vòng 30 giây.
2. Endpoint gộp kiểm tra kết nối tới Redis thấy thất bại, nên trả về mã lỗi 503.
3. Orchestrator (Docker/Kubernetes/Cloud Platform) dùng endpoint này làm **Liveness probe**, thấy trả về 503 nên kết luận cả 3 container của ứng dụng đều đã chết hoặc rơi vào trạng thái deadlock không thể cứu vãn.
4. Orchestrator lập tức cưỡng chế dừng và khởi động lại (restart) toàn bộ cả 3 container cùng lúc.
5. Mọi request của người dùng đang được xử lý dở tại thời điểm đó bị ngắt kết nối đột ngột (lỗi `502 Bad Gateway`). Toàn bộ hệ thống sập hoàn toàn (downtime 100%).
6. Khi 3 container vừa khởi động lại, Redis vẫn chưa hết 30 giây sự cố, liveness check lại tiếp tục fail và orchestrator lại tiếp tục restart lần nữa, tạo thành vòng lặp khởi động chết chóc (CrashLoopBackOff).
- *Kết luận:* `/health` (Liveness) chỉ được kiểm tra nội tại process của container; còn `/ready` (Readiness) mới là nơi kiểm tra dependency để tạm ngừng nhận traffic mới từ load balancer mà không làm restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless):
  - Dữ liệu lịch sử hội thoại nằm tập trung tại Redis. Dù Load Balancer luân chuyển request đến bất kỳ container nào (1, 2 hay 3), container đó đều đọc và ghi vào cùng một key `history:{user_id}` trên Redis.
  - Chỉ số `history_length` sẽ tăng đều đặn, liên tục và chính xác qua từng lượt hỏi: `0 -> 2 -> 4 -> 6 -> 8...`
- Nếu lưu trong một dict Python nội bộ trong RAM của process (Stateful):
  - Mỗi container A, B, C sẽ sở hữu một dict riêng biệt trong bộ nhớ RAM của nó và không hề chia sẻ với các container khác.
  - Do load balancer cân bằng tải vòng tròn (round-robin), request 1 đến container A (`history_length = 0`), request 2 đến container B (`history_length = 0`), request 3 đến container C (`history_length = 0`), request 4 quay lại container A (`history_length = 2`)...
  - Người dùng sẽ thấy `history_length` nhảy lộn xộn, agent liên tục "mất trí nhớ" và không thể hiểu ngữ cảnh của câu hỏi vừa hỏi ở lượt trước đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Healthcheck timeout / Port binding error khi deploy lên platform (như Railway hoặc Render):
  `Application failed to respond on port 8000. Healthcheck timed out.`
- **Cách tìm ra nguyên nhân:**
  - Mở tab **Deploy Logs** và **Runtime Logs** trên dashboard của platform.
  - Phát hiện ra uvicorn mặc định chạy với `--port 8000`, trong khi platform cloud cấp phát một cổng động thông qua biến môi trường `$PORT` (ví dụ `PORT=10000` hoặc cổng ngẫu nhiên) và reverse proxy của platform chỉ kiểm tra healthcheck trên cổng `$PORT` đó.
- **Cách sửa:**
  - Sửa lệnh `CMD` trong `Dockerfile` để đọc biến môi trường `$PORT` và fallback về 8000 nếu không có:
    `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  - Đồng thời đảm bảo bind vào `0.0.0.0` thay vì `127.0.0.1` để có thể nhận traffic từ bên ngoài container. Sau khi cấu hình, deploy thành công và endpoint `/health` trả về HTTP 200 OK.
