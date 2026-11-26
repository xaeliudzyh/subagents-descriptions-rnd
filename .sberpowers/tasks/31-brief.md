# Бриф задачи 3.1: Wiring — серверная регистрация + наблюдаемость (волна 3)

- REQ: REQ-mcp-internet-judge-01, REQ-mcp-internet-judge-03, REQ-mcp-internet-judge-08
- Blocked by: задача 2.1 (выполнена, c22a10d4 + fix b9d804a9)
- Ты работаешь в изолированном worktree от ветки mcp-internet-judge (HEAD b9d804a9)

## Глобальные ограничения плана (дословно)

- deepagents==0.7.14 — версию не трогаем.
- Python 3.13, venv проекта `.venv`; тесты запускать
  `.venv/bin/python -m pytest <путь> -v` из корня репо (worktree).
- Ruff: line length 120, type hints обязательны.
- Конфигурация — только поля `AppSettings` в `app/core/config.py` (уже есть из
  задачи 2.1; новых полей этой задаче не нужно).
- Один коммит: `feat(03): серверная регистрация mcp_internet-judge + вердикты в step_metadata (REQ-01,03,08)`.
- Стейджить только файлы задачи.

## Разрешённые диспетчером неоднозначности

- Номера строк в этом брифе — ориентиры из плана; фактические позиции сверяй
  по содержимому файлов (код дрейфанул от плана в задачах 1.1/2.1).
- Ruff-канонизация импортов/аннотаций допустима (как в задачах 1.1/2.1),
  логика — дословно.
- Известное предсуществующее (НЕ трогать): interceptor.py ~:297 `exec_request`
  NameError при pre(skip)+post — судья pre-only, не касается этой задачи;
  падения tests/test_deepagent_subagents.py на BASE — вне зоны.
- Async-тесты: @pytest.mark.asyncio (конвенция проекта).

## Контекст задачи

Судья (задача 2.1, `app/services/react/deepagent/internet_judge.py`) реализован,
но не подключён. Эта задача регистрирует его серверно на каждый запрос
(main + routing-сабагенты — interceptor уже прокидывается вниз механизмом
DeepAgent-harness) и прокидывает вердикты в per-step телеметрию stat-feedback
(`step_metadata` в `_send_cycle_feedback`). При выключенном флаге
`MCP_INTERNET_JUDGE_ENABLED` поведение системы неотличимо от текущего
(`build_internet_judge_config` возвращает None, стек middleware не меняется).

Фактические интерфейсы задачи 2.1 (сверены диспетчером после фикса):
- `InternetJudgeGate(chat_id, message_id, user_question)` — dataclass, атрибут
  `verdicts: dict[str, InternetJudgeVerdict]` по `tool_call_id`;
  `pre_execute(request)` — async, возвращает None | ToolOverrideResult.
- `InternetJudgeVerdict.to_metadata() -> {"internet_judge": {"verdict", "reason"}}`.
- Конфиг-поля: `app_settings.MCP_INTERNET_JUDGE_ENABLED/_MODEL/_TIMEOUT_SECONDS`.
- В `internet_judge.py` блок TYPE_CHECKING уже содержит импорты из interceptor
  (ToolOverrideResult; InterceptorConfig может отсутствовать — допиши при
  необходимости).

## Файлы

- Изменить: `app/services/react/deepagent/interceptor.py` (поле `internet_gate` + clear())
- Изменить: `app/services/react/deepagent/stateful_routing.py` (`merge_interceptor_configs`)
- Изменить: `app/services/react/deepagent/internet_judge.py` (helpers вниз файла)
- Изменить: `app/services/react/deepagent/service.py` (регистрация в `_generate_agent_stream`)
- Изменить: `app/services/react/deepagent/streaming.py` (step_metadata на on_tool_end)
- Создать: `tests/test_deepagent_internet_judge_wiring.py`

## Шаги

1. Напиши падающий тест `tests/test_deepagent_internet_judge_wiring.py`:

```python
"""Wiring судьи: build/merge/middleware/step-metadata (без живого LLM)."""

import pytest
from langchain_core.messages import ToolMessage
from langgraph.prebuilt.tool_node import ToolCallRequest

from app.core.config import app_settings
from app.services.react.deepagent.interceptor import (
    InterceptorConfig,
    InterceptorMiddleware,
    ToolInterceptor,
)
from app.services.react.deepagent.internet_judge import (
    InternetJudgeGate,
    InternetJudgeVerdict,
    attach_internet_judge_metadata,
    build_internet_judge_config,
)
from app.services.react.deepagent.stateful_routing import merge_interceptor_configs


def _request(call_id: str = "call_w") -> ToolCallRequest:
    return ToolCallRequest(
        tool_call={"name": "mcp_internet", "args": {"query": "погода в Москве"}, "id": call_id},
        tool=None,
        state={},
        runtime=None,
    )


# ── build_internet_judge_config ────────────────────────────────────────────


def test_build_disabled_returns_none(monkeypatch):
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
    assert build_internet_judge_config("c", "m", "вопрос") is None


def test_build_enabled_registers_gate_and_interceptor(monkeypatch):
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
    cfg = build_internet_judge_config("c", "m", "вопрос")
    assert cfg is not None
    assert isinstance(cfg.internet_gate, InternetJudgeGate)
    assert cfg.internet_gate.user_question == "вопрос"
    ti = cfg.tool_interceptors["mcp_internet"]
    assert ti.pre_execute == cfg.internet_gate.pre_execute


# ── merge сохраняет gate ────────────────────────────────────────────────────


def test_merge_preserves_internet_gate():
    cfg = build_internet_judge_config("c", "m", "q")
    other = InterceptorConfig()
    other.register_tool_interceptor("some_tool", ToolInterceptor())
    merged = merge_interceptor_configs(other, cfg)
    assert merged.internet_gate is cfg.internet_gate
    assert "some_tool" in merged.tool_interceptors
    assert "mcp_internet" in merged.tool_interceptors


def test_merge_none_base_passes_gate_through():
    cfg = build_internet_judge_config("c", "m", "q")
    merged = merge_interceptor_configs(None, cfg)
    assert merged is cfg


# ── e2e через middleware: deny блокирует handler (REQ-01, REQ-03) ──────────


@pytest.mark.asyncio
async def test_middleware_deny_blocks_internet(monkeypatch):
    cfg = build_internet_judge_config("c", "m", "Хочу обучение по Java")
    gate = cfg.internet_gate

    async def fake_llm(messages):
        return '{"verdict": "deny", "reason": "погода не связана с обучением"}'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    mw = InterceptorMiddleware(cfg)
    handler_called = False

    async def handler(req):
        nonlocal handler_called
        handler_called = True
        return ToolMessage(content="INTERNET RESULT", name=req.tool_call["name"], tool_call_id=req.tool_call["id"])

    result = await mw.awrap_tool_call(_request(), handler)
    assert handler_called is False  # в интернет не ходили
    assert result.status == "error"
    assert "погода не связана с обучением" in result.content


@pytest.mark.asyncio
async def test_middleware_allow_passes_through(monkeypatch):
    cfg = build_internet_judge_config("c", "m", "Хочу обучение по Java")
    gate = cfg.internet_gate

    async def fake_llm(messages):
        return '{"verdict": "allow", "reason": "поиск курса"}'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    mw = InterceptorMiddleware(cfg)

    async def handler(req):
        return ToolMessage(content="INTERNET RESULT", name=req.tool_call["name"], tool_call_id=req.tool_call["id"])

    result = await mw.awrap_tool_call(_request(), handler)
    assert result.content == "INTERNET RESULT"
    assert gate.verdicts["call_w"].verdict == "allow"


# ── step_metadata (REQ-08) ─────────────────────────────────────────────────


def test_attach_metadata_adds_verdict():
    cfg = build_internet_judge_config("c", "m", "q")
    gate = cfg.internet_gate
    gate.verdicts["call_w"] = InternetJudgeVerdict("deny", "не обучение")
    meta = attach_internet_judge_metadata({"query": "погода"}, cfg, "mcp_internet", "call_w")
    assert meta["internet_judge"] == {"verdict": "deny", "reason": "не обучение"}
    assert meta["query"] == "погода"


def test_attach_metadata_noop_for_other_tools():
    cfg = build_internet_judge_config("c", "m", "q")
    meta = attach_internet_judge_metadata({"a": 1}, cfg, "mcp_query", "call_x")
    assert meta == {"a": 1}


def test_attach_metadata_none_config():
    meta = attach_internet_judge_metadata(None, None, "mcp_internet", "call_x")
    assert meta == {}
```

Примечание: `build_internet_judge_config` в этих тестах вызывается при
дефолтном `MCP_INTERNET_JUDGE_ENABLED=True`; если нужен явный контроль —
добавь monkeypatch True в соответствующие фикстуры.

2. Запусти `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_wiring.py -v`
   — ожидай FAIL (нет helpers/поля internet_gate).

3. Реализуй изменения:

3a. `interceptor.py`, dataclass `InterceptorConfig` — добавь ПОСЛЕДНИМ полем:

```python
    # Per-request объект судьи mcp_internet (см. internet_judge.py):
    # streaming читает отсюда вердикты для step_metadata stat-feedback.
    internet_gate: Any | None = None
```

и добавь в `clear()` строку `self.internet_gate = None`.

3b. `stateful_routing.py::merge_interceptor_configs` — в конструктор
`InterceptorConfig(...)` добавь аргумент:

```python
        internet_gate=extra.internet_gate or base.internet_gate,
```

3c. `internet_judge.py` — добавь в конец файла два helper'а:

```python
def build_internet_judge_config(
    chat_id: str,
    message_id: str,
    user_question: str,
) -> "InterceptorConfig | None":
    """Конфиг interceptor'а с судьёй mcp_internet. None — судья выключен.

    Регистрируется серверно на каждый запрос (не через декларативный API):
    политика назначения применяется ко всем чатам единообразно.
    """
    if not app_settings.MCP_INTERNET_JUDGE_ENABLED:
        return None
    from app.services.react.deepagent.interceptor import InterceptorConfig, ToolInterceptor

    gate = InternetJudgeGate(chat_id=chat_id, message_id=message_id, user_question=user_question)
    config = InterceptorConfig()
    config.register_tool_interceptor("mcp_internet", ToolInterceptor(pre_execute=gate.pre_execute))
    config.internet_gate = gate
    logger.info("[chat: %s] mcp_internet judge gate enabled (model=%s)", chat_id, app_settings.MCP_INTERNET_JUDGE_MODEL)
    return config


def attach_internet_judge_metadata(
    step_metadata: Any,
    interceptor_config: Any,
    tool_name: str,
    tool_call_id: str,
) -> dict[str, Any]:
    """Дополнить step_metadata вердиктом судьи для шага mcp_internet (REQ-08)."""
    meta: dict[str, Any] = dict(step_metadata) if isinstance(step_metadata, dict) else {}
    gate = getattr(interceptor_config, "internet_gate", None) if interceptor_config is not None else None
    if tool_name != "mcp_internet" or gate is None:
        return meta
    verdict = getattr(gate, "verdicts", {}).get(tool_call_id)
    if verdict is not None and hasattr(verdict, "to_metadata"):
        meta.update(verdict.to_metadata())
    return meta
```

(Проверь блок TYPE_CHECKING в начале файла: там должен быть
`from app.services.react.deepagent.interceptor import InterceptorConfig`
для аннотации возвращаемого типа; runtime-импорт внутри функции остаётся.)

3d. `service.py` в `_generate_agent_stream` — найди место сборки перехватчиков
(stateful_config через make_stateful_interceptor_config) и вставь между ним и
формированием merged/конфига агента:

```python
        judge_config = build_internet_judge_config(
            request.chat_id, request.message_id, request.message,
        )
        merged_config = merge_interceptor_configs(
            merge_interceptor_configs(self.interceptor_config, stateful_config),
            judge_config,
        )
```

(если merged_config собирается иначе — сохрани существующую структуру:
base=self.interceptor_config + stateful_config, extra=judge_config;
судейский конфиг — последним/extra, чтобы его internet_gate победил).
Добавь импорт: `from app.services.react.deepagent.internet_judge import build_internet_judge_config`.

3e. `streaming.py` — в обработчике `on_tool_end` найди вызов
`await _send_cycle_feedback(...)` с `step_metadata=_current_tool_input` и
замени аргумент на:

```python
                step_metadata=attach_internet_judge_metadata(
                    _current_tool_input, interceptor_config, tool_name, _current_tool_call_id,
                ),
```

(переменная `interceptor_config` — параметр generate_agent_stream, видна в
scope; `tool_name` и `_current_tool_call_id` вычисляются выше в этом же
обработчике — сверь фактические имена переменных по коду!).
Импорт в шапке: `from app.services.react.deepagent.internet_judge import attach_internet_judge_metadata`.

4. GREEN: `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_wiring.py -v`
   — ожидай 8 PASS.

5. Регресс зоны: `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py tests/test_deepagent_interceptor_async.py tests/test_deepagent_session_id.py -v`
   — ожидай PASS без изменений контракта.

6. Смоук импорта: `.venv/bin/python -c "import app.services.react.deepagent.service; import app.services.react.deepagent.streaming"`
   — без ошибок (проверка wiring-цикла импортов).

7. `.venv/bin/ruff check app/services/react/deepagent/ tests/test_deepagent_internet_judge_wiring.py` — 0 замечаний.

8. Commit (только файлы задачи): `feat(03): серверная регистрация mcp_internet-judge + вердикты в step_metadata (REQ-01,03,08)`

## Критерии выхода

- 8/8 wiring-тестов зелёные; deny не пускает handler (в интернет не ходили);
  allow прозрачен; gate сохраняется через merge поверх stateful-конфига.
- При выключенном флаге `build_internet_judge_config` → None и стек не меняется.
- `step_metadata` шага `mcp_internet` обогащается `internet_judge.{verdict,reason}`.
- service.py/streaming.py импортируются без ошибок; регресс судьи/interceptor/session_id зелёная.
