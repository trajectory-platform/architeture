# MVP 1.0 — январская сессия

> Формат: командный проект  
> Контрольная точка: январь  
> Статус документа: предлагаемая декомпозиция

## Назначение этапа

MVP 1.0 — первый работающий вертикальный срез Trajectory. Он должен доказать основной продуктовый и технический сценарий: ученик регистрируется, находит свободное время преподавателя, пополняет внутренний баланс через администратора, бронирует индивидуальный урок, проводит его в комнате и оплачивает из зарезервированных средств.

Этап не является полным выполнением [требований к MVP-версии](Требования%20к%20МВП%20версии.md). Полное соответствие обязательно для [MVP 2.1](MVP-2.1.md).

## Демонстрационный сценарий

1. Ученик и преподаватель регистрируются по email, подтверждают адрес и входят в систему.
2. Преподаватель заполняет профиль, задаёт цену и открывает разовое окно доступности.
3. Администратор вносит тестовое пополнение внутреннего баланса ученика.
4. Ученик видит слот и создаёт разовую индивидуальную бронь.
5. Core Education резервирует стоимость урока в Billing и подтверждает бронь.
6. Участники входят в комнату, используют видео, аудио, базовую доску и чат.
7. Состояние доски переживает reconnect; для урока используется отдельная новая доска.
8. Преподаватель завершает урок, Core фиксирует посещаемость, Billing списывает hold.
9. Ученик видит операцию в истории, оба участника получают in-app уведомление.
10. Повтор запроса или Kafka event не создаёт вторую бронь, проводку или нотификацию.

## Задачи этапа

| ID | Задача | Результат | Требования |
|---|---|---|---|
| `M10-01` | Базовая инфраструктура | Репозитории и service skeletons, PostgreSQL per service, Kafka KRaft, Redis, MinIO, LiveKit, Nginx, dev/stage Compose, миграции и CI/CD. Learning на этом этапе допустим как health-only skeleton | 2.1, 2.6, 2.14, 2.19, 2.21–2.22 |
| `M10-02` | Эксплуатационный baseline | `/healthz`, `/readyz`, graceful shutdown, HTTPS на stage, secrets вне кода, структурированные логи, metrics, traces и минимальные alerts | 2.13–2.19 |
| `M10-03` | Contracts-first foundation | Event envelope, базовые `auth.v1`, `education.v1`, `billing.v1`, `notification.v1`, OpenAPI первого среза; генерация и compatibility checks | 2.1, 2.18, 2.21 |
| `M10-04` | API Gateway | REST edge, JWT/JWKS validation, REST → gRPC, единый формат ошибок, CORS и базовый rate limiting | 2.13–2.18, 3.9–3.10 |
| `M10-05` | Auth baseline | Регистрация и подтверждение email, login, refresh rotation/reuse detection, logout, восстановление пароля, роли `student`, `teacher`, seed-роли администратора | 3.1–3.3, 3.6–3.11 |
| `M10-06` | Notification minimum | Отдельные `notification_db` и Kafka consumer group, transactional email для подтверждения/восстановления, in-app inbox, unread counter, `MarkRead`, WebSocket delivery | 3.2, 3.6, 14.1–14.3, 14.7–14.9 |
| `M10-07` | Базовые профили | Профиль пользователя, timezone, основные поля ученика и преподавателя, цена и длительность индивидуального занятия, верификация/блокировка администратором | 2.20, 4.1–4.3, 4.5–4.6, 16.8 |
| `M10-08` | Минимальные справочники | Предметы, направления, уровни подготовки, типы и длительности занятий | 5.1–5.4 |
| `M10-09` | Billing core | Immutable ledger, вычисляемый balance, admin TopUp, история ученика, `PlaceHold`, `CaptureHold`, `ReleaseHold`, идемпотентность | 16.1–16.5, 16.7–16.8, 16.15, 16.17 |
| `M10-10` | Разовое расписание | Разовые окна, materialized slots, просмотр слотов, day/week/month calendar преподавателя и упрощённый календарь ученика | 6.2, 6.4–6.5, 6.18–6.19 |
| `M10-11` | Индивидуальная booking saga | Разовая бронь, DB-защита от double booking, резерв средств, освобождение слота при отказе, идемпотентный reconciler | 6.6, 6.10–6.13 |
| `M10-12` | Жизненный цикл урока | Создание после подтверждения брони, статусы, membership, временное окно входа, завершение, auto-complete, attendance и no-show | 7.1–7.5, 7.21–7.25 |
| `M10-13` | Комната и базовая доска | Equipment check, LiveKit audio/video, microphone/camera, screen sharing, drawing/text, CRDT sync, persist-before-fanout, reconnect, undo/redo и отдельная доска урока | 7.5–7.11, 7.13–7.14, 7.26 |
| `M10-14` | Чат комнаты | Отправка и сохранение сообщений, история, read state, unread counter, идемпотентный client message ID | 7.17, 13.3, 13.5–13.6, 13.8–13.9 |
| `M10-15` | S3 foundation | Scoped presigned upload/download, membership check, ограничения типа/размера, пространства для avatars и chat attachments | 17.1–17.4, 17.6 |
| `M10-16` | Минимальные кабинеты | Login/registration, профиль, расписание, бронирование, ближайший урок, вход в комнату, чат, баланс и история операций | 19.1.1–19.1.2, 19.1.8, 19.1.11, 19.2.1–19.2.2, 19.2.6, 19.2.8–19.2.9 частично |
| `M10-17` | Минимальная admin panel | Пользователи, преподаватели, справочники, брони/уроки, журнал операций и ручное пополнение | 18.1–18.3, 18.5–18.6, 18.9, 18.13 частично |
| `M10-18` | Вертикальные тесты | Unit и integration tests, гонки refresh/booking/hold, Kafka duplicate delivery, board reconnect, Compose smoke и E2E основного сценария | 2.6, 2.13, 2.18–2.19 |

## Архитектурные ограничения

- Сервисные границы соответствуют принятой архитектуре: API Gateway, Auth, Core Education, Learning, Billing, Realtime и Notification.
- Публичные и межсервисные контракты изменяются сначала в `contracts`.
- Доменное изменение и outbox event фиксируются в одной PostgreSQL transaction.
- Kafka consumers идемпотентны и фиксируют offset только после commit.
- Цена индивидуального урока фиксируется в booking snapshot.
- Пароли и raw tokens не попадают в логи, Kafka или DLQ.
- Проверка доступа к комнате, чату и файлам выполняется по membership, а не только по роли.

## Последовательность реализации

```text
Infrastructure + contracts
  → Auth + API Gateway + Notification delivery
  → Profiles + Billing
  → Scheduling + Booking
  → Lesson lifecycle
  → LiveKit + Realtime + room chat
  → Client/Admin UI
  → Compose E2E
```

## Definition of Done

- Stage разворачивается с нуля одной документированной процедурой; миграции применяются автоматически.
- Демонстрационный сценарий проходит end-to-end через реальные PostgreSQL, Kafka, Redis, MinIO и LiveKit.
- Повторные REST-запросы и Kafka-события не создают дублей доменного или денежного состояния.
- Double booking исключён constraint на уровне БД.
- После reconnect восстанавливаются доска и чат урока.
- Один trace связывает REST, gRPC, outbox и Kafka consumer.
- Нет открытых дефектов P0/P1 по сценарию этапа.

## Сознательно отложено до следующих этапов

- Курсы, группы, enrollments и Learning business modules.
- Recurring availability/bookings, переносы и настраиваемая cancellation policy.
- Групповые уроки до восьми участников и полный набор инструментов доски.
- Записи уроков, homework, tests, materials и progress.
- Личные и групповые чаты вне комнаты урока.
- OAuth, несколько ролей аккаунта, granular staff permissions и управление сессиями.
- Пакеты, абонементы, комиссии и выплаты преподавателям.
- Полная admin panel, support, analytics, design-system hardening и адаптивность.
