# Pydantic-Settings YAML Config Pattern

A pattern for application configuration using pydantic-settings v2 with YAML as primary source and environment variable override. Use this spec to create or adapt a configuration module for any Python service.

## Stack

```toml
pydantic-settings>=2.7.1
pyyaml>=6.0.2
# python >=3.12 (str | None syntax, list[T] lowercase generics)
```

## Structure rules

1. All config lives in a single `config.py`
2. Only `Settings` extends `BaseSettings`; all other models extend `BaseModel`
3. Last line of `config.py` is `settings = Settings()` — module-level singleton
4. All consumers do `from .config import settings` — never re-instantiate

## Model hierarchy

Build models bottom-up: leaf → composite → Settings.

### Leaf models

Simple typed structs. Field names match YAML keys exactly.

```python
class ServerConfig(BaseModel):
    host: str = "localhost"
    port: int = 8080

class WidgetConfig(BaseModel):
    name: str
    enabled: bool = True
    timeout_sec: int = 30
```

- Use camelCase field names only when the YAML key is camelCase (e.g. external/legacy schemas)
- Use `str | None = None` for optional string values (e.g. paths, resource limits)
- Use `Field(default_factory=list)` for list fields — never `= []`
- Resource limits and size strings stay as `str | None` (e.g. `"4GB"`) — not numeric

### Catalog / multi-backend container

When the service supports multiple pluggable backends, model each backend as a separate class and collect them in a container using nullable lists:

```python
from typing import Annotated

class BackendAEntry(BaseModel):
    instance: str        # logical name — always present on catalog entries
    ...

class BackendBEntry(BaseModel):
    instance: str
    ...

class CatalogEntry(BaseModel):
    backend_a: Annotated[list[BackendAEntry] | None, Field(default=None)]
    backend_b: Annotated[list[BackendBEntry] | None, Field(default=None)]
    # hyphenated YAML key → underscore Python name + alias:
    my_backend: Annotated[list[BackendBEntry] | None, Field(default=None, alias="my-backend")]
```

Use `Annotated[list[T] | None, Field(default=None)]` — not a bare `= None`. Omit absent backends from YAML entirely.

### Settings class

When all sections have sensible defaults, use `Field(default_factory=LeafModel)` so startup succeeds even when `application.yaml` is absent:

```python
class Settings(BaseSettings):
    # one field per top-level YAML section
    server: ServerConfig = Field(default_factory=ServerConfig)
    widget: WidgetConfig = Field(default_factory=WidgetConfig)

    model_config = SettingsConfigDict(
        env_file=".env",
        yaml_file=os.getenv("<SERVICE>_CONFIG", "application.yaml"),
        extra="ignore",
        populate_by_name=True,
        populate_by_alias=True,
    )

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls: type[BaseSettings],
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ) -> tuple[PydanticBaseSettingsSource, ...]:
        return (YamlConfigSettingsSource(settings_cls), env_settings, dotenv_settings)
```

`init_settings` and `file_secret_settings` are intentionally excluded from the chain.

Use `Field(default_factory=LeafModel)` when the section is fully optional (all fields have defaults). Use a bare type annotation (`server: ServerConfig`) when the section is required and must be present in the YAML.

## Source priority

Tuple order = lowest to highest precedence:

| Priority | Source | Notes |
|---|---|---|
| 1 (lowest) | YAML file | Path from `<SERVICE>_CONFIG` env var, default `application.yaml` |
| 2 | Environment variables | pydantic-settings default env parsing |
| 3 (highest) | `.env` file | Highest precedence in the tuple — use for local dev overrides only. Omit in production so real env vars take effect. |

## model_config flags

| Flag | Value | Purpose |
|---|---|---|
| `env_file` | `".env"` | Local dev dotenv; absent in prod |
| `yaml_file` | `os.getenv("<SERVICE>_CONFIG", "application.yaml")` | Runtime-selectable YAML path |
| `extra` | `"ignore"` | Unknown keys silently dropped — safe for forward-compat |
| `populate_by_name` | `True` | Python attribute name works even when alias defined |
| `populate_by_alias` | `True` | Alias (e.g. `my-backend`) works when loading from YAML |

## Environment variable overrides

### Naming the config env var

Name the YAML-path env var after the service, in `SCREAMING_SNAKE_CASE`:

```
<SERVICE_NAME>_CONFIG
```

Examples: `FLIGHT_CACHE_CONFIG`, `DATA_PROXY_CONFIG`, `CATALOG_SERVICE_CONFIG`.

### Switch the entire config file

```bash
<SERVICE>_CONFIG=application.production.yaml
```

Primary deployment mechanism — prefer named YAML files over per-value env var overrides.

### Override individual values

pydantic-settings maps nested fields via double-underscore separator, uppercased:

```bash
SERVER__HOST=0.0.0.0
SERVER__PORT=8080
WIDGET__TIMEOUT_SEC=60
```

Top-level field name becomes the prefix. No custom `env_prefix` needed.

## YAML file conventions

| File | Purpose |
|---|---|
| `application.yaml` | Default used in development; must be sufficient to start the service |
| `application.tmpl.yaml` | Template documenting every supported key — see below |
| `application.<name>.yaml` | Named variant for a specific environment or instance |

YAML structure mirrors the model hierarchy exactly. Using the `ServerConfig` / `WidgetConfig` models above:

```yaml
server:
  host: "localhost"
  port: 8080

widget:
  name: "default"
  enabled: true
  timeout_sec: 30
```

### `application.tmpl.yaml` conventions

The template must include every key the service recognises. Follow these rules:

- Required fields (no Python default): show a representative placeholder value with a `# required` comment.
- Optional fields: show their default value.
- Use inline comments to describe the field and list valid values or units.
- Use `null` for disabled / use-built-in-default semantics.
- Group keys under the same section headings as the model.

```yaml
# application.tmpl.yaml — copy to application.yaml and edit.

server:
  host: "localhost"       # bind address; use 0.0.0.0 to listen on all interfaces
  port: 8080              # TCP port

widget:
  name: "default"         # required — logical name for this widget instance
  enabled: true           # set false to disable widget processing
  timeout_sec: 30         # request timeout in seconds; increase for slow upstreams
  data_path: null         # null = use in-memory store; set to a file path to persist
```

## Hyphenated YAML keys

When a YAML key contains a hyphen (common in legacy or external schemas):

```python
my_backend: Annotated[list[MyEntry] | None, Field(default=None, alias="my-backend")]
```

Requires both `populate_by_name=True` and `populate_by_alias=True` in `model_config`.

## Naming conventions

| Context | Convention |
|---|---|
| YAML keys | snake_case (use camelCase only to match an external schema) |
| Python model fields | match YAML key exactly |
| Hyphenated alias fields | snake_case Python name + `Field(alias="hyphen-name")` |
| Catalog entry class names | `<BackendName>Entry` |
| Top-level config models | PascalCase |
| Settings class | always `Settings` |
| Singleton | always `settings` |
| Config env var | `<SERVICE_NAME>_CONFIG` in SCREAMING_SNAKE_CASE |

## Required imports

```python
# Always required
import os
from pydantic import BaseModel, Field
from pydantic_settings import (
    BaseSettings,
    SettingsConfigDict,
    PydanticBaseSettingsSource,
    YamlConfigSettingsSource,
)

# Catalog pattern only (multi-backend nullable lists)
from typing import Annotated   # list[T] is built-in on Python >=3.12; no need to import List
```

## Checklist for a new codebase

- [ ] Add `pydantic-settings>=2.7.1` and `pyyaml>=6.0.2` to dependencies
- [ ] Create `config.py` with all models; end with `settings = Settings()`
- [ ] Implement `settings_customise_sources` returning `(YamlConfigSettingsSource, env_settings, dotenv_settings)`
- [ ] Name the config env var `<SERVICE_NAME>_CONFIG`; set `yaml_file=os.getenv("<SERVICE_NAME>_CONFIG", "application.yaml")`
- [ ] Set `extra="ignore"`, `populate_by_name=True`, `populate_by_alias=True`
- [ ] Use `Field(default_factory=LeafModel)` on optional sections; bare annotation for required sections
- [ ] Create `application.yaml` (working defaults — service must start without edits)
- [ ] Create `application.tmpl.yaml` (every key, inline comments, `null` for disabled fields)
- [ ] Use `Field(alias="...")` for any hyphenated YAML key
- [ ] Use `Annotated[list[T] | None, Field(default=None)]` for optional catalog backend lists
- [ ] Resource limit / size fields → `str | None`, not `int`
- [ ] Mutable list defaults → `Field(default_factory=list)`

