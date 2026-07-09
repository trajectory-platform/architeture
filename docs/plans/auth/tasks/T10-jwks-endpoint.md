# T10 — JWKS HTTP endpoint

**Size:** S · **Depends on:** T04, T08 · **Phase:** 2

## Goal

`GET /.well-known/jwks.json` on the HTTP server from T04, serving public keys with status NEXT/ACTIVE/RETIRING. Consumers (Gateway, Realtime) cache by TTL and refetch on unknown `kid` ([auth.md — HTTP](../../../services/auth.md), [07 — Security](../../../architecture/07-security.md)).

## Scope

In:

- Handler in `internal/transport/http/jwks.go`: response `{"keys":[ ...public JWKs... ]}` (RFC 7517), each entry includes `kid`, `kty`, `alg`, `use:"sig"`.
- Source: `keys.Manager.Publishable(ctx)` (T08) — RETIRED keys excluded by construction.
- `Cache-Control: public, max-age=<AUTH_JWKS_MAX_AGE>` (default 300s) + `ETag` (hash of key set) with `If-None-Match` → 304.
- Method guard (GET/HEAD only), no auth required (public keys).

Out: consumer-side caching (Gateway/Realtime plans), key lifecycle (T08).

## Acceptance criteria

- [ ] httptest suite: empty set (pre-bootstrap) → 200 with empty `keys` array; ACTIVE+NEXT+RETIRING all present; RETIRED absent.
- [ ] Response parses with `jwx` JWKS parser; signature made by T09 verifies against the served set.
- [ ] ETag stable across calls with unchanged keys; changes after rotation; 304 path works.
- [ ] Never contains `d`/private material — explicit test asserting only public fields serialized.
