# MVP 2.0 — апрельская сессия

> Формат: командный проект  
> Контрольная точка: апрель  
> Статус документа: предлагаемая декомпозиция

## Назначение этапа

MVP 2.0 — кумулятивное расширение [MVP 1.0](MVP-1.0.md) до рабочего учебного контура. Платформа должна поддерживать курсы и группы, регулярное обучение, домашние задания, тесты, материалы, записи уроков, базовый прогресс и операционную работу администратора.

MVP 2.0 ещё может иметь явно перечисленный остаток. Полное выполнение [требований к MVP-версии](Требования%20к%20МВП%20версии.md) принимается только в [MVP 2.1](MVP-2.1.md).

## Демонстрационный сценарий

1. Администратор создаёт предмет, курс, программу и группу до семи учеников.
2. Ученики получают доступ к курсу на период и уведомление о сроке доступа.
3. Преподаватель задаёт recurring availability; система материализует слоты и уроки.
4. Для группового урока создаётся отдельный hold каждого ученика по фиксированной цене; стоимость не делится между участниками.
5. Группа проводит урок до восьми участников, использует полную доску, чат и запись с consent.
6. При завершении фиксируются attendance, no-show и participant-level monetary outcomes.
7. Запись и board snapshot становятся материалами урока; запись автоматически удаляется через месяц.
8. Преподаватель выдаёт homework и test, ученик отправляет работу и получает результат.
9. Learning обновляет progress идемпотентно по событиям Core.
10. Пользователь видит in-app уведомления по booking, lesson, homework, chat, enrollment и money events.

## Задачи этапа

| ID | Задача | Результат | Требования |
|---|---|---|---|
| `M20-01` | Расширение контрактов | `learning.v1`, расширенные Core/Billing/Auth/Notification contracts, новые Kafka events и OpenAPI кабинетов/admin; additive evolution либо новая версия при breaking change | 2.1, 2.18, 2.21 |
| `M20-02` | Auth access model | Учебные и staff identities, несколько ролей, active cabinet, access context, granular permissions, приглашения сотрудников | 3.11–3.14 |
| `M20-03` | Управление сессиями и security flows | Список активных сессий, завершение одной/всех сессий, session limits, password change/recovery и подтверждённая смена email | 3.6, 3.8–3.9, 3.15 |
| `M20-04` | Полные профили и связи | Поля ученика/преподавателя, representative link, teacher verification, блокировка, агрегированная карточка пользователя | 4.1–4.7 |
| `M20-05` | Курсы и программы | Курсы, versioned program, темы и порядок, tariff reference | 5.5–5.6 |
| `M20-06` | Группы и enrollments | Группы до семи учеников, создание teacher/admin, доступ на период, статусы, продление, прекращение и перевод | 5.7–5.14 |
| `M20-07` | Полное расписание | Recurring availability, исключения, sliding horizon, recurring booking/cancel, trial lesson | 6.1, 6.3–6.4, 6.7–6.9 |
| `M20-08` | Перенос и cancellation policy | Перенос с подтверждением, отмена, настраиваемые пороги/dоля возврата, admin override с audit | 6.14–6.17 |
| `M20-09` | Групповая booking saga | Participant rows и отдельный `PlaceHold` на ученика, фиксированная цена с каждого, независимый отказ места, no-show capture после бесплатного окна | 6.10–6.13, 16.25–16.26 |
| `M20-10` | Полная комната урока | До восьми участников, полный набор инструментов доски, formulas/images/files, participant state, active speaker/quality, audio fallback, notes и timer | 7.6–7.8, 7.12–7.20 |
| `M20-11` | Записи и board snapshot | Consent, LiveKit Egress, recording indicator, S3 metadata, membership access, удаление по запросу, retention один месяц, снимок доски | 8.1–8.9 |
| `M20-12` | Homework | Создание/назначение, submissions, повторная отправка, review, revision, статусы, очередь, templates и события | 9.1–9.13 |
| `M20-13` | Tests foundation | Question bank, основные типы вопросов, назначения, limits, attempts, automatic/manual grading и результаты | 10.1–10.9, 10.11 |
| `M20-14` | Materials | Upload, связи course/topic/lesson, типы, access control, library, recording/board imports и поиск | 11.1–11.8 |
| `M20-15` | Progress foundation | Attendance, completed lessons, освоение тем, контрольные точки, защита от дублей, общий процент и teacher report | 12.1–12.6, 12.9 |
| `M20-16` | Полные чаты | Direct, group и lesson chats, attachments, read state, unread counters, history search и message deduplication | 13.1–13.9 |
| `M20-17` | Полные in-app уведомления | Все события Core/Learning/Billing/Auth, reminders, preferences, templates, popups, replay-safe inbox, retry и DLQ | 6.20, 14.1–14.9 |
| `M20-18` | Support foundation | Создание ticket, category/priority/status, переписка с вложениями, admin queue и assignee | 15.1–15.6 |
| `M20-19` | Пакеты, абонементы и комиссии | Tariffs, глобальная/индивидуальная комиссия, teacher accrual, packages, subscriptions, остаток и срок доступа, manual correction | 16.6, 16.9–16.16, 16.18–16.19 |
| `M20-20` | File pipeline | Все bucket namespaces, type/size policies, image previews, retention metadata и безопасный presigned access | 17.1–17.7 |
| `M20-21` | Расширение кабинетов | Teacher: ученики, homework, tests, materials, messages, базовые отчёты/начисления. Student: teacher/course catalog, homework, materials, progress, chat и payment products | 19.1.1–19.1.11 и 19.2.1–19.2.9, кроме явно оставленного для 2.1 |
| `M20-22` | Операционная admin panel | Пользователи, преподаватели, ученики, обучение, расписание, базовые финансы, коммуникации, материалы, справочники; filters/sort/pagination для реализованных списков | 18.1–18.9, 18.12–18.13 частично |
| `M20-23` | Надёжность второго среза | Consumer duplicate/out-of-order tests, recurring job idempotency, group partial failure, cancel/no-show, recording access, notification replay, migration upgrade с 1.0 | Сквозные NFR раздела 2 |
| `M20-24` | Stage E2E | Course → group → enrollment → recurring lesson → homework/test/material → progress → notifications | Сквозная приёмка функциональности этапа |

## Архитектурные ограничения

- Core Education остаётся владельцем scheduling, booking, lesson, attendance, recording и chat domains.
- Learning владеет courses, groups, enrollments, homework, tests, materials и progress.
- Learning не вызывает Core синхронно для обработки завершённого урока: projections строятся из Kafka events.
- Для группового занятия цена фиксируется с каждого ученика и не делится между участниками.
- No-show приводит к списанию, только если бесплатное cancellation window уже истекло.
- Каждый урок получает новую доску; snapshot прошлого урока доступен только как материал.
- Запись хранится один месяц с момента готовности, затем удаляется retention job.
- Transactional Auth email допустим и обязателен; продуктовые email/push channels остаются вне MVP.

## Последовательность реализации

```text
Contracts + Auth access model
  → Courses
  → Groups
  → Enrollments
  → Recurring/group booking
  → Lesson completion events
  → Homework + Tests + Materials
  → Progress + Notifications
  → Cabinets + Admin
  → Stage E2E
```

## Definition of Done

- Функции MVP 1.0 продолжают работать после миграции stage с данными предыдущей версии.
- Групповой сценарий проходит end-to-end без ручной правки БД или Kafka.
- Комната выдерживает восемь участников в согласованном нагрузочном профиле.
- Duplicate/out-of-order events не задваивают progress, ledger, inbox или materials.
- Recurring jobs и booking reconciler безопасно повторяются после сбоя.
- Kafka lag, outbox backlog, PENDING bookings, retention jobs и WebSocket connections наблюдаемы.
- Нет открытых дефектов P0/P1 по функциональности этапов 1.0–2.0.

## Остаток, обязательный для MVP 2.1

- OAuth Яндекс ID, VK ID и Google с безопасным linking/unlinking.
- Account deletion/anonymization и полный audit trail.
- Аттестации, окончательная модель progress, gamification и representative summary.
- Support templates, SLA/response time и полный административный workflow.
- Payout details, запросы/обработка выплат и финансовые выгрузки.
- Все разделы admin panel, analytics dashboard, статические страницы, banners и platform settings.
- Полная design system, light/dark themes, mobile adaptation, consent/cookie и web analytics.
- Итоговое security, authorization, fault-injection, load/soak и acceptance evidence.
