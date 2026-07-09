# Auth Service

Auth Service владеет идентичностью пользователей: регистрация, логин, refresh-токены, роли (`student` / `teacher` / `admin`), ключи подписи JWT и публикация JWKS. Это единственный сервис, который умеет *выдавать* access-токены; *проверяют* их все остальные сами, локально по JWKS — никакого `VerifyToken` RPC (см. [07 — Безопасность](../architecture/07-security.md)).

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-auth.puml) · [ER — auth_db](../diagrams/db/auth-db.puml) · [Sequence — регистрация](../diagrams/sequence/auth-registration.puml) · [Sequence — ротация refresh-токена](../diagrams/sequence/auth-refresh-rotation.puml)

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| identities (email, роль, статус) | Профили (имя, аватар, биография) — Core, создаются по `user.registered` |
| credentials (argon2id-хэши паролей) | Авторизация доступа к ресурсам (ownership) — проверяет владелец данных |
| refresh-токены и их ротация | Room JWT и LiveKit-токены — минтит Core в `JoinLesson` |
| Ключи подписи и JWKS | Rate limiting логина — Gateway (Redis, по IP) |

Правило из [07](../architecture/07-security.md): **roles в токене, ownership в данных**. Auth кладёт роль в клейм access JWT; «может ли этот teacher видеть этот урок» решает Core по своим таблицам.

## API

### gRPC `auth.v1` (вызывает только Gateway)

| RPC | Назначение | Ошибки |
|---|---|---|
| `Register` | Создание identity + credentials, выдача первой пары токенов | `AlreadyExists` (email занят) |
| `Login` | Проверка пароля (argon2id), выдача пары access + refresh | `Unauthenticated` (неверная пара) — без уточнения, что именно неверно |
| `Refresh` | Ротация refresh-токена, новый access | `Unauthenticated` (истёк / отозван / повторное использование) |
| `Logout` | Отзыв refresh-цепочки (family) | — (идемпотентен) |

Все RPC — с deadline на стороне вызывающего ([03 — Взаимодействие](../architecture/03-communication.md)). Refresh-токен ходит между SPA и Gateway в httpOnly Secure cookie; в gRPC передаётся как поле запроса.

### HTTP

| Endpoint | Назначение |
|---|---|
| `GET /.well-known/jwks.json` | Публичные ключи подписи. Кэшируется Gateway и Realtime (Redis + in-memory, обновление по TTL и по `kid`-промаху) |

## События

| Событие | Направление | Механика |
|---|---|---|
| `user.registered` | публикует | Через transactional outbox в той же транзакции, что и INSERT identity. Core (durable consumer) создаёт профиль; обработка идемпотентна по `user_id` |

Auth ничего не потребляет из JetStream.

## Данные: auth_db

ER-диаграмма: [auth-db.puml](../diagrams/db/auth-db.puml). DDL упрощён (без индексов аудита):

```sql
CREATE TABLE identities (
    id         uuid        PRIMARY KEY,
    email      citext      UNIQUE NOT NULL,
    role       text        NOT NULL,              -- student | teacher | admin
    status     text        NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE | BLOCKED
    created_at timestamptz NOT NULL DEFAULT now()
);

-- Отдельно от identities: хэш не тянется обычными SELECT'ами,
-- а при появлении OAuth-провайдеров у identity может не быть пароля вовсе.
CREATE TABLE credentials (
    identity_id   uuid        PRIMARY KEY REFERENCES identities(id),
    password_hash text        NOT NULL,           -- argon2id
    updated_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE refresh_tokens (
    id          uuid        PRIMARY KEY,          -- = jti токена
    identity_id uuid        NOT NULL REFERENCES identities(id),
    family_id   uuid        NOT NULL,             -- цепочка ротаций одного login
    token_hash  bytea       NOT NULL UNIQUE,      -- sha256; сам токен не храним
    status      text        NOT NULL,             -- ACTIVE | ROTATED | REVOKED
    replaced_by uuid        REFERENCES refresh_tokens(id),
    expires_at  timestamptz NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
-- в один момент в семье активен ровно один токен
CREATE UNIQUE INDEX refresh_one_active_per_family
    ON refresh_tokens (family_id) WHERE status = 'ACTIVE';

CREATE TABLE signing_keys (
    kid         text        PRIMARY KEY,
    alg         text        NOT NULL,             -- EdDSA | RS256
    public_jwk  jsonb       NOT NULL,             -- отдаётся в JWKS
    private_key bytea       NOT NULL,             -- зашифрован KEK из secret-хранилища
    status      text        NOT NULL,             -- NEXT | ACTIVE | RETIRING | RETIRED
    not_after   timestamptz,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE outbox (                              -- стандартная схема, см. 03
    id           bigserial   PRIMARY KEY,
    aggregate_id uuid        NOT NULL,
    subject      text        NOT NULL,             -- 'trajectory.user.registered'
    payload      bytea       NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

Инварианты:

- **Токены не хранятся в открытом виде** — только `sha256`-хэш. Утечка дампа auth_db не даёт рабочих refresh-токенов.
- **Одна семья — один активный токен** (partial unique index). Ротация = `ROTATED` старому + INSERT нового в одной транзакции.
- **Баланс ролей в БД, не в коде**: `role` — колонка identity, в токен попадает как клейм при каждой выдаче.

## Ключевые потоки

### Регистрация → профиль в Core

Одна транзакция: `INSERT identities` + `INSERT credentials` + `INSERT outbox('user.registered')`. Relay публикует в JetStream после коммита; Core создаёт профиль идемпотентно. Сразу выдаётся пара токенов — отдельный login после регистрации не нужен. Диаграмма: [auth-registration.puml](../diagrams/sequence/auth-registration.puml).

### Login

1. `SELECT identities JOIN credentials` по email; argon2id-проверка пароля. На неверный email и неверный пароль — одинаковый `Unauthenticated` и сопоставимое время ответа (dummy-hash при отсутствии email).
2. Новая семья refresh-токенов: `family_id = uuid`, первый токен `ACTIVE`.
3. Access JWT подписывается текущим `ACTIVE`-ключом (`kid` в заголовке токена); TTL ~10 мин. Refresh — httpOnly Secure cookie, TTL — дни.

### Ротация refresh и детект кражи

Диаграмма: [auth-refresh-rotation.puml](../diagrams/sequence/auth-refresh-rotation.puml).

1. `Refresh(token)` → поиск по `token_hash`.
2. Токен `ACTIVE` и не истёк → транзакция: старый `→ ROTATED` (+ `replaced_by`), новый `ACTIVE` в той же семье; новый access JWT.
3. Токен `ROTATED` или `REVOKED` → **повторное использование = признак кражи**: вся семья (`WHERE family_id = ...`) → `REVOKED`, jti живых access-токенов — в Redis-блэклист (опционально, см. [07](../architecture/07-security.md)). Ответ `Unauthenticated`, Gateway сбрасывает cookie.
4. Истёкший токен → просто `Unauthenticated` (не кража — нормальное протухание).

### Logout

`UPDATE refresh_tokens SET status='REVOKED' WHERE family_id = ...`. Идемпотентен. Окно жизни уже выданного access ограничено его коротким TTL.

### Ротация ключей подписи

Жизненный цикл `signing_keys.status`:

```
NEXT ──(прогрев JWKS-кэшей, ≥ TTL кэша)──► ACTIVE ──(новый ключ стал ACTIVE)──► RETIRING ──(истекли все подписанные токены)──► RETIRED
```

- `NEXT`: ключ уже опубликован в JWKS, но им ещё не подписывают — кэши потребителей успевают его подобрать.
- `ACTIVE`: им подписываются новые access JWT (ровно один в момент времени).
- `RETIRING`: не подписывает, но остаётся в JWKS, пока живы подписанные им токены (≥ access TTL).
- `RETIRED`: удалён из JWKS.

## Чего здесь нет — и почему

- **`VerifyToken` RPC** — подпись асимметричная, валидация локальная у потребителей; Auth не точка отказа на каждый запрос.
- **Сессии в Redis** — состояние сессии целиком в refresh-цепочке в Postgres; Redis у Auth опционален (только блэклист jti при отзыве).
- **ACL/permissions в токене** — только роль; ownership проверяет владелец данных.
