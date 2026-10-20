# Шаг 12: Cron + Heartbeat

> Агент работает, пока вы спите.

## Предварительные условия

Так же, как в шаге 09 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим

Запланированные задачи — агент может выполнять задания по расписанию cron.

<img src="12-cron-heartbeat.svg" align="center" width="100%" />


## Ключевые компоненты

- **CRON.md и CronDef** — определение cron-задачи
- **CronWorker** — фоновый воркер, который каждую минуту проверяет наступившие задачи
- **DispatchEvent** — тип события для внутренней диспетчеризации сообщений
- **DispatchResultEvent** — событие-результат, возвращаемое выполненными задачами
- **Cron-Ops Skill** — навык для создания, просмотра и удаления запланированных cron-задач (реализован как навык, чтобы не раздувать реестр инструментов)

[src/mybot/core/cron_loader.py](src/mybot/core/cron_loader.py)

```python
class CronDef(BaseModel):
    id: str
    name: str
    description: str
    agent: str
    schedule: str
    prompt: str
    one_off: bool = False
```

[src/mybot/server/cron_worker.py](src/mybot/server/cron_worker.py)

```python
class CronWorker(Worker):
    async def run(self) -> None:
        while True:
            await self._tick()
            await asyncio.sleep(60)

    async def _tick(self) -> None:
        jobs = self.context.cron_loader.discover_crons()
        due_jobs = find_due_jobs(jobs)

        for cron_def in due_jobs:
            event = DispatchEvent(
                session_id=session.session_id,
                source=CronEventSource(cron_id=cron_def.id),
                content=cron_def.prompt,
            )
            await self.context.eventbus.publish(event)
```

[default_workspace/crons/hello-world/CRON.md](../default_workspace/skills/cron-ops/SKILL.md)

Функциональность работы с cron реализована через **систему навыков (SKILL)**, а не через отдельные инструменты, что позволяет не раздувать реестр инструментов.

## Попробуйте

```bash
cd 12-cron-heartbeat
uv run my-bot server

# Из выбранного вами канала:

# You: Send me some Cat Meme every morning.
# pickle: I've scheduled a "Cat Meme" cron job you every morning 9 AM. You'll find those meme shortly! *purrs* 🐱
```

## Примечания

В OpenClaw помимо системы CRON есть механизм HEARTBEAT.

- Может быть только один HEARTBEAT, он запускается в основной сессии с регулярным интервалом без проверки времени.
- Может быть несколько CRON-задач, они выполняются в фоне с учётом cron-выражений.

## Что дальше

[Шаг 13: Многослойные промпты](../13-multi-layer-prompts/) — Динамический системный промпт.