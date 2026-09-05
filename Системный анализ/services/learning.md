# Learning Service

Learning Service владеет содержанием обучения: курсами, программами, группами, зачислениями, домашними заданиями, тестами, материалами и прогрессом. Сервис не владеет расписанием, статусом урока или деньгами.

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-learning.puml) · [Модули и зависимости](../diagrams/components/learning-modules.puml) · [ER — learning_db](../diagrams/db/learning-db.puml).

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| Courses, modules, topics и публикация программы | Profiles и identity — Core/Auth |
| Groups и enrollments | Slots, bookings, lessons и attendance source — Core |
| Homework lifecycle и submissions | Ledger, holds, packages и payouts — Billing |
| Tests, question bank, attempts и scores | WebSocket room и board log — Realtime |
| Materials и связи с course/topic/lesson | Bytes объектов — S3 |
| Progress, achievements и learning projections | Доставка inbox/email/push — Notification |

## Внутренние пакеты

Каждый пакет ниже является бизнес-модулем с собственными application ports, domain model и repository adapter. Модуль может вызывать другой модуль только через его публичный application port. Прямые импорты internal-пакетов и SQL к таблицам другого модуля запрещены. Допустимые зависимости показаны на [диаграмме модулей](../diagrams/components/learning-modules.puml).

| Пакет | Ответственность |
|---|---|
| `courses` | Курсы, modules, topics, версии программы и публикация |
| `groups` | Группы, преподаватели, вместимость до 7 учеников |
| `enrollments` | Доступ на период, продление, остановка и expiry jobs |
| `homework` | Шаблоны, assignments, submissions, review и score |
| `tests` | Question bank, tests, attempts, auto/manual review |
| `materials` | Metadata файлов и связь с course/topic/lesson |
| `progress` | Attendance projection, topic completion, grades и achievements |
| `outbox` | Публикация Kafka events |
| `consumers` | Идемпотентная обработка Core/Billing events |

## API

Контракты находятся в `proto/trajectory/learning/v1/`.

| Область | Основные RPC |
|---|---|
| courses | `CreateCourse`, `UpdateCourse`, `PublishCourse`, `GetCourse`, `ListCourses` |
| groups | `CreateGroup`, `AddGroupMember`, `RemoveGroupMember`, `ListGroups` |
| enrollments | `GrantEnrollment`, `ExtendEnrollment`, `RevokeEnrollment`, `ListEnrollments` |
| homework | `CreateHomework`, `AssignHomework`, `SubmitHomework`, `ReviewHomework` |
| tests | `CreateTest`, `StartAttempt`, `SubmitAnswer`, `FinishAttempt`, `ReviewAttempt` |
| materials | `CreateMaterial`, `AttachMaterial`, `ListMaterials` |
| progress | `GetProgress`, `ListAchievements`, `GetProgressJournal` |

Все mutating RPC проверяют ownership либо granular admin permission. Создание assignment, попытки теста и зачисления идемпотентны.

## События

| Событие | Направление | Реакция |
|---|---|---|
| `lesson.completed` | потребляет | Обновить attendance projection и progress по каждому ученику |
| `lesson.recorded` | потребляет | Создать material metadata со сроком доступности записи |
| `payment.captured` | потребляет | Обновить paid access projection, если продукт требует оплаты |
| `homework.assigned` | публикует | Уведомить ученика |
| `homework.submitted` | публикует | Уведомить преподавателя |
| `homework.reviewed` | публикует | Обновить progress и уведомить ученика |
| `enrollment.granted` | публикует | Зафиксировать новый доступ |
| `enrollment.expiring` | публикует | Предупредить Core/Notification о скором окончании |
| `progress.milestone_reached` | публикует | Создать пользовательское уведомление |

Learning не вызывает Core для обработки завершённого урока. Kafka обеспечивает развязку. Consumer сохраняет `processed_events.event_id` в одной транзакции с projection и фиксирует offset после commit.

## Материалы урока

Финальный board snapshot и запись урока создаются владельцами Core/Realtime. Learning получает `lesson.recorded` либо другой material event и сохраняет только metadata и S3 object key. Доступ к материалу проверяется по enrollment и участию в уроке.

Запись урока хранится один месяц. `expires_at` material равен `delete_after` из Core. Learning перестаёт выдавать ссылку после expiry, даже если фоновое удаление S3 задержалось.

## Прогресс

Progress обновляется по `(lesson_id, student_id)` и не зависит от порядка событий разных учеников. Повтор `lesson.completed` становится no-op. Неявка хранится как отдельный attendance outcome и не увеличивает completed progress, даже если Billing списал hold по cancellation policy.

## Инварианты данных

- Group capacity не превышает 7 активных учеников.
- Enrollment имеет один явный status и период доступа.
- Submission принадлежит одному assignment и ученику.
- Test attempt фиксирует version теста на момент старта.
- Material bytes не проходят через Learning; в БД хранится object key.
- `processed_events.event_id` уникален.
- Доменное изменение и outbox event фиксируются в одной transaction.

## Масштабирование и безопасность

Learning масштабируется stateless-репликами. Kafka partitions распределяют event load. Background jobs используют row locking и `SKIP LOCKED`.

Сервис не логирует ответы ученика, private material content или presigned URLs. Преподаватель видит только свои courses/groups. Ученик видит только действующие enrollments. Административный доступ требует permission и audit.
