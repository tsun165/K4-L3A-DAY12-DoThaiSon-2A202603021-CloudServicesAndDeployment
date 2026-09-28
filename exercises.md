# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder "Câu trả lời của bạn" bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Thái Sơn  Mã học viên: 2A202603021

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tôi gặp đúng tình huống này khi deploy lên Railway: biến `AGENT_API_KEY` chưa
được áp dụng cho service agent. Lúc đó `Settings` chỉ được tạo khi có request
gọi `get_settings()`, nên app vẫn khởi động, `/health` trả 200, Railway báo
Active — nhưng `/ready` và `/ask` đều trả 500. Tôi phải so sánh với local mới
tìm ra nguyên nhân. Sau đó tôi gọi `get_settings()` ngay trong `lifespan`; chạy
container không có key thì nó dừng ngay với `agent_api_key: Field required` và
exit code 3, deploy sẽ báo Failed thay vì "xanh giả".

Nếu để mặc định `"changeme"` thì còn tệ hơn: app chạy bình thường trên URL
công khai với một khóa ai cũng đoán được, không có lỗi nào để phát hiện, và
người lạ gọi `/ask` bằng tiền của mình cho tới khi thấy hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được khi dừng container agent bằng `docker compose stop agent`:

```
{"event": "service_stopped", "level": "info", "timestamp": "2026-09-28T08:41:45.292027+00:00", "service": "day12-agent"}
```

Mỗi request `/ask` cũng ghi một dòng `ask_completed` kèm `user_id`,
`tokens_in`, `tokens_out`, `cost_usd`.

1. **Lọc và tìm theo trường**: trên Railway, Deploy Logs tự tách dòng JSON thành
   các trường (`event: service_started`, `service`, `version`...), nên tôi lọc
   được đúng `event = ask_completed` của một `user_id` cụ thể. Với chuỗi
   `print` tự do thì chỉ tìm chữ được, dễ sót và dễ nhầm.
2. **Tính toán / cảnh báo**: cộng `cost_usd` theo user hoặc theo ngày, đếm số
   request, đặt cảnh báo khi chi phí tăng bất thường. `print("đã trả lời xong")`
   không có con số nào để tính, cũng không có timestamp chuẩn UTC để so thứ tự
   giữa nhiều container.

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
| 1 stage (bản đầu) | chưa đo được (build bản `python:3.11` đầy đủ chưa xong lúc nộp) |
| Multi-stage | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch chủ yếu đến từ base image: bản đầu dùng `python:3.11` đầy đủ
(dựa trên Debian đầy đủ, có sẵn gcc, header, thư viện dev, công cụ build...),
còn bản của tôi dùng `python:3.11-slim` chỉ có đủ để chạy Python. Ngoài ra bản
đầu `COPY . .` nên mang theo cả `.git`, `tests`, ảnh chụp, và `pip install`
không có `--no-cache-dir` nên giữ lại cache pip. Bản multi-stage cài thư viện
ở stage `builder` rồi chỉ copy `/install` sang stage runtime, cộng với
`.dockerignore` và chỉ copy `app/`, `utils/`, nên image cuối gọn hơn nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile của tôi, các layer `FROM`, `COPY requirements.txt`,
`RUN pip install`, `COPY --from=builder`, `RUN useradd` đều báo `CACHED` vì
input của chúng không đổi. Chỉ từ `COPY app ./app` trở xuống phải chạy lại, và
vì các bước đó chỉ là copy file nên build lại mất vài giây. Lần build đầu tiên
của tôi mất khoảng 12 phút (chủ yếu là tải base image và cài thư viện), nên sự
khác biệt rất rõ.

Nếu đặt `COPY . .` trước `RUN pip install`, thì sửa một ký tự trong
`main.py` làm layer `COPY . .` thay đổi, và Docker phải chạy lại mọi layer phía
sau, kể cả `pip install` — tức là mỗi lần sửa code lại cài lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

1. Code có lỗ hổng (ví dụ một thư viện bị lỗi cho phép thực thi lệnh từ xa).
2. Kẻ tấn công chạy được lệnh trong container với quyền của process — nếu
   container chạy root thì đó là root trong container: đọc/sửa mọi file trong
   image, cài thêm công cụ, đọc biến môi trường chứa secret.
3. Từ root trong container, kẻ tấn công lợi dụng cấu hình lỏng (volume mount
   thư mục của host, Docker socket, container privileged) hoặc một lỗi của
   kernel/container runtime để thoát ra ngoài. Vì UID 0 trong container thường
   cũng là UID 0 trên host, thoát ra được là có quyền root trên host.

Lệnh `USER appuser` (UID 10001) cắt chuỗi ở bước 2: kẻ tấn công chỉ có quyền
của một user thường, không ghi được vào file hệ thống, không cài được gói, và
kể cả thoát được ra ngoài thì cũng chỉ là UID 10001 không có quyền gì trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request** trong 2 giây. Cách làm: gửi 10 request lúc 10:00:59 (hết
hạn mức của phút 10:00), đợi tới 10:01:00 bộ đếm reset về 0, rồi gửi tiếp 10
request lúc 10:01:00–10:01:01. Mỗi phút đồng hồ đều "đúng luật" 10 request,
nhưng thực tế là 20 request dồn trong 2 giây.

Với sliding window, mỗi lần kiểm tra tôi đếm các request trong đúng 60 giây
tính ngược từ thời điểm hiện tại (Redis ZSET, score là timestamp). Request thứ
11 lúc 10:01:00 vẫn thấy 10 request từ 10:00:59 nằm trong cửa sổ nên bị 429.
Khi thử trên Railway, gọi 15 lần liên tiếp tôi nhận
`200 ×9` (1 lượt đã dùng trước đó) rồi `429 ×6`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **tốc độ** (số request trong 60 giây gần nhất, trả 429, tự
hồi sau 1 phút). Cost guard giới hạn **tổng tiền** theo user trong tháng (trả
402, chỉ hồi khi sang tháng mới). Một cái chống spam/quá tải, một cái chống
cháy ngân sách.

- **Rate limit cho qua, cost guard chặn**: một user gửi đều 5 request/phút,
  không bao giờ vượt 10/phút, nhưng mỗi câu hỏi rất dài hoặc cả tháng đều gọi
  như vậy. Tổng `cost_usd` dần vượt 10 USD → request tiếp theo bị 402 dù tốc độ
  vẫn hợp lệ.
- **Cost guard cho qua, rate limit chặn**: một user mới, chưa tiêu gì trong
  tháng, chạy vòng lặp gửi 15 request trong vài giây với câu hỏi ngắn. Chi phí
  mỗi câu chỉ khoảng 0.00002 USD nên còn rất xa ngân sách, nhưng từ request thứ
  11 bị 429 — đúng như lúc tôi thử 15 lần liên tiếp trên Railway.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối → cả 3 container cùng lúc trả 503 ở endpoint gộp.
2. Load balancer ngừng gửi traffic vào cả 3 → user nhận lỗi.
3. Orchestrator cũng dùng endpoint đó làm liveness, thấy fail vài lần liên tiếp
   → kết luận container "chết" và **restart cả 3**, dù bản thân app không hỏng.
4. Container mới khởi động trong lúc Redis vẫn chưa về → lại fail → restart
   tiếp; có thể rơi vào vòng restart liên tục.
5. Redis về sau 30 giây, nhưng các container đang khởi động lại hoặc đang chờ
   backoff nên thời gian gián đoạn dài hơn 30 giây nhiều, request đang xử lý dở
   lúc restart cũng bị cắt.

Khi tách ra, tôi thử `docker compose stop redis`: `/ready` trả 503
`{"status":"not ready","redis":false}` nhưng `/health` vẫn 200 — container
không bị restart. Bật lại Redis thì `/ready` tự về 200. Khi dừng agent bằng
SIGTERM, log cho thấy `Shutting down` → `service_stopped` →
`Finished server process [1]`: app tắt êm, không bị SIGKILL.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với compose hiện tại, service `agent` map cố định `8000:8000`, nên scale lên 3
sẽ bị trùng cổng trên host — chỉ một container bind được cổng 8000. Muốn chạy
thật 3 instance cần bỏ port cố định và đặt nginx phía trước làm load balancer.

Khi lịch sử nằm trong Redis, dù request rơi vào container nào,
`history_length` vẫn tăng đều 0 → 2 → 4 → 6..., vì mọi container đọc/ghi chung
một key `history:<user_id>`. Test `test_state_khong_nam_trong_process` mô phỏng
đúng điều này: container A ghi, container B đọc thấy ngay.

Nếu lưu trong dict Python, mỗi container có RAM riêng. Load balancer chia
request vòng tròn A → B → C nên `history_length` sẽ nhảy lung tung, ví dụ
0, 0, 0, 2, 2, 2, 4...: mỗi container chỉ nhớ phần hội thoại đi qua nó, agent
"mất trí nhớ" giữa các câu. Container restart thì mất sạch lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Thông báo lỗi**: sau khi deploy lên Railway, `curl .../ready` trả
`Internal Server Error` (500), trong khi `/health` vẫn 200 và dashboard báo
Active.

**Tìm nguyên nhân**: tôi gọi thêm `POST /ask` không có key — đáng lẽ phải 401
nhưng cũng ra 500. Như vậy lỗi nằm ở chỗ chung của `/ready` và `/ask`: cả hai
đều gọi `get_settings()` (qua `get_redis_client` và `verify_api_key`), còn
`/health` thì không. Tôi chạy app ở local với `AGENT_API_KEY` bị xóa và ra đúng
bộ kết quả 200 / 500 / 500; còn nếu chỉ sai `REDIS_URL` thì `/ask` vẫn 401.
Kết luận: service trên Railway không nhận được `AGENT_API_KEY`.

**Sửa**: đặt `AGENT_API_KEY` (khóa mới tạo bằng `secrets.token_urlsafe(32)`)
và `REDIS_URL=${{Redis.REDIS_URL}}` trong tab Variables của đúng service agent,
rồi bấm Deploy để áp dụng thay đổi đang staged. Sau đó `/health` 200, `/ready`
200 `{"redis":true}`, `/ask` không key 401. Tôi cũng sửa code để gọi
`get_settings()` lúc khởi động, lần sau thiếu biến thì deploy fail ngay.

Ngoài ra `railway.toml` ban đầu có `startCommand` dùng `$PORT`; tôi bỏ dòng này
để Railway dùng `CMD` của Dockerfile (chạy qua `sh -c` nên `${PORT}` được
expand). Log trên Railway xác nhận `Uvicorn running on http://0.0.0.0:8080` —
app đọc đúng cổng do platform cấp.
