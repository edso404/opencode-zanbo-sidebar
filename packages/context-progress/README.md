# @oh-my-sidebar/opencode-context-progress

[![npm](https://img.shields.io/npm/v/@oh-my-sidebar/opencode-context-progress)](https://www.npmjs.com/package/@oh-my-sidebar/opencode-context-progress)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](../../LICENSE)

OpenCode TUI sidebar plugin that displays the current session's context window usage as a progress bar with cost tracking.

Renders in the OpenCode sidebar as a native TUI widget:

```
Context
████████████████████░░░ 84%
12,847 / 200,000 / $0.42
```

## Features

- **Progress bar** — visual indicator of context window fill level (24-character granular bar)
- **Token count** — current session tokens used, formatted with locale-aware number formatting
- **Context window limit** — displays the model's maximum context window size (or `--` if unknown)
- **Session cost** — cumulative dollar cost for the current session
- **Color-coded warnings** — accent color normal, yellow at ≥70%, red at ≥90%

## How It Works

The plugin hooks into the OpenCode TUI `sidebar_content` slot and reads session state from the plugin API. It picks the most recent assistant message with output tokens, calculates usage against the model's advertised context window limit, and renders the bar + detail line reactively — updating in real time as new messages arrive.

## Prerequisites

- **OpenCode** >= 1.4.3 (TUI plugin system required)

## Installation

```bash
npm install @oh-my-sidebar/opencode-context-progress
```

## Usage

Add the plugin to your OpenCode TUI configuration (`tui.jsonc`):

```jsonc
{
  "plugin": [
    "@oh-my-sidebar/opencode-context-progress"
  ]
}
```

The plugin registers at slot `order: 100`, so it appears near the top of the sidebar (before `session-tokens` at order 120).

### Configuration

This plugin does not require any configuration. It reads all data from the OpenCode plugin API at runtime.

## Plugin API

| Export | Type | Description |
|--------|------|-------------|
| `default` | `TuiPluginModule & { id: string }` | Plugin entry point with ID `oh-my-sidebar.context-progress` |

### Internal Behavior

The plugin uses Solid.js `createMemo` for reactive rendering. The `View` component derives:

1. **Messages** from `api.state.session.messages(sessionID)`
2. **Session cost** from `api.state.session.get(sessionID)?.cost`, falling back to summing `message.cost` across assistant messages
3. **Usage** from the last assistant message with non-zero output tokens:
   - Looks up the message's provider + model to find the `contextWindow` limit
   - Computes `percent = tokens / contextWindow`
4. **Bar** rendered with block chars (`█` filled, `░` empty) and colored per threshold:
   - `< 70%`: accent color
   - `≥ 70%`: warning color (yellow)
   - `≥ 90%`: error color (red)

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
packages/context-progress/
├── src/
│   └── index.tsx          # Plugin implementation
├── dist/                  # Build output (dist/tui.tsx)
├── package.json
├── tsup.config.ts
├── tsconfig.json
└── README.md
```

## Keywords

`opencode` `opencode-plugin` `sidebar` `context` `progress` `tokens`

## License

MIT — see [LICENSE](../../LICENSE).
