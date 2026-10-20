# Шаг 04: Слэш-команды

> Прямое управление сессиями пользователем.

## Предварительные условия

Так же, как в шаге 00 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим?

Детерминированные функции, вызываемые в чате: `/help`, `/skills` и `/session`.

### Архитектура

<img src="04-slash-commands.svg" align="center" width="100%" />

## Ключевые компоненты

- **Command**: базовый класс для слэш-команд (асинхронный метод execute)
- **CommandRegistry**: регистрирует и диспетчеризует команды
- **Commands**: `/help`, `/skills`, `/session`



[src/mybot/core/commands/base.py](src/mybot/core/commands/base.py) - Новый файл

```python
class Command(ABC):
    """Base class for slash commands."""

    name: str
    aliases: list[str] = []
    description: str = ""

    @abstractmethod
    async def execute(self, args: str, session: "AgentSession") -> str:
        """Execute the command and return response string."""
        pass
```

[src/mybot/core/commands/registry.py](src/mybot/core/commands/registry.py) - Новый файл

```python
class CommandRegistry:
    def register(self, cmd: Command) -> None:
        """Register a command and its aliases."""

    async def dispatch(self, input: str, session: "AgentSession") -> str | None:
        """Parse and execute a slash command. Returns None if not a command."""
```

[src/mybot/cli/chat.py](src/mybot/cli/chat.py) - Добавлена диспетчеризация команд

```python
 async def run(self) -> None:
        # ... Say Hello
        while True:
            # ... Get user input

            # Check for slash commands
            cmd_response = await self.session.command_registry.dispatch(
                user_input, self.session
            )
            if cmd_response is not None:
                self.console.print(cmd_response)
                continue

            # Normal chat
            response = await self.session.chat(user_input)
            self.display_agent_response(response)

```

## Примечания

Слэш-команды могут добавляться или не добавляться в историю сессии (журнал сообщений, отправляемый LLM). Это проектное решение: команды — это пользовательское управление, а не содержимое беседы. Оба подхода допустимы. Выбор зависит от вашего сценария.

## Попробуйте

```bash
cd 04-slash-commands
uv run my-bot chat

# Попробуйте команды:
# You: /help
# **Available Commands:**
# /help, /? - Show available commands
# /skills - List all skills or show skill details
# /session - Show current session details

# You: /session
# **Session ID:** `abc123...`
# **Agent:** Pickle (pickle)
# **Created:** 2026-03-08T12:00:00
# **Messages:** 0
```

## Что дальше

[Шаг 05: Сжатие](../05-compaction/) — Продолжайте общение...