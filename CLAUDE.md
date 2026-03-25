# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**This file is living documentation.** Update it whenever architecture changes, new commands are added, conventions shift, or you discover something that would have saved you time if you'd known it at the start. Every session should leave CLAUDE.md more useful than it found it.

## What Is Mosaic

Mosaic is a testing CLI built for AI agents. One command (`mosaic test <name>`) runs test scripts **inside** a running application — with direct access to state stores, internal functions, and the DOM — then returns structured, machine-readable results. The agent reads the output and decides: pass, or fix and retry.

The core loop: `User prompt → Agent builds → mosaic test → Pass? → Done. Fail? → Fix → Retest.`

**Primary users are AI agents** (Claude Code, Codex, Gemini CLI). Every design decision optimizes for the agent experience: zero config, auto-detection, structured output, actionable errors, no interactive prompts.

## Project Status

**Pre-development.** The full design spec exists in `docs/spec/`. No implementation code yet.

- `docs/spec/spec-full.md` — Complete architecture and API reference
- `docs/spec/spec-v1.md` — v1: Tauri desktop app testing
- `docs/spec/spec-v1.1.md` — v1.1: Backend / Node.js testing
- `docs/spec/spec-v1.2.md` — v1.2: CLI / TUI tool testing
- `docs/spec/spec-v1.3.md` — v1.3: Browser webapp testing
- `docs/identity/what-is-mosaic.md` — Product definition and content rules

Always read the relevant spec before implementing or making architectural decisions.

## Architecture

Two layers:

1. **`mosaic` — Rust binary.** CLI orchestrator. Handles environment auto-detection, protocol connections, script injection, output formatting. Ships as a standalone binary with zero runtime dependencies.

2. **`@mosaic/runtime` — TypeScript.** Test API (`t.step()`, `t.dom.click()`, `t.store.get()`, etc.) that gets injected into the target application. Ships pre-bundled inside the Rust binary.

Tests share the target app's module graph via dynamic `import()` through the app's dev server (Vite, webpack, etc.). This means test imports like `import { useStore } from '@/stores/...'` resolve to the **same singleton instances** the running app uses.

### Protocol Adapters

Each adapter implements a common Rust trait (`connect`, `evaluate`, `on_console`, `close`):

| Adapter | Target | Protocol |
|---------|--------|----------|
| WebKit Inspector | Tauri macOS/Linux, Safari | WebKit Inspector Protocol |
| CDP | Electron, Chrome, Tauri Windows | Chrome DevTools Protocol |
| Node Inspector | Node.js backends | V8 Inspector Protocol |
| Process | CLI/TUI tools | stdin/stdout/stderr pipes |

### Platform Priority

v1: Tauri → v1.1: Backend → v1.2: CLI/TUI → v1.3: Browser webapps

## Key Design Decisions

- **Rust, not TypeScript** for the CLI binary. Foundation from day one, no rewrite path. The user explicitly rejected "prototype in TS, rewrite in Rust later."
- **Zero config by default.** Auto-detect project type, dev server, debug protocol. Auto-start dev server if not running. Flags override when auto-detection fails.
- **Loud failures.** When Mosaic can't self-recover, it reports exactly what was checked, what failed, and the exact command to fix it. Never silent.
- **Rich test API.** 10 domains: Core, State, DOM, Network, Viewport, Filesystem, Events, IPC, HTTP, Process. Agents need full access.
- **Module graph sharing via dev server.** Tests import app modules through Vite/webpack — no custom bundler, no SDK, no `window` globals.

## Test File Conventions

Tests live in `.mosaic/` (hidden directory, like `.github/`):

```
.mosaic/
├── rewind-stress.test.ts
├── sidebar-width.test.ts
└── config.ts                ← optional, not required
```

`mosaic test rewind-stress` → `.mosaic/rewind-stress.test.ts`

## CLI Commands

```
mosaic test <name>              Run a test
mosaic test <name> --json       JSON output
mosaic test <name> --stream     Real-time NDJSON streaming
mosaic test                     Run all tests
mosaic list                     List available tests
mosaic doctor                   Environment diagnostic
mosaic init                     Scaffold .mosaic/ with example test
```

Exit codes: `0` = all passed, `1` = test failure, `2` = connection error, `3` = test file error.

## Planned Rust Project Structure

```
mosaic/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── cli/                    # Command handlers
│   ├── detect/                 # Auto-detection (tauri.rs, vite.rs, etc.)
│   ├── protocol/               # Adapter trait + implementations
│   ├── runtime/                # Embedded @mosaic/runtime.js
│   ├── output/                 # Text, JSON, streaming formatters
│   └── server/                 # Dev server lifecycle management
├── runtime/                    # TypeScript source for @mosaic/runtime
│   ├── src/api/                # Test API domains (core, dom, state, etc.)
│   └── build.ts
└── tests/
```

## Content Rules

From `docs/identity/what-is-mosaic.md`:

- Mosaic is built by **Recursive Labs**
- Mosaic runs tests **inside** running applications, not from outside
- Mosaic is **not** a test generator — the agent writes the test, Mosaic runs it
- Mosaic is **not** a replacement for unit tests — different jobs, they coexist
- Do not claim web or backend testing works today — it's planned
- Do not cite performance numbers — none exist yet
- Do not claim users — the product is pre-development
