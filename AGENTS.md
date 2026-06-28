# Task Manager Monorepo — Project Rules

These rules apply to every session in this repository.
Adapt conventions to the language/runtime of the service being worked on — do not assume a language.

---

## Monorepo Layout

- All backend services live in `services/<name>/`
- Frontend lives in `frontend/`
- All infrastructure lives in `infra/`
- Shared types go in `packages/` — never import across services directly from src/
- One Dockerfile per service in the service root
- Each service is independently runnable and deployable

---

## Language and Runtime

This monorepo contains services in multiple languages. Before writing code for any service,
check existing files to determine the runtime — do not assume a single language.

Current services and their languages — chosen for each service's strength:
- auth-service: Go / net/http + chi router + pgx + golang-jwt (low latency, crypto perf, JWT handling)
- task-service: Python / FastAPI + asyncpg + PostgreSQL (readable CRUD logic, rich query handling)
- notification-service: Node.js / TypeScript / Fastify + MongoDB + ioredis (event-driven, WebSocket, pub/sub)
- analytics-service: Go / net/http + chi router + clickhouse-go (high-throughput ingestion, minimal overhead)
- worker-service: Python / arq + Redis (background jobs, scheduling, digest logic — Python's async job ecosystem)
- frontend: Next.js 14 App Router / TypeScript / Tailwind CSS

Each language was chosen for the service's primary workload. Do not rewrite a service in a different
language without an explicit decision recorded in a DECISION.md at the repo root.

If a new service is added:
- Use the language best suited to its workload — document the rationale in its README.md
- Apply the same structural conventions regardless of language:
  health endpoint, metrics endpoint, structured logging, env-based config, multistage Dockerfile
- Use the idiomatic tooling for that language (see Logging, Metrics, Testing sections)
- Do not force patterns from one language's ecosystem onto another

---

## Project Structure per Service

Every service, regardless of language, must follow this layout adapted to language conventions:

Node.js/TypeScript:
```
services/<name>/
├── src/
├── Dockerfile
├── .dockerignore
├── package.json
├── tsconfig.json
└── README.md
```

Go:
```
services/<name>/
├── cmd/<name>/       # main entrypoint
├── internal/         # private packages
├── Dockerfile
├── .dockerignore
├── go.mod
└── README.md
```

Python:
```
services/<name>/
├── app/              # application code
├── tests/
├── Dockerfile
├── .dockerignore
├── pyproject.toml    # or requirements.txt + requirements-dev.txt
└── README.md
```

---

## Docker

- All Dockerfiles must be multistage — builder stage and runner stage minimum
- Builder stage: use the full SDK/toolchain image to install deps and compile
- Runner stage: use the smallest alpine variant — never a full OS image, never distroless, never debian/ubuntu:
  - Node.js services: node:20-alpine
  - Go services: alpine:3.19 (copy compiled binary only — no Go toolchain in runner)
  - Python services: python:3.12-alpine
- Runner stage must create a non-root group and user with gid/uid 1001,
  chown the app directory, and switch to that user before CMD:
  ```dockerfile
  RUN addgroup -g 1001 -S appgroup && \
      adduser -u 1001 -S appuser -G appgroup
  RUN chown -R appuser:appgroup /app
  USER appuser
  ```
- Copy only what is needed to run — no source files, no dev tooling, no test files:
  - Node.js: dist/ and node_modules (production only — run npm ci --omit=dev in builder)
  - Go: single compiled binary
  - Python: virtual environment or installed site-packages + app/ source
- No secrets, tokens, or .env files baked into any image layer
- Every image must have a HEALTHCHECK instruction:
  ```dockerfile
  HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -qO- http://localhost:<PORT>/healthz || exit 1
  ```
- .dockerignore must exist per service and exclude:
  - Node.js: node_modules, src, *.ts, .env*, coverage, .git
  - Go: vendor (if not vendoring), .git, *_test.go, .env*
  - Python: __pycache__, .venv, tests, .env*, .git, *.pyc

---

## Health Endpoint

Every service must expose GET /healthz that:
- Returns HTTP 200 with `{ "status": "ok", "service": "<name>" }` when healthy
- Returns HTTP 503 if a critical dependency (DB, Redis) is unreachable
- Requires no authentication
- Is cheap — no writes, no heavy computation

---

## Configuration and Environment Variables

- All configuration comes from environment variables — no hardcoded URLs, ports, or credentials
- At startup, validate all required env vars. On missing required var: log a clear error and exit(1)
- Validation approach by language:
  - Go: Config struct in internal/config/config.go populated via os.Getenv,
    validated in main() before the server starts — missing required vars cause immediate exit(1)
  - Python: pydantic BaseSettings in app/config.py — field without a default is required;
    missing vars raise a ValidationError at import time, caught in main with exit(1)
  - Node.js/TypeScript: zod schema in src/config.ts, parsed at startup before anything else
- Env var naming: SCREAMING_SNAKE_CASE, prefixed with the service name:
  - Good: AUTH_SERVICE_DB_URL, TASK_SERVICE_DB_URL, ANALYTICS_SERVICE_CLICKHOUSE_DSN
  - Bad: DB_URL, PORT (ambiguous across services)
- Document every env var in the service README.md: name, description, required/optional, example value

---

## Logging

Use the idiomatic structured logging library for the language — emit JSON in production:
- Go (auth-service, analytics-service): zerolog — output to stdout as JSON
- Python (task-service, worker-service): structlog with JSON renderer in production
- Node.js/TypeScript (notification-service): pino

Rules that apply regardless of language:
- No fmt.Println, print(), or console.log in production code paths
- Every log entry must include: service (hardcoded string), traceId, level, timestamp (UTC ISO8601)
- traceId: read from X-Trace-Id request header if present; generate a UUID v4 if absent;
  store in request context and pass as X-Trace-Id header on all downstream HTTP calls
- Log on every request completion: method, path, statusCode, responseTimeMs
- Log levels:
  - info: normal lifecycle events, job start/complete, service startup with config summary
  - warn: retried operation, slow query >500ms, degraded dependency state
  - error: caught exceptions, failed DB operations, downstream service failures —
    always include the full error object or stack trace
  - debug: development only, gated by LOG_LEVEL=debug env var
- Never log: passwords, tokens, API keys, PII, full request/response bodies on auth routes
- Initialise the logger once at startup — inject via context or pass explicitly; never instantiate per function

---

## Metrics

Use the idiomatic Prometheus client for the language:
- Go: prometheus/client_golang
- Python: prometheus-client
- Node.js/TypeScript: prom-client

Rules that apply regardless of language:
- Expose GET /metrics on the same port as the service (worker-service exposes a minimal HTTP server for this)
- Use a named/isolated registry — not the default global registry
- Every service must expose at minimum:
  - http_request_duration_seconds: histogram, labels: method, route, status_code
  - http_requests_total: counter, labels: method, route, status_code
- Service-specific required metrics:
  - auth-service (Go): auth_login_total, auth_login_failed_total, auth_token_issued_total
  - task-service (Python): tasks_created_total, tasks_completed_total, db_query_duration_seconds
  - notification-service (Node.js): notifications_sent_total, notifications_failed_total, redis_publish_duration_seconds
  - analytics-service (Go): events_ingested_total, clickhouse_insert_duration_seconds
  - worker-service (Python): jobs_processed_total, jobs_failed_total, job_processing_duration_seconds (label: job_type)
- Define histogram buckets explicitly — do not rely on library defaults for latency metrics:
  - Recommended latency buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
- Metrics middleware/interceptor must be registered before route handlers

---

## Code Comments

Adapt syntax to the language — rules apply to all:
- Go: GoDoc comments on all exported functions, types, and methods
- Python: docstrings (Google style) on all public functions, classes, and methods
- Node.js/TypeScript: JSDoc on all exported functions, classes, and route handlers

Every doc comment must describe:
- What the function/method does
- Parameters and return value (even when types are declared)
- Side effects: DB write, cache invalidation, event publish, external HTTP call

Inline comments required for:
- Non-obvious business logic
- DB query rationale (why this query, not just what it does)
- Error handling decisions (why swallow vs propagate)
- Any workaround or known limitation — include TODO: or FIXME: with a short reason

No commented-out code committed to the repository.

---

## Error Handling

- Every HTTP service must have a global/centralised error handler at the framework level:
  - Go/chi: middleware that recovers from panics and formats error responses
  - Python/FastAPI: exception handlers registered via app.add_exception_handler()
  - Node.js/Fastify: setErrorHandler()
- All 4xx responses: `{ "error": "<type>", "message": "<human readable>", "statusCode": <number> }`
- All 5xx responses: same shape — must NOT leak stack traces, internal error messages, or DB errors to clients
- Log 5xx errors at error level with full detail server-side before sending the sanitised response
- DB connection failure at startup: log at error level and exit(1) — do not serve traffic in a broken state
- Go: always check returned errors explicitly — never use _ for errors that matter
- Python: never use bare except — always catch specific exception types
- Node.js: no unhandled promise rejections — use try/catch in async functions or .catch() on promises

---

## Auth Architecture

- auth-service (Go) owns all user identity: registration, login, JWT issuance, token refresh, logout
- auth-service has its own PostgreSQL database — no other service reads the users table directly
- JWT tokens are issued by auth-service and verified by each downstream service independently
  using the shared JWT_SECRET env var — no per-request round-trip to auth-service
- JWT payload must include: userId, email, role, iat, exp
- Token expiry: access token 15 minutes, refresh token 7 days
- auth-service routes:
  - POST /auth/register
  - POST /auth/login
  - POST /auth/refresh
  - POST /auth/logout
  - GET /auth/me (decodes token, returns user profile — no DB call)
- Refresh tokens stored in Redis with TTL matching token expiry — invalidated on logout
- All other services validate JWT locally using a shared middleware — do not call auth-service per request
- Frontend stores access token in memory (Zustand), refresh token in httpOnly cookie

---

## Inter-Service Communication

- Services communicate via HTTP only — no direct cross-service DB access
- All backend services are reachable by the frontend directly (no API gateway)
- Service discovery via env vars: TASK_SERVICE_URL, NOTIFICATION_SERVICE_URL, ANALYTICS_SERVICE_URL, etc.
- All outbound HTTP calls must have:
  - Timeout: 5000ms default
  - One retry on 503 or network error
  - warn log on retry, error log on final failure
- Pass X-Trace-Id header on all outbound calls — use the traceId from the current request context
- Never hardcode http://localhost in inter-service calls — always use env var URLs

---

## Database Conventions

Use the official driver for each language/database combination — no ORMs unless explicitly decided:

PostgreSQL:
- Go (auth-service): pgx/v5
- Python (task-service): asyncpg

MongoDB:
- Node.js (notification-service): mongodb official driver

ClickHouse:
- Go (analytics-service): clickhouse-go/v2

Redis:
- Go (auth-service): go-redis/v9
- Python (worker-service): redis-py (async: aioredis or redis.asyncio)
- Node.js (notification-service): ioredis

General rules:
- Initialise the DB client once at startup, export/inject as a singleton — no new connections per request
- Connection pool sizes must be configurable via env vars
- Migrations (PostgreSQL services only):
  - SQL files in services/<name>/migrations/, named with numeric prefix: 001_create_users.sql
  - Run migrations at service startup before the HTTP server begins accepting traffic
  - Never modify an existing migration file — always add a new numbered file

---

## Testing

Use the idiomatic test toolchain for each language:
- Go (auth-service, analytics-service): standard testing package + testify/assert + testify/require
- Python (task-service, worker-service): pytest + pytest-asyncio + httpx (for FastAPI route tests)
- Node.js/TypeScript (notification-service): Vitest + fastify.inject() for route tests
- Frontend: React Testing Library + Vitest for components, Playwright for E2E

Rules that apply regardless of language:
- Every service must have all three test types:
  - Unit tests: business logic only — no real DB, no network — mock/stub all external dependencies
  - Integration tests: real DB spun via testcontainers — not mocks, not SQLite, not in-memory substitutes
  - Route/handler tests: every HTTP endpoint tested for:
    - Happy path (200/201 with correct response shape)
    - Validation error (400 on invalid/missing fields)
    - Auth failure (401 when no token or invalid token)
    - Not found (404 when resource does not exist)

Test file location by language:
- Go: same package directory, <name>_test.go
- Python: tests/ subdirectory, test_<name>.py
- Node.js: colocated with source as <name>.test.ts

Coverage threshold: 80% line coverage minimum — CI must fail below this

Integration tests must clean up after each test:
- Go: t.Cleanup() with truncate or DROP + recreate
- Python: pytest fixture with yield + teardown truncate
- Node.js: afterEach with truncate or transaction rollback

E2E tests go in frontend/e2e/ using Playwright — run against the docker-compose stack
No permanently skipped tests — fix or delete them

---

## Git and Commits

- Conventional commits: feat:, fix:, chore:, docs:, test:, refactor:
- One logical change per commit
- Before marking any task done: tests pass, linter passes, docker build succeeds
- Do not commit: .env files, node_modules, __pycache__, build output, coverage reports, vendor/ (Go)
