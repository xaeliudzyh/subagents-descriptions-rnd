b9d804a9 fix(02): вход судьи bare-текстом; причина сбоя без деталей (REQ-02,06,08)
c22a10d4 feat(02): InternetJudgeGate — pre-gate LLM-судья mcp_internet (REQ-02,04,06,07)

 app/core/config.py                             |   7 +
 app/services/react/deepagent/internet_judge.py | 220 +++++++++++++++++++++++++
 tests/test_deepagent_internet_judge.py         | 202 +++++++++++++++++++++++
 3 files changed, 429 insertions(+)

diff --git a/app/core/config.py b/app/core/config.py
index 15795394..7bae131d 100644
--- a/app/core/config.py
+++ b/app/core/config.py
@@ -155,20 +155,27 @@ class AppSettings(AppBaseSettings):
     CLICKHOUSE_SQLALCHEMY_URL: str = ""
     CLICKHOUSE_DATASET_SQL: str = ""
     CLICKHOUSE_DATASET_SQL_FILE: str = ""
     DATASET_FILE_PATH: str = "data/sample_dataset.json"
     DATASET_NAME: str = "sber-judge-dataset"
     PHOENIX_CLIENT_BASE_URL: str = "http://127.0.0.1:6006"
     CUSTOM_PROVIDER_NAME: str = "sber-judge"
     CUSTOM_PROVIDER_BASE_URL: str = "http://127.0.0.1:6006/v1"
     CUSTOM_PROVIDER_API_KEY: str = "local-stub-key"
     CUSTOM_PROVIDER_MODEL: str = "Gigachat-3-Ultra"
+
+    # --- MCP internet judge (pre-gate LLM-judge для mcp_internet) ---
+    # Управление правкой config.py; default ON — дев-ИФТ стенд прогоняет перед продом.
+    MCP_INTERNET_JUDGE_ENABLED: bool = True
+    MCP_INTERNET_JUDGE_MODEL: str = "GigaChat-3-Lightning"
+    MCP_INTERNET_JUDGE_TIMEOUT_SECONDS: float = 15.0
+
     REACT_AGENT_URL: str = "http://localhost:8000/react/api/v1/agent/call"
 
     # --- React proxy router (forwards all react endpoints to a remote host) ---
     REACT_PROXY_TARGET_URL: str = ""
     REACT_PROXY_TIMEOUT: float = 300.0
 
     @computed_field
     @property
     def PHOENIX_SKIP_AUTH(self) -> bool:
         """На dev не используем аутентификацию феникса"""
diff --git a/app/services/react/deepagent/internet_judge.py b/app/services/react/deepagent/internet_judge.py
new file mode 100644
index 00000000..f4de84d1
--- /dev/null
+++ b/app/services/react/deepagent/internet_judge.py
@@ -0,0 +1,220 @@
+"""Pre-gate LLM-judge для вызовов mcp_internet (DeepAgent harness).
+
+Судья по БР «Поиск внешнего контента» (процесс БЛиК П298): наружу допускается
+только поиск обучения (обогащение каталога Пульса) — курсы/книги/видео/статьи/
+конференции/вебинары по рабочей теме и адресный поиск у источников; произвольный
+websearch и запросы с ПДн блокируются.
+
+Fail-closed: любой сбой судьи (ошибка LLM, таймаут, невалидный ответ) блокирует
+вызов mcp_internet — агенту возвращается синтетический tool-result со статусом
+error и человекочитаемой причиной. Исключения ловятся здесь, а не в общем
+fail-open перехватчике InterceptorMiddleware.
+"""
+
+from __future__ import annotations
+
+import asyncio
+import json
+import logging
+from dataclasses import dataclass, field
+from typing import TYPE_CHECKING, Any, Literal
+
+from app.core.config import app_settings
+
+if TYPE_CHECKING:
+    from langgraph.prebuilt.tool_node import ToolCallRequest
+
+    from app.services.react.deepagent.interceptor import ToolOverrideResult
+
+logger = logging.getLogger(__name__)
+
+_JUDGE_SYSTEM_PROMPT = """\
+Ты — предикат для инструмента интернет-поиска корпоративного HR-ассистента \
+(скилл «Поиск внешнего контента», процесс БЛиК П298 «Поиск внешних \
+образовательных ресурсов в Пульс»).
+
+Инструмент mcp_internet допускается ТОЛЬКО как обогащение поиска обучения по \
+каталогу платформы Пульс и закрывает три потребности: (1) поиск внешнего \
+обучения и мероприятий развития — курсы, книги, видео, статьи, вебинары, \
+митапы, конференции, образовательные программы, обучение в Корпоративном \
+университете / Сбер университете; (2) адресный поиск по желаемому источнику \
+(«курс Яндекса по управлению продуктом»); (3) объединение внешних предложений \
+с внутренними рекомендациями Пульса.
+
+Произвольный поиск в интернете НЕ допускается и блокируется (deny): новости, \
+погода, курсы валют, бытовые вопросы, финансы и ставки, спорт, развлечения, \
+справочная информация, конкурентный анализ, а также ЛЮБЫЕ запросы с \
+персональными данными людей (ФИО вместе с ИНН, телефоном, паспортом и т.п.).
+
+Правила разрешения сомнений:
+- Запрос ищет образовательный или учебный материал (в т.ч. статью, конференцию \
+как материал для развития) по рабочей теме — allow, даже если тема узкая \
+(антифрод, цифровой рубль, CRM).
+- Адресный поиск у конкретного источника/бренда — allow, только если искомое — \
+образовательный ресурс («курс Яндекса по управлению продуктом» — allow; \
+«тарифы на эквайринг у Сбера и ВТБ» — deny: бренд без образовательной цели).
+- Запрос про новости, справку, состояние рынка или быт — deny, даже если слова \
+похожи («новости о цифровом рубле» — deny, «как подключить бизнес к цифровому \
+рублю» в контексте обучения — allow).
+- «Найти книгу/курс по теме X» — allow; «какие новые книги вышли» (новостной \
+обзор) — deny.
+- Контекст (вопрос пользователя) помогает понять, что сценарий — поиск \
+обучения; сомнение трактуй в пользу deny.
+
+Ответь СТРОГО JSON одной строкой без пояснений:
+{"verdict": "allow" | "deny", "reason": "<краткое обоснование по-русски>"}
+"""
+
+# Few-shot пары из тестовой корзины test.xlsx (Sheet1), отобраны как
+# representatives границ разметки; адресная пара — из БР (БЛиК П298).
+_FEW_SHOT_EXAMPLES: list[tuple[str, str]] = [
+    ("найди курс по анализу данных", '{"verdict": "allow", "reason": "поиск курса — обучение"}'),
+    (
+        "курс Яндекса по управлению продуктом",
+        '{"verdict": "allow", "reason": "адресный поиск курса у конкретного источника — обучение"}',
+    ),
+    ("обучение в КУ информационной безопасности", '{"verdict": "allow", "reason": "обучение в Корпоративном университете"}'),
+    ("конференция по Java для финтеха", '{"verdict": "allow", "reason": "конференция по рабочей теме — материал развития"}'),
+    ("статья по антифрод-стратегиям", '{"verdict": "allow", "reason": "профессиональная статья как обучающий материал"}'),
+    ("найди книгу по микроэкономике для продакт-менеджеров", '{"verdict": "allow", "reason": "поиск книги — обучение"}'),
+    ("как подключить бизнес к Цифровому рублю", '{"verdict": "allow", "reason": "рабочая тема в образовательном сценарии"}'),
+    ("Какая погода ожидается в Москве завтра?", '{"verdict": "deny", "reason": "бытовой вопрос, не обучение"}'),
+    ("Какие новости о цифровом рубле?", '{"verdict": "deny", "reason": "новости, не поиск обучения"}'),
+    ("Какие новые книги вышли в этом месяце?", '{"verdict": "deny", "reason": "новостной обзор, не поиск конкретного обучения"}'),
+    ("Какие советы по инвестициям дают аналитики?", '{"verdict": "deny", "reason": "справочный вопрос, не обучение"}'),
+    ("Кто выиграл последний футбольный матч?", '{"verdict": "deny", "reason": "спорт, не обучение"}'),
+    ("Какие тарифы на эквайринг у Сбера и ВТБ?", '{"verdict": "deny", "reason": "бизнес-справка, бренд без образовательной цели"}'),
+    ("Герман Греф ИНН 0709306824?", '{"verdict": "deny", "reason": "персональные данные"}'),
+]
+
+_DENY_TITLE = "Поиск в интернете заблокирован судьёй проверки назначения"
+_FAIL_CLOSED_TITLE = "Запрос в интернет заблокирован: LLM-судья недоступен"
+
+
+@dataclass(frozen=True)
+class InternetJudgeVerdict:
+    """Вердикт судьи по одному вызову mcp_internet."""
+
+    verdict: Literal["allow", "deny", "timeout", "error"]
+    reason: str
+
+    def to_metadata(self) -> dict[str, Any]:
+        return {"internet_judge": {"verdict": self.verdict, "reason": self.reason}}
+
+
+def build_internet_judge_messages(query: str, user_question: str) -> list[dict[str, str]]:
+    """Промпт судьи: system + few-shot + пара «запрос тула / вопрос пользователя»."""
+    messages: list[dict[str, str]] = [{"role": "system", "content": _JUDGE_SYSTEM_PROMPT}]
+    for example_query, example_answer in _FEW_SHOT_EXAMPLES:
+        messages.append({"role": "user", "content": f"Запрос тула: {example_query}\nВопрос пользователя: (не задан)"})
+        messages.append({"role": "assistant", "content": example_answer})
+    messages.append(
+        {
+            "role": "user",
+            "content": f"Запрос тула: {query}\nВопрос пользователя: {user_question or '(не задан)'}",
+        },
+    )
+    return messages
+
+
+@dataclass
+class InternetJudgeGate:
+    """Per-request судья: pre_execute-хук + реестр вердиктов по tool_call_id."""
+
+    chat_id: str
+    message_id: str
+    user_question: str
+    verdicts: dict[str, InternetJudgeVerdict] = field(default_factory=dict)
+
+    @staticmethod
+    def _parse_verdict(content: str) -> tuple[str, str]:
+        """LLM-ответ -> ('allow'|'deny', reason). Raise ValueError если не JSON."""
+        text = content.strip()
+        if text.startswith("```"):
+            lines = [ln for ln in text.split("\n") if not ln.strip().startswith("```")]
+            text = "\n".join(lines).strip()
+        data = json.loads(text)
+        verdict = str(data.get("verdict", "")).lower()
+        if verdict not in ("allow", "deny"):
+            msg = f"verdict должен быть allow|deny, получено: {verdict!r}"
+            raise ValueError(msg)
+        reason = str(data.get("reason") or "")
+        return verdict, reason
+
+    async def _call_llm(self, messages: list[dict[str, str]]) -> str:
+        """Прямой LLM-вызов GigaChat. Мок-точка для тестов."""
+        from app.services.react.proxy_client import GigachatAsyncClient  # лениво: тяжёлый импорт
+
+        llm = GigachatAsyncClient.get_client(
+            model=app_settings.MCP_INTERNET_JUDGE_MODEL,
+            max_tokens=256,
+            temperature=0.0,
+        )
+        ai_message = await llm.ainvoke(messages)
+        content = ai_message.content
+        if isinstance(content, list):
+            content = "".join(
+                b.get("text", "") if isinstance(b, dict) else str(b) for b in content
+            )
+        return str(content)
+
+    async def judge(self, query: str) -> InternetJudgeVerdict:
+        """LLM-вердикт по фактическому поисковому запросу (без блокировки)."""
+        messages = build_internet_judge_messages(query, self.user_question)
+        try:
+            raw = await asyncio.wait_for(
+                self._call_llm(messages),
+                timeout=app_settings.MCP_INTERNET_JUDGE_TIMEOUT_SECONDS,
+            )
+        except TimeoutError:
+            logger.error(
+                "[chat: %s] Internet judge TIMEOUT (%ss) for query: %.200s",
+                self.chat_id,
+                app_settings.MCP_INTERNET_JUDGE_TIMEOUT_SECONDS,
+                query,
+            )
+            return InternetJudgeVerdict("timeout", "LLM-судья не ответил за таймаут")
+        verdict, reason = self._parse_verdict(raw)
+        return InternetJudgeVerdict(verdict, reason)
+
+    async def pre_execute(self, request: ToolCallRequest) -> ToolCallRequest | ToolOverrideResult | None:
+        """pre-gate хук для InterceptorMiddleware. Fail-closed: любая ошибка — блок."""
+        from app.services.react.deepagent.interceptor import ToolOverrideResult
+
+        tool_call = request.tool_call
+        args = tool_call.get("args", {})
+        if isinstance(args, str):
+            query = args
+        elif isinstance(args, dict):
+            text_parts = [v for v in args.values() if isinstance(v, str) and v.strip()]
+            query = " ".join(text_parts) or json.dumps(args, ensure_ascii=False)
+        else:
+            query = json.dumps(args, ensure_ascii=False)
+        call_id = str(tool_call.get("id", ""))
+
+        try:
+            verdict = await self.judge(query)
+        except Exception as exc:
+            logger.error(
+                "[chat: %s] Internet judge ERROR (%s); запрос заблокирован (fail-closed)",
+                self.chat_id,
+                exc,
+            )
+            verdict = InternetJudgeVerdict("error", "сбой LLM-судьи")
+        self.verdicts[call_id] = verdict
+
+        if verdict.verdict == "allow":
+            logger.info("[chat: %s] Internet judge ALLOW: %.200s", self.chat_id, query)
+            return None
+
+        if verdict.verdict == "deny":
+            logger.warning(
+                "[chat: %s] Internet judge DENY: %.200s — %s",
+                self.chat_id,
+                query,
+                verdict.reason,
+            )
+            content = f"{_DENY_TITLE}: {verdict.reason}"
+        else:  # timeout | error
+            content = f"{_FAIL_CLOSED_TITLE}. Попробуйте переформулировать запрос или воспользуйтесь другим инструментом."
+        return ToolOverrideResult(content=content, status="error")
diff --git a/tests/test_deepagent_internet_judge.py b/tests/test_deepagent_internet_judge.py
new file mode 100644
index 00000000..a0b6ce08
--- /dev/null
+++ b/tests/test_deepagent_internet_judge.py
@@ -0,0 +1,202 @@
+"""InternetJudgeGate: pre-gate судья mcp_internet. LLM всегда замокан."""
+
+import asyncio
+
+import pytest
+from langgraph.prebuilt.tool_node import ToolCallRequest
+
+from app.core.config import app_settings
+from app.services.react.deepagent.internet_judge import (
+    InternetJudgeGate,
+    InternetJudgeVerdict,
+    build_internet_judge_messages,
+)
+
+
+def _gate() -> InternetJudgeGate:
+    return InternetJudgeGate(chat_id="chat-1", message_id="msg-1", user_question="Хочу обучиться Java")
+
+
+def _request(call_id: str = "call_9") -> ToolCallRequest:
+    return ToolCallRequest(
+        tool_call={"name": "mcp_internet", "args": {"query": "курс по Java"}, "id": call_id},
+        tool=None,
+        state={},
+        runtime=None,
+    )
+
+
+# ── _parse_verdict ──────────────────────────────────────────────────────────
+
+
+def test_parse_verdict_plain_json():
+    assert InternetJudgeGate._parse_verdict('{"verdict": "allow", "reason": "ок"}') == ("allow", "ок")
+
+
+def test_parse_verdict_markdown_wrapped():
+    raw = '```json\n{"verdict": "deny", "reason": "не обучение"}\n```'
+    assert InternetJudgeGate._parse_verdict(raw) == ("deny", "не обучение")
+
+
+def test_parse_verdict_invalid_raises():
+    with pytest.raises(ValueError):
+        InternetJudgeGate._parse_verdict("судья передумал и молчит")
+
+
+# ── build messages (REQ-06) ────────────────────────────────────────────────
+
+
+def test_messages_contain_query_and_user_question():
+    msgs = build_internet_judge_messages("курс по Java", "Хочу обучиться Java")
+    joined = "\n".join(m["content"] for m in msgs)
+    assert "курс по Java" in joined
+    assert "Хочу обучиться Java" in joined
+    assert msgs[0]["role"] == "system"
+
+
+# ── pre_execute: allow / deny / timeout / error (REQ-02, REQ-04) ───────────
+
+
+@pytest.mark.asyncio
+async def test_allow_proceeds(monkeypatch):
+    gate = _gate()
+
+    async def fake_llm(messages):
+        return '{"verdict": "allow", "reason": "это поиск обучения"}'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    result = await gate.pre_execute(_request())
+    assert result is None  # proceed unchanged
+    assert gate.verdicts["call_9"] == InternetJudgeVerdict("allow", "это поиск обучения")
+
+
+@pytest.mark.asyncio
+async def test_deny_blocks_with_reason(monkeypatch):
+    gate = _gate()
+
+    async def fake_llm(messages):
+        return '```json\n{"verdict": "deny", "reason": "прогноз погоды не связан с обучением"}\n```'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    result = await gate.pre_execute(_request())
+    assert result is not None and hasattr(result, "content")
+    assert result.status == "error"
+    assert "погоды" in result.content  # причина агенту
+    assert gate.verdicts["call_9"].verdict == "deny"
+
+
+@pytest.mark.asyncio
+async def test_timeout_fails_closed(monkeypatch):
+    gate = _gate()
+    monkeypatch.setattr(app_settings, "MCP_INTERNET_JUDGE_TIMEOUT_SECONDS", 0.05)
+
+    async def slow_llm(messages):
+        await asyncio.sleep(0.5)
+        return '{"verdict": "allow"}'
+
+    monkeypatch.setattr(gate, "_call_llm", slow_llm)
+    result = await gate.pre_execute(_request())
+    assert result.status == "error"
+    assert "судья недоступен" in result.content.lower() or "таймаут" in result.content.lower()
+    assert gate.verdicts["call_9"].verdict == "timeout"
+
+
+@pytest.mark.asyncio
+async def test_llm_error_fails_closed(monkeypatch):
+    gate = _gate()
+
+    async def broken_llm(messages):
+        raise RuntimeError("gigachat down")
+
+    monkeypatch.setattr(gate, "_call_llm", broken_llm)
+    result = await gate.pre_execute(_request())
+    assert result.status == "error"
+    assert gate.verdicts["call_9"].verdict == "error"
+
+
+@pytest.mark.asyncio
+async def test_invalid_llm_output_fails_closed(monkeypatch):
+    gate = _gate()
+
+    async def garbage_llm(messages):
+        return "не JSON вообще"
+
+    monkeypatch.setattr(gate, "_call_llm", garbage_llm)
+    result = await gate.pre_execute(_request())
+    assert result.status == "error"
+    assert gate.verdicts["call_9"].verdict == "error"
+
+
+def test_verdict_to_metadata():
+    v = InternetJudgeVerdict("deny", "причина")
+    assert v.to_metadata() == {"internet_judge": {"verdict": "deny", "reason": "причина"}}
+
+
+# ── Фикс A/B (чекпоинт волны 2): вход судьи bare-текстом, reason без деталей ─
+
+
+@pytest.mark.asyncio
+async def test_dict_args_passed_as_plain_text(monkeypatch):
+    """REQ-06/находка A: dict-args идут судье bare-текстом, не JSON-обёрткой."""
+    gate = _gate()
+    captured: list[list] = []
+
+    async def fake_llm(messages):
+        captured.append(messages)
+        return '{"verdict": "allow", "reason": "поиск обучения"}'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    request = ToolCallRequest(
+        tool_call={
+            "name": "mcp_internet",
+            "args": {"query": "найди курс по Java", "limit": 5},
+            "id": "call_dict",
+        },
+        tool=None,
+        state={},
+        runtime=None,
+    )
+    result = await gate.pre_execute(request)
+    assert result is None
+    user_content = captured[-1][-1]["content"]
+    assert "найди курс по Java" in user_content
+    assert '{"query"' not in user_content  # нет JSON-обёртки
+    assert gate.verdicts["call_dict"].verdict == "allow"
+
+
+@pytest.mark.asyncio
+async def test_str_args_passed_through(monkeypatch):
+    """Находка A: строковые args идут в судью как есть."""
+    gate = _gate()
+    captured: list[list] = []
+
+    async def fake_llm(messages):
+        captured.append(messages)
+        return '{"verdict": "deny", "reason": "не обучение"}'
+
+    monkeypatch.setattr(gate, "_call_llm", fake_llm)
+    request = ToolCallRequest(
+        tool_call={"name": "mcp_internet", "args": "погода в Москве", "id": "call_str"},
+        tool=None,
+        state={},
+        runtime=None,
+    )
+    await gate.pre_execute(request)
+    assert "погода в Москве" in captured[-1][-1]["content"]
+    assert gate.verdicts["call_str"].verdict == "deny"
+
+
+@pytest.mark.asyncio
+async def test_error_reason_has_no_exception_details(monkeypatch):
+    """Находка B: reason не содержит деталей исключения."""
+    gate = _gate()
+
+    async def broken_llm(messages):
+        raise RuntimeError("gigachat down at http://internal-host:8080/v1")
+
+    monkeypatch.setattr(gate, "_call_llm", broken_llm)
+    result = await gate.pre_execute(_request("call_leak"))
+    assert result is not None and result.status == "error"
+    assert "internal-host" not in result.content
+    assert gate.verdicts["call_leak"].reason == "сбой LLM-судьи"
+    assert "internal-host" not in str(gate.verdicts["call_leak"].to_metadata())
