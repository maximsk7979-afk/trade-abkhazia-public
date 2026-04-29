---
title: База данных — обзор
status: active
last_updated: 2026-04-28
references:
  - storage-keys.md
referenced_by: []
---

# База данных

## Что есть

- **СУБД**: PostgreSQL
- **БД**: `trade_db`
- **Пользователь**: `trade_user`
- **Пароль**: в локальном `~/secrets-trade/credentials.md`, не в репо
- **Хост**: localhost на VPS

## Структура

**Одна таблица:**

```sql
CREATE TABLE app_storage (
  key        VARCHAR(255) PRIMARY KEY,
  value      TEXT,                      -- JSON как строка
  updated_at TIMESTAMP DEFAULT NOW()
);
```

По сути — **KV-store**. Никаких индексов кроме PK, никаких FK, никаких схем. Вся доменная модель сериализуется в JSON и лежит в `value`.

## Ключи

См. [storage-keys.md](storage-keys.md) — полный перечень ключей и что в каждом.

## Миграции

**Не существуют как процесс.** Изменения схемы доменных объектов делаются:
- В коде через **seed-блоки** во `App.jsx` (`_SD3` ... `_SD8`, версия v45)
- При загрузке фронт проверяет marker и применяет seed
- Никакого alembic/flyway/migrations нет

См. [GAP-004](../../00-meta/gaps.md) — это требует упорядочивания.

## Бэкапы

(не описаны) — это пробел, см. [GAP в gaps.md](../../00-meta/gaps.md).

## Что менять под Еву

- Замена KV-store на нормальную реляционную схему — задача в [roadmap](../../65-roadmap/current.md).
- Регулярные бэкапы (текущее состояние неизвестно).
- Миграции через стандартный инструмент (когда уйдём от seed-блоков).

## Связанные документы

- [api.md](../api.md)
- [storage-keys.md](storage-keys.md)
