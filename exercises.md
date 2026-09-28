# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Minh Quân  Mã học viên: 2A202602958

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên đặt `AGENT_API_KEY` trên Render, app sẽ dừng lúc khởi động với lỗi thiếu biến cấu hình, nên mình phát hiện ngay trong deploy log và sửa trước khi nhận traffic. Nếu có mặc định `changeme`, app vẫn chạy nhưng người biết khóa mặc định có thể gọi API; mình chỉ phát hiện sau khi đã mở endpoint ra ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một log thật từ lần gọi `/ask` bằng Docker:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-09-28T13:36:42.148747+00:00","user_id":"sv-lab","tokens_in":8,"tokens_out":46,"cost_usd":2.88e-05}
```

Với JSON này, mình lọc được request theo `user_id` để tìm lịch sử của một người và cộng/so sánh `cost_usd` hoặc token theo thời gian để theo dõi chi phí. Một `print("đã trả lời xong")` không có các trường có cấu trúc đó.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chênh lệch chủ yếu do bản đầu dùng base image `python:3.11` đầy đủ và cài package trong cùng stage với runtime; lệnh pip cũng giữ cache cài đặt trong layer. Bản multi-stage dùng `python:3.11-slim`, không mang stage builder sang image cuối và cài pip với `--no-cache-dir`, nên image cuối còn khoảng 271 MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình đổi đúng một dấu câu trong `app/main.py` rồi build lại. Ở multi-stage, Docker dùng lại layer `COPY requirements.txt`, `pip install` và `COPY --from=builder`; layer `COPY app`, các layer sau nó như `COPY utils` và `RUN useradd/chown` chạy lại. Ở Dockerfile một stage ban đầu, `COPY . .` thay đổi nên `RUN pip install` sau nó cũng phải chạy lại dù `requirements.txt` không đổi. Vì vậy đặt cài dependencies trước source giúp sửa code mà không cài lại package.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu có lỗ hổng cho phép chạy lệnh trong app, tiến trình root có quyền đọc/ghi nhiều file hơn và có thể tận dụng quyền/cấu hình container rộng để tìm đường thoát sang host. `USER appuser` giới hạn quyền của tiến trình ngay từ đầu, nên kẻ tấn công không nhận quyền root trong container. Đây là giảm thiểu hậu quả, không thay thế việc vá lỗ hổng hay cô lập container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với giới hạn 10/phút theo fixed window, có thể gửi 10 request ngay trước khi bộ đếm reset ở giây `:00`, rồi gửi thêm 10 request ngay sau reset. Tổng cộng là 20 request trong khoảng 2 giây. Sliding window xét mọi khoảng 60 giây liên tiếp nên không cho burst vượt quá 10 trong tình huống đó.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request trong một khoảng thời gian; cost guard giới hạn tổng chi phí của user trong tháng. Ví dụ user còn ngân sách nhưng gửi burst request liên tục thì rate limit chặn, dù cost guard vẫn cho qua. Ngược lại, một request có thể nằm trong rate limit nhưng vẫn bị cost guard chặn nếu user đã dùng hết ngân sách tháng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu Redis mất kết nối, endpoint gộp sẽ trả lỗi cho cả liveness lẫn readiness. Bộ điều phối có thể đánh dấu cả ba container unhealthy rồi restart chúng; Redis vẫn đang lỗi nên container mới lại fail probe và có thể lặp lại, làm cụm mất instance phục vụ. Tách probe ra thì `/health` vẫn 200 để không restart app, còn `/ready` trả 503 để load balancer tạm ngừng gửi request tới chúng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Mình gọi `/ask` hai lần trên Compose với cùng user `sv-lab`: lần đầu `history_length` là 0, lần sau là 2 vì mỗi lượt trước đã lưu câu hỏi và câu trả lời vào Redis. Với ba instance, Redis giữ chung lịch sử nên `history_length` vẫn tăng dù request tới instance nào. Nếu giữ lịch sử trong dict Python, mỗi instance có bản riêng nên con số có thể quay về 0 hoặc tăng không đều khi request chuyển instance; restart instance cũng làm mất dict đó. Mình chưa chạy ba replica thật vì Compose hiện map cố định `8000:8000` cho từng agent, gây xung đột cổng host.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi mở URL Render ở đường dẫn `/`, mình gặp `{"detail":"Not Found"}`; log cũng ghi `GET /` trả 404, trong khi `/health` vẫn chạy. Nguyên nhân là FastAPI chưa khai báo route gốc, chỉ có `/health`, `/ready` và `/ask`. Mình thêm `GET /` làm trang giới thiệu service và link `/docs`, push commit `7a315e0`; Render sau đó ghi nhận `GET /` trả 200.
