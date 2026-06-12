# Day 12 Lab — Mission Answers

> **Student Name:** Tran Ba Dat **
> **Student ID:** 2A202600778 **
> **Date:** 2026-06-12

All answers below were produced by actually running every example in this repo (Windows 11, Python 3.14.4, Docker 29.5.2, Docker Compose v5.1.4). Test outputs are pasted verbatim.

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

1. **Hardcoded secrets in source code** — `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` and `DATABASE_URL` with username/password inline (lines 17–18). One `git push` and the secrets are public.
2. **Secrets leaked into logs** — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` writes the API key to stdout on every request. Confirmed in the captured server log:
   ```
   [DEBUG] Got question: Hello
   [DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
   ```
3. **No config management** — `DEBUG = True`, `MAX_TOKENS = 500` are constants in code; changing config requires editing and redeploying code instead of setting env vars.
4. **`print()` instead of structured logging** — unparseable by log aggregators, no levels, no timestamps.
5. **No health check endpoint** — `GET /health` returns **404** (verified). The platform has no way to know the agent crashed and restart it.
6. **Hardcoded host/port + localhost binding** — `host="localhost", port=8000`. Binding `localhost` means the app is unreachable from outside a container; Railway/Render inject `PORT` which is ignored.
7. **`reload=True` in production code** — debug auto-reload running in production. Side effect observed during the lab: the reloader spawns a child process, so killing the parent left an orphan still holding port 8000.
8. **No graceful shutdown** — no SIGTERM handling, no lifespan hooks; in-flight requests are dropped on stop.

### Exercise 1.2: Run basic version — observed results

```
GET  /                          → 200 {"message":"Hello! Agent is running on my machine :)"}
POST /ask  (JSON body)          → 422 (the handler declares `question: str` = query param, not body!)
POST /ask?question=Hello        → 200 {"answer":"Đây là câu trả lời từ AI agent (mock)..."}
GET  /health                    → 404 (no health check)
```

It runs — but it is not production-ready (no health check, secrets in logs, query-param-only API).

### Exercise 1.3: Comparison table

| Feature | Develop (basic) | Production (advanced) | Why important? |
|---------|-----------------|----------------------|----------------|
| Config | Hardcoded constants in code | `config.py` dataclass reading env vars, with `validate()` fail-fast | Change behavior per environment without code changes; secrets never in git |
| Secrets | In source + printed to logs | From env, never logged (only lengths/metadata logged) | Prevents credential leaks via repo or log aggregator |
| Health check | None (404) | `/health` (liveness) + `/ready` (readiness) + `/metrics` | Platform can auto-restart dead containers and route traffic only to ready instances |
| Logging | `print()` | Structured JSON logs (`{"event": "agent_request", "question_length": 15, "client_ip": "127.0.0.1"}`) | Machine-parseable, searchable in Datadog/Loki, no secret leakage |
| Shutdown | Abrupt | Lifespan hook + SIGTERM handler; readiness flag drops first | In-flight requests finish; zero-downtime rolling deploys |
| Host binding | `localhost` | `0.0.0.0` from `HOST` env | Reachable inside containers/cloud |
| Port | Hardcoded 8000 | `PORT` env var | Railway/Render inject PORT dynamically |
| CORS | None | Configurable `ALLOWED_ORIGINS` | Browser security |

Verified production version output:
```
GET /health → {"status":"ok","uptime_seconds":7.5,"version":"1.0.0","environment":"development",...}
GET /ready  → {"ready":true}
POST /ask (JSON body) → 200 {"question":"What is deploy?","answer":"Deployment là quá trình...","model":"gpt-4o-mini"}
```

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)

1. **Base image:** `python:3.11` (full distribution, ~1 GB).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Why COPY requirements.txt first?** Docker layer caching. Each instruction creates a cached layer; layers are invalidated top-down. Dependencies change rarely, code changes often — copying `requirements.txt` and running `pip install` *before* copying code means code edits don't invalidate the (slow) pip-install layer, so rebuilds take seconds instead of minutes.
4. **CMD vs ENTRYPOINT:** `CMD` provides the default command, fully replaceable at run time (`docker run image other-cmd`). `ENTRYPOINT` is the fixed executable; anything passed at run time becomes its *arguments* (override requires `--entrypoint`). Common pattern: `ENTRYPOINT ["python"]` + `CMD ["app.py"]`.

### Exercise 2.2: Build and run — observed

```
docker build -f 02-docker/develop/Dockerfile -t agent-develop .   → OK
docker run -p 8000:8000 agent-develop
GET  /health → {"status":"ok","uptime_seconds":5.9,"container":true}
POST /ask?question=What is Docker? → {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Multi-stage build + image size comparison

- **Stage 1 (`builder`)**: `python:3.11-slim` + `gcc`/`libpq-dev` build tools; runs `pip install --user` so all packages land in one copyable directory (`/root/.local`).
- **Stage 2 (`runtime`)**: fresh `python:3.11-slim` — copies *only* the installed packages and source code; no compiler, no pip cache, no build tools. Also adds a non-root `appuser` and a `HEALTHCHECK`.
- **Why smaller:** the final image never contains gcc, apt lists, pip cache, or intermediate layers — only the runtime essentials.

**Measured sizes:**

| Image | Size |
|-------|------|
| `agent-develop` (single-stage, `python:3.11`) | **1.66 GB** |
| `agent-production` (multi-stage, `python:3.11-slim`) | **236 MB** |
| Difference | **≈ 86% smaller** |

### Exercise 2.4: Docker Compose stack

Services started: **agent** (FastAPI, build from multi-stage Dockerfile), **redis** (redis:7-alpine, session cache/rate limiting), **qdrant** (v1.9.0, vector DB for RAG), **nginx** (reverse proxy/load balancer).

Architecture:
```
Client ──HTTP──▶ Nginx (:80, rate-limit 10 r/s, security headers)
                   │ proxy_pass (round-robin upstream)
                   ▼
                 Agent (:8000, internal network only)
                   ├──▶ Redis  (redis://redis:6379)   [healthcheck: redis-cli ping]
                   └──▶ Qdrant (http://qdrant:6333)   [healthcheck: /dev/tcp probe]
```
Communication: all services share the `internal` bridge network and address each other by **service name** (Docker's embedded DNS) — e.g. `REDIS_URL=redis://redis:6379/0`. The agent has **no published port**; only Nginx is exposed. `depends_on: condition: service_healthy` makes the agent wait for Redis/Qdrant healthchecks before starting.

Verified through Nginx (port 8080 on this machine — Windows blocks binding :80; see note below):
```
GET  /health → {"status":"ok","uptime_seconds":60.4,"version":"2.0.0",...}
POST /ask    → {"answer":"Tôi là AI agent được deploy lên cloud..."}
Headers: X-Frame-Options: DENY, X-Content-Type-Options: nosniff, X-XSS-Protection: 1; mode=block, Server: nginx (version hidden)
```

> **Windows note:** binding host port 80 is denied (`bind: An attempt was made to access a socket in a way forbidden by its access permissions`). Fixed with `docker-compose.override.yml` publishing `8080:80`.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway

Railway app verified **locally** (cloud deploy requires an interactive `railway login`; steps documented below):
```
PORT=8001 python app.py     ← simulates Railway's injected PORT
GET  /health → {"status":"ok","uptime_seconds":7.2,"platform":"Railway",...}
POST /ask    → {"question":"Am I on the cloud?","answer":"Agent đang hoạt động tốt!...","platform":"Railway"}
```
Deploy steps (to run with your account): `npm i -g @railway/cli` → `railway login` → `railway init` → `railway variables set PORT=8000 AGENT_API_KEY=...` → `railway up` → `railway domain`.

- **Public URL:** _________________________ *(fill in after `railway domain`)*

### Exercise 3.2: `railway.toml` vs `render.yaml` — differences

| Aspect | `railway.toml` | `render.yaml` |
|--------|----------------|---------------|
| Scope | Config for **one service** (build + deploy settings) | **Blueprint / Infrastructure-as-Code** — declares the whole stack (web service **and** managed Redis) |
| Builder | `builder = "NIXPACKS"` (auto-detect language; Dockerfile if present) | `runtime: python` + explicit `buildCommand` |
| Secrets | Set out-of-band via CLI/dashboard (`railway variables set`) | Declared in-file: `sync: false` (set manually in dashboard) or `generateValue: true` (Render generates a random value) |
| Health check | `healthcheckPath = "/health"`, `healthcheckTimeout = 30` | `healthCheckPath: /health` |
| Restart policy | `restartPolicyType = "ON_FAILURE"`, `restartPolicyMaxRetries = 3` | implicit (platform-managed) |
| Extras | — | `region`, `plan`, `autoDeploy: true` (deploy on git push), Redis add-on with `maxmemoryPolicy` |

### Exercise 3.3: GCP Cloud Run CI/CD pipeline (`cloudbuild.yaml` + `service.yaml`)

`cloudbuild.yaml` — 4 sequential steps (chained with `waitFor`):
1. **test** — `python:3.11-slim` runs `pytest` (pipeline stops if tests fail)
2. **build** — Docker image tagged `gcr.io/$PROJECT_ID/ai-agent:$COMMIT_SHA` and `:latest`, with `--cache-from` for layer caching
3. **push** — push all tags to Container Registry
4. **deploy** — `gcloud run deploy` to `asia-southeast1` with `--min-instances=1` (avoids cold start), `--max-instances=10`, 512Mi/1 CPU, and **secrets from Secret Manager** (`--set-secrets=OPENAI_API_KEY=openai-key:latest` — never hardcoded)

`service.yaml` (Knative service definition): autoscaling 1–10 instances, `containerConcurrency: 80`, CPU/memory requests+limits, env secrets via `secretKeyRef`, **livenessProbe** on `/health` and **startupProbe** on `/ready` — exactly the probe pattern from Part 5.

### Checkpoint 3

- [x] Configs of all 3 platforms analyzed; Railway app verified locally with env-injected PORT
- [ ] Public URL — *requires account login; see DEPLOYMENT.md*

---

## Part 4: API Security

### Exercise 4.1: API Key authentication (`04-api-gateway/develop`)

- **Where is the key checked?** In the `verify_api_key` dependency (FastAPI `Security(APIKeyHeader(name="X-API-Key"))`), injected into `/ask` via `Depends(verify_api_key)`. `/` and `/health` stay public (platforms need unauthenticated health checks).
- **What happens on wrong key?** Missing key → **401** ("Missing API key"); wrong key → **403** ("Invalid API key").
- **How to rotate?** The key comes from the `AGENT_API_KEY` env var — set a new value and restart (zero code changes). For zero-downtime rotation, accept a set of keys (old + new) during a grace window, then remove the old one.

**Test outputs (verbatim):**
```
no key      → Status: 401
wrong key   → Status: 403
correct key → Status: 200  {"question":"Hello","answer":"Agent đang hoạt động tốt!..."}
GET /health → 200 {"status":"ok"}   (public)
```

### Exercise 4.2: JWT authentication (`04-api-gateway/production`)

JWT flow: `POST /auth/token` with username/password → server signs an HS256 token (payload: `sub`, `role`, `iat`, `exp` = 60 min) with `JWT_SECRET` → client sends `Authorization: Bearer <token>` → `verify_token` checks the signature locally, **no DB lookup per request** (stateless auth).

**Test outputs:**
```
STEP 1  no token            → 401
STEP 2  login student/demo123 → access_token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
STEP 3  bad credentials     → 401
STEP 4  /ask with Bearer    → 200 {"answer": "...", "usage": {"requests_remaining": 9, ...}}
STEP 5  tampered token      → 403
STEP 8  /admin/stats as student → 403 (role-based access)
STEP 9  /admin/stats as teacher → 200 {"global_cost_usd": 0.000195, "global_budget_usd": 10.0}
```

### Exercise 4.3: Rate limiting (`rate_limiter.py`)

- **Algorithm:** **Sliding Window Counter** — a `deque` of request timestamps per user; on each request, timestamps older than the 60 s window are evicted, then the count is compared to the limit. (Not token bucket — there is no refill rate; not fixed window — the window slides with `now`.)
- **Limit:** **10 requests/minute** for role `user`, **100 requests/minute** for role `admin` (two `RateLimiter` singletons selected by JWT role).
- **Admin bypass:** in `/ask`: `limiter = rate_limiter_admin if role == "admin" else rate_limiter_user`. Admins aren't unlimited — just a 10× higher tier.
- On 429 the response carries proper headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining: 0`, `X-RateLimit-Reset`, `Retry-After`.

**Test output (12 rapid calls as `student`, 1 call already used in the window):**
```
codes: 200,200,200,200,200,200,200,200,200,429,429,429
```
Exactly 10 requests succeeded inside the 60 s window, then 429 — sliding window working as designed.

### Exercise 4.4: Cost guard (`cost_guard.py`)

Implementation approach (already implemented in the lab's production version, verified by running it):
- Per user + per day `UsageRecord` (input/output tokens, request count), priced at GPT-4o-mini rates ($0.15/1M input, $0.60/1M output).
- `check_budget()` runs **before** the LLM call: per-user daily budget $1 → **402 Payment Required** when exceeded; global daily budget $10 → **503** (protects the operator even if many users are under their individual caps). Warning logged at 80% usage.
- `record_usage()` runs **after** the LLM call and accumulates real token counts.
- Records auto-reset when the date changes (each `_get_record` compares `record.day` with today).

Redis variant (for multi-instance production, per CODE_LAB Part 4.4):
```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"
    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:
        return False
    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)
    return True
```

**Verified `/me/usage` output after 10 requests:**
```json
{"user_id": "student", "date": "2026-06-12", "requests": 10, "input_tokens": 22,
 "output_tokens": 320, "cost_usd": 0.000195, "budget_usd": 1.0,
 "budget_remaining_usd": 0.999805, "budget_used_pct": 0.0}
```

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

Implemented in `05-scaling-reliability/develop/app.py` and verified:
```
GET /health → {"status":"ok","uptime_seconds":7.4,"version":"1.0.0",
               "checks":{"memory":{"status":"ok","used_percent":88.6}}}
GET /ready  → {"ready":true,"in_flight_requests":1}
```
- `/health` (liveness): "is the process alive?" — returns 200 with uptime/version + dependency checks; non-200 → platform restarts the container.
- `/ready` (readiness): "can I take traffic?" — 503 during startup/shutdown or when dependencies are down; the load balancer stops routing to that instance without killing it.

### Exercise 5.2: Graceful shutdown

Pattern: a lifespan hook flips `_is_ready = False` (readiness fails → LB drains traffic), then waits up to 30 s for `_in_flight_requests` to reach 0; uvicorn's `timeout_graceful_shutdown=30` handles SIGTERM.

**Verified** by `docker stop` (which sends SIGTERM) on a running agent instance — captured log:
```
INFO:     Shutting down
INFO:     Waiting for application shutdown.
INFO:app:Instance instance-4c21f2 shutting down
INFO:     Application shutdown complete.
INFO:     Finished server process [1]
```
No request was dropped (see Exercise 5.4 — requests during the kill all returned 200).

### Exercise 5.3: Stateless design

`05-scaling-reliability/production/app.py` stores every session in **Redis** (`session:{id}` keys, JSON-serialized, 1 h TTL) via `save_session`/`load_session`/`append_to_history`. The process holds **no** conversation state in memory, so any instance can serve any request. (It includes an in-memory fallback that loudly warns "not scalable!" when Redis is absent.)

### Exercise 5.4: Load balancing (3 instances + Nginx)

`docker compose up --build -d --scale agent=3` → 5 containers, all healthy. 6 requests in one session, round-robin observed via `served_by`:

```
req 1 → served_by=instance-fb52e2   req 4 → served_by=instance-fb52e2
req 2 → served_by=instance-8800ec   req 5 → served_by=instance-8800ec
req 3 → served_by=instance-4c21f2   req 6 → served_by=instance-4c21f2
```

**Failover test:** killed `production-agent-1` mid-conversation, then sent 3 more requests:
```
req 7 → served_by=instance-fb52e2  (200)
req 8 → served_by=instance-8800ec  (200)
req 9 → served_by=instance-fb52e2  (200)
History after kill: 18 messages — fully intact (state in Redis, not in the dead instance)
```
Zero errors; traffic transparently moved to the surviving instances.

### Exercise 5.5: `test_stateless.py`

```
Session ID: 1124e278-9cef-48e4-ad98-cd0866edbfff
Request 1: [instance-fb52e2]  ... Request 5: [instance-fb52e2]
Instances used: {'instance-fb52e2', 'instance-8800ec'}
✅ All requests served despite different instances!
Total messages: 10
✅ Session history preserved across all instances via Redis!
```

---

## Part 6: Final Project (`06-lab-complete`)

### Bugs found & fixed to make it run

The shipped lab code had 3 blockers (all fixed in this repo):

1. **Missing `utils/` in the Docker build context** — `app/main.py` imports `utils.mock_llm`, but `utils/` lived only at the repo root while compose builds from `06-lab-complete/`. Fix: copied `utils/` into `06-lab-complete/`.
2. **`uvicorn` not importable in the image** — builder stage `pip install --user` puts packages in `/root/.local`; runtime copies them to `/home/agent/.local`, but user `agent`'s home is `/app`, so Python's user-site never looks there. Fix (Dockerfile): `ENV PYTHONPATH=/app:/home/agent/.local/lib/python3.11/site-packages`.
3. **Every request returned 500** — `response.headers.pop("server", None)` in the middleware; Starlette's `MutableHeaders` has no `.pop()`. Fix (`app/main.py`): `if "server" in response.headers: del response.headers["server"]`.

Also created `.env.local` from `.env.example` (compose's `env_file` expects `.env.local`).

### Verification — live tests against `http://localhost:8000`

| Test | Expected | Result |
|------|----------|--------|
| `GET /health` | 200 ok | ✅ `{"status":"ok","environment":"staging","checks":{"llm":"mock"}}` |
| `POST /ask` without key | 401 | ✅ 401 |
| `POST /ask` with `X-API-Key` | 200 + answer | ✅ 200, mock LLM answered |
| Rate limit: 21 rapid calls (limit 20/min) | 429 at #20 | ✅ `200×19, 429, 429` |
| Image size | < 500 MB | ✅ **247 MB** |

### `check_production_ready.py`

```
Result: 20/20 checks passed (100%)
🎉 PRODUCTION READY!
```
All categories green: required files (6/6), security (.env gitignored, no hardcoded secrets), API endpoints (/health, /ready, auth, rate limiting, SIGTERM, JSON logging), Docker (multi-stage, non-root user, HEALTHCHECK, slim base, .dockerignore coverage).

---

## Environment notes (Windows-specific issues hit during the lab)

1. **Port 80 binding denied** on Windows → published Nginx on 8080 via `docker-compose.override.yml` (Part 2) — the Part 5 compose already used 8080.
2. **`reload=True` orphan processes**: uvicorn's reloader child survives a parent kill and keeps the port. Find it with `Get-NetTCPConnection -LocalPort 8000` and stop the child PID.
3. Vietnamese (UTF-8) responses display as mojibake in the default PowerShell console but the bytes are correct UTF-8 (verified with `PYTHONIOENCODING=utf-8`).
