---
name: lookup_employee_by_name
description: "Найти сотрудника по ФИО (части) через mcp_query к dm_core.employee_actual. Возвращает person_id, employee_id (табельный), должность, грейд, структуру. Использовать как вспомогательный шаг: резолвить ФИО в person_id/employee_id для других скиллов. Если найдено несколько — уточни у пользователя."
---

# Поиск сотрудника по ФИО

Резолвит ФИО → `person_id` + `employee_id` (табельный) через heimdall.
Это вспомогательный скилл: его вызывают, когда у запроса есть только ФИО,
а нужному скиллу требуется person_id/employee_id.

## Инструмент

- MCP-сессия: `heimdall`
- Инструмент: `mcp_query`
- Модель: схема `dm_core`, логическая модель `employee_actual`

## Шаблон запроса

Подставь `<ФАМИЛИЯ ИМЯ>` — часть полного ФИО (ILIKE, `%` = любые символы,
регистр не важен).

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "columns": [
    "person_id",
    "employee_id",
    "employee_full_name",
    "position_name",
    "grade_level",
    "fblock_name",
    "oshs_level_2_unit_name_main"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition", "column": "employee_status", "operator": "=", "value": "активный"},
      {"type": "condition_array", "column": "oshs.fact_flag", "operator": "has", "value": 1},
      {"type": "condition_like", "column": "employee_full_name", "operator": "ILIKE", "pattern": "%<ФАМИЛИЯ ИМЯ>%"}
    ]
  },
  "order_by": [
    {"field": "employee_full_name", "kind": "column", "direction": "asc"}
  ],
  "limit": 50
}
```

## Как читать результат

- `person_id` — UUID сотрудника (для запросов по person_id).
- `employee_id` — табельный номер (для `find_vacation_overlaps`).
- `employee_full_name` — полное ФИО.
- `position_name` / `grade_level` / `fblock_name` / `oshs_level_2_unit_name_main` — для уточнения у пользователя, если нашлось несколько.

## Правила уточнения

- Нашёлся 1 человек — сразу бери его `person_id`/`employee_id`.
- Нашлось несколько — выведи список (ФИО + должность) и попроси уточнить,
  о ком речь. Не продолжай расчет до ответа.
- Пусто — ответь: `Сотрудник с ФИО "<ФИО>" не найден. Проверьте написание фамилии и имени.`