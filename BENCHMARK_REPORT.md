# Benchmark & Observability Report — Day 14

**Họ tên:** Trần Văn Tài — 2A202603506
**Ngày chạy:** 2026-11-25
**Môi trường:** Laptop Windows 11, AMD Ryzen 7 4800H (16 threads), 23 GB RAM; Docker Desktop 29.9.3 (16 CPU / 11.4 GB cấp cho VM). Bước 1 chạy Python 3.12 trên Windows (agent và load generator cùng máy); Bước 2 chạy bằng Docker Compose.

Số liệu lấy từ: `reports/*.json` (load test, eval), `benchmark-results/scale_*_stats.csv` (Locust), Prometheus API và Jaeger API lúc chạy stack, ảnh trong `screenshots/`.

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | LLM mock mất ~120–200 ms/call; 500 ms cho dư ~2.5 lần để chứa retrieval + hàng đợi khi có tải, vẫn đủ nhanh cho chat. Đặt theo p95 vì ~3% request chậm bất thường (tail) làm average và p99 nhảy vọt. |
| Error rate `/ask` | < 2 % | Upstream LLM lỗi ngẫu nhiên ~1% (`MOCK_ERROR_RATE=0.01`); error budget 2% cho phép gấp đôi mức nền trước khi coi là vi phạm. Alert đặt lỏng hơn (5% trong 1 phút) để tránh alert fatigue. |
| Accuracy trên golden dataset | ≥ 80 % (gate mặc định), mục tiêu 100% | 80% là quality gate của `run_eval.py`; bản v2 sau khi sửa KB đạt 23/23 nên dùng `--min-accuracy 1.0` + "không regression" để chặn merge. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

`python load_test.py --url http://localhost:8002 --requests 300 --concurrency C --out reports/load_v2_cC.json`

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.39 | 201.7 | 233.7 | 1677.6 | 1.00% | ✅ PASS |
| 10 | 34.25 | 199.6 | 466.7 | 1695.5 | 0.00% | ✅ PASS |
| 50 | 94.05 | 215.6 | 557.9 | 1709.3 | 1.00% | ❌ FAIL (p95) |
| 100 | 44.17 | 797.1 | 4193.7 | 4744.5 | 0.67% | ❌ FAIL (p95) |

**Nhận xét:** RPS **không tăng tuyến tính**. Từ 1 → 10 tăng ~7.8 lần (gần tuyến tính, vì mỗi request chủ yếu là `await asyncio.sleep` của mock LLM nên event loop phục vụ song song được). Từ 10 → 50 chỉ tăng ~2.7 lần và p95 vượt 500 ms: bắt đầu có hàng đợi. Ở 100, RPS còn **giảm** xuống 44 và p50 lên 797 ms: agent chạy 1 process uvicorn, cộng thêm load generator chạy trên cùng laptop Windows nên tranh CPU / socket. Request phải xếp hàng, latency tăng mà throughput không tăng thêm. Đây là điểm bão hoà, trên mức này chỉ còn tăng latency. Muốn tăng RPS thì phải scale ngang (thêm instance/worker), tăng concurrency phía client không giúp.

### 2.2 Tail latency

`MOCK_SLOW_RATE=<x> AGENT_VERSION=v2 PORT=8002 python app.py`, rồi `load_test.py --requests 300 --concurrency 20 --out reports/load_v2_slow_<x>.json`

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 213.8 | 516.5 | 527.8 |
| 0.03 (mặc định) | 256.1 | 487.0 | 1726.3 |
| 0.1 | 368.8 | 1692.1 | 1724.8 |

**Nhận xét:** Chỉ số nhạy nhất là **p99** (và p95 khi tỉ lệ request chậm vượt 5%). Từ 0 → 0.03, avg chỉ tăng 20% (213.8 → 256.1 ms) nhưng p99 tăng hơn 3 lần (527.8 → 1726.3 ms). Ở 0.1, 10% request chậm nên request chậm lọt vào cả top 5%, p95 lên 1692 ms, trong khi avg vẫn chỉ 369 ms, trông có vẻ "ổn". Average gộp số ít request rất chậm với số đông request nhanh nên che mất trải nghiệm của nhóm người dùng bị chậm. Đặt SLO theo average thì 1/10 người dùng có thể chờ ~1.7 s mà SLO vẫn xanh. (Riêng hàng `0` có p95 = 516 ms dù không có request chậm: đây là thời gian xếp hàng ở concurrency 20 khi agent và load generator chạy chung máy, không phải tail của LLM. Max chỉ 540 ms.)

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.3 | 109.3 | 116.5 |
| Tổng cost (USD) | 0.000408 | 0.000764 | 0.000815 |
| Cost / câu đúng (USD) | 0.00040755 | 0.00003639 | 0.00003543 |
| Quality gate | FAIL | PASS (vs baseline v1, 20 câu cải thiện, 0 regression) | PASS (`--min-accuracy 1.0`, vs baseline v2, 2 câu cải thiện, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách sửa:**

| ID | Keyword bị thiếu | Đã sửa gì |
|---|---|---|
| `security-02` "JWT khác gì API key trong xác thực?" | `token`, `hết hạn`, `chữ ký` | KB không có entry nào khớp `jwt` / `api key` nên v2 trả lời chung chung. Thêm entry trigger `["jwt", "api key", "xác thực"]` với nội dung: JWT là **token** có **chữ ký** số, chứa thời điểm **hết hạn** (exp); API key là chuỗi cố định, không tự hết hạn. |
| `scaling-03` "Kubernetes HPA scale dựa trên những chỉ số nào?" | `cpu`, `memory`, `custom metrics` | Thêm entry trigger `["kubernetes", "hpa", "autoscal"]`: HPA tăng giảm số pod dựa trên **CPU**, **memory** hoặc **custom metrics**. |

Chỉ sửa `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` (và chép sang `production/utils/mock_llm.py` để stack Docker dùng cùng KB), không sửa `golden_dataset.jsonl`. Lệnh cuối: `python eval/run_eval.py --url http://localhost:8002 --min-accuracy 1.0 --baseline reports/eval_v2.json --out reports/eval_v2_fixed.json`, kết quả `Quality gate (accuracy ≥ 100%, không regression): ✅ PASS`, exit code 0.

**Quality gate chặn merge (Exercise 7.3):** chạy eval v1 với `--baseline reports/eval_v2.json` thì **exit code = 1** (20 câu regression, accuracy 4.3% < 80%). CI chỉ cần chạy lệnh eval như một step: exit code ≠ 0 làm job fail, branch protection "require status checks" sẽ chặn merge. File job: [`.github/workflows/eval-gate.yml`](.github/workflows/eval-gate.yml). Job bật agent v2 (tắt lỗi ngẫu nhiên `MOCK_ERROR_RATE=0` để gate không fail vì lỗi giả lập), chạy `run_eval.py --min-accuracy 1.0 --baseline reports/eval_v2.json`, và upload report.

**Nhận xét trade-off:** v2 tốn gấp ~1.9 lần token/câu (prompt dài hơn vì có context) và tổng cost gần gấp đôi, latency p95 tăng ~33 ms do thêm bước retrieval. Đổi lại accuracy tăng từ 4.3% lên 91–100%, nên **cost trên mỗi câu đúng rẻ hơn ~11 lần** (0.000408 → 0.000036 USD). So "cost / câu" thì v1 trông rẻ hơn, nhưng câu trả lời sai là chi phí bỏ đi. Vì vậy "cost / câu đúng" mới là chỉ số đúng để so hai phiên bản. (p95 = 1673.7 ms ở lần eval cuối chỉ là 1 request rơi vào tail 3% trong mẫu 23 câu: mẫu eval quá nhỏ để kết luận về latency, latency phải đo bằng load test.)

---

## 4. Observability stack (production)

Stack bật bằng `docker compose -p k5-day14-observability up -d --build`. Grafana map ra cổng 3001 thay vì 3000 vì máy đang có process khác chiếm 3000 (dùng file override cục bộ, không đổi `docker-compose.yml`).

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải: [screenshots/grafana-dashboard.png](screenshots/grafana-dashboard.png). Chụp lúc đang chạy thí nghiệm A+B: error rate 22.6%, p95 1.88 s, cả hai panel đỏ.
- [x] Trace chậm: [screenshots/jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png)
- [x] Trace lỗi (span `chat mock-llm-small` có event `exception`, `error.type=upstream_unavailable`): [screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png)
- [x] Trang Alerts của Prometheus, 3 alert cùng firing (`AgentHighErrorRate`, `AgentHighLatencyP95`, `LLMCostBurnRateHigh`): [screenshots/prometheus-alert-firing.png](screenshots/prometheus-alert-firing.png)

Unit test alert: `promtool test rules /p/alert_rules_test.yml` ra `SUCCESS`.

### 4.2 Thí nghiệm "phá hệ thống"

Locust chạy nền (30 user ở C/D/E; 30 user sau benchmark 100 user ở A+B). Thời gian tính từ lúc `docker compose up -d` áp cấu hình mới tới lúc trạng thái chuyển sang `firing` (poll Prometheus `/api/v1/alerts` mỗi 5–10 s).

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` (chạy cùng lúc với B) | `AgentHighErrorRate` (value 18.6%) | pending sau ~18 s, **firing sau ~79 s** (`for: 1m`) | Golden Signals → *Error rate* (22.6%, đỏ); *Requests/s theo status code* xuất hiện đường 503 |
| B | `MOCK_SLOW_RATE=0.3` (chạy cùng lúc với A) | `AgentHighLatencyP95` (value 1.89 s) | pending sau ~18 s, **firing sau ~141 s** (`for: 2m`) | *Latency p50/p95/p99*: p95 nhảy từ ~0.25 s lên ~1.9 s, avg tăng ít hơn nhiều; *LLM latency p95* |
| C | `LLM_MODEL=mock-llm-large` | `LLMCostBurnRateHigh` (cost/h 1.29 → 16.66 USD) | pending sau ~22 s, **firing sau ~138 s** (`for: 2m`); `AgentHighLatencyP95` lên pending (p95 ~1.1–1.3 s) nhưng chưa firing | *Cost burn rate (USD/giờ)* tăng ~13 lần; *LLM latency p95* tăng ~2.5 lần |
| D | `docker pause k5-day14-observability-agent-1` | `AgentInstanceDown` (`up` 2 → 1) | pending sau ~11 s, **firing sau ~39 s** (`for: 30s`) | *Instances up* giảm 2 → 1. Error rate của agent vẫn ~1% (xanh) dù Nginx ghi **31 lỗi 504** trong ~1 phút |
| E | `AGENT_VERSION=v1` | **Không có** | — | *Retrieval hit ratio* mất dữ liệu (không còn retrieval), *Tokens/s* giảm 2236 → 1072, bảng *Phiên bản đang chạy* đổi sang v1. p95 còn **giảm** 293 → 199 ms, error rate ~1%: mọi thứ trông "tốt hơn" |

(Lúc reset sau C, `LLMCostBurnRateHigh` vẫn firing thêm vài phút do `rate(...[5m])` còn tính cả giai đoạn dùng model lớn. Lúc này alert phản ánh độ trễ của cửa sổ 5 phút, không phải sự cố mới.)

Ảnh `prometheus-alert-firing.png` chụp lúc A+B đang chạy: `AgentHighErrorRate` và `AgentHighLatencyP95` cùng firing (cùng với `LLMCostBurnRateHigh` vì tải 100 user trước đó đẩy cost/h lên ~$4).

**Thí nghiệm E không làm alert nào firing. Phát hiện loại sự cố này trong production bằng cách nào?**

Metric hạ tầng chỉ đo *nhanh hay chậm, lỗi hay không*, không đo *đúng hay sai*. v1 trả lời sai 22/23 câu nhưng vẫn trả HTTP 200 rất nhanh. Cần thêm tín hiệu chất lượng:
- **Eval gate trong CI/CD** (như `.github/workflows/eval-gate.yml`): không cho deploy bản có accuracy giảm hoặc có regression so với baseline.
- **Online eval / canary:** định kỳ (vd mỗi 5–15 phút) chạy golden dataset vào production, xuất `agent_eval_accuracy` thành metric và alert khi giảm dưới ngưỡng.
- **Proxy metric chất lượng:** retrieval hit ratio (rơi về 0 ở E), độ dài câu trả lời, tỉ lệ câu trả lời "chung chung" (fallback), feedback 👍/👎 của người dùng, LLM-as-judge chấm mẫu ngẫu nhiên. Alert khi `agent_retrieval_total` hit ratio < 50% trong 10 phút là bắt được đúng sự cố E.

### 4.3 Debug bằng trace

Trace chậm nhất tìm được (Jaeger, `minDuration=1.5s`):

- `trace_id`: `4275a17963d30100a2937656821ca0cc`
- Tổng thời gian: 1697.3 ms (`POST /ask`, HTTP 200)
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (1671.6 ms, **98.5%**). `agent.retrieve_context` chỉ 23.6 ms (1.4%). Request chậm vì LLM upstream chậm (tail latency), không phải code của agent.
- Dòng log tương ứng (`docker compose -p k5-day14-observability logs agent | grep 4275a17963d30100a2937656821ca0cc`):

```json
{"ts": "2026-11-25T14:33:52.444625+00:00", "level": "INFO", "service": "observable-agent", "instance": "b0f96ae9b121", "event": "llm_call", "trace_id": "4275a17963d30100a2937656821ca0cc", "span_id": "676090f25f6425f8", "request_id": "1d02afb3a3ff", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 83, "output_tokens": 44, "cost_usd": 3.9e-05, "llm_ms": 1671.1}
{"ts": "2026-11-25T14:33:52.445064+00:00", "level": "INFO", "service": "observable-agent", "instance": "b0f96ae9b121", "event": "http_request", "trace_id": "4275a17963d30100a2937656821ca0cc", "span_id": "676090f25f6425f8", "request_id": "1d02afb3a3ff", "method": "POST", "route": "/ask", "status": 200, "ms": 1696.5}
```

Trace lỗi: `d3ba76764e852d01656a51c30b8624c4` (100.7 ms, span `chat mock-llm-small` status ERROR, event `exception`):

```json
{"ts": "2026-11-25T14:33:47.975717+00:00", "level": "INFO", "service": "observable-agent", "instance": "b0f96ae9b121", "event": "llm_error", "trace_id": "d3ba76764e852d01656a51c30b8624c4", "span_id": "73555bb6021b9319", "request_id": "88967f31ae77", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
{"ts": "2026-11-25T14:33:47.976108+00:00", "level": "INFO", "service": "observable-agent", "instance": "b0f96ae9b121", "event": "http_request", "trace_id": "d3ba76764e852d01656a51c30b8624c4", "span_id": "73555bb6021b9319", "request_id": "88967f31ae77", "method": "POST", "route": "/ask", "status": 503, "ms": 100.1}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

```bash
docker compose -p k5-day14-observability up -d --scale agent=N
docker compose -p k5-day14-observability run --rm --no-deps locust -f /mnt/locust/locustfile.py --host http://nginx \
  --headless -u 100 -r 10 -t 2m --csv /mnt/results/scale_N
```

Số liệu dòng `/ask` trong `benchmark-results/scale_N_stats.csv`:

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 59.32 | 200 | 350 | 1700 | 79 / 7053 (1.12%) |
| 3 | 59.73 | 200 | 260 | 1700 | 83 / 7107 (1.17%) |

(Dòng Aggregated, gồm cả `/health`: 1 instance 65.05 RPS, p95 330 ms; 3 instance 65.72 RPS, p95 250 ms.)

> **Lưu ý về lệnh:** lệnh trong CHECKPOINTS dùng `docker compose run --rm locust ...` **không có** `--no-deps`. Khi chạy như vậy, Compose khởi động lại các service phụ thuộc (`locust → nginx → agent`) theo cấu hình mặc định `deploy.replicas: 2`, nên số agent thực tế bị đưa về 2 cho cả hai lần chạy (đã kiểm tra bằng `count(up{job="agent"})` trên Prometheus). Lần đo đầu bị như vậy nên đã bỏ, rồi chạy lại với `--no-deps`. Đã kiểm tra số container agent trước và sau mỗi lần chạy: đúng 1 và 3.

**Giải thích kết quả:** RPS **không đổi** (~59.5 req/s) vì Locust là tải *closed-loop*: 100 user, mỗi user nghỉ 0.5–2 s (trung bình 1.25 s) giữa các request. Tải tối đa vì vậy ≈ 100 / (1.25 + 0.24) ≈ 67 req/s, đúng bằng mức đo được, và 1 instance đã gánh được. Thêm instance không tạo thêm nhu cầu. Scale có cải thiện **p95** (350 → 260 ms, giảm ~26%) và max (2385 → 1860 ms): với 1 instance (giới hạn 0.5 CPU), event loop bắt đầu xếp hàng lúc tải đỉnh. Chia cho 3 instance thì mỗi instance nhận ~20 req/s, gần như không phải chờ. **p50 và p99 không đổi** (200 ms và 1700 ms) vì đó là thời gian của chính LLM: p50 ≈ latency cơ bản của mock (120 ms + jitter + retrieval), p99 ≈ request rơi vào tail +1500 ms. **Nút thắt là LLM upstream** (panel *LLM latency p95* ~1.7 s khi có tail), không phải số instance của agent. Scale ngang chỉ giảm thời gian xếp hàng, không làm LLM nhanh hơn. Muốn giảm p99 phải xử lý ở tầng LLM: timeout + retry/hedging, cache câu trả lời, model nhanh hơn, streaming.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. **Average nói dối.** Với 3% request chậm, avg chỉ tăng 20% nhưng p99 tăng hơn 3 lần. SLO và alert phải theo p95/p99, và phải đo bằng load test đủ mẫu. 23 câu eval không đủ để kết luận về latency.
  2. **Hạ tầng xanh không có nghĩa là agent đúng.** Thí nghiệm E (v1) làm dashboard trông *tốt hơn* (p95 giảm, token giảm) trong khi accuracy rơi từ 100% xuống 4.3%. Chỉ eval trên golden dataset (gate trong CI + online eval) mới bắt được lỗi chất lượng. "Cost / câu đúng" là chỉ số đúng để so phiên bản.
  3. **Metric chỉ phản ánh những gì service đó thấy.** Khi pause 1 instance, Nginx trả 31 lỗi 504 nhưng error rate của agent vẫn xanh. Cần giám sát cả tầng load balancer / synthetic check từ bên ngoài. Cũng cần kiểm chứng chính công cụ đo: lệnh benchmark scale mặc định âm thầm đưa số instance về 2, nếu không đếm `up` thì báo cáo đã sai.
- Nếu đưa agent này lên production thật, sẽ thêm:
  - Metric và alert ở tầng Nginx/LB: tỉ lệ 5xx/504 và upstream latency, cộng synthetic probe từ bên ngoài gọi `/ask` mỗi phút.
  - `agent_eval_accuracy` từ online eval định kỳ, alert khi < 90%; alert retrieval hit ratio < 50% trong 10 phút.
  - Alert theo **error budget burn rate** (multi-window, vd 1h + 6h) thay vì ngưỡng cứng, để giảm alert fatigue.
  - Cost theo ngày so với budget (alert khi dự báo vượt `DAILY_BUDGET_USD`), số request bị 429/402 từ rate limiter và cost guard.
  - Saturation: CPU/memory của container, `agent_in_flight_requests` theo instance, LLM timeout/retry count.
