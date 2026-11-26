---
name: get_employee_team
description: "Получить состав команды сотрудника, список и количество подчиненных. 
Используется если человек спрашивает о команде конкретного сотрудника: кто в команде, состав команды, участники команды, члены команды, подчиненные,
кто в подчинении, сколько подчиненных в команде, команда сотрудника по UUID."
---

# Инструкция для агента: получение состава команды сотрудника

## Когда использовать

Если необходимо получить:
- состав команды конкретного сотрудника;
- список подчиненных сотрудника;
- количество подчиненных в команде сотрудника.

Запрос выполняется по `person_id` (UUID) руководителя. Если задача — найти команду
или посчитать подчиненных конкретного сотрудника по его UUID, используй ЭТОТ
готовый кейс, а не собирай запрос с нуля.

---

## Используемый инструмент

- **Инструмент:** `mcp_query`
- **Модель данных:** схема `dm_core`, логическая модель `employee_actual`

---

## Предварительные условия

- Есть `person_id` руководителя (UUID, передавать строкой).
- Если известен только ФИО сотрудника — сначала получи его `person_id` с помощью
  тула `search_persons_by_fio`.

---

## Обязательные фильтры

| Колонка | Оператор | Значение | Зачем |
|---|---|---|---|
| `employee_status` | `=` | `активный` | Исключаем уволенных |

Фильтр `employee_status = активный` НЕ убирай — иначе в команду попадут
уволенные.

---

## Параметры вызова

| Параметр | Обязательный | Описание |
|---|---|---|
| `person_id` | да | UUID руководителя (строкой); одно и то же значение подставляется в каждый из четырех hierarchy-фильтров `has` |

Один и тот же UUID подставляется в каждый из четырех фильтров:

- `agile_curator_hierarchy_person_id_main`
- `agile_director_hierarchy_person_id_main`
- `oshs_curator_hierarchy_person_id_main`
- `oshs_director_hierarchy_person_id_main`

---

## Структура фильтров

- Четыре фильтра объединены в ОДИН узел `or` (поле `conditions`):
  сотрудник считается подчиненным, если он есть хотя бы в одной из иерархий.
- Весь этот or-узел соединен AND-ом с фильтром по статусу.
- НЕ заменяй or-узел одиночными фильтрами — несколько условий объединяются
  в один узел `or`, а не добавляются по отдельности.
- В запросе меняются ТОЛЬКО значения фильтров (значение `person_id`).
  Структуру шаблона, имена колонок и операторы не меняй.

---

## Шаблон запроса

Запрос для оценки размера искомой команды (если размер команды больше 10 человек, то мы их все НЕ выводим только 
	10 с наивысшим грейдом с возможностью посмотреть дальше).
```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "columns": [
    "<НАИМЕНОВАНИЕ КОЛОНКИ С РОЛЬЮ РУКОВОДИТЕЛЯ>" 
  ],
  "metrics": [
    "fact_count"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {
        "type": "condition",
        "column": "employee_status",
        "operator": "=",
        "value": "активный"
      },
      {
        "type": "or",
        "conditions": [
          {
            "type": "condition_array",
            "column": "agile_curator_hierarchy_person_id_main",
            "operator": "has",
            "value": "<PERSON_ID>"
          }
        ]
      }
    ]
  },
  "limit": 10
}
```
Этот запрос нужно повторить для каждой отдельной роли руководителя для оценки полного размера команды:
НАИМЕНОВАНИЕ КОЛОНКИ С РОЛЬЮ РУКОВОДИТЕЛЯ - `agile_curator_hierarchy_person_id_main`,
	`agile_director_hierarchy_person_id_main`,
	`oshs_curator_hierarchy_person_id_main`,
	`oshs_director_hierarchy_person_id_main`


Подставь `<PERSON_ID>` — UUID руководителя строкой — в каждый из четырех
hierarchy-фильтров. Выводим список сотрудников по 10 штук, по убыванию грейда. (используем offset для пролистывания)
```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "columns": [
    "person_id",
    "employee_id",
    "employee_full_name",
	"grade_level"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {
        "type": "condition",
        "column": "employee_status",
        "operator": "=",
        "value": "активный"
      },
      {
        "type": "or",
        "conditions": [
          {
            "type": "condition_array",
            "column": "agile_curator_hierarchy_person_id_main",
            "operator": "has",
            "value": "<PERSON_ID>"
          },
          {
            "type": "condition_array",
            "column": "agile_director_hierarchy_person_id_main",
            "operator": "has",
            "value": "<PERSON_ID>"
          },
          {
            "type": "condition_array",
            "column": "oshs_curator_hierarchy_person_id_main",
            "operator": "has",
            "value": "<PERSON_ID>"
          },
          {
            "type": "condition_array",
            "column": "oshs_director_hierarchy_person_id_main",
            "operator": "has",
            "value": "<PERSON_ID>"
          }
        ]
      }
    ]
  },
  "order_by": [
    {
      "field": "grade_level",
      "kind": "column",
      "direction": "desc"
    }
  ],
  "limit": 10,
  "offset": 0
}
```

---

## Как читать результат

| Поле | Что вернуть пользователю |
|---|---|
| `person_id` | ID сотрудника из команды |
| `employee_id` | Основной табельный номер сотрудника из команды |
| `employee_full_name` | ФИО сотрудника из команды |
| `grade_level` | Числовой идентификатор грейда |

- Лимит `10` — кол-во выводимых за раз сотрудников (иначе забиваем контекст нашего агента). 
Если объем команды больше установленного предела в 10, обязательно подсвети это пользователю.
Если пользоваетль хочет посмотреть всех сотрудников своей команды иди с шагом в `limit`=100 и `offset`=100, показывая сотрудников команды порционно.

---
