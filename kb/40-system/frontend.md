---
title: Frontend
status: active
last_updated: 2026-04-28
references:
  - api.md
  - database/storage-keys.md
  - deployment.md
referenced_by: []
---

# Frontend

## Что это

Single-file React SPA для админки/работы с системой.

- **Исходник (мастер)**: `code/trade_app_v3.jsx` в репозитории (~9,6 тыс. строк; расчёты
  вынесены в `code/trade_app_calc.mjs` и общие ядра `code/trade-api/*.cjs`, ADR-025)
- **На VPS**: `/var/www/trade/frontend/src/App.jsx` (копия) → `npx vite build` → `/var/www/trade/public/`
- **Стек**: React 18.3 + Vite 5.4, без Redux/MobX, чисто `useState/useEffect`
- **Хранилище**: PostgreSQL `app_storage` через `/api/storage/:key`
- **URL**: https://trade-abkhazia.com

## Модули

Реализованы:
- 💰 Продажи (ПС-XXX)
- 🚚 Доставки (ДС-XXX / Д-XXX)
- 🚛 Рейсы (Р-XXX)
- 📦 Склад (FIFO-партии)
- 💰 Прайс-лист
- 💳 Взаиморасчёты
- 📋 Заказы
- ⚙️ Справочники (товары, SKU, клиенты, поставщики, контрагенты, типы расходов, сотрудники, офисы)

## Ключевые константы

См. в коде (`trade_app_v3.jsx`):

```javascript
SK_SALES = "trade-sales-v1"
SK_DLV   = "trade-deliveries-v1"
SK_T     = "trade-trips-v8"
SK_BATCH = "trade-batches-v1"
SK_WH    = "trade-whops-v1"
SK_SETT  = "trade-settlements-v2"
SK_O     = "trade-orders-v4"
SK_PL    = "trade-pricelist-v1"
SK_PROD  = "trade-cat-products"
SK_SKU   = "trade-cat-skus"
SK_CLI   = "trade-cat-clients"
SK_SUP   = "trade-cat-suppliers"
SK_CONT  = "trade-cat-contragents"
SK_EXP   = "trade-cat-exptypes"
SK_STAFF = "trade-cat-staff"
SK_OFF   = "trade-cat-offices"
```

См. [database/storage-keys.md](database/storage-keys.md).

## Особенности кода

- Все функции компонентов — обычные функции, не классы. React-хуки.
- Числа хранятся **как строки** (`"qty":"5"`) — парсятся через `pN(s) = parseFloat(s) || 0`.
- ID сущностей — кириллические префиксы (`ПС-087`, `Р-001`).
- Seed-блоки (`_SD3` ... `_SD8`) — внутри JSX, обновляют БД при загрузке. Текущая версия — v45 (`_SD8`).

## Деплой

См. [deployment.md](deployment.md).

## Связанные документы

- [data-formats/](data-formats/) — JSON-формат каждой сущности
- [api.md](api.md) — как фронт общается с API
