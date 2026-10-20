# Шаг 13: Многослойные промпты

> Больше контекста, больше контекста, больше контекста.

## Предварительные условия

Так же, как в шаге 09 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим

Насыщенные системные промпты, собранные из нескольких слоёв: идентичность, личность, контекст рабочего пространства, информация времени выполнения.

<img src="13-multi-layer-prompts.svg" align="center" width="100%" />

## Ключевые компоненты

- **AgentDef** — расширение `soul_md`
- **PromptBuilder** — собирает все слои промпта в итоговый системный промпт

[src/mybot/core/agent_loader.py](src/mybot/core/agent_loader.py)

```python
class AgentDef(BaseModel):
    id: str
    name: str
    description: str = ""
    agent_md: str
    soul_md: str = ""  # NEW: Personality layer (optional)
    llm: LLMConfig
    allow_skills: bool = False
```

[src/mybot/core/prompt_builder.py](src/mybot/core/prompt_builder.py)
```python
class PromptBuilder:
    def build(self, state: "SessionState") -> str:
        layers = []

        # Layer 1: Identity
        layers.append(state.agent.agent_def.agent_md)

        # Layer 2: Soul (optional)
        if state.agent.agent_def.soul_md:
            layers.append(f"## Personality\n\n{state.agent.agent_def.soul_md}")

        # Layer 3: Bootstrap context (BOOTSTRAP.md + AGENTS.md + crons)
        bootstrap = self._load_bootstrap_context()
        if bootstrap:
            layers.append(bootstrap)

        # Layer 4: Runtime context
        layers.append(self._build_runtime_context(agent_id, timestamp))

        # Layer 5: Channel hint
        layers.append(self._build_channel_hint(source))

        return "\n\n".join(layers)
```

### Пример настройки рабочего пространства

- [default_workspace/agents/pickle/AGENT.md](../default_workspace/agents/pickle/AGENT.md) — идентичность агента, возможности и поведенческие правила (с YAML-frontmatter для конфигурации)
- [default_workspace/agents/pickle/SOUL.md](../default_workspace/agents/pickle/SOUL.md) — слой личности, определяющий характер и тон агента
- [default_workspace/BOOTSTRAP.md](../default_workspace/BOOTSTRAP.md) — руководство по рабочему пространству с описанием структуры каталогов, назначения файлов и шаблонов путей для агентов, навыков, cron-задач и памяти
- [default_workspace/AGENTS.md](../default_workspace/AGENTS.md) — перечисляет всех доступных агентов и даёт шаблоны диспетчеризации задач специализированным агентам (например, операции с памятью)


## Попробуйте

```bash
cd 13-multi-layer-prompts
uv run my-bot server

# Из выбранного вами канала:

# You: When are Where are we talking?
# pickle: Meow! Let me check... We're chatting right now via Telegram! *twitches ears*

# The current time is 2026-03-13 at 23:04:45. So we're here, in this conversation, happening in real-time. 🐱
```

## Примечание

Эта архитектура расширяема — при необходимости можно добавлять новые слои промпта. Например, **слой памяти** может внедрять релевантные воспоминания из прошлых бесед, предоставляя контекст о предпочтениях пользователя, текущих проектах или прежних взаимодействиях.

## Что дальше

[Шаг 14: Обратная отправка сообщений](../14-post-message-back/) — Общение, инициируемое агентом.