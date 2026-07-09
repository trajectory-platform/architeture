# Trajectory — архитектурная документация

**Trajectory** — платформа индивидуальных онлайн-занятий с интерактивной доской (CRDT-синхронизация), видеосвязью с низкой задержкой (WebRTC / LiveKit), расписанием, биллингом, чатами и отчётностью.

Этот каталог содержит высокоуровневую документацию бэкенд-архитектуры: текстовые документы, архитектурные решения (ADR) и диаграммы (C4, sequence, use case) в формате PlantUML.

## Структура

### Архитектура

| Документ | Содержание |
|---|---|
| [01 — Обзор системы](architecture/01-overview.md) | Цели, ключевые инженерные вызовы, стек, архитектурные принципы |
| [02 — Сервисы и владение данными](architecture/02-services.md) | 5 микросервисов: зоны ответственности, владение данными, границы |
| [03 — Межсервисное взаимодействие](architecture/03-communication.md) | gRPC (sync) vs NATS JetStream (async), outbox, идемпотентность, трассировка |
| [04 — Саги и консистентность](architecture/04-sagas-and-consistency.md) | Booking-сага, отмена/завершение урока, recurring-бронирования, журнал прогресса |
| [05 — Realtime и вайтборд](architecture/05-realtime-whiteboard.md) | CRDT-синхронизация (Yjs relay на Go), протокол reconnect, LiveKit join flow |
| [06 — Данные и хранилища](architecture/06-data-storage.md) | Postgres per-service, ключевые таблицы (ledger, outbox, bookings), Redis, MinIO |
| [07 — Безопасность](architecture/07-security.md) | JWT/JWKS, refresh-токены, room JWT, LiveKit-токены, роли |
| [08 — Деплой и наблюдаемость](architecture/08-deployment-observability.md) | Docker-топология, Nginx, Prometheus/Grafana, метрики, логи |

### Документация сервисов

Детальная документация на сервис: API, владение данными (DDL + ER-диаграмма), события, ключевые потоки.

| Сервис | Кратко |
|---|---|
| [API Gateway](services/api-gateway.md) | REST-граница: JWT/JWKS, rate limiting, REST → gRPC, OpenAPI; stateless, без БД |
| [Auth Service](services/auth.md) | Идентичность: регистрация, login, ротация refresh-токенов, ключи и JWKS |
| [Core Education Service](services/core-education.md) | Предметное ядро: расписание, booking-сага, уроки, чаты, поддержка, отчёты, профили |
| [Billing Service](services/billing.md) | Деньги: append-only ledger, холды, идемпотентность |
| [Realtime Service](services/realtime.md) | WebSocket-хаб: Yjs-relay, чат комнат, presence; почти stateless |

### Архитектурные решения (ADR)

| ADR | Решение |
|---|---|
| [ADR-001](adr/ADR-001-microservices-granularity.md) | Гранулярность: 5 сервисов, чаты/отчёты/нотификации — внутри Core |
| [ADR-002](adr/ADR-002-crdt-sync-path-a.md) | CRDT-синхронизация: Go relay с персистентностью (vs Hocuspocus) |
| [ADR-003](adr/ADR-003-nats-jetstream.md) | Шина событий: NATS JetStream + transactional outbox (vs Redis Pub/Sub) |
| [ADR-004](adr/ADR-004-grpc-internal-rest-edge.md) | gRPC внутри, REST на границе; контракты в `proto/` под buf |

### Диаграммы

**C4:**

- [C1 — Контекст системы](diagrams/c4/c1-context.puml)
- [C2 — Контейнеры](diagrams/c4/c2-containers.puml)
- [C3 — Компоненты API Gateway](diagrams/c4/c3-api-gateway.puml)
- [C3 — Компоненты Auth Service](diagrams/c4/c3-auth.puml)
- [C3 — Компоненты Billing Service](diagrams/c4/c3-billing.puml)
- [C3 — Компоненты Core Education Service](diagrams/c4/c3-core-education.puml)
- [C3 — Компоненты Realtime Service](diagrams/c4/c3-realtime.puml)

**ER (базы данных):**

- [auth_db](diagrams/db/auth-db.puml)
- [billing_db](diagrams/db/billing-db.puml)
- [education_db](diagrams/db/education-db.puml)
- [realtime_db](diagrams/db/realtime-db.puml)

**Sequence:**

- [Регистрация: identity → профиль в Core](diagrams/sequence/auth-registration.puml)
- [Ротация refresh-токена и детект кражи](diagrams/sequence/auth-refresh-rotation.puml)
- [Booking-сага](diagrams/sequence/booking-saga.puml)
- [Вход в комнату урока](diagrams/sequence/lesson-join.puml)
- [Синхронизация вайтборда](diagrams/sequence/whiteboard-sync.puml)
- [Завершение урока: биллинг и прогресс](diagrams/sequence/lesson-completed-billing.puml)

**Use case:**

- [Сценарии использования](diagrams/usecase/usecases.puml)

## Как рендерить диаграммы

Диаграммы — обычные `.puml`-файлы PlantUML:

- **WebStorm / IntelliJ**: плагин «PlantUML Integration» — превью прямо в IDE.
- **CLI**: `plantuml docs/diagrams/**/*.puml` (нужна Java) — генерирует PNG рядом с файлами; `plantuml -checkonly ...` — только проверка синтаксиса.
- **Онлайн**: [plantuml.com/plantuml](https://www.plantuml.com/plantuml) — вставить содержимое файла.

C4-диаграммы используют стандартную библиотеку PlantUML (`!include <C4/C4_Container>`), которая входит в свежие версии PlantUML и работает офлайн. Если локальная версия PlantUML старая и stdlib-инклюды не находятся, замените их на удалённые:

```
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
```

## Конвенции

- Текст документации — русский; имена технологий, сервисов, событий (`booking.confirmed`), RPC-методов и полей БД — английские.
- На диаграммах: имена элементов — английские, описания — русские.
- ADR-формат: Контекст / Решение / Альтернативы / Последствия.
