---
title: Integrations — внешние сервисы
status: active
last_updated: 2026-04-28
---

# 25-integrations — Внешние интеграции

Сторонние сервисы, от которых зависит наша система. Один сервис = один файл.

## Текущие интеграции

| Файл | Сервис | Статус |
|---|---|---|
| [telegram.md](telegram.md) | Telegram Bot API | active |
| [aws-bedrock.md](aws-bedrock.md) | AWS Bedrock (Claude) | active |
| [anthropic-api.md](anthropic-api.md) | Anthropic API (резерв) | suspended |
| [google-sheets.md](google-sheets.md) | Google Sheets | paused |
| [yandex-maps.md](yandex-maps.md) | Yandex Maps | passive |

## Шаблон файла

См. [00-meta/templates/integration-template.md](../00-meta/templates/integration-template.md).

## Принцип

- Здесь — **что** за сервис, **как** мы с ним связаны, **кто** платит, **что** делать при отказе
- В [40-system/](../40-system/) — какие компоненты системы используют интеграции
- В [50-agents/eva/tools/](../50-agents/eva/tools/) — как Ева использует их через свои инструменты
