# Spec Delta

## Purpose

Provides a secure, observable, scalable, and cost-controlled AI agent service deployed to cloud environments adhering to 12-Factor principles.

## ADDED Requirements

### Requirement: 12-Factor Environment Configuration
The service SHALL externalize all operational settings through environment variables with typed validation and SHALL fail fast during startup if essential secrets are missing.

#### Scenario: Missing secret causes startup failure
- **WHEN** the service starts without `AGENT_API_KEY` defined in the environment
- **THEN** configuration loading SHALL raise a validation error and terminate immediately

#### Scenario: Non-secret variables use sensible defaults
- **WHEN** non-secret variables (`PORT`, `REDIS_URL`, `RATE_LIMIT_PER_MINUTE`, `MONTHLY_BUDGET_USD`, `LOG_LEVEL`) are omitted
- **THEN** default values (8000, "redis://localhost:6379/0", 10, 10.0, "INFO") SHALL be populated without error

### Requirement: Structured Single-line JSON Logging
The service SHALL output log records to standard output formatted as single-line JSON objects with standard metadata.

#### Scenario: Log event emitted as single-line JSON
- **WHEN** an event is logged via `log_event("ask_completed", user_id="user1", cost_usd=0.05)`
- **THEN** a single JSON line SHALL be written to standard output containing `event`, lowercase `level`, ISO-8601 UTC `timestamp`, and provided keyword arguments

### Requirement: Isolated Liveness Health Probe
The service SHALL provide a `/health` HTTP endpoint that indicates whether the application process is running without querying any external dependencies.

#### Scenario: Normal liveness check
- **WHEN** a client performs a GET request to `/health` while the service is operating normally
- **THEN** the service SHALL respond with HTTP 200 and `{"status": "ok", "service": "day12-agent", "version": "1.0.0"}` without invoking Redis or databases

#### Scenario: Liveness check during shutdown
- **WHEN** a client performs a GET request to `/health` after a shutdown signal has been received
- **THEN** the service SHALL respond with HTTP 503 and `{"status": "shutting_down"}`

### Requirement: Production Multi-stage Container Packaging
The container image SHALL be built using a multi-stage process with a minimal base image, non-root execution, dynamic port binding, and layer caching optimization.

#### Scenario: Container execution security
- **WHEN** the container image is executed
- **THEN** it SHALL run under an unprivileged user (`appuser`) instead of root

#### Scenario: Dynamic port assignment
- **WHEN** the container is launched with an arbitrary `$PORT` environment variable
- **THEN** the web server SHALL bind to `0.0.0.0` on that designated port

### Requirement: API Key Authentication with Timing Attack Mitigation
The `/ask` endpoint SHALL require valid authentication via the `X-API-Key` HTTP header verified using constant-time string comparison.

#### Scenario: Request without API key
- **WHEN** a POST request is sent to `/ask` with missing `X-API-Key`
- **THEN** the server SHALL immediately reject the request with HTTP 401 Unauthorized

#### Scenario: Request with valid API key
- **WHEN** a POST request is sent to `/ask` with a valid `X-API-Key`
- **THEN** the server SHALL authenticate the client and associate the request with `X-User-Id` (or `anonymous`)

### Requirement: Sliding Window Rate Limiting
The service SHALL enforce a per-user sliding-window rate limit using Redis Sorted Sets to prevent request burst attacks across window boundaries.

#### Scenario: Request count exceeds limit within 60 seconds
- **WHEN** a user submits more requests than `RATE_LIMIT_PER_MINUTE` within any rolling 60-second window
- **THEN** subsequent requests within the window SHALL receive HTTP 429 Too Many Requests with a `Retry-After` header

#### Scenario: Expired requests roll out of window
- **WHEN** 60 seconds have elapsed since earlier requests
- **THEN** the expired requests SHALL be removed from the window and new requests permitted

### Requirement: Monthly Cost Guard Budgeting
The service SHALL track cumulative token costs per user per month and reject requests once the budget threshold is reached.

#### Scenario: Monthly budget exceeded
- **WHEN** an authenticated user whose monthly expenditure reaches `MONTHLY_BUDGET_USD` submits a request to `/ask`
- **THEN** the server SHALL reject the request with HTTP 402 Payment Required prior to invoking the LLM

### Requirement: Stateless Redis Conversation Persistence
The service SHALL maintain all conversation session history in Redis Lists rather than local memory, ensuring consistency across horizontally scaled instances.

#### Scenario: Multi-turn dialogue retrieval across instances
- **WHEN** user dialogue turns are appended by one instance
- **THEN** any other instance connected to the same Redis instance SHALL retrieve the full conversation history

#### Scenario: History trimming and expiration
- **WHEN** conversation history exceeds `HISTORY_MAX_MESSAGES` (20)
- **THEN** older entries SHALL be trimmed to keep only the most recent 20 messages, and a 7-day TTL SHALL be set

### Requirement: Dependency Readiness Verification
The service SHALL provide a `/ready` endpoint that validates external dependency connectivity before accepting incoming traffic.

#### Scenario: Redis connection active
- **WHEN** Redis is responsive and the service is not shutting down
- **THEN** GET `/ready` SHALL return HTTP 200 with `{"status": "ready", "redis": true}`

#### Scenario: Redis connection lost
- **WHEN** Redis is unresponsive or unreachable
- **THEN** GET `/ready` SHALL return HTTP 503 with `{"status": "not ready", "redis": false}`

### Requirement: Graceful Process Shutdown
The service SHALL trap `SIGTERM` and `SIGINT` signals, mark itself as shutting down to deflect load balancer traffic, and delegate process termination to the underlying server handler.

#### Scenario: Signal trap during deployment
- **WHEN** orchestrator emits `SIGTERM` to the agent process
- **THEN** the lifecycle manager SHALL set `shutting_down = True` and invoke the previously registered signal handler
