---
id: 2026-10-02-nest-app-foundation
title: Основа NestJS-приложения
status: done
updated_at: 2026-10-02
---

# Основа NestJS-приложения

## Цель

Сделать основу NestJS-приложения в пустом репозитории `studio-desk-backend`, на которой затем будет выполняться `01-product/platform-admin/backend-brief.md`. Состав основы определяется на этапе обсуждения.

## Границы

- Не делаем ничего из `backend-brief.md`: API-контракт, вход по коду, студии, токены, деактивацию, вход под студией.
- Не трогаем фронтенд и дизайн.
- Открытый вопрос: входит ли в основу решение по локальной базе данных для разработки (открытый пункт в `plan.md`).

## Этап

`feature-workflow.md`: обсуждение завершено, канон записан (`03-architecture/backend-structure.md`, `03-architecture/api-conventions.md`, `04-engineering-rules/backend.md`). Основа реализована в `studio-desk-backend` и принята владельцем; открытые вопросы `api-conventions.md` закрыты. Ведёт бэкенд.

## Закрытие

**Результат.**
- `studio-desk-backend`: Nest 12 на Node 24 и pnpm; три API (`/api/platform`, `/api/studio`, `/api/public`) со своими guards; четыре пользователя PostgreSQL и стартовый скрипт; Drizzle, первая миграция со служебной таблицей `foundation_check` и выдачей прав, файл ожидаемых прав; проверка данных на Zod, единый формат ошибок, OpenAPI вне production; `/api/health`; Vitest с четырьмя тестами механизма; `docker-compose.yml` с PostgreSQL 18, `Dockerfile`, GitHub Actions.
- `studio-desk-docs`: канон `03-architecture/backend-structure.md`, `03-architecture/api-conventions.md`, `04-engineering-rules/backend.md` с README зон; в `plan.md` локальная разработка отмечена решённой.
- Открытый вопрос из границ решён: локальная база вошла в основу (Docker Desktop, PostgreSQL в контейнере).

**Чем проверено.**
- Вывод команд 2026-10-02: `pnpm typecheck`, `pnpm lint`, `pnpm format:check`, `pnpm build` без ошибок; `pnpm test` — `Tests 6 passed (6)`; `pnpm test:e2e` — `Tests 24 passed (24)`.
- Контрольная поломка (открытый guard платформы и лишнее право `public_api`) уронила 4 теста; после отката всё зелёное.
- Docker-образ: миграции отдельным шагом — `Migrations applied`; `/api/health` — `{"status":"ok","database":"ok"} [200]`; документация в production — `[404]`.
- Приёмка владельца: «да. все норм».
- GitHub Actions до пуша не запускался.

**Коммиты:** найти через `git log --grep "2026-10-02-nest-app-foundation"`.
