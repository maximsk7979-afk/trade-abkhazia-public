---
title: Anthropic API
status: suspended
last_updated: 2026-04-28
references:
  - ./aws-bedrock.md
referenced_by:
  - ../75-incidents/INC-001-bot-anthropic-balance.md
---

# Anthropic API

## Назначение

**Резервный канал** к Claude (помимо AWS Bedrock). На сейчас — не используется.

## Параметры подключения

- **API Key**: `ANTHROPIC_API_KEY` в `.env`
- **SDK**: `@anthropic-ai/sdk`
- **Endpoint**: `https://api.anthropic.com/v1/messages`

## Текущий статус

- Аккаунт **с пустым балансом** (получены ошибки "credit balance too low" в апреле — см. [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md))
- Старый бот пытался использовать этот канал параллельно с Bedrock — отсюда падения
- В Еве по умолчанию **не активируется**. Решение: один основной канал (Bedrock).

## Лимиты

- pay-per-use
- Лимиты пер-минут зависят от tier аккаунта

## Кто оплачивает

(пока нет — баланс пуст)

## Что делать при отказе Bedrock — fallback

1. Пополнить баланс Anthropic
2. Активировать переключение в коде Евы (если есть feature flag)
3. Проверить, что ID моделей актуальны (формат отличается от Bedrock — без префикса `eu.`, например `claude-sonnet-4-6`)

## Уроки

См. [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md):
- Двойной канал без выбора одного основного приводит к скрытому фейлу
- Биллинг должен мониториться (алерт при низком балансе)

## Связанные документы

- [aws-bedrock.md](aws-bedrock.md) — основной канал
- [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md)
