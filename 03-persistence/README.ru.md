# Шаг 03: Сохранение

> Сохраняйте ваши диалоги.

Сохраняйте и восстанавливайте историю диалогов, чтобы агент помнил прошлые взаимодействия.

## Предварительные условия

Так же, как в шаге 00 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
```

## Что мы создадим?

<img src="03-persistence.svg" align="center" width="100%" />

Структура файловой системы:

```
.history/
├── index.jsonl              # Метаданные сессий
└── sessions/
    └── {session_id}.jsonl   # Сообщения (один файл на сессию)
```

## Ключевые компоненты

- **.history/index.jsonl**: файловый индекс сессий в формате JSONL, включая метаданные
- **.history/sessions/{id}.jsonl**: файловое хранилище сообщений в формате JSONL



[src/mybot/core/history.py](src/mybot/core/history.py) - Новый файл

```python
class HistoryStore:
    def create_session(self, agent_id: str, session_id: str) -> dict:
        """Create a new conversation session."""

    def save_message(self, session_id: str, message: HistoryMessage) -> None:
        """Save a message to history."""

    def get_messages(self, session_id: str) -> list[HistoryMessage]:
        """Get all messages for a session."""
```

## Примечание

Файловое хранилище намеренно простое — именно так многие агенты работают с сохранением данных и сегодня. По мере роста проекта такое хранилище естественным образом перерастёт в решение на основе базы данных. Sqlite, Postgres — что угодно. В проекте уже есть абстракция `HistoryStore`, поэтому создание более сложного класса сохранения не должно затронуть остальные части проекта.

## Попробуйте

```bash
cd 03-persistence
uv run my-bot chat

# Каждый запуск создаёт новую сессию
# Сообщения сохраняются в каталог .history/
```

## Что дальше

[Шаг 04: Слэш-команды](../04-slash-commands/) — Прямой вызов команд