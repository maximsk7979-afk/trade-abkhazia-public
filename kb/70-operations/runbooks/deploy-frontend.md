---
title: Runbook — деплой фронтенда
status: active
last_updated: 2026-04-28
references:
  - ../../40-system/deployment.md
  - ../infrastructure.md
referenced_by: []
---

# Runbook: Деплой фронтенда

## Когда применять

После изменения файла `~/Documents/trade_app/trade_app_v3.jsx` локально, когда нужно выкатить новую версию на https://trade-abkhazia.com.

## Предусловия

- SSH-пароль к VPS (в локальном `~/secrets-trade/credentials.md`)
- Установлен `sshpass` локально (или используется обычный `scp` с интерактивным вводом пароля)
- Локальный файл `trade_app_v3.jsx` готов к деплою

## Шаги

### 1. Загрузить файл на VPS

```bash
sshpass -p '<password>' scp \
  ~/Documents/trade_app/trade_app_v3.jsx \
  root@108.61.167.168:/var/www/trade/frontend/src/App.jsx
```

(пароль — из локального credentials.md, не из репо)

### 2. Собрать через Vite

```bash
sshpass -p '<password>' ssh root@108.61.167.168 \
  "cd /var/www/trade/frontend && npx vite build"
```

Vite пишет в `../public/`, Nginx сразу раздаёт новый бандл.

### 3. Проверка

Открыть https://trade-abkhazia.com в браузере, убедиться, что изменения применились (можно нужен hard reload Cmd+Shift+R).

## Откат

При проблемах:
1. Вернуть локальный мастер-файл к предыдущей версии (через git: `git checkout <commit> -- trade_app_v3.jsx`)
2. Повторить шаги 1-2

## Что менять под git-flow

Сейчас деплой — из мастер-копии на Маке. Правильнее:
- Деплой из git-репозитория (git pull на VPS, потом vite build)
- Или CI/CD через GitHub Actions

Задача в [roadmap](../../65-roadmap/current.md).

## Связанные документы

- [40-system/deployment.md](../../40-system/deployment.md)
- [infrastructure.md](../infrastructure.md)
