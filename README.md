# Skill Hunter

全渠道 Claude Code Skill 搜索器 — 覆盖 skills.sh、GitHub API、WebSearch（awesome/ClawHub/SkillsMP），搜索结果按安装量和可信度排序，支持一键安装。

---

## 安装

| 方式 | 命令 | 平台 | 说明 |
|------|------|------|------|
| npx（推荐） | `npx skills add nlj626/skill-hunter -g -a claude-code` | 全平台 | 自动安装到 `~/.claude/skills/` |
| 手动 macOS / Linux | `mkdir -p ~/.claude/skills/skill-hunter && cp SKILL.md ~/.claude/skills/skill-hunter/` | macOS / Linux | 手动复制到 skills 目录 |
| 手动 Windows | 见下方代码块 | Windows | 手动复制到 skills 目录 |

**Windows 手动安装**（PowerShell）：
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\skill-hunter"
Copy-Item SKILL.md "$env:USERPROFILE\.claude\skills\skill-hunter\"
```

> **注意**：npx 方式必须带 `-a claude-code` 参数，否则可能安装到错误路径导致无法识别。Windows 用户如遇 symlink 权限问题，可加 `--copy`：`npx skills add nlj626/skill-hunter -g -a claude-code --copy`。skill 文件必须放在 `~/.claude/skills/` 目录下，Claude Code 才能识别。

## 使用

```
/skill-hunter [关键词]
```

示例：
- `/skill-hunter docker` — 搜索 Docker 相关 Skill
- `/skill-hunter PRD` — 搜索 PRD 相关 Skill
- `/skill-hunter 回测` — 中文也能搜索（自动翻译为英文）

## 功能

### 搜索渠道

| 渠道 | 数据 | 说明 |
|------|------|------|
| skills.sh | 名称、来源、安装量 | 核心渠道，最稳定 |
| GitHub API `search/code` | 名称、来源、文件路径 | 搜 SKILL.md 文件内容 |
| WebSearch × 3 | 名称、来源 | awesome 列表、ClawHub、SkillsMP |
| 本地已安装 | 已安装 skill 列表 | 自动过滤，分开展示 |

### 排序规则

1. **安装量**（主排序）— 降序
2. **Stars**（次排序）— 降序，通过 `gh api` 批量获取
3. **可信度** — 官方 🟢🟢 > 高信誉 🟢 > 良好 🟡 > 一般 🔵

### 展示格式

列表格式，每条 skill 一行：
```
 1. prd | github/awesome-copilot | 安装 15.4K | ⭐ 30639 | 🟢 | PRD 生成与管理
 2. write-a-prd | mattpocock/skills | 安装 14.2K | ⭐ 16733 | 🟢 | 结构化 PRD 编写
```

### 安装方式

选择编号后自动安装，支持三种方式：
- **方式 A**：`npx skills add` — skills.sh 包（优先）
- **方式 B**：`git clone --sparse` — GitHub 子路径 skill
- **方式 C**：`git clone --depth=1` — GitHub 根目录 skill

## 项目结构

```
skill-hunter/
├── SKILL.md              # Skill 定义文件（核心产物）
├── LICENSE               # MIT 协议
└── README.md
```

## 协议

本项目采用 [MIT License](LICENSE)。

---

## English

A multi-channel Claude Code Skill search tool — covering skills.sh, GitHub API, WebSearch (awesome/ClawHub/SkillsMP), with install-count ranking and one-click installation.

---

### Installation

| Method | Command | Platform | Description |
|--------|---------|----------|-------------|
| npx (Recommended) | `npx skills add nlj626/skill-hunter -g -a claude-code` | All | Auto-installs to `~/.claude/skills/` |
| Manual macOS / Linux | `mkdir -p ~/.claude/skills/skill-hunter && cp SKILL.md ~/.claude/skills/skill-hunter/` | macOS / Linux | Copy to skills directory |
| Manual Windows | See code block below | Windows | Copy to skills directory |

**Manual Windows install** (PowerShell):
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\skill-hunter"
Copy-Item SKILL.md "$env:USERPROFILE\.claude\skills\skill-hunter\"
```

> **Note**: The `-a claude-code` flag is required for npx — without it, the skill may be installed to the wrong path. Windows users experiencing symlink permission issues can add `--copy`: `npx skills add nlj626/skill-hunter -g -a claude-code --copy`. The skill file must be placed under `~/.claude/skills/` for Claude Code to detect it.

### Usage

```
/skill-hunter [keyword]
```

Examples:
- `/skill-hunter docker` — Search Docker-related skills
- `/skill-hunter PRD` — Search PRD-related skills

### Features

| Feature | Description |
|---------|-------------|
| Multi-channel Search | skills.sh, GitHub API, WebSearch (awesome/ClawHub/SkillsMP) |
| Install-count Ranking | Primary sort by install count, secondary by stars |
| Credibility Tiers | Official 🟢🟢 > Trusted 🟢 > Good 🟡 > Average 🔵 |
| Stars Display | Batch-fetched via `gh api`, displayed for top results |
| Local Dedup | Automatically filters out already-installed skills |
| One-click Install | Select by number, supports npx/sparse-clone/full-clone |

### Project Structure

```
skill-hunter/
├── SKILL.md              # Core skill definition
├── LICENSE               # MIT License
└── README.md
```

### License

This project is licensed under the [MIT License](LICENSE).
