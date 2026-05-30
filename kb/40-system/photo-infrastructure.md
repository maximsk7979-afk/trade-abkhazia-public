---
title: Фото-инфраструктура — архитектура
status: active
last_updated: 2026-05-30
references:
  - ../00-meta/gaps.md
  - ../65-roadmap/current.md
  - ../90-decisions/ADR-014-client-project-profile.md
  - ../90-decisions/ADR-010-invoice-entity.md
  - ../90-decisions/ADR-013-eva-agent-architecture.md
---

# Фото-инфраструктура — архитектура

Общий пайплайн загрузки, хранения, привязки и просмотра фотографий в системе Trade Abkhazia. Четыре независимые базы фотографий ([GAP-018](../00-meta/gaps.md#gap-018), [GAP-020](../00-meta/gaps.md#gap-020), [GAP-021](../00-meta/gaps.md#gap-021), [GAP-022](../00-meta/gaps.md#gap-022)) разделяют общий технический фундамент. Архитектура должна позволять без переделки фундамента добавлять новые базы фото (пункт 5 разбора 2026-05-30).

> **Статус:** active. Архитектура зафиксирована в сессии 2026-05-30 по слоям (а)-(д). Ключевые решения вынесены в [ADR-016](../90-decisions/ADR-016-photo-infrastructure.md). Этот документ — детальная техническая спецификация и эталон для реализации.

## Слои

| Слой | Что | Статус |
|---|---|---|
| (а) Приём фото из Telegram | Распознать тип сообщения, скачать, нормализовать в `RawPhoto` | ✅ зафиксировано 2026-05-30 |
| (б) Хранение на VPS | Структура папок, конвенция имён, доступ через Nginx, метаданные | ✅ зафиксировано 2026-05-30 |
| (в) Общий контракт endpoint загрузки | API для записи фото из Евы и из trade_app | ✅ зафиксировано 2026-05-30 |
| (г) Диспетчер контекста | Решение «к какой базе/сущности привязать»; интеграция с активным сценарием Евы | ✅ зафиксировано 2026-05-30 |
| (д) Просмотр в trade_app + конвенция URL | UI галерей в карточках, формат URL, удаление/replace | ✅ зафиксировано 2026-05-30 |

---

## Слой (а) — Приём фото из Telegram

**Статус:** ✅ зафиксировано 2026-05-30.

Задача слоя — собрать «сырьё» из Telegram-update в нормализованный `RawPhoto` и отдать диспетчеру контекста (слой «г»). На этом слое решения «куда привязать» не принимаются.

### Принимаемые типы сообщений

- **`msg.photo: [...]`** — сжатое фото из галереи/камеры. Берём **последний элемент** массива (самый большой размер).
- **`msg.document`** c `mime_type ~ "image/*"` — фото, присланное «как файл», без сжатия. **Принимаем тоже.** В подсказках менеджерам/экспедиторам — отгрузочные накладные просить присылать как файл (точнее распознавание).
- **`msg.caption`** — подпись к фото; передаём в диспетчер контекста как сырьё.
- **`msg.media_group_id`** — альбом; **буферизуем** (см. ниже).
- **Видео / voice / прочее** — вежливый отказ: «Я умею принимать только фото и изображения».

### Альбомы (`media_group_id`)

Telegram отправляет каждое фото альбома отдельным `update`, без явного «последнее». Реализация: ловим первый `update` с `media_group_id`, ставим таймер **~700 мс**, добавляем все приходящие с тем же id в буфер, по таймауту выпускаем пакетом в диспетчер.

Нужно для: GAP-018 (несколько фото витрины), GAP-022 (несколько фото товаров с одной закупки), потенциально GAP-020 (многостраничная накладная).

### Скачивание

Скачиваем **сразу при приёме** (асинхронно, не блокируя обработку) в `/tmp/eva/<chatId>/<fileUniqueId>.<ext>`. Cron чистит `/tmp/eva/` старше **24 часов** на случай прерванных диалогов.

Это снимает риск истечения `file_id`, пока менеджер раздумывает или Ева задаёт уточняющие вопросы. Финальное перемещение в `/var/www/trade/photos/<база>/...` — на слое (б), после распознавания контекста.

### Идемпотентность

Дедуп по `file_unique_id` в рамках активной сессии диалога. Повторно присланный файл → «Это фото уже принято».

### Лимиты

Bot API `getFile` ограничен **20 МБ** на файл. При ошибке → «Фото слишком большое, пришлите меньше или как сжатое фото».

### Идентификация отправителя

- `chat_id` находится через `whoami_staff` (есть в `trade-cat-staff`) → `senderRole = "staff"`.
- `chat_id` находится в `client.tgChatId` → `senderRole = "client"`.
- Иначе → `senderRole = "unknown"` → ответ «Сервис в тестовом режиме», файл **не сохраняем**.

**Сейчас (2026-05-30) активна только ветка `staff`.** Архитектурно оставляем поле `senderRole` как точку расширения на клиентов в будущем (например, фото жалобы) — отдельный сценарий, когда появится конкретный кейс.

### Привязка к активному сценарию

- Если у `chat_id` есть **активный сценарий** Евы, ожидающий фото (например, `activeScenario = "client-onboarding"`, шаг «фото витрины») — фото идёт туда **автоматически**.
- Если активного нет — Ева спрашивает inline-кнопками: «Это фото [витрины клиента / товара / отгрузочной накладной / с закупки]?».

Хранение `activeScenario` — сейчас в in-memory сессии Евы (`Map<chatId, {...}>`); миграция в Postgres вместе с Ит.4 (`eva_messages`, слой A памяти, [ADR-013](../90-decisions/ADR-013-eva-agent-architecture.md)).

### Выходная структура `RawPhoto`

```json
{
  "senderChatId": 118206343,
  "senderRole": "staff",
  "receivedAt": "2026-05-30T12:34:56Z",
  "fileId": "AgACAgIA...",
  "fileUniqueId": "AQADXX",
  "source": "photo",
  "width": 1280,
  "height": 960,
  "size": 245678,
  "caption": "Клубника, П-004, 150 кг",
  "mediaGroupId": null,
  "localPath": "/tmp/eva/118206343/AQADXX.jpg"
}
```

После слоя (а) этот объект уходит в **диспетчер контекста** (слой «г»).

### Псевдокод

```
on telegram update:
  if msg.photo or (msg.document && mime ~ "image/*"):
    sender = identify(chat_id)  // staff | client | unknown via whoami_staff + client.tgChatId
    if sender == unknown:
      reply("Сервис в тестовом режиме"); return
    if msg.media_group_id:
      bufferAlbum(media_group_id, msg); setTimer(700ms, flushAlbum)
      return
    rawPhoto = await downloadAndNormalize(msg, sender)
    dispatch(rawPhoto)  // → слой (г)
  elif msg.video || msg.voice || ...:
    reply("Я умею принимать только фото и изображения")
  else:
    handleTextOrCallback(msg)  // существующий tool loop
```

---

## Слой (б) — Хранение на VPS

**Статус:** ✅ зафиксировано 2026-05-30.

Задача слоя — после распознавания контекста (слой «г») переместить файл из `/tmp/eva/...` в постоянное хранилище, построить URL, добавить запись о фото в массив на бизнес-сущности.

### Где хранятся файлы

**Локально на VPS** `trade-abkhazia.com`, корень — `/var/www/trade/photos/`. Та же машина, где живут сайт, trade-api, KV-Postgres и Ева. S3/CDN — преждевременная сложность; бэкап ([GAP-016](../00-meta/gaps.md#gap-016)) и архивация ([GAP-017](../00-meta/gaps.md#gap-017)) решаются отдельно.

### Структура папок

Иерархия **по базе → по родительской сущности → по времени**:

```
/var/www/trade/photos/
├── client-locations/<clientId>/<unixTs>_<fileUniqueId>.<ext>
├── shipments/<saleId>/<unixTs>_<fileUniqueId>.<ext>
├── skus/<skuId>/<unixTs>_<fileUniqueId>.<ext>
└── purchase-offers/<purchaseId|tripId>/<unixTs>_<fileUniqueId>.<ext>
```

Плюсы: легко найти все фото одной сущности (`ls client-locations/К-002/`), легко выборочно бэкапить, легко удалить при удалении сущности. Безопасность: ID сущностей по конвенции системы **не меняются** (это инвариант), так что URL стабильны.

### Конвенция имён файлов

`<unixTs>_<fileUniqueId>.<ext>`:
- `unixTs` — секунды с эпохи (короткое, сортируется лексикографически по возрастанию времени).
- `fileUniqueId` — из Telegram `file_unique_id`, гарантирует уникальность даже при одновременной загрузке.
- `ext` — берём как есть: `.jpg` для `msg.photo`, расширение из `document.file_name` или mime — для `msg.document`. **Не конвертируем.**

HEIC/HEIF: не конвертируем сейчас. В норме менеджеры жмут «Отправить фото» в Telegram → Telegram сжимает в JPEG. HEIC прилетит только если «Отправить файлом». Если выстрелит — добавим конвертацию через `sharp` в endpoint.

### Доступ к фото

**Nginx публикует `/var/www/trade/photos/` как `https://trade-abkhazia.com/photos/...`** — простые публичные URL. Защита — непредсказуемое имя файла (`<unixTs>_<fileUniqueId>`, угадать нельзя). «Security through obscurity», но для фото витрин/товаров/накладных приемлемо.

RBAC по проектам ([ADR-009](../90-decisions/ADR-009-multitenancy-projects.md) п. 9) сейчас не реализован; защищать страничный доступ опережая RBAC — преждевременно. При появлении чувствительных кейсов перейдём на signed URLs или endpoint `/api/photos/...` с auth.

### Кто пишет файлы

**trade-api** (сервер сайта `trade-abkhazia.com`) — единая точка записи. Ева отправляет фото через HTTP в endpoint trade-api, тот валидирует, сохраняет, обновляет KV. Контракт endpoint — слой (в).

Преимущества над прямой записью из Евы:
- Единая валидация (mime, размер, существование сущности-родителя).
- trade-api уже владеет KV-обновлениями (`POST /api/storage/:key`) — добавляет фото в массив одной транзакцией.
- Если потом trade_app (веб-форма) тоже захочет загружать фото из формы — тот же endpoint.
- Ева остаётся «тонким» сервисом; фото-логика не размазана.

Накладные расходы (Ева скачала с Telegram → отдала по HTTP в trade-api → trade-api записал) на одном VPS — миллисекунды.

### Атомарность перемещения

`/tmp/eva/...` и `/var/www/trade/photos/...` — на одном томе. `fs.rename` атомарен. Перед `rename` — `fs.mkdir({recursive: true})` на родительскую папку.

### Запись о фото — массив объектов в KV

Поле фото на бизнес-сущности — **массив объектов**, не голых URL. Это даёт аудит и дедуп без отдельной таблицы:

```json
"locationPhotos": [
  {
    "url": "https://trade-abkhazia.com/photos/client-locations/К-002/1748600096_AQADXX.jpg",
    "uploadedBy": "С-001",
    "uploadedAt": "2026-05-30T12:34:56Z",
    "caption": "Витрина со стороны входа",
    "fileUniqueId": "AQADXX",
    "size": 245678,
    "width": 1280,
    "height": 960
  }
]
```

Совместимость: на 2026-05-30 поле `locationPhotos[]` в ADR-014 У2 ещё нигде не заполнено — ломать нечего. Для остальных баз поля создаются сразу как массив объектов: `Sale.shipmentPhotos[]`, `Sku.photos[]`, `Purchase.purchasePhotos[]` (названия финализируются при дизайне каждой базы).

### Удаление

**Физическое удаление файла + удаление записи из массива.** Soft-delete для фото не нужен — не транзакционная сущность, история не важна, экономим место. Менеджер удалил — фото исчезло.

Endpoint — на trade-api, контракт в слое (в).

### Размеры (resize/thumbnail)

**Не делаем сейчас.** Сжатые Telegram-фото ~200-300 КБ; для мини-галереи в UI достаточно `<img>` с CSS `max-height: 80px; object-fit: cover`. Браузер скачает оригинал.

Если потом окажется, что галерея 50+ фото тормозит — добавим thumbnail-генерацию через `sharp` лениво, по запросу. Сейчас — преждевременная оптимизация.

### Маппинг базы → KV-ключ + поле сущности

| `base` | KV-ключ | Поле в сущности | ID сущности |
|---|---|---|---|
| `client-locations` | `trade-cat-clients` | `locationPhotos[]` | `clientId` (К-NNN) |
| `shipments` | `trade-trips-v8` или эквивалент Sale | `shipmentPhotos[]` | `saleId` (зависит от модели Sale) |
| `skus` | `trade-cat-skus` | `photos[]` | `skuId` (Т-NNN) |
| `purchase-offers` | `trade-purchases-v1` или `trade-trips-v8` | `purchasePhotos[]` | `purchaseId` (ЗК-NNN) или `tripId` (Т-NNN) |

Маппинг — конфигом trade-api, не хардкод по if-цепочке. Для добавления **новой базы** (пункт 5 «расширяемость»): добавить строку в конфиг + при необходимости новое поле в сущность.

## Слой (в) — Общий контракт endpoint загрузки

**Статус:** ✅ зафиксировано 2026-05-30.

Единая точка записи фото на сервере — `POST /api/photos` на trade-api. Симметричный `DELETE /api/photos` для удаления. REST-стиль (в будущем расширяемо до `GET /api/photos?base=...&entityId=...` для перечисления).

### Формат запроса

```
POST /api/photos
Content-Type: multipart/form-data
X-Photo-Token: <secret>
```

Multipart-поля:

| Поле | Тип | Обязат. | Что |
|---|---|---|---|
| `base` | string | ✅ | `client-locations` \| `shipments` \| `skus` \| `purchase-offers` |
| `entityId` | string | ✅ | ID родителя (К-NNN / S-NNN / Т-NNN / ЗК-NNN или Т-NNN) |
| `uploadedBy` | string | ✅ | `staffId` загрузившего |
| `fileUniqueId` | string | ✅ | для дедупа (из Telegram `file_unique_id`; для веб-формы — UUID, генерируемый на бэке) |
| `caption` | string | ⬜ | подпись (из caption Telegram или из диалога Евы) |
| `file` | binary | ✅ | сам файл |

### Авторизация

Shared secret в заголовке `X-Photo-Token`, сравнивается с `process.env.PHOTO_TOKEN`. Слабая защита от случайных запросов и сканеров.

`PHOTO_TOKEN` — длинная случайная строка (~32 байта hex), генерируется при реализации; кладётся в `.env` Евы и trade-api, фиксируется в `~/secrets-trade/credentials.md`.

При появлении RBAC ([roadmap Q3 2026](../65-roadmap/current.md)) переходим на JWT-проверку пользовательской сессии.

### Валидация (в порядке проверки)

1. `X-Photo-Token` совпадает → иначе `401 {error: "unauthorized", code: "UNAUTHORIZED"}`.
2. `base` ∈ enum → иначе `400 {code: "BAD_BASE"}`.
3. Все обязательные поля присутствуют → иначе `400 {code: "BAD_PAYLOAD"}`. **Эта проверка прямо реализует урок [GAP-019](../00-meta/gaps.md#gap-019)** — endpoint не молчит при отсутствии полей.
4. `file.mimeType` начинается с `image/` → иначе `415 {code: "BAD_MIME"}`.
5. `file.size` ≤ 25 МБ → иначе `413 {code: "TOO_LARGE"}`.
6. `entityId` существует в соответствующем KV-ключе → иначе `404 {code: "ENTITY_NOT_FOUND"}`.
7. Если `fileUniqueId` уже есть в массиве `photoField` сущности → **дедуп, возврат `200 {duplicate: true, url, photoMeta}`** (не ошибка).

### Ответы

```json
// 200 OK
{
  "ok": true,
  "url": "https://trade-abkhazia.com/photos/client-locations/К-002/1748600096_AQADXX.jpg",
  "photoMeta": {
    "url": "...",
    "uploadedBy": "С-001",
    "uploadedAt": "2026-05-30T12:34:56Z",
    "caption": "Витрина со стороны входа",
    "fileUniqueId": "AQADXX",
    "size": 245678,
    "width": 1280,
    "height": 960
  },
  "duplicate": false
}

// 4xx
{
  "ok": false,
  "error": "human-readable",
  "code": "BAD_BASE | BAD_PAYLOAD | BAD_MIME | TOO_LARGE | ENTITY_NOT_FOUND | UNAUTHORIZED"
}
```

### Удаление

```
DELETE /api/photos
Headers: X-Photo-Token, Content-Type: application/json
Body: { "base": "...", "entityId": "...", "fileUniqueId": "..." }
```

Поведение:
- Убирает запись из массива `photoField` сущности.
- Физически удаляет файл (`fs.unlink`).
- Если файла нет, но запись есть (или наоборот) — лог-варнинг, `200 OK` (eventual consistency).
- Если ни записи, ни файла — `404 {code: "PHOTO_NOT_FOUND"}`.

### Маппинг «база → KV-ключ + поле + finder»

Конфигом на trade-api (`server.js` или отдельный `photo-bases.js`):

```js
const PHOTO_BASES = {
  "client-locations": {
    kvKey: "trade-cat-clients",
    photoField: "locationPhotos",
    findById: (arr, id) => arr.find(c => c.id === id),
    label: "Витрина / вход клиента"
  },
  "shipments": {
    kvKey: "trade-trips-v8", // или новый ключ Sale — финализируется при дизайне GAP-020
    photoField: "shipmentPhotos",
    findById: /* зависит от модели Sale */,
    label: "Отгрузочная накладная"
  },
  "skus": {
    kvKey: "trade-cat-skus",
    photoField: "photos",
    findById: (arr, id) => arr.find(s => s.id === id),
    label: "Фото товара"
  },
  "purchase-offers": {
    kvKey: "trade-purchases-v1", // или "trade-trips-v8" — финализируется при дизайне GAP-022
    photoField: "purchasePhotos",
    findById: (arr, id) => arr.find(p => p.id === id),
    label: "Фото с закупки"
  }
};
```

**Расширяемость (пункт 5)**: добавить новую базу = добавить строку в `PHOTO_BASES`, при необходимости — поле в сущность. Сам endpoint, диспетчер контекста (слой г) и UI просмотра (слой д) не меняются.

### Извлечение `width/height`

- Для `msg.photo` — берётся из Telegram (есть в `RawPhoto`).
- Для `msg.document` — на стороне endpoint через `image-size` (легковесная либа, читает только заголовок, без полной декодировки). HEIC может не разобрать → кладём `0/0`, UI справится.

### Атомарность KV

Read-modify-write на KV-ключе через существующий API. У KV нет встроенных транзакций — конкурентная запись может затереть. Гонок не предвидится (один клиент = один менеджер за раз). При необходимости — `version` + optimistic lock; сейчас оверкилл.

### Поток в одном кадре

```
Ева (или фронт trade_app)
  ↓ POST /api/photos
  Headers: X-Photo-Token, Content-Type: multipart/form-data
  Body: {base, entityId, uploadedBy, fileUniqueId, caption?, file}
  ↓
trade-api: validate {token, base, payload, mime, size, entity-exists}
  ↓ если fileUniqueId уже в массиве → return 200 {duplicate: true, url}
  ↓ image-size(file) → {w, h}
  ↓ mkdir -p /var/www/trade/photos/<base>/<entityId>/
  ↓ fs.rename(/tmp/eva/... → final)
  ↓ build URL по конвенции
  ↓ KV update: append photoMeta to entity.photoField
  ↓
return 200 {ok, url, photoMeta, duplicate: false}
```

## Слой (г) — Диспетчер контекста

**Статус:** ✅ зафиксировано 2026-05-30.

Слой решает: получив `RawPhoto` от (а), определить `(base, entityId)` — возможно, через диалог — и передать в `POST /api/photos`.

### Архитектура: диспетчер + per-base handlers

```
RawPhoto → ДИСПЕТЧЕР → определяет base → HANDLER базы → решает entityId
                                                      → постобработка
                                                      → POST /api/photos
```

- **Диспетчер** — общий для всех баз, отвечает за «какая база».
- **Handler базы** — свой для каждой (см. карту ниже). Знает, как достать `entityId`, что делать после загрузки (для `shipments` — запустить распознавание).

Новая база (пункт 5) = новый handler + строка в `PHOTO_BASES`. Диспетчер не меняется.

### Источники сигнала «какая база» (по приоритету)

1. **Активный сценарий ждёт фото** — `session.expectingPhoto = {base, entityId}`. Например, шаг 9 онбординга ставит `expectingPhoto = {base: "client-locations", entityId: <создаваемый клиент>}`. Фото идёт без диалога.

2. **Inline-кнопки выбора** — если активного нет:

   > Это фото:
   > [🏪 Витрина клиента] [📦 Товар]
   > [📋 Отгрузочная накладная] [🛒 С закупки]

   Кнопки строятся **динамически из конфига `PHOTO_BASES`** (раздел `label`) — новая база появляется в опросе автоматически.

3. **Caption** — на старте игнорируется как сигнал для выбора базы (см. Р1), но сохраняется в `photoMeta.caption`.

4. **Роль отправителя** — фильтрует доступные базы. На старте все staff-роли видят все 4 (упрощение); фильтр включим после formal описания ролей ([GAP-013](../00-meta/gaps.md#gap-013) + sales-manager).

### Источник `entityId` по базам

| База | Источник `entityId` |
|---|---|
| `client-locations` | Из активного сценария онбординга; иначе — диалог «Какой клиент?» (поиск по имени) |
| `shipments` | Из контекста доставки (если в сессии); иначе — диалог «Чья накладная?» → распознавание создаёт/находит Sale |
| `skus` | Диалог «Какой товар?» — выбор из справочника SKU |
| `purchase-offers` | Из контекста закупки; иначе — диалог «К какой закупке/рейсу?» |

### Альбом → один контекст

Пакет фото с одинаковым `media_group_id` обрабатывается **как единый набор** (приходит из (а) уже пакетом). Один уточняющий диалог о `(base, entityId)`, затем каждое фото пакета → `POST /api/photos` с одинаковым контекстом.

### Принятые решения

**Р1. Активный сценарий пилотит даже при caption.** Если идёт онбординг и менеджер прислал фото с подписью «фото товара» — фото уйдёт в `locationPhotos` (caption игнорируется как сигнал base). Снимает класс ошибок «перебивание контекста».

**Р2. Caption передаётся в `photoMeta.caption` всегда.** Полезен для людей при просмотре галереи.

**Р3. На старте все 4 базы доступны всем staff-ролям.** Фильтр по ролям — после formal описания ([GAP-013](../00-meta/gaps.md#gap-013)).

**Р4. Подтверждение выбора базы с undo.** После выбора Ева: «Принял, грузим в "📋 Отгрузочная накладная". Если ошибка — [Отменить]». Кнопка отмены сбрасывает выбор и возвращает к первому диалогу.

**Р5. Фото без понятного `entityId` — отказ, не запоминание.** Если менеджер выбрал базу, но не довёл диалог (отвлёкся) — Ева через 10 минут чистит сессию ожидания, файл из `/tmp/eva/...` удалит cron через 24 ч.

**Р6. `expectingPhoto` сбрасывается после получения.** Активный сценарий идёт дальше. На второе фото подряд Ева спросит: «Ещё одно фото витрины или это другое?».

**Р7. Хранение состояния сессии:** сейчас in-memory у Евы (`Map<chatId, session>`), при `pm2 restart eva` теряется. Переезд в Postgres (`eva_messages` + `eva_sessions`, [ADR-013](../90-decisions/ADR-013-eva-agent-architecture.md)) — Ит.4.

**Р8. Per-base handler как отдельный файл.** Структура `code/eva/src/photo/`:

```
photo/
  raw-photo.js              ← слой (а): скачивание + нормализация
  album-buffer.js           ← слой (а): буферизация альбомов
  dispatcher.js             ← слой (г): выбор base
  upload-client.js          ← слой (в): обёртка POST /api/photos
  handlers/
    client-locations.js     ← GAP-018
    shipments.js            ← GAP-020
    skus.js                 ← GAP-021
    purchase-offers.js      ← GAP-022
```

Новая база = новый файл `handlers/<base>.js`.

**Р9. Постобработка — точка расширения.** Каждый handler возвращает `{photoSaved, postProcess: () => ...}`. Для `shipments` (GAP-020) post-process запускает LLM-распознавание и создаёт/обновляет Sale. Для остальных — пустой post-process.

### Псевдокод

```
on RawPhoto[] (один или пакет альбома):
  session = getSession(chatId)
  
  // 1. Активный сценарий имеет приоритет
  if session.expectingPhoto:
    {base, entityId} = session.expectingPhoto
    handler = getHandler(base)
    for photo in photos:
      await uploadAndConfirm(handler, photo, base, entityId, session)
    session.expectingPhoto = null
    advanceActiveScenario(session)
    return
  
  // 2. Спрашиваем базу
  base = await askBaseInline(chatId, basesAvailable(senderRole))
  if !base: return cleanup()
  handler = getHandler(base)
  
  // 3. Handler решает про entityId
  entityId = await handler.resolveEntity(chatId, session, photos)
  if !entityId: return cleanup()
  
  // 4. Загрузка
  for photo in photos:
    await uploadAndConfirm(handler, photo, base, entityId, session)
  
  // 5. Постобработка handler-специфичная
  await handler.postProcess(session, photos, entityId)


uploadAndConfirm(handler, photo, base, entityId, session):
  resp = POST /api/photos (multipart: {base, entityId,
                                       uploadedBy: session.staffId,
                                       fileUniqueId: photo.fileUniqueId,
                                       caption: photo.caption,
                                       file: read(photo.localPath)})
  if resp.duplicate:
    notify(chatId, "Это фото уже было — пропускаю")
  else:
    notify(chatId, `Принял ✅ ${handler.label}`)
  fs.unlink(photo.localPath)
```

### Влияние на роли/процессы/сценарии (Обязательство 6 [ADR-015](../90-decisions/ADR-015-role-process-eva-structure.md))

При реализации каждой базы обновляем:

| База | Что правим |
|---|---|
| `client-locations` (GAP-018) | Сценарий [client-onboarding](../50-agents/eva/scenarios/client-onboarding.md) (шаг 9 ставит `expectingPhoto`); процесс [client-onboarding](../30-roles/to-be/sales-manager/processes/client-onboarding.md); инструкция sales-manager; системный промпт Евы |
| `shipments` (GAP-020) | Роль [expeditor](../30-roles/to-be/) ([GAP-013](../00-meta/gaps.md#gap-013)) + процесс «доставка»; сценарий Евы «приём отгрузки» |
| `skus` (GAP-021) | Роль закупщика/экспедитора (по пополнению фото); сценарий Евы «обновление каталога» |
| `purchase-offers` (GAP-022) | Сценарий Евы «рассылка предложений с закупки» + связь с `order-intake` |

Делаем **по факту** реализации каждой базы, не превентивно.

---

## Слой (д) — Просмотр в trade_app + конвенция URL

**Статус:** ✅ зафиксировано 2026-05-30.

Финальный слой — как пользователь trade_app видит и управляет фотографиями.

### Конвенция URL

```
https://trade-abkhazia.com/photos/<base>/<entityId>/<unixTs>_<fileUniqueId>.<ext>
```

URL стабилен пока сущность существует. Можно шарить ссылкой.

### Кэширование Nginx

Папку `/photos/` отдаём с заголовками:

```
Cache-Control: public, max-age=31536000, immutable
```

Имена файлов содержат `unixTs + fileUniqueId` — не меняются никогда → immutable. Браузер кэширует год; при удалении фото сервер просто перестаёт отдавать (404), кэшу неоткуда взяться (старый URL у пользователя только если уже был на странице).

### Где фотографии в trade_app

| База | Где в UI |
|---|---|
| `client-locations` | Карточка клиента: секция «Фото витрины». Просмотр + редактирование |
| `shipments` | Карточка Sale в журнале продаж: секция «Отгрузочная накладная» |
| `skus` | Карточка товара: секция «Фото товара». Первое фото (`photos[0]`) — опционально в списке товаров и выпадающих SKU-выборах (бонус, не блокер) |
| `purchase-offers` | Карточка Purchase/Trip: секция «Фото с закупки» (для рассылок) |

### Универсальный компонент `<PhotoGallery>`

Один компонент на все 4 базы:

```js
<PhotoGallery
  photos={photoMeta[]}      // массив photoField из карточки
  editable={isEditMode}     // показать × и кнопку загрузки
  onUpload={(files) => ...}  // вызовет POST /api/photos
  onDelete={(meta) => ...}   // вызовет DELETE /api/photos
  emptyText="Фото пока нет"
/>
```

Внешне:
- Горизонтальная полоса миниатюр (~80×80, `object-fit: cover`).
- Над каждой в режиме редактирования — крестик «×» (с `confirm("Удалить?")`).
- В конце ряда — плитка `+ Загрузить` (`<input type=file accept="image/*" multiple>`).
- Клик по миниатюре → lightbox.

Добавление галереи в новую базу/новую секцию = одна строка `<PhotoGallery ... />` — расширяемость пункта 5 на стороне UI.

### Lightbox

Простой модал поверх страницы:
- Затемнение фона + клик по подложке = закрыть.
- Фото в полный экран с сохранением aspect ratio.
- Клавиатура: `←/→` листать, `Esc` закрыть.
- Под фото — `caption`, `uploadedBy`, `uploadedAt`.

Реализация — ~100 строк своего кода без либы (либа = лишняя зависимость для такой простой задачи).

### Загрузка из trade_app (запасной канал)

Через тот же endpoint `POST /api/photos`:
- `entityId` — текущая редактируемая карточка.
- `uploadedBy` — текущий пользователь (после RBAC); сейчас — owner.
- `fileUniqueId` — UUID4 на фронте (`crypto.randomUUID()`).
- `caption` — опциональный input.
- `X-Photo-Token` — на фронте в build-time env.

Зачем: запасной канал, если Ева недоступна или менеджер уже сидит в trade_app.

### Удаление из trade_app

Кнопка `×` на миниатюре → `confirm` → `DELETE /api/photos {base, entityId, fileUniqueId}` → обновление локального массива на фронте.

### Что НЕ делаем сейчас (KISS)

- Переупорядочивание фото (drag-drop).
- Выбор «главного» для SKU (используем просто `photos[0]`).
- Crop / rotate / фильтры на фронте.
- Прогресс-бар (sequential `await` достаточно).
- Превью при наведении в списках клиентов/товаров.

Всё перечисленное легко добавить позже без поломок структуры.

### Принятые решения

**Р10. Один универсальный `<PhotoGallery>`** на все 4 базы.
**Р11. UUID4 на фронте** для `fileUniqueId` при загрузке из trade_app.
**Р12. `Cache-Control immutable` — год.**
**Р13. Прямой `<img src="...">` без проксирования** — Nginx отдаёт публично, токенов на чтение нет (см. слой (б)).

---

## Резюме архитектуры

| Слой | Что | Где живёт |
|---|---|---|
| (а) Приём | Telegram update → `RawPhoto` в `/tmp/eva/...` | `code/eva/src/photo/raw-photo.js`, `album-buffer.js` |
| (б) Хранение | `/var/www/trade/photos/<base>/<entityId>/<unixTs>_<fileUniqueId>.<ext>` | VPS filesystem + Nginx |
| (в) Endpoint | `POST/DELETE /api/photos` (multipart, `X-Photo-Token`) | trade-api `server.js` + `photo-bases.js` |
| (г) Диспетчер | `RawPhoto` → `(base, entityId)` через handlers | `code/eva/src/photo/dispatcher.js`, `handlers/<base>.js` |
| (д) UI | `<PhotoGallery>` в карточках + конвенция URL + Nginx cache | `trade_app_v3.jsx` + `nginx.conf` |

Поток end-to-end:

```
Менеджер → Telegram (фото)
  ↓ (а) identify staff, download → /tmp/eva/
  ↓ (г) диспетчер: активный сценарий → handler → entityId
  ↓ (в) POST /api/photos на trade-api
  ↓ (в+б) validate → mkdir → rename → KV update
  ↓ (д) фото видно в карточке клиента в trade_app через ~секунду
```

### Расширяемость (пункт 5 разбора 2026-05-30)

Чтобы добавить новую базу фото `<X>`:
1. Добавить строку в `PHOTO_BASES` (trade-api): `kvKey`, `photoField`, `findById`, `label`.
2. Добавить поле `photos[]` в карточку сущности (если ещё нет).
3. Создать `code/eva/src/photo/handlers/X.js` (resolveEntity, postProcess).
4. Добавить `<PhotoGallery>` в секцию карточки в trade_app.

Никаких изменений в слоях (а), (б) (Nginx), (в) (endpoint), (д) (компонент). Это и есть инженерная гарантия пункта 5.
