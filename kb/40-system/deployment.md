---
title: Deployment Frontend
status: active
last_updated: 2026-04-28
references:
  - frontend.md
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

## Что нужно менять под Еву и git-flow

- Сейчас деплой делается **из мастер-копии на Маке** через `scp`. После git-репозитория правильно — деплой **из репозитория** через CI/CD или хотя бы git pull на VPS.
- Это задача в [roadmap](../65-roadmap/current.md).
