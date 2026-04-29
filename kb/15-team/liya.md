---
title: Лия
status: active
last_updated: 2026-04-28
data_source:
  type: database
  storage_keys:
    - trade-cat-clients (К-006)
    - trade-trips-v8 (как partner в trip.partners[])
references:
  - ../20-domain/settlements/liya-partner.md
  - ../30-roles/partner.md
  - ../30-roles/client.md
referenced_by: []
---

# Лия

> Оперативные данные — в БД. Этот файл — заметки про двойную роль.

## Заметки

Лия одновременно выступает в **двух разных ролях**, которые в системе учитываются раздельно:

### 1. Партнёр по транспорту
- Делит машину при рейсах из Грузии в Абхазию.
- Расходы рейса делятся пропорционально весу товара.
- В системе хранится **по имени** (строкой "Лия") в массиве `trip.partners[]` каждого рейса.
- **Не имеет ID контрагента КА-XXX.**

### 2. Клиент-покупатель К-006
- Покупает у нас часть товара по оптовой цене (`pt: "opt"`).
- В справочнике `trade-cat-clients` как стандартная запись.

**Эти две линии не пересекаются** — балансы партнёра и клиента К-006 ведутся независимо. См. [20-domain/settlements/liya-partner.md](../20-domain/settlements/liya-partner.md) для детальной механики расчётов.

## Связанные документы

- [20-domain/settlements/liya-partner.md](../20-domain/settlements/liya-partner.md) — главный файл по учёту
- [30-roles/partner.md](../30-roles/partner.md) — роль партнёра вообще
- [30-roles/client.md](../30-roles/client.md) — роль клиента вообще
- [20-domain/data-model/](../20-domain/data-model/) — формат данных рейса с partners[]
