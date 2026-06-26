# @oh-my-sidebar/opencode-session-tokens

[![npm](https://img.shields.io/npm/v/@oh-my-sidebar/opencode-session-tokens)](https://www.npmjs.com/package/@oh-my-sidebar/opencode-session-tokens)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](../../LICENSE)

OpenCode TUI sidebar plugin that displays the current session's token usage breakdown by model.

Renders in the OpenCode sidebar as a native TUI widget with expandable detail:

```
▶ Session Tokens  42,831
```

Click to expand:

```
▼ Session Tokens  42,831
  claude-sonnet-4-20250514  28,100
  claude-3-opus-20240229    14,731
```

## Features

- **Total session token count** — aggregate across all models in the session
- **Per-model breakdown** — token counts grouped by model ID, sorted descending
- **Expandable/collapsible view** — toggle details with mouse click
- **Comprehensive token accounting** — includes input, output, reasoning, and cache write tokens
- **10-row cap** — shows up to 10 models, with a "+N more" indicator for overflow
- **Duplicate message deduplication** — skips duplicate message entries by ID for accurate totals

## How It Works

The plugin hooks into the OpenCode TUI `sidebar_content` slot. It iterates over all assistant messages in the current session, sums token counts per model (input + output + reasoning + cache write), and renders a collapsible breakdown.

The collapsible header acts as a button — click to toggle the detail view open/closed.

## Prerequisites

- **OpenCode** >= 1.4.3 (TUI plugin system required)

## Installation

```bash
npm install @oh-my-sidebar/opencode-session-tokens
```

## Usage

Add the plugin to your OpenCode TUI configuration (`tui.jsonc`):

```jsonc
{
  "plugin": [
    "@oh-my-sidebar/opencode-session-tokens"
  ]
}
```

The plugin registers at slot `order: 120`, so it appears below `context-progress` (order 100) in the sidebar.

### Configuration

This plugin does not require any configuration. It reads all data from the OpenCode plugin API at runtime.

## Plugin API

| Export | Type | Description |
|--------|------|-------------|
| `default` | `TuiPluginModule & { id: string }` | Plugin entry point with ID `oh-my-sidebar.session-tokens` |

### Internal Behavior

The plugin uses Solid.js `createMemo` for reactive rendering. The `View` component:

1. **Data derivation** — iterates over session messages, filters to assistant messages, deduplicates by message ID, and sums tokens (input + output + reasoning + cache write) grouped by `modelID` / `info.modelID`
2. **Total fallback** — if the per-message iteration yields zero (e.g., no assistant messages parsed), falls back to `session().tokens` from the session state object
3. **Collapsible UI** — uses a Solid.js `createSignal` for open/closed state, rendered as a `<button>` with a `▶`/`▼` toggle
4. **Model label truncation** — labels longer than 28 characters are truncated to 25 chars with `...`

### Token Categories Counted

| Category | Source Field |
|----------|-------------|
| Input | `tokens.input` |
| Output | `tokens.output` |
| Reasoning | `tokens.reasoning` |
| Cache write | `tokens.cache.write` |

## Development

```bash
# Install dependencies (from monorepo root)
pnpm install

# Build
pnpm build

# Watch mode
pnpm dev
```

### Project Structure

```
packages/session-tokens/
├── src/
│   └── index.tsx          # Plugin implementation
├── dist/                  # Build output (dist/tui.tsx)
├── package.json
├── tsup.config.ts
├── tsconfig.json
└── README.md
```

## Keywords

`opencode` `opencode-plugin` `sidebar` `tokens` `session` `model`

## License

MIT — see [LICENSE](../../LICENSE).
