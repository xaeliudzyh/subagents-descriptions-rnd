---
name: agentic-skill-overview
description: Обзор агентных навыков, распределение по ролям и уровням. Сводка выводится в режиме агрегата — число сотрудников и средняя доля ИИ-кода с разбивкой по основной роли (вариант — по уровню навыка). Запускай, когда нужна картина распределения агентных навыков, ролей и внедрения GigaCode, а не отдельные сотрудники. Синонимы — обзор навыков, распределение по ролям, распределение по уровням навыка, средняя доля ИИ-кода, сколько людей по ролям, сегментация по уровням.
---

# Обзор агентных навыков: распределение по ролям и уровням

Возвращает распределение числа сотрудников и средней доли ИИ-кода по
основным ролям (агрегат, витрина `stable.agentic_skill_level`). Вариант —
распределение по уровню навыка (например, prompt engineering).

## Когда использовать

- Пользователь спрашивает «сколько людей по ролям», «как распределены
  агентные навыки», «средняя доля ИИ-кода по ролям», «обзор внедрения
  GigaCode» — нужна сводная картина, а не перечень сотрудников.
- Распределение по уровню навыка: «сколько человек на каждом уровне
  prompt engineering».

## Алгоритм

1. **Определи разрез группировки.**
   - По умолчанию и для вопроса «по ролям» — группировка по `role_main`.
   - Если пользователь спрашивает распределение по уровню навыка (один из
     `n1_prompting`, `n2_prompt_engineering`, `n3_context_managment`,
     `n4_creating_tools`, `n5_creating_skills`, `n6_agent_team`) —
     используйте вариант (группировка по этому навыку).
2. **Выполни готовый запрос** через `mcp_query`. Для ролей используй
   основной запрос с метриками `employee_count` и
   `avg_ai_code_flow_curr`; для уровня навыка — вариант.
3. **Собери ответ**: по каждой группе (роль или уровень) — число
   сотрудников и среднюю долю ИИ-кода за текущий месяц.

## ГОТОВЫЙ ЗАПРОС (по ролям)

```json
{
  "schema": "stable",
  "logic_model": "agentic_skill_level",
  "columns": ["role_main"],
  "metrics": ["employee_count", "avg_ai_code_flow_curr"],
  "order_by": [
    {"field": "employee_count", "kind": "metric", "direction": "desc"}
  ],
  "limit": 50,
  "offset": 0
}
```

## ВАРИАНТ (по уровню навыка, пример — prompt engineering)

```json
{
  "schema": "stable",
  "logic_model": "agentic_skill_level",
  "columns": ["n2_prompt_engineering"],
  "metrics": ["employee_count"],
  "order_by": [
    {"field": "n2_prompt_engineering", "kind": "column", "direction": "asc"}
  ],
  "limit": 50,
  "offset": 0
}
```

## ВАРИАНТ (как посмотреть значения метрик по владению ai-инструментами для конкретного сотрудника)

```json
{
  "schema": "stable",
  "logic_model": "agentic_skill_level",
  "columns": [
    "tn", "calc_date", "role_main", "role_2", "expertise_area",
    "n1_prompting", "n2_prompt_engineering", "n3_context_managment",
    "n4_creating_tools", "n5_creating_skills", "n6_agent_team",
    "use_mode", "use_frequency", "ai_code_flow_curr", "ai_code_flow_prev",
    "gigacode_ca", "ag_coding_course", "ai_pdlc_course", "gigacode_course"
  ],
  "filters": {
    "type": "condition",
    "column": "tn",
    "operator": "=",
    "value": "<подставь табельный номер>"
  },
  "order_by": [
    {"field": "calc_date", "kind": "column", "direction": "desc"}
  ],
  "limit": 12,
  "offset": 0
}
```

## Что подставить

- Для варианта по уровню навыка — замени `n2_prompt_engineering` (в
  `columns` и `order_by`) на нужный навык (`n1_prompting`,
  `n3_context_managment`, `n4_creating_tools`, `n5_creating_skills`,
  `n6_agent_team`).
- Если нужен другой показатель в варианте — добавь метрику в `metrics`,
  не меняя логику группировки.

## На что обратить внимание

- `columns` плюс `metrics` дают режим aggregate: группировка по `columns`,
  показатели в `metrics`. Группируй по `role_main`, `expertise_area`,
  `use_mode` или по уровню навыка `n1..n6`.
- `avg_ai_code_flow_curr` — это avgOrNull, NULL не учитываются. Значения
  доли ИИ-кода — в диапазоне **от 0 до 1** (1 = 100%); для процентов
  умножай на 100.
- Витрина содержит несколько срезов по дате расчёта. Без отбора по
  `calc_date` агрегат идёт по всем срезам сразу; для одного среза добавь
  фильтр по `calc_date`.
- `employee_count` — число строк; при нескольких `calc_date` один табельный
  номер посчитается несколько раз.
