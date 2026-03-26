# Plan 05: Dev Server Lifecycle Management

**Objective:** Implement the ability to detect a running dev server, auto-start one with correct environment variables and flags, poll for readiness, and manage the child process lifecycle (keep alive, graceful shutdown).

**Depends on:** Plan 01, Plan 04

**Estimated scope:** M (1-3 days)

## Context

When an agent runs `mosaic test`, the dev server must be running. If it's not, Mosaic starts it automatically with the right env vars (e.g., `WEBKIT_INSPECTOR_HTTP_SERVER` on Linux). After the test, the server stays running (the agent will test again soon) unless `--cleanup` is passed. This is the "environment assurance" layer from the spec.

## Research Findings

- **tokio::process::Command** — mirrors std but async. `kill_on_drop(true)` sends `SIGKILL` on drop.
- **Port readiness polling**: `TcpStream::connect()` with 200ms interval, wrapped in timeout. More reliable than HTTP GET for initial detection.
- **Stdout monitoring**: Read lines from child stdout for readiness signals (e.g., Vite prints "Local: http://localhost:1420/"). Use as fast path before falling back to port polling.
- **Graceful shutdown**: Send `SIGTERM` first, wait 5s, then `SIGKILL`. Use `nix` crate for signal sending on Unix.
- **Zombie prevention**: Always `child.wait().await` after child exits. `kill_on_drop` handles best-effort cleanup.
- **Dev server env vars**: `BROWSER=none` prevents Vite from opening a browser. `WEBKIT_INSPECTOR_HTTP_SERVER=127.0.0.1:9222` on Linux enables the HTTP inspector server.

## Files to Create

- `src/server/dev_server.rs` — dev server start/stop/detect logic
- `src/server/health.rs` — port polling and HTTP health check

## Files to Modify

- `src/server/mod.rs` — re-export server API

## Implementation Details

### DevServer Struct

```rust
use tokio::process::{Child, Command};
use std::process::Stdio;

pub struct DevServer {
    child: Option<Child>,
    port: u16,
    started_by_mosaic: bool,
}

impl DevServer {
    /// Check if a dev server is already running on the expected port.
    pub async fn detect(port: u16) -> bool {
        is_port_open(port).await
    }

    /// Start a dev server with the given command and environment.
    pub async fn start(config: DevServerConfig) -> Result<Self> {
        let mut cmd = Command::new(&config.command_parts[0]);
        cmd.args(&config.command_parts[1..])
            .current_dir(&config.cwd)
            .stdout(Stdio::piped())
            .stderr(Stdio::piped())
            .kill_on_drop(true);

        // Set environment variables
        for (key, value) in &config.env {
            cmd.env(key, value);
        }
        cmd.env("BROWSER", "none"); // Prevent browser auto-open

        let mut child = cmd.spawn()?;

        // Wait for readiness
        wait_for_ready(&mut child, config.port, config.timeout).await?;

        Ok(DevServer {
            child: Some(child),
            port: config.port,
            started_by_mosaic: true,
        })
    }

    /// Graceful shutdown. Only shuts down if Mosaic started the server.
    pub async fn shutdown(&mut self) -> Result<()> {
        if !self.started_by_mosaic {
            return Ok(());
        }
        if let Some(ref mut child) = self.child {
            graceful_kill(child).await?;
        }
        Ok(())
    }
}
```

### DevServerConfig

```rust
pub struct DevServerConfig {
    pub command_parts: Vec<String>,  // e.g., ["cargo", "tauri", "dev"]
    pub cwd: PathBuf,
    pub port: u16,
    pub timeout: Duration,
    pub env: Vec<(String, String)>,  // Additional env vars
}
```

### Readiness Detection

Two-track approach: monitor stdout for readiness signal AND poll the port. Whichever fires first wins.

```rust
async fn wait_for_ready(child: &mut Child, port: u16, timeout: Duration) -> Result<()> {
    let stdout = child.stdout.take()
        .ok_or_else(|| anyhow!("stdout not captured"))?;

    let (ready_tx, ready_rx) = tokio::sync::oneshot::channel::<()>();
    let ready_tx = std::sync::Arc::new(tokio::sync::Mutex::new(Some(ready_tx)));

    // Track 1: Monitor stdout for readiness signals
    let ready_tx_clone = ready_tx.clone();
    let stdout_handle = tokio::spawn(async move {
        let reader = tokio::io::BufReader::new(stdout);
        let mut lines = reader.lines();
        while let Ok(Some(line)) = lines.next_line().await {
            // Vite: "Local: http://localhost:..."
            // Tauri: "Watching for file changes..."
            // Generic: check for common patterns
            if line.contains("Local:") || line.contains("ready in") ||
               line.contains("Watching") || line.contains("listening on") {
                if let Some(tx) = ready_tx_clone.lock().await.take() {
                    let _ = tx.send(());
                }
                break;
            }
        }
    });

    // Track 2: Poll the port
    let port_ready = poll_port(port, Duration::from_millis(300), timeout);

    // Race: whichever fires first
    tokio::select! {
        _ = ready_rx => Ok(()),
        result = port_ready => result,
        // Also check if the child process died
        status = child.wait() => {
            Err(anyhow!(
                "Dev server exited with status {} before becoming ready",
                status?
            ))
        }
    }
}
```

### Port Polling

```rust
async fn poll_port(port: u16, interval: Duration, max_wait: Duration) -> Result<()> {
    let start = std::time::Instant::now();
    loop {
        if is_port_open(port).await {
            return Ok(());
        }
        if start.elapsed() > max_wait {
            return Err(anyhow!(
                "Dev server did not start within {}s on port {}",
                max_wait.as_secs(), port
            ));
        }
        tokio::time::sleep(interval).await;
    }
}
```

### Graceful Shutdown

```rust
async fn graceful_kill(child: &mut Child) -> Result<()> {
    #[cfg(unix)]
    {
        use nix::sys::signal::{kill, Signal};
        use nix::unistd::Pid;

        if let Some(pid) = child.id() {
            let _ = kill(Pid::from_raw(pid as i32), Signal::SIGTERM);
            match tokio::time::timeout(Duration::from_secs(5), child.wait()).await {
                Ok(Ok(_)) => return Ok(()),
                _ => {}
            }
        }
    }

    // Fallback: force kill
    child.kill().await?;
    child.wait().await?;
    Ok(())
}
```

### Building the Start Command from Detection

```rust
pub fn build_dev_server_config(env: &DetectedEnvironment, cwd: &Path) -> DevServerConfig {
    let mut env_vars = vec![];

    // Add inspector env var for Linux WebKit
    if std::env::consts::OS == "linux" && matches!(env.protocol, ProtocolKind::WebKit) {
        env_vars.push((
            "WEBKIT_INSPECTOR_HTTP_SERVER".to_string(),
            format!("127.0.0.1:{}", env.debug_port),
        ));
    }

    DevServerConfig {
        command_parts: vec!["cargo".into(), "tauri".into(), "dev".into()],
        cwd: cwd.to_path_buf(),
        port: env.dev_server_port,
        timeout: Duration::from_secs(120), // Tauri dev can be slow (Rust compile)
        env: env_vars,
    }
}
```

## Acceptance Criteria

- [ ] `DevServer::detect()` correctly identifies a running server on a given port
- [ ] `DevServer::start()` spawns a child process with correct env vars
- [ ] Readiness detection works via both stdout monitoring and port polling
- [ ] Start times out after the configured duration with a clear error
- [ ] If the child process exits before becoming ready, the error includes the exit status
- [ ] `WEBKIT_INSPECTOR_HTTP_SERVER` is set on Linux for WebKit protocol
- [ ] `BROWSER=none` is always set to prevent browser auto-open
- [ ] `shutdown()` sends SIGTERM first, waits 5s, then SIGKILL
- [ ] `shutdown()` is a no-op if Mosaic didn't start the server
- [ ] Unit tests for port polling logic (can test with a temporary TCP listener)
- [ ] `--no-auto-start` flag prevents server startup (wiring in Plan 16)
- [ ] `--cleanup` flag triggers shutdown after test (wiring in Plan 16)
