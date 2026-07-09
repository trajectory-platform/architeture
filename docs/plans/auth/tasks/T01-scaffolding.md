# T01 — Monorepo & service scaffolding

**Size:** M · **Depends on:** — · **Phase:** 0

## Goal

Empty-but-running skeleton: repo layout from [code-conventions.md §1](../../../engineering/code-conventions.md), auth service module that compiles and starts, CI gates, lint, Makefile. Everything later tasks plug into.

## Scope

In:

- Root: `go.work`, `.golangci.yml`, `Makefile`, `.gitignore`, `.env.example`.
- Modules: `gen/` (placeholder `go.mod` `trajectory/gen`), `pkg/` (`trajectory/pkg`, empty packages `logger`, `grpcx`, `postgres`, `natsx`, `otelx`, `outbox` may be stubs with doc.go), `services/auth/` (`trajectory/services/auth`).
- `services/auth/cmd/auth/main.go` — starts, logs "auth service starting", exits cleanly on SIGINT/SIGTERM.
- `services/auth/Dockerfile` — multi-stage (golang → distroless), build arg pattern reusable by other services.
- `deploy/docker-compose.yml` — initial stack: `postgres-auth`, `nats` (JetStream enabled), `redis`, `auth` service (build from Dockerfile). Healthchecks on infra containers.
- CI workflow: `go build ./...`, `go vet ./...`, `golangci-lint run`, `go test ./...` across workspace.
- Makefile targets: `lint`, `test`, `test-integration`, `generate`, `up`, `down`.

Out: any business logic, proto (T02), migrations (T03), real config/observability (T04).

## Implementation notes

- golangci-lint enable at minimum: `errcheck`, `govet`, `staticcheck`, `revive`, `gosec`, `errorlint`, `sqlclosecheck`, `noctx`, `depguard` (forbid importing `services/*/internal` across modules).
- Pin Go version in `go.work` and Dockerfile; document in root README of the Makefile targets.
- Compose: internal Docker network only; no service ports published except what local dev needs (Postgres/NATS/Redis on localhost for tooling is fine in dev compose).

## Acceptance criteria

- [ ] `go build ./...` and `golangci-lint run` pass from repo root.
- [ ] `docker compose up` brings up postgres-auth, nats, redis healthy; auth container starts and stays up.
- [ ] `make lint test` works on a clean checkout.
- [ ] CI runs and is green on the PR.
- [ ] Layout matches conventions §1 exactly (reviewer checks tree).
