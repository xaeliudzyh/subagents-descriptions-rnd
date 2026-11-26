c134b2ea fix(03): досл вердиктов на всех выходах стрима; продвижение step_sequence; step_input вопроса (REQ-08)
31da4c6f fix(03): SSE-паритет судьи + досыл deny-вердиктов в телеметрию (REQ-01,03,08)
ea886a87 feat(03): серверная регистрация mcp_internet-judge + вердикты в step_metadata (REQ-01,03,08)

 app/services/react/deepagent/interceptor.py      |   6 +
 app/services/react/deepagent/internet_judge.py   | 101 ++++++-
 app/services/react/deepagent/service.py          |  14 +-
 app/services/react/deepagent/services_sse.py     |  13 +-
 app/services/react/deepagent/stateful_routing.py |   1 +
 app/services/react/deepagent/streaming.py        |  63 ++++-
 tests/test_deepagent_internet_judge_wiring.py    | 329 +++++++++++++++++++++++
 7 files changed, 514 insertions(+), 13 deletions(-)

diff --git a/app/services/react/deepagent/interceptor.py b/app/services/react/deepagent/interceptor.py
index 749c11bd..ebce58b8 100644
--- a/app/services/react/deepagent/interceptor.py
+++ b/app/services/react/deepagent/interceptor.py
@@ -57,20 +57,22 @@ from langchain.agents.middleware.types import AgentMiddleware
 from langchain_core.messages import ToolMessage
 
 if TYPE_CHECKING:
     from collections.abc import Awaitable
 
     from langchain.agents.middleware.types import ModelRequest, ModelResponse
     from langchain_core.messages import AIMessage
     from langgraph.prebuilt.tool_node import ToolCallRequest
     from langgraph.runtime import Runtime
 
+    from app.services.react.deepagent.internet_judge import InternetJudgeGate
+
 logger = logging.getLogger(__name__)
 
 
 # ── Result types ────────────────────────────────────────────────────────────
 
 
 @dataclass
 class ToolOverrideResult:
     """Return this from ``pre_execute`` to skip tool execution entirely.
 
@@ -188,39 +190,43 @@ class InterceptorConfig:
             }),
         ))
     """
 
     tool_interceptors: dict[str, ToolInterceptor] = field(default_factory=dict)
     model_interceptor: ModelInterceptor | None = None
     before_model_hook: StateHook | None = None
     after_model_hook: StateHook | None = None
     before_agent_hook: StateHook | None = None
     after_agent_hook: StateHook | None = None
+    # Per-request объект судьи mcp_internet (см. internet_judge.py):
+    # streaming читает отсюда вердикты для step_metadata stat-feedback.
+    internet_gate: InternetJudgeGate | None = None
 
     def register_tool_interceptor(self, tool_name: str, interceptor: ToolInterceptor) -> None:
         """Register an interceptor for a specific tool by name."""
         self.tool_interceptors[tool_name] = interceptor
         logger.debug("Registered tool interceptor for '%s'", tool_name)
 
     def set_model_interceptor(self, interceptor: ModelInterceptor) -> None:
         """Set the model interceptor (only one at a time)."""
         self.model_interceptor = interceptor
         logger.debug("Registered model interceptor")
 
     def clear(self) -> None:
         """Remove all registered interceptors and hooks."""
         self.tool_interceptors.clear()
         self.model_interceptor = None
         self.before_model_hook = None
         self.after_model_hook = None
         self.before_agent_hook = None
         self.after_agent_hook = None
+        self.internet_gate = None
 
 
 # ── Middleware ──────────────────────────────────────────────────────────────
 
 
 class InterceptorMiddleware(AgentMiddleware):
     """Middleware for intercepting and overriding agent execution events.
 
     Provides pre-execution interception for tool calls and model calls,
     plus hooks for before/after model and agent events.
diff --git a/app/services/react/deepagent/internet_judge.py b/app/services/react/deepagent/internet_judge.py
index f4de84d1..0f8835d8 100644
--- a/app/services/react/deepagent/internet_judge.py
+++ b/app/services/react/deepagent/internet_judge.py
@@ -10,28 +10,30 @@ Fail-closed: любой сбой судьи (ошибка LLM, таймаут, 
 error и человекочитаемой причиной. Исключения ловятся здесь, а не в общем
 fail-open перехватчике InterceptorMiddleware.
 """
 
 from __future__ import annotations
 
 import asyncio
 import json
 import logging
 from dataclasses import dataclass, field
-from typing import TYPE_CHECKING, Any, Literal
+from typing import TYPE_CHECKING, Any, Callable, Literal
 
 from app.core.config import app_settings
 
 if TYPE_CHECKING:
+    from collections.abc import Awaitable
+
     from langgraph.prebuilt.tool_node import ToolCallRequest
 
-    from app.services.react.deepagent.interceptor import ToolOverrideResult
+    from app.services.react.deepagent.interceptor import InterceptorConfig, ToolOverrideResult
 
 logger = logging.getLogger(__name__)
 
 _JUDGE_SYSTEM_PROMPT = """\
 Ты — предикат для инструмента интернет-поиска корпоративного HR-ассистента \
 (скилл «Поиск внешнего контента», процесс БЛиК П298 «Поиск внешних \
 образовательных ресурсов в Пульс»).
 
 Инструмент mcp_internet допускается ТОЛЬКО как обогащение поиска обучения по \
 каталогу платформы Пульс и закрывает три потребности: (1) поиск внешнего \
@@ -211,10 +213,105 @@ class InternetJudgeGate:
             logger.warning(
                 "[chat: %s] Internet judge DENY: %.200s — %s",
                 self.chat_id,
                 query,
                 verdict.reason,
             )
             content = f"{_DENY_TITLE}: {verdict.reason}"
         else:  # timeout | error
             content = f"{_FAIL_CLOSED_TITLE}. Попробуйте переформулировать запрос или воспользуйтесь другим инструментом."
         return ToolOverrideResult(content=content, status="error")
+
+
+def build_internet_judge_config(
+    chat_id: str,
+    message_id: str,
+    user_question: str,
+) -> InterceptorConfig | None:
+    """Конфиг interceptor'а с судьёй mcp_internet. None — судья выключен.
+
+    Регистрируется серверно на каждый запрос (не через декларативный API):
+    политика назначения применяется ко всем чатам единообразно.
+    """
+    if not app_settings.MCP_INTERNET_JUDGE_ENABLED:
+        return None
+    from app.services.react.deepagent.interceptor import InterceptorConfig, ToolInterceptor
+
+    gate = InternetJudgeGate(chat_id=chat_id, message_id=message_id, user_question=user_question)
+    config = InterceptorConfig()
+    config.register_tool_interceptor("mcp_internet", ToolInterceptor(pre_execute=gate.pre_execute))
+    config.internet_gate = gate
+    logger.info("[chat: %s] mcp_internet judge gate enabled (model=%s)", chat_id, app_settings.MCP_INTERNET_JUDGE_MODEL)
+    return config
+
+
+def attach_internet_judge_metadata(
+    step_metadata: dict[str, Any] | None,
+    interceptor_config: InterceptorConfig | None,
+    tool_name: str,
+    tool_call_id: str,
+) -> dict[str, Any]:
+    """Дополнить step_metadata вердиктом судьи для шага mcp_internet (REQ-08)."""
+    meta: dict[str, Any] = dict(step_metadata) if isinstance(step_metadata, dict) else {}
+    gate = getattr(interceptor_config, "internet_gate", None) if interceptor_config is not None else None
+    if tool_name != "mcp_internet" or gate is None:
+        return meta
+    verdict = getattr(gate, "verdicts", {}).get(tool_call_id)
+    if verdict is not None and hasattr(verdict, "to_metadata"):
+        meta.update(verdict.to_metadata())
+    return meta
+
+
+def merge_request_interceptor_configs(
+    base: InterceptorConfig | None,
+    stateful: InterceptorConfig | None,
+    chat_id: str,
+    message_id: str,
+    user_question: str,
+) -> InterceptorConfig | None:
+    """Interceptor-конфиг запроса: base + stateful + судья mcp_internet.
+
+    Единая точка сборки для обоих хэрнессов DeepAgent (service.py /
+    services_sse.py). Судейский конфиг применяется последним (extra): его
+    internet_gate и перехватчик mcp_internet побеждают в merge. При
+    выключенном MCP_INTERNET_JUDGE_ENABLED эквивалентно merge(base, stateful)
+    и стек middleware не меняется.
+    """
+    from app.services.react.deepagent.stateful_routing import merge_interceptor_configs
+
+    judge_config = build_internet_judge_config(chat_id, message_id, user_question)
+    return merge_interceptor_configs(merge_interceptor_configs(base, stateful), judge_config)
+
+
+def extract_judge_tool_call_id(tool_output: Any, fallback: str) -> str:
+    """Реальный tool_call_id из ToolMessage-выхода on_tool_end, иначе фолбэк на кадр-ид."""
+    return getattr(tool_output, "tool_call_id", None) or fallback
+
+
+async def send_unsent_judge_verdicts(
+    gate: InternetJudgeGate | None,
+    sent_tool_call_ids: set[str] | None,
+    sender: Callable[[dict[str, Any], int], Awaitable[None]],
+    base_step_sequence: int,
+) -> int:
+    """REQ-08: дослать в stat-feedback вердикты без tool-события (deny/timeout/error).
+
+    При блокировке тул не исполняется и LangChain не эмитит on_tool_end —
+    такие вердикты иначе невидимы в per-step телеметрии. ``sender`` получает
+    ``(step_metadata, step_sequence)``. Телеметрия не должна ломать стрим:
+    сбой одного шага логируется и пропускается. Возвращает число досланных.
+    """
+    if gate is None:
+        return 0
+    sent = sent_tool_call_ids or set()
+    sent_count = 0
+    for call_id, verdict in gate.verdicts.items():
+        if not call_id or call_id in sent or not hasattr(verdict, "to_metadata"):
+            continue
+        base_step_sequence += 1
+        try:
+            await sender(verdict.to_metadata(), base_step_sequence)
+        except Exception:
+            logger.exception("[chat] Не удалось дослать вердикт судьи %s", call_id)
+            continue
+        sent_count += 1
+    return sent_count
diff --git a/app/services/react/deepagent/service.py b/app/services/react/deepagent/service.py
index 831381e1..df3242a5 100644
--- a/app/services/react/deepagent/service.py
+++ b/app/services/react/deepagent/service.py
@@ -16,39 +16,37 @@ from contextlib import aclosing
 from dataclasses import dataclass
 from typing import TYPE_CHECKING, Any
 
 from sber_dataplatform_py.utils.context import service_jwt_token_context
 from sber_hrp_py_libs_auth.types import JWTToken
 
 from app.services.delegation_service import DelegationPriorityService, DelegationService
 from app.services.react.agent import AbstractAgentService
 from app.services.react.deepagent.backend import build_filesystem_backend as _build_filesystem_backend
 from app.services.react.deepagent.chat import prepare_chat, prepare_user_scopes
+from app.services.react.deepagent.internet_judge import merge_request_interceptor_configs
 from app.services.react.deepagent.lifecycle import (
     _cancel_keepalive,
     _chunk_to_event_type,
     _is_expired,
     _keepalive_loop,
 )
 from app.services.react.deepagent.lifecycle import (
     execute as _execute,
 )
 from app.services.react.deepagent.lifecycle import (
     pool as _lifecycle_pool,
 )
 from app.services.react.deepagent.mcp_converter import convert_mcp_tools
 from app.services.react.deepagent.mcp_converter import mcp_tool_to_base_tool as _mcp_tool_to_base_tool
 from app.services.react.deepagent.model import build_gigachat_model, prepare_models
-from app.services.react.deepagent.stateful_routing import (
-    make_stateful_interceptor_config,
-    merge_interceptor_configs,
-)
+from app.services.react.deepagent.stateful_routing import make_stateful_interceptor_config
 from app.services.react.deepagent.streaming import generate_agent_stream
 from app.services.react.repository import (
     AbstractChatMessagesRepository,
     AbstractChatRepository,
     AbstractPollingStateRepository,
     ChatMessagesRepository,
     ChatRepository,
     PollingStateRepository,
 )
 from app.services.react.settings import react_settings
@@ -281,21 +279,27 @@ class DeepAgentCallAgentService(AbstractAgentService):
             )
 
         # ── 3. Stateful routing: force delegation when stateful subagents are active ──
         # Only subagents available to this user — a disabled / out-of-scope
         # stateful subagent must not be picked by the routing.
         all_configs = get_enabled_subagents(scopes)
         stateful_names = [c.name for c in all_configs if c.stateful]
         stateful_config = make_stateful_interceptor_config(
             request.chat_id, stateful_names, chat_history=history,
         )
-        merged_config = merge_interceptor_configs(self.interceptor_config, stateful_config)
+        # Судья mcp_internet: единая точка сборки interceptor-конфига запроса
+        # (base + stateful + judge) — общий хелпер для polling и SSE хэрнессов,
+        # судейский конфиг применяется последним, чтобы его internet_gate победил.
+        merged_config = merge_request_interceptor_configs(
+            self.interceptor_config, stateful_config,
+            request.chat_id, request.message_id, request.message,
+        )
 
         # ── 4. Передаём всё в streaming engine ────────────────────────────
         async for chunk in generate_agent_stream(
             service=self,
             request=request,
             user_token=user_token,
             scopes=scopes,
             history=history,
             all_models=all_models,
             interceptor_config=merged_config,
diff --git a/app/services/react/deepagent/services_sse.py b/app/services/react/deepagent/services_sse.py
index 0b30dcf1..499091cb 100644
--- a/app/services/react/deepagent/services_sse.py
+++ b/app/services/react/deepagent/services_sse.py
@@ -22,25 +22,23 @@ from sber_hrp_py_libs_auth.types import JWTToken
 from app.services.common.selectors import (
     DrugScopeUserSelector,
     HeimdallPuzScopeUserSelector,
     PulsePersonalScopeUserSelector,
     PulseTechnicalScopeUserSelector,
     RecruiterScopeUserSelector,
     SsiodScopeUserSelector,
     TestUserSelector,
 )
 from app.services.delegation_service import DelegationPriorityService, DelegationService
+from app.services.react.deepagent.internet_judge import merge_request_interceptor_configs
 from app.services.react.deepagent.lifecycle import _chunk_to_event_type
-from app.services.react.deepagent.stateful_routing import (
-    make_stateful_interceptor_config,
-    merge_interceptor_configs,
-)
+from app.services.react.deepagent.stateful_routing import make_stateful_interceptor_config
 from app.services.react.entities import EventType
 from app.services.react.repository import (
     AbstractChatMessagesRepository,
     AbstractChatRepository,
     ChatMessagesRepository,
     ChatRepository,
 )
 from app.services.react.settings import react_settings
 from app.services.react.sse import AbstractSseAgentService
 from app.services.react.subagents_config import get_enabled_subagents
@@ -366,21 +364,26 @@ class SseDeepAgentCallAgentService(AbstractSseAgentService):
             "[chat: %s] SSE stateful subagents: %s (from %d total)",
             request.chat_id,
             stateful_names,
             len(all_configs),
         )
         stateful_config = make_stateful_interceptor_config(
             request.chat_id,
             stateful_names,
             chat_history=history,
         )
-        merged_config = merge_interceptor_configs(self.interceptor_config, stateful_config)
+        # Судья mcp_internet: единая точка сборки interceptor-конфига запроса
+        # (base + stateful + judge) — общий хелпер для polling и SSE хэрнессов.
+        merged_config = merge_request_interceptor_configs(
+            self.interceptor_config, stateful_config,
+            request.chat_id, request.message_id, request.message,
+        )
 
         # ── 4. Передаём всё в streaming engine (deepagents) ─────────────
         from app.services.react.deepagent.streaming import generate_agent_stream
 
         async for chunk in generate_agent_stream(
             service=self,
             request=request,
             user_token=user_token,
             scopes=scopes,
             history=history,
diff --git a/app/services/react/deepagent/stateful_routing.py b/app/services/react/deepagent/stateful_routing.py
index b6304912..d26b4ec3 100644
--- a/app/services/react/deepagent/stateful_routing.py
+++ b/app/services/react/deepagent/stateful_routing.py
@@ -441,20 +441,21 @@ def merge_interceptor_configs(
     if extra is None:
         return base
 
     merged = InterceptorConfig(
         tool_interceptors={**base.tool_interceptors, **extra.tool_interceptors},
         model_interceptor=extra.model_interceptor or base.model_interceptor,
         before_model_hook=extra.before_model_hook or base.before_model_hook,
         after_model_hook=extra.after_model_hook or base.after_model_hook,
         before_agent_hook=extra.before_agent_hook or base.before_agent_hook,
         after_agent_hook=extra.after_agent_hook or base.after_agent_hook,
+        internet_gate=extra.internet_gate or base.internet_gate,
     )
     return merged
 
 
 # ── Subagent tool steps in chat history ───────────────────────────────────
 
 # Key in a ``tool_calls`` JSONB element: the tool step was made by this
 # subagent (not by the orchestrator).
 SUBAGENT_ROW_KEY = "subagent"
 
diff --git a/app/services/react/deepagent/streaming.py b/app/services/react/deepagent/streaming.py
index df6350a5..26579c1f 100644
--- a/app/services/react/deepagent/streaming.py
+++ b/app/services/react/deepagent/streaming.py
@@ -27,20 +27,25 @@ from app.schemas.react_schemas import MessageAttachment, MessageResponse
 from app.services.react.callbacks import GigaChatFallbackMiddleware
 from app.services.react.deepagent.backend import build_filesystem_backend as _build_filesystem_backend
 from app.services.react.deepagent.backend import cleanup
 from app.services.react.deepagent.events import (
     event_type_to_message_type,
     extract_text_from_event,
     is_nested_graph_event,
     is_subagent_event,
 )
 from app.services.react.deepagent.interceptor import InterceptorConfig, InterceptorMiddleware
+from app.services.react.deepagent.internet_judge import (
+    attach_internet_judge_metadata,
+    extract_judge_tool_call_id,
+    send_unsent_judge_verdicts,
+)
 from app.services.react.deepagent.model import build_summarization_middleware
 from app.services.react.deepagent.restricted_backend import restricted_fs_middleware
 from app.services.react.deepagent.shell_policy_middleware import ShellToolPolicyMiddleware
 from app.services.react.deepagent.session_middleware import maybe_session_middleware
 from app.services.react.deepagent.stateful_routing import (
     SUBAGENT_ROW_KEY,
     history_for_orchestrator,
     parse_stateful_marker,
     strip_stateful_marker,
 )
@@ -560,20 +565,24 @@ async def generate_agent_stream(
     response_message_id = generate_message_id()
 
     # ── 7.5 Tool-call tracking for stats feedback & history
     # Dict keyed by run_id (not LIFO stack) to correctly pair
     # on_tool_start / on_tool_end for parallel invocations.
     # A dict keyed by run_id ensures the result matches the correct
     # invocation, not the last-started tool in parallel calls.
     _tool_step_sequence: int = 0
     _cycle_start_dt: datetime = datetime.now(timezone.utc)
     _current_tool_stack: dict[str, dict[str, Any]] = {}
+    # REQ-08: id вердиктов судьи, уже ушедших в stat-feedback с tool-шагами
+    # on_tool_end (allow-путь). Заблокированные вызовы (deny/timeout/error)
+    # on_tool_end не эмитят — их вердикты досылаются после цикла событий.
+    _sent_judge_call_ids: set[str] = set()
     # Словарь по run_id (не LIFO-стек) — правильно маппит on_tool_start /
     # on_tool_end для параллельных вызовов. Вложенные тулы: вложенный тул
     # кладёт свой кадр в _current_tool_stack (ключ — его run_id), при
     # завершении забирает свой кадр по своему run_id, фрейм родителя не
     # трогает.
     subagent_names: set[str] = {s["name"] for s in subagent_specs} if subagent_specs else set()
 
     # Track the last `task` tool output — used as fallback if the main agent
     # generates an empty response after delegating to a stateful subagent.
     _last_task_tool_output: str = ""
@@ -592,20 +601,50 @@ async def generate_agent_stream(
     # event loop. If it did, we must NOT yield the raw subagent output again
     # post-loop — that would duplicate the response.
     _main_agent_text_yielded: bool = False
 
     # Track whether a stateful subagent returned via the `task` tool with a
     # [STATEFUL:*] marker. When True, the main agent's post-task LLM text
     # (paraphrase/relay) is suppressed — the raw subagent output goes to the
     # user as-is via the post-loop yield.
     _stateful_task_returned: bool = False
 
+    # REQ-08: вердикты заблокированных вызовов mcp_internet (deny/timeout/error)
+    # не эмитят on_tool_end — их вердикты досылаем отдельными шагами
+    # stat-feedback. Flush выполняется на всех завершающих путях стрима:
+    # терминальные pptx-тулы (early return), нормальный выход из цикла и
+    # аварийный выход 7.7 — чтобы deny-вердикты не терялись в телеметрии.
+    async def _judge_feedback(step_metadata: dict[str, Any], step_sequence: int) -> None:
+        await _send_cycle_feedback(
+            chat_id=chat_id,
+            chat_message_id=chat_message_id,
+            step_input=request.message,
+            user_token=user_token,
+            start_dt=_cycle_start_dt,
+            end_dt=datetime.now(timezone.utc),
+            step_output=None,
+            step_metadata=step_metadata,
+            step_type="tool",
+            step_sequence=step_sequence,
+            step_name="mcp_internet",
+            error=None,
+        )
+
+    async def _flush_unsent_judge_verdicts() -> None:
+        nonlocal _tool_step_sequence
+        _tool_step_sequence += await send_unsent_judge_verdicts(
+            gate=getattr(interceptor_config, "internet_gate", None),
+            sent_tool_call_ids=_sent_judge_call_ids,
+            sender=_judge_feedback,
+            base_step_sequence=_tool_step_sequence,
+        )
+
     async for event in agent.astream_events(
         {"messages": lc_messages},
         version="v2",
         config={"recursion_limit": react_settings.MAX_REACT_ITERATIONS, "configurable": {"thread_id": chat_id}},
     ):
         event_name = event["event"]
 
         # ── Tool tracking ──────────────────────────────────────────
         if event_name == "on_tool_start":
             _current_tool_name = event.get("name", "unknown")
@@ -656,29 +695,40 @@ async def generate_agent_stream(
             tool_output_json = None
             if isinstance(tool_output, dict):
                 tool_output_json = tool_output
             elif tool_output is not None:
                 try:
                     tool_output_json = json.loads(tool_output) if isinstance(tool_output, str) else tool_output
                 except (json.JSONDecodeError, TypeError):
                     tool_output_json = str(tool_output)
 
             _tool_step_sequence += 1
+            # REQ-08: судья ключует вердикты по РЕАЛЬНОМУ tool_call_id из
+            # ToolCallRequest (LLM-вызов тулa). Синтетический call_id кадра
+            # стриминга (call_<uuid>) с ним не совпадает — реальный id берём
+            # из ToolMessage-выхода (доступен в on_tool_end), иначе фолбэк на
+            # call_id кадра.
+            _judge_call_id = extract_judge_tool_call_id(tool_output, _current_tool_call_id)
+            # Вердикт по этому id уходит с текущим tool-шагом — после цикла
+            # досылаем только те, что не попали ни в один tool-шаг.
+            _sent_judge_call_ids.add(_judge_call_id)
             await _send_cycle_feedback(
                 chat_id=chat_id,
                 chat_message_id=chat_message_id,
                 step_input=request.message,
                 user_token=user_token,
                 start_dt=_cycle_start_dt,
                 end_dt=datetime.now(timezone.utc),
                 step_output=tool_output_json,
-                step_metadata=_current_tool_input,
+                step_metadata=attach_internet_judge_metadata(
+                    _current_tool_input, interceptor_config, tool_name, _judge_call_id,
+                ),
                 step_type="tool",
                 step_sequence=_tool_step_sequence,
                 step_name=tool_name,
                 error=_current_tool_error,
             )
 
             logger.info(
                 "[chat: %s] on_tool_end: %s (step %d)",
                 chat_id,
                 tool_name,
@@ -806,20 +856,23 @@ async def generate_agent_stream(
                     yield MessageResponse(
                         uuid=request.message_id,
                         message=msg_text,
                         role=Role.assistant,
                         message_type=MessageType.text,
                         updated_at=datetime.now(timezone.utc),
                         response_message_id=file_response_message_id,
                         attachments=file_attachments,
                     )
 
+                    # REQ-08: терминальный тул завершает генератор до
+                    # post-loop flush — досылаем вердикты здесь.
+                    await _flush_unsent_judge_verdicts()
                     await cleanup(service, request.chat_id, tool_mgr)
                     return
 
             # Reset tracker
             _current_tool_name = None
             _current_tool_input = None
             _current_tool_output = None
             _current_tool_error = None
 
             # Capture `task` tool output for stateful subagent fallback.
@@ -963,22 +1016,30 @@ async def generate_agent_stream(
             "Попробуйте повторить запрос."
         )
         async for error_response in yield_error_response(
             error_content=emergency_error,
             chat_id=chat_id,
             request=request,
             service=service,
             tool_mgr=tool_mgr,
         ):
             yield error_response
+        # REQ-08: аварийный выход завершает генератор — досылаем
+        # вердикты до return, чтобы deny-шаги попали в телеметрию.
+        await _flush_unsent_judge_verdicts()
         return
 
+    # REQ-08: нормальный выход из цикла — досылаем вердикты заблокированных
+    # вызовов до финального message-шага (секция 8); счётчик шагов
+    # продвигается внутри flush, финал получит следующий свободный номер.
+    await _flush_unsent_judge_verdicts()
+
     # ── 8. Сохраняем итоговый ответ модели в историю чата ────────────
     # If the `task` tool returned a response with a [STATEFUL:*] marker,
     # prefer it over the main agent's paraphrased response for DB storage.
     # Only yield to the user if the main agent did NOT already stream text
     # during the event loop — otherwise the user would see duplicate output.
     task_marker = parse_stateful_marker(_last_task_tool_output)
     if task_marker is not None:
         # Bare marker — name it after the subagent that actually answered,
         # otherwise routing falls back to the first stateful subagent.
         if task_marker.name is None and _last_task_subagent in _stateful_names:
diff --git a/tests/test_deepagent_internet_judge_wiring.py b/tests/test_deepagent_internet_judge_wiring.py
new file mode 100644
index 00000000..c34594e7
--- /dev/null
+++ b/tests/test_deepagent_internet_judge_wiring.py
@@ -0,0 +1,329 @@
+"""Wiring судьи: build/merge/middleware/step-metadata (без живого LLM)."""
+
+import pytest
+from langchain_core.messages import ToolMessage
+from langgraph.prebuilt.tool_node import ToolCallRequest
+
+from app.core.config import app_settings
+from app.services.react.deepagent.interceptor import (
+    InterceptorConfig,
+    InterceptorMiddleware,
+    ToolInterceptor,
+)
+from app.services.react.deepagent.internet_judge import (
+    InternetJudgeGate,
+    InternetJudgeVerdict,
+    attach_internet_judge_metadata,
+    build_internet_judge_config,
+    extract_judge_tool_call_id,
+    merge_request_interceptor_configs,
+    send_unsent_judge_verdicts,
+)
+from app.services.react.deepagent.stateful_routing import merge_interceptor_configs
+
+
+def _request(call_id: str = "call_w") -> ToolCallRequest:
+    return ToolCallRequest(
+        tool_call={"name": "mcp_internet", "args": {"query": "погода в Москве"}, "id": call_id},
+        tool=None,
+        state={},
+        runtime=None,
+    )
+
+
+# ── build_internet_judge_config ────────────────────────────────────────────
+
+
+def test_build_disabled_returns_none(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
+    assert build_internet_judge_config("c", "m", "вопрос") is None
+
+
+def test_build_enabled_registers_gate_and_interceptor(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "вопрос")
+    assert cfg is not None
+    assert isinstance(cfg.internet_gate, InternetJudgeGate)
+    assert cfg.internet_gate.user_question == "вопрос"
+    ti = cfg.tool_interceptors["mcp_internet"]
+    assert ti.pre_execute == cfg.internet_gate.pre_execute
+
+
+# ── merge сохраняет gate ────────────────────────────────────────────────────
+
+
+def test_merge_preserves_internet_gate(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "q")
+    other = InterceptorConfig()
+    other.register_tool_interceptor("some_tool", ToolInterceptor())
+    merged = merge_interceptor_configs(other, cfg)
+    assert merged.internet_gate is cfg.internet_gate
+    assert "some_tool" in merged.tool_interceptors
+    assert "mcp_internet" in merged.tool_interceptors
+
+
+def test_merge_none_base_passes_gate_through(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "q")
+    merged = merge_interceptor_configs(None, cfg)
+    assert merged is cfg
+
+
+# ── e2e через middleware: deny блокирует handler (REQ-01, REQ-03) ──────────
+
+
+@pytest.mark.asyncio
+async def test_middleware_deny_blocks_internet(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "Хочу обучение по Java")
+    gate = cfg.internet_gate
+
+    async def fake_llm(messages):
+        return '{"verdict": "deny", "reason": "погода не связана с обучением"}'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    mw = InterceptorMiddleware(cfg)
+    handler_called = False
+
+    async def handler(req):
+        nonlocal handler_called
+        handler_called = True
+        return ToolMessage(content="INTERNET RESULT", name=req.tool_call["name"], tool_call_id=req.tool_call["id"])
+
+    result = await mw.awrap_tool_call(_request(), handler)
+    assert handler_called is False  # в интернет не ходили
+    assert result.status == "error"
+    assert "погода не связана с обучением" in result.content
+
+
+@pytest.mark.asyncio
+async def test_middleware_allow_passes_through(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "Хочу обучение по Java")
+    gate = cfg.internet_gate
+
+    async def fake_llm(messages):
+        return '{"verdict": "allow", "reason": "поиск курса"}'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    mw = InterceptorMiddleware(cfg)
+
+    async def handler(req):
+        return ToolMessage(content="INTERNET RESULT", name=req.tool_call["name"], tool_call_id=req.tool_call["id"])
+
+    result = await mw.awrap_tool_call(_request(), handler)
+    assert result.content == "INTERNET RESULT"
+    assert gate.verdicts["call_w"].verdict == "allow"
+
+
+# ── step_metadata (REQ-08) ─────────────────────────────────────────────────
+
+
+def test_attach_metadata_adds_verdict(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "q")
+    gate = cfg.internet_gate
+    gate.verdicts["call_w"] = InternetJudgeVerdict("deny", "не обучение")
+    meta = attach_internet_judge_metadata({"query": "погода"}, cfg, "mcp_internet", "call_w")
+    assert meta["internet_judge"] == {"verdict": "deny", "reason": "не обучение"}
+    assert meta["query"] == "погода"
+
+
+def test_attach_metadata_noop_for_other_tools(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    cfg = build_internet_judge_config("c", "m", "q")
+    meta = attach_internet_judge_metadata({"a": 1}, cfg, "mcp_query", "call_x")
+    assert meta == {"a": 1}
+
+
+def test_attach_metadata_none_config(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    meta = attach_internet_judge_metadata(None, None, "mcp_internet", "call_x")
+    assert meta == {}
+
+
+# ── merge_request_interceptor_configs (F1: SSE-паритет) ────────────────────
+
+
+def test_merge_request_configs_includes_judge(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    base = InterceptorConfig()
+    base.register_tool_interceptor("x", ToolInterceptor())
+    merged = merge_request_interceptor_configs(base, None, "c", "m", "q")
+    assert isinstance(merged.internet_gate, InternetJudgeGate)
+    assert merged.internet_gate.user_question == "q"
+    assert set(merged.tool_interceptors) >= {"x", "mcp_internet"}
+
+
+def test_merge_request_configs_disabled_keeps_plain_stack(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
+    base = InterceptorConfig()
+    base.register_tool_interceptor("x", ToolInterceptor())
+    st = InterceptorConfig()
+    st.register_tool_interceptor("y", ToolInterceptor())
+    merged = merge_request_interceptor_configs(base, st, "c", "m", "q")
+    assert merged.internet_gate is None
+    assert set(merged.tool_interceptors) == {"x", "y"}
+
+
+def test_merge_request_configs_all_none_returns_none(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
+    assert merge_request_interceptor_configs(None, None, "c", "m", "q") is None
+
+
+# ── send_unsent_judge_verdicts (F2: телеметрия deny) ───────────────────────
+
+
+@pytest.mark.asyncio
+async def test_send_unsent_none_gate_noop():
+    calls = []
+
+    async def sender(metadata, sequence):
+        calls.append((metadata, sequence))
+
+    assert await send_unsent_judge_verdicts(None, None, sender, 5) == 0
+    assert calls == []
+
+
+@pytest.mark.asyncio
+async def test_send_unsent_empty_verdicts_noop(monkeypatch):
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    calls = []
+
+    async def sender(metadata, sequence):
+        calls.append((metadata, sequence))
+
+    gate = build_internet_judge_config("c", "m", "q").internet_gate
+    assert await send_unsent_judge_verdicts(gate, set(), sender, 0) == 0
+    assert calls == []
+
+
+@pytest.mark.asyncio
+async def test_send_unsent_sends_only_blocked_verdicts():
+    gate = InternetJudgeGate(chat_id="c", message_id="m", user_question="q")
+    gate.verdicts["call_a"] = InternetJudgeVerdict("allow", "ок")
+    gate.verdicts["call_b"] = InternetJudgeVerdict("deny", "не обучение")
+    gate.verdicts["call_c"] = InternetJudgeVerdict("timeout", "таймаут")
+    calls = []
+
+    async def sender(metadata, sequence):
+        calls.append((metadata, sequence))
+
+    sent = await send_unsent_judge_verdicts(gate, {"call_a"}, sender, 7)
+    assert sent == 2
+    assert [c[0]["internet_judge"]["verdict"] for c in calls] == ["deny", "timeout"]
+    assert [c[1] for c in calls] == [8, 9]
+
+
+@pytest.mark.asyncio
+async def test_send_unsent_sender_failure_is_isolated(caplog):
+    gate = InternetJudgeGate(chat_id="c", message_id="m", user_question="q")
+    gate.verdicts["call_b"] = InternetJudgeVerdict("deny", "не обучение")
+    gate.verdicts["call_c"] = InternetJudgeVerdict("error", "сбой")
+
+    async def boom(metadata, sequence):
+        raise RuntimeError("feedback storage down")
+
+    assert await send_unsent_judge_verdicts(gate, set(), boom, 0) == 0
+
+
+# ── extract_judge_tool_call_id (F4: пиннинг lookup-id) ─────────────────────
+
+
+def test_extract_id_from_tool_message():
+    tm = ToolMessage(content="x", name="mcp_internet", tool_call_id="call_REAL")
+    assert extract_judge_tool_call_id(tm, "call_SYNTH") == "call_REAL"
+
+
+def test_extract_id_fallback_for_non_tool_message():
+    assert extract_judge_tool_call_id("plain string", "call_SYNTH") == "call_SYNTH"
+    assert extract_judge_tool_call_id(None, "call_SYNTH") == "call_SYNTH"
+
+
+# ── e2e: реальный tool_call_id доезжает до lookup'а step_metadata ───────────
+# Судья ключует вердикты по РЕАЛЬНОМУ tool_call_id из ToolCallRequest
+# (streaming-кадр использует синтетический call_<uuid> — с ним lookup не
+# совпадает). Реальный id доступен в on_tool_end из ToolMessage-выхода.
+# Интеграционный прогон (scripted model + create_deep_agent) фиксирует оба
+# факта и итоговое обогащение step_metadata вердиктом.
+
+
+@pytest.mark.asyncio
+async def test_real_tool_call_id_reaches_step_metadata(monkeypatch):
+    from langchain_core.language_models.chat_models import BaseChatModel
+    from langchain_core.messages import AIMessage, HumanMessage
+    from langchain_core.outputs import ChatGeneration, ChatResult
+    from langchain_core.tools import tool
+    from pydantic import Field
+
+    from app.services.react.deepagent.interceptor import InterceptorMiddleware
+
+    class _ScriptedModel(BaseChatModel):
+        responses: list = Field(default_factory=list)
+        idx: int = 0
+
+        @property
+        def _llm_type(self) -> str:
+            return "scripted"
+
+        def bind_tools(self, tools, **kwargs):
+            return self
+
+        def _produce(self):
+            i = min(self.idx, len(self.responses) - 1)
+            self.idx += 1
+            return ChatResult(generations=[ChatGeneration(message=self.responses[i])])
+
+        def _generate(self, messages, stop=None, run_manager=None, **kwargs):
+            return self._produce()
+
+        async def _agenerate(self, messages, stop=None, run_manager=None, **kwargs):
+            return self._produce()
+
+    @tool
+    def mcp_internet(query: str) -> str:
+        """Internet search."""
+        return f"RESULT FOR {query}"
+
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    real_call_id = "call_REAL_9"
+    model = _ScriptedModel(
+        responses=[
+            AIMessage(
+                content="",
+                tool_calls=[{"name": "mcp_internet", "args": {"query": "курс Java"}, "id": real_call_id}],
+            ),
+            AIMessage(content="готово"),
+        ],
+    )
+    cfg = build_internet_judge_config("c", "m", "Хочу обучение по Java")
+
+    async def fake_llm(messages):
+        return '{"verdict": "allow", "reason": "поиск курса"}'
+
+    monkeypatch.setattr(cfg.internet_gate, "_call_llm", fake_llm)
+
+    from deepagents import create_deep_agent
+
+    agent = create_deep_agent(
+        model,
+        [mcp_internet],
+        system_prompt="Оркестратор.",
+        middleware=[InterceptorMiddleware(cfg)],
+    )
+
+    seen = False
+    async for event in agent.astream_events({"messages": [HumanMessage("q")]}, version="v2"):
+        if event["event"] == "on_tool_end" and event.get("name") == "mcp_internet":
+            seen = True
+            output = event.get("data", {}).get("output")
+            # Пиннинг прод-кода: тот же lookup-id, что использует streaming.on_tool_end
+            extracted_id = extract_judge_tool_call_id(output, "")
+            assert extracted_id == real_call_id
+            input_args = event.get("data", {}).get("input") or {}
+            meta = attach_internet_judge_metadata(input_args, cfg, "mcp_internet", extracted_id)
+            assert meta["internet_judge"] == {"verdict": "allow", "reason": "поиск курса"}
+
+    assert seen
+    assert cfg.internet_gate.verdicts[real_call_id].verdict == "allow"
