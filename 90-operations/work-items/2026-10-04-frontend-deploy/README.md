---
id: 2026-10-04-frontend-deploy
title: Развёртывание фронтенда на сервер
status: done
updated_at: 2026-10-04
---

# Развёртывание фронтенда на сервер

## Цель

Пуш в ветку `development` репозитория `studio-desk-frontend` собирает образ и разворачивает фронтенд на сервере `31.220.80.11` по схеме бэкенда (`03-architecture/deployment.md`, `04-engineering-rules/deploy-new-service.md`). Страница админки платформы открывается по HTTPS на `admin.studio-desk.axondigital.xyz` и показывает версию API боевого бэкенда.

## Границы

- Не меняем экраны, прокси по хостам и бэкенд.
- Хосты `app.` и публичных сайтов студий, их DNS и сертификаты не входят.
- Серверные шаги под root (ключ, папка, nginx, сертификат) выполняет владелец; секреты GitHub создаются через `gh` с подтверждением владельца.
- Пуш и коммит только по явной просьбе владельца.

## Этап

`feature-workflow.md`, лёгкий маршрут не применим (затрагивает сервер и CI): реализация выполнена, выкладка принята владельцем. Ведёт фронтенд.

## Закрытие

**Результат.**
- `studio-desk-frontend`: `Dockerfile` (Next в режиме `standalone`, Node 24), `.dockerignore`, `deploy/` (compose на `127.0.0.1:3101`, `deploy.sh` с проверкой по заголовку `Host`, конфиг nginx, README), задание `deploy` в GitHub Actions. Секреты репозитория `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS` созданы через `gh` с отдельным ключом деплоя.
- Сервер (шаги владельца): запись DNS `admin.studio-desk` → `31.220.80.11`, ключ деплоя, папка `/opt/studio-desk/frontend/`, сайт nginx, сертификат `certbot`.
- `studio-desk-backend`: найден и исправлен дефект предыдущей задачи — `CORS_EXTRA_ORIGINS` не передавалась в контейнер; добавлена в `deploy/docker-compose.yml`.

**Чем проверено.**
- Локально до выкладки: образ собран, запрос с `Host: admin.studio-desk.axondigital.xyz` — 200, хосты `app.` и чужой — 404, процесс под пользователем `node`; `docker compose config` с настройкой и без неё; `typecheck`, `lint`, `format:check`, `build` без ошибок, `pnpm test` — `Tests 13 passed (13)`.
- GitHub Actions 2026-10-04: фронтенд (`Add Docker image and automatic deploy to server`) — `test: success`, `deploy: success`; бэкенд (`Pass CORS_EXTRA_ORIGINS to the backend container`) — успешный запуск.
- Боевые адреса: `https://admin.studio-desk.axondigital.xyz/` — 200, `http://` — 301 на HTTPS, `/platform` — 404. Запрос к `https://api.studio-desk.axondigital.xyz/api/platform/version` с `Origin: http://admin.localhost:3001` вернул `Access-Control-Allow-Origin: http://admin.localhost:3001`; с `Origin: http://localhost:3001` заголовка нет.
- Скриншот `https://admin.studio-desk.axondigital.xyz`: «API version: 0.0.1».
- Приёмка владельца: «делай close work SD».

**Не проверено.** Сохранение сессии при входе с локальной страницы на боевой API (`SameSite=Strict`): вопрос фазы 1. Переход `ubuntu-latest` на Ubuntu 26 (19 октября 2026): решено ничего не делать и посмотреть на результат.

**Коммиты:** найти через `git log --grep "2026-10-04-frontend-deploy"`, в трёх репозиториях (`studio-desk-frontend`, `studio-desk-backend`, `studio-desk-docs`).
