<p align="center">
  <img src="assets/logo.png" width="120" alt="Mosaic" />
</p>

<h1 align="center">Mosaic</h1>

<p align="center">
  <strong>The testing CLI built for AI agents.</strong><br/>
  One command. Real app. Real state. The agent knows if it works.
</p>

<p align="center">
  <a href="https://mosaic.sh">Website</a> &middot;
  <a href="#quick-start">Quick Start</a> &middot;
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
Phase 1 — Layout:
  Step 1: Set viewport to 1200px → check sidebar visible          ✓  1.2s
  Step 2: Set viewport to 800px  → check sidebar collapsed        ✓  0.8s
  Step 3: Set viewport to 600px  → check sidebar hidden           ✓  0.9s

Result: 3/3 passed
Duration: 2.9s
```

The agent reads the output. All green? Done. Something failed? Fix and rerun.

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
| **State management** | Zustand, Redux, MobX, custom stores |
| **Internal functions** | `handleSend()`, `processQueue()`, any export |
| **File system** | Read, write, verify file contents |
| **DOM** | When you need it — but you're not limited to it |

Playwright clicks a button and checks if the DOM changed. **Mosaic calls the function behind the button**, verifies the store updated, confirms the file system reflects the change, *and* checks the DOM. Same runtime. Same process. Same state.

## Agent Compatible

Mosaic doesn't care which agent runs it. If it can execute a CLI command and read stdout, it works.

| Agent | Integration |
|-------|-------------|
| **Claude Code** | Bash tool / MCP |
| **Codex** | Shell execution |
| **Gemini CLI** | Shell execution |
| **Orbit** | Native integration |

Output is structured, machine-readable, and streamable. No interactive prompts. No TUI. Built for the agent.

## Platforms

| Platform | Status |
|----------|--------|
| Desktop apps | Proven internally |
| Web / Browser | Planned |
| Backend | Planned |

## What Mosaic Is Not

| | |
|-|-|
| **Not a mock tester** | Real state. Real app. If it passes in Mosaic, it works in prod. |
| **Not an E2E driver** | Runs inside the app — not a robot clicking from outside. |
| **Not a test generator** | The agent writes the test. Mosaic runs it. |
| **Not framework-locked** | Desktop, web, backend. The injection strategy adapts. |
| **Not replacing unit tests** | Different jobs. They coexist. |

## Contributing

Mosaic is open source and contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

```bash
git clone https://github.com/Recusive/Mosaic.git
cd Mosaic
```

## Community

- [GitHub Issues](https://github.com/Recusive/Mosaic/issues) — bugs and feature requests
- [mosaic.sh](https://mosaic.sh) — waitlist and updates

## License

MIT &copy; [Recursive Labs](https://recursive.sh)
