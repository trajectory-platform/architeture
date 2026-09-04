# 03 — Межсервисное взаимодействие

Два механизма, жёсткое правило выбора:

- **gRPC (синхронно)** — когда вызывающему нужен ответ *сейчас*, чтобы продолжить свою операцию.
- **NATS JetStream (асинхронно)** — когда сервис сообщает *свершившийся факт*, на который другие реагируют. Вызывающий не ждёт.

Если сомневаетесь — это событие. Синхронный вызов создаёт связность по доступности: Core, вызывающий Billing синхронно, недоступен, когда недоступен Billing. Таких пар должно быть мало и по делу.

## Синхронное взаимодействие (gRPC)

### Где используется

| Вызов | Зачем синхронно |
|---|---|
| Gateway → Auth / Core / Billing | Каждый REST-запрос клиента — это ответ, который клиент ждёт |
| Core → Billing `PlaceHold` | Бронирование не может стать CONFIRMED, пока деньги не зарезервированы — нужен ответ в той же операции |
| Realtime → Core `SaveMessage`, `AuthorizeRoomJoin` | Персистентность чата и проверка прав — короткие запрос/ответ |

### Правила

1. **Deadline на каждом вызове.** `context.WithTimeout` на стороне вызывающего; deadline распространяется по цепочке автоматически через gRPC. Вызов без дедлайна — баг.
2. **Retry — только идемпотентных методов.** Методы помечаются идемпотентными явно (в proto-комментариях и в retry-конфиге клиента). `PlaceHold` с `Idempotency-Key` ретраить можно; метод без ключа — нет.
3. **Idempotency-Key сквозной.** Клиент шлёт заголовок `Idempotency-Key` на любой POST, двигающий деньги или создающий бронирование. Gateway кладёт его в gRPC metadata, Core передаёт в Billing. Billing хранит ключ в уникальном индексе и на повтор возвращает сохранённый результат.
4. **OpenTelemetry-интерсепторы с первого дня.** Unary/stream-интерсепторы на клиенте и сервере каждого сервиса; trace context — через metadata. Запрос трассируется от REST-границы через все хопы до SQL.
5. **Ошибки — коды gRPC**, не строки: `NotFound`, `FailedPrecondition` (слот занят), `ResourceExhausted` (недостаточно средств), `AlreadyExists` (повтор идемпотентного вызова с другим телом).

### Контракты

Все `.proto` живут в отдельном репозитории `contracts` — единственном источнике межсервисных контрактов — и управляются **buf**:

```
proto/
  trajectory/auth/v1/auth.proto
  trajectory/education/v1/{scheduling,booking,lessons,chats,support,reports}.proto
  trajectory/billing/v1/billing.proto
  trajectory/events/v1/events.proto      # payload'ы событий JetStream — тоже protobuf
```

- `buf breaking` в CI — несовместимое изменение контракта не проходит ревью.
- Go-стабы публикуются как версионированный модуль; каждый сервис обновляет зависимость отдельным совместимым PR.
- OpenAPI хранится и проверяется в `contracts`; из него генерируется TypeScript-клиент фронтенда. Руками REST-типы не пишутся.

## Асинхронное взаимодействие (NATS JetStream)

Обоснование выбора JetStream и отказа от Redis Pub/Sub — [ADR-003](../adr/ADR-003-nats-jetstream.md). Кратко: Pub/Sub — fire-and-forget; если Billing лежит в момент `lesson.completed`, деньги не сойдутся никогда. JetStream даёт персистентные стримы, ack, redelivery и durable consumers.

### Каталог событий

Subject-схема: `trajectory.<domain>.<event>`. Payload — protobuf из `proto/trajectory/events/v1/`.

| Событие | Издатель | Потребители | Реакция |
|---|---|---|---|
| `booking.confirmed` | Core | Notification | Уведомления участникам, материализация календаря |
| `booking.cancelled` | Core | Billing, Notification | `ReleaseHold` — вернуть резерв на баланс; уведомить участников |
| `lesson.started` | Core (по вебхуку LiveKit) | Realtime, reports, Notification | Метка фактического начала, таймер и уведомление при необходимости |
| `lesson.completed` | Core | Billing, reports, Notification | Billing: `CaptureHold`; reports: инкремент прогресса; Notification: событие пользователю |
| `payment.captured` | Billing | Core, Notification | Пометить урок оплаченным; отправить чек/подтверждение |
| `progress.milestone_reached` | Core (reports) | Notification | Уведомление о новой записи журнала прогресса |
| `user.registered` | Auth | Core, Notification | Создание профиля и начальных настроек уведомлений |
| `password.reset.requested` | Auth | Notification | Отправка reset-ссылки; секрет URL забирается по защищённому Auth RPC, не из события |

### Dead-letter queue (DLQ)

У JetStream нет безопасного «автоматически переместить сообщение в DLQ» без логики консьюмера, поэтому это явный контракт обработки:

1. Каждый durable consumer задаёт `max_deliver` и backoff. Обработчик сохраняет receipt `event_id`/результат в своей БД; повторная доставка становится no-op.
2. После последней неудачной попытки он публикует `events.v1.DeadLettered` в отдельный stream `TRAJECTORY_DLQ` с subject `trajectory.dlq.<service>.<consumer>` и ждёт JetStream ack.
3. Только после подтверждённой публикации он ack'ает исходное сообщение. Падение между этими операциями может создать дубликат DLQ-сообщения, поэтому ключ дедупликации — `(event_id, consumer)`.
4. DLQ — закрытый stream с ограниченным retention (14 дней по умолчанию), отдельными ACL и алертом на каждое новое сообщение. `failure_message` санитизируется; raw payload не пишется в логи. Replay — явная операторская команда в исходный subject с новым audit record, не автоматическая повторная доставка.

### Transactional outbox

Проблема: «записал в БД и опубликовал событие» — две системы, между ними можно упасть. Решение — outbox в той же транзакции:

```sql
CREATE TABLE outbox (
    id           bigserial PRIMARY KEY,
    aggregate_id uuid        NOT NULL,
    subject      text        NOT NULL,   -- 'trajectory.booking.confirmed'
    payload      bytea       NOT NULL,   -- protobuf
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz             -- NULL = не опубликовано
);
```

Поток:

1. Сервис в одной транзакции пишет доменное изменение **и** строку в `outbox`.
2. Relay-горутина (в том же процессе) выбирает неопубликованные строки (`FOR UPDATE SKIP LOCKED` — безопасно при нескольких репликах), публикует в JetStream, ждёт ack, проставляет `published_at`.
3. Упали между публикацией и `published_at` → событие уйдёт повторно → консьюмеры идемпотентны, всё сходится.

Гарантия: **at-least-once, после коммита — обязательно**. Событие не может потеряться и не может уйти для откатившейся транзакции.

## Что НЕ ходит через шину

- **Yjs-апдейты доски** — это не межсервисные события, а realtime-поток клиентов; живут в WebSocket + Redis fan-out + лог апдейтов ([05](05-realtime-whiteboard.md)).
- **Медиа** — только LiveKit.
- **Запросы данных** («дай баланс») — это gRPC, события не используются как RPC.
