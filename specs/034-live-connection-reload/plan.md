# Implementation Plan: Live Connection Hot-Reload and check_config Tool

**Branch**: `034-live-connection-reload` | **Date**: 2026-05-08 | **Spec**: `specs/034-live-connection-reload/spec.md`
**Input**: Feature specification from `/specs/034-live-connection-reload/spec.md`

## Summary

Three related changes that eliminate the need to restart the iris-dev MCP session when the target IRIS connection changes:

1. **Lazy config watcher** — a `ConfigWatcher` struct stored on `IrisTools` holds the `.iris-dev.toml` path and last-seen mtime. A new `check_reload()` method is called at the top of every tool handler; if mtime changed it reloads and re-probes (including SystemMode). The active connection is stored in `Arc<Mutex<Option<IrisConnection>>>` so the swap is atomic.

2. **Working `iris_select_container`** — fixes issue #11 by actually storing the new connection via the same mutex. After the call, `check_reload()` will not override it until the config file changes again.

3. **`check_config` tool** — reads the `ConnectionState` snapshot from `IrisTools` without making any IRIS calls. Always succeeds.

## Technical Context

**Language/Version**: Rust 1.92 (`crates/iris-dev-core` + `crates/iris-dev-bin`)
**Primary Dependencies**: No new crates. Uses `std::sync::Mutex`, `std::fs::metadata`, `std::time::SystemTime` (all std).
**Storage**: In-memory `Arc<Mutex<ConnectionState>>` on `IrisTools`
**Testing**: `cargo test` — unit tests (no IRIS), `#[ignore]` E2E tests against `iris-dev-iris`
**Target Platform**: macOS arm64/x86_64, Linux x86_64, Windows x86_64
**Performance Goals**: `stat()` syscall at tool call entry < 1ms overhead
**Constraints**: No new crate dependencies (Constitution VII). Lazy check — no background task.
**Scale/Scope**: Two struct changes + one new tool. Primary files: `src/tools/mod.rs`, `crates/iris-dev-bin/src/cmd/mcp.rs`

## Constitution Check

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Zero-Install Binary | ✅ PASS | No new crates. Single binary unchanged. |
| II. ObjectScript Sanity | ✅ PASS | No new ObjectScript APIs. Re-probe uses existing `IrisConnection::probe()` which is already verified. |
| III. HTTP-First Execution | ✅ PASS | `check_config` is HTTP-free. Connection swap is internal; doesn't change tool execution paths. |
| IV. Test-First, Fixture-Driven | ✅ PASS | Unit tests with `None` iris for `check_config`; `#[ignore]` E2E for swap + hot-reload. |
| V. Output Shape Parity | ✅ PASS | `check_config` is a new tool with its own contract. `iris_select_container` adds swap behavior but keeps the same response keys (plus `"switched": true`). |
| VI. Environment Guard | ✅ PASS | SystemMode re-probed on every connection swap — `write_tools_enabled` is always fresh after reload or select. |
| VII. Dependency Minimalism | ✅ PASS | Zero new crates. `std::fs::metadata` for mtime. `std::sync::Mutex` already in workspace. |

**Post-design re-check**: All gates pass.

## Project Structure

### Documentation (this feature)

```text
specs/034-live-connection-reload/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output — ConnectionState, ConfigWatcher
├── quickstart.md        # Phase 1 output
├── contracts/
│   ├── check_config.md  # New tool contract
│   └── iris_select_container.md  # Updated contract
└── tasks.md             # Phase 2 output
```

### Source Code

```text
# Core library — crates/iris-dev-core/src/tools/mod.rs
#   - IrisTools.iris: Option<Arc<IrisConnection>> → Arc<Mutex<ConnectionState>>
#   - IrisTools: add config_watcher: Option<ConfigWatcher> field
#   - Add ConnectionState struct (snapshot for check_config)
#   - Add ConfigWatcher struct (path + last_mtime)
#   - Add IrisTools::check_reload() — stat + conditional reconnect
#   - Add IrisTools::get_connection_state() → ConnectionState snapshot
#   - All tool handlers: call self.check_reload().await at entry
#   - iris_select_container: actually store new connection after probe
#   - New check_config tool handler
#   - Update get_iris() to read from Arc<Mutex<...>>

# Binary — crates/iris-dev-bin/src/cmd/mcp.rs
#   - Pass config file path to IrisTools at construction

# Tests
crates/iris-dev-core/tests/unit/test_live_reload.rs      # NEW — unit tests (no IRIS)
crates/iris-dev-core/tests/integration/test_live_reload_e2e.rs  # NEW — #[ignore] E2E
```

**Structure Decision**: `ConnectionState` is a plain struct (not Arc'd separately) — it's a snapshot captured under the mutex lock and returned by value. `ConfigWatcher` is an `Option<ConfigWatcher>` on `IrisTools` — None when no `.iris-dev.toml` exists to watch.

## Complexity Tracking

| Decision | Why Needed | Simpler Alternative Rejected Because |
|----------|------------|--------------------------------------|
| `Arc<Mutex<ConnectionState>>` instead of `Option<Arc<IrisConnection>>` | Must swap connection atomically from `&self` tool handlers | `&mut self` impossible in `#[tool]` handlers; `Arc<RwLock<...>>` overkill for this access pattern |
| Lazy mtime check at tool entry | Avoids background task and cancellation complexity | Background tokio task adds concurrency surface; clarification Q4 resolved this |
| Re-probe SystemMode on every swap | Safety gate must be accurate after swap | Inheriting stale `write_tools_enabled` could allow writes on a newly-connected prod instance |
| `iris_select_container` switch persists until next config file change | File always wins (clarification Q1) | Sticky switch would ignore developer's intentional file edit |
