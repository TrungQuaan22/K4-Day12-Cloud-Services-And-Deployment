# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Giang Trung Quân |
| Mã học viên | 2A202601098 |
| Repo | https://github.com/TrungQuaan22/K4-Day12-Cloud-Services-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-day12-cloud-services-and-deployment-hf25.onrender.com |
| Platform | Render |
| Ngày deploy | 10/8/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Redis addon (day12-chat-redis) |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness — mong đợi 200 {"status":"ok"}
HTTP/1.1 200 OK
date: Mon, 10 Aug 2026 10:35:57 GMT
content-type: application/json
transfer-encoding: chunked
connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: b9e3b2e3-81f3-49a8
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
content-encoding: gzip
cf-ray: a28e5b542ce5cded-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

==================================================

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
HTTP/1.1 503 Service Unavailable
date: Mon, 10 Aug 2026 10:35:58 GMT
content-type: application/json
transfer-encoding: chunked
connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 5b41e2da-b71f-4b2c
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-ray: a28e5b59ff3bb486-SIN
alt-svc: h3=":443"; ma=86400

{"status":"not ready","redis":false}

==================================================

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
HTTP/1.1 401 Unauthorized
date: Mon, 10 Aug 2026 10:35:59 GMT
content-type: application/json
transfer-encoding: chunked
connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: bd18bdab-731c-4fba
server: cloudflare
vary: Accept-Encoding
www-authenticate: Bearer
x-render-origin-server: uvicorn
cf-ray: a28e5b5eeb9809c8-HKG
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing bearer token"}

==================================================

# 4. Có token — mong đợi 200 kèm câu trả lời
HTTP/1.1 500 Internal Server Error
date: Mon, 10 Aug 2026 10:36:00 GMT
content-type: text/plain; charset=utf-8
transfer-encoding: chunked
connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: eb0a3212-1584-429e
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-ray: a28e5b64785bfe8c-SIN
alt-svc: h3=":443"; ma=86400

Internal Server Error

==================================================

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
500 500 500 500 500 500 500 500 500 500 500 500 500 500 500
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---


