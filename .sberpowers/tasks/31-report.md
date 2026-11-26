# Отчёт задачи 3.1: Wiring — серверная регистрация mcp_internet-judge + вердикты в step_metadata

**Статус: DONE_WITH_CONCERNS** (одна обоснованная минимальная правка буквы брифа — см. раздел «Ключевое отклонение»)
**Коммит:** `ea886a87` — `feat(03): серверная регистрация mcp_internet-judge + вердикты в step_metadata (REQ-01,03,08)` (ветка `wave3-task31`, от b9d804a9)
**REQ:** REQ-mcp-internet-judge-01, -03, -08

## Что реализовано

| # | Файл | Правка |
|---|------|--------|
| 3a | `app/services/react/deepagent/interceptor.py` | Поле `internet_gate: Any \| None = None` последним полем `InterceptorConfig` + `self.internet_gate = None` в `clear()` |
| 3b | `app/services/react/deepagent/stateful_routing.py` | `internet_gate=extra.internet_gate or base.internet_gate` в конструктор merged-конфига `merge_interceptor_configs` |
| 3c | `app/services/react/deepagent/internet_judge.py` | Два helper'а: `build_internet_judge_config(chat_id, message_id, user_question) -> InterceptorConfig \| None` (None при выключенном флаге; регистрирует `mcp_internet` → `ToolInterceptor(pre_execute=gate.pre_execute)`, кладёт gate в `config.internet_gate`) и `attach_internet_judge_metadata(step_metadata, interceptor_config, tool_name, tool_call_id)` (no-op для не-mcp_internet / отсутствующего gate; добавляет `internet_judge.{verdict,reason}`). TYPE_CHECKING-блок дополнен импортом `InterceptorConfig` |
| 3d | `app/services/react/deepagent/service.py` | В `_generate_agent_stream`: `judge_config = build_internet_judge_config(request.chat_id, request.message_id, request.message)`; `merged_config = merge(merge(self.interceptor_config, stateful_config), judge_config)` — судейский конфиг последним (extra), его `internet_gate` побеждает. Импорт добавлен |
| 3e | `app/services/react/deepagent/streaming.py` | В обработчике `on_tool_end` аргумент `_send_cycle_feedback(step_metadata=...)` заменён на `attach_internet_judge_metadata(_current_tool_input, interceptor_config, tool_name, <lookup-id>)`. Импорт добавлен |
| — | `tests/test_deepagent_internet_judge_wiring.py` | Создан: 8 тестов брифа дословно + 1 интеграционный (см. ниже) |

Выключенный флаг: `build_internet_judge_config` → None; `merge_interceptor_configs(base, None)` возвращает base как есть → стек middleware не меняется, поведение неотличимо от текущего.

## Ключевое отклонение от буквы брифа (обоснование)

Бриф (шаг 3e) предписывал передавать в `attach_internet_judge_metadata` переменную
`_current_tool_call_id`. Live-проверкой стека (deepagents 0.7.14 + langchain 1.4.2 +
langgraph; скрипт со scripted-моделью и реальным вызовом тула через
`InterceptorMiddleware`) установлено:

1. Судья ключует вердикты **реальным** `tool_call["id"]` из `ToolCallRequest`
   (`gate.verdicts['call_REAL_1']`).
2. В стриминге `_current_tool_call_id = f"call_{uuid4().hex[:20]}"` — **синтетический**,
   генерируется независимо; реальный id в `on_tool_start` недоступен вовсе
   (`data.input` = только args).
3. Реальный id доступен в `on_tool_end` из ToolMessage-выхода:
   `data.output = ToolMessage(..., tool_call_id='call_REAL_1')`.

Литеральная реализация по брифу дала бы вечный miss при lookup → REQ-08 молча не
работал бы в рантайме при зелёных юнит-тестах (в тестах id консистентны внутри
сценария). Поэтому внесена минимальная правка в той же точке (streaming.py,
перед вызовом `_send_cycle_feedback`):

```python
# REQ-08: судья ключует вердикты по РЕАЛЬНОМУ tool_call_id ...
_judge_call_id = getattr(tool_output, "tool_call_id", None) or _current_tool_call_id
```

— и в `attach_internet_judge_metadata` передаётся `_judge_call_id` вместо
`_current_tool_call_id`. Бриф сам требует «сверь фактические имена/позиции по
коду» и допускает канонизацию; семантика REQ сохранена (вердикты реально
доезжают до step_metadata). Синтетический `call_id` кадра НЕ тронут — он
продолжает использоваться для персистенции истории сабагентов (строки 720+),
его замена сломала бы чужую функциональность.

**Охрана регрессии:** добавлен 9-й интеграционный тест
`test_real_tool_call_id_reaches_step_metadata` (scripted model →
`create_deep_agent` + middleware с судьёй, LLM замокан): фиксирует, что (а)
реальный id извлекается из ToolMessage-выхода on_tool_end, (б) verdict найден
lookup'ом, (в) step_metadata обогащается `{"internet_judge": {...}}`.

## Команды и результаты

### TDD RED (до реализации)

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge_wiring.py -v
→ ImportError: cannot import name 'attach_internet_judge_metadata' from
  '...internet_judge.py' — collection error (helpers отсутствуют). Ожидаемый RED.
```

### TDD GREEN (после реализации)

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge_wiring.py -v
→ 9 passed (8 из брифа + 1 интеграционный), 0 failed
```

### Регресс зоны

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py \
    tests/test_deepagent_interceptor_async.py tests/test_deepagent_session_id.py -v
→ 36 passed
Совместный прогон с wiring: 46 passed, 0 failed
```

### Смоук импорта service/streaming

Прямая команда брифа (`python -c "import ..."`) на этой машине невозможна без
бегущего event-loop (module-level DB-pool `get_db_pool()` в
`app/services/react/services.py` — предсуществующее) и вешает процесс на exit в
офлайне (pool-воркеры ретраят коннект; exit=124 по таймауту уже ПОСЛЕ завершения
скрипта). Проведено эквивалентно:

```
python -I /tmp/smoke_imports_31b.py   # asyncio.run + import обоих модулей + маркер
→ маркер "SMOKE OK" записан ПОСЛЕ успешного импорта; 0 ImportError/Traceback
.venv/bin/python -m pytest --collect-only tests/test_deepagent_internet_judge_wiring.py
→ 10 tests collected (импорт цепочки deepagent/__init__ → service → streaming без ошибок)
```

### Ruff

```
.venv/bin/ruff check --output-format=concise \
  interceptor.py stateful_routing.py internet_judge.py service.py tests/..._wiring.py
→ All checks passed!
```

`ruff check app/services/react/deepagent/` целиком даёт замечания в непроходимых
файлах (direct_streaming.py, mcp_bootstrap.py, restricted_backend.py,
streaming_errors.py) и 8 замечаний в streaming.py. Stash-сравнение с BASE
(b9d804a9) доказало: все 8 streaming.py замечаний (I001 + 7×F401: AIMessage/
HumanMessage/ToolMessage/finalize_stream/report_pass_through_error/EventType/
AsyncGenerator — используются только в комментариях) **предсуществующие**, мой
дифф не добавил ни одного. Единственное моё новое замечание (UP037, кавычки
аннотации `"InterceptorConfig | None"` из текста брифа) исправлено — бриф
явно разрешает ruff-канонизацию аннотаций.

## Изменённые файлы

- `app/services/react/deepagent/interceptor.py` (+4)
- `app/services/react/deepagent/stateful_routing.py` (+1)
- `app/services/react/deepagent/internet_judge.py` (+41/-1, вкл. TYPE_CHECKING-импорт)
- `app/services/react/deepagent/service.py` (+13/-1)
- `app/services/react/deepagent/streaming.py` (+11/-1)
- `tests/test_deepagent_internet_judge_wiring.py` (новый, 222 строки)

Итого: 6 файлов, 288 insertions, 3 deletions (по коммиту).

## Находки self-review

- Все правки брифа внесены: поле+clear ✓, merge internet_gate ✓, два helper'а ✓,
  серверная регистрация в service ✓ (judge-конфиг последним/extra → его gate
  побеждает в merge), step_metadata в streaming ✓.
- deny не пускает handler (assert handler_called is False; статус error; reason
  в content) ✓; allow прозрачен (handler вызван, вердикт записан по id) ✓; merge
  сохраняет gate поверх stateful-подобного base ✓; None-base passthrough ✓.
- Распространение на routing-сабагентов уже обеспечено хэрнессом
  (subagents.py шарит `tool_interceptors` по ссылке → тот же gate-инстанс
  пишет вердикты; файл не менялся, вне зоны брифа).
- Интеграционный тест создаёт OTEL-спаны (TRACE_ENABLED=true из .env проекта);
  хвостовое `RuntimeError: PhoenixKafkaProducer не запущен` воспроизводится и на
  BASE без моих изменений (stash-проверено) — предсуществующий офлайн-шум стенда,
  тесты проходят (exit 0, все passed).
- `exec_request` NameError (interceptor.py, pre-skip+post) — известное
  предсуществующее, судья pre-only, не тронуто.

## Сомнения / риски

1. **Отклонение от буквы 3e** (описано выше) — требует подтверждения
   диспетчера; считаю оправданным: без него REQ-08 неработоспособен.
2. **Фолбэк-семантика** `_judge_call_id = getattr(tool_output, "tool_call_id", None) or _current_tool_call_id`:
   если выход тула — не ToolMessage (например, Command у FileTools или list у
   некоторых MCP), lookup откатится на синтетический id и вердикт не найдётся.
   Для `mcp_internet` (MCP BaseTool) выход — ToolMessage (проверено живой
   пробой), так что основной путь покрыт; edge остаётся теоретическим.
3. Прямой смоук-командой брифа на этой машине пользоваться нельзя (нет
   event-loop при module-level пуле) — использованы эквиваленты (описано).
