---
title: Карта базы знаний
status: active
last_updated: 2026-05-30
---

# Карта базы знаний (INDEX)

Это карта всей базы знаний `kb/`. Здесь — что в каком слое лежит.

Для **рекомендованной последовательности чтения** см. [`00-meta/onboarding.md`](00-meta/onboarding.md).

Для **словаря терминов** см. [`00-meta/glossary.md`](00-meta/glossary.md).

---

## Слои

### 00-meta/ — Правила работы с базой
Шаблоны, конвенции, словарь, протоколы. Читать первыми.

- [conventions.md](00-meta/conventions.md) — именование файлов, front matter
- [style-guide.md](00-meta/style-guide.md) — как писать документацию
- [maintenance.md](00-meta/maintenance.md) — обязанности CC по ведению базы
- [code-sync-protocol.md](00-meta/code-sync-protocol.md) — сверка документации с кодом
- [gaps.md](00-meta/gaps.md) — очередь "белых пятен"
- [glossary.md](00-meta/glossary.md) — словарь терминов
- [onboarding.md](00-meta/onboarding.md) — рекомендованный порядок чтения
- [architect-onboarding.md](00-meta/architect-onboarding.md) — протокол для Архитектора в начале каждого чата
- [templates/](00-meta/templates/) — шаблоны новых файлов (включая `architect-task.md` — шаблон ТЗ)

### 10-business/ — Бизнес как целое
Бизнес-модель, операционная модель, стратегия.

- [overview.md](10-business/overview.md) — чем занимаемся, кто клиенты
- operating-model.md — как работает операционка сейчас (заглушка)
- domains/vegetables/ — текущий бизнес овощей и фруктов
- strategy/ — стратегические принципы (без конкретных цифр — они в БД)

### 15-team/ — Конкретные люди
Заметки о людях. Не дублирует БД (роли, контакты живут в `trade-cat-staff`).

- [maxim.md](15-team/maxim.md), [nelli.md](15-team/nelli.md), [maho.md](15-team/maho.md), [liya.md](15-team/liya.md)

### 20-domain/ — Бизнес-логика
Правила учёта, формы отчётности, логическая модель данных.

- data-model/ — логическая модель: что такое сущности, как связаны
- trips.md, sales.md, deliveries.md, pricing.md, ... — правила по доменам
- settlements/ — взаиморасчёты (клиенты, поставщики, Лия, контрагенты)
- invoices/ — накладные (двухуровневая база: `_common/` + `suppliers/<id>/`, см. ADR-011)
- financial-reporting/ — формы отчётов (P&L, акт сверки)

### 25-integrations/ — Внешние сервисы
Telegram, AWS Bedrock, Anthropic API, Google Sheets, Yandex Maps.

### 30-roles/ — Роли в бизнесе
Описание ролей как абстракций (что делает оп-менеджер, экспедитор и т.д.).

- as-is/ — состояние **до** Евы (заполняется из описаний с Архитектором); зафиксировано совмещение «менеджер+экспедитор»
- to-be/ — целевое состояние **с** Евой
  - [sales-manager/](30-roles/to-be/sales-manager/README.md) — роль + процессы [завод клиента](30-roles/to-be/sales-manager/processes/client-onboarding.md), [приём заявки](30-roles/to-be/sales-manager/processes/order-intake.md)
  - [expeditor/](30-roles/to-be/expeditor/README.md) — роль + процесс [отгрузка/доставка](30-roles/to-be/expeditor/processes/shipment.md) (2026-06-06)

### 35-security/ — Безопасность и права доступа
Матрица прав, политика доступа, классификация данных.

### 40-system/ — Текущая система
Frontend (`trade_app_v3.jsx`), API, БД, физический формат данных, деплой.

- [architecture.md](40-system/architecture.md), [frontend.md](40-system/frontend.md), [api.md](40-system/api.md), [deployment.md](40-system/deployment.md)
- [photo-infrastructure.md](40-system/photo-infrastructure.md) — фото-инфраструктура (draft, разбор по слоям с 2026-05-30)
- [mini-app.md](40-system/mini-app.md) — Telegram Mini App клиента: архитектура, эндпоинты, автообновление, деплой (2026-06-04)
- database/, data-formats/

### 50-agents/ — ИИ-агенты
- _shared/ — общее для всех агентов (память, инструменты, протоколы)
- eva/ — Ева (текущий проект)
- finance/ — будущий универсальный финагент

### 65-roadmap/ — План развития
- [current.md](65-roadmap/current.md) — что делаем сейчас и дальше

### 70-operations/ — Инфраструктура
- [infrastructure.md](70-operations/infrastructure.md) — VPS, Nginx, PM2, Postgres
- runbooks/ — пошаговые инструкции (deploy, add-new-role)
- monitoring.md — мониторинг (заглушка)

### 75-incidents/ — Инциденты
Журнал сбоев и уроков. См. [INC-001](75-incidents/INC-001-bot-anthropic-balance.md).

### 80-instructions/ — Сгенерированные инструкции
Вторичный слой: должностные инструкции, регламенты, инструкции для клиентов и контрагентов. Собирается из первоисточников.

- for-team/ — [sales-manager](80-instructions/for-team/sales-manager.md), [expeditor](80-instructions/for-team/expeditor.md), [eva-quickstart](80-instructions/for-team/eva-quickstart.md)

### 90-decisions/ — Журнал решений (ADR)
Каждое архитектурное решение зафиксировано отдельным файлом.

- [ADR-001](90-decisions/ADR-001-git-github.md) — Git+GitHub для хранения
- [ADR-002](90-decisions/ADR-002-modular-knowledge-base.md) — модульная база знаний
- [ADR-003](90-decisions/ADR-003-cc-maintenance-contract.md) — контракт CC
- [ADR-004](90-decisions/ADR-004-data-source-of-truth.md) — БД vs kb/ для данных
- [ADR-005](90-decisions/ADR-005-eva-replaces-bot.md) — Ева замещает бота
- [ADR-006](90-decisions/ADR-006-as-is-to-be-principle.md) — принцип AS-IS/TO-BE
- [ADR-007](90-decisions/ADR-007-sales-canonical-model.md) — канонические форматы продаж
- [ADR-008](90-decisions/ADR-008-ai-comms-architecture.md) — архитектура коммуникации между AI-агентами
- [ADR-009](90-decisions/ADR-009-multitenancy-projects.md) — мультипроектная архитектура (Project + projectId сквозной)
- [ADR-010](90-decisions/ADR-010-invoice-entity.md) — сущность Invoice (накладная поставщика)
- [ADR-011](90-decisions/ADR-011-eva-supplier-segmentation.md) — сегментация знаний Евы по поставщикам
- [ADR-012](90-decisions/ADR-012-eva-self-learning-plan.md) — поэтапный план самообучения Евы
- [ADR-013](90-decisions/ADR-013-eva-agent-architecture.md) — архитектура Евы (агентный рантайм, четырёхслойная память, навыки, CRM)
- [ADR-014](90-decisions/ADR-014-client-project-profile.md) — проектный профиль клиента, справочник сегментов, проектная привязка товара
- [ADR-015](90-decisions/ADR-015-role-process-eva-structure.md) — структура роль ↔ процесс ↔ сценарий Евы ↔ инструкция; диагностика влияния изменений
- [ADR-016](90-decisions/ADR-016-photo-infrastructure.md) — фото-инфраструктура: единый пайплайн (5 слоёв) + 4 независимых базы + расширяемость
- [ADR-017](90-decisions/ADR-017-eva-memory-architecture.md) — архитектура памяти Евы (слои A/B/C/D)
- [ADR-018](90-decisions/ADR-018-eva-universal-assistant.md) — Ева как универсальный помощник + изоляция данных клиента
- [ADR-019](90-decisions/ADR-019-eva-mini-app.md) — Telegram Mini App как поверхность структурных действий Евы
- [ADR-020](90-decisions/ADR-020-order-intake-banana-pricing.md) — приём заявок под рейс + банановое ценообразование + границы полномочий
- [ADR-021](90-decisions/ADR-021-daily-mixed-trips-day-order.md) — ежедневные смешанные рейсы + заказ на день + единый каталог по проектам клиента

### 95-changelog/ — История изменений
- [2026-Q2.md](95-changelog/2026-Q2.md) — текущий квартал

### 99-archive/ — Устаревшее
- old-context-md/ — копия исходного `CONTEXT.md` для разбора по новой структуре
