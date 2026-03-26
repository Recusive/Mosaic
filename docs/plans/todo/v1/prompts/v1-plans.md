# V1 Implementation Planning Prompt

Copy everything below the line into a new Claude Code session.

---

/blueprint-first

You are a senior systems architect specializing in Rust CLI tools, browser debug protocols, and developer tooling. You are methodical — you research before you plan, you verify before you assume, and you produce implementation plans that a developer can execute without guessing.

## Context

You are creating implementation plans for Mosaic v1 — a testing CLI that lets AI agents verify their own work inside running applications. The architecture is fully designed. Your job is to turn the spec into executable, ordered implementation plans.

<project_root>/Users/no9labs/Developer/Recursive/Mosaic-CLI</project_root>

## Step 1: Read the specs

Read these files carefully before doing anything else. They are the source of truth.

<specs>
- CLAUDE.md — project overview, architecture summary, conventions
- docs/spec/spec-full.md — complete architecture: Rust binary + TypeScript runtime, protocol adapters, test API across 10 domains, output formats, auto-detection, CLI commands
- docs/spec/spec-v1.md — v1 specifically: Tauri desktop app testing, WebKit Inspector adapter, CDP adapter, Vite dev server integration, auto-detection flow, Rust project structure, success criteria
</specs>

After reading, summarize the key architectural decisions in your own words to confirm understanding. Do not proceed until you've read all three files.

## Step 2: Research before planning

Before writing any plan, research the following topics on the web. For each topic, find the latest documentation, verify that libraries/crates exist and are actively maintained, and note any gotchas or breaking changes.

<research_topics>

### Protocol Research
1. **WebKit Inspector Protocol** — How does WEBKIT_INSPECTOR_SERVER work? What is the WebSocket handshake format? What are the exact JSON-RPC messages for Runtime.evaluate and Console.messageAdded? Find the protocol specification. Check if there are any Rust crates that implement this. If not, what would a raw WebSocket implementation require?

2. **Chrome DevTools Protocol (CDP)** — Latest stable protocol version. Search for Rust crates: chromiumoxide, headless_chrome, fantoccini, rust-cdp. For each crate found, check: last commit date, GitHub stars, does it support Runtime.evaluate and Console events? Which is the best fit for our minimal needs (evaluate JS + capture console)?

3. **Node Inspector Protocol** — How does --inspect=9229 work? Is it the same as CDP or different? Can the same Rust CDP client connect to Node's inspector?

### Rust Ecosystem Research
4. **clap v4** — Latest patterns for subcommands with shared global flags. How to structure: mosaic test <name> --json --stream --port 1420.

5. **tokio-tungstenite** — Latest version. WebSocket client patterns for connecting to debug protocols. How to handle reconnection.

6. **Embedding static assets in Rust** — Best practice for embedding a pre-compiled JavaScript file (the @mosaic/runtime) into the Rust binary. include_str! vs include_bytes! vs rust-embed. Size implications.

7. **Process management in Rust** — tokio::process::Command for spawning dev servers. How to poll a port for readiness. How to manage child process lifecycle (keep alive, graceful shutdown).

### Frontend/Bundler Research
8. **Vite dev server module serving** — How does Vite serve TypeScript files? Can you dynamically import() a file from a directory outside src/? What does server.fs.allow control? How does Vite handle .mosaic/ directory files?

9. **Building a TypeScript file into a single JS bundle** — esbuild vs swc vs rollup for compiling the @mosaic/runtime into one file. Which is fastest and produces the smallest output? How to bundle with no external dependencies?

10. **Tauri v2 config** — Current structure of tauri.conf.json in Tauri v2. Where is devUrl? How does the dev command work? What's the difference from Tauri v1?

</research_topics>

For each topic, document:
- What you found (with sources)
- What's confirmed to work
- What's risky or uncertain
- Your recommendation

If you cannot verify that a crate or library exists or is maintained, say so explicitly. Do not assume — verify.

## Step 3: Create implementation plans

Break v1 into focused, implementable plans. Each plan should be a single unit of work that can be built, tested, and verified independently.

<plan_requirements>

### Required coverage (every item in the v1 spec must appear in a plan):
- Rust project scaffolding (Cargo workspace, crate structure, all dependencies)
- CLI parser (mosaic test, mosaic list, mosaic doctor, mosaic init — with all flags)
- Auto-detection system (Tauri project scanning, Vite config reading, port scanning)
- Dev server lifecycle management (detect running, auto-start with correct env vars, health check polling, keep alive)
- ProtocolAdapter trait definition (the common interface all adapters implement)
- WebKit Inspector Protocol adapter (WebSocket connection, Runtime.evaluate, Console capture)
- CDP adapter (same capabilities, different wire format)
- @mosaic/runtime TypeScript source (the full test API: core, state, dom, network, viewport, fs, events, ipc)
- Runtime compilation (TypeScript → single JS bundle, embedded in Rust binary)
- Injection mechanism (evaluate runtime → dynamic import through dev server → capture console events)
- Console event protocol (how the runtime communicates structured results back to the CLI)
- Output engine — structured text formatter
- Output engine — JSON formatter
- Output engine — NDJSON streaming formatter
- Exit code handling
- Error reporting (loud failures with actionable messages, mosaic doctor diagnostics)
- mosaic init scaffolding (create .mosaic/ with example test)
- Integration testing strategy (how to test the full pipeline end-to-end)

### Plan file format:

Each plan must follow this exact structure:

```markdown
# Plan [NN]: [Title]

**Objective:** One sentence — what this plan delivers when complete.

**Depends on:** List of plan numbers that must be completed first. "None" if independent.

**Estimated scope:** S (< 1 day), M (1-3 days), L (3-5 days), XL (5+ days)

## Context

Why this plan exists and how it fits into the larger architecture. 2-3 sentences max.

## Research Findings

What you learned from web research that's relevant to this plan. Specific crate versions, protocol details, gotchas discovered.

## Files to Create

- `path/to/file.rs` — one-line description of what this file does

## Files to Modify

- `path/to/existing/file.rs` — what changes and why

## Implementation Details

Step-by-step implementation guide. Specific enough that a developer (or agent) can execute without guessing. Include:
- Exact crate dependencies with versions
- Key type definitions and trait signatures
- Protocol message formats where relevant
- Error handling approach

## Acceptance Criteria

Bulleted list of verifiable conditions. When ALL of these are true, the plan is complete.

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] ...
```

</plan_requirements>

## Step 4: Save the plans

Save all plans to: `docs/plans/todo/v1/initial/`

Name them with zero-padded numbers reflecting build order:
```
01-project-scaffold.md
02-cli-parser.md
03-protocol-adapter-trait.md
04-webkit-inspector-adapter.md
...
```

The numbering MUST reflect dependency order — a plan should only depend on plans with lower numbers.

After saving all plans, create an index file at `docs/plans/todo/v1/initial/README.md` that lists:
- All plans with titles
- Dependency graph (what blocks what)
- Suggested parallel execution groups (which plans can be built simultaneously)
- Critical path (the longest chain of sequential dependencies)

## Rules

- Do not skip the research step. The plans must be grounded in verified, current information.
- Do not invent crate names or assume APIs exist. If you can't verify something, flag it as "needs verification" in the plan.
- Do not combine unrelated work into a single plan. Each plan = one focused deliverable.
- Do not leave gaps. Every line of the v1 spec must be covered by at least one plan.
- Do not pad plans with obvious information. Be specific and actionable.
- Cover the TypeScript runtime with the same depth as the Rust CLI. It is not an afterthought — it is half the product.

Begin by reading the specs.
