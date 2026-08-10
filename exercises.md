# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng hướng dẫn dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Vũ  Mã học viên: 2A202601666

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy mà quên đặt `API_TOKEN`, ứng dụng sẽ dừng ngay lúc khởi động. Nhờ vậy mình phát hiện cấu hình cloud bị thiếu trước khi service nhận request. Nếu mặc định là `changeme`, app vẫn chạy với token dễ đoán và tạo ra lỗ hổng bảo mật khó nhận biết.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ: `{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T07:00:00+00:00","client_id":"sv-test","usd_cost":0.0001}`. Dòng JSON này có thể được hệ thống log lọc/đếm theo event hoặc severity, tính tổng chi phí theo usd_cost và truy vết theo client_id. Một câu print thông thường không có cấu trúc ổn định để làm các việc đó.

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
| 1 stage (bản đầu) | khoảng 1.8 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage khoảng 1.8 GB; bản multi-stage mình đo bằng `docker images` là 270 MB. Bản một stage dùng base Python đầy đủ và giữ các thành phần build, cache pip cùng file không cần khi chạy. Multi-stage chỉ chép dependency và source cần thiết sang runtime `python:3.11-slim`, nên loại bỏ phần build thừa. Con số 1.8 GB là số đo tham chiếu của Dockerfile một stage ban đầu trong đề bài.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer từ `FROM` đến `RUN pip install` và layer cài user được dùng lại từ cache; các layer copy source và phần sau đó build lại. Vì `requirements.txt` được copy và cài trước source code nên sửa code không làm cài lại dependency. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source làm layer COPY đổi và kéo theo pip install chạy lại, khiến build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng cho phép kẻ tấn công thực thi lệnh trong process. Nếu process chạy bằng root, các lệnh đó có quyền đọc/ghi nhiều file, cài phần mềm hoặc truy cập tài nguyên container; nếu khai thác tiếp được lỗi cô lập thì rủi ro trên host nghiêm trọng hơn. `USER appuser` chạy process bằng UID 10001 không có quyền quản trị, cắt chuỗi leo thang quyền trước bước có quyền root trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` cho client biết endpoint yêu cầu cơ chế Bearer và phải gửi lại header Authorization đúng dạng. Dùng cùng một thông báo cho thiếu header, sai scheme và sai token không để lộ cho người dò quét biết token có tồn tại hay request sai ở bước nào. Đây là cách giảm thông tin phục vụ brute-force, trong khi vẫn trả đúng mã 401 và header hướng dẫn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Token bucket có tối đa 10 token. Sau 10 phút, lượng tính theo refill là 10 + 10×10 = 110 nhưng `min(capacity, ...)` giới hạn còn 10, nên client gửi được 10 request rồi request tiếp theo nhận 429. Nếu bỏ `min` thì bucket tích được 110 token và client gửi được 110 request liên tiếp, phá giới hạn burst và có thể dồn tải vào backend.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, sự cố bắt đầu lúc 2h sáng có thể tiêu gần 30 USD trước khi bị chặn và chỉ tự mở lại khi sang tháng mới. Với hạn mức 1 USD/ngày, thiệt hại tối đa trong ngày đó là 1 USD; sang ngày UTC tiếp theo bộ đếm hết hạn và service tự cho phép dùng ngân sách mới. Hạn mức theo ngày giới hạn tốt hơn phạm vi thiệt hại.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/healthz` cũng kiểm tra Redis, khi Redis mất kết nối cả ba container sẽ bị health check đánh dấu unhealthy. Load balancer ngừng gửi traffic; orchestrator có thể restart hoặc thay cả ba container dù process vẫn chạy, rồi phải chờ Redis trở lại để chúng pass health check. Tách `/healthz` (process còn sống) khỏi `/readyz` (Redis sẵn sàng) tránh restart hàng loạt khi dependency chỉ tạm thời lỗi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi mình gặp là Railway không khởi động đúng vì lệnh start nhận literal `$PORT`, trong khi cloud cấp cổng động. Mình xem deployment log trên Railway và đối chiếu `railway.toml`, xác định shell chưa expand biến. Mình sửa start command thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'`, deploy lại rồi kiểm tra `/healthz`, `/readyz` và `/chat`. Railway báo deployment successful và các endpoint public trả đúng.
