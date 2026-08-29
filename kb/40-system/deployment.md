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

- `code/trade_app_v3.jsx` + `code/trade_app_calc.mjs` из репозитория (мастер-копия)
- вместе с ними — три общих ядра `code/trade-api/*.cjs` (ADR-025), которые vite вбандливает
- Vite собирает всё это в статический бандл на VPS
- Nginx раздаёт бандл на https://trade-abkhazia.com

## Поток

```
[Станция: /home/max/trade_app_repo]        [На VPS]
code/trade_app_v3.jsx      ──┐
code/trade_app_calc.mjs      ├─ scp ──►  /var/www/trade/frontend/src/
code/trade-api/*.cjs (×3)  ──┘             ├── App.jsx
   (trip-cost, purchase-cost, cash)        ├── calc.mjs
                                           └── trade-api/*.cjs
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

Всю цепочку выполняет `./scripts/deploy.sh trade-app` (тесты → бэкап → scp → build →
проверка бандла). Руками файлы не копируем: без ядер фронт считает по старой логике.

## Чем деплоится остальное

Тот же скрипт, другие цели — `eva`, `trade-api`, `mini-app`, `retail-app`, `all`
(`all` = trade-api + eva + trade-app + mini-app; розничный прототип выкатывается отдельно).

## Что НЕ деплоится автоматически

- Конфиг Nginx — вручную
- БД-миграции — нет (пока через seed-блоки)
- Фото-файлы — раздаются Nginx из `/var/www/trade/photos/`, отдельно не деплоятся

## Требования инфраструктуры под фото-инфраструктуру ([ADR-016](../90-decisions/ADR-016-photo-infrastructure.md))

При первичной настройке (Блок 1 GAP-018) понадобится:

1. **Папка хранения фото:** `/var/www/trade/photos/` (создать с правами на запись для пользователя trade-api / Евы).
2. **Nginx-конфиг:** добавить `location /photos/ { alias /var/www/trade/photos/; expires 1y; add_header Cache-Control "public, max-age=31536000, immutable"; }` в server-блок `trade-abkhazia.com`.
3. **`.env` trade-api:** опционально — `PHOTOS_ROOT=/var/www/trade/photos` и `PHOTOS_PUBLIC_BASE=https://trade-abkhazia.com/photos` (если отличаются от дефолтов в `photos-router.js`). Auth не требуется (см. [ADR-016](../90-decisions/ADR-016-photo-infrastructure.md), решение Блока 1).
4. **`.env` Евы:** `STORAGE_API_URL=http://localhost:3001/api/storage` уже есть; `/api/photos` берётся из того же базового URL.
5. **Cron для чистки `/tmp/eva/`:** `find /tmp/eva -mtime +1 -delete` раз в сутки + `find /tmp/api-uploads -mtime +1 -delete`.

Детали по слоям — [photo-infrastructure.md](photo-infrastructure.md).

## Куда развивать дальше

Деплой идёт **из репозитория** (`scripts/deploy.sh`, с 2026-06-11) — прежняя схема
«мастер-копия на Маке → scp» отменена. Следующий шаг, когда появится потребность —
CI/CD через GitHub Actions вместо запуска скрипта с рабочего места.
