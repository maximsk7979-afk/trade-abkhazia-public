---
title: Архитектура текущей системы
status: active
last_updated: 2026-04-28
references:
  - frontend.md
  - api.md
  - database/overview.md
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
| API | Express 5, 4 эндпоинта, 50 строк кода | [api.md](api.md) |
| БД | PostgreSQL, 1 таблица KV-store | [database/overview.md](database/overview.md) |
| Бот (отключён) | `bot_expeditor.js`, замещается Евой | (архив) |
| Eva | В проектировании | [`50-agents/eva/`](../50-agents/eva/) |

## Потоки данных

- **Frontend ↔ API**: только через 3 эндпоинта `/api/storage/:key` (GET/POST/DELETE). Фронт читает/пишет JSON-блобы под именованными ключами.
- **Бот (когда был активен) ↔ API**: ходил туда же. Бот ↔ Bedrock — отдельный SDK-канал.
- **Eva ↔ API**: будет ходить туда же или, возможно, через специализированный API (проектируется).

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
- [25-integrations/](../25-integrations/) — внешние сервисы
