# Отчёт задачи 2.1: Модуль судьи InternetJudgeGate

- Статус: **DONE_WITH_CONCERNS** (одно минимальное отклонение от сниппета теста в брифе — см. «Отклонения»)
- Коммит: `c22a10d4` feat(02): InternetJudgeGate — pre-gate LLM-судья mcp_internet (REQ-02,04,06,07)
- Ветка: `mcp-internet-judge` (родитель BASE `f23bd5bb`)

## Что реализовано

1. **`app/core/config.py`** — три поля `AppSettings` сразу после блока `CUSTOM_PROVIDER_*`
   (стиль соседних секций `# --- ... ---`):
   - `MCP_INTERNET_JUDGE_ENABLED: bool = True`
   - `MCP_INTERNET_JUDGE_MODEL: str = "GigaChat-3-Lightning"`
   - `MCP_INTERNET_JUDGE_TIMEOUT_SECONDS: float = 15.0`
2. **`app/services/react/deepagent/internet_judge.py`** (новый, 214 строк):
   - `_JUDGE_SYSTEM_PROMPT` — назначение по БР/БЛиК П298: три потребности (внешнее обучение
     и мероприятия; адресный поиск по источнику; объединение с рекомендациями Пульса),
     адресное правило («курс Яндекса по управлению продуктом» → allow; «тарифы на эквайринг
     у Сбера и ВТБ» → deny), блок ПДн, «сомнение трактуй в пользу deny», ответ строго JSON.
   - `_FEW_SHOT_EXAMPLES` — 14 пар: 7 allow / 7 deny (критерий выхода «≥10, 7/7» соблюдён);
     граничные пары корзины test.xlsx (погода, новости, новые книги, инвестиции, футбол,
     эквайринг, ИНН) + адресная пара из БР.
   - `InternetJudgeVerdict` — frozen dataclass, `verdict: Literal["allow","deny","timeout","error"]`,
     `reason: str`, `to_metadata() -> {"internet_judge": {"verdict": ..., "reason": ...}}`.
   - `build_internet_judge_messages(query, user_question)` — system + few-shot + user-пара.
   - `InternetJudgeGate` — dataclass (`chat_id`, `message_id`, `user_question`,
     `verdicts: dict[str, InternetJudgeVerdict]` по `tool_call_id`);
     `async pre_execute(request)` → None | ToolOverrideResult(status="error");
     `async judge(query)` с `asyncio.wait_for(..., timeout=app_settings.MCP_INTERNET_JUDGE_TIMEOUT_SECONDS)`;
     `async _call_llm(messages)` — мок-точка, ленивый импорт `GigachatAsyncClient.get_client(...)`,
     конкатенация content-блоков по паттерну openai_compatible.py;
     `_parse_verdict` (статик): markdown-``` обёртка, ValueError при невалидном.
   - Fail-closed: TimeoutError → verdict "timeout"; любая ошибка LLM/ValueError парсинга ловится
     `except Exception` ВНУТРИ `pre_execute` → verdict "error" → синтетический tool-result
     «LLM-судья недоступен». Внешний fail-open перехватчик InterceptorMiddleware не задействуется.
3. **`tests/test_deepagent_internet_judge.py`** (новый, 132 строки, 10 тестов): `_parse_verdict`
   (plain/markdown/invalid), build messages, allow/deny/timeout/error/garbage через
   `monkeypatch.setattr(gate, "_call_llm", ...)`, `to_metadata`. GigaChat всегда мок.

## Изменённые файлы

- `app/core/config.py` (+7)
- `app/services/react/deepagent/internet_judge.py` (+214, новый)
- `tests/test_deepagent_internet_judge.py` (+132, новый)

## TDD-доказательства

### RED (до реализации)

Команда: `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v`

```
ERROR collecting tests/test_deepagent_internet_judge.py
ImportError while importing test module '.../tests/test_deepagent_internet_judge.py'
E   ModuleNotFoundError: No module named 'app.services.react.deepagent.internet_judge'
Interrupted: 1 error during collection
```

Падение ожидаемо: модуль ещё не создан — ровно прогноз шага 3 брифа.

### GREEN (первый прогон после реализации)

Команда та же. Результат: **9 passed, 1 failed** — падение `test_deny_blocks_with_reason`
(несостыковка брифа, см. ниже):

```
>       assert "погода" in result.content  # причина агенту
E       AssertionError: assert 'погода' in 'Поиск в интернете заблокирован судьёй проверки
        назначения: прогноз погоды не связан с обучением'
```

Логика deny при этом работает корректно — причина доходит до агента (видна в content);

### GREEN (финальный)

- `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v` → **10 passed**
- Регресс (шаг 6): `.venv/bin/python -m pytest tests/test_deepagent_interceptor_async.py -v`
  → **6 passed** (задача 1.1 не задета)
- Совместный прогон обоих файлов → **16 passed**

### Ruff (шаг 7)

`.venv/bin/ruff check app/core/config.py app/services/react/deepagent/internet_judge.py
tests/test_deepagent_internet_judge.py` → **All checks passed!**
(0 замечаний; при первом прогоне было 1 замечание COM812 — добавлена trailing comma
в `messages.append({...,})`, канонизация сниппета).

## Отклонения от брифа (все — канонизация или несостыковка брифа)

1. **Ассерт `test_deny_blocks_with_reason`: `"погода"` → `"погоды"`.** Внутренняя несостыковка
   сниппетов брифа: мок-причина в родительном падеже («прогноз погод**ы**»), ассерт искал
   именительный («погод**а**») — подстроки «погода» в content нет. Доказано: python-проверка
   до старта + RED-after-implementation (вывод выше). Исправлен один символ с сохранением
   интента ассерта «причина агенту» (deny-причина действительно попадает в tool-result).
   Остальной тест и реализация — дословно.
2. Убраны неиспользуемые импорты сниппетов (F401 недопустим, разрешено брифом):
   `InterceptorConfig` в TYPE_CHECKING-блоке internet_judge.py; `ToolMessage` в тесте.
3. `pre_execute` — аннотация возврата без кавычек (как в секции «Интерфейсы» брифа; UP037).
4. COM812: trailing comma в последнем `messages.append(...)` (дословный сниппет нарушал ruff).
5. Перенос строки `("курс Яндекса...", ...)` few-shot пары на две строки (читаемость; E501
   отключён, поведение идентично).

## Self-review

- **Полнота**: все сущности секции «Производит» присутствуют, сигнатуры сверены с секцией
  «Интерфейсы»: frozen `InternetJudgeVerdict(verdict: Literal[...], reason: str)` +
  `to_metadata()`; `InternetJudgeGate(chat_id, message_id, user_question)` + `verdicts` dict;
  `async pre_execute(request) -> ToolCallRequest | ToolOverrideResult | None`;
  `async judge(query) -> InternetJudgeVerdict`; `async _call_llm(messages) -> str`;
  static `_parse_verdict(content) -> tuple[str, str]`; `build_internet_judge_messages`.
  Промпт содержит три потребности БЛиК П298 + адресное правило + 14 (7/7) few-shot пар.
  Конфиг-дефолты проверены живым чтением:
  `app_settings.MCP_INTERNET_JUDGE_ENABLED/_MODEL/_TIMEOUT_SECONDS` →
  `True | GigaChat-3-Lightning | 15.0`.
- **Явная проверка асинхронности хука** (закрывает NOT-VERIFIABLE ревью 1.1): grep
  `run_async_sync|ainvoke|wait_for` по internet_judge.py — `run_async_sync` **0 совпадений**,
  синхронного LLM-клиента нет. Весь путь асинхронный:
  `async pre_execute` → `async judge` → `await asyncio.wait_for(self._call_llm(...))` →
  `await llm.ainvoke(messages)` (GigachatAsyncClient — httpx.AsyncClient, ленивый импорт).
  Event loop не блокируется.
- **Fail-closed подтверждён тестами**: timeout → verdicts["call_9"].verdict == "timeout" и
  error-tool-result; RuntimeError LLM → "error"; garbage-ответ → ValueError → "error".
  Исключения ловятся внутри pre_execute, внешний except interceptor'а не используется.
- **Дисциплина**: стейджинг — ровно 3 файла задачи; посторонние untracked-файлы пользователя
  (.env.env.bak-tokens, SALUTE_TTS.md, lib.salute-0.1.13.tgz, test.xlsx, tools/) не тронуты;
  deepagent/__init__.py не менялся (вивинг — задача 3.1); известные падения на BASE
  (test_deepagent_subagents.py / stateful_routing-зона) не чинились — вне зоны.
- **Тесты**: проверяют реальное поведение gate (парсинг, маршрутизация вердиктов, реестр,
  содержимое tool-result), а не моки; LLM мокается только в мок-точке `_call_llm`, как
  предписывает бриф; вывод тестов чистый (только стандартные deprecation-warnings conftest).

## Сомнения и проблемы

1. Единственное содержательное отклонение — ассерт «погода»→«погоды» (см. выше). Если
   диспетчер считает иначе, откат — однострочный; интент «причина агенту» при этом сохранён.
2. Замечание known-issue interceptor.py (`exec_request` NameError при pre(skip)+post) не
   касается судьи: он регистрирует только `pre_execute`, без `post_execute`.
3. Полный прогон всего tests/ не выполнялся — по брифу зона проверки это судейский набор +
   регресс 1.1 + ruff; прочие падения объявлены диспетчером как предсуществующие на BASE.

---

# Фикс A/B (чекпоинт волны 2): находки ревью задачи 2.1

- Статус: **DONE**
- Коммит: `b9d804a9` fix(02): вход судьи bare-текстом; причина сбоя без деталей (REQ-02,06,08)
- База фикса: `c22a10d4` (HEAD ветки на старте); fix-бриф: `.sberpowers/tasks/21-fix-brief.md`

## Что исправлено

1. **Находка A (REQ-02, REQ-06)** — `pre_execute`, конверсия args заменена: dict-args со
   строковыми значениями идут судье **bare-текстом** (`" ".join(text_parts)`), а не
   JSON-обёрткой `{"query": ...}`. Имя поля схемы тула приходит с MCP-сервера динамически —
   судья больше от него не зависит; нестроковые значения (лимиты, флаги) не шумят промпт;
   пустой/экзотический dict/list — fallback `json.dumps`. Формат входа теперь синхронен с
   few-shot корзиной (`_FEW_SHOT_EXAMPLES` — bare-строки, как и приёмочная корзина REQ-02).
2. **Находка B (REQ-08)** — error-вердикт получает reason `"сбой LLM-судьи"` без `{exc}`:
   текст исключения (URL эндпоинтов, внутренние детали) больше не уходит в agent-facing
   `to_metadata()` / tool-result. Полные детали по-прежнему пишутся в `logger.error`
   (строка лога не менялась).

## TDD-доказательства

### RED (тесты из fix-брифа добавлены, фиксы не внесены)

`.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v` → **2 failed, 11 passed**:

```
FAILED tests/test_deepagent_internet_judge.py::test_dict_args_passed_as_plain_text
  - assert '{"query"' not in 'Запрос тула: {"query": "найди курс по Java", "limit": 5}\n...'
    (находка A воспроизведена: JSON-обёртка в промпте судьи)
FAILED tests/test_deepagent_internet_judge.py::test_error_reason_has_no_exception_details
  - AssertionError: assert 'сбой LLM-суд...host:8080/v1' == 'сбой LLM-судьи'
    (находка B воспроизведена: внутренний URL утекал в reason)
```

`test_str_args_passed_through` прошёл уже на RED — прогноз fix-брифа «первые два упадут»
для него неточен: это регрессионный тест (str-args уже проходили как есть до фикса); его
роль — зафиксировать, что фикс A не сломает str-путь. На GREEN он остался зелёным.

### GREEN (после обоих фиксов)

- `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v` → **13 passed**
  (10 прежних + 3 новых), warnings — только предсуществующие Pydantic-deprecation из conftest.
- Регресс 1.1: `.venv/bin/python -m pytest tests/test_deepagent_interceptor_async.py -v`
  → **6 passed**.
- Ruff: `.venv/bin/ruff check app/services/react/deepagent/internet_judge.py
  tests/test_deepagent_internet_judge.py` → **All checks passed!** (0 замечаний).

## Self-review

- Оба фикса внесены дословно по fix-брифу: конверсия args (hunk 1) и reason без `{exc}`
  (hunk 2). Дифф модуля — только эти два hunk; ветки allow/deny/timeout, fail-closed
  try/except, реестр verdicts, строка `logger.error(..., exc)` не изменены.
- Тесты проверяют реальное поведение: перехватывают фактические `messages` на мок-точке
  `_call_llm` (единственная точка мокирования LLM, как в исходном наборе), ассертят
  содержимое user-сообщения судьи, реестр вердиктов и `to_metadata()` — не моки-стабы.
- Коммит стейджил ровно два файла задачи; посторонние untracked (.env.env.bak-tokens,
  SALUTE_TTS.md, lib.salute-0.1.13.tgz, test.xlsx, tools/) не тронуты.
- Отчёт 21-report.md дополнен постфактум и в коммит судейского фикса не входит (вне
  стейджинга по шагам fix-брифа).
