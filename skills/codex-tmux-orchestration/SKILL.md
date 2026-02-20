---
name: codex-tmux-orchestration
description: Use when delegating coding tasks to OpenAI Codex CLI agent (GPT-5.3-Codex), when you need parallel AI execution, or when tasks benefit from a separate OpenAI agent running alongside Claude - orchestrates Codex via tmux exec mode since TUI uses alternate screen buffer incompatible with capture-pane
---

# Codex CLI Orchestration via tmux

## Overview

Codex CLI TUI (`codex`) uses alternate screen buffer — `tmux capture-pane` captures nothing useful. Use **`codex exec` mode** via tmux: reliable process-based completion detection + optional JSONL output.

tmux runs in the **background** (detached, `-d` flag) — no new terminal window opens. Claude Code controls Codex invisibly via bash commands.

## Quick Reference

| Action | Command |
|--------|---------|
| Start background session | `tmux new-session -d -s codex -x 200 -y 50` |
| Setup shell | `tmux send-keys -t codex "PS1='$ ' bash --norc" Enter` |
| Send task | `tmux send-keys -t codex 'codex exec --full-auto --skip-git-repo-check --model gpt-5.3-codex "task"' ""` |
| Send Enter | `tmux send-keys -t codex Enter` |
| Check output | `tmux capture-pane -t codex -p -S -100` |
| Kill session | `tmux kill-session -t codex` |

**Important:** `send-keys` for text and Enter must be **two separate calls**.

## Why exec mode, not interactive TUI

| Mode | tmux capture-pane | Completion detection | File writes |
|------|-------------------|----------------------|-------------|
| `codex` (TUI) | Broken (alt screen) | No reliable marker | Yes |
| `codex exec` | Shell prompt `$` | Process exit | Yes |
| `codex exec --json` | Shell prompt `$` | Process exit + JSONL | Yes |

## Key Flags

| Flag | Purpose |
|------|---------|
| `--full-auto` | Auto-approve all actions + sandbox workspace-write |
| `--skip-git-repo-check` | Allow running outside a git repository (required when CWD has no git) |
| `--model gpt-5.3-codex` | Use GPT-5.3-Codex model |
| `--cd /path` | Set working directory for the agent |

## Status Markers (exec mode)

| Marker | State |
|--------|-------|
| `$ ` shell prompt visible | Idle — process exited, task done |
| `$ codex exec ...` still running | Working |
| `error:` in output | Task failed |
| `tokens used` line | Task completed, shows token count |

## Core Workflow

```bash
# 1. Create background tmux session (no new window opens!)
tmux new-session -d -s codex -x 200 -y 50
sleep 1

# 2. Setup clean shell for reliable prompt detection
tmux send-keys -t codex "PS1='$ ' bash --norc" Enter
sleep 1

# 3. Send task (TWO separate calls - critical!)
tmux send-keys -t codex 'codex exec --full-auto --skip-git-repo-check --model gpt-5.3-codex "your task here"' ""
tmux send-keys -t codex Enter

# 4. Poll for completion (shell prompt = process exited = done)
while true; do
  output=$(tmux capture-pane -t codex -p -S -10)
  last_line=$(echo "$output" | grep -v '^$' | tail -1)

  # Idle = short $ prompt OR prompt after tokens used line
  if echo "$last_line" | grep -qE '^\$$|^\$ $'; then
    break
  fi

  # Error detection
  if echo "$output" | grep -q "^error:"; then
    echo "Codex error detected"
    break
  fi

  sleep 5
done

# 5. Read result and clean up
tmux capture-pane -t codex -p -S -200
tmux kill-session -t codex
```

## Code Review Use Case (main use case)

Claude writes code → Codex reviews it in the background:

```bash
# After Claude finishes writing code, send to Codex for review

# 1. Start background Codex session
tmux new-session -d -s codex-review -x 200 -y 50
sleep 1
tmux send-keys -t codex-review "PS1='$ ' bash --norc" Enter
sleep 1

# 2. Send review task pointing at the project directory
tmux send-keys -t codex-review 'codex exec --full-auto --model gpt-5.3-codex --cd /path/to/project "Review the code changes in src/. Check for: bugs, security issues, edge cases, code quality. Write your findings to /tmp/codex-review.md"' ""
tmux send-keys -t codex-review Enter

# 3. Poll for completion
while true; do
  output=$(tmux capture-pane -t codex-review -p -S -5)
  last_line=$(echo "$output" | grep -v '^$' | tail -1)
  echo "$last_line" | grep -qE '^\$$|^\$ $' && break
  sleep 10
done

# 4. Read the review
cat /tmp/codex-review.md

# 5. Clean up
tmux kill-session -t codex-review
```

## JSONL Output (structured results)

```bash
tmux new-session -d -s codex -x 200 -y 50
sleep 1
tmux send-keys -t codex "PS1='$ ' bash --norc" Enter
sleep 1

tmux send-keys -t codex 'codex exec --full-auto --skip-git-repo-check --model gpt-5.3-codex --json "task" > /tmp/codex.jsonl 2>&1' ""
tmux send-keys -t codex Enter

# Poll
while true; do
  output=$(tmux capture-pane -t codex -p -S -5)
  last_line=$(echo "$output" | grep -v '^$' | tail -1)
  echo "$last_line" | grep -qE '^\$$|^\$ $' && break
  sleep 10
done

# Parse result
python3 -c "
import json
for line in open('/tmp/codex.jsonl'):
    obj = json.loads(line)
    if obj.get('type') == 'agent_message':
        print(obj.get('content', ''))
" | tail -1

tmux kill-session -t codex
```

## Session Continuity

```bash
# First task
tmux send-keys -t codex 'codex exec --full-auto --model gpt-5.3-codex "Build initial structure"' ""
tmux send-keys -t codex Enter
# ... wait for $ ...

# Follow-up in same session context
tmux send-keys -t codex 'codex exec resume --last --full-auto --model gpt-5.3-codex "Now add tests"' ""
tmux send-keys -t codex Enter
```

## Model and Reasoning Effort

```bash
# Run with GPT-5.3-Codex (Extra High reasoning)
codex exec --full-auto --model gpt-5.3-codex "task"

# Or set defaults in ~/.codex/config.toml to avoid repeating flags:
# model = "gpt-5.3-codex"
# Check `codex --help` for reasoning_effort flag name
```

## Long Prompts

```bash
# Use a file to avoid shell escaping issues
tmux send-keys -t codex "codex exec --full-auto --model gpt-5.3-codex \"\$(cat /tmp/task.txt)\"" ""
tmux send-keys -t codex Enter
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `send-keys 'text' Enter` in one call | Enter не регистрируется — два отдельных вызова |
| Starting `codex` (TUI) in tmux | Alternate screen = capture-pane пустой — используй `codex exec` |
| Custom PS1 in pane | Нестандартный prompt ломает grep — используй `PS1='$ ' bash --norc` |
| Running outside git without flag | Добавь `--skip-git-repo-check` если нет `.git` в рабочей директории |
| Fixed `sleep 120` | Polling по shell prompt — быстрее и надёжнее |
| Chaining: `send-keys && send-keys` | Используй отдельные bash-вызовы, не `&&` |

## Use Cases

- **Parallel execution** — Codex works in the background while Claude continues writing code; no waiting
- **Code review** — Claude writes code, Codex reviews it independently and writes findings to a file
- **Large codebase analysis and refactoring** — Codex reads entire codebase with its large context window, Claude applies targeted changes based on findings

## When NOT to Use

- CI/CD pipeline → use `codex exec` directly without tmux
- Simple one-off questions → `codex exec "question"` without tmux overhead
