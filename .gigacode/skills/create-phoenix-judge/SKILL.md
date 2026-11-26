---
name: create-phoenix-judge
description: Скилл для генерации скелета новых агентов и judge-функций в Phoenix experiments. Автоматически создаёт 5 файлов с 3 декораторами регистрации (@register_agent_function, @register_judge_function, @register_dataset_judge_function). Покрывает полный цикл: запуск агента через HTTP POST, мониторинг завершения через Phoenix span по message_id, per-example оценка через Span dataclass (tool accuracy, token usage, latency, task success, semantic accuracy), и whole-run агрегация verdicts.
metadata:
  version: 1.0.0
  display_name: create-phoenix-judge
  source: Локальная разработка (см. Provenance)
  network_requirement: none
  provenance: >
    Исходный код находится в app/phoenix/experiments/custom/_templates/*.py.
    Все операции локальные.
    Сетевые вызовы отсутствуют.
    Зависимости: нет (только стандартная библиотека Python 3).
---


# Phoenix Experiment Agent/Judge Scaffolding

Add new span-based agents and judges to Phoenix experiments in under 10 minutes.

## Overview

Phoenix experiments evaluate LLM agents against datasets using **agent functions** (which launch agents and return span data) and **judge functions** (which analyze spans and return evaluations). This skill provides a scaffold generator that produces all 5 required files with 3 auto-registered functions — ready to import, configure, and use.

## Files Created

```
custom/<agent_name>/
├── __init__.py              # 3 decorators: agent + judge + dataset judge
├── config.py                # Pydantic settings with env var support
├── runner.py                # Agent HTTP launcher + span status monitor
├── span_judge.py            # Per-example judge using Span dataclass
└── dataset_judge.py         # Whole-run aggregation (runs once per experiment)
```

## Quick Start

```bash
cd app/phoenix/experiments/custom

# Basic usage
python generate_agent.py my_agent

# With custom env prefix (controls env var names)
python generate_agent.py my_agent --env-prefix MY_AGENT

# Custom output directory
python generate_agent.py my_agent --output ../my-agent-dir
```

## Registration

After generation, register the module:

```python
# app/phoenix/experiments/custom/__init__.py
from . import my_agent  # noqa: F401
```

Restart Phoenix — functions auto-register on import.

## Auto-Registered Functions

| Decorator | Function Name | Role |
|-----------|--------------|------|
| `@register_agent_function("agent-<name>")` | `agent-<name>` | Launches agent, monitors via span, returns `{"output": {...}}` |
| `@register_judge_function("judge-<name>-span")` | `judge-<name>-span` | Per-example evaluation using Span dataclass |
| `@register_dataset_judge_function("judge_dataset_<name>")` | `judge_dataset_<name>` | Whole-run aggregation after all examples |

## Configuration

Edit `config.py` — set agent HTTP endpoint and model:

```python
class MyAgentSettings(AgentSettings):
    HOST: str = "http://my-agent:8080"
    REACT_URL: str = "/api/v1/agent/call"
    MODEL: str = "glm-5.1"
    DEFAULT_PERSON_ID: str = "e6628524-..."
```

Or via environment variables:

```bash
export MY_AGENT_HOST="http://my-agent:8080"
export MY_AGENT_CALL_URL="/api/v1/agent/call"
export MY_AGENT_MODEL="glm-5.1"
```

## Running Experiments

```bash
curl -X POST /ext/api/experiments/run \
  -H "Content-Type: application/json" \
  -d '{
    "dataset_name": "my-dataset",
    "agent_function": "agent-my_agent",
    "judge_function": "judge-my_agent-span",
    "dataset_judge_function": "judge_dataset_my_agent",
    "splits": ["test"]
  }'
```

## What Each Component Does

### Agent Function (runner.py)

Launches the agent via HTTP POST, then monitors completion through Phoenix span status by `message_id`.

**Flow:**
1. POST `{message, model, chatId, messageId}` to agent endpoint
2. Poll `phoenix.spans` for `metadata.message_id` matching
3. Return `{"output": {message_id, status, span, events, timed_out}}`

### Judge Function (span_judge.py)

Analyzes agent execution output via the `Span` dataclass (`app.phoenix.experiments.span_dataclass`).

**Extracted metrics:**
- `task_success` — span finished without timeout/error
- `tool_selection_accuracy` — precision vs expected tools from dataset
- `steps_per_task` — number of ReAct steps
- `latency_seconds` — span duration
- `token_usage` — prompt/completion/total tokens
- `semantic_accuracy` — LLM-as-judge score (optional, via GigaChat)
- `semantic_explanation` — LLM reasoning

**Returns:**
```python
{
    "label": "correct" | "partial" | "incorrect",
    "score": 1.0 | 0.5 | 0.0,
    "explanation": "task_success=1, tool_accuracy=1.0, ...",
    "metrics": {...},
}
```

### Dataset Judge (dataset_judge.py)

Runs **once** per experiment after all examples complete. Aggregates per-example verdicts.

**Returns:**
```python
{
    "total_examples": 10,
    "average_score": 0.85,
    "pass_rate": 0.8,
    "label_counts": {"correct": 6, "partial": 2, "incorrect": 2},
    "missed_expected_tools": ["describe_model"],
    "examples_report": [...],
    "summary": "Run 'experiment': 10 examples, avg score 0.85...",
}
```

## Customization

### Disable semantic judge (skip GigaChat call)

```python
# In span_judge.py
@register_judge_function("judge-my_agent-span")
async def judge_my_agent_span(*, input, output, **_):
    return await judge_span(
        input=input,
        output=output,
        expected_tools=_extract_expected_tools(output, input),
        enable_semantic_judge=False,  # Skip LLM call
    )
```

### Custom prompts

```python
# In span_judge.py
CUSTOM_SEMANTIC_SYSTEM_PROMPT = "Ты эксперт по оценке SQL-запросов..."
CUSTOM_SEMANTIC_PROMPT_TEMPLATE = "Вопрос: {input_data}\nSQL: {output_data}\n..."
```

### Custom scoring logic

Override `_compute_final_score()` in `span_judge.py` to change correct/partial/incorrect thresholds.

## Template Architecture

```
custom/_templates/                  # Source templates (don't edit directly)
├── _template_config.py            # SpanAgentSettings base class
├── _template_runner.py            # Generic call_agent_by_span()
├── _template_span_reader.py       # get_span_status_by_message_id()
├── _template_span_judge.py        # judge_span() with all metrics
├── _template_dataset_judge.py     # aggjudge_dataset()
└── README.md                      # Template docs
```

Generated files import from `_templates/` for shared logic. When updating templates, re-run `generate_agent.py` to propagate changes.

## Data Flow

```
Dataset Record: {"input": {"question": "..."}, "output": {"tools_calls_expected": [...]}}
    │
    ▼
Agent Function (agent-<name>)
    │  1. POST to agent URL → get message_id
    │  2. Poll Phoenix span by metadata.message_id
    │  3. Wait for status != "running"
    │  4. Return {"output": {message_id, status, span, events, timed_out}}
    │
    ▼
Service._run_example()
    │  task_output = {message_id, span, status, timed_out}
    │  judge_output = {**task_output, **output_data}
    │               = {message_id, span, status, timed_out, tools_calls_expected: [...]}
    │
    ▼
Judge Function (judge-<name>-span)
    │  1. _build_span(output) → Span.from_dict(...)
    │  2. span.get_metrics(expected_tools) → dict
    │  3. span.get_task_success(), span.get_tool_selection_accuracy(...)
    │  4. (Optional) LLM-as-judge for semantic_accuracy
    │  5. Return {label, score, explanation, metrics}
    │
    ▼
Service writes annotation to DB
    │  {"label": "correct", "score": 1.0, "explanation": "...",
    │   metadata_: {"metrics": {...}}}
```

## Troubleshooting

### Function not registered

```bash
# Check if functions are listed
curl "http://localhost:6006/ext/api/experiments/functions"
```

### Span not found

- Ensure agent writes `message_id` to span metadata: `agent.config['metadata']['message_id']`
- Check Phoenix DB:
  ```sql
  SELECT attributes->'metadata'->>'message_id' FROM phoenix.spans WHERE ...
  ```

### Timeout waiting for span

- Increase `TASK_RUN_TIMEOUT` in `config.py`
- Check `SPAN_POLL_INTERVAL` — too low causes DB load, too high delays detection

### Auth errors

- Runner uses `aget_httpx_client` with JWT token
- For local testing, remove auth: `httpx.AsyncClient(verify=False)`

## Key Files

| File | Path | Purpose |
|------|------|---------|
| Generator | `app/phoenix/experiments/custom/generate_agent.py` | Scaffold script |
| Templates | `app/phoenix/experiments/custom/_templates/` | Source templates |
| Judge functions | `app/phoenix/experiments/custom/` | Generated agent modules |
| Span dataclass | `app/phoenix/experiments/span_dataclass.py` | `Span` class for metric extraction |
| Span reader | `app/phoenix/experiments/span_reader.py` | DB query for span status |
| Registration | `app/phoenix/experiments/functions.py` | `@register_*` decorators |
