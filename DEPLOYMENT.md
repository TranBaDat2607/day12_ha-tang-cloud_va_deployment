# Deployment Information

> **Student Name:** Tran Ba Dat **
> **Student ID:** 2A202600778 **

## Public URL

```
https://day12-vinuni-lab-production.up.railway.app
```

## Platform

- [x] Railway
- [ ] Render
- [ ] Cloud Run

Railway project: `day12-vinuni-lab` · service `production` · deployed 2026-06-12.

## Live verification (deployed URL — all passing)

Tested against the public Railway URL:

| Check | Result |
|-------|--------|
| `GET /health` | ✅ 200 `{"status":"ok","environment":"production","checks":{"llm":"mock"}}` |
| `GET /` | ✅ 200 (info/endpoints) |
| `POST /ask` without API key | ✅ 401 Unauthorized |
| `POST /ask` with `X-API-Key: dat19283746` | ✅ 200 + answer from mock LLM |

## Local verification (also passing)

The complete production agent (`06-lab-complete/`) was built, run, and verified locally with Docker Compose:

```bash
cd 06-lab-complete
docker compose up --build -d
```

| Check | Result |
|-------|--------|
| `GET /health` | ✅ 200 `{"status":"ok",...}` |
| `POST /ask` without API key | ✅ 401 Unauthorized |
| `POST /ask` with `X-API-Key` | ✅ 200 + answer |
| Rate limit (20 req/min) | ✅ 429 after limit |
| `check_production_ready.py` | ✅ 20/20 (100%) |
| Image size | ✅ 247 MB (< 500 MB) |

## How to deploy (run these yourself — needs your account)

### Option A: Railway (fastest)

```bash
cd 06-lab-complete
npm i -g @railway/cli
railway login                       # opens browser — your account
railway init
railway variables set AGENT_API_KEY=<pick-a-strong-secret>
railway variables set ENVIRONMENT=production
railway up
railway domain                      # ← copy this URL into "Public URL" above
```

### Option B: Render (Blueprint)

1. Push this repo to your GitHub
2. Render Dashboard → New → Blueprint → connect the repo
3. Render reads `06-lab-complete/render.yaml`
4. Set `AGENT_API_KEY` in the dashboard (or let Render generate it)
5. Deploy → copy the URL into "Public URL" above

## Test commands (after deploy — replace URL and KEY)

### Health check
```bash
curl https://YOUR-APP-URL/health
# Expected: {"status": "ok", ...}
```

### Auth required
```bash
curl -X POST https://YOUR-APP-URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: 401
```

### API test with authentication
```bash
curl -X POST https://YOUR-APP-URL/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: 200 with an answer
```

### Rate limiting
```bash
for i in {1..25}; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST https://YOUR-APP-URL/ask \
    -H "X-API-Key: YOUR_KEY" -H "Content-Type: application/json" \
    -d '{"question":"test"}'
done
# Expected: 200s then 429s after request 20
```

## Environment Variables Set

- `PORT` — injected by platform
- `ENVIRONMENT=production`
- `AGENT_API_KEY` — *(set your own; never commit it)*
- `REDIS_URL` — *(if using a managed Redis add-on)*
- `LOG_LEVEL=INFO`

## Screenshots

*(add after deploying)*

- [ ] `screenshots/dashboard.png` — deployment dashboard
- [ ] `screenshots/running.png` — service running
- [ ] `screenshots/test.png` — test results
