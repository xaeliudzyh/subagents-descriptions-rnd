# Fix-бриф задачи 2.1: находки A и B (решение человека на чекпоинте волны 2)

- Ветка: mcp-internet-judge; BASE фикса: c22a10d4
- Утверждено человеком: починить обе находки сейчас (чекпоинт волны 2)
- Файлы: `app/services/react/deepagent/internet_judge.py`, `tests/test_deepagent_internet_judge.py` — только эти два

## Находка A — вход судьи человекочитаемым текстом (REQ-02, REQ-06)

Что не так (обе оси ревью задачи 2.1): `internet_judge.py`, метод `pre_execute`,
строки ~184-186: `query = args if isinstance(args, str) else json.dumps(args,
ensure_ascii=False)` — судья получает JSON-обёртку вида
`{"query": "статья..."}`, тогда как few-shot-примеры промпта и тестовая корзина
(test.xlsx, приёмка REQ-02 ≥ 0.90) — bare-строки. Рассинхрон формата входа и
few-shot снижает ожидаемую accuracy корзины.

Фикс (заменить конверсию в `pre_execute`):

```python
        tool_call = request.tool_call
        args = tool_call.get("args", {})
        if isinstance(args, str):
            query = args
        elif isinstance(args, dict):
            text_parts = [v for v in args.values() if isinstance(v, str) and v.strip()]
            query = " ".join(text_parts) or json.dumps(args, ensure_ascii=False)
        else:
            query = json.dumps(args, ensure_ascii=False)
```

Смысл: судья видит голый текст запроса независимо от имени поля в схеме тула
(имя поля приходит с MCP-сервера динамически); нестроковые значения (лимиты,
флаги) не шумят промпт; fallback на json.dumps для пустых/экзотических args.
REQ-06 сохранён: судья по-прежнему получает фактические аргументы вызова
(все строковые параметры), в читабельной форме.

Новые тесты (добавить в tests/test_deepagent_internet_judge.py):

```python
@pytest.mark.asyncio
async def test_dict_args_passed_as_plain_text(monkeypatch):
    """REQ-06/находка A: dict-args идут судье bare-текстом, не JSON-обёрткой."""
    gate = _gate()
    captured: list[list] = []

    async def fake_llm(messages):
        captured.append(messages)
        return '{"verdict": "allow", "reason": "поиск обучения"}'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    request = ToolCallRequest(
        tool_call={
            "name": "mcp_internet",
            "args": {"query": "найди курс по Java", "limit": 5},
            "id": "call_dict",
        },
        tool=None,
        state={},
        runtime=None,
    )
    result = await gate.pre_execute(request)
    assert result is None
    user_content = captured[-1][-1]["content"]
    assert "найди курс по Java" in user_content
    assert '{"query"' not in user_content  # нет JSON-обёртки
    assert gate.verdicts["call_dict"].verdict == "allow"


@pytest.mark.asyncio
async def test_str_args_passed_through(monkeypatch):
    """Находка A: строковые args идут в судью как есть."""
    gate = _gate()
    captured: list[list] = []

    async def fake_llm(messages):
        captured.append(messages)
        return '{"verdict": "deny", "reason": "не обучение"}'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    request = ToolCallRequest(
        tool_call={"name": "mcp_internet", "args": "погода в Москве", "id": "call_str"},
        tool=None,
        state={},
        runtime=None,
    )
    await gate.pre_execute(request)
    assert "погода в Москве" in captured[-1][-1]["content"]
    assert gate.verdicts["call_str"].verdict == "deny"
```

## Находка B — причина сбоя без деталей исключения (REQ-08, безопасность)

Что не так: `pre_execute`, строка ~199:
`verdict = InternetJudgeVerdict("error", f"сбой LLM-судьи: {exc}")` — текст
исключения (могут быть URL эндпоинтов, внутренние детали) уходит в agent-facing
metadata вердикта (`to_metadata()` → step_metadata stat-feedback) и потенциально
в tool-result. Полные детали уже пишутся в `logger.error` строкой выше.

Фикс:

```python
            verdict = InternetJudgeVerdict("error", "сбой LLM-судьи")
```

(exc остаётся только в logger.error; строку лога не менять.)

Новый тест:

```python
@pytest.mark.asyncio
async def test_error_reason_has_no_exception_details(monkeypatch):
    """Находка B: reason не содержит деталей исключения."""
    gate = _gate()

    async def broken_llm(messages):
        raise RuntimeError("gigachat down at http://internal-host:8080/v1")

    monkeypatch.setattr(gate, "_call_llm", broken_llm)
    result = await gate.pre_execute(_request("call_leak"))
    assert result is not None and result.status == "error"
    assert "internal-host" not in result.content
    assert gate.verdicts["call_leak"].reason == "сбой LLM-судьи"
    assert "internal-host" not in str(gate.verdicts["call_leak"].to_metadata())
```

## Шаги

1. Добавь три новых теста (RED: первые два упадут на JSON-обёртке/ассертах,
   третий — на деталях в reason). Зафиксируй RED.
2. Внеси оба фикса в internet_judge.py (конверсия args + reason без exc).
3. GREEN: `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v`
   — ожидай 13 passed (10 прежних + 3 новых).
4. Регресс: `.venv/bin/python -m pytest tests/test_deepagent_interceptor_async.py -v` — 6 passed.
5. `.venv/bin/ruff check app/services/react/deepagent/internet_judge.py tests/test_deepagent_internet_judge.py` — 0 замечаний.
6. Commit: `fix(02): вход судьи bare-текстом; причина сбоя без деталей (REQ-02,06,08)`
   — стейджить только эти два файла.
7. Допиши раздел «Фикс A/B» в конец отчёта .sberpowers/tasks/21-report.md
   (RED/GREEN выводы, self-review).

## Критерии выхода

- 13 passed в тестах судьи; регресс 1.1 зелёная; ruff чист.
- Судья получает bare-текст при dict-args со строковыми значениями; str-args —
  как есть; пустой/экзотический dict — fallback json.
- reason error-вердикта не содержит текста исключения.
