# Шаг 16: Управление конкурентностью

> Слишком много Пиклей работают одновременно?

## Предварительные условия

Те же, что и в шаге 09 — скопируйте файл конфигурации и добавьте ваш API-ключ:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml, добавив ваш API-ключ
```

## Что мы построим

Предотвращение исчерпания ресурсов за счёт ограничения количества одновременно работающих экземпляров агента.

<img src="16-concurrency-control.svg" align="center" width="100%" />

## Ключевые компоненты

- **AgentDef.max_concurrency** — настраиваемое ограничение для каждого агента
- **Управление конкурентностью на основе семафоров** — блокировка при достижении лимита конкурентности


[src/mybot/server/agent_worker.py](src/mybot/server/agent_worker.py)

```python
class AgentWorker(SubscriberWorker):
    def __init__(self, context: "SharedContext"):
        super().__init__(context)
        self._semaphores: dict[str, asyncio.Semaphore] = {}

    async def exec_session(self, event, agent_def: "AgentDef") -> None:
        sem = self._get_or_create_semaphore(agent_def)

        async with sem:  # Blocks if limit reached
            # ... execute session ...

        self._maybe_cleanup_semaphores(agent_def)

    def _get_or_create_semaphore(self, agent_def: "AgentDef") -> asyncio.Semaphore:
        if agent_def.id not in self._semaphores:
            self._semaphores[agent_def.id] = asyncio.Semaphore(
                agent_def.max_concurrency
            )
        return self._semaphores[agent_def.id]
```

## Попробуйте

У `Cookie` параметр `max_concurrency` равен 1, поэтому диспетчеризация из двух разных источников (не CLI) должна вызвать срабатывание этого ограничения.

## Примечание

Управление конкурентностью можно реализовать с разной гранулярностью в зависимости от вашего сценария:

- **По агенту** (показано здесь) — ограничивает конкурентные выполнения для каждого типа агента. Полезно, когда определённые агенты ресурсоёмки или имеют ограничения по частоте запросов.
- **По источнику** — ограничивает конкурентные запросы от одного пользователя или клиента. Полезно для предотвращения злоупотреблений и справедливого распределения ресурсов.
- **По приоритету** — разные лимиты конкурентности для разных уровней приоритета. Для высокоприоритетных задач может быть зарезервирована ёмкость.

## Что дальше

[Шаг 17: Память](../17-memory/) — Система долговременных знаний.