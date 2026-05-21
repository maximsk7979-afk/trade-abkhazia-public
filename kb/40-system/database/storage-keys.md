---
title: Ключи storage
status: active
last_updated: 2026-04-28
references:
  - overview.md
referenced_by:
  - ../frontend.md
---

# Ключи storage

Перечень всех ключей в таблице `app_storage`. **Не менять имена** — это историческое соглашение, подвязанное к коду.

## Транзакционные данные

| Ключ | JS-константа | Содержит |
|---|---|---|
| `trade-sales-v1` | SK_SALES | Продажи ПС/ЗД/ЗР (одна сущность Sale, см. [ADR-007](../../90-decisions/ADR-007-sales-canonical-model.md)) |
| `trade-deliveries-v1` | SK_DLV | Доставки Д-XXX |
| `trade-trips-v8` | SK_T | Рейсы Р-XXX |
| `trade-purchases-v1` | SK_PUR | Закупки вне рейса ЗК-XXX (см. [purchase.md](../data-formats/purchase.md), ADR-009 Слой Б) |
| `trade-batches-v1` | SK_BATCH | FIFO-партии B-XXXX (источник `trip` или `purchase`) |
| `trade-whops-v1` | SK_WH | Складские операции |
| `trade-settlements-v2` | SK_SETT | Взаиморасчёты |

> ⚠️ Ключ `trade-orders-v4` (`SK_O`) **удалён** 2026-04-28. Это была мёртвая ветка ранней архитектуры. См. [ADR-007](../../90-decisions/ADR-007-sales-canonical-model.md).

## Справочники

| Ключ | JS-константа | Содержит |
|---|---|---|
| `trade-pricelist-v1` | SK_PL | Прайс-лист |
| `trade-cat-products` | SK_PROD | Справочник товаров |
| `trade-cat-skus` | SK_SKU | Справочник SKU |
| `trade-cat-clients` | SK_CLI | Клиенты |
| `trade-cat-suppliers` | SK_SUP | Поставщики |
| `trade-cat-contragents` | SK_CONT | Контрагенты |
| `trade-cat-exptypes` | SK_EXP | Типы расходов |
| `trade-cat-staff` | SK_STAFF | Сотрудники |
| `trade-cat-offices` | SK_OFF | Офисы (склады) |
| `trade-cat-projects` | SK_PRJ | Проекты PRJ-NNN (мультипроектность, [ADR-009](../../90-decisions/ADR-009-multitenancy-projects.md)) |

## Маркеры seed-блоков

| Ключ | Что значит |
|---|---|
| `trade-seed-v40-full` ... `trade-seed-v45-full` | Маркеры применённых seed-блоков. Если ключ есть — соответствующий seed уже применён, повторно не применять. |

Текущая активная версия — `_SD8` / `trade-seed-v45-full`.

## Размеры (на 2026-04-28 после очистки)

После канонизации модели ([ADR-007](../../90-decisions/ADR-007-sales-canonical-model.md)) транзакционные ключи **очищены**, оставлены только справочники. Production стартует с чистой базы.

| Ключ | Размер | Состояние |
|---|---:|---|
| trade-sales-v1 | пусто (`[]`) | очищен 2026-04-28 |
| trade-trips-v8 | пусто (`[]`) | очищен 2026-04-28 |
| trade-deliveries-v1 | пусто (`[]`) | очищен 2026-04-28 |
| trade-batches-v1 | пусто (`[]`) | очищен 2026-04-28 |
| trade-whops-v1 | пусто (`[]`) | очищен 2026-04-28 |
| trade-settlements-v2 | пусто (`[]`) | очищен 2026-04-28 |
| trade-orders-v4 | (удалён ключ) | удалён 2026-04-28 |
| trade-cat-* | сохранены | справочники активны |
| trade-pricelist-v1 | сохранён | |
| trade-seed-vXX-full | сохранены | маркеры — чтобы старые seed не загружались повторно |

## Принципиальное

- **Все значения** — это JSON-строка
- **Транзакционные ключи** хранят **массив** объектов (`[{...}, {...}, ...]`)
- **Справочники** хранят **массив** объектов

⚠️ POST в `/api/storage/:key` **переписывает всё значение** — частичных обновлений нет.
