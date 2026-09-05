# Core Education Service

Core Education владеет временем и фактом занятия: профилями, расписанием, бронированием, уроками, посещаемостью, записями, чатами и поддержкой. Учебное содержание и прогресс принадлежат [Learning Service](learning.md).

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-core-education.puml) · [Модули и зависимости](../diagrams/components/core-education-modules.puml) · [ER — education_db](../diagrams/db/education-db.puml) · [Booking saga](../diagrams/sequence/booking-saga.puml) · [Вход в урок](../diagrams/sequence/lesson-join.puml) · [Завершение урока](../diagrams/sequence/lesson-completed-billing.puml)

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| Профили пользователей | Identity, credentials, roles и sessions — Auth |
| Доступность, слоты и recurring rules | Ledger, holds, тарифы и выплаты — Billing |
| Бронирования, участники и price snapshot на ученика | Курсы, группы, задания, тесты, материалы и прогресс — Learning |
| Уроки, посещаемость и доступ в комнату | WebSocket transport и board log — Realtime |
| Записи урока и срок удаления | Байты файлов и записей — S3 |
| Чаты и обращения поддержки | Inbox и доставка уведомлений — Notification |

Core — авторитет по членству в уроке, чате и обращении. Learning может предоставить состав группы, но Core фиксирует участников конкретного урока.

## Внутренние пакеты

Каждый пакет ниже является бизнес-модулем с собственными application ports, domain model и repository adapter. Модуль может вызывать другой модуль только через его публичный application port. Прямые импорты internal-пакетов и SQL к таблицам другого модуля запрещены. Допустимые зависимости показаны на [диаграмме модулей](../diagrams/components/core-education-modules.puml).

| Пакет | Ответственность |
|---|---|
| `profiles` | Профили, timezone и минимальные projections ролей |
| `scheduling` | Доступность, исключения, materialized slots и recurring rules |
| `booking` | Анти-double-booking, participants, price snapshots, `PlaceHold`, перенос, отмена и reconciler |
| `lessons` | Жизненный цикл, участники, посещаемость, `JoinLesson`, LiveKit webhooks |
| `recordings` | Согласия, LiveKit Egress, metadata записи, удаление через один месяц |
| `chats` | Direct/group/lesson-room чаты, сообщения, вложения и read state |
| `support` | Обращения, назначение, приоритеты и статусы |
| `uploads` | Presigned URLs после проверки доступа |
| `outbox` | Публикация событий в Kafka после commit |
| `consumers` | Идемпотентные projections `user.registered`, `payment.captured`, `enrollment.expiring` |

## API

Контракты находятся в `proto/trajectory/education/v1/`.

| Область | RPC | Правило |
|---|---|---|
| profiles | `GetProfile`, `UpdateProfile` | Запись — владелец или отдельное admin permission |
| scheduling | `CreateAvailabilityRule`, `ListSlots` | Время хранится в UTC; правило содержит IANA timezone |
| booking | `CreateBooking`, `CancelBooking`, `ListBookings` | Мутации идемпотентны; price фиксируется до `PlaceHold` |
| booking | `CreateRecurringRule`, `CancelRecurringRule` | Каждое вхождение проходит обычную booking saga |
| lessons | `JoinLesson`, `EndLesson`, `GetLesson`, `ListLessons` | Доступ по фактическому membership |
| recordings | `StartRecording`, `StopRecording`, `GetRecording` | Старт только после зафиксированных согласий |
| chats | `ListChats`, `ListMessages`, `SendMessage`, `MarkRead` | Membership проверяется для каждого вызова |
| support | `CreateTicket`, `ReplyTicket`, `ChangeTicketStatus` | Admin actions требуют granular permission |
| uploads | `CreateUpload`, `CreateDownload` | Короткий presigned URL и scoped object key |

## Бронирование и цена

Цена фиксируется **с каждого ученика**, а не делится между участниками группы. `booking_participants` хранит `price_amount`, `currency` и `hold_reference_id` для каждого ученика. Изменение тарифа после подтверждения не меняет существующую бронь.

Для индивидуальной брони есть один participant. Для групповой — до 7 учеников и отдельный `PlaceHold` на каждого. Ошибка резерва одного ученика отклоняет только его место; подтверждённые места остальных не меняются.

Double booking преподавателя исключает constraint по `teacher_id + time_range`. Повторное место того же ученика исключает уникальный ключ `(booking_id, student_id)`.

## Отмена и неявка

Core вычисляет результат cancellation policy по snapshot правил брони:

- до окончания бесплатного окна — полный возврат;
- после окна — частичный возврат либо полное списание;
- при неявке ученика hold списывается, если бесплатное окно отмены уже пропущено.

Core публикует итоговые суммы в `booking.cancelled` или participant outcome в `lesson.completed`. Billing не пересчитывает расписание и policy.

## Урок и доска

Один урок имеет одну отдельную комнату и одну новую доску. `lesson_id` является room ID и ключом board log в Realtime. Состояние предыдущей доски доступно как материал, но не становится начальным состоянием нового урока автоматически.

`JoinLesson` возвращает короткий room JWT и LiveKit token после проверки membership и временного окна. Для группы проверяется строка `lesson_participants`, а не роль пользователя.

## Запись урока

Перед стартом Core фиксирует согласие каждого участника. LiveKit Egress пишет файл напрямую в S3. После webhook готовности Core сохраняет object key, ставит `delete_after = ready_at + interval '1 month'` и публикует `lesson.recorded`.

Фоновая задача удаляет объект и помечает metadata удалённой после одного месяца. Legal hold либо явное административное исключение требует отдельного решения и audit record.

## События

| Событие | Направление | Семантика |
|---|---|---|
| `user.registered` | потребляет | Создать минимальный профиль идемпотентно по `event_id` |
| `booking.confirmed` | публикует | Бронь и participant holds подтверждены |
| `booking.cancelled` | публикует | Итог refund/capture по каждому participant |
| `lesson.started` | публикует | Зафиксировано фактическое начало |
| `lesson.completed` | публикует | Статус, attendance и денежный outcome каждого ученика |
| `lesson.recorded` | публикует | Запись готова; payload содержит безопасную ссылку на metadata, не публичный URL |
| `payment.captured` | потребляет | Обновить локальную payment projection |
| `enrollment.expiring` | потребляет | Показать ограничение доступа без копирования Learning domain |

Все публикации идут через transactional outbox в Kafka. Consumers фиксируют Kafka offset только после commit и дедуплицируют по `event_id`.

## Инварианты данных

- У каждой подтверждённой participant row есть неизменяемые price snapshot и hold reference.
- У каждого `lesson_id` отдельная доска и LiveKit room.
- Attendance хранится по участнику; reconnect создаёт новый интервал.
- Recording object не доступен без проверки membership.
- `delete_after` записи равен одному месяцу после готовности.
- Доменное изменение и outbox event находятся в одной Postgres transaction.

## Масштабирование и безопасность

Реплики Core stateless. Postgres хранит доменное состояние; S3 хранит bytes. Reconciler завершает PENDING booking sagas. Kafka consumers используют одну group на projection.

Core не логирует private chat body, presigned URL, tokens или recording content. Admin operations требуют permission и audit. `education_db` credentials не дают доступа к БД других сервисов.
