# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Phạm Thái Vũ  Mã học viên: 2A202602695

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên production, nếu quên cấu hình biến môi trường `AGENT_API_KEY`, việc app "chết ngay" (fail fast) sẽ khiến quá trình deploy báo lỗi ngay lập tức (Deploy failed). Ta phát hiện và bổ sung biến môi trường ngay trong lúc đang theo dõi deployment.
Nếu để mặc định `"changeme"`, ứng dụng vẫn khởi động thành công và công khai ra Internet. Các bot tự động có thể đoán hoặc dùng ngay key mặc định này để gọi API, làm tiêu tốn tài nguyên và hạn mức LLM mà ta chỉ biết khi nhận hóa đơn cuối tháng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:47:15.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 28, "cost_usd": 0.00015}`

Hai việc làm được với log JSON mà print thường không làm được:
1. Tự động tổng hợp và phân tích định lượng bằng các hệ thống gom log (Datadog, CloudWatch): chạy truy vấn tính tổng chi phí theo ngày (`SUM(cost_usd) GROUP BY user_id`).
2. Thiết lập cảnh báo (alert) tự động khi có sự cố: lọc nhanh các event có `level="error"` để cảnh báo qua Slack/Telegram khi tỷ lệ lỗi tăng đột biến trong 5 phút.

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
| 1 stage (bản đầu) | ~1050 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~780 MB) bao gồm:
- Các trình biên dịch C/C++, headers và công cụ build (gcc, make) có sẵn trong bản Python đầy đủ nhưng đã được loại bỏ ở bản `python:3.11-slim`.
- Bộ nhớ cache của pip (`/root/.cache/pip`) và các package hệ điều hành thừa thãi chỉ dùng khi cài đặt thư viện ở stage `builder`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Câu trả lời của bạn*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python có lỗ hổng (ví dụ Remote Code Execution).
2. Kẻ tấn công khai thác lỗ hổng để chạy lệnh shell bên trong container.
3. Nếu container chạy với root, kẻ tấn công có toàn quyền root trong container, có thể can thiệp socket và volume chia sẻ với host.
4. Kẻ tấn công khai thác lỗ hổng kernel để thoát khỏi container (container breakout) và giành quyền root trên máy host thật.

Lệnh `USER appuser` cắt đứt chuỗi này ngay tại bước 3: kẻ tấn công chỉ có quyền user thường (UID 10001), không thể can thiệp file hệ thống hay leo thang đặc quyền để breakout.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Câu trả lời của bạn*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt: Rate limit kiểm soát tần suất/số lượng request trong thời gian ngắn (ví dụ: 10 req/phút). Cost guard kiểm soát tổng số tiền (USD) chi tiêu trong cả tháng dựa trên lượng token LLM thực tế đã dùng.

- Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 1 request trong 10 phút (dưới hạn mức 10 req/phút), nhưng prompt rất dài tốn 10 USD trong khi ngân sách tháng chỉ còn 2 USD -> Cost guard chặn với mã lỗi 402.
- Cost guard cho qua nhưng Rate limit chặn: User mới bắt đầu tháng, ngân sách còn nguyên 100 USD. Tuy nhiên họ gửi spam liên tục 20 request trong 5 giây -> Rate limit chặn với mã 429 để chống nghẽn hệ thống.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis gặp sự cố tạm thời trong 30 giây.
2. Liveness check định kỳ gọi vào endpoint gộp chung, nhận về fail vì Redis chết.
3. Vì liveness check fail đồng nghĩa process bị coi là đã chết, Orchestrator phát lệnh restart cả 3 container web.
4. Cả 3 container khởi động lại nhưng Redis vẫn chưa xong -> tiếp tục fail và bị restart liên tục (CrashLoopBackOff).
5. Toàn bộ người dùng bị gián đoạn dịch vụ, một sự cố tạm thời ở Redis biến thành sự cố sập toàn bộ hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
