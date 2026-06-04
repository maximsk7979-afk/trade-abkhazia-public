---
title: API
status: active
last_updated: 2026-06-04
references:
  - database/overview.md
  - frontend.md
  - photo-infrastructure.md
  - ../70-operations/infrastructure.md
referenced_by: []
---

# API

## Стек

- **Express 5.2** на Node.js
- Зависимости: `cors`, `dotenv`, `pg`
- Файл: `/var/www/trade/server.js` — ~57 строк кода
- Порт: `3001` (внутренний, проксируется через Nginx)
- PM2 процесс: `trade-api`

## Эндпоинты

```
GET    /api/storage/:key     → { value: "<json-string>" } | null
POST   /api/storage/:key     → { ok: true }   body: { value: "<json-string>" }
DELETE /api/storage/:key     → { ok: true }
GET    /api/health           → { status: "ok" }

POST   /api/photos           → { ok, url, photoMeta, duplicate }     multipart/form-data
DELETE /api/photos           → { ok }                                JSON body
```

Раздел про `/api/photos` — добавлен 2026-05-30 в составе фото-инфраструктуры ([ADR-016](../90-decisions/ADR-016-photo-infrastructure.md)). Полный контракт — [photo-infrastructure.md, слой (в)](photo-infrastructure.md#слой-в--общий-контракт-endpoint-загрузки). Реализация — в Блоке 1 GAP-018.

Бизнес-логики на бэкенде минимум: `/api/storage/*` — прокси к KV-таблице `app_storage`; `/api/photos` — валидация + сохранение файла на диск + обновление массива `photoField` на бизнес-сущности.

## Авторизация

- `/api/storage/*`, `/api/health`, `/api/photos` — **нет авторизации**. CORS `*`. JWT_SECRET и `bcryptjs` лежат в `.env` и зависимостях, но не используются.

⚠️ **Это блокер для запуска Евы с клиентами и контрагентами.** Задача "Реализовать ролевой доступ (RBAC)" — в [roadmap](../65-roadmap/current.md). После RBAC оба endpoint-а одинаково перейдут на JWT-проверку пользовательской сессии.

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

## ⚠️ Обязательный формат POST: только `{ value: "<json-string>" }`

`trade-api` **валидирует payload** (с 2026-06-04, [GAP-019](../00-meta/gaps.md#gap-019) closed): если `value` — не строка (нет поля / `null` / объект / число), сервер отвечает `400 { error: "missing or invalid 'value' field (expected string)" }` и **в БД ничего не пишет**. Раньше такой запрос молча записывал `null` и **стирал существующее значение ключа** при 200 OK — отсюда инцидент 2026-05-27. Для удаления ключа используйте `DELETE`, не POST с пустым телом.

**Корректно (фронт и Ева — оба так делают):**
```js
fetch("/api/storage/trade-cat-clients", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ value: JSON.stringify(clientsArray) })
});
```

**Неправильно (теперь → 400, ключ цел):**
```js
// голый массив без обёртки { value: ... }
body: JSON.stringify(clientsArray)
// сервер ответит 400, KV[key] НЕ изменится (раньше — 200 OK и null)
```

**Прецедент:** 2026-05-27 — Ева писала без обёртки `{value}`, перетёрла справочник клиентов (33 записи утеряны). Фикс на стороне клиента — `code/eva/src/storage-api.js:34` (коммит после инцидента); первопричина на сервере закрыта 2026-06-04 валидацией выше. См. CHANGELOG 2026-05-27 и 2026-06-04, [GAP-019](../00-meta/gaps.md#gap-019).

**Правило для CC при имплементации любых клиентов storage-api:** перед коммитом — выполнить round-trip smoke-тест (POST с известным значением → GET → сравнить с исходным). Если не совпало — не деплоить.

## Что менять под Еву

Существующий API годится для прямой работы с `app_storage`, но Еве понадобится:
1. **Авторизация** (роли, токены, ограничения по ключам)
2. **Атомарные операции** уровня сущности (например, "добавить продажу", не "перепиши весь массив продаж")
3. **Эндпоинты для агента** — отдельные операции для типичных сценариев (создать клиента, провести продажу, и т.д.)
4. **Логирование** действий

Решение по архитектуре API под Еву — отдельная задача в [roadmap](../65-roadmap/current.md).

## `/api/photos` — загрузка и удаление фото

Единая точка записи фото для Евы и trade_app. Multipart, валидация всех обязательных полей (`base`, `entityId`, `uploadedBy`, `fileUniqueId`, `file`), сохранение в `/var/www/trade/photos/<base>/<entityId>/<unixTs>_<fileUniqueId>.<ext>`, обновление массива `photoField` на бизнес-сущности через KV.

Полный контракт (поля, валидация, коды ошибок, дедуп по `fileUniqueId`, конфиг `PHOTO_BASES`) — [photo-infrastructure.md, слой (в)](photo-infrastructure.md#слой-в--общий-контракт-endpoint-загрузки). Решение — [ADR-016](../90-decisions/ADR-016-photo-infrastructure.md). Код — `code/trade-api/photos-router.js`.

Контракт **строго валидирует payload** (в отличие от `/api/storage/:key` — урок [GAP-019](../00-meta/gaps.md#gap-019)): отсутствие обязательных полей → `400 BAD_PAYLOAD`, не молча. Авторизации нет, как и у `/api/storage` — после RBAC оба endpoint-а перейдут на JWT.

Nginx публикует папку `/var/www/trade/photos/` по URL `https://trade-abkhazia.com/photos/...` с `Cache-Control: public, max-age=31536000, immutable`.

## Связанные документы

- [database/overview.md](database/overview.md)
- [photo-infrastructure.md](photo-infrastructure.md) — фото-инфраструктура (5 слоёв)
- [70-operations/infrastructure.md](../70-operations/infrastructure.md)
