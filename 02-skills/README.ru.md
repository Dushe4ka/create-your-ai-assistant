# Шаг 02: Навыки

> Расширьте агента с помощью `SKILL.md`.

Навыки — это лениво загружаемые возможности. Подробнее см. [официальную документацию](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview).

## Предварительные условия

Так же, как в шаге 00 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим?

Навыки — это возможности с отложенной загрузкой во время выполнения. Это не изобретение OpenClaw, а открытый стандарт. Подробнее см. [официальную документацию](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview).

<img src="02-skills.svg" align="center" width="100%" />

## Ключевые компоненты

- **SkillDef**: определения навыков (id, имя, описание, содержимое)
- **SKILL.md**: формат с YAML-frontmatter и телом в markdown
- **skill tool**: динамический инструмент, который перечисляет доступные навыки и загружает содержимое по требованию



[src/mybot/tools/skill_tool.py](src/mybot/tools/skill_tool.py)

```python
def create_skill_tool(skill_loader: "SkillLoader"):
    """Factory function to create skill tool with dynamic schema."""
    skill_metadata = skill_loader.discover_skills()

    # Build XML description of available skills
    skills_xml = "<skills>\n"
    for meta in skill_metadata:
        skills_xml += f'  <skill name="{meta.name}">{meta.description}</skill>\n'
    skills_xml += "</skills>"

    @tool(name="skill", description=f"Load skill. {skills_xml}", ...)
    async def skill_tool(skill_name: str, session: "AgentSession") -> str:
        skill_def = skill_loader.load_skill(skill_name)
        return skill_def.content

    return skill_tool
```

## Примечания

OpenClaw реализует систему навыков не отдельным инструментом, а с помощью **инъекции в системный промпт с чтением файлов**.

### Два подхода к навыкам

**Подход с инструментом (в этом руководстве):**
- Выделенный инструмент `skill` перечисляет доступные навыки и загружает их содержимое
- Схема инструмента включает метаданные навыков в его описание
- Агент вызывает инструмент `skill` для получения содержимого навыка
- Автономное обнаружение и загрузка навыков

**Подход с системным промптом (OpenClaw):**
- Метаданные навыков (id, имя, описание) внедряются в системный промпт
- Агент использует стандартный инструмент `read` для чтения SKILL.md
- Не нужен специализированный инструмент для навыков
- Более простой реестр инструментов

> Чтобы реализовать навыки как часть системного промпта, внедрите их как ещё один слой промпта, о чём рассказывается в [шаге 13: Многослойные промпты](../13-multi-layer-prompts/).

## Попробуйте

```bash
cd 02-skills
uv run my-bot chat

# You: What skills do you have available?
# pickle: Hi there! 🐱 I have access to two specialized skills:
#
# - **cron-ops**: Create, list, and delete scheduled cron jobs
# - **skill-creator**: Guide for creating effective skills
#
# Is there something specific you'd like to do with either of these, or do you have another task I can help you with?
#
# You: Create a skill to access Weather Information
# pickle: [Loads and create a weather-info skill]
```

## Что дальше

[Шаг 03: Сохранение](../03-persistence/) — Помнить диалоги между сессиями