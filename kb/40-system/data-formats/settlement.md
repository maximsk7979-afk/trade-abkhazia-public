---
title: Settlement — JSON-формат записи взаиморасчёта
status: active
last_updated: 2026-04-28
references:
  - ../database/storage-keys.md
  - ../../20-domain/settlements/README.md
referenced_by: []
---

# Settlement — JSON-формат записи взаиморасчёта

## Storage

- Ключ: `trade-settlements-v2`
- JS-константа: `SK_SETT`
- Содержимое: массив объектов

## Назначение

Записи в `trade-settlements-v2` — это **ручные операции** по взаиморасчётам с разными контрагентами (клиентами, поставщиками, партнёрами, прочими) **поверх** автоматических начислений из рейсов и продаж.

Автоматические начисления (закупки, отгрузки) — **не** в `settlements`, а вычисляются на лету из `trips`, `sales`, `deliveries`. См. [20-domain/settlements/README.md](../../20-domain/settlements/README.md).

`settlements` дополняет автоматику ручными записями (оплаты, корректировки, возвраты за брак).

## Формат

Базовая структура:

```json
{
  "id": "S-...",
  "acctType": "client | supplier | partner | expense | warehouse",
  "opType": "payment | charge | defect_refund | balance",
  "date": "YYYY-MM-DD",
  "amount": "положительное число строкой",
  "desc": "комментарий"
}
```

### Дополнительные поля по `acctType`

| `acctType` | Дополнительные поля |
|---|---|
| `client` | `clientId: "К-XXX"` |
| `supplier` | `supId: "П-XXX"` |
| `partner` | (без идентификатора — партнёр сейчас один: Лия) |
| `expense` | `contragentId: "КА-XXX"` |
| `warehouse` | (без идентификатора) |

### Типы операций (`opType`)

| `opType` | Семантика |
|---|---|
| `payment` | Платёж — мы получили от стороны (для клиента) или мы заплатили (для поставщика/партнёра/контрагента). Знак trace в коде. |
| `charge` | Ручное начисление |
| `defect_refund` | Возврат за брак (только для `acctType: "client"`) |
| `balance` | Корректировка баланса (для `acctType: "partner"`) |

## Возврат за брак

```json
{
  "id": "S-REFUND-001",
  "acctType": "client",
  "clientId": "К-XXX",
  "opType": "defect_refund",
  "date": "2026-04-28",
  "amount": "1000",
  "desc": "Возмещение за брак: черри 5кг × 200₽"
}
```

- `amount` — всегда **положительное** число строкой
- Баланс клиента = Отгрузки − Оплаты − Defect_refund (система сама вычитает)
- Распознаётся ботом из строк "Возврат / Брак" в накладной

## Партнёрские операции (Лия)

См. [20-domain/settlements/liya-partner.md](../../20-domain/settlements/liya-partner.md).

Автоматические доли по рейсам **не пишутся** в `settlements` — они вычисляются на лету из `trip.partners[]` и `trip.expenses[]`. В `settlements` пишутся только **ручные** операции (платежи, корректировки):

```json
{
  "id": "S-...",
  "acctType": "partner",
  "opType": "payment",
  "date": "2026-04-28",
  "amount": "53000",
  "desc": "Оплата Лие за рейс Р-005"
}
```

## Особенности сериализации

- `amount` — **строка**, парсится через `parseFloat || 0`
- `id` обычно вида `S-` + случайный/timestamp suffix
- Никаких `history[]` (это ручные одношаговые операции)

## История

- 2026-04-28: Формат описан в kb/.
