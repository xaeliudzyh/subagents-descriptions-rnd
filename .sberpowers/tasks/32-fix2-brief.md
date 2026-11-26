# Бриф задачи 3.2-FIX2: сбой судьи не должен засчитываться как совпадение

- Основано на повторном two-stage ревью b9d804a9..227a93be: ось A PASS,
  ось B NEEDS_FIXES — все прежние находки G1–G6 CLOSED, отступления N1–N4
  подтверждены; ОДНА новая Important.
- Ты работаешь в worktree `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.worktrees/wave3-task32`,
  ветка `wave3-task32`, HEAD `227a93be`. Третий коммит поверх.

## Глобальные ограничения (без изменений)

- venv общий: `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.venv/bin/python -m pytest <путь> -v`;
  ruff тем же venv; line length 120; type hints.
- Живой GigaChat не трогать; test.xlsx в worktree нет (SKIP load_basket ожидаем).
- Один коммит: `fix(04): неопределённые вердикты судьи не засчитываются как совпадения (REQ-02)`.
- Стейджить только два файла задачи.

## Находка повторного ревью (I2, Important) и решение диспетчера

`app/services/react/deepagent/internet_judge_check.py`, функция `run_basket`:
для вердикта `error`/`timeout` (псевдо-вердикт из try/except либо вердикт самого
судьи) на строке корзины с меткой `expected=0` получается `actual=0 == expected`
→ строка засчитывается как совпадение `[OK ]`, reason «прогон упал: …» не
попадает в mismatches, match растёт.

Сценарий отказа: систематический сбой судьи (сеть/невалидный JSON от GigaChat)
завышает accuracy ровно на долю deny-строк корзины (85/136 ≈ 62%); при почти
полностью мёртвом судье accuracy может пройти порог 0.90 с exit code 0, и
оператор не увидит ни одного сбоя на deny-строках. Приёмка REQ-02 теряет смысл.

**Решение диспетчера:** неопределённый вердикт (`error`/`timeout`) — всегда miss,
независимо от метки строки; знаменатель (total) не меняется:

```python
        ok = verdict.verdict in ("allow", "deny") and actual == expected
```

Строка при неопределённом вердикте печатается `[MISS]` и попадает в mismatches
(там уже есть verdict/reason) — оператор видит каждый сбой.

## Реализация

### 1. RED: новый тест в tests/test_deepagent_internet_judge_basket.py

```python
@pytest.mark.asyncio
async def test_run_basket_error_on_deny_row_is_miss(monkeypatch):
    """I2: сбой судьи на deny-строке (expected=0) — miss, не совпадение."""
    gate = InternetJudgeGate(chat_id="b", message_id="b", user_question="")

    async def broken_llm(messages):
        return "этот ответ не JSON"  # _parse_verdict кинет ValueError

    monkeypatch.setattr(gate, "_call_llm", broken_llm)
    items = [("Какая погода в Москве?", 0), ("найди курс по Java", 1)]
    report = await run_basket(gate, items)
    assert report["total"] == 2
    assert report["match"] == 0          # обе строки: deny-метка и allow-метка — miss
    assert report["accuracy"] == 0.0
    assert len(report["mismatches"]) == 2
    assert all(m["verdict"] == "error" for m in report["mismatches"])
```

До правки этот тест падает: первая строка (expected=0, error→actual=0)
засчитывается OK → match==1. Запусти — зафиксируй RED.

### 2. Правка run_basket (internet_judge_check.py)

После вычисления `actual = 1 if verdict.verdict == "allow" else 0` замени
строку `ok = actual == expected` (или эквивалент в текущем коде — сверись)
на:

```python
        ok = verdict.verdict in ("allow", "deny") and actual == expected
```

Больше ничего в файле не менять.

### 3. GREEN и регресс

1. `tests/test_deepagent_internet_judge_basket.py` — прежние 7 passed + новый PASS,
   1 SKIP (load_basket — ожидаемо).
2. Регресс судьи: `tests/test_deepagent_internet_judge.py` — 13 passed.
3. Совместный прогон — без падений.
4. Ruff по двум файлам — 0 замечаний.

### 4. Коммит

Стейджить только два файла:
`fix(04): неопределённые вердикты судьи не засчитываются как совпадения (REQ-02)`

## Критерии выхода

- Сбой/таймаут судьи на любой строке (0 или 1) — видимый miss в mismatches,
  accuracy честно падает, exit code отражает порог.
- Новый тест зелёный; регресс 13 passed; ruff чист.

Полный отчёт: `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.sberpowers/tasks/32-fix2-report.md`
(ГЛАВНОЕ дерево, абсолютный путь). В чат ≤15 строк: статус, коммит SHA+тема,
строка о тестах, сомнения, путь к отчёту.
