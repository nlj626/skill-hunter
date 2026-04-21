---
name: skill-hunter
description: 全渠道搜索 Claude Code Skills，覆盖 GitHub API、skills.sh、awesome、ClawHub、SkillsMP，top 10 都显示 stars，用户选择后直接安装
allowed-tools:
  - Bash
  - WebSearch
  - Read
  - Glob
---

# Skill Hunter — 全渠道 Claude Code Skill 搜索器

工作流程：**搜索展示 → 用户选择 → 安装**。安全审查不在本 skill 职责范围内，建议使用 `skill-vetter`。

## 搜索执行

**并行执行**（无 Agent，减少 token）：

### 1. 解析关键词

从用户描述中提取中英文搜索关键词。短关键词（<4 字符）自动扩展同义词：
- 规则：单个英文 < 4 字符 → 补 2-3 同义词用 `+` 连接；中文 → 同时准备英文翻译
- 示例：`prd` → `PRD+product+requirement+specification`，`db` → `database+sql+sqlite+mysql`

### 2. 并行搜索（4 路同时发起）

| 渠道 | 命令 | 降级 |
|------|------|------|
| GitHub API（核心） | `gh api "search/code?q=[query]+filename:SKILL.md+allowed-tools&per_page=10" --jq '.items[] | "\(.repository.full_name)\|\(.path)\|\(.repository.stargazers_count)"'` | WebSearch `github "SKILL.md" claude skill [query]` |
| skills.sh | `npx skills find "[query]" 2>&1 \| head -30` | 超时 → WebSearch `site:skills.sh [query]` |
| WebSearch×3 | `awesome claude skills [query] 2026`、`openclaw skill [query] github`、`site:skillsmp.com [query] claude skill` | — |
| 本地已安装 | `ls ~/.claude/skills/` + `cat ~/.claude/skills/.skill-lock.json` | — |

> GitHub API 的 `search/code` 返回 `repository.stargazers_count`，一次调用即可获取 stars，无需额外查询。

### 3. 补充 stars（仅非 GitHub 渠道结果）

对缺少 stars 的仓库，优先 `gh api "repos/$r" --jq '.stargazers_count'`，`gh` 不可用时 WebSearch `"[repo] github stars"` 提取。

### 4. 筛选排序

1. 去掉已安装 → 仅在"已安装"区域展示
2. 过滤不相关 → 文件名碰巧含关键词但功能不匹配的
3. 按 stars 排序 → 取 top 10

**可信度分级**（同时用于筛选和输出展示）：

| 等级 | 标识 | 条件 |
|------|------|------|
| 官方 | 🟢🟢 | anthropics、vercel-labs、microsoft |
| 高信誉 | 🟢 | stars > 1000 或安装量 > 10K |
| 良好 | 🟡 | stars 100-1000，安装量 > 1K，或 awesome 收录 |
| 一般 | 🔵 | stars < 100 或无数据 |

> stars 为仓库级别，非 skill 自身。同一仓库的多个 skill 共享 stars 数值。

筛选后 0 个 → 放宽纳入 🔵（最多 3 个，标注"质量较低"）。超出上限 → 展示 top N + "还有 M 个，需要请告知"。

### 5. 格式化输出

**对齐规则**：表格使用 Unicode box-drawing（`┌─┬─┐│├─┼─┤└─┴─┘`），按显示宽度对齐（CJK/Emoji 占 2 列，ASCII 占 1 列）。先计算每列最大宽度，再统一左对齐。

```
═══════════════════════════════════════════════════
  🔍 Skill Hunter：「[关键词]」（全量模式）
  耗时：~Xs
═══════════════════════════════════════════════════

## 已安装

┌──────────────┬────────────────────────┐
│ Skill        │ 来源                   │
├──────────────┼────────────────────────┤
│ write-a-prd  │ mattpocock/skills      │
└──────────────┴────────────────────────┘

## 未安装

┌────┬──────────────┬──────────────────────┬────────┬────────┬──────────────────────────────┐
│ #  │ Skill        │ 来源                 │ Stars  │ 可信度 │ 描述                         │
├────┼──────────────┼──────────────────────┼────────┼────────┼──────────────────────────────┤
│  1 │ prd          │ Mehdibargach/...      │  87 ⭐  │  🟡    │ 精简结构化 PRD 生成            │
│  2 │ easy-prd     │ instantX-research/... │  11 ⭐  │  🔵    │ 简易 PRD                     │
└────┴──────────────┴──────────────────────┴────────┴────────┴──────────────────────────────┘

输入编号选择要安装的 skill（多选用逗号，如 1,3），或输入 q 退出。

═══════════════════════════════════════════════════
```

> **已安装的 skill 必须排在最前面**。

---

## 安装

**触发**：用户输入编号（如 `1` 或 `1,3`）。

### 1. 确认列表

```
即将安装以下 skill：
1. prd (来自 Mehdibargach/...) — npx skills add ...
2. docker-init (来自 mgiovani/cc-arsenal) — git clone + 手动复制

确认安装？（y/n）
```

### 2. 执行安装

**方式 A — skills.sh 包（优先）**：`npx skills add [package] -g -a claude-code`

**方式 B — GitHub 子路径 skill**：
```bash
tmpdir=$(mktemp -d 2>/dev/null || mkdir -p "$TMPDIR/skill-install-[name]" && echo "$TMPDIR/skill-install-[name]")
git clone --filter=blob:none --sparse --depth=1 https://github.com/[owner]/[repo].git "$tmpdir"
cd "$tmpdir" && git sparse-checkout set [skill-path]
mkdir -p ~/.claude/skills/[skill-name] && cp -r [skill-path]/. ~/.claude/skills/[skill-name]/
rm -rf "$tmpdir"
```

**方式 C — GitHub 根目录 skill**：
```bash
tmpdir=$(mktemp -d 2>/dev/null || mkdir -p "$TMPDIR/skill-install-[name]" && echo "$TMPDIR/skill-install-[name]")
git clone --depth=1 https://github.com/[owner]/[repo].git "$tmpdir"
mkdir -p ~/.claude/skills/[skill-name] && cp "$tmpdir/SKILL.md" ~/.claude/skills/[skill-name]/
rm -rf "$tmpdir"
```

安装后验证 `ls ~/.claude/skills/[skill-name]/SKILL.md`，成功后更新 `~/.claude/skills/.skill-lock.json`。

### 3. 安装后提示

```
✅ 安装完成！如需安全审查，可使用 /skill-vetter 进行扫描。
```

---

## 异常处理

| 场景 | 处理 |
|------|------|
| `gh api` 限流/未登录 | 回退 WebSearch |
| `npx skills find` 超时 | 跳过，依赖其他渠道 |
| git clone 超时 | `--depth=1` + `--filter=blob:none` |
| 安装失败 | 提示手动安装，给出 GitHub 链接 |
| 无结果 | 建议：换关键词 或 `npx skills init` 自建 |
