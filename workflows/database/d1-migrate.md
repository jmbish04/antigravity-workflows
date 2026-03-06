---
description: Manage D1 database migrations with Drizzle ORM
---

# D1 Database Migrations

I will help you create and run database migrations for Cloudflare D1 using Drizzle ORM and Wrangler.

## Guardrails
- Always use Drizzle ORM with D1 for type-safe queries
- Use `drizzle-kit generate` to create migrations from schema
- Use `wrangler d1 migrations apply` to apply migrations
- Test migrations locally before applying remotely
- Never run destructive migrations without backup/confirmation

## Steps

### 1. Understand the Change
Ask clarifying questions:
- What schema change is needed?
- Is this adding, modifying, or removing tables/columns?
- Any data that needs to be preserved/migrated?
- Is this for local development or production?

### 2. Analyze D1 Setup
Check existing configuration:
- Look for `wrangler.toml` or `wrangler.jsonc` with D1 bindings
- Check for `drizzle.config.ts`
- Look for existing schema in `src/db/schema.ts` or similar
- Check for `migrations/` directory

If D1 is not set up, create a database first:
```bash
npx wrangler d1 create <DATABASE_NAME>
```

### 3. Set Up Drizzle (If Not Configured)
Install dependencies:
```bash
npm install drizzle-orm better-sqlite3
npm install -D drizzle-kit @types/better-sqlite3
```

Create `drizzle.config.ts`:
```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
  dbCredentials: {
    accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
    databaseId: process.env.CLOUDFLARE_DATABASE_ID!,
    token: process.env.CLOUDFLARE_D1_TOKEN!,
  },
} satisfies Config;
```

### 4. Update Schema
Modify your Drizzle schema file:
```typescript
// src/db/schema.ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name'),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
});

export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  content: text('content'),
  userId: integer('user_id').references(() => users.id),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
});
```

### 5. Generate Migration
Generate SQL migration from schema changes:
```bash
npx drizzle-kit generate
```

This creates a migration file in `migrations/` like:
```
migrations/
└── 0001_add_posts_table.sql
```

Review the generated SQL to ensure it's correct.

### 6. Add Migration Scripts to package.json
Add convenient npm scripts:
```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate:local": "wrangler d1 migrations apply <DATABASE_NAME> --local",
    "db:migrate:remote": "wrangler d1 migrations apply <DATABASE_NAME> --remote",
    "db:studio": "drizzle-kit studio"
  }
}
```

### 7. Test Migration Locally
Apply migration to local D1 database:
```bash
# First time: create local database
npx wrangler d1 execute <DATABASE_NAME> --local --command "SELECT 1"

# Apply migrations locally
npm run db:migrate:local
```

Verify the migration:
```bash
npx wrangler d1 execute <DATABASE_NAME> --local --command "SELECT name FROM sqlite_master WHERE type='table'"
```

### 8. Apply to Production
After testing locally, apply to production:
```bash
npm run db:migrate:remote
```

**Important**: This will prompt for confirmation. Review changes carefully.

### 9. Update Wrangler Configuration
Ensure D1 binding is configured:
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
  ]
}
```

### 10. Use in Worker Code
Access D1 with Drizzle ORM:
```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/d1';
import * as schema from './schema';

export function getDb(env: Env) {
  return drizzle(env.DB, { schema });
}

// src/index.ts
import { getDb } from './db';
import { users } from './db/schema';

export default {
  async fetch(request, env) {
    const db = getDb(env);

    // Type-safe query
    const allUsers = await db.select().from(users);

    return Response.json(allUsers);
  }
}
```

## Common Migration Patterns

### Adding a New Table
```typescript
// schema.ts
export const products = sqliteTable('products', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  price: integer('price').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
});
```

Run: `npm run db:generate && npm run db:migrate:local`

### Adding a Column
```typescript
// schema.ts - add new column to existing table
export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name'),
  bio: text('bio'), // New column
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
});
```

### Adding Indexes
```typescript
import { index } from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
}, (table) => ({
  emailIdx: index('email_idx').on(table.email),
  createdAtIdx: index('created_at_idx').on(table.createdAt),
}));
```

### Foreign Keys
```typescript
export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  userId: integer('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  title: text('title').notNull(),
});
```

## Principles
- **Schema as source of truth**: Define schema in TypeScript, generate SQL
- **Type safety**: Drizzle provides full TypeScript types
- **Version control migrations**: Commit migration files to Git
- **Test locally first**: Always test with `--local` before `--remote`
- **Reversible when possible**: Consider rollback scenarios

## Workflow Summary
```bash
# 1. Update schema.ts with changes
# 2. Generate migration
npm run db:generate

# 3. Test locally
npm run db:migrate:local

# 4. Verify it works
npx wrangler dev

# 5. Apply to production
npm run db:migrate:remote
```

## Drizzle Studio
Explore your database with Drizzle Studio:
```bash
npm run db:studio
```
Opens a web interface to browse and query your D1 database.

## Reference
- [D1 Documentation](https://developers.cloudflare.com/d1/)
- [Drizzle ORM with D1](https://orm.drizzle.team/docs/get-started-sqlite#cloudflare-d1)
- [Wrangler D1 Commands](https://developers.cloudflare.com/workers/wrangler/commands/#d1)
- [D1 Migrations](https://developers.cloudflare.com/d1/reference/migrations/)
