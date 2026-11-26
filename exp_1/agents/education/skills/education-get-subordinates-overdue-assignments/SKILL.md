---
name: education-get-subordinates-overdue-assignments
description: "Просроченное и предстоящее обучение команды (подчинённых) по дедлайнам."
---
# Инструкция: просроченное обучение команды (подчинённых)

> ⚠️ **Формат `mcp_query`:** `columns`/`order_by`/`metrics` — массивы `[...]`, `filters` — объект `{...}`, `limit` — число (без кавычек). НЕ передавай их строками (`"[...]"`, `"100"`). См. AGENTS.md «КАК ПРАВИЛЬНО ВЫПОЛНЯТЬ ЗАПРОСЫ».

## Когда использовать

Вопросы «кто из команды просрочил обучение», «что горит у команды»,
«предстоящие дедлайны команды». Скилл сам получает подчинённых руководителя.

## Модель и метрики

- **Модель:** `employee_actual` (команда, ШАГ 1) и `employee_education` (ШАГ 2–4).
- **Метрики (`employee_education`):** `employees_count` (уникальные `person_id`),
  `assignments_count` (все назначения), `earliest_deadline` (`min(item_deadline_dt)`).

---

## Параметры

| Параметр | Обязательный | Описание |
|---|---|---|
| `manager_person_id` | да | UUID руководителя → в 4 hierarchy-фильтра `has` |
| `current_date` | да | Текущая дата `YYYY-MM-DD` |
| `horizon_date` | нет | Верхняя граница окна (для «предстоящие») |
| `period_start` | нет | Нижняя граница окна (для «за период»); иначе — с начала времён |
| `mode` | нет | Что показывать: `overdue` / `upcoming` / `both` (по умолчанию `both`) |
| `horizon_days` | нет | Кол-во дней для «предстоящие» (по умолчанию 10) → `horizon_date` |

Меняй только значения. Структуру шаблонов, колонок и операторы не меняй.

---

## Обязательное уточнение: группа людей

Этот скилл работает по **команде / подчинённым руководителя**. Перед запросом
ВСЕГДА уточни, по какой группе считаем — какая именно команда:

- Подчинённые какого руководителя? Нужен `manager_person_id`.
- Если руководитель в вопросе не назван — уточни, чью команду смотрим.
- Если вопрос про одного сотрудника или курс — это НЕ сюда: используй
  `get-employee-overdue-assignments` / `find-overdue-by-course` /
  `get-overdue-summary`.

Не начинай запрос, пока команда не определена.

## Уточнение у пользователя (короткое)

После определения команды уточни одним блоком, что хочет пользователь.
Предлагай дефолты — не превращай в допрос.

1. **Что отобразить?** «Просроченные, предстоящие или оба?» (`overdue` /
   `upcoming` / `both`). Не спрашивай, если ответ очевиден («что горит» —
   `overdue`, «скоро дедлайн» — `upcoming`).
2. **На сколько дней вперёд?** для предстоящих (дефолт 10) → `horizon_date`.
3. **Период.** По умолчанию — всё время (без нижней границы). Предложи «за
   последние N месяцев», но сохрани вариант «за всё время». Для периода добавь
   `item_deadline_dt >= period_start`.
4. **Порядок.** Старые/новые/алфавит — меняет только `order_by` в ШАГАХ 2–4.
   Дефолт — старые сначала (`earliest_deadline` asc).

---

## ШАГ 1: команда руководителя

Один UUID в каждый из 4 hierarchy-фильтров `has`:

- `agile_curator_hierarchy_person_id_main`
- `agile_director_hierarchy_person_id_main`
- `oshs_curator_hierarchy_person_id_main`
- `oshs_director_hierarchy_person_id_main`

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

- `employee_status = активный` не убирай. Собери `person_id` — это `person_ids`
  для ШАГИ 2–4.
- Ровно 50 строк — команда больше: не листай, отвечай сводкой
  `education-get-overdue-summary` по команде или сузь срез.

## ШАГ 2: счёт по команде (агрегированный, без выгрузки строк)

Метрики по команде через `condition_in`. `limit` в агрегации без группы не нужен.
По умолчанию без нижней границы; для периода добавь `item_deadline_dt >= <PERIOD_START>`.

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
      { "type": "condition", "column": "item_deadline_dt", "operator": ">=", "value": "<PERIOD_START>" }
    ]
  },
  "order_by": []
}
```

## ШАГ 3: решение по числу

- `employees_count = 0` — просрочки в команде нет (не ошибка). Сообщи об этом.
- `employees_count > 10` — верни счёт как ответ, список не показывай.
- `employees_count ≤ 10` — покажи поимённый перечень (ШАГ 4).

## ШАГ 4: поимённый перечень по команде (группировка по сотруднику)

Одна строка = один сотрудник. `earliest_deadline asc` поднимает самые давние
просрочки. `limit` равен 10 и он обязателен.

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
      { "type": "condition", "column": "item_deadline_dt", "operator": ">=", "value": "<PERIOD_START>" }
    ]
  },
  "order_by": [
    { "field": "earliest_deadline", "kind": "metric", "direction": "asc" }
  ],
  "limit": 10
}
```

## Вариант «Предстоящие дедлайны команды»

Тот же ход в 3–4 шага, но без фильтра дедлайна `item_deadline_dt < current_date`,
с окном `>= <CURRENT_DATE>` и `<= <HORIZON_DATE>` (верхняя граница =
`current_date + horizon_days`).

## Вариант «Оба среза»

Если выбран `both` — выполни «просроченные» (ШАГИ 2–4) и «предстоящие» порознь,
затем сведи в ответ. Для каждого среза счёт → топ ≤ 10.

## Как читать результат

| Поле | Что вернуть |
|---|---|
| `employees_count` / `assignments_count` | сколько человек / назначений в команде (ШАГ 2) |
| `employee_full_name` | ФИО сотрудника (ШАГ 4) |
| `earliest_deadline` | самый давний дедлайн |

- Всегда начинай со счёта (ШАГ 2). Поимённый перечень — только если `≤ 10`
  человек. Пустой ответ — отсутствие просрочки, не ошибка.
- Для «сколько/кто больше» по команде используй сводку `education-get-overdue-summary`.
- Не выгружай сырые строки и не листай `offset`. Не пересказывай сырой ответ
  инструмента.

---