---
name: answer-widget-team-profile
description: "Mandatory widget formatting of the consolidated profile report answer: before delivering to the user a report about role, grade, segment, agile team, managers, skills, professional tags, interests, or linear structure, read this skill and format the answer with the canonical set of json-render blocks according to the reference template. Not applied to clarifying questions about ambiguity of a full name or unit, refusals, short factual answers with a single number, paginated listings of team composition, and tag top-counts."
---

# Formatting the profile report with widgets (answer-widget-team-profile)

The skill defines **how the consolidated profile report answer looks**. A profile report is an answer that consolidates mart data about an employee's role, grade, segment, agile team, managers, skills, professional tags, interests, and linear structure («покажи профиль моей роли», «какой у меня портрет в команде», «профиль роли этого сотрудника»).

## When to apply

Apply the skill when, based on the tool data, you are preparing a consolidated profile report for the user: their own role or another employee's role, grade and segment, agile team and managers, skills, professional tags, interests, linear structure.

**Do not apply** to answers without consolidated profile data: clarifications of ambiguity of a full name or unit, refusals («данные недоступны», «нет прав»), short factual answers with a single number («сколько подчинённых» — «7 человек»), paginated listings of team composition, tag top-counts with coverage. Such answers remain in the formats of their own track skills, without the canonical blocks.

## Mandatory procedure

1. Collect the mart data according to the methodology of the track skill (`get_employee_team`, `get_team_tag_profile`) — composition, role, grade, tags, interests, structure.
2. **Read this SKILL.md before forming the answer text.**
3. Form the answer in the canonical composition and order (see "Answer composition").

Do not draw the profile with manual markdown tables: the report part always goes as a set of `json-render` blocks.

## Canonical multi-block template

The canon is a fixed sequence of blocks. Copy the frame of each block as is, replacing the `<...>` placeholders with real mart values. The components are limited to the template's list (**Dashboard, DashboardItem, Stack, Title, Text, Tag, Table**): do not invent other components and do not change the nesting.

Block 1 — a Dashboard of six tiles (goes right after the lead phrase, without a section heading):

```json
{
  "root": "dashboard",
  "elements": {
    "dashboard": {
      "type": "Dashboard",
      "props": { "title": "Профиль роли — <ФИО сотрудника>" },
      "children": [
        "roleItem",
        "gradeItem",
        "segmentItem",
        "teamItem",
        "managerItem",
        "skillsItem"
      ]
    },
    "roleItem": {
      "type": "DashboardItem",
      "props": { "title": "Роль в команде", "size": "s" },
      "children": ["role"]
    },
    "role": {
      "type": "Stack",
      "props": { "gap": 4 },
      "children": ["roleValue", "roleLabel"]
    },
    "roleValue": {
      "type": "Title",
      "props": { "text": "<роль в команде>", "size": "H3" }
    },
    "roleLabel": {
      "type": "Text",
      "props": { "text": "<должность>", "variant": "body2Regular" }
    },
    "gradeItem": {
      "type": "DashboardItem",
      "props": { "title": "Грейд", "size": "s" },
      "children": ["grade"]
    },
    "grade": {
      "type": "Stack",
      "props": { "gap": 4 },
      "children": ["gradeValue", "gradeLabel"]
    },
    "gradeValue": {
      "type": "Title",
      "props": { "text": "<грейд>", "size": "H2" }
    },
    "gradeLabel": {
      "type": "Text",
      "props": { "text": "категория: <категория грейда>", "variant": "body2Regular" }
    },
    "segmentItem": {
      "type": "DashboardItem",
      "props": { "title": "Сегмент", "size": "s" },
      "children": ["segment"]
    },
    "segment": {
      "type": "Stack",
      "props": { "gap": 4 },
      "children": ["segmentValue", "segmentLabel"]
    },
    "segmentValue": {
      "type": "Title",
      "props": { "text": "<сегмент>", "size": "H3" }
    },
    "segmentLabel": {
      "type": "Text",
      "props": { "text": "Функциональный блок: <функциональный блок>", "variant": "body2Regular" }
    },
    "teamItem": {
      "type": "DashboardItem",
      "props": { "title": "Команда (agile)", "size": "s" },
      "children": ["team"]
    },
    "team": {
      "type": "Stack",
      "props": { "gap": 4 },
      "children": ["teamValue", "teamLabel"]
    },
    "teamValue": {
      "type": "Title",
      "props": { "text": "<название команды>", "size": "H3" }
    },
    "teamLabel": {
      "type": "Text",
      "props": { "text": "Кластер: <кластер> · Трайб: <трайб>", "variant": "body2Regular" }
    },
    "managerItem": {
      "type": "DashboardItem",
      "props": { "title": "Руководители", "size": "s" },
      "children": ["managers"]
    },
    "managers": {
      "type": "Stack",
      "props": { "gap": 8 },
      "children": ["linearManager", "agileManager"]
    },
    "linearManager": {
      "type": "Text",
      "props": { "text": "Линейный: <линейный руководитель>", "variant": "body2Regular" }
    },
    "agileManager": {
      "type": "Text",
      "props": { "text": "Agile: <agile-руководитель>", "variant": "body2Regular" }
    },
    "skillsItem": {
      "type": "DashboardItem",
      "props": { "title": "Профессиональные навыки", "size": "full" },
      "children": ["skillsTags"]
    },
    "skillsTags": {
      "type": "Stack",
      "props": { "direction": "row", "gap": 4 },
      "children": ["skillTag1", "skillTag2", "skillTag3", "skillTag4", "skillTag5", "skillTag6"]
    },
    "skillTag1": {
      "type": "Tag",
      "props": { "label": "<навык 1>", "color": "<цвет>", "size": "s" }
    },
    "skillTag2": {
      "type": "Tag",
      "props": { "label": "<навык 2>", "color": "<цвет>", "size": "s" }
    },
    "skillTag3": {
      "type": "Tag",
      "props": { "label": "<навык 3>", "color": "<цвет>", "size": "s" }
    },
    "skillTag4": {
      "type": "Tag",
      "props": { "label": "<навык 4>", "color": "<цвет>", "size": "s" }
    },
    "skillTag5": {
      "type": "Tag",
      "props": { "label": "<навык 5>", "color": "<цвет>", "size": "s" }
    },
    "skillTag6": {
      "type": "Tag",
      "props": { "label": "<навык 6>", "color": "<цвет>", "size": "s" }
    }
  }
}
```

Block 2 — the "### Профессиональные теги" section with a Tag-stack block (one tag node per each value from `prof_tags`; the frame has three — create as many as needed):

```json
{
  "root": "profTags",
  "elements": {
    "profTags": {
      "type": "Stack",
      "props": { "direction": "row", "gap": 4 },
      "children": ["profTag1", "profTag2", "profTag3"]
    },
    "profTag1": {
      "type": "Tag",
      "props": { "label": "<профессиональный тег 1>", "color": "<цвет>", "size": "s" }
    },
    "profTag2": {
      "type": "Tag",
      "props": { "label": "<профессиональный тег 2>", "color": "<цвет>", "size": "s" }
    },
    "profTag3": {
      "type": "Tag",
      "props": { "label": "<профессиональный тег 3>", "color": "<цвет>", "size": "s" }
    }
  }
}
```

Block 3 — the "### Интересы" section with a Tag-stack block (one node per each value from `interests`):

```json
{
  "root": "interests",
  "elements": {
    "interests": {
      "type": "Stack",
      "props": { "direction": "row", "gap": 4 },
      "children": ["interestTag1", "interestTag2", "interestTag3"]
    },
    "interestTag1": {
      "type": "Tag",
      "props": { "label": "<интерес 1>", "color": "<цвет>", "size": "m" }
    },
    "interestTag2": {
      "type": "Tag",
      "props": { "label": "<интерес 2>", "color": "<цвет>", "size": "m" }
    },
    "interestTag3": {
      "type": "Tag",
      "props": { "label": "<интерес 3>", "color": "<цвет>", "size": "m" }
    }
  }
}
```

Block 4 — the "### Линейная структура" section with a Table block (a row per each populated `oshs_level_*_unit_name_main` level):

```json
{
  "root": "table",
  "elements": {
    "table": {
      "type": "Table",
      "props": {
        "columns": [
          {
            "key": "level",
            "title": "Уровень",
            "type": "number",
            "align": "right"
          },
          {
            "key": "unit_name",
            "title": "Подразделение"
          }
        ],
        "rows": [
          {
            "level": <номер уровня>,
            "unit_name": "<подразделение уровня 1>"
          },
          {
            "level": <номер уровня>,
            "unit_name": "<подразделение уровня 2>"
          },
          {
            "level": <номер уровня>,
            "unit_name": "<подразделение уровня 3>"
          }
        ]
      }
    }
  }
}
```

Choose varied Tag colors from the palette (green, blue, cyan, purple, orange, grey, yellow, teal, magenta, red); repeating adjacent colors is acceptable; colors have no semantic rules.

In the answer, format each block as a fenced block labeled `json-render`. Do not wrap the blocks in additional quotes and do not add explanations inside them.

## Data rules

- **Accuracy.** Transfer values from the tool responses as is: without recalculation, without rounding, without recategorization, and without renaming to your own taste. Grade 11 stays 11, unit names and full names — word for word. Pass the numeric structure level as a number, not a string.
- **Categories are not mixed.** Skills (`skills_name`), professional tags (`prof_tags`), and interests (`interests`) are three different mart columns. Skills are shown in the "Профессиональные навыки" tile, prof tags — in the "Профессиональные теги" block, interests — in the "Интересы" block. Mixing categories into one list is forbidden.
- **Empty values are not shown.** Missing tags, unpopulated managers, and empty structure levels do not appear in the answer.
- **Service information is hidden.** Do not show the user the mart field names (`employee_full_name`, `grade_level`, `prof_tags`, `skills_name`, `interests`, `oshs_level_*_unit_name_main`) or identifiers (`person_id`, `employee_id`) — only user-facing values and wording. The "Грейд" tile shows the grade and category without the column name.

## Skipping sections

- **A section without data is omitted together with its heading.** No prof tags — no "### Профессиональные теги" section and its block; no team data — no "### О команде" section. The composition and order of the remaining sections do not change.
- **The skills tile is limited.** Up to 6 Tags — the first skills in mart order. The full skills list is not expanded in the tile; professional tags and interests are shown as full lists in their own blocks.
- **The "Руководители" tile.** Add the "Agile: …" line only if the agile manager is populated in the mart; do not invent missing managers.
- **The team caption.** Include the "Кластер: …" and "Трайб: …" parts based on whether they are populated, separating them with " · ".
- Skipping sections does not produce other formats: the remaining blocks stay canonical `json-render`.

## Answer composition

Assemble the answer strictly in this order:

1. Lead phrase — one sentence about whose profile you are showing («Вот профиль вашей роли в команде»).
2. A `json-render` Dashboard block of six tiles (without a section heading above it).
3. The "### Профессиональные теги" section with a Tag-stack block.
4. The "### Интересы" section with a Tag-stack block.
5. The "### Линейная структура" section with a Table block.
6. The "### О команде" section with a final text about the team (headcount, product owner, competency profile — based on mart data, without speculation).
