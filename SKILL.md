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

从用户描述中提取中英文搜索关键词。**按渠道分别处理**：

| 渠道 | 关键词格式 | 示例 |
|------|-----------|------|
| `npx skills find` | 原始关键词，空格分隔 | `prd` 或 `pr review` |
| GitHub API | 同义词用 `+` 连接（AND 语义） | `PRD+product+requirement` |
| WebSearch | 自然语言短语 | `claude skill for PRD writing` |

短关键词（≤3 字母）触发同义词扩展（仅 GitHub API 和 WebSearch）：`prd` → `PRD+product+requirement`，`db` → `database+sql+sqlite`。`npx skills find` 始终用原始关键词。

### 2. 并行搜索（4 路同时发起）

| 渠道 | 命令 | 降级 |
|------|------|------|
| skills.sh（核心） | `npx skills find "[原始关键词]" 2>&1 \| head -30` | 超时 → WebSearch `site:skills.sh [query]` |
| GitHub API | `gh api "search/code?q=[同义词query]+filename:SKILL.md+allowed-tools&per_page=10" --jq '.items[] | "\(.repository.full_name)\|\(.path)\|\(.repository.stargazers_count)"'` | WebSearch `github "SKILL.md" claude skill [query]` |
| WebSearch×3 | `awesome claude skills [query] 2026`、`openclaw skill [query] github`、`site:skillsmp.com [query] claude skill` | — |
| 本地已安装 | `ls ~/.claude/skills/` + `cat ~/.claude/skills/.skill-lock.json` | — |

> skills.sh 是最稳定的数据源，直接返回 skill 名称、来源、安装量。GitHub API 的 `search/code` 搜文件内容但**不返回 stars**，需额外查询（见第 3 步）。

### 3. 补充 Stars（单次 Bash 调用）

> `search/code` 的 repository 对象是精简版，**不包含** `stargazers_count`（见 GitHub 文档）。需单独查询。

合并所有渠道结果后，提取去重的唯一仓库，**单次 Bash 调用**批量获取 stars：
```bash
for r in owner1/repo1 owner2/repo2 owner3/repo3; do echo "$r $(gh api "repos/$r" --jq '.stargazers_count' 2>/dev/null || echo -)"; done
```
> 只查 top 10 结果涉及的唯一仓库（通常 3-5 个）。同一仓库的多个 skill 共享 stars。`gh` 不可用时跳过，stars 显示 `-`。

**安装量获取**：`npx skills find` 输出中直接包含（如 `5.9K installs`），无需额外查询。

**去重**：合并所有渠道结果，按 `owner/repo@skill名` 去重，保留数据最完整的条目。

### 4. 筛选排序

1. 去掉已安装 → 仅在"已安装"区域展示
2. 过滤不相关 → 文件名碰巧含关键词但功能不匹配的
3. 综合排序 → 取 top 10：
   - 主排序：安装数量降序
   - 次排序：stars 降序
   - 末排序：skill 名称字母序

**可信度分级**（按优先级判定：官方 > 高信誉 > 良好 > 一般）：

| 等级 | 标识 | 条件 |
|------|------|------|
| 官方 | 🟢🟢 | 仓库 owner 为 anthropics、vercel-labs、microsoft、openai、figma |
| 高信誉 | 🟢 | stars ≥ 1000 或安装量 ≥ 10K |
| 良好 | 🟡 | 100 ≤ stars < 1000，或 1K ≤ 安装量 < 10K，或 awesome 收录 |
| 一般 | 🔵 | stars < 100 且安装量 < 1K |

> stars 为仓库级别，非 skill 自身。同一仓库的多个 skill 共享 stars 数值。

筛选后 0 个 → 放宽纳入 🔵（最多 3 个，标注"质量较低"）。超出上限 → 展示 top N + "还有 M 个，需要请告知"。

### 5. 格式化输出

**使用列表格式**，每条 skill 一行，字段用固定分隔符对齐：

```
🔍 Skill Hunter：「[关键词]」（全渠道搜索）

## 已安装
- write-a-prd | mattpocock/skills

## 未安装
 1. prd | Mehdibargach/skill-repo | 安装 5.9K | ⭐ 87 | 🟢 | 精简结构化 PRD 生成
 2. easy-prd | instantX-research/... | 安装 200 | ⭐ 11 | 🔵 | 简易 PRD

输入编号选择要安装的 skill（多选用逗号，如 1,3），或输入 q 退出。
```

**输出规则**：
- 已安装和未安装分开展示，已安装在前
- 未安装每条格式：`编号. skill名 | 来源 | 安装 数量 | ⭐ 数量 | 可信度emoji | 描述`
- 安装量格式：`5.9K`、`1.2K`、`200`；无数据时省略该字段
- Stars 格式：`⭐ 87`；无数据时省略该字段
- 可信度 emoji：🟢🟢、🟢、🟡、🔵
- 描述截断到 30 字符，超出用 `…` 结尾
- 来源过长时截断为 `owner/rep...`

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
