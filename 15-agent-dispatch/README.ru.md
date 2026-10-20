# Шаг 15: Диспетчеризация агентов

> Ваш агент хочет работать с друзьями!

## Предварительные условия

Те же, что и в шаге 09 — скопируйте файл конфигурации и добавьте ваш API-ключ:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml, добавив ваш API-ключ
```

## Что мы построим

Агенты могут делегировать работу другим агентам.

<img src="15-agent-dispatch.svg" align="center" width="100%" />

## Ключевые компоненты

- **subagent_tool** — фабрика, создающая инструмент диспетчеризации с динамической схемой

[src/mybot/tools/subagent_tool.py](src/mybot/tools/subagent_tool.py)

```python
def create_subagent_dispatch_tool(
    current_agent_id: str,
    context: "SharedContext",
) -> BaseTool | None:
    available_agents = context.agent_loader.discover_agents()
    dispatchable_agents = [a for a in available_agents if a.id != current_agent_id]

    agents_desc = "<available_agents>\n"
    for agent_def in dispatchable_agents:
        agents_desc += f'  <agent id="{agent_def.id}">{agent_def.description}</agent>\n'
    agents_desc += "</available_agents>"

    @tool(
        name="subagent_dispatch",
        description=f"Dispatch a task to a specialized subagent.\n{agents_desc}",
        parameters={...},
    )
    async def subagent_dispatch(
        agent_id: str, task: str, session: "AgentSession", context: str = ""
    ) -> str:
        agent_def = shared_context.agent_loader.load(agent_id)
        agent = Agent(agent_def, shared_context)
        agent_source = AgentEventSource(agent_id=current_agent_id)
        agent_session = agent.new_session(agent_source)
        session_id = agent_session.session_id

        user_message = task
        if context:
            user_message = f"{task}\n\nContext:\n{context}"

        loop = asyncio.get_running_loop()
        result_future: asyncio.Future[str] = loop.create_future()

        # Create temp handler that filters by session_id
        async def handle_result(event: DispatchResultEvent) -> None:
            if event.session_id == session_id:
                if not result_future.done():
                    if event.error:
                        result_future.set_exception(Exception(event.error))
                    else:
                        result_future.set_result(event.content)

        # Subscribe to DispatchResultEvent events
        shared_context.eventbus.subscribe(DispatchResultEvent, handle_result)

        try:
            event = DispatchEvent(
                session_id=session_id,
                source=AgentEventSource(agent_id=current_agent_id),
                content=user_message,
                timestamp=time.time(),
                parent_session_id=session.session_id,
            )
            await shared_context.eventbus.publish(event)

            response = await result_future
        finally:
            shared_context.eventbus.unsubscribe(handle_result)

        result = {"result": response, "session_id": session_id}
        return json.dumps(result)

    return subagent_dispatch
```

### Как работает механизм диспетчеризации

Механизм диспетчеризации основан на паттерне **eventbus** для общения между агентами. Вот как это устроено:

1. **Публикация**: когда основной агент вызывает `subagent_dispatch`, он публикует `DispatchEvent` в шину событий, содержащий задачу и информацию о сессии.
2. **Подписка**: временный обработчик подписывается на события `DispatchResultEvent`, фильтруя их по идентификатору сессии, чтобы получать только ответ именно на эту диспетчеризацию.
3. **Ожидание**: основной агент ожидает future, который разрешается, когда субагент завершает работу и публикует `DispatchResultEvent`.
4. **Очистка**: после получения результата обработчик отписывается от шины событий.


## Попробуйте

```bash
cd 15-agent-dispatch
uv run my-bot chat

# You: Ask Cookie to read our README.
# pickle: Cookie has sent our README back! *purrs* 🐱

# # Step 15: Agent Dispatch

# > Your Agent want friends to work with!
# ...
```

## Примечания

### Альтернативные паттерны мультиагентности

Прямая диспетчеризация субагентов — лишь один из подходов к оркестрации нескольких агентов. Вот несколько других распространённых паттернов:

- **Общие списки задач**: агенты координируются, читая и записывая в общую очередь задач или базу данных. Каждый агент берёт задачи по мере их появления; агенты никогда не общаются друг с другом напрямую.
- **Tmux/Screen-сессии**: `tmux` позволяет запускать несколько процессов. Агенту можно предоставить навык Tmux, который поможет ему выполнять несколько задач, в некоторой степени достигая мультиагентности.

## Что дальше

[Шаг 16: Управление конкурентностью](../16-concurrency-control/) — Ограничение частоты запросов и управление очередями.