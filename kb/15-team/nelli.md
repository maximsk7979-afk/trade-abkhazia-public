---
title: Нелли
status: active
last_updated: 2026-04-28
data_source:
  type: database
  storage_key: trade-cat-staff
  staff_id: С-002
references:
  - ../30-roles/ops-manager.md
referenced_by: []
---

# Нелли

> Оперативные данные — в БД, ключ `trade-cat-staff`, запись С-002.
> Этот файл — только заметки.

## Заметки

- Операционный менеджер компании. Ведёт всю операционку.
- Сейчас весь учёт у неё — в **Excel-файлах**. Это базовое состояние AS-IS.
- Главный пользователь Евы в роли оп-менеджера. Её сценарии работы с агентом — приоритетные при проектировании.
- Telegram: `@Nelli2023N`, chat_id `1719753990`. Используется в `NOTIFY_CHAT_IDS` для системных уведомлений.
- Получает уведомления при создании нового клиента (см. [50-agents/eva/scenarios/](../50-agents/eva/scenarios/)).

## Связанные документы

- [30-roles/ops-manager.md](../30-roles/ops-manager.md)
- [10-business/operating-model.md](../10-business/operating-model.md) — там будет описан её текущий рабочий процесс
