---
name: get_outstaff_summary
description: "Сводные метрики по сотрудникам на упразднённых должностях: за штатом (более 90 дней), потенциальные заштатники (до 90 дней) и сотрудники, которым скоро за штат (до упразднения не более 15 дней). Используется если человек спрашивает: сколько сотрудников за штатом, сводка по заштатникам, статистика по упразднённым должностям, сколько потенциальных заштатников, кто скоро за штат, сколько заштатников с льготами, сколько декретников за штатом, сколько внешних совместителей за штатом, распределение заштатников по срокам. Для детальных строк используйте скилл get_outstaff_details."
---

# Инструкция для агента: сводные данные по сотрудникам за штатом

## Когда использовать

Если необходимо получить общие метрики по сотрудникам, находящимся за штатом
(на упразднённых должностях):

- всего за штатом (более 90 дней);
- потенциальные заштатники (до 90 дней);
- скоро за штат (до упразднения не более 15 дней).

Варианты ответа добавляют фильтры по категориям: льготники, декретники,
внешние совместители.

Для детальных строк по конкретным сотрудникам используй скилл
`get_outstaff_details`.

---

## Используемый инструмент

- **MCP-сессия:** `heimdall`
- **Инструмент:** `mcp_query`
- **Модель данных:** схема `dm_core`, логическая модель `employee_actual`,
  таблица `dm_core.staff_employee_digital_profile_actual_dm`

Запрос выполняется в режиме агрегата: вместо `columns` передаются `metrics`,
лимит равен 1. Для группировки добавь нужные поля в `columns`.

---

## Базовый запрос (общие метрики)

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "metrics": [
    "outstaff_90_plus_count",
    "outstaff_less_90_count",
    "soon_outstaff_count",
    "outstaff_special_attention_count",
    "outstaff_duration_3_6_count",
    "outstaff_duration_6_12_count",
    "outstaff_duration_1_2_count",
    "outstaff_duration_2plus_count"
  ],
  "limit": 1,
  "filters": {
    "type": "and",
    "conditions": [
      {
        "type": "condition",
        "column": "employee_status",
        "operator": "=",
        "value": "активный"
      }
    ]
  }
}
```

---

## Варианты запросов

### Метрики по сотрудникам за штатом с льготой (включая декрет)

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "metrics": [
    "outstaff_90_plus_count",
    "outstaff_less_90_count",
    "soon_outstaff_count",
    "outstaff_90_plus_with_benefit_count",
    "outstaff_90_plus_decree_count",
    "outstaff_duration_3_6_count",
    "outstaff_duration_6_12_count",
    "outstaff_duration_1_2_count",
    "outstaff_duration_2plus_count"
  ],
  "limit": 1,
  "filters": {
    "type": "and",
    "conditions": [
      {
        "type": "condition",
        "column": "has_active_benefit",
        "operator": "=",
        "value": 1
      },
      {
        "type": "condition",
        "column": "employee_status",
        "operator": "=",
        "value": "активный"
      }
    ]
  }
}
```

### Метрики по сотрудникам за штатом в декрете

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "metrics": [
    "outstaff_90_plus_count",
    "outstaff_less_90_count",
    "soon_outstaff_count",
    "outstaff_duration_3_6_count",
    "outstaff_duration_6_12_count",
    "outstaff_duration_1_2_count",
    "outstaff_duration_2plus_count"
  ],
  "limit": 1,
  "filters": {
    "type": "and",
    "conditions": [
      {
        "type": "condition",
        "column": "is_decree",
        "operator": "=",
        "value": 1
      },
      {
        "type": "condition",
        "column": "employee_status",
        "operator": "=",
        "value": "активный"
      }
    ]
  }
}
```

### Метрики по внешним совместителям за штатом

```json
{
  "schema": "dm_core",
  "logic_model": "employee_actual",
  "metrics": [
    "outstaff_90_plus_count",
    "outstaff_less_90_count",
    "soon_outstaff_count",
    "outstaff_duration_3_6_count",
    "outstaff_duration_6_12_count",
    "outstaff_duration_1_2_count",
    "outstaff_duration_2plus_count"
  ],
  "limit": 1,
  "filters": {
    "type": "and",
    "conditions": [
      {
        "type": "condition",
        "column": "form_work_code",
        "operator": "=",
        "value": "4"
      },
      {
        "type": "condition",
        "column": "employee_status",
        "operator": "=",
        "value": "активный"
      }
    ]
  }
}
```

Во всех вариантах метрики считаются только для сотрудников, удовлетворяющих
фильтру варианта (льготники / декретницы / внешние совместители).

---

## Как читать результат

| Метрика | Что вернуть пользователю |
|---|---|
| `outstaff_90_plus_count` | Сотрудники за штатом |
| `outstaff_less_90_count` | Потенциальные заштатники |
| `soon_outstaff_count` | Скоро за штат |
| `outstaff_90_plus_with_benefit_count` | За штатом с льготами (включая декрет) |
| `outstaff_90_plus_decree_count` | За штатом в декрете |
| `outstaff_special_attention_count` | Особое внимание (не внешние совместители, без льгот, не в декрете и нет запланированных кадровых мероприятий перевод или увольнение) |
| `outstaff_duration_3_6_count` | За штатом 3-6 месяцев |
| `outstaff_duration_6_12_count` | За штатом 6-12 месяцев |
| `outstaff_duration_1_2_count` | За штатом 1-2 года |
| `outstaff_duration_2plus_count` | За штатом более 2 лет |

---

## Правила интерпретации

### Определения метрик

- `outstaff_90_plus_count` — сотрудники на упразднённых должностях, которые
  упразднены более 90 дней;
- `outstaff_less_90_count` — сотрудники на упразднённой должности не более
  90 дней (потенциальные заштатники, ещё не достигли порога 90 дней);
- `soon_outstaff_count` — сотрудники, у которых до упразднения должности
  осталось не более 15 дней.

Критерий заштатника: сотрудник считается заштатником, если находится на
упразднённой должности более 90 дней.

### Что выводить пользователю

- Выводим общие метрики — «Сотрудники за штатом», «Потенциальные
  заштатники», «Скоро за штат» и отдельно с фильтрами по наличию льготы,
  декрету и внешним совместителям.
- Показываем пользователю статистику по сроку нахождения за штатом —
  «За штатом 3-6 месяцев», «За штатом 6-12 месяцев», «За штатом 1-2 года»,
  «За штатом более 2 лет».
- Метрика «Особое внимание» применяется только к сотрудникам за штатом
  (сотрудникам на упразднённой должности более 90 дней).
- Для внешних совместителей, льготников и декретников метрика «Особое
  внимание» всегда равна 0, так как она исключает таких сотрудников.

### Вложенность метрик — НЕ суммируй

- Вложенность: `outstaff_90_plus_decree_count` ≤
  `outstaff_90_plus_with_benefit_count` ≤ `outstaff_90_plus_count`.
  Не складывай эти значения — они пересекаются и вложены друг в друга.
  Суммирование приведёт к завышенным значениям.
- Не суммируй метрики `outstaff_90_plus_count` и `outstaff_less_90_count`.
  Пользователь не хочет видеть общее число сотрудников на упраздненных
  должностях.

---

