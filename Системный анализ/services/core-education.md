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
| booking | `CreateBooking`, `CancelBooking`, `ListBookings`, `ProposeReschedule`, `AcceptReschedule`, `RejectReschedule` (перенос с 2.0) | Мутации идемпотентны; price фиксируется до `PlaceHold` |
| booking | `CreateRecurringRule`, `CancelRecurringRule` | Каждое вхождение проходит обычную booking saga |
| lessons | `JoinLesson`, `EndLesson`, `GetLesson`, `ListLessons` | Комната — по membership; административные карточки и действия — по отдельным permissions, без выдачи доступа в комнату |
| recordings | `StartRecording`, `StopRecording`, `GetRecording`, `DeleteRecording` (с 2.0) | Старт по membership и согласиям; административный просмотр — отдельное permission, причина и audit; удаление — отдельное permission и решение |
| chats | `ListChats`, `ListMessages`, `SendMessage`, `SaveMessage` (internal), `MarkRead` | Membership проверяется для каждого вызова |
| support | `CreateTicket`, `ReplyTicket`, `ChangeTicketStatus` | Admin actions требуют granular permission |
| uploads | `CreateUpload`, `CompleteUpload`, `CreateDownload` | Короткий presigned URL, scoped object key и lifecycle проверки до READY |

## Бронирование и цена

Административные сценарии определены в [бизнес-анализе расписания и уроков](../../Бизнес%20анализ/Админ-панель/Расписание%20и%20уроки/README.md). Сотрудник действует под собственной identity с инициатором и причиной. Индивидуальный административный перенос не требует подтверждения участников, групповой требует согласия преподавателя и всех записанных учеников; старое время сохраняется до применения, новое временно удерживается на срок ответа. Изменение доступности не отменяет подтверждённые брони скрытно; пересечения преподавателя и учеников проверяются при подтверждении.

Пробное занятие допускает нулевую цену без денежного hold. Право ограничено парой ученик–преподаватель и временно занято подтверждённой или обрабатываемой пробной бронью; используется после проведения или неявки ученика после бесплатного окна. Подробные исходы — в [Управлении бронями](../../Бизнес%20анализ/Админ-панель/Расписание%20и%20уроки/Управление%20бронями.md).

Цена фиксируется **с каждого ученика**, а не делится между участниками группы. `booking_participants` хранит `price_amount`, `currency` и `hold_reference_id` для каждого ученика. Изменение тарифа после подтверждения не меняет существующую бронь.

Для индивидуальной брони есть один participant. Для групповой — до 7 учеников и отдельный `PlaceHold` на каждого. Ошибка резерва одного ученика отклоняет только его место; подтверждённые места остальных не меняются.

Double booking преподавателя исключает constraint по `teacher_id + time_range`. Повторное место того же ученика исключает уникальный ключ `(booking_id, student_id)`.

## Отмена и неявка

Core вычисляет результат cancellation policy по snapshot правил брони:

- до окончания бесплатного окна — полный возврат;
- после окна — частичный возврат либо полное списание;
- при неявке ученика hold списывается, если бесплатное окно отмены уже пропущено.

При неявке преподавателя выполняется полный возврат по затронутым participant rows. Технический и неопределённый исход направляется на ручной разбор без автоматического capture до решения. Менеджер может утвердить обычный итог и завершить зависший урок по отдельным permissions; исправление уже проведённых денег и произвольная компенсация требуют финансового действия суперадминистратора. Исходные attendance и денежные записи не переписываются. Автоматическое подтверждение проведения использует настраиваемый порог совместного присутствия преподавателя и каждого ученика. Стартовые значения порогов и сроков выбираются в настройках, не подставляются consumer-ом.

Core публикует итоговые суммы в `booking.cancelled` или participant outcome в `lesson.completed`. Billing не пересчитывает расписание и policy.

## Урок и доска

Один урок имеет одну отдельную комнату и одну новую доску. `lesson_id` является room ID и ключом board log в Realtime. Состояние предыдущей доски доступно как материал, но не становится начальным состоянием нового урока автоматически.

`JoinLesson` возвращает короткий room JWT и LiveKit token после проверки membership и временного окна. Для группы проверяется строка `lesson_participants`, а не роль пользователя.

## Запись урока

Перед стартом Core фиксирует согласие каждого участника. LiveKit Egress пишет файл напрямую в S3. После webhook готовности Core сохраняет object key, ставит `delete_after = ready_at + interval '1 month'` и публикует `lesson.recorded`.

При входе участника без согласия запись останавливается до включения его медиа в запись; отзыв consent также останавливает запись. Возобновление требует согласия всех присутствующих. Уже записанные сегменты сохраняются до удаления по общим правилам; каждый готовый файл имеет свой `ready_at` и неизменяемый срок хранения.

Для staff-просмотра Core проверяет `schedule_lesson_view`, `schedule_recording_view`, причину и доступность конкретного файла, фиксирует audit до выдачи доступа. Этот сценарий не создаёт membership и не даёт room JWT, доступа к чатам и иным материалам.

Фоновая задача закрывает доступ и удаляет объект после одного календарного месяца от готовности. Досрочный запрос одного участника не удаляет общую запись автоматически: решение принимает обладатель `schedule_recording_delete`, указывает причину, уведомляются все участники. Удаление аккаунта не заменяет это решение. Legal hold либо исключение из срока хранения требует отдельного будущего решения и в текущие административные сценарии MVP не входит.

## Верификация и публикация преподавателя

Core хранит профессиональный профиль, проверяемые версии и состояние публикации; Auth остаётся владельцем роли `teacher`. [Верификация](../../Бизнес%20анализ/Админ-панель/Преподаватели/Верификация.md) проверяет анкету, документ об образовании и опыт; собеседование факультативно. Публикация выполняется отдельно после одобрения. Изменения предметов, образования и опыта проходят повторную проверку, прежняя одобренная версия действует до решения.

Снятие с публикации запрещает новые брони, включая новые recurring-вхождения, но сохраняет существующие уроки. Отзыв верификации отменяет будущие уроки с полным возвратом всех резервов; идущий урок сохраняется до штатного завершения. Повторное одобрение не восстанавливает публикацию и отменённые брони автоматически. Действия имеют отдельные permissions и audit; владение сервисами не меняется.

## События

| Событие | Направление | Семантика |
|---|---|---|
| `user.registered` | потребляет | Создать минимальный профиль идемпотентно по `event_id` |
| `booking.confirmed` | публикует | Бронь и participant holds подтверждены |
| `booking.cancelled` | публикует | Итог refund/capture по каждому participant |
| `lesson.started` | публикует | Зафиксировано фактическое начало |
| `lesson.completed` | публикует | Статус, attendance и денежный outcome каждого ученика |
| `lesson.recorded` | публикует | Запись готова; payload содержит безопасную ссылку на metadata, не публичный URL |
| `recording.deleted` | публикует | Запретить доступ и инвалидировать metadata материала в Learning |
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

## Релизные гарантии и отсутствующие исходы

1.0 поставляет одну participant row, confirmation профилей, block/verification, минимальную policy, индивидуальный lifecycle и durable room chat. Курсы/группы/recording/reschedule — 2.0, полный набор — 2.1. Целевая схема не означает раннюю реализацию всех модулей.

Booking row резервирует время преподавателя один раз; добавление места группы проверяет вместимость под row lock. Participant ID — hold_reference_id; для каждого PlaceHold свой стабильный ключ. PENDING/cancel intent, CAS и компенсация неизвестного исхода — [04](../architecture/04-sagas-and-consistency.md).

SendMessage/SaveMessage используют один application port: membership → INSERT/проверка message_id и payload → commit → durable result. Realtime подтверждает и рассылает только после него. История восстанавливает потерянный fan-out.

LiveKit webhooks сохраняются в durable inbox с уникальным provider event ID. EndLesson, timer и reconciliation применяют один state machine с блокировкой урока. room_finished — наблюдение, не самостоятельное основание capture. Правила неявки преподавателя, технического срыва и основания «состоялся» согласованы в [разборе проблемных занятий](../../Бизнес%20анализ/Админ-панель/Расписание%20и%20уроки/Разбор%20проблемных%20занятий.md); числовые значения DEC-01 остаются настройками до приёмки, consumer не изобретает их.

Recording (2.0): REQUESTED→RECORDING→PROCESSING→READY либо FAILED; удаление DELETING→DELETED. До READY object_key может отсутствовать. Retry Egress использует постоянный recording ID; готовность публикуется один раз, повтор webhook безопасен. Поздний вход и отзыв consent останавливают запись по правилам выше; сценарии действий представителя в DEC-04 уточняются отдельно. При expiry/одобренном решении об удалении Core сначала запрещает выдачу URL, затем удаляет object/derivatives с повторами и публикует recording.deleted; Learning инвалидирует material metadata. Snapshot retention не сбрасывается при повторном ready webhook.

## File lifecycle по срезам

CreateUpload создаёт metadata PENDING_UPLOAD с серверным ключом, owner/object scope, declared type/size и сроком. После прямой загрузки CompleteUpload проверяет фактический размер/checksum, переводит объект в QUARANTINED и запускает проверку содержимого; до READY presigned download другим пользователям не выдаётся. Проверка типа по заголовку запроса не заменяет проверку байтов. REJECTED и истёкшие незавершённые загрузки удаляются worker-ом, квота возвращается идемпотентно. Путь DELETING→DELETED прекращает выдачу новых ссылок до физической уборки.

Это минимальная безопасность используемых avatars/chat attachments уже в 1.0 по ADR-005; полный набор namespaces, previews и политики материалов поставляются в 2.0. RPC CompleteUpload использует тот же uploads application port, что CreateUpload; job статусы и отказ видны клиенту. Владельцы Learning material проверяют свою membership перед upload/download и не читают Core tables.
