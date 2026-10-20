# Шаг 17: Память

> Запомни меня!

## Предварительные условия

Те же, что и в шаге 09 — скопируйте файл конфигурации и добавьте ваш API-ключ:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml, добавив ваш API-ключ
```

## Что мы построим

Долговременная память, общая для всех диалогов.

```
pickle: @cookie Do you know <topic> about user?
cookie: Yes, <content>.
```

## Ключевые компоненты

- **Memory agent** — специализированный агент для управления памятью
- [default_workspace/agents/cookie/AGENT.md](../default_workspace/agents/cookie/AGENT.md)

## Попробуйте

```bash
cd 17-memory
uv run my-bot chat

# Вы: Remember that my name is Zane
# Pickle: Got it! I've saved that preference.

uv run my-bot chat

# User: What's my name?
# Pickle: Based on your memory, you name is Zane! Hi Zane! 😸
```

## Примечание

В этой реализации используется подход **«специализированный агент»**. Альтернативы:

| Подход | Описание |
|----------|-------------|
| **Специализированный агент** (здесь) | Выделенный агент памяти, доступный через диспетчеризацию |
| **Прямые инструменты** | Инструменты памяти в основном агенте |
| **На основе навыков** | Использование CLI-инструментов, таких как grep |
| **Векторная база данных** | Семантический поиск по эмбеддингам |

### Структура каталога памяти (Pickle Bot)

```
memories/
├── topics/
│   ├── preferences.md    # Пользовательские предпочтения
│   └── identity.md       # Информация о пользователе
├── projects/
│   └── my-project.md     # Заметки по конкретному проекту
└── daily-notes/
    └── 2024-01-15.md     # Ежедневный журнал
```

## Что дальше

Разворачивайте, расширяйте и настраивайте!