# Auth Service

Auth Service владеет учётными записями, способами входа, сессиями и моделью доступа. Сервис регистрирует пользователей, подтверждает email, выполняет вход по паролю и OAuth, восстанавливает доступ, выдаёт и отзывает токены, управляет учебными и административными ролями и публикует JWKS.

Это единственный сервис, который выдаёт access-токены. Gateway и профильные сервисы проверяют подпись локально по JWKS. `VerifyToken` RPC отсутствует. Актуальная продуктовая модель задана в [Архитектуре](../../Архитектура.md) и документах [«Роли и разрешения»](../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Роли%20и%20разрешения.md) и [«Сессии и безопасность»](../../Бизнес%20анализ/Админ-панель/Пользователи%20и%20доступ/Сессии%20и%20безопасность.md). Эволюция ранней модели зафиксирована в [ADR-007](../adr/ADR-007-auth-identity-access-evolution.md).

Диаграммы раннего среза: [C3 — компоненты](../diagrams/c4/c3-auth.puml) · [ER — auth_db](../diagrams/db/auth-db.puml) · [Sequence — регистрация](../diagrams/sequence/auth-registration.puml) · [Sequence — ротация refresh-токена](../diagrams/sequence/auth-refresh-rotation.puml). До обновления диаграмм этот документ имеет приоритет при расхождениях.

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| identities, уникальностью email и жизненным циклом аккаунта | Профилями, ФИО, аватарами и биографиями — Core Education |
| Учебными ролями `student`, `teacher`, `representative` | Заявкой и верификацией преподавателя — Core Education; Auth только применяет доверенное решение о роли |
| Отдельными staff identities, административными ролями и каталогом permissions | Предметной авторизацией по членству в уроке, чате, задании, курсе, файле — владелец данных |
| Парольными credentials и OAuth connections | Связью представитель–ученик — Core Education; Auth создаёт приглашённую identity |
| Подтверждением email, приглашениями и восстановлением доступа | Доставкой email и in-app уведомлений — Notification Service |
| Сессиями, refresh-токенами и их ротацией | Внешним rate limiting и CAPTCHA — Gateway |
| Ключами подписи access JWT и JWKS | Room JWT и LiveKit-токенами — Core Education |
| Auth audit records и transactional outbox | Единым представлением межсервисного журнала — read model административного контура |

Главное правило доступа: **roles и permissions описывают возможности, membership подтверждает доступ к конкретным данным**. Наличие `teacher` или `education_lesson_view` не даёт доступ к произвольному уроку.

## Identity и роли

### Пользовательские identities

Один пользовательский аккаунт имеет одну или несколько учебных ролей:

- `student`;
- `teacher`;
- `representative`.

Учебные роли могут совмещаться. `student` выдаётся при регистрации ученика и не отзывается отдельно от удаления аккаунта. `teacher` может появиться при регистрации, но право публикации в каталоге и приёма бронирований определяется статусом заявки в Core Education. `representative` является самостоятельной ролью; связь с подопечным хранит Core Education.

### Staff identities

Сотрудник — отдельный тип identity. Staff identity:

- создаётся только приглашением владельца `auth_permission_edit`;
- входит через отдельный Gateway route и password flow;
- не использует OAuth;
- имеет одну или несколько административных ролей;
- не может одновременно иметь учебную роль;
- всегда сохраняет хотя бы одну административную роль.

Система поставляется с ролями `superadmin` и `manager`. `superadmin` системная и неизменяемая; она включает весь каталог permissions. Остальные роли конфигурируются в БД. Итоговые permissions сотрудника равны объединению permissions всех назначенных ролей. Прямые назначения permission сотруднику отсутствуют.

Инварианты:

- нельзя удалить назначенную административную роль;
- нельзя сохранить `*_edit` без соответствующего `*_view`;
- нельзя отозвать `superadmin` у последнего активного суперадминистратора;
- последнего активного суперадминистратора нельзя заблокировать или удалить;
- `auth_permission_edit` эквивалентен возможности получить полный административный доступ и всегда журналируется.

## Access token и `init`

Access JWT подписывается Ed25519 и содержит:

- `sub`, `jti`, `iat`, `exp`, `iss`, `aud`;
- `identity_type`: `user` или `staff`;
- `roles`: массив кодов ролей;
- `permissions`: готовый плоский массив эффективных permissions;
- `session_id`.

Access TTL — около 10 минут. При изменении ролей новый состав применяется при следующем refresh. Для немедленного применения сотрудник с правом отзыва завершает сессии субъекта.

Gateway формирует клиентский `init` из доверенных сервисов. Auth предоставляет access context: `identity_id`, `identity_type`, `status`, `email_confirmed`, `roles`, `permissions`, `session_id` и `active_cabinet`. Клиент использует roles и permissions только для интерфейса. Gateway и профильный сервис повторно проверяют permission на каждом запросе.

## API

### gRPC `auth.v1` / следующая совместимая версия

Контракт меняется contract-first. Совместимые поля добавляются в `auth.v1`; несовместимая смена wire-формы требует `auth.v2`.

| Группа | RPC | Назначение |
|---|---|---|
| Authentication | `Register` | Создать user identity в `pending_email`, credential, учебную роль и первую сессию |
| Authentication | `Login`, `StaffLogin` | Вход пользователя или сотрудника по паролю без раскрытия причины отказа |
| Authentication | `Refresh`, `Logout` | Ротация refresh-токена и отзыв текущей session family |
| Access | `GetAccessContext`, `SetActiveCabinet` | Вернуть данные Auth для Gateway `init` и запомнить выбранный кабинет |
| Email | `ConfirmEmail`, `ResendEmailConfirmation` | Подтвердить email или повторно запросить письмо |
| Password | `RequestPasswordReset`, `ResetPassword`, `ChangePassword` | Восстановить или сменить пароль |
| Email change | `RequestEmailChange`, `ConfirmEmailChange`, `CancelEmailChange` | Безопасно сменить email с подтверждением нового адреса |
| Sessions | `ListSessions`, `RevokeSession`, `RevokeOtherSessions`, `RevokeAllSessions` | Просмотр и завершение своих или административно выбранных сессий |
| OAuth | `ExchangeOAuthCode`, `LinkOAuthIdentity`, `UnlinkOAuthIdentity` | Вход, привязка и отвязка Яндекс ID, VK ID и Google |
| Invitations | `InviteStaff`, `InviteStudent`, `AcceptInvitation`, `ResendInvitation`, `RevokeInvitation` | Создать staff identity или приглашённого ученика и задать пароль |
| Roles | `GrantLearningRole`, `RevokeLearningRole` | Доверенное управление учебными ролями |
| Admin access | `ListAdminRoles`, `CreateAdminRole`, `UpdateAdminRole`, `DuplicateAdminRole`, `DeleteAdminRole` | Управление конфигурируемыми административными ролями |
| Admin access | `AssignStaffRoles`, `ListPermissions` | Назначить staff roles и получить каталог permissions |
| Lifecycle | `BlockIdentity`, `UnblockIdentity`, `ScheduleDeletion`, `CancelDeletion` | Управление статусом аккаунта |
| Administration | `SearchIdentities`, `GetIdentity` | Выдать Auth-owned часть списка и карточки пользователя для Gateway aggregation |
| Administration | `StartImpersonation`, `EndImpersonation` | Создать и завершить read-only impersonation context на 30 минут |
| Delivery | `GetPasswordResetDelivery`, `GetDeliveryPayload` | Защищённо выдать Notification Service одноразовые данные для password reset и других delivery requests |

RPC управления чужой identity требуют actor context и соответствующий permission. Gateway аутентифицирует пользователя, но Auth повторно проверяет permission и бизнес-инварианты.

Ошибки login, recovery и OAuth linking не раскрывают наличие email, статус аккаунта или уже существующую связь. `Logout` и повторное завершение сессии идемпотентны.

### HTTP

| Endpoint | Назначение |
|---|---|
| `GET /.well-known/jwks.json` | Публичный набор NEXT, ACTIVE и RETIRING signing keys |
| `GET /healthz` | Liveness процесса |
| `GET /readyz` | Readiness зависимостей, включая auth_db и Kafka |
| `GET /metrics` | Prometheus metrics |

Публичные REST routes `/auth/*` принадлежат Gateway и отображаются на gRPC Auth.

## События и Notification Service

Auth публикует события в Kafka topic `trajectory.auth.events.v1` через transactional outbox. Доставка at-least-once; `event_id` стабилен при повторной публикации. Ключ Kafka message выбирается по `identity_id`, чтобы сохранить порядок событий одной identity.

| Событие | Назначение |
|---|---|
| `user.registered` | Core Education создаёт профиль идемпотентно |
| `identity.email_confirmation.requested` | Notification отправляет ссылку подтверждения |
| `password.reset.requested` | Notification отправляет ссылку восстановления |
| `identity.email_change.requested` | Notification сообщает старому адресу и подтверждает новый |
| `identity.invitation.requested` | Notification отправляет приглашение сотруднику или ученику |
| `identity.security_changed` | Notification создаёт in-app уведомление о смене пароля, email или OAuth connection |
| `identity.new_device_login` | Notification сообщает о входе с нового устройства |
| `identity.session_compromised` | Notification сообщает о refresh-token reuse |
| `identity.roles_changed` | Заинтересованные read models инвалидируют access context |
| `identity.status_changed` | Сервисы прекращают операции для заблокированной или удаляемой identity |

Email confirmation token, password reset token, invitation token и готовые URL не попадают в Kafka, логи или DLQ. Событие содержит `delivery_request_id`. Для совместимости password reset использует `GetPasswordResetDelivery`; остальные сценарии используют общий `GetDeliveryPayload`. Повторный запрос с тем же authenticated delivery context идемпотентен в ограниченном окне. Доступ разрешён только workload identity Notification Service.

Auth не отправляет email и in-app уведомления самостоятельно. Auth не зависит от успешной доставки после фиксации бизнес-транзакции: pending outbox сохраняет событие, а Notification повторяет обработку идемпотентно по `event_id`.

## Данные: auth_db

Ниже показана целевая логическая схема. Существующие T01–T11 migrations не переписываются; новые таблицы и ограничения добавляются эволюционными migrations.

```sql
CREATE TABLE identities (
    id                 uuid PRIMARY KEY,
    identity_type      text NOT NULL, -- USER | STAFF
    email              citext UNIQUE,
    status             text NOT NULL, -- PENDING_EMAIL | ACTIVE | BLOCKED | DELETION_SCHEDULED | DELETED
    email_confirmed_at timestamptz,
    blocked_at         timestamptz,
    deletion_at        timestamptz,
    anonymized_at      timestamptz,
    created_at         timestamptz NOT NULL,
    updated_at         timestamptz NOT NULL
);

CREATE TABLE user_identities (
    identity_id uuid PRIMARY KEY REFERENCES identities(id)
);

CREATE TABLE staff_identities (
    identity_id uuid PRIMARY KEY REFERENCES identities(id),
    display_name text NOT NULL
);

CREATE TABLE identity_learning_roles (
    identity_id uuid NOT NULL REFERENCES user_identities(identity_id),
    role_code   text NOT NULL, -- student | teacher | representative
    created_at  timestamptz NOT NULL,
    PRIMARY KEY (identity_id, role_code)
);

CREATE TABLE permissions (
    code        text PRIMARY KEY,
    description text NOT NULL,
    dangerous   boolean NOT NULL DEFAULT false
);

CREATE TABLE admin_roles (
    id          uuid PRIMARY KEY,
    code        text UNIQUE NOT NULL,
    name        text UNIQUE NOT NULL,
    description text,
    system      boolean NOT NULL DEFAULT false,
    created_at  timestamptz NOT NULL,
    updated_at  timestamptz NOT NULL
);

CREATE TABLE admin_role_permissions (
    role_id         uuid NOT NULL REFERENCES admin_roles(id),
    permission_code text NOT NULL REFERENCES permissions(code),
    PRIMARY KEY (role_id, permission_code)
);

CREATE TABLE staff_role_assignments (
    identity_id uuid NOT NULL REFERENCES staff_identities(identity_id),
    role_id     uuid NOT NULL REFERENCES admin_roles(id),
    created_at  timestamptz NOT NULL,
    PRIMARY KEY (identity_id, role_id)
);

CREATE TABLE credentials (
    identity_id   uuid PRIMARY KEY REFERENCES identities(id),
    password_hash text NOT NULL,
    updated_at    timestamptz NOT NULL
);

CREATE TABLE oauth_connections (
    id               uuid PRIMARY KEY,
    identity_id      uuid NOT NULL REFERENCES user_identities(identity_id),
    provider         text NOT NULL, -- YANDEX | VK | GOOGLE
    provider_subject text NOT NULL,
    provider_email   citext,
    created_at       timestamptz NOT NULL,
    UNIQUE (provider, provider_subject)
);

CREATE TABLE sessions (
    id              uuid PRIMARY KEY, -- refresh family_id
    identity_id     uuid NOT NULL REFERENCES identities(id),
    login_method    text NOT NULL,
    user_agent      text NOT NULL,
    last_ip         inet NOT NULL,
    active_cabinet  text,
    created_at      timestamptz NOT NULL,
    last_active_at  timestamptz NOT NULL,
    expires_at      timestamptz NOT NULL,
    revoked_at      timestamptz
);

CREATE TABLE refresh_tokens (
    id          uuid PRIMARY KEY,
    identity_id uuid NOT NULL REFERENCES identities(id),
    family_id   uuid NOT NULL REFERENCES sessions(id),
    token_hash  bytea UNIQUE NOT NULL,
    status      text NOT NULL, -- ACTIVE | ROTATED | REVOKED
    replaced_by uuid REFERENCES refresh_tokens(id),
    expires_at  timestamptz NOT NULL,
    created_at  timestamptz NOT NULL
);
CREATE UNIQUE INDEX refresh_one_active_per_family
    ON refresh_tokens (family_id) WHERE status = 'ACTIVE';

CREATE TABLE auth_action_requests (
    id             uuid PRIMARY KEY,
    identity_id    uuid REFERENCES identities(id),
    kind           text NOT NULL, -- EMAIL_CONFIRMATION | PASSWORD_RESET | EMAIL_CHANGE | INVITATION
    token_hash     bytea UNIQUE NOT NULL,
    pending_email  citext,
    attempt_count  integer NOT NULL DEFAULT 0,
    expires_at     timestamptz NOT NULL,
    consumed_at    timestamptz,
    created_at     timestamptz NOT NULL
);

CREATE TABLE audit_identity_links (
    audit_ref   uuid PRIMARY KEY,
    identity_id uuid UNIQUE NOT NULL REFERENCES identities(id)
);

CREATE TABLE auth_audit_log (
    id                uuid PRIMARY KEY,
    event_type        text NOT NULL,
    actor_audit_ref   uuid,
    actor_roles       text[] NOT NULL,
    object_type       text NOT NULL,
    object_audit_ref  uuid,
    reason            text,
    before_state      jsonb,
    after_state       jsonb,
    result            text NOT NULL,
    request_id        text NOT NULL,
    ip                inet,
    created_at        timestamptz NOT NULL
);

CREATE TABLE signing_keys (...); -- NEXT | ACTIVE | RETIRING | RETIRED
CREATE TABLE outbox (...);       -- event_id, topic, message_key, payload, published_at
```

`user_identities` и `staff_identities` взаимоисключающие; это проверяет deferred constraint trigger. `identities.email` становится `NULL` только при анонимизации, поэтому уникальный индекс допускает повторное использование освобождённого адреса.

Audit rows неизменяемы. Они хранят псевдонимные `audit_ref`, а связь с identity вынесена в `audit_identity_links`. Анонимизация удаляет mapping, не изменяя журнал. Пароли, hashes, raw tokens, OAuth access tokens, KEK и private signing keys в audit и outbox запрещены.

## Сессии и токены

Сессия соответствует одной refresh family и одному устройству. Login создаёт новую session и сохраняет user-agent, последний IP и login method. Refresh обновляет `last_active_at`, `last_ip` и продлевает inactivity expiry на 30 дней.

На identity допускается не более 10 активных sessions. Создание одиннадцатой session атомарно отзывает session с самым старым `last_active_at`. Операция сериализуется блокировкой identity row, чтобы конкурентные login не нарушили лимит.

Ротация refresh-токена выполняется одной транзакцией: старый ACTIVE становится ROTATED, новый ACTIVE создаётся в той же family. Повторное использование ROTATED или REVOKED token считается компрометацией и отзывает всю family. В БД хранятся только SHA-256 hashes raw refresh tokens.

Пользователь может завершить любую свою session, кроме текущей, или все другие sessions. Сотрудник с `auth_session_revoke` может завершить одну или все sessions пользователя с обязательной причиной. Блокировка, успешный password reset, завершение удаления и компрометация завершают все sessions. Смена пароля и OAuth connection завершает все, кроме текущей; смена email завершает включая текущую.

Первый login с новым устойчивым device fingerprint публикует `identity.new_device_login`. Fingerprint используется только для сравнения известных устройств и не заменяет session id. Auth не пытается определять геолокацию по IP.

## Регистрация и подтверждение email

Self-register принимает email, пароль, обязательные согласия и одну роль: `student`, `teacher` или `representative`. Создаётся user identity в `PENDING_EMAIL`. До подтверждения login разрешён, но access context ограничивает доступ экраном подтверждения email.

Confirmation link живёт 24 часа. Повторная отправка разрешена не чаще одного раза в 60 секунд. Подтверждение переводит identity в `ACTIVE`. Подтверждённый email OAuth provider сразу создаёт `ACTIVE` identity.

Неподтверждённая identity старше 30 дней автоматически анонимизируется. Регистрация создаёт identity, credential, роль, action request, auth audit row и outbox records в одной транзакции. Сбой Notification не откатывает регистрацию.

## Пароль и восстановление доступа

Пароль содержит 8–128 Unicode characters, минимум одну букву и одну цифру, не совпадает с нормализованным email и отсутствует в локальном списке распространённых паролей. До Argon2id дополнительно применяется предел 1024 UTF-8 bytes; transport и use case проверяют одинаковое значение.

`RequestPasswordReset` всегда отвечает одинаково. Для существующей незаблокированной identity Auth создаёт одноразовый token на 1 час и outbox event. Ограничение: не более 3 запросов на адрес в час, 5 в сутки и 5 попыток на одну ссылку. У OAuth-only identity flow впервые задаёт пароль.

Успешный reset атомарно потребляет request, меняет credential, отзывает все sessions, пишет audit и outbox. Повторное использование ссылки даёт безопасный идемпотентный результат без enumeration.

Принудительный reset сотрудником с `auth_credential_reset` доступен только identity с password credential. Он немедленно отзывает все sessions и создаёт отдельную ссылку задания пароля на 24 часа.

## Ограничение частоты

Gateway применяет IP-based limits и adaptive CAPTCHA: 5 неудачных login на пару email+IP за 15 минут, 20 login attempts с одного IP за 15 минут и 5 registrations с одного IP в час. Блокировка rate limit не меняет identity status.

Auth применяет stateful limits, которые нельзя доверять только внешнему caller: email confirmation resend — один раз в 60 секунд; password recovery — 3 раза в час и 5 раз в сутки на normalized email; email change — 3 раза в сутки; один recovery request принимает не более 5 попыток. Gateway передаёт нормализованный client IP в authenticated metadata для audit и составного login limit.

## OAuth

Поддерживаются Яндекс ID, VK ID и Google. Auth валидирует `state`, authorization code, issuer, audience, nonce и provider signature по правилам конкретного провайдера. Provider access/refresh tokens не сохраняются дольше обмена, если они не нужны для последующих provider API.

Ключ связи — пара `(provider, provider_subject)`, а не email. Совпадение provider email с существующей identity не создаёт связь автоматически. Пользователь подтверждает владение существующим аккаунтом паролем или одноразовой email-ссылкой. Один внешний аккаунт связан только с одной platform identity.

Нельзя отвязать последний способ входа. Привязка или отвязка завершает все sessions, кроме текущей, и создаёт security notification. Staff identity никогда не использует OAuth.

## Жизненный цикл аккаунта

Статусы: `PENDING_EMAIL`, `ACTIVE`, `BLOCKED`, `DELETION_SCHEDULED`, `DELETED`.

- `BLOCKED`: login и refresh возвращают общий `Unauthenticated`; активные sessions отзываются.
- `DELETION_SCHEDULED`: хранится дата удаления через 30 дней; login разрешает только экран отмены удаления.
- `DELETED`: credentials, OAuth connections, sessions и прямые персональные идентификаторы удалены; email освобождён; identity остаётся технической ссылкой без ПДн.
- Автоочистка `PENDING_EMAIL` старше 30 дней выполняет ту же анонимизацию.

Блокировка, разблокировка, планирование и отмена удаления требуют permission и причины. До планирования удаления оркестратор получает подтверждение Billing/Core Education об отсутствии блокирующего баланса, пакета и открытого финансового обращения. Auth сохраняет idempotency key этого решения, но не читает чужие БД. Ограничение последнего активного `superadmin` проверяется внутри той же транзакции под блокировкой соответствующих rows.

## Административное чтение и impersonation

Auth предоставляет поиск и чтение только своих полей: identity id, email, тип, roles, status, login methods, created/last-login timestamps и session summary. ФИО, телефон, обучение и финансы Gateway получает у сервисов-владельцев. Выгрузка Auth-owned данных требует `auth_user_export` и создаёт audit record с фильтрами и числом строк.

Read-only impersonation требует `auth_user_impersonate`, причины и активной staff identity. Auth выдаёт отдельный JWT на 30 минут с claims `impersonator_id`, `subject_id`, `scope=read_only` и запретом room access. Impersonation не создаёт user session и не выдаёт refresh token. Gateway и сервисы отклоняют любые mutation requests с таким token. Начало, ручное завершение и expiry журналируются.

## Audit внутри границы сервиса

Auth атомарно журналирует изменения данных, которыми владеет:

- блокировку, разблокировку, удаление и отмену удаления;
- выдачу и отзыв учебных и административных ролей;
- создание, изменение, дублирование и удаление admin roles;
- приглашения сотрудников;
- принудительный reset password и отзыв sessions;
- изменение email, password и OAuth connections;
- refresh-token reuse;
- отказы по permission и бизнес-инвариантам;
- выгрузку auth audit records.

Успешное изменение и audit row находятся в одной транзакции. Отказ, который не меняет domain state, получает отдельную audit transaction. Если обязательную запись создать нельзя, административная операция завершается ошибкой. Audit append-only; SQL UPDATE/DELETE для журнала запрещены сервисной DB role.

Каждый сервис ведёт audit своих операций в своей БД. Межсервисный экран журнала строится из событий или отдельной read model и не может требовать общей ACID-транзакции нескольких service-owned DB.

Оперативный audit хранится 12 месяцев, затем архивируется по общей retention policy. Архивирование не даёт приложению права изменять или удалять отдельные records.

## Ключи подписи

Lifecycle сохраняется без изменений:

```
NEXT --(JWKS cache warm)--> ACTIVE --> RETIRING --(all access tokens expired)--> RETIRED
```

Private Ed25519 material зашифрован AES-256-GCM под KEK. Ровно один ACTIVE key обеспечивается partial unique index. NEXT публикуется до активации, RETIRING остаётся в JWKS до истечения всех подписанных им access tokens, RETIRED не публикуется.

## Инварианты безопасности

- Auth не логирует passwords, raw refresh tokens, action tokens, OAuth tokens, KEK и private keys.
- Refresh/action tokens хранятся только как SHA-256 hashes.
- Ошибки не раскрывают существование email, статус identity или способ входа.
- Dummy password verify использует введённый password и сопоставимую стоимость Argon2id.
- Все mutation events создаются в одной транзакции с domain state через outbox.
- Kafka delivery at-least-once; consumers идемпотентны по `event_id`.
- Admin permissions не заменяют membership checks владельца данных.
- Изменение доступа ограничивается коротким access TTL; принудительный отзыв sessions применяет новый доступ немедленно.

## Эволюция реализации

T01–T11 создали ранний срез: одна роль в `identities.role`, статусы ACTIVE/BLOCKED, password login, refresh family, signing keys, JWKS и базовый outbox. Эти migrations и опубликованные данные не переписываются.

Переход к целевой модели выполняется forward-only:

1. Добавить новые таблицы и nullable/discriminator columns.
2. Backfill существующих identities и их одиночных ролей в user/staff subtype и role bindings.
3. Перевести чтение на новую модель с проверкой эквивалентности результатов.
4. Перевести запись и token claims на roles/permissions arrays.
5. Удалить чтение legacy columns только отдельной migration после наблюдаемого периода совместимости.

Для опубликованного `auth.v1` несовместимая форма не меняется на месте. Contracts repository определяет, достаточно ли additive fields или требуется `auth.v2`.
