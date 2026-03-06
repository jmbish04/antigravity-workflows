---
description: Design D1 database schemas with Drizzle ORM
---

# D1 Database Schema Design

I will help you design database schemas for Cloudflare D1 using Drizzle ORM.

## Guardrails
- Always use Drizzle ORM for type-safe schema definitions
- Design for SQLite (D1 is built on SQLite)
- Consider D1 limitations (no ALTER COLUMN, limited constraints)
- Use integer timestamps (SQLite has no native datetime)
- Plan for edge deployment (optimize for read-heavy workloads)

## Steps

### 1. Understand Requirements
Ask clarifying questions:
- What entities/tables are needed?
- What are the relationships between entities?
- What queries will be most common?
- What's the expected data volume?
- Any specific performance requirements?

### 2. Set Up D1 and Drizzle
If not already configured:

**Create D1 database**:
```bash
npx wrangler d1 create <DATABASE_NAME>
```

**Install Drizzle**:
```bash
npm install drizzle-orm
npm install -D drizzle-kit
```

**Configure wrangler.jsonc**:
```json
{
  "name": "my-worker",
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database",
      "database_id": "<database-id>"
    }
  ]
}
```

### 3. Create Schema File
Create `src/db/schema.ts` with Drizzle schema:

```typescript
import { sqliteTable, text, integer, index } from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  bio: text('bio'),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
}, (table) => ({
  emailIdx: index('email_idx').on(table.email),
}));
```

### 4. Design Tables
For each entity, define:
- **Primary key**: Usually `integer().primaryKey({ autoIncrement: true })`
- **Fields**: Use appropriate SQLite types (text, integer, real, blob)
- **Constraints**: NOT NULL, UNIQUE where needed
- **Timestamps**: Use integer timestamps, not text
- **Indexes**: Add indexes on frequently queried columns

### 5. Define Relationships
Establish connections between tables:

**One-to-many**:
```typescript
export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
});

export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  content: text('content'),
  userId: integer('user_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
});

// Define relations for type-safe joins
import { relations } from 'drizzle-orm';

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  user: one(users, {
    fields: [posts.userId],
    references: [users.id],
  }),
}));
```

**Many-to-many** (junction table):
```typescript
export const tags = sqliteTable('tags', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull().unique(),
});

export const postTags = sqliteTable('post_tags', {
  postId: integer('post_id')
    .notNull()
    .references(() => posts.id, { onDelete: 'cascade' }),
  tagId: integer('tag_id')
    .notNull()
    .references(() => tags.id, { onDelete: 'cascade' }),
}, (table) => ({
  pk: primaryKey({ columns: [table.postId, table.tagId] }),
}));
```

### 6. Add Indexes
Index columns used in WHERE, JOIN, ORDER BY:
```typescript
export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  userId: integer('user_id').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
  status: text('status', { enum: ['draft', 'published'] }).notNull().default('draft'),
}, (table) => ({
  userIdIdx: index('user_id_idx').on(table.userId),
  createdAtIdx: index('created_at_idx').on(table.createdAt),
  statusIdx: index('status_idx').on(table.status),
}));
```

### 7. Handle Common Patterns
**Soft deletes**:
```typescript
export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  deletedAt: integer('deleted_at', { mode: 'timestamp' }),
});
```

**Enums** (use text with enum):
```typescript
export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  status: text('status', { enum: ['draft', 'published', 'archived'] }).notNull(),
});
```

**JSON data**:
```typescript
export const settings = sqliteTable('settings', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  userId: integer('user_id').notNull(),
  preferences: text('preferences', { mode: 'json' }).$type<{
    theme: 'light' | 'dark';
    notifications: boolean;
  }>(),
});
```

### 8. Create Drizzle Config
Create `drizzle.config.ts`:
```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
} satisfies Config;
```

### 9. Generate Initial Migration
```bash
npx drizzle-kit generate
```

This creates SQL migration files from your schema.

### 10. Apply Schema
```bash
# Test locally
npx wrangler d1 migrations apply <DATABASE_NAME> --local

# Apply to production
npx wrangler d1 migrations apply <DATABASE_NAME> --remote
```

## D1-Specific Considerations

### SQLite Type Mapping
| TypeScript | SQLite | Drizzle |
|------------|--------|---------|
| string | TEXT | `text()` |
| number | INTEGER | `integer()` |
| number (decimal) | REAL | `real()` |
| boolean | INTEGER (0/1) | `integer('col', { mode: 'boolean' })` |
| Date | INTEGER (timestamp) | `integer('col', { mode: 'timestamp' })` |
| JSON | TEXT | `text('col', { mode: 'json' })` |
| Buffer | BLOB | `blob()` |

### Limitations
- No ALTER COLUMN (must recreate table)
- No concurrent writes to same database (use sharding for high write volume)
- 100MB database size limit (per database, create multiple if needed)
- Foreign keys are enforced

### Optimization Tips
- **Denormalize for reads**: D1 is read-optimized, denormalize when needed
- **Use indexes wisely**: Index foreign keys and frequently filtered columns
- **Batch operations**: Group multiple queries in transactions
- **Cache aggressively**: Use Workers KV for hot data

## Example: E-commerce Schema

```typescript
import { sqliteTable, text, integer, real, index } from 'drizzle-orm/sqlite-core';
import { relations } from 'drizzle-orm';

// Users
export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
}, (table) => ({
  emailIdx: index('users_email_idx').on(table.email),
}));

// Products
export const products = sqliteTable('products', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  description: text('description'),
  price: integer('price').notNull(), // Store in cents
  stock: integer('stock').notNull().default(0),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
}, (table) => ({
  nameIdx: index('products_name_idx').on(table.name),
}));

// Orders
export const orders = sqliteTable('orders', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  userId: integer('user_id')
    .notNull()
    .references(() => users.id),
  status: text('status', { enum: ['pending', 'paid', 'shipped', 'delivered'] }).notNull(),
  total: integer('total').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
}, (table) => ({
  userIdIdx: index('orders_user_id_idx').on(table.userId),
  statusIdx: index('orders_status_idx').on(table.status),
}));

// Order Items
export const orderItems = sqliteTable('order_items', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  orderId: integer('order_id')
    .notNull()
    .references(() => orders.id, { onDelete: 'cascade' }),
  productId: integer('product_id')
    .notNull()
    .references(() => products.id),
  quantity: integer('quantity').notNull(),
  price: integer('price').notNull(), // Price at time of purchase
}, (table) => ({
  orderIdIdx: index('order_items_order_id_idx').on(table.orderId),
  productIdIdx: index('order_items_product_id_idx').on(table.productId),
}));

// Relations
export const usersRelations = relations(users, ({ many }) => ({
  orders: many(orders),
}));

export const ordersRelations = relations(orders, ({ one, many }) => ({
  user: one(users, { fields: [orders.userId], references: [users.id] }),
  items: many(orderItems),
}));

export const orderItemsRelations = relations(orderItems, ({ one }) => ({
  order: one(orders, { fields: [orderItems.orderId], references: [orders.id] }),
  product: one(products, { fields: [orderItems.productId], references: [products.id] }),
}));
```

## Principles
- **Type safety first**: Let Drizzle generate types from schema
- **Index strategically**: Index foreign keys and filter columns
- **Use integer timestamps**: More efficient than text dates
- **Plan for scale**: Consider sharding for write-heavy apps
- **Denormalize when needed**: Optimize for read performance

## Reference
- [Drizzle ORM with D1](https://orm.drizzle.team/docs/get-started-sqlite#cloudflare-d1)
- [D1 Documentation](https://developers.cloudflare.com/d1/)
- [SQLite Data Types](https://www.sqlite.org/datatype3.html)
- [D1 Limits](https://developers.cloudflare.com/d1/platform/limits/)
