---
title: System — текущая система
status: active
last_updated: 2026-04-28
---

# 40-system — Текущая система

Описание уже работающей системы учёта: фронтенд, API, БД, физический формат данных, деплой.

## Что внутри

- [architecture.md](architecture.md) — общая схема компонентов
- [frontend.md](frontend.md) — `trade_app_v3.jsx`, single-file React SPA
- [api.md](api.md) — Express API на `:3001`
- [database/](database/) — PostgreSQL, KV-схема `app_storage`
- [data-formats/](data-formats/) — физический формат сущностей в JSON
- [deployment.md](deployment.md) — как выкатывается фронт

## Чего здесь нет

- **Инфраструктура** (VPS, Nginx, PM2) — в [`70-operations/`](../70-operations/).
- **Логическая модель данных** (что такое сущность как абстракция) — в [`20-domain/data-model/`](../20-domain/data-model/).
- **Бизнес-правила** — в [`20-domain/`](../20-domain/).
