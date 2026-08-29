---
title: Протокол сверки документации с кодом
status: active
last_updated: 2026-04-28
references:
  - maintenance.md
  - gaps.md
---

# Протокол сверки документации с кодом

Реализация Обязательства 3 из [контракта CC](maintenance.md).

## Зачем

Документация в `kb/` отражает то, как **мы думаем** про систему. Код — то, как система **реально работает**. Эти две картины могут расходиться. Протокол — регулярная проверка, что они совпадают.

## Источники истины кода

**Источник истины — репозиторий** (с 2026-05-21, [ADR-001](../90-decisions/ADR-001-git-github.md)).
На VPS лежит задеплоенный артефакт: сверять надо с репо, а правки вносить в репо и
выкатывать через `./scripts/deploy.sh`, а не править файлы на боевом.

| Что | В репозитории (истина) | На боевом VPS (артефакт) |
|---|---|---|
| Frontend | `code/trade_app_v3.jsx` + `code/trade_app_calc.mjs` | `/var/www/trade/frontend/src/App.jsx`, `calc.mjs` |
| API | `code/trade-api/*.js` (`server.js` и сервисы) | `/var/www/trade/*.js` |
| Общие ядра (ADR-025) | `code/trade-api/*.cjs` (`trip-cost-core`, `purchase-cost-core`, `cash-core`) | и в `/var/www/trade/`, и в `frontend/src/trade-api/` |
| Ева | `code/eva/src/` | `/var/www/eva/src/` |
| Мини-апп | `code/mini-app/` | `/var/www/trade/public/mini/` |
| Розничный мини-апп (прототип) | `code/retail-app/` | `/var/www/trade/public/retail/` |
| Бот (отключён, как референс) | `code/bot_expeditor_v10.js`, `v9` | `/var/www/trade/bot_expeditor.js` |
| База данных | — | PostgreSQL `trade_db`, таблица `app_storage` |
| Конфиг VPS | — | Nginx, PM2, `.env` |

Рабочая копия репозитория — на станции: `/home/max/trade_app_repo/`
(см. [workstation-rebuild.md](../70-operations/runbooks/workstation-rebuild.md)).

## Триггеры сверки

### Перед фиксацией архитектурного решения
**Когда**: мы обсуждаем новое решение, которое касается уже работающей части системы.
**Что проверить**: текущее поведение кода в этой области. Если решение конфликтует — поднять флаг.

### Перед началом большой темы
**Когда**: стартуем новую роль / новый сценарий Евы / изменение БД.
**Что проверить**: всё, что в коде касается смежных областей. Чтобы не было сюрпризов.

### Плановая ревизия — раз в месяц
**Когда**: первая неделя каждого месяца.
**Что проверить**: целиком обходим список модулей, ищем расхождения.

## Объекты сверки (чек-лист)

### Frontend (`trade_app_v3.jsx`)
- [ ] Список SK_* констант (ключи storage) — сверить с [40-system/database/storage-keys.md](../40-system/database/storage-keys.md)
- [ ] DEF_EXP_TYPES — сверить с [25-integrations/](../25-integrations/) и [20-domain/settlements/](../20-domain/settlements/)
- [ ] DEF_CONTRAGENTS — есть в данных, описаны в [20-domain/settlements/contractors.md](../20-domain/settlements/contractors.md)
- [ ] Список ролей (ROLES) — сверить с [30-roles/](../30-roles/)
- [ ] Бизнес-логика расчётов (calcTripG, calcTripData, getPartnerData, и т.д.) — сверить с правилами в [20-domain/](../20-domain/)
- [ ] Seed-блоки (_SD3.._SD8) — фиксировать актуальную версию
- [ ] Версия фронта (v3, v4...) — отражена ли в [40-system/frontend.md](../40-system/frontend.md)

### API (`server.js`)
- [ ] Список эндпоинтов — сверить с [40-system/api.md](../40-system/api.md)
- [ ] Структура `app_storage` — сверить с [40-system/database/](../40-system/database/)

### БД (`app_storage`)
- [ ] Перечень ключей в БД — сверить с [40-system/database/storage-keys.md](../40-system/database/storage-keys.md)
- [ ] Размер каждого ключа (sanity-check, не выросло ли неожиданно)

### Инфраструктура
- [ ] PM2 процессы (`pm2 list`) — сверить с [70-operations/infrastructure.md](../70-operations/infrastructure.md)
- [ ] Nginx конфиг — сверить с [70-operations/infrastructure.md](../70-operations/infrastructure.md)

## Что делать при расхождении

1. **Записать находку** в [gaps.md](gaps.md) с пометкой даты обнаружения.
2. **Принять решение**:
   - Код прав → обновить документацию
   - Документация права → обновить код (это уже задача в [roadmap](../65-roadmap/current.md))
   - Спорно → обсудить с Максимом и Архитектором
3. **Зафиксировать решение** в ADR, если архитектурно значимо.

## Команды для сверки

```bash
# Список ключей storage в БД
sshpass -p '<VPS_PASSWORD>' ssh root@108.61.167.168 \
  "PGPASSWORD=<DB_PASSWORD> psql -h localhost -U trade_user -d trade_db \
   -c \"SELECT key, length(value) as bytes, updated_at FROM app_storage ORDER BY key;\""

# PM2 процессы
sshpass -p '<password>' ssh root@108.61.167.168 "pm2 list"

# Логи trade-api
sshpass -p '<password>' ssh root@108.61.167.168 \
  "pm2 logs trade-api --err --lines 100 --nostream"
```

(Реальные пароли — в локальном `~/secrets-trade/credentials.md`, не в репо.)
