---
title: "Setting Up the Workspace and Toolchain"
description: "Create the monorepo workspace with npm workspaces, configure TypeScript with strict mode and Node16 modules, and set up the build, test, and lint toolchain you'll use throughout the build."
category: getting-started
type: how-to
tags: [monorepo, workspace, TypeScript, tsconfig, Node16, strict mode, build order, toolchain, package.json, vitest, biome, npm workspaces, Node.js, prerequisites]
keywords: [setup, npm workspaces, tsconfig, esm, Node16, erasableSyntaxOnly, monorepo structure]
sources: [S3, S4, S18, S33, S57]
---

**TL;DR** — Before we write any agent code, we need a workspace. We'll create a monorepo with four packages (`llm-toolkit`, `agent-core`, `tui`, `coding-agent`), configure TypeScript with strict mode and Node16 module resolution, and set up our build pipeline. By the end of this chapter, you'll be able to run `npm run build` across all packages and have a clean TypeScript project ready for development.

## Why a monorepo?

Our AI coding agent has four layers, each a separate package with its own dependencies, tests, and build configuration. We could put them in four separate git repositories, but that makes development painful — you'd need to `npm link` between them, version-coordinate releases, and run tests across repos.

A monorepo gives us a single place where all four packages live together, share tooling configuration, and can reference each other directly during development. npm workspaces (built into npm since version 7) handle the linking automatically.

Here's the directory structure we're building toward:

```
best-ai-agent/
├── package.json              # root workspace config
├── tsconfig.base.json        # shared TypeScript settings
├── biome.json                # linter + formatter config
├── packages/
│   ├── llm-toolkit/          # Layer 1: unified LLM API
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   ├── agent-core/           # Layer 2: agent loop + harness
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   ├── tui/                  # Layer 3: terminal UI
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   └── coding-agent/         # Layer 4: CLI assembly
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
└── tsconfig.json             # root tsconfig (references base)
```

## Step 1: Create the root workspace

Start with an empty directory and initialize it as an npm package:

```bash
mkdir best-ai-agent && cd best-ai-agent
npm init -y
```

Now edit `package.json` to declare the workspace. The key field is `workspaces`, which tells npm which directories contain packages:

```json
{
  "name": "best-ai-agent",
  "private": true,
  "type": "module",
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "check": "biome check --write --error-on-warnings . && tsgo --noEmit",
    "test": "npm run test --workspaces --if-present",
    "clean": "npm run clean --workspaces --if-present"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.3.0",
    "@types/node": "^22.0.0",
    "typescript": "^5.9.0",
    "vitest": "^3.2.0"
  },
  "engines": {
    "node": ">=22.0.0"
  }
}
```

A few things to notice:

- **`"private": true`** — the root package is never published. It only exists to coordinate the workspace.
- **`"type": "module"`** — we're using ES modules (import/export syntax), not CommonJS (require). This is the modern standard.
- **`"workspaces": ["packages/*"]`** — every directory under `packages/` is a workspace package. npm will symlink them so they can import each other directly.
- **`--workspaces`** in scripts — runs the command in every workspace package that has that script defined.
- **`vitest`** — our test runner (fast, compatible with TypeScript, built on Vite).
- **`biome`** — our linter and formatter (fast, zero-config for TypeScript).

Run `npm install` to set up the workspace:

```bash
npm install
```

## Step 2: Configure TypeScript

Our TypeScript configuration has two files: a **base config** that all packages share, and a **root config** that references it.

Create `tsconfig.base.json` at the root:

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
    "moduleResolution": "Node16",
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "types": ["node"]
  }
}
```

Let's understand the important settings:

| Setting | What it does | Why we need it |
|---|---|---|
| `target: "ES2022"` | Compiles to ES2022 JavaScript | Node.js 22+ supports ES2022 natively |
| `module: "Node16"` | Uses Node.js native ESM | Required for `import`/`export` in Node.js |
| `strict: true` | Enables all strict type checks | Catches bugs at compile time, not runtime |
| `erasableSyntaxOnly: true` | Disallows enums, namespaces, decorators | Keeps TypeScript as "types only" — no runtime codegen |
| `declaration: true` | Generates `.d.ts` files | Package consumers get type information |
| `moduleResolution: "Node16"` | Resolves imports the Node.js way | Matches `module: "Node16"` |
| `allowImportingTsExtensions: true` | Allows `.ts` in import paths | Lets us write `import "./foo.ts"` |
| `rewriteRelativeImportExtensions: true` | Rewrites `.ts` to `.js` in output | Required because Node.js loads `.js`, not `.ts` |

The `erasableSyntaxOnly` setting deserves attention. TypeScript has features that don't just add types — they generate runtime code. Enums, namespaces, parameter properties, and `experimentalDecorators` all produce JavaScript that TypeScript has to emit. With `erasableSyntaxOnly: true`, we forbid all of those. Our `.ts` files contain only standard JavaScript with type annotations — strip the types, and you have valid JS. This keeps our code predictable and our output clean.

Now create `tsconfig.json` at the root:

```json
{
  "extends": "./tsconfig.base.json",
  "files": [],
  "references": [
    { "path": "./packages/llm-toolkit" },
    { "path": "./packages/agent-core" },
    { "path": "./packages/tui" },
    { "path": "./packages/coding-agent" }
  ]
}
```

The `references` field tells TypeScript this is a **project reference** setup — each package builds independently, and the root `tsgo --noEmit` type-checks them all. The `files: []` means the root config itself checks no files (it delegates to the references).

## Step 3: Configure Biome (linting and formatting)

Create `biome.json` at the root:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.3.5/schema.json",
  "formatter": {
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "linter": {
    "rules": {
      "recommended": true
    }
  },
  "javascript": {
    "formatter": {
      "semicolons": "always",
      "trailingCommas": "all",
      "quoteStyle": "double"
    }
  }
}
```

Biome gives us fast, opinionated formatting and linting. It replaces ESLint + Prettier with a single tool that's 10–100× faster.

## Step 4: Create each package

Now we'll scaffold the four packages. Each one follows the same pattern:

```
packages/<name>/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts
```

### `packages/llm-toolkit/package.json`

```json
{
  "name": "llm-toolkit",
  "version": "0.1.0",
  "description": "Unified LLM API with provider adapters",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    }
  },
  "scripts": {
    "clean": "rm -rf dist",
    "build": "tsgo -p tsconfig.build.json",
    "test": "vitest --run"
  }
}
```

### `packages/llm-toolkit/tsconfig.json`

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src"]
}
```

Each package's tsconfig extends the base config and adds `outDir` (where compiled JS goes) and `rootDir` (where source TS lives). The base config handles all the strictness and module settings; each package only adds output location and file scope.

We'll also need `tsconfig.build.json` in each package for the build step:

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

The build config excludes test files so they don't end up in `dist/`.

### The other three packages

The `agent-core`, `tui`, and `coding-agent` packages follow the same template. The only difference is in dependencies — `agent-core` depends on `llm-toolkit`, `coding-agent` depends on all three:

**`packages/agent-core/package.json`** adds:
```json
"dependencies": {
  "llm-toolkit": "^0.1.0"
}
```

**`packages/coding-agent/package.json`** adds:
```json
"dependencies": {
  "llm-toolkit": "^0.1.0",
  "agent-core": "^0.1.0",
  "tui": "^0.1.0"
}
```

Because we're using npm workspaces, `"llm-toolkit": "^0.1.0"` resolves to the local package in `packages/llm-toolkit/` automatically — no `npm link` needed.

### Verify the structure

After creating all four packages, your directory should look like this:

```
best-ai-agent/
├── package.json
├── tsconfig.base.json
├── tsconfig.json
├── biome.json
├── node_modules/
└── packages/
    ├── llm-toolkit/
    │   ├── package.json
    │   ├── tsconfig.json
    │   ├── tsconfig.build.json
    │   └── src/index.ts
    ├── agent-core/
    │   ├── package.json
    │   ├── tsconfig.json
    │   ├── tsconfig.build.json
    │   └── src/index.ts
    ├── tui/
    │   ├── package.json
    │   ├── tsconfig.json
    │   ├── tsconfig.build.json
    │   └── src/index.ts
    └── coding-agent/
        ├── package.json
        ├── tsconfig.json
        ├── tsconfig.build.json
        └── src/index.ts
```

Run `npm install` again to link the workspace packages:

```bash
npm install
```

Now verify everything type-checks:

```bash
npx tsgo --noEmit
```

You should see no errors. The packages are empty for now, but the toolchain is ready.

## The build order

Our packages depend on each other in a chain:

```
llm-toolkit  ←  agent-core  ←  coding-agent
                 tui         ←  coding-agent
```

The `tui` package is independent — it doesn't need any of the other packages. `agent-core` needs `llm-toolkit`. `coding-agent` needs everything.

When we run `npm run build`, npm workspaces builds packages in dependency order automatically. But it's worth understanding the chain: when we write code in `llm-toolkit`, we run `npm run build` there first, then move up the chain. Each layer is testable before the next one exists.

## What we have so far

At this point we have:

- A **monorepo** with four packages linked by npm workspaces
- **TypeScript** configured with strict mode, Node16 ESM, and project references
- **Biome** for linting and formatting
- **Vitest** for testing
- A **build pipeline** that compiles all packages in dependency order

No agent code yet — that starts in the next chapter, where we'll define the core type system for talking to LLMs.

---

← Previous: [What We're Building: The AI Coding Agent Architecture](./what-we-are-building.md) · Next: [Message Types and the Core Streaming API](../llm-toolkit/message-types-and-core-api.md) →
