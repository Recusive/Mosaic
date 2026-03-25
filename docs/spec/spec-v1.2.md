# Mosaic v1.2 — CLI / TUI Tool Testing

**Version:** 1.0
**Date:** March 25, 2026
**Status:** Pre-development
**Depends on:** spec-full.md (architecture, API reference), spec-v1.md (core CLI, output engine)

> The agent builds a CLI tool. Mosaic proves it works.

---

## Scope

v1.2 extends Mosaic to test command-line tools and terminal UI applications. This is fundamentally different from webview and backend testing — there's no browser, no debug protocol, no module graph. The test interacts with the target tool through stdin/stdout/stderr pipes.

**In scope:**
- Process adapter for spawning and interacting with CLI tools
- Interactive session support for TUI applications (ncurses, ratatui, ink, etc.)
- stdin/stdout/stderr streaming and pattern matching
- Exit code verification
- Auto-detection of CLI tools from package.json `bin` and Cargo.toml `[[bin]]`
- API scope: Core, Process (extended with interactive sessions), Filesystem

**Out of scope:**
- Visual terminal rendering comparison (pixel-perfect TUI testing)
- Terminal emulator integration (tests use raw PTY output)

---

## How CLI/TUI Testing Works

Unlike webview targets, there's nothing to inject into. The test doesn't run inside the CLI tool — it orchestrates from outside, interacting through pipes.

```
┌─────────────────────────────┐
│  Mosaic Test Runner          │
│                             │
│  .mosaic/my-cli.test.ts     │
│  Uses: t.proc.spawn()      │
│        t.proc.interactive() │
│        t.proc.output()      │
│                             │
│  stdin ──────────────→  ┌──────────────┐
│  stdout ←────────────── │  Target CLI  │
│  stderr ←────────────── │  (my-cli)    │
│  exit code ←──────────  └──────────────┘
└─────────────────────────────┘
```

The test spawns the target CLI, sends input, reads output, asserts on results. Simple. Effective. Works for any language.

---

## Test Types

### 1. Simple Command Testing

Test that commands produce expected output and exit codes.

```typescript
// .mosaic/my-cli-basic.test.ts

export default mosaic.test('my-cli-basic', async (t) => {

  await t.phase('Commands', async () => {
    await t.step('--help shows usage', async () => {
      const proc = await t.proc.spawn('my-cli', ['--help']);
      const output = await t.proc.output(proc);
      await t.proc.waitForExit(proc);

      t.assertMatch(output, /Usage: my-cli/);
      t.assertMatch(output, /Options:/);
      t.assertEqual(await t.proc.exitCode(proc), 0);
    });

    await t.step('--version shows version', async () => {
      const proc = await t.proc.spawn('my-cli', ['--version']);
      const output = await t.proc.output(proc);

      t.assertMatch(output, /\d+\.\d+\.\d+/);
      t.assertEqual(await t.proc.exitCode(proc), 0);
    });

    await t.step('Invalid command shows error', async () => {
      const proc = await t.proc.spawn('my-cli', ['nonexistent-command']);
      const stderr = await t.proc.error(proc);
      await t.proc.waitForExit(proc);

      t.assertMatch(stderr, /unknown command/i);
      t.assert(await t.proc.exitCode(proc) !== 0, 'Should exit with non-zero');
    });
  });
});
```

### 2. Interactive CLI Testing

Test tools that accept interactive input — REPLs, wizards, prompts.

```typescript
// .mosaic/my-cli-interactive.test.ts

export default mosaic.test('my-cli-interactive', async (t) => {

  await t.phase('Interactive Wizard', async () => {
    await t.step('Project setup wizard completes', async () => {
      const session = await t.proc.interactive('my-cli', ['init']);

      // Wizard asks for project name
      await session.waitForOutput('Project name:');
      await session.type('test-project\n');

      // Wizard asks for language
      await session.waitForOutput('Language:');
      await session.type('typescript\n');

      // Wizard asks for confirmation
      await session.waitForOutput('Create project?');
      await session.type('y\n');

      // Wait for completion
      await session.waitForOutput('Project created!');
      await session.waitForExit(0);

      // Verify the wizard actually created the project
      t.assert(await t.fs.exists('./test-project/package.json'), 'Should create package.json');
      t.assert(await t.fs.exists('./test-project/tsconfig.json'), 'Should create tsconfig.json');
    });
  });

  t.cleanup(async () => {
    // Remove created project
    const proc = await t.proc.spawn('rm', ['-rf', './test-project']);
    await t.proc.waitForExit(proc);
  });
});
```

### 3. TUI Application Testing

Test terminal UI apps (ratatui, ink, blessed, etc.) that render complex layouts.

```typescript
// .mosaic/my-tui.test.ts

export default mosaic.test('my-tui', async (t) => {

  await t.phase('TUI Navigation', async () => {
    await t.step('Launches and shows main screen', async () => {
      const session = await t.proc.interactive('my-tui', [], {
        pty: true,  // Use pseudo-terminal for proper TUI rendering
        cols: 80,
        rows: 24,
      });

      // Wait for the TUI to render
      await session.waitForOutput('Dashboard');
      const screen = await session.screen();

      t.assertMatch(screen, /Dashboard/);
      t.assertMatch(screen, /\[q\] Quit/);
      t.assertMatch(screen, /\[↑↓\] Navigate/);
    });

    await t.step('Arrow keys navigate menu', async () => {
      const session = await t.proc.interactive('my-tui', [], { pty: true });
      await session.waitForOutput('Dashboard');

      // Press down arrow to move to next menu item
      await session.type('\x1b[B');  // Down arrow escape sequence
      await t.sleep(100);

      const screen = await session.screen();
      t.assertMatch(screen, /> Settings/);  // Cursor moved to Settings
    });

    await t.step('Enter selects menu item', async () => {
      const session = await t.proc.interactive('my-tui', [], { pty: true });
      await session.waitForOutput('Dashboard');

      await session.type('\x1b[B');  // Down to Settings
      await session.type('\n');       // Enter

      await session.waitForOutput('Settings Panel');
      const screen = await session.screen();
      t.assertMatch(screen, /Settings Panel/);
      t.assertMatch(screen, /Theme:/);
    });

    await t.step('q quits cleanly', async () => {
      const session = await t.proc.interactive('my-tui', [], { pty: true });
      await session.waitForOutput('Dashboard');

      await session.type('q');
      await session.waitForExit(0);
    });
  });
});
```

### 4. Pipe and File Processing Testing

Test CLI tools that process files or piped input.

```typescript
// .mosaic/my-formatter.test.ts

export default mosaic.test('my-formatter', async (t) => {

  await t.phase('File Processing', async () => {
    await t.step('Formats a file in place', async () => {
      await t.fs.write('./test-input.ts', 'const x=1;const y=2;');

      const proc = await t.proc.spawn('my-formatter', ['./test-input.ts']);
      await t.proc.waitForExit(proc);
      t.assertEqual(await t.proc.exitCode(proc), 0);

      const result = await t.fs.read('./test-input.ts');
      t.assertEqual(result, 'const x = 1;\nconst y = 2;\n');
    });

    await t.step('Reads from stdin, writes to stdout', async () => {
      const proc = await t.proc.spawn('my-formatter', ['--stdin'], {
        stdin: 'const x=1;const y=2;',
      });

      const output = await t.proc.output(proc);
      await t.proc.waitForExit(proc);

      t.assertEqual(output, 'const x = 1;\nconst y = 2;\n');
      t.assertEqual(await t.proc.exitCode(proc), 0);
    });

    await t.step('Reports errors on invalid input', async () => {
      await t.fs.write('./test-bad.ts', 'const x = {{{;');

      const proc = await t.proc.spawn('my-formatter', ['./test-bad.ts']);
      const stderr = await t.proc.error(proc);
      await t.proc.waitForExit(proc);

      t.assertMatch(stderr, /parse error/i);
      t.assert(await t.proc.exitCode(proc) !== 0);
    });
  });

  t.cleanup(async () => {
    for (const f of ['./test-input.ts', './test-bad.ts']) {
      if (await t.fs.exists(f)) {
        const proc = await t.proc.spawn('rm', [f]);
        await t.proc.waitForExit(proc);
      }
    }
  });
});
```

---

## Interactive Session API

The `t.proc.interactive()` API returns an `InteractiveSession` with PTY support:

```typescript
interface InteractiveSession {
  // Send input
  type(input: string): Promise<void>;
    // Sends raw input to stdin. Use '\n' for Enter, escape sequences for special keys.

  // Read output
  waitForOutput(pattern: string | RegExp, opts?: { timeout?: number }): Promise<string>;
    // Wait until stdout contains the pattern. Returns the full output buffer.

  screen(): Promise<string>;
    // Returns the current terminal screen content.
    // For PTY sessions, this is the rendered terminal state.
    // For non-PTY sessions, this is the accumulated stdout.

  // Lifecycle
  exitCode(): Promise<number>;
  waitForExit(expectedCode?: number): Promise<void>;
    // If expectedCode is provided, asserts it matches.
  kill(signal?: string): Promise<void>;
    // Default signal: SIGTERM
}

interface InteractiveOptions {
  pty?: boolean;          // Use pseudo-terminal (required for TUI apps)
  cols?: number;          // Terminal columns (default: 80)
  rows?: number;          // Terminal rows (default: 24)
  env?: Record<string, string>;
  cwd?: string;
  timeout?: number;       // Session timeout (default: 30s)
}
```

### Common Key Sequences

For agents writing TUI tests, reference for common escape sequences:

```
Enter:      \n
Tab:        \t
Escape:     \x1b
Backspace:  \x7f
Up:         \x1b[A
Down:       \x1b[B
Right:      \x1b[C
Left:       \x1b[D
Ctrl+C:     \x03
Ctrl+D:     \x04
Ctrl+Z:     \x1a
```

The runtime could provide helpers for these:

```typescript
// Possible future convenience API
await session.press('up');
await session.press('ctrl+c');
await session.press('enter');
```

---

## Auto-Detection for CLI Tools

```
Step 1: Identify CLI tool
  ├── package.json "bin" field → Node.js CLI
  ├── Cargo.toml [[bin]] section → Rust CLI
  ├── go.mod + main.go → Go CLI
  ├── pyproject.toml [tool.poetry.scripts] → Python CLI
  └── IDENTIFIED as CLI project

Step 2: Determine binary path
  ├── Node.js: package.json "bin" → ./node_modules/.bin/<name> or npx <name>
  ├── Rust: cargo build → target/debug/<name>
  ├── Go: go build → ./<name>
  ├── Python: poetry run <name> or python -m <name>
  └── FOUND binary

Step 3: Build if needed
  ├── Rust: cargo build (if target/debug/<name> is stale)
  ├── Go: go build (if binary is stale)
  ├── TypeScript: npm run build (if dist/ is stale)
  └── BUILT

Step 4: Run test
  ├── Spawn the binary with test arguments
  ├── No server, no protocol, no injection
  └── Pure process interaction
```

---

## Success Criteria for v1.2

1. **Simple CLI commands are testable** — spawn, send args, read stdout/stderr, check exit code.

2. **Interactive wizards are testable** — send input line by line, wait for prompts, verify flow completes.

3. **TUI applications are testable** — PTY mode, screen() captures rendered state, keyboard navigation works.

4. **File processing CLIs are testable** — pipe input via stdin, verify output, check file mutations.

5. **Cross-language** — any CLI binary works. Rust, Go, Python, Node — if it reads stdin and writes stdout, Mosaic can test it.

6. **Auto-build** — for compiled languages, Mosaic builds the binary if it's stale before testing.
