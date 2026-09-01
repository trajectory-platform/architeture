# Code Conventions & Repository Structure

Engineering guide for implementing Trajectory services. Written for both humans and coding agents: follow it exactly unless a task file explicitly overrides it. Architecture docs ([docs/README.md](../README.md)) define *what* to build; this document defines *how* and *where*.

> Repo documentation prose is in Russian (see [docs/README.md — Конвенции](../README.md)); engineering artifacts (this file, task plans, code, comments, commit messages) are in English.

## 1. Repository layout (multi-repo)

```
trajectory/                         # workspace, not a Git monorepo
├── architecture/                   # documentation/ADR/diagrams repository
├── contracts/                      # buf + OpenAPI source and generated Go module
├── auth-service/                   # standalone Go service repository
├── api-gateway/                    # standalone Go service repository
├── core-education-service/         # standalone Go service repository
├── billing-service/                # standalone Go service repository
├── realtime-service/               # standalone Go service repository
├── notification-service/           # standalone Go service repository
├── platform-go/                    # optional, versioned shared technical libraries only
└── deploy/                         # deployment manifests and pinned image versions
```

Rules:

- **Один репозиторий и Go-модуль на сервис.** Сервис никогда не импортирует другой сервис или его `internal/`; локальный `go.work` не является частью продукта и не нужен для сборки CI.
- **`contracts` — единственное место межсервисных контрактов.** Он публикует versioned Go-модуль с кодогенерацией. Сервисы обновляют зависимость явно, а `buf lint` + `buf breaking` и OpenAPI lint/diff работают в CI `contracts` ([ADR-004](../adr/ADR-004-grpc-internal-rest-edge.md)).
- **Общие библиотеки — только в отдельном versioned `platform-go` и только для технического кода.** Если библиотекой пользуется один сервис, она остаётся в его `internal/`; доменная логика никогда не выносится в shared package.
- **Каждый сервис выпускает собственные образ, CI, миграции и Makefile.** `deploy` ссылается на immutable теги образов и не собирает исходники соседних репозиториев.

## 2. Service-internal layout (6-layer)

Every Go service follows the same shape. Example for `auth-service/`:

```
auth-service/
├── cmd/auth/main.go                # entry point only: parse config, call app.Run. No logic.
├── internal/
│   ├── app/                        # composition root / Dependency Injection: builds every
│   │                               #   dependency (pool, logger, repositories, services, transport)
│   │                               #   and runs the server. The only package that knows concretes.
│   ├── config/                     # env parsing into one Config struct; fail-fast validation
│   ├── models/                     # entities, value objects, business errors, status enums. PURE:
│   │                               #   no SQL, no proto, no logging, stdlib-only imports
│   ├── service/                    # business logic (use cases). One file per use case. DEFINES the
│   │                               #   interfaces (repositories, hasher, token issuer, clock) it
│   │                               #   needs, on the consumer side. Depends only on models.
│   ├── repository/                 # database access; implements the interfaces service declares
│   │   ├── postgres/               # SQL lives ONLY here
│   │   └── redis/                  # e.g. jti blacklist
│   ├── infra/                      # non-DB infrastructure, each behind a service interface:
│   │   ├── argon2id/               #   password hasher
│   │   ├── token/                  #   JWT/EdDSA issuer
│   │   ├── clock/                  #   wall clock (deterministic in tests)
│   │   └── keycrypto/              #   signing-key sealer (KEK). Each knows models only.
│   └── transport/                  # driving side: validate input, call service, format response
│       ├── grpc/                   # proto handler → service call → error mapping to gRPC codes
│       └── http/                   # JWKS, /healthz, /readyz, /metrics
├── migrations/                     # goose SQL migrations, embedded; owned by this service only
├── Dockerfile                      # multi-stage, distroless, same pattern for every service
└── go.mod
```

**The dependency rule** (compiler-enforced by import direction):

```
transport                          ──► service ──► models
repository                         ──► service (implements its interfaces) ──► models
infra/* (argon2id, token, clock, …) ──► service (implement its interfaces) ──► models
app                                ──► everything (wiring only)
```

- `models` imports nothing from the project (and no third-party libs beyond stdlib).
- `service` defines the interfaces for what it needs (repositories, hasher, token issuer, clock); it never imports `repository`, `transport`, or the infrastructure packages. This is Dependency Inversion: implementations depend on the service's interfaces, not the reverse.
- DI is constructor-based (`NewManager(repo SigningKeyRepo, ...)`). `internal/app` is the single place that imports the concrete implementations, wires them into services, and starts the servers; `cmd/auth/main.go` only parses config and calls `app.Run`.
- Proto-generated types do not leak past `transport/grpc` — handlers convert proto ⇄ service inputs / models.
- SQL strings exist only in `repository/postgres`. JWT/crypto libs only in the dedicated infrastructure package that owns them (`token`, `keycrypto`, `argon2id`).

**Folder discipline:** the canonical layers are `app, config, models, repository, service, transport`. Database access groups under `repository/`; each non-DB external dependency gets its own package under `internal/infra/` (kept behind a service interface for testability) rather than a catch-all bucket. Do not create `ports/`, `adapters/`, or `domain/` folders — interfaces belong in `service/`, data types in `models/`.

## 3. Go code style

Baseline: [Effective Go](https://go.dev/doc/effective_go) + [Google Go Style Guide](https://google.github.io/styleguide/go/). On top of that:

### Naming & files

- Package names: short, lowercase, singular, no underscores: `models`, `postgres`, `token`. **Never** `utils`, `common`, `helpers`, `misc` — name the package after what it provides.
- File names: snake_case, named for content: `refresh_rotation.go`, `identity_repo.go`. One use case per file in `service/`.
- Test files next to code: `refresh_rotation_test.go`.
- Constructors: `New(...)` if package exports one type, `NewXxx(...)` otherwise. Constructors take dependencies explicitly; no `init()` magic, no package-level mutable state, no global singletons (logger is passed or embedded in deps).

### Errors

- Business errors are typed sentinels in `models/errors.go`: `ErrEmailTaken`, `ErrInvalidCredentials`, `ErrTokenReused`, `ErrTokenExpired`, `ErrKeyNotFound`.
- Wrap with context: `fmt.Errorf("rotate refresh token: %w", err)`. Lowercase, no punctuation, verb-first describing the failed operation.
- Mapping to gRPC codes happens **once**, in `transport/grpc/errors.go` — nowhere else. Internal details never leak into client-facing messages (e.g. Login returns plain `Unauthenticated`, never "email not found").
- Never ignore an error silently. `_ =` requires a comment explaining why discarding is safe.

### Context & concurrency

- `ctx context.Context` is the first parameter of every function that does I/O. Never stored in structs.
- Every **outbound** call (SQL, Redis, NATS, gRPC client) runs under a deadline — either the inherited one or `context.WithTimeout`. A call without a deadline is a bug ([03 — Communication](../architecture/03-communication.md)).
- Background goroutines (outbox relay, key rotation ticker) are started in `internal/app`, stop on context cancellation, and register their teardown with `pkg/closer` for graceful shutdown.

### Logging

- `log/slog` via `pkg/logger` (JSON to stdout). The logger is propagated through `context` (`logger.WithContext` / `logger.FromContext`); code logs via `logger.FromContext(ctx)`, never a stored field or a global.
- Every entry carries `service`, `trace_id` (from ctx via `logger.WithTrace`); add `user_id` where applicable.
- **Never log**: passwords, password hashes, raw tokens, token hashes, full email at INFO level (PII policy — [08](../architecture/08-deployment-observability.md)).
- Keep logging out of `models/`; in `service/` log only the few security-relevant events (e.g. refresh reuse) via `logger.FromContext(ctx)`. Otherwise log at the transport, repository, and infrastructure edges and in background loops.

### Comments

- Godoc comments on every exported symbol. Comment *why*, not *what*. Document invariants the code can't express ("exactly one ACTIVE token per family — enforced by partial unique index").

### Library choices (fixed — do not introduce alternatives)

| Concern | Library |
|---|---|
| Postgres | `jackc/pgx/v5` (pgxpool) + `Masterminds/squirrel` query builder — no ORM |
| Migrations | `pressly/goose/v3`, embedded via `embed.FS`, applied on service start |
| JWT / JWK / JWKS | `lestrrat-go/jwx/v2` |
| Password hashing | `golang.org/x/crypto/argon2` (argon2id) |
| gRPC | `google.golang.org/grpc` + buf-generated stubs |
| NATS | `nats-io/nats.go` (jetstream API) |
| Redis | `redis/go-redis/v9` |
| Logging | `log/slog` (stdlib), JSON to stdout, propagated via `context` |
| Config | `caarlos0/env/v11` (struct tags) + `joho/godotenv` (.env) |
| Tracing/metrics | OpenTelemetry SDK + `otelgrpc`; Prometheus exporter |
| Tests | stdlib `testing` + `stretchr/testify` (require/assert) + `testcontainers-go` |
| UUID | `google/uuid` (v7 for new PKs where ordering helps, v4 elsewhere) |

## 4. Database conventions

- Schema changes only via goose migrations in the owning service's `migrations/`; sequentially numbered `0001_init.sql`, `0002_....sql`, with `-- +goose Up` / `-- +goose Down`.
- Tables/columns: snake_case; timestamps `timestamptz` named `created_at` / `updated_at` / `expires_at`; status columns `text` with CHECK constraint, mirrored by a Go enum type in `models`.
- Invariants live in the database where possible (unique indexes, partial indexes, FKs, CHECKs) — code relies on them instead of re-checking racily.
- Multi-statement consistency: one transaction via the tx manager (`pkg/postgres`), composed in the `service` layer (`txm.WithTx(ctx, func(ctx) error {...})`); repos read the tx from ctx.
- No cross-database queries, ever ([02 — Services](../architecture/02-services.md)).

## 5. Testing

- **Unit tests** (no I/O): all of `models/` and `service/` with port fakes — hand-written fakes in `internal/service/fakes_test.go`, no mock-generation frameworks. Table-driven where natural. Use case logic should reach high coverage because it carries the security invariants.
- **Integration tests**: `repository/` and the infrastructure packages against real Postgres/Redis/NATS via testcontainers, build tag `//go:build integration`, run with `make test-integration`.
- **E2E (service-level)**: spin up the whole service binary against containers and exercise the gRPC API for key flows (register → login → refresh → reuse-detection → logout).
- Determinism: time is a port (`Clock` interface) defined in `service`; tests inject a fixed clock. No `time.Sleep` synchronization in tests.
- A task is "done" only with tests proving its acceptance criteria.

## 6. Configuration

- Env vars only, prefixed per service: `AUTH_DB_DSN`, `AUTH_GRPC_ADDR`, `AUTH_HTTP_ADDR`, `AUTH_NATS_URL`, `AUTH_REDIS_ADDR`, `AUTH_ACCESS_TTL`, `AUTH_REFRESH_TTL`, `AUTH_KEK` ...
- Parsed in `internal/config` into a single `Config` struct at startup with `caarlos0/env/v11` struct tags, after a best-effort `joho/godotenv` `.env` read (real env wins); invalid/missing required values → process exits non-zero with a clear message. No config reads anywhere else in the code.
- Secrets (KEK, DB creds) never logged, never committed; compose uses `.env` (gitignored) with a committed `.env.example`.

## 7. Observability (every service, day one)

- gRPC server with `pkg/grpcx` interceptor chain: recovery → OTel → logging → (validation).
- HTTP server exposes `/metrics` (Prometheus), `/healthz` (liveness), `/readyz` (readiness: DB ping, NATS connected).
- RED metrics per gRPC method come from interceptors; each service adds its domain metrics (auth: outbox depth, refresh reuse detections, active signing key age).

## 8. Git & process

- Branch per task: `auth/T07-password-hasher`. One task = one reviewable PR.
- Conventional commits: `feat(auth): refresh rotation with reuse detection`, `chore(proto): auth.v1 contracts`.
- CI gates on every PR: `go build ./...`, `go vet`, `golangci-lint run`, unit tests, `buf lint`, `buf breaking` (against main).
- Generated code (`gen/`) is committed in the same PR as the proto change that produced it.

## 9. Cleanliness checklist (before marking any task done)

1. `make lint test` green; integration tests green if touched repository or infrastructure packages.
2. No TODOs without a linked task; no commented-out code; no dead exports.
3. New exported symbols have godoc; invariants documented.
4. Dependency rule holds (models pure, service defines and owns its interfaces, SQL only in `repository/postgres`, proto only in `transport`).
5. Errors wrapped with operation context; gRPC mapping table updated if a new business error appeared.
6. No secrets/PII in logs or test fixtures.
