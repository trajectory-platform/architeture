# Core Education Service

Предметное ядро системы и самый большой сервис: расписание, booking-сага, жизненный цикл уроков, чаты, тикеты поддержки, заметки и журнал прогресса, профили пользователей. Состоит из внутренних пакетов с чистыми границами — кандидатов на выделение в отдельные сервисы в будущем ([ADR-001](../adr/ADR-001-microservices-granularity.md)), но на MVP это один деплой и одна база.

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-core-education.puml) · [ER — education_db](../diagrams/db/education-db.puml) · [Sequence — booking-сага](../diagrams/sequence/booking-saga.puml) · [Sequence — вход в комнату](../diagrams/sequence/lesson-join.puml) · [Sequence — завершение урока](../diagrams/sequence/lesson-completed-billing.puml) · [Sequence — регистрация → профиль](../diagrams/sequence/auth-registration.puml)

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| Профили пользователей (имя, био, аватар; teacher: предметы, цена) | Идентичность, пароли, refresh-токены — [Auth](auth.md) |
| Доступность преподавателей, слоты, recurring-правила | Деньги: леджер, холды — Billing (Core знает только `booking_id` ↔ hold `reference_id`) |
| Бронирования (анти-double-booking) и booking-сага | Транспорт realtime: WebSocket, Yjs-лог, presence — Realtime |
| Уроки: жизненный цикл, посещаемость, `JoinLesson` | Медиа — LiveKit (Core только минтит токены и принимает вебхуки) |
| Чаты и сообщения (все виды), тикеты поддержки | Хранение байтов файлов — MinIO (Core выдаёт presigned URL) |
| Заметки после уроков, журнал прогресса | |

Core — **единственный авторитет по «кто имеет доступ к чему»** в предметной области: членство в уроке, в чате, в тикете проверяется по его таблицам ([07 — Безопасность](../architecture/07-security.md): roles в токене, ownership в данных).

## Внутренние пакеты

| Пакет | Ответственность | Подробно |
|---|---|---|
| `scheduling` | Правила доступности, материализация слотов на rolling-горизонт, календарь | ниже |
| `booking` | Booking-сага: advisory lock, exclusion constraint, `PlaceHold`, reconciler зависших PENDING | [04 — Саги](../architecture/04-sagas-and-consistency.md) |
| `lessons` | Жизненный цикл урока, `JoinLesson` (минт room JWT + LiveKit-токена), вебхуки LiveKit, посещаемость | ниже |
| `chats` | Сообщения всех видов чатов, вложения (MinIO-ключи), статусы прочтения, full-text поиск | [06 — Данные](../architecture/06-data-storage.md) |
| `support` | Тикеты: open / pending / resolved / closed; отдельная модель доступа (admin участвует) | |
| `reports` | Заметки после уроков; консьюмер `lesson.completed`: счётчик прогресса, журнал каждые 5 уроков | ниже |
| `profiles` | Проекция identity: создаётся консьюмером `user.registered`, редактируется пользователем | |
| `uploads` | Выдача presigned URL MinIO после проверки прав на объект | [06](../architecture/06-data-storage.md) |
| `outbox` | Relay-горутина: `FOR UPDATE SKIP LOCKED` → publish в JetStream → `published_at` | [03 — Взаимодействие](../architecture/03-communication.md) |
| `consumers` | Durable-консьюмеры `payment.captured`, `user.registered`; обработка идемпотентна | |

Правило границ пакетов: пакеты не лезут в таблицы друг друга напрямую — только через Go-интерфейсы. Это дешёвая страховка будущего выделения в сервис.

## API

### gRPC `education.v1` (вызывают Gateway и Realtime)

Контракты — `proto/trajectory/education/v1/{scheduling,booking,lessons,chats,support,reports}.proto` ([03](../architecture/03-communication.md)).

| Группа | RPC | Заметки |
|---|---|---|
| scheduling | `SetAvailability`, `ListSlots`, `GetCalendar` | Запись — только владелец-teacher; чтение слотов — любой студент |
| booking | `CreateBooking`, `CancelBooking`, `ListBookings` | `CreateBooking` — идемпотентен по `Idempotency-Key`, запускает сагу |
| booking | `CreateRecurringRule`, `CancelRecurringRule` | Отмена правила = отмена будущих CONFIRMED-вхождений штатной отменой |
| lessons | `JoinLesson` | Членство + окно урока → room JWT + LiveKit-токен |
| lessons | `EndLesson`, `GetLesson`, `ListLessons` | `EndLesson` — только teacher; закрывает комнату LiveKit |
| chats | `CreateChat`, `SaveMessage`, `ListMessages`, `MarkRead`, `SearchMessages` | `SaveMessage` идемпотентен по client-generated `message_id` — его же зовёт Realtime |
| chats | `AuthorizeRoomJoin` | Внутренний, для Realtime: проверка членства (fallback, основной путь — room JWT) |
| support | `CreateTicket`, `ListTickets`, `UpdateTicketStatus` | Смена статуса — участник или admin |
| reports | `CreateLessonNote`, `ListLessonNotes`, `GetProgressJournal` | Заметки — только teacher урока |
| profiles | `GetProfile`, `UpdateProfile` | Чужой профиль — публичная часть |

Все входящие RPC проходят интерсепторы: OTel-трейсинг, извлечение идентичности пользователя из metadata (подписанный контекст, не «системный суперпользователь» — [07](../architecture/07-security.md)), deadline обязателен.

### HTTP

| Endpoint | Назначение |
|---|---|
| `POST /webhooks/livekit` | Вебхуки LiveKit (`participant_joined`, `participant_left`, `room_finished`); подпись запроса проверяется до обработки |
| `POST /uploads` (через Gateway) | Выдача presigned PUT в MinIO: проверка прав, валидация типа/размера |

### Исходящие синхронные вызовы

| Вызов | Когда |
|---|---|
| Billing `PlaceHold` | Booking-сага; с `Idempotency-Key` и deadline — единственная синхронная зависимость Core от другого сервиса |
| LiveKit Server SDK | Минт токенов в `JoinLesson`, закрытие комнаты в `EndLesson` |
| MinIO S3 API | Подпись presigned URL |

## События

| Событие | Направление | Триггер / реакция |
|---|---|---|
| `booking.confirmed` | публикует | Транзакция №2 саги (вместе с `status=CONFIRMED`) |
| `booking.cancelled` | публикует | Отмена брони; payload несёт политику возврата (`release_full` / `release_partial`) |
| `lesson.started` | публикует | Вебхук `participant_joined` — фактическое начало |
| `lesson.completed` | публикует | Вебхук `room_finished` + посещаемость → `status=COMPLETED` |
| `progress.milestone_reached` | публикует | Каждый 5-й завершённый урок пары студент×преподаватель |
| `payment.captured` | потребляет | Проставить `lessons.paid_at` (локальная проекция «урок оплачен») |
| `user.registered` | потребляет | `INSERT profiles` идемпотентно по `user_id` |

Все публикации — только через outbox в той же транзакции, что и доменное изменение. Консьюмеры durable, идемпотентны (redelivery = at-least-once).

## Данные: education_db

ER-диаграмма: [education-db.puml](../diagrams/db/education-db.puml). DDL `bookings` (exclusion constraint), `chats`/`chat_members`/`messages` и `outbox` — в [06 — Данные](../architecture/06-data-storage.md); ниже — остальные несущие таблицы.

```sql
-- Проекция identity из Auth (user.registered); PK = user_id из Auth, без FK между базами
CREATE TABLE profiles (
    user_id      uuid        PRIMARY KEY,
    role         text        NOT NULL,             -- student | teacher (копия клейма на момент регистрации)
    display_name text        NOT NULL,
    bio          text,
    avatar_key   text,                              -- ключ объекта в MinIO (bucket avatars)
    timezone     text        NOT NULL DEFAULT 'UTC',
    created_at   timestamptz NOT NULL DEFAULT now()
);

-- Преподавательская часть: отдельно, чтобы не таскать NULL'ы у студентов
CREATE TABLE teacher_profiles (
    user_id          uuid   PRIMARY KEY REFERENCES profiles(user_id),
    subjects         text[] NOT NULL DEFAULT '{}',
    experience_years int,
    price_per_lesson bigint NOT NULL,               -- минорные единицы, как в Billing
    headline         text
);

-- Повторяющаяся доступность преподавателя (источник для материализации слотов)
CREATE TABLE availability_rules (
    id         uuid PRIMARY KEY,
    teacher_id uuid NOT NULL,
    weekday    smallint NOT NULL CHECK (weekday BETWEEN 0 AND 6),
    start_time time NOT NULL,
    end_time   time NOT NULL,
    valid_from date NOT NULL,
    valid_to   date
);

-- Материализованные окна, которые видит студент (rolling-горизонт, например 4 недели)
CREATE TABLE slots (
    id          uuid      PRIMARY KEY,
    teacher_id  uuid      NOT NULL,
    time_range  tstzrange NOT NULL,
    source_rule uuid      REFERENCES availability_rules(id),  -- NULL = разовый слот
    status      text      NOT NULL DEFAULT 'OPEN'             -- OPEN | CLOSED
);

-- Правило регулярной брони студента; вхождения материализует генератор через booking-сагу
CREATE TABLE recurring_rules (
    id          uuid     PRIMARY KEY,
    teacher_id  uuid     NOT NULL,
    student_id  uuid     NOT NULL,
    weekday     smallint NOT NULL,
    start_time  time     NOT NULL,
    duration    interval NOT NULL,
    valid_from  date     NOT NULL,
    valid_until date,
    status      text     NOT NULL DEFAULT 'ACTIVE'  -- ACTIVE | CANCELLED
);

CREATE TABLE lessons (
    id          uuid        PRIMARY KEY,
    booking_id  uuid        UNIQUE NOT NULL REFERENCES bookings(id),
    teacher_id  uuid        NOT NULL,
    student_id  uuid        NOT NULL,
    time_range  tstzrange   NOT NULL,
    status      text        NOT NULL DEFAULT 'SCHEDULED',
                            -- SCHEDULED | IN_PROGRESS | COMPLETED | CANCELLED | NO_SHOW
    started_at  timestamptz,                        -- фактический старт (вебхук participant_joined)
    ended_at    timestamptz,                        -- room_finished
    paid_at     timestamptz,                        -- проекция payment.captured
    livekit_room text
);

-- Посещаемость по вебхукам LiveKit
CREATE TABLE lesson_attendance (
    lesson_id uuid        NOT NULL REFERENCES lessons(id),
    user_id   uuid        NOT NULL,
    joined_at timestamptz NOT NULL,
    left_at   timestamptz,
    PRIMARY KEY (lesson_id, user_id, joined_at)     -- переподключения = несколько интервалов
);

-- Тикеты поддержки: chat.kind='support' + статусная надстройка
CREATE TABLE support_tickets (
    chat_id     uuid PRIMARY KEY REFERENCES chats(id),
    status      text NOT NULL DEFAULT 'open',       -- open | pending | resolved | closed
    assignee_id uuid,                                -- admin
    created_at  timestamptz NOT NULL DEFAULT now()
);

-- reports: заметки и прогресс
CREATE TABLE lesson_notes (
    id         uuid        PRIMARY KEY,
    lesson_id  uuid        NOT NULL REFERENCES lessons(id),
    teacher_id uuid        NOT NULL,
    body       text        NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

-- Идемпотентность консьюмера lesson.completed: ON CONFLICT DO NOTHING
CREATE TABLE progress_processed_lessons (
    lesson_id    uuid PRIMARY KEY,
    processed_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE progress_counters (
    teacher_id      uuid NOT NULL,
    student_id      uuid NOT NULL,
    completed_count int  NOT NULL DEFAULT 0,
    PRIMARY KEY (teacher_id, student_id)
);

CREATE TABLE progress_journal (
    id         uuid        PRIMARY KEY,
    teacher_id uuid        NOT NULL,
    student_id uuid        NOT NULL,
    milestone  int         NOT NULL,                -- 5, 10, 15...
    summary    text,
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (teacher_id, student_id, milestone)      -- повторная обработка не создаст дубль
);
```

Инварианты:

- **Двойная бронь невозможна на уровне БД** — exclusion constraint на `bookings (teacher_id, time_range)` для статусов PENDING/CONFIRMED ([06](../architecture/06-data-storage.md)); advisory lock по `teacher_id` сериализует гонки до constraint'а.
- **Урок ↔ бронь 1:1** (`lessons.booking_id UNIQUE`); урок создаётся при подтверждении брони.
- **Прогресс не задвоится**: отметка обработки урока (`progress_processed_lessons`, `ON CONFLICT DO NOTHING`) + инкремент счётчика + milestone — одна транзакция; `UNIQUE (teacher, student, milestone)` — вторая линия обороны.
- **Ссылки на чужие агрегаты — uuid без FK**: `profiles.user_id` (identity в Auth), холд в Billing находит бронь по `reference_id`.

## Ключевые потоки

- **Booking-сага** — [booking-saga.puml](../diagrams/sequence/booking-saga.puml), [04](../architecture/04-sagas-and-consistency.md). Транзакция №1 (PENDING + constraint) → `PlaceHold` → транзакция №2 (CONFIRMED + outbox). Неизвестный исход → reconciler повторяет `PlaceHold` с тем же ключом.
- **Отмена** — `CANCELLED` + outbox `booking.cancelled`; Billing вернёт холд асинхронно. Политика возврата (полный/частичный) вычисляется в Core и едет в payload.
- **Вход в комнату** — [lesson-join.puml](../diagrams/sequence/lesson-join.puml). `JoinLesson` = членство + окно времени → room JWT (~1 мин TTL) + LiveKit-токен с grants только на эту комнату. Realtime и LiveKit в базы за правами не ходят.
- **Завершение урока** — [lesson-completed-billing.puml](../diagrams/sequence/lesson-completed-billing.puml). Вебхук `room_finished` → `COMPLETED` + outbox `lesson.completed` → Billing захватывает холд, reports инкрементит прогресс. Потребители независимы.
- **Recurring-брони** — джоба-генератор материализует вхождения правила на rolling-горизонт через обычную booking-сагу; идемпотентный ключ детерминирован: `rule_id + occurrence_date`. FAILED вхождение (нет денег) не трогает остальные.
- **Слоты** — джоба `scheduling` материализует `slots` из `availability_rules` на тот же горизонт; правка правила пересобирает будущие свободные слоты, забронированные не трогаются.
- **Профили** — консьюмер `user.registered` создаёт строку `profiles` (идемпотентно по `user_id`); дальше профиль редактируется только через `UpdateProfile`.

## Авторизация (ownership-проверки)

| Ресурс | Проверка |
|---|---|
| Урок / комната | `user_id ∈ {lesson.teacher_id, lesson.student_id}` + текущее время в окне урока |
| Чат | строка в `chat_members` |
| Тикет | участник чата тикета или `role = admin` |
| Расписание | запись — только владелец-teacher; чтение слотов — любой студент |
| Заметки/прогресс | участники пары студент×преподаватель |
| Файл (presigned URL) | права на объект до выдачи URL |

## Чего здесь нет — и почему

- **Денег.** Балансы и холды — Billing; Core оперирует только `booking_id`/суммой в `PlaceHold` и слушает `payment.captured`. Компрометация education_db не открывает леджер.
- **Yjs-апдейтов и presence.** Realtime-поток — не предметные данные; Core хранит только результат (`messages` через `SaveMessage`).
- **Отдельных Chat/Reporting сервисов.** Пакеты внутри Core до доказанной причины выделения ([ADR-001](../adr/ADR-001-microservices-granularity.md)). Уведомления намеренно выделены в Notification Service и получают доменные факты из outbox/JetStream.
- **Байтов файлов.** Только MinIO-ключи; байты ходят клиент↔MinIO по presigned URL.
