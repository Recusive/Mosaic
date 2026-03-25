# What Is Mosaic

**by Recursive Labs**

Version 1.0 | March 2026 | Internal — Product Definition

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

**Who uses it:** AI agents — Claude Code, Codex, Gemini CLI, Orbit. Humans can use it. Agents should.

**Status:** Pre-development. Waitlist open at mosaic.sh.

**Source:** Closed. May open in the future.

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

### Script Injection — Testing From Inside

Mosaic tests don't run outside the application. They run inside it.

A Mosaic test script has direct access to:

- **Application stores** — Zustand, Redux, MobX, custom state management
- **Application functions** — `handleSend()`, `handleRewind()`, internal APIs
- **File system** — read files, verify content, check existence
- **DOM** — when needed, but Mosaic is not limited to DOM assertions

This is the fundamental difference. Playwright clicks a button and checks if the DOM changed. Mosaic calls the function behind the button, verifies the store updated, confirms the file system reflects the change, and checks the DOM — all from inside the application. Same runtime. Same process. Same state.

### What a Test Looks Like

Based on real stress tests used internally on Orbit:

```
Phase 1 — Warmup:
  Step 1: Send 5 messages → verify message count in store       ✓ 12.4s
  Step 2: Create file via agent → verify file exists            ✓  8.7s
  Step 3: Edit file via agent → verify content changed          ✓  9.1s

Phase 2 — Rewind Gauntlet:
  Step 4: Rewind to message 8 → verify file reverted            ✓ 15.2s
  Step 5: Rewind to message 7 → verify file state               ✓ 14.8s
  Step 6: Rewind past file creation → verify file gone           ✗ TIMEOUT

Result: 5/6 passed, 1 failed
Duration: 143.6s
```

Every step produces structured output: step name, phase, duration in milliseconds, pass/fail, error message, state snapshots. Machine-readable. Agent-consumable.

---

## Part 5: Testing by Platform

Mosaic is not one trick. Different applications need different injection strategies. Mosaic adapts.

### Desktop Applications

**Status:** Proven internally.

Script injection into the application's runtime. Test scripts import directly from the app's stores and modules, call internal functions, verify state from inside the process.

Tests can access: state management stores, internal functions and APIs, file system operations, process state, any runtime-accessible data.

**Evidence:** The Orbit rewind mega stress test — 18 steps across 3 phases. Sends messages, creates files, edits files, rewinds conversation state, verifies file system integrity at every checkpoint. All from inside the running app.

### Web and Browser Applications

**Status:** Planned. Architecture defined, implementation TBD.

Rust-built automation layer with direct DOM access. Mosaic drives a browser session, interacts with the DOM, and accesses application state — combining browser automation reach with internal state depth.

Planned: headless browsing for CI/CD, direct DOM interaction, application state access, network request interception, visual regression.

### Backend Applications

**Status:** Planned. Not yet designed.

Backend testing is confirmed scope. The approach has not been finalized.

---

## Part 6: CLI

```
mosaic test <name>
```

One command. The agent runs it. Results come back. The loop closes.

**Exact syntax, flags, and configuration format are TBD.** The confirmed design intent:

- **Standalone binary** — no runtime dependency, no package manager required
- **Machine-readable output** — structured logs, exit codes
- **Streamable** — real-time tailing for agents that want to watch
- **Agent-first** — no interactive prompts, no TUI, no human-centric formatting unless requested

### Any Agent Can Use It

Mosaic doesn't care which agent invokes it. If it can run a CLI command and read the output, it can use Mosaic.

- **Claude Code** — Bash tool or MCP integration
- **Codex** — shell execution
- **Gemini CLI** — shell execution
- **Orbit** — native integration via terminal and agent tool system

---

## Part 7: What Mosaic Is Not

**Not a mock tester.** Real state. Real app. Real backend. If the test passes in Mosaic, it works in the app. No fake anything.

**Not an external driver.** Mosaic doesn't click through the DOM from outside like a robot pretending to be a user. It runs inside the app with access to the same internals the app uses.

**Not a test generator.** The agent writes the test. Mosaic runs it. The agent is the brain. Mosaic is the hands.

**Not a human-first tool.** Humans can use it. But structured output, CLI interface, machine-readable logs — every decision optimizes for the agent.

**Not framework-locked.** Not React-only. Not Zustand-only. Not web-only. Desktop apps, web apps, backend (planned). The injection strategy adapts to the platform.

**Not a replacement for unit tests.** Unit tests verify isolated functions. Mosaic verifies the whole system works from inside the running application. Different jobs. They coexist.

---

## Part 8: Current Status

| Item | Status |
|------|--------|
| Product concept | Defined |
| Architecture | Proven for desktop, planned for web and backend |
| CLI interface | `mosaic test <name>` — exact syntax TBD |
| Implementation language | Likely Rust — TBD |
| Distribution | Standalone binary — TBD |
| Source | Closed (may open in the future) |
| Product code | Pre-development |
| Marketing site | Live at mosaic.sh |
| Waitlist | Open |
| Internal validation | Stress tests on Orbit confirm the script injection approach |

### TBD

Confirmed scope, not yet finalized:

- Implementation language (likely Rust)
- Distribution and installation
- Full CLI syntax, flags, and options
- Configuration format
- Web app automation specifics
- Backend testing approach
- Headless browsing implementation
- Test file format and conventions
- Framework/store support matrix

---

## Part 9: Content Rules

For anyone creating website pages, blog posts, or documentation from this document.

### Say This

- Mosaic is built by Recursive Labs
- Mosaic is a standalone CLI testing tool built for AI agents
- Mosaic closes the agentic loop
- Mosaic runs tests inside running applications — not from outside
- Mosaic accesses real application state: stores, functions, file system
- Mosaic produces structured, machine-readable test results
- Desktop app testing is proven internally
- Web and backend testing are planned
- Any agent with CLI access can use Mosaic
- Mosaic is invoked via `mosaic test <name>`
- Mosaic is currently in development
- Mosaic is closed source

### Don't Say This

- Don't call it open source — it's not
- Don't claim web or backend testing works today — it's planned
- Don't cite performance numbers — none exist
- Don't claim users — there are none, the product is pre-development
- Don't say Mosaic generates tests — the agent generates tests, Mosaic runs them
- Don't say Mosaic replaces unit testing — it complements it
- Don't state the implementation language as decided — it's likely Rust, not confirmed
- Don't reference framework support beyond what's been tested (Zustand on Orbit)
- Don't fabricate testimonials, logos, user counts, or social proof

---

## Changelog

### March 2026

- v1.0 — Initial product definition. All claims verified against internal stress tests and founder input.
