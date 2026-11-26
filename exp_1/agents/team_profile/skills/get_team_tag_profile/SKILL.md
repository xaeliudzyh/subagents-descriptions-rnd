---
name: get_team_tag_profile
description: "Team characterization by tags: retrieve the employees of a team, tribe, or unit together with interests,
professional tags, and skills. Used if a person asks about the team profile, team interests, colleagues' hobbies and pastimes,
team building, prof tags, team skills, tag prevalence, how many holders of a tag or interest there are (for example, «сколько бегунов», «топ команд по Python»)."
---

# Agent instruction: team characterization by tags (interests, skills, prof tags)

## When to use

If it is necessary to:
- build a team profile: interests, professional tags, and skills of employees;
- count holders of a specific tag, interest, or skill
  («сколько бегунов в команде», «кто увлекается фотографией»);
- build a tag top list and a prevalence analysis («топ команд по Python»);
- understand the distribution of the team's interests for team building.

The search is performed by team name or org-structure level: team, tribe,
cluster, division, block, department.

### Glossary of colloquial terms

- «ЛиК» = «Люди и культура» (People and Culture) — expand it before filtering;
- «трайб / кластер» = agile levels of the org structure — if `oshs_level_*_unit_name_main`
  are empty for all employees in the sample, the agile structure is not populated in the mart:
  work with the linear structure and say so;
- «футболисты, бегуны, любители X» = employees in whom X occurs
  in `interests` or `skills_name`, not a job title.

---

## Tool used

- **Tool:** `mcp_query`
- **Data model:** schema `dm_core`, logical model `employee_actual`,
  table `dm_core.employee_actual`

---

## Mandatory filters

| Column            | Operator | Value | Why |
|-------------------|----------|---|---|
| `employee_status` | `=`      | `активный` | Exclude dismissed and not-yet-hired employees |
| `oshs.fact_flag`  | `has`    | 1 | Only employees in actual headcount |

---

## Call parameters

### `tag_value` — optional

The value of a specific tag/interest/skill for counting holders.
Substitute it into a `condition_array` node with the `has` operator on the `interests`
and/or `skills_name` column (exact match of an array element; for spelling variants,
use `hasAny` with a list).

IMPORTANT: `condition_like` does NOT work on Array columns. `interests` and
`skills_name` are of type `Array(Nullable(String))`: filter only with a
`condition_array` node:

- `has` — exact match of a single element;
- `hasAny` — any of the spelling variants in the list, for example
  `["бег", "бегун", "running", "trail running"]`.

---

## Three substitution template nodes

The query template has three substitution nodes; apply them according to the scenario:

1. **Search for a team/unit/division/center/cluster/tribe** → performed using the `get_structure_names_tool` tool.
	The tool returns the exact unit name and the name of the column (attribute) by which to filter:
		- "name" — the exact unit name found in the DB;
		- "structure_type" — the name of the column for which a filter with the found unit name must be written;
		- "db_filter" — returns a ready-made filter to insert into the query (payload).
2. **Counting holders of a specific tag** → an or-block (a block of OR operators) of `condition_array has`
   on `interests` and `skills_name`; add it TO the team or tribe filter.

---

## Example scenarios

**«Сколько бегунов в трайбе X»**
Export the employees of tribe X (via `get_structure_names_tool` — it returns the exact name and the TYPE of the unit column) with an or-block
`condition_array hasAny` on `interests` and `skills_name` with a list of forms
(бег, бегун, running, trail running), and count unique employees.

**«Топ команд по тегу Python»**
Filter holders via `condition_array hasAny` on `skills_name` and
`interests` with a list of spellings (python, питон), then count unique
employees by `oshs_unit_name_main`, sort by the number of holders,
and show the share of each team's headcount.

---

## Query template

The query for estimating the size of the employees exported by the specified attribute/tag (if the sample size is more than 10 people, we do NOT output all of them — only the 10 with the highest grade, with the ability to view further).

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "metrics": [
    "oshs_fact_count"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition", "column": "employee_status", "operator": "=", "value": "активный"},
      {"type": "condition_array", "column": "oshs.fact_flag", "operator": "has", "value": 1},
	  <db_filter - use the one previously found with the get_structure_names_tool tool>
      {
        "type": "or",
        "conditions": [
          {"type": "condition_array", "column": "interests", "operator": "has", "value": "тег или интерес"},
          {"type": "condition_array", "column": "skills_name", "operator": "has", "value": "тег или навык"}
        ]
      }
    ]
  },
  "order_by": [
    {"field": "person_id", "kind": "column", "direction": "asc"}
  ],
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

The query for directly outputting employees with the found tag


```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "columns": [
    "person_id",
	"grade_level",
    "employee_full_name",
    "oshs_unit_id_main",
    "oshs_unit_name_main",
    "oshs_level_1_unit_name_main",
    "oshs_level_2_unit_name_main",
    "oshs_level_3_unit_name_main",
    "oshs_level_4_unit_name_main",
    "oshs_level_5_unit_name_main",
    "prof_tags",
    "skills_name",
    "interests"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition", "column": "employee_status", "operator": "=", "value": "активный"},
      {"type": "condition_array", "column": "oshs.fact_flag", "operator": "has", "value": 1},
      {"type": "condition", "column": "field to be formatted - <structure_type>", "operator": "=", "pattern": "field to be formatted - <name>"},
      {
        "type": "or",
        "conditions": [
          {"type": "condition_array", "column": "interests", "operator": "has", "value": "тег или интерес"},
          {"type": "condition_array", "column": "skills_name", "operator": "has", "value": "тег или навык"}
        ]
      }
    ]
  },
  "order_by": [
    {"field": "person_id", "kind": "column", "direction": "asc"}
  ],
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

## How to read the result

| Field | What to return to the user |
|---|---|
| `person_id` | Employee ID |
| `grade_level` | Numeric grade identifier |
| `employee_full_name` | Employee full name |
| `oshs_unit_id_main` | Unit ID |
| `oshs_unit_name_main` | Unit name |
| `oshs_level_1_unit_name_main` | Unit of level 01 of the linear structure |
| `oshs_level_2_unit_name_main` | Unit of level 02 of the linear structure |
| `oshs_level_3_unit_name_main` | Unit of level 03 of the linear structure |
| `oshs_level_4_unit_name_main` | Unit of level 04 of the linear structure |
| `oshs_level_5_unit_name_main` | Unit of level 05 of the linear structure |
| `prof_tags` | Professional tags |
| `skills_name` | Skills |
| `interests` | Interests |

---

- The limit `10` is the number of employees output at a time (otherwise we clog our agent's context).
  If the team size exceeds the set limit of 10, you must highlight this to the user.
  If the user wants to view all employees of their team, go with a step of `limit`=100 and `offset`=100, showing the team's employees in portions.

## Aggregation and calculations

- Before aggregating, remove empty and null tag values and deduplicate
  employees: one employee is counted in the team headcount and in the tag counter
  once.
- For each tag, count the number of unique employees in whom the tag
  is present, and its share of the total headcount of the selected team.
- By default, show the top 10 tags of each category by the number of employees
  (format — "Response template" below). Grouping tags into enlarged categories
  (sport, technology, creativity) is allowed as a supplement to the table,
  but not instead of it.
- For the team profile, compute `interests`, `skills_name` and `prof_tags`
  SEPARATELY and do not mix the categories.
- Before counting, normalize the tags: bring them to lowercase, remove hyphenated
  compounds, parentheses, or service separators.

---

## Data coverage

- Always report coverage: «интересы заполнены у X из N сотрудников (X%)» —
  separately for each of the three categories. Low tag coverage must be
  reflected in the response; the absence of a tag in an employee does not mean the absence of an
  interest or skill.
- If in the export the `interests` / `skills_name` / `prof_tags` fields are empty for ALL
  employees in the sample — do NOT conclude "data is unavailable in the system". First
  check coverage on a broader slice (for example, by the parent
  unit or by the company): if the fields are populated there — the problem is in the sample
  or in the filter; recheck `team_name_pattern` and `org_scope`. The conclusion "the field
  is not populated globally" is admissible only after such a check.

---

## Paginated export

- If the team contains more than 10 employees, page the result through `offset`
  until `has_next_page = false`.

---

## Response template for the team profile

```text
Портрет команды «<название>» (N активных сотрудников)
Покрытие профилей: интересы заполнены у X из N (X%), навыки — у Y из N, профтеги — у Z из N.

Топ-10 интересов          | Топ-10 навыков            | Топ-10 профтегов
Тег | Чел. | Доля         | Тег | Чел. | Доля          | Тег | Чел. | Доля
...                          ...                           ...

Замечания: нормализация тегов (что склеено), низкое покрытие, ограничения данных.
```

---

## Limitations and ethics

- If several units with a similar name are found — you must clarify
  with the user which unit is needed, using the full org structure.
- Do not form negative assessments of specific employees and do not use tags
  as an indicator of productivity or engagement. To questions about the link between tags
  and performance/scores, reply that such a correlation is not built from this data,
  and offer a descriptive analysis of the team's interests.
- Do not show raw digital-footprint data, correspondence, files, or logs;
  the skill works only with the prepared fields of the model.

---

