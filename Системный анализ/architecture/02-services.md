# 02 — Сервисы и владение данными

Система состоит из семи сервисов. Принцип границ: сервис = зона владения данными + протокольная роль. Диаграммы: [C2 — контейнеры](../diagrams/c4/c2-containers.puml), [C3 — Core](../diagrams/c4/c3-core-education.puml), [модули Core](../diagrams/components/core-education-modules.puml), [C3 — Learning](../diagrams/c4/c3-learning.puml), [модули Learning](../diagrams/components/learning-modules.puml), [C3 — Notification](../diagrams/c4/c3-notification.puml), [C3 — Realtime](../diagrams/c4/c3-realtime.puml).

## Сводная таблица

| Сервис | Владеет данными | API наружу | API внутрь (gRPC) | Публикует события | Потребляет события |
|---|---|---|---|---|---|
| **API Gateway** | — (stateless) | REST (OpenAPI) | — (только клиент gRPC) | — | — |
| **Auth Service** | identities, credentials, roles, permissions, sessions, reset requests, signing keys | JWKS endpoint через Gateway | `auth.v1`: регистрация, login, OAuth, refresh, logout, восстановление, управление доступом | `user.registered`, `password.reset.requested`, `identity.*` | — |
| **Core Education** | profiles, availability, slots, bookings, lessons, attendance, recordings, chats, support | LiveKit webhooks | `education.v1`: профили, расписание, брони, уроки, записи, чаты, поддержка | `booking.confirmed`, `booking.cancelled`, `lesson.started`, `lesson.completed`, `lesson.recorded` | `user.registered`, `payment.captured`, `enrollment.expiring` |
| **Learning Service** | courses, groups, enrollments, homework, tests, materials, progress | — | `learning.v1`: курсы, группы, зачисления, задания, тесты, материалы, прогресс | `homework.*`, `enrollment.*`, `progress.milestone_reached` | `lesson.completed`, `lesson.recorded`, `payment.captured` |
| **Billing Service** | ledger entries, holds, packages, subscriptions, tariffs, commissions, payouts, idempotency keys | — | `billing.v1`: резервы, баланс, пополнение, пакеты, выплаты | `payment.captured`, `hold.released`, `payout.*` | `booking.cancelled`, `lesson.completed` |
| **Realtime Service** | board updates и snapshots; presence в Redis | WebSocket | — (gRPC-клиент Core) | — | `lesson.started` (опциональный прогрев) |
| **Notification Service** | preferences, inbox, templates, deliveries, processed events | WebSocket для живых in-app уведомлений | `notification.v1`: ListNotifications, MarkRead, UpdatePreferences | `notification.created`, `notification.delivered`, `notification.failed` | доменные события Auth, Core, Learning и Billing |

## API Gateway

Единственная REST-точка входа для SPA. Gateway транслирует REST в gRPC, валидирует JWT по JWKS, передаёт identity и `Idempotency-Key`, применяет rate limiting и единый формат ошибок. Он маршрутизирует запросы к Auth, Core, Learning, Billing и Notification, но не содержит бизнес-правил.

## Auth Service

Владеет идентичностью и доступом: способами входа, сессиями, несколькими ролями аккаунта и granular permissions административных ролей. Auth подписывает access tokens и публикует JWKS. Gateway, Realtime и Notification валидируют токены локально. Подробности: [Auth Service](../services/auth.md) и [07 — Безопасность](07-security.md).

## Core Education Service

Владеет временем и фактом занятия:

- `profiles` — профили пользователей;
- `scheduling` — доступность, исключения, слоты и recurring rules;
- `booking` — бронь, защита от пересечений, резерв средств, перенос и отмена;
- `lessons` — жизненный цикл, посещаемость и `JoinLesson`;
- `recordings` — согласия, управление LiveKit Egress и удаление через один месяц;
- `chats` — личные, групповые и комнатные чаты;
- `support` — обращения и их жизненный цикл;
- `outbox` — публикация фактов в Kafka.

Core создаёт отдельную комнату и новую доску для каждого урока. Цена бронирования фиксируется отдельно для каждого ученика. После подтверждения она не пересчитывается.

## Learning Service

Владеет содержанием обучения:

- `courses` — курсы, программы, темы и публикация;
- `groups` — группы и состав до 7 учеников;
- `enrollments` — доступы и сроки;
- `homework` — создание, выдача, сдача и проверка;
- `tests` — банк вопросов, попытки и аттестации;
- `materials` — файлы курса, урока, записи и snapshots доски;
- `progress` — посещаемость, освоение тем, оценки и достижения.

Learning не определяет время или статус урока. Он идемпотентно реагирует на события Core и Billing. Подробности: [Learning Service](../services/learning.md).

## Billing Service

Владеет деньгами. Ledger append-only. Резерв создаётся на ученика и фиксированную цену его занятия. Для группового урока каждый ученик имеет отдельный hold. При завершении или неявке после пропущенного окна бесплатной отмены Billing захватывает соответствующий hold по событию Core. При своевременной отмене возвращает резерв полностью либо частично согласно зафиксированной политике.

Billing не интерпретирует расписание, посещаемость или правила отмены. Он исполняет полученный денежный результат идемпотентно по `reference_id`.

## Notification Service

Владеет коммуникацией с пользователем. Сервис потребляет Kafka events отдельной consumer group, выбирает получателей и разрешённые каналы, создаёт inbox и delivery jobs. `notification_db` содержит `notification_preferences`, `templates`, `notifications`, `deliveries` и `processed_events`.

- Бизнес-операция не ждёт Notification или внешнего провайдера.
- `(event_id, recipient_id, kind)` уникален; redelivery не создаёт повторную отправку.
- Gateway вызывает `notification.v1`, но не читает `notification_db`.
- Notification терминирует отдельный WebSocket-поток in-app уведомлений. После reconnect клиент восстанавливает пропущенное через `ListNotifications`.
- Сервис хранит минимальный snapshot параметров, но не копирует профиль, пароль, token или содержимое чата.

Подробности: [Notification Service](../services/notification.md).

## Realtime Service

Realtime терминирует WebSocket и обслуживает board updates, room chat transport и presence. Каждому `lesson_id` соответствует отдельный board log и отдельная доска. Инстансы хранят durable board state в `realtime_db`, эфемерное presence и fan-out — в Redis. Чаты сохраняются в Core.

## Что не выделяем в сервисы

| Кандидат | Где живёт | Почему не сервис |
|---|---|---|
| Chat Service | пакет `chats` в Core | Нет отдельного профиля масштабирования; выделение добавит сетевой хоп в каждое сообщение |
| Reporting Service | projections в Core и Learning | Отчёты следуют владению исходными данными; отдельное хранилище на MVP избыточно |
| Media Service | LiveKit | Собственный SFU не является задачей проекта |
| File Service | S3 + адаптеры сервисов-владельцев | Клиент загружает байты напрямую по presigned URL |

Новый сервис появляется только при доказанной причине: иной профиль масштабирования, цикл релизов или команда.
