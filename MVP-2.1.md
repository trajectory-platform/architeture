# MVP 2.1 — диплом

> Формат: дипломный проект  
> Контрольная точка: июнь  
> Статус документа: предлагаемая декомпозиция
> Год и точная дата контрольной точки требуют подтверждения.
> Сверка базы: `origin/master@a054171`, 2026-09-05; критерии и открытые решения — в [релизной готовности](Системный%20анализ/architecture/09-release-readiness.md).
> Покрытие каждого требования — [матрица MVP](Матрица%20покрытия%20MVP.md); план не подтверждает реализацию.

## Назначение этапа

MVP 2.1 — кумулятивный релиз, включающий [MVP 1.0](MVP-1.0.md) и [MVP 2.0](MVP-2.0.md). На этой контрольной точке должны быть выполнены **все требования разделов 2–19** документа [«Требования к MVP-версии»](Требования%20к%20МВП%20версии.md).

Этап предназначен не для первой интеграции критичных сервисов, booking saga или Realtime, а для закрытия функционального остатка, hardening и получения проверяемых доказательств готовности.

## Итоговый демонстрационный сценарий

1. Пользователь входит по паролю либо OAuth, управляет ролями и активными сессиями.
2. Администратор назначает staff roles/permissions и управляет пользователем с audit trail.
3. Представитель связан с учеником и видит разрешённую сводку прогресса.
4. Администратор создаёт курс, программу, группу, enrollment, тариф и policy.
5. Ученик покупает пакет или абонемент из внутреннего баланса и бронирует занятия.
6. Группа проводит урок с видео, полной доской, чатом, attendance, записью и board snapshot.
7. Отмена и no-show применяют согласованную policy; деньги отражаются в immutable ledger.
8. Ученик выполняет homework, test и аттестацию; progress и gamification обновляются без дублей.
9. Преподаватель формирует отчёт, видит начисления и создаёт запрос выплаты.
10. Пользователи работают с материалами, уведомлениями и support; администратор обрабатывает операции в соответствующих разделах.
11. Все сценарии доступны из завершённых кабинетов и admin panel, наблюдаемы и защищены негативными тестами доступа.

## Задачи этапа

| ID | Задача | Результат | Требования |
|---|---|---|---|
| `M21-01` | OAuth и login methods | Яндекс ID, VK ID, Google, safe linking/unlinking без silent email match, запрет удаления последнего способа входа | 3.4–3.5 |
| `M21-02` | Auth completeness | Окончательная multi-role/RBAC модель, active sessions, forced revoke, rate/CAPTCHA policies, backward-compatible transport и полный Auth E2E | 3.1–3.16 |
| `M21-03` | Account lifecycle | Block/unblock, user-requested deletion, anonymization, отзыв credentials/sessions и безопасное повторное использование освобождённого email | 4.6, 4.9 |
| `M21-04` | Представитель | Связь representative–student, управление связью и минимально необходимая сводка progress | 3.11–3.12, 4.3–4.4, 12.10, 19.2.9 |
| `M21-05` | Полная комната и attendance | Закрытие всех room tools/states, precise reconnect, timer/auto-complete, notes, audio degradation, attendance и no-show edge cases | 7.1–7.26 |
| `M21-06` | Recording closure | Проверка consent/access/delete, board snapshot import и доказательство автоматического удаления записи через один месяц | 8.1–8.9, 11.4 |
| `M21-07` | Аттестации | Все типы вопросов, versioned attempts, limits, auto/manual grading и итоговая аттестация | 10.1–10.11 |
| `M21-08` | Progress completeness | Журнал контрольных точек, освоение тем, результаты tests, teacher report, representative summary и защита от duplicate events | 12.1–12.6, 12.9–12.10 |
| `M21-09` | Gamification | Streak, achievements и weekly goal только в кабинете ученика | 12.7–12.8 |
| `M21-10` | Support completeness | Templates, response time, окончательные статусы, assignee workflow, attachments и audit | 15.1–15.8 |
| `M21-11` | Billing и payouts | Реквизиты, payout balance, запрос выплаты, admin statuses, начисления/выплаты за период, табличный export и стабильная граница будущего PSP adapter | 16.18–16.24 |
| `M21-12` | Полная admin panel | Все одиннадцать разделов, ownership-aware aggregation, search, filters, sort, pagination и audit для пользовательских/денежных mutations | 2.2, 18.1–18.13 |
| `M21-13` | Полные кабинеты | Закрыты все страницы и состояния кабинетов преподавателя и ученика, включая role switch, reports, payments, settings, documents и support | 19.1.1–19.1.11, 19.2.1–19.2.9 |
| `M21-14` | UI foundation closure | Единая design system, высокая плотность teacher UI, упрощённый student UI, light/dark themes и адаптивные mobile layouts | 2.3–2.9 |
| `M21-15` | Audit и privacy | Журналы действий ключевых сущностей, consent на ПДн/cookie/recording, retention/delete policies, redaction sensitive data | 2.10–2.11, 2.15–2.17, 4.8, 18.13 |
| `M21-16` | Web analytics | Подключаемый adapter аналитики с consent-aware включением и без утечки secrets/ПДн | 2.12 |
| `M21-17` | Security hardening | SQL injection/XSS/CSRF, authz negative suite, membership checks, secret scanning, TLS, fail-closed limits для auth/money | 2.13–2.17, 3.9–3.10, 3.16 |
| `M21-18` | Reliability hardening | Kafka retry/DLQ/replay, outbox recovery, DB/broker/Redis/LiveKit fault injection, backup/restore drill, retention jobs | 2.6, 2.18–2.22 |
| `M21-19` | Performance validation | Load/soak для API, Kafka consumers и WebSocket; комната до восьми участников; resource limits и capacity notes | 7.12, сквозные NFR раздела 2 |
| `M21-20` | Traceability matrix | Каждое требование разделов 2–19 связано с задачей, реализацией, тестом и приёмочным evidence | Все требования MVP |
| `M21-21` | Дипломный system E2E | Clean install, upgrade с 1.0/2.0, полный demo, automated suites и manual acceptance для визуальных требований | Все требования MVP |

## Матрица покрытия по разделам

Таблица фиксирует основную точку поставки. Если функция появилась раньше, MVP 2.1 всё равно повторно подтверждает её regression/acceptance evidence.

| Раздел требований | MVP 1.0 | MVP 2.0 | MVP 2.1 — обязательное закрытие |
|---|---|---|---|
| 2. Общие требования | Инфраструктура, Kafka, Notification DB, security/observability baseline | Масштабирование учебного контура и operational dashboards | Design system, themes, adaptive UI, privacy, analytics и полный NFR hardening |
| 3. Аутентификация и роли | Email/password, confirmation, recovery, refresh/logout, базовые роли | Multi-role, staff permissions, invitations и session management | OAuth, lifecycle edge cases и полный security/E2E набор |
| 4. Пользователи и профили | Основные student/teacher profiles, verification/blocking | Полные поля, representative link и агрегированная admin card | Audit, deletion/anonymization и полная приёмка |
| 5. Справочники и учебная структура | Базовые справочники | Courses, programs, groups и enrollments | Переводы/expiry edge cases и полная приёмка |
| 6. Расписание и бронирование | Разовые slots/bookings и holds | Recurring/trial/group, reschedule, cancel/no-show policy | Admin/audit и concurrency/failure closure |
| 7. Урок и комната | Индивидуальный урок, video/audio, базовая CRDT-доска и chat | Группа до восьми, полная доска и room UX | Все edge cases, нагрузка и итоговая приёмка |
| 8. Запись занятий | — | Consent, recording, snapshot, S3 и retention | Delete/access/retention evidence |
| 9. Домашние задания | — | Полный workflow | Regression и связь с progress |
| 10. Тесты и аттестации | — | Tests foundation | Все типы вопросов и аттестация |
| 11. Материалы | S3 foundation | Library, access, search, lesson imports | Полная приёмка access/retention |
| 12. Прогресс и геймификация | Attendance foundation | Progress и teacher report | Gamification и representative summary |
| 13. Чаты | Room chat | Direct/group/lesson chats полностью | Search/access/reconnect closure |
| 14. Уведомления | Notification Service, Auth email, core in-app events | Все in-app mappings, preferences/templates | Полнота событий и delivery reliability |
| 15. Поддержка | — | Ticket и admin queue foundation | Templates, SLA и полный workflow |
| 16. Биллинг | Ledger, balance, hold/capture/release, admin top-up | Tariffs, commissions, packages/subscriptions и accruals | Payout workflow, reports и export |
| 17. Файлы | Presigned access, membership, базовые ограничения | Namespaces, previews и policies | Retention/security evidence |
| 18. Административная панель | Минимум для первого среза | Основные operational sections | Все разделы, list controls, settings, analytics и audit |
| 19. Кабинеты | Экраны первого сценария | Основные teacher/student workflows | Все страницы, состояния, UX и adaptive acceptance |

## Definition of Done

- Все требования разделов 2–19 имеют статус `DONE` в traceability matrix.
- Для каждого требования указаны implementation reference и test/acceptance evidence в [матрице MVP](Матрица%20покрытия%20MVP.md). Значение PLANNED и ссылка только на задачу не являются evidence.
- DEC-01–DEC-08 из релизного реестра закрыты либо явно неприменимы с основанием; согласованы год, владельцы и профиль итоговой приёмки.
- Итоговые численные NFR проверены в согласованной среде; успешный demo двух участников не считается проверкой 100 комнат/800 участников.
- Пройдены unit, integration, contract, migration, E2E и Compose/stage smoke suites.
- Пройдены authorization/security negative tests, Kafka replay/DLQ tests и основные fault-injection проверки.
- Пройдены reconnect и load/soak проверки Realtime; подтверждена комната до восьми участников.
- Проверены clean install и upgrade с данных MVP 1.0 и MVP 2.0.
- Настроены dashboards и alerts для сервисов, БД, Kafka, outbox, jobs и WebSocket.
- Нет открытых P0/P1; P2 не блокируют заявленные пользовательские сценарии и имеют согласованный план.
- Дипломный сценарий воспроизводится на stage без ручной правки данных.

## За пределами MVP 2.1

Раздел 20 исходного документа не включается в MVP 2.1:

- внешний платёжный шлюз, фискализация и чеки;
- автоматические массовые выплаты преподавателям;
- продуктовые email/push/messenger notifications, кроме обязательных transactional Auth emails;
- мобильные приложения;
- промокоды, акции и referrals;
- публичные отзывы и рейтинги;
- мультиязычность и мультивалютность;
- predictive analytics;
- полноценный branching LMS-конструктор;
- открытый API внешним системам.

Внешний PSP и автоматические выплаты должны оставаться сменными adapters: их отсутствие не должно требовать переделки ledger, holds, packages, subscriptions или manual payout workflow.
