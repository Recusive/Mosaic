# Mosaic v1.1 — Backend / Node.js Testing

**Version:** 1.0
**Date:** March 25, 2026
**Status:** Pre-development
**Depends on:** spec-full.md (architecture, API reference), spec-v1.md (core CLI, output engine)

> Test the backend the same way you test the frontend — from inside.

---

## Scope

v1.1 extends Mosaic to backend applications. Starting with Node.js (the most common backend for projects that also have a frontend), then expanding to any backend accessible via HTTP.

**In scope:**
- Node Inspector Protocol adapter
- Direct process execution for Node.js tests
- HTTP adapter for testing any backend API
- Auto-detection of Node.js backend projects
- Process management (spawn, monitor, lifecycle)
- API scope: Core, HTTP, Process, Filesystem

**Out of scope (later versions):**
- Python/Go/Rust backend deep integration (can test via HTTP)
- Database adapters (test via app's API or direct imports)
- Message queue testing (test via app's API)

---

## Two Modes of Backend Testing

### Mode 1: Inside the Process (Node.js)

For Node.js backends, Mosaic can run the test INSIDE the same process as the app — same approach as frontend testing. The test imports app modules directly.

```typescript
// .mosaic/user-service.test.ts
import { UserService } from '../src/services/user-service';
import { db } from '../src/db';

export default mosaic.test('user-service', async (t) => {
  t.cleanup(async () => {
    await db.query('DELETE FROM users WHERE email LIKE $1', ['%@test.mosaic%']);
  });

  await t.phase('CRUD Operations', async () => {
    await t.step('Create user', async () => {
      const user = await UserService.create({
        name: 'Test User',
        email: 'test@test.mosaic',
      });
      t.assert(user.id !== undefined, 'Should return user with ID');
      t.assertEqual(user.name, 'Test User');
      t.ctx.set('userId', user.id);
    });

    await t.step('Read user', async () => {
      const user = await UserService.findById(t.ctx.get('userId'));
      t.assert(user !== null, 'User should exist');
      t.assertEqual(user.email, 'test@test.mosaic');
    });

    await t.step('Delete user', async () => {
      await UserService.delete(t.ctx.get('userId'));
      const user = await UserService.findById(t.ctx.get('userId'));
      t.assert(user === null, 'User should be deleted');
    });
  });
});
```

**How it works:**

1. Mosaic detects a Node.js project (package.json with backend entry point)
2. Starts the app with Node Inspector enabled: `node --inspect=9229 src/index.js`
3. Connects via Node Inspector Protocol
4. Injects `@mosaic/runtime.js` via `Runtime.evaluate`
5. Evaluates dynamic import of the test file
6. Test imports resolve from the app's `node_modules` — same module instances
7. Structured output captured via `Runtime.consoleAPICalled`

**Alternatively**, for simpler setups, Mosaic can run the test directly:

1. Start a Node process with the runtime preloaded:
   `node --import @mosaic/runtime.js .mosaic/user-service.test.ts`
2. The test imports app modules — they load fresh but with the same code
3. Structured output captured via stdout

### Mode 2: Outside the Process (HTTP)

For any backend — Node.js, Python, Go, Rust, Java — test via HTTP. The test doesn't run inside the app; it makes API calls and verifies responses.

```typescript
// .mosaic/api-auth.test.ts

export default mosaic.test('api-auth', async (t) => {
  const BASE = 'http://localhost:3000';

  t.cleanup(async () => {
    // Clean up test data via admin endpoint
    await t.http.delete(`${BASE}/admin/test-users`);
  });

  await t.phase('Authentication Flow', async () => {
    await t.step('Register new user', async () => {
      const res = await t.http.post(`${BASE}/auth/register`, {
        email: 'test@mosaic.sh',
        password: 'test-password-123',
      });
      t.http.assertStatus(res, 201);
      t.assert(res.body.token !== undefined, 'Should return auth token');
      t.ctx.set('token', res.body.token);
    });

    await t.step('Login with credentials', async () => {
      const res = await t.http.post(`${BASE}/auth/login`, {
        email: 'test@mosaic.sh',
        password: 'test-password-123',
      });
      t.http.assertStatus(res, 200);
      t.assert(res.body.token !== undefined, 'Should return auth token');
    });

    await t.step('Access protected route with token', async () => {
      const res = await t.http.get(`${BASE}/api/me`, {
        headers: { Authorization: `Bearer ${t.ctx.get('token')}` },
      });
      t.http.assertStatus(res, 200);
      t.assertEqual(res.body.email, 'test@mosaic.sh');
    });

    await t.step('Reject invalid token', async () => {
      const res = await t.http.get(`${BASE}/api/me`, {
        headers: { Authorization: 'Bearer invalid-token' },
      });
      t.http.assertStatus(res, 401);
    });

    await t.step('Rate limiting works', async () => {
      // Hit the login endpoint repeatedly
      for (let i = 0; i < 10; i++) {
        await t.http.post(`${BASE}/auth/login`, {
          email: 'test@mosaic.sh',
          password: 'wrong-password',
        });
      }

      const res = await t.http.post(`${BASE}/auth/login`, {
        email: 'test@mosaic.sh',
        password: 'wrong-password',
      });
      t.http.assertStatus(res, 429);
    });
  });
});
```

**How it works:**

1. Mosaic detects a backend project with a start script
2. Starts the server: `npm run dev` or `node src/index.js`
3. Waits for health check: polls `http://localhost:3000` until responding
4. Runs the test in a helper Node process (for `t.http.*` API)
5. Test makes HTTP requests to the running server
6. Structured output captured via stdout
7. Server stays running for next test

---

## Auto-Detection for Backend

```
Step 1: Identify backend project
  ├── package.json exists
  ├── No tauri.conf.json (not a Tauri project)
  ├── No src/index.html or public/index.html (not a frontend-only project)
  ├── Has "main" or "scripts.start" in package.json
  └── IDENTIFIED as Node.js backend

Step 2: Determine entry point
  ├── package.json "main" field → src/index.js
  ├── package.json "scripts.dev" → parse command for entry point
  ├── Common patterns: src/index.ts, src/server.ts, src/app.ts, index.js
  └── FOUND entry point

Step 3: Determine server port
  ├── Read entry point for PORT or listen() calls
  ├── Check .env file for PORT
  ├── Check package.json scripts for --port flag
  ├── Default: 3000
  └── FOUND port

Step 4: Start server if needed
  ├── Check if port is responding
  ├── YES → skip, use existing server
  ├── NO → start via "scripts.dev" or "node <entry>"
  ├── Wait for health check on detected port
  └── READY

Step 5: Determine test mode
  ├── Test imports app modules directly → Mode 1 (inside, use Node Inspector)
  ├── Test only uses t.http.* → Mode 2 (outside, HTTP only)
  └── Selected mode
```

### Mixed Projects (Fullstack)

Many projects have both frontend and backend. Mosaic detects based on what the TEST needs:

- Test uses `t.dom.*`, `t.store.*` → frontend test, use webview adapter
- Test uses `t.http.*`, `t.proc.*` → backend test, use Node/HTTP adapter
- Test uses both → run frontend adapter (can also make HTTP calls via `t.net.fetch`)

---

## Process Management for Backend

### Starting the Server

```typescript
// Mosaic internally manages the server process
const server = spawn('node', ['src/index.js'], {
  env: {
    ...process.env,
    NODE_ENV: 'test',
    PORT: '3000',
  },
});

// Wait for readiness
await pollUntil(() => httpGet('http://localhost:3000/health').status === 200, {
  timeout: 30_000,
  interval: 500,
});
```

### Health Check Strategies

Mosaic tries multiple strategies to detect when the server is ready:

1. **HTTP health endpoint** — `GET /health`, `GET /api/health`, `GET /`
2. **TCP port open** — connection succeeds on the expected port
3. **Stdout marker** — server logs "listening on port 3000" or similar

### Graceful Shutdown

After tests complete, the server keeps running (same principle as Tauri dev server). The agent will test again soon. Only tear down with `--cleanup` flag.

---

## HTTP Client Details

The `t.http.*` API uses a built-in HTTP client with sensible defaults for testing:

```typescript
interface HttpOptions {
  headers?: Record<string, string>;
  timeout?: number;       // Default: 30s
  followRedirects?: boolean;  // Default: true
  validateStatus?: (status: number) => boolean;  // Default: accept all
}

interface HttpResponse {
  status: number;
  statusText: string;
  headers: Record<string, string>;
  body: any;              // Auto-parsed JSON, raw string otherwise
  duration: number;       // Response time in ms
  raw: string;            // Raw response body
}
```

**Features:**
- Automatic JSON parsing when `Content-Type: application/json`
- Request/response duration tracking (useful for `t.assertTiming`)
- Cookie jar per test (maintains session across requests)
- Automatic content-type headers for object bodies (sends as JSON)

---

## Example: Full Backend Test

```typescript
// .mosaic/api-full.test.ts

export default mosaic.test('api-full', async (t) => {
  const BASE = 'http://localhost:3000/api';

  await t.phase('Health Check', async () => {
    await t.step('Server is responsive', async () => {
      const res = await t.http.get(`${BASE}/health`);
      t.http.assertStatus(res, 200);
      t.assertEqual(res.body.status, 'ok');
    });

    await t.step('Response time under 100ms', async () => {
      await t.assertTiming(async () => {
        await t.http.get(`${BASE}/health`);
      }, { max: 100 });
    });
  });

  await t.phase('Data Operations', async () => {
    await t.step('Create resource', async () => {
      const res = await t.http.post(`${BASE}/items`, {
        name: 'Test Item',
        value: 42,
      });
      t.http.assertStatus(res, 201);
      t.ctx.set('itemId', res.body.id);
    });

    await t.step('List includes new resource', async () => {
      const res = await t.http.get(`${BASE}/items`);
      t.http.assertStatus(res, 200);
      const item = res.body.find(i => i.id === t.ctx.get('itemId'));
      t.assert(item !== undefined, 'New item should appear in list');
    });

    await t.step('Update resource', async () => {
      const res = await t.http.put(`${BASE}/items/${t.ctx.get('itemId')}`, {
        name: 'Updated Item',
        value: 99,
      });
      t.http.assertStatus(res, 200);
      t.assertEqual(res.body.name, 'Updated Item');
    });

    await t.step('Verify file side effect', async () => {
      // If the API writes to disk, verify it
      const exists = await t.fs.exists('./data/items.json');
      t.assert(exists, 'Items file should exist');

      const content = await t.fs.read('./data/items.json');
      const data = JSON.parse(content);
      const item = data.find(i => i.id === t.ctx.get('itemId'));
      t.assertEqual(item.value, 99, 'File should reflect updated value');
    });
  });

  t.cleanup(async () => {
    if (t.ctx.get('itemId')) {
      await t.http.delete(`${BASE}/items/${t.ctx.get('itemId')}`);
    }
  });
});
```

---

## Success Criteria for v1.1

1. **`mosaic test` runs against a Node.js backend** — detects project, starts server, waits for ready, runs test.

2. **Inside-the-process testing works** — test imports app modules directly, accesses real database connections, real service instances.

3. **HTTP testing works against any backend** — Node.js, Python, Go, Rust — anything with an HTTP API.

4. **Server lifecycle is managed** — auto-start, health check, keep alive between tests.

5. **Output is identical in format to v1** — same structured text, same JSON, same streaming. An agent can't tell if it tested a frontend or backend based on output format alone.
