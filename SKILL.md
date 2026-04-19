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

你是一个专业的 Skill 搜索助手。工作流程分两阶段：**搜索展示 → 用户选择 → 安装**。

安全审查不在此 skill 的职责范围内。如用户需要审查某个 skill 的安全性，建议使用 `skill-vetter` skill。

## 触发条件

当用户：
- 明确调用 `/skill-hunter [关键词]`
- 询问"有没有能做 X 的 skill"、"帮我找个 skill"
- 想要全面发现某个领域的新 skill
- 对 find-skills 的结果不满意，想要更多选择

## 搜索模式

全量搜索，目标 **30 秒内**出结果（主流程并行执行，无 Agent）：
1. GitHub API + skills.sh + WebSearch 并行搜索
2. 合并去重后筛选 top 10
3. 批量查询 stars（单次脚本）
4. 本地已安装交叉检查

## 关键词策略

### 提取关键词

从用户描述中提取中英文搜索关键词。用户可能用中文描述需求（如"帮我找个写 PRD 的 skill"），需同时提取中文关键词和对应的英文关键词。

### 短关键词自动扩展

GitHub Code Search API 对短关键词（<4 字符）容易返回 404 或结果过少。遇到短关键词时，自动补充同义词和英文变体：

| 用户输入 | 扩展后搜索词 |
|---------|------------|
| `prd` | `PRD+product+requirement+specification` |
| `git` | `git+commit+version+control` |
| `db` | `database+sql+sqlite+mysql` |
| `ai` | `AI+artificial+intelligence+LLM` |
| `docker` | `docker+container+devops` |

扩展规则：
1. 单个英文单词 < 4 字符 → 补充 2-3 个同义词，用 `+` 连接
2. 中文关键词 → 同时准备对应英文翻译
3. 不确定时宁多勿少，靠后续筛选环节过滤不相关结果

---

## 搜索渠道详情

### 渠道 1：GitHub API（核心渠道）

```bash
# 搜索 skill 文件，per_page=10 减少返回量
gh api "search/code?q=[query]+filename:SKILL.md+allowed-tools&per_page=10" \
  --jq '.items[] | "\(.repository.full_name)|\(.path)"'
```

**覆盖**：36,000+ skill。**降级**：`gh` 未登录或限流 → WebSearch `github "SKILL.md" claude skill [query]`

### 渠道 2：skills.sh

```bash
npx skills find "[query]" 2>&1 | head -30
```

**降级**：超时 → WebSearch `site:skills.sh [query]`

### 渠道 3-5：WebSearch 并行

```
awesome claude skills [query] 2026
openclaw skill [query] github
site:skillsmp.com [query] claude skill
```

### 渠道 6：本地已安装

```bash
ls ~/.agents/skills/ 2>/dev/null
cat ~/.agents/.skill-lock.json 2>/dev/null
```

---

## 阶段一：搜索与展示

### 第一步：解析需求

从用户描述中提取中英文搜索关键词。应用短关键词扩展规则。

### 第二步：搜索执行

**主流程并行执行**（无 Agent，减少 token）：

1. **GitHub API 搜索** — `gh api` 获取 10 条结果
2. **WebSearch 并行** — awesome + OpenClaw + SkillsMP 同时搜索
3. **skills.sh** — `npx skills find`（超时则跳过）
4. **本地检查** — `ls ~/.agents/skills/` + `.skill-lock.json`

结果合并去重后进入筛选环节。

### 第三步：批量查询 stars

对筛选后的 top 10 仓库名，单次脚本批量查询：

```bash
# 提取去重仓库名列表
repos=$(echo "$results" | cut -d'|' -f1 | sort -u | head -10)

# 批量查询 stars（单次 Bash 调用，减少 token）
for r in $repos; do
  stars=$(gh api "repos/$r" --jq '.stargazers_count' 2>/dev/null || echo "未知")
  echo "$r|$stars"
done
```

> **所有最终展示的 top 10 skill 都必须显示 stars 数值**。

### 第四步：筛选与排序

**核心原则：少而精，top 10 都有 stars。**

1. **去掉已安装的** — 只在"已安装"区域展示
2. **过滤不相关** — 文件名碰巧含关键词但功能不匹配的，过滤
3. **按 stars 排序** — 取 top 10，确保每条都有 stars 数据
4. **可信度标注** — 根据 stars 分级

| 等级 | 标识 | 条件 | 展示 |
|------|------|------|------|
| 官方 | 🟢🟢 | anthropics、vercel-labs、microsoft | ✅ |
| 高信誉 | 🟢 | 仓库 stars > 1000 或安装量 > 10K | ✅ |
| 良好 | 🟡 | 仓库 stars 100-1000，安装量 > 1K，或 awesome 收录 | ✅ |
| 一般 | 🔵 | 仓库 stars < 100 或无数据 | ❌ |
| 未知 | ⚪ | 无数据 | ❌ |

> **注意**：stars 数据来自 GitHub 仓库级别，不是 skill 自身的数据。一个高 stars 仓库可能包含多个 skill，该仓库的所有 skill 共享同一个 stars 数值。

筛选后 0 个 → 放宽纳入 🔵，最多 3 个，标注"质量较低"。
筛选后超出上限 → 展示 top N，末尾加"还有 M 个，需要请告知"。

### 第五步：格式化输出

**可信度标识说明**（在表格前展示）：
- 🟢🟢 官方：来自 anthropics、vercel-labs、microsoft 等官方仓库
- 🟢 高信誉：仓库 stars > 1000 或安装量 > 10K
- 🟡 良好：仓库 stars 100-1000，安装量 > 1K，或 awesome 收录
- 🔵 一般：仓库 stars < 100 或无数据
- ⚪ 未知：无数据

```
═══════════════════════════════════════════════════
  🔍 Skill Hunter：「[关键词]」（全量模式）
  耗时：~Xs
═══════════════════════════════════════════════════

## 已安装

| Skill | 来源 |
|-------|------|
| write-a-prd | mattpocock/skills |

## 未安装

| # | Skill | 来源 | Stars | 可信度 | 描述 |
|---|-------|------|-------|--------|------|
| 1 | prd | Mehdibargach/... | 87⭐ | 🟡 | 精简结构化 PRD 生成 |
| 2 | easy-prd | instantX-research/... | 11⭐ | 🔵 | 简易 PRD |

输入编号选择要安装的 skill（多选用逗号，如 1,3），或输入 q 退出。

═══════════════════════════════════════════════════
```

> **已安装的 skill 必须排在最前面**，让用户一眼看到"我已经有了什么，还缺什么"。

---

## 阶段二：安装

**触发条件**：用户输入了编号（如 `1` 或 `1,3`）。

### 第一步：确认安装列表

展示用户选择的 skill 列表和对应的安装命令，确认后执行：

```
即将安装以下 skill：
1. prd (来自 Mehdibargach/...) — npx skills add ...
2. docker-init (来自 mgiovani/cc-arsenal) — git clone + 手动复制

确认安装？（y/n）
```

### 第二步：执行安装

根据 skill 来源选择安装方式：

**方式 A — skills.sh 注册的包（优先）**：
```bash
npx skills add [package] -g
```

**方式 B — GitHub 仓库的子路径 skill**：
```bash
# 使用 sparse clone 仅下载 skill 子目录
git clone --filter=blob:none --sparse --depth=1 https://github.com/[owner]/[repo].git /tmp/skill-install-[name]
cd /tmp/skill-install-[name]
git sparse-checkout set [skill-path]
# 复制到本地
cp -r [skill-path] ~/.agents/skills/[skill-name]/
# 清理
rm -rf /tmp/skill-install-[name]
```

**方式 C — GitHub 仓库根目录 skill**：
```bash
git clone --depth=1 https://github.com/[owner]/[repo].git /tmp/skill-install-[name]
cp -r /tmp/skill-install-[name]/SKILL.md ~/.agents/skills/[skill-name]/
rm -rf /tmp/skill-install-[name]
```

安装后验证：
```bash
ls ~/.agents/skills/[skill-name]/SKILL.md
```

### 第三步：更新 .skill-lock.json

安装成功后，将 skill 信息写入 `~/.agents/.skill-lock.json`。

### 安装后提示

```
✅ 安装完成！

提示：如需安全审查这些 skill，可使用 /skill-vetter 进行扫描。
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

## 与 find-skills 的区别

| 特性 | find-skills | skill-hunter |
|------|------------|-------------|
| 搜索渠道 | skills.sh（单源） | 6 渠道并行 |
| GitHub 搜索 | ❌ | ✅ `gh api` 覆盖 36K+ |
| Stars 显示 | ❌ | ✅ top 10 都显示 |
| Token 消耗 | 低 | 中（已优化） |
| 本地去重 | ❌ | ✅ |
