---
title: Deployment Frontend
status: active
last_updated: 2026-05-30
references:
  - frontend.md
  - photo-infrastructure.md
  - ../70-operations/runbooks/deploy-frontend.md
referenced_by: []
---

# Деплой фронтенда

Этот документ описывает **что** деплоится. Пошаговая инструкция деплоя — в [70-operations/runbooks/deploy-frontend.md](../70-operations/runbooks/deploy-frontend.md).

## Что деплоится

- Файл `~/Documents/trade_app/trade_app_v3.jsx` (мастер-копия)
- Vite собирает его в статический бандл
- Nginx раздаёт бандл на https://trade-abkhazia.com

## Поток

```
[Локально на Маке]                        [На VPS]
trade_app_v3.jsx  ──── scp ────►   /var/www/trade/frontend/src/App.jsx
                                          │
                                          │ npx vite build (на VPS)
                                          ▼
                                   /var/www/trade/public/
                                   ├── index.html
                                   └── assets/index-*.js
                                          │
                                          │ Nginx раздаёт
                                          ▼
                                   https://trade-abkhazia.com
```

## Что НЕ деплоится автоматически

- API (`server.js`) — вручную, при изменении (редко)
- Конфиг Nginx — вручную
- БД-миграции — нет (пока через seed-блоки)
- Сервис Евы (`/var/www/eva/`) — вручную через `scp` + `pm2 restart eva`
- Фото-файлы — раздаются Nginx из `/var/www/trade/photos/`, отдельно не деплоятся

## Требования инфраструктуры под фото-инфраструктуру ([ADR-016](../90-decisions/ADR-016-photo-infrastructure.md))

При первичной настройке (Блок 1 GAP-018) понадобится:

1. **Папка хранения фото:** `/var/www/trade/photos/` (создать с правами на запись для пользователя trade-api / Евы).
2. **Nginx-конфиг:** добавить `location /photos/ { alias /var/www/trade/photos/; expires 1y; add_header Cache-Control "public, max-age=31536000, immutable"; }` в server-блок `trade-abkhazia.com`.
3. **`.env` trade-api:** добавить `PHOTO_TOKEN=<32 hex>` (сгенерировать одной командой при реализации, симметрично положить в `.env` Евы и зафиксировать в `~/secrets-trade/credentials.md`).
4. **`.env` Евы:** `PHOTO_TOKEN=<тот же>`, `TRADE_API_URL=https://trade-abkhazia.com` (или внутренний URL).
5. **Cron для чистки `/tmp/eva/`:** `find /tmp/eva -mtime +1 -delete` раз в сутки.

Детали по слоям — [photo-infrastructure.md](photo-infrastructure.md).

## Что нужно менять под Еву и git-flow

- Сейчас деплой делается **из мастер-копии на Маке** через `scp`. После git-репозитория правильно — деплой **из репозитория** через CI/CD или хотя бы git pull на VPS.
- Это задача в [roadmap](../65-roadmap/current.md).
