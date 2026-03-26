# Plan 03: Console Event Protocol

**Objective:** Define the structured JSON wire format that the TypeScript runtime emits via `console.warn()` and the Rust CLI parses from debug protocol console events — the contract between the two halves of Mosaic.

**Depends on:** Plan 01

**Estimated scope:** S (< 1 day)

## Context

The runtime runs inside the target app's webview. The CLI watches from outside via the debug protocol's console capture. They need a structured format to communicate test events (step started, step passed, step failed, log, snapshot, etc.). This is the shared contract — both sides implement it independently, so it must be defined first.

The format uses `console.warn()` with a magic prefix to distinguish Mosaic events from the app's own console output. The CLI filters by this prefix and parses the JSON payload.

## Research Findings

- WebKit sends console events via `Console.messageAdded` with a `text` field (pre-formatted string).
- CDP sends console events via `Runtime.consoleAPICalled` with `args` array of `RemoteObject` values.
- Both protocols can capture `console.warn()` calls. Using `warn` (not `log`) reduces collision with app output.
- The magic prefix pattern is used by tools like Playwright's internal messaging.

## Files to Create

- `src/events/mod.rs` — Rust types for all console events (deserialization)
- `src/events/types.rs` — Event enum and associated structs

## Files to Modify

- `src/main.rs` — add `mod events;`

## Implementation Details

### Wire Format

Every Mosaic event is a `console.warn()` call with this format:

```
__MOSAIC__:{"event":"step:pass","data":{...}}
```

- **Prefix:** `__MOSAIC__:` — constant string, used for filtering.
- **Payload:** JSON object with `event` (string) and `data` (object).

### Event Types

```rust
use serde::Deserialize;

const MOSAIC_PREFIX: &str = "__MOSAIC__:";

#[derive(Debug, Deserialize)]
#[serde(tag = "event", content = "data")]
pub enum MosaicEvent {
    #[serde(rename = "test:start")]
    TestStart { name: String },

    #[serde(rename = "phase:start")]
    PhaseStart { name: String },

    #[serde(rename = "phase:end")]
    PhaseEnd { name: String },

    #[serde(rename = "step:start")]
    StepStart { name: String },

    #[serde(rename = "step:pass")]
    StepPass {
        name: String,
        duration_ms: u64,
    },

    #[serde(rename = "step:fail")]
    StepFail {
        name: String,
        duration_ms: u64,
        error: StepError,
        snapshot: Option<serde_json::Value>,
    },

    #[serde(rename = "step:skip")]
    StepSkip {
        name: String,
        reason: Option<String>,
    },

    #[serde(rename = "log")]
    Log { message: String },

    #[serde(rename = "snapshot")]
    Snapshot {
        label: String,
        data: serde_json::Value,
    },

    #[serde(rename = "test:end")]
    TestEnd {
        passed: u32,
        failed: u32,
        skipped: u32,
        duration_ms: u64,
    },

    #[serde(rename = "test:error")]
    TestError { message: String },
}

#[derive(Debug, Deserialize)]
pub struct StepError {
    #[serde(rename = "type")]
    pub error_type: String,
    pub message: String,
    pub expected: Option<String>,
    pub actual: Option<String>,
}
```

### Parsing Function

```rust
pub fn parse_console_message(text: &str) -> Option<MosaicEvent> {
    let json_str = text.strip_prefix(MOSAIC_PREFIX)?;
    serde_json::from_str(json_str).ok()
}
```

### TypeScript Side (reference — implemented in Plan 09)

```typescript
function emit(event: string, data: Record<string, unknown>): void {
  console.warn(`__MOSAIC__:${JSON.stringify({ event, data })}`);
}

// Usage in runtime:
emit('step:pass', { name: 'Send message 1', duration_ms: 1200 });
```

### Why console.warn, not console.log

- `console.log` is the most common call in app code — high collision risk.
- `console.warn` is rarely used by apps for routine output.
- Both are captured by WebKit `Console.messageAdded` and CDP `Runtime.consoleAPICalled`.
- The magic prefix provides a second layer of filtering regardless of which level is used.

## Acceptance Criteria

- [ ] `MosaicEvent` enum covers all event types: test:start, phase:start, phase:end, step:start, step:pass, step:fail, step:skip, log, snapshot, test:end, test:error
- [ ] `parse_console_message` correctly parses a valid `__MOSAIC__:` prefixed string into a `MosaicEvent`
- [ ] `parse_console_message` returns `None` for non-Mosaic console messages
- [ ] `parse_console_message` returns `None` for malformed JSON after the prefix
- [ ] `StepError` captures error type, message, expected, and actual
- [ ] All types derive `Debug` and `Deserialize`
- [ ] Unit tests cover: valid events, non-prefixed strings, malformed JSON, missing fields
- [ ] `cargo test` passes for all event parsing tests
