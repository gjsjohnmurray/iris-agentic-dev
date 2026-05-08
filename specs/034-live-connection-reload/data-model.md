# Data Model: Live Connection Hot-Reload and check_config Tool

## New Types

### ConnectionState
Replaces the bare `Option<Arc<IrisConnection>>` stored in `IrisTools`. Holds all metadata needed by `check_config` alongside the live connection.

| Field | Type | Description |
|-------|------|-------------|
| `iris` | `Option<Arc<IrisConnection>>` | The live connection (None if unreachable) |
| `source` | `ConnectionSource` | How this connection was established |
| `config_file` | `Option<PathBuf>` | Absolute path to `.iris-dev.toml` that produced this connection (None for env/auto) |
| `loaded_at` | `SystemTime` | When this connection was last established (startup or reload) |
| `write_tools_enabled` | `bool` | Cached from `IrisConnection::is_write_allowed()` at probe time |
| `config_parse_error` | `Option<String>` | Last config parse error if reload failed; None if last load succeeded |

### ConnectionSource
```
ConfigFile          — loaded from .iris-dev.toml
EnvVars             — from IRIS_HOST / IRIS_CONTAINER env vars  
IrisSelectContainer — set by iris_select_container tool call
AutoDiscovered      — from localhost port scan
```

### ConfigWatcher
Owned by `IrisTools` as `Option<ConfigWatcher>`. None when no `.iris-dev.toml` exists.

| Field | Type | Description |
|-------|------|-------------|
| `config_path` | `PathBuf` | Absolute path to the `.iris-dev.toml` being watched |
| `last_mtime` | `SystemTime` | mtime at last load; compared on each tool call entry |

## Modified Types

### IrisTools (modified)
| Field | Before | After |
|-------|--------|-------|
| `iris` | `Option<Arc<IrisConnection>>` | removed — subsumed into `connection` |
| `write_tools_enabled` | `bool` | removed — now in `ConnectionState` |
| `connection` | *(new)* | `Arc<Mutex<ConnectionState>>` |
| `config_watcher` | *(new)* | `Option<ConfigWatcher>` |

All existing fields (`registry`, `client`, `history`, `elicitation_store`, `log_store`, `toolset`, `tool_router`) unchanged.

## Error Codes (new)

| Code | When |
|------|------|
| `CONFIG_RELOAD_FAILED` | Hot-reload attempted but new connection probe failed; old connection preserved |

## State Transitions

```
Session start
  → ConnectionState(source=ConfigFile|EnvVars|AutoDiscovered, loaded_at=now)
  → ConfigWatcher(path, mtime=now) if config_file exists

Tool call entry (check_reload)
  → mtime unchanged → no-op
  → mtime changed → reload config → probe new connection
      → probe ok  → swap ConnectionState(source=ConfigFile, loaded_at=now)
      → probe fail → keep old ConnectionState, set config_parse_error

iris_select_container called
  → probe new container
      → probe ok  → swap ConnectionState(source=IrisSelectContainer, loaded_at=now)
      → probe fail → keep old ConnectionState, return error

Next config file change after iris_select_container
  → file always wins → swap to ConfigFile source (FR-009 / clarification Q1)
```
