# Codex CLI Orchestration via tmux

A Claude Code skill for delegating tasks to OpenAI Codex CLI agent (GPT-5.3-Codex) via tmux exec mode — enabling parallel AI execution with the latest OpenAI coding model.

## Problem

Codex CLI interactive TUI uses alternate screen buffer, which breaks `tmux capture-pane`. Unlike Gemini CLI, there is no reliable idle marker in Codex TUI output.

## Solution

Use **`codex exec` mode** via tmux: each task runs as a separate process. Completion is detected by the shell prompt reappearing (process exited). Optional JSONL output provides structured results.

## Structure

```
codex-tmux-orchestration/
└── SKILL.md    # Main skill with commands and workflow
```

## Installation

```bash
cp -r skills/codex-tmux-orchestration ~/.claude/skills/
```

## Requirements

- **tmux** — `brew install tmux` (macOS) or `apt install tmux` (Linux)
- **Codex CLI** — `npm install -g @openai/codex`
- **OpenAI account** with Plus/Pro/API access
- Claude Code running inside tmux session

## Quick Start

```bash
# 1. Open split pane with controlled shell
tmux split-window -h -d "PS1='$ ' bash --norc"

# 2. Send task (TWO separate calls!)
tmux send-keys -t {right} 'codex exec --full-auto --model gpt-5.3-codex "Build the app per PLAN.md"'
tmux send-keys -t {right} Enter

# 3. Poll for shell prompt (= task done)
while true; do
  output=$(tmux capture-pane -t {right} -p -S -5)
  echo "$output" | grep -qE '^\$ ' && break
  sleep 5
done
```

## Key Differences from gemini-tmux-orchestration

| Aspect | Gemini | Codex |
|--------|--------|-------|
| Mode | Interactive TUI (persistent) | exec mode (per-task process) |
| Idle marker | `"Type your message"` | Shell prompt `$` (process exits) |
| Loop detection | `potential loop` → send `2` | Not needed (process exits) |
| Session resume | N/A | `codex exec resume --last` |
| Output format | Terminal capture | Optional JSONL (structured) |
| Approval | `--yolo` | `--full-auto` or `--yolo` |

## Use Cases

- **Parallel execution** — Codex works in the background while Claude continues writing code; no waiting
- **Code review** — Claude writes code, Codex reviews it independently and writes findings to a file
- **Large codebase analysis and refactoring** — Codex reads entire codebase with its large context window, Claude applies targeted changes based on findings

## See Also

- [openai/codex on GitHub](https://github.com/openai/codex)
- [tmux Manual](https://man7.org/linux/man-pages/man1/tmux.1.html)
