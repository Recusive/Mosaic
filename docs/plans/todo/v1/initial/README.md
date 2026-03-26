# Mosaic v1 — Implementation Plans

**Total plans:** 17
**Estimated total scope:** ~5-8 weeks of focused development
**Generated:** March 25, 2026

---

## Plan Index

| # | Title | Scope | Depends On |
|---|-------|-------|------------|
| 01 | [Project Scaffold & Dependencies](01-project-scaffold.md) | S | None |
| 02 | [CLI Parser](02-cli-parser.md) | S | 01 |
| 03 | [Console Event Protocol](03-console-event-protocol.md) | S | 01 |
| 04 | [Auto-Detection System](04-auto-detection.md) | M | 01 |
| 05 | [Dev Server Lifecycle](05-dev-server-lifecycle.md) | M | 01, 04 |
| 06 | [Protocol Transport Layer & Adapter Trait](06-protocol-transport.md) | M | 01 |
| 07 | [WebKit Inspector Adapter](07-webkit-inspector-adapter.md) | M | 06 |
| 07b | [Tauri Plugin Adapter](07b-tauri-plugin-adapter.md) | M | 06 |
| 08 | [CDP Adapter](08-cdp-adapter.md) | M | 06 |
| 09 | [@mosaic/runtime — Core API](09-runtime-core-api.md) | L | 03 |
| 10 | [@mosaic/runtime — Domain APIs](10-runtime-domain-apis.md) | L | 09 |
| 11 | [Runtime Build Pipeline](11-runtime-build-pipeline.md) | S | 09 |
| 12 | [Output Engine](12-output-engine.md) | M | 03 |
| 13 | [Test Injection & Execution Engine](13-test-injection-engine.md) | M | 06, 03, 11 |
| 14 | [Error Reporting & mosaic doctor](14-error-reporting.md) | M | 02, 04 |
| 15 | [mosaic init & mosaic list](15-init-and-list.md) | S | 02 |
| 16 | [Test Runner Orchestrator](16-test-runner-orchestrator.md) | L | 02, 04, 05, 07b, 12, 13, 14 |
| 17 | [Integration Testing Strategy](17-integration-testing.md) | L | 16 |

---

## Dependency Graph

```
01 Project Scaffold
├── 02 CLI Parser
│   ├── 14 Error Reporting & Doctor
│   ├── 15 Init & List
│   └── 16 Test Runner Orchestrator ←──────────────────────┐
├── 03 Console Event Protocol                               │
│   ├── 09 Runtime Core API                                 │
│   │   ├── 10 Runtime Domain APIs                          │
│   │   └── 11 Runtime Build Pipeline                       │
│   │       └── 13 Test Injection Engine ──────────────────→│
│   └── 12 Output Engine ─────────────────────────────────→│
├── 04 Auto-Detection                                       │
│   ├── 05 Dev Server Lifecycle ──────────────────────────→│
│   └── 14 Error Reporting ───────────────────────────────→│
└── 06 Protocol Transport                                   │
    ├── 07 WebKit Adapter (Linux) ────────────────────────→│
    ├── 07b Tauri Plugin Adapter (PRIMARY for Tauri) ─────→│
    ├── 08 CDP Adapter (Windows/Electron) ────────────────→│
    └── 13 Test Injection Engine ─────────────────────────→│
                                                            │
                                               16 Orchestrator
                                                    │
                                               17 Integration Testing
```

---

## Parallel Execution Groups

These groups can be built simultaneously by different developers or agents:

### Group A: Rust Infrastructure
```
01 → 02 → 15
01 → 04 → 05
01 → 04 → 14
```

### Group B: Protocol Layer
```
01 → 06 → 07
01 → 06 → 08
```

### Group C: TypeScript Runtime
```
03 → 09 → 10
03 → 09 → 11
```

### Group D: Output & Events
```
03 → 12
```

### Convergence Point
```
Groups A + B + C + D → 13 → 16 → 17
```

**Maximum parallelism:** After Plan 01, up to 5 plans can proceed simultaneously (02, 03, 04, 06, and parts of 09 if the console event format is agreed upfront).

---

## Critical Path

The longest chain of sequential dependencies determines the minimum build time:

```
01 → 03 → 09 → 10 → 11 → 13 → 16 → 17
 S     S     L     L     S     M     L     L
```

**Critical path duration:** S + S + L + L + S + M + L + L ≈ 3-4 weeks

Plans 10 (domain APIs) and 09 (core API) are the widest items on the critical path. Splitting them or staffing them heavily would reduce total time.

---

## Known Risks

### RESOLVED: macOS WKWebView Debugging

Research confirmed that `WEBKIT_INSPECTOR_SERVER` does NOT work on macOS. Apple's WKWebView uses private XPC APIs — no WebSocket debug protocol.

**Solution: `tauri-plugin-mosaic` (Plan 07b).** A Tauri Rust plugin that uses Tauri's native `Webview::eval()` and `initialization_script()` APIs — both call public Apple APIs (`WKWebView.evaluateJavaScript()` and `WKUserScript`) that work on all platforms. The plugin communicates with the Mosaic CLI via a secured local WebSocket (UUID token, origin validation, single-client mode).

- **Plan 07b (Tauri Plugin Adapter)** is the **PRIMARY** adapter for all Tauri testing (macOS, Linux, Windows).
- **Plan 07 (WebKit Adapter)** remains for non-Tauri Linux WebKitGTK targets.
- **Plan 08 (CDP Adapter)** remains for Electron, Chrome, and non-Tauri Windows targets.
- **One-time setup:** `mosaic init --plugin` adds the Tauri plugin (1 Cargo dep + 1 line). Feature-gated — compiles out of production builds.

### MEDIUM: Vite server.fs.allow

If a Tauri project has a custom Vite `root` or `server.fs.allow` configuration, Vite may refuse to serve `.mosaic/` test files with a 403. Mosaic detects this and reports the fix (Plan 13, Plan 14). Default Vite configs work without changes.

### LOW: Tauri v1 vs v2 Config Differences

`devPath` (v1) vs `devUrl` (v2) is handled by serde aliases in Plan 04. Tested with fixtures in Plan 17.

---

## Spec Coverage Verification

Every item from the v1 spec maps to at least one plan:

| Spec Item | Plan(s) |
|-----------|---------|
| Rust project scaffolding | 01 |
| CLI parser (all commands and flags) | 02 |
| Auto-detection (Tauri, Vite, ports) | 04 |
| Dev server lifecycle | 05 |
| ProtocolAdapter trait | 06 |
| Tauri Plugin adapter (PRIMARY for Tauri) | 07b |
| WebKit Inspector adapter (Linux non-Tauri) | 07 |
| CDP adapter (Windows/Electron) | 08 |
| @mosaic/runtime TypeScript (full API) | 09, 10 |
| Runtime compilation (TS → JS → embedded) | 11 |
| Injection mechanism | 13 |
| Console event protocol | 03 |
| Output: structured text | 12 |
| Output: JSON | 12 |
| Output: NDJSON streaming | 12 |
| Exit code handling | 12, 16 |
| Error reporting (loud failures) | 14 |
| mosaic doctor | 14 |
| mosaic init | 15 |
| mosaic list | 15 |
| Integration testing | 17 |
