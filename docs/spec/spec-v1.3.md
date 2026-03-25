# Mosaic v1.3 — Browser Webapp Testing

**Version:** 1.0
**Date:** March 25, 2026
**Status:** Pre-development
**Depends on:** spec-full.md (architecture, API reference), spec-v1.md (core CLI, output engine, WebKit/CDP adapters)

> Same power as desktop testing. Different target. Same `mosaic test`.

---

## Scope

v1.3 extends Mosaic to test web applications running in a browser. The architecture is nearly identical to Tauri desktop testing — connect via debug protocol, inject through the dev server, share the module graph. The main addition is browser lifecycle management (launching, headless mode, multi-browser support).

**In scope:**
- CDP adapter for Chrome/Edge (reused from v1)
- WebKit Inspector adapter for Safari (reused from v1)
- Browser launch and lifecycle management
- Framework auto-detection (React, Next.js, Vue, Svelte, Remix, Astro, etc.)
- Dev server auto-start for all major frameworks
- Headless mode for CI/CD
- Full API scope: Core, State, DOM, Network, Viewport, Events, Filesystem, HTTP

**Out of scope:**
- Cross-browser parallel testing (test one browser at a time)
- Visual regression testing (screenshot comparison — future feature)
- Browser extension testing

---

## Architecture for Browser Webapps

The browser testing architecture is a superset of Tauri testing. In Tauri, the webview is embedded in the desktop app. In browser testing, Mosaic manages the browser itself.

```
┌──────────────────────────────────────────────────┐
│  mosaic test sidebar-responsive                   │
│                                                  │
│  1. Scan project → find vite.config.ts           │
│  2. Read config → React + Vite on :5173          │
│  3. Check dev server → not running               │
│  4. Start: npx vite --port 5173                  │
│  5. Wait for dev server on :5173                 │
│  6. Launch Chrome:                               │
│     chrome --headless --remote-debugging-port=   │
│     9222 http://localhost:5173                    │
│  7. Connect CDP on :9222                         │
│  8. Inject @mosaic/runtime.js                    │
│  9. Import test via dev server:                  │
│     import('http://localhost:5173/               │
│            .mosaic/sidebar-responsive.test.ts')   │
│  10. Stream console events → format output       │
│  11. Report results → exit code                  │
└──────────────────────────────────────────────────┘
```

### Key Difference from Tauri

In Tauri testing, Mosaic connects to an already-running app's webview. In browser testing, Mosaic may need to:

1. **Launch a browser** — start Chrome/Edge/Safari with debug flags
2. **Navigate to the app** — open the dev server URL
3. **Manage browser lifecycle** — close after tests (or keep alive)

Everything after connection is identical — same injection mechanism, same runtime, same test API.

---

## Framework Auto-Detection

Mosaic identifies the framework to determine the dev server command and default port.

| Framework | Detection | Dev Command | Default Port |
|-----------|-----------|-------------|-------------|
| Vite (React, Vue, Svelte) | `vite.config.*` | `npx vite` | 5173 |
| Next.js | `next.config.*` | `npx next dev` | 3000 |
| Remix | `remix.config.*` or `@remix-run` in deps | `npx remix dev` | 5173 |
| Astro | `astro.config.*` | `npx astro dev` | 4321 |
| Nuxt | `nuxt.config.*` | `npx nuxi dev` | 3000 |
| SvelteKit | `svelte.config.*` | `npx vite dev` | 5173 |
| Angular | `angular.json` | `npx ng serve` | 4200 |
| Create React App | `react-scripts` in deps | `npx react-scripts start` | 3000 |
| Webpack (custom) | `webpack.config.*` | `npx webpack serve` | 8080 |
| Plain HTML | `index.html` in root | `npx serve .` | 3000 |

The port is always read from the actual config file first. Defaults are fallbacks.

### Detection Flow

```
Step 1: Identify web framework
  ├── next.config.* → Next.js
  ├── astro.config.* → Astro
  ├── nuxt.config.* → Nuxt
  ├── svelte.config.* → SvelteKit
  ├── angular.json → Angular
  ├── vite.config.* → Vite (check deps for React/Vue/Svelte)
  ├── webpack.config.* → Webpack
  ├── package.json "react-scripts" → Create React App
  ├── index.html in root → Static site
  └── IDENTIFIED framework

Step 2: Read port from config
  ├── Vite: vite.config.ts → server.port
  ├── Next.js: package.json scripts → --port flag
  ├── Angular: angular.json → serve.options.port
  ├── Fallback: use framework default
  └── FOUND port

Step 3: Start dev server if needed
  ├── Check if detected port is responding
  ├── YES → use existing server
  ├── NO → run framework's dev command
  ├── Wait for health check
  └── READY

Step 4: Launch browser
  ├── Detect installed browser (Chrome → Edge → Chromium → Safari)
  ├── Launch with debug flags:
  │   Chrome: --remote-debugging-port=9222 --headless=new
  │   Safari: (WebKit Inspector via safaridriver)
  ├── Navigate to dev server URL
  └── BROWSER READY

Step 5: Connect and test
  ├── Connect via CDP/WebKit Inspector (same as Tauri v1)
  ├── Inject runtime, import test, stream results
  └── DONE
```

---

## Browser Management

### Launching Chrome/Edge

```
google-chrome --headless=new \
              --remote-debugging-port=9222 \
              --disable-gpu \
              --no-sandbox \
              --window-size=1440,900 \
              http://localhost:5173
```

Mosaic checks for browser binaries in order:
1. `google-chrome` / `Google Chrome` (macOS app path)
2. `chromium`
3. `microsoft-edge` / `Microsoft Edge`
4. Falls back to bundled Chromium (future consideration)

### Headless vs Headed

```
mosaic test sidebar-responsive              # Headless (default for agents)
mosaic test sidebar-responsive --headed     # Show the browser (debugging)
```

Default is headless — the agent doesn't need to see the browser. Headed mode is useful for humans debugging a test or for the agent to take screenshots.

### Browser Lifecycle

```
First test run:
  1. Launch browser with debug flags
  2. Navigate to app URL
  3. Run test
  4. Keep browser open

Subsequent test runs:
  1. Detect existing browser on debug port
  2. Reuse it (navigate to app URL if needed)
  3. Run test
  4. Keep browser open

Cleanup:
  mosaic test --cleanup    → close browser after test
```

The browser stays alive between test runs, same philosophy as the dev server. Fast iteration.

---

## State Management Access

Browser webapps use the same state management libraries as Tauri apps. The `t.store` API works identically because tests share the module graph.

```typescript
// .mosaic/cart-flow.test.ts
import { useCartStore } from '@/stores/cart-store';

export default mosaic.test('cart-flow', async (t) => {
  t.cleanup(async () => {
    useCartStore.getState().clearCart();
  });

  await t.phase('Shopping Cart', async () => {
    await t.step('Add item to cart', async () => {
      const addItem = useCartStore.getState().addItem;
      addItem({ id: '1', name: 'Widget', price: 29.99 });

      const items = useCartStore.getState().items;
      t.assertEqual(items.length, 1);
      t.assertEqual(items[0].name, 'Widget');
    });

    await t.step('Cart badge updates in DOM', async () => {
      await t.dom.waitFor('[data-testid="cart-badge"]');
      const badge = t.dom.text('[data-testid="cart-badge"]');
      t.assertEqual(badge, '1');
    });

    await t.step('Cart total is correct', async () => {
      const total = useCartStore.getState().total();
      t.assertEqual(total, 29.99);
    });
  });
});
```

### Framework-Specific Store Access

| Framework | State Library | Access Pattern |
|-----------|--------------|----------------|
| React | Zustand | `import { useStore } from '@/stores/...'` → `useStore.getState()` |
| React | Redux | `import { store } from '@/store'` → `store.getState()` |
| React | Jotai | `import { store } from '@/store'` → `store.get(atom)` |
| Vue | Pinia | `import { useStore } from '@/stores/...'` → `useStore()` |
| Svelte | Svelte stores | `import { store } from '@/stores/...'` → `get(store)` |
| Angular | NgRx | `import { store } from '@/store'` → `store.select(...)` |

The test imports the store the same way the app does. No special Mosaic API needed — the shared module graph handles it.

---

## Examples

### Responsive Layout Test

```typescript
// .mosaic/sidebar-responsive.test.ts

export default mosaic.test('sidebar-responsive', async (t) => {

  await t.phase('Layout Breakpoints', async () => {
    await t.step('Desktop: sidebar visible', async () => {
      await t.viewport.desktop();  // 1440x900
      await t.dom.waitFor('.sidebar');
      t.assert(t.dom.visible('.sidebar'), 'Sidebar should be visible on desktop');

      const rect = t.dom.rect('.sidebar');
      t.assert(rect.width >= 240, 'Sidebar should be at least 240px wide');
    });

    await t.step('Tablet: sidebar collapsed', async () => {
      await t.viewport.tablet();  // 768x1024
      await t.sleep(300);  // Wait for CSS transition

      const sidebar = t.dom.query('.sidebar');
      const style = t.dom.style('.sidebar', 'width');
      t.assert(parseInt(style) < 80, 'Sidebar should be collapsed on tablet');
    });

    await t.step('Mobile: sidebar hidden', async () => {
      await t.viewport.mobile();  // 375x812
      await t.sleep(300);

      t.assert(!t.dom.visible('.sidebar'), 'Sidebar should be hidden on mobile');
    });

    await t.step('Mobile: hamburger menu opens sidebar', async () => {
      await t.dom.click('[data-testid="hamburger-menu"]');
      await t.dom.waitFor('.sidebar.open');
      t.assert(t.dom.visible('.sidebar'), 'Sidebar should be visible after hamburger click');
    });
  });
});
```

### Form Validation Test

```typescript
// .mosaic/signup-form.test.ts

export default mosaic.test('signup-form', async (t) => {

  await t.phase('Form Validation', async () => {
    await t.step('Empty form shows errors on submit', async () => {
      await t.dom.click('[data-testid="submit-btn"]');
      await t.dom.waitFor('.error-message');

      t.assertMatch(t.dom.text('.error-message'), /email is required/i);
    });

    await t.step('Invalid email shows error', async () => {
      await t.dom.fill('[name="email"]', 'not-an-email');
      await t.dom.click('[data-testid="submit-btn"]');

      t.assertMatch(t.dom.text('[name="email"] + .error'), /valid email/i);
    });

    await t.step('Valid form submits successfully', async () => {
      await t.dom.fill('[name="email"]', 'test@mosaic.sh');
      await t.dom.fill('[name="password"]', 'secure-password-123');
      await t.dom.check('[name="terms"]');

      // Intercept the API call
      const apiCall = t.net.waitFor('/api/auth/register');
      await t.dom.click('[data-testid="submit-btn"]');

      const req = await apiCall;
      t.assertEqual(JSON.parse(req.body).email, 'test@mosaic.sh');

      // Wait for success state
      await t.dom.waitFor('[data-testid="success-message"]');
      t.assertMatch(t.dom.text('[data-testid="success-message"]'), /welcome/i);
    });
  });
});
```

### Network Mocking Test

```typescript
// .mosaic/api-error-handling.test.ts

export default mosaic.test('api-error-handling', async (t) => {

  await t.phase('Error States', async () => {
    await t.step('Shows error message on API failure', async () => {
      // Mock the API to return 500
      const unmock = t.net.mock('/api/data', {
        status: 500,
        body: { error: 'Internal Server Error' },
      });

      // Trigger the API call (e.g., by navigating or clicking)
      await t.dom.click('[data-testid="load-data-btn"]');

      // Verify error state
      await t.dom.waitFor('[data-testid="error-banner"]');
      t.assertMatch(t.dom.text('[data-testid="error-banner"]'), /something went wrong/i);

      // Verify retry button appears
      t.assert(t.dom.visible('[data-testid="retry-btn"]'), 'Retry button should be visible');

      unmock();
    });

    await t.step('Retry recovers from error', async () => {
      // First call fails, second succeeds
      let callCount = 0;
      const unmock = t.net.intercept('/api/data', () => {
        callCount++;
        if (callCount === 1) {
          return { status: 500, body: { error: 'fail' } };
        }
        return { status: 200, body: { items: [{ id: 1, name: 'Widget' }] } };
      });

      await t.dom.click('[data-testid="load-data-btn"]');
      await t.dom.waitFor('[data-testid="error-banner"]');

      await t.dom.click('[data-testid="retry-btn"]');
      await t.dom.waitFor('[data-testid="data-list"]');

      t.assertEqual(callCount, 2, 'Should have made 2 API calls');
      t.assert(t.dom.visible('[data-testid="data-list"]'), 'Data should be visible after retry');

      unmock();
    });

    await t.step('Loading state shown during slow request', async () => {
      const unmock = t.net.intercept('/api/data', async () => {
        await new Promise(resolve => setTimeout(resolve, 2000));
        return { status: 200, body: { items: [] } };
      });

      await t.dom.click('[data-testid="load-data-btn"]');

      // Loading should appear immediately
      await t.dom.waitFor('[data-testid="loading-spinner"]');
      t.assert(t.dom.visible('[data-testid="loading-spinner"]'), 'Spinner should show');

      // Wait for load to complete
      await t.dom.waitFor('[data-testid="data-list"]', { timeout: 5000 });
      t.assert(!t.dom.visible('[data-testid="loading-spinner"]'), 'Spinner should be gone');

      unmock();
    });
  });
});
```

---

## Navigation and Routing

Browser webapps have client-side routing. Tests may need to navigate between pages:

```typescript
await t.step('Navigate to settings', async () => {
  // Option 1: Click a link (tests the actual navigation)
  await t.dom.click('a[href="/settings"]');
  await t.dom.waitFor('[data-page="settings"]');

  // Option 2: Direct URL navigation (faster, skips click)
  // Uses the protocol adapter to navigate
  await t.navigate('/settings');
  await t.dom.waitFor('[data-page="settings"]');
});
```

The `t.navigate()` method is available for browser targets. It uses the protocol adapter's page navigation:
- CDP: `Page.navigate`
- WebKit: `Page.navigate`

---

## CI/CD Integration

Browser testing in CI is headless by default. Mosaic handles this:

```yaml
# GitHub Actions example
- name: Run Mosaic tests
  run: |
    npm run dev &          # Start dev server
    sleep 5                # Wait for it
    mosaic test            # Run all tests (headless by default)
```

Or let Mosaic handle everything:

```yaml
- name: Run Mosaic tests
  run: mosaic test         # Auto-starts dev server and browser
```

Mosaic detects CI environments (`CI=true`, `GITHUB_ACTIONS=true`, etc.) and forces headless mode.

---

## Differences from Tauri Testing

| Aspect | Tauri (v1) | Browser (v1.3) |
|--------|-----------|----------------|
| Debug protocol connection | Connect to existing webview | Launch browser + connect |
| Dev server | Tauri manages via `cargo tauri dev` | Mosaic starts framework's dev server |
| IPC API (`t.ipc`) | Available (Tauri invoke) | Not available |
| Viewport control | Webview resize | Browser window resize |
| Filesystem access | Via Tauri's fs plugin | Limited (browser sandbox) |
| Headless mode | N/A (desktop app is always visible) | Available and default |
| Browser choice | N/A (uses system WebView) | Chrome, Edge, Safari |

The test API is the same. A DOM test, state test, or network test written for a Tauri app works in a browser and vice versa (minus platform-specific APIs like `t.ipc`).

---

## Success Criteria for v1.3

1. **`mosaic test` launches a browser and tests a web app** — auto-detects framework, starts dev server, launches Chrome headless, connects via CDP.

2. **All major frameworks supported** — React (Vite), Next.js, Vue, Svelte, Angular, Remix, Astro. Detection and dev server startup just works.

3. **Module graph sharing works in the browser** — tests import from the app's stores and modules. Same singletons, same state.

4. **Network mocking works** — intercept, mock, and verify API calls from within the test.

5. **Headless mode works in CI** — no display server required. Mosaic handles Chrome headless automatically.

6. **Same output format** — identical to v1 and v1.1. The agent doesn't need to know if it tested a desktop app, backend, or browser webapp.
