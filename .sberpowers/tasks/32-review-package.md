42128e75 fix(04): неопределённые вердикты судьи не засчитываются как совпадения (REQ-02)
227a93be fix(04): прогон корзины не падает на сбоях судьи; строгие метки; понятные отказы CLI (REQ-02)
603e96d8 feat(04): CLI-прогон тестовой корзины mcp_internet-judge (REQ-02)

 .../react/deepagent/internet_judge_check.py        | 119 ++++++++++++++
 tests/test_deepagent_internet_judge_basket.py      | 176 +++++++++++++++++++++
 2 files changed, 295 insertions(+)

diff --git a/app/services/react/deepagent/internet_judge_check.py b/app/services/react/deepagent/internet_judge_check.py
new file mode 100644
index 00000000..eaea4789
--- /dev/null
+++ b/app/services/react/deepagent/internet_judge_check.py
@@ -0,0 +1,119 @@
+"""CLI-прогон тестовой корзины mcp_internet-judge против живого GigaChat.
+
+Использование (корень репо, дев-стенд с доступным GigaChat):
+    PYTHONPATH=. .venv/bin/python -m app.services.react.deepagent.internet_judge_check [test.xlsx]
+
+Печатает accuracy против разметки колонки «Должен ли вызываться mcp_internet?»
+и список несовпадений. Exit code 1 при accuracy < 0.90.
+"""
+
+# ruff: noqa: T201  # CLI-модуль: печать в stdout — контракт инструмента
+
+from __future__ import annotations
+
+import asyncio
+import sys
+from typing import TYPE_CHECKING, Any
+
+from app.services.react.deepagent.internet_judge import InternetJudgeVerdict
+
+if TYPE_CHECKING:
+    from app.services.react.deepagent.internet_judge import InternetJudgeGate
+
+_ACCURACY_TARGET = 0.90
+
+
+def load_basket(xlsx_path: str) -> list[tuple[str, int]]:
+    """Пары (query, expected 0|1) из первого листа: A=query, B=метка."""
+    import openpyxl
+
+    wb = openpyxl.load_workbook(xlsx_path, data_only=True)
+    ws = wb.worksheets[0]
+    items: list[tuple[str, int]] = []
+    header_seen = False
+    for row in ws.iter_rows(values_only=True):
+        query = row[0] if len(row) > 0 else None
+        label = row[1] if len(row) > 1 else None
+        if not header_seen:
+            header_seen = True  # первая строка — заголовки «query» / «Должен ли …»
+            continue
+        if not isinstance(query, str) or not query.strip():
+            continue
+        if isinstance(label, bool):
+            continue
+        if isinstance(label, int):
+            expected = label
+        elif isinstance(label, float) and label.is_integer():
+            expected = int(label)
+        else:
+            continue
+        if expected not in (0, 1):
+            continue
+        items.append((query.strip(), expected))
+    return items
+
+
+async def run_basket(gate: InternetJudgeGate, items: list[tuple[str, int]]) -> dict[str, Any]:
+    """Прогон basket через gate.judge(); verdict==allow сопоставляется метке 1."""
+    mismatches: list[dict[str, Any]] = []
+    match = 0
+    for query, expected in items:
+        try:
+            verdict = await gate.judge(query)
+        except Exception as exc:
+            verdict = InternetJudgeVerdict("error", f"прогон упал: {exc}")
+        actual = 1 if verdict.verdict == "allow" else 0
+        ok = verdict.verdict in ("allow", "deny") and actual == expected
+        match += ok
+        if not ok:
+            mismatches.append(
+                {
+                    "query": query,
+                    "expected": expected,
+                    "actual": actual,
+                    "verdict": verdict.verdict,
+                    "reason": verdict.reason,
+                },
+            )
+        print(f"[{'OK ' if ok else 'MISS'}] expected={expected} actual={actual} :: {query}")
+    total = len(items)
+    return {
+        "total": total,
+        "match": match,
+        "accuracy": match / total if total else 0.0,
+        "mismatches": mismatches,
+    }
+
+
+async def main(argv: list[str] | None = None) -> None:
+    argv = argv if argv is not None else sys.argv[1:]
+    xlsx_path = argv[0] if argv else "test.xlsx"
+    from app.core.config import app_settings
+    from app.services.react.deepagent.internet_judge import InternetJudgeGate
+
+    if not app_settings.MCP_INTERNET_JUDGE_ENABLED:
+        raise SystemExit("MCP_INTERNET_JUDGE_ENABLED=False — судья отключён в config.py, прогон невозможен")
+
+    try:
+        items = load_basket(xlsx_path)
+    except FileNotFoundError:
+        raise SystemExit(f"Корзина не найдена: {xlsx_path}")
+    if not items:
+        raise SystemExit(f"Корзина пуста: {xlsx_path}")
+
+    gate = InternetJudgeGate(chat_id="basket-run", message_id="basket-run", user_question="")
+    report = await run_basket(gate, items)
+    print(
+        f"\n=== Итог: {report['match']}/{report['total']} === "
+        f"accuracy={report['accuracy']:.4f} (цель >= {_ACCURACY_TARGET})",
+    )
+    if report["mismatches"]:
+        print("\nНесовпадения:")
+        for m in report["mismatches"]:
+            print(f"  - [{m['verdict']}] exp={m['expected']} act={m['actual']} :: {m['query']} :: {m['reason']}")
+    if report["accuracy"] < _ACCURACY_TARGET:
+        raise SystemExit(1)
+
+
+if __name__ == "__main__":
+    asyncio.run(main())
diff --git a/tests/test_deepagent_internet_judge_basket.py b/tests/test_deepagent_internet_judge_basket.py
new file mode 100644
index 00000000..50a4e045
--- /dev/null
+++ b/tests/test_deepagent_internet_judge_basket.py
@@ -0,0 +1,176 @@
+"""Прогон корзины: загрузка xlsx + подсчёт accuracy на моке судьи."""
+
+import os
+
+import pytest
+
+from app.core.config import app_settings
+from app.services.react.deepagent.internet_judge import InternetJudgeGate, InternetJudgeVerdict
+from app.services.react.deepagent.internet_judge_check import load_basket, main, run_basket
+
+BASKET = "test.xlsx"  # в корне репо; в изолированном worktree файла нет -> skip
+
+
+def _gate_with_stub(monkeypatch, mapping: dict[str, str]) -> InternetJudgeGate:
+    gate = InternetJudgeGate(chat_id="basket", message_id="basket", user_question="")
+
+    async def fake_llm(messages):
+        q = messages[-1]["content"]
+        for needle, verdict in mapping.items():
+            if needle in q:
+                return f'{{"verdict": "{verdict}", "reason": "stub"}}'
+        return '{"verdict": "deny", "reason": "stub default"}'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    return gate
+
+
+@pytest.mark.asyncio
+async def test_run_basket_counts_accuracy(monkeypatch):
+    gate = _gate_with_stub(monkeypatch, {"Java": "allow"})
+    items = [("найди курс по Java", 1), ("Какая погода в Москве?", 0), ("обучение по Go", 1)]
+    report = await run_basket(gate, items)
+    # Java -> allow (совпало), погода -> deny (совпало), Go -> deny по stub-default (не совпало: ожидали 1)
+    assert report["total"] == 3
+    assert report["match"] == 2
+    assert abs(report["accuracy"] - 2 / 3) < 1e-9
+    assert len(report["mismatches"]) == 1
+    assert report["mismatches"][0]["query"] == "обучение по Go"
+
+
+def test_load_basket_reads_xlsx():
+    if not os.path.exists(BASKET):
+        pytest.skip("test.xlsx отсутствует в корне репо (в worktree untracked-файлы не попадают)")
+    items = load_basket(BASKET)
+    assert len(items) >= 130
+    labels = {label for _, label in items}
+    assert labels <= {0, 1}
+    assert any(label == 1 for _, label in items)
+    assert any(label == 0 for _, label in items)
+
+
+@pytest.mark.asyncio
+async def test_basket_border_pairs_with_stub(monkeypatch):
+    """Граничные пары REQ-02 на стабе судьи (CI-проверка механики прогона)."""
+    mapping = {
+        "антифрод": "allow",
+        "инвестициям": "deny",
+        "Цифровому рублю": "allow",
+        "новости о цифровом рубле": "deny",
+        "микроэкономике": "allow",
+        "новые книги": "deny",
+        "ИНН": "deny",
+        "Яндекса": "allow",
+        "эквайринг": "deny",
+    }
+    gate = _gate_with_stub(monkeypatch, mapping)
+    pairs = [
+        ("статья по антифрод-стратегиям", 1),
+        ("Какие советы по инвестициям дают аналитики?", 0),
+        ("как подключить бизнес к Цифровому рублю", 1),
+        ("Какие новости о цифровом рубле?", 0),
+        ("найди книгу по микроэкономике для продакт-менеджеров", 1),
+        ("Какие новые книги вышли в этом месяце?", 0),
+        ("Герман Греф ИНН 0709306824?", 0),
+        ("курс Яндекса по управлению продуктом", 1),
+        ("Какие тарифы на эквайринг у Сбера и ВТБ?", 0),
+    ]
+    report = await run_basket(gate, pairs)
+    assert report["accuracy"] == 1.0, report["mismatches"]
+
+
+@pytest.mark.asyncio
+async def test_run_basket_survives_judge_crash(monkeypatch):
+    """G1: сбой судьи на одной строке не рушит прогон; строка = mismatch error."""
+    gate = _gate_with_stub(monkeypatch, {"Java": "allow"})
+
+    async def crashing_llm(messages):
+        q = messages[-1]["content"]
+        if "Go" in q:
+            return "этот ответ не JSON"  # _parse_verdict кинет ValueError
+        return '{"verdict": "allow", "reason": "stub"}'
+
+    monkeypatch.setattr(gate, "_call_llm", crashing_llm)
+    items = [("найди курс по Java", 1), ("обучение по Go", 1)]
+    report = await run_basket(gate, items)
+    assert report["total"] == 2
+    assert report["match"] == 1
+    assert abs(report["accuracy"] - 0.5) < 1e-9
+    assert len(report["mismatches"]) == 1
+    assert report["mismatches"][0]["verdict"] == "error"
+    assert "прогон упал" in report["mismatches"][0]["reason"]
+
+
+@pytest.mark.asyncio
+async def test_run_basket_error_on_deny_row_is_miss(monkeypatch):
+    """I2: сбой судьи на deny-строке (expected=0) — miss, не совпадение."""
+    gate = InternetJudgeGate(chat_id="b", message_id="b", user_question="")
+
+    async def broken_llm(messages):
+        return "этот ответ не JSON"  # _parse_verdict кинет ValueError
+
+    monkeypatch.setattr(gate, "_call_llm", broken_llm)
+    items = [("Какая погода в Москве?", 0), ("найди курс по Java", 1)]
+    report = await run_basket(gate, items)
+    assert report["total"] == 2
+    assert report["match"] == 0  # обе строки: deny-метка и allow-метка — miss
+    assert report["accuracy"] == 0.0
+    assert len(report["mismatches"]) == 2
+    assert all(m["verdict"] == "error" for m in report["mismatches"])
+
+
+def test_load_basket_rejects_dirty_labels(tmp_path):
+    """G4: bool/дробные/строковые метки отбраковываются, целые 0/1 принимаются."""
+    import openpyxl
+
+    wb = openpyxl.Workbook()
+    ws = wb.active
+    ws.append(["query", "label"])
+    ws.append(["вопрос один", 1])
+    ws.append(["вопрос два", 0])
+    ws.append(["вопрос три", 1.9])  # дробное — отбраковка
+    ws.append(["вопрос четыре", True])  # bool — отбраковка
+    ws.append(["вопрос пять", "1"])  # строка — отбраковка
+    ws.append(["вопрос шесть", None])  # пустая — отбраковка
+    path = tmp_path / "mini.xlsx"
+    wb.save(path)
+    items = load_basket(str(path))
+    assert items == [("вопрос один", 1), ("вопрос два", 0)]
+
+
+@pytest.mark.asyncio
+async def test_main_disabled_flag_exits_with_message(monkeypatch):
+    """G6: выключенный MCP_INTERNET_JUDGE_ENABLED → SystemExit с понятным текстом."""
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", False)
+    with pytest.raises(SystemExit, match="MCP_INTERNET_JUDGE_ENABLED"):
+        await main([])
+
+
+@pytest.mark.asyncio
+async def test_main_missing_file_exits_with_message(monkeypatch):
+    """G2: отсутствующий xlsx → SystemExit с понятным сообщением, не трейсбек."""
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    with pytest.raises(SystemExit, match="Корзина не найдена"):
+        await main(["__no_such_file__.xlsx"])
+
+
+@pytest.mark.asyncio
+async def test_main_low_accuracy_exit_code_1(monkeypatch, tmp_path):
+    """G6: accuracy ниже цели → SystemExit(1); стаб judge возвращает deny."""
+    import openpyxl
+
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)
+    wb = openpyxl.Workbook()
+    ws = wb.active
+    ws.append(["query", "label"])
+    ws.append(["вопрос один", 1])  # ждём allow, стаб вернёт deny -> accuracy 0.0
+    path = tmp_path / "mini.xlsx"
+    wb.save(path)
+
+    async def deny_judge(self, query: str) -> InternetJudgeVerdict:
+        return InternetJudgeVerdict("deny", "stub")
+
+    monkeypatch.setattr(InternetJudgeGate, "judge", deny_judge)
+    with pytest.raises(SystemExit) as exc_info:
+        await main([str(path)])
+    assert exc_info.value.code == 1
