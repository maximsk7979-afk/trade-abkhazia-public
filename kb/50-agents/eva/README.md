---
title: Eva — ИИ-агент
status: design
last_updated: 2026-05-22
related_decisions:
  - ../../90-decisions/ADR-005-eva-replaces-bot.md
  - ../../90-decisions/ADR-013-eva-agent-architecture.md
---

# Eva — ИИ-агент

Основной интерфейс работы с системой для команды, клиентов и контрагентов. Замещает старый `trade-bot`.

## Текущий статус

🛠 **В проектировании**. Архитектура зафиксирована в [ADR-013](../../90-decisions/ADR-013-eva-agent-architecture.md) (2026-05-22): агентный рантайм, четырёхслойная память, навыки, CRM-функции, хранилище в Postgres. Дальше — Postgres под Еву, затем поля профиля клиента и сценарий онбординга.

## Что внутри

- **vision.md** — видение, характер, принципы (фундамент системного промпта) ✅
- **architecture.md** — компоненты, потоки, навыки ✅
- **memory-system.md** — четыре слоя памяти + схема БД ✅
- **tools/** — контракты инструментов ⏳ в проектировании
- **prompts/** — промпты по ролям (system-base + role-specific) ⏳
- **scenarios/** — бизнес-сценарии: [онбординг клиента](scenarios/client-onboarding.md) ✅, приём заявки ⏳, напоминания ⏳
- **deployment.md** — как Ева запускается, где живёт ⏳ заглушка

## Что Ева делает (план)

- Распознаёт грузинские накладные по фото (заместит старый `bot_expeditor`)
- Работает с Нелли в роли оп-менеджера
- Работает с клиентами в Telegram (продажи, статусы заказов)
- Работает с Лиёй (расчёты по рейсам)
- Работает с Махо (закупки, цены)
- Работает с Максимом (отчётность, контроль)

## Технические основы

- LLM: Claude Sonnet 4.6 через **AWS Bedrock** ([25-integrations/aws-bedrock.md](../../25-integrations/aws-bedrock.md))
- Канал: Telegram Bot API на унаследованном токене ([25-integrations/telegram.md](../../25-integrations/telegram.md))
- Память: PostgreSQL — четыре слоя, схема в [memory-system.md](memory-system.md). Поднимается под Еву отдельно (ADR-013 п. 4)
- Хост: тот же VPS (см. [70-operations/infrastructure.md](../../70-operations/infrastructure.md))

## Принципы (из решения)

См. [ADR-005](../../90-decisions/ADR-005-eva-replaces-bot.md):
- Ева **замещает** старого бота, не работает параллельно
- Старый бот выключен 2026-04-27
- Telegram-токен сохраняется за Евой
- OCR накладных — переписывается с нуля, не из старого бота. База знаний — из обученного Claude у Максима ([GAP-003](../../00-meta/gaps.md))

## Связанные документы

- [INC-001](../../75-incidents/INC-001-bot-anthropic-balance.md) — уроки от старого бота
- [25-integrations/](../../25-integrations/) — внешние сервисы Евы
