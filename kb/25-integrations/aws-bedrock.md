---
title: AWS Bedrock
status: active
last_updated: 2026-04-28
references:
  - ../50-agents/eva/architecture.md
  - ../70-operations/infrastructure.md
referenced_by:
  - ../75-incidents/INC-001-bot-anthropic-balance.md
---

# AWS Bedrock

## Назначение

LLM-ядро Евы. Все запросы к Claude (Sonnet 4.6) идут через AWS Bedrock — это **основной канал**, не Anthropic API напрямую.

## Параметры подключения

- **Регион**: `eu-north-1`
- **Модель**: `eu.anthropic.claude-sonnet-4-6`
  - ⚠️ **Префикс `eu.` обязателен.** Без него Bedrock возвращает ошибку *"invalid model identifier"* (см. [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md))
- **Аутентификация**: AWS access key + secret. В `.env`: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.
- **SDK**: `@aws-sdk/client-bedrock-runtime`

## Лимиты

- pay-per-use (per-token billing)
- Текущие лимиты — стандартные AWS (десятки запросов/мин)

## Кто оплачивает

Максим, корпоративная карта на AWS аккаунте.

## Модели и маршрутизация (GAP-026)

- **Sonnet** (основная, торговля/работа): `eu.anthropic.claude-sonnet-4-6` — `claude.MODEL_SONNET`.
- **Haiku** (дешёвая, болтовня клиента): задаётся `EVA_HAIKU_MODEL` в `.env`. ⚠️ **Сейчас НЕ активна** — модель Haiku **не включена** в Bedrock этого аккаунта (любой id → `ValidationException`). Чтобы включить маршрутизацию: AWS Bedrock Console → **Model access** (регион eu-north-1) → разрешить Claude Haiku → взять рабочий **inference-profile id** (с префиксом `eu.`) → прописать в `EVA_HAIKU_MODEL` → `pm2 restart eva`. Маршрутизатор — `code/eva/src/router.js`.

## Мониторинг и бюджет

- **App-side алерт бюджета (реализован 2026-06-06, GAP-026 A):** Ева считает ~стоимость каждого хода (`code/eva/src/usage-tracker.js`), копит за месяц в KV `eva-usage-v1`; при 50/80/100/150% от `EVA_BUDGET_USD` (дефолт $200) шлёт владельцу алерт в Telegram. Закрывает урок INC-001 со стороны приложения.
- AWS Console → CloudWatch → Bedrock metrics — для детальной телеметрии.
- ⚠️ **AWS Budgets** (нативный алерт в консоли) — рекомендуется как **второй слой** к app-side (на случай, если Ева не работает). Настраивается в консоли (действие Максима).

## Что делать при отказе

1. Проверить статус AWS Bedrock в `eu-north-1`: https://health.aws.amazon.com/health/status
2. Проверить остаток на AWS-аккаунте (биллинг)
3. **Проверить корректность model ID** — `eu.anthropic.claude-sonnet-4-6` (с префиксом `eu.`)
4. **Fallback**: Anthropic API напрямую (см. [anthropic-api.md](anthropic-api.md)) — если активирован

## Правила использования

- В коде агента **не хардкодить** ID модели. Брать из `.env` или конфига, валидировать при старте.
- Региональные ограничения: модель работает в `eu-north-1`. На другие регионы — нужен другой ID.

## История

- В коде старого бота был случай неправильного ID без префикса `eu.` → ошибки "invalid model identifier"
- 2026-04-27: фиксируем правило префикса как обязательное при старте Евы

## Связанные документы

- [50-agents/eva/architecture.md](../50-agents/eva/architecture.md) — где Bedrock используется
- [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md) — урок про префикс и fallback
