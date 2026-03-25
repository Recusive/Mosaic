# Mosaic v1 — Tauri Desktop App Testing

**Version:** 1.0
**Date:** March 25, 2026
**Status:** Pre-development
**Depends on:** spec-full.md (architecture, API reference)

> The first platform. Prove the loop on the app that inspired Mosaic.

---

## Scope

v1 delivers end-to-end testing for Tauri desktop applications. This is where the proof of concept lives — Mosaic was born from testing Orbit, a Tauri app. v1 makes that process repeatable for any Tauri project.

**In scope:**
- WebKit Inspector Protocol adapter (macOS/Linux)
- CDP adapter for Tauri on Windows (WebView2)
- Vite dev server integration (module graph sharing)
- Auto-detection of Tauri projects
- Auto-start of `cargo tauri dev` with inspector flags
- Full test API: Core, State, DOM, Events, IPC, Filesystem, Viewport, Network
- Structured text + JSON + streaming output
- CLI: `mosaic test`, `mosaic list`, `mosaic doctor`, `mosaic init`

**Out of scope (later versions):**
- Backend/API testing (v1.1)
- CLI/TUI testing (v1.2)
- Browser webapp testing (v1.3)

---

## Architecture for Tauri

```
┌──────────────────────────────────────────────────┐
│  mosaic test rewind-stress                        │
│                                                  │
│  1. Scan project → find tauri.conf.json          │
│  2. Read devUrl → http://localhost:1420           │
│  3. Check dev server → not running               │
│  4. Start: WEBKIT_INSPECTOR_SERVER=              │
│     127.0.0.1:9222 cargo tauri dev               │
│  5. Wait for dev server ready on :1420           │
│  6. Connect WebKit Inspector on :9222            │
│  7. Inject @mosaic/runtime.js                    │
│  8. Import test via dev server:                  │
│     import('http://localhost:1420/               │
│            .mosaic/rewind-stress.test.ts')        │
│  9. Stream console events → format output        │
│  10. Report results → exit code                  │
└──────────────────────────────────────────────────┘
```

### Tauri Project Detection

Mosaic identifies a Tauri project by the presence of `tauri.conf.json` (Tauri v1) or `src-tauri/tauri.conf.json` (common layout).

**Fields read from `tauri.conf.json`:**

```json
{
  "build": {
    "devUrl": "http://localhost:1420",
    "beforeDevCommand": "npm run dev"
  }
}
```

- `build.devUrl` — the dev server URL. Mosaic connects here.
- `build.beforeDevCommand` — confirms a dev server is part of the workflow.

For Tauri v2, the config may be `tauri.conf.json` or `Tauri.toml`. Mosaic checks both.

### WebKit Inspector Connection (macOS/Linux)

Tauri on macOS and Linux uses WKWebView (WebKit). To enable remote debugging:

```
WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222 cargo tauri dev
```

This opens a WebSocket server at `ws://127.0.0.1:9222` that speaks the WebKit Inspector Protocol.

**Connection sequence:**

1. HTTP GET `http://127.0.0.1:9222/json` — lists inspectable targets
2. Each target has a `webSocketDebuggerUrl`
3. Mosaic connects to the WebSocket for the main page target
4. Sends `Runtime.enable` to start receiving console events
5. Ready to evaluate JavaScript and capture output

**Key protocol messages used:**

```
→ {"id":1,"method":"Runtime.enable"}
← {"id":1,"result":{}}

→ {"id":2,"method":"Runtime.evaluate","params":{"expression":"..."}}
← {"id":2,"result":{"result":{"type":"string","value":"..."}}}

← {"method":"Console.messageAdded","params":{"message":{"text":"...","level":"warning"}}}
```

### CDP Connection (Windows)

Tauri on Windows uses WebView2 (Chromium). The debug protocol is CDP, same as Electron and Chrome.

```
cargo tauri dev  (WebView2 exposes CDP automatically in dev mode)
```

Connection is the same concept — WebSocket to the debug endpoint — but with CDP message format.

### Vite Dev Server Integration

Most Tauri projects use Vite for the frontend. This is critical for Mosaic because the dev server handles module resolution.

**How it works:**

1. Test file lives at `.mosaic/rewind-stress.test.ts`
2. Mosaic injects: `await import('http://localhost:1420/.mosaic/rewind-stress.test.ts')`
3. Vite receives the request, sees a `.ts` file, compiles it
4. Vite resolves all imports in the test against the app's module graph:
   - `@/stores/chat/chat-store` → `src/stores/chat/chat-store.ts` (via tsconfig paths)
   - `zustand` → `node_modules/zustand` (same instance the app uses)
5. Returns compiled JavaScript with shared module instances
6. Test runs with access to the SAME stores and functions as the app

**Vite configuration requirement:**

Vite must be able to serve files from `.mosaic/`. By default, Vite serves from the project root, so `.mosaic/` is accessible. If the project has a custom `root` config or `server.fs.allow` restrictions, those may need adjustment.

Mosaic should detect this and report it:

```
[mosaic] ✗ Vite refused to serve .mosaic/rewind-stress.test.ts
[mosaic]   Your Vite config may restrict file serving.
[mosaic]   Add to vite.config.ts:
[mosaic]     server: { fs: { allow: ['.mosaic'] } }
```

---

## Auto-Detection Flow for Tauri

```
Step 1: Find tauri.conf.json
  ├── Check ./tauri.conf.json
  ├── Check ./src-tauri/tauri.conf.json
  └── NOT FOUND → this isn't a Tauri project, skip to next platform detector

Step 2: Read devUrl
  ├── Parse tauri.conf.json → build.devUrl
  ├── Found "http://localhost:1420" → use port 1420
  └── NOT FOUND → default to 1420

Step 3: Check dev server
  ├── HTTP GET http://localhost:1420
  ├── RESPONDING → proceed to Step 4
  └── NOT RESPONDING → go to Step 3a

Step 3a: Start dev server
  ├── Detect OS: macOS/Linux → set WEBKIT_INSPECTOR_SERVER=127.0.0.1:9222
  ├── Run: cargo tauri dev (with env var)
  ├── Poll http://localhost:1420 every 500ms
  ├── READY within 120s → proceed to Step 4
  └── TIMEOUT → report error with full diagnostics

Step 4: Connect debug protocol
  ├── Detect OS: macOS/Linux → WebKit Inspector on 9222
  ├── Detect OS: Windows → CDP on 9222
  ├── WebSocket connect → ws://127.0.0.1:9222
  ├── CONNECTED → proceed to test injection
  └── FAILED → report error: inspector not available
```

---

## Test API — Tauri-Specific Details

### IPC (Tauri Commands)

Tauri's `invoke` system calls Rust backend commands from the frontend. Mosaic's `t.ipc` wraps this.

```typescript
// In the Tauri app, a command might be:
// #[tauri::command]
// fn get_user(id: String) -> Result<User, String> { ... }

await t.step('Backend returns user', async () => {
  const user = await t.ipc.invoke('get_user', { id: 'abc123' });
  t.assertEqual(user.name, 'Test User');
});
```

Under the hood, `t.ipc.invoke` calls `window.__TAURI__.invoke` (Tauri v1) or `window.__TAURI_INTERNALS__.invoke` (Tauri v2).

### Tauri Events

Tauri has a cross-process event system. `t.events` maps to it:

```typescript
await t.step('Backend emits event', async () => {
  // Trigger something that makes the backend emit an event
  await t.ipc.invoke('start_processing', { file: 'test.txt' });

  // Wait for the backend to emit the completion event
  const result = await t.events.waitFor('processing-complete', { timeout: 10_000 });
  t.assertEqual(result.status, 'success');
});
```

### Filesystem via Tauri

For Tauri apps, filesystem operations go through Tauri's fs plugin. `t.fs` can either:
- Use Tauri's fs plugin (`window.__TAURI__.fs`) for sandboxed access
- Use the debug protocol to evaluate Node.js-style fs operations (if applicable)

The runtime detects which is available and uses the appropriate method.

---

## Example: Full Tauri Test

The rewind stress test rewritten for Mosaic:

```typescript
// .mosaic/rewind-stress.test.ts

import { useChatStore } from '@/stores/chat/chat-store';
import { useCheckpointStore } from '@/stores/agent/checkpoint-store';

export default mosaic.test('rewind-stress', async (t) => {
  const handleSend = useChatStore.getState().handleSend;
  const handleRewind = useChatStore.getState().handleRewind;
  const getMessages = () => {
    const state = useChatStore.getState();
    const sid = state.activeSessionId;
    return sid ? (state.sessions[sid]?.messages ?? []) : [];
  };

  t.cleanup(async () => {
    // Restore clean state after test
    const state = useChatStore.getState();
    if (state.activeSessionId) {
      state.clearSession(state.activeSessionId);
    }
  });

  // ── Phase 1: Send messages to build conversation ──

  await t.phase('Setup', async () => {
    await t.step('Send message 1', async () => {
      handleSend('Remember the number 42. Reply with "Got it, 42." and nothing else.');
      await t.waitFor(() => {
        const msgs = getMessages();
        const last = msgs.at(-1);
        return msgs.length >= 2 && last?.role === 'assistant' && !last.isStreaming;
      }, { timeout: 60_000 });

      t.snapshot('after-msg1', {
        messageCount: getMessages().length,
        lastMessage: getMessages().at(-1)?.content.slice(0, 50),
      });
    });

    await t.step('Send message 2', async () => {
      handleSend('Remember the number 73. Reply with "Got it, 73." and nothing else.');
      await t.waitFor(() => {
        const msgs = getMessages();
        const last = msgs.at(-1);
        return msgs.length >= 4 && last?.role === 'assistant' && !last.isStreaming;
      }, { timeout: 60_000 });
    });

    await t.step('Send message 3', async () => {
      handleSend('Remember the number 99. Reply with "Got it, 99." and nothing else.');
      await t.waitFor(() => {
        const msgs = getMessages();
        const last = msgs.at(-1);
        return msgs.length >= 6 && last?.role === 'assistant' && !last.isStreaming;
      }, { timeout: 60_000 });
    });
  });

  // ── Phase 2: Rewind and verify state integrity ──

  await t.phase('Rewind', async () => {
    await t.step('Rewind to message 2 response', async () => {
      const assistantMessages = getMessages().filter(m => m.role === 'assistant');
      const target = assistantMessages[1]; // 2nd assistant message (response to msg 2)
      t.assert(target !== undefined, 'Should have at least 2 assistant messages');
      t.ctx.set('rewindTargetId', target.id);

      const preCount = getMessages().length;
      handleRewind(target.id);

      await t.waitFor(() => getMessages().length < preCount, { timeout: 30_000 });

      t.snapshot('after-rewind', {
        messageCount: getMessages().length,
        rewindEpoch: useChatStore.getState().rewindEpoch,
      });
    });

    await t.step('Send post-rewind message', async () => {
      handleSend('What numbers do you remember? Reply with just the numbers, comma-separated.');

      await t.waitFor(() => {
        const msgs = getMessages();
        const last = msgs.at(-1);
        return last?.role === 'assistant' && !last.isStreaming;
      }, { timeout: 60_000 });

      const response = getMessages().at(-1)?.content ?? '';
      t.log('Agent response: ' + response);

      t.assert(response.includes('42'), 'Should remember 42');
      t.assert(response.includes('73'), 'Should remember 73');
      t.assert(!response.includes('99'), 'Should NOT remember 99 (it was after the rewind point)');
    });

    await t.step('Second rewind to message 2 response', async () => {
      const assistantMessages = getMessages().filter(m => m.role === 'assistant');
      const target = assistantMessages[1];
      t.assert(target !== undefined, 'Should still have 2nd assistant message');

      const preCount = getMessages().length;
      handleRewind(target.id);

      await t.waitFor(() => getMessages().length < preCount, { timeout: 30_000 });
    });

    await t.step('Verify context after double rewind', async () => {
      handleSend('How many messages have I sent in this chat? Count every user message you can see.');

      await t.waitFor(() => {
        const msgs = getMessages();
        const last = msgs.at(-1);
        return last?.role === 'assistant' && !last.isStreaming;
      }, { timeout: 60_000 });

      const response = getMessages().at(-1)?.content ?? '';
      t.log('Context verification response: ' + response);
      // After rewinding to msg 2 response, user should see:
      // msg 1 ("Remember 42"), msg 2 ("Remember 73"), and the "how many" question = 3
    });
  });
});
```

**Expected output:**

```
[mosaic] rewind-stress
[mosaic] Tauri → WebKit Inspector → localhost:1420
[mosaic] ──────────────────────────────────────────

Phase 1 — Setup:
  ✓ Send message 1                                   12.4s
  ✓ Send message 2                                    8.7s
  ✓ Send message 3                                    9.1s

Phase 2 — Rewind:
  ✓ Rewind to message 2 response                      0.4s
  ✓ Send post-rewind message                          15.2s
    Agent response: 42, 73
  ✓ Second rewind to message 2 response                0.3s
  ✓ Verify context after double rewind                14.8s
    Context verification response: You've sent 3 messages...

[mosaic] ──────────────────────────────────────────
[mosaic] 7 passed · 0 failed · 0 skipped · 60.9s
[mosaic] Exit 0
```

---

## Rust Implementation — Key Crates

| Crate | Purpose |
|-------|---------|
| `clap` | CLI argument parsing |
| `tokio` | Async runtime |
| `tokio-tungstenite` | WebSocket client for protocol connections |
| `serde` / `serde_json` | JSON parsing (protocol messages, test output) |
| `reqwest` | HTTP client (dev server health checks) |
| `notify` | File watching (future: `--watch` mode) |
| `colored` | Terminal output formatting |
| `include_str!` / `include_bytes!` | Embed @mosaic/runtime.js in binary |

### Project Structure

```
mosaic/
├── Cargo.toml
├── src/
│   ├── main.rs                    # CLI entry point
│   ├── cli/
│   │   ├── mod.rs
│   │   ├── test_cmd.rs            # mosaic test
│   │   ├── list_cmd.rs            # mosaic list
│   │   ├── doctor_cmd.rs          # mosaic doctor
│   │   └── init_cmd.rs            # mosaic init
│   ├── detect/
│   │   ├── mod.rs                 # Auto-detection orchestrator
│   │   ├── tauri.rs               # Tauri project detection
│   │   ├── electron.rs            # Electron detection (stub for v1)
│   │   ├── vite.rs                # Vite config reading
│   │   └── port_scanner.rs        # Scan for running dev servers
│   ├── protocol/
│   │   ├── mod.rs                 # ProtocolAdapter trait
│   │   ├── webkit.rs              # WebKit Inspector Protocol
│   │   ├── cdp.rs                 # Chrome DevTools Protocol
│   │   ├── node.rs                # Node Inspector (stub for v1.1)
│   │   └── process.rs             # Process adapter (stub for v1.2)
│   ├── runtime/
│   │   ├── mod.rs
│   │   └── mosaic_runtime.js      # Embedded TypeScript runtime (pre-compiled)
│   ├── output/
│   │   ├── mod.rs
│   │   ├── text.rs                # Structured text formatter
│   │   ├── json.rs                # JSON formatter
│   │   └── stream.rs              # NDJSON streaming formatter
│   └── server/
│       ├── mod.rs
│       └── dev_server.rs          # Dev server management (start/stop/health)
├── runtime/                       # TypeScript source for @mosaic/runtime
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── index.ts               # Runtime entry point
│   │   ├── api/
│   │   │   ├── core.ts            # phase, step, assert, waitFor...
│   │   │   ├── dom.ts             # DOM interaction
│   │   │   ├── state.ts           # Store access
│   │   │   ├── network.ts         # Network interception
│   │   │   ├── viewport.ts        # Viewport control
│   │   │   ├── filesystem.ts      # File operations
│   │   │   ├── events.ts          # Event system
│   │   │   └── ipc.ts             # IPC (Tauri/Electron)
│   │   └── emitter.ts             # Console event emitter (structured output)
│   └── build.ts                   # Builds runtime → single JS file
└── tests/                         # Mosaic's own tests
    ├── detect_tauri_test.rs
    ├── webkit_protocol_test.rs
    └── output_format_test.rs
```

---

## Success Criteria for v1

1. **`mosaic test` runs against a Tauri app on macOS** — auto-detects project, starts dev server, connects via WebKit Inspector, injects test, streams results.

2. **The Orbit rewind stress test works** — the 792-line manual test is rewritten as a ~50-line Mosaic test and produces equivalent results.

3. **An AI agent can use it in a loop** — agent writes code, runs `mosaic test`, reads output, fixes, repeats. The loop closes without human intervention.

4. **Failure output is actionable** — when a test fails, the output tells the agent exactly what went wrong (assertion error, expected vs actual, state snapshot) so it can fix the issue without guessing.

5. **Auto-detection works for standard Tauri projects** — `cargo tauri init` project structure is detected without configuration.

6. **Manual override works** — when auto-detection doesn't fit, flags override every setting.
