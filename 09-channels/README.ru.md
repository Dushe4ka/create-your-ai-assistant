# Шаг 09: Каналы

> Общайтесь с агентом с телефона.

## Предварительные условия

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свои ключи API
# Настройте токен Telegram-бота
```

## Что мы создадим

Теперь вы можете общаться с агентом из Telegram, Discord или любой другой платформы.

<img src="09-channels.svg" align="center" width="100%" />

- Пользователь отправляет сообщение через платформу (Telegram, Discord)
- Канал получает сообщение и создаёт EventSource
- ChannelWorker публикует InboundEvent в EventBus
- AgentWorker обрабатывает событие и генерирует ответ
- AgentWorker публикует OutboundEvent в EventBus
- DeliveryWorker получает OutboundEvent
- DeliveryWorker находит источник сессии и отправляет через соответствующий канал

## Ключевые компоненты

- **EventSource** — абстрактная база для платформенных источников событий (CLI, Telegram, Discord)
- **Channel** — абстрактная база для платформ обмена сообщениями с интерфейсом run/reply/stop
- **ChannelWorker** — управляет несколькими каналами и публикует InboundEvents
- **DeliveryWorker** — подписывается на OutboundEvents и доставляет их через подходящий канал
- **Event Persistence** — сохранение исходящих событий и восстановление после сбоев, чтобы не терять сообщения


[src/mybot/channel/base.py](src/mybot/channel/base.py)

```python
class Channel(ABC, Generic[T]):
    @property
    @abstractmethod
    def platform_name(self) -> str:
        pass

    @abstractmethod
    async def run(self, on_message: Callable[[str, T], Awaitable[None]]) -> None:
        """Run the channel. Blocks until stop() is called."""
        pass

    @abstractmethod
    async def reply(self, content: str, source: T) -> None:
        """Reply to incoming message."""
        pass

    @abstractmethod
    async def stop(self) -> None:
        """Stop listening and cleanup resources."""
        pass
```

[src/mybot/server/channel_worker.py](src/mybot/server/channel_worker.py)

```python
class ChannelWorker(Worker):
    async def run(self) -> None:
        channel_tasks = [
            channel.run(self._create_callback(channel.platform_name))
            for channel in self.channels
        ]
        await asyncio.gather(*channel_tasks)

    def _create_callback(self, platform: str):
        async def callback(message: str, source: EventSource) -> None:
            session_id = self._get_or_create_session_id(source)

            event = InboundEvent(
                session_id=session_id,
                source=source,
                content=message,
            )
            await self.context.eventbus.publish(event)

        return callback

    def _get_or_create_session_id(self, source: EventSource) -> str:
        source_session = self.context.config.sources.get(str(source))
        if source_session:
            return source_session.session_id

        agent_def = self.context.agent_loader.load(self.context.config.default_agent)
        agent = Agent(agent_def, self.context)
        session = agent.new_session(source)

        # Cache the session
        self.context.config.set_runtime(
            f"sources.{source}", SourceSessionConfig(session_id=session.session_id)
        )

        return session.session_id
```

- Каждый EventSource (например, "platform-telegram:123:456") соответствует одной сессии
- Первое сообщение создаёт сессию, последующие сообщения переиспользуют её
- ID сессии кэшируется в config.runtime.yaml

[src/mybot/server/delivery_worker.py](src/mybot/server/delivery_worker.py)

```python
class DeliveryWorker(SubscriberWorker):
    """Delivers outbound messages to platforms."""

    async def handle_event(self, event: OutboundEvent) -> None:
        """Handle an outbound message event."""
        session_info = self._get_session_source(event.session_id)
        source = self._get_delivery_source(session_info)

        if source and source.platform_name:
            channel = self._get_channel(source.platform_name)
            if channel:
                await channel.reply(event.content, source)

        self.context.eventbus.ack(event)
```

[src/mybot/core/eventbus.py](src/mybot/core/eventbus.py.py)

``` python
class EventBus(Worker):
    async def run(self) -> None:
        await self._recover()
        while True:
            # ... Dispatching Events

    async def _dispatch(self, event: Event) -> None:
        await self._persist_outbound(event)
        await self._notify_subscribers(event)
    
    async def _recover(self) -> int:
        pending_files = list(self.pending_dir.glob("*.json"))

        for file_path in pending_files:
            with open(file_path, "r", encoding="utf-8") as f:
                data = json.load(f)
            event = deserialize_event(data)
            await self._notify_subscribers(event)

        return len(pending_files)

    def ack(self, event: Event) -> None:
        filename = f"{event.timestamp}_{event.session_id}.json"
        final_path = self.pending_dir / filename
        if final_path.exists():
            final_path.unlink()
```

- **Поток сохранения исходящих событий**:
  - `EventBus.publish()` ставит событие во внутреннюю очередь asyncio
  - `EventBus._dispatch()` вызывается для каждого события
  - `_persist_outbound()` атомарно записывает OutboundEvent на диск (временный файл + fsync + переименование)
  - `_notify_subscribers()` доставляет событие всем подписчикам (например, DeliveryWorker)

- **Поток восстановления после сбоя**:
  - При запуске EventBus `_recover()` сканирует каталог pending в поисках файлов `.json`
  - Каждое ожидающее событие десериализуется и повторно отправляется подписчикам
  - Только после успешной доставки DeliveryWorker вызывает `eventbus.ack(event)`
  - `ack()` удаляет сохранённый файл, подтверждая завершение доставки

## Попробуйте

```bash
cd 09-channels
uv run my-bot server
# Отправьте сообщение из выбранного вами канала.
```

## Что дальше

[Шаг 10: WebSocket](../10-websocket/) — веб-интерфейс реального времени для взаимодействия с агентами.