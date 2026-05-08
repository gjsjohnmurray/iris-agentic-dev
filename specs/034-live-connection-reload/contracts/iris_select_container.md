# Contract: iris_select_container (updated)

## Change from previous behavior
Previously: probed the container but discarded the result (issue #11). After this feature: the probe result is stored as the active connection for the session.

## Request Parameters (unchanged)
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| name | string | yes | — | Docker container name to switch to |
| namespace | string | no | "USER" | IRIS namespace |
| username | string | no | "_SYSTEM" | IRIS username |
| password | string | no | "SYS" | IRIS password |

## Response — Success (updated)
```json
{
  "status": "ok",
  "switched": true,
  "container": "gqs-ivg-test",
  "port_superserver": 1972,
  "port_web": 52773,
  "namespace": "USER",
  "version": "IRIS for UNIX 2025.1",
  "write_tools_enabled": true
}
```
Note: `"note"` field with restart instruction removed. `"switched": true` added.

## Response — Failure (container not found or unreachable)
```json
{
  "success": false,
  "error": "CONTAINER_NOT_FOUND",
  "requested": "no-such-container",
  "available": ["iris-dev-iris", "gqs-ivg-test"]
}
```
Previous connection is preserved on failure.
