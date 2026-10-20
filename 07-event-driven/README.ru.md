# Шаг 07: Событийно-ориентированная архитектура

> Выведите агента за пределы CLI.

## Предварительные условия

Так же, как в шаге 06 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свои ключи API
```

## Что мы создадим

Крупный архитектурный сдвиг. Паттерн шины событий отделяет источники сообщений от выполнения агента и закладывает основу для последующих шагов.

<img src="07-event-driven.svg" align="center" width="100%" />

## Ключевые компоненты

- **EventBus** — центральный pub/sub для распределения событий
- **Events** — InboundEvent, OutboundEvent
- **Workers** — фоновые задачи, обрабатывающие события
- **AgentWorker** — обрабатывает InboundEvent → выполняет сессию агента → публикует OutboundEvent



[src/mybot/core/events.py](src/mybot/core/events.py)

```python
@dataclass
class InboundEvent(Event):
    session_id: str
    content: str
    retry_count: int = 0

@dataclass
class OutboundEvent(Event):
    session_id: str
    content: str
    error: str | None = None
```

[src/mybot/core/eventbus.py](src/mybot/core/eventbus.py)

```python
class EventBus(Worker):
    def subscribe(
        self, event_class: type[E], handler: Callable[[E], Awaitable[None]]
    ) -> None:
        """Subscribe a handler to an event class."""
        self._queue: asyncio.Queue[Event] = asyncio.Queue()

    def unsubscribe(self, handler: Handler) -> None:
        """Remove a handler from all subscriptions."""
        
    async def publish(self, event: Event) -> None:
        """Publish an event to the internal queue (non-blocking)."""
        await self._queue.put(event)

    async def run(self) -> None:
        """Process events from queue, starting with recovery."""
        logger.info("EventBus started")

        try:
            while True:
                event = await self._queue.get()
                try:
                    await self._dispatch(event)
                except Exception as e:
                    logger.error(f"Error dispatching event: {e}")
                finally:
                    self._queue.task_done()
        except asyncio.CancelledError:
            logger.info("EventBus stopping...")
            raise
```

[src/mybot/server/agent_worker.py](src/mybot/server/agent_worker.py)

```python
class AgentWorker(SubscriberWorker):
    def __init__(self, context):
        self.context.eventbus.subscribe(InboundEvent, self.dispatch_event)

    async def dispatch_event(self, event: InboundEvent):
        agent = Agent(agent_def, self.context)
        session = agent.resume_session(event.session_id)
        response = await session.chat(event.content)

        result = OutboundEvent(
            session_id=event.session_id,
            content=response,
        )
        await self.context.eventbus.publish(result)
```

## Попробуйте

На самом деле ничего не должно отличаться от предыдущего шага...

Нетерпеливый читатель может сразу перейти к [шагу 09: Каналы](../09-channels/).

## Что дальше

[Шаг 08: Горячая перезагрузка конфигурации](../08-config-hot-reload/) — Слияние конфигураций и горячая перезагрузка