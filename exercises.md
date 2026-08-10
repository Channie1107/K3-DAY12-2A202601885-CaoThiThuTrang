# Phiếu Phản Ánh - K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code - không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> **Họ và tên:** Cao Thị Thu Trang. **Mã học viên:** 2A202601885

---

### Câu 1 - Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Mình thấy cách này cứu mình ở lúc deploy lên Railway: nếu quên set
`AGENT_API_KEY` thì app dừng ngay từ đầu, mình biết phải sửa secret trước khi
service public. Nếu để mặc định như `"changeme"` thì app vẫn chạy, nhưng lại vô
tình mở cổng cho người lạ gọi API bằng khóa mặc định đó.

---

### Câu 2 - Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ một dòng log JSON:
`{"event":"ask_completed","level":"info","timestamp":"2026-08-10T09:15:00Z","user_id":"sv01","tokens_in":42,"tokens_out":58,"cost_usd":0.000123}`

Với dòng log này, mình có thể lọc theo `event` để đếm số lần `/ask` hoàn tất, và
có thể gom theo `user_id` hoặc `cost_usd` để theo dõi ai đang tốn nhiều chi phí
nhất. `print("đã trả lời xong")` không cho máy phân tích được các trường đó.

---

### Câu 3 - Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đối chứng) | 287 MB |
| Multi-stage | 270 MB |

Giải thích: chênh lệch chỉ khoảng 17MB, tức gần 6%. Nghĩa là trong dự án này,
multi-stage không "cắt" được rất nhiều vì `requirements.txt` chủ yếu là các gói
có sẵn wheel dựng sẵn, nên bản 1-stage gần như không phải mang theo một đống
compiler hay `build-essential` nặng như ví dụ minh hoạ trong LAB_GUIDE. Phần
tiết kiệm thực tế chủ yếu đến từ việc stage runtime không phải giữ lại layer cài
đặt tạm, cache pip và các file trung gian không cần cho chạy thật.

---

### Câu 4 - Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, chỉ layer `COPY . .` và `RUN chown` phía sau nó phải
chạy lại khi mình sửa `app/main.py`; các layer `FROM`, `COPY requirements.txt`
và `pip install` vẫn được cache. Nếu đảo ngược thứ tự và copy toàn bộ source
trước khi cài dependency, thì chỉ cần đổi một dòng code là Docker sẽ phải cài
lại toàn bộ thư viện, build chậm hơn nhiều.

---

### Câu 5 - Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng trong
code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và lệnh `USER`
cắt đứt chuỗi đó ở chỗ nào.

Nếu app chạy bằng root, một lỗi RCE hoặc upload file độc hại có thể cho kẻ tấn
công quyền ghi/xóa file trong container với quyền root, đọc secret, hoặc làm
hỏng mounted volume. Lệnh `USER app` chuyển process sang user không đặc quyền
nên nếu có lỗ hổng thì blast radius nhỏ hơn rất nhiều, không còn quyền root
mặc định.

---

### Câu 6 - Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo phút
đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu request
trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được con số đó.

Con số tối đa là 20 request trong 2 giây liên tiếp. Người dùng có thể gửi 10
request ngay trước mốc đổi phút, rồi gửi thêm 10 request ngay sau khi phút mới
bắt đầu. Vì bộ đếm reset theo ranh giới phút nên hai cụm request đó nằm ở hai
window khác nhau.

---

### Câu 7 - Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request trong một khoảng thời gian, còn cost guard giới
hạn số tiền đã tiêu. Ví dụ rate limit cho qua nhưng cost guard chặn: một request
rất dài, token quá nhiều, làm vượt ngân sách tháng. Tình huống ngược lại: user
gửi rất nhiều request ngắn, tổng tiền vẫn thấp nhưng vượt 10 request/phút nên bị
rate limit chặn.

---

### Câu 8 - `/health` khác `/ready` (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu `/health` cũng kiểm tra Redis thì khi Redis mất 30 giây, liveness của cả 3
container sẽ trả lỗi. Orchestrator sẽ tưởng process chết thật và bắt đầu restart
hoặc loại các container, dù bản thân app vẫn còn chạy. Đúng thứ tự phải là:
Redis chập chờn -> `/ready` báo 503 để không đẩy traffic mới -> `/health` vẫn
giữ 200 nếu process còn sống.

---

### Câu 9 - Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử nằm trong dict Python của từng container thì `history_length` sẽ
nhảy loạn giữa các request. Có request trả 0 vì vào container mới, có request
trả 2 nếu vừa đúng container đã giữ lại lịch sử cũ. Nói cách khác, mỗi instance
nhớ một phần khác nhau của cuộc hội thoại nên agent bị "mất trí" khi scale
ngang.

---

### Câu 10 - Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Mình gặp lỗi `The executable 'uvicorn' could not be found.` trên Railway. Mình
kiểm tra log runtime và thấy `startCommand` gọi thẳng `uvicorn`, trong khi image
không có binary này trên `PATH`. Mình sửa bằng cách gọi `python -m uvicorn
app.main:app --host 0.0.0.0 --port $PORT` để dùng đúng interpreter trong image
và đọc được cổng do Railway cấp.
