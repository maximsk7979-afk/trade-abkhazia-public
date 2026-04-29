---
title: Decisions — журнал решений
status: active
last_updated: 2026-04-28
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
