---
name: education-get-overdue-summary
description: "Сводка по просроченному обучению: сколько сотрудников и назначений просрочило, по курсам/сотрудникам, обученность."
---
# Инструкция: сводка по просроченному обучению

> ⚠️ **Формат `mcp_query`:** `columns`/`order_by`/`metrics` — массивы `[...]`, `filters` — объект `{...}`, `limit` — число (без кавычек). НЕ передавай их строками (`"[...]"`, `"100"`). См. AGENTS.md «КАК ПРАВИЛЬНО ВЫПОЛНЯТЬ ЗАПРОСЫ».

## Когда использовать

Вопросы «сколько просрочили», «у кого больше всего», «самая длительная
просрочка», «разбивка по курсам/сотрудникам», «обученность». Это **сводка с
агрегацией** — поимённый перечень делают другие скиллы.

## Модель и метрики

- **Модель:** `employee_education` (сводка) и `employee_actual` (команда, ШАГ 1).
- **Метрики:** `employees_count` (уникальные `person_id`), `assignments_count`
  (все назначения), `earliest_deadline` (`min(item_deadline_dt)`).

---

## Параметры

| Параметр | Обязательный | Описание |
|---|---|---|
| `current_date` | да | Текущая дата `YYYY-MM-DD` → `value` в `item_deadline_dt` |
| `manager_person_id` | только «моя команда» | UUID руководителя → в 4 hierarchy-фильтра `has` |
| `period_start` | нет | Нижняя граница окна (для «за период»); иначе — накоплена с начала времён |

Меняй только значения (дата в `value`, фильтры `condition_in`). Структуру
шаблонов, колонки и операторы не меняй.

---

## Обязательное уточнение: группа людей

Перед запросом ВСЕГДА уточни, по какой группе людей считается сводка. Это нужно,
даже если в вопросе группа не задана — предложи вариант на выбор:

- «По всей компании» (без фильтра по `person_id`);
- «По вашей команде / подчинённым» (нужен `manager_person_id`);
- «По конкретному сотруднику» (нужен `person_id`);
- «По конкретному курсу» (нужен `course_name`/`item_id`).

Если пользователь не выбрал — дефолт «вся компания». Не пропускай этот блок и не
выводи сводку, пока группа не определена.

## Уточнение периода (по желанию)

Сводка по умолчанию считается **с начала времён** (без нижней границы).
Предложи уточнить период, сохранив вариант «за всё время»:
- «За всё время или за последние N месяцев?» Дефолт — «всё время».
- Если выбран период — добавь в фильтр условие `item_deadline_dt >= <PERIOD_START>`.

---

## ШАГ 1: команда (только для «моя команда»)

Получи подчинённых на `employee_actual` тем же иерархическим запросом, что
`education-get-subordinates-overdue-assignments` (ШАГ 1): 4 фильтра `has`
в узле `or`, значение `<MANAGER_PERSON_ID>`. Собери `person_id` подчинённых.
- Ровно 100 строк — команда больше: не листай, отвечай сводкой по команде.
- «Вся компания» — пропусти этот шаг.

## ШАГ 2: счёт и разбивка — один шаблон, разные `columns`

База — `employee_education` с фильтром просрочки `assignment_status = ACTIVE`
и `item_deadline_dt < <CURRENT_DATE>`. По умолчанию без нижней границы;
если выбран период — добавь `item_deadline_dt >= <PERIOD_START>`.

```json
{
  "schema": "dm_core",
  "logic_model": "employee_education",
  "columns": [],
  "metrics": ["employees_count", "assignments_count"],
  "filters": {
    "type": "and",
    "conditions": [
      { "type": "condition", "column": "assignment_status", "operator": "=", "value": "ACTIVE" },
      { "type": "condition", "column": "item_deadline_dt", "operator": "<", "value": "<CURRENT_DATE>" },
      { "type": "condition", "column": "item_deadline_dt", "operator": ">=", "value": "<PERIOD_START>" }
    ]
  },
  "order_by": []
}
```

- «За всё время» — убери условие `item_deadline_dt >= <PERIOD_START>`.
- «Моя команда» — добавь `condition_in` по `person_id` со списком команды.

Выбери вариант, меняя `columns`/`metrics`/`order_by`:

| Вариант | `columns` | `metrics` | `order_by` (desc) | `limit` |
|---|---|---|---|---|
| Общий итог | `[]` | `employees_count, assignments_count` | `[]` | — |
| По курсам | `item_name` | `employees_count, assignments_count` | `assignments_count` | 10 |
| По сотрудникам («у кого больше») | `person_id, employee_full_name` | `assignments_count` | `assignments_count` | ≤10 |
| «Самая длительная» | `person_id, employee_full_name` | `assignments_count, earliest_deadline` | `earliest_deadline` | ≤10 |

- «Сколько ЧЕЛОВЕК» — `employees_count`; «сколько НАЗНАЧЕНИЙ» — `assignments_count`.
- «Моя команда» — добавь `condition_in` по `person_id` со списком команды.

## Обученность (процент)

Отдельный агрегационный запрос: `columns = ["assignment_status"]`,
`metrics = ["assignments_count"]`, фильтр
`assignment_status IN ('CLOSED','ACTIVE')`. Процент =
`assignments_count(CLOSED) / (assignments_count(CLOSED)+assignments_count(ACTIVE)) * 100%`.
Только корпоративное обучение.

---

## Как читать результат

| Поле | Что вернуть |
|---|---|
| `employees_count` | число сотрудников |
| `assignments_count` | число назначений (курс × человек) |
| `earliest_deadline` | самый ранний дедлайн |

- Отдай готовый счёт и (при необходимости) разбивку/топ ≤ 10 строк.
- «Моя команда» — отметь, что цифры по команде руководителя.
- Не пересказывай сырой ответ инструмента.

---