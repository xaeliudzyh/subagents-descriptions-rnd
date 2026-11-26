# Бриф задачи 1.1: Async-хуки interceptor (волна 1)

- Ветка: mcp-internet-judge; BASE задачи: 6a4041ed
- REQ: REQ-mcp-internet-judge-05
- Blocked by: нет

## Глобальные ограничения плана (дословно)

- deepagents==0.7.14 — версию не трогаем (`requirements/base.txt:41`).
- Python 3.13, venv проекта `.venv`; тесты запускать
  `.venv/bin/python -m pytest <путь> -v` из корня репо.
- Ruff: line length 120, type hints обязательны (`pyproject.toml`).
- Конфигурация — только поля `AppSettings` в `app/core/config.py` (значения в
  коде); .env не используем.
- Тесты без живого LLM: GigaChat всегда мок (патчим фабричный метод gate).
- Конвенция имён тестов: `tests/test_deepagent_<тема>.py`.
- Один таск → один атомарный коммит; сообщение содержит номер задачи и REQ-ID:
  `feat(01): async-хуки ToolInterceptor, pre/post_execute (REQ-05)`.
- Каждый прогон тестов читать выводом, не кодом возврата памяти.

## Разрешённые диспетчером неоднозначности

- Async-тесты: проектная конвенция — `@pytest.mark.asyncio` + `async def test_...`
  (см. tests/test_deepagent_session_id.py:170+); pytest-asyncio установлен в .venv.
- Коммит-хук харнесса автоматически добавляет Co-authored-by trailer — это норма,
  отдельного действия не требуется.

## Контекст задачи

`ToolInterceptor.pre_execute`/`post_execute` сейчас принимают только синхронные
callable, а LLM-судья (следующая задача 2.1) — асинхронный. Расширяем контракт
без ломки обратной совместимости: coroutine-callable вызывается с `await`,
обычные callable работают как раньше. Это фундамент для задачи 2.1.

## Файлы

- Изменить: `app/services/react/deepagent/interceptor.py`
- Создать: `tests/test_deepagent_interceptor_async.py`

## Интерфейсы

- Потребляет: `awrap_tool_call(request: ToolCallRequest, handler) -> ToolMessage`
  — существующая сигнатура, не меняется.
- Производит: `ToolInterceptor.pre_execute`/`post_execute` принимают
  `Callable | Coroutine-function`; возвращает то же, что раньше. На это
  опираются задачи 2.1 и 3.1.

## Шаги

1. Напиши падающий тест `tests/test_deepagent_interceptor_async.py`:

```python
"""Async-хуки ToolInterceptor: coroutine pre/post_execute без блокировки loop."""

import asyncio

import pytest
from langchain_core.messages import ToolMessage
from langgraph.prebuilt.tool_node import ToolCallRequest

from app.services.react.deepagent.interceptor import (
    InterceptorConfig,
    InterceptorMiddleware,
    ToolInterceptor,
    ToolOverrideResult,
)


def _request(name: str = "mcp_internet", call_id: str = "call_1") -> ToolCallRequest:
    return ToolCallRequest(
        tool_call={"name": name, "args": {"query": "тест"}, "id": call_id},
        tool=None,
        state={},
        runtime=None,
    )


def _middleware(**kwargs) -> InterceptorMiddleware:
    return InterceptorMiddleware(InterceptorConfig(**kwargs))


async def _ok_handler(request: ToolCallRequest) -> ToolMessage:
    return ToolMessage(
        content="real result", name=request.tool_call["name"],
        tool_call_id=request.tool_call["id"],
    )


@pytest.mark.asyncio
async def test_async_pre_execute_skip():
    """Coroutine pre_execute может вернуть ToolOverrideResult — тул пропущен."""
    async def pre(req):
        await asyncio.sleep(0)
        return ToolOverrideResult(content="заблокировано", status="error")

    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
    handler_called = False

    async def handler(req):
        nonlocal handler_called
        handler_called = True
        return await _ok_handler(req)

    result = await mw.awrap_tool_call(_request("t"), handler)
    assert handler_called is False
    assert result.status == "error"
    assert "заблокировано" in result.content


@pytest.mark.asyncio
async def test_async_pre_execute_override_args():
    """Coroutine pre_execute может вернуть модифицированный request.override(...)."""
    async def pre(req):
        await asyncio.sleep(0)
        return req.override(tool_call={**req.tool_call, "args": {"query": "изменено"}})

    seen: dict = {}
    async def handler(req):
        seen.update(req.tool_call["args"])
        return await _ok_handler(req)

    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
    result = await mw.awrap_tool_call(_request("t"), handler)
    assert seen == {"query": "изменено"}
    assert result.content == "real result"


@pytest.mark.asyncio
async def test_sync_pre_execute_still_works():
    """Синхронный pre_execute не сломан (обратная совместимость)."""
    def pre(req):
        return ToolOverrideResult(content="sync skip")

    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
    assert result.content == "sync skip"


@pytest.mark.asyncio
async def test_async_pre_execute_exception_fails_open():
    """Исключение в coroutine pre_execute глотается — идёт оригинальный запрос."""

    async def pre(req):
        raise RuntimeError("boom")

    mw = _middleware(tool_interceptors={"t": ToolInterceptor(pre_execute=pre)})
    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
    assert result.content == "real result"


@pytest.mark.asyncio
async def test_async_post_execute_modifies_result():
    """Coroutine post_execute возвращает изменённый ToolMessage."""

    async def post(req, msg):
        await asyncio.sleep(0)
        return ToolMessage(
            content=msg.content + " +judge",
            name=msg.name,
            tool_call_id=msg.tool_call_id,
        )

    mw = _middleware(tool_interceptors={"t": ToolInterceptor(post_execute=post)})
    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
    assert result.content == "real result +judge"


@pytest.mark.asyncio
async def test_sync_post_execute_still_works():
    """Синхронный post_execute не сломан."""
    def post(req, msg):
        return ToolMessage(
            content="replaced", name=msg.name, tool_call_id=msg.tool_call_id,
        )

    mw = _middleware(tool_interceptors={"t": ToolInterceptor(post_execute=post)})
    result = await mw.awrap_tool_call(_request("t"), _ok_handler)
    assert result.content == "replaced"
```

2. Запусти `.venv/bin/python -m pytest tests/test_deepagent_interceptor_async.py -v`
   — ожидай FAIL на всех async-кейсах: coroutine-объект не распознаётся,
   `isinstance(result, ToolOverrideResult)` == False, handler получает мусор.
   (TDD RED — зафиксируй падающий вывод в отчёте.)

3. Реализуй в `interceptor.py`:

В `ToolInterceptor` расширь аннотации полей:

```python
from collections.abc import Coroutine
from typing import Awaitable

    pre_execute: (
        Callable[[ToolCallRequest], "ToolCallRequest | ToolOverrideResult | None"]
        | Callable[[ToolCallRequest], Awaitable["ToolCallRequest | ToolOverrideResult | None"]]
        | None
    ) = None
    post_execute: (
        Callable[[ToolCallRequest, ToolMessage], ToolMessage]
        | Callable[[ToolCallRequest, ToolMessage], Awaitable[ToolMessage]]
        | None
    ) = None
```

Добавь модульный helper:

```python
async def _resolve_hook_result(result: Any) -> Any:
    """Await hook result if it's awaitable; sync results pass through."""
    if inspect.isawaitable(result):
        return await result
    return result
```

(модуль `inspect` уже импортируется локально в `_is_async_callable` — подними
`import inspect` наверх файла и убери локальный импорт).

В `awrap_tool_call` замени вызовы хуков (сохраняя try/except-семантику и логи):

```python
        if interceptor.pre_execute is not None:
            try:
                result = interceptor.pre_execute(request)
                result = await _resolve_hook_result(result)
            except Exception:
                logger.exception(
                    "Tool interceptor pre_execute for '%s' raised; proceeding with original request",
                    tool_name,
                )
                result = None
```

и в post-ветке:

```python
        if interceptor.post_execute is not None:
            try:
                modified = interceptor.post_execute(exec_request, tool_message)
                modified = await _resolve_hook_result(modified)
                if modified is not None:
                    tool_message = modified
            except Exception:
                logger.exception(
                    "Tool interceptor post_execute for '%s' raised; returning original result",
                    tool_name,
                )
```

4. Запусти `.venv/bin/python -m pytest tests/test_deepagent_interceptor_async.py -v`
   — ожидай PASS всех 6. (TDD GREEN.)

5. Регресс: `.venv/bin/python -m pytest tests/test_deepagent_session_id.py -v`
   плюс прочие существующие тесты interceptor-зоны: если есть
   `tests/test_deepagent_interceptor*.py` или использующие InterceptorMiddleware —
   прогони их же. Ожидай PASS без изменений контракта.

6. `.venv/bin/ruff check app/services/react/deepagent/interceptor.py tests/test_deepagent_interceptor_async.py` — 0 замечаний.
   Также `.venv/bin/ruff format --check` на этих двух файлах, если проект им
   пользуется; при расхождениях с соседним стилем — приоритет стилю файла.

7. Commit: `feat(01): async-хуки ToolInterceptor, pre/post_execute (REQ-05)`
   (стейджить только два файла задачи).

## Критерии выхода

- `tests/test_deepagent_interceptor_async.py` зелёный (6 PASS).
- Прежние sync-использования interceptor'а (stateful_routing, declarative builder) не менялись.
- `ruff check` чист.
