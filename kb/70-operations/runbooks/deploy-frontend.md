---
title: Runbook — деплой фронтенда
status: active
last_updated: 2026-08-29
references:
  - ../../40-system/deployment.md
  - ../infrastructure.md
  - ../../90-decisions/ADR-025-single-domain-layer.md
referenced_by: []
---

# Runbook: Деплой фронтенда

## Когда применять

После правки `code/trade_app_v3.jsx` (или `code/trade_app_calc.mjs`) в репозитории, когда
нужно выкатить новую версию на https://trade-abkhazia.com.

## Как деплоить

Из корня репозитория, одной командой:

```bash
./scripts/deploy.sh trade-app
```

Скрипт делает всё сам: локальные тесты `calc` → бэкап `App.jsx` на VPS → загрузка файлов →
`vite build` на VPS → проверка, что новый бандл реально отдаётся. При красных тестах или
упавшей сборке деплой останавливается, прод остаётся на прежнем бандле.

## Почему нельзя копировать один jsx руками

Фронт не самодостаточен: вместе с ним обязаны уехать `trade_app_calc.mjs` и **три ядра**,
которые vite вбандливает (ADR-025, единый доменный слой):

| Файл в репо | Куда на VPS |
|---|---|
| `code/trade_app_v3.jsx` | `/var/www/trade/frontend/src/App.jsx` |
| `code/trade_app_calc.mjs` | `/var/www/trade/frontend/src/calc.mjs` |
| `code/trade-api/trip-cost-core.cjs` | `.../src/trade-api/trip-cost-core.cjs` |
| `code/trade-api/purchase-cost-core.cjs` | `.../src/trade-api/purchase-cost-core.cjs` |
| `code/trade-api/cash-core.cjs` | `.../src/trade-api/cash-core.cjs` |

Скопировать только `jsx` — значит выкатить фронт со **старыми ядрами** себестоимости и
кассы: цифры на экране разъедутся с сервером. Именно этот класс расхождений закрывал
ADR-025, поэтому ручная scp-цепочка из прежней версии runbook'а отменена.

## Предусловия

- `sshpass` установлен; пароль root — из `~/secrets-trade/credentials.md` (раздел `## VPS`),
  скрипт достаёт его сам (можно переопределить через `SSHPASS`)
- Рабочее место — станция (`/home/max/trade_app_repo`); писатель одновременно один

## Проверка

Скрипт печатает имя нового бандла (`index-XXXX.js`) и сам проверяет, что тот отдаётся по
HTTPS. В браузере — hard reload (Cmd/Ctrl+Shift+R), Telegram-клиент кэш сбрасывает не сразу.

## Откат

Бэкап предыдущего `App.jsx` лежит на VPS: `/var/www/trade/frontend/src/App.jsx.bak-<STAMP>`,
общий каталог деплоя — `/var/www/_bak/deploy-<STAMP>/`. Штатный путь отката — вернуть файл
в репозитории (`git checkout <commit> -- code/trade_app_v3.jsx`) и повторить деплой: так
прод и репозиторий остаются согласованы.

## Связанные документы

- [40-system/deployment.md](../../40-system/deployment.md)
- [infrastructure.md](../infrastructure.md)
