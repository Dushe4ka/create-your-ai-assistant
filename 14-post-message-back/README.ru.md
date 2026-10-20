"# Шаг 14: Отправка сообщений обратно

> Ваш агент хочет с вами поговорить.

## Предварительные условия

Те же, что и в шаге 09 — скопируйте файл конфигурации и добавьте ваш API-ключ:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml, добавив ваш API-ключ
```

## Что мы построим

Теперь агент может сам инициировать отправку сообщений вам, а не только отвечать, особенно когда работает в контексте cron-задачи.

<img src="14-post-message-back.svg" align="center" width="100%" />

## Ключевые компоненты

- **post_message_tool** — фабрика, создающая инструмент при включённых каналах
- **DeliveryWorker** — обрабатывает доставку OutboundEvent на платформы

[src/mybot/tools/post_message_tool.py](src/mybot/tools/post_message_tool.py)

```python
@tool(...)
async def post_message(content: str, session: "AgentSession") -> str:
    event = OutboundEvent(
        session_id=session.session_id,
        source=AgentEventSource(agent_id=session.agent.agent_def.id),
        content=content,
        timestamp=time.time(),
    )
    await context.eventbus.publish(event)
    return "Message queued for delivery"

return post_message
```

## Попробуйте

```bash
cd 14-post-message-back
uv run my-bot server

# Из канала на ваш выбор:

# Вы: Say Hi to me after 5 minutes.
# pickle: I've scheduled a one-time "Hi" for you in about 2 minutes. You'll hear from me shortly! *purrs* ✅

# примерно через 5 минут

# pickle: Hi there! 👋 Just wanted to pop in and say hello! Hope you're having a wonderful day!
```

## Примечания

Инструмент `post_message` доступен только в Cron-задаче.

## Что дальше

[Шаг 15: Диспетчеризация агентов](../15-agent-dispatch/) — Взаимодействие нескольких агентов."