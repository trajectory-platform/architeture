# Billing Service

Billing владеет деньгами — и только деньгами. Модель — **append-only ledger**: баланс пользователя — это `SUM()` строк леджера, никогда не мутируемая колонка. Billing ничего не знает про уроки, слоты и расписание: он оперирует холдами с внешним `reference_id` (`booking_participant_id`) и реагирует на события. Любое движение денег идемпотентно по `Idempotency-Key`.

Диаграммы: [C3 — компоненты](../diagrams/c4/c3-billing.puml) · [ER — billing_db](../diagrams/db/billing-db.puml) · [Sequence — booking-сага (PlaceHold)](../diagrams/sequence/booking-saga.puml) · [Sequence — завершение урока (CaptureHold)](../diagrams/sequence/lesson-completed-billing.puml)

## Зона ответственности

| Владеет | Не владеет |
|---|---|
| Леджер (ledger_entries): полная неизменяемая история денег | Цены уроков — teacher_profiles в Core; Billing получает сумму в `PlaceHold` |
| Холды: резерв → захват / возврат | Политика отмены (полный/частичный возврат) — Core решает, Billing исполняет payload |
| Идемпотентные ключи денежных операций | Связь «холд ↔ урок» — для Billing `reference_id` опаковый uuid |
| Балансы (вычислимые из леджера) | Профили, идентичность — `account_id` опаковый uuid (identity из Auth) |

Принцип: **Core решает «что», Billing гарантирует «как»** — атомарно, идемпотентно, с полной историей.

## API

### gRPC `billing.v1`

Контракт — `proto/trajectory/billing/v1/billing.proto`. Все мутирующие методы требуют `Idempotency-Key` в gRPC metadata; повтор с тем же ключом возвращает сохранённый результат, повтор с тем же ключом и **другим телом** — `AlreadyExists` ([03](../architecture/03-communication.md)).

| RPC | Вызывает | Назначение | Ошибки |
|---|---|---|---|
| `PlaceHold` | Core (booking-сага, синхронно) | Резерв средств: проверка доступного баланса, `INSERT holds (ACTIVE)` | `ResourceExhausted` (недостаточно средств) |
| `CaptureHold` | консьюмер `lesson.completed` (внутренний) | Холд → пара строк леджера + `payment.captured` | `FailedPrecondition` (холд не ACTIVE — consumer различает повтор результата и конфликт) |
| `ReleaseHold` | консьюмер `booking.cancelled` (внутренний) | Холд закрыт, средства снова доступны | идемпотентен для того же результата; противоположный terminal outcome — conflict |
| `GetBalance` | Gateway | Доступный баланс = `SUM(ledger)` − активные холды | — |
| `TopUp` | Gateway | Пополнение: строка `topup` в леджер | `AlreadyExists` (повтор ключа с другим телом) |

`CaptureHold`/`ReleaseHold` существуют и как RPC (для admin-операций и reconciler'а), но штатный путь — события Kafka, не синхронные вызовы: отставание Billing не должно блокировать Core ([03](../architecture/03-communication.md)).

`TopUp` на MVP — внутренняя операция (тестовые начисления, admin). Интеграция с внешним PSP — будущее отдельное решение: вебхук провайдера → `TopUp` с идемпотентным ключом из id платежа провайдера.

## События

| Событие | Направление | Реакция |
|---|---|---|
| `booking.cancelled` | потребляет | `ReleaseHold(reference_id)`; payload несёт политику: `release_full` / `release_partial(amount)` — частичный возврат = release + строки леджера на штраф |
| `lesson.completed` | потребляет | По каждому participant применить переданный outcome: capture фиксированной цены либо ранее рассчитанный release |
| `payment.captured` | публикует | Через outbox в транзакции захвата; Core помечает урок оплаченным |
| `hold.released` | публикует | Через outbox в транзакции release; для нотификаций/аудита |

Kafka consumers фиксируют offset после успешной транзакции. Обработка идемпотентна: повтор того же event/outcome — no-op. Несовместимый outcome для терминального hold фиксируется как conflict для reconciliation; это не обычный дубликат.

## Данные: billing_db

ER-диаграмма: [billing-db.puml](../diagrams/db/billing-db.puml). DDL `ledger_entries` и `holds` — в [06 — Данные](../architecture/06-data-storage.md); дополнение — таблица идемпотентности не-холдовых операций:

```sql
-- Идемпотентность операций без собственной таблицы-агрегата (TopUp и т.п.).
-- Для PlaceHold ключ живёт прямо в holds.idempotency_key (UNIQUE).
CREATE TABLE idempotency_keys (
    key          text        PRIMARY KEY,          -- из gRPC metadata
    method       text        NOT NULL,             -- TopUp | ...
    request_hash bytea       NOT NULL,             -- повтор с другим телом -> AlreadyExists
    response     bytea       NOT NULL,             -- сохранённый protobuf-ответ
    created_at   timestamptz NOT NULL DEFAULT now()
);
```

Постоянная таблица `accounts` содержит `id`, `currency` и дату создания, без изменяемого balance. Account не удаляется при нулевом балансе: его строка служит границей блокировки. В 1.0 счёт ученика/преподавателя связан с identity, есть системный счёт; модель плательщика-представителя фиксируется до 2.0 (DEC-03).

Инварианты:

- **Леджер append-only.** Строки не обновляются и не удаляются — на уровне прав БД (роль сервиса без UPDATE/DELETE на `ledger_entries`). Корректировка — компенсирующая строка (`refund`), не правка истории.
- **Баланс воспроизводим на любой момент**: `SUM(amount) WHERE account_id = X AND created_at <= T`.
- **Доступный баланс** = `SUM(ledger)` − `SUM(holds WHERE status = 'ACTIVE')` — проверяется в транзакции `PlaceHold` после `SELECT accounts ... FOR UPDATE`. Все изменения доступных средств используют ту же блокировку; несколько счетов блокируются по возрастанию ID.
- **Захват атомарен**: `holds.status → CAPTURED` + строки леджера + outbox `payment.captured` — одна транзакция. Никакого состояния «холд захвачен, а денег нет».
- **Сумма захвата = сумме холда**: захватывается ровно то, что резервировалось; пересчёты цены после брони невозможны.
- **`holds.idempotency_key UNIQUE`** — гонка двух `PlaceHold` с одним ключом разрешается БД.

## Жизненный цикл холда

```
PlaceHold ──► ACTIVE ──┬──(lesson.completed)──► CAPTURED   + ledger: debit student / credit teacher + payment.captured
                       └──(booking.cancelled)─► RELEASED   + hold.released; частичный возврат: + строки штрафа
```

Терминальные статусы финальны: CAPTURED-холд нельзя release, RELEASED — нельзя capture (`FailedPrecondition` для RPC; consumer повторяет тот же outcome как no-op, а противоположный outcome сохраняет в reconciliation queue с алертом).

## Ключевые потоки

- **PlaceHold (синхронный, в booking-саге)** — [booking-saga.puml](../diagrams/sequence/booking-saga.puml). Транзакция: заблокировать account → проверить scoped key/request_hash → доступный баланс ≥ amount → `INSERT holds (ACTIVE, idempotency_key, request_hash)` и сохранённый исход. Потерянный ответ → Core-reconciler повторяет с тем же ключом, Billing возвращает сохранённый исход.
- **CaptureHold (по событию)** — [lesson-completed-billing.puml](../diagrams/sequence/lesson-completed-billing.puml). Одна транзакция: статус, пара строк ledger, outbox. Kafka offset фиксируется после commit.
- **ReleaseHold (по событию)** — своевременная отмена валидна сразу в Core; деньги вернутся при первой доступности Billing, Kafka гарантирует доставку ([04](../architecture/04-sagas-and-consistency.md)). После окна бесплатной отмены Core публикует денежный результат, который сохраняет полное или частичное списание.
- **Reconciliation** — инвариант для мониторинга: нет ACTIVE-холдов старше окна урока + порога; сумма захватов = сумме соответствующих строк леджера. Расхождение = алерт, не «авось».

## Безопасность

- Баланс и леджер видит только владелец счёта; admin — через отдельные admin-эндпоинты ([07](../architecture/07-security.md)).
- Учётка billing_db недоступна другим сервисам: компрометация Core не открывает леджер.
- Внутренние RPC несут идентичность исходного пользователя в metadata — Billing разрешает `GetBalance` владельцу либо сотруднику с правом просмотра. `TopUp` требует отдельного финансового разрешения STAFF и audit reason; принадлежность счёта вызывающему не разрешает пополнение. В 1.0 это проверка seed STAFF на сервере, в 2.0 — granular permission; PSP identity появится вне MVP.

## Чего здесь нет — и почему

- **Мутируемой колонки `balance`.** Кэш баланса — это оптимизация чтения (materialized view / Redis с инвалидацией), но источник истины всегда `SUM(ledger)`; на MVP индекса по `account_id` достаточно.
- **Знания о предметной области.** Никаких `lesson_id`, `teacher_id`, attendance или cancellation windows — только `account_id`, зафиксированный `amount` и опаковый `reference_id`. Это позволяет переиспользовать Billing для любых будущих платных сущностей.
- **Синхронных вызовов в Core.** Связь «урок завершён → захват» — только события: Billing может лежать час, деньги сойдутся после рестарта.
- **Внешнего PSP.** Эквайринг — отдельная интеграция в будущем; модель готова (TopUp идемпотентен по внешнему id платежа).

## Границы релизов и денежный контракт

В MVP 1.0 поставляется денежное ядро с тестовыми admin-пополнениями. Нулевой commission snapshot означает только отсутствие комиссии в этом срезе; teacher credit не обещает пользовательский payout до 2.1. Настраиваемые tariffs/commission, packages/subscriptions и корректировки — 2.0; payout workflow/выгрузки — 2.1. Точный расчёт DEC-02/DEC-03 закрывается до M20-19 и не считается готовым из наличия ledger.

`Idempotency-Key` scoped по caller/service, методу и владельцу операции; хранится hash канонического запроса и исход, включая бизнес-отказ. Повтор с другим телом — conflict. Для группы используется отдельный ключ `place-hold:<participant_id>`, стабильный при повторе, и уникальный hold reference на participant. Транспортный timeout не записывается как окончательный отказ. Истечение ключа не разрешает повтор денежной операции с тем же reference.

Все расходы и отрицательные корректировки блокируют account до проверки баланса. Capture/release блокируют account, затем hold; несколько счетов — в одинаковом порядке. Совместимые проводки, изменение hold, receipt события и outbox коммитятся атомарно. Частичный результат удовлетворяет `capture_amount + release_amount = hold.amount`; получатели и суммы берутся из неизменяемого settlement snapshot. У повторного capture отсутствует новый денежный эффект.

Распределённое событие с несколькими participant outcomes обрабатывается одной ограниченной транзакцией либо с durable receipt `(event_id, participant_id)`; общий event не отмечается обработанным до завершения всех частей. Offset продвигается по правилам [03](../architecture/03-communication.md).
