---
name: skill-hunter
description: 全渠道搜索 Claude Code Skills，覆盖 GitHub API、skills.sh、awesome、ClawHub、SkillsMP，top 10 都显示 stars，用户选择后直接安装
allowed-tools:
  - Bash
  - WebSearch
  - Read
  - Glob
---

> **首先执行**：`cat ~/.claude/skills/skill-hunter/config.json 2>/dev/null`
> 输出为空（首次使用）→ 询问 Key → 保存 → 执行搜索
> 输出非空 → 跳过本段，直接执行搜索流程
>
> **无关键词？** 如果用户只输入 `/skill-hunter` 没带关键词，先问 "你想搜索什么 skill？" 再搜索。
>
> **语言**：跟随用户使用的语言。用户用中文则用中文回复，用英语则用英语回复。默认英语。不要在一条回复中混用语言。

# Skill Hunter — 全渠道 Claude Code Skill 搜索器

## 首次配置（config.json 不存在时）

每次只问一个问题。显示问题 1 后停止等待用户回复，再显示问题 2。不要在同一条回复中显示两个问题。

1. "请输入 SkillsMP API Key（免费注册：skillsmp.com/docs/api），或输入 s 跳过：" → **停止，等待用户输入。**
2. "请输入 ClawHub Token（clawhub CLI 登录获取），或输入 s 跳过：" → **停止，等待用户输入。**

用一条 Bash 命令保存（不要多想，直接执行）：
```bash
mkdir -p ~/.claude/skills/skill-hunter && echo '{"asked":true,"skillsmp_key":"[用户输入或空]","clawhub_token":"[用户输入或空]"}' > ~/.claude/skills/skill-hunter/config.json
```

如果两个都跳过：`echo '{"asked":true}' > ~/.claude/skills/skill-hunter/config.json`

然后说："配置已保存。后续可手动编辑 `~/.claude/skills/skill-hunter/config.json`。"

**保存后，如果用户提供了关键词 → 立即搜索。如果没有关键词 → 问 "你想搜索什么 skill？"**

## 搜索执行

目标 **30 秒**。全量并行，无 Agent 子进程。

### 1. 解析关键词

| 渠道 | 关键词格式 | 示例 |
|------|-----------|------|
| `npx skills find` | 原始关键词 | `prd` 或 `pr review` |
| GitHub API | 同义词用 `+` 连接（AND） | `PRD+product+requirement` |
| WebSearch / API | 原始关键词 | `prd` |

短关键词（≤3 字母）触发同义词扩展（仅 GitHub API）：`prd` → `PRD+product+requirement`。

### 2. 并行搜索（6 路同时发起）

**所有 6 路必须在同一次响应中并行发起，不得串行。**

读取 config.json 中的 Key，决定 ClawHub 和 SkillsMP 的调用方式：

| 渠道 | 有 Key | 无 Key（回退） |
|------|--------|---------------|
| skills.sh | `npx skills find "[关键词]" 2>&1 \| head -30` | 同左 |
| GitHub API | `gh api "search/code?q=[同义词]+filename:SKILL.md+allowed-tools&per_page=10" --jq '.items[] \| "\(.repository.full_name)\|\(.path)"'` | 同左 |
| ClawHub | `curl -s --max-time 10 -H "Authorization: Bearer [token]" "https://clawhub.ai/api/v1/search?q=[关键词]&limit=10" 2>/dev/null` | WebSearch `"clawhub [关键词] claude skill"` |
| SkillsMP | `curl -s --max-time 10 -H "Authorization: Bearer [key]" "https://skillsmp.com/api/v1/skills/search?q=[关键词]&limit=10" 2>/dev/null` | WebSearch `"skillsmp [关键词] claude skill"` |
| awesome | WebSearch `"awesome claude skills [关键词] github 2026"` | 同左 |
| 本地已安装 | Glob `~/.claude/skills/*/SKILL.md` | 同左 |

> curl 超时或报错 → 静默跳过该渠道（不要向用户显示错误）。GitHub API 降级为 WebSearch `"github SKILL.md claude skill [关键词]"`。

### 3. 合并与 Stars

**去重**：所有渠道结果按 `owner/repo@skill名` 去重，优先保留信息最完整的条目。

**描述**：从各渠道结果中直接提取。无法获取时显示 `-`，不额外查询。

**Stars**：对 top 10 涉及的唯一 GitHub 仓库（通常 3-5 个），单次 Bash 批量查询：
```bash
for r in owner1/repo1 owner2/repo2; do echo "$r $(gh api "repos/$r" --jq '.stargazers_count' 2>/dev/null || echo -)"; done
```
- 404 → 直接跳过
- 非 GitHub 格式 → 标记"非 GitHub"，不查 stars
- `gh` 未安装 → 所有 stars 显示 `-`

### 4. 筛选排序

1. 已安装 → 仅在"已安装"区域展示
2. 过滤不相关（文件名碰巧匹配但功能不匹配）
3. 综合排序 → 取 top 10：
   - 主排序：安装量降序（缺失视为 0）
   - 次排序：stars 降序
   - 末排序：可信度
   - 最终：skill 名称字母序

**可信度**（必须严格按规则判断，不得猜测）：官方🟢🟢（anthropics/vercel-labs/microsoft/openai/figma）> 高信誉🟢（stars≥1K 或安装量≥10K）> 良好🟡（stars≥100 或安装量≥1K）> 一般🔵

筛选后 0 个 → 放宽纳入 🔵（最多 3 个，标注"质量较低"）。

### 5. 格式化输出

已安装和未安装使用 **相同的 markdown 表格格式**。已安装部分也必须包含所有列（来源、安装量、Stars、可信度、描述），不得简化。

```
🔍 Skill Hunter：「[关键词]」（全渠道搜索）

## 已安装

| Skill | 来源 | 安装量 | Stars | 可信度 | 描述 |
|-------|------|--------|-------|--------|------|
| ✓ pptx | anthropics/skills | 75.7K | ⭐ 121886 | 🟢🟢 | PPTX 生成 |

## 未安装

| # | Skill | 来源 | 安装量 | Stars | 可信度 | 描述 |
|---|-------|------|--------|-------|--------|------|
| 1 | pptx | github/awesome-copilot | 15.4K | ⭐ 30639 | 🟢 | PPTX 生成与管理 |
| 2 | easy-prd | instantX-research/... | 200 | ⭐ 11 | 🔵 | 简易 PRD |

输入编号选择要安装的 skill（多选用逗号，如 1,3），或输入 q 退出。
```

规则：无数据的列省略（安装量/Stars）。可信度标识始终显示。描述截断 30 字符，无数据用 `-`。

---

## 安装

**触发**：用户输入编号（如 `1` 或 `1,3`）。多个 skill 顺序安装。

### 1. 确认

```
即将安装：
1. prd (来自 Mehdibargach/...) — npx skills add ...

确认安装？（y/n）
```

### 2. 安装方式

| 条件 | 方式 | 命令 |
|------|------|------|
| 来源 `owner/repo@skill` 且在 skills.sh | A | `npx skills add owner/repo@skill -g -a claude-code` |
| GitHub，SKILL.md 在子目录 | B | sparse clone |
| GitHub，SKILL.md 在根目录 | C | shallow clone |
| 非以上 | A 优先，失败回退 C | — |

安装前检查 `ls ~/.claude/skills/[skill-name]/SKILL.md`，已存在则提示覆盖或跳过。

**方式 A**：`npx skills add owner/repo@skill -g -a claude-code`

**方式 B**（sparse clone）：
```bash
tmpdir=$(mktemp -d) && git clone --filter=blob:none --sparse --depth=1 https://github.com/[owner]/[repo].git "$tmpdir" && cd "$tmpdir" && git sparse-checkout set [skill-path] && mkdir -p ~/.claude/skills/[skill-name] && cp -r [skill-path]/. ~/.claude/skills/[skill-name]/ && rm -rf "$tmpdir"
```

**方式 C**（shallow clone）：
```bash
tmpdir=$(mktemp -d) && git clone --depth=1 https://github.com/[owner]/[repo].git "$tmpdir" && mkdir -p ~/.claude/skills/[skill-name] && cp "$tmpdir/SKILL.md" ~/.claude/skills/[skill-name]/ && rm -rf "$tmpdir"
```

安装后验证 `ls ~/.claude/skills/[skill-name]/SKILL.md`，确认包含 frontmatter。

### 3. 安装后提示

```
✅ 安装完成！如需安全审查，可使用 /skill-vetter 进行扫描。
```

---

## 异常处理

| 场景 | 处理 |
|------|------|
| `gh` 未安装 | 跳过 GitHub API 和 stars，仅用其他渠道 |
| `gh api` 限流 | 回退 WebSearch |
| `npx` 未安装 | 仅 GitHub API + WebSearch，安装限 B/C |
| curl API 超时/失败 | 静默跳过该渠道，不向用户显示错误 |
| 所有渠道失败 | 提示检查网络后重试 |
| 无结果 | 建议换关键词或 `npx skills init` 自建 |
