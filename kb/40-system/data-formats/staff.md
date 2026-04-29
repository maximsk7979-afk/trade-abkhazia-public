---
title: Staff — JSON-формат сотрудника
status: active
last_updated: 2026-04-28
references:
  - ../database/storage-keys.md
  - ../../15-team/README.md
referenced_by: []
---

# Staff — JSON-формат сотрудника

## Storage

- Ключ: `trade-cat-staff`
- JS-константа: `SK_STAFF`
- Содержимое: массив объектов

## Формат

```json
{
  "id": "С-001",
  "name": "Максим",
  "roles": ["owner"],
  "officeId": "О-001",
  "telegramChatId": "118206343",
  "telegramUsername": "",
  "whatsapp": "",
  "phone": "",
  "active": true
}
```

## Поля

| Поле | Тип | Обязательное | Описание |
|---|---|---|---|
| `id` | string | да | ID с префиксом С (`С-001`) |
| `name` | string | да | Имя |
| `roles` | array<string> | да | Список ролей (см. ниже) |
| `officeId` | string | да | Привязка к офису (О-XXX) |
| `telegramChatId` | string | нет | Chat ID в Telegram (для уведомлений и Евы) |
| `telegramUsername` | string | нет | `@username` без @ |
| `whatsapp` | string | нет | Телефон для WhatsApp |
| `phone` | string | нет | Обычный телефон |
| `active` | boolean | да | Активен ли сейчас |

## Возможные роли

```
owner | gen_director | fin_director | ops_manager |
expeditor | sales_manager | warehouse | buyer |
accountant | loader | driver
```

У одного сотрудника может быть **несколько** ролей одновременно.

См. [30-roles/README.md](../../30-roles/README.md) для описания ролей.

## Связи

- `officeId` → офис в `trade-cat-offices`
- `roles[]` → ролевая модель в [30-roles/](../../30-roles/)
- `telegramChatId` используется в `NOTIFY_CHAT_IDS` (`.env`) и для маршрутизации сообщений Евы

## История

- 2026-04-28: Формат описан в kb/.
