# Бриф задачи 3.1-FIX2: досыл вердиктов на всех выходах + счётчики шагов

- Основано на повторном two-stage ревью b9d804a9..31da4c6f: ось A PARTIAL
  (MISSING: deny → терминальный pptx-тул завершает генератор до досыла),
  ось B NEEDS_FIXES (все прежние находки CLOSED; новые: 2 Important + 2 Minor).
- Ты работаешь в worktree `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.worktrees/wave3-task31`,
  ветка `wave3-task31`, HEAD `31da4c6f`. Третий коммит поверх.

## Глобальные ограничения (без изменений)

- venv общий: `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.venv/bin/python -m pytest <путь> -v`;
  ruff тем же venv; line length 120; type hints.
- Прямой `python -c import` вне event loop не работает (pre-existing DB-pool) — смоук
  через pytest --collect-only.
- Один коммит: `fix(03): досл вердиктов на всех выходах стрима; продвижение step_sequence; step_input вопроса (REQ-08)`.
- Стейджить только файлы задачи (2 файла: streaming.py, tests/test_deepagent_internet_judge_wiring.py).

## Находки повторного ревью и решения диспетчера

### H1 (Important + ось A MISSING): досыл только на нормальном выходе из цикла

`streaming.py`: терминальные pptx-тулы (`convert_to_pptx`/`upload_pptx` с download_url)
завершают генератор через `await cleanup(...); return` ВНУТРИ цикла astream_events
(ветка `if file_url and is_dl:` около строк 813–836) — ДО точки досыла после цикла
(~996–1019). Поток «mcp_internet → deny → модель адаптировалась → convert_to_pptx»
теряет deny-вердикт в телеметрии → приёмка REQ-08 «включая заблокированные» не
выполняется на штатном пути. Спека-ревьюер классифицировал как MISSING.

**Решение диспетчера:** вынести досыл в локальную async-функцию
`_flush_unsent_judge_verdicts()` и вызвать её на ВСЕХ завершающих путях
generate_agent_stream: (а) перед `cleanup(...); return` терминальной ветки;
(б) на нормальном выходе после цикла (текущая точка); (в) перед `return`
аварийного блока 7.7 (orphaned tools) — диспетчер расширил прежнее исключение:
«после любого вызова mcp_internet» должно выполняться на всех путях, где это
технически возможно до завершения генератора.

### H2 (Important): step_sequence не продвигается досылом

`send_unsent_judge_verdicts` инкрементирует свою локальную копию
`base_step_sequence`; прод-счётчик `_tool_step_sequence` не меняется.
Финальный message-шаг (секция 8) берёт `N+1`, хотя досыл уже израсходовал
`N+1..N+k` → дубли/убывание порядковых номеров шагов в витрине agents_feedback.

**Решение:** использовать возвращаемое значение:
`_tool_step_sequence += await send_unsent_judge_verdicts(...)`
внутри `_flush_unsent_judge_verdicts()` через `nonlocal _tool_step_sequence`.

### H3 (Important): step_input=None у досланных шагов

Соседние tool-шаги (on_tool_end) передают `step_input=request.message`
(«Вопрос пользователя» по схеме витрины). Досланные deny-шаги уходят с `None`
→ фильтровка/джойны по вопросу их теряют.

**Решение:** в `_judge_feedback` передать `step_input=request.message`.
Аннотацию `step_input: str` в `_send_cycle_feedback` не менять (теперь она
честна: str всегда).

### H4 (Minor): test_send_unsent_empty_verdicts_noop без явного флага

Добавить `monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)`
(+ параметр monkeypatch), консистентно с F5-принципом первого фикса.

### Решение диспетчера о тестовом покрытии (важно)

Прямого юнит-теста на перенос функций и продвижение счётчика НЕ требуется:
механика send_unsent_judge_verdicts уже покрыта юнитами (возврат счёта,
фильтрация sent-ids), изменения — тривиальный перенос и `+=`; верификация
переноса выполняется ревью по диффу. Не пиши тяжёлых интеграционных тестов
на generate_agent_stream (его зависимости не мокабельны по конвенции зоны).

## Реализация

### 1. RED: обнови тесты tests/test_deepagent_internet_judge_wiring.py

H4: в `test_send_unsent_empty_verdicts_noop` добавь параметр `monkeypatch` и
первой строкой `monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)`.

Других новых тестов не требуется (решение диспетчера выше). Прежние 19 должны
оставаться зелёными — прогони и зафиксируй как baseline перед правкой
прод-кода (RED-этапа по юнитам нет; RED здесь — невозможен без тяжёлых
моков, что диспетчером явно исключено; это фикс-перенос, а не новая логика).

### 2. streaming.py — перенос и продвижение

2a. НАЙДИ блок, вставленный первым фиксом после цикла (около строк 996–1019):

```python
    # REQ-08: заблокированные вызовы mcp_internet (deny/timeout/error) не
    # эмитят on_tool_end — их вердикты досылаем отдельными шагами stat-feedback.
    async def _judge_feedback(step_metadata: dict[str, Any], step_sequence: int) -> None:
        await _send_cycle_feedback(
            ...
            step_input=None,
            ...
        )

    await send_unsent_judge_verdicts(
        gate=getattr(interceptor_config, "internet_gate", None),
        sent_tool_call_ids=_sent_judge_call_ids,
        sender=_judge_feedback,
        base_step_sequence=_tool_step_sequence,
    )
```

УДАЛИ его (на его месте в п. 2d останется один вызов).

2b. ПЕРЕНЕСИ определения ДО цикла `async for event in agent.astream_events(`
(строка ~603): сразу после объявления `_sent_judge_call_ids: set[str] = set()`
или позже — главное, после определения `_send_cycle_feedback` и всех
захваченных локалов (chat_id, chat_message_id, user_token, _cycle_start_dt,
interceptor_config, request); сверь фактические позиции по коду. Новые
определения:

```python
    # REQ-08: вердикты заблокированных вызовов mcp_internet (deny/timeout/error)
    # не эмитят on_tool_end — их вердикты досылаем отдельными шагами
    # stat-feedback. Flush выполняется на всех завершающих путях стрима:
    # терминальные pptx-тулы (early return), нормальный выход из цикла и
    # аварийный выход 7.7 — чтобы deny-вердикты не терялись в телеметрии.
    async def _judge_feedback(step_metadata: dict[str, Any], step_sequence: int) -> None:
        await _send_cycle_feedback(
            chat_id=chat_id,
            chat_message_id=chat_message_id,
            step_input=request.message,
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

    async def _flush_unsent_judge_verdicts() -> None:
        nonlocal _tool_step_sequence
        _tool_step_sequence += await send_unsent_judge_verdicts(
            gate=getattr(interceptor_config, "internet_gate", None),
            sent_tool_call_ids=_sent_judge_call_ids,
            sender=_judge_feedback,
            base_step_sequence=_tool_step_sequence,
        )
```

Внимание: если `_send_cycle_feedback` в этом файле — локальная функция,
определённая РАНЬШЕ точки вставки (она вызывается из on_tool_end), порядок
безопасен; если вдруг она определена позже объявления `_sent_judge_call_ids`,
размести оба определения сразу после неё, но строго до `async for event in agent.astream_events`.

2c. Терминальная ветка pptx-тулов (внутри цикла, ветка `if file_url and is_dl:`):

```python
                    await cleanup(service, request.chat_id, tool_mgr)
                    return
```

замени на:

```python
                    await _flush_unsent_judge_verdicts()
                    await cleanup(service, request.chat_id, tool_mgr)
                    return
```

(если таких `cleanup(...); return` в файле несколько — правь ТОЛЬКО ту, что
в ветке `file_url and is_dl` в обработчике on_tool_end; сверяй контекст по U10
из review-пакета / фактическому коду).

2d. На месте удалённого блока после цикла (п. 2a) оставь:

```python
    await _flush_unsent_judge_verdicts()
```

2e. Аварийный выход 7.7 (orphaned tools): найди блок, где после цикла при
orphaned tool calls формируется `error_response` и генератор завершается
(`async for error_response in ...: yield error_response` и `return`).
Непосредственно ПЕРЕД этим `return` добавь `await _flush_unsent_judge_verdicts()`.
Сверь контекст: вставка не должна попасть внутрь yield-цикла — только между
исчерпанием цикла и return.

### 3. GREEN и регресс

1. `tests/test_deepagent_internet_judge_wiring.py` — 19 passed.
2. Регресс зоны: `tests/test_deepagent_internet_judge.py tests/test_deepagent_interceptor_async.py
   tests/test_deepagent_session_id.py` — 36 passed; суммарно 55.
3. Ruff: streaming.py + wiring-тест — 0 новых замечаний (8 предсуществующих
   streaming.py остаются).
4. Смоук: pytest --collect-only по wiring-тесту (цепочка импортов цела).

### 4. Коммит

Стейджить только streaming.py и tests/test_deepagent_internet_judge_wiring.py:
`fix(03): досл вердиктов на всех выходах стрима; продвижение step_sequence; step_input вопроса (REQ-08)`

## Критерии выхода

- `_flush_unsent_judge_verdicts()` вызывается на трёх завершающих путях:
  терминальный pptx-return, нормальный выход, аварийный выход 7.7.
- `_tool_step_sequence` продвигается на число досланных вердиктов; финальный
  message-шаг не конфликтует по sequence.
- Досланные шаги несут `step_input=request.message`.
- 55 passed; ruff без новых замечаний.

Полный отчёт: `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.sberpowers/tasks/31-fix2-report.md`
(ГЛАВНОЕ дерево, абсолютный путь). В чат ≤15 строк: статус, коммит SHA+тема,
строка о тестах, сомнения, путь к отчёту.
