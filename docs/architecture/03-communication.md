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

Все `.proto` — в одном модуле `proto/` в корне монорепы, под управлением **buf**:

```
proto/
  trajectory/auth/v1/auth.proto
  trajectory/education/v1/{scheduling,booking,lessons,chats,support,reports}.proto
  trajectory/billing/v1/billing.proto
  trajectory/events/v1/events.proto      # payload'ы событий JetStream — тоже protobuf
```

- `buf breaking` в CI — несовместимое изменение контракта не проходит ревью.
- Go-стабы генерируются для всех сервисов из одного источника.
- Gateway генерирует OpenAPI-спеку → из неё TypeScript-клиент фронтенда. Руками REST-типы не пишутся.

## Асинхронное взаимодействие (NATS JetStream)

Обоснование выбора JetStream и отказа от Redis Pub/Sub — [ADR-003](../adr/ADR-003-nats-jetstream.md). Кратко: Pub/Sub — fire-and-forget; если Billing лежит в момент `lesson.completed`, деньги не сойдутся никогда. JetStream даёт персистентные стримы, ack, redelivery и durable consumers.

### Каталог событий

Subject-схема: `trajectory.<domain>.<event>`. Payload — protobuf из `proto/trajectory/events/v1/`.

| Событие | Издатель | Потребители | Реакция |
|---|---|---|---|
| `booking.confirmed` | Core | (нотификации в Core) | Уведомления участникам, материализация календаря |
| `booking.cancelled` | Core | Billing | `ReleaseHold` — вернуть резерв на баланс |
| `lesson.started` | Core (по вебхуку LiveKit) | Realtime, reports | Метка фактического начала, таймер |
| `lesson.completed` | Core | Billing, reports | Billing: `CaptureHold` (списание); reports: инкремент счётчика прогресса |
| `payment.captured` | Billing | Core | Пометить урок оплаченным |
| `progress.milestone_reached` | Core (reports) | (нотификации в Core) | Запись в журнал прогресса каждые 5 уроков, уведомление |
| `user.registered` | Auth | Core | Создание профиля студента/преподавателя |

Правила консьюмеров: durable consumer на сервис, обработка **идемпотентна** (redelivery возможна — at-least-once), ack только после успешной обработки, после N неудачных доставок — событие в dead-letter subject + алерт.

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
