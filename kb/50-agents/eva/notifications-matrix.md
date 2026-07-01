---
title: "Матрица уведомлений: Операция → уведомление → кому"
status: draft
last_updated: 2026-06-30
related_decisions:
  - ../../90-decisions/ADR-022-role-model-rbac.md
  - ../../90-decisions/ADR-025-single-domain-layer.md
references:
  - ../../35-security/role-catalog.md
---

# Матрица уведомлений Евы/системы

> **Назначение.** Единый реестр всех исходящих уведомлений сотрудникам: какая операция,
> какое уведомление, кому, по какому каналу, с какой важностью. Это спецификация для
> `notify-core` и секция «Роли и доступ» по [ADR-022](../../90-decisions/ADR-022-role-model-rbac.md).
> Fail-closed: уведомление без строки в этой матрице не отправляется.
>
> **Статус — DRAFT.** Колонки «СЕЙЧАС» = аудит на 2026-06-30 (две разведки по коду).
> Колонки «ЦЕЛЬ» = предложение, ждёт сверки владельцем. После сверки — статус `active`.
>
> **Очерёдность ([ADR-026](../../90-decisions/ADR-026-notifications-unified-layer.md)):** код-реализация
> — ПОСЛЕ сборки финблока (Ф-5). Финблок вводит кошельки/`fin_director`/вторую ногу денег и
> переписывает D-блок (и часть A/C); сверку открытых вопросов ведём по ходу финблока.

## Источники уведомлений (как есть)

Сейчас уведомления шлют **два независимых отправителя** с дублирующим резолвом роль→chat_id
(нарушение [ADR-025](../../90-decisions/ADR-025-single-domain-layer.md)):

1. **trade-api** (`code/trade-api/notify.js` + `server.js`) — на бизнес-операции.
   ⚠️ Висят ТОЛЬКО на роутах `/api/mini/*` (мини-апп). Веб (`/api/v2/*`) **не уведомляет**.
2. **Процесс Евы** (`code/eva/src/…`) — кроны, инструменты, онбординг, оркестратор банана.

Условные обозначения важности (ЦЕЛЬ):
`crit` — сразу в ЛС (сбой/конфликт/деньги-аномалия); `op` — операционный фид (в роль-ЛС, кандидат в дайджест); `info` — фоново (дайджест).

---

## A. Закупка / рейс

| # | Операция | Триггер (сейчас) | Кому сейчас | Веб? | Чувств. | ЦЕЛЬ: кому | Важн. |
|---|---|---|---|---|---|---|---|
| A1 | Оплата поставщику | `/api/mini/buyer/supplier-op` | owner, исполнитель, ops_manager (**ГД скрыт**) | ✗ | закуп. цены | owner, buyer, ops_manager | op |
| A2 | Местная закупка | `/api/mini/purchase` | owner, исполнитель, ops_manager (ГД скрыт) | ✗ | закуп. цены | owner, buyer, ops_manager | op |
| A3 | Расходы рейса обновлены | `/api/mini/exp/save` | owner, исполнитель | ✗ | закуп. | owner, buyer | info |
| A4 | Рейс завершён (в пути) | `/api/mini/buyer/complete-trip` + Eva `complete_trip` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, gen_director, ops_manager | op |
| A5 | Оркестратор: kickoff закупки | cron `procurement-tick` (ВЫКЛ) | gen_director | n/a | — | gen_director | op |
| A6 | Оркестратор: заказ поставщику | событие PLACE_ORDER | supplier | n/a | — | supplier | op |
| A7 | Оркестратор: поставщик отказал | SUPPLIER_REJECTED | gen_director | n/a | — | gen_director | crit |
| A8 | Оркестратор: лимит разобран | LIMIT_REACHED | gen_director | n/a | — | gen_director | op |
| A9 | Оркестратор: назначить водителя | ASSIGN_DRIVER | buyer | n/a | — | buyer | op |
| A10 | Оркестратор: запрос накладной | REQUEST_INVOICE | supplier | n/a | — | supplier | op |
| A11 | Оркестратор: подтвердить проезд | CONFIRM_PROEZD | gen_director | n/a | — | gen_director | crit |
| A12 | Оркестратор: напоминание о проезде | PROEZD_REMIND (за час) | gen_director | n/a | — | gen_director | crit |

## B. Склад

| # | Операция | Триггер (сейчас) | Кому сейчас | Веб? | Чувств. | ЦЕЛЬ: кому | Важн. |
|---|---|---|---|---|---|---|---|
| B1 | Приёмка на склад | `/api/mini/wh/accept` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, warehouse, ops_manager, gen_director | op |
| B2 | Приёмка возврата | `/api/mini/wh/accept-return` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, warehouse, ops_manager | op |
| B3 | Сверка остатков проведена | `/api/mini/inventory` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, ops_manager, gen_director | op |
| B4 | Напоминание сверить остатки | cron ~19:00 | warehouse | n/a | — | warehouse | op |
| B5 | Утренняя сводка сверки | cron ~09:00 | owner, gen_director | n/a | — | owner, gen_director | info |

## C. Продажи / заявки / доставка

| # | Операция | Триггер (сейчас) | Кому сейчас | Веб? | Чувств. | ЦЕЛЬ: кому | Важн. |
|---|---|---|---|---|---|---|---|
| C1 | Новая/изменённая заявка | `/api/mini/order` POST | owner, менеджер клиента | ✗ | — | owner, sales_manager(свой), ops_manager | op |
| C2 | Заявка отменена | `/api/mini/order` DELETE | owner, менеджер | ✗ | — | owner, sales_manager(свой), ops_manager | op |
| C3 | Выезд доставки | `/api/mini/dlv/depart` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, expeditor, ops_manager, gen_director | op |
| C4 | Отгрузка по точке | `/api/mini/dlv/ship` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, expeditor, ops_manager, gen_director | op |
| C5 | Доставка завершена | `/api/mini/dlv/complete` | owner, исполнитель, ops_manager, ГД | ✗ | — | owner, ops_manager, gen_director | op |
| C6 | Отмена отгрузки | `/api/mini/dlv/ship-cancel` | исполнитель + owner, ops_manager, ГД | ✗ | — | исполнитель, owner, ops_manager, gen_director | op |
| C7 | Продажа со склада | `/api/mini/sale-direct/save` | исполнитель + owner, ops_manager, ГД | ✗ | — | исполнитель, owner, ops_manager, gen_director | op |
| C8 | Отмена продажи со склада | `/api/mini/sale-direct/cancel` | исполнитель + owner, ops_manager, ГД | ✗ | — | исполнитель, owner, ops_manager, gen_director | op |

## D. Деньги / расчёты

| # | Операция | Триггер (сейчас) | Кому сейчас | Веб? | Чувств. | ЦЕЛЬ: кому | Важн. |
|---|---|---|---|---|---|---|---|
| D1 | Оплата/операция клиента | `/api/mini/clients/op` | исполнитель + owner, ops_manager, ГД | ✗ | — | исполнитель, owner, fin_director, ops_manager | op |
| D2 | Бюджет Bedrock (Ева) 50/80/100/150% | после хода Claude | owner | n/a | — | owner | crit (≥100), info (<100) |

## E. Клиенты / онбординг

| # | Операция | Триггер (сейчас) | Кому сейчас | Веб? | Чувств. | ЦЕЛЬ: кому | Важн. |
|---|---|---|---|---|---|---|---|
| E1 | Создан клиент | Eva `create_client` | owner, ops_manager | n/a | — | owner, ops_manager, sales_manager(назнач.) | info |
| E2 | Клиент привязал Telegram | Eva `/start` | менеджер + owner | n/a | — | sales_manager(свой), owner | info |
| E3 | Токен приглашения истёк | Eva `/start` | менеджер | n/a | — | sales_manager(свой) | op |
| E4 | Конфликт привязки клиента | Eva `/start` | менеджер | n/a | — | sales_manager(свой), owner | crit |
| E5 | Смена якоря цены банана → переоценка | Eva `manage_banana_pricing` | менеджеры с фикс-ценой | n/a | цены | sales_manager(с фикс-клиентами) | op |

## F. Система / служебное

| # | Операция | Триггер (сейчас) | Кому сейчас | Веб? | Чувств. | ЦЕЛЬ: кому | Важн. |
|---|---|---|---|---|---|---|---|
| F1 | Health-алерт Евы (сбой) | cron каждые 5 мин | owner (NOTIFY_CHAT_IDS) | n/a | — | owner | crit |
| F2 | Ева восстановилась | смена состояния | owner | n/a | — | owner | info |
| F3 | Сообщение от перевозчика (дубль) | вход. сообщение `contragent` | owner (весь текст) | n/a | — | owner, ops_manager | op |

---

## Сводные проблемы аудита (донастройка)

1. **Веб не уведомляет ничего** (колонка «Веб? ✗» во всех A–D). После Фазы 1 — уведомляет через сервисный слой.
2. **`active`-фильтр непоследователен**: есть у owner/warehouse/ops, **нет** у buyer/sales_manager/supplier → деактивированный получает пинги. Цель — фильтр в одном месте (`notify-core`).
3. **Резолв роль→chat_id скопирован** в `notify.js` и `eva/message-log.js` → свести в `notify-core`.
4. **Владельца заваливает**: owner есть почти в каждой строке. Снять важностью (`crit`/`op`/`info`) + дайджест (Фаза 2).
5. **Нет дедупа** (кроме health 1/час) → ключ `(kind, refId)` в `notify-core`.
6. **ГД-чувствительность**: A1/A2 (закупочные цены) — ГД скрыт, сохранить fail-closed.

## Открытые вопросы к владельцу (на сверку)

- C1/C2: добавить `ops_manager` в фид заявок? (сейчас нет)
- D1: вместо ops+ГД направить клиентские деньги на `fin_director`? (роль появилась)
- E1: добавлять назначенного `sales_manager` в уведомление о создании клиента?
- Какие строки `info` уводим в утренний/вечерний дайджест (Фаза 2)?
