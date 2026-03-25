# Mosaic

**by Recursive Labs**

Mosaic closes the agentic loop. One CLI command, and the agent knows if its work actually works.

---

## The Problem

Every AI agent today ships blind. It writes code, says "done," and has no way to verify what it built inside the running application.

## The Fix

```
mosaic test <name>
```

Runs test scripts inside the live app. Real state. Real stores. Real backend. Zero mocks. Structured results back to the agent. Pass or iterate.

```
User prompt → Agent builds → mosaic test → Pass? → Done.
                   ↑                          |
                   └──── Fail: keep going ────┘
```

## How It Works

Mosaic tests don't run outside the application — they run **inside** it.

A Mosaic test script has direct access to:

- **Application stores** — Zustand, Redux, MobX, custom state management
- **Application functions** — internal APIs, handlers, mutations
- **File system** — read files, verify content, check existence
- **DOM** — when needed, but not limited to DOM assertions

The agent writes a test. Mosaic runs it inside the app. Results come back structured and machine-readable. The agent reads them and decides: pass, or try again.

## Who Uses It

AI agents — Claude Code, Codex, Gemini CLI, Orbit. Humans can use it. Agents should.

## What Mosaic Is Not

- **Not a mock tester** — real state, real app, real backend
- **Not an external driver** — runs inside the app, not from outside
- **Not a test generator** — the agent writes the test, Mosaic runs it
- **Not framework-locked** — desktop, web, backend (planned)

## Status

Pre-development. Closed source.

## Links

- **Website:** [mosaic.sh](https://mosaic.sh)
- **Parent company:** [Recursive Labs](https://recursive.sh)
- **Orbit:** [orbit.build](https://orbit.build)

---

&copy; Recursive Labs Inc.
