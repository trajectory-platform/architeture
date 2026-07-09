# 02 — Сервисы и владение данными

Система состоит из пяти сервисов. Принцип границ: сервис = зона владения данными + протокольная роль. Диаграммы: [C2 — контейнеры](../diagrams/c4/c2-containers.puml), [C3 — Core](../diagrams/c4/c3-core-education.puml), [C3 — Realtime](../diagrams/c4/c3-realtime.puml).

## Сводная таблица

| Сервис | Владеет данными | API наружу | API внутрь (gRPC) | Публикует события | Потребляет события |
|---|---|---|---|---|---|
| **API Gateway** | — (stateless) | REST (OpenAPI) | — (только клиент gRPC) | — | — |
| **Auth Service** | identities, credentials, refresh tokens, ключи JWKS | JWKS endpoint (через gateway) | `auth.v1`: Register, Login, Refresh, Logout | `user.registered` | — |
| **Core Education** | availability, slots, bookings, lessons, чаты/сообщения, заметки, журнал прогресса, тикеты поддержки | LiveKit webhooks (HTTP) | `education.v1`: слоты, бронирования, уроки, чаты, отчёты; `JoinLesson` | `booking.confirmed`, `booking.cancelled`, `lesson.started`, `lesson.completed`, `progress.milestone_reached` | `payment.captured` |
| **Billing Service** | ledger_entries, holds, idempotency keys | — | `billing.v1`: PlaceHold, ReleaseHold, CaptureHold, GetBalance, TopUp | `payment.captured`, `hold.released` | `booking.cancelled`, `lesson.completed` |
| **Realtime Service** | почти ничего (membership/presence — в Redis; лог Yjs-апдейтов — в своей Postgres/MinIO) | WebSocket (WSS) | — (клиент gRPC к Core) | — | `lesson.started` (опционально, для пре-создания комнат) |

## API Gateway

Единственная REST-точка входа для SPA. Обязанности:

- Терминирует REST, транслирует в gRPC-вызовы внутренних сервисов.
- Валидирует JWT локально по JWKS (без похода в Auth на каждый запрос).
- Генерирует `Idempotency-Key`-прокидку: принимает заголовок от клиента, передаёт в gRPC metadata.
- Rate limiting (Redis), CORS, единый формат ошибок.
- Публикует OpenAPI-спеку — из неё генерируется TypeScript-клиент фронтенда.

Никакой бизнес-логики. Если в обработчике гейтвея появляется `if` про предметную область — он переезжает в Core.

## Auth Service

Владеет идентичностью: регистрация, логин, refresh-токены, роли (student / teacher / admin).

Ключевое решение — **никакого `VerifyToken` RPC**. Auth подписывает access-токены асимметрично и публикует публичные ключи (JWKS). Gateway и Realtime валидируют подпись сами. Auth дергают по сети только для login/refresh/logout. Подробности: [07 — Безопасность](07-security.md).

## Core Education Service

Самый большой сервис — предметное ядро. Внутренние пакеты с чистыми границами (кандидаты на выделение в будущем, см. [ADR-001](../adr/ADR-001-microservices-granularity.md)):

- **scheduling** — доступность преподавателей, слоты, recurring-правила, календарь.
- **booking** — booking-сага: анти-double-booking (exclusion constraint на `tstzrange`), вызов Billing.PlaceHold, статусы PENDING → CONFIRMED/FAILED. См. [04 — Саги](04-sagas-and-consistency.md).
- **lessons** — жизненный цикл урока, `JoinLesson` (проверка членства и временного окна → минт LiveKit-токена и room JWT), приём вебхуков LiveKit (`participant_joined`, `room_finished`) для трекинга посещаемости и таймера.
- **chats** — персистентность всех сообщений: личные чаты, групповые, чаты комнат уроков; вложения (ссылки на MinIO), статусы прочтения, поиск.
- **support** — тикеты поддержки со статусами open / pending / resolved / closed. Отдельная сущность, не «ещё один чат» — другая модель доступа (участвует админ) и жизненный цикл.
- **reports** — заметки после уроков, журнал прогресса; консьюмер `lesson.completed` инкрементит счётчик и на каждом 5-м завершённом уроке создаёт запись журнала + событие `progress.milestone_reached`.
- **outbox** — relay-горутина: читает таблицу `outbox`, публикует в JetStream. См. [03 — Взаимодействие](03-communication.md).

## Billing Service

Владеет деньгами. Модель — **append-only ledger**:

- `ledger_entries` — строки debit/credit; баланс пользователя = `SUM()` по его строкам, никогда не мутируемая колонка.
- `holds` — резервы под бронирования: PlaceHold при бронировании, CaptureHold при завершении урока, ReleaseHold при отмене.
- Все методы, двигающие деньги, идемпотентны: `Idempotency-Key` из gRPC metadata пишется в уникальный индекс; повторный вызов возвращает сохранённый результат.

Billing не знает про уроки и слоты — только про холды с внешним `reference_id`. Связь «урок завершён → захвати холд» — через события JetStream.

## Realtime Service

Сознательно почти stateless — чтобы горизонтально масштабироваться и переживать рестарты:

- **WS hub** — терминирует WebSocket-соединения, валидирует room JWT, маршрутизирует сообщения по типам (chat / board / presence / signal).
- **Room registry** — membership и presence в Redis (TTL-ключи); fan-out между инстансами realtime — Redis Pub/Sub.
- **Board log** — Yjs-апдейты как опаковые блобы: append в per-room лог, broadcast остальным, на reconnect — snapshot + tail. См. [05 — Realtime и вайтборд](05-realtime-whiteboard.md).
- **Chat forwarder** — сообщения чата комнаты рассылаются участникам сразу, затем асинхронно персистятся вызовом Core по gRPC.

Упавший инстанс realtime теряет только TCP-соединения: клиенты переподключаются к другому инстансу, состояние комнаты восстанавливается из Redis + лога апдейтов.

## Что НЕ выделяем в сервисы и почему

| Кандидат | Где живёт | Почему не сервис |
|---|---|---|
| Chat Service | пакет `chats` в Core | Нет своего масштабирующего профиля на MVP; выделение добавит сетевой хоп в каждое сообщение и распределённую транзакцию «сообщение + нотификация» |
| Notification Service | пакет в Core (консьюмер событий) | Пока единственный канал — внутри приложения; выделять стоит при появлении email/push-провайдеров |
| Reporting Service | пакет `reports` в Core | Чистый CRUD + один консьюмер; нет причин для отдельного деплоя |
| Media Service | LiveKit (готовый) | Писать свой SFU — не наша задача |

Правило: новый сервис появляется только при доказанной причине — иной профиль масштабирования, иной цикл релизов или иная команда. «Концептуально отдельная область» причиной не является.
