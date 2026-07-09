# T15 — gRPC server: handlers, interceptors, error mapping

**Size:** M · **Depends on:** T02, T11, T12, T13, T14 · **Phase:** 3

## Goal

`internal/transport/grpc`: `auth.v1.AuthService` implementation wiring proto ⇄ `service` use cases, with the standard interceptor chain and the single business-error→gRPC error mapping. Caller is Gateway only ([auth.md — API](../../../services/auth.md)).

## Scope

In:

- `server.go`: handler struct holding the four use cases; proto ⇄ service DTO conversion (proto types stop here — conventions §2).
- `errors.go` — the only mapping point:

| Business error | gRPC code |
|---|---|
| `ErrEmailTaken` | `AlreadyExists` |
| `ErrInvalidCredentials`, `ErrTokenNotFound`, `ErrTokenExpired`, `ErrTokenReused`, `ErrIdentityBlocked` | `Unauthenticated` (uniform message "invalid credentials" / "invalid token" — no cause detail) |
| validation failures | `InvalidArgument` (field-level detail OK — not secret) |
| `ErrNoActiveKey`, unknown | `Internal` (generic message; real error logged server-side) |
| ctx deadline/cancel | `DeadlineExceeded` / `Canceled` passthrough |

- `pkg/grpcx` server interceptors, in order: panic recovery (→ `Internal`, stack logged) → OTel (`otelgrpc`) → request logging (method, code, duration, `trace_id`; **never** request payloads — they contain passwords/tokens) → deadline guard (no client deadline → log WARN; per [03](../../../architecture/03-communication.md) calls without deadlines are caller bugs worth surfacing).
- Input validation at handler edge: required fields non-empty, sizes bounded (email ≤ 320, password ≤ 1024, token ≤ 512) before hitting use cases.
- Server construction and registration wired in `internal/app` on `AUTH_GRPC_ADDR`; grpc-health-probe service (`grpc.health.v1`) registered; reflection enabled in dev builds only (config flag).
- Graceful drain on shutdown (`GracefulStop` with timeout fallback to `Stop`).

Out: REST mapping (Gateway plan), TLS (internal Docker network, [07](../../../architecture/07-security.md) — gRPC ports not published).

## Acceptance criteria

- [ ] In-process tests (bufconn, fake use cases): every mapping row verified; `Unauthenticated` message identical for wrong-email vs wrong-password vs reused-token.
- [ ] Logged entries on errors contain real cause + `trace_id`; response messages do not.
- [ ] Panic in a use case → `Internal`, server keeps serving.
- [ ] Payload fields (password, tokens) absent from logs — explicit assertion test on captured log output.
- [ ] Health service responds SERVING when `/readyz` conditions hold.
