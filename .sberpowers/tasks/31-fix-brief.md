# Бриф задачи 3.1-FIX: SSE-паритет судьи + телеметрия deny-вердиктов

- Основано на two-stage ревью коммита ea886a87: ось B → NEEDS_FIXES
  (1 Critical + 1 Important + 3 Minor); ось A дала PASS, но её вывод о
  deny-ветке опровергнут живой пробой диспетчера.
- Ты работаешь в worktree `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.worktrees/wave3-task31`,
  ветка `wave3-task31`, текущий HEAD `ea886a87`. Второй коммит поверх него.

## Глобальные ограничения (как в основном брифе)

- deepagents==0.7.14 — версию не трогаем.
- Python 3.13; venv ОБЩИЙ на машину, в worktree его нет. Тесты:
  `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.venv/bin/python -m pytest <путь> -v`.
  Ruff: тот же venv, line length 120, type hints обязательны.
- Конфигурация — только поля `AppSettings` (новых полей не нужно).
- Один коммит: `fix(03): SSE-паритет судьи + досыл deny-вердиктов в телеметрию (REQ-01,03,08)`.
- Стейджить только файлы задачи.
- Прямой `python -c "import ..."` вне event loop падает на module-level DB-pool
  (pre-existing, conftest мокает) — смоук делай через pytest --collect-only
  по тестам, импортирующим service/streaming/services_sse.

## Находки ревью и решения диспетчера

### F1 (CRITICAL): судья не подключён в SSE-хэрнессе

`app/services/react/deepagent/services_sse.py:376` собирает
`merged_config = merge_interceptor_configs(self.interceptor_config, stateful_config)`
без судьи → при `comms=sse` (живой канал роутера) и включённом флаге вызовы
mcp_internet уходят в интернет вообще без гейта.

**Решение диспетчера:** единая точка сборки — новый хелпер
`merge_request_interceptor_configs` в `internet_judge.py`, вызываемый из ОБЕИХ
копий `_generate_agent_stream` (service.py и services_sse.py). Это устраняет
сам класс ошибки «в одну копию внесли, в другую забыли».

### F2 (IMPORTANT): блокированные шаги невидимы в телеметрии

Живая проба диспетчера (scripted model + create_deep_agent +
InterceptorMiddleware + deny-мок судьи) доказала: при deny/timeout/error тул
не исполняется, LangChain НЕ эмитит on_tool_start/on_tool_end вовсе
(`tool events emitted: []`), вердикт записан в `gate.verdicts`. Значит
step_metadata-обогащение в on_tool_end работает только на allow-пути, а самые
значимые события (блокировки) не попадают в agents_feedback — нарушена
приёмка REQ-08 «вердикт после любого вызова, включая заблокированные».

**Решение диспетчера:** после цикла событий досылать отдельные шаги
stat-feedback для вердиктов, чей id не совпал ни с одним отправленным
tool-шагом. Хелпер `send_unsent_judge_verdicts` (тестируемый юнитово через
injected sender) + вызов из `generate_agent_stream` (он общий для polling и
SSE — закрывает оба канала сразу).

### F3–F5 (Minor, дешёвые, в тех же файлах — сделать)

- Аннотации `Any` → типизированные (`InternetJudgeGate | None`,
  `InterceptorConfig | None`) через TYPE_CHECKING-импорты.
- Извлечение lookup-id вынести в хелпер `extract_judge_tool_call_id` и
  запиннить его в интеграционном тесте (сейчас тест дублирует паттерн вместо
  пиннинга прод-кода).
- В тестах, вызывающих `build_internet_judge_config` без явного контроля
  флага, добавить явный `monkeypatch.setattr(app_settings,
  "MCP_INTERNET_JUDGE_ENABLED", True)`.

Отклонений диспетчером нет: все находки адресуются этим фиксом.

## Реализация

### 1. RED: новые тесты в tests/test_deepagent_internet_judge_wiring.py

Добавь секции (импорты расширь в шапке: merge_request_interceptor_configs,
send_unsent_judge_verdicts, extract_judge_tool_call_id из internet_judge):

```python
from app.core.config import app_settings
# уже есть; добавь к импортам internet_judge:
    attach_internet_judge_metadata,          # есть
    build_internet_judge_config,            # есть
    extract_judge_tool_call_id,             # новое
    merge_request_interceptor_configs,      # новое
    send_unsent_judge_verdicts,             # новое
```


# ── merge_request_interceptor_configs (F1: SSE-паритет) ────────────────────


def test_merge_request_configs_includes_judge(monkeypatch):
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
    base = InterceptorConfig()
    base.register_tool_interceptor("x", ToolInterceptor())
    merged = merge_request_interceptor_configs(base, None, "c", "m", "q")
    assert isinstance(merged.internet_gate, InternetJudgeGate)
    assert merged.internet_gate.user_question == "q"
    assert set(merged.tool_interceptors) >= {"x", "mcp_internet"}


def test_merge_request_configs_disabled_keeps_plain_stack(monkeypatch):
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
    base = InterceptorConfig()
    base.register_tool_interceptor("x", ToolInterceptor())
    st = InterceptorConfig()
    st.register_tool_interceptor("y", ToolInterceptor())
    merged = merge_request_interceptor_configs(base, st, "c", "m", "q")
    assert merged.internet_gate is None
    assert set(merged.tool_interceptors) == {"x", "y"}


def test_merge_request_configs_all_none_returns_none(monkeypatch):
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
    assert merge_request_interceptor_configs(None, None, "c", "m", "q") is None


# ── send_unsent_judge_verdicts (F2: телеметрия deny) ───────────────────────


@pytest.mark.asyncio
async def test_send_unsent_none_gate_noop():
    calls = []

    async def sender(metadata, sequence):
        calls.append((metadata, sequence))

    assert await send_unsent_judge_verdicts(None, None, sender, 5) == 0
    assert calls == []


@pytest.mark.asyncio
async def test_send_unsent_empty_verdicts_noop():
    calls = []

    async def sender(metadata, sequence):
        calls.append((metadata, sequence))

    gate = build_internet_judge_config("c", "m", "q").internet_gate
    assert await send_unsent_judge_verdicts(gate, set(), sender, 0) == 0
    assert calls == []


@pytest.mark.asyncio
async def test_send_unsent_sends_only_blocked_verdicts():
    gate = InternetJudgeGate(chat_id="c", message_id="m", user_question="q")
    gate.verdicts["call_a"] = InternetJudgeVerdict("allow", "ок")
    gate.verdicts["call_b"] = InternetJudgeVerdict("deny", "не обучение")
    gate.verdicts["call_c"] = InternetJudgeVerdict("timeout", "таймаут")
    calls = []

    async def sender(metadata, sequence):
        calls.append((metadata, sequence))

    sent = await send_unsent_judge_verdicts(gate, {"call_a"}, sender, 7)
    assert sent == 2
    assert [c[0]["internet_judge"]["verdict"] for c in calls] == ["deny", "timeout"]
    assert [c[1] for c in calls] == [8, 9]


@pytest.mark.asyncio
async def test_send_unsent_sender_failure_is_isolated(caplog):
    gate = InternetJudgeGate(chat_id="c", message_id="m", user_question="q")
    gate.verdicts["call_b"] = InternetJudgeVerdict("deny", "не обучение")
    gate.verdicts["call_c"] = InternetJudgeVerdict("error", "сбой")

    async def boom(metadata, sequence):
        raise RuntimeError("feedback storage down")

    assert await send_unsent_judge_verdicts(gate, set(), boom, 0) == 0


# ── extract_judge_tool_call_id (F4: пиннинг lookup-id) ─────────────────────


def test_extract_id_from_tool_message():
    tm = ToolMessage(content="x", name="mcp_internet", tool_call_id="call_REAL")
    assert extract_judge_tool_call_id(tm, "call_SYNTH") == "call_REAL"


def test_extract_id_fallback_for_non_tool_message():
    assert extract_judge_tool_call_id("plain string", "call_SYNTH") == "call_SYNTH"
    assert extract_judge_tool_call_id(None, "call_SYNTH") == "call_SYNTH"
```

Запусти тест-файл: новые импорты упадут с ImportError (RED). Существующие 10
тестов должны остаться зелёными.

### 2. internet_judge.py — три новых хелпера (после attach_internet_judge_metadata)

Проверь шапку файла: в typing-импортах должны быть `Any`, `Awaitable`,
`Callable` (доопаси недостающие); в TYPE_CHECKING уже есть импорты
interceptor (ToolOverrideResult, InterceptorConfig — сверь).

```python
def merge_request_interceptor_configs(
    base: "InterceptorConfig | None",
    stateful: "InterceptorConfig | None",
    chat_id: str,
    message_id: str,
    user_question: str,
) -> "InterceptorConfig | None":
    """Interceptor-конфиг запроса: base + stateful + судья mcp_internet.

    Единая точка сборки для обоих хэрнессов DeepAgent (service.py /
    services_sse.py). Судейский конфиг применяется последним (extra): его
    internet_gate и перехватчик mcp_internet побеждают в merge. При
    выключенном MCP_INTERNET_JUDGE_ENABLED эквивалентно merge(base, stateful)
    и стек middleware не меняется.
    """
    from app.services.react.deepagent.stateful_routing import merge_interceptor_configs

    judge_config = build_internet_judge_config(chat_id, message_id, user_question)
    return merge_interceptor_configs(merge_interceptor_configs(base, stateful), judge_config)


def extract_judge_tool_call_id(tool_output: Any, fallback: str) -> str:
    """Реальный tool_call_id из ToolMessage-выхода on_tool_end, иначе фолбэк на кадр-ид."""
    return getattr(tool_output, "tool_call_id", None) or fallback


async def send_unsent_judge_verdicts(
    gate: Any,
    sent_tool_call_ids: set[str] | None,
    sender: Callable[[dict[str, Any], int], Awaitable[None]],
    base_step_sequence: int,
) -> int:
    """REQ-08: дослать в stat-feedback вердикты без tool-события (deny/timeout/error).

    При блокировке тул не исполняется и LangChain не эмитит on_tool_end —
    такие вердикты иначе невидимы в per-step телеметрии. ``sender`` получает
    ``(step_metadata, step_sequence)``. Телеметрия не должна ломать стрим:
    сбой одного шага логируется и пропускается. Возвращает число досланных.
    """
    if gate is None:
        return 0
    sent = sent_tool_call_ids or set()
    sent_count = 0
    for call_id, verdict in gate.verdicts.items():
        if not call_id or call_id in sent or not hasattr(verdict, "to_metadata"):
            continue
        base_step_sequence += 1
        try:
            await sender(verdict.to_metadata(), base_step_sequence)
        except Exception:
            logger.exception("[chat] Не удалось дослать вердикт судьи %s", call_id)
            continue
        sent_count += 1
    return sent_count
```

Аннотацию параметра `gate` оставь `Any` (см. F3: строгая типизация ниже);
или `"InternetJudgeGate | None"`, если интернет_judge самодостаточен для
локальной аннотации — выбери то, что проходит ruff без циклов.

Типизация F3 в этом же файле: у `attach_internet_judge_metadata` замени
`interceptor_config: Any` на `interceptor_config: "InterceptorConfig | None"`
(TYPE_CHECKING-импорт InterceptorConfig уже есть — добавь при необходимости).
`step_metadata: Any` → `step_metadata: dict[str, Any] | None`.

### 3. interceptor.py — типизация поля (F3)

В TYPE_CHECKING-блоке (если нет — создай после обычных импортов при
`from __future__ import annotations`; он должен уже быть):

```python
if TYPE_CHECKING:
    from app.services.react.deepagent.internet_judge import InternetJudgeGate
```

Поле `internet_gate: Any | None = None` замени на
`internet_gate: InternetJudgeGate | None = None` (TYPE_CHECKING + future
annotations = нет рантайм-цикла: internet_judge тоже импортирует interceptor
только в TYPE_CHECKING/runtime-inside-function). Если внезапно рантайм-цикл
или ruff против — верни `Any | None` и зафиксируй в отчёте (это Minor).

### 4. service.py — переход на единый хелпер (F1)

В `_generate_agent_stream` найди блок, вставленный задачей 3.1:

```python
        judge_config = build_internet_judge_config(...)
        merged_config = merge_interceptor_configs(
            merge_interceptor_configs(self.interceptor_config, stateful_config),
            judge_config,
        )
```

Замени его на:

```python
        merged_config = merge_request_interceptor_configs(
            self.interceptor_config, stateful_config,
            request.chat_id, request.message_id, request.message,
        )
```

Импорт в шапке: `build_internet_judge_config` → `merge_request_interceptor_configs`
(если `merge_interceptor_configs` больше нигде в файле не используется — убери
его из импорта; проверь grep'ом по файлу).

### 5. services_sse.py — тот же хелпер (F1, Critical)

Строка ~376: `merged_config = merge_interceptor_configs(self.interceptor_config, stateful_config)`
замени на тот же вызов `merge_request_interceptor_configs(self.interceptor_config,
stateful_config, request.chat_id, request.message_id, request.message)`
(имена полей request сверь по фактическому коду этой функции — должен быть
доступен `request` того же типа, что в service.py).

Импорт в шапке services_sse.py: из `...stateful_routing import ...,
merge_interceptor_configs` — замени `merge_interceptor_configs` на
`merge_request_interceptor_configs` (импортируй его из
`app.services.react.deepagent.internet_judge`); проверь grep'ом, используется
ли `merge_interceptor_configs` в файле ещё где-то — если нет, старый импорт
убери (ruff F401).

### 6. streaming.py — сбор sent-ids + досыл после цикла (F2) + хелпер (F4)

6a. Импорт в шапке расширь (уже есть attach_internet_judge_metadata):

```python
from app.services.react.deepagent.internet_judge import (
    attach_internet_judge_metadata,
    extract_judge_tool_call_id,
    send_unsent_judge_verdicts,
)
```

6b. В `on_tool_end` inline-строку
`_judge_call_id = getattr(tool_output, "tool_call_id", None) or _current_tool_call_id`
замени на `_judge_call_id = extract_judge_tool_call_id(tool_output, _current_tool_call_id)`
(комментарий над ней сохрани/обнови).

6c. Рядом с объявлениями локалов (строка ~570, `_current_tool_stack: dict[...] = {}`)
добавь: `_sent_judge_call_ids: set[str] = set()`.

6d. В обработчике `on_tool_end`, сразу после вычисления `_judge_call_id` и до
вызова `_send_cycle_feedback(...)`, добавь: `_sent_judge_call_ids.add(_judge_call_id)`.

6e. После завершения цикла `async for event in agent.astream_events(...)`
(найди фактический конец цикла — ДО блока сохранения итогового ответа в БД
и финального step_type="message" feedback, до cleanup) вставь:

```python
    # REQ-08: заблокированные вызовы mcp_internet (deny/timeout/error) не
    # эмитят on_tool_end — их вердикты досылаем отдельными шагами stat-feedback.
    async def _judge_feedback(step_metadata: dict[str, Any], step_sequence: int) -> None:
        await _send_cycle_feedback(
            chat_id=chat_id,
            chat_message_id=chat_message_id,
            step_input=None,
            user_token=user_token,
            start_dt=_cycle_start_dt,
            end_dt=datetime.now(timezone.utc),
            step_output=None,
            step_metadata=step_metadata,
            step_type="tool",
            step_sequence=step_sequence,
            step_name="mcp_internet",
            error=None,
        )

    await send_unsent_judge_verdicts(
        gate=getattr(interceptor_config, "internet_gate", None),
        sent_tool_call_ids=_sent_judge_call_ids,
        sender=_judge_feedback,
        base_step_sequence=_tool_step_sequence,
    )
```

Сверь по фактическому коду: сигнатуру `_send_cycle_feedback` (kwargs должны
совпадать с существующими вызовами; параметры step_name/error имеют дефолты —
если каких-то параметров нет в сигнатуре, приведи вызов в соответствие с ней);
что `interceptor_config`, `user_token`, `chat_message_id`, `_cycle_start_dt`
видны в scope этой точки; вставка не должна выполняться в finally/except-путях
ошибок — только нормальный выход из цикла.

6f. В интеграционном тесте `test_real_tool_call_id_reaches_step_metadata`
замени собственный паттерн извлечения
(`extracted_id = getattr(output, "tool_call_id", None) or ""`) на пиннинг
прод-кода: `extracted_id = extract_judge_tool_call_id(output, "")`.

### 7. Явные флаги в существующих тестах (F5)

Всем тестам файла, вызывающим `build_internet_judge_config` или
`merge_request_interceptor_configs` без контроля флага, добавь первой строкой
`monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)`
(и параметр `monkeypatch` в сигнатуру, если его нет):
test_build_enabled_registers_gate_and_interceptor, test_merge_preserves_internet_gate,
test_merge_none_base_passes_gate_through, test_middleware_deny_blocks_internet,
test_middleware_allow_passes_through, test_attach_metadata_adds_verdict,
test_attach_metadata_noop_for_other_tools, test_attach_metadata_none_config.

## GREEN и регресс

1. `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_wiring.py -v` — все тесты
   (прежние 10 + новые ~10) зелёные.
2. Регресс зоны: `tests/test_deepagent_internet_judge.py tests/test_deepagent_interceptor_async.py
   tests/test_deepagent_session_id.py` — 36 passed как раньше.
3. Смоук цепочки: `.venv/bin/python -m pytest --collect-only -q
   tests/test_deepagent_internet_judge_wiring.py tests/test_deepagent_session_id.py`
   (импорты service/streaming/services_sse проходят в conftest-окружении).
4. Ruff по файлам задачи: `internet_judge.py interceptor.py stateful_routing.py service.py
   services_sse.py streaming.py tests/test_deepagent_internet_judge_wiring.py` — 0 новых
   замечаний (8 предсуществующих streaming.py вне зоны).

## Коммит

Стейджить только изменённые файлы задачи (7 файлов):
`fix(03): SSE-паритет судьи + досыл deny-вердиктов в телеметрию (REQ-01,03,08)`

## Критерии выхода

- SSE-хэрнесс собирает конфиг через тот же хелпер, что polling (F1 закрыт;
  юнит-тесты хелпера зелёные, включая disabled-ветку).
- Вердикты deny/timeout/error доходят до stat-feedback отдельными шагами
  (F2; юнит-тесты хелпера зелёные, sender-сбои изолированы).
- Lookup-id пиннится хелпером в тесте; аннотации типизированы (F3–F5).
- Регресс зоны зелёная; ruff без новых замечаний; отчёт обновлён.

Полный отчёт запиши в
`/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.sberpowers/tasks/31-fix-report.md`
(ГЛАВНОЕ дерево, абсолютный путь): команды и результаты, RED/GREEN, self-review,
сомнения. В чат верни ≤15 строк: статус, коммит SHA+заголовок, строка о тестах,
сомнения, путь к отчёту.
