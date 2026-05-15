# claude-resume

> 唯一支持 Background Session、终端复用器集成、双模式（TUI + CLI）的 Claude Code 会话恢复工具。

[English](README.en.md) | [设计文档](docs/superpowers/specs/2026-05-15-claude-resume-v2-design.md)

`claude-resume` 让你秒级恢复任何 Claude Code 会话——内置 `/resume` 只显示最近 10 个交互会话，看不到后台任务和孤儿会话。**纯 Python 3，零依赖，单文件 980 行。**

---

## 为什么选 claude-resume？

| 能力 | claude-resume | cc-session | agr-cli | claude-recent | 内置 /resume |
|------|:---:|:---:|:---:|:---:|:---:|
| 浏览全部会话 | ✅ | ✅ | ✅ | ✅ | ❌ 仅 10 个 |
| **后台 (bg) 会话** | ✅ **唯一** | ❌ | ❌ | ❌ | ❌ |
| **孤儿会话** | ✅ **唯一** | ❌ | ❌ | ❌ | ❌ |
| **恢复到后台** (`--bg`) | ✅ **唯一** | ❌ | ❌ | ❌ | ❌ |
| 终端复用器集成 | ✅ **唯一** | ❌ | ❌ | ❌ | ❌ |
| 非交互 CLI | ✅ | ✅ | ❌ 仅 TUI | ❌ 仅 fzf | ✅ |
| 时间过滤 | ✅ | ✅ | ❌ | ❌ | ❌ |
| 全文搜索 | ✅ | ✅ | ✅ | ❌ | ❌ |
| JSON 输出 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 删除会话 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 零依赖 | ✅ Python | ❌ Rust | ❌ Node | ✅ Bash | ✅ |
| 跨机器 | ❌ | ❌ | ❌ | ✅ | ❌ |

---

## 安装

```bash
curl -O https://raw.githubusercontent.com/<user>/claude-resume/main/claude-resume
chmod +x claude-resume
mv claude-resume ~/.local/bin/
```

或者：

```bash
git clone https://github.com/<user>/claude-resume.git
cd claude-resume
ln -s "$(pwd)/claude-resume" ~/.local/bin/claude-resume
```

要求：Python 3.10+，无其他依赖。

---

## 30 秒上手

```bash
# 交互模式——浏览、搜索、恢复
claude-resume

# 一键恢复最近会话
claude-resume -l

# 恢复最近会话并放入后台（Agent View）
claude-resume -l -B

# 今天有哪些后台任务？
claude-resume -t -b --dry-run
```

---

## 完整命令参考

### 选择方式

| 标志 | 说明 |
|------|------|
| `-l, --last` | 最近一个会话（= `--index 1` 快捷方式） |
| `-i N, --index N` | 按列表第 N 个（1-indexed） |
| `-s KW, --search KW` | 关键词搜索；命中 1 条直接恢复，多条进交互筛选 |
| `--search-full` | 搜完整 JSONL 内容（默认只搜 Preview） |
| `--max-results N` | 全文搜索最多扫 N 个文件（默认 50） |

### 过滤

| 标志 | 说明 |
|------|------|
| `-b, --bg` | **只看后台 + 孤儿会话**（内置 `/resume` 看不到的） |
| `-t, --today` | 今天 |
| `-y, --yesterday` | 昨天 |
| `-w, --this-week` | 本周（周一起） |
| `--since DATE` | 起始 `YYYY-MM-DD`（本地时间） |
| `--until DATE` | 截止 `YYYY-MM-DD` |
| `-A, --all-projects` | 跨项目浏览 |
| `-p PATH, --project PATH` | 指定项目 |
| `-n N, --limit N` | 最多 N 条 |

### 输出格式

| 标志 | 说明 |
|------|------|
| `-j, --json` | JSON 输出，给脚本消费 |
| `--oneline` | 紧凑单行，`fzf`/`grep` 友好 |
| `--no-color` | 纯文本，无 ANSI |

### 终端复用器

| 标志 | 说明 |
|------|------|
| `--zellij [tab\|pane-right\|pane-down\|float]` | Zellij 新 tab / 分屏 / 浮动窗 |
| `--tmux [window\|pane-right\|pane-down]` | tmux 新 window / 分屏 |
| `--terminal-cmd TMPL` | 自定义终端命令 `{cwd}` `{cmd}` 占位 |

交互模式下**自动检测** Zellij/tmux，选完会话弹出 "在哪儿打开"。

### Claude 参数

| 标志 | 说明 |
|------|------|
| `-B, --background` | **恢复到后台** `claude --bg`（Agent View） |
| `-C ARGS, --claude-args ARGS` | 透传给 `claude` |

### 会话管理

| 标志 | 说明 |
|------|------|
| `--info` | 查看会话详情 |
| `--rm` | 删除会话（需确认） |
| `-f, --force` | 跳过确认 |
| `-c, --count` | 仅输出数量（最快路径） |
| `--dry-run` | 预览不恢复 |

---

## 实战场景

### 后台任务工作流（独占能力）

```bash
# 早上来：看看昨天有哪些后台任务跑完了
claude-resume -y -b --dry-run

# 恢复最近一个后台任务继续
claude-resume -b -l

# 恢复到后台（不占终端）
claude-resume -b -l -B

# 本周后台任务数量
claude-resume -w -b -c
```

### 终端复用器集成（独占能力）

```bash
# Zellij 新 tab
claude-resume -l --zellij

# Zellij 右侧分屏
claude-resume -l --zellij pane-right

# Zellij 浮动窗口恢复后台任务
claude-resume -b -l --zellij float -B

# tmux 新 window
claude-resume -l --tmux

# 自定义：WezTerm
claude-resume -l --terminal-cmd "wezterm cli spawn --cwd {cwd} -- {cmd}"

# 自定义：kitty
claude-resume -l --terminal-cmd "kitty --directory {cwd} {cmd}"
```

### 搜索

```bash
# 搜 Preview（快）
claude-resume -s refactor

# 全文搜索（深但慢）
claude-resume -s migration --search-full

# 全文搜索限制范围
claude-resume -s migration --search-full --max-results 20

# 取第一个搜索结果
claude-resume -s "fix login" -i 1
```

### 脚本与自动化

```bash
# JSON → jq
claude-resume -l --json | jq '.[0].session_id'

# 统计所有项目会话数
for proj in ~/project/*; do
  echo "$proj: $(claude-resume -p "$proj" -c)"
done

# oneline → fzf
claude-resume --oneline | fzf

# 定时清理：超过 3 个后台任务就报警
count=$(claude-resume -w -b -c)
if [ "$count" -gt 3 ]; then
  echo "本周后台任务较多: $count 个"
fi
```

### 会话管理

```bash
# 查看详情
claude-resume -l --info

# 删除第 3 个
claude-resume -i 3 --rm

# 强制删除
claude-resume -i 3 --rm -f

# 预览不恢复
claude-resume -t --dry-run
```

### 组合技

```bash
# 搜索 + 分屏 + 后台恢复，一行搞定
claude-resume -s "monitoring" -i 1 --zellij pane-right -B

# 时间范围 + 搜索 → JSON
claude-resume --since 2026-05-01 -s "refactor" -j

# 今天后台任务预览
claude-resume -t -b --dry-run
```

---

## 设计

```
claude-resume
  │
  ├─ 扫描 ~/.claude/projects/
  ├─ 过滤（bg / 时间 / 搜索）
  │
  ├─ --last / --index / 单条结果？
  │     │
  │     是 → 非交互：直接恢复（或 --info / --rm / --dry-run）
  │     │
  │     否 → 交互：TUI 选会话 → 终端选择 → Claude 参数 → 恢复
```

- **数据源：** `~/.claude/projects/<normalized-path>/<sessionId>.jsonl`
- **类型标记：** `~/.claude/sessions/<sessionId>.json`（interactive / bg / orphan）
- **路径编码：** `/home/user/proj` → `-home-user-proj`
- **JSON 输出：** session_id, size_kb, msg_count, last_active, preview, kind, status, cwd

---

## 与其他工具对比

### vs 内置 `/resume`

- 内置只显示 10 个交互会话，看不到 bg/orphan
- 内置没有过滤、搜索、管理功能
- claude-resume 可以 `--json` 给脚本用

### vs cc-session (Rust)

cc-session 是最快的 TUI（2000+ 会话 <500ms），带对话预览和语法高亮。但它：
- **不支持 bg/orphan 过滤**——所有会话混在一起
- **不支持终端复用器**——只能在当前终端打开
- **不支持 `--json`**——不能给脚本消费
- **需要 Rust 工具链或 Homebrew**

如果你的场景是"在一个项目里快速预览大量会话内容"，cc-session 更好。如果你的场景是"管理后台任务、跨终端恢复、脚本自动化"，claude-resume 更强。

### vs agr-cli (Node)

agr-cli 有 tag 系统和统计功能，但它：
- **不支持 bg/orphan 过滤**
- **不支持终端复用器**
- **不支持非交互模式**——只能 TUI
- **需要 Node 环境**

### vs claude-recent (Bash)

claude-recent 解决了跨机器共享会话的问题（多 WSL / 容器），但它：
- **不支持 bg/orphan 过滤**
- **不支持终端复用器**
- **只能 fzf 交互**——无 CLI 模式

---

## 开发

```bash
python3 -c "import py_compile; py_compile.compile('claude-resume', doraise=True)"
```

纯 Python 3 stdlib，无构建步骤。
