---
title: Operations — инфраструктура и операции
status: active
last_updated: 2026-04-28
---

# 70-operations — Инфраструктура и операции

Где система живёт, как поддерживается, как деплоится.

## Что внутри

- [infrastructure.md](infrastructure.md) — VPS, Nginx, PM2, PostgreSQL
- [runbooks/](runbooks/) — пошаговые инструкции (deploy-frontend, add-new-role, ...)
- monitoring.md — мониторинг (заглушка)

## Принцип

Этот слой отвечает на: **где** запущено и **как** поддерживается.

- В [40-system/](../40-system/) — **что** запущено (компоненты системы)
- В [50-agents/eva/](../50-agents/eva/) — **что** запущено (Ева)
- Здесь — **общая инфра** для всего этого

См. [ADR-002](../90-decisions/ADR-002-modular-knowledge-base.md), пункт о границе между `40-system/` и `70-operations/`.
