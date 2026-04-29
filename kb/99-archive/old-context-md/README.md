---
title: Old CONTEXT.md
status: archived
last_updated: 2026-04-28
---

# Архив: старый CONTEXT.md

Снимок исходного `~/Documents/trade_app/CONTEXT.md` на момент перехода на модульную базу знаний.

## Файл

- [CONTEXT-2026-04-26.md](CONTEXT-2026-04-26.md) — копия исходного CONTEXT.md от 2026-04-26 (последнее обновление). Сохранена ровно как была, **без правок**.

## Чек-лист переноса

Содержимое CONTEXT.md разбирается **поэтапно** по новой структуре. По мере переноса блоков — отмечаем тут.

| Раздел CONTEXT.md | Куда переехало | Статус |
|---|---|---|
| 1. О проекте | [10-business/overview.md](../../10-business/overview.md) | ✅ |
| 2. Принцип работы с документами | устарел (новая модульная структура) | ✅ зафиксировано в ADR-002 |
| 3. Архитектура | [40-system/architecture.md](../../40-system/architecture.md), [70-operations/infrastructure.md](../../70-operations/infrastructure.md) | ✅ частично |
| 4. API ключи | вне репо: `~/secrets-trade/credentials.md` | ✅ 2026-04-28 (полный перенос в защищённый локальный файл; в репо нигде не остались) |
| 5. Ключи хранилища | [40-system/database/storage-keys.md](../../40-system/database/storage-keys.md) | ✅ |
| 6. Текущее состояние данных (последний ID) | устарело (тестовые данные стираются 1 апреля) | ✅ зафиксировано |
| 7. Форматы данных (ПС, ДС, staff, client, office) | [40-system/data-formats/](../../40-system/data-formats/) | ✅ 2026-04-28 (sale.md, staff.md, client.md, office.md, settlement.md). Раздел 7 в CONTEXT.md содержал ошибки, исправлено в ADR-007 |
| 8. Правила расчёта весов | [20-domain/weight-calculation.md](../../20-domain/weight-calculation.md) | ✅ 2026-04-28 (обнаружен баг с запятой как разделителем — [GAP-007](../../00-meta/gaps.md)) |
| 9. Правила прайс-листа | [20-domain/pricing.md](../../20-domain/pricing.md) | ✅ 2026-04-28 (4 механизма override обнаружены при сверке с кодом, не было в CONTEXT.md) |
| 10. Правила seed | устарело (seed-блоки удалены) | ✅ 2026-04-28 (раздел потерял актуальность после ADR-007: весь seed-блок удалён из кода, маркеры в БД сохранены, новых seed-блоков не планируется) |
| 11. Алгоритмы бота | [20-domain/business-rules-from-bot.md](../../20-domain/business-rules-from-bot.md) (бизнес-правила); сценарии Евы — Q3 в [50-agents/eva/scenarios/](../../50-agents/eva/) | ✅ 2026-04-29 (бизнес-правила извлечены и сохранены: автоопределение ПС/ЗД, defect_refund, расхождение ≤50₽, уведомления, разовый покупатель). Реализация в Еве — Q3 |
| 12. Google Sheets | [25-integrations/google-sheets.md](../../25-integrations/google-sheets.md) | ✅ |
| 13. Словарь грузинских накладных | [20-domain/invoices/georgian-dictionary.md](../../20-domain/invoices/georgian-dictionary.md) | 🟡 2026-04-29 (стартовый словарь 22 термина и 4 правила перенесены частично; ждёт полной выгрузки из обученного Claude — [GAP-003](../../00-meta/gaps.md)) |
| 14. Реализованные модули | [40-system/frontend.md](../../40-system/frontend.md) | ✅ кратко |
| 15. Задачи на очереди | [65-roadmap/current.md](../../65-roadmap/current.md) | ✅ |
| 16. Деплой фронтенда | [70-operations/runbooks/deploy-frontend.md](../../70-operations/runbooks/deploy-frontend.md) | ✅ |
| 17. Деплой бота | устарело (бот отключён) | ✅ |
| 18. Файлы проекта | устарело (новая структура) | ✅ |
| 19. Как работать в Claude Code | устарело (новая структура) | ✅ |
| 20. Архитектура офисов | [20-domain/offices.md](../../20-domain/offices.md) | ✅ 2026-04-28 (все 6 утверждений сверены с кодом и подтверждены) |

## Когда удалить

Когда все строки в чек-листе выше будут ✅, оригинальный CONTEXT.md в `~/Documents/trade_app/` можно удалить (или оставить для истории — как удобно). Архивная копия здесь — остаётся всегда.
