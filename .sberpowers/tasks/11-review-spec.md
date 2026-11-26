# Ревью по оси A (соответствие спеке): задача 1.1 — Async-хуки interceptor

- Изменение: `6a4041ed..15cc86f5`, коммит `feat(01): async-хоки ToolInterceptor, pre/post_execute (REQ-05)`
- Ось: соответствие REQ-mcp-internet-judge-05 из delta-спеки
  `docs/sberpowers/specs/2026-10-06-mcp-internet-judge.md` + бриф задачи 1.1
- Дифф: `app/services/react/deepagent/interceptor.py`, `tests/test_deepagent_interceptor_async.py` (новый)

## Что сделано хорошо

- Логика из сниппетов брифа реализована дословно: helper `_resolve_hook_result`
  (interceptor.py:361-365) — один в один; вставки `await _resolve_hook_result(...)`
  в pre-ветке (interceptor.py:259) и post-ветке (interceptor.py:298) — дословно,
  внутри прежних try/except, логи и fail-open-семантика не тронуты.
- `import inspect` поднят наверх (interceptor.py:51), локальный импорт из
  `_is_async_callable` убран — ровно как предписывал бриф.
- Тест-файл — идентичен сниппету брифа (сверено построчно): 6 кейсов покрывают
  skip / override-args / fail-open / post-modify + обе обратные совместимости.
- Docstring `ToolInterceptor` дополнен описанием async-контракта — соответствие
  тексту требования спеки зафиксировано в самом коде.
- Заявленная адаптация под ruff проверена и подтверждена: суть контракта сохранена
  (детали ниже в таблице сверок).

## Сверка с REQ-mcp-internet-judge-05

Требование дословно: «`ToolInterceptor` принимает асинхронные
`pre_execute`/`post_execute`: coroutine-callable вызывается с await, без
блокировки event loop стриминга. Существующие синхронные хуки продолжают работать
без изменений (обратная совместимость); судья не блокирует параллельные
tool-выовы агента.» Приёмка: async-callback работает (юнит-тест на interceptor
с coroutine-функцией); все прежние использования sync-хуков проходят существующие
тесты без изменений контракта.

### Декомпозиция и вердикты по подпунктам

| Пункт | Класс | Доказательства |
|---|---|---|
| coroutine-callable pre_execute вызывается с await | SATISFIED | код app/services/react/deepagent/interceptor.py:127-131 (аннотация допускает `Callable[..., Awaitable[...]]`), :259 (`result = await _resolve_hook_result(result)`); тест tests/test_deepagent_interceptor_async.py:44, 70 |
| coroutine-callable post_execute вызывается с await | SATISFIED | код interceptor.py:132-136, :298 (`modified = await _resolve_hook_result(modified)`); тест tests/test_deepagent_interceptor_async.py:111 |
| без блокировки event loop стриминга | SATISFIED | код interceptor.py:259, 298, 361-365 — чистый `await` через `inspect.isawaitable`, без `run_async_sync`, потоков и блокирующих вызовов; тест :44, 111 (`await asyncio.sleep(0)` внутри хука не блокирует loop) |
| обратная совместимость: sync-хуки работают без изменений | SATISFIED | код interceptor.py:361-365 (sync-результат проходит насквозь), try/except и логи не изменены (дифф добавляет только строку resolve); тест :82, 124 |
| судья не блокирует параллельные tool-вызовы | NOT-VERIFIABLE | механизм не блокирующий: каждый вызов идёт через свой `awrap_tool_call` (interceptor.py:240) с независимым await, глобальных локов в диффе нет; полная проверка — задача 2.1 (судья ещё не существует). Диспетчеру: при ревью 2.1 убедиться, что хук судья не оборачивается в `run_async_sync`/синхронный LLM-клиент |
| Приёмка: юнит-тест coroutine-перехватчика работает | SATISFIED | tests/test_deepagent_interceptor_async.py — 6 passed (независимо подтверждено диспетчером) |
| Приёмка: прежние sync-использования проходят существующие тесты без изменений контракта | SATISFIED | контракт sync-пути не изменён (см. выше); stateful_routing и declarative builder диффом не затронуты; tests/test_deepagent_session_id.py — 17 passed (отчёт исполнителя); нюанс по существующим wiring-тестам — находка M1 ниже (причина вне диффа) |

### Сверка шагов брифа

| Шаг брифа | Класс | Доказательства |
|---|---|---|
| 1. Тест создан по сниппету | SATISFIED | tests/test_deepagent_interceptor_async.py — построчно идентичен сниппету брифа (126 строк, 6 тестов) |
| 2. RED зафиксирован | SATISFIED | отчёт исполнителя: 4 failed (async-кейсы, `AttributeError: 'coroutine' object has no attribute 'tool_call'`) / 2 passed (sync уже работали) — соответствует механизму корутины-как-результата; соответствует уточнению брифа «FAIL на всех async-кейсах» |
| 3а. Аннотации расширены | SATISFIED | interceptor.py:127-136 — см. блок «Адаптация под ruff» |
| 3б. Helper `_resolve_hook_result` | SATISFIED | interceptor.py:361-365 — дословно из брифа; `import inspect` наверху (:51), локальный импорт убран |
| 3в. Обе ветки `awrap_tool_call` | SATISFIED | interceptor.py:259 (pre), :298 (post) — вставки дословные, try/except и логи сохранены |
| 4. GREEN | SATISFIED | 6 passed — независимо подтверждено диспетчером |
| 5. Регресс interceptor-зоны | SATISFIED c находкой M1 | session_id 17 passed; wiring-тесты subagents падают по предсуществующей причине (M1) |
| 6. ruff check чист | SATISFIED | проверено мной независимо: `.venv/bin/ruff check interceptor.py tests/test_deepagent_interceptor_async.py` → `All checks passed!`; `ruff format --check` расходится, но расходился и с BASE interceptor.py, форматтер проектом не используется (COM812 в конфиге) — бриф шаг 6 даёт приоритет стилю файла, замечание снято |
| 7. Коммит: один атомарный, № задачи + REQ-ID, 2 файла | SATISFIED | `15cc86f5 feat(01): async-хоки ToolInterceptor, pre/post_execute (REQ-05)`; stat: ровно 2 файла задачи |

### Адаптация под ruff (заявлена исполнителем — суть контракта проверена)

| Отклонение от сниппета брифа | Проверка |
|---|---|
| Аннотации без кавычек (`UP037` ×2 в сниппете) | Файл под `from __future__ import annotations` — все аннотации ленивые строки в любом случае, семантика идентична. Контракт сохранён |
| `Awaitable` из `collections.abc` вместо `typing` (`UP035` в сниппете) | Тот же объект в Python 3.13; семантика идентична |
| `Awaitable` в TYPE_CHECKING-блоке (`TC003`) | Безопасно: `ToolInterceptor` — обычный dataclass, аннотации не вычисляются в рантайме; ruff чист (проверено) |
| Неиспользуемый `Coroutine` из сниппета не импортирован (`F401`) | В аннотациях самого брифа `Coroutine` не участвует (только `Awaitable`) — ничего не потеряно |

Вывод: все четыре адаптации канонизационные, контракт по существу не изменён.
Заявка исполнителя о «разумной адаптации» подтверждается.

## Обратный проход (EXTRA)

Каждый поведенческий фрагмент диффа отображён в REQ-05: расширение аннотаций,
docstring контракта, подъём `inspect`, две вставки await, helper, тест-файл.
Единственное поведенческое следствие вне букв текста требования — исключения,
поднятые в теле корутины хука, теперь глотаются тем же fail-open except'ом
(interceptor.py:259 внутри try). Это прямо предписано брифом («сохраняя
try/except-семантику»), покрыто тестом :94 (`test_async_pre_execute_exception_fails_open`)
и для sync-хуков ничего не меняет. EXTRA-поведения нет.

## Именуемые риски, проверенные вне диффа

1. **R1: скрытый sync-двойник `wrap_tool_call`, делящий логику хуков** — grep
   `def wrap_tool_call|def awrap_tool_call` по `app/`: у InterceptorMiddleware
   только async `awrap_tool_call` (interceptor.py:240); других реализаций в
   interceptor.py нет (совпадения — tool_loop_guard.py:206, shell_policy_middleware.py:187,
   чужие middleware). Пути, где корутина могла бы уйти в sync-код, нет.
2. **R2: прямые вызовы хуков вне interceptor.py** — grep `\.pre_execute\(|\.post_execute\(`
   по `app/` без interceptor.py: совпадений нет. Все потребители идут через
   `awrap_tool_call`; ни один внешний call site не может получить сырой
   coroutine-объект.
3. **R3: единственный существующий тест InterceptorMiddleware** —
   `tests/test_deepagent_subagents.py:662-672` (wiring). Прогнал сфокусированно:
   `pytest tests/test_deepagent_subagents.py -k "interceptor"` → 2 failed
   (`test_interceptor_middleware_added` — `assert 6 == 4`,
   `test_no_interceptor_middleware_when_none` — `assert 5 == 3`). Assertion —
   жёсткая длина middleware-стека (`len(mw) == 4` / `== 3`, тест :668, :681),
   сломанная дрейфом develop-коммитов (ShellToolPolicyMiddleware и др.), НЕ
   хуками. Дифф middleware не добавляет и стек не трогает — причинная связь
   с задачей исключена; исполнитель подтвердил те же падения stash-прогоном на BASE.

## Находки

### M1 (Minor, окружение/stale-тесты — решает диспетчер)

`tests/test_deepagent_subagents.py:662-672` — оба wiring-теста InterceptorMiddleware
падают (`assert 6 == 4`, `assert 5 == 3`): жёсткие размеры middleware-стека
устарели после develop-коммитов, добавивших middleware в стек сабагентов.
Не является дефектом этого диффа (падения существуют на BASE, причинно не
связаны с хуками; исполнитель зафиксировал это в отчёте как сомнение).
Буквальная часть приёмки REQ-05 «прежние использования проходят существующие
тесты» в этой зоне фактически не зелёная — но контракт синхронного пути
не изменён, что и есть суть приёмки («без изменений контракта»).
Как чинить (вне задачи): отдельной задачей актуализировать ожидания длины
стека или перевести asserts на `any(isinstance(m, InterceptorMiddleware) ...)`.
Решение за диспетчером — уже поднято исполнителем.

### M2 (Minor, наблюдение — не дефект)

Отчёт исполнителя указывает `+21/−8` для interceptor.py, пакет ревью — 30
изменённых строк; расхождение счёта ±1-2 строки, на суть не влияет.

## Не проверяемо из диффа

- Подпункт «судья не блокирует параллельные tool-вызовы» полностью: судья
  появится в задаче 2.1. На уровне interceptor'а механизм параллельности
  поддержан (независимые `awrap_tool_call` без блокировок). Диспетчеру: в задаче
  2.1 проверить, что judge-хук не использует `run_async_sync`/синхронные клиентские
  вызовы, стриминг параллельных тулов не сериализуется.
- Предсуществование 10 падений `test_stateful_routing.py` /
  `test_deepagent_subagents.py` на BASE независимо перепроверено только для
  interceptor-wiring пары (R3, причинно); остальные 8 принимаются по
  stash-проверке исполнителя (read-ревью не мог переключать дерево).

## Таблица по всем пунктам

| Пункт | Класс | Доказательства |
|---|---|---|
| Requirement: REQ-mcp-internet-judge-05 (общий) | SATISFIED | код interceptor.py:127-136, 259, 298, 361-365; тест tests/test_deepagent_interceptor_async.py:44, 70, 82, 94, 111, 124 — 6 passed |
| Scenario: coroutine pre_execute → skip (ToolOverrideResult) | SATISFIED | код interceptor.py:259; тест tests/test_deepagent_interceptor_async.py:44 |
| Scenario: coroutine pre_execute → override args | SATISFIED | код interceptor.py:259; тест :70 |
| Scenario: sync pre_execute (обратная совместимость) | SATISFIED | код interceptor.py:361-365; тест :82 |
| Scenario: исключение в coroutine pre_execute → fail-open | SATISFIED | код interceptor.py:255-264 (await внутри try); тест :94 |
| Scenario: coroutine post_execute модифицирует ToolMessage | SATISFIED | код interceptor.py:298; тест :111 |
| Scenario: sync post_execute (обратная совместимость) | SATISFIED | код interceptor.py:294-303; тест :124 |
| Scenario: судья не блокирует параллельные tool-вызовы | NOT-VERIFIABLE | механизм: interceptor.py:259/298 — await без блокировок; полная проверка — задача 2.1 |
| вне требований | — (EXTRA нет) | обратный проход: все фрагменты диффа отображены в REQ-05 |

## Вердикт

MISSING, MISUNDERSTOOD и EXTRA отсутствуют: поведение REQ-mcp-internet-judge-05
реализовано полностью и дословно по брифу с сохранением сути контракта при
ruff-канонизации; обратная совместимость подтверждена тестами; единственные
падения в зоне — предсуществующие на BASE и причинно не связанные с задачей
(M1, за диспетчером). NOT-VERIFIABLE: полная проверка неблокирующей
параллельности судьи отложена до задачи 2.1.

Verdict: PASS
