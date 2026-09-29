# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng placeholder bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Danh Gia Minh  Mã học viên: 2A202602441

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên đặt `AGENT_API_KEY` khi deploy, app dừng ngay lúc khởi động thay vì
chấp nhận key mặc định `changeme`. Nhờ vậy mình phát hiện thiếu secret trước
khi mở service; nếu key mặc định được chấp nhận, người khác có thể gọi `/ask`
và làm phát sinh chi phí LLM.


### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON lấy từ lần chạy test `/ask`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:11:27.465874+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Mình có thể lọc request theo `user_id`, và tổng hợp token/chi phí theo user
hoặc thời gian. `print("đã trả lời xong")` không có dữ liệu cấu trúc để lọc
hay tính các chỉ số này.

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
| 1 stage (bản đầu) | 1000 MB |
| Multi-stage | 271 MB (đo bằng `docker images`) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Image multi-stage mình đo được là 271 MB. Bản một stage dùng base
`python:3.11` đầy đủ và giữ cả thành phần build trong image cuối; bản multi-stage
dùng `python:3.11-slim` và chỉ copy dependency từ builder sang runtime. 

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, Docker dùng lại các layer builder cài dependency vì
`requirements.txt` không đổi. Layer copy source code và các layer sau nó phải
chạy lại. Nếu `COPY . .` đặt trước `pip install`, thay đổi code làm layer copy
đổi, nên Docker phải cài lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu lỗ hổng cho phép kẻ tấn công thực thi lệnh, lệnh đó có quyền của process
Python. Khi process chạy root, kẻ tấn công có thể đọc/sửa file trong container
và có cơ hội khai thác cấu hình hoặc lỗ hổng container để ảnh hưởng host. Lệnh
`USER appuser` hạ quyền process bên trong container, giới hạn tác động nếu app
bị chiếm quyền; nó bổ sung chứ không thay thế các lớp cô lập khác của Docker.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây: gửi 10 request ngay trước giây 00, rồi thêm
10 request ngay sau khi phút mới bắt đầu. Bộ đếm theo phút reset giữa hai nhóm
nên cho qua cả hai; sliding window 60 giây vẫn đếm đủ 20 request trong cùng
cửa sổ và chặn phần vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request theo thời gian; cost guard giới hạn chi phí theo
user trong tháng. Một request rất tốn token có thể nằm trong hạn mức request
nhưng làm tổng chi phí dự kiến vượt ngân sách, nên cost guard chặn bằng 402.
Ngược lại, nhiều request rẻ vẫn có thể vượt 10 request/phút khi ngân sách còn,
nên rate limit chặn bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Redis mất kết nối → cả 3 container cùng báo unhealthy nếu liveness kiểm tra
Redis → orchestrator hiểu là process hỏng và restart cả 3 → Redis vẫn mất trong
thời gian đó nên các container mới cũng không phục vụ được → Redis hồi phục
nhưng cụm có thể chưa có instance sẵn sàng. `/health` chỉ kiểm tra process;
`/ready` kiểm tra Redis để load balancer rút instance khỏi traffic mà không
restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis dùng chung, `history_length` tăng đều dù request vào instance nào:
request đầu thấy 0, request kế tiếp thấy 2, rồi 4 vì mỗi lượt thêm message user
và assistant. Nếu lưu trong dict, mỗi container giữ lịch sử riêng; khi request
chuyển container, con số có thể tụt về 0 hoặc dao động theo instance.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Railway báo `/bin/sh: 1: exec: docker-entrypoint.sh: not found` dù build và
push image đã xong. Mình xác định lỗi ở bước khởi động bằng cách đọc deploy
logs rồi đối chiếu `railway.toml` với Dockerfile: image khởi chạy bằng Uvicorn,
không có `docker-entrypoint.sh`, nên Start Command cũ đang được dùng thay cho
cấu hình trong repo. Sau khi đặt Start Command thành
`uvicorn app.main:app --host 0.0.0.0 --port $PORT`, `/health` và `/ready` trả
200, còn `/ask` không có key trả 401.
