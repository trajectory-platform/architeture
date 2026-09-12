# 06 — Данные и хранилища

## Принцип: база на сервис

Каждый сервис владеет своей базой Postgres (отдельные инстансы или отдельные базы с раздельными учётками — но **никогда** не общие таблицы). Запрещены:

- cross-service join'ы и foreign key между базами разных сервисов;
- чтение чужих таблиц «потому что быстрее».

Чужие данные получают через gRPC владельца или реплицируют у себя по событиям (например, Core держит локальную проекцию «урок оплачен» по `payment.captured`). Ссылки между агрегатами разных сервисов — просто uuid без FK (`holds.reference_id` → booking_participant).

```
auth_db      — identities/subtypes, roles/permissions, sessions, credentials, refresh_tokens, auth_action_requests, auth_delivery_payloads, signing keys, audit, outbox
education_db — profiles, slots, bookings, booking_participants, lessons, attendance, recordings, chats, tickets, outbox
learning_db  — courses, groups, enrollments, homework, tests, materials, progress, processed_events, outbox
billing_db   — accounts, ledger_entries, holds, idempotency_keys, outbox
realtime_db  — board_rooms, board_updates, board_snapshots (состояние доски)
notification_db — notification_preferences, templates, notifications, deliveries, processed_events
```

## Ключевые таблицы

Ниже — несущие конструкции схемы (DDL упрощён: без всех индексов и полей аудита).

### education_db: бронирования и анти-double-booking

```sql
CREATE TABLE bookings (
    id              uuid PRIMARY KEY,
    teacher_id      uuid        NOT NULL,
    time_range      tstzrange   NOT NULL,
    status          text        NOT NULL,  -- PENDING | CONFIRMED | CANCELLED | FAILED
    recurring_rule  uuid,                   -- NULL для разовых
    idempotency_key text        UNIQUE,
    created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE booking_participants (
    id               uuid        PRIMARY KEY,
    booking_id       uuid        NOT NULL REFERENCES bookings(id),
    student_id       uuid        NOT NULL,
    status           text        NOT NULL,  -- PENDING | CONFIRMED | CANCELLED | NO_SHOW | FAILED
    price_amount     bigint      NOT NULL CHECK (price_amount > 0),
    currency         char(3)     NOT NULL,
    hold_reference_id uuid       UNIQUE NOT NULL,
    UNIQUE (booking_id, student_id),
    CHECK (hold_reference_id = id)
);
-- Для группы цена и hold фиксируются отдельно с каждого ученика.

-- двойная бронь невозможна на уровне БД, при любых гонках
ALTER TABLE bookings ADD CONSTRAINT no_double_booking
    EXCLUDE USING gist (teacher_id WITH =, time_range WITH &&)
    WHERE (status IN ('PENDING', 'CONFIRMED'));
```

(требуется `CREATE EXTENSION btree_gist` — для `teacher_id WITH =` в gist-индексе.)

### billing_db: append-only ledger

```sql
CREATE TABLE accounts (
    id         uuid PRIMARY KEY,
    currency   char(3) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);
-- Все изменения доступных средств: SELECT accounts ... FOR UPDATE до проверки.

CREATE TABLE ledger_entries (
    id           bigserial   PRIMARY KEY,
    account_id   uuid        NOT NULL REFERENCES accounts(id),
    amount       bigint      NOT NULL,      -- минорные единицы; >0 credit, <0 debit
    entry_type   text        NOT NULL,      -- topup | capture | refund | payout
    reference_id uuid,                       -- booking / lesson / hold
    created_at   timestamptz NOT NULL DEFAULT now()
);
-- Баланс = SUM(amount) по account_id. Строки не обновляются и не удаляются.

CREATE TABLE holds (
    id              uuid        PRIMARY KEY,
    account_id      uuid        NOT NULL REFERENCES accounts(id),
    amount          bigint      NOT NULL CHECK (amount > 0),
    reference_id    uuid        UNIQUE NOT NULL, -- booking_participant_id
    status          text        NOT NULL,   -- ACTIVE | CAPTURED | RELEASED
    idempotency_key text        UNIQUE NOT NULL, -- scoped caller/method/operation
    request_hash    bytea       NOT NULL,
    created_at      timestamptz NOT NULL DEFAULT now()
);
-- Доступный баланс = SUM(ledger) - SUM(holds WHERE status='ACTIVE').
```

Capture холда = одна транзакция: `holds.status → CAPTURED` + пара строк леджера (debit студента / credit получателя) + outbox `payment.captured`. История денег полная и неизменяемая — любой баланс воспроизводим на любой момент времени.

### outbox (в БД сервисов-издателей)

```sql
CREATE TABLE outbox (
    id           bigserial   PRIMARY KEY,
    event_id     uuid        UNIQUE NOT NULL,
    aggregate_id uuid        NOT NULL,
    aggregate_version bigint NOT NULL,
    topic        text        NOT NULL,
    event_type   text        NOT NULL,
    payload      bytea       NOT NULL,      -- protobuf события
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz,
    UNIQUE (aggregate_id, aggregate_version)
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

Механика — в [03 — Взаимодействие](03-communication.md).

### realtime_db: лог доски

```sql
CREATE TABLE board_rooms (
    lesson_id    uuid PRIMARY KEY,
    last_seq     bigint NOT NULL DEFAULT 0,
    snapshot_seq bigint NOT NULL DEFAULT 0,
    CHECK (last_seq >= snapshot_seq AND snapshot_seq >= 0)
);
-- В одной tx: UPDATE board_rooms SET last_seq=last_seq+1 ... RETURNING last_seq;
-- затем INSERT update с этим seq. Счётчик не удаляется при pruning.

CREATE TABLE board_updates (
    lesson_id  uuid        NOT NULL,
    seq        bigint      NOT NULL,        -- монотонный per-room счётчик
    update     bytea       NOT NULL,        -- opaque Yjs update
    created_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (lesson_id, seq)
);

CREATE TABLE board_update_receipts (
    lesson_id uuid NOT NULL REFERENCES board_rooms(lesson_id),
    update_id uuid NOT NULL,
    digest bytea NOT NULL,
    seq bigint NOT NULL,
    expires_at timestamptz NOT NULL,
    PRIMARY KEY (lesson_id, update_id)
);
-- Receipts переживают pruning bytes до конца согласованного offline replay window.

CREATE TABLE board_snapshots (
    lesson_id  uuid        NOT NULL,
    upto_seq   bigint      NOT NULL,        -- снапшот покрывает updates с seq <= upto_seq
    snapshot   bytea       NOT NULL,        -- Y.encodeStateAsUpdate
    created_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (lesson_id, upto_seq)
);
-- sync_response = последний snapshot + updates WHERE seq > upto_seq (см. 05)
```

Крупные снапшоты и архив старых апдейтов можно выносить в MinIO (в таблице остаётся ссылка) — Postgres хранит горячий хвост.

Каждый `lesson_id` создаёт независимую доску. Новый урок начинает новый board log; состояние прошлого урока доступно только как отдельный material snapshot.

### education_db: чаты

```sql
CREATE TABLE chats (
    id        uuid PRIMARY KEY,
    kind      text NOT NULL,   -- direct | group | lesson_room | support
    lesson_id uuid,             -- для lesson_room
    title     text
);

CREATE TABLE chat_members (
    chat_id      uuid NOT NULL REFERENCES chats(id),
    user_id      uuid NOT NULL,
    role         text NOT NULL,           -- member | owner | admin
    last_read_at timestamptz,             -- статусы прочтения
    PRIMARY KEY (chat_id, user_id)
);

CREATE TABLE messages (
    id         uuid        PRIMARY KEY,   -- client-generated: ретраи без дублей
    chat_id    uuid        NOT NULL REFERENCES chats(id),
    sender_id  uuid        NOT NULL,
    body       text,
    attachment text,                       -- ключ объекта в MinIO
    created_at timestamptz NOT NULL DEFAULT now()
);
```

Один механизм для всех видов чатов; тикеты поддержки — `chats.kind = 'support'` + отдельная таблица `support_tickets (chat_id, status, assignee_id)` со статусами open / pending / resolved / closed. Поиск по сообщениям — Postgres full-text (`tsvector`-индекс по `body`) — на MVP достаточно.

### education_db: входящие webhook observations

```sql
CREATE TABLE webhook_inbox (
    provider_event_id text PRIMARY KEY,
    event_type text NOT NULL,
    room_id text NOT NULL,
    received_at timestamptz NOT NULL DEFAULT now(),
    safe_payload jsonb NOT NULL
);
```

Подпись проверяется до INSERT; Core state machine дедуплицирует и применяет наблюдение отдельно от транспортной доставки. Payload исключает секреты и имеет срок хранения.

### education_db: записи уроков

```sql
CREATE TABLE lesson_recordings (
    id          uuid        PRIMARY KEY,
    lesson_id   uuid        NOT NULL,
    object_key  text        UNIQUE, -- известен после успешной обработки
    status      text        NOT NULL,  -- REQUESTED | RECORDING | PROCESSING | READY | DELETING | DELETED | FAILED
    ready_at    timestamptz,
    delete_after timestamptz,
    deleted_at  timestamptz,
    created_at  timestamptz NOT NULL DEFAULT now()
);
-- При READY: delete_after = ready_at + interval '1 month'.
```

Согласия участников хранятся отдельно от recording metadata. Получение presigned URL требует проверки membership. После `delete_after` Core прекращает выдачу URL и удаляет S3 object фоновой задачей.

### learning_db: учебный контур

```sql
CREATE TABLE courses (
    id          uuid PRIMARY KEY,
    owner_id    uuid NOT NULL,
    title       text NOT NULL,
    status      text NOT NULL,  -- DRAFT | PUBLISHED | ARCHIVED
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE groups (
    id          uuid PRIMARY KEY,
    course_id   uuid NOT NULL REFERENCES courses(id),
    teacher_id  uuid NOT NULL,
    capacity    smallint NOT NULL CHECK (capacity BETWEEN 1 AND 7)
);

CREATE TABLE enrollments (
    id          uuid PRIMARY KEY,
    course_id   uuid NOT NULL REFERENCES courses(id),
    group_id    uuid REFERENCES groups(id),
    student_id  uuid NOT NULL,
    status      text NOT NULL,  -- PENDING | ACTIVE | SUSPENDED | EXPIRED | REVOKED
    starts_at   timestamptz NOT NULL,
    ends_at     timestamptz,
    UNIQUE (course_id, student_id)
);

CREATE TABLE materials (
    id          uuid PRIMARY KEY,
    course_id   uuid REFERENCES courses(id),
    lesson_id   uuid,
    kind        text NOT NULL,  -- FILE | BOARD_SNAPSHOT | RECORDING
    object_key  text NOT NULL,
    expires_at  timestamptz,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE processed_events (
    event_id     uuid PRIMARY KEY,
    event_type   text NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

Полная ER-модель: [learning-db.puml](../diagrams/db/learning-db.puml). Homework, tests и progress имеют свои таблицы; связи не пересекают границу `learning_db` foreign keys.

### notification_db: inbox и доставки

```sql
CREATE TABLE notification_preferences (
    user_id    uuid PRIMARY KEY,
    email      boolean NOT NULL DEFAULT true,
    push       boolean NOT NULL DEFAULT true,
    in_app     boolean NOT NULL DEFAULT true,
    updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE templates (
    id         uuid PRIMARY KEY,
    kind       text NOT NULL,
    channel    text NOT NULL,  -- in_app | email | push
    version    integer NOT NULL,
    subject    text,
    body       text NOT NULL,
    active     boolean NOT NULL DEFAULT true,
    UNIQUE (kind, channel, version)
);

CREATE TABLE notifications (
    id           uuid PRIMARY KEY,
    user_id      uuid NOT NULL,
    kind         text NOT NULL,
    title        text NOT NULL,
    payload      jsonb NOT NULL,
    event_id     uuid NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    read_at      timestamptz,
    UNIQUE (event_id, user_id, kind)
);

CREATE TABLE deliveries (
    id              uuid PRIMARY KEY,
    notification_id uuid NOT NULL REFERENCES notifications(id),
    channel         text NOT NULL,       -- in_app | email | push
    status          text NOT NULL,       -- PENDING | SENT | RETRYING | FAILED
    attempts        integer NOT NULL DEFAULT 0,
    next_attempt_at timestamptz,
    provider_ref    text,
    last_error_code text,
    updated_at      timestamptz NOT NULL DEFAULT now(),
    UNIQUE (notification_id, channel)
);

CREATE TABLE processed_events (
    event_id     uuid PRIMARY KEY,
    event_type   text NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now()
);
```

Сервис не принимает пользовательские уведомления как бизнес-команды: он материализует факты из Kafka. Уникальные ключи делают обработку событий и ретраи доставки идемпотентными.

## Redis

Redis — **эфемерные** данные; потеря Redis = деградация (переподключения, сброс кэша), но не потеря данных:

| Использование | Кто | Структуры |
|---|---|---|
| Presence / membership комнат | Realtime | TTL-ключи, sets |
| Fan-out между инстансами realtime | Realtime | Pub/Sub каналы per-room |
| Кэш (профили, расписания, JWKS) | Gateway, Core | строки с TTL |
| Rate limiting | Gateway | счётчики с TTL |
| Сессионные данные / блэклист отозванных токенов | Auth, Gateway | TTL-ключи |

## MinIO

S3-совместимое объектное хранилище, бакеты:

- `attachments` — файлы чатов и вложения доски;
- `lesson-materials` — материалы уроков, домашние задания;
- `board-archive` — архив снапшотов/логов досок завершённых уроков;
- `avatars` — статика профилей.

Загрузка — через **presigned URL**: клиент просит Core (`POST /uploads`), получает подписанный URL, грузит напрямую в MinIO; в БД хранится только ключ объекта. Бэкенд не проксирует байты файлов. Скачивание — симметрично, presigned GET с проверкой прав в Core.

## Миграции

На сервис — свой каталог миграций (golang-migrate / goose), применяются при деплое сервиса-владельца. Общих миграций нет — это следствие принципа «база на сервис».

## Условия использования схем по релизам

DDL здесь логический, не migration script. `booking_participants.id` — стабильный hold reference; строка booking резервирует время один раз, вместимость проверяется под её row lock. Индивидуальный сценарий 1.0 использует одну participant row; группы — 2.0. Миграции должны сохранять существующие ID/денежные связи при upgrade.

`board_rooms.snapshot_seq` переключается только на проверенный snapshot в одной транзакции с pruning; consistent read (например REPEATABLE READ) возвращает его и хвост до last_seq. Для 1.0 допустим полный лог без pruning; клиентский snapshot не разрешает удаление. `auth_delivery_payloads` и правила очистки описаны в [Auth](../services/auth.md) и ADR-008. Наличие поля payout/recording не означает поставку соответствующего workflow в 1.0.
