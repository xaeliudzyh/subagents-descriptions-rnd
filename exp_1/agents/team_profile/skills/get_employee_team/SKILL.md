---
name: get_employee_team
description: "Retrieve an employee's team composition, the list and number of subordinates.
Used if a person asks about the team of a specific employee: who is on the team, team composition, team participants, team members, subordinates,
who reports to whom, how many subordinates are on the team, an employee's team by UUID."
---

# Agent instruction: retrieving an employee's team composition

## When to use

If it is necessary to retrieve:
- the composition of a specific employee's team;
- the list of an employee's subordinates;
- the number of subordinates on an employee's team.

## First — the get_head_team tool; the mart only if it is not applicable

Before the first `mcp_query` call, call **`get_head_team`** with the employee ID of the
employee whose team is in question:

- `handled: true` — work ONLY with the tool response: `mode: "head_team"` is
  the manager's team composition (`team`, `subordinates_count` for each),
  `mode: "team_member_subordinates"` — the subordinates of a specific team member
  (`subordinates`, `subordinates_total`). Do not run `mcp_query` requests on the
  `*_hierarchy_person_id_main` fields for this team: the hierarchy in the mart
  for it is currently unreliable. If the response is truncated (`has_next_page: true`) — page through
  by increasing `offset`.
- `handled: false` — the employee ID is not in the Chairman of the Management Board's team: the tool is not applicable, continue
  per the instructions below (`mcp_query` requests).

The request is executed by the manager's `person_id` (UUID). If the task is to find the team
or count the subordinates of a specific employee by their UUID, use THIS
ready-made case rather than building the query from scratch.

---

## Tool used

- **Tool:** `mcp_query`
- **Data model:** schema `dm_core`, logical model `employee_actual`

---

## Preconditions

- The manager's `person_id` is available (a UUID, to be passed as a string).
- If only the employee's full name is known — first obtain their `person_id` using
  the `search_persons_by_fio` tool.

---

## Mandatory filters

| Column | Operator | Value | Why |
|---|---|---|---|
| `employee_status` | `=` | `активный` | Exclude dismissed employees |

Do NOT remove the `employee_status = активный` filter — otherwise dismissed
employees will end up in the team.

---

## Call parameters

| Parameter | Required | Description |
|---|---|---|
| `person_id` | yes | The manager's UUID (as a string); the same value is substituted into each of the four hierarchy filters `has` |

The same UUID is substituted into each of the four filters:

- `agile_curator_hierarchy_person_id_main`
- `agile_director_hierarchy_person_id_main`
- `oshs_curator_hierarchy_person_id_main`
- `oshs_director_hierarchy_person_id_main`

---

## Filter structure

- The four filters are combined into ONE `or` node (the `conditions` field):
  an employee is considered a subordinate if they appear in at least one of the hierarchies.
- This entire or-node is connected with AND to the status filter.
- Do NOT replace the or-node with individual filters — several conditions are combined
  into one `or` node rather than added separately.
- In the query, ONLY the filter values (the `person_id` value) change.
  Do not change the template structure, column names, or operators.

---

## Query template

The query for estimating the size of the target team (if the team size is more than 10 people, we do NOT output all of them — only
	the 10 with the highest grade, with the ability to view further). If the request explicitly specifies only a certain structure:
    - ОШС (or linear) — the corresponding columns for search — oshs_curator_hierarchy_person_id_main, oshs_director_hierarchy_person_id_main
    - agile — the corresponding columns for search — agile_curator_hierarchy_person_id_main, agile_director_hierarchy_person_id_main
    Then use filters only with the corresponding columns; if not specified, use an OR filter across all 4 columns.
```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "columns": [
    "<MANAGER ROLE COLUMN NAME>"
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
  "limit": 10
}
```
This query must be repeated for each individual manager role to estimate the full team size:
MANAGER ROLE COLUMN NAME - `agile_curator_hierarchy_person_id_main`,
	`agile_director_hierarchy_person_id_main`,
	`oshs_curator_hierarchy_person_id_main`,
	`oshs_director_hierarchy_person_id_main`


Substitute <PERSON_ID> — the manager's UUID as a string — into each of the four
hierarchy filters. We output the list of employees 10 at a time, in descending grade order. (using offset to paginate)
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

## How to read the result

| Field | What to return to the user |
|---|---|
| `person_id` | The ID of an employee from the team |
| `employee_id` | The primary employee ID of an employee from the team |
| `employee_full_name` | The full name of an employee from the team |
| `grade_level` | The numeric grade identifier |

- The limit `10` is the number of employees output at a time (otherwise we clog our agent's context).
If the team size exceeds the set limit of 10, you must highlight this to the user.
If the user wants to view all employees of their team, go with a step of `limit`=100 and `offset`=100, showing the team's employees in portions.

---