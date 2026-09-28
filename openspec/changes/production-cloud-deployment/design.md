# Design: Production Cloud Deployment Architecture

## Context

The repository provides skeleton modules in `app/` with `NotImplementedError` stubs and a comprehensive pytest suite in `tests/`. See `proposal.md` for overall motivation. Constraints include:
- Must run in Python 3.11 with FastAPI and Pydantic v2 / Pydantic Settings.
- Must support Redis both via a real Redis server (`redis://...`) and an in-memory mock (`fake://...` via `fakeredis`) during test runs.
- Container image must stay under 500MB and run as an unprivileged user.
- All secrets must be externalized; no hardcoded credentials anywhere in the repository.

## Goals / Non-Goals

**Goals:**
- Provide a robust implementation of all 5 checkpoints (`CP1` through `CP5`) achieving 100/100 points on `python grade.py`.
- Architect the system to be strictly stateless and resilient to container restarts and scaling.
- Implement multi-layered protection (Auth -> Rate Limiter -> Cost Guard) before any LLM inference invocation.
- Support both live cloud deployment and local fallback modes.

**Non-Goals:**
- Implementing real external LLM API billing (the project utilizes `utils/mock_llm.py`).
- Creating distributed multi-region databases (single Redis instance is standard for this scope).
- Mandating Kubernetes orchestration (Docker Compose and cloud container platforms are targeted).

## Decisions

### 1. 12-Factor Configuration with Strict Secret Validation
- **Decision**: Define `Settings(BaseSettings)` in `app/config.py` with 6 explicit attributes. `agent_api_key: str` has no default value.
- **Rationale**: Any omitted secret raises `pydantic.ValidationError` upon instantiation, preventing the container from booting in an insecure state.
- **Alternatives Considered**: Supplying a fallback string like `"changeme"` or `""` was rejected because it creates false security and exposes endpoints if environment variables are omitted on cloud platforms.

### 2. Single-line Structured JSON Logging
- **Decision**: In `app/logging_utils.py`, `log_event()` constructs a dictionary with `event`, `level.lower()`, `timestamp` (ISO UTC), and extra keyword arguments, serializing it with `json.dumps(..., ensure_ascii=False)` on a single line.
- **Rationale**: Cloud log ingestion platforms (AWS CloudWatch, Datadog, Render) aggregate logs line by line; multi-line JSON breaks into detached log fragments.
- **Alternatives Considered**: Standard Python `logging` module with a custom formatter was considered, but direct dictionary serialization avoids module-level configuration collision and meets exact pytest stdout assertion criteria.

### 3. Isolation of Liveness (`/health`) from Readiness (`/ready`)
- **Decision**:
  - `/health` inspects only process state (`lifecycle.shutting_down`). It takes zero dependency injections and performs no I/O.
  - `/ready` injects `ConversationStore` to verify `store.ping()` and checks `lifecycle.shutting_down`.
- **Rationale**: If `/health` queried Redis, a momentary Redis network glitch would trigger container orchestrators to kill and restart all agent containers simultaneously, causing cascading downtime.
- **Alternatives Considered**: A single health check was rejected because orchestrator liveness checks serve a completely different purpose than load balancer traffic routing.

### 4. Multi-Stage Dockerfile with Non-Root User
- **Decision**:
  - Stage 1 (`builder`): Installs dependencies to `--prefix=/install` using `python:3.11-slim`.
  - Stage 2 (`runtime`): Copies `/install` packages to `/usr/local`, creates user `appuser` (UID 10001), switches via `USER appuser`, adds `HEALTHCHECK`, and executes Uvicorn with `--port ${PORT:-8000}`.
- **Rationale**: Keeps runtime image size under 250MB (well below the 500MB limit) and eliminates root privileges to prevent container breakout exploits.
- **Alternatives Considered**: Alpine Linux was evaluated, but Debian slim avoids C-extension compilation delays for Python wheels.

### 5. API Key Authentication with Timing-Attack Defense
- **Decision**: In `app/auth.py`, `verify_api_key` extracts `X-API-Key` and compares it against `get_settings().agent_api_key` using `secrets.compare_digest`.
- **Rationale**: Standard `==` operator exhibits variable execution time depending on matching prefix length, enabling timing attacks. `secrets.compare_digest` runs in constant time.
- **Alternatives Considered**: Passing API key in query parameters was rejected due to URL logging security vulnerabilities.

### 6. Sliding-Window Rate Limiting via Redis Sorted Sets
- **Decision**: In `app/rate_limiter.py`, store request timestamps in a Redis Sorted Set (`ratelimit:<user_id>`).
  1. Purge entries with score `< now - 60` using `zremrangebyscore`.
  2. Count remaining members using `zcard`.
  3. If count `>= limit`, raise HTTP 429 with `Retry-After: 60`.
  4. If allowed, record entry with member `f"{now}:{uuid4().hex}"` and score `now`, and set key TTL to 60s.
- **Rationale**: Eliminates the boundary burst exploit where fixed-window limiters permit 2x traffic across minute marks.
- **Alternatives Considered**: Fixed-window counter was rejected due to vulnerability to burst attacks.

### 7. Monthly Cost Guard with Atomic Increment
- **Decision**: In `app/cost_guard.py`, key format is `cost:<user_id>:<YYYY-MM>`. `spent()` retrieves current value. `check()` validates `spent + estimated <= budget` (raising 402 if exceeded). `record()` invokes `incrbyfloat` with 40-day TTL.
- **Rationale**: Prevents users from consuming excessive LLM tokens within allowed request limits.
- **Alternatives Considered**: Storing spend in an in-memory dictionary was rejected because multi-instance scaling would cause split-brain budget tracking.

### 8. Redis List Conversation Store with Sliding Trim
- **Decision**: In `app/store.py`, `append()` pushes serialized turns to `history:<user_id>` via `rpush`, executes `ltrim(key, -20, -1)` to keep the 20 most recent messages, and refreshes 7-day TTL.
- **Rationale**: Enables horizontally scaled agents behind a load balancer to access identical conversation context regardless of which container handles the request.
- **Alternatives Considered**: In-process dict in `app/main.py` violates statelessness and loses state across deploys.

### 9. Signal Interception & Delegation for Graceful Shutdown
- **Decision**: `Lifecycle` captures `SIGTERM` and `SIGINT`, stores previous handlers, sets `shutting_down = True`, and invokes the previous handler.
- **Rationale**: Uvicorn registers its own signal handler to stop the asyncio event loop. If custom handlers overwrite Uvicorn's handler without delegating back, the process ignores shutdown signals and gets killed abruptly by SIGKILL.
- **Alternatives Considered**: Relying purely on Uvicorn's default handler without intercepting would prevent `/health` from returning 503 early to inform the load balancer.

## Risks / Trade-offs

- **[Risk] Redis connectivity failure during operation** → Mitigation: `store.ping()` catches all exceptions and returns `False`; `/ready` immediately returns 503 so load balancers stop sending traffic, while `/health` stays 200 so containers are not needlessly restarted.
- **[Risk] Cloud platform assigns non-standard port** → Mitigation: Docker `CMD` uses shell form `"uvicorn ... --port ${PORT:-8000}"` to honor runtime `$PORT`.
- **[Risk] Secret leakage in Git history or Docker images** → Mitigation: `.dockerignore` excludes `.env`, `docker-compose.yml` uses `${AGENT_API_KEY}` interpolation, and `.env` is checked before commit.

## Migration Plan

1. Implement CP1 (`config.py`, `logging_utils.py`, `/health` in `main.py`). Validate with `pytest tests/test_cp1.py`.
2. Implement CP2 (`Dockerfile`, `.dockerignore`, `docker-compose.yml`). Validate with `pytest tests/test_cp2.py`.
3. Implement CP3 (`auth.py`, `rate_limiter.py`, `cost_guard.py`, `/ask` integration). Validate with `pytest tests/test_cp3.py`.
4. Implement CP4 (`store.py`, `lifecycle.py`, `/ready`, `/health` 503). Validate with `pytest tests/test_cp4.py`.
5. Complete documentation in `DEPLOYMENT.md` and complete answers in `exercises.md`.
6. Run `python grade.py` to confirm 100/100 score.
