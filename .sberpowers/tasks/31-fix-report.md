# Отчёт задачи 3.1-FIX: SSE-паритет судьи + телеметрия deny-вердиктов

- **Статус: DONE**
- **Коммит:** `31da4c6f` — `fix(03): SSE-паритет судьи + досыл deny-вердиктов в телеметрию (REQ-01,03,08)` поверх `ea886a87`, ветка `wave3-task31` (worktree `.worktrees/wave3-task31`)
- **Файлы:** 6 изменённых, 231 insertions / 32 deletions. `stateful_routing.py` правок не требовал (фигурировал только в ruff-списке брифа) — не менялся и не стейджился.

## Изменения по пунктам ревью

| Finding | Файл | Решение |
|---|---|---|
| F1 Critical | `services_sse.py`, `service.py`, `internet_judge.py` | Единая точка сборки `merge_request_interceptor_configs(base, stateful, chat_id, message_id, user_question)` — вызывается из ОБЕИХ копий `_generate_agent_stream`. Судья теперь в SSE-канале (раньше: только polling). |
| F2 Important | `streaming.py`, `internet_judge.py` | `_sent_judge_call_ids` собирается в `on_tool_end` (id уже ушедших tool-шагов); после цикла событий `send_unsent_judge_verdicts(...)` досылает вердикты deny/timeout/error отдельными шагами stat-feedback (`step_type="tool"`, `step_name="mcp_internet"`). Общий `generate_agent_stream` закрывает polling и SSE сразу. |
| F3 Minor | `internet_judge.py`, `interceptor.py` | `attach_internet_judge_metadata(step_metadata: dict[str, Any] | None, interceptor_config: InterceptorConfig | None)`; поле `internet_gate: InternetJudgeGate | None` (TYPE_CHECKING, без рантайм-цикла); gate-параметр хелпера — `InternetJudgeGate | None` (самодостаточный вариант из брифа). |
| F4 Minor | `internet_judge.py`, `streaming.py`, тесты | Хелпер `extract_judge_tool_call_id(tool_output, fallback)`; прод-код `on_tool_end` использует его; интеграционный тест пиннит прод-паттерн вместо дублирования. |
| F5 Minor | тесты | Явный `monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)` во всех перечисленных брифом тестах, вызывавших `build_internet_judge_config` без контроля флага. |

## Ход работы (TDD)

### Baseline (до правок)
```
pytest tests/test_deepagent_internet_judge_wiring.py -v      -> 10 passed
pytest tests/test_deepagent_internet_judge.py \
       tests/test_deepagent_interceptor_async.py \
       tests/test_deepagent_session_id.py                     -> 36 passed
ruff check <7 файлов>                                        -> 8 findings (все pre-existing в streaming.py: I001 + 7×F401)
```

### RED
Тест-файл расширен (9 новых тестов: 3 merge_request + 4 send_unsent + 2 extract_id; + F5-флаги; + пиннинг 6f). Запуск:
```
ImportError: cannot import name 'extract_judge_tool_call_id'
  from 'app.services.react.deepagent.internet_judge'
collected 0 items / 1 error
```
Новые хелперы отсутствуют -> RED подтверждён.

### GREEN (после реализации)
```
pytest tests/test_deepagent_internet_judge_wiring.py -v      -> 19 passed (10 прежних + 9 новых)
pytest <регресс зоны: internet_judge + interceptor_async + session_id> -> 36 passed
суммарно одним прогоном                                       -> 55 passed
pytest --collect-only -q wiring + session_id                  -> 36 tests collected (импорты service/streaming/services_sse в conftest-окружении проходят)
ruff check <7 файлов>                                         -> 8 findings == baseline, 0 новых
```
Проверка I001 (`--select I001 --diff streaming.py`): предлагаемый фикс затрагивает только предсуществующие строки (direct_streaming/subagents/message_converter), новый блок импортов не в диффе.

Повторный полный прогон ПОСЛЕ коммита: `git status --short` пусто, 55 passed.

## Self-review по критериям выхода брифа

- **F1 закрыт:** SSE-хэрнесс (~строка 374 services_sse.py) и polling собирают конфиг одним хелпером; disabled-ветка и all-None покрыты юнит-тестами; при выключенном флаге стек middleware не меняется (merge с judge_config=None — no-op). ✓
- **F2 закрыт:** deny/timeout/error вердикты доходят до stat-feedback отдельными шагами; allow-вердикты не дублируются (их id в sent-set, они уже обогатили step_metadata tool-шага через `attach_internet_judge_metadata`). ✓
- **Вставка досыла — только нормальный выход из цикла:** блок стоит ПОСЛЕ emergency-блока 7.7 (orphaned tools -> yield_error_response -> return) и ДО секции 8 (сохранение ответа + финальный feedback) и cleanup. В finally/except-путях ошибок не участвует. ✓
- **Сигнатура `_send_cycle_feedback` сверена:** все kwargs вызова существуют; шаги получают возрастающие sequence (base `_tool_step_sequence` + 1, +2...), не конфликтующие с финальным message-шагом секции 8. ✓
- **Изоляция сбоев телеметрии двойная:** try/except + `logger.exception` в `send_unsent_judge_verdicts`, плюс внутренний try/except в `_send_cycle_feedback` — стрим не ломается. ✓
- **Импорты чисты:** `merge_interceptor_configs`/`build_internet_judge_config` больше не импортируются в `service.py`/`services_sse.py` (grep по deepagent/: остались только определение в stateful_routing и ленивый импорт внутри хелпера). ✓
- **F3–F5:** аннотации типизированы; lookup-id пиннится прод-хелпером в интеграционном тесте; явные флаги во всех перечисленных тестах. ✓

## Отклонения от буквы брифа (согласованы с его приоритетами)

1. **`Awaitable` импортирован из `collections.abc` внутри TYPE_CHECKING**, а не из `typing` в общей строке (как предполагал бриф): буквальный вариант давал НОВОЕ замечание ruff `UP035 Import from collections.abc instead: Awaitable`. Критерий «0 новых замечаний» брифа приоритетнее; паттерн идентичен существующему в `interceptor.py`. `Callable` оставлен в `typing` (не флагуется).
2. **stateful_routing.py не менялся**: бриф не предписывает ему изменений (он был только в ruff-списке), поэтому стейджено 6 файлов вместо 7.

## Сомнения / известные границы

1. **Терминальные файловые тулы** (`convert_to_pptx` и др.) завершают стрим `return`-ом ИЗ цикла — на этом пути досыл вердиктов не выполняется. До фикса таких шагов телеметрии не было вовсе (не регресс); бриф расширение early-return не предписывает. Вероятность мала: судья блокирует/пропускает mcp_internet ДО вызова, а pptx-тулы — другой класс тулов.
2. **`test_send_unsent_empty_verdicts_noop`** вызывает `build_internet_judge_config` без контроля флага — оставлен дословно по сниппету брифа (F5-список этот новый тест не включает; default `MCP_INTERNET_JUDGE_ENABLED=True`; переменная окружением не переопределяется).
3. **`step_input=None`** в `_judge_feedback`: параметр `_send_cycle_feedback` типизирован `str`, но бриф явно предписывает None (stat_service корректно принимает). Отмечено как косметическая нетипизированность, рантайм покрыт тестами хелпера.

## Команды для воспроизведения

```bash
VENV=/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.venv/bin/python
$VENV -m pytest tests/test_deepagent_internet_judge_wiring.py -v        # 19 passed
$VENV -m pytest tests/test_deepagent_internet_judge.py \
       tests/test_deepagent_interceptor_async.py \
       tests/test_deepagent_session_id.py                                # 36 passed
$VENV -m pytest --collect-only -q \
       tests/test_deepagent_internet_judge_wiring.py \
       tests/test_deepagent_session_id.py                                # 36 collected
/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.venv/bin/ruff check \
       app/services/react/deepagent/{internet_judge,interceptor,stateful_routing,service,services_sse,streaming}.py \
       tests/test_deepagent_internet_judge_wiring.py                     # 8 pre-existing, 0 новых
```
