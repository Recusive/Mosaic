# Mosaic CLI — Complete Design Specification

**Version:** 1.0
**Date:** March 25, 2026
**Status:** Pre-development
**Author:** Recursive Labs

> The testing CLI built for AI agents. One command. Real app. Real state. The agent knows if it works.

---

## Table of Contents

1. [Thesis](#1-thesis)
2. [Architecture](#2-architecture)
3. [Protocol Adapters](#3-protocol-adapters)
4. [Auto-Detection & Environment Management](#4-auto-detection--environment-management)
5. [Test Files & Conventions](#5-test-files--conventions)
6. [Runtime Injection](#6-runtime-injection)
7. [Test API — Complete Reference](#7-test-api--complete-reference)
8. [Output Formats](#8-output-formats)
9. [CLI Interface](#9-cli-interface)
10. [Platform Roadmap](#10-platform-roadmap)
11. [Security Considerations](#11-security-considerations)
12. [Technical Decisions Log](#12-technical-decisions-log)

---

## 1. Thesis

Every AI agent today ships blind. The workflow is always the same:

1. User gives a prompt
2. Agent writes code
3. Agent says "done"

There is no step 4. No verification. No "let me check if that actually works."

Existing tools don't fix this. Unit tests mock everything — a mocked test passes while the real app breaks. E2E drivers like Playwright click the DOM from outside — they can't see stores, can't access internal state, can't verify mutations. Neither gives the agent what it needs: the ability to verify its own work **inside the running application**.

Mosaic closes the loop:

```
User prompt → Agent builds → mosaic test → Pass? → Done.
                   ^                          |
                   └──── Fail: keep going ────┘
```

The agent writes a test. Mosaic runs it inside the app. Results come back structured and machine-readable. The agent reads them and decides: pass, or try again.

**One CLI. One command. Every platform.**

---

## 2. Architecture

Mosaic is two layers:

1. **`mosaic` — Rust binary.** The CLI orchestrator. Handles environment detection, protocol connections, script injection, output formatting. Ships as a standalone binary with zero runtime dependencies.

2. **`@mosaic/runtime` — TypeScript.** The test API that runs inside the target application. Provides `t.step()`, `t.dom.click()`, `t.store.get()`, and every other primitive the agent uses. Ships pre-bundled inside the Rust binary.

```
┌─────────────────────────────────────────────────────────────────┐
│                    mosaic (Rust binary)                          │
│                                                                 │
│  ┌── CLI Parser (clap) ──────────────────────────────────────┐  │
│  │  mosaic test <name>                                        │  │
│  │  mosaic test <name> --json --stream                        │  │
│  │  mosaic test <name> --port 1420 --protocol webkit          │  │
│  │  mosaic list                                               │  │
│  │  mosaic doctor                                             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│  ┌── Auto-Detector ──────────┴───────────────────────────────┐  │
│  │  1. Scan project files (tauri.conf.json, package.json...) │  │
│  │  2. Find or start dev server                               │  │
│  │  3. Detect and connect debug protocol                      │  │
│  │  4. On failure → loud, actionable error                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│  ┌── Protocol Adapters ──────┴───────────────────────────────┐  │
│  │                                                            │  │
│  │  WebKit Inspector ── Tauri macOS/Linux, Safari             │  │
│  │  CDP ─────────────── Electron, Chrome, Tauri Windows       │  │
│  │  Node Inspector ──── Node.js backends                      │  │
│  │  Process ─────────── CLI/TUI tools (stdin/stdout/stderr)   │  │
│  │  HTTP ────────────── Any backend via API calls              │  │
│  │                                                            │  │
│  │  Common trait:                                             │  │
│  │    connect() → evaluate(js) → on_console(cb) → close()    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│  ┌── Output Engine ──────────┴───────────────────────────────┐  │
│  │  Structured text (default) │ JSON (--json) │ Stream mode   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌── Embedded Assets ────────────────────────────────────────┐  │
│  │  @mosaic/runtime.js (pre-bundled TypeScript test API)      │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Principle

Every protocol adapter implements the same Rust trait:

```rust
trait ProtocolAdapter {
    async fn connect(&mut self, config: &ConnectionConfig) -> Result<()>;
    async fn evaluate(&self, javascript: &str) -> Result<EvalResult>;
    async fn on_console(&self, callback: impl Fn(ConsoleEvent)) -> Result<()>;
    async fn close(&mut self) -> Result<()>;
}
```

The rest of Mosaic doesn't know or care which protocol is active. Add a new platform = add a new adapter. Everything else stays the same.

### Why Rust

- **Standalone binary** — zero runtime dependencies. Agent runs `mosaic test`, it just works. No `npm install`, no Node required.
- **Fast startup** — agents run this in a tight loop. Milliseconds matter.
- **Protocol handling** — excellent WebSocket and async I/O via `tokio`.
- **Foundation** — built right from day one. No rewrite path needed.

### Why TypeScript for the Runtime

- **Browser reality** — tests that run inside webviews execute JavaScript. TypeScript is the only option.
- **Module graph sharing** — tests import from the app's source. Same language, same module system.
- **Agent familiarity** — every coding agent writes TypeScript fluently.

---

## 3. Protocol Adapters

Different platforms expose different debug protocols. Mosaic adapts transparently.

| Platform | Engine | Protocol | Port (default) |
|----------|--------|----------|----------------|
| Tauri (macOS) | WebKit (WKWebView) | WebKit Inspector Protocol | 9222 |
| Tauri (Linux) | WebKitGTK | WebKit Inspector Protocol | 9222 |
| Tauri (Windows) | WebView2 (Chromium) | Chrome DevTools Protocol | 9222 |
| Electron | Chromium | Chrome DevTools Protocol | 9222 |
| Chrome / Edge | Chromium | Chrome DevTools Protocol | 9222 |
| Safari | WebKit | WebKit Inspector Protocol | 9222 |
| Node.js | V8 | Node Inspector Protocol | 9229 |
| CLI/TUI tools | N/A | Process (stdin/stdout/stderr) | N/A |

### Common Capabilities (all adapters)

Every adapter supports:

- **`evaluate(js)`** — execute JavaScript in the target context
- **`on_console(cb)`** — capture console output (structured events from the runtime)
- **`close()`** — clean disconnect

### WebKit Inspector Adapter

For Tauri on macOS and Linux. Connects via WebSocket to the WebKit Inspector Protocol.

**Activation:** The target app must be launched with the `WEBKIT_INSPECTOR_SERVER` environment variable:

```
WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222 cargo tauri dev
```

Mosaic handles this automatically when it starts the dev server (see Section 4).

**Key protocol methods used:**
- `Runtime.evaluate` — inject and execute JavaScript
- `Console.messageAdded` — capture console output
- `Page.navigate` — navigation control
- `DOM.*` — DOM inspection when needed

### CDP Adapter

For Electron, Chrome, Edge, and Tauri on Windows. Connects via WebSocket to the Chrome DevTools Protocol.

**Activation:** Launch with `--remote-debugging-port=9222`.

**Key protocol methods used:**
- `Runtime.evaluate` — inject and execute JavaScript
- `Runtime.consoleAPICalled` — capture console output
- `Page.*` — page lifecycle
- `Network.*` — network interception
- `DOM.*` — DOM inspection

### Node Inspector Adapter

For Node.js backend testing. Connects via WebSocket to V8's inspector protocol.

**Activation:** Launch with `--inspect=9229`.

**Key protocol methods used:**
- `Runtime.evaluate` — execute JavaScript in the Node process
- `Runtime.consoleAPICalled` — capture output

### Process Adapter

For CLI/TUI tool testing. No debug protocol — communicates via stdin/stdout/stderr pipes.

**Activation:** Mosaic spawns the target process directly.

---

## 4. Auto-Detection & Environment Management

Mosaic's design principle: **the agent runs `mosaic test <name>` and everything else is handled automatically.** No config files. No manual setup. No extra commands.

### Detection Layers

**Layer 1 — Project Scanning** (reads files, instant):

```
tauri.conf.json found       → Tauri project
  → Read devUrl for dev server port
  → macOS/Linux: expect WebKit Inspector Protocol
  → Windows: expect CDP

electron-builder.* found    → Electron project
  → Read main entry point
  → Expect CDP

next.config.* found         → Next.js project
  → Default port 3000
  → Expect CDP (browser)

vite.config.* found         → Vite project
  → Read server.port for dev server port
  → Default 5173

package.json found          → Read "dev" script for port hints
  → Check "bin" field for CLI tools
  → Check dependencies for framework detection

Cargo.toml found            → Check for Tauri dependency
  → Check [[bin]] for CLI tools
```

**Layer 2 — Environment Assurance** (not just detection — makes it running):

```
Dev server alive on detected port?
  YES → proceed to protocol connection
  NO  → start it automatically:
        Tauri:    WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222 cargo tauri dev
        Electron: electron . --remote-debugging-port=9222
        Vite:     npx vite --port 5173
        Next.js:  npx next dev --port 3000
        Node:     node --inspect=9229 <entry>

Debug protocol responding?
  YES → connect
  NO  → app was started with right flags (Layer 2 handles this)

After test completes:
  → Keep dev server running (agent will test again soon)
  → Only tear down if mosaic started it AND --cleanup flag is passed
```

**Layer 3 — Failure with Full Transparency** (nothing silent, ever):

When Mosaic cannot self-recover, it reports exactly what was tried:

```
[mosaic] ✗ Could not connect to your app.

  What I checked:
    ✓ Found tauri.conf.json → Tauri project
    ✓ Config says devUrl: http://localhost:1420
    ✗ No response on localhost:1420 — is your dev server running?
    ✗ No WebKit Inspector on localhost:9222

  What I tried:
    ✗ Ran: WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222 cargo tauri dev
    ✗ Error: cargo-tauri not found

  How to fix:
    1. Install Tauri CLI: cargo install tauri-cli
    2. Then retry: mosaic test rewind-stress

  Or connect manually:
    mosaic test rewind-stress --dev-server http://localhost:1420 \
                              --protocol webkit \
                              --debug-port 9222

  Run 'mosaic doctor' for a full diagnostic.
```

Every check is logged. Every failure shows what was tried and the exact command to fix it. The agent reads this and either fixes the issue or passes flags explicitly.

### Manual Override

Any auto-detected value can be overridden with flags:

```
mosaic test <name> --port 1420           Override dev server port
mosaic test <name> --debug-port 9222     Override debug protocol port
mosaic test <name> --protocol webkit     Force protocol (webkit | cdp | node)
mosaic test <name> --dev-server <url>    Explicit dev server URL
mosaic test <name> --no-auto-start       Don't start dev server automatically
mosaic test <name> --target tauri        Force target platform
```

### Optional Config File

For CI pipelines, shared team settings, or unusual setups — an optional config file:

```typescript
// .mosaic/config.ts
export default {
  target: 'tauri',
  devServer: 'http://localhost:1420',
  debugPort: 9222,
  protocol: 'webkit',
  startCommand: 'cargo tauri dev',
  timeout: 60_000,
}
```

This is never required. It exists for cases where auto-detection doesn't fit.

### `mosaic doctor`

Standalone diagnostic command for debugging environment issues:

```
[mosaic] Environment Diagnostic

  Project:
    ✓ tauri.conf.json found
    ✓ Tauri project detected
    ✓ devUrl: http://localhost:1420
    ✓ .mosaic/ directory found (3 tests)

  Dev Server:
    ✓ localhost:1420 responding (Vite 5.4.2)

  Debug Protocol:
    ✗ No WebKit Inspector detected
    → Launch with: WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222 cargo tauri dev

  Tests:
    rewind-stress.test.ts    (last modified: 2m ago)
    sidebar-width.test.ts    (last modified: 1h ago)
    api-auth.test.ts         (last modified: 3d ago)

  Status: Almost ready — enable WebKit Inspector to start testing.
```

---

## 5. Test Files & Conventions

### Directory Structure

```
your-project/
├── src/
├── .mosaic/
│   ├── rewind-stress.test.ts
│   ├── sidebar-width.test.ts
│   ├── api-auth.test.ts
│   └── config.ts              ← optional
├── tauri.conf.json
├── package.json
```

Tests live in `.mosaic/` in the project root. Hidden directory — it's tooling, not source code. Same convention as `.github/`, `.vscode/`, `.husky/`.

### Naming Convention

- File: `<name>.test.ts`
- Command: `mosaic test <name>`
- Resolution: `mosaic test rewind-stress` → `.mosaic/rewind-stress.test.ts`

Flat directory. No nesting. The test name IS the file name.

### Test File Format

```typescript
// .mosaic/rewind-stress.test.ts

// Import directly from the app — shared module graph via dev server
import { useChatStore } from '@/stores/chat/chat-store';
import { useCheckpointStore } from '@/stores/agent/checkpoint-store';

export default mosaic.test('rewind-stress', async (t) => {

  const handleSend = useChatStore.getState().handleSend;
  const getMessages = () => useChatStore.getState().messages;

  t.cleanup(async () => {
    // Runs after test, pass or fail
    useChatStore.getState().clearSession();
  });

  await t.phase('Setup', async () => {
    await t.step('Send message 1', async () => {
      handleSend('Remember 42. Reply with "Got it, 42." only.');
      await t.waitFor(() => getMessages().length === 2, { timeout: 30_000 });
    });

    await t.step('Send message 2', async () => {
      handleSend('Remember 73. Reply with "Got it, 73." only.');
      await t.waitFor(() => getMessages().length === 4, { timeout: 30_000 });
    });
  });

  await t.phase('Rewind', async () => {
    await t.step('Rewind to msg 2 response', async () => {
      const target = getMessages().filter(m => m.role === 'assistant')[1];
      t.ctx.set('rewindTarget', target.id);
      handleRewind(target.id);
      await t.waitFor(() => getMessages().length === 4);
    });

    await t.step('Verify context after rewind', async () => {
      handleSend('What numbers do you remember?');
      await t.waitFor(() => {
        const last = getMessages().at(-1);
        return last?.role === 'assistant' && !last.isStreaming;
      }, { timeout: 60_000 });

      const response = getMessages().at(-1).content;
      t.assert(response.includes('42'), 'Should remember 42');
      t.assert(response.includes('73'), 'Should remember 73');
      t.assert(!response.includes('99'), 'Should NOT remember 99');
    });
  });
});
```

Key points:
- `mosaic.test()` is a global — injected by the runtime before the test loads
- App imports resolve through the dev server's module graph — same singletons as the running app
- `t.cleanup()` ensures teardown happens regardless of test outcome
- `t.ctx` shares data between steps without closure gymnastics

### Running All Tests

```
mosaic test                    Run all tests in .mosaic/
mosaic test rewind-stress      Run a specific test
mosaic test rewind sidebar     Run multiple tests by name
```

---

## 6. Runtime Injection

The critical mechanism: how Mosaic gets a test running inside the target application.

### Injection Sequence (Webview/Browser targets)

```
1. Rust CLI connects to target via protocol adapter (WebKit/CDP)
2. Evaluates @mosaic/runtime.js in the page context
   → Sets up global `mosaic` object with full API
   → Sets up console event emitter for structured output
   → Sets up error boundaries and timeout handling
3. Evaluates dynamic import:
   → await import('http://localhost:<port>/.mosaic/rewind-stress.test.ts')
4. Dev server (Vite/webpack/etc.) resolves the test file
   → Compiles TypeScript → JavaScript
   → Resolves all imports against the app's module graph
   → Returns the bundled test with shared module instances
5. Test's default export (mosaic.test callback) registers with runtime
6. Runtime executes the test:
   → Runs phases and steps in order
   → Captures pass/fail/duration for each step
   → Emits structured events to console
7. Rust CLI captures console events via protocol
8. Output engine formats and streams to stdout
9. Runtime emits completion event with summary
10. Rust CLI reports final results and exits with appropriate code
```

### Why Dynamic Import Through the Dev Server

This is the key insight that makes Mosaic work without an SDK or app modifications.

When the test does `import { useChatStore } from '@/stores/chat/chat-store'`, that import goes through the dev server (Vite, webpack, etc.) which resolves it to the SAME module instance the app is using. The test gets the real store, with real state, connected to the real app.

If Mosaic bundled the test separately, each import would create new module instances — a separate store with empty state, disconnected from the app. That defeats the entire purpose.

The dev server IS the bridge between "test file on disk" and "code running inside the app with full access."

### Injection Sequence (Node.js Backend)

```
1. Rust CLI starts Node process:
   → node --import @mosaic/runtime <test-file>
   OR
   → node --require @mosaic/runtime-cjs <test-file>
2. Runtime initializes in the Node process
   → Sets up global `mosaic` object
   → Sets up stdout event emitter for structured output
3. Test file is loaded and executed
   → Imports resolve from the app's node_modules
4. Same execution flow as webview targets
5. Rust CLI captures stdout events
6. Output engine formats and streams
```

### Injection Sequence (CLI/TUI Tools)

```
1. Rust CLI loads the test file directly (it's a test definition, not injected code)
2. Runtime runs in the Rust process (or a helper Node process)
3. Test steps spawn/interact with the target CLI tool via process pipes
4. No injection into the target — the test orchestrates from outside
5. Same structured output flow
```

---

## 7. Test API — Complete Reference

All methods are available on `t` inside a `mosaic.test()` callback.

### Core

Every test uses these. Structure, assertions, timing, logging.

```typescript
// Test structure
t.phase(name: string, fn: async () => void)
  // Group steps into named phases. Appears as a section in output.

t.step(name: string, fn: async () => void, opts?: { retries?: number })
  // Named test step. Pass/fail/duration tracked individually.
  // Optional retries for flaky async operations.

// Assertions
t.assert(condition: boolean, message: string)
  // Boolean assertion. Fails the step if false.

t.assertEqual(actual: any, expected: any, message?: string)
  // Deep equality assertion.

t.assertMatch(str: string, pattern: RegExp | string)
  // Regex or substring match.

t.assertTiming(fn: async () => void, opts: { max: number })
  // Assert that fn completes within max milliseconds.

// Flow control
t.waitFor(predicate: () => boolean | Promise<boolean>, opts?: { timeout?: number, interval?: number })
  // Poll predicate until true or timeout. Default timeout: 30s. Default interval: 100ms.

t.sleep(ms: number)
  // Delay execution.

t.fail(message: string)
  // Explicitly fail the current step.

t.skip(reason?: string)
  // Skip the current step with an optional reason.

// Logging and debugging
t.log(message: string)
  // Structured log line. Appears in output under the current step.

t.snapshot(label: string, data: any)
  // Capture a state snapshot. Included in output on failure.

t.measure(label: string, fn: async () => void)
  // Time a block. Duration reported in output.

// Lifecycle
t.cleanup(fn: async () => void)
  // Register teardown. Runs after test completes, pass or fail.
  // Multiple cleanup handlers run in reverse registration order.

// Shared context
t.ctx.set(key: string, value: any)
  // Store data for use in later steps.

t.ctx.get(key: string): any
  // Retrieve stored data.
```

### State — Store Access

Reach into any state management system. Works with Zustand, Redux, MobX, or any store accessible in the runtime.

```typescript
t.store.get(path: string): any
  // Read a value from a store by dot-notation path.
  // e.g., t.store.get('chatStore.activeSessionId')

t.store.set(path: string, value: any): void
  // Write a value to a store.

t.store.subscribe(path: string, callback: (value: any) => void): () => void
  // Watch for mutations. Returns unsubscribe function.

t.store.snapshot(): Record<string, any>
  // Dump the full state of all registered stores.

t.store.waitFor(path: string, predicate: (value: any) => boolean, opts?: { timeout?: number }): Promise<void>
  // Wait until a store value matches the predicate.
```

Note: Since tests share the app's module graph, agents can also import stores directly. `t.store` is a convenience layer for common patterns.

### DOM — UI Interaction

Interact with and assert against the DOM.

```typescript
// Querying
t.dom.query(selector: string): Element | null
t.dom.queryAll(selector: string): Element[]
t.dom.text(selector: string): string
t.dom.attr(selector: string, name: string): string | null
t.dom.style(selector: string, property: string): string
t.dom.rect(selector: string): DOMRect
t.dom.visible(selector: string): boolean

// Interaction
t.dom.click(selector: string): Promise<void>
t.dom.type(selector: string, text: string): Promise<void>
  // Simulates keystroke-by-keystroke typing.
t.dom.fill(selector: string, text: string): Promise<void>
  // Clears the field, then sets the value. Faster than type().
t.dom.scroll(selector: string, position: { top?: number, left?: number }): Promise<void>
t.dom.focus(selector: string): Promise<void>
t.dom.blur(selector: string): Promise<void>

// Form elements
t.dom.check(selector: string): Promise<void>
t.dom.uncheck(selector: string): Promise<void>
t.dom.select(selector: string, value: string): Promise<void>

// Keyboard
t.dom.press(key: string): Promise<void>
  // Single key or shortcut: 'Enter', 'Ctrl+Z', 'Cmd+Shift+P'
t.dom.hotkey(combo: string): Promise<void>
  // Alias for press() with complex combos.
t.dom.keyDown(key: string): Promise<void>
t.dom.keyUp(key: string): Promise<void>

// Drag and drop
t.dom.drag(fromSelector: string, toSelector: string): Promise<void>
t.dom.dragTo(selector: string, coords: { x: number, y: number }): Promise<void>

// Waiting
t.dom.waitFor(selector: string, opts?: { timeout?: number, visible?: boolean }): Promise<Element>
  // Wait until element exists (and optionally is visible).

// Capture
t.dom.screenshot(selector?: string): Promise<string>
  // Capture screenshot. Returns base64. Full page if no selector.
```

### Network — Intercept and Verify

```typescript
t.net.intercept(pattern: string | RegExp, handler: (req: Request) => Response | void): () => void
  // Intercept matching requests. Return a Response to mock. Return void to passthrough.
  // Returns unsubscribe function.

t.net.waitFor(pattern: string | RegExp, opts?: { timeout?: number }): Promise<Request>
  // Wait for a matching request to fire.

t.net.mock(pattern: string | RegExp, response: { status?: number, body?: any, headers?: Record<string, string> }): () => void
  // Shorthand for intercept that returns a static response.

t.net.requests(): Request[]
  // List all captured requests since test start.

t.net.fetch(url: string, opts?: RequestInit): Promise<Response>
  // Make a request from the app's context (same origin, same cookies).
```

### Viewport — Responsive Testing

```typescript
t.viewport.set(width: number, height: number): Promise<void>
t.viewport.get(): { width: number, height: number }
t.viewport.mobile(): Promise<void>    // 375 x 812
t.viewport.tablet(): Promise<void>    // 768 x 1024
t.viewport.desktop(): Promise<void>   // 1440 x 900
```

### Filesystem

```typescript
t.fs.read(path: string): Promise<string>
t.fs.write(path: string, content: string): Promise<void>
t.fs.exists(path: string): Promise<boolean>
t.fs.glob(pattern: string): Promise<string[]>
t.fs.diff(path: string, expected: string): Promise<string | null>
  // Returns null if identical, diff string if different.
```

### Events — App Event System

```typescript
t.events.emit(name: string, data?: any): void
  // Dispatch a custom event in the app.

t.events.on(name: string, callback: (data: any) => void): () => void
  // Listen for events. Returns unsubscribe.

t.events.waitFor(name: string, opts?: { timeout?: number }): Promise<any>
  // Wait for a specific event. Returns the event data.
```

### IPC — Desktop App Backend Communication

For Tauri (`invoke`) and Electron (`ipcRenderer`/`ipcMain`).

```typescript
t.ipc.invoke(command: string, args?: any): Promise<any>
  // Call a backend command (Tauri invoke / Electron ipcRenderer.invoke).

t.ipc.on(channel: string, callback: (data: any) => void): () => void
  // Listen to IPC channel.

t.ipc.emit(channel: string, data?: any): void
  // Send data to backend process.
```

### HTTP — Backend API Testing

```typescript
t.http.get(url: string, opts?: { headers?: Record<string, string> }): Promise<HttpResponse>
t.http.post(url: string, body?: any, opts?: { headers?: Record<string, string> }): Promise<HttpResponse>
t.http.put(url: string, body?: any, opts?: { headers?: Record<string, string> }): Promise<HttpResponse>
t.http.delete(url: string, opts?: { headers?: Record<string, string> }): Promise<HttpResponse>

t.http.assertStatus(response: HttpResponse, code: number): void
t.http.assertBody(response: HttpResponse, expected: any): void
  // Deep equality on response body.
t.http.assertHeader(response: HttpResponse, key: string, value: string): void

// HttpResponse shape:
interface HttpResponse {
  status: number;
  headers: Record<string, string>;
  body: any;
}
```

### Process — Spawn and Verify

```typescript
t.proc.spawn(command: string, args?: string[], opts?: { env?: Record<string, string>, cwd?: string }): Promise<ProcessHandle>
t.proc.output(handle: ProcessHandle): Promise<string>
  // Read stdout.
t.proc.error(handle: ProcessHandle): Promise<string>
  // Read stderr.
t.proc.exitCode(handle: ProcessHandle): Promise<number>
t.proc.waitForExit(handle: ProcessHandle, opts?: { timeout?: number }): Promise<number>
  // Wait for process to exit. Returns exit code.
t.proc.kill(handle: ProcessHandle): Promise<void>
t.proc.env(key: string): string | undefined
  // Read environment variable.

// Interactive sessions (for CLI/TUI testing)
t.proc.interactive(command: string, args?: string[], opts?: SpawnOpts): Promise<InteractiveSession>

interface InteractiveSession {
  type(input: string): Promise<void>;
  waitForOutput(pattern: string | RegExp, opts?: { timeout?: number }): Promise<string>;
  screen(): Promise<string>;       // Current terminal screen content
  exitCode(): Promise<number>;
  waitForExit(code?: number): Promise<void>;
  kill(): Promise<void>;
}
```

### API Availability by Platform

| API Domain | v1 (Tauri) | v1.1 (Backend) | v1.2 (CLI/TUI) | v1.3 (Webapp) |
|------------|------------|----------------|-----------------|---------------|
| Core       | ✓          | ✓              | ✓               | ✓             |
| State      | ✓          | -              | -               | ✓             |
| DOM        | ✓          | -              | -               | ✓             |
| Network    | ✓          | -              | -               | ✓             |
| Viewport   | ✓          | -              | -               | ✓             |
| Filesystem | ✓          | ✓              | ✓               | ✓             |
| Events     | ✓          | -              | -               | ✓             |
| IPC        | ✓          | -              | -               | -             |
| HTTP       | -          | ✓              | -               | ✓             |
| Process    | -          | ✓              | ✓               | -             |

---

## 8. Output Formats

### Structured Text (default)

Human-scannable, agent-parseable. Used 90% of the time.

```
[mosaic] rewind-stress
[mosaic] Tauri → WebKit Inspector → localhost:1420
[mosaic] ──────────────────────────────────────────

Phase 1 — Setup:
  ✓ Send message 1                                    1.2s
  ✓ Send message 2                                    0.9s
  ✓ Send message 3                                    1.1s

Phase 2 — Rewind:
  ✓ Rewind to msg 2 response                          0.4s
  ✗ Verify context after rewind                        2.1s
    AssertionError: Should remember 42
      Expected: response contains "42"
      Actual:   "I don't have any prior context."
    Snapshot (store):
      { activeSessionId: "abc12345", messages: 4, rewindEpoch: 2 }
  ○ Send post-rewind message                           — SKIPPED

[mosaic] ──────────────────────────────────────────
[mosaic] 4 passed · 1 failed · 1 skipped · 5.7s
[mosaic] Exit 1
```

Design choices:
- Header shows connection info (agent confirms infra is working)
- Passing steps: one line — icon, name, duration
- Failed steps expand: error type, expected vs actual, state snapshot
- Skipped steps show reason
- Footer: totals, total duration, exit code — one parseable line

### JSON (`--json` flag)

Pure machine output. For piping, programmatic analysis, or integration with other tools.

```json
{
  "test": "rewind-stress",
  "target": "tauri",
  "protocol": "webkit",
  "devServer": "http://localhost:1420",
  "duration": 5700,
  "passed": 4,
  "failed": 1,
  "skipped": 1,
  "exit": 1,
  "phases": [
    {
      "name": "Setup",
      "steps": [
        { "name": "Send message 1", "status": "pass", "duration": 1200 },
        { "name": "Send message 2", "status": "pass", "duration": 900 },
        { "name": "Send message 3", "status": "pass", "duration": 1100 }
      ]
    },
    {
      "name": "Rewind",
      "steps": [
        { "name": "Rewind to msg 2 response", "status": "pass", "duration": 400 },
        {
          "name": "Verify context after rewind",
          "status": "fail",
          "duration": 2100,
          "error": {
            "type": "AssertionError",
            "message": "Should remember 42",
            "expected": "response contains \"42\"",
            "actual": "I don't have any prior context."
          },
          "snapshot": {
            "activeSessionId": "abc12345",
            "messages": 4,
            "rewindEpoch": 2
          }
        },
        {
          "name": "Send post-rewind message",
          "status": "skip",
          "reason": "previous step failed"
        }
      ]
    }
  ]
}
```

### Streaming (`--stream` flag)

NDJSON — one event per line, emitted in real-time as steps complete. The agent doesn't wait for the full test to finish.

```
{"event":"start","test":"rewind-stress","target":"tauri","protocol":"webkit"}
{"event":"phase","name":"Setup"}
{"event":"step","name":"Send message 1","status":"pass","duration":1200}
{"event":"step","name":"Send message 2","status":"pass","duration":900}
{"event":"step","name":"Send message 3","status":"pass","duration":1100}
{"event":"phase","name":"Rewind"}
{"event":"step","name":"Rewind to msg 2 response","status":"pass","duration":400}
{"event":"step","name":"Verify context after rewind","status":"fail","duration":2100,"error":"Should remember 42"}
{"event":"step","name":"Send post-rewind message","status":"skip","reason":"previous step failed"}
{"event":"end","passed":4,"failed":1,"skipped":1,"duration":5700,"exit":1}
```

### Exit Codes

```
0    All steps passed
1    One or more steps failed
2    Connection error (couldn't reach the app)
3    Test file error (syntax error, missing file, import failure)
```

The agent doesn't need to parse output for simple pass/fail — check the exit code. `0` means move on. Non-zero means read the output.

---

## 9. CLI Interface

### Commands

```
mosaic test <name>              Run a test by name
mosaic test                     Run all tests in .mosaic/
mosaic test <n1> <n2> <n3>      Run multiple tests by name
mosaic list                     List all tests in .mosaic/
mosaic doctor                   Full environment diagnostic
mosaic init                     Scaffold .mosaic/ directory with example test
```

### Flags for `mosaic test`

```
Output:
  --json                        JSON output instead of structured text
  --stream                      Real-time NDJSON streaming
  --verbose                     Include state snapshots and debug info for all steps

Execution:
  --bail                        Stop on first failure (default: run all steps)
  --timeout <seconds>           Override default timeout (default: 30s per step)

Connection (override auto-detection):
  --port <number>               Dev server port
  --debug-port <number>         Debug protocol port
  --protocol <webkit|cdp|node>  Force protocol
  --dev-server <url>            Explicit dev server URL
  --target <tauri|electron|browser|node|cli>  Force target platform

Lifecycle:
  --no-auto-start               Don't start dev server automatically
  --cleanup                     Tear down dev server after test (if mosaic started it)
```

### `mosaic list` Output

```
[mosaic] Tests in .mosaic/

  rewind-stress          .mosaic/rewind-stress.test.ts       2m ago
  sidebar-width          .mosaic/sidebar-width.test.ts       1h ago
  api-auth               .mosaic/api-auth.test.ts            3d ago

  3 tests found
```

### `mosaic init` Output

Creates `.mosaic/` with an example test:

```
[mosaic] Created .mosaic/
[mosaic] Created .mosaic/example.test.ts

  Edit .mosaic/example.test.ts to write your first test, then run:
    mosaic test example
```

---

## 10. Platform Roadmap

### v1 — Tauri Desktop Apps

**Focus:** Full end-to-end testing of Tauri applications.

**Adapter:** WebKit Inspector Protocol (macOS/Linux), CDP (Windows).

**Injection:** Dynamic import through Vite dev server.

**API scope:** Core, State, DOM, Events, IPC, Filesystem, Viewport, Network.

**Auto-start:** `WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222 cargo tauri dev`

**Validation:** Rewrite the existing Orbit rewind stress test as a Mosaic test. If the 792-line manual test becomes ~50 lines and produces the same results, v1 works.

See [spec-v1.md](spec-v1.md) for full details.

### v1.1 — Backend / Node.js

**Focus:** API testing, database verification, server-side logic.

**Adapter:** Node Inspector Protocol / direct process execution.

**Injection:** Import test file in Node with Mosaic runtime preloaded.

**API scope:** Core, HTTP, Process, Filesystem.

**Auto-start:** Detect entry point from package.json, start with `--inspect`.

See [spec-v1.1.md](spec-v1.1.md) for full details.

### v1.2 — CLI/TUI Tools

**Focus:** Testing command-line tools and terminal UI applications.

**Adapter:** Process adapter with stdin/stdout/stderr streaming.

**Injection:** Test orchestrates from outside — spawns and interacts with target process.

**API scope:** Core, Process (extended with interactive sessions), Filesystem.

**Auto-start:** Detect binary from package.json `bin` field or Cargo.toml `[[bin]]`.

See [spec-v1.2.md](spec-v1.2.md) for full details.

### v1.3 — Browser Webapps

**Focus:** Testing web applications running in a browser.

**Adapter:** CDP (Chrome/Edge), WebKit Inspector (Safari).

**Injection:** Dynamic import through dev server (same mechanism as Tauri).

**API scope:** Full — Core, State, DOM, Network, Viewport, Events, Filesystem, HTTP.

**Auto-start:** Detect framework, start dev server, launch browser with debug flags.

See [spec-v1.3.md](spec-v1.3.md) for full details.

---

## 11. Security Considerations

### Dev Mode Only

Mosaic connects via debug protocols that should only be available in development. The `WEBKIT_INSPECTOR_SERVER`, `--remote-debugging-port`, and `--inspect` flags are dev-time features.

**Mosaic should refuse to connect to production applications.** If a connection target doesn't look like a local dev server (e.g., it's not `localhost` or `127.0.0.1`), Mosaic should warn and require an explicit `--allow-remote` flag.

### Test Isolation

Tests run inside the app with full access — same as a developer in devtools. There is no sandboxing. This is intentional — the power of Mosaic is unrestricted access to the app's internals.

The `t.cleanup()` mechanism exists so tests can restore state after running. Agents should use it.

### No Secrets in Tests

Test files live in `.mosaic/` and may be committed to version control. Tests should never contain secrets, API keys, or credentials. If a test needs authentication, it should use environment variables or the app's existing auth state.

---

## 12. Technical Decisions Log

A record of every major architectural decision and why it was made.

| # | Decision | Chosen | Rejected | Why |
|---|----------|--------|----------|-----|
| 1 | Injection mechanism | Debug protocols (CDP/WebKit Inspector) | SDK in app, build-time injection | Zero app modifications required. Agent runs `mosaic test` against any running app. |
| 2 | Test API style | Rich API with domains (`t.dom.*`, `t.store.*`, etc.) | Raw scripts (agent writes all helpers) | Reduces 792-line test to ~50 lines. Agent focuses on test logic, not plumbing. |
| 3 | API richness | Comprehensive — DOM, state, network, keyboard, drag, forms | Minimal (just assert/waitFor) | Agents build complex apps. They need full access to test everything. |
| 4 | Output format | Structured text + JSON + streaming | JSON only, text only | Structured text for reading, JSON for piping, streaming for long tests. Covers all agent needs. |
| 5 | CLI language | Rust | TypeScript/Node, Go | Standalone binary, fast startup, no runtime deps. Foundation from day one, no rewrite. |
| 6 | Test language | TypeScript | Multiple language SDKs | Tests run in browser/Node contexts. TypeScript is the runtime language. |
| 7 | Module resolution | Dynamic import through app's dev server | Custom bundler, window globals | Tests share the app's module graph — same singletons, same state. No SDK setup needed. |
| 8 | Configuration | Auto-detect + flags fallback, optional config file | Config file required | Agent wants zero friction. `mosaic test` should just work. |
| 9 | Dev server management | Auto-start if not running, keep alive after test | Manual start required | One command. Mosaic handles everything. |
| 10 | Error handling | Loud failure with exact fix commands | Silent fallback, generic errors | Agent needs actionable information to self-recover. |
| 11 | Test file location | `.mosaic/` hidden directory | `mosaic/`, `tests/`, alongside source | Tool directory convention. Doesn't pollute project root. |
| 12 | Platform priority | Tauri → Backend → CLI/TUI → Webapps | All at once, webapps first | Start where the proof of concept exists. Expand with the architecture already proven. |

---

## Appendix: Proof of Concept

The existing Orbit rewind stress test (`apps/agent/src/stress-tests/rewind-stress-test.ts`) proves the core thesis. 792 lines of TypeScript that:

1. Runs inside a Tauri desktop app
2. Directly accesses Zustand stores (`useChatStore`, `useCheckpointStore`)
3. Calls real application functions (`handleSend`, `handleRewind`)
4. Subscribes to state mutations in real-time
5. Produces structured pass/fail output with timing and state snapshots

This test validates the fundamental approach. Mosaic generalizes it into a tool any agent can use on any platform.

**Before Mosaic** (manual, 792 lines):
```typescript
import { useChatStore } from '@/stores/chat/chat-store';
import { useCheckpointStore } from '@/stores/agent/checkpoint-store';

// 80 lines of state recorder
// 60 lines of helper functions
// 30 lines of wait utilities
// 15 lines of snapshot logging
// ... 600+ lines of infrastructure

export async function runRewindStressTest(deps, config) {
  // The actual test is buried in here
}
```

**With Mosaic** (~50 lines):
```typescript
import { useChatStore } from '@/stores/chat/chat-store';

export default mosaic.test('rewind-stress', async (t) => {
  const handleSend = useChatStore.getState().handleSend;
  const getMessages = () => useChatStore.getState().messages;

  await t.phase('Setup', async () => {
    await t.step('Send message 1', async () => {
      handleSend('Remember 42');
      await t.waitFor(() => getMessages().length === 2);
    });
    // ... more steps
  });

  await t.phase('Rewind', async () => {
    await t.step('Rewind to msg 2', async () => {
      handleRewind(target.id);
      await t.waitFor(() => getMessages().length < prevCount);
    });
    // ... verification
  });
});
```

Same coverage. Same depth. 15x fewer lines. Agent writes test logic, Mosaic handles everything else.
