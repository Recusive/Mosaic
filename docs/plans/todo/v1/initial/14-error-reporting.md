# Plan 14: Error Reporting & mosaic doctor

**Objective:** Implement loud, actionable error reporting for all failure modes and the `mosaic doctor` diagnostic command — ensuring agents always get exact information about what failed and how to fix it.

**Depends on:** Plan 02, Plan 04

**Estimated scope:** M (1-3 days)

## Context

Mosaic's error philosophy: **nothing is ever silent**. When something fails, the agent gets a structured diagnostic that tells it exactly what was checked, what failed, and the exact command to fix it. `mosaic doctor` is the standalone diagnostic that verifies the entire environment without running a test. This is critical for the agent loop — the agent reads error output and self-recovers.

## Research Findings

- **colored 3.x** — `✓` in green, `✗` in red, with descriptive text.
- Error output must follow the exact format from the spec (§ Auto-Detection, Layer 3).
- `mosaic doctor` checks: project detection, dev server status, debug protocol status, test file inventory.
- Each check should be independently testable — a `DiagnosticCheck` trait pattern.

## Files to Create

- `src/diagnostics/mod.rs` — diagnostics module
- `src/diagnostics/checks.rs` — individual diagnostic checks
- `src/diagnostics/report.rs` — diagnostic report formatting

## Files to Modify

- `src/main.rs` — add `mod diagnostics;`
- `src/cli/mod.rs` — wire `Commands::Doctor` to diagnostic runner

## Implementation Details

### Diagnostic Check System

```rust
use colored::*;

#[derive(Debug)]
pub enum CheckResult {
    Pass(String),
    Fail(String, Option<String>),  // (message, fix suggestion)
    Warn(String),
}

pub struct DiagnosticReport {
    pub project: Vec<CheckResult>,
    pub dev_server: Vec<CheckResult>,
    pub debug_protocol: Vec<CheckResult>,
    pub tests: Vec<CheckResult>,
}

impl DiagnosticReport {
    pub fn print(&self) {
        println!("{}", "[mosaic] Environment Diagnostic".bold());
        println!();

        Self::print_section("Project", &self.project);
        Self::print_section("Dev Server", &self.dev_server);
        Self::print_section("Debug Protocol", &self.debug_protocol);
        Self::print_section("Tests", &self.tests);

        // Summary
        let has_fail = [&self.project, &self.dev_server, &self.debug_protocol, &self.tests]
            .iter()
            .any(|section| section.iter().any(|c| matches!(c, CheckResult::Fail(..))));

        println!();
        if has_fail {
            println!("  {}: Fix the issues above, then retry.", "Status".bold());
        } else {
            println!("  {}: {}.", "Status".bold(), "Ready to test".green());
        }
    }

    fn print_section(name: &str, checks: &[CheckResult]) {
        println!("  {}:", name.bold());
        for check in checks {
            match check {
                CheckResult::Pass(msg) => {
                    println!("    {} {}", "✓".green(), msg);
                }
                CheckResult::Fail(msg, fix) => {
                    println!("    {} {}", "✗".red(), msg.red());
                    if let Some(fix) = fix {
                        println!("      → {}", fix);
                    }
                }
                CheckResult::Warn(msg) => {
                    println!("    {} {}", "!".yellow(), msg.yellow());
                }
            }
        }
        println!();
    }
}
```

### Running Diagnostics

```rust
pub async fn run_doctor(cwd: &Path) -> DiagnosticReport {
    let mut report = DiagnosticReport {
        project: vec![],
        dev_server: vec![],
        debug_protocol: vec![],
        tests: vec![],
    };

    // -- Project checks --
    match detect_tauri(cwd) {
        Ok(Some(tauri)) => {
            report.project.push(CheckResult::Pass(
                format!("{} found", tauri.config_path.display())
            ));
            report.project.push(CheckResult::Pass(
                format!("Tauri {} project detected", match tauri.version {
                    TauriVersion::V1 => "v1",
                    TauriVersion::V2 => "v2",
                })
            ));
            report.project.push(CheckResult::Pass(
                format!("devUrl: {}", tauri.dev_url)
            ));

            let port = parse_port_from_url(&tauri.dev_url).unwrap_or(1420);

            // -- Dev server checks --
            if is_port_open(port).await {
                report.dev_server.push(CheckResult::Pass(
                    format!("localhost:{} responding", port)
                ));
            } else {
                report.dev_server.push(CheckResult::Fail(
                    format!("No response on localhost:{}", port),
                    Some(format!("Start your dev server or run: cargo tauri dev")),
                ));
            }

            // -- Debug protocol checks --
            let debug_port = 9222;
            match std::env::consts::OS {
                "linux" => {
                    if is_port_open(debug_port).await {
                        report.debug_protocol.push(CheckResult::Pass(
                            format!("WebKit Inspector on localhost:{}", debug_port)
                        ));
                    } else {
                        report.debug_protocol.push(CheckResult::Fail(
                            format!("No WebKit Inspector on localhost:{}", debug_port),
                            Some(format!(
                                "Launch with: WEBKIT_INSPECTOR_HTTP_SERVER=127.0.0.1:{} cargo tauri dev",
                                debug_port
                            )),
                        ));
                    }
                }
                "macos" | "linux" | "windows" => {
                    // Tauri Plugin adapter works on all platforms
                    if has_mosaic_plugin(&cwd) {
                        report.debug_protocol.push(CheckResult::Pass(
                            "tauri-plugin-mosaic found in Cargo.toml".to_string()
                        ));
                    } else {
                        report.debug_protocol.push(CheckResult::Fail(
                            "tauri-plugin-mosaic not found".to_string(),
                            Some("Run 'mosaic init --plugin' to add Tauri testing support".to_string()),
                        ));
                    }
                }
                "windows" => {
                    report.debug_protocol.push(CheckResult::Pass(
                        "WebView2: CDP available in dev mode".to_string()
                    ));
                }
                os => {
                    report.debug_protocol.push(CheckResult::Warn(
                        format!("Unsupported OS: {}", os)
                    ));
                }
            }
        }
        Ok(None) => {
            report.project.push(CheckResult::Fail(
                "No tauri.conf.json found".to_string(),
                Some("Run from a Tauri project root, or use --target to specify the platform".to_string()),
            ));
        }
        Err(e) => {
            report.project.push(CheckResult::Fail(
                format!("Error reading config: {}", e),
                None,
            ));
        }
    }

    // -- Test file checks --
    let mosaic_dir = cwd.join(".mosaic");
    if mosaic_dir.exists() {
        match list_test_files(cwd) {
            Ok(tests) if !tests.is_empty() => {
                report.tests.push(CheckResult::Pass(
                    format!(".mosaic/ directory found ({} tests)", tests.len())
                ));
                for test in &tests {
                    let path = mosaic_dir.join(format!("{}.test.ts", test));
                    let modified = std::fs::metadata(&path)
                        .and_then(|m| m.modified())
                        .ok()
                        .map(|t| format_relative_time(t))
                        .unwrap_or("unknown".to_string());
                    report.tests.push(CheckResult::Pass(
                        format!("  {:<24} {}", test, modified)
                    ));
                }
            }
            Ok(_) => {
                report.tests.push(CheckResult::Warn(
                    ".mosaic/ exists but no .test.ts files found".to_string()
                ));
            }
            Err(e) => {
                report.tests.push(CheckResult::Fail(
                    format!("Error reading .mosaic/: {}", e),
                    None,
                ));
            }
        }
    } else {
        report.tests.push(CheckResult::Fail(
            "No .mosaic/ directory found".to_string(),
            Some("Run 'mosaic init' to create an example test".to_string()),
        ));
    }

    report
}
```

### Connection Error Reporting

When `mosaic test` fails to connect, the error follows the spec's exact format:

```rust
pub fn report_connection_error(
    env: &DetectedEnvironment,
    error: &anyhow::Error,
    cwd: &Path,
) {
    eprintln!("{}", "[mosaic] ✗ Could not connect to your app.".red().bold());
    eprintln!();
    eprintln!("  {}:", "What I checked".bold());
    // ... list all checks with ✓/✗ ...

    eprintln!();
    eprintln!("  {}:", "How to fix".bold());
    // ... actionable steps ...

    eprintln!();
    eprintln!("  Or connect manually:");
    eprintln!("    mosaic test <name> --dev-server {} \\", env.dev_server_url);
    eprintln!("                            --protocol {} \\",
        match env.protocol { ProtocolKind::WebKit => "webkit", ProtocolKind::Cdp => "cdp" });
    eprintln!("                            --debug-port {}", env.debug_port);
    eprintln!();
    eprintln!("  Run '{}' for a full diagnostic.", "mosaic doctor".bold());
}
```

### Test File Error Reporting

```rust
pub fn report_test_file_error(test_name: &str, error: &str) {
    eprintln!("{}", format!("[mosaic] ✗ Could not load test '{}'.", test_name).red().bold());
    eprintln!();
    eprintln!("  Error: {}", error);
    eprintln!();

    if error.contains("403") || error.contains("Forbidden") {
        eprintln!("  Your Vite config may restrict file serving.");
        eprintln!("  Add to vite.config.ts:");
        eprintln!("    server: {{ fs: {{ allow: ['.mosaic'] }} }}");
    } else if error.contains("404") || error.contains("Not Found") {
        eprintln!("  The test file was not found by the dev server.");
        eprintln!("  Check that .mosaic/{}.test.ts exists.", test_name);
    } else if error.contains("SyntaxError") || error.contains("parse") {
        eprintln!("  The test file has a syntax error. Fix the TypeScript and retry.");
    }
}
```

## Acceptance Criteria

- [ ] `mosaic doctor` prints a full diagnostic matching the spec's format
- [ ] Doctor checks: project detection, devUrl, dev server status, debug protocol, test files
- [ ] Each check shows `✓` (pass), `✗` (fail with fix), or `!` (warning)
- [ ] Failed checks include an actionable fix command
- [ ] Connection errors show what was checked, what failed, and manual override flags
- [ ] Test file errors (403, 404, syntax) are detected and reported with specific fixes
- [ ] macOS shows a warning about limited WKWebView debugging support
- [ ] `mosaic doctor` exit code: 0 if all checks pass, 1 if any fail
- [ ] Error output is structured text (not JSON) regardless of `--json` flag
- [ ] All error strings are actionable — an agent can parse them and take corrective action
- [ ] Unit tests for each check result type and formatting
