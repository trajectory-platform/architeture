# 03 — Межсервисное взаимодействие

Два механизма:

- **gRPC** — вызывающему нужен ответ для продолжения операции;
- **Apache Kafka** — сервис сообщает совершившийся факт, а издатель не ждёт реакции потребителей.

## Синхронное взаимодействие

| Вызов | Зачем синхронно |
|---|---|
| Gateway → Auth / Core / Learning / Billing / Notification | REST-запрос клиента требует ответа |
| Core → Billing `PlaceHold` | Бронь не подтверждается до успешного резерва |
| Realtime → Core `SaveMessage`, `AuthorizeRoomJoin` | Персистентность чата и fallback-проверка доступа |
| Notification → Auth `GetPasswordResetDelivery` | Одноразовый reset URL нельзя помещать в Kafka |

Правила:

1. Каждый вызов имеет deadline.
2. Автоматический retry разрешён только для идемпотентной операции.
3. Денежные и создающие операции передают `Idempotency-Key` через gRPC metadata.
4. OpenTelemetry context проходит через metadata.
5. Доменные ошибки передаются кодами gRPC, не текстовым сравнением.

## Контракты

Все `.proto` живут в репозитории `contracts`:

```text
proto/trajectory/auth/v1/auth.proto
proto/trajectory/education/v1/*.proto
proto/trajectory/learning/v1/*.proto
proto/trajectory/billing/v1/billing.proto
proto/trajectory/notification/v1/notification.proto
proto/trajectory/events/v1/*.proto
```

Kafka payloads используют protobuf envelope. `buf lint`, `buf breaking` и code generation обязательны в CI. OpenAPI остаётся источником REST-контракта Gateway.

## Асинхронное взаимодействие

Решение закреплено в [ADR-003](../adr/ADR-003-apache-kafka.md). Kafka работает в KRaft mode. Каждый потребитель использует отдельную consumer group. Обработка имеет семантику at-least-once.

### Topics и ключи

| Topic | Издатель | Примеры событий |
|---|---|---|
| `trajectory.auth.events.v1` | Auth | `user.registered`, `password.reset.requested`, `identity.email_confirmation.requested`, `identity.email_change.requested`, `identity.invitation.requested`, `identity.security_changed`, `identity.session_compromised` |
| `trajectory.education.events.v1` | Core | `booking.*`, `lesson.*` |
| `trajectory.learning.events.v1` | Learning | `homework.*`, `enrollment.*`, `progress.*` |
| `trajectory.billing.events.v1` | Billing | `payment.captured`, `hold.released`, `payout.*` |
| `trajectory.notification.events.v1` | Notification | `notification.created`, `notification.delivered`, `notification.failed` |

Message key — `aggregate_id`. События одного агрегата сохраняют порядок в partition. Envelope содержит `event_id`, `event_type`, `occurred_at`, `schema_version`, `producer` и protobuf payload.

### Каталог событий

| Событие | Издатель | Потребители | Реакция |
|---|---|---|---|
| `user.registered` | Auth | Core, Notification | Профиль; начальные настройки уведомлений |
| `password.reset.requested` | Auth | Notification | Получить reset URL через защищённый Auth RPC и доставить |
| `identity.email_confirmation.requested` | Auth | Notification | Получить одноразовые delivery data и отправить подтверждение email |
| `identity.email_change.requested` | Auth | Notification | Подтвердить новый email и уведомить старый адрес |
| `identity.invitation.requested` | Auth | Notification | Отправить приглашение сотруднику или ученику |
| `identity.security_changed` | Auth | Notification | Создать in-app уведомление об изменении способа входа |
| `identity.session_compromised` | Auth | Notification | Сообщить о refresh-token reuse и отзыве сессии |
| `booking.confirmed` | Core | Notification | Уведомить участников |
| `booking.cancelled` | Core | Billing, Notification | Исполнить полный/частичный возврат; уведомить |
| `lesson.started` | Core | Realtime, Notification | Прогрев комнаты; in-app уведомление |
| `lesson.completed` | Core | Billing, Learning, Notification | Захватить holds; обновить прогресс; уведомить |
| `lesson.recorded` | Core | Learning, Notification | Добавить запись в материалы; уведомить участников |
| `payment.captured` | Billing | Core, Learning, Notification | Пометить оплату; обновить доступ; уведомить |
| `homework.assigned` | Learning | Notification | Уведомить ученика |
| `homework.submitted` | Learning | Notification | Уведомить преподавателя |
| `homework.reviewed` | Learning | Notification | Уведомить ученика |
| `enrollment.expiring` | Learning | Core, Notification | Показать состояние доступа; уведомить |
| `progress.milestone_reached` | Learning | Notification | Уведомить о достижении |

### Consumer contract

1. Consumer читает событие в своей consumer group.
2. Consumer проверяет `event_id` и применяет изменение в одной транзакции своей БД.
3. Consumer фиксирует offset только после commit.
4. Redelivery с тем же `event_id` становится no-op.
5. Kafka offset не является бизнес-идентификатором.

### Dead-letter topics

После ограниченного числа попыток consumer публикует `events.v1.DeadLettered` в `trajectory.dlq.<service>.<consumer>`. Затем consumer фиксирует offset исходного сообщения. Ключ дедупликации DLQ — `(event_id, consumer)`.

DLQ имеет retention 14 дней, отдельные ACL и алерт. `failure_message` санитизируется. Raw password, token, private message и другие секреты не попадают в DLQ. Replay выполняется вручную с audit record.

### Transactional outbox

```sql
CREATE TABLE outbox (
    id           bigserial PRIMARY KEY,
    aggregate_id uuid        NOT NULL,
    topic        text        NOT NULL,
    event_type   text        NOT NULL,
    payload      bytea       NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);
```

Сервис пишет доменное изменение и outbox row в одной Postgres transaction. Relay выбирает строки через `FOR UPDATE SKIP LOCKED`, публикует с `acks=all` и message key = `aggregate_id`, затем ставит `published_at`. Падение после publish до update приводит к повторной публикации; consumer deduplication обеспечивает корректность.

## Что не ходит через Kafka

- Yjs board updates: WebSocket, Redis fan-out и Postgres log.
- Медиа: LiveKit.
- Запросы данных: gRPC.
- Raw password, refresh/reset tokens, KEK и private signing keys.
