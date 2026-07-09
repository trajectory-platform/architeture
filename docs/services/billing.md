# Billing Service

Billing владеет деньгами — и только деньгами. Модель — **append-only ledger**: баланс пользователя — это `SUM()` строк леджера, никогда не мутируемая колонка. Billing ничего не знает про уроки, слоты и расписание: он оперирует холдами с внешним `reference_id` (uuid брони) и реагирует на события. Любое движение денег идемпотентно по `Idempotency-Key`.

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
| `CaptureHold` | консьюмер `lesson.completed` (внутренний) | Холд → пара строк леджера + `payment.captured` | `FailedPrecondition` (холд не ACTIVE — но для consumer-пути это no-op, не ошибка) |
| `ReleaseHold` | консьюмер `booking.cancelled` (внутренний) | Холд закрыт, средства снова доступны | идемпотентен: повтор на закрытый холд — no-op |
| `GetBalance` | Gateway | Доступный баланс = `SUM(ledger)` − активные холды | — |
| `TopUp` | Gateway | Пополнение: строка `topup` в леджер | `AlreadyExists` (повтор ключа с другим телом) |

`CaptureHold`/`ReleaseHold` существуют и как RPC (для admin-операций и reconciler'а), но штатный путь — события JetStream, не синхронные вызовы: отставание Billing не должно блокировать Core ([03](../architecture/03-communication.md)).

`TopUp` на MVP — внутренняя операция (тестовые начисления, admin). Интеграция с внешним PSP — будущее отдельное решение: вебхук провайдера → `TopUp` с идемпотентным ключом из id платежа провайдера.

## События

| Событие | Направление | Реакция |
|---|---|---|
| `booking.cancelled` | потребляет | `ReleaseHold(reference_id)`; payload несёт политику: `release_full` / `release_partial(amount)` — частичный возврат = release + строки леджера на штраф |
| `lesson.completed` | потребляет | `CaptureHold(reference_id)`: debit студента / credit преподавателя |
| `payment.captured` | публикует | Через outbox в транзакции захвата; Core помечает урок оплаченным |
| `hold.released` | публикует | Через outbox в транзакции release; для нотификаций/аудита |

Консьюмеры durable, ack после успешной транзакции, обработка идемпотентна: повторная доставка на уже CAPTURED/RELEASED холд — no-op.

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

Счета (`account_id`) — без отдельной таблицы: это uuid identity из Auth (студенты, преподаватели) плюс фиксированный системный uuid платформы (комиссии, штрафы). Счёт «существует», как только у него появилась первая строка леджера.

Инварианты:

- **Леджер append-only.** Строки не обновляются и не удаляются — на уровне прав БД (роль сервиса без UPDATE/DELETE на `ledger_entries`). Корректировка — компенсирующая строка (`refund`), не правка истории.
- **Баланс воспроизводим на любой момент**: `SUM(amount) WHERE account_id = X AND created_at <= T`.
- **Доступный баланс** = `SUM(ledger)` − `SUM(holds WHERE status = 'ACTIVE')` — проверяется в транзакции `PlaceHold`.
- **Захват атомарен**: `holds.status → CAPTURED` + строки леджера + outbox `payment.captured` — одна транзакция. Никакого состояния «холд захвачен, а денег нет».
- **Сумма захвата = сумме холда**: захватывается ровно то, что резервировалось; пересчёты цены после брони невозможны.
- **`holds.idempotency_key UNIQUE`** — гонка двух `PlaceHold` с одним ключом разрешается БД.

## Жизненный цикл холда

```
PlaceHold ──► ACTIVE ──┬──(lesson.completed)──► CAPTURED   + ledger: debit student / credit teacher + payment.captured
                       └──(booking.cancelled)─► RELEASED   + hold.released; частичный возврат: + строки штрафа
```

Терминальные статусы финальны: CAPTURED-холд нельзя release, RELEASED — нельзя capture (`FailedPrecondition` для RPC-пути, no-op для consumer-пути — событие могло приехать повторно).

## Ключевые потоки

- **PlaceHold (синхронный, в booking-саге)** — [booking-saga.puml](../diagrams/sequence/booking-saga.puml). Транзакция: доступный баланс ≥ amount → `INSERT holds (ACTIVE, idempotency_key)`. Потерянный ответ → Core-reconciler повторяет с тем же ключом, Billing возвращает сохранённый исход.
- **CaptureHold (по событию)** — [lesson-completed-billing.puml](../diagrams/sequence/lesson-completed-billing.puml). Одна транзакция: статус, пара строк леджера, outbox. Ack — после коммита.
- **ReleaseHold (по событию)** — отмена брони валидна сразу в Core; деньги вернутся при первой доступности Billing, JetStream гарантирует доставку ([04](../architecture/04-sagas-and-consistency.md)).
- **Reconciliation** — инвариант для мониторинга: нет ACTIVE-холдов старше окна урока + порога; сумма захватов = сумме соответствующих строк леджера. Расхождение = алерт, не «авось».

## Безопасность

- Баланс и леджер видит только владелец счёта; admin — через отдельные admin-эндпоинты ([07](../architecture/07-security.md)).
- Учётка billing_db недоступна другим сервисам: компрометация Core не открывает леджер.
- Внутренние RPC несут идентичность исходного пользователя в metadata — Billing проверяет, что `TopUp`/`GetBalance` запрошен владельцем счёта.

## Чего здесь нет — и почему

- **Мутируемой колонки `balance`.** Кэш баланса — это оптимизация чтения (materialized view / Redis с инвалидацией), но источник истины всегда `SUM(ledger)`; на MVP индекса по `account_id` достаточно.
- **Знания о предметной области.** Никаких `lesson_id`, `teacher_id`, цен — только `account_id`, `amount`, опаковый `reference_id`. Это позволяет переиспользовать Billing для любых будущих платных сущностей.
- **Синхронных вызовов в Core.** Связь «урок завершён → захват» — только события: Billing может лежать час, деньги сойдутся после рестарта.
- **Внешнего PSP.** Эквайринг — отдельная интеграция в будущем; модель готова (TopUp идемпотентен по внешнему id платежа).
