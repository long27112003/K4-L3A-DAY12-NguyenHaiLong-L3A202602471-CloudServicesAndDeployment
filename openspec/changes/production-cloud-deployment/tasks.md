# Tasks: Production Cloud Deployment Implementation

## 1. Checkpoint 1 — 12-Factor Configuration, Health & Logging

- [x] 1.1 Implement `Settings` in `app/config.py` declaring the 6 required fields (`port`, `agent_api_key`, `redis_url`, `rate_limit_per_minute`, `monthly_budget_usd`, `log_level`) with no default value for `agent_api_key`, and verify with `pytest tests/test_cp1.py -k test_settings`
- [x] 1.2 Implement `log_event()` in `app/logging_utils.py` to output a single-line JSON string containing `event`, lowercase `level`, ISO-8601 UTC `timestamp`, and arbitrary kwargs, and verify with `pytest tests/test_cp1.py -k test_log_event`
- [x] 1.3 Implement `/health` endpoint in `app/main.py` returning 200 OK with service metadata without external dependencies, and verify with `pytest tests/test_cp1.py`

## 2. Checkpoint 2 — Production Containerization & Docker Compose

- [x] 2.1 Update `.dockerignore` to exclude `.env`, `.git`, `.venv`, and `__pycache__` while retaining `app`, `utils`, and `requirements.txt`, and verify with `pytest tests/test_cp2.py -k test_dockerignore`
- [x] 2.2 Refactor `Dockerfile` into a multi-stage build using `python:3.11-slim`, non-root user `appuser` (UID 10001), layer caching for dependencies, `HEALTHCHECK`, and `${PORT:-8000}`, and verify with `pytest tests/test_cp2.py -k test_dockerfile`
- [x] 2.3 Update `docker-compose.yml` adding the `agent` service with `depends_on: redis`, environment variable `${AGENT_API_KEY}` interpolation, and `redis://redis:6379/0`, and verify with `pytest tests/test_cp2.py`

## 3. Checkpoint 3 — API Security: Authentication, Rate Limiting & Cost Guard

- [x] 3.1 Implement `verify_api_key()` in `app/auth.py` validating `X-API-Key` using `secrets.compare_digest` and resolving `user_id`, and verify with `pytest tests/test_cp3.py -k test_auth`
- [x] 3.2 Implement `RateLimiter` in `app/rate_limiter.py` using Redis Sorted Sets with a rolling 60s window, unique member identifiers, and 429 exceptions with `Retry-After`, and verify with `pytest tests/test_cp3.py -k test_rate_limiter`
- [x] 3.3 Implement `CostGuard` in `app/cost_guard.py` tracking monthly expenses per user in Redis with atomic `incrbyfloat` and raising 402 when exceeding budget, and verify with `pytest tests/test_cp3.py -k test_cost_guard`
- [x] 3.4 Wire security dependencies into `/ask` in `app/main.py` strictly enforcing auth -> rate limit -> budget check before invoking `ask_llm`, followed by recording cost and logging, and verify with `pytest tests/test_cp3.py`

## 4. Checkpoint 4 — Scaling & Reliability: Stateless Store & Graceful Shutdown

- [x] 4.1 Implement `ConversationStore` in `app/store.py` with `ping()`, `append()` (`rpush`, `ltrim(-20, -1)`, `expire`), and `get_history()`, and verify with `pytest tests/test_cp4.py -k test_conversation_store`
- [x] 4.2 Implement `Lifecycle` in `app/lifecycle.py` to capture `SIGTERM` and `SIGINT`, set `shutting_down = True`, and invoke previous handlers, and verify with `pytest tests/test_cp4.py -k test_lifecycle`
- [x] 4.3 Update `/health` and implement `/ready` in `app/main.py` to return 503 during shutdown and when Redis is unresponsive, and verify with `pytest tests/test_cp4.py`

## 5. Checkpoint 5 — Cloud Deployment & Documentation

- [ ] 5.1 Fill out `DEPLOYMENT.md` with student metadata, public URL or local fallback configuration, environment variable names, and curl verification logs, and verify with `pytest tests/test_cp5.py`
- [ ] 5.2 Provide comprehensive answers for all 10 reflection prompts in `exercises.md`, and verify question completion count with `python grade.py`

## 6. Bonus CI/CD & Final Verification

- [ ] 6.1 Create `.github/workflows/ci.yml` defining `test`, `build`, and `deploy` jobs with push/PR triggers, dummy CI secrets, and smoke tests, and verify with `pytest tests/test_bonus_cicd.py`
- [ ] 6.2 Execute full test suite `pytest tests/ -v` and run `python grade.py` to confirm target score (>= 100/100)
