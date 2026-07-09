# T14 — Logout use case

**Size:** S · **Depends on:** T06 · **Phase:** 2

## Goal

`internal/service/logout.go`: revoke the refresh family of the presented token. Idempotent — repeated/unknown logout succeeds silently. Spec: [auth.md — Logout](../../../services/auth.md).

## Scope

In:

- Input `LogoutInput{RawToken}` → no output (error only for infrastructure failures).
- Steps: hash → `GetByHash`; not found → **success** (idempotent, nothing to leak); found (any status) → `RevokeFamily(family_id)`; optionally blacklist returned jtis (same `service.AccessBlacklist` interface as T13 — reuse, behind same config flag).
- No theft semantics here: logout of a ROTATED token is a stale-cookie scenario, still just revoke the family.

Out: cookie clearing (Gateway), access token invalidation beyond blacklist (short TTL handles it, [07](../../../architecture/07-security.md)).

## Acceptance criteria

- [ ] Active family → all tokens REVOKED.
- [ ] Second logout with same token → success, no error, no state change (assert no extra writes via fake).
- [ ] Garbage/unknown token → success.
- [ ] Infrastructure error (DB down) → error propagates (gRPC `Internal`/`Unavailable` later in T15) — idempotency ≠ swallowing real failures.
