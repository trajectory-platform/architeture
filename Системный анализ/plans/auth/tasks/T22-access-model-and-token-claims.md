# T22 — Access model and token claims

**Size:** L · **Depends on:** T09, T20, T21 · **Phase:** 6

## Goal

Replace the runtime single-role model with user/staff access context and issue JWT claims required by Gateway and client `init`.

## Scope

In:

- Domain models and repositories for identity subtype, learning roles, staff roles and effective permissions.
- Dual-read verification against legacy `identities.role`; metrics expose mismatches before cutover.
- Token issuer claims: `identity_type`, sorted/deduplicated `roles`, sorted/deduplicated `permissions`, `session_id` plus existing standard claims.
- `GetAccessContext` use case for Gateway `init`, including status, email confirmation and active cabinet.
- Permission evaluator for Auth-owned operations. Data-owner membership remains outside Auth.
- Cache only if invalidation is explicit. Correct DB result takes priority over latency in MVP.

## Acceptance criteria

- [ ] A user with `student` + `teacher` receives both roles and no admin permissions.
- [ ] Staff permissions equal the union of all assigned admin roles.
- [ ] User and staff roles cannot coexist.
- [ ] Claims are deterministic and bounded; benchmark records worst-case token size for the full MVP permission catalog.
- [ ] Changing roles appears on next refresh; revoking sessions forces immediate re-login.
- [ ] Legacy rows return equivalent access throughout migration.

