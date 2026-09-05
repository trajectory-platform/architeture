# T01 — Service repository scaffolding

**Size:** M · **Depends on:** — · **Phase:** 0

> Evolution note: the original T01 implementation scaffolded NATS JetStream. [ADR-003](../../../adr/ADR-003-apache-kafka.md) replaces that choice. The target state below uses Kafka; T16 performs the broker/config migration before event delivery is accepted.

## Goal

Empty-but-running auth-service repository: service layout from [code-conventions.md §1](../../../engineering/code-conventions.md), a module that compiles and starts, CI gates, lint and Makefile. Contracts are consumed from the separate versioned `contracts` repository. Everything later tasks plug into.

## Scope

In:

- Root of `auth-service`: `go.mod`, `.golangci.yml`, `Makefile`, `.gitignore`, `.env.example`.
- `cmd/auth/main.go` — starts, logs "auth service starting", exits cleanly on SIGINT/SIGTERM.
- `Dockerfile` — multi-stage (golang → distroless), self-contained build of this repository.
- `docker-compose.yaml` — development dependencies: `postgres-auth`, Kafka in KRaft mode, `redis`, `auth` service. Healthchecks on infra containers.
- CI workflow: `go build ./...`, `go vet ./...`, `golangci-lint run`, `go test ./...` in this module. Contract compatibility is validated in `contracts` CI.
- Makefile targets: `lint`, `test`, `test-integration`, `generate`, `up`, `down`.

Out: any business logic, proto (T02), migrations (T03), real config/observability (T04).

## Implementation notes

- golangci-lint enable at minimum: `errcheck`, `govet`, `staticcheck`, `revive`, `gosec`, `errorlint`, `sqlclosecheck`, `noctx`, `depguard` (forbid importing `services/*/internal` across modules).
- Pin Go version in `go.mod` and Dockerfile; document Makefile targets in this repository README.
- Compose: internal Docker network only; no service ports published except what local dev needs (Postgres/Kafka/Redis on localhost for tooling is fine in dev compose).

## Acceptance criteria

- [ ] `go build ./...` and `golangci-lint run` pass from the auth-service repository root.
- [ ] `docker compose up` brings up postgres-auth, Kafka and redis healthy; auth container starts and stays up.
- [ ] `make lint test` works on a clean checkout.
- [ ] CI runs and is green on the PR.
- [ ] Layout matches conventions §1 exactly (reviewer checks tree).
