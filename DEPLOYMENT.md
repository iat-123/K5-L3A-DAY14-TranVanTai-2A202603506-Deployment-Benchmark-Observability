# Deployment Information

## Public URL

https://agent-production-dd52d810.up.railway.app

## Platform

**Railway** (Trial plan, region US East). Project `k5-day14-agent`, environment `production`, gồm 2 service:

- `agent`: deploy thư mục `06-lab-complete/` bằng `railway up`. Railway đọc `railway.toml` (builder `DOCKERFILE`, `healthcheckPath = "/health"`) và build từ `Dockerfile`. CMD `exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000} --workers 2` nghe theo `PORT` do Railway cấp (log startup: `port: 8080`).
- `Redis`: database Redis của Railway (`redis:8.2`, có volume). Agent kết nối qua mạng nội bộ `redis.railway.internal:6379`.

Các bước đã chạy:

```bash
npm i -g @railway/cli
railway login --browserless               # xác nhận trên railway.com/activate
cd 06-lab-complete
railway init --name k5-day14-agent
railway add --database redis
railway add --service agent
railway variables --service agent --skip-deploys \
  --set "AGENT_API_KEY=<secret>" --set 'REDIS_URL=${{Redis.REDIS_URL}}' \
  --set "ENVIRONMENT=production" --set "LOG_LEVEL=INFO"
railway up --service agent --ci           # build Dockerfile → "Deploy complete"
railway domain --service agent            # → agent-production-dd52d810.up.railway.app
```

## Test Commands

```bash
BASE=https://agent-production-dd52d810.up.railway.app
AGENT_API_KEY=<key đặt trên Railway>

curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả thật (2026-11-25 15:46 UTC):

```
200
200
401
200

$ curl -sS "$BASE/health"
{"status":"ok","environment":"production"}

$ curl -sS -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"What is deployment?"}'
{"question":"What is deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được."}
```

Script chấm: `python check_production_ready.py --public-url https://agent-production-dd52d810.up.railway.app` cho `✅ Public URL: GET .../health → 200`.

## Environment Variables Set

Đặt trên Railway (service `agent` → Variables), **không** ghi trong code hay git:

| Biến | Giá trị |
|---|---|
| `AGENT_API_KEY` | secret ngẫu nhiên 32 ký tự (không ghi ở đây) |
| `REDIS_URL` | `${{Redis.REDIS_URL}}`, tham chiếu tới Redis service (`redis://...@redis.railway.internal:6379`) |
| `ENVIRONMENT` | `production` |
| `LOG_LEVEL` | `INFO` |
| `PORT` | Railway tự cấp (8080) |
| `RATE_LIMIT_PER_MINUTE`, `DAILY_BUDGET_USD`, `MONTHLY_BUDGET_USD` | không đặt, dùng mặc định trong `app/config.py` (10 / 1.0 / 10.0) |

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png): project canvas, `agent` và `Redis` đều Online
- [Service running](screenshots/running.png): deployment Active, deploy logs có log JSON (`event: startup`, `request`, `llm_call`) và các request 200/401
- [Test results](screenshots/test.png): 4 lệnh tự kiểm cho ra 200 / 200 / 401 / 200
