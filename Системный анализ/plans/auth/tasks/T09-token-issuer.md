# T09 — Token issuer

**Size:** M · **Depends on:** T08 · **Phase:** 1

## Goal

`internal/infra/token` implementing the `service.TokenIssuer` interface: access JWT signing with the current ACTIVE key, and refresh-token raw/hash generation. Spec: [auth.md — Login](../../../services/auth.md), [07 — Security](../../../architecture/07-security.md).

> This task records the delivered single-role token baseline. T22 evolves claims to `identity_type`, `roles`, `permissions` and `session_id` per ADR-007.

## Scope

In:

- `IssueAccess(ctx, identity) (token string, jti uuid, expiresAt time, err error)`:
  - Claims: `sub` (identity id), `role`, `jti` (uuid v4), `iat`, `exp` (= now + `AUTH_ACCESS_TTL`, ~10m), `iss` (`trajectory-auth`), `aud` (`trajectory`).
  - Header: `alg: EdDSA`, `kid` of ACTIVE key (from `keys.Manager.Active`).
  - Only role goes in — **no ACLs/permissions in token** (ownership checked by data owners, [07](../../../architecture/07-security.md)).
- `NewRefreshToken() (raw string, hash []byte, err error)`:
  - Raw: 32 bytes `crypto/rand`, base64url — opaque string, **not** a JWT (nothing to parse client-side; spec stores only hash).
  - Hash: `sha256(raw)` — the only thing persisted ([auth.md invariant](../../../services/auth.md): DB dump leaks no usable refresh tokens).
- Verification helper `ParseAccess` for tests only (verify against a given public JWK) — production verification belongs to Gateway/Realtime/Notification, not Auth.

Out: refresh token persistence/rotation logic (T13), JWKS (T10).

## Acceptance criteria

- [ ] Issued JWT verifies with the ACTIVE public JWK; `kid` header matches; all claims present and correct with fixed clock.
- [ ] After key rotation (swap fake manager's ACTIVE), new tokens carry new `kid`; old token still verifies with old (RETIRING) public key.
- [ ] Refresh raw tokens: 1M-iteration uniqueness sanity test in CI-cheap form (e.g. 10k, no dupes), correct length/alphabet; hash deterministic.
- [ ] `ErrNoActiveKey` propagated when manager has no key (issuer never signs unsigned/none-alg fallback).
