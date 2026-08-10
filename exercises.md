# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu trong mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Văn Nhân,  Mã học viên: 2A202601050

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ khi deploy quên set `API_TOKEN`: app dừng ngay với lỗi validation thay vì mở API bằng token `changeme`, nhờ vậy không có bot nào dùng token đoán được để phát sinh chi phí

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:01:52.330206+00:00", "client_id": "reflection-check", "prompt_tokens": 3, "completion_tokens": 37, "usd_cost": 2.265e-05}`. Có thể lọc/tính tổng chi phí theo `client_id`, và đếm hoặc cảnh báo theo `event`/`severity`; `print("đã trả lời xong")` không có trường máy đọc để làm hai việc đó.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73GB |
| Multi-stage | 221MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh khoảng 1.51GB chủ yếu là base image Python đầy đủ, dependency/build artifact và các công cụ chỉ cần lúc build. Multi-stage dùng slim và chỉ chép gói runtime đã cài sang stage cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa `app/main.py` chỉ làm lại layer `COPY app/ app/` và các layer sau nó; layer copy requirements và `RUN pip install` vẫn cache. Nếu `COPY . .` đứng trước pip install thì thay đổi đó làm mất cache từ COPY, nên phải cài lại dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng RCE cho kẻ tấn công chạy lệnh với quyền user của process: nếu process là root trong container, họ có root trong container và có thể khai thác thêm cấu hình nguy hiểm như Docker socket/privileged container hoặc lỗi kernel để leo lên host. `USER appuser` giải quyết vấn đề này ở bước đầu: lệnh từ RCE chỉ có quyền appuser, nên không thể trực tiếp sửa file hệ thống hay cài quyền root trong container, vẫn phải giữ Docker isolation an toàn.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là challenge chuẩn bắt buộc với 401, cho client biết phải xác thực theo scheme nào. Dùng cùng lỗi tránh biến API thành oracle: thông báo chi tiết sẽ tiết lộ header có tồn tại, scheme đã đúng hay token gần đúng cho người đang dò token.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `capacity=10`, client có thể gửi liên tiếp **10 request**, và request thứ **11** sẽ nhận `429`.
> Nếu bỏ `min(capacity, ...)`, sau 10 phút không gửi request, bucket có thể tích lũy tới **100 token** (`10 token/phút × 10 phút`), nên client có thể burst **100 request** liên tiếp. `min(capacity, ...)` đảm bảo số token không bao giờ vượt quá sức chứa bucket, tránh việc thời gian im lặng càng lâu thì burst cho phép càng lớn.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, nếu client gặp sự cố và gọi liên tục từ 2h sáng, nó có thể tiêu hết $30 trước khi bị chặn. Sau đó service sẽ không tự hoạt động lại cho client đó cho đến kỳ ngân sách tháng tiếp theo, trừ khi có can thiệp thủ công.
> Với hạn mức $1/ngày, thiệt hại tối đa trong ngày chỉ là $1. Khi sang ngày mới, hạn mức được reset nên service có thể tự phục hồi vào ngày hôm sau mà không cần can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối => endpoint gộp trả 503 => liveness probe của cả 3 container fail => orchestrator lần lượt restart chúng và load balancer không còn instance healthy để gửi traffic. Trong 30 giây sẽ có gián đoạn không cần thiết; Redis hồi lại thì các container khởi động lại, probe 200 và traffic mới quay lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi kiểm tra URL Render trong `DEPLOYMENT.md`, `pytest tests/test_cp5.py -v` báo lỗi `httpx.ConnectError: [Errno 8] nodename nor servname provided, or not known` khi gọi `/healthz`.
> Em kiểm tra lại URL mà test lấy từ tài liệu và nhận ra request chưa tới được server mà đã lỗi ở bước DNS, nên nguyên nhân nhiều khả năng là domain/Public URL trên Render không đúng hoặc service chưa hoạt động. Cách sửa là kiểm tra lại service trên Render, redeploy nếu cần, cập nhật đúng Public URL trong `DEPLOYMENT.md`, rồi chạy lại `/healthz`, `/readyz` và test CP5 để xác nhận deployment.

