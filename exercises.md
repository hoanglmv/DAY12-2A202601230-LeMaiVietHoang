# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Mai Việt Hoàng  Mã học viên: 2A202601230

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường Production/Cloud, nếu developer quên cấu hình biến môi trường `API_TOKEN` trên Dashboard của platform, việc `Settings` không có giá trị mặc định sẽ làm ứng dụng báo lỗi `ValidationError` và dừng khởi động ngay lúc deploy (fail fast). Nhờ đó developer phát hiện thiếu sót ngay lập tức khi đang xem màn hình deploy. Ngược lại, nếu để mặc định `"changeme"`, ứng dụng vẫn khởi động thành công và chạy lặng lẽ; bot tự động quét trên Internet sẽ nhanh chóng tìm ra endpoint và dùng token mặc định `"changeme"` để gọi API miễn phí, tiêu tốn toàn bộ ngân sách LLM trước khi bạn phát hiện ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON mẫu:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T07:00:00.000000+00:00", "client_id": "sv-test", "usd_cost": 0.000025, "prompt_tokens": 4, "completion_tokens": 35}`

Hai việc làm được với log JSON:
1. **Lọc và phân tích tự động (Structured Query)**: Các công cụ tập trung log (Datadog, GCP Logging, Elastic) có thể parse các trường JSON để chạy câu vấn chính xác như *"client_id nào tiêu tốn chi phí usd_cost cao nhất trong ngày?"* hoặc *"tổng số prompt_tokens đã tiêu tốn trong 1 giờ qua là bao nhiêu?"*.
2. **Cảnh báo tự động (Automated Alerting)**: Hệ thống giám sát có thể đọc trường `severity` chuẩn để kích hoạt cảnh báo tức thì về Slack/PagerDuty nếu số lượng log `ERROR` tăng đột biến hoặc chi phí `usd_cost` của một request vượt ngưỡng an toàn.

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
| 1 stage (bản đầu) | 1.22 GB |
| Multi-stage | 285 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~935 MB) bao gồm các công cụ build/biên dịch hệ thống (như `gcc`, `g++`, `make`), bộ nhớ tạm của trình quản lý gói `pip cache`, các file header C/C++ (`.h`), và các thư viện hệ thống thừa chỉ dùng trong quá trình build wheel. Trong multi-stage build, stage `builder` chịu trách nhiệm cài đặt và biên dịch, sau đó stage `runtime` nhẹ chỉ copy kết quả từ `/install` sang `/usr/local`, loại bỏ hoàn toàn bộ công cụ biên dịch nặng nề.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại (COPY `requirements.txt` -> `RUN pip install` -> `COPY app ./app`): Các layer cài đặt base image và layer cài `pip install` được dùng lại từ Docker cache (vì file `requirements.txt` không đổi). Chỉ các layer từ `COPY app ./app` trở đi mới phải chạy lại, quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa 1 ký tự trong code, hash của layer `COPY . .` bị thay đổi, làm vô hiệu hóa (invalidate) cache của tất cả các layer phía sau. Kết quả là Docker buộc phải chạy lại lệnh `RUN pip install` từ đầu, làm lại toàn bộ quá trình tải và cài đặt thư viện tốn hàng phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: 1. Code Python có lỗ hổng (ví dụ Remote Code Execution / Command Injection). 2. Kẻ tấn công lợi dụng lỗ hổng để thực thi lệnh hệ thống bên trong container. 3. Vì container chạy mặc định với quyền `root`, kẻ tấn công sở hữu quyền root trong container. 4. Kẻ tấn công lợi dụng thêm các kỹ thuật escape container (như khai thác kernel vulnerability hoặc truy cập docker.sock nếu bị mount nhầm) để thoát khỏi container và ngay lập tức có quyền `root` quản trị tối cao trên máy host.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại bước 3: Khi chuyển sang user thường không có đặc quyền (UID 10001), mã độc của kẻ tấn công chỉ có quyền hạn cực kỳ hạn chế bên trong container, không thể ghi sửa file hệ thống container và không đủ đặc quyền để thực hiện hành vi leo thang escape ra máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- Trả kèm header `WWW-Authenticate: Bearer`: Chuẩn HTTP (RFC 6750 / RFC 7235) bắt buộc mã phản hồi `401 Unauthorized` phải đi kèm header `WWW-Authenticate` để chỉ dẫn rõ cho client chuẩn xác thực và scheme cần thiết (Bearer token) để client điều chỉnh request.
- Trả cùng một thông báo lỗi: Giúp bảo vệ khỏi tấn công rò rỉ thông tin (Information Disclosure / Enumeration Attack). Nếu phân biệt chi tiết giữa "thiếu header", "sai scheme" hay "token không tồn tại", kẻ tấn công có thể lợi dụng sự khác biệt này để dò tìm cấu hình hoặc quét tìm danh sách token hợp lệ.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Với `capacity=10`, `refill_per_minute=10`: Client gửi được tối đa **10 request** liên tiếp trước khi bị từ chối bằng lỗi 429 (vì số token trong xô bị chặn trần ở `capacity = 10`).
- Nếu bỏ `min(capacity, ...)`: Client im lặng 10 phút sẽ tích lũy được $10 \text{ phút} \times 10 \text{ token/phút} = 100 \text{ token}$. Khi đó client có thể gửi **100 request** liên tiếp cùng một lúc. Điều này làm mất đi tác dụng giới hạn tần suất, khiến kẻ tấn công hoặc client có thể tạo ra các đợt bùng nổ request (burst) quá lớn gây quá tải hệ thống.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- Hạn mức $30/tháng: Khi sự cố xảy ra lúc 2h sáng, client gọi liên tục sẽ đốt sạch toàn bộ $30 ngân sách của cả tháng chỉ trong vài giờ. Thiệt hại tối đa là **$30**, và service sẽ bị vô hiệu hóa cho tới đầu tháng sau (trừ khi có người can thiệp nạp thêm ngân sách thủ công).
- Hạn mức $1/ngày: Với sự cố tương tự, thiệt hại tối đa bị chặn đứng ở **$1**. Ngay khi tổng chi tiêu chạm $1, service trả về 402 Payment Required để chặn tất cả request tiếp theo trong ngày. Đến 00:00 UTC sáng hôm sau, key Redis theo ngày `spend:<client>:<YYYY-MM-DD>` chuyển sang ngày mới, service **tự động khôi phục** hoạt động bình thường mà không cần bất kỳ sự can thiệp thủ công nào.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra Redis thất bại và trả về HTTP 503.
3. Orchestrator (Docker/Kubernetes) nhận 503 từ Liveness check và hiểu lầm rằng ứng dụng Python bị treo -> Orchestrator tiến hành tiêu diệt và khởi động lại (restart) cả 3 container `chat`.
4. Trong quá trình 3 container đang khởi động lại, Redis vẫn chưa sẵn sàng -> healthcheck khi boot up tiếp tục thất bại -> Orchestrator rơi vào vòng lặp restart liên tục (CrashLoopBackOff).
5. Ngay cả khi Redis đã kết nối trở lại sau 30s, toàn bộ hệ thống vẫn bị ngừng phục vụ (downtime) kéo dài do 3 container vẫn đang bận boot up lại từ đầu, làm biến sự cố nhỏ của Redis thành sự cố sập toàn bộ dịch vụ.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Endpoint `/readyz` trả về lỗi HTTP 503 `{"status": "not ready", "redis": false}` sau khi deploy lên Cloud.
Nguyên nhân: Qua việc kiểm tra log dịch vụ (`railway logs`), tôi phát hiện báo lỗi `ConnectionRefusedError: [Errno 111] Connect call failed ('127.0.0.1', 6379)`. Nguyên nhân do biến môi trường `REDIS_URL` trên dashboard cloud vẫn đang để giá trị mặc định `redis://localhost:6379/0` (dùng cho local), nên container app cố kết nối tới Redis trên chính nó thay vì Redis add-on của Cloud.
Cách sửa: Vào mục Variables trên Platform Dashboard, cập nhật biến `REDIS_URL` thành URL kết nối do Redis service cung cấp (ví dụ `redis://default:pass@redis.railway.internal:6379`), sau đó bấm Redeploy.
