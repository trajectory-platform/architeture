# T04 — App bootstrap: config, logging, OTel, health, shutdown

**Size:** M · **Depends on:** T01 · **Phase:** 1

> Evolution note: the original T04 implementation exposed `AUTH_NATS_URL`. [ADR-003](../../../adr/ADR-003-apache-kafka.md) replaces it with `AUTH_KAFKA_BROKERS`; T16 updates wiring and readiness while preserving the delivered T04 history.

## Goal

Production-shaped process skeleton: the composition-root / DI layer (`internal/app`) that wires every dependency and runs the servers, fronted by a thin `cmd/auth/main.go`; typed config, structured logging, tracing/metrics providers, HTTP admin endpoints, graceful shutdown. Per [08 — Deployment & Observability](../../../architecture/08-deployment-observability.md) and conventions §2, §6–7.

## Scope

In:

- `internal/config`: `Config` struct + `Load()` from env. Keys: `AUTH_GRPC_ADDR`, `AUTH_HTTP_ADDR`, `AUTH_DB_DSN`, `AUTH_KAFKA_BROKERS`, `AUTH_REDIS_ADDR` (optional — empty disables blacklist), `AUTH_KEK` (base64, required), `AUTH_ACCESS_TTL` (default `10m`), `AUTH_REFRESH_TTL` (default `720h`), `AUTH_ARGON2_*` (memory/iterations/parallelism with safe defaults), `AUTH_LOG_LEVEL`, `AUTH_OTLP_ENDPOINT`. Fail-fast with a message naming the bad key.
- `pkg/logger`: `log/slog` constructor (JSON, `service` field, level) and context helpers (`WithContext`/`FromContext`, `trace_id` attach) — the logger propagates via `context`, not a passed field.
- `pkg/otelx`: tracer + meter provider setup (OTLP exporter, Prometheus registry for metrics), no-op when endpoint unset.
- `pkg/postgres`: `NewPool(ctx, dsn)` with sane pool limits + ping.
- `internal/transport/http`: HTTP server on `AUTH_HTTP_ADDR` with `/healthz` (always 200 when process alive), `/readyz` (DB ping, migrations applied, Kafka connected — wired up as components land), `/metrics`.
- `internal/app`: composition root `app.Run(ctx)` — compose config → logger → otel → pool → migrate (T03) → run servers; SIGINT/SIGTERM → graceful stop via `pkg/closer` (drain HTTP/gRPC → close pool → flush otel) with timeout. The only package that imports the concrete implementations.
- `cmd/auth/main.go`: thin entry point — parse config, call `app.Run`, exit non-zero on error. No logic.

Out: gRPC server itself (T15), Kafka connection (T16) — but `/readyz` API must allow registering readiness probes so they plug in later.

## Implementation notes

- Readiness: small `ready.Registry` (name → func(ctx) error) in transport/http; components register themselves in `internal/app` during wiring.
- No global logger/tracer; the tracer/providers pass via constructors, the logger via `context` (`logger.WithContext` in `app.Run`, `FromContext` at use sites).

## Acceptance criteria

- [ ] Service starts with valid env, exits non-zero with clear message on missing `AUTH_KEK` or bad `AUTH_DB_DSN`.
- [ ] `/healthz` 200, `/readyz` 503 until DB migrated/reachable then 200, `/metrics` serves Prometheus text.
- [ ] SIGTERM → clean shutdown, exit code 0, no goroutine-leak panic (verify with `goleak` in a start/stop test).
- [ ] Logs are valid JSON with `service:"auth"`; secrets never logged (grep test for KEK/DSN values in test output).
