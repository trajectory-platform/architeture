# ADR-007 — Эволюция identity, ролей и доступа в Auth Service

**Статус:** принято  
**Дата:** 2026-09-04

## Контекст

Ранний срез Auth Service моделировал аккаунт одной строкой `identities`, одной ролью `student`, `teacher` или `admin` и статусом `ACTIVE` или `BLOCKED`. Access JWT содержал одиночный claim `role`, а permissions в токене были явно исключены. ADR-006 закрепил `ROLE_ADMIN` как промежуточную эволюцию минимального контракта.

Новый бизнес-анализ требует другую модель:

- один пользователь совмещает учебные роли `student`, `teacher`, `representative`;
- staff identities отделены от пользовательских identities и не совмещают административные и учебные роли;
- административные роли хранятся в БД как конфигурируемые наборы permissions;
- `superadmin` и `manager` поставляются как предустановленные роли;
- access token и клиентский `init` содержат массивы roles и permissions;
- аккаунт проходит состояния `pending_email`, `active`, `blocked`, `deletion_scheduled`, `deleted`;
- Auth владеет email confirmation, OAuth connections, invitations, sessions, password recovery и auth audit;
- изменения должны внедряться поверх уже созданной схемы T01–T11 без переписывания истории migrations.

Эти требования несовместимы с одиночной колонкой `identities.role` и частью решения ADR-006. Нужна явная целевая модель и безопасный переход.

## Решение

1. **Разделить типы identity.** Общая `identities` сохраняет глобальную уникальность email и жизненный цикл. Взаимоисключающие `user_identities` и `staff_identities` хранят типовые данные. Staff identity создаётся только приглашением, входит через отдельный password route и не использует OAuth.
2. **Хранить учебные роли связью many-to-many.** `identity_learning_roles` поддерживает `student`, `teacher`, `representative` и их совмещение. Auth владеет назначением роли; Core Education владеет заявкой преподавателя и связью представитель–ученик.
3. **Хранить административные роли и permissions в БД.** `admin_roles`, `permissions`, `admin_role_permissions` и `staff_role_assignments` заменяют общий `admin`. Permissions нескольких staff roles объединяются на сервере. Прямые назначения permission сотруднику отсутствуют.
4. **Сохранить системные инварианты административного доступа.** `superadmin` неизменяем и включает весь каталог permissions. Нельзя лишить последнего активного суперадминистратора роли, заблокировать или удалить его. Staff identity всегда имеет минимум одну административную роль. Административные и учебные роли не совмещаются.
5. **Передавать эффективный доступ массивами.** Access JWT содержит `identity_type`, `roles`, `permissions` и `session_id`. Auth возвращает те же данные для Gateway `init`. Клиент использует их только для отображения; Gateway и сервис-владелец данных повторно проверяют permission и membership.
6. **Сделать session отдельной сущностью.** Session соответствует refresh family и хранит устройство, user-agent, последний IP, login method, active cabinet и last activity. На identity разрешено не более 10 активных sessions; новая session атомарно отзывает самую старую. Inactivity TTL — 30 дней.
7. **Расширить lifecycle identity.** Целевые статусы: `PENDING_EMAIL`, `ACTIVE`, `BLOCKED`, `DELETION_SCHEDULED`, `DELETED`. Email confirmation, invitation, password reset и email change используют одноразовые hashed action tokens. Анонимизация освобождает email и удаляет credentials, OAuth connections и sessions.
8. **Не связывать OAuth по совпавшему email.** Внешняя identity определяется `(provider, subject)`. Совпадение email требует подтверждения владения platform account паролем или одноразовой email-ссылкой. Поддерживаются Яндекс ID, VK ID и Google. У аккаунта всегда остаётся минимум один способ входа.
9. **Вести audit у владельца данных.** Auth пишет неизменяемые audit rows для действий над identities, sessions, credentials, OAuth connections, roles и permissions. Успешное изменение и audit row фиксируются одной транзакцией. Другие сервисы журналируют свои действия в своих БД; общий экран использует read model, а не распределённую ACID-транзакцию.
10. **Выполнить forward-only migration.** Существующие migrations и legacy columns не переписываются. Новые таблицы добавляются отдельно, данные backfill-ятся, чтение переключается после проверки эквивалентности, legacy read path удаляется позднее отдельной migration.
11. **Эволюционировать contracts contract-first.** Additive изменения остаются в `auth.v1`. Любая несовместимая wire-форма получает `auth.v2`. Опубликованный `v1` не меняется на месте.

Пункты ADR-006 о password reset ownership, response wrappers, безопасной передаче delivery данных и стабильном `event_id` сохраняются. Решение ADR-006 об одиночном `ROLE_ADMIN` заменяется этим ADR.

## Альтернативы

- **Добавить `representative`, `superadmin` и `manager` в одиночную колонку `role`.** Отклонено: пользователь не сможет совмещать учебные роли, а конфигурируемые staff roles потребуют кодировать динамическую модель статическим enum.
- **Хранить permissions напрямую у staff identity.** Отклонено: бизнес-модель требует управляемые роли, повторное использование наборов permissions и аудит состава роли.
- **Использовать одну identity без subtype.** Отклонено: запрет совмещения учебного и административного контуров останется только прикладной договорённостью, а staff-specific invariants будут размыты.
- **Не включать permissions в access JWT.** Отклонено: новый источник истины требует проверку на Gateway и передачу готового access context клиенту. Короткий TTL ограничивает устаревание; отзыв sessions применяет изменения немедленно.
- **Молча связывать OAuth по verified email.** Отклонено: совпавший email не доказывает намерение связать аккаунты и создаёт риск account takeover при ошибках или различиях provider policy.
- **Хранить единый audit log в общей БД.** Отклонено: общая запись ломает service-owned database boundary и требует распределённой транзакции.
- **Переписать ранние migrations.** Отклонено: развёрнутые среды потеряют воспроизводимую историю, а migration drift станет скрытым.

## Последствия

### Положительные

- Модель Auth соответствует пользовательским и административным сценариям MVP.
- Учебные роли совмещаются без дублирования аккаунтов.
- Staff access изменяется данными и аудитируется без релиза кода.
- Session metadata, лимит и отзыв становятся явными и проверяемыми.
- OAuth linking не создаёт скрытого account takeover path.
- Эволюционные migrations сохраняют совместимость существующих сред.

### Отрицательные

- Схема, contracts и token issuer требуют существенной эволюции после T01–T11.
- Access JWT становится больше; каталог permissions должен иметь контролируемый размер.
- Permissions в токене могут быть устаревшими до refresh или принудительного отзыва sessions.
- Инварианты последнего `superadmin`, лимита sessions и subtype требуют транзакционных блокировок и конкурентных integration tests.
- Gateway `init`, admin UI и все сервисы-потребители должны перейти с одиночного `role` на массивы.
- Общий audit screen требует отдельной асинхронной read model.
