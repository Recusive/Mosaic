# Plan 09: @mosaic/runtime — Core API

**Objective:** Implement the TypeScript runtime's core test API — `mosaic.test()`, `t.phase()`, `t.step()`, all assertions, flow control, lifecycle, logging, and the console event emitter — producing a working test runner that communicates results via the console event protocol.

**Depends on:** Plan 03 (console event protocol format)

**Estimated scope:** L (3-5 days)

## Context

The `@mosaic/runtime` is the TypeScript half of Mosaic. It gets injected into the target app's webview via `Runtime.evaluate`, sets up the global `mosaic` object, and provides the test API that agents use to write tests. This plan covers the core API — the foundation that every test uses regardless of platform.

## Research Findings

- **esbuild 0.27.4** — will bundle this into a single IIFE file (Plan 11 handles the build pipeline).
- **IIFE format** — `var __mosaic = (()=>{...})()` — self-executing, no module system needed.
- **Console event protocol** — emits `console.warn('__MOSAIC__:' + JSON.stringify({event, data}))` for each event.
- **The runtime executes inside the target app's JavaScript context** — it has full access to the DOM, stores, and all app globals.
- No external dependencies. Pure TypeScript. Everything self-contained.

## Files to Create

```
runtime/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          — Entry point, sets up global `mosaic` object
│   ├── emitter.ts        — Console event emitter (the wire protocol)
│   ├── runner.ts         — Test runner (executes phases/steps in order)
│   ├── context.ts        — Test context (the `t` object)
│   ├── assertions.ts     — assert, assertEqual, assertMatch, assertTiming
│   └── types.ts          — TypeScript type definitions
```

## Implementation Details

### package.json

```json
{
  "name": "@mosaic/runtime",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "esbuild src/index.ts --bundle --format=iife --global-name=__mosaic --platform=browser --target=es2022 --minify --outfile=dist/runtime.js",
    "build:dev": "esbuild src/index.ts --bundle --format=iife --global-name=__mosaic --platform=browser --target=es2022 --outfile=dist/runtime.js",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "esbuild": "^0.27.0",
    "typescript": "^5.7.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"]
  },
  "include": ["src/**/*.ts"]
}
```

### Emitter (src/emitter.ts)

The bridge between the runtime and the Rust CLI. Supports two paths:
1. **Tauri Plugin path** — events go via Tauri IPC (`__MOSAIC_EMIT__` global set by the plugin's bridge script)
2. **Debug protocol path** — events go via `console.warn` (captured by WebKit Inspector / CDP)

```typescript
const MOSAIC_PREFIX = '__MOSAIC__:';

export function emit(event: string, data: Record<string, unknown>): void {
  const json = JSON.stringify({ event, data });

  if ((globalThis as any).__MOSAIC_EMIT__) {
    // Tauri Plugin path — direct IPC, no console.warn overhead
    (globalThis as any).__MOSAIC_EMIT__(json);
  } else {
    // Debug protocol path — captured via Console.messageAdded / Runtime.consoleAPICalled
    console.warn(`${MOSAIC_PREFIX}${json}`);
  }
}
```

### Types (src/types.ts)

```typescript
export type StepStatus = 'pass' | 'fail' | 'skip';

export interface StepResult {
  name: string;
  status: StepStatus;
  duration_ms: number;
  error?: StepError;
  snapshot?: unknown;
}

export interface StepError {
  type: string;
  message: string;
  expected?: string;
  actual?: string;
}

export interface TestOptions {
  timeout?: number;
}

export type TestFn = (t: TestContext) => Promise<void>;
export type StepFn = () => Promise<void>;
export type CleanupFn = () => Promise<void>;
```

### Test Context (src/context.ts)

The `t` object passed to every test:

```typescript
import { emit } from './emitter';
import { AssertionError, assertEqual, assertMatch, assertTiming } from './assertions';
import type { StepError, CleanupFn } from './types';

export class TestContext {
  private _cleanups: CleanupFn[] = [];
  private _ctx = new Map<string, unknown>();
  private _currentPhase: string | null = null;
  private _failed = false;
  private _stepTimeout: number;

  public readonly ctx = {
    set: (key: string, value: unknown) => this._ctx.set(key, value),
    get: (key: string) => this._ctx.get(key),
  };

  constructor(options: { timeout: number }) {
    this._stepTimeout = options.timeout;
  }

  // -- Structure --

  async phase(name: string, fn: () => Promise<void>): Promise<void> {
    this._currentPhase = name;
    emit('phase:start', { name });
    try {
      await fn();
    } finally {
      emit('phase:end', { name });
      this._currentPhase = null;
    }
  }

  async step(name: string, fn: () => Promise<void>, opts?: { retries?: number }): Promise<void> {
    if (this._failed) {
      emit('step:skip', { name, reason: 'previous step failed' });
      return;
    }

    emit('step:start', { name });
    const start = performance.now();
    const maxAttempts = (opts?.retries ?? 0) + 1;

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        await this._withTimeout(fn(), this._stepTimeout);
        const duration_ms = Math.round(performance.now() - start);
        emit('step:pass', { name, duration_ms });
        return;
      } catch (err) {
        if (attempt < maxAttempts) continue;

        const duration_ms = Math.round(performance.now() - start);
        const error = this._toStepError(err);
        emit('step:fail', { name, duration_ms, error });
        this._failed = true;
        return;
      }
    }
  }

  // -- Assertions --

  assert(condition: boolean, message: string): void {
    if (!condition) {
      throw new AssertionError(message);
    }
  }

  assertEqual(actual: unknown, expected: unknown, message?: string): void {
    assertEqual(actual, expected, message);
  }

  assertMatch(str: string, pattern: RegExp | string): void {
    assertMatch(str, pattern);
  }

  async assertTiming(fn: () => Promise<void>, opts: { max: number }): Promise<void> {
    await assertTiming(fn, opts);
  }

  // -- Flow control --

  async waitFor(
    predicate: () => boolean | Promise<boolean>,
    opts?: { timeout?: number; interval?: number }
  ): Promise<void> {
    const timeout = opts?.timeout ?? 30_000;
    const interval = opts?.interval ?? 100;
    const start = performance.now();

    while (true) {
      const result = await predicate();
      if (result) return;

      if (performance.now() - start > timeout) {
        throw new Error(`waitFor timed out after ${timeout}ms`);
      }

      await this.sleep(interval);
    }
  }

  async sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  fail(message: string): never {
    throw new AssertionError(message);
  }

  skip(reason?: string): void {
    // Emit skip for current step context
    throw new SkipError(reason);
  }

  // -- Logging --

  log(message: string): void {
    emit('log', { message });
  }

  snapshot(label: string, data: unknown): void {
    emit('snapshot', { label, data });
  }

  async measure(label: string, fn: () => Promise<void>): Promise<void> {
    const start = performance.now();
    await fn();
    const duration_ms = Math.round(performance.now() - start);
    this.log(`[measure] ${label}: ${duration_ms}ms`);
  }

  // -- Lifecycle --

  cleanup(fn: CleanupFn): void {
    this._cleanups.push(fn);
  }

  /** Called by the runner after test completes. */
  async _runCleanups(): Promise<void> {
    // Run in reverse registration order
    for (const fn of this._cleanups.reverse()) {
      try {
        await fn();
      } catch (err) {
        emit('log', { message: `[cleanup error] ${String(err)}` });
      }
    }
  }

  get hasFailed(): boolean {
    return this._failed;
  }

  // -- Internal --

  private async _withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
    return Promise.race([
      promise,
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error(`Step timed out after ${ms}ms`)), ms)
      ),
    ]);
  }

  private _toStepError(err: unknown): StepError {
    if (err instanceof AssertionError) {
      return {
        type: 'AssertionError',
        message: err.message,
        expected: err.expected,
        actual: err.actual,
      };
    }
    return {
      type: err instanceof Error ? err.constructor.name : 'Error',
      message: String(err),
    };
  }
}

class SkipError extends Error {
  constructor(public reason?: string) {
    super(reason ?? 'skipped');
  }
}
```

### Assertions (src/assertions.ts)

```typescript
export class AssertionError extends Error {
  constructor(
    message: string,
    public expected?: string,
    public actual?: string,
  ) {
    super(message);
    this.name = 'AssertionError';
  }
}

export function assertEqual(actual: unknown, expected: unknown, message?: string): void {
  if (!deepEqual(actual, expected)) {
    throw new AssertionError(
      message ?? `Expected values to be equal`,
      JSON.stringify(expected),
      JSON.stringify(actual),
    );
  }
}

export function assertMatch(str: string, pattern: RegExp | string): void {
  const matches = typeof pattern === 'string'
    ? str.includes(pattern)
    : pattern.test(str);

  if (!matches) {
    throw new AssertionError(
      `String does not match pattern`,
      String(pattern),
      str,
    );
  }
}

export async function assertTiming(fn: () => Promise<void>, opts: { max: number }): Promise<void> {
  const start = performance.now();
  await fn();
  const duration = performance.now() - start;
  if (duration > opts.max) {
    throw new AssertionError(
      `Exceeded timing limit`,
      `< ${opts.max}ms`,
      `${Math.round(duration)}ms`,
    );
  }
}

function deepEqual(a: unknown, b: unknown): boolean {
  if (a === b) return true;
  if (a === null || b === null) return false;
  if (typeof a !== typeof b) return false;

  if (Array.isArray(a) && Array.isArray(b)) {
    if (a.length !== b.length) return false;
    return a.every((v, i) => deepEqual(v, b[i]));
  }

  if (typeof a === 'object' && typeof b === 'object') {
    const keysA = Object.keys(a as Record<string, unknown>);
    const keysB = Object.keys(b as Record<string, unknown>);
    if (keysA.length !== keysB.length) return false;
    return keysA.every(key =>
      deepEqual(
        (a as Record<string, unknown>)[key],
        (b as Record<string, unknown>)[key]
      )
    );
  }

  return false;
}
```

### Test Runner (src/runner.ts)

```typescript
import { TestContext } from './context';
import { emit } from './emitter';
import type { TestFn } from './types';

export async function runTest(name: string, fn: TestFn, timeout: number): Promise<void> {
  emit('test:start', { name });
  const start = performance.now();

  const t = new TestContext({ timeout });
  let passed = 0;
  let failed = 0;
  let skipped = 0;

  try {
    await fn(t);
  } catch (err) {
    emit('test:error', { message: String(err) });
  } finally {
    await t._runCleanups();

    const duration_ms = Math.round(performance.now() - start);
    // Note: step counts are tracked by the CLI from step events
    emit('test:end', { passed: 0, failed: 0, skipped: 0, duration_ms });
  }
}
```

### Entry Point (src/index.ts)

```typescript
import { runTest } from './runner';
import type { TestFn } from './types';

interface MosaicGlobal {
  test(name: string, fn: TestFn): { name: string; fn: TestFn };
  _registeredTest: { name: string; fn: TestFn } | null;
  _run(timeout: number): Promise<void>;
}

const mosaic: MosaicGlobal = {
  test(name: string, fn: TestFn) {
    const registration = { name, fn };
    mosaic._registeredTest = registration;
    return registration;
  },

  _registeredTest: null,

  async _run(timeout: number) {
    const test = mosaic._registeredTest;
    if (!test) {
      throw new Error('No test registered. Did your test file export default mosaic.test(...)?');
    }
    await runTest(test.name, test.fn, timeout);
  },
};

// Expose as global
(globalThis as any).mosaic = mosaic;

export { mosaic };
```

### How It Works End-to-End

1. Rust CLI evaluates `runtime.js` in the webview → sets up `window.mosaic`
2. Rust CLI evaluates `await import('http://localhost:1420/.mosaic/test.test.ts')` → test file runs, calls `mosaic.test()`, which registers the test function
3. Rust CLI evaluates `await mosaic._run(30000)` → runner executes the test, emitting events
4. Rust CLI captures `console.warn` events, filters by `__MOSAIC__:` prefix, parses JSON

## Acceptance Criteria

- [ ] `npm run build` produces a single `dist/runtime.js` IIFE file
- [ ] `npm run typecheck` passes with zero TypeScript errors
- [ ] `mosaic.test()` registers a test function on the global
- [ ] `mosaic._run()` executes the registered test
- [ ] `t.phase()` emits `phase:start` and `phase:end` events
- [ ] `t.step()` emits `step:start` and `step:pass` or `step:fail` events
- [ ] Failed steps cause subsequent steps in the same test to be skipped
- [ ] `t.step()` with `retries` retries the specified number of times before failing
- [ ] `t.assert()` throws `AssertionError` on false condition
- [ ] `t.assertEqual()` performs deep equality comparison
- [ ] `t.assertMatch()` works with both string and RegExp patterns
- [ ] `t.waitFor()` polls the predicate and times out correctly
- [ ] `t.cleanup()` runs cleanup handlers in reverse order after test completion
- [ ] `t.ctx.set/get` shares data between steps
- [ ] `t.log()` emits a `log` event
- [ ] `t.snapshot()` emits a `snapshot` event
- [ ] All emitted events match the format defined in Plan 03
- [ ] No external runtime dependencies — the bundle is fully self-contained
- [ ] Unit tests for assertions, deep equality, and event emission
