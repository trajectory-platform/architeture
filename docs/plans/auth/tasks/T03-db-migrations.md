# T03 — auth_db schema migrations

**Size:** S · **Depends on:** T01 · **Phase:** 0

## Goal

Full auth_db schema as goose migrations, embedded and applied on service start. Source DDL: [auth.md — Данные](../../../services/auth.md), ER: [auth-db.puml](../../../diagrams/db/auth-db.puml).

## Scope

In — `services/auth/migrations/`:

- `0001_extensions.sql` — `CREATE EXTENSION IF NOT EXISTS citext;`
- `0002_identities.sql` — `identities` per spec + `CHECK (role IN ('student','teacher','admin'))`, `CHECK (status IN ('ACTIVE','BLOCKED'))`.
- `0003_credentials.sql` — `credentials` (separate table by design: hash not pulled by normal SELECTs; OAuth identities may have no password).
- `0004_refresh_tokens.sql` — `refresh_tokens` + status CHECK (`ACTIVE|ROTATED|REVOKED`) + **partial unique index** `refresh_one_active_per_family ON refresh_tokens(family_id) WHERE status='ACTIVE'` + index on `identity_id`, index on `expires_at` (cleanup queries).
- `0005_signing_keys.sql` — `signing_keys` + status CHECK (`NEXT|ACTIVE|RETIRING|RETIRED`) + partial unique index ensuring **at most one ACTIVE key**: `ON signing_keys(status) WHERE status='ACTIVE'`.
- `0006_outbox.sql` — `outbox` + partial index `outbox_unpublished ON outbox(id) WHERE published_at IS NULL`.
- Each with `-- +goose Down`.
- `internal/repository/postgres/migrate.go` — `embed.FS` + `Migrate(ctx, pool)` called from `internal/app` before serving; readiness fails until migrated.

Out: repositories (T06), seed data (none — first signing key is created by key manager, T08).

## Implementation notes

- `token_hash bytea UNIQUE`; lookups by hash → unique index covers it.
- All timestamps `timestamptz`. PKs `uuid` supplied by service code (uuid v7), not DB-generated — keeps id generation in the service, not the DB.
- The one-ACTIVE-key index is an addition to the spec DDL: makes the key-manager invariant DB-enforced, same pattern as refresh families. Note it in PR description.

## Acceptance criteria

- [ ] `goose up` from zero and `goose down` to zero both succeed on empty Postgres.
- [ ] Re-running up is a no-op (idempotent).
- [ ] Integration test (testcontainers): inserting two ACTIVE tokens in one family fails with unique violation; two ACTIVE signing keys fail likewise.
- [ ] Service start on fresh DB applies migrations automatically; `/readyz` (once T04 lands) gated on it.
