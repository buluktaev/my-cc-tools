# my-cc-tools

Personal Claude Code customizations: skills, scripts, and workflows.

## Skills

- **codex-tmux-orchestration** — delegate tasks to OpenAI Codex CLI (GPT-5.3-Codex) agent via tmux for parallel AI execution

## Statusline

`statusline/statusline.sh` — bash statusline for Claude Code that shows:

- username and current model
- **5-hour usage limit** with time until reset
- **weekly usage limit** with days/hours until reset
- context window usage (color-coded: green → yellow → red)

Fetches usage data from Anthropic API via OAuth token stored in macOS Keychain, cached every 2 minutes.

### Installation

1. Copy `statusline/statusline.sh` to `~/.claude/statusline.sh`
2. Make it executable: `chmod +x ~/.claude/statusline.sh`
3. Add to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "/Users/YOUR_USERNAME/.claude/statusline.sh"
  }
}
```

Requires: `jq`, `python3`, `curl`, macOS Keychain with Claude Code credentials.

## Installation

Each skill folder contains its own README with installation instructions.
