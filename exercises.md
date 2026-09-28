# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Gia Khải  Mã học viên: 02786

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên cloud (như Railway hoặc Render), nếu lỡ quên thiết lập biến môi trường `AGENT_API_KEY`:
- Nếu đặt giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường, vượt qua liveness check và mở public ra Internet. Kẻ tấn công hoặc bot tự động quét các endpoint lộ khóa mặc định sẽ dễ dàng gọi vào API, tiêu tốn sạch ngân sách token LLM mà ta chỉ phát hiện ra khi nhận hóa đơn thanh toán cuối tháng.
- Khi không có giá trị mặc định, `pydantic-settings` sẽ ném ngoại lệ `ValidationError` ngay trong pha khởi động (Fail Fast), khiến container dừng lập tức và báo đỏ trên Deployment log. Điều này buộc ta phải cấu hình đúng secret trước khi dịch vụ được kích hoạt, loại bỏ hoàn toàn rủi ro bảo mật nghiêm trọng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:41:39.123456+00:00", "user_id": "sv-123", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.00015}
```

Hai việc làm được với log JSON mà `print()` thông thường không làm được:
1. **Truy vấn và tổng hợp định lượng trên hệ thống quản lý log (Log Aggregator như Loki, CloudWatch, Datadog):** Máy có thể tự động parse JSON để lọc, group và aggregate (ví dụ: truy vấn tìm Top 5 người dùng tiêu tốn nhiều chi phí nhất trong ngày bằng câu lệnh kiểu `SELECT sum(cost_usd) GROUP BY user_id`).
2. **Thiết lập cảnh báo tự động theo ngưỡng (Automated Alerting):** Có thể đặt trigger cảnh báo ngay lập tức nếu một request có `cost_usd > 0.05` hoặc đếm tần suất sự kiện theo thời gian thực để phát hiện các đợt bùng nổ traffic bất thường.

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
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | 447 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~573 MB) bao gồm:
1. Hệ điều hành nền: Bản gốc dùng Debian đầy đủ của `python:3.11`, trong khi bản multi-stage dùng base `python:3.11-slim` đã lược bỏ hầu hết các package tiện ích không cần thiết.
2. Công cụ build và biên dịch: Toàn bộ bộ công cụ `build-essential` (gcc, g++, make), các file C/C++ header, thư viện phát triển tĩnh (`-dev`) và apt cache (`/var/lib/apt/lists/*`) chỉ nằm lại ở stage `builder` và bị hủy sau khi build.
3. Cache của package manager: Bộ nhớ đệm pip wheels (`~/.cache/pip`) không bị đưa sang stage runtime vì chỉ copy thư mục cài đặt sạch `/install` sang `/usr/local`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`: Docker kiểm tra layer cache tuần tự từ trên xuống. Các layer từ `FROM`, `WORKDIR`, `RUN apt-get`, `COPY requirements.txt .`, `RUN pip install` và `RUN useradd` đều không đổi nên được tái sử dụng 100% từ cache (`CACHED`). Chỉ từ layer `COPY app ./app` trở đi mới bị cache-bust và phải thực hiện lại, giúp thời gian build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa bất kỳ ký tự nào trong code, checksum của layer `COPY . .` bị thay đổi, làm mất hiệu lực toàn bộ cache của tất cả các layer phía sau. Hậu quả là Docker buộc phải chạy lại `RUN pip install` từ đầu, tải và cài đặt lại toàn bộ thư viện, biến mỗi lần build từ 2 giây thành vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện leo thang:
  1. Ứng dụng Python có lỗ hổng (ví dụ RCE thông qua deserialization, injection hoặc lỗ hổng zero-day trong thư viện C underlying).
  2. Kẻ tấn công gửi payload kích hoạt RCE và chiếm được shell thực thi trong container. Do container chạy mặc định bằng root (UID 0), shell này có đặc quyền root bên trong container.
  3. Từ quyền root container, kẻ tấn công khai thác tiếp các lỗ hổng container breakout (kernel privilege escalation, mounted docker socket `/var/run/docker.sock`, hoặc container capabilities như `CAP_SYS_ADMIN`).
  4. Sau khi thoát khỏi namespace container, tiến trình chạy trên máy host với đúng UID 0 (root máy host), cho phép kẻ tấn công kiểm soát toàn bộ server vật lý hoặc node máy chủ.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại Bước 2: Bằng cách gán process chạy dưới quyền user không đặc quyền (UID 10001), kẻ tấn công nếu có chiếm được quyền điều khiển app cũng chỉ có quyền của user thường, không thể chỉnh sửa file hệ thống trong container và không đủ thẩm quyền để thực hiện các cuộc tấn công container breakout sang host OS.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được:
- Người dùng gửi 10 request vào giây `10:00:59` (giây cuối cùng của phút thứ nhất). Vì hạn mức là 10 req/phút nên cả 10 request này đều được chấp nhận hợp lệ.
- Ngay tại giây `10:01:00`, đồng hồ bước sang phút mới và bộ đếm fixed window tự động reset về 0. Người dùng lập tức gửi thêm 10 request nữa trong giây `10:01:00`.
- Tổng cộng từ `10:00:59` đến `10:01:00` (khoảng thời gian đúng 2 giây), hệ thống phải hứng chịu 20 request, gấp đôi tải thiết kế cho phép. Sliding window loại bỏ hoàn toàn kẽ hở này vì nó liên tục xét khoảng thời gian 60 giây trượt lùi từ thời điểm hiện tại `[now - 60, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác nhau:
  - Rate limit bảo vệ **tính sẵn sàng của hạ tầng (Availability/DoS)** bằng cách kiểm soát tần suất số lượng request trong khoảng thời gian ngắn (ví dụ: 10 requests / 60 giây).
  - Cost guard bảo vệ **ngân sách tài chính (Budget/Cost)** bằng cách kiểm soát tổng chi phí tích lũy theo tháng (ví dụ: $10.0 / tháng), ngăn chặn việc tiêu hao token quá mức.
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  Đầu tháng, một user đã gửi nhiều prompt dài và tích lũy chi phí đạt $10.0. Sau đó user nghỉ vài tiếng và gửi 1 request mới: Rate limit thấy user chỉ gửi 1 request trong 60 giây gần nhất nên cho qua, nhưng Cost guard kiểm tra thấy ngân sách tháng đã chạm trần $10.0 nên chặn ngay với mã HTTP 402 Payment Required.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  Một user mới tinh trong tháng mới tiêu $0.05 (ngân sách còn dư $9.95), nhưng user này dùng script gửi liên tục 15 request trong vòng 5 giây: Cost guard thấy ngân sách còn rất dồi dào, nhưng Rate limit phát hiện request thứ 11 vượt quá hạn mức 10 req/phút và chặn lại với mã HTTP 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra thấy Redis không phản hồi nên trả về mã lỗi HTTP 503.
3. Orchestrator (Docker Swarm/Kubernetes/ECS) hiểu nhầm rằng process của container đã bị treo/hỏng (liveness failure), do đó gửi SIGKILL để tiêu diệt và khởi động lại toàn bộ 3 container.
4. Cả 3 container liên tục khởi động lại, nhưng vì Redis vẫn chưa sống lại trong 30 giây đó, các container mới khởi động lại tiếp tục bị coi là hỏng và bị restart lặp đi lặp lại (CrashLoopBackOff).
5. Khi Redis phục hồi sau 30 giây, toàn bộ 3 container đều đang trong trạng thái restart dở dang, không có container nào sẵn sàng phục vụ người dùng. Sự cố mạng tạm thời của dependency bên ngoài đã biến thành sự cố sập hoàn toàn hệ thống dịch vụ (Cascading Failure).
*(Khi tách riêng, `/health` vẫn 200 để giữ container sống, chỉ `/ready` trả 503 để Load Balancer tạm ngắt định tuyến traffic).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong dict Python trong bộ nhớ RAM của container:
- Mỗi container trong cụm 3 instance sẽ sở hữu một dict riêng biệt không chia sẻ với nhau.
- Load balancer phân phối các request luân phiên (Round-Robin) tới 3 container A, B, C:
  - Lượt 1 vào Container A: `history_length` = 0 (A lưu turn 1 vào RAM của A).
  - Lượt 2 vào Container B: `history_length` = 0 (B chưa hề nhận request nào của user này).
  - Lượt 3 vào Container C: `history_length` = 0 (C cũng rỗng).
  - Lượt 4 vào Container A: `history_length` = 2 (A chỉ thấy turn 1 trước đó của nó, bỏ qua các câu hỏi ở B và C).
- Con số `history_length` sẽ nhảy lộn xộn không thể đoán trước (0, 0, 0, 2, 2, 2...), làm cho Agent phản hồi như bị mất trí nhớ ngẫu nhiên. Khi chuyển state sang Redis, cả 3 container cùng truy xuất một nguồn dữ liệu duy nhất, `history_length` luôn tăng đều đặn: 0 -> 2 -> 4 -> 6 -> 8...

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi gặp phải:
  `Health check timeout: Service failed to respond on port 10000 after 180 seconds. Deployment failed.`
- Nguyên nhân:
  Các nền tảng PaaS như Render và Railway tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`). Trong khi đó, lệnh khởi động mặc định ban đầu bị hardcode `uvicorn app.main:app --port 8000`. Khi container chạy, uvicorn chỉ lắng nghe trên cổng 8000, khiến bộ phận probe kiểm tra sức khỏe của Cloud Router gửi request vào cổng `$PORT` không nhận được phản hồi.
- Cách khắc phục:
  Trong `Dockerfile`, sửa lệnh `CMD` sử dụng cú pháp shell parameter expansion để đọc động biến môi trường `$PORT` do platform cấp:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  Sau khi sửa, service tự động bind đúng cổng của cloud provider và vượt qua health check ngay trong lần deploy tiếp theo.
