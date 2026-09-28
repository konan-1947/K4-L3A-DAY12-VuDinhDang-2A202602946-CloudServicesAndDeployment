# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Đình Đăng  Mã học viên: 2A202602946

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu quên đặt `AGENT_API_KEY` trên cloud, app dừng ngay khi khởi động và log báo thiếu biến bắt buộc. Nhờ vậy mình phát hiện lỗi cấu hình lúc deploy, trước khi mở API cho người dùng. Nếu mặc định là `changeme`, app vẫn chạy và có thể bị gọi bằng khóa công khai đó.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON mình lấy được khi gọi `/ask` qua `TestClient` với câu “Deploy là gì?”:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:18:52.361898+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}
> ```
>
> Từ đây mình có thể lọc hoặc đếm số lượt theo `event`/`user_id`, và tổng hợp token/chi phí theo thời gian. Với một câu `print` tự do, máy khó tách các trường để truy vấn như vậy.

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
| 1 stage (bản đầu) | 1.69 GB |
| Multi-stage | 247 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Đo bằng `docker images`, multi-stage giảm khoảng **1.44 GB**. Bản một stage dùng base `python:3.11` đầy đủ; bản cuối của multi-stage dùng `python:3.11-slim` và chỉ nhận các dependency đã cài, không mang theo toàn bộ base image đầy đủ hay nội dung builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile copy `requirements.txt` rồi cài thư viện trước khi copy `app/` và `utils/`. Vì vậy, sửa một ký tự trong `app/main.py` chỉ làm lại các layer copy source và những layer phía sau; layer cài dependency được lấy từ cache. Nếu `COPY . .` đứng trước `pip install`, mọi sửa code sẽ làm mất cache ở bước cài và chạy pip lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu có lỗ hổng chạy lệnh, kẻ tấn công chạy được lệnh với quyền của tiến trình Python. Chạy container bằng root cho họ UID 0 và quyền cao bên trong container; nếu còn mount host nhạy cảm, cấp capability rộng hoặc có lỗi ở lớp container/kernel thì tác động có thể lan tới host. `USER app` bỏ quyền root của tiến trình và giảm quyền truy cập đó. Nó giảm rủi ro, còn ranh giới container vẫn do Docker/Linux áp đặt.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với giới hạn 10 request mỗi phút theo phút đồng hồ, có thể gửi 10 request ngay trước ranh giới phút, rồi 10 request ngay sau ranh giới đó. Tổng cộng là 20 request trong khoảng hai giây mà vẫn nằm trong hai phút lịch khác nhau. Sliding window kiểm tra 60 giây gần nhất nên chặn nhóm thứ hai cho tới khi đủ request cũ hết hạn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất request; cost guard giới hạn tổng tiền theo user trong tháng. Ví dụ user còn quota request nhưng một request dự kiến vượt phần ngân sách còn lại thì cost guard phải chặn. Ngược lại, một user gửi dồn nhiều request rẻ trong một phút có thể bị rate limit chặn dù ngân sách tháng còn nhiều.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối thì `/ready` của cả ba container trả 503 và load balancer ngừng gửi request mới tới chúng; `/health` vẫn 200 vì process còn sống. Nếu gộp probe và dùng kết quả Redis làm liveness, orchestrator đánh dấu cả ba unhealthy rồi restart chúng. Redis vẫn mất kết nối, nên instance mới cũng không qua probe và cụm tiếp tục restart thay vì chờ dependency hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi `/ask` lưu user và assistant thành hai message. Vì response trả `history_length` trước khi ghi lượt mới, các lần gọi liên tiếp thường cho `0`, `2`, `4`, ... dù request đi qua instance nào. Nếu dùng dict riêng trong từng process, giá trị sẽ phụ thuộc instance: có thể quay về `0` hoặc tăng theo từng cặp trên instance đó, thay vì có một lịch sử thống nhất.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi kiểm tra service Render, lần gọi đầu tới `/health` timeout sau 20 giây. Mình gọi `/ready` tiếp thì nhận 200 với `{"status":"ready","redis":true}`, rồi gọi lại `/health` cũng nhận 200 với `{"status":"ok"...}`. Vì Redis đã sẵn sàng và health kế tiếp phản hồi, mình nghĩ khả năng cao service đang thức dậy sau khi idle; lần timeout đầu không đủ để kết luận app hoặc Redis bị lỗi. Nếu gặp lại lúc deploy, bước tiếp theo là đối chiếu thời điểm đó với runtime log trên Render.
