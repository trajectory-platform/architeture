# Auth Service — Implementation Plan

Implementation breakdown of the [Auth Service](../../services/auth.md) into small, independently reviewable tasks. Each task lives in [`tasks/`](tasks/) as a standalone brief an engineer (or coding agent) can execute without reading the whole plan.

**Read first, in order:**

1. [docs/engineering/code-conventions.md](../../engineering/code-conventions.md) — code style, folder structure, library choices. Mandatory for every task.
2. [docs/services/auth.md](../../services/auth.md) — service spec (API, DDL, flows, invariants).
3. Supporting: [07 — Security](../../architecture/07-security.md), [03 — Communication](../../architecture/03-communication.md), [C3 components](../../diagrams/c4/c3-auth.puml), sequence diagrams [registration](../../diagrams/sequence/auth-registration.puml) / [refresh rotation](../../diagrams/sequence/auth-refresh-rotation.puml).

## Task list & order

Phases are sequential; tasks inside a phase can run in parallel once their listed dependencies are done. Size: S ≈ ≤half day, M ≈ a day, L ≈ 1–2 days.

| # | Task | Size | Depends on |
|---|---|---|---|
| **Phase 0 — Foundation** | | | |
| [T01](tasks/T01-scaffolding.md) | Monorepo & service scaffolding, CI, lint, Makefile | M | — |
| [T02](tasks/T02-proto-contracts.md) | Proto contracts: `auth.v1` + `events.v1 UserRegistered` | S | T01 |
| [T03](tasks/T03-db-migrations.md) | auth_db schema migrations + goose wiring | S | T01 |
| **Phase 1 — Core building blocks** | | | |
| [T04](tasks/T04-app-bootstrap.md) | App bootstrap: composition root (`internal/app`) + thin `main.go`, config, logger, OTel, health/metrics, graceful shutdown | M | T01 |
| [T05](tasks/T05-domain-model.md) | Models (entities, statuses, errors) + service interfaces | S | T01 |
| [T06](tasks/T06-postgres-repositories.md) | Postgres repositories + tx manager | M | T03, T05 |
| [T07](tasks/T07-password-hasher.md) | Password hasher (argon2id) with timing-safe dummy verify | S | T05 |
| [T08](tasks/T08-key-manager.md) | Signing-key manager: lifecycle NEXT→ACTIVE→RETIRING→RETIRED, KEK encryption | L | T05, T06 |
| [T09](tasks/T09-token-issuer.md) | Token issuer: access JWT (EdDSA, kid) + refresh token generation | M | T08 |
| **Phase 2 — Use cases** | | | |
| [T10](tasks/T10-jwks-endpoint.md) | JWKS HTTP endpoint | S | T04, T08 |
| [T11](tasks/T11-register-usecase.md) | Register: identity + credentials + outbox in one tx, first token pair | M | T06, T07, T09 |
| [T12](tasks/T12-login-usecase.md) | Login: timing-equalized verify, new refresh family | M | T06, T07, T09 |
| [T13](tasks/T13-refresh-rotation-usecase.md) | Refresh rotation + theft detection (family revoke, jti blacklist) | L | T06, T09 |
| [T14](tasks/T14-logout-usecase.md) | Logout: idempotent family revoke | S | T06 |
| **Phase 3 — Transport & async** | | | |
| [T15](tasks/T15-grpc-server.md) | gRPC server: handlers, interceptors, error mapping | M | T02, T11–T14 |
| [T16](tasks/T16-outbox-relay.md) | Outbox relay → NATS JetStream (`pkg/outbox`) | M | T02, T03, T04 |
| **Phase 4 — Hardening & delivery** | | | |
| [T17](tasks/T17-observability.md) | Domain metrics, log fields audit, alert-ready signals | S | T15, T16 |
| [T18](tasks/T18-integration-e2e-tests.md) | Integration & E2E test suite (testcontainers) | L | T15, T16 |
| [T19](tasks/T19-compose-smoke.md) | docker-compose integration + smoke runbook | S | T18 |

## Dependency graph

```
T01 ─┬─ T02 ──────────────────────┬─ T15 ─┬─ T17
     ├─ T03 ─┬─ T06 ─┬─ T08 ─ T09 ┤       ├─ T18 ─ T19
     ├─ T04 ─┤       ├─ T11 ──────┤       │
     └─ T05 ─┴─ T07 ─┼─ T12 ──────┤  T16 ─┘
                     ├─ T13 ──────┤
                     └─ T14 ──────┘
(T10 JWKS: after T04+T08, parallel to use cases)
(T16 relay: after T02+T03+T04, parallel to Phase 2)
```

Critical path: **T01 → T03 → T06 → T08 → T09 → T13 → T15 → T18**.

## Definition of done (service level)

- All gRPC methods (`Register`, `Login`, `Refresh`, `Logout`) and `GET /.well-known/jwks.json` behave per [auth.md](../../services/auth.md), including error semantics (`AlreadyExists`, indistinguishable `Unauthenticated`, idempotent logout).
- Invariants hold under concurrency: one ACTIVE refresh token per family (DB-enforced), token reuse revokes the family, tokens stored only as sha256 hashes, key lifecycle never signs with a non-ACTIVE key.
- `user.registered` is published at-least-once via outbox; never for a rolled-back registration.
- E2E suite (T18) green in CI; service runs in `docker compose up` with health checks passing (T19).

## Conventions for executing a task

- Branch `auth/Txx-short-name`, one PR per task, conventional commits.
- Acceptance criteria in the task file are the review checklist; tests proving them ship in the same PR.
- If implementation reveals a spec conflict, update the task file (and flag it in the PR) — task files are living documents; architecture docs change only with team agreement.
