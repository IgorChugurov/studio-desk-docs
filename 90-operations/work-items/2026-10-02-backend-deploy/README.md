---
id: 2026-10-02-backend-deploy
title: Автоматическое развёртывание бэкенда на сервер
status: done
updated_at: 2026-10-02
---

# Автоматическое развёртывание бэкенда на сервер

## Цель

При пуше в ветку `development` репозитория `studio-desk-backend` GitHub Actions собирает образ и разворачивает бэкенд на сервере `31.220.80.11` вместе с PostgreSQL: миграции под `studio_desk_owner`, затем запуск приложения. Бэкенд доступен по HTTPS на `api.studio-desk.axondigital.xyz`.

## Границы

- Не делаем: фронтенд, рекламную страницу, сайты студий и общий сертификат на все поддомены, резервные копии.
- Не трогаем чужие проекты на сервере (nginx-сайты `api.oblikflow.com`, `assistant.axondigital.xyz` и их процессы).
- API-контракт админки платформы — следующая задача в очереди, не эта.

## Этап

Завершено.

## Закрытие

**Результат.** Пуш в `development` репозитория `studio-desk-backend` прогоняет тесты, собирает образ `ghcr.io/igorchugurov/studio-desk-backend`, и деплой на сервер `31.220.80.11` применяет миграции и запускает бэкенд вместе с PostgreSQL. Бэкенд доступен по `https://api.studio-desk.axondigital.xyz`. Канон: `03-architecture/deployment.md`; рецепт для следующих сервисов: `04-engineering-rules/deploy-new-service.md`.

**Изменения по ходу.** Видимость образа в GHCR наследуется от открытого репозитория — ручное переключение в Public не понадобилось. Все сервисы StudioDesk разворачиваются под пользователем `studio-desk`, новые — в `/opt/studio-desk/<сервис>/`, у каждого репозитория свой ключ.

**Проверка.** Два запуска CI зелёные (тесты и деплой): https://github.com/IgorChugurov/studio-desk-backend/actions/runs/37037113148 и https://github.com/IgorChugurov/studio-desk-backend/actions/runs/37037904971. В логе деплоя: `Migrations applied`, `{"status":"ok","database":"ok"}`, `Deployed eab5c73`. Снаружи: `https://api.studio-desk.axondigital.xyz/api/health` → `200 {"status":"ok","database":"ok"}`; HTTP → `301` на HTTPS; `/api/platform/docs` → `404` (документация в production выключена). Отпечаток сервера в `DEPLOY_KNOWN_HOSTS` сверен владельцем.

Commits: find with `git log --grep "2026-10-02-backend-deploy"`.
