# T02 — Proto contracts: `auth.v1` + `events.v1`

**Size:** S · **Depends on:** T01 · **Phase:** 0

## Goal

gRPC contract for Auth and the `user.registered` event payload, under buf with codegen into `gen/`. Single source of truth per [ADR-004](../../../adr/ADR-004-grpc-internal-rest-edge.md).

> **Принятая эволюция:** [ADR-006](../../../adr/ADR-006-auth-api-evolution-and-password-reset.md) заменяет исходную форму контракта ниже в части response wrappers для `Register` / `Login` / `Refresh`, общего `ROLE_ADMIN`, password reset RPC и `UserRegistered.event_id`. Этот brief сохраняет исходный срез T02; актуальным источником формы API остаётся versioned contract в репозитории `contracts`.

## Scope

In:

- `proto/buf.yaml` (lint: STANDARD; breaking: FILE) and `proto/buf.gen.yaml` (go + go-grpc plugins, output `gen/go`).
- `proto/trajectory/auth/v1/auth.proto`:

```proto
service AuthService {
  rpc Register(RegisterRequest) returns (TokenPair);   // AlreadyExists if email taken
  rpc Login(LoginRequest)       returns (TokenPair);   // Unauthenticated, no detail which part wrong
  rpc Refresh(RefreshRequest)   returns (TokenPair);   // Unauthenticated: expired/revoked/reused
  rpc Logout(LogoutRequest)     returns (LogoutResponse); // idempotent
}
```

  - `RegisterRequest{email, password, role}` — role enum `ROLE_STUDENT|ROLE_TEACHER` (admin not self-registerable).
  - `LoginRequest{email, password}`; `RefreshRequest{refresh_token}`; `LogoutRequest{refresh_token}`.
  - `TokenPair{access_token, refresh_token, access_expires_at, refresh_expires_at}` (timestamps as `google.protobuf.Timestamp`).
  - Proto comments mark `Refresh`/`Logout` idempotency semantics per [03 — Communication](../../../architecture/03-communication.md).
- `proto/trajectory/events/v1/events.proto`: `UserRegistered{user_id, email, role, registered_at}` with comment `subject: trajectory.user.registered`.
- `make generate` runs `buf generate`; generated code committed to `gen/go/...`.
- CI: add `buf lint` and `buf breaking --against` main.

Out: education/billing protos (other services' plans), gateway REST mapping.

## Acceptance criteria

- [ ] `buf lint` clean; `buf generate` deterministic (re-run produces no diff).
- [ ] Generated Go stubs compile and are importable from `auth-service` through a versioned `contracts` module dependency.
- [ ] CI fails on a deliberately breaking change (verify once locally, don't commit).
- [ ] No floats, no required-feeling optionals; field numbering leaves room (reserve nothing yet, start clean).
