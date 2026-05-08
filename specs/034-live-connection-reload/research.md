# Research: Live Connection Hot-Reload and check_config Tool

## Architectural Decision: Mutex Wrapper for Live Swap

**Decision**: Replace `IrisTools.iris: Option<Arc<IrisConnection>>` with `Arc<Mutex<ConnectionState>>` where `ConnectionState` bundles the connection, metadata, and last-loaded timestamp.

**Rationale**: Tool handlers take `&self` (immutable) — they can't swap `self.iris` directly. Wrapping in `Arc<Mutex<...>>` allows any handler to acquire the lock, replace the inner value, and release. The `Arc` outer layer means clones (e.g. for background tasks) share the same state.

**Alternatives considered**:
- `tokio::sync::RwLock` — allows concurrent reads without blocking. Rejected: connection swap is rare; the added complexity of async locking in sync contexts isn't worth it. `std::sync::Mutex` with short critical sections is simpler.
- `Arc<RwLock<Option<Arc<IrisConnection>>>>` — deeply nested. Rejected: `ConnectionState` struct is cleaner.
- Keep `Option<Arc<IrisConnection>>` and add separate mutable state via `AtomicPtr` — rejected: unsafe and complex.

## Architectural Decision: Lazy mtime Check

**Decision**: Check `.iris-dev.toml` mtime at the start of each tool handler via `std::fs::metadata(path).modified()`. If mtime > last_seen, reload and re-probe.

**Rationale**: Clarification Q4 resolved this to lazy check. A `stat()` syscall costs ~1µs — negligible for tool calls that take 10–5000ms. No background task, no cancellation, no new concurrency surface.

**Implementation pattern**:
```
fn check_reload(&self) -> bool {
    let Some(ref watcher) = self.config_watcher else { return false };
    let Ok(meta) = std::fs::metadata(&watcher.config_path) else { return false };
    let Ok(mtime) = meta.modified() else { return false };
    if mtime <= watcher.last_mtime { return false }
    // mtime changed — reload
    ...
}
```

The tool macro wrapper can't be async for this pattern; the check itself is sync (`std::fs::metadata` is blocking but < 1ms). Connection probe is async — `check_reload` must be `async fn`.

## get_iris() Migration

Current: `fn get_iris(&self) -> Result<&IrisConnection, McpError>` — returns reference into `self.iris`.

New: `fn get_iris(&self) -> Result<Arc<IrisConnection>, McpError>` — clones the `Arc` out of the mutex. All call sites that were `iris.method(...)` become `iris.method(...)` unchanged — the deref coercion through `Arc` handles it. The return type change requires updating all ~40 call sites from `let iris = self.get_iris()?;` (no change to usage, just the inner `Arc` clone).

**Alternative**: Keep `&IrisConnection` by holding the mutex guard. Rejected: the guard lifetime would span the entire async tool call, blocking other tool calls from swapping the connection.

## iris_select_container Fix

Current code at line ~1480 in `mod.rs`:
```rust
// Bug 5: ... IrisTools.iris is Arc<IrisConnection> behind &self — can't be mutated here.
// Instead, probe the connection to verify it works and return accurate info.
let mut new_conn = IrisConnection::new(...);
new_conn.probe().await;
// ← new_conn dropped here, never stored
```

Fix: after probe, lock `self.connection.lock()` and replace with the new connection. Mark `connection_source: ConnectionSource::IrisSelectContainer`. Set `config_switch_epoch` to current timestamp so subsequent file changes (which reset the epoch) take precedence.

## ConnectionState Fields

```rust
pub struct ConnectionState {
    pub iris: Option<Arc<IrisConnection>>,
    pub source: ConnectionSource,
    pub config_file: Option<PathBuf>,
    pub loaded_at: std::time::SystemTime,
    pub write_tools_enabled: bool,
    pub config_parse_error: Option<String>,
}

pub enum ConnectionSource {
    ConfigFile,
    EnvVars,
    IrisSelectContainer,
    AutoDiscovered,
}
```

## ConfigWatcher Fields

```rust
pub struct ConfigWatcher {
    pub config_path: PathBuf,
    pub last_mtime: std::time::SystemTime,
}
```

## check_config Response — No IRIS Calls

`check_config` reads `ConnectionState` from `Arc<Mutex<ConnectionState>>` under a lock, then returns a JSON snapshot. No IRIS network calls made. `iris_version` comes from `connection.version` (cached from last probe). This is intentionally stale — re-probing on every `check_config` call would add unnecessary latency and network calls.

## Constitution II Verification

`IrisConnection::probe()` is an existing verified method. No new ObjectScript APIs introduced. Constitution II: N/A for this feature.
