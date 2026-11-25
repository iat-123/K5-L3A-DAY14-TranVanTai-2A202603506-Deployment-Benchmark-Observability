# Deployment Information

> **TODO (chưa deploy):** điền URL, ảnh và kết quả 4 lệnh tự kiểm sau khi deploy `06-lab-complete/` lên Railway.

## Public URL

TODO: https://<your-agent>.up.railway.app

## Platform

Railway. Deploy thư mục `06-lab-complete/` bằng `railway.toml`: builder `DOCKERFILE`, `healthcheckPath = "/health"`, app nghe theo `$PORT` (CMD trong Dockerfile: `exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}`).

Có thể deploy thay bằng Render qua `06-lab-complete/render.yaml` (Blueprint: web service Docker + Key Value Redis, `AGENT_API_KEY` dùng `generateValue: true`).

## Các bước deploy

```bash
npm i -g @railway/cli
railway login
cd 06-lab-complete
railway init                              # tạo project
railway add --database redis              # thêm Redis service
railway up                                # build từ Dockerfile và deploy
railway variables --set "AGENT_API_KEY=<key ngẫu nhiên>" \
                  --set "REDIS_URL=\${{Redis.REDIS_URL}}" \
                  --set "ENVIRONMENT=production"
railway domain                            # tạo public URL
```

## Test Commands

```bash
BASE=https://<your-agent>.up.railway.app
AGENT_API_KEY=<key đã đặt trên Railway>

curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả: TODO

Kiểm tra thêm bằng script chấm: `python check_production_ready.py --public-url $BASE`

## Environment Variables Set

Đặt trên Railway (Variables), **không** ghi trong code hay git:

| Biến | Giá trị |
|---|---|
| `AGENT_API_KEY` | secret ngẫu nhiên (không ghi ở đây) |
| `REDIS_URL` | tham chiếu tới Redis service của Railway: `${{Redis.REDIS_URL}}` |
| `ENVIRONMENT` | `production` |
| `PORT` | do Railway tự cấp |
| `RATE_LIMIT_PER_MINUTE` | `10` (mặc định trong code nếu không đặt) |
| `DAILY_BUDGET_USD` / `MONTHLY_BUDGET_USD` | `1.0` / `10.0` (mặc định) |
| `LOG_LEVEL` | `INFO` |

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png): TODO
- [Service running](screenshots/running.png): TODO
- [Test results](screenshots/test.png): TODO
