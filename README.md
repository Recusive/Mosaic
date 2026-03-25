<p align="center">
  <img src="assets/logo.png" width="120" alt="Mosaic" />
</p>

<h1 align="center">Mosaic</h1>

<p align="center">
  <strong>The testing CLI built for AI agents.</strong><br/>
  One command. Real app. Real state. The agent knows if it works.
</p>

<p align="center">
  <a href="https://github.com/Recusive/Mosaic/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License" /></a>
  <a href="https://github.com/Recusive/Mosaic/stargazers"><img src="https://img.shields.io/github/stars/Recusive/Mosaic?style=social" alt="GitHub Stars" /></a>
  <a href="https://mosaic.sh"><img src="https://img.shields.io/badge/website-mosaic.sh-black" alt="Website" /></a>
  <a href="https://github.com/Recusive/Mosaic"><img src="https://img.shields.io/badge/status-pre--development-orange" alt="Status: Pre-development" /></a>
  <a href="https://www.rust-lang.org"><img src="https://img.shields.io/badge/built%20with-Rust-dea584" alt="Built with Rust" /></a>
</p>

<p align="center">
  <a href="https://mosaic.sh">Website</a> &middot;
  <a href="#quick-start">Quick Start</a> &middot;
  <a href="#write-a-test">Write a Test</a> &middot;
  <a href="#how-it-works">How It Works</a> &middot;
  <a href="https://github.com/Recusive/Mosaic/issues">Issues</a>
</p>

---

## Why Mosaic?

Every AI agent today ships blind.

```
1. User gives a prompt
2. Agent writes code
3. Agent says "done"
```

There is no step 4. No verification. No *"let me check if that actually works."*

Unit tests mock everything. E2E tests click the DOM from outside. Neither gives the agent what it needs — the ability to verify its own work **inside the running application**.

Mosaic closes the loop.

## Quick Start

```bash
# Install
curl -fsSL https://mosaic.sh/install | sh

# Run a test
mosaic test sidebar-width
```

```
[mosaic] sidebar-width
[mosaic] Tauri → WebKit Inspector → localhost:1420
[mosaic] ──────────────────────────────────────────

Phase 1 — Layout:
  ✓ Set viewport to 1200px → sidebar visible           1.2s
  ✓ Set viewport to 800px  → sidebar collapsed          0.8s
  ✓ Set viewport to 600px  → sidebar hidden              0.9s

[mosaic] ──────────────────────────────────────────
[mosaic] 3 passed · 0 failed · 0 skipped · 2.9s
[mosaic] Exit 0
```

The agent reads the output. Exit 0? Done. Non-zero? Read the failure, fix, rerun.

## Write a Test

Tests live in `.mosaic/` and have direct access to your app's internals — stores, functions, DOM, everything.

```typescript
// .mosaic/sidebar-width.test.ts

import { useLayoutStore } from '@/stores/layout-store';

export default mosaic.test('sidebar-width', async (t) => {

  await t.phase('Layout', async () => {
    await t.step('Desktop: sidebar visible', async () => {
      await t.viewport.desktop();
      await t.dom.waitFor('.sidebar');
      t.assert(t.dom.visible('.sidebar'), 'Sidebar should be visible');

      const rect = t.dom.rect('.sidebar');
      t.assert(rect.width >= 240, 'Sidebar should be at least 240px');
    });

    await t.step('Tablet: sidebar collapsed', async () => {
      await t.viewport.tablet();
      await t.sleep(300);

      const width = parseInt(t.dom.style('.sidebar', 'width'));
      t.assert(width < 80, 'Sidebar should collapse on tablet');
    });

    await t.step('Mobile: sidebar hidden', async () => {
      await t.viewport.mobile();
      await t.sleep(300);
      t.assert(!t.dom.visible('.sidebar'), 'Sidebar should be hidden on mobile');
    });
  });
});
```

Mosaic handles phases, steps, timing, pass/fail formatting, streaming — you write the test logic.

The `mosaic.test()` global and the full `t.*` API are injected automatically. No imports from Mosaic needed. Your app imports (`@/stores/...`) resolve through your dev server — same module graph, same singletons, same state as the running app.

## How It Works

Mosaic tests run **inside** your application — not from outside.

```
User prompt → Agent builds → mosaic test → Pass? → Done.
                   ↑                          |
                   └──── Fail: keep going ────┘
```

A test script gets direct access to:

| What | Examples |
|------|----------|
| **State management** | Zustand, Redux, MobX, Pinia, custom stores |
| **Internal functions** | `handleSend()`, `processQueue()`, any export |
| **DOM** | Click, type, scroll, keyboard shortcuts, drag & drop |
| **Network** | Intercept, mock, verify API calls |
| **File system** | Read, write, verify file contents |
| **IPC** | Tauri invoke, Electron IPC |

Playwright clicks a button and checks if the DOM changed. **Mosaic calls the function behind the button**, verifies the store updated, confirms the file system reflects the change, *and* checks the DOM. Same runtime. Same process. Same state.

## Architecture

Two layers:

```
┌─────────────────────────────────────────────────┐
│  mosaic (Rust binary)                            │
│                                                 │
│  CLI → Auto-Detect → Protocol Adapter → Output  │
│                                                 │
│  Adapters:                                      │
│    WebKit Inspector (Tauri macOS, Safari)        │
│    CDP (Electron, Chrome, Tauri Windows)         │
│    Node Inspector (Node.js backends)             │
│    Process (CLI/TUI tools)                       │
└─────────────────────────────────────────────────┘
           │ injects
           ▼
┌─────────────────────────────────────────────────┐
│  @mosaic/runtime (TypeScript)                    │
│                                                 │
│  t.phase() t.step() t.assert() t.waitFor()     │
│  t.dom.*  t.store.*  t.net.*  t.http.*         │
│  t.fs.*  t.events.*  t.ipc.*  t.proc.*         │
│                                                 │
│  Runs inside the target app                     │
└─────────────────────────────────────────────────┘
```

**Rust** handles orchestration — connects to the app via debug protocols, injects the runtime, captures output, formats results. Ships as a standalone binary with zero dependencies.

**TypeScript** handles the test API — runs inside the app's runtime with full access to its module graph. Tests import your app's stores and functions directly.

The bridge: Mosaic injects a dynamic `import()` through your dev server (Vite, webpack, etc.). Your dev server resolves the test's imports against the app's module graph. The test gets the **same store instances** the app uses.

## CLI

```bash
mosaic test <name>              # Run a test
mosaic test <name> --json       # JSON output
mosaic test <name> --stream     # Real-time NDJSON streaming
mosaic test                     # Run all tests in .mosaic/
mosaic list                     # List available tests
mosaic doctor                   # Environment diagnostic
mosaic init                     # Scaffold .mosaic/ with example test
```

**Zero config.** Mosaic auto-detects your project type, dev server, and debug protocol. It starts your dev server if it's not running. If something fails, it tells you exactly what went wrong and how to fix it.

Override anything with flags:

```bash
mosaic test <name> --port 1420 --protocol webkit --debug-port 9222
```

## Agent Compatible

Mosaic doesn't care which agent runs it. If it can execute a CLI command and read stdout, it works.

| Agent | Integration |
|-------|-------------|
| **Claude Code** | Bash tool / MCP |
| **Codex** | Shell execution |
| **Gemini CLI** | Shell execution |
| **Cursor** | Terminal |
| **Orbit** | Native integration |

Output is structured, machine-readable, and streamable. No interactive prompts. No TUI. Exit codes tell the story: `0` passed, `1` failed, `2` connection error, `3` test file error.

## Platforms

| Platform | Status | Spec |
|----------|--------|------|
| Tauri desktop apps | v1 | [spec-v1.md](docs/spec/spec-v1.md) |
| Backend / Node.js | v1.1 | [spec-v1.1.md](docs/spec/spec-v1.1.md) |
| CLI / TUI tools | v1.2 | [spec-v1.2.md](docs/spec/spec-v1.2.md) |
| Web / Browser | v1.3 | [spec-v1.3.md](docs/spec/spec-v1.3.md) |

Full design spec: [spec-full.md](docs/spec/spec-full.md)

## What Mosaic Is Not

| | |
|-|-|
| **Not a mock tester** | Real state. Real app. If it passes in Mosaic, it works in prod. |
| **Not an E2E driver** | Runs inside the app — not a robot clicking from outside. |
| **Not a test generator** | The agent writes the test. Mosaic runs it. |
| **Not framework-locked** | Desktop, web, backend, CLI. The injection strategy adapts. |
| **Not replacing unit tests** | Different jobs. They coexist. |

## Contributing

Mosaic is open source. Contributions are welcome.

```bash
git clone https://github.com/Recusive/Mosaic.git
cd Mosaic
```

See the [design specs](docs/spec/) to understand the architecture before diving in.

## Community

- [GitHub Issues](https://github.com/Recusive/Mosaic/issues) — bugs and feature requests
- [mosaic.sh](https://mosaic.sh) — updates and docs

## License

MIT &copy; [Recursive Labs](https://recursive.ac)
