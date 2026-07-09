# 06 — Данные и хранилища

## Принцип: база на сервис

Каждый сервис владеет своей базой Postgres (отдельные инстансы или отдельные базы с раздельными учётками — но **никогда** не общие таблицы). Запрещены:

- cross-service join'ы и foreign key между базами разных сервисов;
- чтение чужих таблиц «потому что быстрее».

Чужие данные получают через gRPC владельца или реплицируют у себя по событиям (например, Core держит локальную проекцию «урок оплачен» по `payment.captured`). Ссылки между агрегатами разных сервисов — просто uuid без FK (`holds.reference_id` → booking).

```
auth_db      — identities, credentials, refresh_tokens, signing keys, outbox
education_db — slots, bookings, lessons, chats, messages, reports, tickets, outbox
billing_db   — ledger_entries, holds, idempotency_keys, outbox
realtime_db  — board_updates, board_snapshots (только лог доски)
```

## Ключевые таблицы

Ниже — несущие конструкции схемы (DDL упрощён: без всех индексов и полей аудита).

### education_db: бронирования и анти-double-booking

```sql
CREATE TABLE bookings (
    id              uuid PRIMARY KEY,
    teacher_id      uuid        NOT NULL,
    student_id      uuid        NOT NULL,
    time_range      tstzrange   NOT NULL,
    status          text        NOT NULL,  -- PENDING | CONFIRMED | CANCELLED | FAILED
    recurring_rule  uuid,                   -- NULL для разовых
    idempotency_key text        UNIQUE,
    created_at      timestamptz NOT NULL DEFAULT now()
);

-- двойная бронь невозможна на уровне БД, при любых гонках
ALTER TABLE bookings ADD CONSTRAINT no_double_booking
    EXCLUDE USING gist (teacher_id WITH =, time_range WITH &&)
    WHERE (status IN ('PENDING', 'CONFIRMED'));
```

(требуется `CREATE EXTENSION btree_gist` — для `teacher_id WITH =` в gist-индексе.)

### billing_db: append-only ledger

```sql
CREATE TABLE ledger_entries (
    id           bigserial   PRIMARY KEY,
    account_id   uuid        NOT NULL,      -- студент, преподаватель, платформа
    amount       bigint      NOT NULL,      -- минорные единицы; >0 credit, <0 debit
    entry_type   text        NOT NULL,      -- topup | capture | refund | payout
    reference_id uuid,                       -- booking / lesson / hold
    created_at   timestamptz NOT NULL DEFAULT now()
);
-- Баланс = SUM(amount) по account_id. Строки не обновляются и не удаляются.

CREATE TABLE holds (
    id              uuid        PRIMARY KEY,
    account_id      uuid        NOT NULL,
    amount          bigint      NOT NULL CHECK (amount > 0),
    reference_id    uuid        NOT NULL,   -- booking_id
    status          text        NOT NULL,   -- ACTIVE | CAPTURED | RELEASED
    idempotency_key text        UNIQUE NOT NULL,
    created_at      timestamptz NOT NULL DEFAULT now()
);
-- Доступный баланс = SUM(ledger) - SUM(holds WHERE status='ACTIVE').
```

Capture холда = одна транзакция: `holds.status → CAPTURED` + пара строк леджера (debit студента / credit получателя) + outbox `payment.captured`. История денег полная и неизменяемая — любой баланс воспроизводим на любой момент времени.

### outbox (в education_db и billing_db)

```sql
CREATE TABLE outbox (
    id           bigserial   PRIMARY KEY,
    aggregate_id uuid        NOT NULL,
    subject      text        NOT NULL,
    payload      bytea       NOT NULL,      -- protobuf события
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

Механика — в [03 — Взаимодействие](03-communication.md).

### realtime_db: лог доски

```sql
CREATE TABLE board_updates (
    lesson_id  uuid        NOT NULL,
    seq        bigint      NOT NULL,        -- монотонный per-room счётчик
    update     bytea       NOT NULL,        -- opaque Yjs update
    created_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (lesson_id, seq)
);

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
