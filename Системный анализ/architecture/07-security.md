# 07 — Безопасность

## Аутентификация и identity

Auth Service владеет identity, способами входа, sessions, ролями, permissions и signing keys. Поддерживаются password и OAuth через Яндекс ID, VK ID и Google. Совпадение OAuth email не связывает аккаунты автоматически.

User identity совмещает роли `student`, `teacher`, `representative`. Staff identity отделена и получает одну или несколько административных ролей. Учебные и staff роли не совмещаются.

Статусы identity: `PENDING_EMAIL`, `ACTIVE`, `BLOCKED`, `DELETION_SCHEDULED`, `DELETED`. Неподтверждённая identity через 30 дней анонимизируется.

## Токены и sessions

| Token | Выдаёт | Назначение |
|---|---|---|
| Access JWT | Auth | Короткий доступ к REST/gRPC context |
| Refresh token | Auth | Ротация внутри session family; raw token хранится только у клиента |
| Room JWT | Core | Вход по WebSocket в один `lesson_id` |
| LiveKit token | Core | WebRTC grants одной комнаты |

Access JWT подписывается Ed25519 и содержит `sub`, `jti`, `identity_type`, `roles[]`, `permissions[]`, `session_id`, `iat`, `exp`, `iss`, `aud`. Gateway и сервисы проверяют подпись локально по JWKS.

Session соответствует refresh family. Максимум 10 активных sessions на identity. Inactivity TTL — 30 дней. Повторное использование ROTATED/REVOKED refresh token отзывает всю family и создаёт security event.

## Авторизация

Правило: **roles и permissions описывают возможность, membership подтверждает доступ к объекту**.

| Ресурс | Проверка владельца |
|---|---|
| Комната урока | Core: строка `lesson_participants`; представитель и сотрудник не входят автоматически, staff-наблюдение вне MVP |
| Карточка урока и административные действия | Core: отдельные `schedule_*` permissions; не создают членство и доступ в комнату |
| Запись | Участник — по membership; сотрудник — `schedule_lesson_view` + `schedule_recording_view`, причина и audit каждого открытия; удаление — отдельное `schedule_recording_delete` |
| Чат пользователей | Core: membership; просмотр карточки и административные permissions не открывают личный или учебный чат |
| Support ticket | Core: membership либо отдельное admin permission |
| Курс, задание, тест, материал | Learning: ownership/enrollment/group membership |
| Баланс и ledger | Billing: владелец account либо finance permission |
| Notification | Notification: `notification.user_id == caller.identity_id` |

Gateway может отклонить запрос без permission, но сервис-владелец всегда повторяет проверку. Клиентские roles/permissions используются только для UI.

Системная роль `superadmin` неизменяема. Нельзя отозвать роль, заблокировать или удалить последнего активного superadmin. `*_edit` требует соответствующего `*_view`.

## Вход в комнату

`JoinLesson` проверяет membership и временное окно. Core выдаёт room JWT и LiveKit token только для конкретного `lesson_id`. Realtime и LiveKit не читают доменные БД. Каждый урок имеет новую отдельную доску.

## Секреты и персональные данные

- Password хранится как Argon2id hash; до hash действует предел 1024 UTF-8 bytes.
- Refresh, reset, confirmation и invitation tokens хранятся только как SHA-256 hashes.
- Private signing keys зашифрованы KEK из secret store.
- Raw tokens, password, OAuth provider tokens, KEK и private keys запрещены в logs, events и DLQ.
- Notification не получает полный профиль или private chat content.
- Reset и confirmation events содержат `delivery_request_id`; URL выдаётся Notification через workload-authenticated Auth RPC.
- Recording требует явного согласия участников; отсутствие или отзыв согласия останавливает запись, возобновление требует согласия всех присутствующих. Срок хранения — календарный месяц от готовности файла. Досрочное удаление общей записи требует решения суперадминистратора, причины и уведомления участников; запрос одного участника не удаляет её автоматически.

Подробные правила административного просмотра и удаления — в [карточке урока](../../Бизнес%20анализ/Админ-панель/Расписание%20и%20уроки/Карточка%20урока.md), каталог разрешений — в [Ролях и разрешениях](../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Роли%20и%20разрешения.md).

Kafka topics с PII используют отдельные ACL и короткий retention. DLQ содержит только разрешённый безопасный payload либо redacted metadata.

## Защита периметра

- HTTPS/WSS снаружи; внутренние gRPC и БД не публикуются.
- LiveKit webhooks проходят signature verification.
- Nginx и Gateway применяют независимые rate limits.
- Login, refresh, password recovery и денежные mutations fail-closed при недоступности stateful limiter.
- S3 доступ выдаётся коротким presigned URL после проверки права, размера и назначения.
- Service DB credentials изолированы; cross-service joins запрещены.

## Audit и анонимизация

Каждый сервис пишет audit своих изменений в своей БД и той же транзакции, что бизнес-изменение. Общий экран строится асинхронной read model, без распределённой ACID transaction.

Auth audit использует псевдонимный `audit_ref`. Анонимизация удаляет mapping identity-to-audit, не изменяя append-only журнал. Audit никогда не содержит secrets.
