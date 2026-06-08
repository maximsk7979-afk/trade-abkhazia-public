---
title: Data formats — физический формат данных
status: stub
last_updated: 2026-04-28
---

# Физический формат данных

JSON-формат сущностей в `app_storage`. Имена полей, типы, особенности сериализации.

> **Логическая модель** — в [`20-domain/data-model/`](../../20-domain/data-model/). Здесь — про **JSON-формат**.

## Описанные форматы

✅ Готово: [sale.md](sale.md), [trip.md](trip.md) (2026-06-08), [purchase.md](purchase.md), [batch.md](batch.md), [settlement.md](settlement.md), [client.md](client.md), [staff.md](staff.md), [office.md](office.md), [project.md](project.md), [whop.md](whop.md).

⏳ Ещё нет: delivery.md, product.md, sku.md (наполнение — по сущности за раз).

## Принципиальные особенности

- **Все суммы — строкой**: `"qty":"5"`, `"price":"260"`. Парсятся через `parseFloat || 0`.
- **ID** — кириллические префиксы (`ПС-087`, `Р-001`).
- **Даты** — формат `YYYY-MM-DD` (ISO-8601 без времени).
- **Время** — `YYYY-MM-DD HH:MM:SS` (для history).

## Формат записи каждой сущности

Каждый файл `<entity>.md` начинается с:
```markdown
> Логическая модель: [20-domain/data-model/entities.md#<имя>](...)

# JSON-формат <сущности> в app_storage

Хранится в `<storage-key>` как массив объектов.

## Пример
```json
{...}
```

## Поля
| Поле | Тип | Описание |
|---|---|---|

## Особенности
- ...
```
