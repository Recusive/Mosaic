# Plan 17: Integration Testing Strategy

**Objective:** Define and implement the testing strategy for Mosaic itself — unit tests for individual components, integration tests with mock protocol servers, and a full end-to-end test against a real Tauri application.

**Depends on:** Plan 16

**Estimated scope:** L (3-5 days)

## Context

Mosaic is a testing tool. It needs to be well-tested itself. The challenge is that the full pipeline requires a running Tauri app, a debug protocol, and a dev server. Unit tests cover components in isolation. Integration tests use mock servers. E2E tests use a real fixture Tauri app.

## Research Findings

- **Mock WebSocket server**: `tokio-tungstenite` can create server-side WebSocket listeners for testing protocol adapters.
- **Fixture Tauri project**: A minimal Tauri project in `tests/fixtures/` that Mosaic can test against.
- **Port allocation**: Use port 0 (OS-assigned) for test servers to avoid conflicts.
- **Rust test patterns**: `#[tokio::test]` for async tests, `tempdir` crate for temp directories.

## Files to Create

```
tests/
├── unit/
│   ├── detect_tauri_test.rs        — Tauri config parsing with fixture files
│   ├── event_parsing_test.rs       — Console event protocol parsing
│   ├── output_text_test.rs         — Structured text output formatting
│   ├── output_json_test.rs         — JSON output formatting
│   └── output_stream_test.rs       — NDJSON streaming output
├── integration/
│   ├── mock_protocol_server.rs     — Shared mock WebSocket server
│   ├── webkit_adapter_test.rs      — WebKit adapter against mock server
│   ├── cdp_adapter_test.rs         — CDP adapter against mock server
│   ├── injection_test.rs           — Injection sequence against mock
│   └── full_pipeline_test.rs       — Full pipeline with mock everything
├── e2e/
│   └── tauri_app_test.rs           — Full E2E against a real Tauri app (CI-optional)
└── fixtures/
    ├── tauri-v2/
    │   └── src-tauri/
    │       └── tauri.conf.json     — Valid Tauri v2 config
    ├── tauri-v1/
    │   └── tauri.conf.json         — Valid Tauri v1 config
    ├── tauri-missing-url/
    │   └── src-tauri/
    │       └── tauri.conf.json     — Config with no devUrl
    ├── tauri-toml/
    │   └── src-tauri/
    │       └── Tauri.toml          — TOML format config
    └── mosaic-tests/
        └── .mosaic/
            ├── passing.test.ts     — Test that always passes
            ├── failing.test.ts     — Test that always fails
            └── mixed.test.ts       — Test with pass, fail, and skip
```

## Implementation Details

### Test Fixtures

**tests/fixtures/tauri-v2/src-tauri/tauri.conf.json:**
```json
{
  "productName": "fixture-app",
  "version": "0.1.0",
  "identifier": "com.test.fixture",
  "build": {
    "devUrl": "http://localhost:1420",
    "frontendDist": "../dist",
    "beforeDevCommand": "npm run dev"
  }
}
```

**tests/fixtures/tauri-v1/tauri.conf.json:**
```json
{
  "package": {
    "productName": "fixture-app",
    "version": "0.1.0"
  },
  "build": {
    "devPath": "http://localhost:1420",
    "distDir": "../dist",
    "beforeDevCommand": "npm run dev"
  }
}
```

### Unit Test: Event Parsing

```rust
#[cfg(test)]
mod tests {
    use crate::events::{parse_console_message, MosaicEvent};

    #[test]
    fn parses_step_pass() {
        let text = r#"__MOSAIC__:{"event":"step:pass","data":{"name":"Send message 1","duration_ms":1200}}"#;
        let event = parse_console_message(text).unwrap();
        match event {
            MosaicEvent::StepPass { name, duration_ms } => {
                assert_eq!(name, "Send message 1");
                assert_eq!(duration_ms, 1200);
            }
            _ => panic!("Expected StepPass"),
        }
    }

    #[test]
    fn ignores_non_mosaic_messages() {
        assert!(parse_console_message("Hello world").is_none());
        assert!(parse_console_message("[vite] hmr update").is_none());
    }

    #[test]
    fn handles_malformed_json() {
        assert!(parse_console_message("__MOSAIC__:{invalid}").is_none());
    }
}
```

### Unit Test: Tauri Detection

```rust
#[cfg(test)]
mod tests {
    use crate::detect::tauri::detect_tauri;
    use std::path::PathBuf;

    #[test]
    fn detects_tauri_v2() {
        let cwd = PathBuf::from("tests/fixtures/tauri-v2");
        let result = detect_tauri(&cwd).unwrap().unwrap();
        assert_eq!(result.dev_url, "http://localhost:1420");
        assert!(matches!(result.version, TauriVersion::V2));
    }

    #[test]
    fn detects_tauri_v1() {
        let cwd = PathBuf::from("tests/fixtures/tauri-v1");
        let result = detect_tauri(&cwd).unwrap().unwrap();
        assert_eq!(result.dev_url, "http://localhost:1420");
        assert!(matches!(result.version, TauriVersion::V1));
    }

    #[test]
    fn handles_missing_project() {
        let cwd = PathBuf::from("tests/fixtures/empty");
        let result = detect_tauri(&cwd).unwrap();
        assert!(result.is_none());
    }
}
```

### Integration Test: Mock Protocol Server

```rust
use tokio::net::TcpListener;
use tokio_tungstenite::accept_async;
use futures_util::{StreamExt, SinkExt};
use serde_json::json;

pub struct MockProtocolServer {
    port: u16,
    handle: JoinHandle<()>,
}

impl MockProtocolServer {
    pub async fn start() -> Self {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let port = listener.local_addr().unwrap().port();

        let handle = tokio::spawn(async move {
            while let Ok((stream, _)) = listener.accept().await {
                let ws = accept_async(stream).await.unwrap();
                let (mut write, mut read) = ws.split();

                while let Some(Ok(msg)) = read.next().await {
                    if let Message::Text(text) = msg {
                        let request: serde_json::Value = serde_json::from_str(&text).unwrap();
                        let id = request["id"].as_u64().unwrap();
                        let method = request["method"].as_str().unwrap();

                        match method {
                            "Runtime.enable" | "Console.enable" => {
                                write.send(Message::Text(
                                    json!({"id": id, "result": {}}).to_string()
                                )).await.unwrap();
                            }
                            "Runtime.evaluate" => {
                                let expression = request["params"]["expression"].as_str().unwrap();
                                // Simulate console events for the runtime
                                if expression.contains("_run") {
                                    // Emit mock test events
                                    emit_mock_events(&mut write).await;
                                }
                                write.send(Message::Text(
                                    json!({"id": id, "result": {"result": {"type": "undefined"}}}).to_string()
                                )).await.unwrap();
                            }
                            _ => {
                                write.send(Message::Text(
                                    json!({"id": id, "error": {"message": "unknown method"}}).to_string()
                                )).await.unwrap();
                            }
                        }
                    }
                }
            }
        });

        MockProtocolServer { port, handle }
    }

    pub fn ws_url(&self) -> String {
        format!("ws://127.0.0.1:{}", self.port)
    }
}

async fn emit_mock_events(write: &mut SplitSink<...>) {
    let events = vec![
        json!({"method": "Console.messageAdded", "params": {"message": {"level": "warning", "text": "__MOSAIC__:{\"event\":\"test:start\",\"data\":{\"name\":\"mock-test\"}}", "source": "javascript"}}}),
        json!({"method": "Console.messageAdded", "params": {"message": {"level": "warning", "text": "__MOSAIC__:{\"event\":\"phase:start\",\"data\":{\"name\":\"Phase 1\"}}", "source": "javascript"}}}),
        json!({"method": "Console.messageAdded", "params": {"message": {"level": "warning", "text": "__MOSAIC__:{\"event\":\"step:pass\",\"data\":{\"name\":\"Step 1\",\"duration_ms\":100}}", "source": "javascript"}}}),
        json!({"method": "Console.messageAdded", "params": {"message": {"level": "warning", "text": "__MOSAIC__:{\"event\":\"phase:end\",\"data\":{\"name\":\"Phase 1\"}}", "source": "javascript"}}}),
        json!({"method": "Console.messageAdded", "params": {"message": {"level": "warning", "text": "__MOSAIC__:{\"event\":\"test:end\",\"data\":{\"passed\":1,\"failed\":0,\"skipped\":0,\"duration_ms\":100}}", "source": "javascript"}}}),
    ];

    for event in events {
        write.send(Message::Text(event.to_string())).await.unwrap();
        tokio::time::sleep(Duration::from_millis(10)).await;
    }
}
```

### Test Tiers

| Tier | Scope | When to Run | CI Required |
|------|-------|-------------|-------------|
| **Unit** | Event parsing, config parsing, output formatting | Every `cargo test` | Yes |
| **Integration** | Protocol adapters with mock servers, injection with mock | Every `cargo test` | Yes |
| **E2E** | Full pipeline against a real Tauri fixture app | Manual or tagged CI | No (requires Tauri toolchain) |

### Running Tests

```bash
# All unit and integration tests
cargo test

# Only unit tests
cargo test --test unit

# Only integration tests
cargo test --test integration

# E2E tests (requires Tauri dev environment)
cargo test --test e2e -- --ignored
```

### TypeScript Runtime Tests

The TypeScript runtime also needs tests:

```bash
cd runtime
npm test  # Runs Vitest or similar against the runtime
```

Test the emitter, assertions, context, and runner in isolation, without a browser.

## Acceptance Criteria

- [ ] Fixture Tauri configs exist for v1, v2, TOML, and edge cases
- [ ] Unit tests cover: event parsing (all event types, edge cases), config parsing (v1, v2, TOML, missing fields), output formatting (text, JSON, NDJSON)
- [ ] Mock WebSocket server correctly simulates WebKit Inspector Protocol responses
- [ ] Mock WebSocket server correctly simulates CDP responses
- [ ] Integration tests verify: adapter connection, Runtime.evaluate, console event capture
- [ ] Integration test verifies the full injection sequence against a mock
- [ ] E2E test skeleton exists (marked `#[ignore]`) for testing against a real Tauri app
- [ ] `cargo test` passes with all unit and integration tests
- [ ] No test requires network access or external services (mocks only)
- [ ] TypeScript runtime has its own unit tests (assertions, emitter, deep equality)
- [ ] Test coverage includes error paths: connection failure, malformed events, timeout
- [ ] CI configuration documented (which tests run automatically vs manually)
