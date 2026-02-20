# Codex CLI Orchestration via tmux

Скилл для Claude Code — делегирует задачи агенту OpenAI Codex CLI (GPT-5.3-Codex) через tmux exec режим для параллельного выполнения.

## Проблема

Codex CLI в интерактивном TUI режиме использует alternate screen buffer, что ломает `tmux capture-pane`. В отличие от Gemini CLI, нет надёжного idle-маркера в выводе TUI.

## Решение

Используй **`codex exec` режим** через tmux: каждая задача запускается как отдельный процесс. Завершение определяется по возврату shell prompt `$`. Опциональный JSONL вывод даёт структурированные результаты.

## Структура

```
codex-tmux-orchestration/
└── SKILL.md    # Основной скилл с командами и workflow
```

## Установка

```bash
cp -r skills/codex-tmux-orchestration ~/.claude/skills/
```

## Требования

- **tmux** — `brew install tmux` (macOS) или `apt install tmux` (Linux)
- **Codex CLI** — `npm install -g @openai/codex`
- **Аккаунт OpenAI** с Plus/Pro/API доступом
- Claude Code запущен внутри tmux сессии

## Быстрый старт

```bash
# 1. Открой split pane с контролируемым shell
tmux split-window -h -d "PS1='$ ' bash --norc"

# 2. Отправь задачу (ДВА отдельных вызова!)
tmux send-keys -t {right} 'codex exec --full-auto --model gpt-5.3-codex "Build the app per PLAN.md"'
tmux send-keys -t {right} Enter

# 3. Жди shell prompt (= задача завершена)
while true; do
  output=$(tmux capture-pane -t {right} -p -S -5)
  echo "$output" | grep -qE '^\$ ' && break
  sleep 5
done
```

## Ключевые отличия от gemini-tmux-orchestration

| Аспект | Gemini | Codex |
|--------|--------|-------|
| Режим | Интерактивный TUI (персистентный) | exec режим (процесс на задачу) |
| Idle маркер | `"Type your message"` | Shell prompt `$` (процесс завершается) |
| Детекция зависания | `potential loop` → send `2` | Не нужна (процесс завершается) |
| Возобновление сессии | Нет | `codex exec resume --last` |
| Формат вывода | Capture терминала | Опционально JSONL (структурированный) |
| Авто-одобрение | `--yolo` | `--full-auto` или `--yolo` |

## Области применения

- **Параллельное выполнение** — Codex работает в фоне, пока Claude продолжает писать код; никакого ожидания
- **Code Review** — Claude пишет код, Codex независимо проводит ревью и записывает замечания в файл
- **Анализ и рефакторинг больших кодовых баз** — Codex читает всю кодовую базу благодаря большому контекстному окну, Claude применяет точечные изменения на основе выводов

## Смотри также

- [openai/codex на GitHub](https://github.com/openai/codex)
- [tmux Manual](https://man7.org/linux/man-pages/man1/tmux.1.html)
