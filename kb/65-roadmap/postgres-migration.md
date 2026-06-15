---
title: Дорожная карта — завершение переезда на Postgres (ADR-024)
status: active
last_updated: 2026-06-15
related: [ADR-024, ADR-025]
---

# Дорожная карта: завершение переезда KV → Postgres (ADR-024)

> Рабочий план на несколько сессий. Решение и принцип — [ADR-024](../90-decisions/ADR-024-kv-to-postgres-migration.md).
> Принцип: **по одной сущности за раз, самое горячее/опасное — раньше**; между шагами система полностью рабочая (гибрид «часть в таблицах, часть массивом»).

## 1. Где мы сейчас (факт на 2026-06-15)

«Массив в ячейке» = весь список лежит одним JSON-текстом в `app_storage` (ключ → значение).
«Таблица» = нормальная таблица с колонками-индексами (выжаты из документа) + `doc JSONB` + `updated_at`; чтение массива остаётся совместимым, запись — точечная.

**Уже в таблице (готово):**
- `trade-sales-v1` (продажи) — M1, 2026-06-11.
- `trade-settlements-v2` (платежи/операции) — 2026-06-14.
- **Шаг A — справочники (2026-06-15):** `trade-cat-clients`, `trade-cat-suppliers`, `trade-cat-products`, `trade-cat-skus`, `trade-cat-staff`, `trade-cat-exptypes`, `trade-cat-contragents`, `trade-cat-projects`, `trade-cat-offices`, `trade-cat-segments` — одна фабрика `cat-store.js`, своя таблица на ключ (`id PK` + `name` + `doc JSONB`). exptypes идентифицируется полем `key` (idField), остальные — `id`.
- **Шаг B — рейсы `trade-trips-v8` (2026-06-15):** `trips-store.js` (колонки-индексы `id PK`, `project_id`, `status`, `arr_date`, `date`, `office_id`, `procurement_cycle_id` + `doc JSONB`). Бэкфилл 2/2 в прод. Писатели переведены: trade-api (4 мутации статусов/вместимости → `tripsStore.upsert(trip)`; мёртвый «ленивый» write удалён), фронт (`saveT` + импорт → диф-батч `/api/v2/trips/batch`), Ева (`createBananaTrip`/`updateTripCapacity`/`complete_trip` → `storage.tripUpsert` через `/api/cat/upsert`).
- **Шаг C1 — доставки `trade-deliveries-v1` + закупки `trade-purchases-v1` (2026-06-15):** `deliveries-store.js` (колонки `status`, `date`, `office_id`) и `purchases-store.js` (колонки `project_id`, `date`, `status`, `office_id`). Бэкфилл 1/1 + 1/1 в прод. Писатели: trade-api (5 мутаций доставок → `deliveriesStore.upsert`; закупок серверных нет), фронт (`saveDlv`/`savePurchases` + импорт → диф-батч `/api/v2/{deliveries,purchases}/batch`), Ева — только читает.
- **Шаг C2.1 — партии `trade-batches-v1` + складские операции `trade-whops-v1` (2026-06-15):** `batches-store.js` (колонки `sku_id`, `qty_rem` NUMERIC, `trip_id`, `purchase_id`, `office_id`) и `whops-store.js` (колонки `op_type`, `date`, `delivery_id`, `sku_id`). Бэкфилл 3/3 + 1/1 в прод. Писатели: trade-api (приёмка рейса/возврата/recost/погрузка `depart`+`load` → `batchesStore`/`whopsStore` через `diffById(prev,next)` + локальный хелпер `diffById`), фронт (`saveBatches`/`saveWh` + импорт → диф-батч `/api/v2/{batches,whops}/batch`). **sale-direct** пока считает FIFO в браузере и пишет результат диф-батчем (промежуточный гибрид — серверный перенос в C2.3). Запись массивом → 410. Тесты 14+14. **C2.2–C2.5 (allocations/storno/атомарная транзакция) — ещё впереди.**

`STORES` в `server.js` держит sales + settlements + рейсы + доставки + закупки + партии + whops + 10 справочников; `MIGRATED_KEYS` выводится из `STORES`.

**Осталось перевести (по числу мест-ссылок в коде — грубая «горячесть»):**

| Ключ | Сущность | ~ссылок | Куда отнесён |
|------|----------|---------|--------------|
| `trade-weighted-pricing-v1` | цены/кг весовых | 1 | Шаг D |
| `trade-banana-pricing-v1` | банановое ценообразование | 1 | Шаг D |
| `trade-pricelist-v1` (SK_PL) | прайс-лист | — | Шаг D |
| `trade-banana-cycles` | циклы закупки (Ева) | — | Шаг D |
| `trade-banana-reprice-v1` | задачи переоценки (Ева) | — | Шаг D |

Константы фронта `SK_*` — `trade_app_v3.jsx:57-74`.

## 2. Повторяемый рецепт переезда одной сущности (проверен на sales и settlements)

Каждый шаг ниже выполняется по этому рецепту. Образец кода — `code/trade-api/sales-store.js` + `settlements-store.js` и их обвязка в `server.js`.

1. **Модуль `<entity>-store.js`** (зеркалит `sales-store.js`):
   - таблица: `id PK` + колонки-индексы, выжатые из документа на записи, + `doc JSONB` + `updated_at`;
   - методы: `ensureSchema`, `getAll`, `upsert(doc, client?)`, `remove(id, client?)`, `batch(items, client?)`, `migrateFromKV`, `validateDoc`;
   - `validateDoc` — **мягкий** (id — непустая строка, doc — объект, разумный размер). Строгие regex запрещены: формы id разные, строгая проверка = риск потери записи при бэкфилле;
   - `KEY = "<ключ>"`, маркер бэкфилла `_migrated:<ключ>`.
2. **Регистрация в `server.js`:** добавить ключ в `STORES` → `kvGet` отдаёт `getAll()`, `kvSet`/POST `/api/storage` → `throw`/410; на старте `store.ensureSchema().then(migrateFromKV)`. `GET /api/storage/:key` для мигрированного отдаёт `{value: JSON.stringify(getAll())}` (совместимое чтение).
3. **Точечные писатели:** перевести КАЖДОГО писателя этого ключа с записи всего массива на point-op/диф:
   - **trade-api:** `kvSet(key, arr)` → `store.upsert/remove` (одиночный) или `store.batch` (много). Generic `/api/cat/{upsert,append,remove}` уже идут через `kvUpsertOne`/`kvRemoveOne` → для мигрированного ключа работают автоматически.
   - **trade_app (фронт):** заменить запись всего массива (`window.storage.set` через `mkSave`/`saveCat`) на диф-батч `POST /api/v2/<entity>/batch` (через `CALC.diffSales` — он generic по id), сохранив сигнатуру «отдаю массив» + откат стейта при ошибке (образец `salesBatchWrite` / `settlementsBatchWrite`). Точечные правки справочников можно слать через `catPost` (`/api/cat/*`).
   - **Ева:** заменить `storage.set(key, arr)` на HTTP point-op к trade-api (X-Internal-Token), либо убрать копию и звать общий эндпоинт.
4. **Эндпоинт `POST /api/v2/<entity>/batch`** (если фронт пишет массивом) — диф по id (как у sales/settlements).
5. **Тесты `<entity>-store.test.js`** (plain node+assert, без живого pg): `validateDoc`, выжимка колонок из doc, форма `batch`. Подключить в `npm test`.
6. **Деплой по рецепту:** бэкап KV-ключа (`<ключ>.bak-<дата>-<entity>-migration`) → деплой trade-api → бэкфилл на старте (маркер) → деплой фронта/Евы. Порядок: **сначала trade-api, потом потребители**.
7. **Смоук на проде** + grep, что писателей всего массива не осталось → ключ «заморожен».

## 3. Очередь шагов (от простого/безопасного к сложному)

### Шаг A — Справочники ✅ ГОТОВО (2026-06-15)
**Ключи:** `trade-cat-clients`, `trade-cat-suppliers`, `trade-cat-products`, `trade-cat-skus`, `trade-cat-staff`, `trade-cat-exptypes`, `trade-cat-contragents`, `trade-cat-projects`, `trade-cat-offices`, `trade-cat-segments`.

**Сделано:** обобщённая фабрика `cat-store.js` (idField настраиваемый: exptypes→`key`, остальные→`id`); регистрация в `STORES`; бэкфилл на старте по маркеру. Писатели переведены на point-op/диф: фронт `saveCat`/`importData` → диф-батч `POST /api/v2/cat/batch` (через `CALC.diffById(arr, arr, idField)`); Ева — `storage.catUpsert(key, item)` → `/api/cat/upsert` (онбординг-привязка `index.js`, `set-client-banana-price.js`); клиенты/поставщики уже были на `catPost`. Тесты: `cat-store.test.js` (idField, name из label, форма batch), `diffById` в calc.test.mjs. Бэкап KV: `*.bak-20260615-cat-migration`. Прод-смоук: add/edit/delete через batch (id- и key-keyed), point-op upsert/remove, 410 на запись массива — пройдено. Прецедент потери клиентов (GAP-019/BUG-007) закрыт.

**Почему первым:** низкая частота записи, низкий риск; правки уже идут через `/api/cat/*` (point-op), которые уважают `STORES` — после регистрации стора многое заработает само. Закрывает «прецедент потери клиента».

**Особенность:** один **обобщённый фабричный стор** (`cat-store.js`: `id PK` + колонка `name` + `doc JSONB`) на все ключи-справочники — не плодить 10 почти одинаковых файлов. Колонки-индексы минимальны (по id + имя).

**Писатели:**
- trade-api: `/api/cat/{upsert,append,remove}` → уже через `kvUpsertOne`/`kvRemoveOne` (готово после регистрации).
- **фронт `saveCat` (`trade_app_v3.jsx:841`) пишет ВЕСЬ массив через `window.storage.set`** — это надо перевести: либо `saveCat` → диф-батч на `/api/cat/*`-эндпоинт, либо каждый обработчик на point-op `catPost` (часть уже так: клиенты/поставщики на `catPost("/upsert")` в 6837/6852). Это основная работа шага.
- Ева: создаёт клиентов/сотрудников через trade-api эндпоинты (проверить, что не пишет массивом напрямую).

**Риск/гейт:** ссылочная целостность при удалении уже есть (Фаза 2 аудита) — сохранить. Импорт-восстановление (`exportData`/import) пишет справочники массивом — учесть.

**Смоук:** добавить/править/удалить в каждом справочнике из админки; создание клиента Евой; импорт-восстановление.

### Шаг B — Рейсы `trade-trips-v8` ✅ ГОТОВО (2026-06-15)
**Почему отдельно и осторожно:** пишут ВСЕ поверхности + Ева + оркестратор закупки.

**Сделано:**
- `trips-store.js` по рецепту (фабрика как sales/settlements): колонки `id PK`, `project_id`, `status`, `arr_date`, `date`, `office_id`, `procurement_cycle_id` + `doc JSONB`; `validateDoc` мягкий (`/^Р-\d+/`, кириллическая Р); `getAll/upsert/remove/batch/migrateFromKV` (бэкфилл идемпотентный по маркеру). Зарегистрирован в `STORES`.
- **trade-api `server.js`:** мёртвый «ленивый» write (`tripsChanged=false`) удалён; 4 живые мутации (`buyer/load`, `wh/accept`, `exp/save`, `buyer/complete-trip` — все внутри `withSaleLock`, мутируют один `trip`) → `tripsStore.upsert(trip)`. Новые ручки `GET /api/v2/trips`, `POST /api/v2/trips/batch` (диф под `withSaleLock`).
- **фронт `trade_app_v3.jsx`:** `tripsBatchWrite` (диф `/api/v2/trips/batch`); `saveT` и импорт-восстановление переведены на диф (целый массив в `/api/storage` → 410).
- **Ева:** `storage.tripUpsert(trip)` (через тот же `/api/cat/upsert`, маршрут по ключу в стор). `effect-exec.createBananaTrip` (создание точечным upsert), `updateTripCapacity`, `complete-trip.js` — переведены. id рейса по-прежнему считается `nextTripId(getAll)` (низкая конкуренция; апсёрт по id атомарен).

**Особенность (выполнено):** ленивое авто-создание/дедуп по дате — теперь атомарный `INSERT ... ON CONFLICT (id)`; колонка `procurement_cycle_id` хранит связь цикл→рейс.

**Гейт пройден:** оркестратор закупки (`trade-banana-cycles`) читает рейсы через `storage.get` (совместимое чтение из таблицы) и находит по `tripId` — связь цикл→рейс цела. Мьютексы не вкладывались (point-op атомарен сам).

**Смоук в прод (2026-06-15):** бэкфилл 2/2; 410 на массив; Ева-путь — round-trip вместимости Р-002 (100→777→100); фронт-путь — create+delete Р-900 через батч; финальное состояние = исходное (без мусора). Все Eva/trade-api/calc тесты зелёные (добавлен `test/trips-store.test.js`).

### Шаг C1 — Доставки `trade-deliveries-v1` + закупки `trade-purchases-v1` ✅ ГОТОВО (2026-06-15)
**Почему отдельно от C2:** это простые ключи «как A/B» (свои стораы, диф-батч). Их переезд снимает 2 из 4 сущностей с пути к атомарной транзакции C2, оставляя последней только FIFO-связку `batches`↔`whops` (где живёт GAP-057). Система остаётся в гибриде: доставки/закупки в таблицах, батчи/whops пока в KV — каждый ключ пишется отдельно (как и до миграции), атомарности не теряется.

**Сделано:**
- `deliveries-store.js` (колонки `id PK`, `status`, `date`, `office_id` + `doc JSONB`; `validateDoc` мягкий `/^Д-\d+/`) и `purchases-store.js` (колонки `id PK`, `project_id`, `date`, `status`, `office_id` + `doc JSONB`; `/^ЗК-\d+/`). Оба по рецепту: `getAll/upsert/remove/batch/migrateFromKV` (бэкфилл идемпотентный по маркеру). Зарегистрированы в `STORES`.
- **trade-api `server.js`:** 5 серверных записей доставок (`wh/accept-return`, новая доставка, добавление заявки, `dlv/complete`, `dlv/load` — все внутри `withSaleLock`, мутируют один `dlv`) → `deliveriesStore.upsert(...)`. Закупок серверных писателей нет (ключ только читается в сводке). Новые ручки `GET/POST /api/v2/deliveries(/batch)` (диф под `withSaleLock`), `GET/POST /api/v2/purchases(/batch)` (диф под `withCatLock`).
- **фронт `trade_app_v3.jsx`:** `deliveriesBatchWrite`/`purchasesBatchWrite`; `saveDlv`, `savePurchases` и импорт-восстановление переведены на диф (целый массив в `/api/storage` → 410).
- **Ева:** только читает доставки/закупки (`get-business-data`, `get-delivery-summary`) через совместимое чтение из таблицы — изменений кода не требуется.

**Гибрид (осознанно до C2):** `dlv/load` пишет doc доставки точечно, а батчи/whops — пока `kvSet` в KV. Истинная атомарность всех трёх таблиц одной транзакцией — цель C2.

**Смоук в прод (2026-06-15):** бэкфилл 1/1 (доставки) + 1/1 (закупки); 410 на массив для обоих ключей; совместимое чтение `/api/storage/...` из таблицы; create+delete round-trip `Д-99001`/`ЗК-99001` через батч; финальное состояние = исходное (без мусора). Все trade-api/calc тесты зелёные (добавлены `test/deliveries-store.test.js`, `test/purchases-store.test.js`). KV-бэкап обоих ключей: `/root/kv-backups/*.20260615-011041.json`.

### Шаг C2 — `trade-batches-v1` + `trade-whops-v1` → Postgres + атомарная погрузка + storno (GAP-057) — ДИЗАЙН (2026-06-15, ждёт ok Максима)

**Почему это не «ещё две таблицы как C1».** Разведка кода (2026-06-15) показала: батчи/whops пишутся **с двух сторон**, и часть FIFO-логики живёт в браузере:

*Серверные писатели (trade-api, поверхность мини-аппа/экспедитор):*
- `POST /api/wh/accept` (`buildBatches`) — приёмка рейса кладовщиком: создаёт партии (`server.js:1080`).
- `POST /api/wh/accept-return` (`acceptReturn`) — приёмка возврата: партии + whops receipt/writeoff (`1131/1132`).
- `POST /api/wh/recost` (`recostBatches`) — пересчёт себестоимости партий рейса (`1187`, меняет только `costPerUnit`).
- `POST /api/dlv/load` (`fifo-batch` allocate/deduct) — погрузка: FIFO-вычет + whops shipment + доставка→«В пути» (`1858–1897`). Уже считает `allAllocs` и кладёт их на доставку как `_fifoAllocs`.

*Фронтовые писатели (trade_app_v3.jsx, поверхность owner/админка) — пишут МАССИВ + клиентский FIFO:*
- `changeSt` рейса → «На складе» (`1726 saveBatches`) — создаёт партии (дубль серверного `buildBatches`!).
- `savePurchase`/`deleteP` (`2386/2398`) — партии закупки вне рейса (source `purchase`).
- **создание sale-direct** (`4152 fifoAllocate` → `4154 fifoDeduct` → `4157 saveBatches`, `4176 saveWh`) — клиентский FIFO-вычет.
- `completeDelivery` (`4982 fifoAddBatch` → `4984/4985`) — возврат непроданного: партии + whops receipt.
- WarehouseMod ручные операции (`3315/3318/3324 saveWh`), `refreshFromSales` (`3297`).
- импорт-восстановление (`987/999 window.storage.set`).

*Клиентский FIFO (`trade_app_calc.mjs` через `CALC.*`):* `fifoStock/fifoAllocate/fifoDeduct/fifoReturn/fifoAddBatch` — параллельная реализация серверного `fifo-batch.js` + копии в `delivery-service.js`. **Тройное дублирование доменной логики — прямое нарушение ADR-025.**

*GAP-057 подтверждён в коде:* `deleteDlv` (`4786`), `delSale` для sale-direct (`4225`, удаляет whops, но **не возвращает `qtyRem`**), `deleteP` (`2398`), `delOp` (`3324`), backward `changeSt` (`1669` — явный комментарий «партии не откатываются») — **ни один не делает компенсирующий возврат в FIFO**. `saveMultiple` (`795`) — мёртвый код (0 вызовов).

**Ключевое архитектурное решение (по ADR-025): FIFO становится серверным и единственным.** Нельзя просто «обернуть массив в стор» — пока sale-direct делает FIFO в браузере и пишет целый массив, после 410 он сломается. Поэтому переезд батчей/whops **обязывает** перенести клиентские FIFO-писатели на серверные транзакционные эндпоинты (как уже сделано для погрузки в `/api/dlv/load`). Это и закрывает дубль FIFO, и даёт настоящую атомарность.

#### Предлагаемое разбиение C2 на под-шаги (система рабочая между каждым)

- **C2.1 ✅ DONE (2026-06-15)** — стораы + чтение + простые/серверные писатели. `batches-store.js` (колонки `sku_id`, `qty_rem` NUMERIC, `trip_id`, `purchase_id`, `office_id`; PK `id` `B-NNNN`), `whops-store.js` (колонки `op_type`, `date`, `delivery_id`, `sku_id`; PK `id` `WH-*`). Зарегистрированы в `STORES`, бэкфилл 3/3 + 1/1 по маркеру. Серверные писатели (`wh/accept`/`accept-return`/`exp/save`-recost/`dlv/depart`/`dlv/load`) переведены на стораы через локальный `diffById(prev,next)` — каждый отдельным `batch`/`upsert` (без общей транзакции — гибрид как C1). Фронт: `saveBatches`/`saveWh` и импорт → диф-батч `/api/v2/{batches,whops}/batch`. Запись массивом → 410. Тесты 14+14. Бандл `index-E0-vyxIa.js`.
  - **Тонкость (остаётся):** sale-direct (клиентский FIFO + saveBatches/saveWh) пишет результат диф-батчем — корректно, но FIFO ещё в браузере. Полный перенос — C2.3.
- **C2.2 — модель аллокаций whop (фундамент storno).** В whops shipment добавить `allocations: [{batchId, qty, costPerUnit}]` при создании (в `/api/dlv/load` данные уже есть — `allAllocs`, привязать поштучно к whop, а не агрегатом на доставку). Бэкфилл существующих whops best-effort из `_fifoAllocs` доставки. Колонка-индекс `batch_id` не нужна (allocations в doc); сверка склада — [GAP-042](../00-meta/gaps.md#gap-042).
- **C2.3 — серверные атомарные писатели sale-direct (закрытие дубля FIFO, ADR-025).** Новые эндпоинты `POST /api/v2/sale-direct/{create,update,delete}` (или общий), выполняющие allocate/deduct/return + запись batches+whops+sales **в одной транзакции БД** через стораы на общем `client`. Фронт `SalesMod` зовёт их вместо клиентского FIFO. Клиентские `CALC.fifo*`-вызовы-писатели удаляются (остаётся только `fifoStock` для отображения — чтение).
- **C2.4 — storno при отмене (ядро GAP-057).** `deleteDlv`/`delSale`/`deleteP`/backward-`changeSt` → серверные эндпоинты, делающие **видимый компенсирующий возврат** (`qtyRem += qty` по сохранённым `allocations`, storno-whop типа `storno`/`reversal`) в одной транзакции; **fail-closed**: тихое удаление отгрузки с FIFO-списанием запрещено без storno. Роли (GAP-057): инициирует sales_manager/expeditor/warehouse, **решает owner/gen_director**, исполняет система, видит owner/ГД/fin/warehouse в «Движении по складу». (NB: trade_app сейчас без per-action гейта — GAP-048; storno-операцию и видимость делаем сразу, ролевой гейт — по мере закрытия GAP-048.)
- **C2.5 — настоящая транзакция погрузки/приёмки.** `/api/dlv/load`, `/api/wh/accept`, `/api/wh/accept-return` пишут все затронутые таблицы (batches+whops+deliveries+sales) одной `BEGIN…COMMIT` через стораы на общем `client` вместо нынешних раздельных `upsert`/`kvSet`. Сбой → откат БД, а не только UI.

**Колонки-индексы:** батчи — `sku_id`, `qty_rem` (FIFO-выборка), `trip_id`/`purchase_id` (источник); whops — `op_type`, `date`, `delivery_id`. (доставки/закупки — сделаны в C1.)

**Риски/гейты:** (1) клиентский FIFO ≠ серверный по нюансам (sale-direct edit делает `fifoReturn` старых аллокаций — серверный аналог нужен) — сверять построчно; (2) дубль приёмки рейса (фронтовый `changeSt`→batches vs серверный `/api/wh/accept`) — определить единый путь, иначе двойные партии; (3) [GAP-042](../00-meta/gaps.md#gap-042) сверка «движение↔партии»; (4) бэкфилл whops без allocations — storno для исторических whops по эвристике/ручной сверке.

**Смоук (на каждом под-шаге):** полный цикл «приёмка рейса → партия → погрузка (FIFO-вычет) → отгрузка → завершение → приёмка возврата»; **отдельно для C2.4**: создать sale-direct → удалить → `qtyRem` вернулся, storno-whop виден; **для C2.5**: искусственный сбой в середине погрузки → откат БД (ничего не записано).

**Объём:** реалистично несколько сессий. Рекомендация — заходить под-шагами C2.1 → C2.5, деплой+смоук после каждого, чекпоинт с Максимом между ними.

### Шаг D — Финал: ценообразование, циклы Евы, заморозка `/api/storage`
**Ключи:** `trade-pricelist-v1`, `trade-weighted-pricing-v1`, `trade-banana-pricing-v1`, `trade-banana-cycles`, `trade-banana-reprice-v1`.

**Особенность Евы:** `cycle-store.js` и `reprice-store.js` пишут весь массив через `storage.set(KEY, ...)` — перевести на point-op эндпоинты trade-api (или на свои стораы, если решим оставить отдельным хранилищем Евы — но по ADR-025 предпочтительно единый источник).

**Завершение ADR-024:**
- перевести `POST/DELETE /api/storage` в **read-only legacy** (все записи — через стораы/эндпоинты);
- убрать мёртвые пути записи всего массива; grep'ом подтвердить, что `kvSet`/`window.storage.set` массивом нигде не остался;
- ADR-024 → закрыт.

## 4. Сквозные риски (помнить на каждом шаге)
- **Мягкий `validateDoc`** — иначе бэкфилл теряет записи с нестандартным id.
- **Не вкладывать мьютексы** (`withSaleLock`/`withCatLock` не реентерабельны); point-op через стор атомарен сам.
- **Старый бот `bot_expeditor_v10.js`** мог писать массивы — подтвердить, что не запущен на VPS, перед каждым шагом, где он касался ключа.
- **Импорт-восстановление** (`exportData`/import в админке) пишет МНОГО ключей массивом — после миграции каждого ключа этот путь надо переводить на батч/эндпоинты, иначе восстановление упрётся в 410.
- **Порядок деплоя:** trade-api (эндпоинты живут) → потом фронт/Ева (потребители).
- **Бэкап KV-ключа и прод-исходников** перед каждым деплоем; смоук после каждого.

## 5. Готово, когда (Definition of Done ADR-024)
- Все ключи из таблицы §1 — в Postgres-таблицах; `STORES` содержит их все.
- `POST/DELETE /api/storage` — read-only legacy; запись массивом в коде отсутствует (grep чист).
- «Погрузка» атомарна на уровне БД.
- Ни одной копии доменной логики поверх KV (ADR-025); Ева/мини-апп/админка — интерфейсы к одной системе.

## 6. После переезда — единый калькулятор для ВСЕХ видов платежей и контрагентов
> Согласовано с Максимом 2026-06-14. Делать **после** завершения миграции (нужен чистый фундамент-таблицы). Это шаг 4 [GAP-058](../00-meta/gaps.md#gap-058).

Сейчас единый источник есть только для **баланса клиента** (`balance-service`: `clientSummary`/`clientsRoster`). Остальные агрегации в админке (`getSupplierSummary`, `getExpenseSummary`, `getPartnerData` и пр.) считаются отдельно и не сведены.

**Задача:** распространить канонический калькулятор `balance-service` на все типы расчётов — поставщики, расходы, партнёры, контрагенты-перевозчики — чтобы каждая поверхность (админка/мини-апп/Ева) звала один общий расчёт (ADR-025: «вызывать, а не копировать»). После переезда все исходные сущности (settlements, sales, purchases, batches) уже в таблицах — расчёт строится на чистом фундаменте без массивов-куч.

Объём: вынести формулы баланса/оборота поставщика, расходов по типам, баланса партнёра/контрагента в `balance-service` (или родственный сервисный модуль), добавить эндпоинты по аналогии с `/api/client(s)/summary`, перевести админку/Еву на их вызов, удалить локальные копии. Тесты — как у клиентского баланса.

## Темп
~1–2 сессии на шаг (C — возможно 2). Система рабочая между шагами. Каждый шаг = отдельный деплой по рецепту §2. Единый калькулятор (§6) — отдельный шаг после миграции.
