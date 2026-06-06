---
title: "INC-001: Бот падал из-за пустого Anthropic API balance"
date: 2026-04-26
severity: medium
status: resolved
references:
  - ../25-integrations/anthropic-api.md
  - ../25-integrations/aws-bedrock.md
  - ../25-integrations/telegram.md
referenced_by: []
---

# INC-001: Бот падал из-за пустого Anthropic API balance

## Что произошло

Старый `trade-bot` за 32 дня перезапускался **27 раз**. В логах — три категории ошибок:

### Категория 1 — Anthropic API: пустой баланс
```
Bot error: 400 ... "Your credit balance is too low to access the Anthropic API.
Please go to Plans & Billing to upgrade or purchase credits."
```

### Категория 2 — AWS Bedrock: неправильный ID модели
```
Bot error: Invocation of model ID anthropic.claude-sonnet-4-6 with on-demand throughput
isn't supported. Retry your request with the ID or ARN of an inference profile that contains this model.
Bot error: The provided model identifier is invalid.
```
Использовался ID `anthropic.claude-sonnet-4-6`, без префикса `eu.`. Bedrock в `eu-north-1` требует **`eu.anthropic.claude-sonnet-4-6`**.

### Категория 3 — Telegram polling: сетевые сбои
```
[polling_error] EFATAL: read ETIMEDOUT
[polling_error] EFATAL: AggregateError
[polling_error] EFATAL: read ECONNRESET
[polling_error] ETELEGRAM: 429 Too Many Requests: retry after 5
[polling_error] ETELEGRAM: 502 Bad Gateway
```
Это нормальные ошибки: разрывы сети, rate limits, временные 502 Telegram. **Должны глотаться** библиотекой и переподключаться.

## Когда обнаружено

2026-04-26, во время первичного аудита системы перед стартом проекта Евы.

## Затронутые компоненты

- `trade-bot` (PM2-процесс) — старый бот экспедитора
- Библиотека `node-telegram-bot-api`
- AWS Bedrock в `eu-north-1`
- Anthropic API

## Корневая причина

| Категория | Причина |
|---|---|
| 1 | Закончился баланс на anthropic.com. Нет мониторинга биллинга |
| 2 | Опечатка в коде бота (model ID без префикса `eu.`) |
| 3 | Библиотека логирует штатные сетевые сбои как fatal; PM2 рестартует процесс |

## Что сделали

- 2026-04-27: Бот выключен (`pm2 stop trade-bot`, `pm2 save`).
- В рамках замещения Евой бот **не чинится** (см. [ADR-005](../90-decisions/ADR-005-eva-replaces-bot.md)).
- Уроки зафиксированы и переносятся в проектирование Евы.

## Уроки для Евы

1. **Один основной канал к LLM.** Использовать AWS Bedrock с префиксом `eu.`, валидировать model ID при старте.
2. **Биллинг-мониторинг.** Алерт при низком балансе любого внешнего API. Особенно AWS Bedrock — нужно настроить алерт на бюджет. ✅ **Реализовано 2026-06-06 (GAP-026 A):** app-side учёт расхода Bedrock в Еве (`usage-tracker.js`) + алерт владельцу в Telegram при пересечении порога `EVA_BUDGET_USD`. AWS Budgets (нативный) — рекомендован как второй слой.
3. **Сетевые ошибки Telegram не должны валить процесс.** Рассмотреть webhook вместо polling. Если polling — корректно обрабатывать `ETIMEDOUT`, `ECONNRESET`, `429`, `502` без рестарта.
4. **ID моделей** — константа в `.env` или конфиге, валидация при старте процесса.

## Связанные документы

- [25-integrations/anthropic-api.md](../25-integrations/anthropic-api.md) — биллинг и текущий статус (suspended)
- [25-integrations/aws-bedrock.md](../25-integrations/aws-bedrock.md) — правило префикса `eu.`
- [25-integrations/telegram.md](../25-integrations/telegram.md) — обработка polling errors
- [50-agents/eva/](../50-agents/eva/) — уроки учитываются при проектировании Евы
- [65-roadmap/current.md](../65-roadmap/current.md) — задача "Настроить алерт на бюджет AWS Bedrock"
