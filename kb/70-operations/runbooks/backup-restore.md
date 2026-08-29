---
title: Runbook — бэкапы и восстановление (БД + откат кода)
status: active
last_updated: 2026-06-06
references:
  - ../infrastructure.md
  - deploy-frontend.md
  - ../../75-incidents/INC-002-vps-billing-outage.md
referenced_by: []
---

# Runbook: бэкапы и восстановление

Как откатиться, если деплой сломал систему, и как восстановить данные при сбое БД/VPS.

## Что и как бэкапится

| Слой | Где источник истины | Бэкап | Восстановление |
|---|---|---|---|
| **Код** (trade_app, Ева, trade-api, мини-апп) | git-репо `trade-abkhazia` (приватный) | каждый коммит запушен; + пофайловые `<файл>.bak-<дата>-<метка>` на VPS перед деплоем | git checkout + редеплой (ниже) |
| **БД `trade_db`** (KV `app_storage`: клиенты, продажи, рейсы, партии, цены) | Postgres на VPS | ежедневный `pg_dump` 03:00 → `/var/backups/trade/trade_db_*.sql.gz`, **14 дней** | restore (ниже) |
| **БД `eva`** (переписка — слой A `eva_messages`; память клиента — слой B `eva_client_memory`) | Postgres на VPS | ежедневный `pg_dump` 03:00 → `/var/backups/trade/eva_db_*.sql.gz`, **14 дней** | restore (ниже) |
| **Фото** (`/var/www/trade/photos/`) | файловая система VPS | ⚠️ **пока не бэкапятся** — [GAP-016](../../00-meta/gaps.md#gap-016) | — |

**Скрипт:** `/usr/local/bin/trade-backup.sh` (cron `0 3 * * *` root, лог `/var/log/trade-backup.log`). Копия скрипта в репо — `code/ops/trade-backup.sh`. Бэкапит обе БД; `eva`-пароль берётся dotenv-парсером из `/var/www/eva/.env` (как сама Ева), `trade_db` — из `/root/.pgpass`.

> ⚠️ **Главный остаточный риск — бэкапы лежат на том же диске VPS** (нет off-site). При потере сервера (вспомни [INC-002](../../75-incidents/INC-002-vps-billing-outage.md) — отключение по биллингу) бэкапы пропадут вместе с ним. Off-site-копия — [GAP-043](../../00-meta/gaps.md#gap-043).

## Откат кода (деплой сломал систему)

### Вариант А — быстрый, из пофайлового бэкапа на VPS
Перед каждым деплоем кладётся `<файл>.bak-<дата>-<метка>`. Откат:
```bash
# trade_app (фронт):
ssh root@trade-abkhazia.com 'cp /var/www/trade/frontend/src/App.jsx.bak-<дата>-<метка> /var/www/trade/frontend/src/App.jsx && cd /var/www/trade/frontend && npx vite build'
# Ева:
ssh root@trade-abkhazia.com 'cp /var/www/eva/src/<файл>.bak-<дата>-<метка> /var/www/eva/src/<файл> && pm2 restart eva'
```

### Вариант Б — из git (надёжный, к любой версии)
```bash
cd /home/max/trade_app_repo       # станция (на ноутбуке — ~/Documents/trade_app_repo)
git log --oneline                 # найти нужный коммит
git checkout <commit> -- code/trade_app_v3.jsx   # (или code/eva/..., code/trade-api/...)
# затем редеплой по runbook deploy-frontend.md / mini-app.md
```
После проверки вернуть рабочую копию к HEAD: `git checkout HEAD -- <файл>` (или закоммитить откат).

## Восстановление БД из бэкапа

> Тест восстановления пройден 2026-06-06 (trade_db → 33 KV-ключа, eva → 70 сообщений подняты во временные БД). Процедуру стоит **повторять раз в ~квартал**.

### Безопасная проверка (во временную БД, прод не трогаем)
```bash
ssh root@trade-abkhazia.com
LATEST=$(ls -t /var/backups/trade/trade_db_*.sql.gz | head -1)
sudo -u postgres createdb restore_test
gunzip -c "$LATEST" | sudo -u postgres psql -q restore_test
sudo -u postgres psql -tAc "SELECT count(*) FROM app_storage" restore_test   # сверить
sudo -u postgres dropdb restore_test
```
(для eva — то же с `eva_db_*.sql.gz`, БД `eva`, таблица `eva_messages`.)

### Боевое восстановление (прод повреждён)
⚠️ Перезатирает данные. Сначала останови писателей.
```bash
ssh root@trade-abkhazia.com
pm2 stop eva trade-api                       # остановить запись
LATEST=$(ls -t /var/backups/trade/trade_db_*.sql.gz | head -1)
# очистить и залить (вариант с пересозданием схемы):
gunzip -c "$LATEST" | sudo -u postgres psql -q trade_db
# eva:
LATEST_EVA=$(ls -t /var/backups/trade/eva_db_*.sql.gz | head -1)
gunzip -c "$LATEST_EVA" | sudo -u postgres psql -q eva
pm2 restart eva trade-api
```
> Дамп без `--clean`; при конфликтах объектов восстанавливать в свежую/очищенную БД. Перед боевым restore сделать свежий дамп текущего (даже битого) состояния.

## Проверка, что бэкапы идут
```bash
ssh root@trade-abkhazia.com 'ls -lt /var/backups/trade/ | head; tail -3 /var/log/trade-backup.log'
```
Признак здоровья: за сегодня есть и `trade_db_*`, и `eva_db_*`; в логе `rc=0`.

## Связанные документы
- [deploy-frontend.md](deploy-frontend.md), [mini-app.md](../../40-system/mini-app.md) — как деплоить (и откатывать) фронт
- [infrastructure.md](../infrastructure.md) — VPS, Postgres, PM2
- [INC-002](../../75-incidents/INC-002-vps-billing-outage.md) — почему нужен off-site
- [GAP-043](../../00-meta/gaps.md#gap-043) — off-site бэкапы (остаток), [GAP-016](../../00-meta/gaps.md#gap-016) — бэкап фото
