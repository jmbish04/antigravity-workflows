---
description: Optimize code for Cloudflare Workers and Pages
---

# Cloudflare Optimization

I will help you retrofit and optimize existing code for Cloudflare Workers and Pages, transforming non-optimized or "AI slop" code into production-ready Cloudflare applications.

## Guardrails
- Never break existing functionality - optimize incrementally
- Identify and remove Node.js-specific code that won't work in Workers
- Replace unsupported APIs with Workers-compatible alternatives
- Optimize for edge computing constraints (CPU time, memory)
- Follow Cloudflare best practices

## Steps

### 1. Analyze Current Codebase
Ask clarifying questions:
- What framework/stack is currently used?
- Where is it currently deployed? (Vercel, Railway, traditional server)
- What features does it use? (Database, file uploads, cron jobs)
- Are there any Node.js-specific dependencies?
- What are the performance bottlenecks?

### 2. Identify Incompatibilities
Scan for Workers-incompatible patterns:

**❌ Not supported in Workers:**
- `fs` (filesystem access)
- `child_process` (spawning processes)
- Most native Node.js modules
- Long-running processes (>30s CPU time)
- WebSockets server (client is OK)
- MongoDB (use D1, KV, or external API instead)

**✅ Use instead:**
- D1 for SQL databases
- R2 for file storage
- KV for key-value data
- Durable Objects for stateful logic
- External APIs for complex operations

### 3. Database Migration
**From traditional databases to D1:**

**Before** (Prisma with PostgreSQL):
```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const users = await prisma.user.findMany();
```

**After** (Drizzle with D1):
```typescript
import { drizzle } from 'drizzle-orm/d1';
import { users } from './db/schema';

export default {
  async fetch(request, env) {
    const db = drizzle(env.DB);
    const allUsers = await db.select().from(users);
    return Response.json(allUsers);
  }
}
```

**Migration steps:**
1. Export existing data to SQL
2. Create D1 database: `npx wrangler d1 create <name>`
3. Define Drizzle schema matching existing structure
4. Generate and apply migrations
5. Import data to D1
6. Update application code to use Drizzle

### 4. Replace Node.js APIs
**File System → R2**:
```typescript
// ❌ Before (Node.js)
import fs from 'fs';
const data = fs.readFileSync('./file.txt', 'utf-8');

// ✅ After (R2)
export default {
  async fetch(request, env) {
    const object = await env.MY_BUCKET.get('file.txt');
    const data = await object.text();
    return new Response(data);
  }
}
```

**Environment Variables → Bindings**:
```typescript
// ❌ Before
const apiKey = process.env.API_KEY;

// ✅ After
export default {
  async fetch(request, env) {
    const apiKey = env.API_KEY;
    return new Response('OK');
  }
}
```

**setTimeout/setInterval → Cron Triggers**:
```typescript
// ❌ Before
setInterval(() => {
  // Cleanup job
}, 3600000);

// ✅ After (wrangler.jsonc)
{
  "triggers": {
    "crons": ["0 * * * *"]  // Every hour
  }
}

// Worker
export default {
  async scheduled(event, env, ctx) {
    // Cleanup job
  }
}
```

### 5. Optimize Request Handling
**Convert to ES Modules format**:
```typescript
// ❌ Before (Service Worker syntax)
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

// ✅ After (ES Modules)
export default {
  async fetch(request, env, ctx) {
    return handleRequest(request);
  }
}
```

**Use proper Response objects**:
```typescript
// ❌ Before (Express-style)
app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});

// ✅ After (Workers)
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === '/api/users') {
      return Response.json({ users: [] });
    }
    return new Response('Not found', { status: 404 });
  }
}
```

### 6. Optimize for Edge Performance
**Remove blocking operations**:
```typescript
// ❌ Before (blocking)
const result = await heavyComputation();
await sendEmail(result);
return Response.json({ success: true });

// ✅ After (non-blocking)
export default {
  async fetch(request, env, ctx) {
    const result = await heavyComputation();

    // Background tasks
    ctx.waitUntil(sendEmail(result));

    return Response.json({ success: true });
  }
}
```

**Implement caching**:
```typescript
export default {
  async fetch(request, env, ctx) {
    const cache = caches.default;
    let response = await cache.match(request);

    if (!response) {
      response = await fetch('https://api.example.com/data');
      ctx.waitUntil(cache.put(request, response.clone()));
    }

    return response;
  }
}
```

### 7. Refactor Dependencies
**Remove bloated packages**:
- Replace moment.js with date-fns or native Date
- Replace lodash with native methods
- Use lightweight alternatives for heavy libraries

**Check bundle size**:
```bash
npx wrangler deploy --dry-run --outdir=dist
```

Review `dist/` to see what's included in your Worker.

### 8. Handle Authentication
**From session cookies to JWT**:
```typescript
// ❌ Before (express-session)
app.use(session({ secret: 'secret' }));

// ✅ After (JWT with KV)
import { SignJWT, jwtVerify } from 'jose';

export default {
  async fetch(request, env) {
    const token = request.headers.get('Authorization');
    if (!token) {
      return new Response('Unauthorized', { status: 401 });
    }

    try {
      const { payload } = await jwtVerify(
        token,
        new TextEncoder().encode(env.JWT_SECRET)
      );
      // User authenticated
      return Response.json({ user: payload });
    } catch {
      return new Response('Invalid token', { status: 401 });
    }
  }
}
```

### 9. Migrate Static Assets
**From Express static → Pages**:
```typescript
// ❌ Before
app.use(express.static('public'));

// ✅ After (Pages with Functions)
// Move static files to /public directory
// Create functions/ for dynamic routes
// Deploy to Pages
```

### 10. Update Build Configuration
Create `wrangler.jsonc`:
```json
{
  "name": "my-optimized-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-db",
      "database_id": "<id>"
    }
  ],
  "kv_namespaces": [
    {
      "binding": "CACHE",
      "id": "<id>"
    }
  ],
  "vars": {
    "ENVIRONMENT": "production"
  }
}
```

## Common Anti-Patterns to Fix

### 1. Using process.env everywhere
**❌ Before:**
```typescript
const apiKey = process.env.API_KEY;
```

**✅ After:**
```typescript
export default {
  async fetch(request, env) {
    const apiKey = env.API_KEY;
  }
}
```

### 2. Synchronous blocking code
**❌ Before:**
```typescript
const data = readFileSync('./data.json');
```

**✅ After:**
```typescript
const data = await env.MY_BUCKET.get('data.json');
```

### 3. Global state
**❌ Before:**
```typescript
let counter = 0; // Bad: shared across requests
app.get('/', () => counter++);
```

**✅ After:**
```typescript
// Use Durable Objects for state
export class Counter {
  state: DurableObjectState;

  constructor(state: DurableObjectState, env: Env) {
    this.state = state;
  }

  async fetch(request: Request) {
    const count = (await this.state.storage.get('count')) || 0;
    await this.state.storage.put('count', count + 1);
    return Response.json({ count });
  }
}
```

### 4. Large dependencies
**❌ Before:**
```typescript
import _ from 'lodash'; // 71KB
import moment from 'moment'; // 289KB
```

**✅ After:**
```typescript
// Use native methods
const unique = [...new Set(array)];
const formatted = new Date().toISOString();
```

### 5. Inefficient database queries
**❌ Before:**
```typescript
const users = await prisma.user.findMany();
const posts = await prisma.post.findMany();
```

**✅ After:**
```typescript
const db = drizzle(env.DB);
const usersWithPosts = await db
  .select()
  .from(users)
  .leftJoin(posts, eq(users.id, posts.userId));
```

## Optimization Checklist

- [ ] Remove Node.js-specific APIs (fs, child_process)
- [ ] Convert to ES modules format
- [ ] Replace traditional DB with D1
- [ ] Use R2 for file storage
- [ ] Implement proper error handling
- [ ] Add request validation
- [ ] Use env bindings instead of process.env
- [ ] Implement caching where appropriate
- [ ] Use ctx.waitUntil() for background tasks
- [ ] Remove unnecessary dependencies
- [ ] Test with `wrangler dev --local`
- [ ] Deploy and monitor performance

## Principles
- **Edge-first**: Design for distributed edge execution
- **Stateless by default**: Use Durable Objects only when needed
- **Minimal dependencies**: Keep bundle size small
- **Async everything**: No blocking operations
- **Type safety**: Use TypeScript with proper types

## Reference
- [Workers Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Migrate from Pages](https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/)
- [D1 with Drizzle](https://orm.drizzle.team/docs/get-started-sqlite#cloudflare-d1)
- [Workers Best Practices](https://developers.cloudflare.com/workers/platform/best-practices/)
