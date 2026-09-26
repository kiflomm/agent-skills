---

name: expo-drizzle-sqlite
description: >-
  Wire Drizzle ORM to Expo SQLite for local persistence in Expo/React Native
  apps. Use when setting up or changing Drizzle with expo-sqlite, drizzle-kit
  with driver expo, local SQLite migrations, drizzle/migrations.js, or
  Expo local database integration.
  
---

# Expo + Drizzle + SQLite

Integrate Drizzle ORM with Expo SQLite for on-device storage. Follow this stack end-to-end; do not use Node SQLite drivers (`better-sqlite3`, `libsql`) in the app runtime.

## Checklist

```
- [ ] Packages installed
- [ ] expo-sqlite in app config plugins
- [ ] Metro + Babel allow .sql imports
- [ ] drizzle.config.ts with dialect sqlite + driver expo
- [ ] Schema in drizzle-orm/sqlite-core
- [ ] db client: openDatabaseSync → drizzle → migrate
- [ ] drizzle/migrations.js bundles journal + SQL
- [ ] npm scripts for generate (user runs them)
- [ ] Queries use .all() / .get() / .run()
```

## 1. Install

```bash
npx expo install expo-sqlite
npm install drizzle-orm babel-plugin-inline-import
npm install -D drizzle-kit
# optional DevTools
npm install expo-drizzle-studio-plugin
```

## 2. Expo plugin

In `app.json` / `app.config.*`, add `"expo-sqlite"` to `expo.plugins`.

## 3. Metro + Babel (required for migrations)

**Metro** — treat `.sql` as source:

```js
const config = getDefaultConfig(__dirname);
config.resolver.sourceExts.push('sql');
```

**Babel** — inline `.sql` file contents:

```js
plugins: [["inline-import", { extensions: [".sql"] }]]
```

Without both, `import './0000_….sql'` in `migrations.js` fails under Metro.

## 4. drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './db/schema.ts',
  out: './drizzle',
  dialect: 'sqlite',
  driver: 'expo', // required for Expo
});
```

## 5. Schema

Use `drizzle-orm/sqlite-core`. Prefer `text` PKs and ISO date strings when matching typical Expo apps. Booleans: `integer('flag', { mode: 'boolean' })`.

Export inferred types:

```ts
export type Item = typeof items.$inferSelect;
export type NewItem = typeof items.$inferInsert;
```

See [reference.md](reference.md) for a minimal table example.

## 6. Client (`db.ts`)

1. `SQLite.openDatabaseSync('app.db', { enableChangeListener: true })`
2. `drizzle(expoDb)` from `drizzle-orm/expo-sqlite`
3. `migrate(db, migrations)` from `drizzle-orm/expo-sqlite/migrator` with `./drizzle/migrations.js`
4. Export `db` for repos/screens

Run migrate on init (module load or app bootstrap). Full skeleton (including optional reconnect Proxy): [reference.md](reference.md).

## 7. Expo migrations bundle

After schema changes, the user runs generate (e.g. `npm run drizzle:generate`). That writes `drizzle/*.sql` and `drizzle/meta/`.

Expo cannot apply folder-based migrations like Node. Keep `drizzle/migrations.js` that:

1. Imports `./meta/_journal.json`
2. Imports each `./NNNN_name.sql`
3. Exports `{ journal, migrations: { m0000, m0001, … } }`

When a new SQL file appears after generate, **add its import and map entry** to `migrations.js`. Do not invent or hand-edit SQL/meta contents.

Official pattern: https://orm.drizzle.team/docs/get-started/expo-new

## 8. Scripts and agent rules

```json
"drizzle:generate": "drizzle-kit generate",
"drizzle:push": "drizzle-kit push"
```

**Agents:**

- Edit schema files only (`db/schema.ts` or project equivalent).
- Do **not** create/edit/rename/delete files under `drizzle/` SQL, `drizzle/meta/`, or `_journal.json`.
- Do **not** run `drizzle-kit generate` unless the user explicitly asks.
- After schema edits: tell the user to run generate, then update `migrations.js` imports if the project uses that bundle (or do the `migrations.js` wiring if they ask you to finish setup).

## 9. Query style

Prefer a thin repo layer over querying from UI.

With the Expo driver, terminate statements explicitly:

| Intent | Method |
|--------|--------|
| Many rows | `.all()` |
| One row | `.get()` |
| Insert/update/delete | `.run()` |

Use `db.transaction(async (tx) => { … })` for multi-step writes. Examples: [reference.md](reference.md).

## 10. Optional DevTools

```ts
import { useDrizzleStudio } from 'expo-drizzle-studio-plugin';
useDrizzleStudio(db.$client);
```

## Additional resources

- Copy-paste skeletons, repo examples, gotchas: [reference.md](reference.md)
