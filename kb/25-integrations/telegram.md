---
title: Telegram Bot API
status: active
last_updated: 2026-06-06
references:
  - ../50-agents/eva/deployment.md
  - ../70-operations/infrastructure.md
  - ../40-system/photo-infrastructure.md
referenced_by: []
---

# Telegram Bot API

## Назначение

Канал общения **Евы с пользователями** — экспедиторы, Нелли, клиенты, Лия, Максим. Долгосрочно — основной интерфейс работы с системой для команды и контрагентов.

## Параметры подключения

- **Token**: хранится в `.env` на VPS как `TELEGRAM_BOT_TOKEN`. Реальное значение — в локальном `~/secrets-trade/credentials.md`.
- **Бот**: тот же, что использовал `trade-bot` (`bot_expeditor.js`). Бот отключён, токен сохранён за Евой.
  - **Отображаемое имя** (с 2026-06-04): `Ева - Trade Abkhazia`.
  - **Username**: `@trade_abkhazia_sales_bot`. Используется в deep-link приглашениях онбординга (`https://t.me/trade_abkhazia_sales_bot?start=onb_<token>`) и как точка входа мини-аппа ([ADR-019](../90-decisions/ADR-019-eva-mini-app.md)).
- **Текущий метод**: long-poll (унаследовано от старого бота).
- **Перспектива**: рассмотреть webhook вместо polling (стабильнее).

## Лимиты

- Telegram Bot API — бесплатный.
- Лимит на отправку сообщений: ~30 сообщений/секунду на одного бота, 1 сообщение/секунду в один чат.

## Кто оплачивает

Не оплачивается (бесплатный API).

## Уведомления системы

Переменная `NOTIFY_CHAT_IDS` в `.env`:
```
NOTIFY_CHAT_IDS=118206343,1719753990
```

- `118206343` — Максим
- `1719753990` — Нелли (@Nelli2023N)

Используется при создании нового клиента, других системных событиях.

## Доступ к Еве — белый список (`ALLOWED_CHAT_IDS`)

Переменная `ALLOWED_CHAT_IDS` в `.env` Евы (`/var/www/eva/.env`) — кто может **вести диалог** с Евой (онбординг, команды). Гейт в `src/auth.js` (Ит.1–3; снять при открытии клиентского чата — GAP-026). **Активация клиента по deep-link** (`/start onb_<token>`) и **заказы через мини-апп** (`initData`) идут **мимо** этого списка — клиента в whitelist добавлять не нужно.

Подключённые сотрудники (с 2026-06-06):
```
ALLOWED_CHAT_IDS=118206343,2007503947,1246188432,5463278894,1719753990,479463516
```
- `118206343` — Максим (С-001, owner)
- `2007503947` — Иван (С-004, sales_manager/gen_director)
- `1246188432` — Алексей (С-006, sales_manager)
- `5463278894` — Саид (С-005, sales_manager/warehouse; телефон не заполнен)
- `1719753990` — Нелли (С-002, ops_manager)
- `479463516` — Наталья (С-007, fin_director/ops_manager)

При добавлении сотрудника: завести карточку в `trade-cat-staff` (с `telegramChatId` + ролью `sales_manager` + `phone`), добавить его chat_id в `ALLOWED_CHAT_IDS`, `pm2 restart eva`. chat_id узнать: сотрудник пишет боту `/start` → в логах Евы (`/var/log/eva-out-2.log`) виден `unauthorized chatId=...`, либо @userinfobot.

## Приём фото от пользователей

Помимо текста и команд, Ева принимает фото — как `msg.photo` (сжатое из галереи/камеры) и как `msg.document` с `mime ~ "image/*"` (без сжатия, для отгрузочных накладных). Альбомы (`media_group_id`) буферизуются ~700 мс. Лимит скачивания через Bot API — 20 МБ.

Слой приёма (`code/eva/src/photo/raw-photo.js` + `album-buffer.js`) — слой (а) фото-инфраструктуры ([ADR-016](../../90-decisions/ADR-016-photo-infrastructure.md), [40-system/photo-infrastructure.md](../../40-system/photo-infrastructure.md)). После приёма фото попадает в диспетчер контекста и handler нужной базы.

## Что делать при отказе

1. Проверить статус Telegram Bot API: https://api.telegram.org/
2. Проверить, не заблокировали ли бота за спам / превышение лимитов
3. Проверить, что токен не отозван (через @BotFather)
4. Перезапустить процесс агента (`pm2 restart eva`)

## История

- ~март 2026: создан старый бот `trade-bot` на этом токене
- 2026-04-27: `trade-bot` отключён, токен сохранён за Евой
- 2026-06-04: задано отображаемое имя `Ева - Trade Abkhazia` (@BotFather `/setname`); username бота — `@trade_abkhazia_sales_bot`
- 2026-04-26..27: в логах старого бота — типичные сетевые ошибки `EFATAL: read ETIMEDOUT`, `ECONNRESET`, `ETELEGRAM: 429`, `502 Bad Gateway`. Это штатные ошибки сети, должны корректно обрабатываться (см. [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md))

## Связанные документы

- [50-agents/eva/deployment.md](../50-agents/eva/deployment.md)
- [INC-001](../75-incidents/INC-001-bot-anthropic-balance.md) — урок о polling errors
