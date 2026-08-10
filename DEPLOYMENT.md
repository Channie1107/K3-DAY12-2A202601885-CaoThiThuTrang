# Thông Tin Deploy - Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Cao Thị Thu Trang |
| Mã học viên | 2A202601885 |
| Repo | `K3-DAY12-2A202601885-CaoThiThuTrang` |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-ecb6.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán, app đọc `$PORT` |
| `AGENT_API_KEY` | ✅ | đặt qua `railway variables`, không commit lên repo |
| `REDIS_URL` | ✅ | tham chiếu `${{Redis.REDIS_URL}}` tới Redis nội bộ Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | `10` |
| `MONTHLY_BUDGET_USD` | ✅ | `10.0` |
| `LOG_LEVEL` | ✅ | `INFO` |

## Lệnh Kiểm Tra

```bash
curl -i https://agent-production-ecb6.up.railway.app/health
curl -i https://agent-production-ecb6.up.railway.app/ready
curl -i -X POST https://agent-production-ecb6.up.railway.app/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```

## Kết Quả Chạy Thật

```
GET  /health                     -> 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready                      -> 200 {"status":"ready","redis":true}
POST /ask (không có API key)     -> 401
POST /ask (có API key thật)      -> 200, trả lời từ mock LLM
```

## Sự Cố Gặp Phải Khi Deploy (và cách sửa)

1. **`The executable 'uvicorn' could not be found.`** — `railway.toml` có
   `startCommand = "uvicorn ..."` gọi thẳng file thực thi `uvicorn`, nhưng
   `Dockerfile` chỉ thêm `/install/lib/python3.11/site-packages` vào
   `PYTHONPATH` chứ không thêm `/install/bin` vào `PATH` hệ thống, nên shell
   không tìm thấy lệnh `uvicorn`. Sửa: đổi `startCommand` sang
   `python -m uvicorn ...` (giống cách `CMD` trong Dockerfile gọi, dùng
   `PYTHONPATH` thay vì `PATH`).

2. **`Error: Invalid value for '--port': '$PORT' is not a valid integer.`** —
   Railway chạy `startCommand` không qua shell nên `$PORT` không được nội suy
   thành giá trị thật, bị truyền nguyên văn chuỗi `"$PORT"` vào uvicorn. Sửa:
   bọc lệnh trong `sh -c "..."` để shell nội suy biến trước khi thực thi.

## Ảnh Chụp Màn Hình

- `screenshots/railway-dashboard.png` — dashboard Railway, service `agent` Online, "Deployment successful"
- `screenshots/ready.png` — `GET /ready` trên domain thật, `{"status":"ready","redis":true}`
- `screenshots/ask.png` — `curl -X POST .../ask` không kèm key, trả `401`
- `screenshots/health-local-fallback-cu.png` — ảnh cũ chụp lúc còn dùng local fallback, giữ lại làm tư liệu
