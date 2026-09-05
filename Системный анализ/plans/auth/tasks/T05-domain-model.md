# T05 — Models & service interfaces

**Size:** S · **Depends on:** T01 · **Phase:** 1

## Goal

Pure models layer (`internal/models`) + the service interfaces (`internal/service`) that all later tasks implement against. Zero I/O, stdlib-only.

> This task records the initial one-role domain model. ADR-007 replaces it with user/staff subtypes, role bindings, permissions and explicit sessions. T21–T22 perform that evolution without erasing the delivered baseline.

## Scope

In — `internal/models`:

- `identity.go`: `Identity{ID, Email, Role, Status, CreatedAt}`; `Role` (`student|teacher|admin`), `IdentityStatus` (`ACTIVE|BLOCKED`) as typed strings with `Valid()`.
- `refresh_token.go`: `RefreshToken{ID, IdentityID, FamilyID, TokenHash, Status, ReplacedBy, ExpiresAt, CreatedAt}`; `RefreshStatus` (`ACTIVE|ROTATED|REVOKED`); methods `Expired(now)`, `Reused()` (status != ACTIVE).
- `signing_key.go`: `SigningKey{KID, Alg, PublicJWK, PrivateKey, Status, NotAfter, CreatedAt}`; `KeyStatus` (`NEXT|ACTIVE|RETIRING|RETIRED`) + allowed-transition method `CanTransitionTo`.
- `errors.go`: `ErrEmailTaken`, `ErrInvalidCredentials`, `ErrIdentityBlocked`, `ErrTokenNotFound`, `ErrTokenExpired`, `ErrTokenReused`, `ErrNoActiveKey`, `ErrInvalidKeyTransition`.

In — `internal/service` interfaces (consumed by use cases, defined consumer-side; implemented in T06–T09):

- `IdentityRepo` (`Create` — identity + credentials in one call, `GetByEmail` — joined with credentials, `GetByID`), `RefreshTokenRepo` (`Create`, `GetByHash`, `Rotate`, `RevokeFamily`), `SigningKeyRepo` (`Create`, `GetActive`, `ListPublishable`, `UpdateStatus`), `OutboxRepo` (`Add`).
- `TxManager` (`WithTx(ctx, fn) error`), `PasswordHasher` (`Hash`, `Verify`, `DummyVerify`), `TokenIssuer` (`IssueAccess(identity) (jwt, jti, exp, error)`, `NewRefreshToken() (raw, hash, error)`), `Clock` (`Now()`), `AccessBlacklist` (`Revoke(ctx, jti, ttl)`).

## Implementation notes

- Interfaces defined on the consumer side (`service`), narrow per use case need — don't grow god-interfaces. Split later if a use case needs less.
- Document invariants as godoc on types: "exactly one ACTIVE per family, enforced by partial unique index"; "token stored only as sha256".

## Acceptance criteria

- [ ] `internal/models` imports: stdlib only (lint depguard rule added).
- [ ] Unit tests for status validation, transitions table (`NEXT→ACTIVE` ok, `RETIRED→ACTIVE` rejected, etc.), `Expired`/`Reused`.
- [ ] All exported symbols documented.
