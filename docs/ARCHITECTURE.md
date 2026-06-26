# Architecture

This document describes the architecture of the `@oh-my-sidebar` monorepo, the OpenCode TUI plugin system it targets, and the design decisions behind each plugin.

## Overview

`@oh-my-sidebar` is a monorepo of [OpenCode](https://github.com/opencopilotdev/opencode) TUI sidebar plugins. Each plugin is an independent npm package that hooks into the OpenCode TUI's plugin slot system to render real-time telemetry widgets.

```
┌─────────────────────────────────────────────┐
│  OpenCode TUI                                │
│  ┌─────────────────────────────────────────┐ │
│  │ Sidebar                                  │ │
│  │  ┌─────────────────────────────────────┐ │ │
│  │  │ context-progress widget (order 100) │ │ │
│  │  │   Context                           │ │ │
│  │  │   ████████████████░░░░░░░░ 66%      │ │ │
│  │  │   12,847 / 200,000 / $0.42          │ │ │
│  │  ├─────────────────────────────────────┤ │ │
│  │  │ session-tokens widget (order 120)   │ │ │
│  │  │   ▶ Session Tokens  42,831          │ │ │
│  │  └─────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Monorepo Structure

```
oh-my-sidebar/
├── packages/
│   ├── context-progress/     # npm: @oh-my-sidebar/opencode-context-progress
│   └── session-tokens/       # npm: @oh-my-sidebar/opencode-session-tokens
├── .changeset/               # Changeset config + pending changesets
├── .github/workflows/
│   ├── ci.yml                # CI: lint → typecheck → test → build
│   └── release.yml           # Release: changesets publish to npm
├── pnpm-workspace.yaml       # Workspace definition + dependency catalog
├── turbo.json                 # Turborepo task pipeline
├── tsconfig.base.json         # Shared TypeScript config
├── biome.json                 # Biome lint/format config
├── package.json               # Root scripts + devDependencies
├── AGENTS.md                  # AI agent workspace instructions (auto-generated)
├── CONTRIBUTING.md            # Contribution guide
└── README.md                  # Project overview
```

## Plugin Architecture

### Plugin Lifecycle

```
1. OpenCode loads tui.jsonc → reads plugin list
2. For each plugin: imports `@oh-my-sidebar/opencode-<name>/tui`
3. Calls the default export's `tui` function with `TuiPluginApi`
4. Plugin registers a slot handler for `sidebar_content` with an `order`
5. OpenCode renders the sidebar slot → calls each handler with session context
6. Handler returns JSX (Solid.js) → rendered as native TUI components
7. Reactivity: Solid.js `createMemo` tracks API state changes → re-renders automatically
```

### Plugin Contract

Every plugin must export a default `TuiPluginModule`:

```typescript
interface TuiPluginModule {
  tui: (api: TuiPluginApi) => void | Promise<void>;
}
```

The `tui` function receives the plugin API and registers slot handlers. The module must also include a unique `id` string:

```typescript
const plugin: TuiPluginModule & { id: string } = {
  id: "oh-my-sidebar.<name>",
  tui,
};
export default plugin;
```

### Slot Registration

Plugins register for the `sidebar_content` slot:

```typescript
api.slots.register({
  order: 100,                          // Render order (lower = first)
  slots: {
    sidebar_content(_ctx, props) {      // props.session_id available
      return <View api={api} sessionID={props.session_id} />;
    },
  },
});
```

### Dependency Stack

| Dependency | Version | Purpose |
|------------|---------|---------|
| `@opencode-ai/plugin` | ^1.17.10 | OpenCode plugin API types + runtime |
| `@opentui/core` | ^0.4.2 | Core TUI rendering primitives (box, text) |
| `@opentui/solid` | ^0.4.2 | Solid.js bindings for OpenTUI |
| `solid-js` | 1.9.12 | Reactive UI library |

### Build Model

Plugins are **not bundled**. The build step copies `src/index.tsx` to `dist/tui.tsx`:

```bash
rm -rf dist && mkdir -p dist && cp src/index.tsx dist/tui.tsx
```

OpenCode compiles the TSX at runtime. The `tsup.config.ts` exists primarily for type declaration generation (`dts: true`), which enables editor intellisense for consumers.

The `package.json#exports` field maps `./tui` → `./dist/tui.tsx`:

```json
{
  "exports": {
    "./tui": "./dist/tui.tsx"
  }
}
```

## Plugin Details

### context-progress (order: 100)

**Purpose:** Visualize context window fill level with cost tracking.

**Data flow:**
1. Read messages from `api.state.session.messages(sessionID)`
2. Find the most recent assistant message with output tokens > 0
3. Sum all token categories (input + output + reasoning + cache read + cache write)
4. Look up the model's context window limit from provider state
5. Compute percentage and render progress bar

**Cost tracking:**
- Primary: `api.state.session.get(sessionID)?.cost` (fast path)
- Fallback: Sum `message.cost` across assistant messages (compatibility)

**Thresholds:**
| Usage | Color |
|-------|-------|
| < 70% | `theme().accent` |
| ≥ 70% | `theme().warning` |
| ≥ 90% | `theme().error` |

### session-tokens (order: 120)

**Purpose:** Break down token usage by model with collapsible detail.

**Data flow:**
1. Iterate all session messages
2. Filter to assistant messages with token counts > 0
3. Deduplicate by message ID (handles overlapping message sources)
4. Accumulate tokens per model ID
5. Sort models by token count descending
6. Render total in header row, per-model breakdown in expanded view

**Token accounting:**
- Counts: input + output + reasoning + cache write
- Cache read is excluded (per OpenCode convention, cache reads don't consume new tokens)
- Falls back to `session()?.tokens` when per-message iteration yields no results

**UI limits:**
- Max 10 model rows displayed; excess shows "+N more"
- Model labels truncated at 28 chars (25 + "...")

## CI/CD Pipeline

### CI (ci.yml)

Triggers on push/PR to `main`:

```
checkout → pnpm setup → pnpm install --frozen-lockfile
  → pnpm lint (Biome)
  → pnpm typecheck (tsc --noEmit)
  → pnpm test (Vitest)
  → pnpm build (Turbo)
```

### Release (release.yml)

Triggers on push to `main`:

```
checkout → pnpm setup → pnpm install --frozen-lockfile
  → changesets/action:
    → Creates/updates a Version PR (bumps versions + changelogs)
    → On merge: publishes to npm via pnpm ci:publish
```

Publishing uses **OIDC trusted publishing** — no npm token stored in secrets.

## Dependency Catalog

All shared dependencies are locked in `pnpm-workspace.yaml` under the `catalog` key with `catalogMode: strict`. This means:

- Every dependency must be defined in the catalog before it can be used
- Versions are consistent across all packages
- Adding a new dependency requires updating the catalog first

## Adding a New Plugin

1. Copy `packages/context-progress` as a template
2. Update `package.json`: `name`, `description`, `keywords`, `repository.directory`
3. Implement the plugin in `src/index.tsx` with a unique ID (`oh-my-sidebar.<name>`)
4. Update root `CONTRIBUTING.md` if conventions differ
5. Add catalog entries to `pnpm-workspace.yaml` for new dependencies
6. Create a changeset

See [CONTRIBUTING.md](../CONTRIBUTING.md) for the full contribution workflow.
