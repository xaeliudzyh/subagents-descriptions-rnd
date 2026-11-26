---
name: education-find-overdue-by-course
description: "Кто просрочил или не прошёл конкретный курс."
---
# Инструкция: найти сотрудников по курсу

> ⚠️ **Формат `mcp_query`:** `columns`/`order_by`/`metrics` — массивы `[...]`, `filters` — объект `{...}`, `limit` — число (без кавычек). НЕ передавай их строками (`"[...]"`, `"100"`). См. AGENTS.md «КАК ПРАВИЛЬНО ВЫПОЛНЯТЬ ЗАПРОСЫ».

## Когда использовать

Вопросы «кто просрочил курс X» / «кто назначен, но не завершил курс X».
Поиск по названию курса (или его части, `ILIKE`), поимённый перечень внутри
конкретного человека — скилл `education-get-employee-overdue-assignments`.

## Модель и метрики

- **Модель:** `dm_core.employee_actual` (подчинённые пользователя, ШАГ 0) и
  `dm_core.employee_education` (просрочка по курсу, ШАГ 1–3).
- **Метрики (`employee_education`):** `employees_count` (уникальные `person_id`),
  `assignments_count` (все назначения), `earliest_deadline` (`min(item_deadline_dt)`).
- **Фильтр поиска курса:** `item_name` через `ILIKE` по подстроке.

---

## Параметры

| Параметр | Обязательный | Описание |
|---|---|---|
| `course_name` | да | Название/часть курса → `pattern` в `item_name` (ILIKE) |
| `current_date` | да | Текущая дата `YYYY-MM-DD` → `value` в `item_deadline_dt` |
| `period_start` | нет | Нижняя граница окна; по умолчанию `current_date − 6 месяцев` |
| `item_id` | нет | Точный UUID курса вместо `item_name` (фильтр `=` по `item_id`) |
| `manager_person_id` | нет | UUID «текущего пользователя» → в 4 hierarchy-фильтра `has` (ШАГ 0) |

**Меняй только значения**: дату в `value`, название курса в `pattern`, при
наличии `item_id` — заменяй `condition_like` на `condition` по `item_id`,
`manager_person_id` в ШАГЕ 0. Структуру шаблонов, колонки и операторы не меняй.

**Дефолты (не уточняются ДО вывода):**
- **Группа:** **подчинённые пользователя** (текущий пользователь, вызвавший
  вопрос, и его команда), пересечённые с фильтром курса. `manager_person_id` =
  `person_id` текущего пользователя (доступен из контекста вызова / RLS).
  Если определить команду пользователя нельзя — прозрачно сообщи и предложи
  выбрать группу.
- **Период:** последние 6 месяцев (`item_deadline_dt >= period_start`, где
  `period_start = current_date − 6 месяцев`).
- **Порядок:** старые сначала (`earliest_deadline` asc).

---

## ПОРЯДОК ВЫПОЛНЕНИЯ: дефолтный результат → предложение изменить

1. Не уточняй срок/группу/порядок заранее. Сразу выполни запрос по дефолтам:
   группа = подчинённые пользователя + курс, период = последние 6 месяцев,
   порядок = старые сначала.
2. Выведи результат: сначала счёт (ШАГ 1), затем при небольшом числе — топ (ШАГ 2).
3. В самом выводе **явно укажи принятые параметры**: «Выведены просрочки по
   курсу **<КУРС> за последние 6 месяцев**, только у **ваших подчинённых**,
   отсортировано **от самых давних**».
4. После вывода предложи изменить:
   «Хотите изменить группу (например, **все по курсу** вместо подчинённых, или
   другую команду/сотрудника), период (например, всё время или другой срок)
   или порядок (новые сначала / алфавит)?»
   Если пользователь захотел — перезапусти запрос с новыми параметрами.

---

## ШАГ 0: команда текущего пользователя

По дефолту группа = **подчинённые пользователя**. Получи их на `employee_actual`
иерархическим запросом: один UUID (персона текущего пользователя) в каждый из
4 filter-условий `has` внутри узла `or`.

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "columns": ["person_id", "employee_full_name"],
  "filters": {
    "type": "and",
    "conditions": [
      { "type": "condition", "column": "employee_status", "operator": "=", "value": "активный" },
      { "type": "or", "conditions": [
        { "type": "condition_array", "column": "agile_curator_hierarchy_person_id_main", "operator": "has", "value": "<MANAGER_PERSON_ID>" },
        { "type": "condition_array", "column": "agile_director_hierarchy_person_id_main", "operator": "has", "value": "<MANAGER_PERSON_ID>" },
        { "type": "condition_array", "column": "oshs_curator_hierarchy_person_id_main", "operator": "has", "value": "<MANAGER_PERSON_ID>" },
        { "type": "condition_array", "column": "oshs_director_hierarchy_person_id_main", "operator": "has", "value": "<MANAGER_PERSON_ID>" }
      ] }
    ]
  },
  "limit": 50
}
```

- `<MANAGER_PERSON_ID>` = `person_id` текущего пользователя (доступен из контекста
  вызова / RLS). `employee_status = активный` не убирай.
- Собери `person_id` подчинённых → это `person_ids` для ШАГ 1–2 (`condition_in`).
- Если подчинённых нет или команду определить не удалось — честно сообщи и
  предложи выбрать другую группу (см. «Порядок выполнения»).
- Если пользователь попросил другую группу («все по курсу», другая команда,
  сотрудник) — пропусти команду текущего пользователя и вместо `condition_in`
  по подчинённым используй нужную группу (без team-фильтра для «все по курсу»).

## ШАГ 1: счёт (сколько человек)

Агрегация без группы — вернёт одну строку с числом просрочивших по курсу у
подчинённых за дефолтный период (последние 6 месяцев). `limit` не нужен.

```json
{
  "schema": "dm_core",
  "logic_model": "employee_education",
  "columns": [],
  "metrics": ["employees_count", "assignments_count"],
  "filters": {
    "type": "and",
    "conditions": [
      { "type": "condition_in", "column": "person_id", "operator": "IN", "value": ["<PERSON_ID_1>", "<PERSON_ID_2>"] },
      { "type": "condition", "column": "assignment_status", "operator": "=", "value": "ACTIVE" },
      { "type": "condition", "column": "item_deadline_dt", "operator": "<", "value": "<CURRENT_DATE>" },
      { "type": "condition", "column": "item_deadline_dt", "operator": ">=", "value": "<PERIOD_START>" },
      { "type": "condition_like", "column": "item_name", "operator": "ILIKE", "pattern": "%<COURSE_NAME>%" }
    ]
  },
  "order_by": []
}
```

- `<PERSON_ID_n>` — `person_id` подчинённых из ШАГА 0.
- `<PERIOD_START>` — по умолчанию `current_date − 6 месяцев`. Для «всё время»
  убери и `condition_in`, и условие `item_deadline_dt >= <PERIOD_START>`.
- Если известен точный `item_id` — замени `condition_like` на `condition` по `item_id`.

## ШАГ 2: счёт → вывести топ (дефолтный порядок)

1. Дай счёт: «По этому курсу у ваших подчинённых — N просрочек у M человек
   (за последние 6 месяцев), начиная с самых давних».
2. Реши по числу:
   - `employees_count = 0` — просрочки нет (не ошибка). Сообщи об этом.
   - `employees_count > 10` — верни счёт как ответ, перечень не выгружай.
   - `employees_count ≤ 10` — покажи поимённый топ (ШАГ 3).
3. После вывода предложи изменить группу/период/порядок (см. «Порядок выполнения»).

## ШАГ 3: поимённый топ (группировка по сотруднику)

Одна строка = один сотрудник. `limit` равен 10 и он обязателен. По умолчанию
`order_by` — `earliest_deadline` asc (старые сначала).

```json
{
  "schema": "dm_core",
  "logic_model": "employee_education",
  "columns": ["person_id", "employee_full_name"],
  "metrics": ["assignments_count", "earliest_deadline"],
  "filters": {
    "type": "and",
    "conditions": [
      { "type": "condition_in", "column": "person_id", "operator": "IN", "value": ["<PERSON_ID_1>", "<PERSON_ID_2>"] },
      { "type": "condition", "column": "assignment_status", "operator": "=", "value": "ACTIVE" },
      { "type": "condition", "column": "item_deadline_dt", "operator": "<", "value": "<CURRENT_DATE>" },
      { "type": "condition", "column": "item_deadline_dt", "operator": ">=", "value": "<PERIOD_START>" },
      { "type": "condition_like", "column": "item_name", "operator": "ILIKE", "pattern": "%<COURSE_NAME>%" }
    ]
  },
  "order_by": [
    { "field": "earliest_deadline", "kind": "metric", "direction": "asc" }
  ],
  "limit": 10
}
```

Варианты `order_by` (если пользователь попросит изменить порядок):
- «старые сначала» / «кто дольше всех» → `earliest_deadline` asc (дефолт);
- «новые сначала» / «сейчас горит» → `earliest_deadline` desc;
- «алфавит» → поле `employee_full_name`, direction `asc`, kind `column`.

## Вариант «назначен, но не завершён»

Тот же ход, но **без фильтра по `item_deadline_dt`** (ни `<current>`, ни нижней
границы) — оставь только `assignment_status = ACTIVE`, `item_name` (ILIKE) и
`condition_in` по подчинённым. Период по умолчанию также не применяется (нет
дедлайна).

---

## Как читать результат

| Поле | Что вернуть пользователю |
|---|---|
| `employees_count` / `assignments_count` | сколько человек / назначений просрочило (ШАГ 1) |
| `employee_full_name` | ФИО сотрудника (ШАГ 3) |
| `earliest_deadline` | дедлайн (по выбранному порядку) |

- Всегда начинай со счёта за дефолтный период (6 месяцев), затем топ ≤ 10.
- В выводе укажи группу (подчинённые пользователя), период и порядок; предложи
  изменить их.
- Не выгружай сырые строки и не листай `offset`. Не пересказывай сырой ответ
  инструмента — давай готовый бизнес-ответ.

---