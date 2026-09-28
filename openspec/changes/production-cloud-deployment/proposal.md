# Proposal: Production Cloud Deployment for AI Agent Service

## Why

Moving an AI agent service from a local development environment (`localhost:8000`) to a public cloud infrastructure requires solving critical production challenges: security, reliability, predictability, and cost control. Without 12-Factor external configuration, a hardcoded secret causes catastrophic leaks; without structured JSON logging, cloud log sinks cannot index or alert on events; without multi-stage Docker builds and non-root users, deployments are slow and vulnerable; without API authentication, sliding-window rate limiting, and monthly cost guards, external bots can exhaust LLM budgets within minutes; and without stateless Redis architecture and graceful shutdown handlers, scaling instances and zero-downtime updates are impossible.

This proposal defines the complete transition of the FastAPI-based AI Agent into a secure, production-grade cloud service compliant with all 5 lab checkpoints and rubric requirements.

## What Changes

- **12-Factor Configuration & Logging (CP1)**:
  - Configure `Settings` with 6 explicit environment fields; enforce fail-fast behavior with no default value for `agent_api_key`.
  - Implement single-line structured JSON logging (`log_event`) with ISO UTC timestamps and lowercase levels.
  - Implement `/health` liveness probe endpoint that returns 200 OK without any external dependencies.
- **Production Containerization (CP2)**:
  - Refactor `Dockerfile` into a multi-stage build using `python:3.11-slim`, non-root user (`appuser`), layer caching order, container `HEALTHCHECK`, and dynamic `${PORT:-8000}`.
  - Update `.dockerignore` to exclude `.env`, `.git`, `__pycache__`, `.venv`.
  - Update `docker-compose.yml` to define the `agent` service interconnected with `redis:7-alpine` via service name networking and environment variable interpolation.
- **Three-Tier API Security (CP3)**:
  - Implement `verify_api_key` dependency using `secrets.compare_digest` to prevent timing attacks, returning 401 on unauthorized calls.
  - Implement sliding-window rate limiting using Redis Sorted Sets (ZSET) over 60s windows, returning 429 on quota breaches.
  - Implement monthly cost guard tracking per-user expenses in Redis, returning 402 on budget exhaustion.
  - Integrate security checks in `/ask` strictly before invoking mock LLM.
- **Stateless Scaling & High Availability (CP4)**:
  - Externalize chat history to Redis List with `HISTORY_MAX_MESSAGES` trim and TTL expiration.
  - Implement `/ready` readiness probe inspecting Redis health and graceful shutdown status.
  - Implement graceful shutdown handler capturing `SIGTERM`/`SIGINT`, switching status to 503, and forwarding signals to Uvicorn's original handler.
- **Cloud Deployment & Documentation (CP5 & Exercises)**:
  - Verify and document deployment to cloud (Railway / Render) or configured local fallback.
  - Provide complete technical responses for the 10 reflection prompts in `exercises.md`.
- **Bonus CI/CD Pipeline (Optional Bonus)**:
  - Author `.github/workflows/ci.yml` running automated tests, Docker builds, and deployment verification.

## Capabilities

### New Capabilities
- `agent-service`: Core cloud-ready AI agent service encompassing 12-factor configuration, structured logging, multi-stage Docker packaging, multi-layer security (auth, rate limiting, budget control), stateless Redis storage, and lifecycle management.

### Modified Capabilities
*(None - no prior OpenSpec specifications exist)*

## Impact

- **Codebase**: Updates to `app/config.py`, `app/logging_utils.py`, `app/main.py`, `app/auth.py`, `app/rate_limiter.py`, `app/cost_guard.py`, `app/store.py`, `app/lifecycle.py`, `Dockerfile`, `.dockerignore`, `docker-compose.yml`, `DEPLOYMENT.md`, and `exercises.md`.
- **External Dependencies**: Requires Redis (local container or managed cloud Redis) and standard Python dependencies in `requirements.txt`.
- **Breaking Changes**: None. All changes align with the predefined test suite in `tests/test_cp1.py` through `tests/test_cp5.py`.
