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
| GitHub API | `gh api "search/code?q=[同义词query]+filename:SKILL.md+allowed-tools&per_page=10" --jq '.items[] | "\(.repository.full_name)\|\(.path)"'` | WebSearch `github "SKILL.md" claude skill [query]` |
| WebSearch×3 | `awesome claude skills [query] 2026`、`openclaw skill [query] github`、`site:skills.sh [query]` | — |
| 本地已安装 | Glob `~/.claude/skills/*/SKILL.md` | — |

> skills.sh 是最稳定的数据源，直接返回 skill 名称、来源、安装量。GitHub API 的 `search/code` 搜文件内容但**不返回 stars 和描述**，需额外查询（见第 3 步）。

### 3. 补充数据与仓库验证

**合并去重**：所有渠道结果按 `owner/repo@skill名` 去重，有安装量 + 有 stars + 有描述 的条目优先保留。

**安装量**：`npx skills find` 输出中直接包含（如 `5.9K installs`）。缺失时视为 0 参与排序。

**描述获取**：`npx skills find` 和 GitHub API 均不返回描述。从 WebSearch 结果摘要中提取，或用 WebSearch `site:skills.sh [owner/repo skill名]` 获取 skills.sh 页面摘要。无法获取时描述显示 `-`。

**Stars 获取**：`search/code` 的 repository 对象是精简版，不含 `stargazers_count`，需单独查询。

1. 过滤非 GitHub 仓库：格式必须为 `owner/repo`（含 `/`）。不合法的（如 `smithery.ai`）→ 不查 stars，标记"非 GitHub 来源"
2. **单次 Bash 调用**批量获取 stars：
```bash
for r in owner1/repo1 owner2/repo2; do echo "$r $(gh api "repos/$r" --jq '.stargazers_count' 2>/dev/null || echo -)"; done
```
3. 处理 404：对返回 404 的仓库，WebSearch `"[owner/repo] github"` 验证：
   - **已删除** → 从结果中移除
   - **已改名** → 用新名称重新获取 stars，**同步更新搜索结果中的来源信息**（后续安装也用新名称）
   - **私有仓库** → 保留，stars 显示 `-`
   > 无法区分"私有"和"已删除"时，优先保留（不错杀）。
4. `gh` 未安装 → 跳过 stars 获取，所有 stars 显示 `-`

### 4. 筛选排序

1. 已安装 → 仅在"已安装"区域展示
2. 过滤不相关 → 文件名碰巧含关键词但功能不匹配的
3. 综合排序 → 取 top 10：
   - 主排序：安装数量降序（缺失视为 0）
   - 次排序：stars 降序
   - 末排序：可信度等级（官方 > 高信誉 > 良好 > 一般）
   - 最终排序：skill 名称字母序

**可信度分级**（按优先级判定）：

| 等级 | 标识 | 条件 |
|------|------|------|
| 官方 | 🟢🟢 | 仓库 owner 为 anthropics、vercel-labs、microsoft、openai、figma |
| 高信誉 | 🟢 | stars ≥ 1000 或安装量 ≥ 10K |
| 良好 | 🟡 | 100 ≤ stars < 1000，或 1K ≤ 安装量 < 10K，或 awesome 收录 |
| 一般 | 🔵 | stars < 100 且安装量 < 1K |

> stars 为仓库级别，非 skill 自身。同一仓库的多个 skill 共享 stars 数值。

筛选后 0 个 → 放宽纳入 🔵（最多 3 个，标注"质量较低"）。超出上限 → 展示 top N + "还有 M 个，需要请告知"。

### 5. 格式化输出

**使用列表格式**，已安装和未安装使用相同格式：

```
🔍 Skill Hunter：「[关键词]」（全渠道搜索）

## 已安装
 ✓ pptx | anthropics/skills | 安装 75.7K | ⭐ 121886 | 🟢🟢 | PPTX 生成

## 未安装
 1. pptx | github/awesome-copilot | 安装 15.4K | ⭐ 30639 | 🟢 | PPTX 生成与管理
 2. easy-prd | instantX-research/... | 安装 200 | ⭐ 11 | 🔵 | 简易 PRD

输入编号选择要安装的 skill（多选用逗号，如 1,3），或输入 q 退出。
```

**输出规则**：
- 已安装和未安装分开展示，已安装在前
- 两者格式一致：`标识. skill名 | 来源 | 安装数量 | ⭐ 数量 | 可信度emoji | 描述`
- 已安装用 `✓` 标识，未安装用编号 `1.` `2.` ...
- 安装量格式：`5.9K`、`200`；无数据时省略
- Stars 格式：`⭐ 87`；无数据时省略
- 可信度 emoji：🟢🟢、🟢、🟡、🔵
- 描述截断到 30 字符，超出用 `…` 结尾
- 来源过长时截断为 `owner/rep...`

---

## 安装

**触发**：用户输入编号（如 `1` 或 `1,3`）。多个 skill 时顺序安装。

### 1. 确认列表

```
即将安装以下 skill：
1. prd (来自 Mehdibargach/...) — npx skills add ...
2. docker-init (来自 mgiovani/cc-arsenal) — git clone + 手动复制

确认安装？（y/n）
```

### 2. 判定安装方式

| 条件 | 方式 | 命令 |
|------|------|------|
| 来源格式为 `owner/repo@skill` 且在 skills.sh 上 | A（优先） | `npx skills add owner/repo@skill -g -a claude-code` |
| 来源为 GitHub，SKILL.md 在子目录中 | B | sparse clone |
| 来源为 GitHub，SKILL.md 在根目录 | C | shallow clone |
| 非以上情况 | A 优先，失败回退 C | — |

> `[package]` 格式为 `owner/repo@skill名`，与 skills.sh 的输出格式一致。同名 skill 已安装时提示用户选择覆盖或跳过。

### 3. 执行安装

安装前检查：`ls ~/.claude/skills/[skill-name]/SKILL.md`，如已存在则提示用户选择覆盖或跳过。

**方式 A**：`npx skills add owner/repo@skill -g -a claude-code`

**方式 B**：
```bash
tmpdir=$(mktemp -d) && git clone --filter=blob:none --sparse --depth=1 https://github.com/[owner]/[repo].git "$tmpdir" && cd "$tmpdir" && git sparse-checkout set [skill-path] && mkdir -p ~/.claude/skills/[skill-name] && cp -r [skill-path]/. ~/.claude/skills/[skill-name]/ && [ -n "$tmpdir" ] && [ -d "$tmpdir" ] && rm -rf "$tmpdir"
```

**方式 C**：
```bash
tmpdir=$(mktemp -d) && git clone --depth=1 https://github.com/[owner]/[repo].git "$tmpdir" && mkdir -p ~/.claude/skills/[skill-name] && cp "$tmpdir/SKILL.md" ~/.claude/skills/[skill-name]/ && [ -n "$tmpdir" ] && [ -d "$tmpdir" ] && rm -rf "$tmpdir"
```

安装后验证 `ls ~/.claude/skills/[skill-name]/SKILL.md`，确认文件包含 SKILL.md frontmatter（`---` 开头）。

### 4. 安装后提示

```
✅ 安装完成！如需安全审查，可使用 /skill-vetter 进行扫描。
```

---

## 异常处理

| 场景 | 处理 |
|------|------|
| `gh` 未安装 | 跳过 GitHub API 和 stars 获取，仅用 skills.sh + WebSearch |
| `gh api` 限流 | 回退 WebSearch |
| `npx skills find` 超时 | 跳过，依赖其他渠道 |
| `npx` 未安装 | 仅用 GitHub API + WebSearch，安装方式限 B/C |
| git clone 失败 | 提示手动安装，给出 GitHub 链接 |
| 所有渠道失败 | 提示网络问题，建议检查网络连接后重试 |
| 无结果 | 建议：换关键词 或 `npx skills init` 自建 |
