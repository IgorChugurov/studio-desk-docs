# Статус проекта StudioDesk

**Обновлено:** 2026-11-12 (главный бот, по состоянию веток `development`)
**Проверенные ревизии:** `studio-desk-docs` `a8f0b81` (2026-11-12 19:48 UTC+1), `studio-desk-frontend` `b2a502a` (2026-11-12 19:48 UTC+1), `studio-desk-backend` `92fc26b` (2026-11-05 21:40 UTC+1).

Не канон. Источник правды — канон в зонах `00`–`04` и код. Здесь сводка с путями-доказательствами. «Не проверено» значит, что бот этого не открывал или не смог подтвердить.

## Срочное

- **CI красный на обеих головах `development`, деплой пропущен** (запуски 2026-11-12 ~19:49 UTC+1):
  - фронтенд `b2a502a`: `pnpm typecheck` падает в `src/studio/schedule/clock.ts` и `src/studio/schedule/model.ts` (TS2769/TS18048/TS2532, «possibly undefined»). Задание `deploy` — `skipped`. Значит, сегодняшнее «прошедшие дни закрыты» на `app.` не выложено. Ссылка: `https://github.com/IgorChugurov/studio-desk-frontend/actions/runs/43283790400`.
  - бэкенд `92fc26b`: 8 из 8 тестов `test/studio-schedule.e2e-spec.ts` падают (`expected 409 to be 201`), остальные 109 зелёные. В тесте зашита дата `const DAY = '2026-11-03'` — она уже в прошлом, создание слота отклоняется как начавшееся. Та же причина уже записана в `01-product/platform-admin/organization-presets-backend-brief-feedback.md` (дату тогда сдвигали с `2026-10-20`). `deploy` — `skipped`. Ссылка: `https://github.com/IgorChugurov/studio-desk-backend/actions/runs/43283792240`.
  - Что сейчас реально выложено на сервер — не проверено.

## Разделы (домены)

| Раздел | Этап | Последнее принятое | Ждёт решения Игоря | Следующее |
| --- | --- | --- | --- | --- |
| **Schedule** (сейчас в работе) | Канон `01-product/organization-admin/schedule.md`; бэкенд и фронтенд реализованы (этапы 3 и 5 от 2026-10-27); идёт обсуждение дизайна — work item `2026-11-05-organization-schedule-design`, этап 2 | 2026-11-12: в виде со столбцами ресурсов шапка дня сворачивает день, иконка справа, прошедший день изначально закрыт, сворачивание не запоминается (`schedule.md`, коммит `a8f0b81`; код `b2a502a`) | Остальные вопросы из цели work item: нужна ли отдельная строка 00–08, живёт ли сетка в границах рабочего времени ресурса, как неделя соотносится с плакатными примерами на один ресурс. Открытые вопросы `schedule.md`: действия специалиста со своими слотами; удаление слота с записями клиентов; автопродление по шаблонам | Починить CI (см. «Срочное»); продолжить обсуждение темы |
| **Публичное приложение для клиентов + Clients and bookings в админке** | Не начато. Канона темы нет (в `01-product/` нет папки публичного сайта и файла клиентов/записей). Вкладка Clients and bookings — пустая страница (`overview.md`, «Пустые разделы»; `src/app/studio/(app)/[section]/page.tsx`). `/api/public` — только модуль и guard без контроллеров (`backend/src/apis/public/public-api.module.ts`). Фронтенд отвечает 404 на любой хост, кроме `admin.`/`app.` (`frontend/src/proxy.ts`) | Принципы в `profile.md`: клиент организации ≠ глобальный вход; `Booking` ≠ `Visit` | Канал входа клиентов на публичном сайте (`platform-admin/overview.md`, «Открытые вопросы»); когда открывать тему | Не знаю: в work items очереди нет |
| **Subscriptions (абонементы)** | Не начато. Пустая вкладка; таблиц нет (в `0000_bumpy_nighthawk.sql` нет таблиц абонементов) | В пресете есть `billingConfig.settlementMode` (`subscription` / `direct_check` / `external_order`), владелец может менять его у своей организации (`overview.md`, 2026-11-05) | Открытые вопросы `profile.md`: условия заморозки; расчёт стоимости визита при списании | Не знаю |
| **Accounting and reports** | Не начато. Пустая вкладка; таблицы `accounting_events` нет | Принцип журнала `accounting_events` как моста к OblikFlow (`profile.md`, п. 4) | Детальная схема `accounting_events` (`profile.md`, «Открытые вопросы») | Не знаю |
| **Документы / Chart** | Не начато. Только упоминание в `profile.md` (п. 5) и как будущий `chartConfig`. В коде секции `chart` нет (`backend/src/common/organization-config.ts`, `SECTIONS`) | — | Детали модуля (`profile.md`, «Открытые вопросы») | Не знаю |
| **Work orders** | Не начато. Только `profile.md` (автосервисы, `external_order`, будущий `workOrderConfig`). Секции `work-orders` в коде нет | — | Детали модуля (`profile.md`, «Открытые вопросы») | Не знаю |
| **Catalogs** (для полноты) | Реализовано: единые ресурсы с кубиками. Канон `catalogs.md`; бэкенд `backend/src/apis/organization/resources/`, таблицы `resource`, `resource_file`, `resource_relation`, `staff_resource`; фронтенд `frontend/src/studio/catalogs/`, `src/app/studio/(app)/catalogs/[typeId]/…` | 2026-10-27 этапы 2 и 5 (feedback-файлы) | Удаление/архивация ресурса с историческими слотами (`catalogs.md`, «Открытые вопросы») | Не знаю |
| **Журнал действий** (для полноты) | Обсуждение, этап 2 (`2026-10-08-studio-admin-activity-log`). Кода и таблиц нет | — | Границы темы не согласованы | Не знаю |

## Сквозное

| Тема | Этап | Последнее принятое | Ждёт решения Игоря | Следующее |
| --- | --- | --- | --- | --- |
| **Пресеты и organization** | Реализовано. Канон `profile.md`, `platform-admin/overview.md`, `organization-admin/overview.md`. Бэкенд `backend/src/apis/platform/presets/`, `.../organizations/`; таблицы `preset`, `organization`, `organization_resource_type`, `organization_section`. Фронтенд `src/app/platform/(app)/presets/…`, `…/organizations/…`, `src/platform/presets/` | 2026-11-05: владелец меняет `scheduleConfig` и `billingConfig` только своей организации (`overview.md`, коммит `449203d`) | Письмо владельцу после создания; свой домен; журнал действий администратора платформы (`platform-admin/overview.md`, «Открытые вопросы») | Не знаю |
| **Auth и сессии** | Реализовано для платформы и организации. Канон `03-architecture/authentication.md`. Код `backend/src/apis/platform/auth/`, `backend/src/apis/organization/auth/` (code, sign-in, select/switch-organization, impersonation/exchange, refresh, sign-out), `backend/src/common/auth/`. Вход клиентов — нет | 2026-10-08 фазы 2–3 бэкенда админки студии | Несколько организаций в разных вкладках (`authentication.md`); вход под организацией под другой ролью (`organization-admin/overview.md`); сохранение сессии с локальной страницы на боевой API (`SameSite=Strict`) — «не проверено» в work items `2026-10-04-*` | Не знаю |
| **Двойной фильтр разделов** | Реализовано. Бэкенд: `backend/src/apis/organization/access/studio-access.service.ts` проверяет и `role_section`, и `organization_section`. Фронтенд: вкладки из `sections` сессии, `src/studio/sections.ts` | `profile.md`, п. 2; бэкенд-этап 1 от 2026-10-27 | Что ещё доступно только владельцу (`overview.md`) | — |
| **Дизайн-система `src/shared/ui`** | Есть: `avatar, breadcrumbs, button, command, dropdown-menu, field, filter-menu, input, modal, multi-select, notice, pagination, popover, search-input, select, table, tabs, toaster` (`frontend/src/shared/ui/`). Канон `02-design/ui-kit-and-pages.md` (Last verified 2026-11-05). Figma в текущей работе нет (`.cursor/rules/00-workspace-core.mdc`) | 2026-11-05: канон кита и типов страниц | Обводка фокуса `focus-visible` — отложена (`2026-10-05-frontend-kit-login`); организация Figma (`02-design/flow-design.md`) | — |
| **API-конвенции** | Канон `03-architecture/api-conventions.md`, открытых вопросов нет. Код `backend/src/common/errors/`, `backend/src/common/validation/`, `backend/src/common/openapi/`. Пути: `/api/platform`, `/api/organization`, `/api/public` (`backend/src/app.module.ts`) | — | Доступ к OpenAPI на сервере (`backend-structure.md`); RLS — когда и как (`backend-structure.md`) | — |
| **Деплой** | Бэкенд и фронтенд разворачиваются GitHub Actions при пуше в `development` (`03-architecture/deployment.md`; `backend/deploy/`, `frontend/deploy/`, nginx для `api.`, `admin.`, `app.`). Сейчас обе последние выкладки пропущены из‑за красных тестов | 2026-10-04 фронтенд-деплой | Резервные копии БД; ограничение ключа деплоя; общий DNS и сертификат для сайтов организаций (`deployment.md`) | Починить CI |

## Зеркало work items

«Статус в файле» — из шапки `90-operations/work-items/<id>/README.md`. «Факт по коду» — оценка бота по структуре репозиториев на ревизиях выше.

| id / название | Статус в файле | Факт по коду | Доказательство |
| --- | --- | --- | --- |
| `2026-10-02-backend-deploy` — автодеплой бэкенда | done | сделано | `backend/deploy/`, `backend/.github/workflows/ci.yml`, `deploy/nginx/api.studio-desk.axondigital.xyz.conf` |
| `2026-10-02-nest-app-foundation` — основа Nest | done | сделано (путь `/api/studio` позже стал `/api/organization`) | `backend/src/app.module.ts` (RouterModule: platform, organization, public), `backend/src/database/` |
| `2026-10-02-platform-admin-design-review` — проверка Figma админки платформы | **in-progress** | **не сделано и, похоже, обойдено**: Figma не принята, но админка платформы построена от `design-brief.md` и кита по решению владельца; правило «Figma в текущей работе нет» | `frontend/src/platform/`, `frontend/src/app/platform/`; work item `2026-10-05-frontend-kit-login` («Решение владельца: страница делается от задания дизайнеру и кита, а не от кадров Figma»); `.cursor/rules/00-workspace-core.mdc` §3 |
| `2026-10-03-platform-admin-api-contract` — контракт API платформы | done | сделано | `01-product/platform-admin/api-contract.md` |
| `2026-10-03-platform-admin-phase-2` — вход по коду | done | сделано | `backend/src/apis/platform/auth/`, `backend/src/common/auth/mailer.ts` |
| `2026-10-04-frontend-deploy` — деплой фронтенда | done | сделано | `frontend/Dockerfile`, `frontend/deploy/` |
| `2026-10-04-frontend-foundation` — основа фронтенда | done | сделано; модуль `src/platform/api-version/` остался и нигде не импортируется | `frontend/src/proxy.ts`, `backend/src/apis/platform/version/version.controller.ts` |
| `2026-10-04-platform-admin-studios` — фаза 3, студии | done | сделано, переименовано в organizations | `backend/src/apis/platform/organizations/` |
| `2026-10-05-backend-studio-status-impersonate` — деактивация, активация, вход под студией | done | сделано | `organizations.controller.ts`: `:id/deactivate`, `:id/activate`, `:id/impersonate`; таблица `handoff_code` |
| `2026-10-05-frontend-kit-login` — кит, вход, каркас | done | сделано; хвосты из «Не сделано» частично живы: `platform-admin/frontend-brief-feedback.md` нет (не найден в `01-product/platform-admin/`), `focus-visible` не решён | `frontend/src/shared/ui/`, `frontend/src/platform/sign-in/`, `frontend/src/platform/studios/` |
| `2026-10-06-backend-studio-admin` — бэкенд админки студии | done | сделано, под `/api/organization` | `backend/src/apis/organization/{auth,me,settings,staff,access}/` |
| `2026-10-06-studio-admin-base-and-roles` — канон основы и ролей | done | сделано, канон переехал | `01-product/organization-admin/overview.md` |
| `2026-10-08-agent-canon-before-options` — правило «сначала канон» | done | сделано | `00-governance/approval-rules.md`, раздел «Агент — неоднозначность» |
| `2026-10-08-studio-admin-activity-log` — журнал действий | **in-progress** | не начато в коде (этап 2 — обсуждение) | нет таблиц журнала в `backend/src/database/migrations/0000_bumpy_nighthawk.sql`; поиск `activity\|audit` в `backend/src`, `frontend/src` — пусто |
| `2026-10-08-studio-admin-catalogs-backend` — бэкенд залов | done | сделано, затем заменено единым каталогом | `backend/src/apis/organization/resources/`; хелперы остались в `backend/src/apis/organization/halls/` |
| `2026-10-08-studio-admin-catalogs` — каталоги (залы, тренеры, виды) | cancelled | заменено | `frontend/src/studio/catalogs/resource-*.tsx` |
| `2026-10-08-studio-admin-frontend` — фронтенд админки студии | done | сделано | `frontend/src/app/studio/`, `frontend/src/studio/` |
| `2026-10-09-studio-admin-catalogs-trainers-backend` — тренеры и виды занятий | cancelled | заменено (миграции `0010`/`0011` больше нет, одна `0000`) | `backend/src/database/migrations/meta/_journal.json` |
| `2026-10-14-studio-admin-catalogs-fields-frontend` | cancelled | заменено | — |
| `2026-10-14-studio-admin-catalogs-schedule-fields` | cancelled | заменено | — |
| `2026-10-17-studio-admin-schedule-week-frontend` | cancelled | заменено | `frontend/src/studio/schedule/week-page.tsx` |
| `2026-11-05-organization-schedule-design` — дизайн расписания | **in-progress** | частично: одно решение (сворачивание дня) уже в каноне и коде 2026-11-12; остальные вопросы цели открыты; CI фронтенда на этом коммите красный | `01-product/organization-admin/schedule.md`, docs `a8f0b81`, frontend `b2a502a` |
| `2026-11-05-workspace-context-canon` — стартовый контекст и главенство канона | **in-progress** | **выглядит сделанным, не закрыто**: коммиты с id есть во всех трёх репо; нет раздела «Закрытие» и артефакта проверки | docs `5c2a33b`, backend `92fc26b`, frontend `23dabdc`; файлы `.cursor/rules/00-workspace-core.mdc`, `frontend/.cursor/rules/project.mdc`, `backend/.cursor/rules/project.mdc`, короткие `AGENTS.md` |

Работа без work item: этапы 1–5 от 2026-10-27 (organization, пресеты, единые ресурсы, слоты расписания, UI) и правка настроек владельца от 2026-11-05 сделаны и закоммичены (docs `f9fce35`…`b3d75a2`, `449203d`; backend `f9e115c`, `8d77fc8`, `b0d10cf`; frontend `b8cbce7`, `350f615`, `419835e`), но отдельной записи в `90-operations/work-items/` для них не найдено.

## Расхождения

### Work items
1. `2026-11-05-workspace-context-canon` — по коду сделано, статус `in-progress`, закрытия нет.
2. `2026-10-02-platform-admin-design-review` — `in-progress` с 2026-10-02, хотя по более поздним решениям Figma в текущей работе нет и экраны уже построены. Нужен статус от Игоря (`parked` / `cancelled` / продолжать).
3. `2026-11-05-organization-schedule-design` — в границах сказано «канон темы не меняется, пока тема не согласована», но 2026-11-12 часть решений уже записана в `schedule.md` и в код. Возможно, на это было отдельное разрешение в чате — бот этого не видел.
4. Этапы 2026-10-27 и правка 2026-11-05 — без work item (см. выше), хотя `00-governance/work-items.md` называет work item единственным местом состояния задачи.

### Старые пути `studio-admin` и `/api/studio`
Канонические документы-владельцы (`profile.md`, `overview.md`, `catalogs.md`, `schedule.md`, `02-design/`, `03-architecture/`, `04-engineering-rules/`) старых путей не содержат. Остатки:

- Задания (производные документы в `01-product/`):
  - `organization-admin/backend-brief.md`: «Derived from: `01-product/studio-admin/overview.md`» (путь не существует), API `/api/studio` (стр. 5, 9, 19).
  - `organization-admin/frontend-brief.md`: base `…/api/studio` (стр. 25, 67).
  - `organization-admin/catalogs-backend-brief.md` (стр. 11), `catalogs-trainers-backend-brief.md` (стр. 35), `schedule-backend-brief.md` (стр. 11): `/api/studio`.
  - `platform-admin/organization-presets-backend-brief.md` (стр. 18) и `platform-admin/stage4-frontend-brief.md` (стр. 19, 64) — упоминают `/api/studio` как то, что надо заменить: это описание перехода, а не ошибка.
- Work items (бот их не правит): ссылки на несуществующие `01-product/studio-admin/*.md` в `2026-10-06-*`, `2026-10-08-*`, `2026-10-09-*`, `2026-10-14-*`, `2026-10-17-*`; `/api/studio` в `2026-10-02-nest-app-foundation`, `2026-10-05-backend-studio-status-impersonate`, `2026-10-06-backend-studio-admin`.
- Рабочие материалы: `plan.md` (стр. 79, 95 — `01-product/studio-admin/schedule.md`), `90-operations/2026-10-07-evolve-ipd-chat-report.md`.
- Код бэкенда — публичный путь правильный (`/api/organization`, `backend/src/app.module.ts`), но:
  - `backend/docs/README.md`: «Вход в админку студии» → `./studio-admin.md`;
  - `backend/docs/studio-admin.md`: шапка ссылается на `organization-admin` (верно), но текст описывает старые таблицы `studio_sign_in_code`, `studio_session`, `studio_selection_ticket`, `studio_staff` и миграции `0004`–`0006`. В коде таблицы `organization_*`, миграция одна — `0000_bumpy_nighthawk.sql`;
  - внутренние имена: `StudioApiModule`, файлы `studio-*.ts`, папка `src/apis/organization/halls/`, коллекция `StudioDesk_Studio.postman_collection.json` (содержимое не проверено), логин PostgreSQL `studio_api` (оставлен намеренно, см. `decisions.md`).
- Код фронтенда — API-путь правильный (`/api/organization`: `src/shared/config/public-env.ts`, `next.config.ts`), внутренние имена старые: `src/app/studio/`, `src/studio/`, `src/platform/studios/`, `NEXT_PUBLIC_STUDIO_ADMIN_URL`, тип поверхности `'studio'` (`src/shared/hosts/surface.ts`). По `platform-admin/stage4-frontend-brief-feedback.md` внутренние папки оставлены сознательно.
- Канон `02-design/ui-kit-and-pages.md` (Scope): «админкой организации (`/studio`)» — это внутренний префикс маршрутов фронтенда, совпадает с кодом; не ошибка, но название старое.

### Прочее
- Схема Drizzle: `backend/src/database/migrations/meta/0000_snapshot.json` не пересобран после ручной правки `0000`; следующий `drizzle-kit generate` даст неверный дифф (`catalogs-unified-backend-brief-feedback.md`, §3; `schedule-unified-backend-brief-feedback.md`, §4).
- В контракте расписания упоминается роль БД `organization_api`, в базе её нет, используется `studio_api` (`schedule-unified-backend-brief-feedback.md`, §3).
- `frontend/README.md` (стр. 16) упоминает `admin.localhost` в отрицательном смысле («answers Not found») — актуально, не ошибка.

## Что бот не проверял
- Живые адреса `admin.`/`app.`/`api.` и какая версия там сейчас.
- Тексты всех заданий и контрактов целиком; шаблоны `00-governance/templates/` — только список файлов.
- Содержимое Postman-коллекций.
- `plan.md` и `research/` — не использовались.
