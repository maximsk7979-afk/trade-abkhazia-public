---
title: Формат данных — Цикл закупки банана (BC-*, KV trade-banana-cycles)
status: active
last_updated: 2026-06-11
references:
  - ../../90-decisions/ADR-023-procurement-orchestrator.md
  - ../../50-agents/eva/scenarios/banana-procurement-orchestration.md
  - ../../30-roles/to-be/buyer/processes/banana-procurement-field-DRAFT.md
  - trip.md
referenced_by: []
---

# Цикл закупки банана (BC-*)

> Сущность оркестратора закупки «из поля» ([ADR-023](../../90-decisions/ADR-023-procurement-orchestrator.md)).
> **Один цикл = одна поставка на день D4** (D4 = день доставки = D0+4). Хранение —
> KV `trade-banana-cycles` (массив записей), id = `BC-<D4>` (идемпотентно: один цикл на дату).
> Код: `code/eva/src/procurement/` (чистый автомат `cycle-machine.js`, storage `cycle-store.js`,
> эффекты `effect-exec.js`, проактивный привод `procurement-tick.js` + реактивный `advance-cycle`).

## Состояния (STATES)

| Состояние | Смысл (шаг процесса) | Кто «должен ответить» (pending) |
|---|---|---|
| `INIT` | цикл заведён, ждём решение ГД: сколько ящиков и у кого (шаг 1) | gd |
| `ORDERING` | заказ отправлен поставщику, ждём подтверждение (шаг 2) | supplier |
| `TRIP_OPEN` | рейс PRJ-004 заведён (лимит X, отсечка 13:00 D4), продажи идут (шаги 3–4) | — |
| `LIMIT_REACHED` | лимит исчерпан — ждём решение ГД о доп. объёме Y (шаг 5) | gd |
| `DRIVER_PENDING` | D3: ждём назначение водителя от ГД (шаг 7) | gd |
| `INVOICE_PENDING` | ждём накладную от поставщика (шаг 8; PDF-сверка — заглушка) | supplier |
| `PROEZD_PENDING` | ждём подтверждение согласования проезда от закупщика (шаг 9) | buyer |
| `LOADING` / `PAID` | погрузка / оплата (шаги 10–11; модули мини-аппа — заглушки) | buyer |
| `DONE` | рейс завершён, доставка D4 (шаги 12–13) | — |
| `CANCELLED` | цикл отменён (отказ поставщика без альтернативы и т.п.) | — |

## События (EVENTS) и приводы

Автомат приводится **двумя приводами**:
- **проактивный** — cron-тик каждые 5 мин (`procurement-tick.js`, время грузинское UTC+4):
  `KICKOFF` (09:00 D0, гейт `PROCUREMENT_KICKOFF_ENABLED`), эскалации по дедлайнам
  (`GD_TIMEOUT` 16:00 D0; `ASK_DRIVER` 14:00 D3; `DRIVER_TIMEOUT` 16:00 D3;
  `PROEZD_REMIND` 17:30 / `PROEZD_TIMEOUT` 18:00 D3);
- **реактивный** — ответы акторов в диалоге с Евой → инструмент `advance_cycle`
  (`GD_DECISION {qtyX, supplierId}`, `SUPPLIER_CONFIRM/REJECT`, `GD_INCREASE {qtyY}`,
  `DRIVER_ASSIGNED {driverContractorId}`, `INVOICE_RECEIVED`, `PROEZD_CONFIRMED`);
  плюс `CAPACITY_FULL` от mini-api при исчерпании лимита; `CANCEL` — из любого состояния.

Переход = чистая `transition(cycle, event, payload, nowGe)` → новый цикл + **семантические
эффекты** (`message` акторам, `createTrip`, `updateCapacity`, `escalate` владельцу), которые
исполняет `effect-exec.js`.

## Запись цикла (поля)

```json
{
  "id": "BC-2026-06-15",
  "d4": "2026-06-15",            // день доставки (якорь всех дедлайнов)
  "state": "TRIP_OPEN",
  "pending": null,                // кто должен ответить: "gd" | "supplier" | "buyer" | null
  "qtyX": 100,                    // согласованный объём (ящиков)
  "qtyY": 0,                      // доп. объём при увеличении лимита
  "supplierId": "П-005",
  "tripId": "Р-003",              // рейс PRJ-004, заведённый циклом
  "driverContractorId": "КА-007",
  "flags": { "gdEscalated": true },  // идемпотентность временных событий
  "log": [ { "at": "2026-06-11T09:00:00+04:00", "event": "KICKOFF", "to": "INIT" } ],
  "updatedAt": "2026-06-11T09:00:00+04:00"
}
```

## Связи и инварианты

- `tripId` → рейс PRJ-004 ([trip.md](trip.md)): `orderCapacity = qtyX (+qtyY)`,
  `orderCutoffAt = D4T13:00:00+04:00`, `procurementCycleId = BC-<D4>`.
- Поставщик видит **только свой** цикл (`supplierId`, изоляция ADR-022).
- Ручное заведение рейса (`create_banana_trip`) **отказывает**, если по дате есть активный цикл.
- Терминальные состояния: `DONE`, `CANCELLED` (tick их не трогает).
- Сейчас (2026-06-11) авто-`KICKOFF` выключен гейтом — циклы заводятся только реактивно;
  включение — после онбординга поставщиков (roadmap 1.6).
