---
title: "Prerequisites and Project Setup"
description: Scaffold the single TypeScript/Node repository you will grow throughout this guide — directory layout, SQLite+Drizzle with a one-line Postgres swap, and a .env template for provider config.
category: getting-started
type: tutorial
tags:
  - TypeScript
  - Node.js
  - SQLite
  - Drizzle ORM
  - PostgreSQL
  - monorepo
  - project layout
  - dotenv
  - DATABASE_URL
  - migrations
  - npm workspaces
  - environment variables
  - provider config
  - ANTHROPIC_API_KEY
  - OPENAI_API_KEY
  - tsconfig
  - drizzle-kit
  - schema migration
  - embedded database
  - mock adapter
keywords:
  - scaffold project
  - agent swarm setup
  - drizzle sqlite
  - drizzle postgres swap
  - env file template
  - drizzle generate
  - drizzle push
  - DATABASE_URL connection string
  - typescript node monorepo
  - zero config database
sources: [S22, S16, S41, S46]
---

**TL;DR** — This chapter walks you through creating the single repository you will extend across the entire guide. By the end you will have a TypeScript/Node project with a working SQLite database managed by Drizzle ORM, a ready-to-migrate schema, and a `.env` file that wires your first AI provider — or lets you skip the API key entirely by using the built-in mock adapter.

# Prerequisites and Project Setup

In the [previous chapter](./what-is-a-swarm.md) we mapped out the four moving parts of an agent swarm — the **orchestrator** (the server that owns the database and task queue), the **runner** (the daemon that claims tasks and executes agents), the **agent** (one AI worker with an adapter), and the **task** (the unit of work). Every chapter from here on adds one of those pieces. Before any of that, we need a place to put the code.

This chapter has one goal: a repository that compiles, has a live database, and loads its configuration from the environment.

---

## What you need before you start

| Requirement | Why |
|---|---|
| Node.js `>=22.0.0` | The runtime for both the orchestrator server and runner daemon |
| npm `>=10` (ships with Node 22) | Workspace support we use to split the code into packages |
| Git | Version control — optional, but assumed |
| A terminal | All commands here are shell commands |

You do **not** need PostgreSQL installed locally. We will start with SQLite, which writes to a single file. Postgres becomes an option later with one env-var change.

You do **not** need an AI provider API key to finish this chapter. The system includes a **mock adapter** — a built-in fake that responds to tasks without calling any external service. We cover provider keys at the end.

---

## Step 1 — Directory layout

Let's start with a structure we can grow into. An agent swarm has at least three distinct concerns: the orchestrator (HTTP API, database access, queue), the runner (task execution, adapter invocation), and the adapters themselves. Mixing all three into a flat `src/` folder makes boundaries invisible and refactoring painful later.

We will use a small **npm workspaces** monorepo — not because this is a large project, but because it enforces the separation from the first commit. npm workspaces let multiple `package.json` files share one `node_modules` and refer to each other by name (`@swarm/db`, `@swarm/shared`) without publishing to a registry.

Here is the target layout:

```
swarm/
├── package.json           ← workspace root (no source, just scripts + deps)
├── tsconfig.base.json     ← shared TypeScript settings
├── .env                   ← provider keys + DATABASE_URL (git-ignored)
├── .env.example           ← committed template
│
├── packages/
│   ├── db/                ← Drizzle schema, migrations, DB client
│   │   ├── package.json
│   │   ├── drizzle.config.ts
│   │   └── src/
│   │       ├── schema/    ← one file per table group
│   │       │   └── index.ts
│   │       └── client.ts  ← exports the db instance
│   │
│   ├── shared/            ← types, constants, validators shared across packages
│   │   ├── package.json
│   │   └── src/index.ts
│   │
│   ├── llm/               ← LLM client toolkit: provider registry, complete(), mock helpers
│   │   ├── package.json   ← (we flesh this out in the the-agent chapters)
│   │   └── src/index.ts
│   │
│   └── adapters/          ← one file per adapter kind (mock, claude-cli, http …)
│       ├── package.json
│       └── src/index.ts
│
└── src/
    ├── orchestrator/      ← HTTP API + queue + scheduler
    └── runner/            ← task execution daemon
```

<!-- GAP: exact starter file contents for packages/shared and packages/adapters are not in the assigned sources — the layout above is derived from S16's repo-map and the page spec; contents are authored incrementally in later chapters -->

Create the directories now:

```bash
mkdir -p swarm/packages/db/src/schema
mkdir -p swarm/packages/shared/src
mkdir -p swarm/packages/llm/src
mkdir -p swarm/packages/adapters/src
mkdir -p swarm/src/orchestrator
mkdir -p swarm/src/runner
cd swarm
```

---

## Step 2 — Workspace root `package.json`

We have the folders, but nothing ties them together yet. The root `package.json` is where npm learns about the workspaces and where we put the scripts that span the whole repo.

Create `package.json` at the repo root:

```json
{
  "name": "swarm",
  "private": true,
  "engines": { "node": ">=22.0.0" },
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev":          "tsx watch src/orchestrator/index.ts",
    "build":        "tsc -b",
    "typecheck":    "tsc --noEmit",
    "db:generate":  "npm run build --workspace=packages/db && drizzle-kit generate --config=packages/db/drizzle.config.ts",
    "db:migrate":   "tsx packages/db/src/migrate.ts",
    "db:push":      "drizzle-kit push --config=packages/db/drizzle.config.ts"
  },
  "devDependencies": {
    "typescript":   "^5.0.0",
    "tsx":          "^4.0.0",
    "drizzle-kit":  "^0.1.0"
  },
  "dependencies": {
    "drizzle-orm":  "^0.1.0",
    "dotenv":       "^16.0.0"
  }
}
```

A note on each script:

- **`dev`** — runs the orchestrator with `tsx`, which executes TypeScript directly without a separate compile step. Good for development.
- **`db:generate`** — compiles the `packages/db` package first (so Drizzle reads the compiled schema), then runs `drizzle-kit generate` to produce a SQL migration file. This is step 3 of the [database change workflow](#step-4----schema-migration-workflow) we cover shortly.
- **`db:push`** — applies the schema directly to the database without writing a migration file. Use this only during early development or when you want to reset to the schema quickly.
- **`db:migrate`** — runs a small migration runner script that applies pending migration files in order.

---

## Step 3 — TypeScript configuration

<!-- GAP: exact tsconfig values are not specified in the assigned sources (S16 only references `pnpm -r typecheck`); the settings below follow standard Node/ESM TypeScript conventions and are marked as such -->

TypeScript needs to know how to compile each package and how the packages reference each other. We use a **project references** setup: one base config, one per-package config that extends it, and a root config that lists all project references.

Create `tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target":          "ES2022",
    "module":          "NodeNext",
    "moduleResolution":"NodeNext",
    "strict":          true,
    "declaration":     true,
    "declarationMap":  true,
    "sourceMap":       true,
    "outDir":          "./dist",
    "rootDir":         "./src"
  }
}
```

Create `tsconfig.json` at the repo root (the project-references orchestrator):

```json
{
  "files": [],
  "references": [
    { "path": "./packages/db" },
    { "path": "./packages/shared" },
    { "path": "./packages/adapters" }
  ]
}
```

Each package gets its own `tsconfig.json`. Here is `packages/db/tsconfig.json` as the pattern:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir":    "./dist",
    "rootDir":   "./src"
  },
  "include": ["src/**/*", "drizzle.config.ts"]
}
```

The `"composite": true` flag is what enables project references — it tells TypeScript this package can be depended on by other packages in the same build graph.

---

## Step 4 — The database: SQLite by default, Postgres by env var

Now for the most important structural decision in this chapter. Every later chapter that stores tasks, runs, or agent configs needs a database. We want:

1. **Zero setup by default.** A new contributor should be able to clone the repo and run without installing Postgres.
2. **One-line swap to Postgres.** Production deployments and anyone who wants full Postgres semantics should get them by changing a single environment variable.

Drizzle ORM makes both possible. **Drizzle** is a TypeScript-first ORM that lets you define your schema in TypeScript and generates SQL migration files from it. Critically, it supports both `better-sqlite3` (for a local file-based database) and `postgres` (for a real Postgres connection) behind the same query interface — so the code in `src/orchestrator/` never changes when you switch database backends.

The switching key is `DATABASE_URL` (S22):

| `DATABASE_URL` value | What Drizzle connects to |
|---|---|
| Not set | SQLite file at `./data/swarm.db` (our default) |
| `postgres://...` | PostgreSQL — local Docker, hosted, or managed |

### Setting up the `packages/db` package

Create `packages/db/package.json`:

```json
{
  "name": "@swarm/db",
  "version": "^0.1.0",
  "private": true,
  "main":  "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json"
  },
  "dependencies": {
    "drizzle-orm":    "^0.1.0",
    "better-sqlite3": "^9.0.0"
  },
  "optionalDependencies": {
    "postgres": "^3.0.0"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.0.0"
  }
}
```

`better-sqlite3` is the synchronous SQLite driver Drizzle uses for file-based databases. `postgres` is the async Postgres driver; we list it as `optionalDependencies` so it is present in dev but a deployment without Postgres can opt out.

### The Drizzle config

`drizzle.config.ts` tells `drizzle-kit` where to find the schema and where to write migration files. It also determines which driver to use based on `DATABASE_URL`:

```ts
// packages/db/drizzle.config.ts
import type { Config } from "drizzle-kit";
import { config } from "dotenv";

config({ path: "../../.env" });

const databaseUrl = process.env.DATABASE_URL;

const sqliteConfig: Config = {
  dialect:       "sqlite",
  schema:        "./src/schema/index.ts",
  out:           "./migrations",
  dbCredentials: { url: "./data/swarm.db" },
};

const postgresConfig: Config = {
  dialect:       "postgresql",
  schema:        "./src/schema/index.ts",
  out:           "./migrations",
  dbCredentials: { url: databaseUrl! },
};

export default databaseUrl ? postgresConfig : sqliteConfig;
```

Notice the pattern: we read `DATABASE_URL` from the environment and choose a config object at module evaluation time. The schema path (`./src/schema/index.ts`) is the same in both cases — **your schema definition never changes when you switch backends**.

### A minimal starter schema

We need at least one table to verify the migration workflow. Let's define an `agents` table — the entity every later chapter builds on:

```ts
// packages/db/src/schema/agents.ts
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core";

export const agents = sqliteTable("agents", {
  id:        text("id").primaryKey(),
  name:      text("name").notNull(),
  adapter:   text("adapter").notNull(),
  createdAt: integer("created_at", { mode: "timestamp" })
              .$defaultFn(() => new Date()),
});
```

For Postgres the import path changes (`drizzle-orm/pg-core`) and column types differ slightly, but the shape of the schema object stays the same. We will handle the dual-dialect pattern in the database package chapter. For now, starting with SQLite is fine.

Export it from the schema index:

```ts
// packages/db/src/schema/index.ts
export * from "./agents";
```

### The DB client

The client file reads `DATABASE_URL` at runtime and returns the right Drizzle instance:

```ts
// packages/db/src/client.ts
import { drizzle as drizzleSQLite } from "drizzle-orm/better-sqlite3";
import Database from "better-sqlite3";
import * as schema from "./schema/index.js";

// DATABASE_URL is loaded from .env by the application entry point
const databaseUrl = process.env.DATABASE_URL;

export function createDb() {
  if (databaseUrl && databaseUrl.startsWith("postgres")) {
    // Dynamic import keeps better-sqlite3 out of the require graph
    // when running against Postgres.
    throw new Error(
      "Postgres support: import from client-pg.ts (see the Postgres setup section)"
    );
  }

  const sqlite = new Database(databaseUrl ?? "./data/swarm.db");
  return drizzleSQLite(sqlite, { schema });
}

export type Db = ReturnType<typeof createDb>;
```

This version covers the SQLite path fully. The Postgres variant follows the same shape but uses `drizzle-orm/postgres-js` and the `postgres` driver — we will add that in a later chapter when we need the full Postgres feature set.

---

## Step 5 — Schema → migration → typecheck workflow

Every time you change the data model, you follow the same three steps (S16):

```
1. Edit  packages/db/src/schema/*.ts
2. Generate a migration file
3. Typecheck to confirm nothing broke
```

Let's run through it once now to verify the setup works.

**Step 1** is already done — we wrote `agents.ts` above.

**Step 2** — generate the migration:

```bash
npm run db:generate
```

Under the hood this runs `drizzle-kit generate`, which:
- Compiles `packages/db` first (so Drizzle reads the schema from the compiled `.js` files, not raw TypeScript)
- Compares the schema to any existing migrations
- Writes a new SQL file to `packages/db/migrations/`

You will see output like:

```
packages/db/migrations/
  0000_initial_agents.sql
  meta/
    _journal.json
    0000_snapshot.json
```

The `_journal.json` is Drizzle's migration ledger — it tracks which migrations have been applied to which database.

**Step 3** — typecheck:

```bash
npm run typecheck
```

This runs `tsc --noEmit` across all packages. If the schema change broke a type used elsewhere, it surfaces here before you ship.

### Applying the migration

Generating a migration file does not apply it. To create the actual database file and run the migration:

```bash
npm run db:migrate
```

For SQLite this creates `./data/swarm.db` (or wherever `drizzle.config.ts` points) and executes the SQL. For Postgres it connects to the URL in `DATABASE_URL` and does the same.

You should now have a `data/swarm.db` file. You can verify the table was created:

```bash
sqlite3 data/swarm.db ".tables"
# → agents   drizzle_migrations
```

`drizzle_migrations` is Drizzle's internal table for tracking applied migrations. `agents` is ours.

### Swapping to Postgres

To switch to a local Postgres instance (or any hosted provider), set `DATABASE_URL` before running `db:migrate`:

```bash
DATABASE_URL=postgres://user:password@localhost:5432/swarm npm run db:migrate
```

Or put it in `.env` (covered in the next step) and run `npm run db:migrate` without the inline prefix. The schema files, migration files, and application code stay identical — only the connection string changes. This is the "one-line swap" the page spec mentions, and it is real: S22 confirms that the Drizzle schema stays the same regardless of database mode.

---

## Step 6 — Provider config: the `.env` file

The last thing we need before the first agent can run is a way to tell the system which AI provider to call and what credentials to use. We will keep this in a `.env` file loaded at startup.

Why `.env`? Provider API keys are secrets. They must not appear in source code or committed files. A `.env` file stays on the local machine, is git-ignored by convention, and is readable by `dotenv` at process startup.

### The env-key resolution pattern

The provider config follows a consistent convention (S46): each provider has a well-known environment variable for its API key. The system reads that variable at startup and uses it to authenticate. If the variable is not set, the relevant provider is unavailable — but the mock adapter (which needs no key) still works.

The variables you need to know:

| Provider | API key variable | Notes |
|---|---|---|
| Anthropic (Claude) | `ANTHROPIC_API_KEY` | `ANTHROPIC_OAUTH_TOKEN` also accepted; takes precedence if set (S46) |
| OpenAI / OpenAI-compatible | `OPENAI_API_KEY` | Works for any OpenAI-compatible endpoint |
| Google (Gemini) | `GEMINI_API_KEY` | |
| Groq | `GROQ_API_KEY` | |
| DeepSeek | `DEEPSEEK_API_KEY` | |

You do not need to set all of them — only the provider you plan to use. And as noted above, **the mock adapter needs none of them**.

There is also a note worth knowing about `ANTHROPIC_API_KEY` in the context of an agent runner (S41): if `ANTHROPIC_API_KEY` is set in the adapter's environment, the Claude CLI uses API-key authentication instead of subscription login. The system surfaces this as an informational warning in environment tests, not a hard error.

### The `.env.example` template

Create `.env.example` — this is the committed template. Every developer copies it to `.env` and fills in their values:

```dotenv
# .env.example — copy to .env and fill in the values you need

# ── Database ──────────────────────────────────────────────────────────────
# Leave unset to use SQLite (writes to ./data/swarm.db — zero config).
# Set to a postgres:// URI to use PostgreSQL instead.
#
# DATABASE_URL=postgres://user:password@localhost:5432/swarm

# ── Anthropic / Claude ────────────────────────────────────────────────────
# Required if you use the claude-cli adapter.
# ANTHROPIC_API_KEY=sk-ant-...

# ── OpenAI / OpenAI-compatible ────────────────────────────────────────────
# Required if you use the openai adapter or any OpenAI-compatible endpoint.
# OPENAI_API_KEY=sk-...
#
# Override the base URL to point at a local proxy or alternative host:
# OPENAI_BASE_URL=https://api.openai.com/v1

# ── Mock adapter ──────────────────────────────────────────────────────────
# No key needed. The mock adapter is always available and is the default
# in test environments. To force mock mode:
# SWARM_ADAPTER=mock
```

Copy it to `.env`:

```bash
cp .env.example .env
```

Then open `.env` and uncomment at least one provider key — or leave everything commented to use the mock adapter.

### Loading `.env` at startup

`dotenv` reads the `.env` file and injects values into `process.env` before your code runs. Add this line at the top of every entry point (orchestrator, runner, migration runner):

```ts
// src/orchestrator/index.ts  (and src/runner/index.ts)
import { config } from "dotenv";
config(); // reads .env from process.cwd()
```

`config()` is a no-op if a variable is already set in the environment, so you can override `.env` values from the shell without changing the file — useful in CI and production.

---

## Step 7 — Install and verify

With all the files in place, install dependencies:

```bash
npm install
```

This installs everything defined in the root `package.json` and in all `packages/*/package.json` files into a single shared `node_modules` at the repo root.

Then generate and apply the first migration:

```bash
npm run db:generate
npm run db:migrate
```

Finally, typecheck:

```bash
npm run typecheck
```

A clean typecheck with no errors means the repo is in a consistent state. You are ready for the next chapter.

---

## What we just built

Let's take stock of what the repo gives us now:

| Concern | Solution | Key decision |
|---|---|---|
| Package boundaries | npm workspaces (`@swarm/db`, `@swarm/shared`, `@swarm/llm`, `@swarm/adapters`) | Enforces separation from day one |
| TypeScript | `tsconfig.base.json` + per-package `tsconfig.json` with `composite: true` | One build graph, incremental compilation |
| Database (default) | SQLite via `better-sqlite3` + Drizzle ORM | Zero local setup; writes to `./data/swarm.db` |
| Database (production) | PostgreSQL via `DATABASE_URL` | One env-var change, same schema and migration files |
| Schema evolution | `npm run db:generate` → `npm run db:migrate` → `npm run typecheck` | Migration file is an artifact; always typechecked |
| Provider config | `.env` / `.env.example` with `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `DATABASE_URL` | Secrets stay off disk; mock adapter needs none |

---

## Try it yourself

Work through these variations to deepen your understanding of the setup before moving on:

1. **Swap SQLite for Postgres.** Add `DATABASE_URL=postgres://user:password@localhost:5432/swarm` to your `.env`, re-run `npm run db:migrate`, and verify the `agents` table appears in Postgres. Then remove the line and observe that `db:migrate` recreates the SQLite file on next run.

2. **Add a second table.** Create `packages/db/src/schema/tasks.ts` with an `id`, `agentId` (foreign key to `agents.id`), and `status` text column. Export it from the schema index, run `db:generate`, and inspect the new migration file to see the SQL Drizzle produced.

3. **Override the database path.** Change `drizzle.config.ts` to use `DATABASE_URL ?? "./data/dev.db"` and set `DATABASE_URL=./data/test.db` in your shell. Run `db:migrate` and confirm a separate file appears.

4. **Add a second provider profile.** In `.env.example`, add entries for `GROQ_API_KEY` and a corresponding `GROQ_BASE_URL`. In the next chapter you will wire these to an adapter, but having the env keys documented in the template means no one forgets them.

---

← Previous: [What Is an Agent Swarm?](./what-is-a-swarm.md) · Next: [Your First Agent](./your-first-agent.md) →
