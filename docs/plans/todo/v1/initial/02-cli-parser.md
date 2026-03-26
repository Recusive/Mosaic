# Plan 02: CLI Parser

**Objective:** Implement the full CLI interface — all commands, subcommands, and flags — using clap 4.6 derive API, producing structured arg types that downstream handlers will consume.

**Depends on:** Plan 01

**Estimated scope:** S (< 1 day)

## Context

The CLI is the entry point for everything. Agents run `mosaic test <name> --json`, `mosaic doctor`, etc. Every flag from the spec must be parseable before any command logic is implemented. This plan defines the types; command handlers are implemented in later plans.

## Research Findings

- **clap 4.6.0 derive API** is the current best practice. `#[derive(Parser)]` for the root, `#[derive(Subcommand)]` for the enum, `#[derive(Args)]` for shared/nested arg groups.
- **Optional positional arg:** `name: Option<String>` makes `mosaic test` (run all) and `mosaic test rewind-stress` (run one) both valid.
- **Multiple test names:** `names: Vec<String>` allows `mosaic test rewind sidebar`.
- No global flags needed — output flags (`--json`, `--stream`) and connection flags (`--port`, `--protocol`) are specific to `mosaic test`.

## Files to Create

- `src/cli/args.rs` — all clap type definitions

## Files to Modify

- `src/cli/mod.rs` — parse args, dispatch to placeholder handlers
- `src/main.rs` — call `cli::run()`

## Implementation Details

### src/cli/args.rs

```rust
use clap::{Parser, Subcommand, Args, ValueEnum};

#[derive(Parser)]
#[command(
    name = "mosaic",
    version,
    about = "Testing CLI built for AI agents",
    long_about = "One command. Real app. Real state. The agent knows if it works."
)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Run test(s) by name, or all tests if no name given
    Test(TestArgs),
    /// List all tests in .mosaic/
    List,
    /// Full environment diagnostic
    Doctor,
    /// Scaffold .mosaic/ directory with example test
    Init,
}

#[derive(Args)]
pub struct TestArgs {
    /// Test name(s). Omit to run all tests in .mosaic/
    pub names: Vec<String>,

    // -- Output flags --

    /// JSON output instead of structured text
    #[arg(long)]
    pub json: bool,

    /// Real-time NDJSON streaming
    #[arg(long)]
    pub stream: bool,

    /// Include state snapshots and debug info for all steps
    #[arg(long)]
    pub verbose: bool,

    // -- Execution flags --

    /// Stop on first failure
    #[arg(long)]
    pub bail: bool,

    /// Override default timeout in seconds (per step)
    #[arg(long, default_value_t = 30)]
    pub timeout: u64,

    // -- Connection override flags --

    /// Dev server port
    #[arg(long)]
    pub port: Option<u16>,

    /// Debug protocol port
    #[arg(long)]
    pub debug_port: Option<u16>,

    /// Force protocol
    #[arg(long, value_enum)]
    pub protocol: Option<Protocol>,

    /// Explicit dev server URL
    #[arg(long)]
    pub dev_server: Option<String>,

    /// Force target platform
    #[arg(long, value_enum)]
    pub target: Option<Target>,

    // -- Lifecycle flags --

    /// Don't start dev server automatically
    #[arg(long)]
    pub no_auto_start: bool,

    /// Tear down dev server after test (if mosaic started it)
    #[arg(long)]
    pub cleanup: bool,
}

#[derive(Clone, ValueEnum)]
pub enum Protocol {
    Webkit,
    Cdp,
    Node,
}

#[derive(Clone, ValueEnum)]
pub enum Target {
    Tauri,
    Electron,
    Browser,
    Node,
    Cli,
}
```

### src/cli/mod.rs

```rust
mod args;

pub use args::*;

use clap::Parser;

pub async fn run() -> anyhow::Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Commands::Test(args) => {
            // Placeholder — implemented in Plan 16
            println!("[mosaic] test command (not yet implemented)");
            Ok(())
        }
        Commands::List => {
            println!("[mosaic] list command (not yet implemented)");
            Ok(())
        }
        Commands::Doctor => {
            println!("[mosaic] doctor command (not yet implemented)");
            Ok(())
        }
        Commands::Init => {
            println!("[mosaic] init command (not yet implemented)");
            Ok(())
        }
    }
}
```

### Verification

```bash
# All should parse without errors:
cargo run -- test rewind-stress --json
cargo run -- test rewind-stress sidebar-width --stream --bail --timeout 60
cargo run -- test --port 1420 --debug-port 9222 --protocol webkit
cargo run -- test --dev-server http://localhost:3000 --target tauri --no-auto-start
cargo run -- list
cargo run -- doctor
cargo run -- init
cargo run -- --help
cargo run -- test --help
```

## Acceptance Criteria

- [ ] `cargo run -- --help` shows the top-level help with all four commands
- [ ] `cargo run -- test --help` shows all test flags documented in the spec
- [ ] `mosaic test` (no name) parses successfully with empty `names` vec
- [ ] `mosaic test rewind-stress` parses with `names = ["rewind-stress"]`
- [ ] `mosaic test a b c` parses with `names = ["a", "b", "c"]`
- [ ] All flag combinations from the spec parse correctly
- [ ] `--protocol` accepts `webkit`, `cdp`, `node` (case-insensitive via ValueEnum)
- [ ] `--target` accepts `tauri`, `electron`, `browser`, `node`, `cli`
- [ ] Invalid flags produce helpful clap error messages
- [ ] `cargo clippy` passes with no warnings on the CLI module
