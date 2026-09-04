# T17 — Observability polish

**Size:** S · **Depends on:** T15, T16 · **Phase:** 4

## Goal

Close the gap between "interceptors exist" and "service is operable": domain metrics, log field audit, alert-ready signals per [08 — Deployment & Observability](../../../architecture/08-deployment-observability.md).

## Scope

In:

- Metrics (Prometheus via OTel meter), on top of RED-per-method from `grpcx`:
  - `auth_refresh_reuse_total` — theft detections (security signal, alertable).
  - `auth_logins_total{result="ok|invalid|blocked"}` (cardinality-safe labels only).
  - `auth_registrations_total`.
  - `auth_outbox_unpublished` gauge + `auth_outbox_oldest_age_seconds` (T16 exposes; verify exported).
  - `auth_active_signing_key_age_seconds` gauge; `auth_jwks_requests_total`.
  - `auth_token_issue_duration_seconds` histogram (argon2 + sign cost visibility).
- Log audit: every WARN/ERROR carries `trace_id`; security events (reuse detection, blocked-login attempt) at WARN with `identity_id`, `family_id` — never email/token. One pass over all packages with a checklist.
- Trace completeness: REST-edge deadline propagates into SQL spans (pgx OTel tracer) and outbox publish spans; refresh flow renders as one trace: grpc → sql → (theft path) redis.
- Grafana: add auth panels to the System health dashboard JSON in `deploy/` (RED, reuse rate, outbox depth, key age) — minimal, no new dashboard.

Out: alert rules deployment (ops-level; document suggested thresholds in this task's PR description), Loki pipeline (infra plan).

## Acceptance criteria

- [ ] `/metrics` exposes every listed metric with correct types; integration test scrapes and asserts presence after exercising flows.
- [ ] Refresh-reuse E2E produces: WARN log with family id + `auth_refresh_reuse_total` increment + connected trace.
- [ ] Grep-test: no `email=`, `password`, `token=` field keys in any log call site.
- [ ] pgx instrumented — SQL spans visible under gRPC span in a local trace dump.
