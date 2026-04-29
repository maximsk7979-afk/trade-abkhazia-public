---
title: Махо
status: active
last_updated: 2026-04-28
data_source:
  type: database
  storage_key: trade-cat-staff
  staff_id: С-003
references:
  - ../30-roles/buyer.md
referenced_by: []
---

# Махо

> Оперативные данные — в БД, ключ `trade-cat-staff`, запись С-003.
> Этот файл — только заметки.

## Заметки

- Закупщик в Грузии. Делает закупки на грузинских оптовых рынках.
- Также фигурирует как **контрагент** КА-001 в `trade-cat-contragents` — когда оплачиваем ему услуги закупки/грузчика. Это **не дубликат**: как сотрудник он один (С-003), как получатель платежей — это контрагент КА-001.
- Платежи ему за услуги — статья `buyer` (закупщик) и иногда `loader` (грузчик), валюта GEL.

## Связанные документы

- [30-roles/buyer.md](../30-roles/buyer.md)
- [20-domain/settlements/contractors.md](../20-domain/settlements/contractors.md) — про него как про контрагента
