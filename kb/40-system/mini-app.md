---
title: Telegram Mini App клиента — архитектура и эксплуатация
status: active
last_updated: 2026-06-06
references:
  - ../90-decisions/ADR-019-eva-mini-app.md
  - ../90-decisions/ADR-020-order-intake-banana-pricing.md
  - ../50-agents/eva/scenarios/order-intake.md
referenced_by: []
---

# Telegram Mini App (мини-апп клиента)

> Поверхность структурных действий клиента/менеджера — [ADR-019](../90-decisions/ADR-019-eva-mini-app.md). Здесь — **как это устроено и эксплуатируется**. Реализован и развёрнут 2026-06-04.

## Где живёт

| Что | Путь |
|---|---|
| Исходники | `code/mini-app/` (`index.html` shell, `app.js`, `app.css`) |
| На VPS | `/var/www/trade/public/mini/` |
| URL | `https://trade-abkhazia.com/mini/` |
| Бэкенд | эндпоинты `/api/mini/*` в `trade-api` (`code/trade-api/server.js`) |
| Вход | menu-кнопка бота «🍌 Заказ» (ставит Ева при старте, `code/eva/src/index.js`) |

## Идентификация и изоляция (GAP-030)

Личность — из подписанного Telegram `initData` (заголовок `X-Telegram-Init-Data`), проверка HMAC на сервере (`code/trade-api/mini-auth.js`), резолв роли (owner/manager/client/unknown) из KV. Сервер фильтрует данные по личности. **Клиент видит только своё.** ⚠️ Тонкость HMAC: исключается только поле `hash`, **`signature` остаётся** в строке проверки (иначе «подпись не сходится»).

## Автообновление (всегда последняя версия) — ВАЖНО

Telegram **агрессивно кэширует мини-аппы по URL** (`no-store` и force-quit не всегда помогают). Решение — **тонкий shell + версионирование по хэшу**:

1. **`index.html`** — крошечный неизменный shell. Telegram его кэширует, но он **не меняется**.
2. Shell зовёт **`GET /api/mini/version`** (без auth, `no-store`) → хэш содержимого `app.js`+`app.css`.
3. Shell грузит **`app.js?v=<хэш>`** и **`app.css?v=<хэш>`** — при изменении кода хэш меняется → новый URL → свежая загрузка.

**Следствие:** при обычном деплое фронта (`app.js`/`app.css`) обновление подхватывается **само** — не нужно бампать версию и не нужно переоткрывать чат. Достаточно закрыть/открыть мини-апп.

**`?v=N` в URL menu-кнопки** (`MINI_APP_URL` в `eva/src/index.js`) бампается **только если меняется сам shell** (`index.html`) — это редко. Текущее: `?v=3`. При смене shell: бампнуть N в коде Евы → restart eva (или прямой `setChatMenuButton`), пользователю один раз переоткрыть чат.

**Nginx:** `location /mini/` отдаёт с `Cache-Control: no-store` (на VPS, `/etc/nginx/sites-enabled/trade`). ⚠️ Бэкапы конфигов Nginx **не класть в `sites-enabled/`** — Nginx грузит все файлы оттуда (будут конфликтующие server-блоки).

## menu-кнопка (точка входа)

Ставится прямым вызовом Bot API (`axios.post .../setChatMenuButton`, JSON-тело) при старте Евы. ⚠️ **`bot.setChatMenuButton` из `node-telegram-bot-api@0.66` НЕ работает** (сериализует `menu_button` как form-поля → Telegram оставляет `type:commands`). Только прямой JSON.

## Эндпоинты `/api/mini/*` (trade-api)

| Эндпоинт | Назначение |
|---|---|
| `GET /me` | роль + имя по `initData` |
| `GET /clients` | клиенты сотрудника (менеджер — свои, owner — все) |
| `GET /version` | хэш версии фронта (для автообновления; без auth) |
| `GET /order-days` | **(ADR-021)** дни заказа: завтра .. +4, с отсечкой (D−1 20:00 МСК) и `open` |
| `GET /day-catalog?date=&clientId=` | **(ADR-021)** единый каталог на день: активные SKU ∩ проекты клиента, секциями Овощи→Фрукты→Бананы→Прочее, с ценой (банан tiers / весовой ₽/кг) + существующая заявка |
| `GET /home` | баланс + активные заказы + лента движения денег + менеджер (для WhatsApp) |
| `POST /order` | создать/изменить заявку ЗР. **(ADR-021)** `{date, delivery, lines:[{skuId,qty}]}` — смешанная корзина, тип строки по SKU, ленивое авто-создание рейса. (Легаси: `tripId`+`kind` банан/weighted — ещё поддерживаются.) |
| `DELETE /order/:id` | отмена до отсечки |
| `GET /balance`, `/history`, `/shipment/:id` | баланс, история, детализация (только своё; `shipment` отдаёт строки `lines[]`) |
| `GET /catalog`, `/trips`, `/weighted-skus`, `/price` | легаси-эндпоинты прежних потоков (банан/весовой по рейсу) — оставлены, фронтом не используются |

## Состав интерфейса (клиент)

- **🏠 Главная** (грузится сразу): приветствие + **WhatsApp-иконка** (чат с менеджером по `staff.phone`); **Баланс** со знаком (`−` = должен нам, `+` = предоплата); **Активные заказы** (дата доставки · позиции · ящики · статус) → модалка; **Движение по балансу** (отгрузки + оплаты/возвраты/брак). Нижний бар: **🛒 Заказ · 📦 История** (без «Главной» — она авто; возврат через `tg.BackButton`).
- **🛒 Заказ (ADR-021)** — экран **«На какой день?»** (4 кнопки: завтра/+2/+3/+4 с датами; прошедшая отсечка → неактивна) → **единый каталог** на день: активные SKU ∩ проекты клиента, секциями **Овощи → Фрукты → Бананы → Прочее**, qty-степпер на строку (банан в ящиках volume-tier, овощи в упаковках ₽/кг), **смешанная корзина**, сводка «N позиций · ~сумма», оформить/сохранить/отменить.
- **📦 История** — все заказы со статусами (**как в системе**, без переименований).
- **Модалка заказа:** №, дата доставки, способ, статус, позиции (имя · кол-во · цена за ящик/мешок/кг), ориентировочная сумма (для весовых — точная после поставки).

## Заказ на день + смешанные рейсы (ADR-021)

Ежедневный рейс Грузия→Абхазия — **смешанная машина** (бананы + турецкие/грузинские овощи). Клиент выбирает **день доставки**, ассортимент — **единый каталог по своим проектам** ([ADR-021](../90-decisions/ADR-021-daily-mixed-trips-day-order.md)).

- **Рейс** = дата + отсечка (`orderCutoffAt` = D−1 20:00 МСК), **без лимита вместимости**, `projectId:""`+`mixed:true`+`auto:true`. Создаётся **лениво** при первом заказе на дату (`nextTripId`). В админке trade_app пустой `projectId` коалесцируется в `DEFAULT_PRJ` — не ломает (полный учёт смешанного рейса — [GAP-041](../00-meta/gaps.md#gap-041)).
- **Каталог** (`/day-catalog`): активные SKU, у которых `applicableProjects ∩ проекты клиента ≠ ∅`; секции **Овощи → Фрукты → Бананы → Прочее**. Клиент только с PRJ-004 видит лишь бананы. `applicableProjects` — **фильтр ассортимента**, не учётная привязка (один товар бывает из Турции/Грузии — проект строки присваивается при закупке).
- **Цена в каталоге:** банан — `volume-tier` (по сумме банановых ящиков, ADR-020), заказ в **ящиках**; весовые (`tp=0`) — **₽/кг** (владелец задаёт по SKU, KV `trade-weighted-pricing-v1`, инструмент Евы `manage_weighted_pricing`), заказ в **упаковках**, теор.вес = `qty×(gw−tw)`, сумма **ориентировочная** (точная — после взвешивания при отгрузке). Фикс-цена клиента: банан `client.bananaFixedPrice`, весовой `client.weightedFixedPrices[skuId]` (хук; UI — [GAP-039](../00-meta/gaps.md#gap-039)). Тарные не-банановые (`tp=1`) — без механизма цены, в каталог не попадают ([GAP-040](../00-meta/gaps.md#gap-040)).
- **Модель ЗР — смешанная корзина:** `miniOrder` v2 `{kind:"mixed"|"weighted"|"banana", version:2, deliveryMode, lines:[{kind,skuId,prodId,qty,nettoPerPkg?,estKg?,unitPrice,lineTotal,priceSource}], total, estKg, estimated}`; `items[]` = строки. Цена замораживается. Нормализатор **`order-model.js`** (`orderLines/orderFrozenTotal/orderUnits/orderPositions/orderEstKg/orderKind`) даёт единую форму строк для легаси-одиночного и корзины v2 — home/history/shipment/notify не ветвятся.
- **Одна заявка на клиента на день**, правка/отмена до отсечки; при правке корзина префиллится (`/day-catalog` → `existing.lines`).

## Активность SKU (что показывать клиенту)

В справочнике SKU (trade_app) у каждого SKU — тумблер `active` (Справочники → 📦 SKU → «Активен»). Мини-апп/каталог берёт только `active !== false` (отсутствие поля = активен). Гейтит **клиентский заказ и новые формы**; история/старые данные не трогаются. Группа-кнопка появляется, если в ней есть ≥1 активный SKU.

## Деплой

- **Фронт мини-аппа:** `scp code/mini-app/{index.html,app.js,app.css} → /var/www/trade/public/mini/`. Версия подхватится сама (хэш). Shell менять — редко (тогда бампнуть `?v=N`).
- **`trade-api`:** `scp code/trade-api/*.js → /var/www/trade/` + `pm2 restart trade-api`. `.env` содержит `TELEGRAM_BOT_TOKEN`, `NOTIFY_CHAT_IDS`.
- **trade_app (админка):** `scp code/trade_app_v3.jsx → /var/www/trade/frontend/src/App.jsx` + `ssh "cd /var/www/trade/frontend && npx vite build"` → `public/` ([runbook](../70-operations/runbooks/deploy-frontend.md)).
- **Ева:** `scp code/eva/src/... → /var/www/eva/src/` + `pm2 restart eva`.

## Связанные документы

- [ADR-019](../90-decisions/ADR-019-eva-mini-app.md) — мини-апп как поверхность · [ADR-020](../90-decisions/ADR-020-order-intake-banana-pricing.md) — заявки/цена
- [order-intake (сценарий Евы)](../50-agents/eva/scenarios/order-intake.md) · [order-intake (процесс)](../30-roles/to-be/sales-manager/processes/order-intake.md)
- [deployment.md](deployment.md) · [deploy-frontend runbook](../70-operations/runbooks/deploy-frontend.md)
