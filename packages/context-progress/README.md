# @oh-my-sidebar/opencode-context-progress

OpenCode TUI sidebar plugin that displays the current session's context usage progress.

## Features

- Real-time context window usage bar
- Token count and context window limit display
- Session cost tracking
- Color-coded warnings (yellow at 70%, red at 90%)

## Installation

```bash
npm install @oh-my-sidebar/opencode-context-progress
```

## Usage

Register the plugin in your OpenCode configuration:

```json
{
  "plugins": {
    "oh-my-sidebar.context-progress": {
      "tui": "@oh-my-sidebar/opencode-context-progress/tui"
    }
  }
}
```

## License

MIT
