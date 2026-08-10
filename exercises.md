# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lưu Xuân Dũng  Mã học viên: 2A202601774

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> **Trả lời:** Nếu để mặc định là "changeme", khi deploy lên cloud mà quên set biến môi trường, app vẫn khởi động bình thường nhưng ai cũng có thể lên mạng dùng token "changeme" để gọi API lậu, làm tốn tài nguyên và mất tiền. Việc "chết sớm" giúp ta phát hiện ngay lỗi cấu hình và dừng lại để sửa ngay lập tức trước khi có hậu quả xấu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> **Trả lời:** `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T14:30:00Z", "client_id": "sv-test", "duration_ms": 150, "usd_cost": 0.000015}`. Hai việc log JSON làm được: 1) Dễ dàng đẩy vào các hệ thống theo dõi (như Datadog, ELK) để parse tự động, thống kê và vẽ biểu đồ chi phí/thời gian phản hồi. 2) Có thể dùng tool dòng lệnh (như jq) để filter nhanh tìm tất cả request của một user hoặc các request tốn nhiều tiền, thay vì đọc text bằng mắt.

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
> **Trả lời:** 
> - 1 stage: ~1GB.
> - Multi-stage: ~150MB.
> Phần dung lượng chênh lệch chính là các công cụ biên dịch (compiler gcc, C++), các thư viện header của hệ điều hành và file cache tải về của pip. Bản multi-stage chỉ copy thư mục cài đặt thư viện đã build xong sang base image siêu nhẹ (slim), vứt bỏ mọi rác thải sinh ra trong lúc build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> **Trả lời:** Sửa `app/main.py` chỉ làm vô hiệu hóa layer `COPY . .` và các layer sau nó. Layer `RUN pip install` ở trên vẫn lấy lại được từ cache (vì requirements.txt không đổi). Nếu đặt `COPY . .` lên trước `RUN pip install`, mọi thay đổi dù nhỏ nhất trong mã nguồn (vd gõ dấu cách) cũng phá vỡ cache, làm Docker phải chạy tải lại và cài đặt lại toàn bộ thư viện từ đầu tốn rất nhiều thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> **Trả lời:** Nếu chạy quyền root, khi có lỗ hổng RCE (thực thi mã từ xa) trong Python, kẻ tấn công sẽ thao tác trong container với quyền root cao nhất. Từ đó hacker có thể khai thác điểm yếu của kernel để nhảy thoát khỏi container (container breakout) và chiếm toàn quyền kiểm soát máy chủ Host thật sự. Lệnh `USER appuser` cắt đứt rủi ro vì giáng quyền xuống user thường, dập tắt đặc quyền để can thiệp sâu vào hệ thống ngay cả khi chui được vào bên trong.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> **Trả lời:** 401 kèm WWW-Authenticate để tuân thủ chuẩn HTTP (RFC 6750), báo hiệu cho client biết server đang yêu cầu chuẩn xác thực Bearer token. Ta trả cùng một thông báo lỗi vì nguyên tắc bảo mật không tiết lộ manh mối hệ thống; nếu báo rõ "token không tồn tại", hacker sẽ biết để khai thác hoặc quét tấn công Brute-force/dò tìm.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> **Trả lời:** Nó gửi được tối đa đúng 10 request rồi bị chặn mã 429. Nếu bỏ hàm `min`, lượng token nạp vào bucket sẽ không bị giới hạn trần, token có thể dồn lên 100 hay 1000. Lúc này, client có thể gọi "bùng nổ" (burst) hàng trăm request liên tiếp cùng 1 giây, phá vỡ mục đích giới hạn tải, khiến máy chủ quá tải và treo ngay lập tức.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> **Trả lời:** 
> - Với mức $30/tháng: client bị vòng lặp lỗi sẽ đốt sạch toàn bộ quỹ $30 chỉ trong một đêm và service sẽ block hoàn toàn khách hàng đó suốt 29 ngày còn lại của tháng. Hậu quả là gián đoạn dịch vụ kéo dài.
> - Với mức $1/ngày: Client bị lỗi sẽ chỉ thiệt hại đúng 1 USD trong ngày đó rồi bị chặn. Sang rạng sáng hôm sau, quota ngày tự reset, hệ thống tự hồi phục và khách hàng có thể sử dụng lại bình thường. Quản lý chi phí rủi ro tốt hơn rất nhiều.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> **Trả lời:**
> 1) Khi Redis ngắt, `/healthz` kết hợp sẽ báo 503 (không phản hồi tốt).
> 2) Load Balancer/Container Orchestrator (như K8s/Render) thấy `/healthz` lỗi liên tục sẽ ra quyết định tắt (kill) sạch và khởi động lại toàn bộ 3 container (vì tưởng app bị treo/đứng). 
> 3) Kết cục: Toàn bộ service ngưng phục vụ, kể cả những logic có thể hoạt động mà không cần tới Redis. Phân tách `readyz` giúp app rút tên khỏi bộ định tuyến nhưng máy chủ vẫn còn sống yên ổn đợi Redis hồi phục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Trả lời:** Lỗi: Quên cập nhật biến `PORT`. Ban đầu app cấu hình cứng `8000`. Khi deploy lên nền tảng đám mây, cloud cấp cho 1 cổng random (ví dụ 10243) nhưng app vẫn cố nghe ở cổng 8000. Kết quả là health check bị timeout và cloud báo deploy failed. Sửa lỗi bằng cách thay `8000` thành khai báo đọc `os.getenv("PORT", 8000)` để ứng dụng lắng nghe linh hoạt cổng mà nền tảng yêu cầu.
