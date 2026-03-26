# Plan 07b: Tauri Plugin Adapter (macOS + Universal Tauri)

**Objective:** Build `tauri-plugin-mosaic` — a lightweight Tauri Rust plugin that provides JavaScript evaluation and event capture on ALL platforms (macOS, Linux, Windows) via Tauri's native webview APIs, plus the `TauriPluginAdapter` in Mosaic that connects to it.

**Depends on:** Plan 06

**Estimated scope:** M (1-3 days)

## Context

Research and 5-agent review confirmed that `WEBKIT_INSPECTOR_SERVER` does **NOT** work on macOS WKWebView. Apple's WKWebView uses private XPC APIs for Safari's Web Inspector — no WebSocket, no HTTP endpoint, nothing Mosaic can connect to externally.

However, Tauri v2 exposes two public APIs that work on ALL platforms including macOS:
- **`Webview::eval()`** — calls `WKWebView.evaluateJavaScript()` on macOS (public Apple API)
- **`WebviewBuilder::initialization_script()`** — uses `WKUserScript` on macOS (public Apple API)

This plan builds a Tauri plugin that uses these native APIs to provide the same capabilities as a debug protocol adapter: inject JavaScript, evaluate expressions, and capture structured events. The plugin communicates with the Mosaic CLI via a local WebSocket server with security mitigations.

**This adapter is the PRIMARY path for all Tauri testing.** It works identically on macOS, Linux, and Windows. The WebKit Inspector (Plan 07) and CDP (Plan 08) adapters remain as alternatives for non-Tauri targets or fallback scenarios.

## Research Findings

- **`Webview::eval()`** — fire-and-forget JS execution. Works on macOS via `evaluateJavaScript:completionHandler:`, Linux via WebKitGTK's `webkit_web_view_evaluate_javascript`, Windows via WebView2's `ExecuteScript`.
- **`initialization_script()`** — injects JS at document start, before HTML parsing. Re-runs on every top-level navigation. Uses `WKUserScript` (macOS), `UserScript` (WebKitGTK), `AddScriptToExecuteOnDocumentCreated` (WebView2).
- **`tauri-webdriver`** by Daniel Raffel — existing Tauri plugin that proves this architecture works: runs HTTP server inside the app, evaluates JS via native APIs, captures results via IPC.
- **Security requirements** (from CVE-2025-49596, CVE-2025-52882): origin validation, UUID token in WebSocket path, 127.0.0.1 only, single-client mode, ephemeral port.
- **Feature gating** — Cargo features allow the plugin to compile out of production builds entirely.

## Deliverables

### 1. `tauri-plugin-mosaic` (separate crate)

A Tauri v2 plugin crate that ships alongside the Mosaic CLI.

### 2. `TauriPluginAdapter` (in Mosaic binary)

An implementation of `ProtocolAdapter` that connects to the plugin's WebSocket server.

## Files to Create

### Plugin crate (new crate, lives in `plugins/tauri-plugin-mosaic/`)

```
plugins/tauri-plugin-mosaic/
├── Cargo.toml
├── src/
│   ├── lib.rs              — Plugin entry point
│   ├── server.rs           — Local WebSocket server for Mosaic CLI
│   ├── bridge.rs           — JS evaluation bridge
│   └── security.rs         — Token generation, origin validation
├── guest-js/
│   └── index.ts            — Tauri IPC client for event forwarding (optional)
└── README.md
```

### Mosaic binary additions

- `src/protocol/tauri_plugin.rs` — `TauriPluginAdapter` implementing `ProtocolAdapter`

## Files to Modify

- `src/protocol/mod.rs` — add tauri_plugin module
- `Cargo.toml` — no changes (the plugin is a separate crate; Mosaic connects to it over WebSocket)

## Implementation Details

### Plugin Architecture

```
┌──────────────────────────────────────────────────┐
│  Tauri App (with tauri-plugin-mosaic)             │
│                                                  │
│  ┌── Plugin ──────────────────────────────────┐  │
│  │                                            │  │
│  │  initialization_script() → injects guard   │  │
│  │  WebSocket server on 127.0.0.1:0 (ephem)   │  │
│  │  Announces port + token via stdout          │  │
│  │                                            │  │
│  │  On WS message "evaluate":                 │  │
│  │    → webview.eval(js)                      │  │
│  │                                            │  │
│  │  On Tauri IPC "mosaic:event":              │  │
│  │    → forward to WS client                  │  │
│  │                                            │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌── Webview ─────────────────────────────────┐  │
│  │  @mosaic/runtime (injected via eval)       │  │
│  │  Test file (imported via dev server)        │  │
│  │                                            │  │
│  │  Events → __TAURI_INTERNALS__.invoke(       │  │
│  │    'plugin:mosaic|event', { data })         │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
          ↕ WebSocket (ws://127.0.0.1:PORT/TOKEN)
┌──────────────────────────────────────────────────┐
│  mosaic CLI (TauriPluginAdapter)                  │
│                                                  │
│  Discovers port+token from app stdout            │
│  Connects WebSocket                              │
│  Sends: evaluate, import_test, run_test          │
│  Receives: MosaicEvent stream                    │
└──────────────────────────────────────────────────┘
```

### Plugin Entry Point (plugins/tauri-plugin-mosaic/src/lib.rs)

```rust
use tauri::{
    plugin::{Builder, TauriPlugin},
    Manager, Runtime, Webview,
};

mod bridge;
mod security;
mod server;

pub fn init<R: Runtime>() -> TauriPlugin<R> {
    Builder::new("mosaic")
        .setup(|app, _api| {
            // Only activate in debug builds
            if !cfg!(debug_assertions) {
                return Ok(());
            }

            // Generate security token
            let token = security::generate_token();

            // Start WebSocket server on ephemeral port
            let port = server::start(app.clone(), token.clone())?;

            // Announce port and token to stdout (Mosaic CLI reads this)
            println!("__MOSAIC_PLUGIN__:{{\"port\":{},\"token\":\"{}\"}}", port, token);

            Ok(())
        })
        .on_webview_ready(|webview| {
            // Inject a guard script that sets up the IPC bridge
            // This runs before any page JS
            let bridge_js = include_str!("../guest-js/bridge.js");
            webview.eval(bridge_js).ok();
        })
        .invoke_handler(tauri::generate_handler![mosaic_event])
        .build()
}

/// Receives events from the runtime in the webview and forwards to WS client.
#[tauri::command]
async fn mosaic_event(data: String, state: tauri::State<'_, server::MosaicServer>) -> Result<(), String> {
    state.forward_event(&data).await.map_err(|e| e.to_string())
}
```

### WebSocket Server (plugins/tauri-plugin-mosaic/src/server.rs)

```rust
use tokio::net::TcpListener;
use tokio_tungstenite::accept_async;
use futures_util::{StreamExt, SinkExt};
use std::sync::Arc;
use tokio::sync::{Mutex, broadcast};

pub struct MosaicServer {
    event_tx: broadcast::Sender<String>,
    ws_writer: Arc<Mutex<Option<SplitSink<...>>>>,
    eval_tx: broadcast::Sender<EvalRequest>,
}

struct EvalRequest {
    id: u64,
    javascript: String,
}

pub fn start<R: Runtime>(app: AppHandle<R>, token: String) -> Result<u16> {
    let listener = std::net::TcpListener::bind("127.0.0.1:0")?;
    let port = listener.local_addr()?.port();

    let server = MosaicServer::new();
    let server_state = server.clone();

    // Store in Tauri managed state
    app.manage(server.clone());

    // Spawn WebSocket server
    tokio::spawn(async move {
        let listener = TcpListener::from_std(listener).unwrap();

        // Accept ONE connection only (single-client mode)
        if let Ok((stream, _)) = listener.accept().await {
            // Verify the path contains the token
            let ws = accept_async(stream).await.unwrap();
            // TODO: Validate upgrade request path == /{token}
            // TODO: Validate Origin header

            let (writer, mut reader) = ws.split();
            *server_state.ws_writer.lock().await = Some(writer);

            // Process incoming commands from Mosaic CLI
            while let Some(Ok(msg)) = reader.next().await {
                if let Message::Text(text) = msg {
                    if let Ok(cmd) = serde_json::from_str::<Command>(&text) {
                        match cmd {
                            Command::Evaluate { id, javascript } => {
                                // Execute JS in webview via Tauri's eval()
                                if let Some(webview) = app.get_webview("main") {
                                    webview.eval(&javascript).ok();
                                }
                            }
                        }
                    }
                }
            }
        }
    });

    Ok(port)
}

impl MosaicServer {
    pub async fn forward_event(&self, data: &str) -> Result<()> {
        if let Some(ref mut writer) = *self.ws_writer.lock().await {
            writer.send(Message::Text(data.to_string())).await?;
        }
        Ok(())
    }
}
```

### Security (plugins/tauri-plugin-mosaic/src/security.rs)

```rust
use uuid::Uuid;

pub fn generate_token() -> String {
    Uuid::new_v4().to_string()
}

pub fn validate_origin(origin: Option<&str>, allowed_port: u16) -> bool {
    match origin {
        None => true,  // Local scripts (non-browser) send no Origin
        Some("null") => true,  // Some local contexts send "null"
        Some(o) => {
            // Only allow the app's own origin
            o == &format!("http://localhost:{}", allowed_port)
                || o == &format!("http://127.0.0.1:{}", allowed_port)
        }
    }
}
```

### Bridge Script (plugins/tauri-plugin-mosaic/guest-js/bridge.js)

Injected into the webview via `on_webview_ready`. Overrides the console event emitter so the runtime sends events via Tauri IPC instead of `console.warn`:

```javascript
// Guard against double-injection
if (!window.__MOSAIC_BRIDGE__) {
    window.__MOSAIC_BRIDGE__ = true;

    // Override the emitter to use Tauri IPC instead of console.warn
    window.__MOSAIC_EMIT__ = function(eventJson) {
        if (window.__TAURI_INTERNALS__) {
            window.__TAURI_INTERNALS__.invoke('plugin:mosaic|mosaic_event', {
                data: eventJson,
            });
        }
    };
}
```

The runtime's emitter (Plan 09) checks for this override:

```typescript
// In runtime/src/emitter.ts
export function emit(event: string, data: Record<string, unknown>): void {
    const json = JSON.stringify({ event, data });
    if ((globalThis as any).__MOSAIC_EMIT__) {
        // Tauri plugin path — IPC
        (globalThis as any).__MOSAIC_EMIT__(json);
    } else {
        // Debug protocol path — console capture
        console.warn(`__MOSAIC__:${json}`);
    }
}
```

### TauriPluginAdapter (src/protocol/tauri_plugin.rs)

```rust
use crate::protocol::adapter::*;
use tokio_tungstenite::connect_async;
use futures_util::{StreamExt, SinkExt};

pub struct TauriPluginAdapter {
    transport: Option<WebSocketTransport>,
    console_tx: broadcast::Sender<ConsoleEvent>,
    event_listener: Option<JoinHandle<()>>,
}

impl TauriPluginAdapter {
    pub fn new() -> Self {
        let (console_tx, _) = broadcast::channel(256);
        Self {
            transport: None,
            console_tx,
            event_listener: None,
        }
    }

    /// Discover the plugin's port and token from the app's stdout.
    pub async fn discover(app_stdout: &str) -> Option<(u16, String)> {
        for line in app_stdout.lines() {
            if let Some(json) = line.strip_prefix("__MOSAIC_PLUGIN__:") {
                if let Ok(info) = serde_json::from_str::<PluginInfo>(json) {
                    return Some((info.port, info.token));
                }
            }
        }
        None
    }
}

#[derive(Deserialize)]
struct PluginInfo {
    port: u16,
    token: String,
}

#[async_trait]
impl ProtocolAdapter for TauriPluginAdapter {
    async fn connect(&mut self, config: &ConnectionConfig) -> Result<()> {
        // Connect to the plugin's WebSocket server
        let url = format!(
            "ws://127.0.0.1:{}/{}",
            config.debug_port, config.token.as_deref().unwrap_or("")
        );
        let transport = WebSocketTransport::connect(&url).await?;

        // Listen for events from the plugin
        let mut events = transport.events();
        let console_tx = self.console_tx.clone();
        self.event_listener = Some(tokio::spawn(async move {
            while let Ok(event) = events.recv().await {
                // Plugin forwards raw MosaicEvent JSON — parse as ConsoleEvent
                if let Some(text) = event.as_str() {
                    let console_event = ConsoleEvent {
                        level: ConsoleLevel::Warn,
                        text: format!("__MOSAIC__:{}", text),
                        args: vec![],
                    };
                    let _ = console_tx.send(console_event);
                }
            }
        }));

        self.transport = Some(transport);
        Ok(())
    }

    async fn evaluate(&self, javascript: &str) -> Result<EvalResult> {
        let transport = self.transport.as_ref()
            .ok_or_else(|| anyhow!("Not connected to Tauri plugin"))?;

        // Send evaluate command to plugin
        transport.send_command("evaluate", json!({
            "javascript": javascript,
        })).await?;

        // eval() is fire-and-forget in Tauri, so we don't get a return value
        // Results come back through the event channel
        Ok(EvalResult { value: None, exception: None })
    }

    fn console_events(&self) -> broadcast::Receiver<ConsoleEvent> {
        self.console_tx.subscribe()
    }

    async fn close(&mut self) -> Result<()> {
        if let Some(handle) = self.event_listener.take() {
            handle.abort();
        }
        if let Some(transport) = self.transport.take() {
            transport.close().await?;
        }
        Ok(())
    }
}
```

### User Setup (one-time, automatable by `mosaic init --plugin`)

```toml
# src-tauri/Cargo.toml
[dependencies]
tauri-plugin-mosaic = { version = "0.1", optional = true }

[features]
default = []
mosaic = ["dep:tauri-plugin-mosaic"]
```

```rust
// src-tauri/lib.rs or src-tauri/main.rs
fn main() {
    let mut builder = tauri::Builder::default();

    #[cfg(feature = "mosaic")]
    {
        builder = builder.plugin(tauri_plugin_mosaic::init());
    }

    builder
        .run(tauri::generate_context!())
        .expect("error while running application");
}
```

Then start the app with:
```bash
cargo tauri dev --features mosaic
```

Or Mosaic does this automatically when it starts the dev server (Plan 05):
```rust
// In dev server config for Tauri projects
DevServerConfig {
    command_parts: vec!["cargo".into(), "tauri".into(), "dev".into(), "--features".into(), "mosaic".into()],
    // ...
}
```

## Acceptance Criteria

- [ ] `tauri-plugin-mosaic` compiles as a standalone Tauri v2 plugin crate
- [ ] Plugin only activates in debug builds (`cfg!(debug_assertions)`)
- [ ] Plugin starts a WebSocket server on 127.0.0.1 with an ephemeral port
- [ ] Port and token are announced via stdout in `__MOSAIC_PLUGIN__:{...}` format
- [ ] WebSocket validates origin header and token in path
- [ ] WebSocket accepts exactly one client connection (single-client mode)
- [ ] `evaluate` command calls `webview.eval(js)` on the target webview
- [ ] Bridge script forwards events from the webview to the WebSocket client via Tauri IPC
- [ ] `TauriPluginAdapter` implements `ProtocolAdapter` trait
- [ ] `TauriPluginAdapter` discovers the plugin from app stdout
- [ ] `TauriPluginAdapter` connects to the plugin's WebSocket
- [ ] Events from the webview are received by the adapter and emitted as `ConsoleEvent`s
- [ ] Works on macOS (WKWebView), Linux (WebKitGTK), and Windows (WebView2)
- [ ] Plugin compiles out when `mosaic` feature is not enabled
- [ ] The runtime emitter uses `__MOSAIC_EMIT__` when available (IPC path), falls back to `console.warn` (debug protocol path)
- [ ] Security: UUID token, origin validation, 127.0.0.1 only, ephemeral port, single-client
