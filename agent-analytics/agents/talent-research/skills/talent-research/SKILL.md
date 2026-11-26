---
name: talent-research
title: Sourcing and search for external executives, leaders, top managers CEO, CTO, CDO, CPO, against a description of the team, task, role, vacancy, requirements.
description: How, given the user's task and sourcing criteria, to find the most relevant high-level specialists, top managers CEO, CTO, CDO, CPO, and experts in the mart, not to invent facts, and to give an answer grounded in the mart's data.
---
# Talent Radar: sourcing top-k candidates for a task

This skill serves one main purpose:

> Given a user request and sourcing criteria, find in `dm_special.talent_radar_people` the top-k most relevant candidates and explain the choice with data from the database.

## What to do on mcp_query errors

If a `mcp_query` request returns an error:

- `get_skill("errors")` — decoding of error codes and how to fix them
- `get_skill("filters")` — correct filter syntax (the most common cause of errors)
- `get_skill("rules")` — general rules and prohibitions

## Talent Radar mart

```text
dm_special.talent_radar_people
```

This is a read-only mart, **1 row = 1 person** among high-level executives and experts. Data coverage is uneven. The absence of a record does not necessarily mean the absence of experience in reality.

**Forbidden:**
- collecting new data;
- changing data;
- adding facts from the internet, memory, or assumptions.

Work only with what MCP returned.

## Source of truth for the schema

Before building a query, use:

```text
describe_model("dm_special", "talent_radar_people")
```

Do not guess column names, metrics, or types. If the needed column is not in `describe_model`, do not use it.

## MCP limitations for this mart

For this mart, work goes through `mcp_query`.

**Unavailable:**
- filtering by internal fields `career`, `education`, `publications`, `sources`, `reputation`, `activities`, `telegram`;
- text semantic search.

## Filtering:

`filters` must not be left empty {}.

Example query with correct filtering:
```json
{
  "schema": "dm_special",
  "logic_model": "talent_radar_people",
  "columns": ["id", "full_name", "current_title", "region", "is_blocked"],
  "filters": {
    "type": "and",
    "conditions": [
      {"type": "condition", "column": "is_blocked", "operator": "=", "value": false},
      {"type": "or", "conditions": [
        {"type": "condition_null", "column": "current_company", "operator": "IS NULL"},
        {"type": "and", "conditions": [
          {"type": "condition_like", "column": "current_company", "operator": "NOT ILIKE", "pattern": "%сбер%"},
          {"type": "condition_like", "column": "current_company", "operator": "NOT ILIKE", "pattern": "%sber%"}
        ]}
      ]},
      {"type": "condition", "column": "region", "operator": "=", "value": "ru"},
      {"type": "or", "conditions": [
        {"type": "condition_like", "column": "current_title", "operator": "ILIKE", "pattern": "%CEO%"},
        {"type": "condition_like", "column": "current_title", "operator": "ILIKE", "pattern": "%генеральный директор%"}
      ]}
    ]
  },
  "limit": 50
}
```

### Mandatory exclusion of current Sber employees

Candidates who currently work at Sber or its subsidiaries do not appear in the output. This is a hard criterion: the block below is included in **every** mcp_query to the mart together with `is_blocked = false`:

```json
{"type": "or", "conditions": [
  {"type": "condition_null", "column": "current_company", "operator": "IS NULL"},
  {"type": "and", "conditions": [
    {"type": "condition_like", "column": "current_company", "operator": "NOT ILIKE", "pattern": "%сбер%"},
    {"type": "condition_like", "column": "current_company", "operator": "NOT ILIKE", "pattern": "%sber%"}
  ]}
]}
```

- Do not remove the `condition_null ... IS NULL` branch: `NOT ILIKE` on NULL yields NULL, and a profile with an empty company will silently drop out of the output. An empty `current_company` means "unknown", not "works at Sber".
- The two patterns together cover all occurring spelling variants: «Сбер», «ПАО Сбербанк — Уральский банк», «Байкальский банк ПАО Сбербанк», «Сбербанк / SberDevices», «Sberbank», as well as subsidiaries («СберЗдоровье», «СберЛизинг», etc.).

## Star Index

`star_index` — overall profile strength 0–100.

**Important:**
- `star_index` does not equal fit to a specific task;
- `star_index NULL` = not scored, not weak;
- do not recalculate `star_index`;
- do not sum `score_career`, `score_media`, `score_science`, `score_reputation`;
- use the components only to explain why the profile looks strong.

Use `star_index` as a secondary signal. The primary one is task fit based on evidence.

## How to build a correct mcp_query

To build queries to Talent Radar, follow the general rules from `get_skill("rules")` and the specific instructions below.

**Key points:**
1. Start with `describe_model("dm_special", "talent_radar_people")` to get the exact field names
2. Use the correct JSON format (not YAML, not strings)
3. Filters must be a single tree with a root node (see `get_skill("filters")`)
4. For array fields (`domains`, `tags`), use `condition_array`
5. Always add the filter `is_blocked = false`


## Main algorithm for top-k against a task

### Step 1. Understand the task

Extract from the user's request the requirements/criteria:
```text
k — how many candidates are needed
region — if not explicitly specified, then RF.
domain
role
seniority
role scale — level of responsibility, team size, P&L, if they are specified
companies / industries
must-have criteria
nice-to-have criteria
what matters more: profile strength or task fit. If not explicitly specified, then task fit.
```

Also determine link_requested:
- link_requested=true: If in the request the user explicitly asks for links to candidate profiles, for example: "Подбери мне кандидатов и дай мне ссылки на их профили" or "Подбери мне кандидатов и дай их профили"
- link_requested=false: If the request says nothing about links to profiles.

link_requested will be needed to determine the response output format.

Do not invent hard criteria that the user did not ask for, beyond those specified earlier.

### Step 2. Split the criteria

Criteria are of three types.

#### 2.1 Hard criteria

Conditions whose violation makes a candidate unsuitable:
- 'is_blocked = false';
- a current Sber or subsidiary employee is excluded always, via the mandatory `current_company` filter block (see "Mandatory exclusion of current Sber employees");
- region, if explicitly set by the user or set by default;
- another requirement that the user named as mandatory and that can be checked via MCP.

##### Regional rules

Known codes:
```text
ru = Russia
eu = Europe
am = America
asia = Asia
au = Australia
us = USA
cn = China
wt = world / undefined
NULL = unknown
```

**IMPORTANT:** If the user did not specify a region, use `ru` as the business default and briefly report this in the answer. NULL means an unknown region, not non-compliance with other requirements; in that case, you may rely on the person's career to determine the region.

#### 2.2 Retrieval signal

Signs that help form the candidate pool but should not, by default, exclude the rest:
- primary_domain / domains;
- primary_tag / tags;
- current_title and its synonyms;
- current_company;
- key business terms from summary;
- star_index;


#### 2.3 Evidence criteria

Requirements checked after retrieval:
- explicitly stated information about P&L or team size;
- team scale;
- details of past career — role, company;
- education;
- publications and activities;
- reputational signals.

Do not turn evidence criteria into non-existent filters. Do not turn the retrieval signal into one common AND filter.

### Step 3. Assemble the candidate pool

`Candidate pool` is an extended list of unique candidates that the agent will collect from several MCP queries before the evidence analysis and final ranking. It is not the final shortlist.
If the user did not specify k (the number of candidates in the final shortlist), use `k=10`.

Maximum size of one retrieval query = 50
Minimum desired size of the combined pool = 50
Maximum size of the combined pool before preliminary assessment = 100

#### How to form the pool

##### 3.1 Execute all main retrieval channels

Execute all channels applicable to the request:

- Search by `primary_domain` / `domains`
- Search by `primary_tag` / `tags`
- Search by `current_title` variants
- Search by `current_company` (if the company is a significant criterion)
- Search by characteristic terms from `summary`

##### 3.2 In every mcp query, always apply:

In every MCP query, always apply:

- Hard criteria
- The previously set limit on the size of one retrieval query

##### 3.3 Do not combine all retrieval signals with one common AND.

Each channel must find a separate group of potentially relevant candidates.
Within one channel, combine alternative values via 'OR'; use the root 'AND' only for 'is_blocked = false', hard constraints, and one channel-specific block.

##### 3.4 Merge the results of all channels by id.

If one person is found by several queries:
  - store them once,
  - but remember which channels found them.

##### 3.5 First execute all main applicable channels

Do not stop searching after reaching the minimum pool size: different channels may find different types of relevant candidates.

If, after the main channels, fewer than the desired number are found:
- expand the list of job title synonyms;
- use broader `domain` / `tag` conditions;
- remove only soft signals and nice-to-have filters;
- run a broad fallback query over the available scalar fields;
- if necessary, get the next page via `offset`.

Never weaken:
- `is_blocked = false`
- the block excluding current Sber employees (`current_company`, see "Mandatory exclusion of current Sber employees")
- must-haves explicitly set by the user
- hard criteria

**If the pool exceeds the maximum size:**

- perform a **preliminary selection** and keep no more than the calculated maximum.

**Priority criteria**
Rank higher candidates:
- found by several independent retrieval channels;
- having a direct match with the role and professional domain;
- having more available data for further verification.

Use `star_index` **only as a tie-breaker** between candidates with comparable structural signals.

##### 3.6 Important note

- At this step, do not draw final conclusions about fit and do not form the user-facing answer.

### Step 4. Obtain the compact profile and evidence

When generating compact profiles of candidates for later use, use compact fields:
```text
id, full_name, current_company, current_title, region, primary_domain, primary_tag, domains, tags, star_index, is_blocked
```

After the preliminary assessment, select an extended shortlist of size `3*k`, but no more than 50.
For the extended shortlist, request 'summary' and evidence blocks: `career`, `education`, `publications`, `activities`;

### Step 5. Assess and rank by task fit

For each requirement, determine the state:

- confirmed — there is direct confirmation;
- contradicted — there is a direct contradiction;
- unknown — insufficient data. Not necessarily contradicted, but not confirmed either.

Do not treat unknown as non-compliance. Do not infer P&L, team scale, education, or past experience from the job title alone.
Separately assess level fit: whether the scale of current or past experience matches the scale of the target role. If the candidate is clearly significantly stronger and above the role in level of responsibility, treat this as an overqualification risk and rank them below candidates comparable on task fit. Do not treat fame or a high star_index as proof of overqualification, and do not invent the candidate's motivation, salary expectations, or willingness to change jobs.

Rank sequentially:

1. exclude candidates with contradicted on a mandatory must-have;
2. rank higher candidates with more confirmed must-haves;
3. take into account role and domain fit, as well as level fit;
4. take into account confirmed nice-to-haves and evidence strength;
5. use star_index only as a tie-breaker.

#### Position fit score

- For each candidate in the final shortlist, form a position fit score from 0.0 to 10.0 inclusive. Output the score with one decimal place, for example 8.7.
- Form the score based on the comparison already performed: confirmed must-haves, role and domain fit, level fit, confirmed nice-to-haves, and evidence strength.
- Preserve the priority order established above. This is a numeric representation of fit to the specific position.
- Preserve the candidate order obtained by the rules of the step.
- Do not compute it from star_index and do not assign it by position in the list. Do not introduce new criteria, weights, or selection thresholds.
- Do not treat unknown as confirmed non-compliance. Explicitly state unconfirmed requirements in the "Риски и неподтверждённые требования" column.
- The scores must be consistent with this order: the next candidate's score must not be higher than the previous one's. Equal scores are acceptable.
- Reflect the rationale for the score in the existing columns "Почему подходит" and "Риски и неподтверждённые требования".
- The score is formed by the agent based on the data already obtained in previous steps and is not a mart field. Do not request it as a column via MCP.

### Step 6. Return top-k

Return up to `k` most relevant candidates. If there are fewer sufficiently confirmed candidates, return the number actually found and briefly explain the limitation.

#### Position fit score

- For each candidate in the final shortlist, form a position fit score from 0.0 to 10.0 inclusive. Output the score with one decimal place, for example 8.7.
- Form the score based on the comparison already performed: confirmed must-haves, role and domain fit, level fit, confirmed nice-to-haves, and evidence strength.
- Preserve the priority order established above. This is a numeric representation of fit to the specific position.
- Preserve the candidate order obtained by the rules of the step.
- Do not compute it from star_index and do not assign it by position in the list. Do not introduce new criteria, weights, or selection thresholds.
- Do not treat unknown as confirmed non-compliance. Explicitly state unconfirmed requirements in the «Риски и неподтверждённые требования» column.
- The scores must be consistent with this order: the next candidate's score must not be higher than the previous one's. Equal scores are acceptable.
- Reflect the rationale for the score in the existing columns «Почему подходит» and «Риски и неподтверждённые требования».
- The score is formed by the agent based on the data already obtained in previous steps and is not a mart field. Do not request it as a column via MCP.

## Response format for the user

Return a table with the columns.

Add the profile link column only if link_requested=true:
```text
Имя кандидата(full name) | Текущая роль и компания | Почему подходит | Риски и неподтверждённые требования | Star Index | Оценка соответствия позиции | Ссылка на профиль
```

Otherwise, if link_requested=false:
```text
Имя кандидата(full name) | Текущая роль и компания | Почему подходит | Риски и неподтверждённые требования | Star Index | Оценка соответствия позиции
```

**IMPORTANT!** The profile link means: `https://hr.sberbank.ru/platform/recruitment-data-migration-service/stars/experts/{id}`, where `{id}` is the candidate's `id` obtained earlier (at the compact profile collection stage) from the mart dm_special.talent_radar_people.
Do not call the absence of data a minus. Distinguish a confirmed contradiction, a possible risk, and the absence of evidence.

At the end of the answer, as a separate footnote, explicitly disclose the description of the "Оценка соответствия" and "Star Index" columns:
`«Оценка соответствия» показывает насколько внешний кандидат подходит под описанную роль. От 0 до 10. Выполнена на основании публично доступной информации и является предварительной.`
`«Star Index» - это универсальная оценка «общей силы» профиля и опыта кандидата на основании его карьерного трека, образования, публичной активности и репутации. Не привязана к описанию роли на которую ищем кандидатов.`

## Honesty rules (specific to Talent Radar)

Always comply:
- do not invent facts;
- do not claim that experience is absent if it is simply not in the database;
- do not confuse star_index with task fit;
- do not hide that some criteria are not checked by filters (evidence criteria);
- do not return blocked profiles (is_blocked = false);
- do not return current Sber or subsidiary employees (mandatory `current_company` filter);
- address people by id (namesakes are possible);
- do not use birth_year, photo_verified, and is_favorite as candidate quality signals without a separate approved/explicitly set policy.