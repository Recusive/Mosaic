# Plan 04: Auto-Detection System

**Objective:** Implement project type detection (Tauri v1/v2), dev server config reading (Vite port, devUrl), OS-based protocol selection, and port scanning — producing a `DetectedEnvironment` struct that downstream systems consume.

**Depends on:** Plan 01

**Estimated scope:** M (1-3 days)

## Context

Auto-detection is the "zero config" backbone. When an agent runs `mosaic test`, this system figures out: what kind of project is this? Where's the dev server? What protocol should I use? What port? The agent never touches a config file unless auto-detection fails.

## Research Findings

- **Tauri v2** (`2.10.3`): `build.devUrl` in `tauri.conf.json`. Always a URL (not a filesystem path).
- **Tauri v1**: `build.devPath` in `tauri.conf.json`. Can be URL or filesystem path.
- **Config file locations**: `src-tauri/tauri.conf.json` (most common), `tauri.conf.json` (flat), `src-tauri/Tauri.toml` (TOML users).
- **Tauri v2 supports**: JSON (default), JSON5 (`config-json5` feature), TOML (`config-toml` feature). JSON and TOML are sufficient for v1.
- **Vite config**: `vite.config.ts` / `vite.config.js`. Read `server.port` (default 5173). For Tauri projects, devUrl in tauri.conf.json takes precedence.
- **OS detection**: `std::env::consts::OS` — `"macos"`, `"linux"`, `"windows"`.
- **Port scanning**: `TcpStream::connect()` with timeout — the standard approach.
- **macOS risk**: `WEBKIT_INSPECTOR_SERVER` does NOT work on macOS WKWebView. See Risk section.

## Files to Create

- `src/detect/tauri.rs` — Tauri project detection and config parsing
- `src/detect/vite.rs` — Vite config reading
- `src/detect/port.rs` — port scanning utilities
- `src/detect/environment.rs` — `DetectedEnvironment` struct and orchestrator

## Files to Modify

- `src/detect/mod.rs` — re-export detection API

## Implementation Details

### DetectedEnvironment

```rust
#[derive(Debug)]
pub struct DetectedEnvironment {
    pub project_type: ProjectType,
    pub dev_server_url: String,
    pub dev_server_port: u16,
    pub dev_server_running: bool,
    pub debug_port: u16,
    pub protocol: ProtocolKind,
    pub start_command: Option<String>,
    pub tauri_config_path: Option<PathBuf>,
}

#[derive(Debug, Clone)]
pub enum ProjectType {
    Tauri { version: TauriVersion },
}

#[derive(Debug, Clone)]
pub enum TauriVersion {
    V1,
    V2,
}

#[derive(Debug, Clone)]
pub enum ProtocolKind {
    WebKit,
    Cdp,
}
```

### Tauri Detection (src/detect/tauri.rs)

Search order:
1. `src-tauri/tauri.conf.json`
2. `tauri.conf.json`
3. `src-tauri/Tauri.toml`
4. `Tauri.toml`

For JSON configs:
```rust
#[derive(Deserialize)]
struct TauriConfig {
    build: Option<BuildConfig>,
}

#[derive(Deserialize)]
struct BuildConfig {
    #[serde(alias = "devUrl", alias = "devPath")]
    dev_url: Option<String>,
    #[serde(alias = "beforeDevCommand")]
    before_dev_command: Option<String>,
}
```

- `devUrl` (v2) and `devPath` (v1) handled by `#[serde(alias)]`.
- If `devPath` is a filesystem path (doesn't start with `http`), default to `http://localhost:1420`.
- Detect v1 vs v2 by checking for `devUrl` (v2) vs `devPath` (v1), or by the presence of top-level `identifier` field (v2 only).

For TOML configs:
```rust
let config: TauriConfig = toml::from_str(&contents)?;
```

Same struct, `toml` crate uses the same serde derive.

### Vite Detection (src/detect/vite.rs)

Check for `vite.config.ts` or `vite.config.js`. If found, attempt a lightweight parse for `server.port`. Since Vite config is TypeScript/JavaScript (not JSON), full parsing is hard. Strategy:

1. Regex for `port:\s*(\d+)` — catches the common case.
2. If no match, default to 5173 (Vite's default).
3. For Tauri projects, `devUrl` from tauri.conf.json takes precedence over Vite config.

This is a best-effort heuristic. The `--port` flag overrides everything.

### Port Scanning (src/detect/port.rs)

```rust
use tokio::net::TcpStream;
use tokio::time::{timeout, Duration};

pub async fn is_port_open(port: u16) -> bool {
    let addr = format!("127.0.0.1:{}", port);
    timeout(Duration::from_millis(500), TcpStream::connect(&addr))
        .await
        .map(|r| r.is_ok())
        .unwrap_or(false)
}
```

### OS-Based Protocol Selection

```rust
pub fn detect_protocol() -> ProtocolKind {
    match std::env::consts::OS {
        "macos" | "linux" => ProtocolKind::WebKit,
        "windows" => ProtocolKind::Cdp,
        _ => ProtocolKind::WebKit, // Default to WebKit
    }
}
```

### Detection Orchestrator (src/detect/environment.rs)

```rust
pub async fn detect(cwd: &Path) -> Result<DetectedEnvironment, DetectionError> {
    // 1. Try Tauri detection
    if let Some(tauri) = detect_tauri(cwd)? {
        let port = parse_port_from_url(&tauri.dev_url).unwrap_or(1420);
        let running = is_port_open(port).await;
        let protocol = detect_protocol();
        let debug_port = 9222; // Default for all protocols

        return Ok(DetectedEnvironment {
            project_type: ProjectType::Tauri { version: tauri.version },
            dev_server_url: tauri.dev_url,
            dev_server_port: port,
            dev_server_running: running,
            debug_port,
            protocol,
            start_command: Some(build_tauri_start_command(&protocol)),
            tauri_config_path: Some(tauri.config_path),
        });
    }

    Err(DetectionError::NoProjectFound)
}
```

### Tauri Plugin Detection

For Tauri projects, the PRIMARY adapter is `tauri-plugin-mosaic` (Plan 07b), which works on ALL platforms including macOS. The detection system should:

1. Check if `tauri-plugin-mosaic` is in `src-tauri/Cargo.toml` dependencies
2. If present, start the app with `cargo tauri dev --features mosaic`
3. If absent, report actionable error: "Run `mosaic init --plugin` to add Tauri testing support"

```rust
fn build_tauri_start_command() -> Vec<String> {
    vec![
        "cargo".into(),
        "tauri".into(),
        "dev".into(),
        "--features".into(),
        "mosaic".into(),
    ]
}

fn has_mosaic_plugin(cwd: &Path) -> bool {
    let cargo_toml = cwd.join("src-tauri/Cargo.toml");
    if let Ok(contents) = std::fs::read_to_string(&cargo_toml) {
        contents.contains("tauri-plugin-mosaic")
    } else {
        false
    }
}
```

**Note:** `WEBKIT_INSPECTOR_SERVER` does NOT work on macOS. The Tauri plugin approach bypasses this limitation entirely by using Tauri's native `Webview::eval()` API.

## Acceptance Criteria

- [ ] Detects Tauri v2 project from `src-tauri/tauri.conf.json` and reads `build.devUrl`
- [ ] Detects Tauri v1 project and reads `build.devPath`
- [ ] Handles both JSON and TOML config formats
- [ ] Falls back to port 1420 when `devUrl` is missing
- [ ] Port scanner correctly reports open/closed ports
- [ ] Selects WebKit protocol on macOS/Linux, CDP on Windows
- [ ] Returns `DetectionError::NoProjectFound` when no project files found
- [ ] Unit tests with fixture config files covering: v1 JSON, v2 JSON, v2 TOML, missing devUrl, malformed JSON
- [ ] All detection results are overridable by CLI flags (verified conceptually — actual override wiring is in Plan 16)
