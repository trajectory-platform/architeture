# 04 — Саги и консистентность

Деньги и расписание живут в разных сервисах (Billing и Core), а должны меняться согласованно. Распределённых транзакций нет — есть саги: последовательность локальных транзакций + компенсации + идемпотентность.

## Booking-сага

Самая критичная цепочка системы. Диаграмма: [booking-saga.puml](../diagrams/sequence/booking-saga.puml).

### Инварианты

1. Слот преподавателя не может быть забронирован дважды (даже при гонке двух студентов).
2. Деньги не списываются и не резервируются без подтверждённого бронирования; бронирование не подтверждается без резерва денег.
3. Повтор запроса (retry клиента, гейтвея, сети) не создаёт второй брони и второго холда.

### Шаги

```
Клиент ── POST /bookings (Idempotency-Key) ──► Gateway ──► Core.CreateBooking
```

1. **Core: локальная транзакция №1.**
   - Advisory lock по `teacher_id` (сериализует конкурентные брони одного преподавателя).
   - `INSERT INTO bookings (..., status = 'PENDING', time_range = tstzrange(start, end))`.
   - Двойную бронь убивает сама БД — exclusion constraint:
     ```sql
     ALTER TABLE bookings ADD CONSTRAINT no_double_booking
       EXCLUDE USING gist (teacher_id WITH =, time_range WITH &&)
       WHERE (status IN ('PENDING', 'CONFIRMED'));
     ```
     Пересечение интервалов одного преподавателя → ошибка constraint → `FailedPrecondition` клиенту. Это последняя линия обороны, работающая даже если логика выше дала сбой.
2. **Core → Billing: `PlaceHold(student_id, amount, reference_id = booking_id)`** по gRPC с тем же `Idempotency-Key` и deadline.
   - Billing в своей транзакции: проверка баланса (`SUM(ledger_entries)` ≥ amount + активные холды), `INSERT INTO holds`, запись идемпотентного ключа.
   - Недостаточно средств → `ResourceExhausted`.
3. **Исход.**
   - Успех → Core: транзакция №2 — `status = 'CONFIRMED'` + строка в `outbox` (`booking.confirmed`).
   - Ошибка → Core: `status = 'FAILED'`. Деньги не тронуты, компенсация не нужна.
   - **Таймаут/неизвестный исход** (ответ Billing потерян): ничего не предполагаем. Booking остаётся PENDING; фоновый reconciler повторяет `PlaceHold` с тем же ключом — Billing вернёт сохранённый результат (холд есть/нет), сага доводится до CONFIRMED или FAILED. PENDING старше порога (например, 1 мин) виден в метриках.

### Идемпотентность по слоям

| Слой | Механизм |
|---|---|
| Gateway | `Idempotency-Key` из заголовка → gRPC metadata |
| Core | Уникальный индекс по ключу на `bookings`: повтор возвращает существующую бронь |
| Billing | Уникальный индекс по ключу на `holds`: повтор возвращает существующий холд |
| БД | Exclusion constraint — гонки невозможны независимо от всего выше |

## Отмена бронирования

1. Core: транзакция — `status = 'CANCELLED'` + outbox `booking.cancelled` (payload содержит `booking_id`, политику возврата).
2. Billing (durable consumer): `ReleaseHold(reference_id)` — холд закрывается, средства снова доступны. Идемпотентно: повторная доставка события на уже закрытый холд — no-op.
3. Никакого синхронного вызова: отмена валидна сразу, деньги вернутся при первой доступности Billing — JetStream гарантирует доставку.

Правила отмены (за сколько часов бесплатно, штраф) — политика в Core; Billing исполняет, что сказано в payload (`release_full` / `release_partial(amount)`).

## Завершение урока

Диаграмма: [lesson-completed-billing.puml](../diagrams/sequence/lesson-completed-billing.puml).

1. LiveKit-вебхук `room_finished` → Core; Core (с учётом фактической посещаемости) помечает `lessons.status = 'COMPLETED'` + outbox `lesson.completed`.
2. **Billing** потребляет: `CaptureHold` — холд превращается в пару строк леджера (debit студента, credit преподавателя/платформы), публикуется `payment.captured`.
3. **Reports (пакет Core)** потребляет то же событие: инкремент счётчика завершённых уроков пары «студент × преподаватель». Каждый 5-й урок → запись журнала прогресса + outbox `progress.milestone_reached`.

Оба потребителя независимы: отставание биллинга не блокирует журнал прогресса, и наоборот.

## Recurring-бронирования

Регулярное расписание — не отдельный механизм, а **генератор + та же сага на каждое вхождение**:

- `recurring_rules` хранит правило (день недели, время, период действия).
- Генератор (периодическая джоба Core) материализует вхождения на rolling-горизонт (например, 4 недели вперёд), прогоняя каждое через booking-сагу с детерминированным идемпотентным ключом `rule_id + occurrence_date` — повторный запуск джобы ничего не дублирует.
- Не хватило денег на N-е вхождение → это вхождение FAILED, уведомление студенту; остальные не страдают.
- Отмена правила → отмена будущих CONFIRMED-вхождений штатным путём отмены.

## Журнал прогресса (каждые 5 уроков)

Надёжность «автоматики» обеспечена цепочкой: `lesson.completed` уходит через outbox (не теряется) → JetStream redelivery (доставится) → консьюмер идемпотентен (счётчик привязан к `lesson_id`, повторная обработка того же урока не инкрементит дважды — `INSERT ... ON CONFLICT DO NOTHING` в таблицу обработанных уроков). Milestone вычисляется из счётчика в той же транзакции, что и отметка обработки.

## Сводка гарантий

| Свойство | Механизм |
|---|---|
| Нет двойных броней | Exclusion constraint (`tstzrange` + gist) + advisory lock |
| Нет потерянных денег | Append-only ledger, холды, capture/release только по событиям с at-least-once |
| Нет дублей при retry | Idempotency-Key на каждом слое |
| Нет потерянных событий | Transactional outbox + JetStream durable consumers |
| Зависшие саги видимы | Reconciler + метрика возраста PENDING-броней |
