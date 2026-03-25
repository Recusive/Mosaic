# What Is Mosaic

**by Recursive Labs**

Version 2.0 | March 2026 | Product Definition

> Single source of truth for what Mosaic is, why it exists, and how it works. Every piece of content — website pages, blog posts, documentation, repo context — draws from this document. Nothing ships that contradicts what's in here.

---

## Quick Reference

**One-liner:** Mosaic closes the agentic loop. One CLI command, and the agent knows if its work actually works.

**The problem:** Every agent today ships blind. It writes code, says "done," and has no way to verify what it built inside the running application.

**The fix:** `mosaic test <name>` — runs test scripts inside the live app. Real state. Real stores. Real backend. Zero mocks. Structured results back to the agent. Pass or iterate.

**The loop:**

```
User prompt → Agent builds → mosaic test → Pass? → Done.
                   ↑                          |
                   └──── Fail: keep going ────┘
```

**Who uses it:** AI agents — Claude Code, Codex, Gemini CLI, Cursor, Orbit. Humans can use it. Agents should.

**Source:** Open source. MIT license.

**Status:** Pre-development. Architecture fully designed. Specs complete.

---

## Part 1: Recursive Labs

Recursive Labs Inc. builds developer tools for the AI-native era.

**Orbit** is the primary product — an AI-native development environment where one agent works across every surface a developer builds with: editor, browser, terminal, vault, and more. Native desktop app. Tauri + Rust + React 19. Available at orbit.build.

**Mosaic** is the second product — the testing layer that was missing from every agent workflow. Standalone CLI. Built for agents to verify their own work inside running applications.

Orbit is where agents build. Mosaic is where agents prove it works.

---

## Part 2: What Mosaic Is

Mosaic is a standalone CLI testing tool that gives AI agents something they've never had: the ability to verify their own work inside a running application.

Not from outside. Not against mocks. Inside. With access to the same state, stores, and functions the application actually uses.

An agent with Mosaic can:

- **Write** test scripts that reach into application internals
- **Run** them inside the live app — `mosaic test <name>`
- **Tail** execution in real time
- **Read** structured logs — pass/fail, durations, errors, state snapshots
- **Decide** — done, or iterate

The primary users are AI agents. Every design decision reflects that — structured output, CLI-first interface, machine-readable logs, no interactive prompts. Humans can use Mosaic directly. But this tool was built for the agent.

---

## Part 3: Why Mosaic Exists

### Every agent today ships blind.

The workflow is always the same:

1. User gives a prompt
2. Agent writes code
3. Agent says "done"

There is no step 4. No verification. No "let me check if that actually works." The agent cannot open the app, click through it, inspect state, or confirm that a sidebar stops at 800 pixels. It writes code and hopes.

### Existing testing tools don't fix this.

**Vitest, Jest, unit runners** — mock-based. They test isolated functions against fake data. A mocked test passes. The real app breaks. These tests lie to the agent.

**Playwright, Cypress, E2E drivers** — external. They click through the DOM from outside. They cannot see stores, cannot access internal state, cannot verify mutations. They were built for human-authored test suites that run on CI — not for an agent that needs to verify its own work in real time and decide whether to keep going.

None of these tools close the loop. They were designed for a world where humans write tests. Mosaic was designed for a world where agents do.

### Mosaic closes the loop.

The agent writes a test. Mosaic runs it inside the app. Results come back. The agent reads them and decides: pass, or try again.

That's it. That's the entire thesis. Every agent today operates in an open loop. Mosaic closes it.

---

## Part 4: How Mosaic Works

### The Loop in Practice

1. **User:** "Make the sidebar collapse at 800px viewport width. Not wider."
2. **Agent writes the code.** Modifies the relevant components.
3. **Agent writes a test.** A Mosaic script that checks sidebar width at various viewport sizes.
4. **Agent runs the test.** `mosaic test sidebar-width`
5. **Mosaic runs inside the app.** The script executes in the application's runtime — measuring actual rendered dimensions against actual state.
6. **Results return.** Structured logs: pass/fail per step, duration, error details.
7. **Agent evaluates.** All steps pass? Done. Step 3 failed? The agent reads why, fixes the code, runs the test again.

The agent can tail execution in real time. It can stream logs as the test runs. It gets the same visibility a developer would get watching the app — except it's automated, structured, and machine-readable.

### Architecture

Two layers:

**`mosaic` — Rust binary.** The CLI orchestrator. Auto-detects the project, starts the dev server if needed, connects via the appropriate debug protocol (WebKit Inspector for Tauri/Safari, Chrome DevTools Protocol for Electron/Chrome, Node Inspector for backends), injects the test, captures output, formats results. Ships as a standalone binary with zero runtime dependencies.

**`@mosaic/runtime` — TypeScript.** The test API that runs inside the target application. Provides `t.step()`, `t.dom.click()`, `t.store.get()`, and every other primitive the agent uses. Ships pre-bundled inside the Rust binary.

### Script Injection — Testing From Inside

Mosaic tests don't run outside the application. They run inside it.

Tests are injected via dynamic `import()` through the app's dev server (Vite, webpack, etc.). The dev server resolves the test's imports against the app's module graph — meaning the test gets the **same store instances**, the **same functions**, the **same state** as the running app.

A Mosaic test script has direct access to:

- **Application stores** — Zustand, Redux, MobX, Pinia, custom state management
- **Application functions** — `handleSend()`, `handleRewind()`, internal APIs
- **DOM** — click, type, scroll, keyboard shortcuts, drag & drop
- **Network** — intercept, mock, verify API calls
- **File system** — read files, verify content, check existence
- **IPC** — Tauri invoke, Electron IPC

This is the fundamental difference. Playwright clicks a button and checks if the DOM changed. Mosaic calls the function behind the button, verifies the store updated, confirms the file system reflects the change, and checks the DOM — all from inside the application. Same runtime. Same process. Same state.

### What a Test Looks Like

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
      t.assert(!t.dom.visible('.sidebar'), 'Sidebar should hide on mobile');
    });
  });
});
```

**Output:**

```
[mosaic] sidebar-width
[mosaic] Tauri → WebKit Inspector → localhost:1420
[mosaic] ──────────────────────────────────────────

Phase 1 — Layout:
  ✓ Desktop: sidebar visible                              1.2s
  ✓ Tablet: sidebar collapsed                              0.8s
  ✓ Mobile: sidebar hidden                                 0.9s

[mosaic] ──────────────────────────────────────────
[mosaic] 3 passed · 0 failed · 0 skipped · 2.9s
[mosaic] Exit 0
```

Every step produces structured output: step name, phase, duration, pass/fail, error message, state snapshots. Machine-readable. Agent-consumable. Exit code 0 = all passed, 1 = test failure, 2 = connection error, 3 = test file error.

### Test API

The full `t.*` API covers 10 domains:

| Domain | What it does | Examples |
|--------|-------------|----------|
| **Core** | Test structure and assertions | `t.phase()`, `t.step()`, `t.assert()`, `t.waitFor()`, `t.cleanup()` |
| **State** | Store access | `t.store.get()`, `t.store.set()`, `t.store.waitFor()` |
| **DOM** | UI interaction | `t.dom.click()`, `t.dom.type()`, `t.dom.press()`, `t.dom.drag()` |
| **Network** | Request interception | `t.net.intercept()`, `t.net.mock()`, `t.net.waitFor()` |
| **Viewport** | Responsive testing | `t.viewport.mobile()`, `t.viewport.tablet()`, `t.viewport.desktop()` |
| **Filesystem** | File operations | `t.fs.read()`, `t.fs.write()`, `t.fs.exists()` |
| **Events** | App events | `t.events.emit()`, `t.events.on()`, `t.events.waitFor()` |
| **IPC** | Desktop app backend | `t.ipc.invoke()`, `t.ipc.on()`, `t.ipc.emit()` |
| **HTTP** | API testing | `t.http.get()`, `t.http.post()`, `t.http.assertStatus()` |
| **Process** | CLI/TUI testing | `t.proc.spawn()`, `t.proc.interactive()`, `t.proc.output()` |

Full API reference: [spec-full.md](../spec/spec-full.md#7-test-api--complete-reference)

---

## Part 5: Testing by Platform

Mosaic supports four platforms. Each uses a different protocol adapter but exposes the same test API and output format.

### Desktop Applications (v1)

**Status:** Architecture designed. Implementation next.

**How it works:** Connects to the app's webview via WebKit Inspector Protocol (Tauri macOS/Linux, Safari) or Chrome DevTools Protocol (Electron, Tauri Windows). Injects tests through the Vite dev server. Tests share the app's module graph.

**Auto-detection:** Reads `tauri.conf.json` or Electron config. Auto-starts `cargo tauri dev` with the right debug flags.

**API scope:** Core, State, DOM, Network, Viewport, Filesystem, Events, IPC.

**Evidence:** The Orbit rewind stress test — 7 steps across 2 phases. Sends messages, rewinds conversation state, verifies context integrity. All from inside the running app. Originally 792 lines of manual code. With Mosaic: ~50 lines.

Full spec: [spec-v1.md](../spec/spec-v1.md)

### Backend / Node.js (v1.1)

**Status:** Architecture designed.

**Two modes:**
- **Inside the process** — test runs inside the Node.js backend, imports app modules directly. Same approach as frontend testing.
- **Outside the process** — test makes HTTP calls to a running API. Works with any backend language (Python, Go, Rust, Java).

**Auto-detection:** Reads `package.json` for entry points and start scripts. Auto-starts the server and waits for health check.

**API scope:** Core, HTTP, Process, Filesystem.

Full spec: [spec-v1.1.md](../spec/spec-v1.1.md)

### CLI / TUI Tools (v1.2)

**Status:** Architecture designed.

**How it works:** Spawns the target CLI tool and interacts via stdin/stdout/stderr pipes. Supports simple command testing, interactive wizards, and full TUI applications with PTY support.

**Auto-detection:** Reads `package.json` bin field or `Cargo.toml` [[bin]] sections. Auto-builds if stale.

**API scope:** Core, Process (extended with interactive sessions), Filesystem.

Full spec: [spec-v1.2.md](../spec/spec-v1.2.md)

### Web / Browser Applications (v1.3)

**Status:** Architecture designed.

**How it works:** Same injection mechanism as desktop — connect via CDP, inject through dev server. Additionally manages browser lifecycle (launch Chrome/Edge headless, navigate to app URL).

**Auto-detection:** Detects framework (React/Vite, Next.js, Vue, Svelte, Angular, Remix, Astro, etc.). Auto-starts dev server and browser.

**API scope:** Full — Core, State, DOM, Network, Viewport, Events, Filesystem, HTTP.

Full spec: [spec-v1.3.md](../spec/spec-v1.3.md)

---

## Part 6: CLI

```
mosaic test <name>              Run a test
mosaic test <name> --json       JSON output
mosaic test <name> --stream     Real-time NDJSON streaming
mosaic test                     Run all tests in .mosaic/
mosaic list                     List available tests
mosaic doctor                   Environment diagnostic
mosaic init                     Scaffold .mosaic/ with example test
```

**Design principles:**

- **Standalone Rust binary** — no runtime dependency, no package manager required
- **Zero config** — auto-detects project type, dev server, debug protocol. Auto-starts everything.
- **Machine-readable output** — structured text (default), JSON (`--json`), streaming NDJSON (`--stream`)
- **Agent-first** — no interactive prompts, no TUI, no human-centric formatting unless requested
- **Loud failures** — when auto-detection fails, reports exactly what was tried and how to fix it
- **Manual override** — every auto-detected value can be overridden with flags

### Any Agent Can Use It

Mosaic doesn't care which agent invokes it. If it can run a CLI command and read the output, it can use Mosaic.

- **Claude Code** — Bash tool or MCP integration
- **Codex** — shell execution
- **Gemini CLI** — shell execution
- **Cursor** — terminal
- **Orbit** — native integration via terminal and agent tool system

---

## Part 7: What Mosaic Is Not

**Not a mock tester.** Real state. Real app. Real backend. If the test passes in Mosaic, it works in the app. No fake anything.

**Not an external driver.** Mosaic doesn't click through the DOM from outside like a robot pretending to be a user. It runs inside the app with access to the same internals the app uses.

**Not a test generator.** The agent writes the test. Mosaic runs it. The agent is the brain. Mosaic is the hands.

**Not a human-first tool.** Humans can use it. But structured output, CLI interface, machine-readable logs — every decision optimizes for the agent.

**Not framework-locked.** Not React-only. Not Zustand-only. Not web-only. Desktop apps, web apps, backend, CLI/TUI tools. The injection strategy adapts to the platform.

**Not a replacement for unit tests.** Unit tests verify isolated functions. Mosaic verifies the whole system works from inside the running application. Different jobs. They coexist.

---

## Part 8: Current Status

| Item | Status |
|------|--------|
| Product concept | Defined |
| Architecture | Fully designed — Rust binary + TypeScript runtime |
| CLI interface | Designed — `mosaic test`, `mosaic list`, `mosaic doctor`, `mosaic init` |
| Implementation language | Rust (CLI binary), TypeScript (test runtime) |
| Distribution | Standalone binary via `curl -fsSL https://mosaic.sh/install \| sh` |
| Source | Open source, MIT license |
| Product code | Pre-development |
| Marketing site | Live at mosaic.sh |
| Internal validation | Stress tests on Orbit confirm the script injection approach |

### Design Specs

All architecture decisions are documented:

- [spec-full.md](../spec/spec-full.md) — Complete design specification
- [spec-v1.md](../spec/spec-v1.md) — Tauri desktop app testing
- [spec-v1.1.md](../spec/spec-v1.1.md) — Backend / Node.js testing
- [spec-v1.2.md](../spec/spec-v1.2.md) — CLI / TUI tool testing
- [spec-v1.3.md](../spec/spec-v1.3.md) — Browser webapp testing

### Remaining TBD

- Build system and CI/CD pipeline
- Distribution infrastructure (install script, binary hosting)
- Exact Rust crate selections (candidates identified in spec-v1.md)
- MCP server integration for deeper agent access
- Plugin system for custom protocol adapters
- `--watch` mode for file-change-triggered reruns

---

## Part 9: Content Rules

For anyone creating website pages, blog posts, or documentation from this document.

### Say This

- Mosaic is built by Recursive Labs
- Mosaic is open source under the MIT license
- Mosaic is a standalone CLI testing tool built for AI agents
- Mosaic closes the agentic loop
- Mosaic runs tests inside running applications — not from outside
- Mosaic accesses real application state: stores, functions, file system, DOM, network
- Mosaic produces structured, machine-readable test results
- Mosaic is built with Rust (CLI) and TypeScript (test runtime)
- Desktop app testing architecture is proven via Orbit stress tests
- Backend, CLI/TUI, and browser testing are designed and specced
- Any agent with CLI access can use Mosaic
- Mosaic is invoked via `mosaic test <name>`
- Mosaic is currently in pre-development — specs are complete, implementation is next
- Mosaic auto-detects project type, dev server, and debug protocol with zero configuration

### Don't Say This

- Don't claim any platform's implementation works today — specs are complete, code is not
- Don't cite performance numbers — none exist
- Don't claim users — there are none, the product is pre-development
- Don't say Mosaic generates tests — the agent generates tests, Mosaic runs them
- Don't say Mosaic replaces unit testing — it complements it
- Don't reference framework support beyond what's been validated (Zustand on Orbit)
- Don't fabricate testimonials, logos, user counts, or social proof
- Don't claim "any language" for test scripts — tests are TypeScript (because they run in browser/Node contexts)

---

## Changelog

### March 25, 2026

- v2.0 — Major update. Architecture fully designed (Rust CLI + TypeScript runtime). All four platforms specced (Tauri, backend, CLI/TUI, browser). Open source under MIT. CLI syntax defined. Test API defined across 10 domains. Updated content rules for open source status.

### March 2026

- v1.0 — Initial product definition. All claims verified against internal stress tests and founder input.
