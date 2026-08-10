# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Khi deploy ứng dụng lên môi trường Production trên Cloud, nếu người làm devops quên thiết lập biến môi trường AGENT_API_KEY. Nếu để giá trị mặc định là "changeme", ứng dụng vẫn khởi động bình thường. Kẻ xấu có thể dò quét và sử dụng key mặc định này để truy cập hệ thống, làm rò rỉ dữ liệu hoặc tiêu tốn ngân sách API LLM. Nhờ nguyên lý Fail-Fast, ứng dụng tung lỗi ValidationError dừng ngay lúc khởi động, ép devops phải cấu hình đúng key thật trước khi ứng dụng có thể phục vụ traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

>Dòng log mẫu: {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T10:05:00+00:00", "user_id": "sv01", "cost_usd": 0.0012, "tokens_in": 15, "tokens_out": 40}
Hai việc làm được với log JSON mà print() không làm được:
-Trích xuất và truy vấn dữ liệu theo trường (ví dụ: truy vấn danh sách người dùng có cost_usd > 0.01 hoặc đếm tổng số request của user_id = 'sv01').
-Cấu hình hệ thống cảnh báo tự động (Alerting) dựa trên thông số định lượng (ví dụ: phát cảnh báo qua Slack/Email khi tỷ lệ log level = 'error' vượt quá 5% trong 5 phút).

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
| Multi-stage | 175 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~835 MB) bao gồm: bộ công cụ biên dịch (compilers như gcc/g++, build-essential), header files của hệ điều hành, pip cache, và các thư viện phát triển (dev tools) vốn chỉ cần thiết ở giai đoạn cài đặt/biên dịch thư viện nhưng hoàn toàn không cần cho việc chạy ứng dụng Python ở giai đoạn runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Khi sửa một ký tự trong `app/main.py`: Các layer cài đặt dependency (`FROM python:3.11-slim AS builder`, `COPY requirements.txt`, `RUN pip install`, `USER appuser`) đều được dùng lại (CACHE) từ lần build trước vì file `requirements.txt` không hề thay đổi. Chỉ có layer `COPY app ./app` và các bước kế tiếp phía sau mới phải chạy lại -> Thời gian build chỉ mất khoảng 1 - 2 giây.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ file code nào (dù chỉ 1 ký tự), Docker layer cache từ dòng `COPY . .` sẽ bị hủy (invalidated). Điều này bắt buộc Docker phải chạy lại lệnh `RUN pip install` bên dưới, tải và cài đặt lại toàn bộ thư viện từ đầu, khiến thời gian build kéo dài thêm vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện tấn công:
> 1. Code Python tồn tại lỗ hổng thực thi lệnh từ xa (RCE) hoặc injection.
> 2. Kẻ tấn công khai thác lỗ hổng để chạy câu lệnh shell độc hại bên trong container.
> 3. Vì container mặc định chạy dưới quyền root (UID 0), kẻ tấn công sở hữu quyền quản trị cao nhất bên trong container.
> 4. Kẻ tấn công lợi dụng quyền root này để khai thác các lỗ hổng container escape (hoặc mount socket Docker, cgroup...) để vượt ra ngoài container và chiếm luôn quyền kiểm soát root trên máy host.
> 
> Lệnh `USER appuser` cắt đứt chuỗi tấn công ở bước 3: Nó ép toàn bộ tiến trình ứng dụng chạy dưới quyền một tài khoản thường bị giới hạn đặc quyền (UID 10001). Ngay cả khi bị khai thác RCE, kẻ tấn công cũng chỉ có quyền của user thường trong container, không thể đọc/ghi các file hệ thống quan trọng hay thực hiện các kỹ thuật container escape cần quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
> - Cách đạt được: Với cơ chế đếm theo phút cố định (reset ở giây 00), người dùng gửi 10 request ở giây 59 của phút thứ nhất (10:00:59) và tiếp tục gửi 10 request ở giây 00 của phút thứ hai (10:01:00). Vì đồng hồ vừa chuyển sang phút mới nên hạn mức 10/phút được reset lại 0, hệ thống cho phép cả 10 request sau đi qua. Tổng cộng trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:00), hệ thống đã phải gánh tới 20 request (gấp đôi hạn mức cho phép).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Điểm khác nhau cốt lõi:
>   + **Rate limit**: Giới hạn *số lượng request* (tần suất gọi) trong một khoảng thời gian (ví dụ: tối đa 10 request/phút) để bảo vệ server khỏi bị quá tải/spam/DDoS.
>   + **Cost guard**: Giới hạn *tổng chi phí tiền bạc (USD)* phát sinh trong tháng để bảo vệ ngân sách, tránh bị "cháy túi" do gọi LLM.
>
> - Tình huống Rate limit cho qua nhưng Cost guard phải chặn:
>   + Người dùng gọi API với tần suất rất thấp (chỉ 1 request/phút, hoàn toàn nằm trong hạn mức 10 request/phút), nhưng mỗi request đính kèm một file tài liệu khổng lồ (200.000 tokens), tốn 2 USD/request. Đến request thứ 6 trong tháng (tổng $12), Cost guard phát hiện vượt quá ngân sách $10/tháng nên chặn lại (trả lỗi 402), dù Rate limit vẫn cho qua.
>
> - Tình huống Cost guard cho qua nhưng Rate limit phải chặn:
>   + Người dùng gửi 11 request liên tiếp chỉ trong vòng 5 giây với câu hỏi siêu ngắn (1 token/request, chi phí $0.00001). Tổng chi phí cực kỳ nhỏ ($0.0001, thừa ngân sách $10), nhưng vì gửi 11 request trong 5 giây vượt quá hạn mức 10 request/phút nên Rate limit chặn lại ngay lập tức ở request thứ 11 (trả lỗi 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự các sự kiện xảy ra:
> 1. Dịch vụ Redis bị gián đoạn mạng hoặc quá tải trong 30 giây.
> 2. Endpoint gộp kiểm tra Redis và thất bại, trả về HTTP Status 500/503.
> 3. Hệ thống Orchestrator (Docker/Kubernetes) nhận thấy Liveness Probe bị fail và kết luận cả 3 container agent đều đã "chết".
> 4. Orchestrator lập tức ra lệnh **khởi động lại (restart) đồng loạt cả 3 container agent**.
> 5. Trong thời gian 30 giây các container đang bị tắt đi và khởi động lại, Redis phục hồi kết nối. Tuy nhiên, vì cả 3 container agent đều đang trong quá trình boot up nên không có bất kỳ instance nào sẵn sàng nhận request -> Hệ thống sập hoàn toàn (Total Outage).
> 6. Sự cố gián đoạn nhỏ 30s của Redis đã biến thành một sự cố sập toàn bộ dịch vụ do khởi động lại dây chuyền.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - Nếu lưu lịch sử trong một dict Python (RAM trong process): Con số `history_length` sẽ thay đổi nhảy múa thất thường và không tăng dần ổn định (ví dụ: request 1 = 0, request 2 = 0, request 3 = 2, request 4 = 2, request 5 = 0...).
> - Lý do: Load Balancer phân phối các request luân phiên (Round-Robin) tới 3 container khác nhau (Agent 1, Agent 2, Agent 3). Mỗi container sở hữu một bộ nhớ RAM riêng độc lập. Nếu Request 1 vào Agent 1 (lịch sử RAM Agent 1 tăng lên 2), nhưng Request 2 lại rơi vào Agent 2 (bộ nhớ RAM Agent 2 vẫn là 0), dẫn đến việc Agent bị "mất trí nhớ" ngẫu nhiên giữa các lượt hỏi.
> - Ngược lại khi dùng Redis (Stateless): Cả 3 container cùng truy vấn và cập nhật vào một Redis tập trung, giúp `history_length` luôn tăng dần ổn định (0 -> 2 -> 4 -> 6 -> 8...) dù request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Thông báo lỗi gặp phải: `Error: ConnectionRefusedError: [Errno 111] Connect call failed ('127.0.0.1', 6379)` khi ứng dụng thực hiện Readiness Probe `/ready` trên Cloud.
> - Nguyên nhân tìm ra: Ứng dụng trên Cloud cố gắng kết nối tới `REDIS_URL=redis://localhost:6379/0`. Tuy nhiên trên môi trường Cloud (Railway/Render), dịch vụ Redis chạy ở một container/instance riêng biệt có hostname khác, chứ không chạy chung trên `localhost` của container ứng dụng.
> - Cách sửa chữa: Đặt lại biến môi trường `REDIS_URL` trên Dashboard của Cloud Platform trỏ tới chuỗi kết nối của dịch vụ Redis (ví dụ: `redis://default:password@redis.railway.internal:6379` hoặc URL riêng do Platform cung cấp). Sau khi cập nhật biến môi trường và redeploy, endpoint `/ready` lập tức trả về HTTP Status 200 `{"status": "ready", "redis": true}`.
