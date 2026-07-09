# T18 — Integration & E2E test suite

**Size:** L · **Depends on:** T15, T16 · **Phase:** 4

## Goal

Service-level proof of the [auth.md](../../../services/auth.md) contract: full binary (or full wiring in-process) against real Postgres/NATS/Redis via testcontainers, exercising flows end-to-end through the gRPC API. This suite is the service's regression net — per-task tests already exist; here we test the composition.

## Scope

In — `services/auth/tests/e2e/` (build tag `e2e`, `make test-e2e`):

- Harness: start containers, run migrations, boot the service wiring (prefer in-process composition root reuse over exec'ing the binary — faster, still real repositories and infrastructure packages), gRPC client via local listener, JetStream test consumer, fixed-ish clock where injectable.
- Scenarios:
  1. **Register happy path**: TokenPair returned; access verifies against `/.well-known/jwks.json`; `user.registered` consumed from JetStream with correct payload; duplicate register → `AlreadyExists`.
  2. **Login**: correct creds → pair; wrong password and unknown email → identical `Unauthenticated` status+message; new family per login.
  3. **Refresh chain**: register → refresh ×3 → each old token dead, chain linked via `replaced_by`, exactly one ACTIVE.
  4. **Theft detection**: reuse token #2 after rotation → `Unauthenticated`; subsequent refresh with the *latest* token also fails (family revoked); with Redis enabled, jtis present in blacklist with TTL ≤ access TTL.
  5. **Concurrent refresh**: N=10 same token in parallel → ≤1 success, family fully revoked afterwards, DB invariant intact.
  6. **Logout**: refresh after logout fails; double logout OK.
  7. **JWKS + rotation**: force `Rotate` (test hook / direct manager call) → new `kid` in JWKS and in newly issued tokens; pre-rotation access still verifies against served set.
  8. **Crash-consistency (outbox)**: registration committed with relay stopped → row pending; relay started → event arrives exactly with `Nats-Msg-Id` = outbox id.
- CI job: e2e suite on PRs touching `services/auth/`, `pkg/`, or `proto/trajectory/auth`.

Out: load testing, Gateway REST behavior (cookie semantics live there), chaos beyond the listed crash test.

## Acceptance criteria

- [ ] All scenarios green and stable (3 consecutive CI runs, no retries/flakes; no sleep-based waits — poll with timeout).
- [ ] Total suite runtime < ~3 min (argon2 test params lowered via config — document that prod params differ).
- [ ] Failures print scenario step + relevant state dump (refresh family rows) for debuggability.
