---
name: profile-data
title: Внутренние данные сотрудника для эссе
description: >-
  Как получить данные для эссе-портрета: точный mcp_query по person_id и
  колонки ФИО и статуса, позиции и оргструктуры, грейда, стажа, руководства,
  целей и их выполнения, оценок, достижений, наград, образования, трудовой
  книжки, тегов, интересов, языков и поля «о себе».
---

# Сбор внутреннего профиля из `dm_core.employee_actual`


## Шаг 1. Запросить профиль

Фильтр — по `person_id` (UUID вида `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`):
так надёжнее, чем по ФИО, тёзки не перепутаются. Обязательно добавляй
`employee_status = 'активный'` и `oshs.fact_flag[1] = 1`: в модели есть и уволенные, и
«будущих» сотрудники.

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "limit": 10,
  "columns": [
    "person_id",
    "employee_id",
    "employee_full_name",
    "employee_status",
    "position_name",
    "position_name_top",
    "position_category",
    "grade",
    "grade_level",
    "is_director",
    "director_type",
    "priority_director_type",
    "position_flag_boss",
    "is_territorial_bank_chairman",
    "management_years",
    "absence_type",
    "oshs_unit_name_main",
    "oshs_level_1_unit_name_main",
    "oshs_level_2_unit_name_main",
    "oshs_level_3_unit_name_main",
    "fblock_name",
    "segment_name",
    "professionlevel_name",
    "last_hire_date",
    "last_fired_date",
    "experience_sber_year",
    "experience_sber_month",
    "experience_in_position_year",
    "last_workplace",
    "goals.title",
    "goals.progress",
    "mean_value_completion",
    "num_goals",
    "risked_goals",
    "goal_is_expired",
    "estimation.year",
    "estimation.quarter",
    "estimation.performance",
    "estimation_year",
    "achievements_key_name",
    "achievements_key_description",
    "achievements_key_end_year",
    "achievements.name",
    "awards_type_name",
    "awards_group",
    "awards_date",
    "education_type_name",
    "educational_institution_name",
    "educational_faculty",
    "educational_speciality",
    "education_period",
    "scientific_degree",
    "scientific_dissertation_name",
    "labor_book_sber.company_name",
    "labor_book_sber.unit_name",
    "labor_book_sber.position_name",
    "labor_book_sber.start_date",
    "labor_book_sber.end_date",
    "labor_book_sber.type_action",
    "labor_book.company_name",
    "labor_book.position_name",
    "labor_book.start_date",
    "labor_book.end_date",
    "prof_tags",
    "skills_name",
    "interests",
    "languages_name",
    "languages_level",
    "bio",
    "corporate_activity_name",
    "mobility_status"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition", "column": "person_id", "operator": "=", "value": "<person_id>"},
      {"type": "condition", "column": "employee_status", "operator": "=", "value": "активный"},
      {"type": "condition_array", "column": "oshs.fact_flag", "operator": "has", "value": 1}
    ]
  }
}
```

Держи `limit` маленьким (5–10): часть полей — массивы, и на широкой выборке
ответ раздувается. Если запрос вернул ошибку по колонке, убери её из `columns`
и повтори. Другого поля в `columns` не добавляй: если колонки не хватает,
проверь её имя один раз через `describe_model("dm_core", "employee_actual")`,
а нет и там — скажи об ограничении, поле не придумывай.

Запрос нужен один раз на человека. Не повторяй его «для проверки» и не
добирай колонки, если данных для эссе уже хватает: лишний запрос может вернуть
другой срез и сбить цифры в готовом тексте.

### Если запрос вернул ошибку

`get_skill("errors")` → `get_skill("filters")` → `get_skill("rules")`, затем
повтор. Данные интернетом не подменяй.

### Если `person_id` неизвестен
Если данных по person_id не пришло с предыдущего гаша - пропускай шаг поиска внутренних данных и переходи на внешний поиск в интернете

## Шаг 2. Финальные данные

Разложи ответ в таблицу «факт → значение → источник». Пустое значение
остаётся пустым, в эссе оно не попадает.

| Колонки `employee_actual` | Куда попадают в эссе (порядок разделов — в скилле `profile-essay`, п. 5.2) |
|---|---|
| `employee_full_name`, `position_name` / `position_name_top`, `position_category`, `grade` / `grade_level`, `employee_status`, `absence_type`, `last_fired_date` | раздел 1 «Кто сегодня» |
| `oshs_unit_name_main`, `oshs_level_1..3_unit_name_main`, `fblock_name`, `segment_name`, `professionlevel_name` | раздел 1 (подразделение) |
| `last_hire_date`, `experience_sber_year`, `experience_sber_month`, `experience_in_position_year` | разделы 1 и 6 (стаж в компании и в роли) |
| `is_director`, `director_type`, `priority_director_type`, `position_flag_boss`, `is_territorial_bank_chairman`, `management_years` | раздел 5 (масштаб управления) |
| `goals.title`, `goals.progress`, `num_goals`, `mean_value_completion`, `risked_goals`, `goal_is_expired` | разделы 5 (цели периода) и 6 (показатели) |
| `estimation.performance` вместе с `estimation.year` / `estimation.quarter`, `estimation_year` | раздел 6 (таблица оценок, правила — ниже) |
| `achievements_key_name`, `achievements_key_description`, `achievements_key_end_year`, `achievements.name`, `awards_type_name`, `awards_group`, `awards_date` | раздел 6 «Достижения и показатели» |
| `labor_book_sber.*` | раздел 4 «Ключевые этапы карьеры» (внутри Сбера: компания → подразделение → должность → период) |
| `labor_book.company_name`, `labor_book.position_name`, `labor_book.start_date` / `end_date`, `last_workplace` | раздел 3 «Начало пути» и раздел 4 (опыт до Сбера) |
| `education_type_name`, `educational_institution_name`, `educational_faculty`, `educational_speciality`, `education_period`, `scientific_degree`, `scientific_dissertation_name` | раздел 2 «Детство и образование» |
| `prof_tags`, `skills_name` | раздел 7 (стиль управления и профессиональные интересы) |
| `interests`, `corporate_activity_name`, `languages_name` / `languages_level`, `mobility_status`, `city_name` | раздел 8 «Сотрудник за пределами работы» (только подтверждённые поля; без здоровья и убеждений) |
| `bio` | раздел 7 (цитата «о себе» — только если заполнено и не содержит личного) |
| `person_id` | только для ссылки на профиль в Пульсе; в тексте не выводится |

## Как читать массивы

- `goals.title` / `goals.progress` — первая цель соответствует первому
  значению прогресса, `goal_is_expired` выровнен так же. Переноси дословно.
- `estimation.performance` — оценки за всё время (A–E), порядок совпадает с
  `estimation.year` и `estimation.quarter` (1–4 — квартал, 5 — годовая, 6–7 —
  отменённые полугодовые). «Текущую» годовую оценку не путай с
  `estimation_year`: там может быть предыдущий год. Как собирается таблица —
  в скилле `profile-essay`, п. 5.3.1.
- `labor_book_sber.*` — массивы одной длины: собирай этапы «дата начала → дата
  конца → компания → подразделение → должность» и сортируй по дате начала.
  `type_action` («перевод», «прием», «перемещение») помогает понять смену
  контура.
- `achievements_key_*` и `awards_*` — параллельные массивы.
- `"[]"` и `null` означают «данных нет», а не ноль.

## Правила

- Запрашивай только перечисленные колонки, а не `*`.
- Не суммируй и не пересчитывай показатели, переноси как есть.
- Не выводи идентификаторы (`person_id`, `employee_id`), названия таблиц и
  полей в тексте эссе.
- Если `employee_actual` вернул пусто по активному фильтру, человек мог быть
  уволен или ещё не вступил в должность: повтори запрос без
  `employee_status = 'активный'`. Что делать с уволенным — в скилле
  `profile-essay`, раздел «Формат выдачи».

