# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Kiều Hồng Phong  Mã học viên: 2A202601020

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử khi deploy lên Production, mình quên cấu hình biến `API_TOKEN`. Nếu để giá trị mặc định là `"changeme"`, app vẫn chạy bình thường. Hậu quả là bất kỳ kẻ tấn công nào dò được mật khẩu "changeme" đều có thể gọi API thoải mái, gây lộ dữ liệu hoặc tốn tiền API (LLM) của mình. Việc "chết sớm" giúp mình phát hiện ngay lập tức lỗi quên cấu hình từ lúc deploy, ngăn chặn việc app lên sóng với cấu hình không an toàn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event": "request_finished", "method": "POST", "path": "/chat", "status": 200, "duration_ms": 142}`
> 1. Có thể query/lọc log dễ dàng trên các hệ thống quản lý log như Datadog/ELK (ví dụ: tìm tất cả request có `status >= 400` hoặc `duration_ms > 1000`).
> 2. Có thể phân tích thống kê tự động (tính trung bình thời gian phản hồi, đếm tổng số lượng request) vì các dữ liệu đã được tách sẵn thành key-value, không cần phải dùng Regex phức tạp như khi dùng log text thường.

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
| 1 stage (bản đầu) | ~800 MB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng lớn chênh lệch ở bản 1-stage là do nó chứa toàn bộ các công cụ dùng để build phần mềm (compiler như gcc, các thư viện headers, build-essential) và cache rác của pip. Ở bản multi-stage, ta chỉ copy những thư viện đã build xong sang một image cực kỳ gọn nhẹ (slim/alpine) để chạy runtime, vứt bỏ toàn bộ những rác sinh ra trong quá trình build đi.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa `app/main.py`: Các layer từ đầu cho đến `COPY requirements.txt` và `RUN pip install` đều không thay đổi nên được Docker lấy từ Cache. Chỉ có layer `COPY . .` và các lệnh phía sau nó mới bị chạy lại.
> Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất kỳ thay đổi nhỏ nào trong code (ví dụ sửa 1 dòng ở `main.py`) cũng sẽ làm mất hiệu lực (invalidate) bộ nhớ đệm của lệnh `COPY . .`, kéo theo việc lệnh `RUN pip install` phía dưới bắt buộc phải download và cài đặt lại toàn bộ thư viện từ đầu, làm quá trình build cực kỳ chậm chạp.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: Khi code Python có lỗ hổng (như RCE - Remote Code Execution), kẻ tấn công khai thác để thực thi lệnh shell bên trong container dưới quyền `root`. Nếu container bị cấu hình lỏng lẻo (chạy privileged hoặc share volume), kẻ tấn công dùng quyền root này để escape ra ngoài, chiếm quyền root của toàn bộ máy vật lý (host).
> Lệnh `USER appuser` cắt đứt chuỗi này ở chỗ: Dù hacker có khai thác được RCE, thì lệnh cũng chỉ được thực thi dưới quyền một tài khoản `appuser` (rất thấp), không thể chỉnh sửa file hệ thống hay escape ra khỏi container để hại máy host được.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> - Phải kèm header `WWW-Authenticate: Bearer`: Đây là tiêu chuẩn của giao thức HTTP (RFC 6750). Nó báo cho Client biết server đang yêu cầu xác thực bằng cơ chế Bearer token, giúp Client (hoặc các thư viện HTTP) biết cách xử lý và gửi lại token cho đúng.
> - Trả cùng một thông báo lỗi: Nhằm tránh "Rò rỉ thông tin" (Information Leakage). Nếu báo rõ "sai token" hay "thiếu header", hacker có thể dựa vào đó để dò tìm và đoán cách thức hệ thống xác thực. Trả về lỗi chung chung giúp hệ thống an toàn hơn trước các cuộc tấn công Brute-force.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> - Client im lặng 10 phút rồi gửi liên tiếp thì nó gửi được tối đa **10 request** trước khi bị 429 (vì cái xô chỉ chứa được tối đa capacity = 10 token).
> - Nếu bỏ đoạn `min(capacity, ...)`: Con số đó sẽ thành **100 request**. Lý do là vì số lượng token sẽ được cộng đồn không giới hạn (10 phút x 10 token/phút = 100 token). Khi đó client có thể bắn dồn dập 100 request cùng lúc, làm sập server, phá vỡ hoàn toàn mục đích chặn Burst Traffic của thuật toán Token Bucket.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> - Hạn mức $30/tháng: Thiệt hại tối đa có thể mất trọn vẹn $30 chỉ trong vài tiếng đầu tiên của ngày đầu tháng. Sau đó service sẽ "chết đói" (hết sạch ngân sách) và không thể gọi được API trong 29 ngày còn lại. Không thể tự hồi phục trong tháng đó.
> - Hạn mức $1/ngày: Thiệt hại tối đa của ngày hôm đó bị chặn ở mức $1. Service sẽ tự hồi phục (có lại token để gọi API tiếp) vào lúc 0h00 của ngày hôm sau khi ngân sách được nạp lại.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Liveness probe (điểm kiểm tra gộp chung) kết nối thử tới Redis nhưng bị Timeout.
> 2. Ứng dụng báo lỗi 503.
> 3. Hệ thống Cloud/K8s tưởng rằng cả 3 container App đều đã "chết lâm sàng" (chết tiến trình).
> 4. Hệ thống Cloud lập tức Kill (tiêu diệt) và Restart toàn bộ 3 container App.
> 5. Container App mới khởi động lên, lại cố check Redis (vẫn đang mất kết nối), lại báo 503, rồi lại bị Kill.
> Vòng lặp Crash Loop diễn ra. Kết quả: App bị khởi động lại liên tục vô ích thay vì chỉ đơn giản là tạm ngừng nhận Request chờ Redis sống lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Lỗi gặp: Request `/readyz` bị báo lỗi 503, log ghi `Redis ping failed: TimeoutError: Timeout connecting to server`.
> - Nguyên nhân tìm được (nhờ log): Server không thể kết nối tới đường truyền mạng `.railway.internal`. Do mình đã tạo Redis Database ở nhầm một Project khác (không nằm cùng phòng với Web App), nên mạng lưới nội bộ của Railway không cho phép thông nhau.
> - Cách sửa: Xóa link cũ, vào đúng cấu hình Project `Day12`, tạo một cục Redis mới. Lấy biến `REDIS_URL` của cục Redis mới sinh ra rồi cập nhật ngược lại cho Web App. Mọi thứ đã xanh mượt!
