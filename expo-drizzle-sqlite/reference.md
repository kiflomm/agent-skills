# Expo Drizzle SQLite — Reference

Skeletons and patterns distilled from a production Expo finance app. Adapt names/paths to the target project.

## drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './db/schema.ts',
  out: './drizzle',
  dialect: 'sqlite',
  driver: 'expo',
});
```

`driver: 'expo'` is what makes kit output Expo-compatible. Do not point this config at `better-sqlite3` for the mobile app.

## Minimal schema

```ts
// db/schema.ts
import { sqliteTable, text, integer, real, index } from 'drizzle-orm/sqlite-core';

export const items = sqliteTable(
  'items',
  {
    id: text('id').primaryKey(),
    name: text('name').notNull(),
    quantity: real('quantity').notNull().default(0),
    deleted: integer('deleted', { mode: 'boolean' }).notNull().default(false),
    officeId: integer('office_id').notNull(),
    createdAt: text('created_at').notNull(),
    updatedAt: text('updated_at').notNull(),
  },
  (t) => [index('idx_items_deleted_office_id').on(t.deleted, t.officeId)]
);

export type Item = typeof items.$inferSelect;
export type NewItem = typeof items.$inferInsert;
```

## db.ts (open → drizzle → migrate)

Minimal:

```ts
import { drizzle } from 'drizzle-orm/expo-sqlite';
import * as SQLite from 'expo-sqlite';
import { migrate } from 'drizzle-orm/expo-sqlite/migrator';
import migrations from './drizzle/migrations.js';

const expoDb = SQLite.openDatabaseSync('app.db', {
  enableChangeListener: true,
});

export const db = drizzle(expoDb);

try {
  migrate(db, migrations);
} catch (e) {
  console.error('SQLite migrations failed:', e);
}
```

### Reconnect Proxy (optional)

Use when the app deletes the DB file and must reopen without stale imports:

```ts
import { drizzle } from 'drizzle-orm/expo-sqlite';
import * as SQLite from 'expo-sqlite';
import * as FileSystem from 'expo-file-system';
import { migrate } from 'drizzle-orm/expo-sqlite/migrator';
import migrations from './drizzle/migrations.js';

let expoDb: SQLite.SQLiteDatabase | null = null;
let dbInstance: ReturnType<typeof drizzle> | null = null;

const initializeDatabase = () => {
  if (expoDb) {
    try {
      expoDb.closeSync();
    } catch (e) {
      console.warn('Error closing database:', e);
    }
  }

  expoDb = SQLite.openDatabaseSync('app.db', { enableChangeListener: true });
  dbInstance = drizzle(expoDb);

  try {
    migrate(dbInstance, migrations);
  } catch (e) {
    console.error('SQLite migrations failed:', e);
  }

  return dbInstance;
};

initializeDatabase();

export const db = new Proxy({} as ReturnType<typeof drizzle>, {
  get(_target, prop) {
    if (!dbInstance) initializeDatabase();
    if (!dbInstance) throw new Error('Failed to initialize database');
    const value = (dbInstance as any)[prop];
    return typeof value === 'function' ? value.bind(dbInstance) : value;
  },
});

export const reinitializeDatabase = () => initializeDatabase();

export const deleteDatabase = async (dbName: string) => {
  const dbPath = `${FileSystem.documentDirectory}SQLite/${dbName}`;
  if (expoDb) {
    const old = expoDb;
    expoDb = null;
    dbInstance = null;
    try {
      old.closeSync();
    } catch (e) {
      console.warn(e);
    }
    await new Promise((r) => setTimeout(r, 100));
  }
  const info = await FileSystem.getInfoAsync(dbPath);
  if (info.exists) {
    await FileSystem.deleteAsync(dbPath, { idempotent: true });
  }
  initializeDatabase();
};
```

## migrations.js (first migration)

After `drizzle-kit generate` creates `0000_….sql` and `meta/_journal.json`:

```js
// drizzle/migrations.js
// https://orm.drizzle.team/docs/get-started/expo-new

import journal from './meta/_journal.json';
import m0000 from './0000_example.sql';

export default {
  journal,
  migrations: {
    m0000,
  },
};
```

When generate adds `0001_….sql`, import `m0001` and add it to `migrations`. Keep keys aligned with journal order (`m0000`, `m0001`, …).

## Babel + Metro

```js
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [['inline-import', { extensions: ['.sql'] }]],
  };
};
```

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);
config.resolver.sourceExts.push('sql');

module.exports = config;
```

## Repo query examples

```ts
import { eq, inArray } from 'drizzle-orm';
import { db } from '@/db';
import { items, type NewItem } from '@/db/schema';

export async function getItemById(id: string) {
  return db.select().from(items).where(eq(items.id, id)).get();
}

export async function listItemsByOffice(officeId: number) {
  return db.select().from(items).where(eq(items.officeId, officeId)).all();
}

export async function insertItem(row: NewItem) {
  await db.insert(items).values(row).run();
}

export async function upsertItems(
  list: Array<Partial<typeof items.$inferSelect> & { id: string }>
) {
  if (!list.length) return;

  const ids = list.map((i) => i.id);

  await db.transaction(async (tx) => {
    const existing = await tx
      .select()
      .from(items)
      .where(inArray(items.id, ids))
      .all();

    const byId = new Map(existing.map((r) => [r.id, r]));

    for (const item of list) {
      const current = byId.get(item.id);
      if (!current) {
        await tx.insert(items).values(item as NewItem).run();
      } else {
        await tx
          .update(items)
          .set({ ...item, updatedAt: new Date().toISOString() })
          .where(eq(items.id, item.id))
          .run();
      }
    }
  });
}
```

## Drizzle Studio

```ts
// app/_layout.tsx (or root layout)
import { useDrizzleStudio } from 'expo-drizzle-studio-plugin';
import { db } from '@/db';

export default function RootLayout() {
  useDrizzleStudio(db.$client);
  // ...
}
```

Requires a dev client / environment that supports the plugin.

## Gotchas

| Issue | Fix |
|-------|-----|
| `Cannot import .sql` / Metro resolve error | Add `sourceExts.push('sql')` and Babel `inline-import` for `.sql` |
| Migrations never apply on device | Wire new SQL into `migrations.js` and call `migrate(db, migrations)` at init |
| Wrong driver / kit output | `drizzle.config.ts` must use `driver: 'expo'` and `dialect: 'sqlite'` |
| Using `better-sqlite3` in app code | Only use `drizzle-orm/expo-sqlite` + `expo-sqlite` at runtime |
| Stale connection after deleting DB file | Close sync, null refs, delete file, re-open (Proxy pattern above) |
| Hand-edited migration SQL drifts from schema | Edit schema only; user runs `drizzle:generate`; then update `migrations.js` imports |
| Forgetting `.all()` / `.get()` / `.run()` | Expo SQLite driver expects these terminators on query builders |
| `enableChangeListener: true` | Needed if you rely on change listeners / some Studio features |

## Package.json scripts

```json
{
  "scripts": {
    "drizzle:generate": "drizzle-kit generate",
    "drizzle:push": "drizzle-kit push"
  }
}
```

Prefer **generate + runtime migrate** for shipped Expo apps. `push` is a shortcut for local prototyping, not a substitute for the `migrations.js` bundle on devices.
