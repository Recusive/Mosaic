# Plan 06: Protocol Transport Layer & Adapter Trait

**Objective:** Build the shared WebSocket transport layer (JSON-RPC request/response matching, event dispatch) and define the `ProtocolAdapter` trait that all protocol adapters implement.

**Depends on:** Plan 01

**Estimated scope:** M (1-3 days)

## Context

Every debug protocol (WebKit Inspector, CDP, Node Inspector) uses the same underlying transport: JSON-RPC 2.0 messages over a WebSocket connection. The difference is which methods they call and how events are named. This plan builds the shared infrastructure once, so adapters (Plans 07, 08) only implement protocol-specific message types.

## Research Findings

- **tokio-tungstenite 0.29.0** — `connect_async()` returns a `WebSocketStream`. Call `.split()` for independent read/write halves.
- **JSON-RPC matching**: Use `HashMap<u64, oneshot::Sender<Value>>` for pending requests. Reader task matches responses by `id`, routes events (no `id`) to a broadcast channel.
- **Manual retry on connect**: 3 attempts with 200ms backoff. Debug connections are short-lived; reconnection during a test should fail loudly.
- **No existing Rust crates for WebKit Inspector**. Raw WebSocket is the only option.
- **cdp-protocol** crate exists for CDP types but no transport. Our transport layer fills this gap.

## Files to Create

- `src/protocol/transport.rs` — WebSocket transport with JSON-RPC dispatcher
- `src/protocol/adapter.rs` — `ProtocolAdapter` trait definition
- `src/protocol/target.rs` — target discovery (`/json` HTTP endpoint)

## Files to Modify

- `src/protocol/mod.rs` — re-export transport and trait

## Implementation Details

### ProtocolAdapter Trait

```rust
use async_trait::async_trait;

#[derive(Debug)]
pub struct EvalResult {
    pub value: Option<serde_json::Value>,
    pub exception: Option<String>,
}

#[derive(Debug, Clone)]
pub struct ConsoleEvent {
    pub level: ConsoleLevel,
    pub text: String,
    pub args: Vec<serde_json::Value>,
}

#[derive(Debug, Clone)]
pub enum ConsoleLevel {
    Log,
    Info,
    Warn,
    Error,
    Debug,
}

#[derive(Debug)]
pub struct ConnectionConfig {
    pub host: String,
    pub debug_port: u16,
}

#[async_trait]
pub trait ProtocolAdapter: Send + Sync {
    /// Connect to the debug target.
    async fn connect(&mut self, config: &ConnectionConfig) -> anyhow::Result<()>;

    /// Execute JavaScript in the target context.
    async fn evaluate(&self, javascript: &str) -> anyhow::Result<EvalResult>;

    /// Subscribe to console events. Returns a receiver for console messages.
    fn console_events(&self) -> tokio::sync::broadcast::Receiver<ConsoleEvent>;

    /// Clean disconnect.
    async fn close(&mut self) -> anyhow::Result<()>;
}
```

### WebSocket Transport

The shared transport handles WebSocket connection, message framing, and JSON-RPC request/response correlation.

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::{StreamExt, SinkExt};
use std::sync::atomic::{AtomicU64, Ordering};
use std::collections::HashMap;
use tokio::sync::{Mutex, oneshot, broadcast};

pub struct WebSocketTransport {
    writer: Arc<Mutex<SplitSink<WebSocketStream<...>, Message>>>,
    pending: Arc<Mutex<HashMap<u64, oneshot::Sender<serde_json::Value>>>>,
    event_tx: broadcast::Sender<serde_json::Value>,
    next_id: AtomicU64,
    reader_handle: Option<JoinHandle<()>>,
}

impl WebSocketTransport {
    /// Connect to a WebSocket URL with retry.
    pub async fn connect(url: &str) -> anyhow::Result<Self> {
        let ws_stream = connect_with_retry(url, 3).await?;
        let (writer, reader) = ws_stream.split();

        let pending = Arc::new(Mutex::new(HashMap::new()));
        let (event_tx, _) = broadcast::channel(256);

        // Spawn reader task
        let pending_clone = pending.clone();
        let event_tx_clone = event_tx.clone();
        let reader_handle = tokio::spawn(async move {
            Self::reader_loop(reader, pending_clone, event_tx_clone).await;
        });

        Ok(Self {
            writer: Arc::new(Mutex::new(writer)),
            pending,
            event_tx,
            next_id: AtomicU64::new(1),
            reader_handle: Some(reader_handle),
        })
    }

    /// Send a JSON-RPC request and await the response.
    pub async fn send_command(
        &self,
        method: &str,
        params: serde_json::Value,
    ) -> anyhow::Result<serde_json::Value> {
        let id = self.next_id.fetch_add(1, Ordering::SeqCst);
        let (tx, rx) = oneshot::channel();

        self.pending.lock().await.insert(id, tx);

        let msg = serde_json::json!({
            "id": id,
            "method": method,
            "params": params,
        });

        self.writer
            .lock()
            .await
            .send(Message::Text(msg.to_string()))
            .await?;

        let response = tokio::time::timeout(
            Duration::from_secs(30),
            rx,
        ).await??;

        Ok(response)
    }

    /// Subscribe to protocol events (messages without an id).
    pub fn events(&self) -> broadcast::Receiver<serde_json::Value> {
        self.event_tx.subscribe()
    }

    /// Reader loop: dispatches responses by id, broadcasts events.
    async fn reader_loop(
        mut reader: SplitStream<...>,
        pending: Arc<Mutex<HashMap<u64, oneshot::Sender<Value>>>>,
        event_tx: broadcast::Sender<Value>,
    ) {
        while let Some(Ok(Message::Text(text))) = reader.next().await {
            if let Ok(msg) = serde_json::from_str::<Value>(&text) {
                if let Some(id) = msg.get("id").and_then(|v| v.as_u64()) {
                    // Response to a request — route to waiting caller
                    if let Some(tx) = pending.lock().await.remove(&id) {
                        let _ = tx.send(msg);
                    }
                } else {
                    // Event — broadcast to all subscribers
                    let _ = event_tx.send(msg);
                }
            }
        }
    }

    /// Close the WebSocket connection.
    pub async fn close(self) -> anyhow::Result<()> {
        self.writer.lock().await.close().await?;
        if let Some(handle) = self.reader_handle {
            handle.abort();
        }
        Ok(())
    }
}
```

### Target Discovery

Most debug protocols expose an HTTP endpoint at `/json` (or `/json/list`) that lists inspectable targets:

```rust
pub struct InspectableTarget {
    pub id: String,
    pub title: String,
    pub url: String,
    pub target_type: String,
    pub ws_url: String,
}

pub async fn discover_targets(host: &str, port: u16) -> anyhow::Result<Vec<InspectableTarget>> {
    let url = format!("http://{}:{}/json", host, port);
    let resp = reqwest::get(&url).await?;
    let targets: Vec<serde_json::Value> = resp.json().await?;

    targets.iter().map(|t| {
        Ok(InspectableTarget {
            id: t["id"].as_str().unwrap_or_default().to_string(),
            title: t["title"].as_str().unwrap_or_default().to_string(),
            url: t["url"].as_str().unwrap_or_default().to_string(),
            target_type: t["type"].as_str().unwrap_or_default().to_string(),
            ws_url: t["webSocketDebuggerUrl"].as_str().unwrap_or_default().to_string(),
        })
    }).collect()
}

/// Find the main page target (not service workers, extensions, etc.)
pub fn find_page_target(targets: &[InspectableTarget]) -> Option<&InspectableTarget> {
    targets.iter().find(|t| t.target_type == "page" || t.target_type == "node")
}
```

### Connection with Retry

```rust
async fn connect_with_retry(url: &str, max_attempts: u32) -> anyhow::Result<WebSocketStream<...>> {
    for attempt in 1..=max_attempts {
        match connect_async(url).await {
            Ok((stream, _)) => return Ok(stream),
            Err(e) if attempt < max_attempts => {
                tokio::time::sleep(Duration::from_millis(200 * attempt as u64)).await;
            }
            Err(e) => return Err(anyhow!("WebSocket connection failed after {} attempts: {}", max_attempts, e)),
        }
    }
    unreachable!()
}
```

## Acceptance Criteria

- [ ] `ProtocolAdapter` trait compiles and is object-safe (can be used as `Box<dyn ProtocolAdapter>`)
- [ ] `WebSocketTransport::connect()` establishes a WebSocket connection to a given URL
- [ ] `send_command()` sends a JSON-RPC message and returns the matched response
- [ ] Protocol events (messages without `id`) are broadcast to all subscribers
- [ ] Connection retry works: 3 attempts with increasing delay
- [ ] `send_command()` times out after 30 seconds if no response
- [ ] `discover_targets()` parses the `/json` endpoint response
- [ ] `find_page_target()` correctly identifies page vs other target types
- [ ] `close()` cleanly shuts down the WebSocket and reader task
- [ ] Unit tests with a mock WebSocket server (tokio-tungstenite can create server-side too)
- [ ] The transport is reusable by both WebKit and CDP adapters without modification
