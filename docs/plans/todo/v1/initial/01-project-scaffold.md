# Plan 01: Project Scaffold & Dependencies

**Objective:** Create the Cargo project with correct directory structure, all dependencies declared, and placeholder modules that compile.

**Depends on:** None

**Estimated scope:** S (< 1 day)

## Context

This is the foundation. Every other plan depends on a compiling Rust project with the right directory layout. The structure follows the spec (`spec-v1.md` § Project Structure) and uses a flat binary crate (not a workspace — there's only one binary).

## Research Findings

- **clap 4.6.0** — derive API, latest stable. `features = ["derive"]`.
- **tokio 1.50.0** — async runtime. `features = ["full"]` for process, net, io, macros.
- **tokio-tungstenite 0.29.0** — WebSocket client for debug protocols. No TLS features needed (localhost only).
- **serde 1 + serde_json 1** — JSON parsing for protocol messages and config files.
- **reqwest** — HTTP client for dev server health checks and `/json` endpoint. `features = ["json"]`, `default-features = false` (no TLS needed for localhost).
- **colored** — terminal output formatting.
- **anyhow** — error handling in binary crate.
- **include_str!()** — for embedding the pre-built runtime JS. Zero-dep, built-in.

## Files to Create

- `Cargo.toml` — binary crate with all dependencies
- `build.rs` — build script stub (will compile TS runtime in Plan 11)
- `src/main.rs` — entry point, delegates to CLI
- `src/cli/mod.rs` — CLI module (stub)
- `src/detect/mod.rs` — auto-detection module (stub)
- `src/protocol/mod.rs` — protocol adapter module (stub)
- `src/runtime/mod.rs` — embedded runtime module (stub)
- `src/output/mod.rs` — output formatting module (stub)
- `src/server/mod.rs` — dev server management module (stub)

## Implementation Details

### Cargo.toml

```toml
[package]
name = "mosaic"
version = "0.1.0"
edition = "2021"
rust-version = "1.85"
description = "Testing CLI built for AI agents"
license = "MIT"
repository = "https://github.com/Recusive/Mosaic"

[dependencies]
clap = { version = "4.6", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.29"
futures-util = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
colored = "3"
anyhow = "1"
url = "2"
toml = "0.8"
```

### Directory Structure

```
mosaic/
├── Cargo.toml
├── build.rs
├── src/
│   ├── main.rs
│   ├── cli/
│   │   └── mod.rs
│   ├── detect/
│   │   └── mod.rs
│   ├── protocol/
│   │   └── mod.rs
│   ├── runtime/
│   │   └── mod.rs
│   ├── output/
│   │   └── mod.rs
│   └── server/
│       └── mod.rs
└── runtime/              # TypeScript source (Plan 09)
    └── .gitkeep
```

### src/main.rs

```rust
mod cli;
mod detect;
mod output;
mod protocol;
mod runtime;
mod server;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    cli::run().await
}
```

Each module's `mod.rs` should contain a brief doc comment describing its purpose and export an empty public API that will be filled in by subsequent plans.

### build.rs

```rust
fn main() {
    // Runtime build step will be added in Plan 11.
    // For now, this is a placeholder.
    println!("cargo::rerun-if-changed=runtime/src/");
}
```

## Acceptance Criteria

- [ ] `cargo check` succeeds with zero errors
- [ ] `cargo build` produces a `mosaic` binary
- [ ] Running `./target/debug/mosaic` does not panic (can print a placeholder message or nothing)
- [ ] All six module directories exist with `mod.rs` files
- [ ] All dependencies in Cargo.toml resolve (no version conflicts)
- [ ] `runtime/` directory exists with `.gitkeep`
