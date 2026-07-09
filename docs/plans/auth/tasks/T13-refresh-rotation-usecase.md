# T13 — Refresh rotation & theft detection

**Size:** L · **Depends on:** T06, T09 · **Phase:** 2

## Goal

`internal/service/refresh.go`: rotation, reuse-as-theft detection with family revocation, optional jti blacklist. The security core of the service. Flow: [auth-refresh-rotation.puml](../../../diagrams/sequence/auth-refresh-rotation.puml), spec: [auth.md — Ротация refresh](../../../services/auth.md).

## Scope

In:

- Input `RefreshInput{RawToken}` → `TokenPairOutput`. Lookup by `sha256(raw)`.
- Decision table (exactly per sequence diagram):

| Found state | Action | Result |
|---|---|---|
| not found | — | `ErrTokenNotFound` → Unauthenticated |
| ACTIVE, expired | nothing (normal expiry ≠ theft, family survives) | `ErrTokenExpired` → Unauthenticated |
| ACTIVE, valid | tx: old→ROTATED(+`replaced_by`), insert new ACTIVE same family; issue access | new pair |
| ROTATED / REVOKED | **theft**: `RevokeFamily`; blacklist returned jtis in Redis (TTL = remaining access lifetime); WARN log + metric | `ErrTokenReused` → Unauthenticated |

- Race handling: `Rotate` CAS from T06 — if two requests race with the same ACTIVE token, loser gets `ErrTokenReused` from the repo. **Loser must then run the theft path too** (the diagram's DB-level guarantee): re-fetch, see ROTATED, revoke family. Implement as: on `ErrTokenReused` from Rotate → fall through to reuse branch.
- New refresh expiry: full `AUTH_REFRESH_TTL` from now (sliding sessions), per spec "новый ACTIVE в той же семье".
- `internal/repository/redis/blacklist.go` implementing the `service.AccessBlacklist` interface: `SET jti:<id> 1 EX <ttl>`. Config-optional: `AUTH_REDIS_ADDR` empty → no-op implementation (spec: blacklist опционален); blacklist write failure logged, **does not** fail the revocation (Postgres state is the source of truth; access TTL bounds the window).
- Identity status check: BLOCKED identity → treat as revoked (refuse refresh, `ErrInvalidCredentials`-style Unauthenticated).

Out: Gateway cookie handling, access-token verification by consumers.

## Acceptance criteria

- [ ] Unit tests cover every row of the decision table with fixed clock.
- [ ] Reuse path: ALL family tokens REVOKED (multi-rotation chain built first), jtis blacklisted with correct TTLs, metric incremented.
- [ ] Integration: 10 concurrent `Refresh` with the same raw token → exactly 1 new pair; family ends fully REVOKED (losers triggered theft path); DB invariant (≤1 ACTIVE per family) never violated.
- [ ] Expired-token path leaves family intact (subsequent refresh of the *current* ACTIVE token still works).
- [ ] Redis down → refresh-reuse revocation still succeeds, WARN logged.
