# T06 — Postgres repositories & transaction manager

**Size:** M · **Depends on:** T03, T05 · **Phase:** 1

## Goal

`internal/repository/postgres`: pgx implementations of all repo interfaces from T05, plus `pkg/postgres` tx manager. All SQL for the service lives here and nowhere else.

## Scope

In:

- `pkg/postgres/tx.go`: `TxManager.WithTx(ctx, fn)` — begins tx, stores it in ctx, commits/rolls back; repos use a `querier(ctx)` helper returning tx-from-ctx or pool. Nested `WithTx` reuses the outer tx.
- `identity_repo.go`: `Create(identity, passwordHash)` (inserts `identities` + `credentials`; maps unique violation on email → `models.ErrEmailTaken`), `GetByEmail` (JOIN credentials — single round trip for login), `GetByID`.
- `refresh_token_repo.go`:
  - `Create(token)`.
  - `GetByHash(hash)` → `models.ErrTokenNotFound` mapping.
  - `Rotate(oldID, newToken)` — single tx: `UPDATE ... SET status='ROTATED', replaced_by=$new WHERE id=$old AND status='ACTIVE'` + INSERT new; 0 rows updated → return `models.ErrTokenReused` (lost the race — another rotation already happened).
  - `RevokeFamily(familyID) (revokedJTIs []uuid, err)` — returns ids for access-blacklist use.
- `signing_key_repo.go`: `Create`, `GetActive`, `ListPublishable()` (status IN NEXT/ACTIVE/RETIRING), `UpdateStatus(kid, from, to)` — compare-and-set style (`WHERE status=$from`), 0 rows → `models.ErrInvalidKeyTransition`.
- `outbox_repo.go`: `Add(ctx, aggregateID, subject, payload)` — must run inside caller's tx.
- Unique-violation → business-error mapping helper using `pgconn.PgError` code `23505` + constraint name.

Out: relay polling of outbox (T16), Redis (T13), migrations (T03).

## Implementation notes

- Rely on DB invariants: don't pre-check email existence or family-active-count — attempt and map the violation. The partial unique index makes concurrent `Rotate` race-free (see note in [auth-refresh-rotation.puml](../../../diagrams/sequence/auth-refresh-rotation.puml)).
- No `SELECT *`; explicit column lists; `pgx.CollectOneRow`/`CollectRows` with row-to-struct helpers.

## Acceptance criteria

- [ ] Integration tests (testcontainers, `//go:build integration`) cover: email-unique mapping; rotate happy path; **two concurrent Rotate calls with same old token → exactly one succeeds, other gets `ErrTokenReused`**; RevokeFamily revokes all and returns ids; CAS `UpdateStatus` rejects wrong `from`.
- [ ] `WithTx` rollback on error leaves no partial rows (test: Create identity then fail → no identity, no credentials).
- [ ] No SQL outside `repository/postgres` (lint/grep check).
