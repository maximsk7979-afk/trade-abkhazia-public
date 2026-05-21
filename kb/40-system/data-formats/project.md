---
title: Формат данных — Project (проект/направление бизнеса)
status: active
last_updated: 2026-05-21
related_decisions:
  - ADR-009-multitenancy-projects.md
---

# Project — направление бизнеса

Введён [ADR-009](../../90-decisions/ADR-009-multitenancy-projects.md). Реализован 2026-05-21 (Слой А Шага 1).

- **Storage-ключ:** `trade-cat-projects` (`SK_PRJ` в `code/trade_app_v3.jsx`)
- **Префикс ID:** `PRJ-NNN` (латиница — чтобы не путать с `П-` поставщик, `ПР-` партнёр)
- **Default-проект:** `PRJ-001` (`DEFAULT_PRJ`) — присваивается существующим данным без `projectId` на чтении

## Формат записи

```json
{
  "id": "PRJ-001",
  "name": "Турецкие овощи",
  "shortName": "TURK-VEG",
  "status": "active",
  "startedAt": "2024-01-01",
  "category": "perishable-vegetables",
  "defaultCurrency": "GEL",
  "defaultOfficeId": "О-001",
  "ownerStaffId": "С-001",
  "knowledgeBasePath": "kb/20-domain/invoices/suppliers/p-004-zaza",
  "uiColor": "#5BA873",
  "description": "..."
}
```

| Поле | Значения | Назначение |
|---|---|---|
| `status` | `active` / `paused` / `archived` | архивные не показываются в селекторах |
| `category` | `perishable-vegetables` / `beverages` / `frozen` / `construction` / `snacks` | UI и аналитика |
| `defaultCurrency` | `GEL` / `RUB` / `USD` / `TRY` / `AMD` / `mixed` | валюта расчётов с поставщиками проекта |
| `defaultOfficeId` | `О-NNN` | основной склад проекта |
| `ownerStaffId` | `С-NNN` | ответственный сотрудник |
| `knowledgeBasePath` | путь в `kb/` | спец-знания проекта для Евы |
| `uiColor` | hex | цвет метки в интерфейсе |

## Стартовый набор (8 проектов)

PRJ-001 Турецкие овощи (active), PRJ-002 Местные овощи (active, RUB), PRJ-003 Грузинские овощи (active, июнь 2026), PRJ-004 Бананы, PRJ-005 Напитки, PRJ-006 Заморозка, PRJ-007 Цемент, PRJ-008 Снеки — последние пять `paused` (резерв ID). Полная таблица — [ADR-009 §2](../../90-decisions/ADR-009-multitenancy-projects.md).

## Где используется `projectId`

| Сущность | Уровень | Статус реализации |
|---|---|---|
| Trip | запись (`trip.projectId`) | ✅ Слой А (источник проекта — задаётся в форме рейса) |
| Batch | запись (`batch.projectId`) | ✅ наследуется от рейса при приёмке |
| WhOp | запись (`whOp.projectId`) | ✅ default PRJ-001 + селектор в ручной операции |
| Settlement | запись (`settlement.projectId`) | ✅ default PRJ-001 + селектор; `allocation` для cross-project — Шаг 3 |
| Sale | **позиции** (`sale.items[].projectId`) | ⏳ Шаг 3 (per-line + рефакторинг FIFO/маржи) |
| Purchase | запись | ⏳ Слой Б |
| Invoice | запись | ⏳ Слой В |

## Фильтрация в UI

Глобальный селектор «🗂 Все проекты / <проект>» в навбаре (`projFilter`). Применяется к project-aware вьюшкам: список рейсов, таблица «Партии FIFO». Общий остаток склада по проекту **не** фильтруется (агрегируется из не-project-aware продаж/доставок до Шага 3).
