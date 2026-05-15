# claude-resume

> A 10× more powerful Claude Code session resumer than the built-in `/resume`.

[中文文档](README.md)

`claude-resume` is a **dual-mode** (interactive TUI + non-interactive CLI) tool that lets you resume any Claude Code conversation in seconds — including `--bg` background tasks and orphaned sessions (which the built-in `/resume` can't see).

Pure Python 3, zero dependencies, single ~980-line script.

---

## Installation

```bash
cp claude-resume ~/.local/bin/
chmod +x ~/.local/bin/claude-resume
```

---

## Quick Start

```bash
# Interactive: browse, pick, resume
claude-resume

# Non-interactive: resume most recent
claude-resume -l

# Resume most recent in background
claude-resume -l -B

# Preview today's sessions
claude-resume -t --dry-run

# Today's most recent background session
claude-resume -t -b -l
```

---

## Full Flag Reference

### Selection (non-interactive)
| Flag | Description |
|------|-------------|
| `-l, --last` | Most recent session |
| `-i N, --index N` | Nth session (1-indexed) |
| `-s KW, --search KW` | Keyword search; 1 hit = auto-resume, multiple = interactive |
| `--search-full` | Search full JSONL content, not just preview |
| `--max-results N` | Max files to scan in full-text search (default 50) |

### Filtering
| Flag | Description |
|------|-------------|
| `-b, --bg` | Background/orphan sessions only |
| `-t, --today` | Today only |
| `-y, --yesterday` | Yesterday only |
| `-w, --this-week` | This week (Monday to now) |
| `--since DATE` | Start date `YYYY-MM-DD` |
| `--until DATE` | End date `YYYY-MM-DD` |
| `-A, --all-projects` | Pick project first, then session |
| `-p PATH, --project PATH` | Target specific project |
| `-n N, --limit N` | Max entries to display |

### Output (non-interactive)
| Flag | Description |
|------|-------------|
| `-j, --json` | JSON to stdout |
| `--oneline` | Compact one-line per session (fzf-friendly) |
| `--no-color` | Strip ANSI escape codes |

### Terminal Multiplexer
| Flag | Description |
|------|-------------|
| `--zellij [tab\|pane-right\|pane-down\|float]` | Open in Zellij (default: tab) |
| `--tmux [window\|pane-right\|pane-down]` | Open in tmux (default: window) |
| `--terminal-cmd TMPL` | Custom template; `{cwd}` and `{cmd}` placeholders |

**Auto-detection:** In interactive mode, if `$ZELLIJ` or `$TMUX` is detected (env var + binary found), an "Open in" prompt appears after session selection. In non-interactive mode, use `--zellij`/`--tmux` explicitly.

### Claude Arguments
| Flag | Description |
|------|-------------|
| `-B, --background` | Launch claude with `--bg` (Agent View background) |
| `-C ARGS, --claude-args ARGS` | Extra arguments passed through to `claude` |

### Session Management
| Flag | Description |
|------|-------------|
| `--info` | Show session details |
| `--rm` | Delete session (confirmation required in TTY; `-f` to skip) |
| `-f, --force` | Skip `--rm` confirmation |
| `-c, --count` | Print count only (no JSONL reads — fastest) |
| `--dry-run` | List matching sessions, never resume |

### Other
| Flag | Description |
|------|-------------|
| `--no-cd` | Don't chdir to original working directory |
| `-h, --help` | Show help |

---

## Usage Examples

### Daily Workflow

```bash
# First thing in the morning: resume yesterday's last session
claude-resume -y -l

# What did I do today?
claude-resume -t --dry-run

# Resume today's most recent background task
claude-resume -t -b -l
```

### Search

```bash
# Search preview text (fast)
claude-resume -s refactor

# Full-text search (deep but slower)
claude-resume -s migration --search-full

# Full-text with limited scan range
claude-resume -s migration --search-full --max-results 20

# Search + auto-pick 1st hit (non-interactive)
claude-resume -s "fix login" -i 1

# Search + preview only
claude-resume -s "bug" --dry-run
```

### Time Filtering

```bash
# All background sessions this week
claude-resume -w -b --dry-run

# Date range
claude-resume --since 2026-05-01 --until 2026-05-10 --dry-run

# Date range + keyword search → JSON
claude-resume --since 2026-05-01 -s "refactor" -j
```

### Scripting & Pipelines

```bash
# JSON → jq
claude-resume -l --json | jq '.[0].session_id'

# Count today's sessions
claude-resume -t -c

# Count across all projects
for proj in ~/project/*; do
  echo "$proj: $(claude-resume -p "$proj" -c)"
done

# Oneline → fzf
claude-resume --oneline | fzf
```

### Terminal Multiplexer

```bash
# Zellij new tab
claude-resume -l --zellij

# Zellij floating pane
claude-resume -l --zellij float

# Zellij split right
claude-resume -l --zellij pane-right

# tmux new window
claude-resume -l --tmux

# tmux split right
claude-resume -l --tmux pane-right

# Interactive (auto-detects available multiplexer)
claude-resume

# Custom: WezTerm
claude-resume -l --terminal-cmd "wezterm cli spawn --cwd {cwd} -- {cmd}"

# Custom: kitty
claude-resume -l --terminal-cmd "kitty --directory {cwd} {cmd}"
```

### Claude Arguments Passthrough

```bash
# Resume in background (Agent View)
claude-resume -l -B

# With specific model
claude-resume -l -C "--model haiku"

# Multiple flags
claude-resume -l -C "--bg --fast"

# Custom message
claude-resume -l -C '--message "continue yesterday refactoring"'
```

### Session Management

```bash
# Show details of most recent session
claude-resume -l --info

# Delete 3rd session (with confirmation)
claude-resume -i 3 --rm

# Force delete (no confirmation)
claude-resume -i 3 --rm -f

# Preview without resuming
claude-resume --dry-run
claude-resume -t -b --dry-run
```

### Cross-Project

```bash
# Pick from all projects
claude-resume -A

# Target a specific project
claude-resume -p ~/project/other-app -l

# Target + search
claude-resume -p ~/project/other-app -s "deploy"
```

### Advanced Combinations

```bash
# Zellij + background + search → one shot
claude-resume -s "monitoring" -i 1 --zellij tab -B

# Preview this week, decide later
claude-resume -w --dry-run

# Count bg sessions this week
count=$(claude-resume -w -b -c)
if [ "$count" -gt 3 ]; then
  echo "Heavy background load this week: $count sessions"
fi
```

---

## Architecture

```
claude-resume
  │
  ├── Scan + filter
  │
  ├── --last / --index / single result?
  │     │
  │     yes → Non-interactive: resume immediately (or --info/--rm/--dry-run)
  │     │
  │     no → Interactive: TUI pick → terminal choice → Claude args → resume
```

**Data model:**
- **Session files:** `~/.claude/projects/<normalized-path>/<sessionId>.jsonl`
- **Session registry:** `~/.claude/sessions/<sessionId>.json`
- **Path normalization:** `/home/user/proj` → `-home-user-proj`

**Output formats:**

JSON:
```json
[{"session_id":"abc12345-...","short_id":"abc12345","size_kb":234,"msg_count":150,
  "last_active":"2026-05-15 14:30:00","preview":"fix the login bug",
  "kind":"bg","status":"idle","name":"","cwd":"/home/matrix/project/foo"}]
```

Oneline:
```
abc12345  2026-05-15 14:30:00  bg  idle  234KB  150msgs  fix the login bug
```

---

## Development

```bash
./claude-resume --help
python3 -c "import py_compile; py_compile.compile('claude-resume', doraise=True)"
```

Pure Python 3 stdlib. No build step. No dependencies.
