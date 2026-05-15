# claude-resume

> 比内置 `/resume` 强大十倍的 Claude Code 会话恢复工具。

[English version](README.en.md)

`claude-resume` 是一个**双模式**（交互式 TUI + 非交互式 CLI）会话恢复工具，让你秒级恢复任何 Claude Code 会话 —— 包括 `--bg` 后台任务和孤儿会话（这些是内置 `/resume` 看不到的）。

纯 Python 3，零依赖，单个脚本 980 行。

---

## 安装

```bash
# 扔到 PATH 里就行
cp claude-resume ~/.local/bin/
chmod +x ~/.local/bin/claude-resume
```

---

## 快速上手

```bash
# 交互模式：浏览、选择、恢复
claude-resume

# 非交互：一键恢复最近会话
claude-resume -l

# 恢复最近会话 + 放入后台
claude-resume -l -B

# 看今天有哪些会话
claude-resume -t --dry-run

# 恢复今天最近的一个后台会话
claude-resume -t -b -l
```

---

## 完整命令参考

### 选择（非交互）

| 标志 | 说明 |
|------|------|
| `-l, --last` | 最近的会话 |
| `-i N, --index N` | 按当前列表的第 N 个（1-indexed） |
| `-s KW, --search KW` | 关键词搜索；命中 1 条 = 直接恢复，多条 = 进入交互筛选 |
| `--search-full` | 搜索完整 JSONL 内容，而非仅 Preview |
| `--max-results N` | 全文搜索最多扫描 N 个文件（默认 50） |

### 过滤

| 标志 | 说明 |
|------|------|
| `-b, --bg` | 只看 background/orphan 会话 |
| `-t, --today` | 今天 |
| `-y, --yesterday` | 昨天 |
| `-w, --this-week` | 本周（周一起） |
| `--since DATE` | 起始日期 `YYYY-MM-DD` |
| `--until DATE` | 截止日期 `YYYY-MM-DD` |
| `-A, --all-projects` | 先选项目，再选会话 |
| `-p PATH, --project PATH` | 指定项目路径 |
| `-n N, --limit N` | 最多显示 N 条 |

### 输出格式（非交互）

| 标志 | 说明 |
|------|------|
| `-j, --json` | JSON 输出，供脚本消费 |
| `--oneline` | 紧凑单行输出，适合 `fzf` / `grep` |
| `--no-color` | 无 ANSI 颜色码 |

### 终端复用器

| 标志 | 说明 |
|------|------|
| `--zellij [tab\|pane-right\|pane-down\|float]` | 在 Zellij 中打开（默认 tab） |
| `--tmux [window\|pane-right\|pane-down]` | 在 tmux 中打开（默认 window） |
| `--terminal-cmd TMPL` | 自定义终端命令，`{cwd}` 和 `{cmd}` 占位 |

**自动检测：** 交互模式下，如果检测到 `$ZELLIJ` 或 `$TMUX` 环境变量且二进制可用，会在选择会话后自动弹出 "在哪儿打开" 的选项。非交互模式下，通过 `--zellij` / `--tmux` 显式指定。

### Claude 启动参数

| 标志 | 说明 |
|------|------|
| `-B, --background` | 启动 claude 时加 `--bg`（进入 Agent View 后台） |
| `-C ARGS, --claude-args ARGS` | 透传给 `claude` 的额外参数 |

### 会话管理

| 标志 | 说明 |
|------|------|
| `--info` | 查看会话详细信息 |
| `--rm` | 删除会话（TTY 下需确认，`-f` 跳过） |
| `-f, --force` | 跳过 `--rm` 的确认 |
| `-c, --count` | 只输出数量（不读 JSONL，最快） |
| `--dry-run` | 列出但不恢复（预览模式） |

### 其他

| 标志 | 说明 |
|------|------|
| `--no-cd` | 不自动切换工作目录 |
| `-h, --help` | 帮助 |

---

## 实战手册

### 日常使用

```bash
# 早上来第一件事：恢复昨天的最后一个会话
claude-resume -y -l

# 看看今天干了啥
claude-resume -t --dry-run

# 恢复今天最近的一个 bg 任务
claude-resume -t -b -l
```

### 搜索与定位

```bash
# 搜索包含 "refactor" 的会话（搜 Preview，快）
claude-resume -s refactor

# 全文搜索 "migration"（慢但全面）
claude-resume -s migration --search-full

# 全文搜索，但只扫最近 20 个文件（提速）
claude-resume -s migration --search-full --max-results 20

# 搜索 + 取第一个结果（非交互）
claude-resume -s "fix login" -i 1

# 搜索 + 预览（不恢复）
claude-resume -s "bug" --dry-run
```

### 时间过滤组合

```bash
# 本周内的所有 bg 会话
claude-resume -w -b --dry-run

# 指定日期范围（5 月 1 日到 5 月 10 日）
claude-resume --since 2026-05-01 --until 2026-05-10 --dry-run

# 组合：指定日期 + 关键词搜索
claude-resume --since 2026-05-01 -s "refactor" -j
```

### 脚本与管道

```bash
# JSON 输出 → jq 解析
claude-resume -l --json | jq '.[0].session_id'

# 统计今天的会话数
claude-resume -t -c

# 统计所有项目的会话数
for proj in ~/project/*; do
  echo "$proj: $(claude-resume -p "$proj" -c)"
done

# oneline 输出 → fzf 模糊搜索
claude-resume --oneline | fzf
```

### 终端复用器

```bash
# Zellij 新 tab 中恢复最近会话
claude-resume -l --zellij

# Zellij 浮动窗口
claude-resume -l --zellij float

# Zellij 右侧分屏
claude-resume -l --zellij pane-right

# tmux 新 window
claude-resume -l --tmux

# tmux 右侧分屏
claude-resume -l --tmux pane-right

# 交互模式（自动检测复用器）
claude-resume
# → 选择会话后：自动弹出 "Open in" 选项

# 自定义终端命令
claude-resume -l --terminal-cmd "wezterm cli spawn --cwd {cwd} -- {cmd}"

# 自定义：kitty
claude-resume -l --terminal-cmd "kitty --directory {cwd} {cmd}"
```

### Claude 参数透传

```bash
# 恢复到后台（Agent View）
claude-resume -l -B

# 指定模型
claude-resume -l -C "--model haiku"

# 组合多个参数
claude-resume -l -C "--bg --fast"

# 传递自定义消息
claude-resume -l -C '--message "继续昨天的重构"'
```

### 会话管理

```bash
# 查看最近会话的详细信息
claude-resume -l --info

# 输出示例：
# Session:  abc12345-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# Kind:     interactive
# Status:   idle
# Name:     my-session
# CWD:      ~/project/foo
# Size:     234KB
# Messages: 150
# Last:     2026-05-15 14:30
# Preview:  fix the login bug

# 删除第 3 个会话（需确认）
claude-resume -i 3 --rm

# 强制删除（无确认）
claude-resume -i 3 --rm -f

# 删除最近会话（需确认，非交互模式）
claude-resume -l --rm

# 预览不恢复
claude-resume --dry-run
claude-resume -t -b --dry-run     # 今天 bg 会话预览
```

### 跨项目管理

```bash
# 在所有项目中挑选
claude-resume -A

# 指定项目
claude-resume -p ~/project/other-app -l

# 指定项目 + 搜索
claude-resume -p ~/project/other-app -s "deploy"
```

### 交互模式下的操作

```bash
# 纯交互（默认）
claude-resume
# → 显示所有会话
# → 输入编号选择
# → 自动检测 Zellij/tmux → 选择打开位置
# → 可选输入 Claude 额外参数
# → 恢复

# 过滤后交互选择
claude-resume -t -b            # 今天 bg 会话 → 交互选择
claude-resume -s "refactor"    # 搜索结果 → 交互选择
claude-resume -n 10            # 最近 10 个 → 交互选择
```

### 高级组合

```bash
# Zellij tab + 后台模式 + 搜索，一气呵成
claude-resume -s "monitoring" -i 1 --zellij tab -B

# 本周会话预览 → 决定要不要恢复
claude-resume -w --dry-run

# 统计本周 bg 会话数，超过 3 个就清理
count=$(claude-resume -w -b -c)
if [ "$count" -gt 3 ]; then
  echo "本周 bg 任务较多: $count 个"
fi
```

---

## 核心概念

### 双模式设计

```
claude-resume
  │
  ├── 扫描 + 过滤
  │
  ├── 有 --last / --index / 唯一结果？
  │     │
  │     是 → 非交互：直接恢复（或 --info / --rm / --dry-run）
  │     │
  │     否 → 交互：TUI 选择 → 终端选择 → Claude 参数 → 恢复
```

### 数据模型

- **会话文件：** `~/.claude/projects/<normalized-path>/<sessionId>.jsonl`
  - 每行一条消息 JSON：`{timestamp, cwd, message: {role, content}}`
- **会话注册表：** `~/.claude/sessions/<sessionId>.json`
  - 元数据：`{sessionId, kind, status, name, cwd, pid}`
  - 后来的文件覆盖之前的；`busy` 优先于 `idle`
- **路径规范化：** `/home/user/proj` → `-home-user-proj`

### 输出格式

**JSON：**
```json
[
  {
    "session_id": "abc12345-...",
    "short_id": "abc12345",
    "size_kb": 234,
    "msg_count": 150,
    "last_active": "2026-05-15 14:30:00",
    "preview": "fix the login bug",
    "kind": "bg",
    "status": "idle",
    "name": "",
    "cwd": "/home/matrix/project/foo"
  }
]
```

**Oneline：**
```
abc12345  2026-05-15 14:30:00  bg  idle  234KB  150msgs  fix the login bug
```

---

## 开发

```bash
# 运行
./claude-resume --help

# Python 编译检查
python3 -c "import py_compile; py_compile.compile('claude-resume', doraise=True)"
```

纯 Python 3 stdlib，无构建步骤，无依赖。
