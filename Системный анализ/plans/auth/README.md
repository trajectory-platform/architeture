# Auth Service — Implementation Plan

Implementation breakdown of the [Auth Service](../../services/auth.md) into small, independently reviewable tasks. Each task lives in [`tasks/`](tasks/) as a standalone brief.

T01–T11 describe the baseline already delivered or reviewed: password authentication, one role, refresh rotation, signing keys and JWKS. T12–T19 continue that baseline. [ADR-007](../../adr/ADR-007-auth-identity-access-evolution.md) accepts a larger Auth MVP based on the new business analysis. Do not rewrite baseline migrations or hide this evolution. T20–T31 add forward-only contracts, migrations and behavior.

**Read first, in order:**

1. [Code conventions](../../engineering/code-conventions.md).
2. [Auth Service specification](../../services/auth.md).
3. [ADR-007 — Auth identity/access evolution](../../adr/ADR-007-auth-identity-access-evolution.md).
4. Product sources: [Roles and permissions](../../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Роли%20и%20разрешения.md), [Sessions and security](../../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Сессии%20и%20безопасность.md), [User card](../../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Карточка%20пользователя.md), [Audit log](../../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Журнал%20действий.md).
5. Supporting system docs: [Security](../../architecture/07-security.md), [Communication](../../architecture/03-communication.md), [C3 baseline](../../diagrams/c4/c3-auth.puml), [registration baseline](../../diagrams/sequence/auth-registration.puml), [refresh rotation](../../diagrams/sequence/auth-refresh-rotation.puml).

## Task list and order

Phases are sequential. Tasks inside a phase may run in parallel only after their dependencies are complete. Size: S ≈ ≤ half day, M ≈ one day, L ≈ 1–2 days.

| # | Task | Size | Depends on |
|---|---|---|---|
| **Phase 0 — Baseline foundation** | | | |
| [T01](tasks/T01-scaffolding.md) | Service repository scaffolding, CI, lint, Makefile | M | — |
| [T02](tasks/T02-proto-contracts.md) | Baseline `auth.v1` and `events.v1` contracts | S | T01 |
| [T03](tasks/T03-db-migrations.md) | Baseline auth_db migrations | S | T01 |
| **Phase 1 — Baseline building blocks** | | | |
| [T04](tasks/T04-app-bootstrap.md) | Composition root, config, OTel, health, shutdown | M | T01 |
| [T05](tasks/T05-domain-model.md) | Baseline domain models and interfaces | S | T01 |
| [T06](tasks/T06-postgres-repositories.md) | Baseline Postgres repositories and transactions | M | T03, T05 |
| [T07](tasks/T07-password-hasher.md) | Argon2id with timing-safe dummy verify | S | T05 |
| [T08](tasks/T08-key-manager.md) | Ed25519 signing-key lifecycle and KEK encryption | L | T05, T06 |
| [T09](tasks/T09-token-issuer.md) | Baseline JWT and refresh-token issuer | M | T08 |
| **Phase 2 — Baseline use cases** | | | |
| [T10](tasks/T10-jwks-endpoint.md) | JWKS HTTP endpoint | S | T04, T08 |
| [T11](tasks/T11-register-usecase.md) | Baseline registration | M | T06, T07, T09 |
| [T12](tasks/T12-login-usecase.md) | Password login | M | T06, T07, T09 |
| [T13](tasks/T13-refresh-rotation-usecase.md) | Refresh rotation and theft detection | L | T06, T09 |
| [T14](tasks/T14-logout-usecase.md) | Idempotent family logout | S | T06 |
| **Phase 3 — Baseline transport and events** | | | |
| [T15](tasks/T15-grpc-server.md) | Baseline gRPC server and error mapping | M | T02, T11–T14 |
| [T16](tasks/T16-outbox-relay.md) | Transactional outbox relay to Kafka | M | T02, T03, T04 |
| **Phase 4 — Baseline hardening** | | | |
| [T17](tasks/T17-observability.md) | Metrics, logging and tracing | S | T15, T16 |
| [T18](tasks/T18-integration-e2e-tests.md) | Baseline integration/E2E suite | L | T15, T16 |
| [T19](tasks/T19-compose-smoke.md) | Compose integration and smoke runbook | S | T18 |
| **Phase 5 — Contract and schema evolution** | | | |
| [T20](tasks/T20-auth-mvp-contracts.md) | Full Auth MVP contracts, compatibility/versioning | L | T02 |
| [T21](tasks/T21-identity-access-migrations.md) | Forward-only identity/access/session migrations and backfill | L | T03, T20 |
| **Phase 6 — Auth MVP domain** | | | |
| [T22](tasks/T22-access-model-and-token-claims.md) | User/staff access model, roles/permissions claims and init context | L | T09, T20, T21 |
| [T29](tasks/T29-auth-audit-log.md) | Immutable Auth-owned audit log | M | T20, T21 |
| [T23](tasks/T23-registration-confirmation-and-invitations.md) | Pending registration, confirmation and invitations | L | T11, T16, T20–T22, T29 |
| [T24](tasks/T24-session-management.md) | Session metadata, limit, expiry and revocation | L | T13, T14, T20–T22, T29 |
| **Phase 7 — Auth MVP security and administration** | | | |
| [T25](tasks/T25-oauth-connections.md) | OAuth login and safe account linking | L | T20–T24, T29 |
| [T26](tasks/T26-password-and-email-security-flows.md) | Password recovery/change and confirmed email change | L | T07, T16, T20–T24, T29 |
| [T27](tasks/T27-admin-role-management.md) | Configurable admin roles, permissions and staff assignments | L | T20–T22, T29 |
| [T28](tasks/T28-account-lifecycle-and-anonymization.md) | Blocking, deletion and anonymization | L | T20–T24, T29 |
| **Phase 8 — Full transport and proof** | | | |
| [T30](tasks/T30-expanded-grpc-transport.md) | Complete Auth MVP gRPC transport | L | T15, T20, T22–T29 |
| [T31](tasks/T31-auth-mvp-e2e.md) | Complete Auth MVP E2E suite | L | T16, T23–T30 |

## Dependency graph

```text
Baseline sequence:
T01 ─┬─ T02 ──────────────────────┬─ T15 ─┬─ T17
     ├─ T03 ─┬─ T06 ─┬─ T08 ─ T09 ┤       ├─ T18 ─ T19
     ├─ T04 ─┤       ├─ T11 ──────┤       │
     └─ T05 ─┴─ T07 ─┼─ T12 ──────┤  T16 ─┘
                     ├─ T13 ──────┤
                     └─ T14 ──────┘

Auth MVP evolution:
T02 ─ T20 ─ T21 ─┬─ T22 ────────────────┐
T03 ──────────────┤                       │
T09 ──────────────┘                       │
T20 ─ T21 ─ T29 ─┬─ T23 ────────────────┤
                 ├─ T24 ─┬─ T25 ────────┤
T16 ─────────────┤       └─ T26 ────────┤
                 ├─ T27 ────────────────┤
                 └─ T28 ────────────────┤
                                         └─ T30 ─ T31
```

Auth MVP evolution critical path: **T20 → T21 → T22 → T24 → T25 → T30 → T31**.

## Migration rules

- Never edit migrations already applied by T03 or implementation delivered through T01–T11.
- Add new numbered migrations and backfill before switching reads.
- Keep legacy `auth.v1` and `identities.role` during a compatibility window.
- Additive contract changes may stay in `v1`; breaking wire changes require `v2`.
- Compare old/new access results with metrics before removing the legacy path.
- Drop legacy columns only in a separately approved cleanup task after all consumers migrate. No cleanup task is included here.

## Definition of done

- Email/password registration supports `student`, `teacher`, `representative`, creates `PENDING_EMAIL`, confirms email within 24 hours and limits resend to 60 seconds.
- Staff identities are invitation-only, separate from users, password-only and always have at least one admin role.
- Access JWT and Gateway init expose deterministic `roles` and `permissions`; services still enforce permission and membership.
- Sessions expose device metadata, expire after 30 inactive days, cap at 10 and support user/admin revocation.
- Refresh reuse revokes the whole family. Raw refresh/action/OAuth tokens are never stored in logs, Kafka or DLQ.
- Яндекс ID, VK ID and Google work without silent linking by email; every account retains one login method.
- Password recovery, password change and email change enforce TTL, rate, attempt and session-revocation rules.
- Admin roles are configurable; `superadmin`, last-role and `*_edit`/`*_view` invariants hold under concurrency.
- Blocking, deletion and anonymization implement the approved lifecycle and preserve unlinkable immutable audit.
- Auth-owned mutations and required denials are audited at the service boundary. No distributed ACID transaction or shared audit DB is introduced.
- Auth events are published at-least-once to Kafka via outbox with stable `event_id` and per-identity ordering. Notification receives delivery secrets only through authenticated RPC.
- Narrow, full, integration and E2E tests pass in CI. Compose smoke uses Postgres, Kafka and optional Redis.

## Conventions for executing a task

- Update `contracts` before service code for public or inter-service API changes.
- Use branch `auth/Txx-short-name`, one PR per task, conventional commits.
- Ship behavior tests with every task. Concurrency invariants require real Postgres integration tests.
- Never replace an accepted ADR or product requirement silently. Raise a new ADR or explicit source correction.
