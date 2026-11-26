15cc86f5 feat(01): async-хуки ToolInterceptor, pre/post_execute (REQ-05)

 app/services/react/deepagent/interceptor.py |  30 +++++--
 tests/test_deepagent_interceptor_async.py   | 126 ++++++++++++++++++++++++++++
 2 files changed, 148 insertions(+), 8 deletions(-)

diff --git a/app/services/react/deepagent/interceptor.py b/app/services/react/deepagent/interceptor.py
index d361cfc9..749c11bd 100644
--- a/app/services/react/deepagent/interceptor.py
+++ b/app/services/react/deepagent/interceptor.py
@@ -41,28 +41,31 @@ Usage::
 
     config.register_tool_interceptor("mcp_query", ToolInterceptor(post_execute=log_result))
 
 Then pass ``InterceptorMiddleware(config)`` to ``create_deep_agent(middleware=[...])``
 or let ``generate_agent_stream`` wire it automatically via the ``interceptor_config``
 parameter.
 """
 
 from __future__ import annotations
 
+import inspect
 import logging
 from dataclasses import dataclass, field
 from typing import TYPE_CHECKING, Any, Callable
 
 from langchain.agents.middleware.types import AgentMiddleware
 from langchain_core.messages import ToolMessage
 
 if TYPE_CHECKING:
+    from collections.abc import Awaitable
+
     from langchain.agents.middleware.types import ModelRequest, ModelResponse
     from langchain_core.messages import AIMessage
     from langgraph.prebuilt.tool_node import ToolCallRequest
     from langgraph.runtime import Runtime
 
 logger = logging.getLogger(__name__)
 
 
 # ── Result types ────────────────────────────────────────────────────────────
 
@@ -99,41 +102,45 @@ class ModelOverrideResult:
     tool_calls: list[dict[str, Any]] | None = None
 
 
 # ── Interceptor configs ─────────────────────────────────────────────────────
 
 
 @dataclass
 class ToolInterceptor:
     """Interceptor for a specific tool, matched by name.
 
-    Both hooks are optional — register only what you need.
+    Both hooks are optional — register only what you need. Each hook may be a
+    plain callable or a coroutine function; coroutine results are awaited in
+    ``awrap_tool_call`` without blocking the event loop.
 
     Attributes:
         pre_execute: Called *before* the tool runs.
             - Return a ``ToolCallRequest`` (via ``request.override(...)``) to
               modify args/name and proceed with execution.
             - Return a ``ToolOverrideResult`` to skip execution entirely.
             - Return ``None`` to proceed with the original request unchanged.
         post_execute: Called *after* the tool runs with the original/modified
             request and the resulting ``ToolMessage``. Return a new or modified
             ``ToolMessage`` to alter the result the agent sees.
     """
 
     pre_execute: (
-        Callable[
-            [ToolCallRequest],
-            ToolCallRequest | ToolOverrideResult | None,
-        ]
+        Callable[[ToolCallRequest], ToolCallRequest | ToolOverrideResult | None]
+        | Callable[[ToolCallRequest], Awaitable[ToolCallRequest | ToolOverrideResult | None]]
+        | None
+    ) = None
+    post_execute: (
+        Callable[[ToolCallRequest, ToolMessage], ToolMessage]
+        | Callable[[ToolCallRequest, ToolMessage], Awaitable[ToolMessage]]
         | None
     ) = None
-    post_execute: Callable[[ToolCallRequest, ToolMessage], ToolMessage] | None = None
 
 
 @dataclass
 class ModelInterceptor:
     """Interceptor for LLM calls.
 
     Attributes:
         pre_execute: Called *before* the model is invoked.
             - Return a ``ModelRequest`` (via ``request.override(...)``) to
               modify messages, tools, system prompt, model, etc.
@@ -154,22 +161,20 @@ class ModelInterceptor:
 # ── Hook types ──────────────────────────────────────────────────────────────
 
 # State hooks — receive (state, runtime), return dict[str, Any] | None
 StateHookSync = Callable[[Any, Any], "dict[str, Any] | None"]
 StateHookAsync = Callable[[Any, Any], "Any"]
 StateHook = StateHookSync | StateHookAsync
 
 
 def _is_async_callable(fn: Callable) -> bool:
     """Check if a callable is a coroutine function or returns a coroutine."""
-    import inspect
-
     return inspect.iscoroutinefunction(fn)
 
 
 @dataclass
 class InterceptorConfig:
     """Registry of all interceptors and hooks.
 
     Create an instance, register interceptors, then pass to
     ``InterceptorMiddleware`` (or to ``generate_agent_stream`` via the
     ``interceptor_config`` parameter).
@@ -244,20 +249,21 @@ class InterceptorMiddleware(AgentMiddleware):
         if interceptor is None:
             return await handler(request)
 
         # ── pre_execute ────────────────────────────────────────────────
         modified_request: ToolCallRequest | None = None
         skip_result: ToolOverrideResult | None = None
 
         if interceptor.pre_execute is not None:
             try:
                 result = interceptor.pre_execute(request)
+                result = await _resolve_hook_result(result)
             except Exception:
                 logger.exception(
                     "Tool interceptor pre_execute for '%s' raised; proceeding with original request",
                     tool_name,
                 )
                 result = None
 
             if isinstance(result, ToolOverrideResult):
                 skip_result = result
             elif result is not None:
@@ -282,20 +288,21 @@ class InterceptorMiddleware(AgentMiddleware):
                 logger.info(
                     "[interceptor] Tool '%s' args modified by interceptor",
                     tool_name,
                 )
             tool_message = await handler(exec_request)
 
         # ── post_execute ───────────────────────────────────────────────
         if interceptor.post_execute is not None:
             try:
                 modified = interceptor.post_execute(exec_request, tool_message)
+                modified = await _resolve_hook_result(modified)
                 if modified is not None:
                     tool_message = modified
             except Exception:
                 logger.exception(
                     "Tool interceptor post_execute for '%s' raised; returning original result",
                     tool_name,
                 )
 
         return tool_message
 
@@ -344,20 +351,27 @@ class InterceptorMiddleware(AgentMiddleware):
     async def abefore_agent(self, state: Any, runtime: Runtime) -> dict[str, Any] | None:
         return await _maybe_await_hook(self.config.before_agent_hook, state, runtime)
 
     async def aafter_agent(self, state: Any, runtime: Runtime) -> dict[str, Any] | None:
         return await _maybe_await_hook(self.config.after_agent_hook, state, runtime)
 
 
 # ── Helpers ─────────────────────────────────────────────────────────────────
 
 
+async def _resolve_hook_result(result: Any) -> Any:
+    """Await hook result if it's awaitable; sync results pass through."""
+    if inspect.isawaitable(result):
+        return await result
+    return result
+
+
 async def _maybe_await_hook(
     hook: StateHook | None,
     state: Any,
     runtime: Any,
 ) -> dict[str, Any] | None:
     """Call a state hook (sync or async) and return its result."""
     if hook is None:
         return None
     try:
         if _is_async_callable(hook):
diff --git a/tests/test_deepagent_interceptor_async.py b/tests/test_deepagent_interceptor_async.py
new file mode 100644
index 00000000..ca6d0d66
--- /dev/null
+++ b/tests/test_deepagent_interceptor_async.py
@@ -0,0 +1,126 @@
+"""Async-хуки ToolInterceptor: coroutine pre/post_execute без блокировки loop."""
+
+import asyncio
+
+import pytest
+from langchain_core.messages import ToolMessage
+from langgraph.prebuilt.tool_node import ToolCallRequest
+
+from app.services.react.deepagent.interceptor import (
+    InterceptorConfig,
+    InterceptorMiddleware,
+    ToolInterceptor,
+    ToolOverrideResult,
+)
+
+
+def _request(name: str = "mcp_internet", call_id: str = "call_1") -> ToolCallRequest:
+    return ToolCallRequest(
+        tool_call={"name": name, "args": {"query": "тест"}, "id": call_id},
+        tool=None,
+        state={},
+        runtime=None,
+    )
+
+
+def _middleware(**kwargs) -> InterceptorMiddleware:
+    return InterceptorMiddleware(InterceptorConfig(**kwargs))
+
+
+async def _ok_handler(request: ToolCallRequest) -> ToolMessage:
+    return ToolMessage(
+        content="real result", name=request.tool_call["name"],
+        tool_call_id=request.tool_call["id"],
+    )
+
+
+@pytest.mark.asyncio
+async def test_async_pre_execute_skip():
+    """Coroutine pre_execute может вернуть ToolOverrideResult — тул пропущен."""
+    async def pre(req):
+        await asyncio.sleep(0)
+        return ToolOverrideResult(content="заблокировано", status="error")
+
+    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
+    handler_called = False
+
+    async def handler(req):
+        nonlocal handler_called
+        handler_called = True
+        return await _ok_handler(req)
+
+    result = await mw.awrap_tool_call(_request("t"), handler)
+    assert handler_called is False
+    assert result.status == "error"
+    assert "заблокировано" in result.content
+
+
+@pytest.mark.asyncio
+async def test_async_pre_execute_override_args():
+    """Coroutine pre_execute может вернуть модифицированный request.override(...)."""
+    async def pre(req):
+        await asyncio.sleep(0)
+        return req.override(tool_call={**req.tool_call, "args": {"query": "изменено"}})
+
+    seen: dict = {}
+    async def handler(req):
+        seen.update(req.tool_call["args"])
+        return await _ok_handler(req)
+
+    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
+    result = await mw.awrap_tool_call(_request("t"), handler)
+    assert seen == {"query": "изменено"}
+    assert result.content == "real result"
+
+
+@pytest.mark.asyncio
+async def test_sync_pre_execute_still_works():
+    """Синхронный pre_execute не сломан (обратная совместимость)."""
+    def pre(req):
+        return ToolOverrideResult(content="sync skip")
+
+    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
+    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
+    assert result.content == "sync skip"
+
+
+@pytest.mark.asyncio
+async def test_async_pre_execute_exception_fails_open():
+    """Исключение в coroutine pre_execute глотается — идёт оригинальный запрос."""
+
+    async def pre(req):
+        raise RuntimeError("boom")
+
+    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
+    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
+    assert result.content == "real result"
+
+
+@pytest.mark.asyncio
+async def test_async_post_execute_modifies_result():
+    """Coroutine post_execute возвращает изменённый ToolMessage."""
+
+    async def post(req, msg):
+        await asyncio.sleep(0)
+        return ToolMessage(
+            content=msg.content + " +judge",
+            name=msg.name,
+            tool_call_id=msg.tool_call_id,
+        )
+
+    mw = _middleware(tool_interceptors={"t": ToolInterceptor(post_execute=post)})
+    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
+    assert result.content == "real result +judge"
+
+
+@pytest.mark.asyncio
+async def test_sync_post_execute_still_works():
+    """Синхронный post_execute не сломан."""
+    def post(req, msg):
+        return ToolMessage(
+            content="replaced", name=msg.name, tool_call_id=msg.tool_call_id,
+        )
+
+    mw = _middleware(tool_interceptors={"t": ToolInterceptor(post_execute=post)})
+    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
+    assert result.content == "replaced"
