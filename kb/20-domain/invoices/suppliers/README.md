---
title: suppliers — per-supplier базы знаний
status: active
last_updated: 2026-05-19
references:
  - ../README.md
  - ../../../90-decisions/ADR-011-eva-supplier-segmentation.md
referenced_by: []
---

# suppliers — per-supplier базы знаний

Каждая подпапка — **полная локальная база** знаний по конкретному поставщику (вариант А из [ADR-011](../../../90-decisions/ADR-011-eva-supplier-segmentation.md), пункт 3). Дублирование терминов между поставщиками **допустимо и желательно** — это плата за независимую эволюцию.

## Текущие поставщики

| ID | Папка | Поставщик | Статус | Период наблюдения |
|---|---|---|---|---|
| П-004 | [p-004-zaza/](p-004-zaza/) | Заза (оптовый склад в Батуми) | active | 2026-02-07 — 2026-05-19 |

## Шаблон при добавлении нового поставщика

Структура папки (по ADR-011, пункт 1):

```
suppliers/<id>-<short-name>/
├── profile.md            # карточка: локация, тип, ассортимент, контакт
├── dictionary.md         # словарь его жаргона (термины, единицы, сокращения)
├── recognition-rules.md  # правила определения сорта + эвристики неоднозначностей
├── format.md             # его формат накладной (компоновка, особенности)
├── handwriting.md        # описание почерка (если рукопись)
├── price-ranges.md       # типичные цены (с пометкой о сезоне сбора)
└── examples/             # его эталонные накладные (10–15)
    ├── README.md
    ├── np-<short>-001.jpg
    └── np-<short>-001.md
```

Имя папки — `<supplier-id>-<short-name>` латиницей. Поле `supplierId` в карточке — `П-NNN` (как в БД `trade-cat-suppliers`).

## Шаблон карточки `profile.md`

См. [ADR-011](../../../90-decisions/ADR-011-eva-supplier-segmentation.md), пункт 7. Минимум:

```yaml
---
title: "Профиль поставщика: <Имя>"
supplierId: П-NNN
applicableProjects: [PRJ-NNN, ...]
status: active | paused | archived
last_updated: YYYY-MM-DD
---
```

## Правило роста `_common/`

Если правило/наблюдение из локальной базы подтвердится у ≥2 поставщиков — **переносить в `_common/`**, оставляя в локальных только специфику. Не наоборот (см. [ADR-011](../../../90-decisions/ADR-011-eva-supplier-segmentation.md), пункт 6).
