# Отчёт задачи 3.2-FIX2: сбой судьи не засчитывается как совпадение

- Worktree: `.worktrees/wave3-task32`, ветка `wave3-task32`.
- База: `227a93be` (второй fix). Коммит фикса: **`42128e75`** —
  `fix(04): неопределённые вердикты судьи не засчитываются как совпадения (REQ-02)`.
- Состав коммита: ровно два файла задачи, 19 insertions / 1 deletion.
  Дерево после коммита чистое.

## Что изменилось

`app/services/react/deepagent/internet_judge_check.py`, `run_basket` (единственная правка кода):

```python
        actual = 1 if verdict.verdict == "allow" else 0
        ok = verdict.verdict in ("allow", "deny") and actual == expected
```

Неопределённый вердикт (`error` из try/except либо `timeout` от самого судьи) — всегда miss
независимо от метки строки; знаменатель (total) не меняется. Строка печатается `[MISS]`
и попадает в mismatches с verdict/reason — существующая механика, доработок не потребовалось.

`tests/test_deepagent_internet_judge_basket.py`: новый тест
`test_run_basket_error_on_deny_row_is_miss` (+18 строк) — текст из брифа, размещён
сразу после `test_run_basket_survives_judge_crash`.

## TDD

### RED (до правки)

```
.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py::test_run_basket_error_on_deny_row_is_miss -v
FAILED tests/test_deepagent_internet_judge_basket.py::test_run_basket_error_on_deny_row_is_miss - assert 1 == 0
1 failed, 87 warnings in 4.88s
```

Падение ровно то, что предсказал бриф: строка `("Какая погода в Москве?", 0)` при verdict=error
давала actual=0==expected → засчиталась OK → match==1.

### GREEN и регресс (после правки)

| Прогон | Команда | Результат |
|---|---|---|
| Корзина (полный файл) | `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v` | **8 passed, 1 skipped** (load_basket — test.xlsx нет в worktree, ожидаемо) |
| Регресс судьи | `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v` | **13 passed** |
| Совместный прогон | `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py tests/test_deepagent_internet_judge_basket.py` | **21 passed, 1 skipped** |
| Ruff | `.venv/bin/ruff check <2 файла>` | **All checks passed!** (0 замечаний) |
| Ruff format | `.venv/bin/ruff format --check <2 файла>` | **2 files already formatted** |

## Отступления от буквы брифа

Одно, косметическое: в брифе строка теста

```python
    assert report["match"] == 0          # обе строки: deny-метка и allow-метка — miss
```

`ruff format` нормализует пробелы перед комментарием до двух; применено форматтером:

```python
    assert report["match"] == 0  # обе строки: deny-метка и allow-метка — miss
```

Семантика теста идентична. Причина: бриф одновременно требует «ruff 0 замечаний по двум
файлам» и текст теста — форматтер является частью проверки, поэтому выбрано format-clean.

## Self-review

- Правка ровно одна строка в `run_basket`, больше ничего в `internet_judge_check.py`
  не трогалось (diff --stat: 2 ±).
- `InternetJudgeVerdict.verdict` — `Literal["allow", "deny", "timeout", "error"]`;
  оба неопределённых значения (`error`, `timeout`) покрыты условием — сверено по
  `internet_judge.py` (строки 169–176: timeout из judge(); 130–140: ValueError из
  `_parse_verdict` → error в try/except `run_basket`).
- Регресс G1 не сломан: `test_run_basket_survives_judge_crash` (error на expected=1)
  остаётся miss и по старой, и по новой логике — зелёный.
- `test_main_low_accuracy_exit_code_1`: стаб-судья возвращает deny → `deny in (...)`,
  но 0 != 1 → miss, accuracy 0 → SystemExit(1) — зелёный, exit code честен.
- Сценарий отказа I2 закрыт: при полностью мёртвом судье все строки — miss,
  accuracy 0.0 → exit code 1, оператор видит каждый сбой в mismatches
  (проверено новым тестом).
- Живой GigaChat не трогался; test.xlsx не использовался.

## Сомнения

Нет. Единственное отклонение описано выше (форматтер в комментарии).
