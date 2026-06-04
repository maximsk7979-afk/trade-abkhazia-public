---
title: Roles — роли в бизнесе
status: active
last_updated: 2026-04-28
related_decisions:
  - ../90-decisions/ADR-006-as-is-to-be-principle.md
---

# 30-roles — Роли в бизнесе

Описание ролей как **абстракций** — что делает функция (а не конкретный человек). Конкретные люди — в [`15-team/`](../15-team/).

## Принцип AS-IS / TO-BE

Этот слой разделён на два состояния:

- **as-is/** — состояние **до** Евы, как реально работает бизнес сейчас (с Excel у Нелли)
- **to-be/** — целевое состояние **с** Евой, после переопределения функций

См. [ADR-006](../90-decisions/ADR-006-as-is-to-be-principle.md).

## Карта трансформации

| Роль | AS-IS | TO-BE | Миграция | Статус |
|---|---|---|---|---|
| owner (Максим) | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| ops-manager (Нелли) | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| buyer (Махо) | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| sales-manager | ⏳ ждём описания | 🛠 [to-be/sales-manager](to-be/sales-manager/README.md) — роль + процессы [завод клиента](to-be/sales-manager/processes/client-onboarding.md), [приём заявки](to-be/sales-manager/processes/order-intake.md) | — | as-is активна, to-be в работе |
| warehouse | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| accountant | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| driver | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| client | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| partner (Лия) | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| contractor | ⏳ ждём описания | ⏳ проектируем | — | as-is активна |
| **expeditor** | — *нет сейчас* | 🆕 запланирована | — | новая роль |

## Ожидающее наполнение

Максим передаст AS-IS-описания ролей из своего чата с Архитектором — задача в [roadmap](../65-roadmap/current.md). После получения CC уложит их в [`as-is/`](as-is/).

## Алгоритм добавления новой роли

См. [`70-operations/runbooks/add-new-role.md`](../70-operations/runbooks/add-new-role.md).

## Шаблон файла роли

См. [`00-meta/templates/role-template.md`](../00-meta/templates/role-template.md).
