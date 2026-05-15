# claude-resume

> The only Claude Code session resumer with background session support, terminal multiplexer integration, and dual-mode (TUI + CLI) operation.

[中文文档](README.md)

Resume any Claude Code session in seconds. The built-in `/resume` only shows the last 10 interactive sessions — it can't see background tasks or orphaned sessions. **Pure Python 3, zero dependencies, single ~980-line file.**

---

## Why claude-resume?

| Feature | claude-resume | cc-session | agr-cli | claude-recent | Built-in /resume |
|---------|:---:|:---:|:---:|:---:|:---:|
| Browse all sessions | ✅ | ✅ | ✅ | ✅ | ❌ 10 only |
| **Background (bg) sessions** | ✅ **unique** | ❌ | ❌ | ❌ | ❌ |
| **Orphaned sessions** | ✅ **unique** | ❌ | ❌ | ❌ | ❌ |
| **Resume to background** (`--bg`) | ✅ **unique** | ❌ | ❌ | ❌ | ❌ |
| Terminal multiplexer integration | ✅ **unique** | ❌ | ❌ | ❌ | ❌ |
| Non-interactive CLI | ✅ | ✅ | ❌ TUI only | ❌ fzf only | ✅ |
| Time filtering | ✅ | ✅ | ❌ | ❌ | ❌ |
| Full-text search | ✅ | ✅ | ✅ | ❌ | ❌ |
| JSON output | ✅ | ❌ | ❌ | ❌ | ❌ |
| Delete sessions | ✅ | ❌ | ❌ | ❌ | ❌ |
| Zero dependencies | ✅ Python | ❌ Rust | ❌ Node | ✅ Bash | ✅ |
| Cross-machine | ❌ | ❌ | ❌ | ✅ | ❌ |

---

## Installation

```bash
curl -O https://raw.githubusercontent.com/matrix1001/claude-resume/main/claude-resume
chmod +x claude-resume
mv claude-resume ~/.local/bin/
```

Or:

```bash
git clone https://github.com/matrix1001/claude-resume.git
cd claude-resume
ln -s "$(pwd)/claude-resume" ~/.local/bin/claude-resume
```

Requires: Python 3.10+. No other dependencies.

---

## Quick Start

```bash
# Interactive TUI
claude-resume

# Resume most recent session
claude-resume -l

# Resume most recent → background (Agent View)
claude-resume -l -B

# Preview today's background tasks
claude-resume -t -b --dry-run
```

---

## Flag Reference

### Selection

| Flag | Description |
|------|-------------|
| `-l, --last` | Most recent session |
| `-i N, --index N` | Nth session (1-indexed) |
| `-s KW, --search KW` | Keyword search (1 hit = auto-resume, multiple = interactive) |
| `--search-full` | Search full JSONL content, not just preview |
| `--max-results N` | Max files for full-text search (default 50) |

### Filtering

| Flag | Description |
|------|-------------|
| `-b, --bg` | **Background + orphan sessions only** |
| `-t, --today` | Today |
| `-y, --yesterday` | Yesterday |
| `-w, --this-week` | This week (Monday to now) |
| `--since DATE` | Start `YYYY-MM-DD` (local time) |
| `--until DATE` | End `YYYY-MM-DD` |
| `-A, --all-projects` | Pick project first |
| `-p PATH, --project PATH` | Target specific project |
| `-n N, --limit N` | Max entries |

### Output

| Flag | Description |
|------|-------------|
| `-j, --json` | JSON to stdout |
| `--oneline` | Compact one-line per session |
| `--no-color` | Strip ANSI codes |

### Terminal Multiplexer

| Flag | Description |
|------|-------------|
| `--zellij [tab\|pane-right\|pane-down\|float]` | New Zellij tab / split / float |
| `--tmux [window\|pane-right\|pane-down]` | New tmux window / split |
| `--terminal-cmd TMPL` | Custom template `{cwd}` `{cmd}` |

**Auto-detection:** In interactive mode, detects `$ZELLIJ` / `$TMUX` and offers an "Open in" prompt.

### Claude Arguments

| Flag | Description |
|------|-------------|
| `-B, --background` | **Launch as background task** (`claude --bg`) |
| `-C ARGS, --claude-args ARGS` | Extra args passed to `claude` |

### Session Management

| Flag | Description |
|------|-------------|
| `--info` | Show session details |
| `--rm` | Delete session (with confirmation) |
| `-f, --force` | Skip confirmation |
| `-c, --count` | Print count only (fastest path) |
| `--dry-run` | List without resuming |

---

## Use Cases

### Background Session Workflow (Unique)

```bash
# Morning: check what background tasks ran yesterday
claude-resume -y -b --dry-run

# Resume most recent background task
claude-resume -b -l

# Resume to background (free your terminal)
claude-resume -b -l -B

# Count background tasks this week
claude-resume -w -b -c
```

### Terminal Multiplexer (Unique)

```bash
# Zellij new tab
claude-resume -l --zellij

# Zellij split right
claude-resume -l --zellij pane-right

# Zellij floating pane + background
claude-resume -b -l --zellij float -B

# tmux new window
claude-resume -l --tmux

# Custom: WezTerm
claude-resume -l --terminal-cmd "wezterm cli spawn --cwd {cwd} -- {cmd}"

# Custom: kitty
claude-resume -l --terminal-cmd "kitty --directory {cwd} {cmd}"
```

### Search

```bash
# Preview search (fast)
claude-resume -s refactor

# Full-text search (deep)
claude-resume -s migration --search-full --max-results 20

# Auto-pick first hit
claude-resume -s "fix login" -i 1
```

### Scripting

```bash
# JSON → jq
claude-resume -l --json | jq '.[0].session_id'

# Count across projects
for proj in ~/project/*; do
  echo "$proj: $(claude-resume -p "$proj" -c)"
done

# Oneline → fzf
claude-resume --oneline | fzf
```

### Session Management

```bash
claude-resume -l --info       # details
claude-resume -i 3 --rm       # delete 3rd
claude-resume -i 3 --rm -f    # force delete
claude-resume -t --dry-run    # preview
```

### Advanced Combos

```bash
# Search + split + background, one command
claude-resume -s "monitoring" -i 1 --zellij pane-right -B

# Date range + search → JSON
claude-resume --since 2026-05-01 -s "refactor" -j
```

---

## Architecture

```
claude-resume
  │
  ├─ Scan ~/.claude/projects/
  ├─ Filter (bg / time / search)
  │
  ├─ --last / --index / single result?
  │     │
  │     yes → Non-interactive: resume (or --info/--rm/--dry-run)
  │     │
  │     no → Interactive: TUI → terminal choice → Claude args → resume
```

- **Data source:** `~/.claude/projects/<normalized-path>/<sessionId>.jsonl`
- **Session types:** `~/.claude/sessions/<sessionId>.json` (interactive / bg / orphan)
- **Path encoding:** `/home/user/proj` → `-home-user-proj`
- **JSON schema:** session_id, size_kb, msg_count, last_active, preview, kind, status, cwd

---

## Comparisons

### vs Built-in `/resume`

Built-in shows 10 interactive sessions only. No bg/orphan. No filtering, no search, no management, no JSON output.

### vs cc-session (Rust)

cc-session is the fastest TUI (2000+ sessions <500ms) with rich conversation preview. But it doesn't filter bg/orphan sessions, can't integrate with terminal multiplexers, can't output JSON, and requires a Rust toolchain or Homebrew.

**Choose cc-session** for rich in-project session browsing with syntax highlighting. **Choose claude-resume** for background task management, cross-terminal workflows, and scripting.

### vs agr-cli (Node)

agr-cli has tagging and stats, but is TUI-only, has no bg/orphan filter, no multiplexer integration, and requires Node.js.

### vs claude-recent (Bash)

claude-recent solves cross-machine session sharing (multiple WSL distros, containers). But it's fzf-only, no CLI mode, no bg/orphan filter, no multiplexer support.

---

## Development

```bash
python3 -c "import py_compile; py_compile.compile('claude-resume', doraise=True)"
```

Pure Python 3 stdlib. No build step.
