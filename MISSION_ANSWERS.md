# Day 14 Lab - Mission Answers

> Môi trường: Windows 11, Python 3.12, Docker Desktop 29.9.3. Mọi output dưới đây là output thật của các lệnh đã chạy (token/key đã che).

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found (`01-localhost-vs-production/develop/app.py`)

1. **Secret hardcode trong code**: `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@..."`. Push lên GitHub là lộ key và mật khẩu DB, và muốn đổi key phải sửa code rồi deploy lại.
2. **Log ra secret**: `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`. Output thật khi gọi `/ask`:
   ```
   [DEBUG] Got question: Hello
   [DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
   ```
   Log thường được gửi lên hệ thống log tập trung, nên ai đọc được log là có key.
3. **Dùng `print()` thay vì logging có cấu trúc**: không có level, timestamp hay field. Không lọc, tìm kiếm hay alert theo log được.
4. **Không có config management**: `DEBUG = True`, `MAX_TOKENS = 500` cố định trong code. Không đổi được theo môi trường dev/staging/prod.
5. **Không có health check**: `GET /health` trả **404**. Platform không biết agent còn sống để restart, load balancer không biết để ngừng route.
6. **Host/port cố định**: `host="localhost"`, `port=8000`. Bind `localhost` thì chạy trong container không nhận được kết nối từ ngoài, còn Railway/Render cấp port qua biến `PORT` nên port cố định sẽ không nhận traffic.
7. **`reload=True` trong production**: tốn tài nguyên để theo dõi file, và tự restart khi file đổi.
8. **Không xử lý shutdown**: khi bị tắt, request đang chạy bị cắt ngang.
9. **Input không validate**: `question: str` là query param, không giới hạn độ dài. Gửi JSON body thì trả 422.

### Exercise 1.2: Chạy basic version

```
$ curl -i -X POST "http://localhost:8000/ask?question=Hello"
HTTP/1.1 200 OK
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}

$ curl -o /dev/null -w "%{http_code}" http://localhost:8000/health   → 404
$ curl ... -X POST /ask -H "Content-Type: application/json" -d '{"question":"Hello"}'   → 422
```
Chạy được, nhưng chưa sẵn sàng cho production (xem 1.1).

### Exercise 1.3: Comparison table

Bản production (`01-localhost-vs-production/production/app.py`) chạy thật:
```
GET /health → {"status":"ok","uptime_seconds":5.3,"version":"1.0.0","environment":"development",...}
GET /ready  → {"ready":true}
POST /ask (JSON) → {"question":"Hello","answer":"...","model":"gpt-4o-mini"}
log: WARNING:root:OPENAI_API_KEY not set — using mock LLM
```

| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trong code (`DEBUG=True`, key, DB URL) | Đọc từ env qua `config.py` (`settings`), có `.env.example` | Cùng một image chạy được ở mọi môi trường; đổi config không cần build lại; secret không nằm trong git |
| Secrets | Ghi thẳng vào code và còn in ra log | Lấy từ env, không log secret (chỉ log `question_length`) | Lộ key nghĩa là mất tiền và mất dữ liệu; log thường được gửi ra ngoài |
| Health check | Không có (`/health` trả 404) | `/health` (liveness, có uptime) + `/ready` (readiness, 503 khi chưa sẵn sàng) | Platform tự restart container chết; LB chỉ route tới instance sẵn sàng |
| Logging | `print()` | JSON qua `logging` (`{"time":..,"level":..,"msg":..}`) | Parse và tìm được trong Loki/Datadog, lọc theo field, gắn alert |
| Shutdown | Đột ngột | `lifespan` + handler SIGTERM: đặt `is_ready=False`, chờ request xong | Không làm rơi request khi deploy hoặc scale down |
| Host / Port | `localhost:8000` cố định | `0.0.0.0` + `PORT` từ env | Chạy được trong container và nhận port do platform cấp |
| Reload/Debug | `reload=True` luôn bật | Chỉ bật khi `DEBUG=true` | Production cần ổn định, không tự restart |
| Input | Query param, không validate | JSON body, 422 khi thiếu `question` | Mã lỗi rõ ràng, chống input xấu |
| CORS | Không có | Chỉ cho `allowed_origins` cấu hình sẵn | Hạn chế site lạ gọi API từ trình duyệt |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)

1. **Base image:** `python:3.11`, bản Python đầy đủ (Debian + build tools, ~1 GB).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao COPY requirements.txt trước:** để tận dụng layer cache. Layer `pip install` chỉ build lại khi `requirements.txt` đổi; sửa code (`app.py`) chỉ làm build lại các layer sau `COPY app.py`. Nếu `COPY . .` trước thì mỗi lần sửa code là cài lại toàn bộ dependencies.
4. **CMD vs ENTRYPOINT:** `ENTRYPOINT` là chương trình cố định của container, tham số `docker run <image> args` được *nối thêm* vào sau. `CMD` là lệnh/tham số mặc định, `docker run <image> <lệnh khác>` sẽ *thay thế* toàn bộ. Hay dùng kết hợp: `ENTRYPOINT ["uvicorn"]` + `CMD ["main:app", "--port", "8000"]`. Cả hai nên viết dạng exec (JSON array) để process là PID 1 và nhận được SIGTERM.

### Exercise 2.2: Build và run

```
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -p 8000:8000 my-agent:develop
$ curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker%3F"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Image size comparison

`02-docker/production/Dockerfile` là multi-stage:
- **Stage 1 (builder)**, `python:3.11-slim` + `gcc`, `libpq-dev`: cài dependencies bằng `pip install --user` vào `/root/.local`. Stage này **không** được deploy.
- **Stage 2 (runtime)**, `python:3.11-slim` sạch: tạo user `appuser` non-root, chỉ copy `/root/.local` (site-packages) + `main.py` + `utils/`, có `HEALTHCHECK`, chạy `uvicorn --workers 2`.
- **Tại sao nhỏ hơn:** image cuối không chứa compiler, apt cache, header file hay pip cache của stage builder, và base `slim` thay cho base đầy đủ (~1 GB).

```
$ docker images | grep my-agent
my-agent:advanced 240MB
my-agent:develop  1.67GB
```
- Develop: **1667.8 MB** (1,667,771,682 bytes)
- Production: **239.8 MB** (239,811,269 bytes)
- Difference: nhỏ hơn **85.6%** (giảm ~6.95 lần)

Image Part 6 (`06-lab-complete/Dockerfile`, multi-stage venv + slim + non-root): **273 MB** theo `check_production_ready.py`.

### Exercise 2.4: Docker Compose stack

```
                 :80
  client ──▶ ┌─────────┐      ┌──────────────┐
             │  nginx  │────▶ │ agent :8000  │──┬──▶ redis :6379  (session, rate limit)
             └─────────┘      └──────────────┘  └──▶ qdrant :6333 (vector DB)
               mạng bridge "internal" — chỉ nginx publish port ra host
```

Services được start (`docker compose -p k5-day14-docker up -d --build`): `redis` (healthy), `qdrant` (healthy), `agent` (healthy, chờ `depends_on: condition: service_healthy` của redis + qdrant), `nginx`. Chúng nói chuyện với nhau qua DNS nội bộ của network `internal` theo tên service (`redis://redis:6379/0`, `http://qdrant:6333`, `proxy_pass http://agent_backend` → `agent:8000`). Agent không publish port, mọi traffic từ ngoài đi qua nginx.

```
$ curl -i http://localhost/health
HTTP/1.1 200 OK
{"status":"ok","uptime_seconds":14.6,"version":"2.0.0","timestamp":"2026-11-25T14:49:06.582424"}

$ curl -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question":"Explain microservices"}'
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment (agent Part 6, thư mục `06-lab-complete/`)

- URL: **https://agent-production-dd52d810.up.railway.app** (chi tiết: [DEPLOYMENT.md](DEPLOYMENT.md))
- Deploy bằng Railway CLI (`railway up` trong `06-lab-complete/`, build từ `Dockerfile` theo `railway.toml`) + Redis của Railway; `AGENT_API_KEY`, `REDIS_URL=${{Redis.REDIS_URL}}`, `ENVIRONMENT` đặt trong Railway Variables.
- 4 lệnh tự kiểm CP6: `/health` → **200**, `/ready` → **200**, `/ask` không key → **401**, `/ask` có key → **200**.
- Xem log: dashboard → service `agent` → Deployments → View logs (hoặc `railway logs --service agent`).
- Screenshot: [screenshots/dashboard.png](screenshots/dashboard.png), [screenshots/running.png](screenshots/running.png), [screenshots/test.png](screenshots/test.png)

### Exercise 3.2: So sánh `railway.toml` với `render.yaml`

| | `railway.toml` | `render.yaml` |
|---|---|---|
| Định dạng / phạm vi | TOML, cấu hình **một service** (build + deploy) | YAML Blueprint, mô tả **cả hạ tầng**: nhiều service (web + Key Value/Redis), env và liên kết giữa chúng |
| Build | `builder = "DOCKERFILE"`, `dockerfilePath` | `runtime: docker`, `dockerfilePath`, `rootDir: 06-lab-complete` (vì blueprint đọc từ gốc repo) |
| Health check | `healthcheckPath = "/health"`, `healthcheckTimeout` | `healthCheckPath: /health` |
| Restart | `restartPolicyType = "ON_FAILURE"`, `restartPolicyMaxRetries` | Render tự restart, không khai báo trong file |
| Biến môi trường / secret | **Không** nằm trong file: đặt bằng `railway variables set` hoặc dashboard | Khai báo trong file: giá trị thường dùng `value`, secret dùng `generateValue: true` (`AGENT_API_KEY`) hoặc `sync: false`, không ghi giá trị thật |
| Redis | Tạo Redis service riêng trên Railway rồi đặt `REDIS_URL` trỏ tới nó | Khai báo luôn service `type: keyvalue` và lấy `REDIS_URL` bằng `fromService ... property: connectionString` |
| Deploy | `railway up` (CLI đẩy code từ máy) | Push GitHub → New Blueprint → Render đọc file và tự deploy mỗi lần push |

Vì sao khác: Railway hướng tới "1 lệnh từ CLI", cấu hình tối thiểu, phần còn lại làm trên dashboard. Render Blueprint theo kiểu Infrastructure-as-Code: toàn bộ stack (web + Redis + env) nằm trong git để dựng lại y hệt được, nên nó phải có cơ chế riêng cho secret (`generateValue`, `sync: false`) để không lộ giá trị.

### Exercise 3.3: Cloud Run CI/CD (`03-cloud-deployment/production-cloud-run/`)

`cloudbuild.yaml` chạy trên Cloud Build mỗi khi có trigger (push lên repo), gồm 4 step nối tiếp bằng `waitFor`:
1. **test**: trong `python:3.11-slim`, cài requirements và chạy `pytest tests/`; test fail thì dừng pipeline.
2. **build**: `docker build` với tag `:$COMMIT_SHA` và `:latest`, dùng `--cache-from :latest` để tận dụng layer cache.
3. **push**: đẩy cả hai tag lên Container Registry `gcr.io/$PROJECT_ID/ai-agent`.
4. **deploy**: `gcloud run deploy` image theo đúng `$COMMIT_SHA` (truy vết và rollback được) lên `asia-southeast1`; `min-instances=1` tránh cold start, `max-instances=10`, 512Mi/1 CPU, timeout 60s; secret lấy từ **Secret Manager** (`--set-secrets`), không nằm trong code.

`service.yaml` là khai báo Knative tương đương: autoscaling theo `containerConcurrency: 80`, `livenessProbe` → `/health`, `startupProbe` → `/ready`, `AGENT_API_KEY` và `OPENAI_API_KEY` lấy từ `secretKeyRef`.

---

## Part 4: API Security

### Exercise 4.1: API Key authentication (`04-api-gateway/develop`, chạy `AGENT_API_KEY=secret-key-123 python app.py`)

- **Key được check ở đâu:** dependency `verify_api_key()` dùng `APIKeyHeader(name="X-API-Key")`, gắn vào `/ask` bằng `Depends(verify_api_key)`. Key so với `API_KEY = os.getenv("AGENT_API_KEY", ...)`.
- **Sai key thì sao:** thiếu header trả **401**, có header nhưng sai trả **403** (bản Part 6 của mình trả 401 cho cả hai, đúng hợp đồng chấm).
- **Rotate key:** đổi biến môi trường `AGENT_API_KEY` trên platform rồi restart/redeploy, không cần sửa code. Muốn rotate không downtime thì cho phép cùng lúc 2 key (cũ + mới), chuyển client sang key mới, rồi bỏ key cũ.

```
$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

$ curl -i -X POST http://localhost:8000/ask -H "X-API-Key: wrong" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid API key."}

$ curl -i -X POST http://localhost:8000/ask -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 4.2: JWT authentication (`04-api-gateway/production`)

JWT flow: `POST /auth/token` kiểm tra username/password (`DEMO_USERS`), sau đó `create_token()` ký HS256 payload `{sub, role, iat, exp=+60 phút}` bằng `JWT_SECRET`. Client gửi `Authorization: Bearer <token>`, và `verify_token()` chỉ cần kiểm tra chữ ký + `exp`, không tra DB. Token hết hạn trả 401 "Token expired".

```
$ curl -i -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" -d '{"username":"student","password":"demo123"}'
HTTP/1.1 200 OK
{"access_token": "eyJhbGciOiJI...<redacted>", "token_type": "bearer", "expires_in_minutes": 60, ...}

$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Explain JWT"}'
HTTP/1.1 401 Unauthorized
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}
```

### Exercise 4.3: Rate limiting

- **Algorithm:** Sliding Window Log. Mỗi user có một `deque` timestamp, bỏ các timestamp cũ hơn 60 s, đếm số còn lại.
- **Limit:** `rate_limiter_user` = **10 req/phút**, `rate_limiter_admin` = 100 req/phút.
- **Bypass cho admin:** `app.py` chọn limiter theo role trong JWT (`rate_limiter_admin if role == "admin" else rate_limiter_user`), nên admin (`teacher`) có hạn mức 100/phút. Muốn bỏ hẳn giới hạn thì bỏ qua `limiter.check()` khi `role == "admin"`.
- **Hạn chế:** state nằm trong RAM nên mỗi instance đếm riêng và mất khi restart. Bản Part 6 của mình dùng Redis sorted set (ZREMRANGEBYSCORE → ZCARD → ZADD → EXPIRE) để mọi instance dùng chung và state vẫn còn sau restart.

Vòng lặp 20 request với token của `student`:
```
req 1 -> 200
...
req 10 -> 200
req 11 -> 429 x-ratelimit-remaining: 0 retry-after: 57  {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":57}}
req 12 -> 429 x-ratelimit-remaining: 0 retry-after: 56  {...}
...
req 20 -> 429 x-ratelimit-remaining: 0 retry-after: 52  {...}

$ curl http://localhost:8000/me/usage -H "Authorization: Bearer <token>"
{"user_id":"student","date":"2026-11-25","requests":10,"input_tokens":40,"output_tokens":304,"cost_usd":0.000188,"budget_usd":1.0,...}
```

### Exercise 4.4: Cost guard implementation

Cách làm trong `06-lab-complete/app/cost_guard.py` (state trong Redis, dùng chung giữa các worker/instance):
- **Tính cost theo token:** `estimate_tokens()` ≈ 4 ký tự/token; `cost = in/1000 × $0.00015 + out/1000 × $0.0006` (giá gpt-4o-mini).
- **Hai key cộng dồn:** `cost:daily:YYYY-MM-DD` (TTL 2 ngày) và `cost:monthly:YYYY-MM` (TTL 32 ngày). Key theo ngày/tháng nên tự "reset đầu tháng" mà không cần cron, TTL dọn key cũ.
- **`check_budget()` gọi TRƯỚC khi gọi LLM:** `MGET` hai key, vượt `DAILY_BUDGET_USD` hoặc `MONTHLY_BUDGET_USD` (từ env) thì `HTTPException(402)`. Redis lỗi thì trả 503 (fail closed: không cho tiêu tiền khi không kiểm soát được budget).
- **`record_cost()` gọi SAU khi LLM trả lời:** `INCRBYFLOAT` cả hai key trong một pipeline. Ghi Redis lỗi thì bỏ qua để không làm hỏng câu trả lời đã có.
- Phiên bản theo user như đề (`budget:{user_id}:{YYYY-MM}`, $10/tháng) chỉ khác ở tên key (thêm `user_id`). Kiểm tra `current + estimated_cost > 10` trước, rồi mới `INCRBYFLOAT` + `EXPIRE 32 ngày`.
- Kết quả chấm: `Khi chạy: hết budget → 402` ✅.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks (`05-scaling-reliability/develop`)

- `/health` (liveness): luôn 200 khi process còn sống, kèm `uptime` và check memory. Không phụ thuộc DB/Redis, vì nếu Redis chết mà liveness fail thì platform restart cả loạt container vô ích.
- `/ready` (readiness): 503 khi đang khởi động hoặc đang shutdown (`_is_ready=False`), hoặc khi dependency (Redis) không kết nối được. LB ngừng route tới instance đó nhưng không restart.

```
GET /health -> {"status":"ok","uptime_seconds":0.4,"version":"1.0.0","environment":"development",...,"checks":{"memory":{"status":"ok","used_percent":9.1}}}
GET /ready  -> {"ready":true,"in_flight_requests":1}
```
Part 6 của mình: `/ready` gọi `redis_healthy()` (PING, timeout 2s), Redis down thì 503 trong khi `/health` vẫn 200 (checker: ✅ "Redis mất kết nối → /ready 503 (còn /health vẫn 200)").

### Exercise 5.2: Graceful shutdown

Uvicorn tự bắt SIGTERM: ngừng nhận kết nối mới, chờ request đang chạy xong, rồi chạy phần sau `yield` của `lifespan` (đặt `_is_ready=False`, chờ `_in_flight_requests` về 0, đóng kết nối). Handler SIGTERM tự viết chỉ để log. Nếu tự đăng ký handler thì phải giữ handler của uvicorn, nếu không process sẽ không bao giờ thoát.

Test (chạy trong container Linux `python:3.11-slim`, vì Python trên Windows không gửi được SIGTERM): gửi `POST /ask?question=Long task`, 0.1 s sau `kill -TERM <pid>`:
```
sent SIGTERM to 15
in-flight request -> 200 {"answer":"Đây là câu trả lời từ AI agent (mock)..."} (3.02s)
app exit code: 0
--- app log
INFO:     Shutting down
INFO:     Waiting for application shutdown.
2026-11-25 14:51:41,239 INFO 🔄 Graceful shutdown initiated...
2026-11-25 14:51:41,239 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [15]
```
Request đang chạy vẫn hoàn thành (200) và process tự thoát với exit code 0.

### Exercise 5.3: Stateless design

`05-scaling-reliability/production/app.py` lưu toàn bộ session/history trong Redis (`session:{id}`, `SETEX` TTL 1h, giữ 20 message gần nhất), không dùng dict trong RAM. Lý do: mỗi instance có bộ nhớ riêng, nên dict `conversation_history` chỉ thấy được ở instance đã tạo nó. Request sau rơi vào instance khác là mất history, và restart/scale down là mất sạch. Part 6 cũng stateless: rate limit và cost đều nằm trong Redis (checker ✅ "state nằm trong Redis — restart agent vẫn bị 429").

### Exercise 5.4: Load balancing

```
$ docker compose -p k5-day14-scaling up -d --build --scale agent=3
agent k5-day14-scaling-agent-1 running
agent k5-day14-scaling-agent-2 running
agent k5-day14-scaling-agent-3 running
nginx k5-day14-scaling-nginx-1 running
redis k5-day14-scaling-redis-1 running
```
10 request `POST /chat` qua Nginx (cổng 8080), đếm `served_by`:
```
      4 instance-85784b
      3 instance-aef852
      3 instance-b5f7f5
```
Nginx chia round-robin (`upstream agent_cluster { server agent:8000; }`, Docker DNS trả 3 IP). `proxy_next_upstream error timeout http_503` cho phép thử instance khác khi một instance lỗi.

### Exercise 5.5: Test stateless

```
$ python test_stateless.py
Session ID: b9163dda-30bd-4d95-9efd-fbe755f82b27
Request 1: [instance-85784b]  Q: What is Docker?
Request 2: [instance-b5f7f5]  Q: Why do we need containers?
Request 3: [instance-aef852]  Q: What is Kubernetes?
Request 4: [instance-85784b]  Q: How does load balancing work?
Request 5: [instance-b5f7f5]  Q: What is Redis used for?
Total requests: 5
Instances used: {'instance-aef852', 'instance-85784b', 'instance-b5f7f5'}
✅ All requests served despite different instances!
--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```
5 câu của cùng một session đi qua cả 3 instance mà history vẫn đủ 10 message, vì instance nào cũng đọc/ghi cùng key trong Redis.

Thử thêm (script không tự tắt instance): tạo session, rồi `docker stop k5-day14-scaling-agent-1`, gửi tiếp 3 câu cùng `session_id`:
```
stopped agent-1
turn 3 served_by instance-85784b
turn 4 served_by instance-b5f7f5
turn 5 served_by instance-85784b
history messages: 8
```
Mất 1 instance thì traffic chuyển sang 2 instance còn lại và session vẫn còn đủ 4 lượt (8 message). (Trường `turn` của app đếm lệch +1 vì tính sau khi đã append câu hỏi, đây là chi tiết của code mẫu.)

---

## Part 6: Final Project — output `check_production_ready.py`

Chạy trong `06-lab-complete/` trên Windows có Docker (`python check_production_ready.py`, bản đầy đủ):

```
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ⚠️  Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 11/12 tiêu chí đạt · 42/42 checks passed
  ⚠️  Chưa kiểm tra được: Graceful shutdown
```

Mục ⚠️ duy nhất là check runtime "SIGTERM lúc đang có request". Script tự bỏ qua check này trên Windows ("Windows không gửi được SIGTERM"). Các check khác của tiêu chí 11 đều ✅, gồm `Docker: docker stop tắt êm (exit code 0, < 10s) — exit code 0, 0.9s`. Đã chạy check SIGTERM đó trong môi trường Linux (container `python:3.11-slim`, `os.name = posix`, Redis thật):

```
⚠️  11. Graceful shutdown
     ✅ Code xử lý SIGTERM / lifespan shutdown
     ✅ Khi chạy: SIGTERM lúc đang có request → request vẫn 200, process tự thoát — exit code 0
     ⚠️  Docker: `docker stop` tắt êm (exit code 0, < 10s) — chưa chạy (--skip-docker)
```

Gộp hai lần chạy thì mọi check của 12 tiêu chí đều ✅. **TODO:** chạy lại một lần trong WSL2 (Docker Desktop bật WSL Integration cho Ubuntu) để có một output `PASS 12/12` duy nhất rồi thay khối trên.

Một số chi tiết từ output: image 273 MB; 429 sau 5 request (`RATE_LIMIT_PER_MINUTE=5`); restart agent vẫn 429 (state trong Redis); 402 khi budget 0.000001; 17 dòng log JSON hợp lệ.

---

## AI log

Bài làm dùng **Claude Code** (Anthropic) làm trợ lý trong VS Code. Tóm tắt từng lần dùng:

| # | Hỏi / giao việc | Nhận được | Tự kiểm tra thế nào |
|---|---|---|---|
| 1 | Đọc và tóm tắt cấu trúc repo, các TODO của Part 6 và yêu cầu nộp bài | Bảng các file TODO, danh sách file nộp, lưu ý chạy SIGTERM trong WSL2 | Đối chiếu với README, SUBMISSION, RUBRIC và đọc `check_production_ready.py` để biết chính xác từng check |
| 2 | Hoàn thành TODO Part 6 (`config`, `auth`, `redis_client`, `rate_limiter`, `cost_guard`, `main`, Dockerfile, compose, `.dockerignore`, `.env.example`, `railway.toml`, `render.yaml`) | Code như trong `06-lab-complete/`: Redis sorted set cho rate limit, `INCRBYFLOAT` theo ngày/tháng cho cost, `JsonFormatter`, `lifespan`, Dockerfile venv multi-stage + `exec uvicorn ${PORT}` | Chạy `check_production_ready.py --static-only` rồi bản đầy đủ (42/42 checks); chạy check SIGTERM trong container Linux; đọc lại từng file, hiểu vì sao `/health` không phụ thuộc Redis và vì sao uvicorn phải là PID 1 |
| 3 | Chạy benchmark Part 7 (load test 4 mức concurrency, 3 mức `MOCK_SLOW_RATE`, eval v1/v2) | Script chạy lệnh; phát hiện `run_eval.py` lỗi encoding cp1252 trên Windows nên chạy lại với `PYTHONUTF8=1` | Mọi số trong `BENCHMARK_REPORT.md` lấy từ `reports/*.json` |
| 4 | Sửa 2 câu sai `security-02`, `scaling-03` | Thêm 2 entry vào `KNOWLEDGE_BASE` (JWT/API key; Kubernetes HPA), không sửa golden dataset | `eval_v2_fixed.json`: 23/23, 0 regression, gate PASS |
| 5 | Bật observability stack, chạy Locust, làm alert firing, tìm trace | Phát hiện Dockerfile Part 7 thiếu `PYTHONUSERBASE` (agent không import được uvicorn) và lệnh benchmark scale thiếu `--no-deps` (số agent bị đưa về 2) | Đếm `count(up{job="agent"})` trên Prometheus và số container trước/sau mỗi lần chạy; trace_id khớp log JSON; ảnh chụp bằng Edge headless |
| 7 | Deploy agent Part 6 lên Railway | Các lệnh Railway CLI (init, add Redis, variables, up, domain); key sinh ngẫu nhiên, chỉ lưu trong Railway Variables và file local đã gitignore | Chạy 4 lệnh tự kiểm CP6 (200/200/401/200), `check_production_ready.py --public-url` ✅, xem deploy logs có log JSON |
| 6 | Chạy các exercise Part 1–5 và viết `MISSION_ANSWERS.md` | Output thật của các lệnh curl, docker, test_stateless; bản nháp câu trả lời | Đọc lại code từng Part để đối chiếu câu trả lời (vd 4.1 sai key trả 403 chứ không phải 401) |

Không dùng AI để bịa số đo: mọi số liệu là output của lệnh đã chạy trên máy này, và file report/CSV/ảnh tương ứng có trong repo.
