# EdgeBase Common Pitfalls

A collection of common mistakes and solutions when developing EdgeBase apps.

---

## 1. onSnapshot only broadcasts for authenticated SDK requests

When `release: false`, unauthenticated direct `fetch()` calls for insert/update go through the D1 path, which does not broadcast to `DatabaseLiveDO`. Only authenticated SDK requests go through the `DatabaseDO` (Durable Object SQLite) path, which properly broadcasts changes.

### Symptoms
- `onSnapshot` subscription succeeds (WebSocket connected, `subscribed` message received)
- No `db_change` events arrive when data is modified
- `getList()` and other HTTP requests work fine

### Cause
- Unauthenticated request → D1 path → `executionCtx.waitUntil()` broadcast is silently dropped
- Authenticated request → DO SQLite path → `this.ctx.waitUntil()` runs properly inside the DO

### Solution
```typescript
// ❌ Wrong: unauthenticated direct fetch
await fetch(`${EDGEBASE_URL}/api/db/shared/tables/messages`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ content: 'hello' })
});

// ✅ Correct: authenticated SDK request
await edgebase.auth.signIn({ email, password }); // or signUp, signInAnonymously
await db().table('messages').insert({ content: 'hello' });
```

> **Note**: Polling fallback is unnecessary. `onSnapshot` works correctly when writes go through the authenticated SDK.

---

## 2. Deployed backend enforces strict schema validation

Local `edgebase dev` auto-creates tables and accepts any fields, but the deployed Cloudflare Workers environment only allows tables and fields defined in `edgebase.config.ts`.

### Symptoms
- Features that work locally fail silently after deployment
- Console shows `Table 'xxx' not found in database 'shared'` errors
- `insert` fails when using fields not in the schema

### Solution
Always cross-check client code table names and field names against your `edgebase.config.ts` schema.

```typescript
// edgebase.config.ts
tables: {
  friendships: {        // ← this name must match
    schema: {
      userId: { type: 'string', required: true },
      friendId: { type: 'string', required: true },
      displayName: { type: 'string' },  // ← only these fields are allowed
    }
  }
}

// Client code
db().table('friendships')  // ← must match exactly ('friends' ❌)
  .insert({
    userId: '...',
    friendId: '...',
    displayName: '...',   // ← only schema-defined fields
  });
```

---

## 3. SvelteKit: use `$env/static/public` for environment variables

`import.meta.env.PUBLIC_*` may not be inlined during SvelteKit builds.

### Symptoms
- `PUBLIC_EDGEBASE_URL` set in `.env.production` but not reflected in the build
- Production app sends requests to `localhost:4050`

### Solution
```typescript
// ❌ Wrong
const url = import.meta.env.PUBLIC_EDGEBASE_URL || 'http://localhost:4050';

// ✅ Correct
import { PUBLIC_EDGEBASE_URL } from '$env/static/public';
const url = PUBLIC_EDGEBASE_URL || 'http://localhost:4050';
```

Set environment variables in both `.env` (dev) and `.env.production` (deploy):
```
# .env
PUBLIC_EDGEBASE_URL=http://localhost:4050

# .env.production
PUBLIC_EDGEBASE_URL=https://your-backend.workers.dev
```

---

## 4. `edgebase dev` uses both port and port+1

Running `edgebase dev --port 4050` uses port 4050 for workerd and port 4051 for the node sidecar.

### Symptoms
- Port conflict with other apps using consecutive ports
- `Address already in use` error

### Solution
Verify both consecutive ports are available before starting:
```bash
lsof -i :4050 -i :4051
```

---

## 5. Cloudflare Free Plan requires `new_sqlite_classes` for all Durable Objects

On the free plan, all DO classes must be listed under `new_sqlite_classes` in wrangler.toml migrations.

### Symptoms
- `wrangler deploy` fails with `In order to use Durable Objects with a free plan, you must create a namespace using a new_sqlite_classes migration`

### Solution
```toml
# ❌ Fails on free plan
[[migrations]]
tag = "v1"
new_sqlite_classes = ["DatabaseDO", "AuthDO"]
new_classes = ["DatabaseLiveDO", "RoomsDO"]

# ✅ Works on free plan
[[migrations]]
tag = "v1"
new_sqlite_classes = ["DatabaseDO", "AuthDO", "DatabaseLiveDO", "RoomsDO"]
```
