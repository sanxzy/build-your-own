---
title: "Introduction: What OpenClaw Is and Why It Exists"
description: OpenClaw is a self-hosted personal AI assistant you reach through the chat apps you already use, with a Gateway process as its control plane.
category: getting-started
type: explanation
tags: [openclaw, introduction, self-hosted, AI gateway, personal assistant, design philosophy, overview, getting started, chat channels, operator control, DM pairing, security defaults, privacy, local-first]
keywords: [personal AI, self-host AI, run your own AI, AI chatbot alternative, openclaw gateway, operator, channels, telegram bot, discord bot, whatsapp AI, local AI assistant]
sources: [S1, S2, S4]
---

**TL;DR** — OpenClaw is a personal AI assistant you install on your own machine or server, reachable on the chat apps you already use (Telegram, WhatsApp, Slack, Discord, and more). Rather than a hosted chatbot controlled by a third-party service, OpenClaw puts a single control plane — the Gateway — on your hardware, so every configuration decision belongs to you. This chapter explains what problem that solves and who the system is designed for.

# Introduction: What OpenClaw Is and Why It Exists

## The one-sentence answer

OpenClaw is a **self-hosted personal AI assistant you reach through the chat apps you already use**, with a Gateway process as its control plane — the product is the assistant.

That sentence has three load-bearing parts. Let's unpack each one before we go further.

**"Self-hosted"** means you run the software on hardware you control: your laptop, a home server, a VPS, a Docker container. No third-party company is running your assistant for you.

**"The chat apps you already use"** means you do not need a new app to talk to your assistant. OpenClaw connects to the messaging channels where you already spend time — Telegram, WhatsApp, Slack, Discord, Google Chat, Signal, iMessage, Microsoft Teams, Matrix, and many others. You send a message there; your assistant replies there.

**"Gateway as its control plane"** means there is one long-running process — the Gateway — that acts as the single point of coordination for all agents, all channels, all tools, and all sessions. Think of it as the switchboard: every message that arrives, every model call that goes out, and every reply that comes back travels through the Gateway. We'll look at its internals in the next chapter; for now, hold it as "the process that runs your assistant."

## The problem a hosted chatbot does not solve

You can already talk to AI through browser-based products. So what does running your own instance add?

The short answer is: **control**. Here is what that word means concretely.

| What you control | With a hosted chatbot | With your own OpenClaw Gateway |
|---|---|---|
| **Your data** | Sent to and stored by the service | Stays on your machine; conversation history is JSONL files in your home directory |
| **Which model you use** | Whatever the service offers | Any of 35+ providers (Anthropic, OpenAI, Google, and more); you supply the API key |
| **Which channels** | The service's web UI or app | Any channel you configure: Telegram, WhatsApp, Slack, Discord, your own custom surface |
| **Which tools the agent can use** | The service's curated set | You define the tool policy: shell access, browser, cron jobs, custom plugins — whatever you allow |
| **Who can trigger the agent** | Whoever has a service account | You, via DM pairing — unknown senders get a pairing code; you approve them explicitly |
| **What the agent remembers** | What the service persists | Workspace files on your filesystem; memory plugins you choose and configure |

This is the design philosophy in VISION.md expressed plainly: "OpenClaw is the AI that actually does things. It runs on your devices, in your channels, with your rules." The emphasis on "your rules" is not a slogan — it maps to specific configuration choices you make as the operator.

The term **operator** appears throughout this library, so let's define it now. The operator is the person who installs and runs the Gateway — typically you, on your own machine. The operator sets configuration, approves channel pairings, installs plugins, and defines tool policy. OpenClaw is built for a single trusted operator per Gateway, not for shared multi-tenant access by adversarial users.

## Who OpenClaw is for

OpenClaw's target user is a person who:

- Wants an AI assistant that can do real work: run shell commands, search the web, write files, manage a schedule, draft messages on your behalf.
- Wants that assistant available on the messaging apps they already use daily, not locked inside a separate tab.
- Wants control over which AI provider and model powers the assistant — and the ability to swap it out without changing how the rest of the system works.
- Is comfortable running a Node.js process and editing a configuration file.

You do not need to be a professional software developer, but the current setup path is terminal-first. VISION.md is direct about this: "OpenClaw is currently terminal-first by design. This keeps setup explicit: users see docs, auth, permissions, and security posture up front."

## How the pieces fit together (a first map)

Before you read the rest of this library, here is a rough map of the territory. Each term below will be given a full chapter; this is enough to avoid confusion as you go.

**Agent** — the AI runtime that actually thinks and replies. An agent has its own workspace directory on your filesystem, its own conversation history, its own tool access. You can run multiple agents inside one Gateway.

**Channel** — a message surface the agent can communicate through. Telegram is a channel. Discord is a channel. A channel plugin registers itself with the Gateway and translates between the channel's wire format and OpenClaw's internal message model.

**Gateway** — the central process that connects everything. It runs on port 18789 by default. It routes incoming messages from channels to the right agent, manages session state, enforces tool policy, and coordinates replies. The next chapter covers its architecture in detail: see [High-Level Architecture: Four Layers and the Gateway Control Plane](./02-architecture.md).

Here is how a message travels from your phone to a reply, at the highest level of abstraction:

```
[Telegram DM] → [Telegram channel plugin in Gateway] → [agent loop] → [model API call] → [reply back to Telegram]
```

Every subsequent chapter in this library is an expansion of one segment of that arrow.

## Security by default

One thing worth calling out before you read further: OpenClaw connects to real messaging surfaces where real people (or bots) can send you messages. The system's defaults are designed to keep strangers out.

The primary safeguard is **DM pairing**. When an unknown sender messages your agent on Telegram, WhatsApp, Discord, Slack, or Signal, the agent does not respond to the message. Instead, the sender receives a short pairing code. You then approve that pairing by running:

```bash
openclaw pairing approve <channel> <code>
```

Only after you approve does the sender get added to your local allowlist and can trigger the agent. This is the default `dmPolicy="pairing"` behavior. If you want the agent open to anyone, you can set `dmPolicy="open"` — but that is an explicit opt-in, not the default.

The security model behind this is straightforward. OpenClaw is designed as a **personal assistant for one trusted operator**, not a shared service for many users. SECURITY.md states this explicitly: "OpenClaw does not model one gateway as a multi-tenant, adversarial user boundary." If you need multiple people to use separate agents with separate data, the recommended approach is one Gateway per person, not one Gateway shared among adversarial users.

We will cover the full security model — sandbox modes, network policy, Gateway auth — in [Security and Governance](../operations/20-security.md). For now, the key point is that the defaults are conservative, and every path to opening the system up is an explicit operator decision.

## What comes next

The rest of this library builds the full picture one layer at a time, following the reading order below:

1. **Architecture** (next chapter) — the four layers of the system and how they connect.
2. **Gateway** — how the Gateway process works, its wire protocol, and how clients connect.
3. **Channels** — how each messaging surface is registered and how DM pairing gates access.
4. **Agents** — what an agent actually is: a workspace directory, a runtime, a set of files.
5. **Agent Loop** — the six stages every message passes through on its way to a reply.
6. And from there: sessions, memory, tools, plugins, configuration, and deployment.

Each chapter introduces the terms it uses and can be read independently, but following the order above means you never hit a term you have not seen defined.

---

Next: [High-Level Architecture: Four Layers and the Gateway Control Plane](./02-architecture.md) →
