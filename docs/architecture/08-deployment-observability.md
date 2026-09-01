# 08 — Деплой и наблюдаемость

## Топология (Docker Compose, MVP)

Все компоненты — контейнеры в одной Docker-сети; наружу опубликованы только Nginx (443) и медиа-порты LiveKit.

```
                        интернет
                           │
              ┌────────────┴───────────┐
              │ 443 (HTTPS/WSS)        │ UDP/TCP медиа-порты
              ▼                        ▼
           [ Nginx ]               [ LiveKit ]
              │                        │ webhooks (HTTP, внутр.)
   ┌──────────┼──────────────┐         │
   │ /api/*   │ /ws/*        │         │
   ▼          ▼              │         │
[ API Gateway ] [ Realtime ×N ]        │
   │ gRPC         │ gRPC │ Redis       │
   ▼              ▼      ▼             │
[ Auth ] [ Core Education ]◄───────────┘
              │        ▲
              │ gRPC   │ events
              ▼        ▼
        [ Billing ] [ NATS JetStream ] ──► [ Notification ]

хранилища: postgres-auth, postgres-education, postgres-billing,
           postgres-realtime, postgres-notification, redis, minio
мониторинг: prometheus, grafana, loki (+ otel-collector)
```

Маршрутизация Nginx:

| Путь | Назначение |
|---|---|
| `/api/*` | API Gateway (REST) |
| `/ws/*` | Realtime Service (WebSocket upgrade, sticky по `lesson_id` не нужен — инстансы равнозначны) |
| `/livekit/*` | LiveKit signaling (или отдельный поддомен) |
| `/storage/*` | MinIO (presigned-доступ) |

Масштабирование: Realtime и Gateway — stateless, масштабируются репликами за Nginx (Realtime координируется через Redis, см. [05](05-realtime-whiteboard.md)). Core/Billing/Auth/Notification — реплики свободно; состояние находится в их Postgres, конкуренция outbox-relay и JetStream-консьюмеров решена `FOR UPDATE SKIP LOCKED` и durable consumers.

Сборка: каждый сервис — отдельный репозиторий с собственным `Dockerfile`-паттерном (multi-stage, distroless) и версионированной зависимостью на `contracts`. Отдельный deployment-репозиторий собирает конкретные версии образов в локальный Compose-стек; это же описание служит базой для прода (Compose → Swarm/K8s при необходимости, без изменения архитектуры).

## Наблюдаемость

Три сигнала, общая шина — OpenTelemetry Collector.

### Трассировка

- OTel-интерсепторы на всех gRPC-клиентах/серверах + HTTP-мидлвара Gateway; trace context — через gRPC metadata и заголовки.
- `trace_id` рождается на REST-границе и доезжает до SQL и публикаций в JetStream (subject + msg headers) — асинхронные цепочки (`booking → hold → event → capture`) связаны в один трейс.

### Метрики (Prometheus)

`/metrics` на каждом сервисе. Продуктово-критичные метрики:

| Область | Метрики |
|---|---|
| Комнаты уроков | активные комнаты, участники, длительность; разрывы WS / мин |
| WebSocket | открытые соединения по инстансам, latency broadcast'а board-апдейтов, размер очередей отправки |
| Доска | апдейты/сек per-room, отставание компакции (updates since last snapshot), размер логов |
| Медиа | LiveKit-метрики: packet loss, jitter, bitrate (LiveKit экспортирует Prometheus) |
| Бронирования | счётчик FAILED-бронирований по причинам (insufficient_funds, slot_conflict), **возраст старейшей PENDING-брони** (зависшие саги, см. [04](04-sagas-and-consistency.md)) |
| События | лаг durable-консьюмеров JetStream, глубина outbox (`published_at IS NULL`), redelivery rate, dead-letter счётчик |
| gRPC | RED per-method: rate, errors, duration (p50/p95/p99) |
| Инфраструктура | Postgres (соединения, репликация), Redis, MinIO, NATS — стандартные экспортеры |

### Алерты (минимальный набор)

- Возраст PENDING-брони > 5 мин — зависшая сага.
- Глубина outbox растёт > N мин — relay не публикует.
- Лаг консьюмера JetStream > порога — Billing/reports отстают от событий.
- Сообщения в dead-letter — ошибка обработки, требует человека.
- Error rate gRPC > порога; p99 `PlaceHold` > порога.
- Массовые разрывы WS на одном инстансе — деградация realtime.

### Логи

Структурированные JSON-логи (log/slog) во stdout → сборщик (Loki/ELK). Каждая запись несёт `trace_id`, `service`, `user_id` (если есть), `lesson_id`/`booking_id` (если применимо) — лог склеивается с трейсом. PII и содержимое сообщений в логи не пишутся.

### Дашборды Grafana

1. **System health** — RED по сервисам, ресурсы, состояние зависимостей.
2. **Lessons live** — активные комнаты, WS-соединения, качество медиа, ошибки досок.
3. **Money** — брони по статусам, FAILED-причины, лаг capture, глубина outbox.
4. **Events** — стримы JetStream, лаги консьюмеров, dead-letter.

## Окружения и CI

- **CI**: build + tests, `buf lint` и `buf breaking` на `proto/` (контракт не ломается незаметно), сборка образов, `plantuml -checkonly` для диаграмм документации.
- **Локально**: полный `docker compose up` (включая LiveKit dev-mode, MinIO, NATS) — окружение разработчика идентично по составу проду.
- **Staging/prod**: тот же состав, отличия — только конфигурация (env), TLS-сертификаты и внешние тома данных.
