# Quickstart: Live Connection Hot-Reload and check_config

## Check active connection
```json
{"name": "check_config", "arguments": {}}
```
Returns full connection state including host, container, when config was loaded, and whether write tools are enabled.

## Hot-reload in action
1. Agent is working against `iris-dev-iris`
2. User edits `.iris-dev.toml`: changes `container = "iris-dev-iris"` to `container = "gqs-ivg-test"`
3. User saves file — **no session restart needed**
4. Next tool call automatically uses `gqs-ivg-test`
5. Agent can call `check_config` to verify: `connection_source: "config_file"`, `container: "gqs-ivg-test"`

## Explicit container switch
```json
{"name": "iris_select_container", "arguments": {"name": "gqs-ivg-test", "namespace": "USER"}}
```
All subsequent tool calls in the session use `gqs-ivg-test`. If `.iris-dev.toml` is later updated on disk, the file change takes precedence.

## Diagnosing a broken connection
```json
{"name": "check_config", "arguments": {}}
```
If `connected: false`: check `host`, `port`, `container` fields and verify the container is running or the host is reachable.  
If `config_parse_error` is set: `.iris-dev.toml` has a syntax error — the last good connection is still active.
