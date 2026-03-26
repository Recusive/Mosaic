# Plan 16: Test Runner Orchestrator

**Objective:** Wire everything together into the main `mosaic test` pipeline — CLI args → auto-detection → dev server → protocol connection → test injection → output → exit code — the complete end-to-end flow.

**Depends on:** Plan 02, Plan 04, Plan 05, Plan 07b (primary), Plan 12, Plan 13, Plan 14

**Estimated scope:** L (3-5 days)

## Context

This is where all the pieces come together. Every other plan delivers a component. This plan assembles them into the pipeline that runs when an agent types `mosaic test rewind-stress`. The orchestrator handles the full lifecycle: read CLI args, apply overrides to detection results, ensure the dev server is running, connect the right protocol adapter, inject and run the test, format output, and exit with the right code.

## Files to Create

- `src/cli/test_cmd.rs` — the complete `mosaic test` command handler

## Files to Modify

- `src/cli/mod.rs` — wire `Commands::Test` to the orchestrator

## Implementation Details

### Main Pipeline

```rust
use crate::cli::args::TestArgs;
use crate::detect::{detect, DetectedEnvironment, ProtocolKind};
use crate::server::DevServer;
use crate::protocol::{TauriPluginAdapter, WebKitAdapter, CdpAdapter, ProtocolAdapter};
use crate::engine::TestInjector;
use crate::output::formatter::{create_formatter, ExitCode};
use crate::diagnostics::report_connection_error;
use crate::engine::list_test_files;

pub async fn run_test(args: TestArgs) -> anyhow::Result<i32> {
    let cwd = std::env::current_dir()?;

    // Step 1: Determine which tests to run
    let test_names = if args.names.is_empty() {
        let names = list_test_files(&cwd)?;
        if names.is_empty() {
            eprintln!("[mosaic] No tests found in .mosaic/");
            eprintln!("  Run 'mosaic init' to create an example test.");
            return Ok(ExitCode::TestFileError.code());
        }
        names
    } else {
        // Verify all test files exist
        for name in &args.names {
            let path = cwd.join(format!(".mosaic/{}.test.ts", name));
            if !path.exists() {
                eprintln!("[mosaic] Test file not found: .mosaic/{}.test.ts", name);
                eprintln!("  Run 'mosaic list' to see available tests.");
                return Ok(ExitCode::TestFileError.code());
            }
        }
        args.names.clone()
    };

    // Step 2: Auto-detect environment (with CLI overrides)
    let mut env = match detect(&cwd).await {
        Ok(env) => env,
        Err(e) => {
            // If user provided manual flags, build a synthetic environment
            if args.dev_server.is_some() || args.protocol.is_some() {
                build_manual_environment(&args)?
            } else {
                eprintln!("[mosaic] ✗ Could not detect project type.");
                eprintln!("  {}", e);
                eprintln!();
                eprintln!("  Use --target and --dev-server flags to configure manually.");
                return Ok(ExitCode::ConnectionError.code());
            }
        }
    };

    // Apply CLI overrides
    apply_overrides(&mut env, &args);

    // Step 3: Ensure dev server is running
    let mut dev_server: Option<DevServer> = None;

    if !args.no_auto_start && !env.dev_server_running {
        match DevServer::start(build_dev_server_config(&env, &cwd)).await {
            Ok(server) => {
                dev_server = Some(server);
                // Print connection header
                print_header(&test_names, &env);
            }
            Err(e) => {
                report_connection_error(&env, &e, &cwd);
                return Ok(ExitCode::ConnectionError.code());
            }
        }
    } else if env.dev_server_running {
        print_header(&test_names, &env);
    } else {
        eprintln!("[mosaic] ✗ Dev server not running and --no-auto-start is set.");
        eprintln!("  Start your dev server manually, or remove --no-auto-start.");
        return Ok(ExitCode::ConnectionError.code());
    }

    // Step 4: Create and connect protocol adapter
    // For Tauri projects, prefer the Tauri Plugin adapter (works on all platforms)
    let mut adapter: Box<dyn ProtocolAdapter> = match &env.project_type {
        ProjectType::Tauri { .. } => Box::new(TauriPluginAdapter::new()),
        // Non-Tauri targets fall back to debug protocol adapters
        // _ => match env.protocol {
        //     ProtocolKind::WebKit => Box::new(WebKitAdapter::new()),
        //     ProtocolKind::Cdp => Box::new(CdpAdapter::new()),
        // }
    };

    let conn_config = ConnectionConfig {
        host: "127.0.0.1".to_string(),
        debug_port: env.debug_port,
    };

    if let Err(e) = adapter.connect(&conn_config).await {
        report_connection_error(&env, &e, &cwd);
        cleanup(dev_server, args.cleanup).await;
        return Ok(ExitCode::ConnectionError.code());
    }

    // Step 5: Run test(s)
    let mut formatter = create_formatter(args.json, args.stream, args.verbose);
    let injector = TestInjector::new(adapter.as_ref(), &env.dev_server_url, args.timeout);
    let mut overall_exit = ExitCode::AllPassed;

    for test_name in &test_names {
        match injector.run_test(test_name, formatter.as_mut()).await {
            Ok(()) => {
                let exit = formatter.finalize();
                if matches!(exit, ExitCode::TestFailed) {
                    overall_exit = ExitCode::TestFailed;
                    if args.bail {
                        break;
                    }
                }
            }
            Err(e) => {
                eprintln!("[mosaic] ✗ Error running test '{}': {}", test_name, e);
                overall_exit = ExitCode::TestFileError;
                if args.bail {
                    break;
                }
            }
        }
    }

    // Step 6: Cleanup
    adapter.close().await?;
    cleanup(dev_server, args.cleanup).await;

    Ok(overall_exit.code())
}
```

### Apply CLI Overrides

```rust
fn apply_overrides(env: &mut DetectedEnvironment, args: &TestArgs) {
    if let Some(port) = args.port {
        env.dev_server_port = port;
        env.dev_server_url = format!("http://localhost:{}", port);
    }
    if let Some(debug_port) = args.debug_port {
        env.debug_port = debug_port;
    }
    if let Some(ref protocol) = args.protocol {
        env.protocol = match protocol {
            Protocol::Webkit => ProtocolKind::WebKit,
            Protocol::Cdp => ProtocolKind::Cdp,
            Protocol::Node => ProtocolKind::Cdp, // Node uses CDP wire protocol
        };
    }
    if let Some(ref url) = args.dev_server {
        env.dev_server_url = url.clone();
        if let Some(port) = parse_port_from_url(url) {
            env.dev_server_port = port;
        }
    }
}
```

### Manual Environment (from flags only)

```rust
fn build_manual_environment(args: &TestArgs) -> anyhow::Result<DetectedEnvironment> {
    let dev_server_url = args.dev_server.clone()
        .unwrap_or_else(|| format!("http://localhost:{}", args.port.unwrap_or(1420)));
    let port = args.port.unwrap_or_else(|| parse_port_from_url(&dev_server_url).unwrap_or(1420));

    Ok(DetectedEnvironment {
        project_type: ProjectType::Tauri { version: TauriVersion::V2 },
        dev_server_url,
        dev_server_port: port,
        dev_server_running: is_port_open(port).await,
        debug_port: args.debug_port.unwrap_or(9222),
        protocol: match args.protocol.as_ref() {
            Some(Protocol::Webkit) => ProtocolKind::WebKit,
            Some(Protocol::Cdp) | Some(Protocol::Node) => ProtocolKind::Cdp,
            None => detect_protocol(),
        },
        start_command: None,
        tauri_config_path: None,
    })
}
```

### Connection Header

```rust
fn print_header(test_names: &[String], env: &DetectedEnvironment) {
    let protocol_str = match env.protocol {
        ProtocolKind::WebKit => "WebKit Inspector",
        ProtocolKind::Cdp => "CDP",
    };
    let target_str = match env.project_type {
        ProjectType::Tauri { .. } => "Tauri",
    };

    if !test_names.is_empty() {
        let name_str = if test_names.len() == 1 {
            test_names[0].clone()
        } else {
            format!("{} tests", test_names.len())
        };
        println!("[mosaic] {}", name_str);
    }
    println!("[mosaic] {} → {} → {}", target_str, protocol_str, env.dev_server_url);
    println!("[mosaic] ──────────────────────────────────────────");
}
```

### Cleanup

```rust
async fn cleanup(dev_server: Option<DevServer>, should_cleanup: bool) {
    if let Some(mut server) = dev_server {
        if should_cleanup {
            if let Err(e) = server.shutdown().await {
                eprintln!("[mosaic] Warning: failed to shut down dev server: {}", e);
            }
        }
        // If not cleaning up, the dev server keeps running (kill_on_drop will handle it on process exit)
    }
}
```

### Exit Code from main.rs

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let exit_code = cli::run().await?;
    std::process::exit(exit_code);
}
```

Update `cli::run()` to return `i32`:

```rust
pub async fn run() -> anyhow::Result<i32> {
    let cli = Cli::parse();
    match cli.command {
        Commands::Test(args) => test_cmd::run_test(args).await,
        Commands::List => { list_cmd::run_list(&cwd)?; Ok(0) },
        Commands::Doctor => { /* ... */ Ok(exit_code) },
        Commands::Init => { init_cmd::run_init(&cwd)?; Ok(0) },
    }
}
```

## Acceptance Criteria

- [ ] `mosaic test rewind-stress` runs the full pipeline end-to-end
- [ ] `mosaic test` (no name) discovers and runs all tests in `.mosaic/`
- [ ] `mosaic test a b c` runs multiple named tests sequentially
- [ ] `--json` produces JSON output instead of structured text
- [ ] `--stream` produces NDJSON streaming output
- [ ] `--bail` stops after the first failing test
- [ ] `--timeout` overrides the per-step timeout
- [ ] `--port`, `--debug-port`, `--protocol`, `--dev-server` override auto-detection
- [ ] `--no-auto-start` prevents dev server auto-start
- [ ] `--cleanup` tears down the dev server after tests complete
- [ ] Auto-detection results are used when no override flags are provided
- [ ] Connection errors produce the structured diagnostic output
- [ ] Test file errors (missing file) produce clear error and exit code 3
- [ ] Exit code 0 when all tests pass
- [ ] Exit code 1 when any test fails
- [ ] Exit code 2 on connection errors
- [ ] Exit code 3 on test file errors
- [ ] Dev server stays running after test unless `--cleanup` is passed
- [ ] Protocol adapter is cleanly closed after all tests
- [ ] The pipeline header shows: test name, target, protocol, dev server URL
