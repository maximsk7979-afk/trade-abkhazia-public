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
| `GET /catalog` | активные группы товаров (кнопки меню «Заказ») |
| `GET /home` | баланс + активные заказы + лента движения денег + менеджер (для WhatsApp) |
| `GET /trips` | открытые рейсы: `kind=banana` (по умолч., + остаток мест) / `kind=weighted` (не-банановые с отсечкой, без лимита; опц. `projectId`) |
| `GET /weighted-skus` | заказные весовые SKU для `tripId`+`group` (цена/кг задана, активен, проект подходит) — GAP-036 |
| `GET /price` | цена: банан по объёму/фикс (+апсейл) или `kind=weighted&skuId=` — ₽/кг × теор.вес упаковок |
| `POST /order` | создать/изменить заявку ЗР: банан (по лимиту) или `kind=weighted` (по упаковкам, сумма ориентировочная) |
| `DELETE /order/:id` | отмена до отсечки |
| `GET /balance`, `/history`, `/shipment/:id` | баланс, история, детализация (только своё) |

## Состав интерфейса (клиент)

- **🏠 Главная** (грузится сразу): приветствие + **WhatsApp-иконка** (чат с менеджером по `staff.phone`); **Баланс** со знаком (`−` = должен нам, `+` = предоплата); **Активные заказы** (дата доставки · позиции · ящики · статус) → модалка; **Движение по балансу** (отгрузки + оплаты/возвраты/брак). Нижний бар: **🛒 Заказ · 📦 История** (без «Главной» — она авто; возврат через `tg.BackButton`).
- **🛒 Заказ** — меню **групп** (из `/catalog`, группа видна если есть активный SKU): «Бананы» → банановый поток (по ящикам, точная цена); прочие группы → **весовой поток** (GAP-036).
- **📦 История** — все заказы со статусами (**как в системе**, без переименований).
- **Модалка заказа:** №, дата доставки, способ, статус, позиции (имя · кол-во · цена за ящик/мешок/кг), ориентировочная сумма (для весовых — точная после поставки).

## Весовые позиции (GAP-036)

Помидоры, огурцы и пр. (`SKU.tp=0`) продаются **по весу** (₽/кг), но клиент заказывает в **упаковках** (ящики/коробки/мешки). Теоретический вес система берёт из справочника SKU: нетто одной упаковки = `gw − tw`. Сумма заявки — **ориентировочная** (`estTotal = теор.вес × цена/кг`); точная станет известна после отгрузки по факт-весу (`shipmentTotal`, проставляет экспедитор/менеджер в trade_app).

- **Поток (корзина, GAP-035):** группа (напр. Помидоры) → открытые весовые рейсы проекта (`/trips?kind=weighted`) → список позиций (`/weighted-skus`: товар · упаковка · нетто · ₽/кг) с **qty-степпером на каждую строку** → сводка «N позиций · ~сумма» (считается на клиенте, заморозка — на сервере) → оформить/сохранить/отменить. При правке корзина **префиллится** из заявки (`/shipment/:id` → `lines`).
- **Цена/кг** задаёт владелец напрямую по SKU (решение 2026-06-06; **не** из себестоимости — она неизвестна до закупки рейса, и **не** из прайс-уровня). Хранилище — KV `trade-weighted-pricing-v1` `{ prices:{skuId:₽/кг}, deliverySurchargePerKg }`. Владелец управляет через Еву (инструмент `manage_weighted_pricing`). SKU без цены — **не заказной**. Доставка — общая надбавка/кг (опц.). Фикс-цена/кг клиента (`client.weightedFixedPrices[skuId]`) перебивает (хук есть, UI — GAP-039).
- **Модель ЗР — корзина:** `miniOrder` v2 `{kind:"weighted",version:2,deliveryMode,lines:[{skuId,prodId,qty,nettoPerPkg,estKg,unitPrice,lineTotal,priceSource}],total,estKg,estimated}`; `items[]` = строки (для отгрузки в trade_app). Замораживается оценка каждой строки; **без лимита вместимости** рейса. Нормализатор `order-model.js` даёт единую форму строк для легаси-одиночного и корзины (home/history/shipment/notify не ветвятся).
- **Банан** остаётся одиночным потоком (банановый рейс = один товар). **Тарные не-банановые** позиции (картошка в мешках) пока не заказные — нет механизма цены ([GAP-040](../00-meta/gaps.md#gap-040)).

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
