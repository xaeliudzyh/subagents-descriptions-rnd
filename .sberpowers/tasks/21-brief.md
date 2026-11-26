# Бриф задачи 2.1: Модуль судьи InternetJudgeGate (волна 2)

- Ветка: mcp-internet-judge; BASE задачи: f23bd5bb
- REQ: REQ-mcp-internet-judge-02, REQ-mcp-internet-judge-04, REQ-mcp-internet-judge-06, REQ-mcp-internet-judge-07
- Blocked by: задача 1.1 (выполнена: ToolInterceptor принимает async pre_execute)

## Глобальные ограничения плана (дословно)

- deepagents==0.7.14 — версию не трогаем (`requirements/base.txt:41`).
- Python 3.13, venv проекта `.venv`; тесты запускать
  `.venv/bin/python -m pytest <путь> -v` из корня репо.
- Ruff: line length 120, type hints обязательны (`pyproject.toml`).
- Конфигурация — только поля `AppSettings` в `app/core/config.py` (значения в
  коде); .env не используем.
- Тесты без живого LLM: GigaChat всегда мок (патчим фабричный метод gate).
- Один таск → один атомарный коммит: `feat(02): InternetJudgeGate — pre-gate LLM-судья mcp_internet (REQ-02,04,06,07)`.
- Стейджить ТОЛЬКО файлы задачи (3 файла ниже). Посторонние untracked-файлы
  пользователя в дереве — не трогать. Предсуществующие падения в
  tests/test_deepagent_subagents.py и/или stateful_routing-зоне известны
  диспетчеру (падают уже на BASE) — НЕ чинить, вне зоны задачи.

## Разрешённые диспетчером неоднозначности

- Ruff-канонизация: код-сниппеты этого брифа транскрибируй дословно по логике,
  но импорты/аннотации канонизируй под ruff как в задаче 1.1 (в сниппетах
  есть НЕиспользуемые импорты — например `inspect` и `Coroutine` в
  internet_judge.py: убери неиспользуемые, F401 недопустим).
- Async-тесты: `@pytest.mark.asyncio` + `async def test_...` (конвенция
  проекта, pytest-asyncio работает).
- Проверка из ревью 1.1 (NOT-VERIFIABLE, закрывается здесь): хук судьи НЕ
  блокирует event loop — в коде судьи не должно быть `run_async_sync` или
  синхронного LLM-клиента; весь путь асинхронный (`asyncio.wait_for`,
  `await llm.ainvoke`). Отрази в self-review.
- Known-issue interceptor.py:297 (`exec_request` NameError при pre(skip)+post)
  тебя НЕ касается: судья регистрирует только pre_execute, post_execute у него нет.

## Контекст задачи

LLM-судья по БР «Поиск внешнего контента» (процесс БЛиК П298 «Поиск внешних
образовательных ресурсов в Пульс»): наружу допускается только поиск обучения
как обогащение каталога Пульса — три потребности: (1) внешнее обучение и
мероприятия развития; (2) адресный поиск по желаемому источнику («курс Яндекса
по управлению продуктом»); (3) объединение внешних предложений с внутренними
рекомендациями Пульса. Рандомный websearch и запросы с ПДн блокируются.
Тестовая корзина `test.xlsx` в корне репо (лист Sheet1: колонки A=query,
B=«Должен ли вызываться mcp_internet?», 51 allow / 85 deny) задаёт эталон;
граничные пары обязаны воспроизводиться. Fail-closed: любой сбой судьи (ошибка
LLM, таймаут, невалидный ответ) блокирует вызов — исключения ловятся ВНУТРИ
pre_execute (внешний except interceptor'а — fail-open, на судью не рассчитываем).

## Файлы

- Изменить: `app/core/config.py` (поля AppSettings)
- Создать: `app/services/react/deepagent/internet_judge.py`
- Создать: `tests/test_deepagent_internet_judge.py`

## Интерфейсы

- Потребляет:
  - `ToolInterceptor(pre_execute=...)` и `ToolOverrideResult` из
    `app/services/react/deepagent/interceptor.py` — после задачи 1.1 pre_execute
    принимает coroutine-callable (вызывается с await).
  - `GigachatAsyncClient.get_client(model=..., max_tokens=..., temperature=...)`
    из `app/services/react/proxy_client.py` — ленивый импорт внутри метода
    (паттерн app/phoenix/ai_provider/openai_compatible.py:97-107: content может
    быть list блоков → конкатенация .get("text", "")).
  - `app_settings` из `app.core.config`.
- Производит (на это опирается задача 3.1; сигнатуры сверь посимвольно):
  - `@dataclass(frozen=True) class InternetJudgeVerdict: verdict: Literal["allow","deny","timeout","error"]; reason: str` c методом `to_metadata() -> dict[str, Any]` → `{"internet_judge": {"verdict": ..., "reason": ...}}`
  - `class InternetJudgeGate: __init__(self, chat_id: str, message_id: str, user_question: str)`; атрибут `verdicts: dict[str, InternetJudgeVerdict]` (ключ — `tool_call_id`); `async pre_execute(self, request: ToolCallRequest) -> ToolCallRequest | ToolOverrideResult | None`; `async judge(self, query: str) -> InternetJudgeVerdict`; `async _call_llm(self, messages: list[dict[str, str]]) -> str` (мок-точка тестов); `_parse_verdict(content: str) -> tuple[str, str]` (статик: `"allow"|"deny"`, reason; ValueError при невалидном)
  - `def build_internet_judge_messages(query: str, user_question: str) -> list[dict[str, str]]`

## Шаги

1. Добавь поля в `app/core/config.py` в `AppSettings` (после блока
   `CUSTOM_PROVIDER_*`, соблюдая стиль соседних секций):

```python
    # --- MCP internet judge (pre-gate LLM-judge для mcp_internet) ---
    # Управление правкой config.py; default ON — дев-ИФТ стенд прогоняет перед продом.
    MCP_INTERNET_JUDGE_ENABLED: bool = True
    MCP_INTERNET_JUDGE_MODEL: str = "GigaChat-3-Lightning"
    MCP_INTERNET_JUDGE_TIMEOUT_SECONDS: float = 15.0
```

2. Напиши падающий тест `tests/test_deepagent_internet_judge.py`:

```python
"""InternetJudgeGate: pre-gate судья mcp_internet. LLM всегда замокан."""

import asyncio

import pytest
from langchain_core.messages import ToolMessage
from langgraph.prebuilt.tool_node import ToolCallRequest

from app.core.config import app_settings
from app.services.react.deepagent.internet_judge import (
    InternetJudgeGate,
    InternetJudgeVerdict,
    build_internet_judge_messages,
)


def _gate() -> InternetJudgeGate:
    return InternetJudgeGate(chat_id="chat-1", message_id="msg-1", user_question="Хочу обучиться Java")


def _request(call_id: str = "call_9") -> ToolCallRequest:
    return ToolCallRequest(
        tool_call={"name": "mcp_internet", "args": {"query": "курс по Java"}, "id": call_id},
        tool=None,
        state={},
        runtime=None,
    )


# ── _parse_verdict ──────────────────────────────────────────────────────────


def test_parse_verdict_plain_json():
    assert InternetJudgeGate._parse_verdict('{"verdict": "allow", "reason": "ок"}') == ("allow", "ок")


def test_parse_verdict_markdown_wrapped():
    raw = '```json\n{"verdict": "deny", "reason": "не обучение"}\n```'
    assert InternetJudgeGate._parse_verdict(raw) == ("deny", "не обучение")


def test_parse_verdict_invalid_raises():
    with pytest.raises(ValueError):
        InternetJudgeGate._parse_verdict("судья передумал и молчит")


# ── build messages (REQ-06) ────────────────────────────────────────────────


def test_messages_contain_query_and_user_question():
    msgs = build_internet_judge_messages("курс по Java", "Хочу обучиться Java")
    joined = "\n".join(m["content"] for m in msgs)
    assert "курс по Java" in joined
    assert "Хочу обучиться Java" in joined
    assert msgs[0]["role"] == "system"


# ── pre_execute: allow / deny / timeout / error (REQ-02, REQ-04) ───────────


@pytest.mark.asyncio
async def test_allow_proceeds(monkeypatch):
    gate = _gate()

    async def fake_llm(messages):
        return '{"verdict": "allow", "reason": "это поиск обучения"}'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    result = await gate.pre_execute(_request())
    assert result is None  # proceed unchanged
    assert gate.verdicts["call_9"] == InternetJudgeVerdict("allow", "это поиск обучения")


@pytest.mark.asyncio
async def test_deny_blocks_with_reason(monkeypatch):
    gate = _gate()

    async def fake_llm(messages):
        return '```json\n{"verdict": "deny", "reason": "прогноз погоды не связан с обучением"}\n```'

    monkeypatch.setattr(gate, "_call_llm", fake_llm)
    result = await gate.pre_execute(_request())
    assert result is not None and hasattr(result, "content")
    assert result.status == "error"
    assert "погода" in result.content  # причина агенту
    assert gate.verdicts["call_9"].verdict == "deny"


@pytest.mark.asyncio
async def test_timeout_fails_closed(monkeypatch):
    gate = _gate()
    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_TIMEOUT_SECONDS", 0.05)

    async def slow_llm(messages):
        await asyncio.sleep(0.5)
        return '{"verdict": "allow"}'

    monkeypatch.setattr(gate, "_call_llm", slow_llm)
    result = await gate.pre_execute(_request())
    assert result.status == "error"
    assert "судья недоступен" in result.content.lower() or "таймаут" in result.content.lower()
    assert gate.verdicts["call_9"].verdict == "timeout"


@pytest.mark.asyncio
async def test_llm_error_fails_closed(monkeypatch):
    gate = _gate()

    async def broken_llm(messages):
        raise RuntimeError("gigachat down")

    monkeypatch.setattr(gate, "_call_llm", broken_llm)
    result = await gate.pre_execute(_request())
    assert result.status == "error"
    assert gate.verdicts["call_9"].verdict == "error"


@pytest.mark.asyncio
async def test_invalid_llm_output_fails_closed(monkeypatch):
    gate = _gate()

    async def garbage_llm(messages):
        return "не JSON вообще"

    monkeypatch.setattr(gate, "_call_llm", garbage_llm)
    result = await gate.pre_execute(_request())
    assert result.status == "error"
    assert gate.verdicts["call_9"].verdict == "error"


def test_verdict_to_metadata():
    v = InternetJudgeVerdict("deny", "причина")
    assert v.to_metadata() == {"internet_judge": {"verdict": "deny", "reason": "причина"}}
```

3. Запусти `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v`
   — ожидай FAIL: `ModuleNotFoundError: No module named 'app.services.react.deepagent.internet_judge'`.
   (TDD RED — зафиксируй падающий вывод.)

4. Реализуй `app/services/react/deepagent/internet_judge.py` целиком (логика
   дословно; импорты канонизируй под ruff — см. выше):

```python
"""Pre-gate LLM-judge для вызовов mcp_internet (DeepAgent harness).

Судья по БР «Поиск внешнего контента» (процесс БЛиК П298): наружу допускается
только поиск обучения (обогащение каталога Пульса) — курсы/книги/видео/статьи/
конференции/вебинары по рабочей теме и адресный поиск у источников; произвольный
websearch и запросы с ПДн блокируются.

Fail-closed: любой сбой судьи (ошибка LLM, таймаут, невалидный ответ) блокирует
вызов mcp_internet — агенту возвращается синтетический tool-result со статусом
error и человекочитаемой причиной. Исключения ловятся здесь, а не в общем
fail-open перехватчике InterceptorMiddleware.
"""

from __future__ import annotations

import asyncio
import json
import logging
from dataclasses import dataclass, field
from typing import TYPE_CHECKING, Any, Literal

from app.core.config import app_settings

if TYPE_CHECKING:
    from langgraph.prebuilt.tool_node import ToolCallRequest

    from app.services.react.deepagent.interceptor import InterceptorConfig, ToolOverrideResult

logger = logging.getLogger(__name__)

_JUDGE_SYSTEM_PROMPT = """\
Ты — предикат для инструмента интернет-поиска корпоративного HR-ассистента \
(скилл «Поиск внешнего контента», процесс БЛиК П298 «Поиск внешних \
образовательных ресурсов в Пульс»).

Инструмент mcp_internet допускается ТОЛЬКО как обогащение поиска обучения по \
каталогу платформы Пульс и закрывает три потребности: (1) поиск внешнего \
обучения и мероприятий развития — курсы, книги, видео, статьи, вебинары, \
митапы, конференции, образовательные программы, обучение в Корпоративном \
университете / Сбер университете; (2) адресный поиск по желаемому источнику \
(«курс Яндекса по управлению продуктом»); (3) объединение внешних предложений \
с внутренними рекомендациями Пульса.

Произвольный поиск в интернете НЕ допускается и блокируется (deny): новости, \
погода, курсы валют, бытовые вопросы, финансы и ставки, спорт, развлечения, \
справочная информация, конкурентный анализ, а также ЛЮБЫЕ запросы с \
персональными данными людей (ФИО вместе с ИНН, телефоном, паспортом и т.п.).

Правила разрешения сомнений:
- Запрос ищет образовательный или учебный материал (в т.ч. статью, конференцию \
как материал для развития) по рабочей теме — allow, даже если тема узкая \
(антифрод, цифровой рубль, CRM).
- Адресный поиск у конкретного источника/бренда — allow, только если искомое — \
образовательный ресурс («курс Яндекса по управлению продуктом» — allow; \
«тарифы на эквайринг у Сбера и ВТБ» — deny: бренд без образовательной цели).
- Запрос про новости, справку, состояние рынка или быт — deny, даже если слова \
похожи («новости о цифровом рубле» — deny, «как подключить бизнес к цифровому \
рублю» в контексте обучения — allow).
- «Найти книгу/курс по теме X» — allow; «какие новые книги вышли» (новостной \
обзор) — deny.
- Контекст (вопрос пользователя) помогает понять, что сценарий — поиск \
обучения; сомнение трактуй в пользу deny.

Ответь СТРОГО JSON одной строкой без пояснений:
{"verdict": "allow" | "deny", "reason": "<краткое обоснование по-русски>"}
"""

# Few-shot пары из тестовой корзины test.xlsx (Sheet1), отобраны как
# representatives границ разметки; адресная пара — из БР (БЛиК П298).
_FEW_SHOT_EXAMPLES: list[tuple[str, str]] = [
    ("найди курс по анализу данных", '{"verdict": "allow", "reason": "поиск курса — обучение"}'),
    ("курс Яндекса по управлению продуктом", '{"verdict": "allow", "reason": "адресный поиск курса у конкретного источника — обучение"}'),
    ("обучение в КУ информационной безопасности", '{"verdict": "allow", "reason": "обучение в Корпоративном университете"}'),
    ("конференция по Java для финтеха", '{"verdict": "allow", "reason": "конференция по рабочей теме — материал развития"}'),
    ("статья по антифрод-стратегиям", '{"verdict": "allow", "reason": "профессиональная статья как обучающий материал"}'),
    ("найди книгу по микроэкономике для продакт-менеджеров", '{"verdict": "allow", "reason": "поиск книги — обучение"}'),
    ("как подключить бизнес к Цифровому рублю", '{"verdict": "allow", "reason": "рабочая тема в образовательном сценарии"}'),
    ("Какая погода ожидается в Москве завтра?", '{"verdict": "deny", "reason": "бытовой вопрос, не обучение"}'),
    ("Какие новости о цифровом рубле?", '{"verdict": "deny", "reason": "новости, не поиск обучения"}'),
    ("Какие новые книги вышли в этом месяце?", '{"verdict": "deny", "reason": "новостной обзор, не поиск конкретного обучения"}'),
    ("Какие советы по инвестициям дают аналитики?", '{"verdict": "deny", "reason": "справочный вопрос, не обучение"}'),
    ("Кто выиграл последний футбольный матч?", '{"verdict": "deny", "reason": "спорт, не обучение"}'),
    ("Какие тарифы на эквайринг у Сбера и ВТБ?", '{"verdict": "deny", "reason": "бизнес-справка, бренд без образовательной цели"}'),
    ("Герман Греф ИНН 0709306824?", '{"verdict": "deny", "reason": "персональные данные"}'),
]

_DENY_TITLE = "Поиск в интернете заблокирован судьёй проверки назначения"
_FAIL_CLOSED_TITLE = "Запрос в интернет заблокирован: LLM-судья недоступен"


@dataclass(frozen=True)
class InternetJudgeVerdict:
    """Вердикт судьи по одному вызову mcp_internet."""

    verdict: Literal["allow", "deny", "timeout", "error"]
    reason: str

    def to_metadata(self) -> dict[str, Any]:
        return {"internet_judge": {"verdict": self.verdict, "reason": self.reason}}


def build_internet_judge_messages(query: str, user_question: str) -> list[dict[str, str]]:
    """Промпт судьи: system + few-shot + пара «запрос тула / вопрос пользователя»."""
    messages: list[dict[str, str]] = [{"role": "system", "content": _JUDGE_SYSTEM_PROMPT}]
    for example_query, example_answer in _FEW_SHOT_EXAMPLES:
        messages.append({"role": "user", "content": f"Запрос тула: {example_query}\nВопрос пользователя: (не задан)"})
        messages.append({"role": "assistant", "content": example_answer})
    messages.append(
        {
            "role": "user",
            "content": f"Запрос тула: {query}\nВопрос пользователя: {user_question or '(не задан)'}",
        }
    )
    return messages


@dataclass
class InternetJudgeGate:
    """Per-request судья: pre_execute-хук + реестр вердиктов по tool_call_id."""

    chat_id: str
    message_id: str
    user_question: str
    verdicts: dict[str, InternetJudgeVerdict] = field(default_factory=dict)

    @staticmethod
    def _parse_verdict(content: str) -> tuple[str, str]:
        """LLM-ответ -> ('allow'|'deny', reason). Raise ValueError если не JSON."""
        text = content.strip()
        if text.startswith("```"):
            lines = [ln for ln in text.split("\n") if not ln.strip().startswith("```")]
            text = "\n".join(lines).strip()
        data = json.loads(text)
        verdict = str(data.get("verdict", "")).lower()
        if verdict not in ("allow", "deny"):
            msg = f"verdict должен быть allow|deny, получено: {verdict!r}"
            raise ValueError(msg)
        reason = str(data.get("reason") or "")
        return verdict, reason

    async def _call_llm(self, messages: list[dict[str, str]]) -> str:
        """Прямой LLM-вызов GigaChat. Мок-точка для тестов."""
        from app.services.react.proxy_client import GigachatAsyncClient  # лениво: тяжёлый импорт

        llm = GigachatAsyncClient.get_client(
            model=app_settings.MCP_INTERNET_JUDGE_MODEL,
            max_tokens=256,
            temperature=0.0,
        )
        ai_message = await llm.ainvoke(messages)
        content = ai_message.content
        if isinstance(content, list):
            content = "".join(
                b.get("text", "") if isinstance(b, dict) else str(b) for b in content
            )
        return str(content)

    async def judge(self, query: str) -> InternetJudgeVerdict:
        """LLM-вердикт по фактическому поисковому запросу (без блокировки)."""
        messages = build_internet_judge_messages(query, self.user_question)
        try:
            raw = await asyncio.wait_for(
                self._call_llm(messages),
                timeout=app_settings.MCP_INTERNET_JUDGE_TIMEOUT_SECONDS,
            )
        except TimeoutError:
            logger.error(
                "[chat: %s] Internet judge TIMEOUT (%ss) for query: %.200s",
                self.chat_id, app_settings.MCP_INTERNET_JUDGE_TIMEOUT_SECONDS, query,
            )
            return InternetJudgeVerdict("timeout", "LLM-судья не ответил за таймаут")
        verdict, reason = self._parse_verdict(raw)
        return InternetJudgeVerdict(verdict, reason)

    async def pre_execute(
        self, request: ToolCallRequest
    ) -> "ToolCallRequest | ToolOverrideResult | None":
        """pre-gate хук для InterceptorMiddleware. Fail-closed: любая ошибка — блок."""
        from app.services.react.deepagent.interceptor import ToolOverrideResult

        tool_call = request.tool_call
        args = tool_call.get("args", {})
        query = args if isinstance(args, str) else json.dumps(args, ensure_ascii=False)
        call_id = str(tool_call.get("id", ""))

        try:
            verdict = await self.judge(query)
        except Exception as exc:
            logger.error(
                "[chat: %s] Internet judge ERROR (%s); запрос заблокирован (fail-closed)",
                self.chat_id, exc,
            )
            verdict = InternetJudgeVerdict("error", f"сбой LLM-судьи: {exc}")
        self.verdicts[call_id] = verdict

        if verdict.verdict == "allow":
            logger.info("[chat: %s] Internet judge ALLOW: %.200s", self.chat_id, query)
            return None

        if verdict.verdict == "deny":
            logger.warning("[chat: %s] Internet judge DENY: %.200s — %s", self.chat_id, query, verdict.reason)
            content = f"{_DENY_TITLE}: {verdict.reason}"
        else:  # timeout | error
            content = f"{_FAIL_CLOSED_TITLE}. Попробуйте переформулировать запрос или воспользуйтесь другим инструментом."
        return ToolOverrideResult(content=content, status="error")
```

5. Запусти `.venv/bin/python -m pytest tests/test_deepagent_internet_judge.py -v`
   — ожидай PASS всех 10. (TDD GREEN.)

6. Регресс: `.venv/bin/python -m pytest tests/test_deepagent_interceptor_async.py -v`
   — ожидай PASS (задача 1.1 не задета).

7. `.venv/bin/ruff check app/core/config.py app/services/react/deepagent/internet_judge.py tests/test_deepagent_internet_judge.py` — 0 замечаний.

8. Commit: `feat(02): InternetJudgeGate — pre-gate LLM-судья mcp_internet (REQ-02,04,06,07)`
   (стейджить только 3 файла задачи).

## Критерии выхода

- Все тесты судьи зелёные (10 PASS), включая fail-closed по timeout/error/garbage.
- Промпт содержит назначение из БР: три потребности БЛиК П298, адресное правило
  («курс Яндекса» → allow; бренд без образовательной цели → deny) и ≥10 few-shot
  пар (7 allow / 7 deny).
- Конфиг-поля читаются из `AppSettings` (дефолты: True / GigaChat-3-Lightning / 15.0).
- Хук судьи полностью асинхронный (нет run_async_sync, нет синхронного LLM-клиента).
