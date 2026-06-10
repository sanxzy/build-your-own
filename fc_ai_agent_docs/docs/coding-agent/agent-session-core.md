---
title: "AgentSession: The Core of the Coding Agent"
description: "Build AgentSession — the shared core that wires the Agent Harness to runtime services (file I/O, subprocess, network), configures model selection and compaction, and owns the session lifecycle."
category: coding-agent
type: tutorial
tags: [AgentSession, Agent class, session lifecycle, model selection, compaction, bash, branching, persistence, coding-agent, composition, shared core]
keywords: [AgentSession, runtime services, session lifecycle, agent composition, Node.js harness]
sources: [S46, S47]
---

**TL;DR** — The coding agent has three run modes (interactive, print, RPC), but they all share the same core: `AgentSession`. It wires the `AgentHarness` to Node.js runtime services (filesystem, subprocess, network), configures model selection and compaction, manages session persistence and branching, and exposes a clean API that all three modes build on.

## Why AgentSession exists

The `AgentHarness` (from the Agent Core section) is generic — it doesn't know about filesystems, bash execution, or coding tools. `AgentSession` specializes it for the coding domain:

```ts
// AgentSession owns the "coding" part of "coding agent"
class AgentSession {
  private harness: AgentHarness;
  private runtime: NodeRuntime;
  private sessionManager: SessionManager;
  private tools: Map<string, AgentTool>;
}
```

## Runtime services

Create `packages/coding-agent/src/core/agent-session-runtime.ts`. The `NodeRuntime` provides OS-level services:

```ts
interface NodeRuntime {
  readFile(path: string, options?: { offset?: number; limit?: number }): Promise<string>;
  writeFile(path: string, content: string): Promise<void>;
  editFile(path: string, oldString: string, newString: string): Promise<string>;
  executeBash(command: string, options?: BashOptions): Promise<BashResult>;
  glob(pattern: string): Promise<string[]>;
  grep(pattern: string, options?: GrepOptions): Promise<GrepMatch[]>;
  getCwd(): string;
  setCwd(dir: string): void;
  listFiles(dir: string): Promise<string[]>;
}
```

Each method wraps a Node.js API with error handling, timeout support, and output truncation.

## Session lifecycle

`AgentSession` manages the complete session lifecycle:

```ts
class AgentSession {
  async start(config: SessionConfig): Promise<void> {
    // 1. Load or create session
    const session = config.sessionId
      ? await this.sessionManager.load(config.sessionId)
      : await this.sessionManager.create({ cwd: config.cwd });

    // 2. Select model
    const model = config.modelId
      ? getModel(config.modelId)
      : await this.selectModel(config);

    // 3. Load skills
    const skills = discoverSkills(config.skillPaths);

    // 4. Build system prompt
    const systemPrompt = buildSystemPrompt({
      template: CODING_AGENT_TEMPLATE,
      tools: this.getTools(),
      skills,
      cwd: session.cwd,
      date: new Date().toISOString().split("T")[0],
    });

    // 5. Initialize harness
    this.harness = new AgentHarness({
      model,
      tools: this.getTools(),
      systemPrompt,
      sessionDir: config.sessionDir,
    });

    // 6. Load messages
    await this.harness.loadSession(session.id);
  }

  async prompt(message: string): Promise<void> {
    this.harness.agent.prompt({
      role: "user",
      content: message,
      timestamp: Date.now(),
    });
  }

  async abort(): Promise<void> {
    this.harness.agent.abort();
  }

  async waitForIdle(): Promise<void> {
    await this.harness.agent.waitForIdle();
  }

  getMessages(): AgentMessage[] {
    return this.harness.agent.context.messages;
  }

  subscribe(fn: (event: AgentEvent) => void): () => void {
    return this.harness.agent.subscribe(fn);
  }
}
```

## Tool registration

Coding-specific tools are registered with the agent:

```ts
private getTools(): AgentTool[] {
  return [
    this.createReadTool(),
    this.createWriteTool(),
    this.createEditTool(),
    this.createBashTool(),
    this.createGlobTool(),
    this.createGrepTool(),
    this.createListTool(),
    // ... more tools
  ];
}
```

Each tool wraps a runtime service with schema validation and error handling. We'll build each tool in detail in the next chapter.

## What we've built

`AgentSession` is the shared foundation for all three coding-agent modes. It:

- Wires `AgentHarness` to Node.js runtime services
- Manages the session lifecycle (create, load, save)
- Registers coding-specific tools
- Discovers and loads skills
- Constructs the coding-agent system prompt
- Exposes `prompt()`, `abort()`, and `subscribe()` for mode implementations

---

← Previous: [Autocomplete and Assembling a Chat Interface](../terminal-ui/autocomplete-and-chat-interface.md) · Next: [Built-In Coding Tools: Bash, Read, Write, Edit, and Search](./built-in-tools.md) →
