# Бриф задачи 3.2: CLI-прогон тестовой корзины (волна 3)

- REQ: REQ-mcp-internet-judge-02 (приёмка: скриптовый прогон корзины печатает accuracy)
- Blocked by: задача 2.1 (выполнена; после фикса A судья получает bare-текст —
  формат корзины совпадает с прод-форматом входа судьи)
- Ты работаешь в изолированном worktree от ветки mcp-internet-judge (HEAD b9d804a9)

## Глобальные ограничения плана (дословно)

- deepagents==0.7.14 — версию не трогаем.
- Python 3.13, venv `.venv`; тесты: `.venv/bin/python -m pytest <путь> -v`.
- Ruff: line length 120, type hints обязательны.
- Один коммит: `feat(04): CLI-прогон тестовой корзины mcp_internet-judge (REQ-02)`.
- Стейджить только файлы задачи (два новых файла).

## Разрешённые диспетчером неоднозначности

- **test.xlsx — untracked-файл, в твоём worktree его НЕТ** (worktree содержит
  только закоммиченное). Кейс `test_load_basket_reads_xlsx` в worktree даст
  SKIP через pytest.skip — это ОЖИДАЕМО и не ошибка; диспетчер после merge
  прогонит его в главном дереве, где файл есть. Не «чинить» SKIP, не копировать
  test.xlsx в worktree, не коммитить его.
- Ruff-канонизация импортов/аннотаций допустима, логика — дословно.
- Живой прогон CLI против GigaChat НЕ делать (судья ходил бы в сеть);
  в CI корзина гоняется только с мокнутым `_call_llm` на граничных парах.
- Отклонение от плана, утверждённое фиксом A (b9d804a9): `run_basket` вызывает
  `gate.judge(query)` с bare-строкой корзины — после фикса A это ровно тот
  формат, который судья видит в проде для строковых args. Корзина честная.

## Контекст задачи

Приёмка REQ-02 требует наблюдаемый прогон корзины `test.xlsx`
(лист Sheet1: колонки A=query, B=«Должен ли вызываться mcp_internet?»;
136 валидных строк: 51 allow / 85 deny) с печатью accuracy ≥ 0.90 и списком
несовпадений. Модуль судьи уже есть: `app/services/react/deepagent/internet_judge.py`
(`InternetJudgeGate(chat_id, message_id, user_question)`, async `judge(query) -> InternetJudgeVerdict`,
`InternetJudgeVerdict.verdict ∈ {"allow","deny","timeout","error"}`).

## Файлы

- Создать: `app/services/react/deepagent/internet_judge_check.py`
- Создать: `tests/test_deepagent_internet_judge_basket.py`

## Шаги

1. Напиши падающий тест `tests/test_deepagent_internet_judge_basket.py`:

```python
"""Прогон корзины: загрузка xlsx + подсчёт accuracy на моке судьи."""

import os

import pytest

from app.services.react.deepagent.internet_judge import InternetJudgeGate
from app.services.react.deepagent.internet_judge_check import load_basket, run_basket

BASKET = "test.xlsx"  # в корне репо; в изолированном worktree файла нет -> skip


def _gate_with_stub(monkeypatch, mapping: dict[str, str]) -> InternetJudgeGate:
    gate = InternetJudgeGate(chat_id="basket", message_id="basket", user_question="")

    async def fake_llm(messages):
        q = messages[-1]["content"]
        for needle, verdict in mapping.items():
            if needle in q:
                return f'{{"verdict": "{verdict}", "reason": "stub"}}'
        return '{"verdict": "deny", "reason": "stub default"}'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    return gate


@pytest.mark.asyncio
async def test_run_basket_counts_accuracy(monkeypatch):
    gate = _gate_with_stub(monkeypatch, {"Java": "allow"})
    items = [("найди курс по Java", 1), ("Какая погода в Москве?", 0), ("обучение по Go", 1)]
    report = await run_basket(gate, items)
    # Java -> allow (совпало), погода -> deny (совпало), Go -> deny по stub-default (не совпало: ожидали 1)
    assert report["total"] == 3
    assert report["match"] == 2
    assert abs(report["accuracy"] - 2 / 3) < 1e-9
    assert len(report["mismatches"]) == 1
    assert report["mismatches"][0]["query"] == "обучение по Go"


def test_load_basket_reads_xlsx():
    if not os.path.exists(BASKET):
        pytest.skip("test.xlsx отсутствует в корне репо (в worktree untracked-файлы не попадают)")
    items = load_basket(BASKET)
    assert len(items) >= 130
    labels = {label for _, label in items}
    assert labels <= {0, 1}
    assert any(label == 1 for _, label in items)
    assert any(label == 0 for _, label in items)


@pytest.mark.asyncio
async def test_basket_border_pairs_with_stub(monkeypatch):
    """Граничные пары REQ-02 на стабе судьи (CI-проверка механики прогона)."""
    mapping = {
        "антифрод": "allow",
        "инвестициям": "deny",
        "Цифровому рублю": "allow",
        "новости о цифровом рубле": "deny",
        "микроэкономике": "allow",
        "новые книги": "deny",
        "ИНН": "deny",
        "Яндекса": "allow",
        "эквайринг": "deny",
    }
    gate = _gate_with_stub(monkeypatch, mapping)
    pairs = [
        ("статья по антифрод-стратегиям", 1),
        ("Какие советы по инвестициям дают аналитики?", 0),
        ("как подключить бизнес к Цифровому рублю", 1),
        ("Какие новости о цифровом рубле?", 0),
        ("найди книгу по микроэкономике для продакт-менеджеров", 1),
        ("Какие новые книги вышли в этом месяце?", 0),
        ("Герман Греф ИНН 0709306824?", 0),
        ("курс Яндекса по управлению продуктом", 1),
        ("Какие тарифы на эквайринг у Сбера и ВТБ?", 0),
    ]
    report = await run_basket(gate, pairs)
    assert report["accuracy"] == 1.0, report["mismatches"]
```

2. Запусти `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v`
   — ожидай FAIL: нет модуля `internet_judge_check` (RED).

3. Реализуй `app/services/react/deepagent/internet_judge_check.py`:

```python
"""CLI-прогон тестовой корзины mcp_internet-judge против живого GigaChat.

Использование (корень репо, дев-стенд с доступным GigaChat):
    PYTHONPATH=. .venv/bin/python -m app.services.react.deepagent.internet_judge_check [test.xlsx]

Печатает accuracy против разметки колонки «Должен ли вызываться mcp_internet?»
и список несовпадений. Exit code 1 при accuracy < 0.90.
"""

from __future__ import annotations

import asyncio
import logging
import sys
from typing import Any

logger = logging.getLogger(__name__)

_ACCURACY_TARGET = 0.90


def load_basket(xlsx_path: str) -> list[tuple[str, int]]:
    """Пары (query, expected 0|1) из первого листа: A=query, B=метка."""
    import openpyxl

    wb = openpyxl.load_workbook(xlsx_path, data_only=True)
    ws = wb.worksheets[0]
    items: list[tuple[str, int]] = []
    header_seen = False
    for row in ws.iter_rows(values_only=True):
        query = row[0] if len(row) > 0 else None
        label = row[1] if len(row) > 1 else None
        if not header_seen:
            header_seen = True  # первая строка — заголовки «query» / «Должен ли …»
            continue
        if not isinstance(query, str) or not query.strip():
            continue
        try:
            expected = int(label)
        except (TypeError, ValueError):
            continue
        if expected not in (0, 1):
            continue
        items.append((query.strip(), expected))
    return items


async def run_basket(gate: Any, items: list[tuple[str, int]]) -> dict[str, Any]:
    """Прогон basket через gate.judge(); verdict==allow сопоставляется метке 1."""
    mismatches: list[dict[str, Any]] = []
    match = 0
    for query, expected in items:
        verdict = await gate.judge(query)
        actual = 1 if verdict.verdict == "allow" else 0
        ok = actual == expected
        match += ok
        if not ok:
            mismatches.append(
                {
                    "query": query,
                    "expected": expected,
                    "actual": actual,
                    "verdict": verdict.verdict,
                    "reason": verdict.reason,
                }
            )
        print(f"[{'OK ' if ok else 'MISS'}] expected={expected} actual={actual} :: {query}")
    total = len(items)
    return {
        "total": total,
        "match": match,
        "accuracy": match / total if total else 0.0,
        "mismatches": mismatches,
    }


async def main(argv: list[str] | None = None) -> None:
    argv = argv if argv is not None else sys.argv[1:]
    xlsx_path = argv[0] if argv else "test.xlsx"
    from app.core.config import app_settings
    from app.services.react.deepagent.internet_judge import InternetJudgeGate

    if not app_settings.MCP_INTERNET_JUDGE_ENABLED:
        raise SystemExit("MCP_INTERNET_JUDGE_ENABLED=False — судья отключён в config.py, прогон невозможен")

    items = load_basket(xlsx_path)
    if not items:
        raise SystemExit(f"Корзина пуста или не найдена: {xlsx_path}")

    gate = InternetJudgeGate(chat_id="basket-run", message_id="basket-run", user_question="")
    report = await run_basket(gate, items)
    print(
        f"\n=== Итог: {report['match']}/{report['total']} === "
        f"accuracy={report['accuracy']:.4f} (цель >= {_ACCURACY_TARGET})"
    )
    if report["mismatches"]:
        print("\nНесовпадения:")
        for m in report["mismatches"]:
            print(f"  - [{m['verdict']}] exp={m['expected']} act={m['actual']} :: {m['query']} :: {m['reason']}")
    if report["accuracy"] < _ACCURACY_TARGET:
        raise SystemExit(1)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    asyncio.run(main())
```

4. GREEN: `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v`
   — ожидай: 2 PASS + 1 SKIP (load_basket — SKIP в worktree, test.xlsx untracked).
   Полный PASS этого кейса диспетчер проверит в главном дереве после merge.

5. Регресс: `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v` — 13 passed.

6. Смоук CLI без сети (валидация argparse/exit-кода отключённого судьи допустима
   только если не требует GigaChat; основной smoke: просто
   `.venv/bin/python -c "import app.services.react.deepagent.internet_judge_check"`).

7. `.venv/bin/ruff check app/services/react/deepagent/internet_judge_check.py tests/test_deepagent_internet_judge_basket.py` — 0 замечаний.

8. Commit (только два файла): `feat(04): CLI-прогон тестовой корзины mcp_internet-judge (REQ-02)`

## Критерии выхода

- Тесты корзины зелёные в рамках worktree (2 PASS + ожидаемый SKIP load_basket);
  механика load/run/граничные пары (включая адресную пару «курс Яндекса» и
  бренд-deny «тарифы на эквайринг») покрыты стабом судьи.
- CLI запускается по документированной команде; формат вывода — accuracy +
  mismatches; exit code 1 при accuracy < 0.90; при выключенном флаге —
  SystemExit с понятным сообщением.
