---
name: find_vacation_overlaps
description: "Сырой расчет пересечений отпусков по списку табельных номеров через оконную агрегацию. НЕ вызывай самостоятельно: только из приема get_vacation_overlaps, минимум двумя employee_id. На входе список табельных номеров и период."
---

# Расчет пересечений отпусков (вспомогательный рецепт)

Считает пересечения отпусков между переданными сотрудниками. Это
технический шаг — вызывается только из скилла `get_vacation_overlaps`.

## Инструмент

- MCP-сессия: `heimdall`
- Инструмент: `mcp_query`
- Модель: схема `dm_core`, логическая модель `vacation_overlap`

## Обязательные фильтры (не убирать)

| Колонка | Оператор | Значение |
|---|---|---|
| `is_vacation` | `=` | `1` |
| `is_grade_above_12` | `=` | `1` |

## Шаблон запроса

Подставь:
- `<EMPLOYEE_IDS>` — список минимум двух табельных номеров строками (массив);
- `<PERIOD_START>` / `<PERIOD_END>` — границы периода `YYYY-MM-DD`.

```json
{
  "schema": "dm_core",
  "logic_model": "vacation_overlap",
  "columns": [
    "employee_id",
    "employee_full_name",
    "company",
    "absence_start",
    "absence_end",
    "absence_code",
    "grade_level",
    "overlap_business"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition_in", "column": "employee_id", "operator": "IN", "value": <EMPLOYEE_IDS>},
      {"type": "condition", "column": "is_vacation", "operator": "=", "value": 1},
      {"type": "condition", "column": "is_grade_above_12", "operator": "=", "value": 1},
      {"type": "condition", "column": "absence_end", "operator": ">=", "value": "<PERIOD_START>"},
      {"type": "condition", "column": "absence_start", "operator": "<=", "value": "<PERIOD_END>"}
    ]
  },
  "order_by": [
    {"field": "employee_id", "kind": "column", "direction": "asc"},
    {"field": "absence_start", "kind": "column", "direction": "asc"}
  ],
  "limit": 1000
}
```

## Как читать результат

Используй только `overlap_business`. Каждый элемент — готовая пара:
`(ФИО_1, начало_1, конец_1, код_1, ФИО_2, начало_2, конец_2, код_2, overlap_start, overlap_end, overlap_days)`.

- Пустой `overlap_business` = пересечений нет.
- Один сотрудник всегда дает пустой результат — нужна пара.
- Технические поля (employee_id, grade_level, факт-флаги) пользователю не выводи.

Если пришло ровно 1000 строк — продолжай выборку с `offset`.