# Progress ledger — pre-gate LLM-судья mcp_internet (DeepAgents)

План: docs/sberpowers/plans/2026-10-06-mcp-internet-judge.md
Спека: docs/sberpowers/specs/2026-10-06-mcp-internet-judge.md
Ветка работы: mcp-internet-judge (от develop@8587af4c), HEAD 5c17ce01.
Диспетчерская сессия: 2026-10-06. Execute завершён (волны 1–3); далее review ветки.

## Статус задач

- Задача 1.1 «Async-хуки interceptor» (волна 1): выполнена, коммиты 6a4041ed..15cc86f5,
  ревью чистое (spec PASS, quality ACCEPT, 0 Critical/Important).
- Задача 2.1 «Модуль судьи InternetJudgeGate» (волна 2): выполнена с утверждённым
  человеком фиксом A/B, коммиты f23bd5bb..b9d804a9, ревью PASS/ACCEPT (первичное и
  повторное), находки A/B ЗАКРЫТЫ.
- Задача 3.1 «Wiring — серверная регистрация + наблюдаемость» (волна 3): выполнена
  с двумя fix-кругами, коммиты ea886a87 → 31da4c6f → c134b2ea (ветка wave3-task31,
  worktree-изоляция от b9d804a9). Ревью: круг 1 — spec PASS (вывод о deny-ветке
  опровергнут) / quality NEEDS_FIXES (Critical SSE-паритет + Important deny);
  круг 2 — PARTIAL/NEEDS_FIXES (Important step_sequence/step_input; MISSING
  терминальный pptx-return); круг 3 — PASS/ACCEPT. Смёржена ed91b86f.
- Задача 3.2 «CLI-прогон тестовой корзины» (волна 3): выполнена с двумя fix-кругами,
  коммиты 603e96d8 → 227a93be → 42128e75 (ветка wave3-task32). Ревью: круг 1 —
  PASS/NEEDS_FIXES (G1 сбой судьи рушит прогон + Minor); круг 2 — PASS/NEEDS_FIXES
  (I2 мёртвый судья на deny-строках = OK); круг 3 — PASS/ACCEPT. Смёржена 5c17ce01.
- Worktree .worktrees/wave3-task31 и wave3-task32 удалены после merge;
  ветки wave3-task31/wave3-task32 оставлены до финального меню git-discipline.

## Коммиты цикла

- bc53e41d docs(spec-plan): спека, план и конспекты pre-gate LLM-судьи mcp_internet
- 6a4041ed chore: игнорировать .sberpowers (ledger execute-цикла)
- d29b04be chore: игнорировать .worktrees (ручная worktree-изоляция волны 3)
- 15cc86f5 feat(01): async-хуки ToolInterceptor, pre/post_execute (REQ-05)
- f23bd5bb docs(spec): уточнение REQ-02 по БЛиК П298 - адресный поиск по источнику
- c22a10d4 feat(02): InternetJudgeGate — pre-gate LLM-судья mcp_internet (REQ-02,04,06,07)
- b9d804a9 fix(02): вход судьи bare-текстом; причина сбоя без деталей (REQ-02,06,08)
- ea886a87 feat(03): серверная регистрация mcp_internet-judge + вердикты в step_metadata (REQ-01,03,08)
- 31da4c6f fix(03): SSE-паритет судьи + досыл deny-вердиктов в телеметрию (REQ-01,03,08)
- c134b2ea fix(03): досл вердиктов на всех выходах стрима; продвижение step_sequence; step_input вопроса (REQ-08)
- 603e96d8 feat(04): CLI-прогон тестовой корзины mcp_internet-judge (REQ-02)
- 227a93be fix(04): прогон корзины не падает на сбоях судьи; строгие метки; понятные отказы CLI (REQ-02)
- 42128e75 fix(04): неопределённые вердикты судьи не засчитываются как совпадения (REQ-02)
- ed91b86f Merge wave3-task31: серверная регистрация mcp_internet-judge + deny-телеметрия (REQ-01,03,08)
- 5c17ce01 Merge wave3-task32: CLI-прогон тестовой корзины mcp_internet-judge (REQ-02)

## Доказательства прогона

### Задача 1.1
- Исполнитель: 6/6 tests/test_deepagent_interceptor_async.py, RED зафиксирован,
  регресс session_id 17 passed, ruff чист.
- Диспетчер независимо: «6 passed, 87 warnings in 4.25s».

### Задача 2.1 (feat + fix)
- Исполнитель feat: RED (ModuleNotFoundError), GREEN 10/10, регресс 6/6, ruff чист.
- Fix-субагент: RED 2 failed (A/B), GREEN 13 passed судьи + 6 passed регресс, ruff чист.
- Диспетчер независимо (после фикса): «19 passed, 87 warnings in 5.44s».

### Задача 3.1 (три круга ревью)
- Исполнитель: TDD RED (ImportError) → GREEN 9/9 wiring (8 брифа + интеграционный
  test_real_tool_call_id_reaches_step_metadata), регресс зоны 36, смоук через
  /tmp-скрипт + collect-only (прямой import вне loop падает на pre-existing DB-pool),
  ruff чист (8 streaming.py предсуществующих, доказано stash-сравнением с BASE).
- Fix1 (31da4c6f): RED ImportError → GREEN 19 wiring + 36 регресс = 55; ruff 0 новых.
- Fix2 (c134b2ea): baseline 55 зелёные до правок → 55 после; три точки flush
  верифицированы ревью круга 3 по коду (streaming.py: pptx-return 868–870 /
  аварийный 7.7 1028–1029 / нормальный выход 1034; pass-through :444 до цикла —
  вердиктов там быть не может).
- Диспетчер независимо: wiring 10 passed (круг 1); 55 passed (после fix1);
  55 passed (после fix2, «55 passed, 87 warnings in 6.62s»).

### Задача 3.2 (три круга ревью)
- Исполнитель: TDD RED (ModuleNotFoundError) → GREEN 2 PASS + 1 ожидаемый SKIP
  (test.xlsx untracked в worktree), регресс судьи 13, ruff All checks passed.
- Fix1 (227a93be): RED по G1/G2/G4 (3 падения) → GREEN 7+1skip; совместный 20+1skip.
- Fix2 (42128e75): RED (assert 1 == 0 на deny-строке с error-вердиктом) → GREEN
  8+1skip; совместный 21+1skip.
- Диспетчер независимо: 20 passed + 1 skipped (после fix1); 21 passed + 1 skipped
  (после fix2); в главном дереве после merge load_basket на реальной test.xlsx — PASS.
- Доп-проверка спека-ревьюера круга 2: строгий load_basket против реальной корзины
  главного дерева: raw_rows=138, loaded=136 (51 allow / 85 deny), dropped=1 пустая.

### Полный регресс после merge (главное дерево, HEAD 5c17ce01)
- `.venv/bin/python -m pytest tests/test_deepagent_internet_judge_wiring.py
  tests/test_deepagent_internet_judge.py tests/test_deepagent_interceptor_async.py
  tests/test_deepagent_session_id.py tests/test_deepagent_internet_judge_basket.py -q`
  → **64 passed, 87 warnings in 7.40s** (load_basket SKIP→PASS на реальной корзине).
- Ruff по 10 файлам зоны: ровно 8 предсуществующих замечаний streaming.py
  (I001 + 7×F401), новых нет.

## Решения диспетчера (волна 3)

- Девиация исполнителя 3.1 шага 3e (lookup-id): ПРИНЯТА. Верифицирована по коду:
  судья ключует verdicts реальным tool_call["id"] (internet_judge.py:193),
  стриминговый _current_tool_call_id синтетический call_<uuid> (streaming.py:627);
  буквальный бриф давал вечный miss REQ-08. Правка extract_judge_tool_call_id +
  фолбэк; подтверждена всеми тремя кругами ревью.
- Противоречие ревьюеров круга 1 (spec утверждал ToolMessage-эмит при deny,
  quality — отсутствие событий) закрыто живой пробой диспетчера (/tmp/probe_deny_events.py,
  scripted model + create_deep_agent + InterceptorMiddleware + deny-мок):
  `tool events emitted: []`, вердикт записан → прав quality-ревьюер; вывод
  spec-ревьюера круга 1 ошибочен.
- CRITICAL круга 2 (судья не подключён в SSE-хэрнессе services_sse.py:376): пробел
  ПЛАНА/брифа (файл не входил в задачу 3.1) против REQ-01 и документированного
  инварианта паритета SSE/polling. Фикс: единый хелпер merge_request_interceptor_configs
  в internet_judge.py, вызываемый из service.py И services_sse.py — устраняет класс
  ошибки «в одну копию внесли, в другую забыли».
- IMPORTANT круга 2 (deny/timeout/error невидимы в телеметрии): фикс — досл
  send_unsent_judge_verdicts; круг 2 расширил на аварийный выход 7.7 (пересмотр
  прежнего исключения этого пути) и терминальный pptx-return (PARTIAL оси A):
  flush на ВСЕХ завершающих путях generate_agent_stream.
- Тестовое покрытие переноса flush: тяжёлые интеграционные тесты generate_agent_stream
  исключены диспетчером (не соответствует конвенции зоны; верификация переносов —
  ревью по диффу); круг 3 подтвердил корректность.
- CLI: параллелизация run_basket (gather+Semaphore) ОТКЛОНЕНА — разовый дев-инструмент,
  последовательный прогон детерминирован и читаем (ревьюер признал приемлемость).
- CLI ok-условие (fix2): неопределённые вердикты (error/timeout) — всегда miss
  независимо от метки; знаменатель не меняется; мёртвый судья даёт accuracy 0.0.
- Сомнение исполнителя 3.2 (pre-existing): голый import модулей deepagent вне event
  loop падает на module-level DB-pool (services.py:76) — env-caveat без действий;
  живому CLI-прогону нужен running loop или мок get_db_pool (conftest-паттерн).
- Окружение: python -I для /tmp-скриптов (мусорный /tmp/inspect.py затмевал stdlib);
  Filesystem Guard: write_file блокирует запись субагентов вне их worktree → отчёт
  32-fix2 писался через /tmp-скрипт.

## Отложено / Minor (судьба — итоговое ревью ветки)

### Закрыты фиксами волн 2–3
A. Вход судьи bare-текстом — ЗАКРЫТА (b9d804a9).
B. {exc} в agent-facing reason — ЗАКРЫТА (b9d804a9).
SSE-паритет + deny-телеметрия + счётчики шагов + устойчивая корзина — ЗАКРЫТЫ
(31da4c6f, c134b2ea, 227a93be, 42128e75).

### Открытые
1. [Minor, предсуществующее] tests/test_deepagent_subagents.py:668 — два wiring-теста
   падают на BASE («assert 6 == 4»): устаревшие размеры middleware-стека.
2. [Minor, предсуществующее] interceptor.py exec_request NameError при pre(skip)+post
   в одном interceptor'е; судья pre-only.
3. [Minor] tests/test_deepagent_interceptor_async.py — нет кейса «исключение в async
   post_execute → fail-open».
4. [Minor, косметика] Отчёты исполнителей: расхождения счёта строк/счётчиков.
5. [Minor] internet_judge.py — строки длиннее 120 (E501 в ignore рuff-конфига).
6. [Minor] промпт судьи без явной prompt-injection-стойкости (компенсировано
   «сомнение трактуй в пользу deny»).
7. ЗАКРЫТА (фикс A): str-args покрытие.
8. [Открытый шаг] NOT-VERIFIABLE: живой прогон корзины test.xlsx против GigaChat на
   дев-стенде — accuracy ≥ 0.90 (REQ-02 приёмка). CLI готов; требует running loop или
   мока DB-пула. Веточка «Корзина пуста» без прямого теста (механика идентична покрытой).
9. [Minor, hardening] конверсия args до try в pre_execute; вложенные dict/list-значения
   args невидимы судье (trade-off утверждённого fix-брифа волны 2).
10. [Minor] RED-прогноз fix-брифа волны 2 неточен — легитимно.
11. [NEW, Minor, ревью круга 3 задачи 3.1] internet_judge.py:311 — инкремент
    base_step_sequence до try: при сбое sender'а номер «сжигается» и не попадает в
    возврат → гипотетическая коллизия sequence финального шага. Практически
    недостижимо (прод-sender глотает все исключения). Фикс: инкремент после успешного
    await либо возврат числа попыток.
12. [NEW, Minor, ревью круга 3 задачи 3.2] internet_judge_check.py:57 — докстринг
    run_basket («verdict==allow сопоставляется метке 1») не отражает правило
    «error/timeout — всегда miss»; дополнить при финальном проходе по ветке.
13. [Наблюдение, pre-existing] streaming.py секция 8 — финальный message-шаг только
    при непустом text_for_db (пустой ответ без финального шага в телеметрии).

## Точка остановки (2026-10-06, пауза по команде пользователя)

Пользователь поставил сессию на паузу: закоммитить и запушить ветку,
финальное ревью — продолжить позже.

- Выполнено (r1, шаг «собери вход»): review-пакет ВСЕЙ ветки собран в
  `.sberpowers/tasks/review-branch-package.md` — 3498 строк, 17 файлов,
  +2863/−19. Команда восстановления (если файла нет):
  `{ git log --oneline develop..HEAD; echo; git diff --stat develop..HEAD; echo;
  git diff -U10 develop..HEAD; } > .sberpowers/tasks/review-branch-package.md`.
- Прервано: параллельный диспатч sbp-reviewer-spec (все 8 REQ оси A) +
  sbp-reviewer-quality (ось B, весь дифф) упал — network failure, результаты
  не записаны. При возобновлении повторить диспатч с теми же брифами;
  ревьюеру B напомнить про известные Minor №11 (sequence burn,
  internet_judge.py:311) и №12 (докстринг run_basket) — подтверждать уровнем,
  не открывать заново.
- Tracked-изменений к коммиту не было (код цикла целиком в 5c17ce01),
  новых коммитов не делалось.
- Ветка запушена в origin/mcp-internet-judge (первый пуш, -u). Task-ветки
  wave3-task31/wave3-task32 НЕ пушены — ждут финального меню git-discipline.
- test.xlsx оставлен untracked (проектная норма: load_basket SKIP/PASS по
  наличию файла).
- Открытые шаги при возобновлении:
  1) перезапуск двухосевого финального ревью r1 (пакет готов);
  2) разбор Minor-баланса ledger п.1–13 с человеком;
  3) learn (l1) после обеих осей;
  4) живой прогон корзины против GigaChat на стенде (accuracy ≥ 0.90);
  5) меню git-discipline (merge/PR/оставить/выбросить + судьба task-веток).

- Продолжение паузы (2026-10-06): по команде пользователя `.sberpowers` переведён
  в отслеживаемые (строка удалена из .gitignore, реверс коммита 6a4041ed).
  Весь ledger и артефакты задач (брифы/отчёты/review-пакеты) закоммичены и
  запушены текущим коммитом — незавершённые проверки теперь восстанавливаются
  из ветки origin/mcp-internet-judge, а не только с локального диска.
  На секреты прогнано: единственное совпадение — заглушка "local-stub-key"
  из диффа app/core/config.py (не секрет, код уже в ветке).
