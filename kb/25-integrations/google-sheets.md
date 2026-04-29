---
title: Google Sheets
status: paused
last_updated: 2026-04-28
references: []
referenced_by: []
---

# Google Sheets

## Назначение

(планировалось) Дублирование данных из системы в Google Sheets для удобного просмотра / экспорта. **Приостановлено** до завершения проектирования Евы.

## Параметры подключения

- **Spreadsheet ID**: `1T_rbtWBONaTHi4vXqNQxQ1f7cJFLQmgJtcE8mNN6MKs`
- **Service account**: JSON-ключ лежит на VPS как `/var/www/trade/google-service-account.json`
- **SDK**: `googleapis`

## Текущий статус

- Service account создан
- JSON-ключ на VPS присутствует
- Интеграция в коде **не активирована**
- Решение об использовании — после проектирования Евы

## Шаги для возобновления

1. Создать Google Cloud Service Account → JSON ключ → положить как `/var/www/trade/google-service-account.json`
2. `npm install googleapis`
3. Настроить чтение/запись в нужный spreadsheet
4. Шаринг spreadsheet на email service-account'а

## Кто оплачивает

Не оплачивается (Google Sheets API в пределах free tier).

## Связанные документы

- [65-roadmap/current.md](../65-roadmap/current.md) — Google Sheets интеграция в "Отложено"
