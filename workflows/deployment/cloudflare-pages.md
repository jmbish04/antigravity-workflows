---
description: Deploy full-stack applications to Cloudflare Pages
---

# Cloudflare Pages

I will help you deploy a full-stack application to Cloudflare Pages with Functions.

## Guardrails
- Detect framework before suggesting build commands
- Use Pages Functions (in `/functions` directory) for server-side logic
- Configure bindings for D1, R2, KV access from Functions
- Leverage zero-config deployment for popular frameworks
- Use preview deployments for testing

## Steps

### 1. Understand Project Type
Ask clarifying questions:
- What framework are you using? (Next.js, React, Vue, Svelte, Astro, etc.)
- Do you need server-side logic? (API routes, SSR)
- Do you need database access? (D1, KV, R2)
- Is this a static site or full-stack app?
- Do you have a Git repository?

### 2. Analyze Project Structure
Detect existing setup:
- Check `package.json` for framework and build scripts
- Look for existing `functions/` directory
- Check for `wrangler.toml` or `wrangler.jsonc`
- Identify output directory (`dist/`, `build/`, `.next/`, etc.)

### 3. Prepare for Deployment
Ensure project is ready:
- Build passes locally: `npm run build`
- Identify correct build command and output directory
- Create `/functions` directory if server-side logic is needed
- Configure environment variables

### 4. Set Up Pages Functions (If Needed)
For server-side logic, create Functions:

**File-based routing** in `/functions`:
```
functions/
├── api/
│   ├── users.ts       # /api/users
│   └── posts/
│       └── [id].ts    # /api/posts/:id
└── _middleware.ts     # Runs before all functions
```

**Function example** (`functions/api/users.ts`):
```typescript
export async function onRequest(context) {
  const { request, env, params } = context;

  // Access D1 database
  const { results } = await env.DB.prepare(
    'SELECT * FROM users'
  ).all();

  return Response.json(results);
}
```

**HTTP method-specific handlers**:
```typescript
// functions/api/posts.ts
export async function onRequestGet(context) {
  // Handle GET
}

export async function onRequestPost(context) {
  // Handle POST
}
```

### 5. Configure Wrangler (If Using Bindings)
Create `wrangler.toml` or `wrangler.jsonc` for bindings:
```json
{
  "name": "my-pages-app",
  "pages_build_output_dir": "dist",
  "compatibility_date": "2024-01-01",
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database",
      "database_id": "<database-id>"
    }
  ]
}
```

### 6. Deploy via Git
**Recommended**: Connect Git repository
1. Push code to GitHub/GitLab
2. Go to Cloudflare Dashboard → Workers & Pages
3. Click **Create Application** → **Pages** → **Connect to Git**
4. Select repository and branch
5. Configure build settings:
   - **Build command**: `npm run build`
   - **Build output directory**: `dist` (or framework-specific)
6. Add environment variables if needed
7. Click **Save and Deploy**

**Auto-deploys**: Every push to main triggers deployment

### 7. Deploy via Wrangler CLI
**Alternative**: Direct deployment
```bash
# Build your project first
npm run build

# Deploy to Pages
npx wrangler pages deploy <BUILD_OUTPUT_DIR>

# Example for most frameworks
npx wrangler pages deploy dist
```

### 8. Configure Custom Domain (Optional)
Add custom domain:
1. Go to Pages project → **Custom domains**
2. Click **Set up a custom domain**
3. Enter your domain
4. Update DNS records as instructed
5. Wait for SSL certificate provisioning

### 9. Set Up Preview Deployments
Configure branch previews:
- **Production branch**: `main` or `master`
- **Preview branches**: All other branches get preview URLs
- Each PR gets a unique preview URL: `<branch>.<project>.pages.dev`

### 10. Verify Deployment
After deployment:
- Test the deployed application
- Check Functions are working: Test API endpoints
- Verify bindings (D1, KV, R2) are accessible
- Check preview deployments work
- Monitor logs in Cloudflare dashboard

## Framework-Specific Configurations

### Next.js
```json
{
  "name": "nextjs-app",
  "compatibility_flags": ["nodejs_compat"],
  "pages_build_output_dir": ".vercel/output/static"
}
```
Use `@cloudflare/next-on-pages` for deployment.

### React / Vite
```json
{
  "build": {
    "command": "npm run build",
    "directory": "dist"
  }
}
```

### Astro
```json
{
  "build": {
    "command": "npm run build",
    "directory": "dist"
  }
}
```

### SvelteKit
Use `@sveltejs/adapter-cloudflare` in `svelte.config.js`.

## Principles
- **Git-based deployments**: Automatic deploys on push
- **Preview everywhere**: Every branch gets a preview URL
- **Zero config**: Most frameworks work out of the box
- **Functions for APIs**: Use `/functions` directory for server logic
- **Bindings for data**: Access D1, KV, R2 via environment bindings

## Common Patterns

### API Route with D1
```typescript
// functions/api/users.ts
export async function onRequestGet({ env }) {
  const { results } = await env.DB.prepare(
    'SELECT * FROM users LIMIT 10'
  ).all();

  return Response.json(results);
}
```

### Middleware for Auth
```typescript
// functions/_middleware.ts
export async function onRequest(context) {
  const { request, next, env } = context;

  // Check authentication
  const token = request.headers.get('Authorization');
  if (!token) {
    return new Response('Unauthorized', { status: 401 });
  }

  // Continue to next function
  return next();
}
```

### CORS Headers
```typescript
// functions/_middleware.ts
export async function onRequest(context) {
  const response = await context.next();

  response.headers.set('Access-Control-Allow-Origin', '*');
  response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');

  return response;
}
```

## Reference
- [Cloudflare Pages Docs](https://developers.cloudflare.com/pages/)
- [Pages Functions](https://developers.cloudflare.com/pages/functions/)
- [Supported Frameworks](https://developers.cloudflare.com/pages/framework-guides/)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/)
