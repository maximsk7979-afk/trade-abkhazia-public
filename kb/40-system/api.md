---
title: API
status: active
last_updated: 2026-04-28
references:
  - database/overview.md
  - frontend.md
  - ../70-operations/infrastructure.md
referenced_by: []
---

# API

## Стек

- **Express 5.2** на Node.js
- Зависимости: `cors`, `dotenv`, `pg`
- Файл: `/var/www/trade/server.js` — **50 строк** кода
- Порт: `3001` (внутренний, проксируется через Nginx)
- PM2 процесс: `trade-api`

## Эндпоинты

```
GET    /api/storage/:key     → { value: "<json-string>" } | null
POST   /api/storage/:key     → { ok: true }   body: { value: "<json-string>" }
DELETE /api/storage/:key     → { ok: true }
GET    /api/health           → { status: "ok" }
```

Это **весь** API. Никакой бизнес-логики на бэкенде нет — он просто прокси к KV-таблице `app_storage`.

## Авторизация

**Нет.** CORS `*`. JWT_SECRET и `bcryptjs` лежат в `.env` и зависимостях, но не используются.

⚠️ **Это блокер для запуска Евы с клиентами и контрагентами.** Задача "Реализовать ролевой доступ (RBAC)" — в [roadmap](../65-roadmap/current.md).

## Что делает GET / POST / DELETE

```javascript
// Получить значение по ключу
GET /api/storage/trade-sales-v1
→ { value: "[{\"id\":\"ПС-087\",...}, ...]" }   // или null если ключа нет

// Записать значение
POST /api/storage/trade-sales-v1
body: { value: "[{...},{...}]" }
→ { ok: true }

// Удалить ключ
DELETE /api/storage/trade-sales-v1
→ { ok: true }
```

POST использует `INSERT ... ON CONFLICT (key) DO UPDATE` — атомарная замена всего значения. Частичных обновлений нет — это причина race condition при одновременной записи разных компонентов.

## Что менять под Еву

Существующий API годится для прямой работы с `app_storage`, но Еве понадобится:
1. **Авторизация** (роли, токены, ограничения по ключам)
2. **Атомарные операции** уровня сущности (например, "добавить продажу", не "перепиши весь массив продаж")
3. **Эндпоинты для агента** — отдельные операции для типичных сценариев (создать клиента, провести продажу, и т.д.)
4. **Логирование** действий

Решение по архитектуре API под Еву — отдельная задача в [roadmap](../65-roadmap/current.md).

## Связанные документы

- [database/overview.md](database/overview.md)
- [70-operations/infrastructure.md](../70-operations/infrastructure.md)
