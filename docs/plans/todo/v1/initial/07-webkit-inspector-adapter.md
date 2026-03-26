# Plan 07: WebKit Inspector Protocol Adapter

**Objective:** Implement the WebKit Inspector Protocol adapter using the shared transport layer — supporting `Runtime.evaluate`, `Console.messageAdded` event capture, and target discovery for Tauri on Linux.

**Depends on:** Plan 06

**Estimated scope:** M (1-3 days)

## Context

Tauri on Linux uses WebKitGTK, which exposes a WebKit Inspector Protocol via `WEBKIT_INSPECTOR_HTTP_SERVER`. This adapter speaks that protocol to inject JavaScript and capture console events. The message format is similar to CDP but not identical — different method names, different console event structure.

**Important:** This adapter works on Linux only. macOS WKWebView does not support `WEBKIT_INSPECTOR_SERVER` (see Risk section in Plan 04). macOS support requires a separate investigation and is not covered by this plan.

## Research Findings

- **WebKit Inspector Protocol** uses JSON-RPC 2.0 over WebSocket — same transport as CDP.
- **No Rust crates exist** for WebKit Inspector. Hand-written types are required.
- **Key methods:** `Runtime.enable`, `Runtime.evaluate`, `Console.enable`, `Console.messageAdded`.
- **Console.messageAdded** delivers a `ConsoleMessage` with `source`, `level`, `text`, and optional `parameters` array.
- **Runtime.evaluate** parameters: `expression` (required), `returnByValue`, `generatePreview`, `contextId`.
- **WEBKIT_INSPECTOR_HTTP_SERVER** (not `WEBKIT_INSPECTOR_SERVER`) provides the HTTP+WebSocket interface on modern WebKitGTK (2.37.1+).
- The `/json` endpoint returns a list of inspectable targets, similar to CDP.
- Protocol spec lives in WebKit source: `Source/JavaScriptCore/inspector/protocol/*.json`.

## Files to Create

- `src/protocol/webkit.rs` — WebKit Inspector adapter implementing `ProtocolAdapter`
- `src/protocol/webkit_types.rs` — WebKit-specific message type definitions

## Files to Modify

- `src/protocol/mod.rs` — add webkit module

## Implementation Details

### WebKit-Specific Types

```rust
use serde::{Deserialize, Serialize};

// -- Request params --

#[derive(Serialize)]
pub struct RuntimeEvaluateParams {
    pub expression: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    #[serde(rename = "returnByValue")]
    pub return_by_value: Option<bool>,
    #[serde(skip_serializing_if = "Option::is_none")]
    #[serde(rename = "generatePreview")]
    pub generate_preview: Option<bool>,
}

// -- Response types --

#[derive(Deserialize)]
pub struct RuntimeEvaluateResult {
    pub result: RemoteObject,
    #[serde(rename = "wasThrown")]
    pub was_thrown: Option<bool>,
}

#[derive(Deserialize)]
pub struct RemoteObject {
    #[serde(rename = "type")]
    pub object_type: String,
    pub value: Option<serde_json::Value>,
    pub description: Option<String>,
    #[serde(rename = "className")]
    pub class_name: Option<String>,
}

// -- Event types --

#[derive(Deserialize)]
pub struct ConsoleMessageAdded {
    pub message: ConsoleMessage,
}

#[derive(Deserialize)]
pub struct ConsoleMessage {
    pub source: String,
    pub level: String,     // "log", "info", "warning", "error", "debug"
    pub text: String,
    #[serde(rename = "type")]
    pub message_type: Option<String>,
    pub url: Option<String>,
    pub line: Option<u32>,
    pub column: Option<u32>,
    pub parameters: Option<Vec<RemoteObject>>,
}
```

### WebKit Adapter

```rust
pub struct WebKitAdapter {
    transport: Option<WebSocketTransport>,
    console_tx: broadcast::Sender<ConsoleEvent>,
    event_listener: Option<JoinHandle<()>>,
}

#[async_trait]
impl ProtocolAdapter for WebKitAdapter {
    async fn connect(&mut self, config: &ConnectionConfig) -> Result<()> {
        // 1. Discover targets via /json endpoint
        let targets = discover_targets(&config.host, config.debug_port).await?;
        let target = find_page_target(&targets)
            .ok_or_else(|| anyhow!("No inspectable page target found"))?;

        // 2. Connect WebSocket to the target
        let transport = WebSocketTransport::connect(&target.ws_url).await?;

        // 3. Enable Runtime and Console domains
        transport.send_command("Runtime.enable", json!({})).await?;
        transport.send_command("Console.enable", json!({})).await?;

        // 4. Start listening for console events
        let mut events = transport.events();
        let console_tx = self.console_tx.clone();
        self.event_listener = Some(tokio::spawn(async move {
            while let Ok(event) = events.recv().await {
                if let Some(method) = event.get("method").and_then(|m| m.as_str()) {
                    if method == "Console.messageAdded" {
                        if let Some(params) = event.get("params") {
                            if let Ok(msg) = serde_json::from_value::<ConsoleMessageAdded>(params.clone()) {
                                let console_event = ConsoleEvent {
                                    level: parse_webkit_level(&msg.message.level),
                                    text: msg.message.text.clone(),
                                    args: msg.message.parameters
                                        .as_ref()
                                        .map(|p| p.iter().filter_map(|r| r.value.clone()).collect())
                                        .unwrap_or_default(),
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
        })).await?;

        let result = response.get("result")
            .ok_or_else(|| anyhow!("No result in evaluate response"))?;

        if let Ok(eval_result) = serde_json::from_value::<RuntimeEvaluateResult>(result.clone()) {
            if eval_result.was_thrown == Some(true) {
                return Ok(EvalResult {
                    value: None,
                    exception: eval_result.result.description.or(Some("Unknown error".into())),
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

### WebKit Console Level Mapping

```rust
fn parse_webkit_level(level: &str) -> ConsoleLevel {
    match level {
        "log" => ConsoleLevel::Log,
        "info" => ConsoleLevel::Info,
        "warning" => ConsoleLevel::Warn,  // WebKit uses "warning", not "warn"
        "error" => ConsoleLevel::Error,
        "debug" => ConsoleLevel::Debug,
        _ => ConsoleLevel::Log,
    }
}
```

### Key Difference from CDP

- WebKit uses `Console.messageAdded` (not deprecated). CDP uses `Runtime.consoleAPICalled` (`Console.messageAdded` is deprecated in CDP).
- WebKit's `Console.messageAdded` delivers `text` as a pre-formatted string. CDP's `consoleAPICalled` delivers `args` as an array of RemoteObjects that need serialization.
- WebKit uses `"warning"` level string. CDP uses `"warning"` too, but the event structure differs.

## Acceptance Criteria

- [ ] `WebKitAdapter` implements `ProtocolAdapter` trait
- [ ] Connects to WebKit Inspector via target discovery (`/json` endpoint)
- [ ] Sends `Runtime.enable` and `Console.enable` on connect
- [ ] `evaluate()` sends `Runtime.evaluate` and returns the result value
- [ ] `evaluate()` correctly reports exceptions (wasThrown = true)
- [ ] Console events are captured via `Console.messageAdded` and broadcast
- [ ] Console level mapping handles WebKit's `"warning"` (not `"warn"`)
- [ ] `close()` cleanly disconnects and stops the event listener
- [ ] Integration test against a real WebKitGTK instance (or documented manual test procedure)
- [ ] Error messages are actionable when connection fails (e.g., "No inspectable page target found — is the app running with WEBKIT_INSPECTOR_HTTP_SERVER?")
