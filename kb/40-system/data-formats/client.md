---
title: Client — JSON-формат клиента
status: active
last_updated: 2026-06-08
references:
  - ../database/storage-keys.md
  - ./sale.md
  - ../photo-infrastructure.md
  - ../../90-decisions/ADR-014-client-project-profile.md
  - ../../90-decisions/ADR-016-photo-infrastructure.md
  - ../../30-roles/to-be/sales-manager/processes/client-onboarding.md
referenced_by: []
---

# Client — JSON-формат клиента

## Storage

- Ключ: `trade-cat-clients`
- JS-константа: `SK_CLI`
- Содержимое: массив объектов

## Формат (текущий, после ADR-014 + полей онбординга 2026-05-23)

```json
{
  "id": "К-026",
  "n": "Магазин «Морской»",
  "contactName": "Аршак",
  "onbStatus": "Активен",
  "projects": [
    {
      "projectId": "PRJ-002",
      "segment": "СГ-002",
      "priceLevel": "opt",
      "saleType": "ds"
    },
    {
      "projectId": "PRJ-003",
      "segment": "СГ-002",
      "priceLevel": "opt",
      "saleType": "ds"
    }
  ],
  "wa": "+79991234567",
  "tg": "@arshak",
  "tgChatId": "123456789",
  "ynavUrl": "https://yandex.ru/maps/...",
  "locationPhotos": [
    {
      "url": "https://trade-abkhazia.com/photos/client-locations/К-026/1748600096_AQADXX.jpg",
      "uploadedBy": "С-006",
      "uploadedAt": "2026-05-30T12:34:56Z",
      "caption": "Витрина со стороны входа",
      "fileUniqueId": "AQADXX",
      "size": 245678,
      "width": 1280,
      "height": 960
    }
  ],
  "managerId": "С-006"
}
```

## Поля

| Поле | Тип | Обязательное | Описание |
|---|---|---|---|
| `id` | string | да | ID с префиксом `К-` (например, `К-007`) |
| `n` | string | да | **Уникальное** название клиента. Уникальность проверяется case-insensitive trim в `saveClient` (форма trade_app) и в инструменте Евы `create_client` |
| `contactName` | string | да для новых | Имя контактного лица — Ева им обращается к клиенту. Для существующих до 2026-05-23 — пустая строка с мягким предупреждением при сохранении |
| `onbStatus` | string | да | Жизненный статус клиента: `"Активен"` или `"На паузе"` (русские строки в KV). По умолчанию — `"Активен"`. Миграция на чтении: legacy без поля = `"Активен"` |
| `deliveryMode` | string | нет | **Способ доставки по умолчанию** (на **все** проекты, не per-project): `"delivery"` (доставка) / `"pickup"` (самовывоз). Мини-апп предзаполняет тумблер заявки этим значением (защита от ошибки — клиент не забудет переключить). По умолчанию `"delivery"`; миграция на чтении из `projects[0].saleType` (`ps`→`pickup`, иначе `delivery`). ADR-021 / 2026-06-06 |
| `projects` | array | да | Проектные блоки клиента (см. ниже). Минимум один блок |
| `wa` | string | да на онбординге | WhatsApp в международном формате `+<код>` |
| `tg` | string | нет | Telegram username (`@username`). Заполняется автоматически при активации deep-link |
| `tgChatId` | string | нет | Chat ID для бота/Евы. Заполняется автоматически при активации deep-link |
| `ynavUrl` | string | нет | URL точки в Яндекс.Навигаторе (для водителей) |
| `locationPhotos` | array | нет | Массив объектов `photoMeta` (URL фото входа/витрины + метаданные). База `client-locations` фото-инфраструктуры ([ADR-016](../../90-decisions/ADR-016-photo-infrastructure.md)). Заполняется через Еву (шаг 9 онбординга) или через форму trade_app (после Блока 1 GAP-018). Формат `photoMeta` — см. ниже |
| `managerId` | string | нет | Закреплённый менеджер продаж (`С-XXX`, роль `sales_manager`). При создании продажи копируется в `sale.managerId` |
| `bananaFixedPrice` | number / string | нет | **Договорная фикс-цена банана** ₽/ящик. Если задана (>0) — перебивает сетку объёма (`volume-tier`) для этого клиента и **не добавляет** надбавку за доставку (`source: "fixed"`, см. [ADR-020](../../90-decisions/ADR-020-order-intake-banana-pricing.md)). Пусто/отсутствует = цена по сетке объёма. Читается в `order-service.clientFixedPrice`. Задаётся в карточке клиента trade_app или через Еву `create_client` (опц. поле). Введено 2026-06-08 (Фаза 0 банана) |

### Блок проекта (`projects[]`)

| Поле | Тип | Описание |
|---|---|---|
| `projectId` | string | `PRJ-NNN` из [trade-cat-projects](./project.md) |
| `segment` | string | `СГ-NNN` из `trade-cat-segments` (только сегменты с этим `projectId` в `applicableProjects`) |
| `priceLevel` | string | `opt` / `mid` / `ret` — уровень цены клиента в этом проекте |
| `saleType` | string | `ps` (со склада) / `ds` (доставка) — тип продажи в этом проекте. Пустая строка допустима |

См. правила хелперов `getClientProjects` и `getClientPriceLevel(client, projectId)` в [ADR-014](../../90-decisions/ADR-014-client-project-profile.md) §4. На онбординге сегмент/уровень цены/тип продажи задаются **одним ответом** и пишутся во все проектные блоки одинаково (см. [процесс client-onboarding](../../30-roles/to-be/sales-manager/processes/client-onboarding.md)).

> ⚠️ **Банан (PRJ-004) не использует `priceLevel`.** Уровень цены `opt/mid/ret` — это cost-markup модель для овощей. Цена банана берётся по **сетке объёма** (`volume-tier`, [ADR-020](../../90-decisions/ADR-020-order-intake-banana-pricing.md)) или по `bananaFixedPrice` (если задана). Поэтому у бананового клиента `projects[PRJ-004].priceLevel` фактически игнорируется ценообразованием банана — он остаётся в карточке для единообразия, но на цену банана не влияет.

### Объект `photoMeta` (элемент массива `locationPhotos[]`)

| Поле | Тип | Описание |
|---|---|---|
| `url` | string | Публичный URL `https://trade-abkhazia.com/photos/client-locations/<clientId>/<unixTs>_<fileUniqueId>.<ext>` |
| `uploadedBy` | string | `staffId` (С-NNN) загрузившего |
| `uploadedAt` | string | ISO-дата загрузки |
| `caption` | string | Подпись (опционально; берётся из Telegram caption или из формы trade_app) |
| `fileUniqueId` | string | Для дедупа (из Telegram `file_unique_id` или UUID4 для веб-формы) |
| `size` | number | Байты |
| `width`, `height` | number | Пиксели (0 если не удалось определить — например, HEIC) |

Формат `photoMeta` **одинаков** для всех четырёх баз фото ([ADR-016](../../90-decisions/ADR-016-photo-infrastructure.md)): `Sale.shipmentPhotos[]`, `Sku.photos[]`, `Purchase.purchasePhotos[]` устроены так же. Источник истины — [photo-infrastructure.md](../photo-infrastructure.md), слой (б).

## Legacy-поля (для обратной совместимости при чтении)

Эти поля встречаются в записях, заведённых до ADR-014 (до 2026-05-22). В новых клиентах не появляются; при чтении старого клиента `getClientProjects` мигрирует их **в `projects[0]`** на лету, без деструктивной массовой миграции данных (по стилю [ADR-009](../../90-decisions/ADR-009-multitenancy-projects.md) §8.2).

| Поле | Тип | Маппинг |
|---|---|---|
| `pt` | string | → `projects[0].priceLevel` (если `projects[]` отсутствует или пуст) |
| `saleType` | string | → `projects[0].saleType` |

## `onbStatus` ≠ `onboarding`

Это **разные** поля — не путать:

- **`onbStatus`** (это поле) — **жизненный цикл клиента**, введено 2026-05-23 (Шаг 3 [roadmap](../../65-roadmap/current.md)). Управляется менеджером через веб-форму trade_app. Значения: `"Активен"` / `"На паузе"`.
- **`onboarding`** — **статус знакомства с Евой** (ADR-014 п.3), будет введён на **Шаге 4** (взаимодействие Ева ↔ клиенты). Сейчас в коде отсутствует.

См. [CHANGELOG 2026-05-23](../../95-changelog/2026-Q2.md).

## Особые случаи

- **Разовый клиент ("с улицы")** — `clientId` в продаже задаётся как `__walk_in__`, имя пишется в `saleData.clientName`. В справочнике клиентов **не создаётся**, онбординг не проходит.
- **Клиент-партнёр** (К-006 = Лия) — обычная запись в справочнике, при этом тот же человек ещё фигурирует как партнёр в `trip.partners[]`. См. [20-domain/settlements/liya-partner.md](../../20-domain/settlements/liya-partner.md).

## Связи

- `id` → используется в `sale.clientId`, `dlv.clients[].clientId`, `settlements[].clientId`
- `managerId` → ссылка на сотрудника в `trade-cat-staff`
- `projects[].projectId` → ссылка на `trade-cat-projects`
- `projects[].segment` → ссылка на `trade-cat-segments`
- Клиент **не привязан** к складу — отгрузки могут быть с любого офиса

## Особенности

- Имена полей частично **сокращены** (`n`, `wa`, `tg`) — историческое решение; новые поля (`contactName`, `onbStatus`, `locationPhotos`, `managerId`) — нормальные читаемые имена.
- Все строковые, кроме `projects[]` и `locationPhotos[]` (массивы).

## История

- 2026-04-28: Формат описан в kb/ (плоская карточка).
- 2026-05-22: ADR-014 — проектные блоки `projects[]`, `locationPhotos[]` (К3, коммиты `a3472d2`, `417dc3f`, `43cfe4d`).
- 2026-05-23: Поля онбординга `contactName`, `onbStatus` + уникальность `n` (коммит `2403e44`).
- 2026-05-30: `locationPhotos[]` переоформлен из массива URL в массив объектов `photoMeta` ([ADR-016](../../90-decisions/ADR-016-photo-infrastructure.md)). До этого момента поле было пустым во всех карточках — миграции не требуется. Реализация — Блок 1 GAP-018.
- 2026-06-08: добавлено поле `bananaFixedPrice` (договорная фикс-цена банана) — UI карточки клиента в trade_app + опц. поле в Еве `create_client` (Фаза 0 банана). Зафиксирован нюанс «банан игнорирует priceLevel».
