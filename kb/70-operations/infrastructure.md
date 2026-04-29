---
title: Инфраструктура
status: active
last_updated: 2026-04-28
references:
  - ../40-system/api.md
  - ../40-system/database/overview.md
referenced_by: []
---

# Инфраструктура

## VPS

- **Провайдер**: (зафиксировать в локальном `~/secrets-trade/`)
- **IP**: `108.61.167.168`
- **OS**: Ubuntu 22.04
- **Доступ**: root + SSH-пароль (в локальном `~/secrets-trade/credentials.md`, не в репо)
- **Папка проекта**: `/var/www/trade/`
- **Домен**: `trade-abkhazia.com` (Namecheap)

## Структура `/var/www/trade/`

```
/var/www/trade/
├── frontend/               ← Vite-проект
│   ├── src/App.jsx         ← копия trade_app_v3.jsx
│   ├── src/main.jsx        ← точка входа React
│   ├── vite.config.js
│   └── package.json
├── public/                 ← собранный бандл (output Vite)
│   ├── index.html
│   └── assets/index-*.js
├── server.js               ← Express API (trade-api, порт 3001)
├── bot_expeditor.js        ← старый бот (отключён 2026-04-27, оставлен как референс)
├── bot_prompt_expeditor.txt
├── google-service-account.json (для будущей Google Sheets интеграции)
├── .env                    ← переменные окружения
├── package.json
└── node_modules/
```

## Процессы PM2

```
trade-api     порт 3001     server.js (Express API)
(будет) eva   PM2-процесс   Ева
```

После 2026-04-27 `trade-bot` остановлен и сохранён в дампе PM2 как stopped (не поднимется автоматически после ребута).

## Nginx

- **Конфиг**: `/etc/nginx/sites-enabled/<имя>` — раздаёт `/var/www/trade/public/` как статику
- **HTTPS**: Let's Encrypt через Certbot, автообновление настроено
- **Прокси**: `/api/*` → `http://localhost:3001`
- **Редирект 80 → 443**

## PostgreSQL

- БД: `trade_db` / Пользователь: `trade_user` / Хост: `localhost`
- Пароль — в локальном `~/secrets-trade/credentials.md`
- Структура: одна таблица `app_storage`. См. [40-system/database/overview.md](../40-system/database/overview.md)

## Переменные окружения

`.env` на VPS содержит:
- `TELEGRAM_BOT_TOKEN`
- `ANTHROPIC_API_KEY` (резерв)
- `OPENAI_API_KEY` (мало используется)
- `JWT_SECRET` (не активно)
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (для Bedrock)
- `SPREADSHEET_ID` (для Google Sheets, приостановлено)
- `NOTIFY_CHAT_IDS` (Максим, Нелли)
- `PORT=3001`, БД-параметры

⚠️ **Реальные значения** — не в репо. В локальном `~/secrets-trade/credentials.md`.

## Бэкапы

(не описано) — пробел, см. [GAP в gaps.md](../00-meta/gaps.md).

## Мониторинг

(не описано) — будет в `monitoring.md`.

## Что менять под Еву

- Добавить процесс `eva` в PM2
- Возможно — изменить Nginx (если webhook вместо polling)
- Настроить алерты (биллинг AWS, доступность сервиса)
- Регулярные бэкапы БД
