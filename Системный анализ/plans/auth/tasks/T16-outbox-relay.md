# T16 — Outbox relay → Apache Kafka

**Size:** M · **Depends on:** T02, T03, T04 · **Phase:** 3

## Goal

Reusable relay in `pkg/outbox` plus Auth wiring: publish committed outbox rows to Kafka with at-least-once delivery, stable event identity, per-aggregate ordering and safe concurrent replicas.

## Scope

In:

- `pkg/kafkax`: Kafka producer construction from `AUTH_KAFKA_BROKERS`; idempotent producer enabled; required acknowledgements `all`; bounded retries and backoff; TLS/SASL options for non-local environments.
- Auth publishes to `trajectory.auth.events.v1`. The service does not create or alter production topics. A development-only provisioner may create local topics with explicit partitions and replication factor.
- `pkg/outbox.Relay`:
  - Select a batch with `SELECT ... FOR UPDATE SKIP LOCKED` inside a transaction.
  - Publish each row with topic, message key, payload, `event_id`, content type, schema version, trace context and outbox id in headers while the row remains locked.
  - Use `aggregate_id`/`identity_id` as message key so one identity stays ordered within one partition.
  - Mark `published_at` after broker acknowledgement, then commit the transaction.
  - On publish failure, leave the row pending, apply bounded exponential backoff and retry.
  - Poll interval defaults to 200 ms; optional wake channel reduces latency after commit.
- Stable `event_id` is generated before the business transaction and stored with payload. Re-publishing the same outbox row keeps the same `event_id`.
- OTel spans and metrics: unpublished depth, oldest pending age, published/error counters, retry count and broker latency.
- `internal/app`: start relay only after migrations, register shutdown drain and add Kafka connectivity/readiness probe.

Out: consumer implementations, topic retention policy deployment, schema registry deployment.

## Implementation notes

- Kafka producer idempotence reduces duplicates inside one producer session but does not remove crash window between broker acknowledgement and `published_at`. Consumers must deduplicate by `event_id`.
- Keep batches bounded because the database transaction remains open until Kafka acknowledges the batch. `FOR UPDATE SKIP LOCKED` prevents another relay from selecting the same pending rows.
- Ordering guarantee is per Kafka message key, not global. All events for one identity use the same key.
- Relay never publishes rolled-back domain state because outbox insert belongs to the domain transaction.

## Acceptance criteria

- [ ] Integration with real Postgres and Kafka: committed `user.registered` arrives on `trajectory.auth.events.v1` with correct key, payload and stable `event_id`; `published_at` is set.
- [ ] Crash after broker acknowledgement but before mark causes redelivery with the same `event_id`; an idempotent test consumer applies it once.
- [ ] Two relay instances drain 1,000 rows without losing rows or blocking each other beyond bounded batches.
- [ ] Events for one identity preserve outbox order in one partition.
- [ ] Kafka outage accumulates rows without process crash; recovery drains backlog.
- [ ] Rolled-back business transaction produces no Kafka message.
- [ ] Logs and headers contain no password, raw token, delivery URL, KEK or private key.
