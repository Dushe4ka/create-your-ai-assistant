# Шаг 11: Мультиагентная маршрутизация

> Направляйте правильную задачу правильному агенту.

## Предварительные условия

Так же, как в шаге 10 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свои ключи API
```

## Что мы создадим

Маршрутизация сообщений разным агентам в зависимости от источника.

<img src="11-multi-agent-routing.svg" align="center" width="100%" />

## Ключевые компоненты

- **AgentLoader** — обнаружение агентов для поддержки нескольких определений агентов
- **RoutingTable** — маршрутизирует источники к агентам с помощью regex-привязок с многоуровневой специфичностью
- **Binding** — шаблон источника + соответствие агенту с автоматическим вычислением уровня
- **New Commands** — `/route`, `/bindings`, `/agents` для управления мультиагентной настройкой

[src/mybot/core/agent_loader.py](src/mybot/core/agent_loader.py)

```python
class AgentLoader:
    def discover_agents(self) -> list[AgentDef]:
        """Scan agents directory and return list of valid AgentDef."""
        return discover_definitions(
            self.config.agents_path, "AGENT.md", self._parse_agent_def
        )
```

Определите собственного агента в `<workspace>/agents/<agent_id>/AGENT.md`

[src/mybot/core/routing.py](src/mybot/core/routing.py)

```python
@dataclass
class Binding:
    agent: str
    value: str
    tier: int 
    pattern: Pattern  # Compiled regex

    def _compute_tier(self) -> int:
        """Compute specificity tier."""
        if not any(c in self.value for c in r".*+?[]()|^$"):
            return 0  # Exact match
        if ".*" in self.value:
            return 2  # Wildcard
        return 1  # Specific regex

@dataclass
class RoutingTable:
    def _load_bindings(self) -> list[Binding]:
        bindings_data = self.context.config.routing.get("bindings", [])

        bindings_with_order = [
            (Binding(agent=b["agent"], value=b["value"]), i)
            for i, b in enumerate(bindings_data)
        ]
        bindings_with_order.sort(key=lambda x: (x[0].tier, x[1]))
        self.bindings = [b for b, _ in bindings_with_order]

        return self.bindings

    def resolve(self, source: str) -> str:
        for binding in self._load_bindings():
            if binding.pattern.match(source):
                return binding.agent
        return self.context.config.default_agent

    def get_or_create_session_id(self, source: EventSource) -> str:
        source_session = self.context.config.sources.get(str(source))
        if source_session:
            return source_session.session_id

        # Resolve agent and create new session
        agent_id = self.resolve(str(source))
        agent_def = self.context.agent_loader.load(agent_id)
        agent = Agent(agent_def, self.context)
        session = agent.new_session(source)

        self.context.config.set_runtime(
            f"sources.{str(source)}", SourceSessionConfig(session_id=session.session_id)
        )
        return session.session_id
```

- **Многоуровневые правила маршрутизации**: находят правила, соответствующие входящему источнику, начиная с самых специфичных.
- **Запасной вариант по умолчанию**: если ни одно правило не подошло, используется глобальный агент по умолчанию.


[src/mybot/server/channel_worker.py](src/mybot/server/channel_worker.py)

```python
async def callback(message: str, source: EventSource) -> None:
    # ... validation ...

    # Use routing_table to resolve agent from bindings
    session_id = self.context.routing_table.get_or_create_session_id(source)

    # Publish event
    event = InboundEvent(session_id=session_id, source=source, content=message)
    await self.context.eventbus.publish(event)
```

## Попробуйте

```bash
cd 11-multi-agent-routing
uv run my-bot agents chat

# You: /agent
# pickle: **Agents:**
# - `cookie`: Memory manager for storing, organizing, and retrieving memories
# - `pickle`: A friendly cat assistant talk to user directly, managing daily tasks. (current)

# You: /bindings
# pickle: No routing bindings configured.

# You: /route platform-ws:* cookie
# pickle: ✓ Route bound: `platform-ws:*` → `cookie`
```

## Что дальше

[Шаг 12: Cron + Heartbeat](../12-cron-heartbeat/) — Запланированные задачи и мониторинг состояния.