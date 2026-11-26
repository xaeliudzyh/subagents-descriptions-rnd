# Ревью оси A — соответствие спеке: задача 2.1 (повторное, после фикса A/B)

- Дифф: f23bd5bb..b9d804a9 (c22a10d4 feat(02) + b9d804a9 fix(02)); 3 файла, +429 строк
- Зона: REQ-mcp-internet-judge-02, -04, -06, -07; фикс A — REQ-02/06, фикс B — REQ-08/03 (поверхность reason)
- Дельта: docs/sberpowers/specs/2026-10-06-mcp-internet-judge.md; бриф .sberpowers/tasks/21-brief.md; fix-бриф .sberpowers/tasks/21-fix-brief.md
- Источник диффа: .sberpowers/tasks/21-review-package.md (прочитан целиком, git не перегонялся)
- Тесты: из отчёта исполнителя 13 passed + 6 регресс, RED 2 failed для A/B; диспетчер независимо подтвердил 19 passed. Сьют не перегонялся (инструкция ревью); сомнений, требующих одиночного запуска, нет.
- Проверка вне диффа (одна, именуемая): grep InternetJudgeGate по репо — риск преждевременного вивинга судьи в harness вне зоны 2.1. Результат: упоминания только в самом модуле app/services/react/deepagent/internet_judge.py, его тесте и план-доке docs/sberpowers/plans/. Вивинга нет — правильно, это задача 3.1.

## Закрытие утверждённых находок

### Находка A (REQ-02/06): вход судьи bare-текстом — ЗАКРЫТА

- Было: query = args if isinstance(args, str) else json.dumps(...) — dict-args уходили судье JSON-обёрткой вида {"query": ...}, рассинхрон с few-shot (bare-строки) и приёмочной корзиной REQ-02.
- Стало: app/services/react/deepagent/internet_judge.py:184-192 — конверсия дословно по fix-брифу: str → как есть; dict → " ".join(строковые непустые значения) с fallback json.dumps для пустых/экзотических; прочее → json.dumps. Формат входа синхронен с _FEW_SHOT_EXAMPLES и корзиной REQ-02.
- Тесты: tests/test_deepagent_internet_judge.py:139-164 (test_dict_args_passed_as_plain_text: в промпте «найди курс по Java», нет '{"query"'-обёртки; нестроковой limit=5 не шумит), tests/test_deepagent_internet_judge.py:167-186 (test_str_args_passed_through). Оба соответствуют fix-брифу дословно (модульно форматированию).
- REQ-06 сохранён: судья по-прежнему получает фактические аргументы вызова (все строковые параметры) + исходный вопрос пользователя; отсечение нестроковых значений — решение человека, зафиксированное в fix-брифе («нестроковые значения (лимиты, флаги) не шумят промпт»).

### Находка B (REQ-08 surface / REQ-03/04 причина): reason без деталей исключения — ЗАКРЫТА

- Стало: app/services/react/deepagent/internet_judge.py:203 — verdict = InternetJudgeVerdict("error", "сбой LLM-судьи") без {exc}; дословно по fix-брифу. Внутренние детали (URL эндпоинтов и пр.) больше не попадают в agent-facing to_metadata() / tool-result.
- Детали сохранены для эксплуатации: internet_judge.py:198-202 logger.error(..., exc) — строка лога не менялась; уровень ERROR соответствует REQ-08 («ERROR для сбоя судьи»).
- Поверхности REQ-03/REQ-04 не повреждены: deny-причина агенту сохранена (internet_judge.py:217, content = _DENY_TITLE + reason судьи); timeout/error получают человекочитаемый _FAIL_CLOSED_TITLE (internet_judge.py:218-219) — соответствует REQ-04 «судья недоступен».
- Тест: tests/test_deepagent_internet_judge.py:189-202 (test_error_reason_has_no_exception_details): internal-host отсутствует и в content, и в to_metadata(); reason == «сбой LLM-судьи». Дословно по fix-брифу.

## Сверка REQ зоны задачи

| Пункт | Класс | Доказательства |
|---|---|---|
| Requirement: REQ-mcp-internet-judge-02 | SATISFIED | код app/services/react/deepagent/internet_judge.py:31-66 (промпт БР/БЛиК П298: три потребности, адресная пара «курс Яндекса»→allow / «тарифы на эквайринг»→deny, ПДн-блок, deny-список, «сомнение в пользу deny», ответ строго JSON), internet_judge.py:70-88 (14 few-shot пар, 7 allow / 7 deny — критерий «≥10, 7/7»; все граничные пары приёмки присутствуют: антифрод-статья→allow, инвестиции→deny, Цифровой рубль→allow, новости о ЦР→deny, книга по микроэкономике→allow, новые книги→deny, ИНН→deny; телефон/паспорт — ПДн-блок промпта); тест tests/test_deepagent_internet_judge.py:32-43, 60-85. Прогон корзины ≥0.90 — задача 3.2 (traceability), в зоне 2.1 не требуется |
| Requirement: REQ-mcp-internet-judge-04 | SATISFIED | код internet_judge.py:161-178 (asyncio.wait_for + except TimeoutError → verdict "timeout"), internet_judge.py:195-203 (except Exception ВНУТРИ pre_execute → verdict "error"; внешний fail-open interceptor не задействуется), internet_judge.py:206-220 (блокирующий ToolOverrideResult status="error" для deny/timeout/error); невалидный ответ → ValueError из _parse_verdict → тот же except → блок. Тесты tests/test_deepagent_internet_judge.py:88-127 (timeout / LLM-error / garbage — ассертят вердикт в реестре и status="error") |
| Requirement: REQ-mcp-internet-judge-06 | SATISFIED | код internet_judge.py:105-117 (build_internet_judge_messages(query, user_question): system + few-shot + финальная пара «Запрос тула: … / Вопрос пользователя: …»; история диалога не передаётся — сигнатура принимает только query и user_question), internet_judge.py:184-192 (фактические args вызова после фикса A); тест tests/test_deepagent_internet_judge.py:49-54 (оба элемента в промпте) + 139-186 (форматы входа после фикса A) |
| Requirement: REQ-mcp-internet-judge-07 | SATISFIED | код app/core/config.py:160-164 (три поля AppSettings сразу после блока CUSTOM_PROVIDER_*; дефолты True / GigaChat-3-Lightning / 15.0 — соответствуют REQ-07); потребление в internet_judge.py:149 (MODEL), internet_judge.py:167 (TIMEOUT). Kill-switch ENABLED end-to-end и серверная регистрация — вивинг задачи 3.1 (traceability REQ-07 = 2.1+3.1); в зоне 2.1 поля определены и читаются, что соответствует критерию выхода брифа |
| Фикс A (утверждённая находка, REQ-02/06) | SATISFIED (закрыта) | код internet_judge.py:184-192; тест tests/test_deepagent_internet_judge.py:139-186 |
| Фикс B (утверждённая находка, REQ-08 surface) | SATISFIED (закрыта) | код internet_judge.py:203, лог с exc сохранён internet_judge.py:198-202; тест tests/test_deepagent_internet_judge.py:189-202 |

## Обратный проход (EXTRA)

- Дифф содержит ровно 3 файла задачи (config.py +7; internet_judge.py новый, 220 строк; tests/test_deepagent_internet_judge.py новый, 202 строки; итого +429 — сходится с diff-stat). Посторонних файлов нет.
- Преждевременного вивинга в harness нет (именованная grep-проверка выше) — подключение судьи корректно отложено в задачу 3.1.
- Логгирование INFO/WARNING/ERROR (internet_judge.py:207, 211, 198) — требуемое поведение REQ-08, не EXTRA. Хранение message_id без внутреннего использования — предписанный интерфейс брифа для задачи 3.1, не EXTRA.
- Косметические отклонения от сниппетов брифа (trailing comma COM812, перенос few-shot строки, аннотация возврата без кавычек, ассерт «погоды» вместо «погода») — канонизация ruff и несостыковка брифа, принятые первым ревью (PASS); на REQ не влияют, в повторном прогоне не изменились.

## Вне зоны 2.1 (не находки по указанию диспетчера и traceability плана)

- REQ-01 (kill-switch end-to-end, регистрация для всех чатов) — задача 3.1.
- REQ-03 — deny tool-result произведён здесь (internet_judge.py:210-217); полная реализация (история шагов, наблюдаемость) — 3.1.
- REQ-05 — задача 1.1 (выполнена). REQ-08 (метаданные шага stat-feedback) — 3.1.
- Прогон корзины test.xlsx ≥0.90 — задача 3.2.

## Замечания Minor (не блокируют, на классы MISSING/MISUNDERSTOOD не влияют)

1. Minor (косметика отчёта): в разделе первой поставки 21-report.md счётчик строк internet_judge.py остался «214», после фикса фактически 220 — расхождение в тексте отчёта, коду не касается.
2. Minor (fix-бриф vs факт): прогноз RED «первые два упадут» неточен — test_str_args_passed_through был зелёным и до фикса (старый код str-args уже пропускал как есть); исполнитель честно это задокументировал. RED «2 failed» корректно воспроизводит сами находки A (JSON-обёртка) и B (утечка деталей).
3. Наблюдение для 3.2 (по духу fix-брифа, не находка): при нескольких строковых параметрах dict-args значения склеиваются через пробел без имён полей — поведение утверждено человеком («имя поля приходит с MCP-сервера динамически»).

## Вердикт

Все пункты зоны 2.1 — SATISFIED; находки A и B закрыты дословно по fix-брифу; новых расхождений и EXTRA нет; пунктов NOT-VERIFIABLE в зоне 2.1 нет (kill-switch end-to-end — задача 3.1, accuracy корзины — задача 3.2, оба вне зоны по traceability).

Verdict: PASS
