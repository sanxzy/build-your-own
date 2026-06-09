---
title: "Agents as a Team: The Org Chart"
description: How a self-referencing foreign key lets agents report to other agents, how tasks get assigned polymorphically, and how every query stays tenant-scoped.
category: coordination
type: explanation
tags: [org chart, reportsTo, hierarchy, routing, polymorphic assignee, assignee_type, assignee_id, agent tree, manager, executive agent, engineer agent, multi-tenancy, workspace_id, self-FK, tenant isolation, agent team, reporting structure]
keywords: [self-referencing foreign key, tree structure, parent agent, child agent, task assignee, human assignee, agent assignee, workspace scoping, workspace_id, multi-tenant, Drizzle, PostgreSQL, index, workspaceReportsToIdx]
sources: [S35, S2]
---

**TL;DR** — A single `reportsTo` column on the agents table creates a tree: any agent can have a parent agent, and that tree is how routing decisions flow from an executive agent down to individual workers. On top of that structure, tasks carry a polymorphic assignee — either a human or an agent — which is what makes human+agent teams work in the same system. Every query is scoped to a workspace so teams from different tenants never see each other's data. By the end of this chapter you'll understand the three pillars that all coordination chapters build on.

# Agents as a Team: The Org Chart

So far we have been thinking about a single agent: it has an adapter that knows how to invoke it, a config that tells it what to do, and a task queue that feeds it work. That model works perfectly for one worker. But a real system has many agents — and the moment you have more than one, you need to answer two questions that a single-agent model cannot: *who does what*, and *who decides*?

This chapter introduces the structural answer to both questions. We will look at three interlocking design decisions — the **org chart**, **polymorphic assignment**, and **workspace scoping** — that together make agent teams possible. These are not features you add later; they are the foundation every coordination pattern in the next chapters assumes.

## The problem: one agent is easy, a team needs structure

Imagine you want two agents: one that plans work and one that executes it. The planner receives a goal, breaks it into subtasks, and hands them off. The executor picks up each subtask and runs it.

This is a perfectly sensible team structure — but there is nothing in the data model yet that captures it. How does the system know that the executor reports to the planner? How does the planner know which executors are "under" it? If the executor finishes a task, who does it report back to?

We need a way to express *hierarchy* in the database, and the right tool for that is a **self-referencing foreign key** — a column on the agents table that points back to another row in the same table.

## The org chart — a `reportsTo` self-foreign-key

Let's look at the agents table. Below is the Drizzle schema, drawn directly from the source and scrubbed of any product-specific branding. Read the comments — they highlight the columns that matter most for this chapter:

```ts
// Simplified view of the agents table — full schema below
import {
  type AnyPgColumn,
  pgTable,
  uuid,
  text,
  integer,
  timestamp,
  jsonb,
  index,
} from "drizzle-orm/pg-core";

export const agents = pgTable(
  "agents",
  {
    id: uuid("id").primaryKey().defaultRandom(),

    // Multi-tenancy: every agent belongs to exactly one workspace.
    // All queries filter by this column — see "Workspace scoping" below.
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id),

    name: text("name").notNull(),
    role: text("role").notNull().default("general"),
    title: text("title"),
    icon: text("icon"),

    // Lifecycle state: "idle" | "running" | "paused" | …
    status: text("status").notNull().default("idle"),

    // *** THE ORG CHART COLUMN ***
    // A nullable UUID that points to another row in this same table.
    // null  → this agent is at the top of its tree (root / executive level).
    // non-null → this agent reports to the agent with that id.
    reportsTo: uuid("reports_to").references((): AnyPgColumn => agents.id),

    // The adapter layer (see "The Adapter Interface"):
    adapterType: text("adapter_type").notNull().default("process"),
    adapterConfig: jsonb("adapter_config")
      .$type<Record<string, unknown>>()
      .notNull()
      .default({}),
    runtimeConfig: jsonb("runtime_config")
      .$type<Record<string, unknown>>()
      .notNull()
      .default({}),

    // Budget tracking
    budgetMonthlyCents: integer("budget_monthly_cents").notNull().default(0),
    spentMonthlyCents: integer("spent_monthly_cents").notNull().default(0),

    // Pause state
    pauseReason: text("pause_reason"),
    pausedAt: timestamp("paused_at", { withTimezone: true }),

    permissions: jsonb("permissions")
      .$type<Record<string, unknown>>()
      .notNull()
      .default({}),
    lastHeartbeatAt: timestamp("last_heartbeat_at", { withTimezone: true }),
    metadata: jsonb("metadata").$type<Record<string, unknown>>(),

    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    // Index 1: fast lookup of all agents in a workspace by status
    workspaceStatusIdx: index("agents_company_status_idx").on(
      table.workspaceId,
      table.status,
    ),
    // Index 2: fast lookup of all agents that report to a given parent,
    //          scoped to the workspace — this is the org-chart traversal index.
    workspaceReportsToIdx: index("agents_company_reports_to_idx").on(
      table.workspaceId,
      table.reportsTo,
    ),
    // Index 3: fast lookup of agents by their default execution environment
    workspaceDefaultEnvironmentIdx: index(
      "agents_company_default_environment_idx",
    ).on(table.workspaceId, table.defaultEnvironmentId),
  }),
);
```

The type annotation `(): AnyPgColumn => agents.id` is the Drizzle pattern for a **self-referencing** column. Drizzle needs the `AnyPgColumn` return type and the thunk (the `() =>`) because, at the time the column definition is evaluated, `agents` is not yet fully defined — the thunk defers evaluation until everything is in scope.

### What the tree looks like in practice

With `reportsTo` in place, a simple two-level team looks like this in the database:

| id | name | reportsTo |
|---|---|---|
| `aaa` | Planner | `null` |
| `bbb` | Executor A | `aaa` |
| `bbb` | Executor B | `aaa` |

And as an ASCII tree:

```
Planner  (reportsTo = null — root of the tree)
├── Executor A  (reportsTo = Planner.id)
└── Executor B  (reportsTo = Planner.id)
```

Any agent can itself become a parent: an "Engineering Manager" agent could sit between the Planner and the Executors, giving you as many levels as the work demands.

```
Planner
└── Engineering Manager
    ├── Executor A
    └── Executor B
```

There is no hard limit on depth — `reportsTo` is a plain foreign key, not a fixed two-level parent/child design. The data model is a general tree.

### Why the index matters

The most common org-chart query is "give me all direct reports of agent X within workspace W":

```sql
SELECT *
FROM agents
WHERE workspace_id = $workspaceId
  AND reports_to = $parentAgentId;
```

Without an index this is a full table scan across every agent in the workspace. The `agents_company_reports_to_idx` index on `(workspace_id, reports_to)` turns that into a fast index seek — important once a workspace has hundreds of agents and the orchestrator is resolving reporting chains on every task dispatch.

Notice the index is **composite** — `workspace_id` comes first. This is deliberate: all queries in the system start by filtering to a workspace (we will come back to this), so the leading column must be the workspace key to make the index useful.

## The problem this creates: who can be assigned work?

Once we have an org chart, we have a second question: when the Planner creates a subtask and wants to assign it to Executor A, what does "assign" mean in the schema?

The naïve approach would be to add an `agent_id` column to the tasks table. That works, but it breaks down as soon as a human needs to take a task. Real teams mix human reviewers and AI agents. If the task table has only an `agent_id`, human assignees require either a separate `user_id` column, or a nullable dual-column design — and then every query needs to handle both cases.

## Polymorphic assignment — one assignee column pair for humans and agents

The cleaner design, drawn from S2, is a **polymorphic assignee**: rather than one typed FK, the tasks table carries two columns:

- `assignee_type` — a string that says what kind of entity is assigned (`"member"` for a human, `"agent"` for an AI agent).
- `assignee_id` — a UUID that identifies the entity within its type.

Together they form a **soft polymorphic reference**: the database does not enforce a FK constraint across two different tables (it cannot, without triggers), but the application layer resolves the pair to the right record at query time.

This is the design described in S2:

> Assignees are polymorphic — can be a member or an agent. `assignee_type` + `assignee_id` on issues. Agents render with distinct styling (purple background, robot icon).

In our system, the same principle applies to tasks. A task row might look like:

| id | title | assignee_type | assignee_id |
|---|---|---|---|
| `t1` | Write unit tests | `agent` | `bbb` (Executor A) |
| `t2` | Review PR | `member` | `u42` (a human) |
| `t3` | Deploy to staging | `agent` | `ccc` (Executor B) |

The query to fetch "all tasks assigned to agent bbb" is:

```sql
SELECT *
FROM tasks
WHERE workspace_id = $workspaceId
  AND assignee_type = 'agent'
  AND assignee_id = $agentId;
```

And the query to fetch "all tasks assigned to human u42":

```sql
SELECT *
FROM tasks
WHERE workspace_id = $workspaceId
  AND assignee_type = 'member'
  AND assignee_id = $userId;
```

Same column pair, different `assignee_type` value. The orchestrator's routing logic checks `assignee_type` before deciding how to dispatch — agent tasks go to the runner daemon; member tasks go to a notification or inbox.

### Why this matters for the org chart

When the Planner agent (at the root of our tree) creates a subtask, it sets `assignee_type = 'agent'` and `assignee_id = <executor_id>`. The org-chart tree tells the system *which* executors are under the Planner; the polymorphic assignee tells the task *where* to go. The two designs fit together: the tree defines reporting relationships, and the assignee pair surfaces on every unit of work.

<!-- GAP: the exact column names for assignee_type and assignee_id on the tasks/issues table are described in S2 prose but the full tasks schema (DDL or Drizzle definition) is not in the assigned sources; the description above is grounded in the S2 prose statement and the overall design intent — source silent on the exact column definitions -->

## Multi-tenancy — every query scoped by workspace

There is a third concern underneath everything we have just built. The agents table, the tasks table, the org chart traversal query — they all assume the data belongs to one team. But in a real deployment, many independent teams (tenants) share the same database. You cannot let team A's agents see team B's org chart.

The mechanism is captured in S2 in a single rule:

> All queries filter by `workspace_id`. Membership checks gate access. `X-Workspace-ID` header routes requests to the correct workspace.

In our schema, the corresponding column is `workspace_id` on the agents table (the name reflects the origin; in our documentation we call this the **workspace**, sometimes called a company or tenant). Every single query that touches agents must include a `WHERE workspace_id = $workspaceId` predicate. No exceptions.

This is not a reminder you enforce case-by-case; it is an architecture rule you bake in. In practice, the orchestrator's data-access layer accepts a workspace context and injects the workspace predicate automatically. A query that omits it should fail to compile (via a typed query builder) or at least be flagged in code review.

The reason the composite indexes in the agents table lead with `workspace_id` (not with `status` or `reports_to`) is exactly this: the workspace filter is always the first predicate. Putting it first in the index keeps every query fast.

### The workspace scoping rule stated plainly

Think of it as a contract the rest of the coordination chapters follow:

> **Every query that touches an agent, task, or related entity is always preceded by a workspace filter.**

When we look at squads in the next chapter, when we build scheduling, when we implement cost tracking — the workspace column appears in every query. We will not repeat this reminder each time; it is simply the rule the system runs on.

## Putting the three pillars together

Let's pause and look at what we have built up conceptually. The three design decisions connect like this:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Workspace W1                                                       │
│                                                                     │
│  agents table                         tasks table                   │
│  ┌─────────────────────────────┐      ┌────────────────────────┐   │
│  │ id   | name     | reportsTo │      │ id | assignee_type     │   │
│  │ aaa  | Planner  | null      │      │    |   + assignee_id   │   │
│  │ bbb  | Exec A   | aaa       │      │ t1 | agent / bbb       │   │
│  │ ccc  | Exec B   | aaa       │      │ t2 | member / u42      │   │
│  └─────────────────────────────┘      └────────────────────────┘   │
│                                                                     │
│  • all rows carry workspace_id = W1                                   │
│  • org-chart index: (workspace_id, reports_to)                        │
│  • task routing resolves assignee_type first, then assignee_id      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

- The **org chart** (`reportsTo`) defines who reports to whom — it is the chain of command.
- **Polymorphic assignment** (`assignee_type` + `assignee_id`) allows any entity — human or agent — to own a task, without the task table needing separate columns per entity type.
- **Workspace scoping** (`workspace_id` on every table, injected into every query) keeps tenants isolated, and the composite indexes keep scoped traversals fast.

## A quick reminder of what came before

Before reading further, it helps to have two earlier building blocks in mind:

- **The Adapter Interface** (see [The Adapter Interface](../the-agent/adapter-interface.md)) — an agent's `adapterType` and `adapterConfig` columns describe *how* to invoke it (process, HTTP, CLI). The org chart describes *who* it reports to; the adapter describes *how* it runs.
- **Modeling Tasks** (see [Modeling Tasks](../tasks-and-queue/modeling-tasks.md)) — tasks carry a single assignee. The `assignee_type`/`assignee_id` polymorphic pair is what makes that "single assignee" flexible enough to accommodate both humans and agents.

If either concept is unfamiliar, a quick read of those chapters will make the rest of the coordination section clearer.

## What comes next

The org chart as defined here is a general tree: any agent can report to any other agent, and the tree can be as deep as you like. But depth alone does not tell us *how* a parent agent decides to route work downward. For that, we need the concept of a **squad** — a named group with a designated leader agent whose sole job is to receive a task, decide which member can handle it, and delegate.

In [Squads: A Leader That Delegates](./squads.md) we add that delegation logic on top of the org-chart foundation built here.

---

← Previous: [Crash Recovery and Liveness](../tasks-and-queue/crash-recovery-and-liveness.md) · Next: [Squads: A Leader That Delegates](./squads.md) →
