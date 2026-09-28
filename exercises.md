# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Ngọc Vinh  Mã học viên: L3A2026002833

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên cloud nhưng quên cấu hình `AGENT_API_KEY`.
> Nếu có mặc định `"changeme"`, service vẫn báo healthy và người lạ có thể dùng
> khóa dễ đoán để gọi `/ask`, làm phát sinh chi phí. Khi trường này là bắt buộc,
> tiến trình dừng ngay với lỗi cấu hình nên tôi phát hiện sai sót trong log deploy
> trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được có dạng:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T00:00:00+00:00","user_id":"sv-test","tokens_in":5,"tokens_out":30,"cost_usd":0.00001875}`.
> Từ các trường có cấu trúc, tôi có thể nhóm và cộng `cost_usd` theo `user_id`,
> đồng thời lập cảnh báo theo tỷ lệ event lỗi trong một khoảng thời gian. Dòng
> `print("đã trả lời xong")` không chứa dữ liệu để làm hai việc đó.

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
| 1 stage (bản đầu) | Chưa hoàn tất đo — tải base image đầy đủ quá chậm |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch dự kiến chủ yếu là base image Python đầy đủ, cache của `pip`,
> compiler/header và các công cụ chỉ cần lúc build. Bản multi-stage cài package
> vào `/install`, rồi chỉ chép kết quả runtime sang `python:3.11-slim`, nên không
> mang các công cụ build sang image cuối. Bản multi-stage đã được đo thật là
> 271 MB, dưới ngưỡng 500 MB. Build đối chứng single-stage đã được chạy nhưng
> dừng khi layer base hơn 350 MB tải quá chậm, nên tôi không ghi một số giả.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Docker có thể dùng lại các layer base image, `WORKDIR`, `COPY requirements.txt`
> và `RUN pip install` vì file dependency không đổi. Từ layer `COPY app ./app`
> trở đi phải tạo lại do source thay đổi. Nếu đặt `COPY . .` trước `pip install`,
> chỉ một ký tự trong source cũng làm mất cache của layer copy và tất cả layer
> phía sau, khiến dependency bị cài lại không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với UID
> của tiến trình trong container. Khi tiến trình là root, một lỗi cấu hình runtime
> hoặc lỗ hổng thoát container có thể biến quyền đó thành quyền cao trên host.
> `USER appuser` chuyển tiến trình sang UID 10001 trước khi app chạy, nên mã bị
> chiếm quyền chỉ có đặc quyền của user thường và giảm đáng kể phạm vi thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request: gửi 10 request sát cuối phút, ví dụ 10:00:59, rồi gửi
> thêm 10 request ngay sau khi bộ đếm reset ở 10:01:00. Hai nhóm nằm ở hai phút
> lịch khác nhau nhưng xảy ra trong khoảng gần 2 giây. Cửa sổ trượt vẫn nhìn cả
> 60 giây gần nhất nên không tạo ra khe hở ở ranh giới phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit khống chế số request trong 60 giây, còn cost guard khống chế tổng
> USD theo tháng. Một user gửi ít request nhưng prompt rất dài có thể vẫn nằm
> dưới rate limit trong khi vượt ngân sách và bị cost guard chặn. Ngược lại, một
> loạt request rất ngắn có tổng chi phí còn thấp nhưng gửi dồn dập sẽ bị rate
> limiter trả 429 trước khi chạm giới hạn tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả 503. Orchestrator hiểu nhầm rằng cả ba
> process đã chết nên lần lượt restart cả ba container. Trong lúc Redis chưa trở
> lại, container mới cũng tiếp tục fail health check và bị restart, khiến cụm
> không còn instance ổn định để phục vụ cả những endpoint không cần Redis. Tách
> `/health` và `/ready` giúp load balancer chỉ rút traffic, không restart process.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi request thêm hai message nên `history_length` mà lượt
> sau đọc được tăng 0, 2, 4, 6... dù request vào instance nào. Nếu dùng dict trong
> từng process, mỗi container có một lịch sử riêng; qua load balancer tôi sẽ thấy
> con số nhảy không đều như 0, 0, 2, 0, 2 hoặc giảm xuống khi request chuyển sang
> instance khác. Khi container restart, lịch sử trong dict còn trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Trong bước chuẩn bị deploy, lệnh kiểm tra Docker báo `failed to connect to the
> Docker API ... dockerDesktopLinuxEngine` vì Docker Desktop/daemon chưa chạy.
> Tôi xác định nguyên nhân bằng `docker info`: lỗi xảy ra trước khi đọc Dockerfile,
> nên không phải lỗi source. Cách sửa là khởi động Docker Desktop, chờ engine sẵn
> sàng rồi chạy lại build và compose. Đây mới là lỗi local chuẩn bị deploy; sau
> khi có URL cloud tôi cần bổ sung lỗi deploy thật nếu platform phát sinh lỗi khác.
