# Бриф задачи 3.2-FIX: устойчивый прогон корзины к сбоям судьи

- Основано на two-stage ревью коммита 603e96d8: ось A → PASS (механика
  REQ-02 SATISFIED; живое суждение LLM — открытый шаг на дев-стенде);
  ось B → NEEDS_FIXES (1 Important + 6 Minor).
- Ты работаешь в worktree `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.worktrees/wave3-task32`,
  ветка `wave3-task32`, текущий HEAD `603e96d8`. Второй коммит поверх него.

## Глобальные ограничения (как в основном брифе)

- venv ОБЩИЙ на машину: тесты
  `/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.venv/bin/python -m pytest <путь> -v`;
  ruff тем же venv, line length 120, type hints обязательны.
- Живой прогон против GigaChat НЕ делать.
- test.xlsx в worktree нет — кейс load_basket остаётся SKIP (не «чинить»).
- Один коммит: `fix(04): прогон корзины не падает на сбоях судьи; строгие метки; понятные отказы CLI (REQ-02)`.
- Стейджить только два файла задачи.

## Находки ревью и решения диспетчера

### G1 (IMPORTANT): run_basket падает целиком от одного сбоя судьи

`await gate.judge(query)` не защищён: невалидный ответ GigaChat
(`ValueError` из `_parse_verdict`) или сетевой сбой `_call_llm` рушат
live-прогон на 136 строках посреди корзины — accuracy не печатается, приёмка
REQ-02 обнуляется. Прод-судья fail-closed, а CLI-обёртка — нет.

**Решение диспетчера:** try/except вокруг judge → псевдо-вердикт
`InternetJudgeVerdict("error", f"прогон упал: {exc}")`, строка учитывается как
mismatch (actual=0, т.к. error ≠ allow), прогон продолжается. CLI-инструмент
не agent-facing: детали exc в reason допустимы (отличие от прод-контракта
судьи — там reason чищен от exc).

### G2–G6 (Minor, дешёвые — сделать)

- **G2:** отсутствующий xlsx — `openpyxl.load_workbook` кидает `FileNotFoundError`
  раньше ветки «Корзина пуста или не найдена» → трейсбек вместо понятного
  сообщения. Обернуть load_basket в main в try/except FileNotFoundError →
  SystemExit с сообщением.
- **G3:** мёртвый `logger` (создан, не используется) — удалить вместе с
  импортом logging.
- **G4:** `int(label)` молча усекает `1.9 → 1` и принимает `True → 1` —
  мусорные метки попадают в корзину. Строгая отбраковка: bool — отброс;
  int — принять; float — принять только при целом значении; всё остальное — отброс.
- **G5:** `gate: Any` на ключевом аргументе run_basket — аннотировать
  `"InternetJudgeGate"` через `if TYPE_CHECKING:` (без рантайм-импорта).
- **G6:** контракт `main()` (SystemExit при выключенном флаге / exit code 1
  при accuracy < 0.90) покрыть только смоуком — добавить два дешёвых теста
  (SystemExit при выключенном флаге; SystemExit(1) при accuracy ниже цели
  через стаб judge).

### Отклонено диспетчером (НЕ делать)

- **Параллелизация asyncio.gather + Semaphore** для 136 последовательных
  LLM-вызовов: разовый дев-инструмент, последовательный прогон даёт
  детерминированный читаемый вывод и минимальный код; ревьюер сам признал
  «для разового дев-инструмента приемлемо». Зафиксировано в ledger.

## Реализация

### 1. RED: новые/изменённые тесты в tests/test_deepagent_internet_judge_basket.py

Расширь импорты: добавь `InternetJudgeVerdict` к импорту internet_judge,
`from app.core.config import app_settings`, `from app.services.react.deepagent.internet_judge_check import main`.

```python
@pytest.mark.asyncio
async def test_run_basket_survives_judge_crash(monkeypatch):
    """G1: сбой судьи на одной строке не рушит прогон; строка = mismatch error."""
    gate = _gate_with_stub(monkeypatch, {"Java": "allow"})

    async def crashing_llm(messages):
        q = messages[-1]["content"]
        if "Go" in q:
            return "этот ответ не JSON"  # _parse_verdict кинет ValueError
        return '{"verdict": "allow", "reason": "stub"}'

    monkeypatch.setattr(gate, "_call_llm", crashing_llm)
    items = [("найди курс по Java", 1), ("обучение по Go", 1)]
    report = await run_basket(gate, items)
    assert report["total"] == 2
    assert report["match"] == 1
    assert abs(report["accuracy"] - 0.5) < 1e-9
    assert len(report["mismatches"]) == 1
    assert report["mismatches"][0]["verdict"] == "error"
    assert "прогон упал" in report["mismatches"][0]["reason"]


def test_load_basket_rejects_dirty_labels(tmp_path):
    """G4: bool/дробные/строковые метки отбраковываются, целые 0/1 принимаются."""
    import openpyxl

    wb = openpyxl.Workbook()
    ws = wb.active
    ws.append(["query", "label"])
    ws.append(["вопрос один", 1])
    ws.append(["вопрос два", 0])
    ws.append(["вопрос три", 1.9])     # дробное — отбраковка
    ws.append(["вопрос четыре", True])  # bool — отбраковка
    ws.append(["вопрос пять", "1"])     # строка — отбраковка
    ws.append(["вопрос шесть", None])   # пустая — отбраковка
    path = tmp_path / "mini.xlsx"
    wb.save(path)
    items = load_basket(str(path))
    assert items == [("вопрос один", 1), ("вопрос два", 0)]


@pytest.mark.asyncio
async def test_main_disabled_flag_exits_with_message(monkeypatch):
    """G6: выключенный MCP_INTERNET_JUDGE_ENABLED → SystemExit с понятным текстом."""
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
    with pytest.raises(SystemExit, match="MCP_INTERNET_JUDGE_ENABLED"):
        await main([])


@pytest.mark.asyncio
async def test_main_missing_file_exits_with_message(monkeypatch):
    """G2: отсутствующий xlsx → SystemExit с понятным сообщением, не трейсбек."""
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
    with pytest.raises(SystemExit, match="Корзина не найдена"):
        await main(["__no_such_file__.xlsx"])
```

Запусти файл: G1-тест упадёт (ValueError наружу), G2-тест упадёт
(FileNotFoundError/SystemExit с другим текстом), G4/G6 — по факту
(RED минимум по G1 и G2 обязателен).

### 2. internet_judge_check.py — правки

Шапка: удали `import logging` и строку `logger = logging.getLogger(__name__)`
(G3); добавь TYPE_CHECKING-блок:

```python
from typing import TYPE_CHECKING, Any

if TYPE_CHECKING:
    from app.services.react.deepagent.internet_judge import InternetJudgeGate
```

Сигнатура `run_basket` (G5):
`async def run_basket(gate: "InternetJudgeGate", items: list[tuple[str, int]]) -> dict[str, Any]:`

Тело цикла `run_basket` (G1) — оберни judge:

```python
    for query, expected in items:
        try:
            verdict = await gate.judge(query)
        except Exception as exc:
            verdict = InternetJudgeVerdict("error", f"прогон упал: {exc}")
        actual = 1 if verdict.verdict == "allow" else 0
```

(строки ниже — unchanged: ok/mismatch/печать; импортируй InternetJudgeVerdict
в шапке вместе с InternetJudgeGate).

Отбраковка меток в `load_basket` (G4) — замени try/int-блок:

```python
        if isinstance(label, bool):
            continue
        if isinstance(label, int):
            expected = label
        elif isinstance(label, float) and label.is_integer():
            expected = int(label)
        else:
            continue
        if expected not in (0, 1):
            continue
```

`main()` (G2) — оберни загрузку:

```python
    try:
        items = load_basket(xlsx_path)
    except FileNotFoundError:
        raise SystemExit(f"Корзина не найдена: {xlsx_path}")
    if not items:
        raise SystemExit(f"Корзина пуста: {xlsx_path}")
```

### 3. GREEN и регресс

1. `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v`
   — 2 прежних PASS + новые PASS + 1 SKIP (load_basket от test.xlsx, ожидаемо).
2. Регресс судьи: `tests/test_deepagent_internet_judge.py` — 13 passed.
3. Совместно: 1+2 в одном прогоне — без падений.
4. Ruff по двум файлам — 0 замечаний (`# ruff: noqa: T201` сохраняется).

## Коммит

Стейджить только два файла:
`fix(04): прогон корзины не падает на сбоях судьи; строгие метки; понятные отказы CLI (REQ-02)`

## Критерии выхода

- Один сбой судьи не рушит корзину: строка фиксируется как error-mismatch,
  accuracy печатается, exit code отражает цель (G1).
- Мусорные метки (bool/дробные/строки) не попадают в корзину (G4).
- CLI-отказы понятны: выключенный флаг, отсутствующий/пустой файл (G2/G6).
- Регресс зелёная; ruff чист.

Полный отчёт запиши в
`/home/work/23686942@sigma.sbrf.ru/Dev/agent-analytics/.sberpowers/tasks/32-fix-report.md`
(ГЛАВНОЕ дерево, абсолютный путь): команды и результаты, RED/GREEN, self-review,
сомнения. В чат ≤15 строк: статус, коммит SHA+заголовок, строка о тестах,
сомнения, путь к отчёту.
