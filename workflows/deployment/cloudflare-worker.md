---
description: Create and deploy Cloudflare Workers with best practices
---

# Cloudflare Worker

I will help you create and deploy a Cloudflare Worker optimized for the Workers runtime.

## Guardrails
- Always use ES modules format (not Service Worker syntax)
- Use wrangler.toml or wrangler.jsonc for configuration
- Leverage bindings for D1, R2, KV, and other Cloudflare resources
- Keep Workers stateless - use Durable Objects for state
- Follow Workers runtime constraints (CPU time, memory limits)

## Steps

### 1. Understand Requirements
Ask clarifying questions:
- What does this Worker need to do? (API, webhook, proxy, edge logic)
- Does it need database access? (D1, KV, R2)
- Does it need to be scheduled? (Cron Triggers)
- What are the performance requirements?
- Does it need authentication/authorization?

### 2. Analyze Existing Setup
Check for existing configuration:
- Look for `wrangler.toml` or `wrangler.jsonc`
- Check for existing Workers in the project
- Identify existing bindings (D1, KV, R2, etc.)
- Check `package.json` for dependencies

If no wrangler config exists, create one.

### 3. Create Worker Structure
Use ES modules format with fetch handler:
```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Worker logic here
    return new Response('Hello World');
  }
}
```

**Key patterns:**
- Use `env` parameter to access bindings (D1, KV, R2)
- Use `ctx.waitUntil()` for background tasks
- Use `ctx.passThroughOnException()` for failover
- Return `Response` objects with proper status codes

### 4. Configure Bindings
Add necessary bindings to wrangler configuration:
- **D1 databases**: For SQL queries
- **KV namespaces**: For key-value storage
- **R2 buckets**: For object storage
- **Environment variables**: For configuration
- **Secrets**: For API keys (never hardcode)

Example wrangler.jsonc:
```json
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-01-01",
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database",
      "database_id": "<database-id>"
    }
  ],
  "vars": {
    "ENVIRONMENT": "production"
  }
}
```

### 5. Implement Request Handling
Follow these patterns:
- **Routing**: Use URL pattern matching or libraries like `itty-router`
- **Validation**: Validate all inputs before processing
- **Error handling**: Catch errors and return appropriate status codes
- **CORS**: Add CORS headers if needed for browser access
- **Rate limiting**: Use KV or Durable Objects for rate limiting

### 6. Optimize for Edge
Workers-specific optimizations:
- Use `fetch()` for subrequests (max 50 per request in free tier)
- Cache responses using Cache API where appropriate
- Minimize cold start time (keep dependencies small)
- Use `waitUntil()` for non-blocking operations
- Stream responses for large payloads

### 7. Test Locally
Test your Worker:
```bash
npx wrangler dev
```
- Test all endpoints
- Verify bindings work correctly
- Check error handling
- Test with different request methods

### 8. Deploy
Deploy to Cloudflare:
```bash
# Deploy to production
npx wrangler deploy

# Deploy to a specific environment
npx wrangler deploy --env staging
```

### 9. Verify Deployment
After deployment:
- Test the deployed Worker URL
- Check logs: `npx wrangler tail`
- Monitor performance in Cloudflare dashboard
- Verify bindings are connected

## Principles
- **Serverless first**: No servers to manage, instant global deployment
- **Use bindings**: Never hardcode credentials, use Cloudflare bindings
- **Fast response times**: Workers run at the edge, keep latency low
- **Fail gracefully**: Return helpful error messages with correct status codes
- **Secure by default**: Validate inputs, use secrets for sensitive data

## Common Patterns

### REST API
```typescript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    if (url.pathname === '/api/users' && request.method === 'GET') {
      // Handle GET /api/users
      const users = await env.DB.prepare('SELECT * FROM users').all();
      return Response.json(users.results);
    }

    return new Response('Not found', { status: 404 });
  }
}
```

### With D1 Database
```typescript
export default {
  async fetch(request, env) {
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE id = ?'
    ).bind(1).all();

    return Response.json(results);
  }
}
```

### With KV Storage
```typescript
export default {
  async fetch(request, env) {
    const value = await env.MY_KV.get('key');
    await env.MY_KV.put('key', 'value', { expirationTtl: 3600 });

    return Response.json({ value });
  }
}
```

## Reference
- [Cloudflare Workers Docs](https://developers.cloudflare.com/workers/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)
- [Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/)
