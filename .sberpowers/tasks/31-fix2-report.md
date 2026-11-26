# Отчёт задачи 3.1-FIX2: досл вердиктов на всех выходах стрима

- Worktree: `.worktrees/wave3-task31`, ветка `wave3-task31`.
- Базовый HEAD: `31da4c6f` → новый коммит **`c134b2ea`**
  `fix(03): досл вердиктов на всех выходах стрима; продвижение step_sequence; step_input вопроса (REQ-08)`.
- Изменённые файлы (ровно 2, как в брифе): `app/services/react/deepagent/streaming.py`,
  `tests/test_deepagent_internet_judge_wiring.py`. Дифф: +42/−25.

## Выполнение по пунктам брифа

### H1 + ось A MISSING: flush на всех завершающих путях

Определения `_judge_feedback` / `_flush_unsent_judge_verdicts` перенесены из post-loop-блока
ДО цикла `async for event in agent.astream_events(` (вставлены после `_stateful_task_returned`,
строки ~612–639 в новом файле). Три точки вызова flush:

| Путь | Место | Контроль по коду |
|------|-------|------------------|
| (а) терминальный pptx-return | ветка `if file_url and is_dl:` on_tool_end, перед `await cleanup(...); return` (~867) | единственный `cleanup(...); return` такой формы в файле (второй cleanup — нормальный выход в секции 9, без return) |
| (б) нормальный выход | на месте удалённого post-loop-блока, один вызов (~1035) | старый блок (определения + прямой вызов `send_unsent_judge_verdicts`) удалён полностью |
| (в) аварийный выход 7.7 | между исчерпанием `async for error_response in yield_error_response(...)` и `return` (~1030) | отступ 8 пробелов = уровень `async for`, вне тела yield-цикла |

Поток «mcp_internet → deny → модель адаптировалась → convert_to_pptx» теперь досылает
deny-вердикт до завершения генератора — приёмка REQ-08 «включая заблокированные» выполняется
на штатном пути.

### H2: продвижение step_sequence

`_flush_unsent_judge_verdicts()` содержит `nonlocal _tool_step_sequence` и
`_tool_step_sequence += await send_unsent_judge_verdicts(...)`. Семантика:
аргумент `base_step_sequence=_tool_step_sequence` читается до await (base=N),
функция внутри инкрементирует свою копию N+1..N+k и возвращает k, затем prod-счётчик
становится N+k. Финальный message-шаг секции 8 (`_tool_step_sequence += 1`) получает
N+k+1 — дублей/убывания порядковых номеров в витрине agents_feedback больше нет.

### H3: step_input вопроса

В `_judge_feedback` параметр `step_input=request.message` (было `None`). Досланные
deny-шаги несут «Вопрос пользователя», консистентно с соседними tool-шагами on_tool_end.
Аннотация `step_input: str` в `_send_cycle_feedback` НЕ менялась — теперь она честна
(str всегда).

### H4: явный флаг в тесте

`test_send_unsent_empty_verdicts_noop(monkeypatch)` — первой строкой
`monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_ENABLED", True)`, консистентно
с остальными wiring-тестами (без флага `build_internet_judge_config` может вернуть None
при выключенном окружении).

## Команды и результаты

| # | Команда (venv общий) | Результат |
|---|----------------------|-----------|
| 1 | baseline wiring: `pytest tests/test_deepagent_internet_judge_wiring.py -v` (до правок) | **19 passed** |
| 2 | baseline регресс: `pytest tests/test_deepagent_internet_judge.py tests/test_deepagent_interceptor_async.py tests/test_deepagent_session_id.py` | **36 passed** |
| 3 | GREEN суммарно (после правок): все 4 файла одним прогоном | **55 passed** (6.1–7.3 s, 87 warnings — pre-existing Pydantic-deprecation шум) |
| 4 | финальный прогон на закоммиченном состоянии | **55 passed**, дерево чистое |
| 5 | `ruff check streaming.py` WORK vs HEAD (через python-скрипт с `--stdin-filename`) | **HEAD 8 = WORK 8**, идентичный список: I001 + 7×F401 (импорт-блок, строки 8–70); **0 новых**; wiring-тест — 0 |
| 6 | смоук `pytest --collect-only -q` wiring-теста | **19 tests collected** |

Примечание к п. 5: прямой пайп `git show … | ruff check -` заблокирован permission-политикой
GigaCode — сверка выполнена python-скриптом через subprocess с `--stdin-filename`
(обход по правилу GIGACODE.md). Список предсуществующих: I001 (un-sorted imports),
F401 ×7 (AIMessage/HumanMessage/ToolMessage/finalize_stream/report_pass_through_error/
EventType/AsyncGenerator) — все вне зон правки.

## Self-review

- **Все три точки выхода покрыты?** Да — (а), (б), (в) перечислены выше; других
  завершающих путей `generate_agent_stream` нет: терминальный return в pptx-ветке,
  аварийный return 7.7, нормальное падение в конец функции (секция 9, cleanup).
  Exception-пути генератора (pass-through/fallback) живут в `direct_streaming.py` /
  `streaming_errors.py` и не относятся к циклу astream_events — там mcp_internet-judge
  gate отсутствует by design.
- **Порядок определений не ломает замыкания?** `_send_cycle_feedback` — модульная функция
  (строка 188 < 331), определена до `generate_agent_stream`. Захваченные локалы:
  `_tool_step_sequence` (572), `_cycle_start_dt` (573), `_sent_judge_call_ids` (578),
  параметры функции (chat_id, chat_message_id, request, user_token, interceptor_config) —
  все связаны до точки вставки определений (~609). `nonlocal` корректен: переменная
  присвоена в охватывающей функции до объявления вложенной.
- **Двойной flush невозможен:** точки (а)/(в) завершают генератор немедленно после
  вызова, точка (б) — единственная на нормальном пути; пути взаимоисключающие.
- **Телеметрия не ломает стрим:** `_send_cycle_feedback` глотает исключения (by design),
  `send_unsent_judge_verdicts` изолирует сбой отдельного sender-вызова (юнит
  `test_send_unsent_sender_failure_is_isolated`).
- **Замечание брифа о `_send_cycle_feedback` как возможной локальной функции** —
  проверено: она модульная, порядок безопасен, переразмещение не потребовалось.

## Тестовое покрытие (решение диспетчера соблюдено)

Тяжёлых интеграционных тестов на `generate_agent_stream` НЕ писалось (зависимости
не мокабельны по конвенции зоны; механика `send_unsent_judge_verdicts` уже покрыта
юнитами: возврат счёта, фильтрация sent-ids, изоляция сбоев — тесты
`test_send_unsent_*`). Перенос и `+=` верифицированы ревью по диффу.

## Сомнения

Существенных нет. Два малых замечания для протокола:

1. В точке (а) flush идёт ПОСЛЕ `yield MessageResponse(...)` — т.е. клиент получает
   файловый ответ раньше досылаемых телеметрических шагов. Это соответствует брифу
   («перед `cleanup(...); return`») и не влияет на корректность телеметрии.
2. Финальный message-шаг секции 8 отправляется только при непустом `text_for_db`
   (pre-existing условие); при пустом ответе финального шага не будет и после fix2 —
   поведение зоны не менялось.

## Итог

Все четыре находки повторного ревью закрыты: H1 (+ ось A MISSING), H2, H3, H4.
55 passed (19 wiring + 36 регресс), ruff 0 новых, смоук пройден. Коммит c134b2ea
поверх 31da4c6f, дерево чистое, застейджены только 2 файла задачи.
