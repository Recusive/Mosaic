# Plan 12: Output Engine

**Objective:** Implement all three output formatters — structured text (default), JSON (`--json`), and NDJSON streaming (`--stream`) — plus exit code handling, consuming `MosaicEvent`s and producing the exact output formats shown in the spec.

**Depends on:** Plan 03

**Estimated scope:** M (1-3 days)

## Context

The output engine receives `MosaicEvent`s from the console event parser and formats them for stdout. This is what the agent reads. The structured text format is human-scannable and agent-parseable. JSON is for programmatic consumption. NDJSON streaming emits events in real-time as they happen. Exit codes (0-3) give the agent a fast pass/fail signal without parsing.

## Research Findings

- **colored 3.x** — terminal color output. `"string".green()`, `"string".red()`, etc.
- Output must be deterministic and machine-parseable — no ANSI codes in `--json` or `--stream` modes.
- The spec defines exact output formats with specific column alignment for step names and durations.
- Streaming output uses NDJSON (newline-delimited JSON) — one JSON object per line, no wrapping array.

## Files to Create

- `src/output/text.rs` — structured text formatter
- `src/output/json.rs` — JSON formatter
- `src/output/stream.rs` — NDJSON streaming formatter
- `src/output/formatter.rs` — `OutputFormatter` trait and factory

## Files to Modify

- `src/output/mod.rs` — re-export formatters

## Implementation Details

### OutputFormatter Trait

```rust
use crate::events::MosaicEvent;

pub trait OutputFormatter: Send {
    /// Handle a single event from the test.
    fn handle_event(&mut self, event: &MosaicEvent);

    /// Called when all events have been received. Produces final output if needed.
    fn finalize(&mut self) -> ExitCode;
}

#[derive(Debug, Clone, Copy)]
pub enum ExitCode {
    AllPassed = 0,
    TestFailed = 1,
    ConnectionError = 2,
    TestFileError = 3,
}

impl ExitCode {
    pub fn code(&self) -> i32 {
        *self as i32
    }
}

pub fn create_formatter(json: bool, stream: bool, verbose: bool) -> Box<dyn OutputFormatter> {
    if stream {
        Box::new(StreamFormatter::new())
    } else if json {
        Box::new(JsonFormatter::new(verbose))
    } else {
        Box::new(TextFormatter::new(verbose))
    }
}
```

### Structured Text Formatter (src/output/text.rs)

Produces the exact format from the spec:

```rust
use colored::*;

pub struct TextFormatter {
    verbose: bool,
    test_name: String,
    current_phase: Option<String>,
    passed: u32,
    failed: u32,
    skipped: u32,
    logs: Vec<String>,  // Logs to print under the current step
}

impl OutputFormatter for TextFormatter {
    fn handle_event(&mut self, event: &MosaicEvent) {
        match event {
            MosaicEvent::TestStart { name } => {
                self.test_name = name.clone();
                println!("{}", format!("[mosaic] {}", name).bold());
            }

            MosaicEvent::PhaseStart { name } => {
                self.current_phase = Some(name.clone());
                let phase_num = self.phase_count();
                println!();
                println!("Phase {} — {}:", phase_num, name);
            }

            MosaicEvent::PhaseEnd { .. } => {}

            MosaicEvent::StepPass { name, duration_ms } => {
                self.passed += 1;
                let duration = format_duration(*duration_ms);
                println!("  {} {:<48} {}", "✓".green(), name, duration.dimmed());
                self.flush_logs();
            }

            MosaicEvent::StepFail { name, duration_ms, error, snapshot } => {
                self.failed += 1;
                let duration = format_duration(*duration_ms);
                println!("  {} {:<48} {}", "✗".red(), name.red(), duration.dimmed());
                println!("    {}: {}", error.error_type.red(), error.message);
                if let Some(expected) = &error.expected {
                    println!("      Expected: {}", expected);
                }
                if let Some(actual) = &error.actual {
                    println!("      Actual:   {}", actual);
                }
                if let Some(snap) = snapshot {
                    println!("    Snapshot:");
                    println!("      {}", serde_json::to_string_pretty(snap).unwrap_or_default());
                }
                self.flush_logs();
            }

            MosaicEvent::StepSkip { name, reason } => {
                self.skipped += 1;
                let reason_str = reason.as_deref().unwrap_or("skipped");
                println!("  {} {:<48} — {}", "○".dimmed(), name.dimmed(), reason_str.dimmed());
            }

            MosaicEvent::StepStart { .. } => {
                // Clear logs for new step
                self.logs.clear();
            }

            MosaicEvent::Log { message } => {
                self.logs.push(message.clone());
            }

            MosaicEvent::Snapshot { label, data } => {
                if self.verbose {
                    self.logs.push(format!("[snapshot:{}] {}", label, serde_json::to_string(data).unwrap_or_default()));
                }
            }

            MosaicEvent::TestEnd { duration_ms, .. } => {
                let duration = format_duration(*duration_ms);
                println!();
                println!("{}", "[mosaic] ──────────────────────────────────────────".dimmed());
                print!("[mosaic] ");
                if self.passed > 0 { print!("{}", format!("{} passed", self.passed).green()); }
                if self.failed > 0 { print!(" · {}", format!("{} failed", self.failed).red()); }
                if self.skipped > 0 { print!(" · {}", format!("{} skipped", self.skipped).dimmed()); }
                println!(" · {}", duration);
                let exit = if self.failed > 0 { 1 } else { 0 };
                println!("[mosaic] Exit {}", exit);
            }

            MosaicEvent::TestError { message } => {
                println!("{}", format!("[mosaic] Error: {}", message).red());
            }
        }
    }

    fn finalize(&mut self) -> ExitCode {
        if self.failed > 0 {
            ExitCode::TestFailed
        } else {
            ExitCode::AllPassed
        }
    }
}

impl TextFormatter {
    fn flush_logs(&mut self) {
        for log in self.logs.drain(..) {
            println!("    {}", log.dimmed());
        }
    }

    fn phase_count(&self) -> u32 {
        // Track internally
        1 // Simplified — actual implementation tracks count
    }
}

fn format_duration(ms: u64) -> String {
    if ms < 1000 {
        format!("{}ms", ms)
    } else {
        format!("{:.1}s", ms as f64 / 1000.0)
    }
}
```

### JSON Formatter (src/output/json.rs)

Accumulates all events, produces a single JSON object at the end:

```rust
use serde::Serialize;

#[derive(Serialize)]
struct JsonOutput {
    test: String,
    target: String,
    protocol: String,
    #[serde(rename = "devServer")]
    dev_server: String,
    duration: u64,
    passed: u32,
    failed: u32,
    skipped: u32,
    exit: i32,
    phases: Vec<JsonPhase>,
}

#[derive(Serialize)]
struct JsonPhase {
    name: String,
    steps: Vec<JsonStep>,
}

#[derive(Serialize)]
struct JsonStep {
    name: String,
    status: String,
    duration: Option<u64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<JsonStepError>,
    #[serde(skip_serializing_if = "Option::is_none")]
    snapshot: Option<serde_json::Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    reason: Option<String>,
}

#[derive(Serialize)]
struct JsonStepError {
    #[serde(rename = "type")]
    error_type: String,
    message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    expected: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    actual: Option<String>,
}

pub struct JsonFormatter {
    output: JsonOutput,
    current_phase: Option<JsonPhase>,
    verbose: bool,
}

impl OutputFormatter for JsonFormatter {
    fn handle_event(&mut self, event: &MosaicEvent) {
        match event {
            MosaicEvent::TestStart { name } => {
                self.output.test = name.clone();
            }
            MosaicEvent::PhaseStart { name } => {
                self.flush_phase();
                self.current_phase = Some(JsonPhase { name: name.clone(), steps: vec![] });
            }
            MosaicEvent::StepPass { name, duration_ms } => {
                self.output.passed += 1;
                self.push_step(JsonStep {
                    name: name.clone(),
                    status: "pass".into(),
                    duration: Some(*duration_ms),
                    error: None, snapshot: None, reason: None,
                });
            }
            MosaicEvent::StepFail { name, duration_ms, error, snapshot } => {
                self.output.failed += 1;
                self.push_step(JsonStep {
                    name: name.clone(),
                    status: "fail".into(),
                    duration: Some(*duration_ms),
                    error: Some(JsonStepError {
                        error_type: error.error_type.clone(),
                        message: error.message.clone(),
                        expected: error.expected.clone(),
                        actual: error.actual.clone(),
                    }),
                    snapshot: snapshot.clone(),
                    reason: None,
                });
            }
            MosaicEvent::StepSkip { name, reason } => {
                self.output.skipped += 1;
                self.push_step(JsonStep {
                    name: name.clone(),
                    status: "skip".into(),
                    duration: None,
                    error: None, snapshot: None,
                    reason: reason.clone(),
                });
            }
            MosaicEvent::TestEnd { duration_ms, .. } => {
                self.flush_phase();
                self.output.duration = *duration_ms;
                self.output.exit = if self.output.failed > 0 { 1 } else { 0 };
                println!("{}", serde_json::to_string_pretty(&self.output).unwrap());
            }
            _ => {}
        }
    }

    fn finalize(&mut self) -> ExitCode {
        if self.output.failed > 0 { ExitCode::TestFailed } else { ExitCode::AllPassed }
    }
}
```

### NDJSON Streaming Formatter (src/output/stream.rs)

Emits one JSON object per line in real-time:

```rust
pub struct StreamFormatter {
    failed: u32,
}

impl OutputFormatter for StreamFormatter {
    fn handle_event(&mut self, event: &MosaicEvent) {
        let json = match event {
            MosaicEvent::TestStart { name } => {
                json!({"event": "start", "test": name})
            }
            MosaicEvent::PhaseStart { name } => {
                json!({"event": "phase", "name": name})
            }
            MosaicEvent::StepPass { name, duration_ms } => {
                json!({"event": "step", "name": name, "status": "pass", "duration": duration_ms})
            }
            MosaicEvent::StepFail { name, duration_ms, error, .. } => {
                self.failed += 1;
                json!({"event": "step", "name": name, "status": "fail", "duration": duration_ms, "error": error.message})
            }
            MosaicEvent::StepSkip { name, reason } => {
                json!({"event": "step", "name": name, "status": "skip", "reason": reason})
            }
            MosaicEvent::TestEnd { passed, failed, skipped, duration_ms } => {
                json!({"event": "end", "passed": passed, "failed": failed, "skipped": skipped, "duration": duration_ms, "exit": if *failed > 0 { 1 } else { 0 }})
            }
            MosaicEvent::Log { message } => {
                json!({"event": "log", "message": message})
            }
            _ => return,
        };
        println!("{}", serde_json::to_string(&json).unwrap());
    }

    fn finalize(&mut self) -> ExitCode {
        if self.failed > 0 { ExitCode::TestFailed } else { ExitCode::AllPassed }
    }
}
```

## Acceptance Criteria

- [ ] `TextFormatter` produces output matching the spec's structured text format exactly
- [ ] Passing steps show `✓` with name and duration, right-aligned
- [ ] Failed steps show `✗` with error type, message, expected/actual, and snapshot
- [ ] Skipped steps show `○` with reason
- [ ] Summary line shows passed/failed/skipped counts and total duration
- [ ] `JsonFormatter` produces a single JSON object matching the spec's JSON format
- [ ] JSON output includes phases with nested steps
- [ ] `StreamFormatter` emits one NDJSON line per event in real-time
- [ ] No ANSI color codes in JSON or streaming output
- [ ] `ExitCode` maps correctly: 0=pass, 1=fail, 2=connection error, 3=test file error
- [ ] `create_formatter()` selects the correct formatter based on `--json` and `--stream` flags
- [ ] Unit tests with fixture `MosaicEvent` sequences producing expected output
- [ ] Duration formatting: ms for < 1s, "X.Xs" for >= 1s
