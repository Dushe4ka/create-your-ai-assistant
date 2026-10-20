# Шаг 01: Дайте агенту инструмент

> Простые инструменты мощнее, чем кажется. Read, Write и Bash вполне достаточно.

## Предварительные условия

Так же, как в шаге 00 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим?

Дадим агенту возможность действительно *делать* что-то — перейдём от простого общения к реальным действиям.

<img src="01-tools.svg" align="center" width="100%" />

## Ключевые компоненты

- **Stop Reason**: цикл общения ветвится по `stop_reason` — `"tool_calls"` для выполнения инструментов, `"stop"` для обычного завершения, `"length"` для обрезанных ответов
- **Tools**: управляет доступными инструментами и выполняет вызовы инструментов
- **Tool Calling Loop**: агент вызывает инструменты, добавляет результаты в историю и продолжает диалог



[src/mybot/tools/base.py](src/mybot/tools/base.py)

```python
class BaseTool(ABC):
    name: str
    description: str
    parameters: dict[str, Any]

    @abstractmethod
    async def execute(self, session: "AgentSession", **kwargs: Any) -> str:
        pass

    def get_tool_schema(self) -> dict[str, Any]:
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }
```

[src/mybot/core/agent.py](src/mybot/core/agent.py)

Интеграция инструментов в цикл общения.

```python
class AgentSession:
    async def chat(self, message: str) -> str:
        user_msg: Message = {"role": "user", "content": message}
        self.state.add_message(user_msg)

        tool_schemas = self.tools.get_tool_schemas()
        logger = logging.getLogger(__name__)

        while True:
            messages = self.state.build_messages()
            content, tool_calls, stop_reason = await self.agent.llm.chat(messages, tool_schemas)

            assistant_msg: Message = {
                "role": "assistant",
                "content": content,
                "tool_calls": [...],
            }
            self.state.add_message(assistant_msg)

            if stop_reason == "tool_calls":
                await self._handle_tool_calls(tool_calls)
                continue

            if stop_reason == "length":
                logger.warning(
                    "LLM response truncated (max_tokens reached), "
                    "returning partial response"
                )

            break

        return content
```


## Примечания

### Почему инструменты захардкожены

Инструменты регистрируются через `ToolRegistry.with_builtins()` в Python, а не объявляются во frontmatter `AGENT.md`.

Конфигурация инструментов на основе YAML предполагает проектные решения, выходящие за рамки этого руководства. Абстракция `BaseTool` уже делает инструменты подключаемыми на уровне кода.

OpenClaw решает эту задачу с помощью системы плагинов и JSON-конфигурации для каждого агента — но это вне рамок данного руководства.

## Попробуйте

```bash
cd 01-tools
uv run my-bot chat

# You: Hey Can you read your README.md please?
# pickle: I found and read the README.md file! 🐱

# # Step 01: Tools - Read, Write, Bash is Powerful Enough

# Give the agent the ability to execute tools (read, write, edit, bash) and interact with the filesystem.
# [More lines]
```

## Что дальше

[Шаг 02: Навыки](../02-skills/) — Динамическая загрузка возможностей с помощью файлов SKILL.md