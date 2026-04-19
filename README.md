# Skill Hunter

全渠道 Claude Code Skill 搜索器 — 覆盖 6 大渠道、880,000+ Skill，搜索结果按可信度排序并支持一键安装。

---

## 安装

将 `SKILL.md` 复制到本地 skills 目录：

```bash
cp SKILL.md ~/.agents/skills/skill-hunter/SKILL.md
```

## 使用

```
/skill-hunter [关键词]      # 全量搜索（~30s）
```

示例：
- `/skill-hunter docker` — 搜索 Docker 相关 Skill
- `/skill-hunter PRD` — 搜索 PRD 相关 Skill

## 特性

| 特性 | 说明 |
|------|------|
| 多渠道搜索 | GitHub API (880K+)、skills.sh、awesome 列表、ClawHub、SkillsMP |
| 可信度排序 | 按仓库 stars 分级：官方 > 高信誉 > 良好 > 一般 |
| Stars 显示 | 展示结果的 top 10 都显示 stars 数值 |
| 本地去重 | 自动过滤已安装的 Skill |
| 一键安装 | 选择编号后自动安装，支持 sparse clone 优化 |

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

A multi-channel Claude Code Skill search tool — covering 6 channels, 880,000+ skills, with credibility-ranked results and one-click installation.

---

### Installation

Copy `SKILL.md` to your local skills directory:

```bash
cp SKILL.md ~/.agents/skills/skill-hunter/SKILL.md
```

### Usage

```
/skill-hunter [keyword]   # Full search (~30s)
```

Examples:
- `/skill-hunter docker` — Search Docker-related skills
- `/skill-hunter PRD` — Search PRD-related skills

### Features

| Feature | Description |
|---------|-------------|
| Multi-channel Search | GitHub API (880K+), skills.sh, awesome lists, ClawHub, SkillsMP |
| Credibility Ranking | Stars-based tiers: Official > Trusted > Good > Average |
| Stars Display | Top 10 results always show star counts |
| Local Dedup | Automatically filters out already-installed skills |
| One-click Install | Select by number and auto-install with sparse clone optimization |

### Project Structure

```
skill-hunter/
├── SKILL.md              # Core skill definition
├── LICENSE               # MIT License
└── README.md
```

### License

This project is licensed under the [MIT License](LICENSE).
