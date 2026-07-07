---
title: Архитектура аутентификации (все поверхности)
status: active
last_updated: 2026-07-07
references:
  - role-catalog.md
  - role-permissions.md
  - ../90-decisions/ADR-022-role-model-rbac.md
  - ../00-meta/gaps.md
---

# Архитектура аутентификации

Четыре механизма — по одному на тип клиента. Реализация веб-входа — [GAP-078](../00-meta/gaps.md#gap-078) (2026-07-07).

| Клиент | Механизм | Код |
|---|---|---|
| **Веб-TradeApp (браузер)** | Логин-пароль → HMAC-подписанный токен сессии в httpOnly-cookie | `trade-api/web-auth.js` + `webGate` в `server.js` |
| **Мини-апп (Telegram)** | Подписанный Telegram `initData` (HMAC по bot-токену), резолв ролей из `trade-cat-staff`/`trade-cat-clients` | `trade-api/mini-auth.js` |
| **Ева / серверные скрипты** | `X-Internal-Token` == `INTERNAL_API_TOKEN` (одинаков в .env trade-api и Евы; реально включён 2026-07-07) | `isInternalCaller` в `server.js`; Ева — `storage-api.js internalHeaders()` |
| **Фото-загрузка** | статический `X-Photo-Token` (отсечка сканеров, НЕ auth: токен в публичном бандле) | `photos-router.js` |

## Веб-вход (GAP-078)

**Хранение учёток.** Приватный singleton-док `app_web_auth` (Postgres):
`{ accounts: [{ login, staffId, passHash, active }], demo: { passHash } }`.
Сознательно **вне** реестра `SINGLETONS` server.js — не читается через `GET /api/storage`,
не пишется через `/api/v2/doc`, хэши не стираются диф-записью справочника. Логины/хэши
**не в staff-карточках**: `trade-cat-staff` читается любым залогиненным. Роли и активность
staff-учётки резолвятся в момент запроса из карточки по `staffId` (деактивация карточки
или учётки гасит вход). `demo` — не staff: фиксированные `roles:['demo']`, `readOnly`.

**Криптография (без внешних зависимостей).** Пароль — `crypto.scrypt` (N=16384, соль 16Б);
токен — `v1.<b64url payload>.<b64url HMAC-SHA256>` по `JWT_SECRET` из .env; сравнение
timing-safe. В payload: `sub` (логин), `exp` (staff 30 суток, demo 24 часа), `pwv` —
отпечаток текущего хэша пароля: **смена пароля мгновенно инвалидирует все выданные
токены** учётки (требование «менять demo после показов»).

**Cookie.** `ta_session`, `HttpOnly; Secure; SameSite=Lax; Path=/`, **без Domain**
(host-only) — одинаково работает на `trade-abkhazia.com` и на RF-зеркале
`maxim.prfo.design` (relative `API_URL`, same-origin). SameSite=Lax = защита от CSRF-записи.

**Гейт `webGate` (fail-closed).** Стоит на всех веб-префиксах API: `/api/storage`, `/api/v2`,
`/api/cat`, `/api/fin`, `/api/dlv`, `/api/cash`, `/api/pnl`, `/api/turnover`,
`/api/client(s)`, `/api/supplier(s)`, `/api/contragent(s)`. Порядок: internal-токен →
пропуск; валидная cookie → `req.webUser`, для `demo` любой не-GET/HEAD → `403 demoReadOnly`;
иначе `401 authRequired`. Прежний **пропуск по Origin домена удалён**. `/api/mini`,
`/api/photos`, `/api/health`, `/api/auth` гейтом не трогаются. Актор `/api/fin/op` для
веба берётся из сессии (спуфинг из тела закрыт).

**Эндпоинты `/api/auth/*`:** `login` (rate-limit 10/мин/IP), `logout`, `me`,
`set-credentials` и `accounts` — только `owner` (Обязательство 7: учётки заводит/блокирует
и пароль demo меняет owner). UI — «Справочники → Сотрудники»: колонка «Веб-вход»,
кнопка «🔑 Пароль demo».

**Уровни доступа веба (старт, 2026-07-07):** две ступени — сотрудник с учёткой = полный
доступ (учётки выдаёт только owner; сейчас: Максим С-001, Нелли С-002), `demo` = только
чтение. Гранулярная ролевая матрица веба — остаток [GAP-048](../00-meta/gaps.md#gap-048),
детализируется в [role-permissions.md](role-permissions.md) по мере надобности.

**Приёмка.** `code/ops/gap078-acceptance.mjs` — автопрогон: аноним → 401 на всех read/write;
demo → чтение 200, все 25 write-эндпоинтов 403; owner — полный цикл; mini/health не задеты.
Гонять после любых правок auth-слоя (боевой, безопасен: пишет только тест-сегмент и удаляет).

**Bootstrap.** Первичное заведение учёток — `gap078-seed-accounts.js` на VPS (пишет док
напрямую в Postgres; set-credentials требует owner-сессию, которой до seed не существует).
Пароли — `~/secrets-trade/credentials.md`.
