# Plan 10: @mosaic/runtime — Domain APIs

**Objective:** Implement the platform-specific test API domains — DOM, State, Network, Viewport, Filesystem, Events, and IPC — extending the core runtime with full access to the target application's internals.

**Depends on:** Plan 09

**Estimated scope:** L (3-5 days)

## Context

The core API (Plan 09) provides test structure and assertions. The domain APIs provide access to the target app's internals — the DOM, state stores, network, viewport, filesystem, events, and IPC. These are what make Mosaic tests powerful: direct access to the same stores, functions, and state the running app uses.

For v1 (Tauri), all domains listed in the spec's API availability table are in scope: Core (done in Plan 09), State, DOM, Network, Viewport, Filesystem, Events, IPC.

## Research Findings

- All domain APIs run inside the browser context — they have native access to `document`, `window`, and the app's globals.
- Tauri IPC: `window.__TAURI_INTERNALS__.invoke` (v2) or `window.__TAURI__.invoke` (v1).
- Tauri events: `window.__TAURI_INTERNALS__.listen` / `window.__TAURI_INTERNALS__.emit` (v2).
- Filesystem: via Tauri's `@tauri-apps/plugin-fs` or `window.__TAURI__.fs` (v1).
- DOM operations use standard browser APIs (`document.querySelector`, `element.click()`, etc.).
- Network interception via `fetch` monkey-patching and `XMLHttpRequest` wrapping.
- Viewport control via `window.resizeTo()` (may require platform-specific handling in Tauri).

## Files to Create

```
runtime/src/api/
├── dom.ts          — DOM querying, interaction, waiting
├── state.ts        — Store access (Zustand, Redux, MobX, generic)
├── network.ts      — Network interception and assertion
├── viewport.ts     — Viewport size control
├── filesystem.ts   — File operations (via Tauri fs plugin)
├── events.ts       — App event system (Tauri events)
└── ipc.ts          — IPC calls (Tauri invoke)
```

## Files to Modify

- `runtime/src/context.ts` — add domain API properties to `TestContext`
- `runtime/src/index.ts` — ensure domains are included in bundle

## Implementation Details

### DOM API (runtime/src/api/dom.ts)

```typescript
export class DomApi {
  // -- Querying --
  query(selector: string): Element | null {
    return document.querySelector(selector);
  }

  queryAll(selector: string): Element[] {
    return Array.from(document.querySelectorAll(selector));
  }

  text(selector: string): string {
    return document.querySelector(selector)?.textContent ?? '';
  }

  attr(selector: string, name: string): string | null {
    return document.querySelector(selector)?.getAttribute(name) ?? null;
  }

  style(selector: string, property: string): string {
    const el = document.querySelector(selector);
    return el ? getComputedStyle(el).getPropertyValue(property) : '';
  }

  rect(selector: string): DOMRect | null {
    return document.querySelector(selector)?.getBoundingClientRect() ?? null;
  }

  visible(selector: string): boolean {
    const el = document.querySelector(selector);
    if (!el) return false;
    const rect = el.getBoundingClientRect();
    const style = getComputedStyle(el);
    return style.display !== 'none' && style.visibility !== 'hidden' && rect.width > 0 && rect.height > 0;
  }

  // -- Interaction --
  async click(selector: string): Promise<void> {
    const el = this._require(selector);
    el.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
  }

  async type(selector: string, text: string): Promise<void> {
    const el = this._require(selector) as HTMLInputElement;
    el.focus();
    for (const char of text) {
      el.dispatchEvent(new KeyboardEvent('keydown', { key: char, bubbles: true }));
      el.dispatchEvent(new KeyboardEvent('keypress', { key: char, bubbles: true }));
      el.value += char;
      el.dispatchEvent(new InputEvent('input', { data: char, bubbles: true }));
      el.dispatchEvent(new KeyboardEvent('keyup', { key: char, bubbles: true }));
      await new Promise(r => setTimeout(r, 10)); // Keystroke delay
    }
  }

  async fill(selector: string, text: string): Promise<void> {
    const el = this._require(selector) as HTMLInputElement;
    el.focus();
    el.value = text;
    el.dispatchEvent(new InputEvent('input', { data: text, bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  }

  async press(key: string): Promise<void> {
    document.dispatchEvent(new KeyboardEvent('keydown', { key, bubbles: true }));
    document.dispatchEvent(new KeyboardEvent('keyup', { key, bubbles: true }));
  }

  async waitFor(selector: string, opts?: { timeout?: number; visible?: boolean }): Promise<Element> {
    const timeout = opts?.timeout ?? 30_000;
    const start = performance.now();
    while (true) {
      const el = document.querySelector(selector);
      if (el && (!opts?.visible || this.visible(selector))) return el;
      if (performance.now() - start > timeout) {
        throw new Error(`waitFor('${selector}') timed out after ${timeout}ms`);
      }
      await new Promise(r => setTimeout(r, 100));
    }
  }

  // -- Internal --
  private _require(selector: string): Element {
    const el = document.querySelector(selector);
    if (!el) throw new Error(`Element not found: ${selector}`);
    return el;
  }
}
```

### State API (runtime/src/api/state.ts)

```typescript
export class StateApi {
  get(path: string): unknown {
    return resolvePath(path);
  }

  set(path: string, value: unknown): void {
    setPath(path, value);
  }

  snapshot(): Record<string, unknown> {
    // Attempt to read known store patterns
    const result: Record<string, unknown> = {};
    // Zustand stores expose getState()
    // This is a best-effort scan
    return result;
  }

  async waitFor(
    path: string,
    predicate: (value: unknown) => boolean,
    opts?: { timeout?: number }
  ): Promise<void> {
    const timeout = opts?.timeout ?? 30_000;
    const start = performance.now();
    while (true) {
      const value = this.get(path);
      if (predicate(value)) return;
      if (performance.now() - start > timeout) {
        throw new Error(`store.waitFor('${path}') timed out after ${timeout}ms`);
      }
      await new Promise(r => setTimeout(r, 100));
    }
  }
}

function resolvePath(path: string): unknown {
  const parts = path.split('.');
  let current: unknown = (globalThis as any);
  for (const part of parts) {
    if (current == null) return undefined;
    current = (current as Record<string, unknown>)[part];
  }
  return current;
}

function setPath(path: string, value: unknown): void {
  const parts = path.split('.');
  const last = parts.pop()!;
  let current: any = globalThis;
  for (const part of parts) {
    current = current[part];
    if (current == null) throw new Error(`Cannot set path: ${path} — ${part} is null`);
  }
  current[last] = value;
}
```

### IPC API (runtime/src/api/ipc.ts)

```typescript
export class IpcApi {
  async invoke(command: string, args?: unknown): Promise<unknown> {
    const tauri = this._getTauriInternals();
    return tauri.invoke(command, args);
  }

  on(channel: string, callback: (data: unknown) => void): () => void {
    const tauri = this._getTauriInternals();
    let unlisten: (() => void) | null = null;
    tauri.listen(channel, (event: { payload: unknown }) => {
      callback(event.payload);
    }).then((fn: () => void) => { unlisten = fn; });
    return () => { unlisten?.(); };
  }

  emit(channel: string, data?: unknown): void {
    const tauri = this._getTauriInternals();
    tauri.emit(channel, data);
  }

  private _getTauriInternals(): any {
    // Tauri v2
    if ((window as any).__TAURI_INTERNALS__) {
      return (window as any).__TAURI_INTERNALS__;
    }
    // Tauri v1
    if ((window as any).__TAURI__) {
      return (window as any).__TAURI__;
    }
    throw new Error('Tauri IPC not available — is this a Tauri app?');
  }
}
```

### Events API (runtime/src/api/events.ts)

```typescript
export class EventsApi {
  private _listeners: Map<string, Set<(data: unknown) => void>> = new Map();

  emit(name: string, data?: unknown): void {
    window.dispatchEvent(new CustomEvent(name, { detail: data }));
  }

  on(name: string, callback: (data: unknown) => void): () => void {
    const handler = (e: Event) => callback((e as CustomEvent).detail);
    window.addEventListener(name, handler);
    return () => window.removeEventListener(name, handler);
  }

  async waitFor(name: string, opts?: { timeout?: number }): Promise<unknown> {
    const timeout = opts?.timeout ?? 30_000;
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`events.waitFor('${name}') timed out after ${timeout}ms`));
      }, timeout);

      const unsub = this.on(name, (data) => {
        clearTimeout(timer);
        unsub();
        resolve(data);
      });
    });
  }
}
```

### Network API (runtime/src/api/network.ts)

```typescript
export class NetworkApi {
  private _interceptors: Array<{ pattern: RegExp; handler: (req: Request) => Response | void }> = [];
  private _requests: Request[] = [];
  private _originalFetch: typeof fetch;

  constructor() {
    this._originalFetch = globalThis.fetch;
    this._patchFetch();
  }

  intercept(pattern: string | RegExp, handler: (req: Request) => Response | void): () => void {
    const re = typeof pattern === 'string' ? new RegExp(pattern) : pattern;
    const entry = { pattern: re, handler };
    this._interceptors.push(entry);
    return () => {
      this._interceptors = this._interceptors.filter(i => i !== entry);
    };
  }

  mock(pattern: string | RegExp, response: { status?: number; body?: unknown; headers?: Record<string, string> }): () => void {
    return this.intercept(pattern, () => {
      return new Response(JSON.stringify(response.body), {
        status: response.status ?? 200,
        headers: response.headers ?? { 'Content-Type': 'application/json' },
      });
    });
  }

  async waitFor(pattern: string | RegExp, opts?: { timeout?: number }): Promise<Request> {
    const re = typeof pattern === 'string' ? new RegExp(pattern) : pattern;
    const timeout = opts?.timeout ?? 30_000;
    const start = performance.now();
    while (true) {
      const found = this._requests.find(r => re.test(r.url));
      if (found) return found;
      if (performance.now() - start > timeout) {
        throw new Error(`net.waitFor(${pattern}) timed out`);
      }
      await new Promise(r => setTimeout(r, 100));
    }
  }

  requests(): Request[] {
    return [...this._requests];
  }

  async fetch(url: string, opts?: RequestInit): Promise<Response> {
    return this._originalFetch(url, opts);
  }

  private _patchFetch(): void {
    const self = this;
    globalThis.fetch = async function(input: RequestInfo | URL, init?: RequestInit): Promise<Response> {
      const req = new Request(input, init);
      self._requests.push(req);

      for (const { pattern, handler } of self._interceptors) {
        if (pattern.test(req.url)) {
          const result = handler(req);
          if (result) return result;
        }
      }

      return self._originalFetch(input, init);
    };
  }
}
```

### Viewport API (runtime/src/api/viewport.ts)

```typescript
export class ViewportApi {
  async set(width: number, height: number): Promise<void> {
    window.resizeTo(width, height);
    await new Promise(r => setTimeout(r, 100)); // Allow layout to settle
  }

  get(): { width: number; height: number } {
    return { width: window.innerWidth, height: window.innerHeight };
  }

  async mobile(): Promise<void> { await this.set(375, 812); }
  async tablet(): Promise<void> { await this.set(768, 1024); }
  async desktop(): Promise<void> { await this.set(1440, 900); }
}
```

### Filesystem API (runtime/src/api/filesystem.ts)

```typescript
export class FilesystemApi {
  async read(path: string): Promise<string> {
    const tauri = this._getTauriFs();
    return tauri.readTextFile(path);
  }

  async write(path: string, content: string): Promise<void> {
    const tauri = this._getTauriFs();
    return tauri.writeTextFile(path, content);
  }

  async exists(path: string): Promise<boolean> {
    try {
      const tauri = this._getTauriFs();
      await tauri.stat(path);
      return true;
    } catch {
      return false;
    }
  }

  private _getTauriFs(): any {
    // Tauri v2: fs is a plugin
    const internals = (window as any).__TAURI_INTERNALS__;
    if (internals) {
      return {
        readTextFile: (path: string) => internals.invoke('plugin:fs|read_text_file', { path }),
        writeTextFile: (path: string, contents: string) => internals.invoke('plugin:fs|write_text_file', { path, contents }),
        stat: (path: string) => internals.invoke('plugin:fs|stat', { path }),
      };
    }
    // Tauri v1
    const tauriV1 = (window as any).__TAURI__?.fs;
    if (tauriV1) return tauriV1;

    throw new Error('Filesystem API not available — is the fs plugin enabled?');
  }
}
```

### Wiring into TestContext

```typescript
// In context.ts, add to TestContext constructor:
public readonly dom = new DomApi();
public readonly store = new StateApi();
public readonly net: NetworkApi;
public readonly viewport = new ViewportApi();
public readonly fs = new FilesystemApi();
public readonly events = new EventsApi();
public readonly ipc = new IpcApi();

constructor(options: { timeout: number }) {
  this._stepTimeout = options.timeout;
  this.net = new NetworkApi();
}
```

## Acceptance Criteria

- [ ] `t.dom.query()`, `t.dom.click()`, `t.dom.type()`, `t.dom.fill()` work against the real DOM
- [ ] `t.dom.waitFor()` polls for element presence with configurable timeout
- [ ] `t.dom.visible()` checks computed style and bounding rect
- [ ] `t.store.get()` resolves dot-notation paths on the global scope
- [ ] `t.ipc.invoke()` calls Tauri's IPC, detecting v1 vs v2 automatically
- [ ] `t.events.waitFor()` waits for a custom event with timeout
- [ ] `t.net.intercept()` patches `fetch` and intercepts matching requests
- [ ] `t.net.mock()` returns canned responses for matching URLs
- [ ] `t.viewport.set()` changes window dimensions
- [ ] `t.fs.read()` and `t.fs.write()` work via Tauri's fs plugin
- [ ] All domain APIs are accessible on the `t` object in tests
- [ ] `npm run build` still produces a single self-contained IIFE
- [ ] `npm run typecheck` passes
- [ ] No external runtime dependencies added
