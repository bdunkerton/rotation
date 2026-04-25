# Rotation v1 — Build Runbook

> Bare-bones backend-first build plan for the Rotation meal planning PWA.
> Goal: get a working "add an ingredient, see it persist across reloads" milestone, with one Vitest test and the foundations in place for every future feature.

---

## Agent prompt (paste at the start of a new session)

> I'm building **Rotation**, a single-user meal planning PWA. The architecture is locked: **SvelteKit + Svelte 5 + TypeScript**, **SQLite WebAssembly** with **Origin Private File System (OPFS)** for storage, **Open Props + scoped CSS** for styling, **adapter-static in SPA mode** with **@vite-pwa/sveltekit**, **hand-rolled query stores** with **implicit invalidation**, **Vitest** for unit tests and **Playwright** for end-to-end. There's no backend — everything runs in the browser.
>
> The schema has these tables: `ingredients`, `stores`, `recipes`, `recipe_ingredients`, `shopping_list_items`, `cook_history`, `migrations`. Ingredients have three modes: `counted`, `self_calibrating`, `untracked`. Time is intentionally absent from the planning model — there's no weeks table, shop plans are ephemeral. Cook history records immutable snapshots.
>
> I'm following a 13-step build runbook. **I'm currently on step [N]: [step title]**. Here's what I've tried / what's not working: [describe].
>
> Help me debug or move forward. Don't redesign the architecture — it's locked.

---

## Before you start

**What "done" means for this runbook:** the app boots, you can add an ingredient via a form, the list of ingredients renders, and after a page reload the ingredients are still there. One Vitest test exercises the ingredient query module. That's the milestone. No recipes, no shopping list, no cooking — those come later.

**Time estimate:** one to two focused days the first time through. Most of that is the SQLite-WebAssembly + cross-origin headers setup, which is genuinely fiddly. Steps after that are mechanical.

**Suggested checkpoints:** stop at step 5 once the database initializes (even with no tables yet). Stop again at step 9 once you have a query module returning real data. Push through 10-13 in one sitting if you can — that's where it becomes an app.

---

## The 13 steps

### 1. Scaffold the SvelteKit project

Run the SvelteKit CLI and accept the TypeScript-flavored defaults.

```bash
pnpm dlx sv create rotation
cd rotation
pnpm install
```

When prompted, pick:
- **Skeleton project** (not demo)
- **TypeScript: Yes**
- **ESLint: Yes**
- **Prettier: Yes**
- **Vitest: Yes**
- **Playwright: Yes**

**Verify:** `pnpm dev` starts a dev server on `localhost:5173` and you see the empty SvelteKit welcome page.

**Docs:**
- [SvelteKit — Creating a project](https://svelte.dev/docs/kit/creating-a-project)
- [Svelte 5 docs](https://svelte.dev/docs/svelte/overview) — note: a lot of older tutorials show Svelte 4 (`$:`, stores). You want Svelte 5 (runes: `$state`, `$derived`, `$effect`).

---

### 2. Configure cross-origin isolation headers

OPFS requires the page to be **cross-origin isolated**, which means two specific HTTP headers must be set:

- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Embedder-Policy: require-corp`

Without these, OPFS access fails silently and you'll waste hours debugging.

In `vite.config.ts`, add a server config:

```ts
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],
  server: {
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    },
  },
});
```

**Note for production:** these headers also need to be set by whatever hosts your static files. Most static hosts (Netlify, Cloudflare Pages) let you configure custom headers via a config file. We'll deal with this when we deploy.

**Verify:** restart `pnpm dev`, open DevTools → Network → click on the document request → Headers tab → confirm both headers are present in the response.

**Docs:**
- [MDN — Cross-Origin-Opener-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy)
- [MDN — Cross-Origin-Embedder-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy)
- [web.dev — Cross-origin isolation explained](https://web.dev/articles/why-coop-coep)

---

### 3. Install SQLite WebAssembly

```bash
pnpm add @sqlite.org/sqlite-wasm
```

This package is a little awkward. It loads its own `.wasm` binary at runtime, which means Vite needs to know how to serve it. The package documentation has the canonical setup; expect to follow it carefully.

The OPFS persistence works through a feature called the **OPFS SyncAccess Handle**, which only runs in a Web Worker, not the main thread. The library handles this for you, but it means SQLite operations are async from your code's perspective.

**Verify:** create a temporary `src/routes/+page.svelte` that imports the library and logs the SQLite version:

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import sqlite3InitModule from '@sqlite.org/sqlite-wasm';

  let version = $state('loading...');

  onMount(async () => {
    const sqlite3 = await sqlite3InitModule();
    version = sqlite3.version.libVersion;
  });
</script>

<p>SQLite version: {version}</p>
```

If you see a version number, the library is loading. If you see errors about cross-origin isolation, revisit step 2.

**Docs:**
- [SQLite WebAssembly — Getting Started](https://sqlite.org/wasm/doc/trunk/index.md)
- [SQLite WebAssembly — Persistent Storage Options](https://sqlite.org/wasm/doc/trunk/persistence.md) (covers OPFS specifics)

---

### 4. Install Open Props

```bash
pnpm add open-props
```

Import the parts you want in your root layout. Create `src/routes/+layout.svelte`:

```svelte
<script lang="ts">
  import 'open-props/style';
  import 'open-props/normalize';
  let { children } = $props();
</script>

{@render children()}
```

**Verify:** in any component, use a CSS variable like `var(--size-4)` or `var(--gray-7)` and confirm it applies.

**Docs:**
- [Open Props — Getting Started](https://open-props.style/#getting-started)
- [Open Props — Browse all properties](https://open-props.style/) (use the sidebar to explore tokens)

---

### 5. Build the database initialization module

Create `src/lib/db/index.ts`. This module is responsible for:

- Initializing SQLite WebAssembly
- Opening the OPFS-backed database
- Running migrations
- Exporting a singleton database handle

Sketch:

```ts
import sqlite3InitModule from '@sqlite.org/sqlite-wasm';
import { runMigrations } from './migrations';

let dbInstance: any = null;
let initPromise: Promise<any> | null = null;

export async function getDb() {
  if (dbInstance) return dbInstance;
  if (initPromise) return initPromise;

  initPromise = (async () => {
    const sqlite3 = await sqlite3InitModule();
    // OPFS-backed database; the file is named "rotation.sqlite3" in OPFS
    const db = new sqlite3.oo1.OpfsDb('rotation.sqlite3');
    await runMigrations(db);
    dbInstance = db;
    return db;
  })();

  return initPromise;
}
```

The `OpfsDb` constructor only works in a Web Worker context. Depending on your sqlite-wasm version, you may need to wire this through a worker — check the docs. Some setups use the `Worker1` promiser API instead.

Because initialization is async, **everything that queries the database needs to await `getDb()` first**. We'll handle this cleanly in the layout in step 11.

**Verify:** call `getDb()` from a component on mount, check the browser's DevTools → Application → Storage → Origin Private File System for a `rotation.sqlite3` file. (Visibility of OPFS in DevTools varies by browser. In Chrome it's under Application → Storage. In Firefox it's harder to inspect.)

**Docs:**
- [SQLite WebAssembly — OO1 API](https://sqlite.org/wasm/doc/trunk/api-oo1.md)
- [SQLite WebAssembly — Persistent Storage Options](https://sqlite.org/wasm/doc/trunk/persistence.md)

---

### 6. Build the migration runner

Create `src/lib/db/migrations.ts`. The runner:

- Ensures a `migrations` table exists
- Reads applied migration versions
- Runs missing migrations in order, each in its own transaction
- Records each successful migration

Sketch:

```ts
type Migration = {
  version: number;
  name: string;
  up: (db: any) => void;
};

import { migration001 } from './migrations/001-initial-schema';

const allMigrations: Migration[] = [migration001];

export async function runMigrations(db: any) {
  db.exec(`
    CREATE TABLE IF NOT EXISTS migrations (
      version INTEGER PRIMARY KEY,
      name TEXT NOT NULL,
      applied_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
  `);

  const applied = new Set<number>();
  db.exec({
    sql: 'SELECT version FROM migrations',
    rowMode: 'object',
    callback: (row: any) => applied.add(row.version),
  });

  const pending = allMigrations
    .filter(m => !applied.has(m.version))
    .sort((a, b) => a.version - b.version);

  for (const migration of pending) {
    db.exec('BEGIN');
    try {
      migration.up(db);
      db.exec({
        sql: 'INSERT INTO migrations (version, name) VALUES (?, ?)',
        bind: [migration.version, migration.name],
      });
      db.exec('COMMIT');
    } catch (err) {
      db.exec('ROLLBACK');
      throw new Error(`Migration ${migration.version} failed: ${err}`);
    }
  }
}
```

The exact API for `db.exec` depends on the sqlite-wasm version — check the docs for current method signatures.

**Verify:** add a `console.log` in the runner that prints which migrations ran. Reload the page once and you should see migration 001 run. Reload again — it should not run a second time.

**Docs:**
- [SQLite — Transactions](https://www.sqlite.org/lang_transaction.html)

---

### 7. Write migration 001 (initial schema)

Create `src/lib/db/migrations/001-initial-schema.ts`. This is the full v1 schema.

```ts
export const migration001 = {
  version: 1,
  name: 'initial-schema',
  up: (db: any) => {
    db.exec(`
      CREATE TABLE stores (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL UNIQUE
      );

      CREATE TABLE ingredients (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL UNIQUE,
        mode TEXT NOT NULL CHECK (mode IN ('counted', 'self_calibrating', 'untracked')),
        uses_per_unit INTEGER,
        uses_remaining INTEGER,
        default_store_id INTEGER REFERENCES stores(id),
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        archived_at TEXT
      );

      CREATE TABLE recipes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        notes TEXT,
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        archived_at TEXT
      );

      CREATE TABLE recipe_ingredients (
        recipe_id INTEGER NOT NULL REFERENCES recipes(id) ON DELETE CASCADE,
        ingredient_id INTEGER NOT NULL REFERENCES ingredients(id),
        uses_required INTEGER NOT NULL DEFAULT 1,
        PRIMARY KEY (recipe_id, ingredient_id)
      );

      CREATE TABLE shopping_list_items (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        label TEXT NOT NULL,
        ingredient_id INTEGER REFERENCES ingredients(id),
        units_to_buy INTEGER,
        store_id INTEGER REFERENCES stores(id),
        checked INTEGER NOT NULL DEFAULT 0
      );

      CREATE TABLE cook_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        recipe_id INTEGER REFERENCES recipes(id),
        cooked_at TEXT NOT NULL DEFAULT (datetime('now')),
        uses_consumed_json TEXT NOT NULL
      );
    `);
  },
};
```

A few notes:

- SQLite uses `INTEGER` for booleans — `checked` is `0` or `1`.
- `ON DELETE CASCADE` on `recipe_ingredients` means deleting a recipe wipes its ingredient links. The reverse (deleting an ingredient) doesn't cascade; we'll handle that with archive logic later.
- The `mode` CHECK constraint enforces the three valid values at the database level.

**Verify:** after a fresh load, query the schema directly. In a temporary component:

```ts
const tables = [];
db.exec({
  sql: "SELECT name FROM sqlite_master WHERE type='table'",
  rowMode: 'object',
  callback: (row: any) => tables.push(row.name),
});
console.log(tables);
```

You should see all seven tables (six from the migration plus the `migrations` table).

**Docs:**
- [SQLite — Data Types](https://www.sqlite.org/datatype3.html)
- [SQLite — CREATE TABLE](https://www.sqlite.org/lang_createtable.html)
- [SQLite — Foreign Keys](https://www.sqlite.org/foreignkeys.html) (note: foreign key enforcement is **off by default** in SQLite — for now it doesn't matter, but remember this for later)

---

### 8. Build the ingredients query module

Create `src/lib/db/ingredients.ts`. Plain functions over the database.

```ts
import { getDb } from './index';

export type IngredientMode = 'counted' | 'self_calibrating' | 'untracked';

export type Ingredient = {
  id: number;
  name: string;
  mode: IngredientMode;
  uses_per_unit: number | null;
  uses_remaining: number | null;
  default_store_id: number | null;
  created_at: string;
  archived_at: string | null;
};

export async function getAllIngredients(): Promise<Ingredient[]> {
  const db = await getDb();
  const rows: Ingredient[] = [];
  db.exec({
    sql: 'SELECT * FROM ingredients WHERE archived_at IS NULL ORDER BY name',
    rowMode: 'object',
    callback: (row: any) => rows.push(row),
  });
  return rows;
}

export async function addIngredient(input: {
  name: string;
  mode: IngredientMode;
  uses_per_unit?: number;
}): Promise<void> {
  const db = await getDb();
  const usesRemaining = input.mode === 'untracked' ? null : 0;
  const usesPerUnit = input.mode === 'untracked' ? null : (input.uses_per_unit ?? 1);
  db.exec({
    sql: `INSERT INTO ingredients (name, mode, uses_per_unit, uses_remaining)
          VALUES (?, ?, ?, ?)`,
    bind: [input.name, input.mode, usesPerUnit, usesRemaining],
  });
}
```

Plain functions, no reactivity yet. Reactivity goes in the store layer next.

**Verify:** call `addIngredient({ name: 'onion', mode: 'counted', uses_per_unit: 2 })` from a component, then `getAllIngredients()`, log the result. You should see your row.

**Docs:**
- [SQLite — INSERT](https://www.sqlite.org/lang_insert.html)
- [SQLite — SELECT](https://www.sqlite.org/lang_select.html)

---

### 9. Build the ingredients store

Create `src/lib/stores/ingredients.svelte.ts`. The `.svelte.ts` extension matters — it tells Svelte's compiler to process runes in this file.

```ts
import { getAllIngredients, type Ingredient } from '$lib/db/ingredients';
import { registerStore } from './invalidation';

export function createIngredientsStore() {
  let ingredients = $state<Ingredient[]>([]);
  let loading = $state(true);
  let error = $state<string | null>(null);

  async function refresh() {
    loading = true;
    try {
      ingredients = await getAllIngredients();
      error = null;
    } catch (e) {
      error = String(e);
    } finally {
      loading = false;
    }
  }

  refresh();
  registerStore(refresh);

  return {
    get ingredients() { return ingredients; },
    get loading() { return loading; },
    get error() { return error; },
    refresh,
  };
}
```

The pattern: a function that creates state, wires up the query, registers itself for invalidation, and returns getters.

**Verify:** import the store in a component, render `store.ingredients` in the template, and confirm reactivity works (add an ingredient via direct call, manually call `store.refresh()`, see the list update).

**Docs:**
- [Svelte 5 — `$state` rune](https://svelte.dev/docs/svelte/$state)
- [Svelte 5 — `.svelte.ts` files](https://svelte.dev/docs/svelte/svelte-files#Other-files)

---

### 10. Build the implicit invalidation system

Create `src/lib/stores/invalidation.ts`.

```ts
const stores: Set<() => Promise<void> | void> = new Set();

export function registerStore(refresh: () => Promise<void> | void) {
  stores.add(refresh);
  return () => stores.delete(refresh);
}

export async function invalidateAll() {
  await Promise.all([...stores].map(fn => fn()));
}
```

Every store registers its refresh function. Every mutation calls `invalidateAll()` after committing.

Wire it into `addIngredient`:

```ts
import { invalidateAll } from '$lib/stores/invalidation';

export async function addIngredient(input: {...}) {
  // ... insert logic ...
  await invalidateAll();
}
```

**Verify:** in a component, add an ingredient via the form. The list (which reads from the store) should update automatically without a page reload.

**Docs:** No external doc — this is your own pattern. The closest analogue is [TanStack Query's invalidation](https://tanstack.com/query/latest/docs/framework/svelte/guides/query-invalidation), which is what you'd reach for if hand-rolled gets too clunky.

---

### 11. Build the bare-bones UI

Replace `src/routes/+page.svelte` with the real form + list page.

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { createIngredientsStore } from '$lib/stores/ingredients.svelte';
  import { addIngredient, type IngredientMode } from '$lib/db/ingredients';

  const store = createIngredientsStore();

  let name = $state('');
  let mode = $state<IngredientMode>('counted');
  let usesPerUnit = $state(1);

  async function handleSubmit(e: Event) {
    e.preventDefault();
    if (!name.trim()) return;
    await addIngredient({
      name: name.trim(),
      mode,
      uses_per_unit: mode === 'untracked' ? undefined : usesPerUnit,
    });
    name = '';
    usesPerUnit = 1;
  }
</script>

<h1>Rotation — Ingredients</h1>

<form onsubmit={handleSubmit}>
  <label>
    Name
    <input bind:value={name} required />
  </label>
  <label>
    Mode
    <select bind:value={mode}>
      <option value="counted">Counted</option>
      <option value="self_calibrating">Self-calibrating</option>
      <option value="untracked">Untracked</option>
    </select>
  </label>
  {#if mode !== 'untracked'}
    <label>
      Uses per unit
      <input type="number" bind:value={usesPerUnit} min="1" />
    </label>
  {/if}
  <button type="submit">Add</button>
</form>

{#if store.loading}
  <p>Loading...</p>
{:else if store.error}
  <p>Error: {store.error}</p>
{:else}
  <ul>
    {#each store.ingredients as ing (ing.id)}
      <li>
        {ing.name} — {ing.mode}
        {#if ing.uses_remaining !== null}
          ({ing.uses_remaining} uses remaining)
        {/if}
      </li>
    {/each}
  </ul>
{/if}
```

Style it however you want. For v0, ugly is fine.

**Note:** in Svelte 5, event handlers use the `on*` attribute syntax (e.g., `onsubmit`), **not** the older `on:submit` directive form. If older tutorials confuse you, this is why.

**Docs:**
- [Svelte 5 — Event handlers](https://svelte.dev/docs/svelte/basic-markup#Events)
- [Svelte 5 — `bind:`](https://svelte.dev/docs/svelte/bind)

---

### 12. Verify persistence

This is the moment of truth.

1. Run `pnpm dev`.
2. Add an ingredient via the form.
3. See it appear in the list.
4. **Hard reload** the page (Cmd-Shift-R / Ctrl-Shift-R).
5. The ingredient should still be there.

If it persists: OPFS is working. The whole stack is real.

If it doesn't persist:
- Check DevTools console for errors during init.
- Verify the cross-origin headers are set (step 2).
- Check Application → Storage → Origin Private File System for the `rotation.sqlite3` file.
- Some browsers need HTTPS for OPFS in production; localhost is exempted.

**Docs:**
- [MDN — Origin Private File System](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system)

---

### 13. Write the first Vitest test

Create `src/lib/db/ingredients.test.ts`.

The challenge: tests shouldn't use OPFS (it's browser-only and stateful across runs). Tests should use an in-memory SQLite database, which sqlite-wasm supports via the `oo1.DB` constructor (no path argument).

A clean approach is to factor the database setup so tests can inject an in-memory db, while production uses OPFS. That's a small refactor of `getDb()` — accept an optional override, or expose a separate `createDb()` for tests.

Sketch test:

```ts
import { describe, it, expect, beforeEach } from 'vitest';
import sqlite3InitModule from '@sqlite.org/sqlite-wasm';
import { runMigrations } from './migrations';

let db: any;

beforeEach(async () => {
  const sqlite3 = await sqlite3InitModule();
  db = new sqlite3.oo1.DB(':memory:');
  await runMigrations(db);
});

describe('ingredients', () => {
  it('inserts and retrieves an ingredient', () => {
    db.exec({
      sql: `INSERT INTO ingredients (name, mode, uses_per_unit, uses_remaining)
            VALUES (?, ?, ?, ?)`,
      bind: ['onion', 'counted', 2, 0],
    });

    const rows: any[] = [];
    db.exec({
      sql: 'SELECT * FROM ingredients',
      rowMode: 'object',
      callback: (row: any) => rows.push(row),
    });

    expect(rows).toHaveLength(1);
    expect(rows[0].name).toBe('onion');
    expect(rows[0].mode).toBe('counted');
  });
});
```

Run it: `pnpm test`.

**Note:** you may need to configure Vitest to handle the WebAssembly loading. Vitest's default environment is Node. SQLite WebAssembly works in Node but the import path may differ. If you hit issues, check the package's docs for Node-specific guidance.

**Docs:**
- [Vitest — Getting Started](https://vitest.dev/guide/)
- [Vitest — In-source testing & test setup](https://vitest.dev/guide/in-source.html)

---

## What this milestone proves

Once step 13 passes, you've validated:

- SvelteKit + Vite + TypeScript build is solid
- SQLite WebAssembly loads and runs in the browser
- OPFS persistence survives reloads
- The migration runner is idempotent
- The schema's first table works end-to-end (write, read, archive flag respected)
- The query module pattern is functional
- The store + invalidation pattern is reactive
- The first Vitest test runs against an in-memory database

Every other feature in v1 (recipes, shopping list, cooking, etc.) is the same pattern repeated. You now have a template to copy.

---

## Things deliberately deferred

These are not in the bare-bones milestone but will be needed before v1 ships:

- PWA setup (`@vite-pwa/sveltekit` plugin, manifest, service worker, prompt-to-update behavior)
- adapter-static configuration for production builds
- Production hosting with cross-origin headers
- JSON export / import functionality
- Edit/Save/Cancel guardrail pattern for inventory edits
- Reconciliation prompt for `uses_per_unit` edits
- The actual app features (recipes, shopping list, cook flow, etc.)

When you're ready for any of these, they're each a focused mini-project on top of the foundation you just built.

---

## When you get stuck

Use the agent prompt at the top of this file. State which step you're on, what you tried, what's not working. The architecture is locked, so the help you get should be debugging-shaped, not redesign-shaped.

Common gotchas to mention if relevant:
- "Cross-origin isolation" errors → step 2 headers
- "OPFS not available" → cross-origin headers, or you're testing in private browsing
- "Database is locked" → you're trying to use OPFS from the main thread; needs a worker
- "Migration ran twice" → the migrations table isn't being read correctly
- Svelte template doesn't update on mutation → store isn't registered for invalidation, or you forgot to `await invalidateAll()`

Good luck. The first time through is the slowest. After that, every feature is the same recipe.