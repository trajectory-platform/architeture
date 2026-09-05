# T11 — Register use case

**Size:** M · **Depends on:** T06, T07, T09 · **Phase:** 2

## Goal

`internal/service/register.go`: create identity + credentials + outbox event in **one transaction**, then issue the first token pair (no separate login needed). Flow: [auth-registration.puml](../../../diagrams/sequence/auth-registration.puml), spec: [auth.md — Регистрация](../../../services/auth.md).

> This task records the delivered ACTIVE-account baseline. T23 evolves registration to `PENDING_EMAIL`, adds `representative`, confirmation tokens and Notification delivery while preserving existing migrations.

## Scope

In:

- Input `RegisterInput{Email, Password, Role}`; output `TokenPairOutput{Access, AccessExp, Refresh, RefreshExp, IdentityID}` (shared output type — Login/Refresh reuse it).
- Validation in use case: email syntactically valid + lowercased before storage (citext makes lookup case-insensitive; normalize anyway for the event payload), password length 8..1024, role ∈ {student, teacher} — **admin not self-registerable**.
- Steps:
  1. `hasher.Hash(password)` (outside tx — argon2 is slow, don't hold a tx open through it).
  2. `txm.WithTx`: `identityRepo.Create` (maps email collision → `ErrEmailTaken`) + `outboxRepo.Add(aggregate_id=identity.ID, subject="trajectory.user.registered", payload=proto UserRegistered)`.
  3. After commit: create refresh family (new `family_id`, first token ACTIVE via `refreshRepo.Create`) + `issuer.IssueAccess`.
- Payload marshaled from `gen` proto `events.v1.UserRegistered` — marshaling happens in the `service` layer here by design exception: event payloads are contracts, not transport (note in code).

Out: relay publishing (T16), gRPC handler (T15), profile creation (Core's consumer).

## Implementation notes

- Token issuance after commit, not inside: a failed token signing must not roll back the registration — retryable by Login; return error with the identity created. Acceptance: decide + document behavior — simplest correct: if post-commit issuance fails, return `Internal`; user logs in. Test it.
- `registered_at` in event = identity.CreatedAt (clock port).

## Acceptance criteria

- [ ] Unit (fakes): happy path produces identity, credentials hash, outbox row in same tx scope, ACTIVE refresh in fresh family, valid access claims.
- [ ] Duplicate email → `ErrEmailTaken`, **no outbox row, no refresh tokens** (tx rolled back).
- [ ] Role `admin` rejected with validation error before any I/O.
- [ ] Outbox payload round-trips through proto unmarshal; subject exact: `trajectory.user.registered`.
- [ ] Integration (T06 repos): rollback leaves zero rows across all three tables.
