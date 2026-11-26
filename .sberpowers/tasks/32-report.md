# Отчёт задачи 3.2: CLI-прогон тестовой корзины mcp_internet-judge

**Статус: DONE_WITH_CONCERNS** (функционал полный, все проверки зелёные; одно окруженческое
наблюдение по смоук-команде из брифа — подробно ниже, дефектом задачи не является).

- REQ: REQ-mcp-internet-judge-02 (скриптовый прогон корзины печатает accuracy)
- Worktree: `.worktrees/wave3-task32`, ветка `wave3-task32`
- Коммит: **603e96d8** `feat(04): CLI-прогон тестовой корзины mcp_internet-judge (REQ-02)`
  (parent: b9d804a9, 2 файла, 186 insertions)

## Что реализовано

1. **`app/services/react/deepagent/internet_judge_check.py`** (новый, 108 строк) — CLI-модуль:
   - `load_basket(xlsx_path)` — пары `(query, expected 0|1)` из первого листа xlsx
     (A=query, B=метка; первая строка — заголовки; невалидные строки пропускаются);
   - `run_basket(gate, items)` — прогон через `gate.judge(query)` bare-строкой корзины
     (формат после фикса A совпадает с прод-форматом входа судьи), verdict==allow ↔ метка 1;
     печатает `[OK]/[MISS]` по каждому item, возвращает `{total, match, accuracy, mismatches}`;
   - `main(argv)` — CLI: флаг `MCP_INTERNET_JUDGE_ENABLED` выключен → `SystemExit` с понятным
     сообщением; пустая корзина → `SystemExit`; печать `=== Итог: match/total ===
     accuracy=X.XXXX (цель >= 0.90)` + список несовпадений; exit code 1 при accuracy < 0.90.
   - Запуск: `PYTHONPATH=. .venv/bin/python -m app.services.react.deepagent.internet_judge_check [test.xlsx]`.
2. **`tests/test_deepagent_internet_judge_basket.py`** (новый, 78 строк):
   - `test_run_basket_counts_accuracy` — подсчёт accuracy 2/3 и mismatches на стабе судьи;
   - `test_load_basket_reads_xlsx` — чтение реального `test.xlsx` (в worktree SKIP, ожидаемо);
   - `test_basket_border_pairs_with_stub` — 9 граничных пар REQ-02, включая адресную
     «курс Яндекса по управлению продуктом» (allow) и бренд-deny «тарифы на эквайринг у Сбера
     и ВТБ», accuracy == 1.0 на стабе.

Код — дословная транскрипция брифа. Отклонения (оба — ruff-канонизация, логика не тронута):
- module-level прагма `# ruff: noqa: T201` после докстринга (print — контракт CLI;
  правило T20 включено глобально в pyproject, per-file-ignores для CLI в проекте нет,
  существующие CLI-файлы вроде `app/phoenix/datasets/loader.py` под ruff не вычищены);
- 2 трейлинг-запятые (COM812) добавлены `ruff check --fix`.

## Команды и результаты

| Шаг | Команда | Результат |
|---|---|---|
| RED | `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_basket.py -v` | Collection error: `ModuleNotFoundError: No module named 'app.services.react.deepagent.internet_judge_check'` — ожидаемый RED |
| GREEN | та же, после реализации | **2 passed, 1 skipped** (`run_basket` PASS, `border_pairs` PASS, `load_basket` SKIP — test.xlsx untracked, в worktree его нет) |
| Регресс | `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v` | **13 passed** |
| Финал (после ruff-фиксов) | оба файла вместе | **15 passed, 1 skipped** |
| Ruff | `.venv/bin/ruff check <оба файла>` | сначала 6 замечаний (2× COM812, 4× T201) → `--fix` + noqa-прагма → **All checks passed!** |
| Смоук | см. ниже | **import OK + SystemExit OK** |

## TDD RED/GREEN

- RED до реализации — воспроизведён и зафиксирован точной причиной (ModuleNotFoundError).
- GREEN после реализации — 2 PASS + 1 ожидаемый SKIP, точно по брифу.
- После ruff-фиксов прогон повторён (15 passed + 1 skipped) — регрессий нет.

## Смоук импорта (наблюдение по брифовской команде)

Дословная команда из брифа `python -c "import app.services.react.deepagent.internet_judge_check"`
на этой машине падает **pre-existing**-причиной, не связанной с задачей: package
`app/services/react/deepagent/__init__.py` исполняется при импорте любого подмодуля и тянет
`service.py → services.py:76`, где на уровне модуля создаётся `AsyncConnectionPool`, требующий
running event loop → `RuntimeError: AsyncConnectionPool open with no running loop`. Проверено:
точно так же падает и уже существующий модуль судьи из задачи 2.1
(`python -c "import app.services.react.deepagent.internet_judge"`). Именно поэтому в
`tests/conftest.py` `get_db_pool` мокается до импортов app.* — и все pytest-прогоны зелёные
(фактический импорт моего модуля в проектном окружении выполняется в каждом прогоне).

Смоук выполнен в эквиваленте pytest-окружения (мок `get_db_pool` как в conftest, скрипт во
временном файле, удалён после):

```
import OK: app.services.react.deepagent.internet_judge_check
SystemExit OK: MCP_INTERNET_JUDGE_ENABLED=False — судья отключён в config.py, прогон невозможен
```

Т.е. разрешённая брифом «валидация exit-кода отключённого судьи без GigaChat» пройдена:
`main([])` при выключенном флаге поднимает `SystemExit` с понятным сообщением, к сети не идёт.

## Self-review

- [x] Транскрипция брифа дословная (логика); отклонения — только прагма noqa: T201 и 2
      трейлинг-запятые (разрешённая канонизация).
- [x] CLI-контракт: вывод accuracy + mismatches — есть; exit code 1 при <0.90 (`SystemExit(1)`);
      SystemExit при выключенном флаге — подтверждён смоуком.
- [x] Граничные пары REQ-02 в border-тесте: адресная «курс Яндекса» и бренд-deny
      «тарифы на эквайринг» + ещё 7 пар, accuracy 1.0.
- [x] `run_basket` зовёт `gate.judge(query)` bare-строкой — как одобрено фиксом A (b9d804a9).
- [x] Регресс судьи: 13 passed.
- [x] Ruff: 0 замечаний на обоих файлах.
- [x] Коммит атомарный, в стейдже только 2 файла задачи; `git status` до коммита — ничего лишнего.
- [x] test.xlsx не копировался и не коммитился; SKIP не «чинился».
- [x] Существующие файлы не изменялись (pyproject не трогал — noqa вместо per-file-ignores).

## Сомнения / наблюдения (не блокеры)

1. **Pre-existing:** голый `python -c import`/`python -m` любого подмодуля `deepagent` вне
   event loop падает на module-level DB-pool (`services.py:76`). Для диспетчерского живого
   прогона корзины после merge это значит: команда из докстринга модуля потребует либо
   запуска в среде с running loop, либо того же мока пула, что в conftest. Это общее свойство
   пакета (включая судью 2.1), выходит за рамки задачи 3.2 (правка `services.py` запрещена
   брифом — «два новых файла, ничего существующего не менять»). Отмечаю для диспетчера.
2. Если `test.xlsx` физически отсутствует, `main()` упадёт с `FileNotFoundError` из openpyxl
   раньше ветки `SystemExit("Корзина пуста или не найдена")` (openpyxl открывает файл до
   возврата пустого списка). Поведение задано брифом дословно; в главном дереве файл есть.
3. Приёмочный прогон против живого GigaChat (accuracy ≥ 0.90 по 136 строкам) сознательно не
   выполнялся (бриф: живой прогон не делать); в CI механика покрыта стабом судьи.

## Изменённые файлы

- `app/services/react/deepagent/internet_judge_check.py` — новый (108 строк)
- `tests/test_deepagent_internet_judge_basket.py` — новый (78 строк)
