# Отчёт задачи 3.2-FIX: устойчивый прогон корзины к сбоям судьи

**Статус: DONE_WITH_CONCERNS**
(4 минимальных отступления от буквы сниппетов брифа, каждое продиктовано его же
критериями/находками — см. «Отступления»; семантика всех REQ сохранена)

**Коммит:** `227a93be` — `fix(04): прогон корзины не падает на сбоях судьи; строгие метки; понятные отказы CLI (REQ-02)` (ветка `wave3-task32`, поверх `603e96d8`)
**Файлы:** 2 (`app/services/react/deepagent/internet_judge_check.py`, `tests/test_deepagent_internet_judge_basket.py`), застейджены только два файла задачи, дерево после коммита чистое.
**Основа:** two-stage ревью коммита `603e96d8`: ось A PASS (REQ-02 SATISFIED), ось B NEEDS_FIXES — G1 Important + G2–G6 Minor (5 делать, параллелизация отклонена диспетчером — НЕ сделана).

## Что реализовано

| # | Находка | Правка |
|---|---------|--------|
| G1 | `run_basket` падал целиком от исключения `gate.judge` (невалидный JSON из `_parse_verdict` -> ValueError наружу) | В `run_basket` try/except вокруг `await gate.judge(query)` -> псевдо-вердикт `InternetJudgeVerdict("error", f"прогон упал: {exc}")`; actual=0 (error != allow) -> строка = mismatch, прогон продолжается, accuracy печатается. Детали exc в reason допустимы: CLI-инструмент, не agent-facing (отличие от прод-контракта судьи) |
| G2 | Отсутствующий xlsx -> трейсбек FileNotFoundError из openpyxl раньше ветки про пустую корзину | В `main()` загрузка обёрнута: `try: items = load_basket(...) except FileNotFoundError: raise SystemExit(f"Корзина не найдена: {xlsx_path}")`; пустая корзина -> `SystemExit(f"Корзина пуста: {xlsx_path}")` |
| G3 | Мёртвый `logger` (создан, не использовался) | Удалены `import logging`, `logger = logging.getLogger(__name__)` и `logging.basicConfig(level=logging.INFO)` из `__main__` (см. отступление N2) |
| G4 | `int(label)` молча усекал `1.9->1`, принимал `True->1`, `"1"->1` | Строгая отбраковка: bool -> отброс; int -> принять; float -> только при `is_integer()`; всё остальное -> отброс; затем контроль `expected in (0, 1)` |
| G5 | `gate: Any` на ключевом аргументе | Шапка: `if TYPE_CHECKING: from ...internet_judge import InternetJudgeGate` (без рантайм-импорта); сигнатура `run_basket(gate: InternetJudgeGate, items: list[tuple[str, int]]) -> dict[str, Any]` (аннотация некавыченная — см. отступление N1). Рантайм-импорт `InternetJudgeVerdict` добавлен в шапку — он нужен самому `run_basket` для псевдо-вердикта (в TYPE_CHECKING его класть нельзя) |
| G6 | Контракт `main()` покрыт только смоуком | Два теста: SystemExit при выключенном флаге + SystemExit(1) при accuracy ниже цели через стаб judge (см. отступление N3) |

Прод-контракт судьи (`internet_judge.py`) не тронут — правки только в CLI-обёртке.

## Отступления от буквы сниппетов брифа (все вынужденные)

1. **G5, аннотация без кавычек.** Бриф показывал `gate: "InternetJudgeGate"`,
   но модуль уже содержит `from __future__ import annotations`, а ruff-конфиг
   проекта включает правило UP037 (удаление кавычек в аннотациях при future-
   annotations) — кавыченный вариант дал бы lint-замечание и противоречил бы
   критерию «Ruff по двум файлам — 0 замечаний». Некавыченная аннотация при
   future-import семантически идентична (не вычисляется в рантайме,
   резолвится статикой через TYPE_CHECKING).
2. **G3, дополнительно удалена `logging.basicConfig(level=logging.INFO)` из
   `__main__`.** Бриф предписывал удалить `import logging` и `logger = ...`;
   оставленная строка `logging.basicConfig(...)` после удаления импорта
   роняла бы CLI-вход (`__main__`) с NameError. Логгер в модуле не
   использовался нигде — вызов basicConfig был так же мёртв.
3. **G6, добавлен второй тест `test_main_low_accuracy_exit_code_1`.** Находка
   G6 явно требует «два дешёвых теста» (SystemExit при выключенном флаге;
   SystemExit(1) при accuracy ниже цели через стаб judge), но в сниппете
   раздела «Реализация» второго теста не было. Добавлен по букве находки:
   мини-xlsx через tmp_path, class-level стаб `InternetJudgeGate.judge` ->
   `InternetJudgeVerdict("deny", ...)`, accuracy 0.0 < 0.90 -> SystemExit,
   проверен `exc_info.value.code == 1`. Побочный эффект: этот тест даёт
   легитимное использование импорту `InternetJudgeVerdict`, предписанному
   брифом для тестов (иначе — F401 unused import, снова конфликт с критерием
   ruff-чистоты).
4. **G4-тест: ruff format нормализовал пробелы перед инлайн-комментариями**
   сниппета («`1.9])     # дробное`» -> «`1.9])  # дробное`»). Тексты
   комментариев и содержимое теста не изменились; оба файла на HEAD были
   format-clean, сохранение этого состояния — часть критерия чистоты.

## Команды и результаты

### Базовая линия до правок (sanity)

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py tests/test_deepagent_internet_judge.py -v
-> 15 passed, 1 skipped (skip — test.xlsx отсутствует в worktree, ожидаемо по брифу)
```

### TDD RED (новые тесты, реализация ещё не сделана)

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v
-> 3 failed, 4 passed, 1 skipped
```

Падения — ровно по ожидаемым причинам из брифа:

- `test_run_basket_survives_judge_crash` FAIL — `json.decoder.JSONDecodeError:
  Expecting value: line 1 column 1 (char 0)` — исключение `_parse_verdict`
  прошло сквозь `run_basket` наружу (G1);
- `test_load_basket_rejects_dirty_labels` FAIL — корзина приняла мусорные
  метки: assert [...5 айтемов...] == [2], первый лишний `('вопрос три', 1)`
  — т.е. 1.9 усечён до 1 (G4);
- `test_main_missing_file_exits_with_message` FAIL — `FileNotFoundError:
  [Errno 2] No such file or directory: '__no_such_file__.xlsx'` вместо
  SystemExit (G2).

«По факту» без RED (контракт уже был на месте, бриф это допускал):
`test_main_disabled_flag_exits_with_message` PASS, `test_main_low_accuracy_exit_code_1` PASS.

### TDD GREEN

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v
-> 7 passed, 1 skipped (2 прежних + 5 новых; skip test.xlsx — ожидаемо)
```

### Регресс судьи и совместный прогон

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py
-> 13 passed

.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py \
    tests/test_deepagent_internet_judge_basket.py -v
-> 20 passed, 1 skipped (после финального ruff format — повторно, без изменений)
```

### Ruff

```
.venv/bin/python -m ruff check app/services/react/deepagent/internet_judge_check.py \
    tests/test_deepagent_internet_judge_basket.py
-> All checks passed!

.venv/bin/python -m ruff format --check <те же 2 файла>
-> 2 files already formatted
```

`# ruff: noqa: T201` в шапке `internet_judge_check.py` сохранён (печать в stdout — контракт инструмента).

### Коммит

```
git add app/services/react/deepagent/internet_judge_check.py tests/test_deepagent_internet_judge_basket.py
git commit -m "fix(04): прогон корзины не падает на сбоях судьи; строгие метки; понятные отказы CLI (REQ-02)"
-> [wave3-task32 227a93be] 2 files changed, 103 insertions(+), 12 deletions(-); рабочее дерево чистое
```

## Self-review по критериям выхода брифа

| Критерий | Статус | Свидетельство |
|----------|--------|---------------|
| Один сбой судьи не рушит корзину: строка = error-mismatch, accuracy печатается, exit code отражает цель (G1) | OK | `test_run_basket_survives_judge_crash`: total=2, match=1, accuracy=0.5, mismatch.verdict="error", reason содержит «прогон упал»; main-контракт exit code покрыт G6-тестами |
| Мусорные метки (bool/дробные/строки) не попадают в корзину (G4) | OK | `test_load_basket_rejects_dirty_labels`: из 6 строк остаются ровно 2 валидные; порядок проверок корректный (bool проверяется до int, т.к. bool — подкласс int) |
| CLI-отказы понятны: выключенный флаг, отсутствующий/пустой файл (G2/G6) | OK | Тесты: флаг-off -> SystemExit(«MCP_INTERNET_JUDGE_ENABLED...»), отсутствующий файл -> SystemExit(«Корзина не найдена...»); пустая корзина -> SystemExit(«Корзина пуста...») — изменено по брифу, тестом не покрывалась (см. «Сомнения») |
| Регресс зелёная; ruff чист | OK | 20 passed + 1 expected skip; ruff check — All checks passed; format-check — оба файла formatted |
| Параллелизация НЕ добавлена (отклонено диспетчером) | OK | Прогон остался последовательным циклом, asyncio.gather/Semaphore не внесены |
| Стейджинг только двух файлов задачи | OK | коммит фиксирует ровно 2 файла |
| Живой GigaChat не тронут; test.xlsx-SKIP не «чинился» | OK | Все LLM-точки мокнуты; skip сохранён как ожидаемый |

## Сомнения

- Ветка «Корзина пуста» (существующий файл с 0 валидных строк) изменена по
  брифу (сообщение «Корзина пуста: ...»), но отдельного теста нет — бриф его не
  запрашивал (G2-тест покрывает отсутствующий файл, G6-находка перечисляет
  только флаг и exit code 1). Механика та же, что и у покрытой ветки отсутствия.
- G6-смоуки прошли и ДО фикса (RED не заявлялся — бриф: «G4/G6 — по факту»):
  контракт SystemExit при выключенном флаге и SystemExit(1) ниже цели уже был
  реализован в исходном коммите, тесты фиксируют его от регрессии.
- Решение по интерпретации «импортируй InternetJudgeVerdict в шапке вместе с
  InternetJudgeGate»: рантайм-импорт вердикта размещён в шапке модуля (run_basket
  конструирует его на каждом сбое судьи — TYPE_CHECKING-блок тут не работал бы);
  Gate остался в TYPE_CHECKING по букве G5 «без рантайм-импорта», его ленивый
  импорт внутри main() сохранён. Круговых импортов нет (internet_judge не
  импортирует _check; тяжёлый proxy_client тянутся только внутри _call_llm).
