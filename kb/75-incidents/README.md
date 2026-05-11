---
title: Incidents — журнал инцидентов
status: active
last_updated: 2026-05-11
---

# 75-incidents — Журнал инцидентов

Файлы с описаниями сбоев и инцидентов. Один инцидент = один файл.

## Формат

Шаблон: [00-meta/templates/incident-template.md](../00-meta/templates/incident-template.md).

ID инцидентов: `INC-NNN-короткое-имя.md`.

## Перечень инцидентов

| ID | Дата | Severity | Статус | Заголовок |
|---|---|---|---|---|
| [INC-001](INC-001-bot-anthropic-balance.md) | 2026-04-26 | medium | resolved | Бот падал из-за пустого Anthropic API balance |
| [INC-002](INC-002-vps-billing-outage.md) | 2026-05-08 | high | resolved | VPS отключён из-за просрочки оплаты Vultr |

## Зачем

- Не повторять ошибки
- Корневые причины оседают в архитектуре (через уроки → правила в [25-integrations/](../25-integrations/), задачи в [roadmap](../65-roadmap/current.md))
- Учебная база при росте команды
