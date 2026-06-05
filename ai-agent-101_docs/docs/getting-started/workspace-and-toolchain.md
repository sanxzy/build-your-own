---
title: "Setting Up the Workspace and Toolchain"
description: "How to lay out a four-package monorepo, configure shared TypeScript settings, wire inter-package dependencies, and run builds and tests before writing any agent code."
category: getting-started
type: how-to
tags: [monorepo, workspace, TypeScript, tsconfig, Node16, strict mode, build order, toolchain, package.json, erasableSyntaxOnly, vitest, biome, prerequisites, npm workspaces, rewriteRelativeImportExtensions, ESM, module resolution, llm-toolkit, agent-core, tui, coding-agent]
keywords: [monorepo setup, workspace configuration, TypeScript base config, Node16 module resolution, strict TypeScript, erasableSyntaxOnly option, inter-package dependencies, build script, test command, tsgo, shx, husky]
sources: [S3, S4, S25, S37, S47]
---

**TL;DR** — Before writing a single line of agent logic, we need a monorepo where four packages can depend on one another in a fixed order, all share the same TypeScript rules, and a single command builds or tests everything. This chapter walks through the workspace layout, the shared `tsconfig.base.json` and what each compiler option actually buys us, how each package wires itself to the layers below it, and which scripts to reach for.

# Setting Up the Workspace and Toolchain

In the [previous chapter](./what-we-are-building.md) we mapped out the four layers of the agent — `llm-toolkit` at the bottom, then `agent-core`, then `tui`, and finally `coding-agent` at the top — and established why that order matters. Now we need to give those four packages a home where they can find each other, agree on TypeScript settings, and be built and tested without ceremony.

We will work through four concerns in sequence:

1. The monorepo layout — four packages under one root, managed by npm workspaces.
2. The shared TypeScript configuration — what each option does and why it is set the way it is.
3. Inter-package dependency wiring — how each package declares that it sits on top of another.
4. Build and test commands — what to run, and in what order.

## Why a monorepo?

Before we look at files, it is worth asking: why put all four packages in one repository rather than four separate ones?

The answer is that our layers depend on each other at *development time*. When we change a type in `llm-toolkit`, we need `agent-core` to pick up that change immediately — not after publishing to a registry, waiting for a version bump, and running `npm install` in every downstream package. A monorepo with npm workspaces gives us *symlinked local packages*: Node resolves `llm-toolkit` to `packages/llm-toolkit/` in the same checkout, so a rebuild of one package is visible to all dependents straight away.

## The workspace layout

The root `package.json` declares which directories participate in the workspace (S3):

```json
{
  "name": "xzy-monorepo",
  "private": true,
  "type": "module",
  "workspaces": [
    "packages/*"
  ]
}
```

The `"workspaces": ["packages/*"]` glob tells npm to treat every subdirectory of `packages/` as a workspace member. In practice that means four packages:

| Directory | Package name | Role |
|---|---|---|
| `packages/llm-toolkit/` | `llm-toolkit` | Streaming LLM client and provider abstraction |
| `packages/agent-core/` | `agent-core` | Generic agent loop, session, and harness |
| `packages/tui/` | `tui` | Terminal UI with differential rendering |
| `packages/coding-agent/` | `coding-agent` | The CLI tool that wires all three together |

> Note: the root `package.json` also lists several extension-example directories as additional workspace members. We can ignore those for now — they are standalone examples, not layers of the agent itself.

`"private": true` tells npm never to publish the root package. Only the four inner packages are publishable. `"type": "module"` sets ESM as the default module format for the whole monorepo — every file is treated as an ES module unless its own `package.json` says otherwise.

## Prerequisites: the four-layer architecture

If you are arriving here directly, here is a quick orientation. The four packages form a strict dependency chain — `llm-toolkit` → `agent-core` → `tui` → `coding-agent` — where each layer depends only on the layers below it. No layer may import from a layer above it. This constraint is what makes the codebase testable in isolation and makes the layers individually reusable. For the full picture, see [What We Are Building: Architecture and the Four Layers](./what-we-are-building.md).

## Shared TypeScript configuration

Every package in the monorepo starts from the same base. The root `tsconfig.base.json` captures options that must be consistent across all four packages (S4):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "lib": ["ES2022"],
    "strict": true,
    "erasableSyntaxOnly": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "inlineSources": true,
    "inlineSourceMap": false,
    "moduleResolution": "Node16",
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "useDefineForClassFields": false,
    "types": ["node"]
  }
}
```

That is a lot of options. Let's work through the notable ones so they are not mysterious when you encounter them later.

### `"module": "Node16"` and `"moduleResolution": "Node16"`

These two always travel together. `"module": "Node16"` tells TypeScript how to *emit* import and export statements — it produces code that matches how Node.js actually loads ESM and CJS modules, including the rule that relative imports must carry a file extension (`.js`, not bare). `"moduleResolution": "Node16"` tells TypeScript how to *resolve* imports when it type-checks — it mirrors Node's own algorithm, including reading `"exports"` fields in `package.json`.

The practical effect: if you write `import { stream } from "./stream"` without an extension, TypeScript will complain. You must write `import { stream } from "./stream.js"` — even though the TypeScript file is `.ts`. This feels odd at first, but it means the compiled output needs no path rewriting; what TypeScript emits is what Node runs.

### `"strict": true`

This one umbrella flag enables a collection of checks that prevent whole categories of bugs: `strictNullChecks` (you cannot assign `null` or `undefined` to a variable that does not expect them), `noImplicitAny` (every value must have a known type), `strictFunctionTypes`, `strictBindCallApply`, and several others. For a tool that manipulates files and calls external APIs, these checks are not pedantry — they catch the class of bug where "this might be null" turns into a runtime crash at the worst moment.

### `"erasableSyntaxOnly": true`

This is a TypeScript 5.5+ option worth understanding because it directly shapes how we write code. When it is enabled, TypeScript rejects any syntax that cannot be removed by a simple erasure pass — in other words, syntax that requires *transformation*, not just stripping of types. The main thing this rules out is `enum` (which compiles to an object literal, not just a type erasure) and `namespace` (which compiles to an IIFE). It does not affect interfaces, type aliases, generics, or decorators in the way we use them here.

Why do we care? Because TypeScript 5.9+ ships a Go-based compiler (`tsgo`) that is designed to erase types without transforming code. Keeping to erasable syntax means our source files are compatible with that fast path, and with tools like `tsx` that also do erasure-only transpilation.

### `"rewriteRelativeImportExtensions": true`

When TypeScript compiles a `.ts` file, relative imports like `import { foo } from "./bar.js"` are emitted unchanged. The `.js` extension in the source is what Node expects at runtime. This option tells TypeScript to rewrite *extensionless* or `.ts`-suffixed imports in the output to their correct `.js` form. In practice it means we can write `import { foo } from "./bar.ts"` in source and have it land correctly at runtime — though the project convention (driven by `allowImportingTsExtensions`) is to write `.ts` in source and let this option fix it on emit.

### `"declaration": true` and `"declarationMap": true`

When we build `llm-toolkit` and `agent-core`, we need to produce `.d.ts` type declaration files so that the packages above them can type-check against the compiled output. `"declaration": true` emits those files; `"declarationMap": true` emits a `.d.ts.map` that lets an IDE jump from a declaration back to the original `.ts` source — which is far more useful than jumping to the generated `.d.ts`.

### `"sourceMap": true` and `"inlineSources": true`

Source maps let a debugger or error reporter translate a line number in compiled `.js` back to the original `.ts` source. `"inlineSources": true` embeds the original source text in the map, so the mapping works even when the `.ts` files are not present at runtime (which is the case after a clean install from the registry).

### `"useDefineForClassFields": false`

This controls how TypeScript emits class fields. The spec-compliant `define` semantics (the default when targeting ES2022+) can break classes that use `emitDecoratorMetadata`, because the field initializer runs before metadata is attached. Setting this to `false` reverts to the older `assign` semantics that decorators expect.

### The full option table

| Option | What it does |
|---|---|
| `target: "ES2022"` | Emit modern JS; async/await, `at()`, `Object.hasOwn` etc. are native |
| `module: "Node16"` | Emit ESM/CJS as Node expects; enforces extension on relative imports |
| `moduleResolution: "Node16"` | Resolve imports the way Node does, reading `"exports"` in `package.json` |
| `strict: true` | Enable all strictness checks (nullability, implicit-any, …) |
| `erasableSyntaxOnly: true` | Ban non-erasable syntax (enum, namespace) for fast-path TS tooling |
| `esModuleInterop: true` | Allow `import fs from "fs"` syntax for CJS modules |
| `skipLibCheck: true` | Do not type-check `.d.ts` files in `node_modules` |
| `forceConsistentCasingInFileNames: true` | Catch case-insensitive filesystem bugs on macOS |
| `declaration: true` | Emit `.d.ts` files so consumers can type-check |
| `declarationMap: true` | Emit source maps for `.d.ts` — IDE "go to definition" lands on `.ts` |
| `sourceMap: true` | Emit `.js.map` for debuggers and error reporters |
| `inlineSources: true` | Embed original source text in source maps |
| `inlineSourceMap: false` | Keep source maps as separate `.map` files (not inlined in `.js`) |
| `resolveJsonModule: true` | Allow `import data from "./data.json"` |
| `allowImportingTsExtensions: true` | Allow writing `./foo.ts` in imports |
| `rewriteRelativeImportExtensions: true` | Fix `.ts` extensions to `.js` in emitted output |
| `experimentalDecorators: true` | Enable the stage-2 decorator syntax |
| `emitDecoratorMetadata: true` | Emit type metadata for `reflect-metadata` patterns |
| `useDefineForClassFields: false` | Use assign semantics for class fields (required with decorators) |
| `types: ["node"]` | Include Node.js global types (`process`, `Buffer`, …) |

Each package's own `tsconfig.json` extends this base and adds only the entries unique to that package (typically `outDir`, `rootDir`, and which files to include or exclude).

## Wiring packages together

Now that the four packages share a TypeScript config, we need the upper layers to declare that they depend on the layers below. Let's follow the chain.

### `agent-core` depends on `llm-toolkit`

`agent-core` (at `packages/agent-core/`) lists `llm-toolkit` as a runtime dependency (S25):

```json
{
  "name": "agent-core",
  "dependencies": {
    "llm-toolkit": "^0.1.0"
  }
}
```

> Note: the version numbers in this chapter are minimal starting values for a brand-new project — `0.1.0` is the conventional first version, and `^0.1.0` is a semver range that permits compatible minor and patch updates. Pick whatever versions you like; nothing here depends on a specific one.

Because npm workspaces symlink local packages, `"llm-toolkit": "^0.1.0"` resolves to `packages/llm-toolkit/` on disk rather than reaching out to a registry — as long as the version in `packages/llm-toolkit/package.json` satisfies the range. This is how `agent-core` gets live, un-published access to the layer below it.

The full dependency declaration for `agent-core` also lists three runtime libraries it needs directly (S25):

```json
{
  "dependencies": {
    "llm-toolkit": "^0.1.0",
    "ignore": "^7.0.0",
    "typebox": "^1.0.0",
    "yaml": "^2.0.0"
  }
}
```

`ignore` is used for `.gitignore`-style pattern matching, `typebox` provides runtime-validated type schemas, and `yaml` handles YAML parsing. The version ranges shown here are illustrative caret ranges — use whatever current versions you install. (In a production project you might later pin exact versions and commit a lockfile so installs are reproducible.)

### `tui` — standalone at its layer

`tui` (at `packages/tui/`) does not depend on `llm-toolkit` or `agent-core` — it is a terminal rendering library with its own concerns (S37). Its runtime dependencies are:

```json
{
  "dependencies": {
    "get-east-asian-width": "^1.0.0",
    "marked": "^15.0.0"
  }
}
```

`get-east-asian-width` handles correct terminal column widths for CJK characters; `marked` is a Markdown parser used to render formatted output. Neither depends on any layer above or below.

This is worth noticing: even though `tui` is above `llm-toolkit` and `agent-core` in the layer diagram, it has no compile-time dependency on them. `coding-agent` is the one that pulls all three together.

### `coding-agent` depends on everything

The top layer (`packages/coding-agent/`) is the one that assembles all three packages below it (S47):

```json
{
  "dependencies": {
    "agent-core": "^0.1.0",
    "llm-toolkit": "^0.1.0",
    "tui": "^0.1.0"
  }
}
```

`coding-agent` also carries a large list of additional runtime dependencies (chalk, cross-spawn, diff, glob, yaml, undici, and others — S47). We will encounter those as we build out the tools and session management in later chapters. For now the key point is the three inter-package dependencies above: they confirm that `coding-agent` sits on top of every other layer and is the only package that touches all three.

### The dependency graph

```
coding-agent
├── llm-toolkit
├── agent-core
│   └── llm-toolkit
└── tui
```

Notice that `llm-toolkit` appears twice — once as a direct dependency of `coding-agent` and once as a transitive dependency through `agent-core`. npm deduplicates this automatically, so there is only one copy on disk, and TypeScript resolves both references to the same module.

## Runtime requirement

Every package declares the same minimum Node version (S3, S25, S37, S47):

```json
{
  "engines": {
    "node": ">=22.0.0"
  }
}
```

A recent Node 22 (or newer) is required because the project uses native ESM, runtime type-stripping for `.ts` files (the `tsx` runner uses a similar erasure approach), and other modern-runtime features. Make sure `node --version` returns `22.0.0` or higher before continuing — newer is fine.

## Build and test commands

### The build script

Building the monorepo is not as simple as running one command at the root — TypeScript needs to compile the lower layers before the upper layers can type-check their imports. The root `package.json` encodes the correct order (S3):

```json
{
  "scripts": {
    "build": "cd packages/tui && npm run build && cd ../llm-toolkit && npm run build && cd ../agent-core && npm run build && cd ../coding-agent && npm run build"
  }
}
```

Working through this command:

1. Build `tui` first (it has no inter-package dependencies, so it is safe to build in any position, but it is conventionally built first).
2. Build `llm-toolkit` (`packages/llm-toolkit/`).
3. Build `agent-core` (`packages/agent-core/`), which can now find the compiled `llm-toolkit`.
4. Build `coding-agent` (`packages/coding-agent/`), which can now find all three compiled layers.

Each package's own `build` script calls `tsgo` (the Go-based TypeScript compiler) with its local `tsconfig.build.json`. `coding-agent` also runs `shx chmod +x dist/cli.js` to make the CLI executable and copies static assets (themes, templates) into `dist/`.

To build the whole monorepo from the repository root:

```bash
npm run build
```

To build a single package — useful during development when you know only one layer changed:

```bash
cd packages/llm-toolkit && npm run build
```

### The test script

Running tests at the monorepo root delegates to each workspace in turn (S3):

```bash
npm run test
```

This expands to `npm run test --workspaces --if-present`, which runs each package's own `test` script if it has one. The `--if-present` flag silently skips packages that do not define a `test` script — so adding a new package without tests will not break the root command.

Individual package test setups differ by layer:

| Package | Test runner | Command |
|---|---|---|
| `llm-toolkit` | vitest | `vitest --run` |
| `agent-core` | vitest | `vitest --run` |
| `tui` | Node test runner | `node --test test/*.test.ts` |
| `coding-agent` | vitest | `vitest --run` |

`vitest` is a Vite-powered test runner with first-class TypeScript support. The `tui` package uses Node's built-in `--test` runner instead, which is lighter weight for a package that has no transitive build dependencies.

### The `check` script

There is one more useful root-level command (S3):

```json
{
  "scripts": {
    "check": "biome check --write --error-on-warnings . && npm run check:pinned-deps && npm run check:ts-imports && npm run check:shrinkwrap && tsgo --noEmit && npm run check:browser-smoke"
  }
}
```

`biome` is the linter and formatter. `biome check --write --error-on-warnings` applies auto-fixes and fails if any warning remains — so the codebase is either clean or the check fails, with no "warnings I'll fix later". After that, the script runs several project-specific validators: pinned-dependency checks, TypeScript import sanity checks, a shrinkwrap consistency check, a full type-check pass via `tsgo --noEmit`, and a browser smoke test. You do not need to understand all of these now; just know that `npm run check` is the gate that must pass before any change ships.

### The `clean` script

When a build behaves unexpectedly, the first thing to try is a clean build (S3):

```json
{
  "scripts": {
    "clean": "npm run clean --workspaces"
  }
}
```

This delegates to each package's own `clean` script, which removes its `dist/` directory. After running `npm run clean`, a subsequent `npm run build` starts from scratch.

## Putting it together: setting up a fresh workspace

If you are creating a four-package monorepo from scratch that mirrors this structure, here is the skeleton that maps to everything we covered.

**Root `package.json`:**

```json
{
  "name": "my-agent-monorepo",
  "private": true,
  "type": "module",
  "workspaces": ["packages/*"],
  "scripts": {
    "build": "cd packages/tui && npm run build && cd ../llm-toolkit && npm run build && cd ../agent-core && npm run build && cd ../coding-agent && npm run build",
    "test": "npm run test --workspaces --if-present",
    "clean": "npm run clean --workspaces"
  },
  "engines": {
    "node": ">=22.0.0"
  },
  "devDependencies": {
    "typescript": "^5.9.0"
  }
}
```

**Root `tsconfig.base.json`:** (copy the full contents shown in the [Shared TypeScript configuration](#shared-typescript-configuration) section above).

**Each package's `package.json`** declares:
- Its own `name` and `version`.
- `"type": "module"`.
- Its inter-package dependencies by name (e.g. `"agent-core": "^0.x.x"`).
- A `build` script that calls `tsgo -p tsconfig.build.json` (or `tsc`).
- A `test` script appropriate to the layer.
- `"engines": { "node": ">=22.0.0" }`.

**Each package's `tsconfig.json`** extends the base:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

After running `npm install` from the root (which sets up the workspace symlinks), the four packages can import from each other and `npm run build` will compile them in the right order.

## What we have

At this point we have a functioning workspace:

- Four packages under `packages/`, linked via npm workspaces.
- A shared `tsconfig.base.json` with strict, modern, erasable-only TypeScript settings.
- Inter-package `"dependencies"` that match the layer order: `llm-toolkit` at the base, `agent-core` on top of it, `coding-agent` on top of everything.
- A build script that compiles in dependency order, a test script that delegates to each package, and a check script that enforces formatting and type correctness.

In the next chapter we step into `llm-toolkit` and look at the message types and streaming API that everything above it depends on.

---

← Previous: [What We Are Building: Architecture and the Four Layers](./what-we-are-building.md) · Next: [Message Types and the Unified Streaming API](../llm-toolkit/message-types-and-streaming-api.md) →
