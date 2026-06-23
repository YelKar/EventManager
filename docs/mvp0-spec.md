# Event Management Service — MVP0 Specification

## 1. Цель MVP0

MVP0 должен запустить единую базу мероприятий института и закрыть две базовые боли:

1. **Потеря архива** — фотографии, видео и материалы после мероприятий не должны теряться в Telegram-чатах и случайных облаках.
2. **Хаос в создании и согласовании мероприятий** — предложения мероприятий должны проходить понятный процесс модерации с корректной моделью прав.

MVP0 реализуется как **один модульный монолит EMS**. Внутри EMS временно находится локальная авторизация, но API и структура проектируются так, чтобы позже вынести Auth в отдельный сервис или заменить его на SSO Спринта.

## 2. Out of scope

В MVP0 не входят:

- сервис Points и начисление баллов;
- Merchshop;
- task-management;
- отзывы и рейтинг мероприятий;
- оценка вклада участников;
- интеграция с Яндекс.Диском / Google Drive API;
- загрузка файлов напрямую в EMS;
- email/browser/Telegram-уведомления;
- SSO Спринта;
- полноценный API Gateway;
- сложная аналитика и отчёты.

MVP0 хранит только ссылки на внешние архивы, а не сами медиафайлы.

## 3. Архитектура MVP0

### 3.1. Runtime-состав

На MVP0 запускается только EMS:

```text
Frontend → EMS → PostgreSQL
```

EMS обслуживает два логических API-префикса:

```text
/auth/v1/...
/ems/v1/...
```

Физически оба префикса находятся в одном Go-сервисе. Это позволит позже вынести `/auth/v1/...` в отдельный Auth-сервис без массового изменения frontend-кода.

### 3.2. Рекомендуемый стек

Backend:

- Go;
- PostgreSQL;
- REST API;
- миграции через `goose` или `golang-migrate`;
- доступ к БД через `pgx` + `sqlc` или через другой выбранный командой слой доступа;
- Docker Compose для локального запуска.

Frontend:

- React;
- TypeScript;
- Vite;
- React Router или TanStack Router;
- TanStack Query для server state;
- React Hook Form + Zod для форм и валидации;
- Reatom или Zustand для client state при необходимости.

### 3.3. Будущая эволюция

План развития архитектуры:

1. **MVP0** — EMS-монолит со встроенным Auth.
2. **MVP1** — отдельный Points-сервис, EMS начисляет баллы через REST.
3. **MVP2+** — отдельный Merchshop, который обращается к Points напрямую.
4. **Future SSO** — встроенный Auth выносится или заменяется адаптером к SSO Спринта.

EMS должен сохранить локальные доменные профили пользователей даже после выноса Auth.

## 4. Auth и пользователи

### 4.1. Принцип

В MVP0 EMS хранит пользователей и пароли локально. При этом Auth-логика должна быть изолирована в отдельном модуле и не должна протекать в бизнес-модули мероприятий.

Бизнес-модули EMS должны работать только с `CurrentUser`:

```text
CurrentUser
- id              // EMS user id
- auth_id         // future external auth id
- login
- is_global_admin
```

### 4.2. Users

Таблица `users`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | Внутренний ID пользователя EMS |
| `auth_id` | uuid/string | да | ID пользователя в Auth. В MVP0 может совпадать с `id` |
| `login` | string | да | Уникальный логин |
| `password_hash` | string | да в MVP0 | Хеш пароля. После выноса Auth поле удаляется или становится nullable |
| `first_name` | string | да | Имя |
| `last_name` | string | да | Фамилия |
| `patronymic` | string | нет | Отчество |
| `avatar_url` | string | нет | Ссылка на аватар |
| `is_global_admin` | boolean | да | Глобальный админ EMS |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

Ограничения:

- `login` уникален;
- `auth_id` уникален;
- `password_hash` не должен возвращаться через API.

### 4.3. Auth API MVP0

```http
POST /auth/v1/register
POST /auth/v1/login
GET  /auth/v1/me
POST /auth/v1/logout
```

`logout` на MVP0 может быть frontend-only операцией, если используется stateless JWT без server-side blacklist.

### 4.4. Валидация регистрации

`login`:

```regex
^[A-Za-z][A-Za-z_\-0-9]{0,255}$
```

`password`:

- длина от 6 до 255 символов.

`first_name`, `last_name`, `patronymic`:

- до 255 символов;
- только кириллица или только латиница в рамках одного поля;
- `patronymic` опционален.

`avatar_url`:

- в MVP0 можно хранить как URL;
- загрузка файлов в EMS не входит в MVP0.

## 5. Группы и модель доступа

### 5.1. Group

Группа определяет контекст доступа к мероприятиям.

Таблица `groups`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID группы |
| `code` | string | да | Уникальный код, например `students`, `official` |
| `name` | string | да | Название группы |
| `description` | text | нет | Описание |
| `is_public` | boolean | да | Видны ли мероприятия группы неавторизованным/всем пользователям |
| `propose_open_to_all` | boolean | да | Может ли любой авторизованный пользователь создавать предложения в группе |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

Начальные группы:

| code | name | is_public | propose_open_to_all |
|---|---|---:|---:|
| `students` | Студенческие мероприятия | false | true |
| `official` | Официальные мероприятия | true | false |

### 5.2. GroupMember

Таблица `group_members`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID членства |
| `group_id` | uuid | да | Группа |
| `user_id` | uuid | да | Пользователь |
| `can_propose` | boolean | да | Может создавать предложения в группе |
| `can_approve` | boolean | да | Может одобрять/отклонять предложения в группе |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

Ограничение:

```text
unique(group_id, user_id)
```

### 5.3. Правила доступа

| Действие | Кто может |
|---|---|
| Создать группу | Глобальный админ |
| Редактировать группу | Глобальный админ |
| Удалить группу | Глобальный админ, если нет связанных мероприятий |
| Управлять членами группы | Глобальный админ |
| Создать proposal в группе | `group.propose_open_to_all = true` или `GroupMember.can_propose = true` |
| Одобрить proposal | `GroupMember.can_approve = true` или глобальный админ |
| Отклонить proposal | `GroupMember.can_approve = true` или глобальный админ |
| Смотреть event | `group.is_public = true`, член группы или глобальный админ |
| Редактировать event | Организатор события или глобальный админ |
| Управлять ролями event | Организатор события или глобальный админ |
| Записаться на роль | Авторизованный пользователь, который может видеть event |
| Смотреть участников | Пользователь, который может видеть event |
| Создать archive invite | Организатор события или глобальный админ |
| Модерировать archive links | Организатор события или глобальный админ |
| Смотреть confirmed archive links | Пользователь, который может видеть event |

`group_id` у `Proposal` и созданного из него `Event` неизменяем после создания.

## 6. Proposal lifecycle

### 6.1. Proposal

Предложение — заявка на проведение мероприятия. Студенты и другие пользователи создают предложения, а модераторы группы одобряют или отклоняют их.

Таблица `proposals`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID предложения |
| `group_id` | uuid | да | Группа, в которой предлагается мероприятие |
| `proposed_by` | uuid | да | Автор предложения |
| `title` | string | да | Название |
| `description` | text | да | Описание |
| `image_url` | string | нет | Ссылка на изображение |
| `start_at` | timestamp | да | Дата и время начала |
| `end_at` | timestamp | да | Дата и время окончания |
| `status` | enum | да | `pending`, `approved`, `rejected` |
| `reviewed_by` | uuid | нет | Кто рассмотрел |
| `reviewed_at` | timestamp | нет | Когда рассмотрел |
| `review_comment` | text | нет | Комментарий модератора |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

### 6.2. Статусы Proposal

```text
pending → approved
pending → rejected
```

После перехода в `approved` или `rejected` повторная модерация запрещена в MVP0.

### 6.3. Одобрение Proposal

При одобрении proposal EMS в одной транзакции:

1. проверяет право `can_approve` в группе;
2. меняет статус proposal на `approved`;
3. заполняет `reviewed_by`, `reviewed_at`, `review_comment`;
4. создаёт `Event` с тем же `group_id`;
5. создаёт основного организатора события.

По умолчанию основным организатором становится автор proposal. Если UI позволяет выбрать другого основного организатора, выбранный пользователь должен существовать в EMS.

## 7. Event lifecycle

### 7.1. Event

Событие создаётся после одобрения proposal.

Таблица `events`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID мероприятия |
| `proposal_id` | uuid | нет | Исходное предложение |
| `group_id` | uuid | да | Неизменяемая группа мероприятия |
| `title` | string | да | Название |
| `description` | text | да | Описание |
| `image_url` | string | нет | Ссылка на изображение |
| `start_at` | timestamp | да | Дата и время начала |
| `end_at` | timestamp | да | Дата и время окончания |
| `status` | enum | да | `upcoming`, `archived` |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

В MVP0 актуальность события для вкладок вычисляется по датам:

- актуальные: `end_at >= now`;
- архив: `end_at < now` или `status = archived`.

`in_progress` и `completed` можно вычислять на чтении и не хранить как отдельный статус MVP0.

### 7.2. EventOrganizer

Таблица `event_organizers`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID записи |
| `event_id` | uuid | да | Мероприятие |
| `user_id` | uuid | да | Организатор |
| `role` | enum | да | `main_organizer`, `organizer` |
| `created_at` | timestamp | да | Дата создания |

Ограничение:

```text
unique(event_id, user_id)
```

### 7.3. EventRole

Роль внутри мероприятия описывает, какие люди нужны для организации или участия.

Таблица `event_roles`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID роли |
| `event_id` | uuid | да | Мероприятие |
| `title` | string | да | Например: Дизайн, Волонтёр, Фотограф |
| `description` | text | нет | Что нужно делать |
| `slots_needed` | int | да | Сколько мест доступно/нужно |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

Ограничения:

- `slots_needed > 0`;
- роль нельзя удалить, если на неё уже есть участники, если команда не реализует безопасное снятие участников с роли.

### 7.4. EventParticipant

В MVP0 один пользователь записывается на одну роль в рамках одного мероприятия. Поддержку нескольких ролей для одного человека можно добавить позже через `event_participant_roles`.

Таблица `event_participants`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID участия |
| `event_id` | uuid | да | Мероприятие |
| `user_id` | uuid | да | Пользователь |
| `event_role_id` | uuid | да | Выбранная роль |
| `status` | enum | да | `registered`, `attended`, `cancelled` |
| `joined_at` | timestamp | да | Дата записи |

Ограничения:

```text
unique(event_id, user_id)
```

Правила записи:

- пользователь должен быть авторизован;
- пользователь должен иметь доступ к просмотру мероприятия;
- мероприятие не должно быть архивным;
- количество активных участников роли не должно превышать `slots_needed`;
- повторная запись на то же мероприятие запрещена, если текущая запись не `cancelled`.

## 8. Archive lifecycle

### 8.1. Назначение

Архив хранит подтверждённые ссылки на внешние облачные хранилища с фото, видео и другими материалами мероприятия.

### 8.2. ArchiveInvite

Организатор может создать публичную ссылку для сбора архивов после окончания мероприятия.

Таблица `archive_invites`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID invite |
| `event_id` | uuid | да | Мероприятие |
| `token_hash` | string | да | Хеш публичного токена |
| `created_by` | uuid | да | Кто создал invite |
| `created_at` | timestamp | да | Дата создания |
| `expires_at` | timestamp | нет | Срок действия |
| `revoked_at` | timestamp | нет | Дата отзыва |

Публичный URL может выглядеть так:

```text
/archive/submit/{token}
```

Frontend по этому URL открывает форму отправки ссылки.

### 8.3. ArchiveLink

Таблица `archive_links`:

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `id` | uuid | да | ID ссылки |
| `event_id` | uuid | да | Мероприятие |
| `url` | string | да | URL на облако/архив |
| `comment` | text | нет | Комментарий отправителя |
| `proposed_by` | uuid | нет | Авторизованный пользователь, если был |
| `proposer_name` | string | нет | Имя неавторизованного отправителя |
| `status` | enum | да | `pending`, `confirmed`, `rejected` |
| `reviewed_by` | uuid | нет | Организатор, который рассмотрел |
| `reviewed_at` | timestamp | нет | Когда рассмотрел |
| `created_at` | timestamp | да | Дата создания |
| `updated_at` | timestamp | да | Дата обновления |

### 8.4. Правила архива

- предлагать архивную ссылку может любой пользователь с валидным archive token;
- авторизация для отправки ссылки не обязательна;
- новые ссылки создаются в статусе `pending`;
- подтверждать или отклонять ссылки может организатор мероприятия или глобальный админ;
- только `confirmed` ссылки видны пользователям на карточке мероприятия;
- видеть подтверждённые ссылки могут только пользователи, которые имеют доступ к мероприятию.

## 9. REST API MVP0

### 9.1. Auth

```http
POST /auth/v1/register
POST /auth/v1/login
GET  /auth/v1/me
POST /auth/v1/logout
```

### 9.2. Profile / Users

```http
GET   /ems/v1/users/me
PATCH /ems/v1/users/me
GET   /ems/v1/admin/users
GET   /ems/v1/admin/users/{user_id}
PATCH /ems/v1/admin/users/{user_id}
```

### 9.3. Groups

```http
GET    /ems/v1/groups
POST   /ems/v1/groups
GET    /ems/v1/groups/{group_id}
PATCH  /ems/v1/groups/{group_id}
DELETE /ems/v1/groups/{group_id}
```

### 9.4. Group members

```http
GET    /ems/v1/groups/{group_id}/members
POST   /ems/v1/groups/{group_id}/members
PATCH  /ems/v1/groups/{group_id}/members/{user_id}
DELETE /ems/v1/groups/{group_id}/members/{user_id}
```

### 9.5. Proposals

```http
POST /ems/v1/proposals
GET  /ems/v1/proposals/my
GET  /ems/v1/proposals/moderation
GET  /ems/v1/proposals/{proposal_id}
POST /ems/v1/proposals/{proposal_id}/approve
POST /ems/v1/proposals/{proposal_id}/reject
```

### 9.6. Events

```http
GET   /ems/v1/events
GET   /ems/v1/events/{event_id}
PATCH /ems/v1/events/{event_id}
POST  /ems/v1/events/{event_id}/archive
```

Примеры фильтров:

```http
GET /ems/v1/events?tab=actual
GET /ems/v1/events?tab=archive
GET /ems/v1/events?group=students
```

### 9.7. Event organizers

```http
GET    /ems/v1/events/{event_id}/organizers
POST   /ems/v1/events/{event_id}/organizers
DELETE /ems/v1/events/{event_id}/organizers/{user_id}
```

### 9.8. Event roles

```http
GET    /ems/v1/events/{event_id}/roles
POST   /ems/v1/events/{event_id}/roles
PATCH  /ems/v1/events/{event_id}/roles/{role_id}
DELETE /ems/v1/events/{event_id}/roles/{role_id}
```

### 9.9. Event participants

```http
GET    /ems/v1/events/{event_id}/participants
POST   /ems/v1/events/{event_id}/participants
DELETE /ems/v1/events/{event_id}/participants/me
POST   /ems/v1/events/{event_id}/participants/{participant_id}/attended
```

### 9.10. Archive

```http
POST /ems/v1/events/{event_id}/archive-invites
GET  /ems/v1/events/{event_id}/archive-links
GET  /ems/v1/archive/public/{token}
POST /ems/v1/archive/public/{token}/links
POST /ems/v1/events/{event_id}/archive-links/{link_id}/confirm
POST /ems/v1/events/{event_id}/archive-links/{link_id}/reject
```

## 10. Frontend pages MVP0

### 10.1. Account

- login page;
- registration popup/page;
- personal profile page;
- admin user profile page.

### 10.2. Events

- event list with tabs:
  - actual;
  - archive;
- group filter;
- proposal creation form;
- moderator proposal list;
- event details page;
- event role registration popup/form;
- participant list grouped by roles;
- organizer controls for roles and archive.

### 10.3. Archive

- public archive submission page by token;
- archive moderation block on event details page;
- confirmed archive links block on archived event page.

### 10.4. Admin

- group list;
- group create/edit form;
- group members management;
- user list;
- user profile edit.

## 11. Validation rules

### 11.1. Proposal / Event

- `title` is required and should be reasonably limited, for example 1–255 chars;
- `description` is required;
- `start_at < end_at`;
- `group_id` must point to an existing group;
- `image_url`, if present, must be a valid URL;
- `group_id` cannot be changed after creation.

### 11.2. Group

- `code` is unique;
- `code` should match `^[a-z][a-z0-9_-]{1,63}$`;
- group cannot be deleted if there are linked proposals/events;
- changing `is_public` affects visibility of all linked events.

### 11.3. EventRole

- `title` is required;
- `slots_needed > 0`;
- role deletion is blocked if active participants exist for the role.

### 11.4. EventParticipant

- participant cannot register if slots are full;
- participant cannot register twice for the same event;
- participant cannot register for archived event;
- only organizer/global admin can mark `attended`.

### 11.5. ArchiveLink

- `url` is required and must be a valid URL;
- `comment` is optional;
- public token must be valid, not expired and not revoked;
- only pending links can be confirmed/rejected.

## 12. Error format

All API errors should use a common format:

```json
{
  "code": "forbidden",
  "message": "You do not have permission to perform this action",
  "details": {}
}
```

Validation errors:

```json
{
  "code": "validation_error",
  "message": "Invalid request body",
  "details": {
    "title": "Title is required"
  }
}
```

## 13. Non-functional requirements

- Every request should have or receive `X-Request-Id`.
- Passwords must be stored only as hashes.
- Archive public tokens must be stored as hashes, not raw tokens.
- API should not leak private group events to unauthorized users.
- Business operations that update multiple tables must be transactional.
- Proposal approval must be atomic.
- Future Points integration must use idempotency keys.

## 14. Suggested internal Go structure

```text
cmd/
  ems/
    main.go

internal/
  platform/
    config/
    database/
    logger/
    httpserver/
    security/

  modules/
    auth/
    users/
    groups/
    proposals/
    events/
    archive/

api/
  openapi.yaml

docs/
  mvp0-spec.md

migrations/
```

Auth module owns login/register/token issuing in MVP0. Other modules must depend only on authenticated `CurrentUser`, not on password or token internals.

## 15. Open questions before implementation

These questions can be finalized during implementation planning:

1. Should one participant be allowed to hold several event roles in MVP0, or only one?
2. Should event status `completed` be stored, or should completion be computed from `end_at`?
3. Should registration use JWT access tokens only, or access + refresh tokens?
4. Should avatar/image uploading be postponed completely, or should MVP0 include local/S3-compatible upload?
5. Should proposal include draft event roles, or should roles be added only after event approval?
6. Should non-authenticated users see public event details, or only public event lists?

Recommended defaults for MVP0:

1. one participant — one role;
2. compute completion from `end_at`;
3. JWT access token is enough for first demo, refresh token can be added later;
4. postpone upload, store URLs;
5. add roles after approval;
6. allow non-authenticated users to see public event details and confirmed archive links.
