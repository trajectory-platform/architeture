# T16 — Outbox relay → NATS JetStream

**Size:** M · **Depends on:** T02, T03, T04 · **Phase:** 3

## Goal

Reusable relay in `pkg/outbox` (other services share the same outbox schema — [03 — Communication](../../../architecture/03-communication.md)) + auth wiring: publish unpublished rows to JetStream, at-least-once, replica-safe.

## Scope

In:

- `pkg/natsx`: JetStream connect helper; stream ensure/bind for `TRAJECTORY` stream covering `trajectory.>` subjects (idempotent `CreateOrUpdateStream` — dev convenience; prod streams may be provisioned externally, keep it config-flagged).
- `pkg/outbox.Relay`:
  - Loop: poll batch (`SELECT ... WHERE published_at IS NULL ORDER BY id LIMIT $batch FOR UPDATE SKIP LOCKED`) inside a tx → for each row `js.Publish(subject, payload)` with `Nats-Msg-Id = outbox.id` (JetStream dedup window makes redelivery cheap for consumers) → wait ack → `UPDATE published_at = now()` → commit.
  - Publish failure: rollback row's update, backoff (exponential, capped), keep ordering per batch simple — abort batch on first failure, retry next tick.
  - Tick interval config (default 200ms) + immediate wake channel (use cases may ping it after commit; optional, polling alone is correct).
  - OTel: trace context into msg headers; metrics: outbox depth gauge (`published_at IS NULL` count, sampled), publish counter/errors, oldest-unpublished age ([08 — alerts](../../../architecture/08-deployment-observability.md): "глубина outbox растёт").
- Auth wiring in `internal/app`: relay goroutine with teardown registered on `pkg/closer`; NATS connectedness registered in `/readyz`.

Out: consumers (Core's plan), event payload creation (T11 writes rows).

## Implementation notes

- `FOR UPDATE SKIP LOCKED` makes concurrent replicas safe — two relays never double-hold a row; crash between publish and `published_at` → republish → consumer idempotency + `Nats-Msg-Id` dedup absorb it. Guarantee: **at-least-once, after commit — обязательно**.
- Relay must not start before migrations applied (ordering in `internal/app`).

## Acceptance criteria

- [ ] Integration (testcontainers Postgres+NATS): committed outbox row arrives on `trajectory.user.registered` durable consumer; `published_at` set.
- [ ] Kill relay between publish and mark (test hook) → restart republishes; `Nats-Msg-Id` equal both times.
- [ ] Two relay instances over 1k rows → every row published, no row published by both (assert by ack count with dedup window disabled, or by `published_at` write conflicts = 0).
- [ ] NATS down: rows accumulate, depth metric grows, no crash; recovery drains backlog.
- [ ] Rolled-back business tx → its outbox row never existed → nothing published (covered with T11 integration).
