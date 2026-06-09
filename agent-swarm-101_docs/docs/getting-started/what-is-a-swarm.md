---
title: "What Is an Agent Swarm?"
description: Mental model for the whole system — orchestrator vs runner vs agent vs task, the squad topology, and why a control plane matters.
category: getting-started
type: explanation
tags: [agent swarm, orchestrator, runner, agent, task, squad, adapter, multi-agent, control plane, mental model, architecture overview, heartbeat, lifecycle, single assignee, org chart, schedule, budget, workspace]
keywords: [multi-agent coordination, agent management, task queue, control plane server, agent backend, worker process, agent execution, squad delegation, autonomous agents]
sources: [S1, S15, S17, S19]
---

**TL;DR** — An agent swarm is a coordinated fleet of AI agents managed by a central control plane called the **orchestrator**. This chapter builds the mental model before any code: what the four core roles are (orchestrator, runner, agent, task), how the squad topology lets agents delegate work to one another, and why separating "managing work" from "running work" is the design decision everything else follows from.

# What Is an Agent Swarm?

## The problem this system solves

Think about how you currently work with a single AI coding agent. You open a terminal, type a prompt, and watch it run. While it runs, you are essentially babysitting it — you cannot usefully hand a second task to a second agent at the same moment unless you open another terminal window and manually track which one is doing what. Restart your machine and you lose everything. Hit an API cost spike at 3 AM and you find out at 9 AM.

Now imagine twenty agents — twenty Claude Code sessions, or a mix of Claude Code, Codex, and Cursor — all working in parallel toward related goals. Who decides which task goes to which agent? Who knows when a task is already being worked on so two agents do not attempt the same thing? Who stops an agent that is burning through its budget? Who wakes the right agent when a scheduled nightly report is due?

Without a dedicated system, the answer is *you* — manually, imperfectly, and only while you are awake. That is the gap an agent swarm fills. The swarm is infrastructure for running a team of AI agents the same way a company runs a team of humans: with roles, assigned work, accountability, and cost control.

## Four roles, one clean division of labour

Every part of Swarm plays exactly one role. Let's walk through them, smallest to largest, so we can see how they compose.

### The task — the unit of work

A **task** (sometimes called an issue) is the atomic unit of work in the system. It represents a single piece of work that needs to be done — "write the homepage copy", "fix the authentication bug", "generate the weekly performance report".

A few properties make tasks important as a design primitive:

- **Single assignee.** A task belongs to exactly one agent at a time. This is not a convention — the system enforces it. Atomic checkout and execution locks prevent two agents from claiming the same task. No double-work, no conflicting edits.
- **Lifecycle / state machine.** A task moves through states: enqueued → claimed → in progress → completed (or failed, or blocked). The orchestrator tracks this state; it is the record of what is actually happening across the whole team.
- **Parentage.** Tasks can have parent tasks, all the way up to a top-level goal. A task always has an answer to "why does this exist?" — it exists in service of its parent, which exists in service of its parent, and so on up to the company's founding goal.
- **Preserved context.** Comments, documents, attachments, and work products stay attached to the task — not to a terminal session that disappears on reboot.

### The agent — one AI worker

An **agent** is one AI worker in the swarm. It has two defining properties: an **adapter** (the technical interface that tells the system how to invoke this agent, observe it, and cancel it) and a **configuration** (what the agent does, what its role is, who it reports to).

Agents have roles and reporting lines — they exist within an org chart, just like employees in a company. An agent can report to another agent. An agent can have a job description ("review assigned tasks, pick the highest priority, and work it") that it receives as part of its execution context.

Notice that an agent is a *definition*, not a process. An agent says "I am the frontend engineer, I run via Claude Code, here is how you invoke me." The actual invocation — the live process doing work right now — is a separate concept we will get to in a moment.

### The adapter — the execution interface

Because a swarm might include Claude Code sessions, Codex instances, Cursor agents, bare shell scripts, or HTTP webhooks all running side by side, the system needs a uniform interface so the orchestrator does not need to know the details of each one.

The **adapter** is that interface. Every agent backend — regardless of whether it is a local CLI session, a subprocess, or an HTTP endpoint — exposes the same contract: how to start (or resume) a run, how to observe it, and how to cancel it.

This means the orchestrator can coordinate agents built on Claude Code, Codex, and OpenClaw using identical control logic. The adapter is the seam between the generic control plane and the specific runtime. Real systems support adapter types like local CLI sessions, process/command-style execution, and HTTP/webhook-based integrations for external agents.

### The runner — the worker process

An **runner** (also called a daemon) is a long-running worker process that you deploy on a machine — your laptop, a cloud VM, wherever your agents will actually execute. It connects to the orchestrator, watches for tasks assigned to agents it manages, claims a task on behalf of an agent, and then uses that agent's adapter to invoke the execution.

The runner is what makes agents real. An agent defined in the orchestrator's database does nothing on its own — it is metadata. The runner is the process that reads that metadata, picks up a task, and actually runs Claude Code or Codex or the shell script.

One runner can manage multiple agents. Many runners can connect to one orchestrator. This lets you spread agents across machines — one runner on your laptop handles local CLI agents, another runner in a cloud environment handles agents that need network access, and the orchestrator does not care where any of them live.

### The orchestrator — the control plane

The **orchestrator** is the long-running server at the centre of the whole system. It owns three things: the database (all persistent state — agents, tasks, runs, costs, history), the task queue (what work needs to happen and who should do it), and the schedulers (recurring work that should fire on a cron schedule or in response to an event).

Here is the critical design principle, and it is worth reading carefully: **the orchestrator orchestrates but does not itself run agents.** It does not open a Claude Code session. It does not execute a shell command. It does not call an API. It assigns work to agents (via tasks), monitors progress (via runs), enforces budgets (via cost events), and coordinates the whole — but the actual execution happens inside runners.

This separation is intentional and important. It means the orchestrator can be a stable, always-on server process, while runners come and go. It means you can add a new runner on a new machine and the orchestrator immediately has more capacity. It means if a runner crashes, the orchestrator can detect the orphaned run and recover it.

Here is how the four roles relate to each other:

```
┌──────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                      │
│                                                      │
│  database  ·  task queue  ·  schedulers  ·  budgets  │
│                                                      │
│  [ agent registry + org chart ]                      │
│  [ task lifecycle manager    ]                       │
│  [ cost tracking + budgets   ]                       │
│  [ schedules + routines      ]                       │
└────────────────────────┬─────────────────────────────┘
                         │  claims tasks, reports runs
            ┌────────────┼────────────┐
            │            │            │
      ┌─────┴────┐  ┌────┴─────┐  ┌──┴───────┐
      │ Runner A │  │ Runner B │  │ Runner C │
      │(laptop)  │  │(cloud VM)│  │(laptop 2)│
      └──┬──┬───┘  └──┬───────┘  └──┬───────┘
         │  │         │              │
      agent agent   agent          agent
       (via  (via   (via HTTP/     (via
      Claude Codex) webhook)      shell)
      Code)
```

Each runner is connected to one orchestrator. Each agent is backed by an adapter. Each task has exactly one assignee at a time.

## The squad topology — agents that delegate

We have described agents as individual workers. But teams have structure: some agents do hands-on work, others direct and coordinate.

A **squad** is a group of agents with a designated **leader**. When a task is assigned to the squad (rather than a specific agent), the leader agent claims it, reviews it, and delegates pieces to the right member. The person assigning the work writes `@FrontendTeam` instead of `@alice-or-bob-or-carol` — the squad's leader decides the routing.

This is the org chart idea made concrete. Agents have reporting lines. A CEO agent coordinates executive agents (CTO, CMO, CFO). Each executive coordinates individual contributor agents under them. A task assigned at the top of the hierarchy flows down through delegation; work products and status flow back up.

The orchestrator enforces the same single-assignee rule even inside a squad. When the leader delegates a sub-task to a member, that sub-task becomes its own task record with the member as assignee. Nothing is implicit.

## The schedule — recurring work without manual kick-offs

Not all work is triggered by a human. A **schedule** (also called a routine or autopilot) fires work on a cron trigger, a webhook, or manually. Each time it fires, it creates a new task record and routes it to the assigned agent.

Schedules are how you make agents feel autonomous: daily standups, weekly reports, and periodic code reviews all run themselves. The orchestrator's scheduler creates the task; a runner picks it up; the agent executes.

## Why a control plane matters

You might wonder: why not just start multiple agent processes and let them coordinate among themselves? Some projects try this — a shared channel, a shared file, a handshake protocol. It works until it does not.

The problem is that distributed coordination is hard. Who has a task checked out right now? Is that agent still alive, or did it crash? How much have we spent this billing cycle across all agents? What was the state of the task when the agent stopped? These questions require a single authoritative record — a place that every participant writes to and reads from.

The orchestrator is that record. Because it owns the database and the task queue, every runner sees the same state. Atomic checkout means two runners cannot claim the same task simultaneously. Budget enforcement is global — the orchestrator sees every cost event from every run, regardless of which runner or agent produced it. Governance and approvals are possible because all decisions flow through one place.

This is what the phrase "control plane" means: a dedicated layer whose job is coordination, not execution. The orchestrator knows what everyone is doing even when it is doing nothing itself.

## The full picture in a table

| Concept | What it is | Where it lives |
|---|---|---|
| **Orchestrator** | The control-plane server; owns the DB, task queue, schedulers, and budget ledger | One long-running process (server or cloud) |
| **Runner** | Worker process that connects to the orchestrator, claims tasks, and invokes agent adapters | One process per machine/environment |
| **Agent** | One AI worker; defined by an adapter type + config + role + org-chart position | A record in the orchestrator's database |
| **Adapter** | The execution interface: how to invoke, observe, and cancel one category of agent backend | Attached to an agent; implemented by the runner |
| **Task** | The unit of work; has a lifecycle, a single assignee, and full parentage back to the company goal | A record in the orchestrator's database |
| **Run** | One execution window for an agent on a task; produces logs, cost events, and session state | Created by the runner when it starts the adapter |
| **Squad** | A leader agent + member agents; the leader claims delegated tasks and routes sub-work to members | A group configuration in the orchestrator |
| **Schedule** | Recurring work; fires on cron, webhook, or manual trigger; creates a task each time | Configured on the orchestrator |
| **Budget** | Spend cap per agent, project, or workspace; the orchestrator pauses work when the cap is hit | Enforced by the orchestrator across all runners |

## Where we go from here

This chapter has given you the mental model — the four roles and the key ideas (single-assignee tasks, adapter-backed agents, runners that separate "managing work" from "running work", squads that delegate). Nothing has been built yet.

The journey this book takes starts from the smallest possible thing: a single agent on a mock adapter, running against a minimal orchestrator, executing one task. From there we add the runner daemon, then real agent adapters (Claude Code, Codex, OpenClaw), then squads, then schedules, and finally budget governance. Each chapter adds one concern and solves one new problem — the same way the orchestrator itself was designed.

Next up: setting up the project and installing the prerequisites so we have somewhere to run code.

---

Next: [Prerequisites and Project Setup](./project-setup.md) →
