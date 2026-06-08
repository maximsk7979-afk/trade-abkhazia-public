---
title: Decisions — журнал решений
status: active
last_updated: 2026-05-31
---

# 90-decisions — Журнал решений (ADR)

Все архитектурные решения проекта. Один файл = одно решение.

## Что такое ADR

**Architecture Decision Record** — стандартный формат записи. Содержит:
- Контекст (почему обсуждали)
- Решение (что приняли)
- Последствия (что хорошего, что плохого)
- Альтернативы, которые отклонили

Шаблон — [00-meta/templates/adr-template.md](../00-meta/templates/adr-template.md).

## Принципиальное

- **Не редактировать ADR после принятия.** Если решение поменялось — новый ADR со статусом `superseded` и ссылкой.
- **Все значимые решения** фиксируются как ADR. Это защита от "это уже обсуждали 10 чатов назад".
- **Имена файлов**: `ADR-NNN-короткое-имя.md`. Номер сквозной, не сбрасывается.

## Перечень

| № | Заголовок | Дата | Статус |
|---|---|---|---|
| [ADR-001](ADR-001-git-github.md) | Git+GitHub для хранения кода и знаний | 2026-04-27 | accepted |
| [ADR-002](ADR-002-modular-knowledge-base.md) | Модульная база знаний | 2026-04-27 | accepted |
| [ADR-003](ADR-003-cc-maintenance-contract.md) | Контракт CC по ведению базы | 2026-04-27 | accepted |
| [ADR-004](ADR-004-data-source-of-truth.md) | Источник истины: БД vs kb/ | 2026-04-27 | accepted |
| [ADR-005](ADR-005-eva-replaces-bot.md) | Ева замещает старого бота | 2026-04-27 | accepted |
| [ADR-006](ADR-006-as-is-to-be-principle.md) | Принцип AS-IS / TO-BE | 2026-04-27 | accepted |
| [ADR-007](ADR-007-sales-canonical-model.md) | Канонические форматы продаж | 2026-04-28 | accepted |
| [ADR-008](ADR-008-ai-comms-architecture.md) | Архитектура коммуникации между AI-агентами через GitHub | 2026-04-30 | accepted |
| [ADR-009](ADR-009-multitenancy-projects.md) | Мультипроектность (projectId сквозной) | 2026-05-19 | accepted |
| [ADR-010](ADR-010-invoice-entity.md) | Сущность Invoice (входящие накладные) | 2026-05-19 | accepted |
| [ADR-011](ADR-011-eva-supplier-segmentation.md) | Сегментация базы распознавания по поставщикам | 2026-05-19 | accepted |
| [ADR-012](ADR-012-eva-self-learning-plan.md) | Поэтапный план обучения Евы (база + модерация человеком) | 2026-05-19 | accepted |
| [ADR-013](ADR-013-eva-agent-architecture.md) | Архитектура агента Евы (4 слоя памяти) | 2026-05-22 | accepted |
| [ADR-014](ADR-014-client-project-profile.md) | Проектный профиль клиента | 2026-05-22 | accepted |
| [ADR-015](ADR-015-role-process-eva-structure.md) | Структура роль ↔ процесс ↔ Ева ↔ инструкция | 2026-05-22 | accepted |
| [ADR-016](ADR-016-photo-infrastructure.md) | Фото-инфраструктура (5 слоёв, 4 базы) | 2026-05-30 | accepted |
| [ADR-017](ADR-017-eva-memory-architecture.md) | Архитектура памяти Евы — финал слоёв A–D | 2026-05-31 | accepted |
| [ADR-018](ADR-018-eva-universal-assistant.md) | Ева как универсальный помощник + изоляция данных клиента | 2026-05-31 | accepted |
| [ADR-019](ADR-019-eva-mini-app.md) | Telegram Mini App как поверхность структурных действий | 2026-06-04 | accepted |
| [ADR-020](ADR-020-order-intake-banana-pricing.md) | Приём заявок под рейс + банановое ценообразование | 2026-06-04 | accepted |
| [ADR-021](ADR-021-daily-mixed-trips-day-order.md) | Ежедневные смешанные рейсы + заказ на день | 2026-06-06 | accepted |
| [ADR-022](ADR-022-role-model-rbac.md) | Ролевая модель и RBAC (жёсткая привязка процессов к ролям) | 2026-06-09 | accepted |
