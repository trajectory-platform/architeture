# Realtime Service

Realtime терминирует WebSocket-соединения комнат уроков и реализует три потока: доска (Yjs-relay), чат комнаты, presence. Сознательно **почти stateless**: вся «память» — лог апдейтов в realtime_db и TTL-ключи в Redis; в самих инстансах состояния нет, поэтому они равнозначны, масштабируются горизонтально и переживают рестарты. Сервер не интерпретирует содержимое доски — он relay с персистентностью ([ADR-002](../adr/ADR-002-crdt-sync-path-a.md)).

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-realtime.puml) · [ER — realtime_db](../diagrams/db/realtime-db.puml) · [Sequence — синхронизация вайтборда](../diagrams/sequence/whiteboard-sync.puml) · [Sequence — вход в комнату](../diagrams/sequence/lesson-join.puml)

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| Лог Yjs-апдейтов и снапшоты (realtime_db, архив — MinIO) | Содержимое доски — блобы опаковые, мерж делают клиенты (Yjs) |
| Membership / presence комнат (Redis, эфемерно) | Авторизация входа — room JWT минтит Core (`JoinLesson`); Realtime только проверяет подпись |
| Fan-out сообщений между инстансами (Redis Pub/Sub) | Персистентность чата — Core (`SaveMessage`); Realtime лишь форвардит |
| Доставка `room.event` участникам | Медиа — LiveKit, мимо нашего бэкенда |

## API

### WebSocket (единственный публичный интерфейс)

Подключение: `wss://.../rooms/{lesson_id}?token=<room JWT>` — короткоживущий (~1 мин на подключение) токен от Core; подпись валидируется локально по JWKS, без сетевого вызова. Одно соединение на участника. Сообщения — envelope с типом ([05 — Realtime и вайтборд](../architecture/05-realtime-whiteboard.md)):

| Тип | Направление | Содержание |
|---|---|---|
| `board.update` | клиент ⇄ сервер | Yjs-апдейт, opaque bytes |
| `board.sync_request` | клиент → сервер | state vector клиента |
| `board.sync_response` | сервер → клиент | последний snapshot + tail |
| `chat.message` | клиент ⇄ сервер | сообщение чата комнаты |
| `presence.update` | клиент ⇄ сервер | курсор, инструмент, online-статус |
| `room.event` | сервер → клиент | вход/выход участника, завершение урока |

gRPC-сервера у Realtime **нет** — никто не зовёт его синхронно.

### Исходящие вызовы

| Вызов | Когда |
|---|---|
| Core `SaveMessage` (gRPC) | Membership и durable commit до ack/broadcast; идемпотентен по client-generated `message_id` и проверяет совпадение payload |
| Core `AuthorizeRoomJoin` (gRPC) | Fallback-проверка членства; штатный путь — room JWT, без сетевого вызова |

### События

| Событие | Направление | Реакция |
|---|---|---|
| `lesson.started` | потребляет (опционально) | Пре-создание структур комнаты (подписка на Redis-канал, прогрев) — оптимизация, не корректность |

Realtime ничего не публикует в Kafka: его факты (presence, апдейты доски) — не межсервисные события ([03](../architecture/03-communication.md)).

## Компоненты

| Компонент | Ответственность |
|---|---|
| WS Hub | Upgrade, валидация room JWT (JWKS), envelope-роутинг по типам |
| Room Registry | Membership/presence в Redis (TTL-ключи, heartbeat), подписка на каналы комнат |
| Board Log | `append(seq, blob)` → персист → fan-out; `sync_request` → snapshot + tail |
| Chat Forwarder | Core SaveMessage с membership → durable commit → ack/broadcast |
| Presence | Курсоры, инструменты, online; best-effort, потеря некритична |
| Board Compactor | **Отдельный Node-сайдкар**: согласованный snapshot+tail → Y.Doc/GC → проверенное поколение и атомарный pruning. Изолирован — читает/пишет только realtime_db ([ADR-002](../adr/ADR-002-crdt-sync-path-a.md)) |

## Данные: realtime_db

ER-диаграмма: [realtime-db.puml](../diagrams/db/realtime-db.puml). DDL — в [06 — Данные](../architecture/06-data-storage.md): `board_rooms` с постоянным last_seq/snapshot_seq, `board_updates`, `board_snapshots` и receipts update_id. Каждый `lesson_id` обозначает новую независимую доску; состояние предыдущего урока не продолжается автоматически.

Механика seq: атомарный UPDATE board_rooms.last_seq + INSERT update/receipt в одной транзакции. Счётчик постоянный и не зависит от pruning; MAX(seq) очищаемого журнала не используется. Повтор update_id/digest возвращает сохранённый seq.

Инварианты:

- **Сначала персист, потом fan-out.** Апдейт, который увидел другой участник, уже в логе — падение инстанса между append и broadcast теряет только доставку (клиент догонит через sync), не данные.
- **`sync_response` = снапшот + `updates WHERE seq > upto_seq`.** Сервер шлёт надмножество, не дифф — клиентский Yjs идемпотентно отбрасывает применённое. Переплата трафиком на reconnect — осознанная цена ([ADR-002](../adr/ADR-002-crdt-sync-path-a.md)).
- **После pruning snapshot обязателен.** Атомарное переключение snapshot_seq, согласованное чтение и backup пары snapshot+tail — по [05](../architecture/05-realtime-whiteboard.md). Полный replay возможен только с проверенным полным архивом.

### Redis (эфемерное)

| Использование | Структуры |
|---|---|
| Membership / presence: `room:{id}:members` | TTL-ключи, sets; heartbeat от инстансов |
| Fan-out апдейтов и presence между инстансами | Pub/Sub канал per-room |
| Кэш JWKS | строка с TTL |

Потеря Redis = переподключения и краткая потеря presence, **не потеря данных** — доска в realtime_db, чат в Core.

## Ключевые потоки

- **Апдейт доски** — [whiteboard-sync.puml](../diagrams/sequence/whiteboard-sync.puml): `board.update` → append в лог → publish в Redis-канал → broadcast всеми инстансами своим клиентам (кроме отправителя).
- **Reconnect / холодный вход** — `sync_request` (state vector) → `sync_response` (snapshot + tail) → офлайн-правки клиента уходят после синхронизации, конфликты разрешает CRDT. Тот же поток для нового участника и после падения инстанса.
- **Чат комнаты** — Core authorization/commit, затем durable ack и broadcast; история после урока — обычный REST к Core.
- **Компакция** — сайдкар по порогу (N апдейтов / M байт) сливает snapshot + tail → новый снапшот; старые апдейты — удалить или в MinIO.
- **Падение инстанса** — клиенты переподключаются через Nginx к любому другому; тот подписывается на Redis-каналы комнаты и отдаёт snapshot + tail. Теряются только TCP-соединения.

## Масштабирование и отказоустойчивость

- Инстансы равнозначны: нет sticky sessions, нет лидера, нет внутреннего состояния. N инстансов держат соединения, Redis Pub/Sub связывает их per-room.
- Узкие места по порядку появления: количество одновременных WS-соединений на инстанс (решается добавлением инстансов), пропускная способность Redis Pub/Sub (решается шардированием каналов), запись лога в Postgres (решается батчингом append'ов).
- Деградации изящные: без Redis — комнаты на одном инстансе продолжают работать; без Postgres новые durable updates не подтверждаются и не рассылаются; клиент сохраняет pending состояние.

## Безопасность

- Вход — только по room JWT (клеймы `lesson_id`, `user_id`, `role`), выданному Core после проверки членства и окна урока. Realtime **не ходит в базы за правами** — всё в токене ([07](../architecture/07-security.md)).
- Соединение привязано к `lesson_id` из токена: сообщения маршрутизируются только в эту комнату, подмена комнаты невозможна без нового токена.
- Блобы доски не валидируются (опаковые) — авторизация только на уровне «кто в комнате»; это осознанное ограничение ([ADR-002](../adr/ADR-002-crdt-sync-path-a.md), последствия).
- Право закрыть комнату — только у преподавателя, и enforce'ится в Core/LiveKit, не в Realtime.

## Чего здесь нет — и почему

- **Состояния в инстансах.** Всё восстановимое — в Redis и realtime_db; иначе падение инстанса = потеря данных и сложный failover.
- **CRDT-мержа на сервере.** Нет зрелого Yjs на Go, и он не нужен: апдейты коммутативны и идемпотентны, мержат клиенты ([ADR-002](../adr/ADR-002-crdt-sync-path-a.md)).
- **Персистентности чата.** Один владелец у всех чатов — Core; Realtime вызывает Core до ack/broadcast; pending retry выполняется с тем же message_id, история Core восстанавливает пропуски.
- **Kafka для board-апдейтов.** Это не межсервисные события, а клиентский поток: Redis Pub/Sub дешевле, а durable-гарантию даёт лог в Postgres, не шина.

## Гарантии первого среза

MVP 1.0 уже проверяет durable ack чата/доски и потерю Redis-подписки без разрыва WS. Snapshot→live использует subscribe/buffer → consistent snapshot+tail → buffered tail. Watermark из БД выявляет пропуски; повторная подписка всегда запускает sync. Board receipt не удаляется вместе с bytes до окончания replay window. Подробный протокол, лимиты и GC — [05](../architecture/05-realtime-whiteboard.md). Групповые инструменты и material export появляются в 2.0; нагрузочный профиль всего продукта подтверждается в 2.1.
