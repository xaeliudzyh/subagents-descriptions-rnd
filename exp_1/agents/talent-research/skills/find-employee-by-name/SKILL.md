---
name: find-employee-by-name
description: How, given a mention of a person in a request (first name, last name, first-last name, full name in any word order), to find an internal employee in dm_special.top_600_stats via mcp_query and return their person_id, full name, and role profile — role title, responsibilities, functions, goal names, key achievements. The word order in the request may differ from how the full name is recorded in the DB.
---
# Finding an internal employee by name

## Step 1. Parse the mention into tokens

From the request, extract the name tokens: last name, first name, patronymic (whichever of these is given).

The word order in the request may not match the DB record: the user says «Александр Иванов», while the database stores «Иванов Александр Сергеевич». Therefore:

- search **by tokens separately**, joining them with `and` — each token as a separate `condition_like` ILIKE;
- do **not** search as a single whole string (`%Александр Иванов%` — a miss with a different word order);
- one token — one node; three tokens — three nodes.
- if a token contains «ё», on an empty result repeat the search replacing «ё» with «е».

## Step 2. Query

Source: `dm_special.top_600_stats`.

```json
{
  "schema": "dm_special",
  "logic_model": "top_600_stats",
  "limit": 50,
  "columns": [
    "person_id",
    "employee_id",
    "employee_full_name",
    "professionlevel_name",
    "professionlevel_duties_name",
    "professionlevel_functions_name",
    "goals_title",
    "achievements_key_name",
    "oshs_unit_names",
    "oshs_unit_names_tree"
  ],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition_like", "column": "employee_full_name", "operator": "ILIKE", "pattern": "%<last name>%"},
      {"type": "condition_like", "column": "employee_full_name", "operator": "ILIKE", "pattern": "%<first name>%"}
    ]
  }
}
```

- request only these columns, not the whole profile.

## Step 3. Parse the result

| Result | Action |
|---|---|
| One found | Return the employee for passing to the `talent-research` skill |
| Several found | Check whether among these several people there is an exact single match with the full name (or first name and last name) specified in the request. If not — do not guess: output the list (full name + role (professionlevel_name)) and ask the user for clarification about whom they mean |
| Empty | Repeat the query by last name only (the most reliable token); still empty — return: «Никого не найдено с таким именем среди сотрудников ТОП-600» |

Return: `person_id`, full name, and role profile: `professionlevel_name` (role title), `professionlevel_duties_name` (responsibilities), `professionlevel_functions_name` (functions), `goals_title` (goal names), `achievements_key_name` (key achievements). Address the person by `person_id` (namesakes are possible).