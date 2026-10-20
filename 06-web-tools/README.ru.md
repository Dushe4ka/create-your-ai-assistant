# Шаг 06: Веб-инструменты

> Ваш агент хочет увидеть большой мир.
> По сути, это всего два новых инструмента.

## Предварительные условия

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свой ключ API
# Раскомментируйте секции websearch и webread
# Добавьте свой ключ API для websearch
```

## Что мы создадим

LLM может уметь писать на Python, но не может знать последние тренды в экосистеме Python. Этот шаг открывает мир за пределами локальной файловой системы и обучающих данных LLM.

<img src="06-web-tools.svg" align="center" width="100%" />

## Ключевые компоненты

- **WebSearchProvider**: провайдеры веб-поиска.
- **WebReadProvider**: провайдеры чтения веб-страниц.
- **Tools**: инструменты `websearch` и `webread`.



[src/mybot/provider/web_search/](src/mybot/provider/web_search/)

```python
class WebSearchProvider(ABC):
    async def search(self, query: str) -> list[SearchResult]: ...
```

[src/mybot/provider/web_read/](src/mybot/provider/web_read/)

```python
class WebReadProvider(ABC):
    async def read(self, url: str) -> ReadResult: ...
```

[src/mybot/tools/websearch_tool.py](src/mybot/tools/websearch_tool.py)

```python
@tool(...)
async def websearch(query: str, session: "AgentSession") -> str:
    results = await provider.search(query)

    if not results:
        return "No results found."
    output = []
    for i, r in enumerate(results, 1):
        output.append(f"{i}. **{r.title}**\n   {r.url}\n   {r.snippet}")
    return "\n\n".join(output)
```

[src/mybot/tools/webread_tool.py](src/mybot/tools/webread_tool.py)

```python
@tool(...)
async def webread(url: str, session: "AgentSession") -> str:
    result = await provider.read(url)
    if result.error:
        return f"Error reading {url}: {result.error}"

    return f"**{result.title}**\n\n{result.content}"
```

## Попробуйте

```bash
cd 06-web-tools
uv run my-bot chat

# You: What is pickle bot? search online please.
# pickle: Based on my search, there are actually a few different things called "Pickle Bot":

# ### 1. **Pickle Robot Company** 🤖
# ### 2. **Pickle Bot (Discord Bot)** 💬
# ### 3. **pickle-bot (GitHub)** 🐱
# An open-source project described as:
# - "Your own AI assistant, speak like a cat"
# - "Pickle is a standard little cat"
# - A customizable AI assistant that you can name, talk to, and teach

# The GitHub version sounds like it could be related to me - a cat-speaking AI assistant! 😺

# Which one were you curious about?
```

## Что дальше

[Шаг 07: Событийная архитектура](../07-event-driven/) — Большой рефакторинг в событийную архитектуру