---
title: Архитектура текущей системы
status: active
last_updated: 2026-05-30
references:
  - frontend.md
  - api.md
  - database/overview.md
  - photo-infrastructure.md
  - ../70-operations/infrastructure.md
referenced_by: []
---

# Архитектура текущей системы

## Высокоуровневая схема

```
┌──────────────────┐         ┌────────────────────┐
│ Browser (admin)  │         │ Telegram clients   │
│ trade-abkhazia.  │         │ (будущее: Eva)     │
│ com              │         │                    │
└────────┬─────────┘         └─────────┬──────────┘
         │ HTTPS                        │ Long-poll / Webhook
         ▼                              ▼
   ┌──────────┐                  ┌────────────────┐
   │  Nginx   │                  │  Eva (PM2)     │
   │  :443    │                  │ (в проектиров.)│
   └────┬─────┘                  └────────┬───────┘
        │                                 │ Bedrock SDK
   ┌────┴────┐                            ▼
   │         │                     ┌──────────────────┐
   ▼         ▼                     │ Claude Sonnet 4.6│
 static/   /api/* → :3001          │ via AWS Bedrock  │
                ┌──────────────┐   └──────────────────┘
                │  trade-api   │
                │  server.js   │
                │  Express 5   │
                └──────┬───────┘
                       │ pg
                       ▼
                ┌─────────────┐
                │ PostgreSQL  │
                │ trade_db    │
                │ → 1 таблица │
                │  app_storage│
                └─────────────┘
```

## Компоненты

| Компонент | Описание | Файл |
|---|---|---|
| Frontend | Single-file React SPA, ~15 326 строк | [frontend.md](frontend.md) |
| API | Express 5, 6 эндпоинтов (`/api/storage/*` + `/api/photos`) | [api.md](api.md) |
| БД | PostgreSQL, 1 таблица KV-store | [database/overview.md](database/overview.md) |
| Файлы фото | `/var/www/trade/photos/<base>/<entityId>/...`, Nginx public | [photo-infrastructure.md](photo-infrastructure.md) |
| Бот (отключён) | `bot_expeditor.js`, замещается Евой | (архив) |
| Eva | Рантайм в проде (онбординг клиента); фото-модуль в реализации | [`50-agents/eva/`](../50-agents/eva/) |

## Потоки данных

- **Frontend ↔ API**: через `/api/storage/:key` (GET/POST/DELETE) для JSON-блобов и `/api/photos` (POST/DELETE) для загрузки фото. Фронт читает фото напрямую по `<img src="https://trade-abkhazia.com/photos/...">` (Nginx public).
- **Бот (когда был активен) ↔ API**: ходил туда же. Бот ↔ Bedrock — отдельный SDK-канал.
- **Eva ↔ API**: пишет через `/api/storage/*` (карточки, продажи и т.д.) и через `/api/photos` (фото). Фото из Telegram качаются в `/tmp/eva/...`, затем уезжают в endpoint trade-api. См. [photo-infrastructure.md](photo-infrastructure.md).

## Принципиальные особенности (и что менять под Еву)

### Что хорошо
- Простота: всё на одной VPS, два PM2-процесса (был ещё бот).
- Single source of truth для фронта: бизнес-логика в одном файле.
- Доменная модель в JSON: добавить поле — это правка JSX, без миграций БД.

### Что плохо (техдолг)
- **Нет авторизации** — CORS `*`, любой может писать в БД. Блокер для Евы с клиентами.
- **Нет git** локально — это **исправлено развёртыванием 28.04**.
- **`app_storage` как KV-store**: race condition при одновременной записи бота и фронта.
- **Single-file 15К строк** — на пределе.
- **Сериализация чисел как строк** (`"qty":"5"`) — историческое.

См. [65-roadmap](../65-roadmap/current.md) — задачи по техдолгу.

## Связанные документы

- [70-operations/infrastructure.md](../70-operations/infrastructure.md) — где это запущено
- [photo-infrastructure.md](photo-infrastructure.md) — фото-инфраструктура (5 слоёв, 4 базы фото)
- [25-integrations/](../25-integrations/) — внешние сервисы
