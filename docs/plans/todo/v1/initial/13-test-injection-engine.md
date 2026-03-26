# Plan 13: Test Injection & Execution Engine

**Objective:** Implement the core mechanism that injects the runtime into a webview, dynamically imports a test file through the dev server, executes the test, captures console events, and feeds them to the output engine.

**Depends on:** Plan 06, Plan 03, Plan 11

**Estimated scope:** M (1-3 days)

## Context

This is the heart of Mosaic — the three-step injection sequence:
1. Evaluate `@mosaic/runtime.js` in the webview (sets up `window.mosaic`)
2. Evaluate `await import('http://localhost:<port>/.mosaic/<name>.test.ts')` (Vite compiles and serves the test, resolving imports against the app's module graph)
3. Evaluate `await window.mosaic._run(<timeout>)` (executes the registered test)

While the test runs, the Rust CLI captures `console.warn` events from the protocol adapter, filters for `__MOSAIC__:` prefixed events, parses them, and routes them to the output formatter.

## Research Findings

- `Runtime.evaluate` with `awaitPromise: true` (CDP) resolves Promises before returning — needed for `await import(...)` and `await mosaic._run()`.
- WebKit's `Runtime.evaluate` does NOT have `awaitPromise`. The runtime must wrap async operations and signal completion via console events.
- Dynamic `import()` through the Vite dev server works as a standard HTTP fetch from the browser — no special configuration needed for default Vite setups.
- If Vite blocks serving `.mosaic/` files (custom `server.fs.allow`), the import will fail with a 403. Mosaic should detect this and report the fix.

## Files to Create

- `src/engine/mod.rs` — test injection engine module
- `src/engine/injector.rs` — injection sequence
- `src/engine/event_loop.rs` — console event capture and routing

## Files to Modify

- `src/main.rs` — add `mod engine;`

## Implementation Details

### Injection Sequence

```rust
use crate::protocol::adapter::{ProtocolAdapter, EvalResult, ConsoleEvent};
use crate::events::{parse_console_message, MosaicEvent};
use crate::output::formatter::OutputFormatter;
use crate::runtime::RUNTIME_JS;

pub struct TestInjector<'a> {
    adapter: &'a dyn ProtocolAdapter,
    dev_server_url: String,
    timeout: u64,
}

impl<'a> TestInjector<'a> {
    pub fn new(adapter: &'a dyn ProtocolAdapter, dev_server_url: &str, timeout: u64) -> Self {
        Self {
            adapter,
            dev_server_url: dev_server_url.to_string(),
            timeout,
        }
    }

    /// Run the full injection and execution sequence for a single test.
    pub async fn run_test(
        &self,
        test_name: &str,
        formatter: &mut dyn OutputFormatter,
    ) -> anyhow::Result<()> {
        // Step 1: Inject the runtime
        self.inject_runtime().await?;

        // Step 2: Import the test file through the dev server
        self.import_test(test_name).await?;

        // Step 3: Start capturing console events
        let mut events_rx = self.adapter.console_events();

        // Step 4: Execute the test
        self.execute_test().await?;

        // Step 5: Process events until test:end is received
        self.process_events(&mut events_rx, formatter).await?;

        Ok(())
    }

    async fn inject_runtime(&self) -> anyhow::Result<()> {
        let result = self.adapter.evaluate(RUNTIME_JS).await?;
        if let Some(exception) = result.exception {
            return Err(anyhow::anyhow!(
                "Failed to inject Mosaic runtime: {}", exception
            ));
        }
        Ok(())
    }

    async fn import_test(&self, test_name: &str) -> anyhow::Result<()> {
        let import_url = format!(
            "{}/.mosaic/{}.test.ts",
            self.dev_server_url.trim_end_matches('/'),
            test_name,
        );

        // Wrap in an async IIFE to handle the await
        let import_js = format!(
            r#"(async () => {{
                try {{
                    await import('{}');
                }} catch (e) {{
                    console.warn('__MOSAIC__:' + JSON.stringify({{
                        event: 'test:error',
                        data: {{ message: 'Failed to import test: ' + e.message }}
                    }}));
                }}
            }})()"#,
            import_url,
        );

        let result = self.adapter.evaluate(&import_js).await?;
        if let Some(exception) = result.exception {
            return Err(anyhow::anyhow!(
                "Failed to import test file '{}': {}\n\
                 If Vite returned 403, add to vite.config.ts:\n  \
                 server: {{ fs: {{ allow: ['.mosaic'] }} }}",
                test_name, exception
            ));
        }
        Ok(())
    }

    async fn execute_test(&self) -> anyhow::Result<()> {
        let run_js = format!(
            r#"(async () => {{
                try {{
                    await window.mosaic._run({});
                }} catch (e) {{
                    console.warn('__MOSAIC__:' + JSON.stringify({{
                        event: 'test:error',
                        data: {{ message: 'Test execution failed: ' + e.message }}
                    }}));
                }}
            }})()"#,
            self.timeout * 1000, // Convert seconds to ms
        );

        self.adapter.evaluate(&run_js).await?;
        Ok(())
    }

    async fn process_events(
        &self,
        events_rx: &mut tokio::sync::broadcast::Receiver<ConsoleEvent>,
        formatter: &mut dyn OutputFormatter,
    ) -> anyhow::Result<()> {
        let deadline = tokio::time::Instant::now()
            + tokio::time::Duration::from_secs(self.timeout + 30); // Extra buffer

        loop {
            let event = tokio::time::timeout_at(
                deadline,
                events_rx.recv(),
            ).await;

            match event {
                Ok(Ok(console_event)) => {
                    // Only process warn-level events with our prefix
                    if let Some(mosaic_event) = parse_console_message(&console_event.text) {
                        formatter.handle_event(&mosaic_event);

                        // test:end signals completion
                        if matches!(mosaic_event, MosaicEvent::TestEnd { .. }) {
                            return Ok(());
                        }

                        // test:error is also terminal
                        if matches!(mosaic_event, MosaicEvent::TestError { .. }) {
                            return Ok(());
                        }
                    }
                }
                Ok(Err(tokio::sync::broadcast::error::RecvError::Lagged(n))) => {
                    eprintln!("[mosaic] Warning: dropped {} console events", n);
                }
                Err(_) => {
                    return Err(anyhow::anyhow!(
                        "Test timed out after {}s. No test:end event received.",
                        self.timeout + 30
                    ));
                }
                _ => break,
            }
        }

        Ok(())
    }
}
```

### Test File Resolution

```rust
pub fn resolve_test_file(cwd: &Path, name: &str) -> anyhow::Result<PathBuf> {
    let test_file = cwd.join(format!(".mosaic/{}.test.ts", name));
    if !test_file.exists() {
        return Err(anyhow::anyhow!(
            "Test file not found: .mosaic/{}.test.ts\n\
             Run 'mosaic list' to see available tests, or \
             'mosaic init' to create an example.",
            name
        ));
    }
    Ok(test_file)
}

pub fn list_test_files(cwd: &Path) -> anyhow::Result<Vec<String>> {
    let mosaic_dir = cwd.join(".mosaic");
    if !mosaic_dir.exists() {
        return Ok(vec![]);
    }

    let mut tests = vec![];
    for entry in std::fs::read_dir(&mosaic_dir)? {
        let entry = entry?;
        let name = entry.file_name().to_string_lossy().to_string();
        if name.ends_with(".test.ts") {
            let test_name = name.trim_end_matches(".test.ts").to_string();
            tests.push(test_name);
        }
    }
    tests.sort();
    Ok(tests)
}
```

### Handling WebKit vs CDP Differences

WebKit's `Runtime.evaluate` doesn't have `awaitPromise`. The workaround is to wrap everything in an async IIFE `(async () => { ... })()`. The evaluate call returns immediately (the IIFE returns a Promise), but the test runs asynchronously inside the webview. The Rust CLI doesn't need to await the evaluate response — it awaits the `test:end` console event instead.

This design works for both WebKit and CDP:
- **CDP**: `evaluate` with `awaitPromise: true` would also work, but using the IIFE pattern makes both adapters identical.
- **WebKit**: The IIFE is mandatory since there's no `awaitPromise`.

## Acceptance Criteria

- [ ] Runtime injection evaluates `RUNTIME_JS` in the webview and `window.mosaic` becomes available
- [ ] Test import constructs the correct Vite URL: `http://localhost:<port>/.mosaic/<name>.test.ts`
- [ ] Import errors (404, 403) are caught and reported with actionable messages
- [ ] The Vite 403 error specifically suggests adding `server.fs.allow`
- [ ] Test execution starts after import completes
- [ ] Console events are captured, filtered by `__MOSAIC__:` prefix, and parsed
- [ ] Non-Mosaic console messages are ignored (not treated as events)
- [ ] Events are routed to the output formatter in real-time
- [ ] Processing terminates when `test:end` or `test:error` is received
- [ ] A timeout kills the event loop if no `test:end` is received
- [ ] `resolve_test_file()` returns clear error when test file doesn't exist
- [ ] `list_test_files()` finds all `.test.ts` files in `.mosaic/`
- [ ] Broadcast channel lag is handled gracefully (warning, not crash)
