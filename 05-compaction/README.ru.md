# Шаг 05: Сжатие

> Упакуйте историю и продолжайте...

## Предварительные условия

Так же, как в шаге 00 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим?

Длинные диалоги рано или поздно упираются в лимиты контекста. Сжатие решает эту проблему: старые сообщения суммируются, и диалог переносится в новую сессию.

<img src="05-compaction.svg" align="center" width="100%" />

- Контекст превысил порог?
- Обрезать слишком большой результат инструмента.
- Всё ещё слишком большой?
- Суммировать старые сообщения.
- Перенести в новую сессию.


## Ключевые компоненты

- **Token Estimation**: использует litellm token_counter для точной оценки
- **Truncation Strategy**: сначала обрезает большие результаты инструментов, затем суммирует старые сообщения
- **Context Compaction**: суммирует старые сообщения и использует сводку как первые промпты новой сессии
- **Commands**: `/compact` (вручную), `/context` (показать использование)



[src/mybot/core/context_guard.py](src/mybot/core/context_guard.py) - Новый файл

```python
@dataclass
class ContextGuard:
    token_threshold: int = 160000  # 80% of 200k context

    def estimate_tokens(self, state: SessionState) -> int:
        return token_counter(model=state.agent.agent_def.llm.model, messages=state.build_messages())

    async def check_and_compact(self, state: SessionState) -> SessionState:
        token_count = 

        if self.estimate_tokens(state) < self.token_threshold:
            return state

        state.messages = self._truncate_large_tool_results(state.messages)

        if self.estimate_tokens(state) < self.token_threshold:
            return state

        return await self._compact_messages(state)
```

[src/mybot/core/agent.py](src/mybot/core/agent.py) - Интеграция

```python
async def chat(self, message: str) -> str:
    # ... add user message ...

    while True:
        messages = self.state.build_messages()
        # Check and compact before LLM call
        self.state = await self.context_guard.check_and_compact(self.state)
        content, tool_calls = await self.agent.llm.chat(messages, tool_schemas)
```

## Попробуйте

```bash
cd 05-compaction
uv run my-bot chat

# Проверяйте использование контекста в любой момент:
# You: /context
# **Messages:** 12
# **Tokens:** 15,420 (9.6% of 160,000 threshold)

# You: /compact
# ✓ Context compacted. 8 messages retained.
```

## Что дальше

[Шаг 06: Веб-инструменты](../06-web-tools/) — Добавьте веб-поиск и чтение URL