# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Nguyễn Thái Dương  Mã học viên: 2A202602383

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu deploy lên cloud mà quên `AGENT_API_KEY`, ứng dụng sẽ báo lỗi ngay khi khởi động thay vì chạy với một khóa mặc định. Nhờ vậy mình phát hiện lỗi cấu hình trong log deploy ngay, trước khi service nhận request. Nếu dùng giá trị mặc định như `changeme`, service vẫn có thể chạy và người khác có thể đoán hoặc dùng khóa đó để gọi API, gây tốn chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ một log hợp lệ là:
`{"event":"ask_completed","level":"info","timestamp":"2026-09-28T10:00:00+00:00","user_id":"sv-test","tokens_in":12,"tokens_out":30,"cost_usd":0.00002}`

Từ dòng này mình có thể lọc các request theo `user_id` để xem ai gọi nhiều, và cộng `cost_usd` để theo dõi chi phí. Mình cũng có thể lọc theo `event` hoặc `level` để tìm lỗi và thống kê số lần hoàn thành request. Một câu `print()` thông thường chỉ là văn bản, khó lọc theo từng trường và khó dùng cho hệ thống cảnh báo.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Số đo cần bổ sung sau khi build hai image:

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | [điền số đo thực tế] |
| Multi-stage | [điền số đo thực tế] |

Multi-stage giữ dependency đã cài ở stage builder rồi chỉ copy kết quả cần chạy sang stage runtime. Vì vậy compiler, cache cài package và các file trung gian không đi vào image cuối. Base image `python:3.11-slim` cũng nhỏ hơn bản Python đầy đủ, nên image multi-stage thường nhẹ hơn và build/deploy nhanh hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, `COPY requirements.txt` và `RUN pip install` nằm trước khi copy `app` và `utils`. Khi chỉ sửa `app/main.py`, Docker có thể dùng lại layer cài dependency và chỉ tạo lại các layer copy source cùng các layer sau đó. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi trong source sẽ làm layer `COPY` đổi, khiến Docker phải chạy lại `pip install` dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu process trong container có lỗ hổng và container đang chạy bằng root, kẻ tấn công có thể chiếm quyền root trong container. Từ đó họ có thể đọc hoặc sửa các file nhạy cảm, khai thác quyền trên Docker socket nếu được mount, hoặc tìm cách tác động tới host. Lệnh `USER appuser` chuyển process sang user UID 10001 không có quyền quản trị, nên nếu ứng dụng bị khai thác thì quyền của kẻ tấn công bị giới hạn trong phạm vi user đó.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa là 20 request trong khoảng 2 giây: gửi 10 request lúc 10:00:59 và thêm 10 request lúc 10:01:01. Cách đếm theo phút đồng hồ reset lúc giây 00 nên xem hai nhóm này thuộc hai phút khác nhau. Sliding window nhìn vào 60 giây gần nhất nên sẽ thấy cả 20 request và chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ hoặc số lượng request, còn cost guard giới hạn tổng chi phí của một user trong tháng. Ví dụ user mới chỉ gửi 2 request nên rate limit cho qua, nhưng hai request đó có prompt rất dài và đã làm chi phí vượt ngân sách tháng thì cost guard trả 402. Ngược lại, user có thể còn nhiều ngân sách nhưng gửi liên tục quá 10 request trong một phút, khi đó rate limit trả 429 còn cost guard chưa cần chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu `/health` cũng gọi Redis, khi Redis mất kết nối thì cả ba container đều trả health fail. Orchestrator có thể hiểu rằng process của cả ba container đã chết và lần lượt restart chúng, dù nguyên nhân thực tế chỉ nằm ở Redis. Trong thời gian Redis chưa hồi phục, các container mới vẫn fail health, tạo ra việc restart không cần thiết. Tách `/health` chỉ kiểm tra process và `/ready` kiểm tra Redis giúp container vẫn được xem là còn sống nhưng bị loại khỏi traffic mới.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi ba instance cùng dùng Redis, request vào instance nào cũng đọc được cùng key `history:<user_id>`, nên `history_length` tăng theo lịch sử chung. Nếu dùng dict trong Python, mỗi container có một vùng nhớ riêng; request chuyển từ A sang B có thể thấy lịch sử ngắn hơn hoặc bằng 0. Kết quả sẽ thay đổi tùy load balancer đưa request vào instance nào, làm agent mất ngữ cảnh ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Trong lúc build Docker mình gặp lỗi `Unknown type "CMD-SHELL" in HEALTHCHECK`. Mình xem dòng lỗi chỉ tới dòng 44 của Dockerfile và đối chiếu cú pháp Docker: `CMD-SHELL` là dạng healthcheck của Docker Compose, còn Dockerfile dùng `CMD`. Mình đổi lệnh thành `HEALTHCHECK ... CMD python -c "..."`, sau đó image build thành công. Bài học là cú pháp healthcheck giữa Dockerfile và Compose không hoàn toàn giống nhau.
