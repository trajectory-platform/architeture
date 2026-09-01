# T19 — docker-compose integration & smoke runbook

**Size:** S · **Depends on:** T18 · **Phase:** 4

## Goal

Auth runs as part of the full local stack ([08 — Топология](../../../architecture/08-deployment-observability.md)): one `docker compose up` from zero to a working service, with a scripted smoke check.

## Scope

In:

- `deploy/docker-compose.yml`: finalize `auth` service — env from `.env`, `depends_on` with `condition: service_healthy` (postgres-auth, nats), healthcheck hitting `/readyz`, restart policy, resource notes. Redis optional → auth must start fine without it (blacklist disabled).
- `.env.example`: every `AUTH_*` var with safe dev defaults and comments; KEK generation one-liner documented.
- Smoke script `deploy/smoke/auth.sh` (bash; runs in CI and locally): wait for healthy → `grpcurl` Register → Login → Refresh → reuse-detect (expect failure) → Logout → fetch JWKS; assert outbox drained (depth metric = 0) and `user.registered` visible via `nats` CLI consumer peek.
- `auth-service/README.md`: how to run, env reference table, smoke instructions, links to [auth.md](../../../services/auth.md) and the [plan](../README.md).
- Prometheus scrape config in `deploy/` includes auth `/metrics`.

Out: staging/prod manifests, TLS (Nginx terminates at edge; auth not exposed), Gateway wiring.

## Acceptance criteria

- [ ] `docker compose up -d && deploy/smoke/auth.sh` passes on a clean machine (CI job proves it).
- [ ] `docker compose down -v && up` → migrations re-apply, key bootstrap runs, smoke passes again.
- [ ] Killing auth container mid-run → restart recovers without manual steps (outbox backlog drains).
- [ ] No service ports except Nginx-published ones exposed in prod-shaped compose profile; dev profile may expose for tooling.
