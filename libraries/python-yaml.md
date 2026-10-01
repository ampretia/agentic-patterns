# Pydantic-Settings YAML Config Pattern

A pattern for application configuration using pydantic-settings v2 with YAML as primary source and environment variable override. Use this spec to create or adapt a configuration module for any Python service.

## Stack

```toml
pydantic-settings>=2.7.1
pyyaml>=6.0.2
# python >=3.12 (str | None syntax, type[T] lowercase generics)
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
    host: str
    port: int

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
class BackendAEntry(BaseModel):
    instance: str        # logical name — always present on catalog entries
    ...

class BackendBEntry(BaseModel):
    instance: str
    ...

class CatalogEntry(BaseModel):
    backend_a: Annotated[List[BackendAEntry] | None, Field(default=None)]
    backend_b: Annotated[List[BackendBEntry] | None, Field(default=None)]
    # hyphenated YAML key → underscore Python name + alias:
    my_backend: Annotated[List[...] | None, Field(default=None)] = Field(alias="my-backend")
```

Use `Annotated[List[T] | None, Field(default=None)]` — not a bare `= None`. Omit absent backends from YAML entirely.

### Settings class

```python
class Settings(BaseSettings):
    # one field per top-level YAML section
    server: ServerConfig
    catalog: CatalogEntry
    datasource: DataEntry

    model_config = SettingsConfigDict(
        env_file=".env",
        yaml_file=os.getenv("APP_CONFIG", "application.yaml"),
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

## Source priority

Tuple order = lowest to highest precedence:

| Priority | Source | Notes |
|---|---|---|
| 1 (lowest) | YAML file | Path from `APP_CONFIG` env var, default `application.yaml` |
| 2 | Environment variables | pydantic-settings default env parsing |
| 3 (highest) | `.env` file | Overrides env vars; omit in production |

## model_config flags

| Flag | Value | Purpose |
|---|---|---|
| `env_file` | `".env"` | Local dev dotenv; absent in prod |
| `yaml_file` | `os.getenv("APP_CONFIG", "application.yaml")` | Runtime-selectable YAML path |
| `extra` | `"ignore"` | Unknown keys silently dropped — safe for forward-compat |
| `populate_by_name` | `True` | Python attribute name works even when alias defined |
| `populate_by_alias` | `True` | Alias (e.g. `my-backend`) works when loading from YAML |

## Environment variable overrides

### Switch the entire config file

```bash
APP_CONFIG=application.production.yaml
```

Primary deployment mechanism — prefer named YAML files over per-value env var overrides.

### Override individual values

pydantic-settings maps nested fields via double-underscore separator, uppercased:

```bash
SERVER__HOST=0.0.0.0
SERVER__PORT=8080
DATASOURCE__MEMORY_LIMIT=8GB
DATASOURCE__DB_PATH=/tmp/mydb.db
```

Top-level field name becomes the prefix. No custom `env_prefix` needed.

## YAML file conventions

| File | Purpose |
|---|---|
| `application.yaml` | Default used in development |
| `application.tmpl.yaml` | Template showing every supported key with comments |
| `application.<name>.yaml` | Named variant for a specific environment or instance |

YAML structure mirrors the model hierarchy exactly. Example:

```yaml
server:
  host: "localhost"
  port: 8080

catalog:
  backend_a:
    - instance: "primary"
      url: "http://..."

datasource:
  db_path: null          # null = in-memory
  memory_limit: "4GB"
  priority_tables: []
```

## Hyphenated YAML keys

When a YAML key contains a hyphen (common in legacy or external schemas):

```python
# Python attribute
my_backend: Annotated[List[MyEntry] | None, Field(default=None)] = Field(alias="my-backend")
```

Requires both `populate_by_name=True` and `populate_by_alias=True` in `model_config`.

## Naming conventions

| Context | Convention |
|---|---|
| YAML keys | snake_case (use camelCase only to match an external schema) |
| Python model fields | match YAML key exactly |
| Hyphenated alias fields | snake_case Python name + `Field(alias="hyphen-name")` |
| Catalog entry class names | `<backendName>CatalogEntry` or `<backendName>Entry` |
| Top-level config models | PascalCase |
| Settings class | always `Settings` |
| Singleton | always `settings` |

## Required imports

```python
import os
from typing import List, Annotated
from pydantic import BaseModel, Field
from pydantic_settings import (
    BaseSettings,
    SettingsConfigDict,
    PydanticBaseSettingsSource,
    YamlConfigSettingsSource,
)
```

## Checklist for a new codebase

- [ ] Add `pydantic-settings>=2.7.1` and `pyyaml>=6.0.2` to dependencies
- [ ] Create `config.py` with all models; end with `settings = Settings()`
- [ ] Implement `settings_customise_sources` returning `(YamlConfigSettingsSource, env_settings, dotenv_settings)`
- [ ] Set `yaml_file=os.getenv("APP_CONFIG", "application.yaml")` — replace `APP_CONFIG` with your service's env var name
- [ ] Set `extra="ignore"`, `populate_by_name=True`, `populate_by_alias=True`
- [ ] Create `application.yaml` (defaults) and `application.tmpl.yaml` (all keys documented)
- [ ] Use `Field(alias="...")` for any hyphenated YAML key
- [ ] Use `Annotated[List[T] | None, Field(default=None)]` for optional catalog backend lists
- [ ] Resource limit / size fields → `str | None`, not `int`
- [ ] Mutable list defaults → `Field(default_factory=list)`

