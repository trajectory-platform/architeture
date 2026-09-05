# Notification Service

Notification Service отвечает за то, чтобы пользователь узнал о уже совершившемся доменном событии. Он не создаёт брони, не меняет баланс и не владеет чатами: его единственная задача — материализовать уведомление и надёжно доставить его по разрешённым каналам.

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-notification.puml) и [ER — notification_db](../diagrams/db/notification-db.puml). Граница сервиса закреплена в [ADR-001](../adr/ADR-001-microservices-granularity.md), правила API — в [ADR-005](../adr/ADR-005-architecture-business-rules.md).

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| Настройками каналов пользователя (in-app, email, push) | Профилем и email пользователя — Auth/Core |
| Inbox-уведомлениями, статусом прочтения, шаблонами и параметрами рендеринга | Бронями, уроками, платежами и их правилами — Core/Billing |
| Очередью доставок, ретраями, дедупликацией и provider reference | WebSocket-подключениями комнат уроков — Realtime |
| WebSocket-потоком живых in-app уведомлений | Бизнес-событиями комнат, расписания, обучения и денег |
| Интеграциями с email/push-провайдерами | Содержимым пользовательских чатов и вложениями |

## Взаимодействие

Notification использует отдельную Kafka consumer group. Он получает `user.registered`, `password.reset.requested`, `identity.*`, `booking.confirmed`, `booking.cancelled`, `lesson.started`, `lesson.completed`, `lesson.recorded`, `payment.captured`, `homework.*`, `enrollment.*` и `progress.milestone_reached`. Все издатели используют transactional outbox; доставка события имеет семантику at-least-once.

Обработка состоит из двух независимых частей:

1. В одной транзакции сервис дедуплицирует событие, определяет получателей и сохраняет inbox-запись с `deliveries` в статусе `PENDING`.
2. Воркеры доставляют PENDING/RETRYING записи в провайдеры с backoff, лимитом параллелизма и лимитом провайдера. Невозможность отправки не откатывает доменное событие и не удерживает Kafka offset после надёжной постановки в `deliveries`.

Уникальный ключ `event_id + recipient_id + kind` обязателен. Consumer фиксирует Kafka offset только после коммита первой транзакции. Создание inbox-записи публикует `notification.created`; успешная внешняя доставка — `notification.delivered`; исчерпание попыток — `notification.failed` без PII и с алертом. Все три события проходят через outbox Notification Service.

Для `password.reset.requested` Notification не получает токен в событии: он по mTLS/workload identity вызывает internal `auth.v1.GetPasswordResetDelivery(reset_request_id)` и получает одноразовый URL только на время отправки. Эта операция не должна попадать в обычные логи или DLQ.

## Пользовательский API

Gateway вызывает будущий `notification.v1` только для пользовательских данных:

| RPC | Назначение |
|---|---|
| `ListNotifications` | Cursor-пагинированный inbox текущего пользователя |
| `MarkRead` | Идемпотентно отмечает своё уведомление прочитанным |
| `UpdatePreferences` | Меняет согласия на каналы и категории |

Вызовы всегда несут identity из Gateway; сервис дополнительно проверяет `notification.user_id == caller.user_id`. Контракт должен появиться в репозитории `contracts` до реализации Gateway или сервиса.

Живые in-app уведомления идут через отдельный `wss://.../ws/notifications` endpoint Notification Service. Сервис локально проверяет access JWT по JWKS и отправляет только уведомления текущего пользователя. Redis Pub/Sub доставляет сигнал экземпляру, который держит соединение; источником истины остаётся inbox в `notification_db`. После reconnect клиент вызывает `ListNotifications`, поэтому потеря эфемерного сигнала не теряет уведомление.

## Данные и приватность

- В `notification_db` хранятся только `user_id`, настройка канала, шаблон, минимальный snapshot параметров и технический статус. Полный профиль и текст приватного чата не копируются.
- Email/push-адреса разрешается получать через выделенную минимальную проекцию либо через Auth/Core по необходимости; они не должны попадать в Kafka payload, логи или dead-letter.
- Шаблоны экранируют пользовательские значения; URLs формируются по allowlist доменов. Вложения не отправляются автоматически.
- Согласие на маркетинговые рассылки отделено от обязательных транзакционных уведомлений и хранится явно.

## Надёжность и масштабирование

Сервис масштабируется репликами в одной Kafka consumer group; partitions распределяются между экземплярами, а `FOR UPDATE SKIP LOCKED` раздаёт доставки воркерам. Внешние провайдеры — главная граница пропускной способности, поэтому для каждого канала нужны отдельные очередь/конкурентность, exponential backoff с jitter и circuit breaker. Метрики: consumer lag, возраст PENDING-доставки, попытки, ошибки и latency по провайдеру, число дедуплицированных событий.
