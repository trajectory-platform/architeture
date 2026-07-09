# T12 — Login use case

**Size:** M · **Depends on:** T06, T07, T09 · **Phase:** 2

## Goal

`internal/service/login.go`: password check with uniform failure behavior, new refresh family, token pair. Spec: [auth.md — Login](../../../services/auth.md).

## Scope

In:

- Input `LoginInput{Email, Password}` → `TokenPairOutput`.
- Steps:
  1. `identityRepo.GetByEmail` (joined with credentials).
  2. **Unknown email** → `hasher.DummyVerify(password)` then return `ErrInvalidCredentials` — comparable response time for wrong-email vs wrong-password; no enumeration oracle.
  3. Known email → `hasher.Verify`; mismatch → `ErrInvalidCredentials` (same error — caller cannot distinguish).
  4. `identity.Status == BLOCKED` → also `ErrInvalidCredentials` externally (do **not** leak blocked status); log the real reason at WARN with identity id. Check status *after* password verify so timing stays uniform.
  5. Success: new family (`family_id = uuid`), first refresh ACTIVE, `IssueAccess`.
  6. Opportunistic rehash: if `NeedsRehash` (T07), update credentials hash best-effort (failure logged, not returned).
- No transaction needed beyond single statements (refresh insert is one row); rehash separate statement.

Out: rate limiting (Gateway-owned, [auth.md](../../../services/auth.md)), lockout policies (not in MVP spec).

## Acceptance criteria

- [ ] Wrong email and wrong password produce the **same error value**; test asserts both paths call argon2 work (fake hasher records DummyVerify/Verify invocations).
- [ ] Blocked identity: correct password → `ErrInvalidCredentials` outward; WARN log contains real cause.
- [ ] Success: fresh family id (≠ any previous), exactly one ACTIVE token in it, access claims carry correct `sub`/`role`.
- [ ] Each login creates a **new** family — prior families untouched (multi-device sessions coexist).
- [ ] Rehash path: weak-param hash upgraded on successful login; upgrade failure doesn't fail login.
