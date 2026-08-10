# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời` `của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Giang Trung Quân  Mã học viên: 2A202601098

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định là `"changeme"`, khi deploy lên Cloud mà quên cấu hình biến môi trường `API_TOKEN` trên dashboard, ứng dụng vẫn sẽ khởi động thành công và chạy bình thường. Khi đó, bất cứ ai (hoặc các bot quét bảo mật) cũng có thể gửi request đến endpoint `/chat` với token `"changeme"` và sử dụng tài nguyên LLM/API của chúng ta miễn phí, dẫn đến mất mát dữ liệu hoặc phát sinh chi phí lớn (hóa đơn API khổng lồ) trước khi chúng ta kịp phát hiện. Ngược lại, việc không đặt giá trị mặc định giúp app crash ngay lập tức (fail fast) khi khởi động trên Cloud, giúp ta phát hiện ra lỗi cấu hình ngay lập tức trên dashboard mà không bị rò rỉ bảo mật hay phát sinh chi phí ngoài ý muốn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một ví dụ dòng log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:35:59.123456+00:00", "client_id": "sv-test", "prompt_tokens": 12, "completion_tokens": 15, "usd_cost": 0.00015}`

Hai việc làm được với dòng log JSON structured logging này mà `print("đã trả lời xong")` không làm được:
1. **Lọc và phân tích dữ liệu tự động bằng công cụ quản lý log (Datadog, Cloud Watch, Render Log Viewer...):** Các hệ thống có thể phân tích cú pháp JSON để vẽ biểu đồ thống kê chi phí (`usd_cost`) theo thời gian hoặc đếm số lượng token sử dụng của từng client (`client_id`).
2. **Cấu hình cảnh báo (Alerting) tự động:** Ta có thể cài đặt hệ thống giám sát tự động kích hoạt cảnh báo gửi qua Slack/Email khi phát hiện các dòng log có `severity` là `ERROR` hoặc khi trường `usd_cost` vượt ngưỡng cho phép, điều mà các dòng text thô `print` rất khó để lọc chính xác và phân loại.

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
| 1 stage (bản đầu) | 450 MB |
| Multi-stage | 140 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch đó chính là các công cụ biên dịch (gcc, make), pip cache, header files (.h), các file không cần thiết cho quá trình chạy runtime (như các package dev) phát sinh trong quá trình build dependencies ở stage `builder`. Khi dùng multi-stage, ta chỉ copy các thư mục thư viện đã được cài đặt (`/usr/local`) sang stage `runtime` sạch, loại bỏ hoàn toàn các file rác nói trên.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

*   Khi sửa một ký tự trong `app/main.py`, các layer cài đặt dependencies (`COPY requirements.txt .` và `RUN pip install ...`) sẽ được dùng lại từ cache vì file `requirements.txt` không thay đổi. Layer `COPY app ./app` và các layer phía sau nó (như `COPY utils ./utils`, cấu hình user, `CMD`) sẽ phải chạy lại.
*   Nếu ta đặt `COPY . .` lên trước `RUN pip install`, thì bất cứ khi nào có sự thay đổi dù là nhỏ nhất ở bất kỳ file code nào (ví dụ `app/main.py`), cache của layer `COPY . .` sẽ bị mất hiệu lực (invalidated). Khi đó, layer tiếp theo là `RUN pip install` cũng bắt buộc phải chạy lại từ đầu thay vì dùng cache. Việc này làm tăng thời gian build rất nhiều lần vì phải tải và cài lại toàn bộ thư viện mỗi khi sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

*   **Chuỗi sự kiện:**
    1. Code Python của bạn có lỗ hổng (ví dụ: Remote Code Execution - RCE qua việc thực thi lệnh hệ thống từ input đầu vào không được lọc).
    2. Kẻ tấn công khai thác lỗ hổng này để thực thi mã độc. Do container mặc định chạy quyền `root`, tiến trình Python chạy dưới quyền `root` trong container. Kẻ tấn công có toàn quyền đọc/ghi/xóa mọi file bên trong container.
    3. Nếu container được mount với các volume của host hoặc có lỗ hổng thoát khỏi container (container escape - ví dụ thông qua các lỗ hổng nhân Linux hoặc cấu hình thiếu an toàn), kẻ tấn công sẽ thoát ra máy host với quyền `root` của container. Vì tiến trình chạy trong container là `root`, khi thoát ra ngoài host, họ sẽ trực tiếp có quyền `root` (quyền cao nhất) trên máy host.
*   **Điểm cắt đứt của lệnh `USER`:**
    Lệnh `USER appuser` chuyển tiến trình chạy container sang một user thường (non-root, ví dụ uid 10001). Khi đó, nếu kẻ tấn công khai thác được lỗi RCE, tiến trình chạy mã độc chỉ có quyền của `appuser` (bị hạn chế rất nhiều, không thể ghi vào các thư mục hệ thống hoặc file nhạy cảm). Ngay cả khi có lỗ hổng container escape, khi thoát ra ngoài máy host, tiến trình đó cũng chỉ là một user thường không có quyền lực cao, ngăn chặn nguy cơ chiếm quyền điều khiển toàn bộ máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

*   **Vì sao phải kèm header `WWW-Authenticate`:** Đây là tiêu chuẩn được quy định trong RFC 6750 (OAuth 2.0 Authorization Framework: Bearer Token Usage). Khi server trả về mã `401 Unauthorized`, nó bắt buộc phải gửi kèm header `WWW-Authenticate` để chỉ ra phương thức xác thực nào được chấp nhận (ở đây là `Bearer`), giúp client biết cách cấu hình header xác thực cho request tiếp theo.
*   **Vì sao trả cùng một thông báo lỗi:** Đây là nguyên tắc bảo mật thông tin (Security through obscurity). Nếu chúng ta trả về các lỗi chi tiết (ví dụ: "Thiếu header", "Token bị sai ký tự thứ 10"), kẻ tấn công có thể lợi dụng các thông tin này để phỏng đoán, thử sai và dò tìm lỗ hổng/token hợp lệ nhanh hơn (user enumeration / token sniffing). Trả về cùng một thông báo lỗi chung chung giúp ngăn chặn kẻ tấn công biết được họ đang đi đúng hướng đến mức nào.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

*   **Số lượng request gửi được:** Gửi được tối đa **10 request** liên tiếp.
*   **Nếu bỏ đoạn `min(capacity, ...)`:** Số lượng token tích lũy sẽ tăng vô hạn theo thời gian im lặng. Sau 10 phút, số token được nạp lại sẽ là `10 phút * 10 token/phút = 100 token`. Client có thể gửi liên tiếp **100 request** mà không bị chặn. Việc này làm mất đi tác dụng kiểm soát rate limit (token bucket) của hệ thống vì client có thể "dồn" request để tấn công DoS/DDoS sau một khoảng thời gian dài không hoạt động.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

*   **Hạn mức $30/tháng:**
    *   *Thiệt hại tối đa:* $30. Client có thể spam cạn sạch ngân sách cả tháng $30 chỉ trong vài giờ đầu tiên của ngày đầu tháng.
    *   *Thời gian tự hồi phục:* Đầu tháng tiếp theo (client phải đợi đến chu kỳ tháng sau mới được reset ngân sách và sử dụng tiếp).
*   **Hạn mức $1/ngày:**
    *   *Thiệt hại tối đa:* Chỉ $1. Khi client gọi hết $1, API sẽ trả về `402 Payment Required` và chặn toàn bộ request tiếp theo của client này trong ngày.
    *   *Thời gian tự hồi phục:* Ngay đầu ngày hôm sau (00:00 UTC khi key redis chứa chi tiêu trong ngày hết hạn hoặc được reset).
*   **Kết luận:** Cách hạn mức theo ngày ($1/ngày) an toàn hơn rất nhiều, giúp hạn chế rủi ro thiệt hại tài chính đột biến khi xảy ra sự cố (như lỗi lặp vô hạn của client) và hệ thống tự hồi phục nhanh hơn.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối 30 giây -> Endpoint chung (đang kiểm tra cả Redis) trả về mã lỗi 503 cho cả 3 container.
2. Các Orchestrator (ví dụ Kubernetes, Railway...) gọi liveness probe liên tục đến endpoint này và thấy cả 3 container đều báo lỗi (do liveness probe thất bại).
3. Orchestrator lập tức coi cả 3 container là đã chết và tiến hành **restart cứng** (kill và khởi động lại) đồng thời cả 3 container để cố gắng tự sửa lỗi.
4. Trong lúc 3 container đang khởi động lại, cụm service chat hoàn toàn bị sập (downtime 100%), không nhận được bất kỳ request nào.
5. Khi các container khởi động lại xong, chúng lại gọi Redis và tiếp tục thất bại (vì Redis vẫn chưa phục hồi trong khoảng 30s đó), dẫn đến vòng lặp restart vô hạn (crash loop backoff), làm trầm trọng hóa sự cố mất kết nối Redis ban đầu.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

*   **Lỗi gặp phải:** Khi deploy lên Render, gọi endpoint `/readyz` bị trả về lỗi `503 {"status":"not ready","redis":false}`.
*   **Cách phát hiện:** Gọi thử endpoint `/readyz` bằng `curl` hoặc chạy bộ test `pytest tests/test_cp5.py` và nhận được thông báo lỗi readyz trả 503.
*   **Nguyên nhân:** Biến môi trường `REDIS_URL` trên Web Service chưa được liên kết đúng với service Redis. Khi tạo bằng Blueprint Render, đôi khi service Redis mất nhiều thời gian để khởi động hơn Web Service, hoặc Render chưa tự động binding biến môi trường giữa 2 service nếu deploy bị gián đoạn.
*   **Cách sửa:** Truy cập dashboard Render, vào cấu hình Service Web -> Environment Variables. Kiểm tra và đảm bảo biến `REDIS_URL` có giá trị trỏ đến connection string nội bộ của service Redis `day12-chat-redis` (có định dạng `redis://day12-chat-redis:6379`). Nếu Redis chưa khởi động xong, bấm **Deploy** -> **Clear Build Cache and Deploy** lại Web Service để Render thiết lập lại kết nối.
