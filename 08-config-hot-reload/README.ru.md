# Шаг 08: Горячая перезагрузка конфигурации

> Редактируйте без перезапуска.

## Предварительные условия

Так же, как в шаге 06 — скопируйте файл конфигурации и добавьте свой ключ API:

```bash
cp default_workspace/config.example.yaml default_workspace/config.user.yaml
# Отредактируйте config.user.yaml и добавьте свои ключи API
```

## Что мы создадим

<img src="08-config-hot-reload.svg" align="center" width="100%" />


## Ключевые компоненты

- **ConfigReloader** — следит за изменениями файлов конфигурации в рабочем каталоге с помощью watchdog
- **Config Merging** — конфигурация времени выполнения переопределяет пользовательскую через глубокое слияние


[src/mybot/utils/config.py](src/mybot/utils/config.py)

```python
class Config(BaseModel):
    """Configuration with hot reload support."""

    @classmethod
    def _load_merged_configs(cls, workspace_dir: Path) -> dict[str, Any]:
        config_data: dict[str, Any] = {}

        user_config = workspace_dir / "config.user.yaml"
        runtime_config = workspace_dir / "config.runtime.yaml"

        with open(user_config) as f:
            config_data = cls._deep_merge(config_data, yaml.safe_load(f) or {})

        with open(runtime_config) as f:
            config_data = cls._deep_merge(config_data, yaml.safe_load(f) or {})

        return config_data

    def reload(self) -> bool:
        config_data = self._load_merged_configs(self.workspace)
        new_config = Config.model_validate(config_data)

        for field_name in Config.model_fields:
            setattr(self, field_name, getattr(new_config, field_name))

        return True


class ConfigHandler(FileSystemEventHandler):
    """Handles config file modification events."""

    def __init__(self, config: Config):
        self._config = config

    def on_modified(self, event):
        """Reload config when config.user.yaml changes."""
        if not event.is_directory and event.src_path.endswith("config.user.yaml"):
            self._config.reload()
```

## Попробуйте

На самом деле ничего не должно отличаться от предыдущего шага...

Нетерпеливый читатель может сразу перейти к [шагу 09: Каналы](../09-channels/).

## Что дальше

[Шаг 09: Каналы](../09-channels/) — поддержка нескольких платформ: CLI, Telegram и другие интерфейсы.