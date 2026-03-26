# Plan 08: CDP Adapter

**Objective:** Implement the Chrome DevTools Protocol adapter using the shared transport layer — supporting `Runtime.evaluate`, `Runtime.consoleAPICalled` event capture, and target discovery for Tauri on Windows and any Chromium-based target.

**Depends on:** Plan 06

**Estimated scope:** M (1-3 days)

## Context

Tauri on Windows uses WebView2 (Chromium), which exposes CDP automatically in dev mode. This adapter also serves Electron apps and Chrome browser targets. Since Node.js Inspector uses the same wire protocol (V8 Inspector = CDP subset), this adapter can be reused for Node.js backends in v1.1 with minimal modification.

## Research Findings

- **CDP stable: v1.3**, but most tools use tip-of-tree. For Mosaic's minimal needs, stable is sufficient.
- **Runtime.consoleAPICalled** is the correct event for console capture in CDP. `Console.messageAdded` is deprecated.
- **Runtime.consoleAPICalled** delivers `args` as an array of `RemoteObject` — each arg needs serialization to extract the text.
- **`awaitPromise: true`** in `Runtime.evaluate` params — resolves Promises before returning. Critical for `await import(...)`.
- **cdp-protocol crate** exists for types but adds dependency weight. For v1's minimal needs (2 methods + 1 event), hand-written types are simpler and consistent with the WebKit adapter approach.
- **Node.js Inspector** is the same wire protocol with fewer domains. Default port is 9229 instead of 9222. `/json/list` endpoint works the same way.

## Files to Create

- `src/protocol/cdp.rs` — CDP adapter implementing `ProtocolAdapter`
- `src/protocol/cdp_types.rs` — CDP-specific message type definitions

## Files to Modify

- `src/protocol/mod.rs` — add cdp module

## Implementation Details

### CDP-Specific Types

```rust
use serde::{Deserialize, Serialize};

// -- Request params --

#[derive(Serialize)]
pub struct CdpRuntimeEvaluateParams {
    pub expression: String,
    #[serde(skip_serializing_if = "Option::is_none", rename = "returnByValue")]
    pub return_by_value: Option<bool>,
    #[serde(skip_serializing_if = "Option::is_none", rename = "awaitPromise")]
    pub await_promise: Option<bool>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub silent: Option<bool>,
}

// -- Response types --

#[derive(Deserialize)]
pub struct CdpEvaluateResult {
    pub result: CdpRemoteObject,
    #[serde(rename = "exceptionDetails")]
    pub exception_details: Option<CdpExceptionDetails>,
}

#[derive(Deserialize)]
pub struct CdpRemoteObject {
    #[serde(rename = "type")]
    pub object_type: String,
    pub value: Option<serde_json::Value>,
    pub description: Option<String>,
    #[serde(rename = "className")]
    pub class_name: Option<String>,
    #[serde(rename = "subtype")]
    pub sub_type: Option<String>,
}

#[derive(Deserialize)]
pub struct CdpExceptionDetails {
    pub text: String,
    #[serde(rename = "lineNumber")]
    pub line_number: Option<u32>,
    #[serde(rename = "columnNumber")]
    pub column_number: Option<u32>,
    pub exception: Option<CdpRemoteObject>,
}

// -- Console event --

#[derive(Deserialize)]
pub struct CdpConsoleAPICalled {
    #[serde(rename = "type")]
    pub call_type: String,  // "log", "debug", "info", "error", "warning", "dir", "trace", etc.
    pub args: Vec<CdpRemoteObject>,
    #[serde(rename = "executionContextId")]
    pub execution_context_id: Option<u64>,
    pub timestamp: Option<f64>,
}
```

### CDP Adapter

```rust
pub struct CdpAdapter {
    transport: Option<WebSocketTransport>,
    console_tx: broadcast::Sender<ConsoleEvent>,
    event_listener: Option<JoinHandle<()>>,
}

#[async_trait]
impl ProtocolAdapter for CdpAdapter {
    async fn connect(&mut self, config: &ConnectionConfig) -> Result<()> {
        // 1. Discover targets
        let targets = discover_targets(&config.host, config.debug_port).await?;
        let target = find_page_target(&targets)
            .ok_or_else(|| anyhow!("No inspectable page target found"))?;

        // 2. Connect WebSocket
        let transport = WebSocketTransport::connect(&target.ws_url).await?;

        // 3. Enable Runtime domain (Console domain is deprecated in CDP)
        transport.send_command("Runtime.enable", json!({})).await?;

        // 4. Listen for Runtime.consoleAPICalled events
        let mut events = transport.events();
        let console_tx = self.console_tx.clone();
        self.event_listener = Some(tokio::spawn(async move {
            while let Ok(event) = events.recv().await {
                if let Some(method) = event.get("method").and_then(|m| m.as_str()) {
                    if method == "Runtime.consoleAPICalled" {
                        if let Some(params) = event.get("params") {
                            if let Ok(call) = serde_json::from_value::<CdpConsoleAPICalled>(params.clone()) {
                                let text = serialize_cdp_args(&call.args);
                                let console_event = ConsoleEvent {
                                    level: parse_cdp_level(&call.call_type),
                                    text,
                                    args: call.args.iter()
                                        .filter_map(|a| a.value.clone())
                                        .collect(),
                                };
                                let _ = console_tx.send(console_event);
                            }
                        }
                    }
                }
            }
        }));

        self.transport = Some(transport);
        Ok(())
    }

    async fn evaluate(&self, javascript: &str) -> Result<EvalResult> {
        let transport = self.transport.as_ref()
            .ok_or_else(|| anyhow!("Not connected"))?;

        let response = transport.send_command("Runtime.evaluate", json!({
            "expression": javascript,
            "returnByValue": true,
            "awaitPromise": true,  // Critical: resolves Promises
        })).await?;

        let result = response.get("result")
            .ok_or_else(|| anyhow!("No result in evaluate response"))?;

        if let Ok(eval_result) = serde_json::from_value::<CdpEvaluateResult>(result.clone()) {
            if let Some(exception) = eval_result.exception_details {
                let msg = exception.exception
                    .and_then(|e| e.description)
                    .unwrap_or(exception.text);
                return Ok(EvalResult {
                    value: None,
                    exception: Some(msg),
                });
            }
            return Ok(EvalResult {
                value: eval_result.result.value,
                exception: None,
            });
        }

        Ok(EvalResult { value: None, exception: Some("Failed to parse evaluate result".into()) })
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

### Serializing CDP Console Args

CDP delivers console args as RemoteObjects, not pre-formatted text. We need to extract readable text:

```rust
fn serialize_cdp_args(args: &[CdpRemoteObject]) -> String {
    args.iter()
        .map(|arg| {
            // Prefer value (for primitives with returnByValue)
            if let Some(ref value) = arg.value {
                match value {
                    serde_json::Value::String(s) => s.clone(),
                    other => other.to_string(),
                }
            } else if let Some(ref desc) = arg.description {
                desc.clone()
            } else {
                format!("[{} object]", arg.object_type)
            }
        })
        .collect::<Vec<_>>()
        .join(" ")
}
```

### CDP Console Level Mapping

```rust
fn parse_cdp_level(call_type: &str) -> ConsoleLevel {
    match call_type {
        "log" => ConsoleLevel::Log,
        "info" => ConsoleLevel::Info,
        "warning" | "warn" => ConsoleLevel::Warn,
        "error" => ConsoleLevel::Error,
        "debug" => ConsoleLevel::Debug,
        _ => ConsoleLevel::Log,
    }
}
```

### Key Differences from WebKit Adapter

| Aspect | WebKit | CDP |
|--------|--------|-----|
| Console event | `Console.messageAdded` | `Runtime.consoleAPICalled` |
| Console text | Pre-formatted `text` field | `args` array (needs serialization) |
| Domain enable | `Console.enable` + `Runtime.enable` | `Runtime.enable` only |
| Promise handling | N/A | `awaitPromise: true` |
| Exception reporting | `wasThrown` boolean | `exceptionDetails` object |

## Acceptance Criteria

- [ ] `CdpAdapter` implements `ProtocolAdapter` trait
- [ ] Connects to CDP target via `/json` endpoint discovery
- [ ] Sends `Runtime.enable` on connect (does NOT send `Console.enable` — deprecated)
- [ ] `evaluate()` sends `Runtime.evaluate` with `awaitPromise: true`
- [ ] `evaluate()` correctly reports exceptions via `exceptionDetails`
- [ ] Console events captured via `Runtime.consoleAPICalled` and broadcast
- [ ] CDP args are serialized to readable text (strings, numbers, objects)
- [ ] Console level mapping handles both `"warning"` and `"warn"`
- [ ] `close()` cleanly disconnects
- [ ] Integration test against Chrome/Chromium with `--remote-debugging-port=9222` (or documented manual test)
- [ ] Same transport layer is used as WebKit adapter (no code duplication)
