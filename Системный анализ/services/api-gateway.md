# API Gateway

Единственная REST-точка входа для SPA: терминирует REST, транслирует в gRPC внутренних сервисов. Полностью **stateless** — ни базы, ни сессий; всё состояние — кэши и счётчики в Redis. Масштабируется репликами за Nginx без координации.

Жёсткое правило: **никакой бизнес-логики**. Если в обработчике появляется `if` про предметную область — он переезжает в сервис-владелец ([02 — Сервисы](../architecture/02-services.md)). Gateway знает про HTTP, токены, ключи идемпотентности и коды ошибок — не про уроки, обучение или деньги.

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-api-gateway.puml) · Gateway виден во всех sequence-диаграммах: [booking-saga](../diagrams/sequence/booking-saga.puml), [lesson-join](../diagrams/sequence/lesson-join.puml), [auth-registration](../diagrams/sequence/auth-registration.puml), [auth-refresh-rotation](../diagrams/sequence/auth-refresh-rotation.puml). ER-диаграммы нет — у Gateway нет базы данных.

## Зона ответственности

| Делает | Не делает |
|---|---|
| REST → gRPC трансляция (маршрутизация по сервисам) | Бизнес-логика, доменные проверки — владельцы данных |
| Локальная валидация access JWT (JWKS, без сетевого вызова) | Выдача токенов — Auth; room JWT / LiveKit-токены — Core |
| Cookie ⇄ gRPC: refresh-токен из httpOnly cookie в поле запроса Auth и обратно | Хранение сессий — их нет; состояние сессии в refresh-цепочке Auth |
| Прокидка `Idempotency-Key`: заголовок → gRPC metadata | Проверка идемпотентности — Core (bookings) и Billing (holds, idempotency_keys) |
| Rate limiting (Redis), CORS, единый формат ошибок | Терминация TLS — Nginx |
| Публикация OpenAPI-спеки | WebSocket — Realtime и Notification (Nginx маршрутизирует `/ws/rooms/*` и `/ws/notifications` мимо Gateway) |

## Маршрутизация REST → gRPC

| Префикс REST | Сервис | gRPC |
|---|---|---|
| `/auth/*` (register, login, refresh, logout) | Auth | `auth.v1` |
| `/teachers/*`, `/slots/*`, `/bookings/*`, `/lessons/*`, `/chats/*`, `/tickets/*`, `/profiles/*`, `/recordings/*` | Core | `education.v1` |
| `/courses/*`, `/groups/*`, `/enrollments/*`, `/homework/*`, `/tests/*`, `/materials/*`, `/progress/*` | Learning | `learning.v1` |
| `/balance`, `/topup` | Billing | `billing.v1` |
| `/notifications/*` | Notification | `notification.v1`: inbox, отметки прочтения, настройки |

Правила каждого исходящего вызова ([03 — Взаимодействие](../architecture/03-communication.md)):

- **Deadline обязателен** — `context.WithTimeout` из бюджета HTTP-запроса; распространяется по цепочке автоматически.
- **Retry — только идемпотентных методов** (помечены в proto и retry-конфиге клиента); `CreateBooking` с ключом — можно, метод без ключа — нет.
- **Идентичность пользователя** — в gRPC metadata (подписанный контекст из валидированного JWT), не «системный суперпользователь»: сервис-получатель повторно проверяет права на свои данные ([07](../architecture/07-security.md)).

## Аутентификация на границе

1. **Access JWT** — из заголовка `Authorization: Bearer`. Подпись проверяется локально: JWKS-кэш (in-memory + Redis), обновление по TTL и по промаху `kid`. Auth по сети не дёргается ([07 — Безопасность](../architecture/07-security.md)).
2. **Refresh-токен** — httpOnly Secure cookie. Gateway — единственное место, где cookie разворачивается: на `/auth/refresh` и `/auth/logout` токен из cookie уходит полем gRPC-запроса в Auth; новая пара из ответа — обратно: access в тело ответа, refresh в `Set-Cookie`. На `Unauthenticated` от Auth cookie сбрасывается.
3. **Блэклист jti** (опционально, Redis) — проверка отозванных access-токенов после детекта кражи refresh ([auth-refresh-rotation](../diagrams/sequence/auth-refresh-rotation.puml)).

Публичные маршруты без токена: `/auth/register`, `/auth/login`, `/auth/refresh`, OpenAPI-спека, health.

## Rate limiting

Лимитирование — два независимых слоя, потому что Gateway не защищает Nginx и не должен быть единственной защитой при отказе Redis:

1. **Nginx:** фиксированный local rate limit и лимит одновременных соединений по доверенному client IP. Этот слой дёшев и остаётся доступен без Redis.
2. **Gateway:** Redis Lua token-bucket (одно атомарное чтение/изменение), ключ включает группу маршрута и субъект. В ответе всегда `429 Too Many Requests`, `Retry-After` и rate-limit headers.

| Группа | Ключи Gateway | Режим при отказе Redis |
|---|---|---|
| Login, register, password reset | IP-префикс + HMAC(normalized email) для login/reset | **fail-closed** (`503`) после local Nginx-лимита: нельзя превращать деградацию Redis в окно для credential stuffing |
| Refresh/logout | IP-префикс + HMAC(refresh token) | **fail-closed**: предотвращает storm/replay cookie |
| Topup, booking, отмена, upload URL | `user_id` + маршрут; дополнительно IP-префикс | **fail-closed** для mutation; ключ идемпотентности не заменяет лимит |
| Обычное чтение | `user_id` или IP-префикс + маршрут | fail-open с метрикой и алертом, если Nginx продолжает ограничивать периметр |
| WebSocket | отдельный лимит upgrade/подключений в Nginx и Realtime | не использует общий HTTP bucket |

IP извлекается только из заголовка, который переписывает доверенный Nginx/LB; нельзя доверять произвольному `X-Forwarded-For`. IPv6 нормализуется до /64. Численные лимиты и burst задаются конфигурацией и подбираются нагрузочным тестом по отдельным группам — единый «100 запросов в минуту» не подходит для auth, чтения и видео-комнаты.

## Формат ошибок

Единый JSON-конверт для всех ошибок: `{code, message, details?}`; `message` безопасен для показа пользователю, внутренние детали — только в логи/трейс. Маппинг кодов gRPC → HTTP:

| gRPC | HTTP | Пример |
|---|---|---|
| `InvalidArgument` | 400 | Некорректное тело запроса |
| `Unauthenticated` | 401 | Невалидный/истёкший токен (+ сброс refresh-cookie на auth-путях) |
| `PermissionDenied` | 403 | Не участник урока/чата |
| `NotFound` | 404 | Нет такой брони |
| `AlreadyExists` | 409 | Email занят; повтор `Idempotency-Key` с другим телом |
| `FailedPrecondition` | 409 | Слот занят (booking-сага) |
| `ResourceExhausted` | 402 | Недостаточно средств (`PlaceHold`) |
| `DeadlineExceeded` | 504 | Внутренний вызов не уложился в бюджет |
| `Unavailable` | 503 | Сервис недоступен (+ `Retry-After`) |

429 — собственный код Gateway (rate limiter), не из gRPC.

## OpenAPI

Gateway публикует OpenAPI-спеку; из неё генерируется TypeScript-клиент фронтенда — REST-типы руками не пишутся ([03](../architecture/03-communication.md)). Спека — производная от proto-контрактов: REST-маршруты и DTO следуют за `education.v1`/`auth.v1`/`billing.v1`, изменения контракта ловит `buf breaking` в CI.

## Наблюдаемость

- **`trace_id` рождается здесь** — HTTP-мидлвара OTel; контекст уезжает в gRPC metadata и доезжает до SQL и Kafka ([08 — Деплой](../architecture/08-deployment-observability.md)).
- `/metrics` — RED по REST-маршрутам и по исходящим gRPC-методам (p50/p95/p99), счётчики rate-limit отказов, JWKS-промахи.
- Структурированные логи: `trace_id`, маршрут, `user_id`, статус; тела запросов и PII не логируются.

## Чего здесь нет — и почему

- **Базы данных.** Нечего владеть: токены проверяются криптографически, идемпотентность хранят владельцы операций, сессий нет. Redis — кэш и счётчики, потеря = деградация, не потеря данных.
- **Агрегации ответов (BFF).** Один REST-вызов → один gRPC-вызов. Композиция данных для экранов — забота фронтенда или будущего BFF-слоя; склейка в Gateway = скрытая бизнес-логика и распределённые join'ы.
- **WebSocket-прокси.** Realtime-трафик идёт от Nginx напрямую в Realtime: лишний хоп в latency-критичном пути не нужен, авторизация там своя (room JWT).
- **Трансформации данных.** DTO = proto-сообщения 1:1 (через OpenAPI-генерацию); «подкрутить поле для фронта» — путь к рассинхрону контрактов.
