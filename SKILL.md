---
name: skill-hunter
description: Multi-channel Claude Code Skill search — GitHub API, skills.sh, awesome, ClawHub, SkillsMP. Shows stars for top 10, one-click install.
allowed-tools:
  - Bash
  - WebSearch
  - Read
  - Glob
---

> **First action**: Run `cat ~/.claude/skills/skill-hunter/config.json 2>/dev/null`
> Empty output (first use) → Ask for API keys → Save → Wait for user's search keyword
> Non-empty output → Skip setup, go directly to search

# Skill Hunter — Multi-Channel Skill Search

## First-Time Setup

If config.json doesn't exist, ask in plain text (two rounds):

1. "Enter your SkillsMP API Key (free at skillsmp.com/docs/api), or type `s` to skip:"
2. "Enter your ClawHub Token (via `clawhub` CLI login), or type `s` to skip:"

Save to `~/.claude/skills/skill-hunter/config.json`:
```json
{"asked":true,"skillsmp_key":"...","clawhub_token":"..."}
```

After saving, output: `✅ Setup complete! Enter a skill keyword to search (e.g. ppt, docker, PRD).`

**Do NOT auto-search. Wait for the user to provide a keyword.**

## Search

Target: **30 seconds**. All channels parallel, no Agent sub-processes.

### 1. Parse Keywords

| Channel | Format | Example |
|---------|--------|---------|
| `npx skills find` | Raw keyword | `prd` or `pr review` |
| GitHub API | Synonyms joined by `+` (AND) | `PRD+product+requirement` |
| WebSearch / API | Raw keyword | `prd` |

Short keywords (≤3 chars) trigger synonym expansion (GitHub API only): `prd` → `PRD+product+requirement`.

### 2. Parallel Search (6 channels, single batch)

**All 6 must fire in one response. No sequential execution.**

Read config.json for keys, then:

| Channel | Has Key | No Key (fallback) |
|---------|---------|-------------------|
| skills.sh | `npx skills find "[keyword]" 2>&1 \| head -30` | same |
| GitHub API | `gh api "search/code?q=[synonyms]+filename:SKILL.md+allowed-tools&per_page=10" --jq '.items[] \| "\(.repository.full_name)\|\(.path)"'` | same |
| ClawHub | `curl -s --max-time 10 -H "Authorization: Bearer [token]" "https://clawhub.ai/api/v1/search?q=[keyword]&limit=10"` | WebSearch `"clawhub [keyword] claude skill"` |
| SkillsMP | `curl -s --max-time 10 -H "Authorization: Bearer [key]" "https://skillsmp.com/api/v1/skills/search?q=[keyword]&limit=10"` | WebSearch `"skillsmp [keyword] claude skill"` |
| awesome | WebSearch `"awesome claude skills [keyword] github 2026"` | same |
| Local | Glob `~/.claude/skills/*/SKILL.md` | same |

> curl timeout = 10s → skip that channel. GitHub API fallback: WebSearch `"github SKILL.md claude skill [keyword]"`.

### 3. Merge & Stars

**Dedup** by `owner/repo@skill-name`, keep entry with most data.

**Description**: Extract from channel results. Missing → show `-`. No extra queries.

**Stars**: For unique repos in top 10 (usually 3-5), single Bash call:
```bash
for r in owner1/repo1 owner2/repo2; do echo "$r $(gh api "repos/$r" --jq '.stargazers_count' 2>/dev/null || echo -)"; done
```
- 404 → skip silently
- Non-GitHub source → mark "non-GitHub", no stars query
- `gh` not installed → all stars show `-`

### 4. Filter & Sort

1. Installed → show only in "Installed" section
2. Remove irrelevant matches
3. Sort top 10 by: installs ↓ → stars ↓ → credibility → name A-Z

**Credibility**: Official 🟢🟢 (anthropics/vercel-labs/microsoft/openai/figma) > Trusted 🟢 (stars≥1K or installs≥10K) > Good 🟡 (stars≥100 or installs≥1K) > Average 🔵

0 results → relax to include 🔵 (max 3, mark "lower quality").

### 5. Output Format

```
🔍 Skill Hunter: "[keyword]" (multi-channel)

## Installed
 ✓ pptx | anthropics/skills | 75.7K installs | ⭐ 121886 | 🟢🟢 | PPTX generation

## Not Installed
 1. pptx | github/awesome-copilot | 15.4K installs | ⭐ 30639 | 🟢 | PPTX generation
 2. easy-prd | instantX-research/... | 200 installs | ⭐ 11 | 🔵 | Simple PRD

Enter number(s) to install (comma-separated, e.g. 1,3), or q to quit.
```

Rules: installs format `5.9K`/`200`, omit if missing; stars `⭐ 87`, omit if missing; description capped at 30 chars.

---

## Install

User enters number(s) → install sequentially.

### 1. Confirm

```
About to install:
1. prd (from Mehdibargach/...) — npx skills add ...

Proceed? (y/n)
```

### 2. Install Method

| Condition | Method | Command |
|-----------|--------|---------|
| `owner/repo@skill` on skills.sh | A | `npx skills add owner/repo@skill -g -a claude-code` |
| GitHub, SKILL.md in subdirectory | B | sparse clone |
| GitHub, SKILL.md at root | C | shallow clone |
| Other | A first, fallback C | — |

Check `ls ~/.claude/skills/[name]/SKILL.md` first; prompt overwrite/skip if exists.

**Method A**: `npx skills add owner/repo@skill -g -a claude-code`

**Method B** (sparse clone):
```bash
tmpdir=$(mktemp -d) && git clone --filter=blob:none --sparse --depth=1 https://github.com/[owner]/[repo].git "$tmpdir" && cd "$tmpdir" && git sparse-checkout set [skill-path] && mkdir -p ~/.claude/skills/[skill-name] && cp -r [skill-path]/. ~/.claude/skills/[skill-name]/ && rm -rf "$tmpdir"
```

**Method C** (shallow clone):
```bash
tmpdir=$(mktemp -d) && git clone --depth=1 https://github.com/[owner]/[repo].git "$tmpdir" && mkdir -p ~/.claude/skills/[skill-name] && cp "$tmpdir/SKILL.md" ~/.claude/skills/[skill-name]/ && rm -rf "$tmpdir"
```

Verify with `ls ~/.claude/skills/[name]/SKILL.md` (must contain frontmatter).

### 3. Post-Install

```
✅ Installed! Run /skill-vetter for a security scan.
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| `gh` not installed | Skip GitHub API + stars, use other channels |
| `gh api` rate-limited | Fallback to WebSearch |
| `npx` not installed | GitHub API + WebSearch only; install limited to B/C |
| curl API timeout/fail | Skip channel, no impact on others |
| All channels fail | Suggest checking network |
| No results | Suggest different keyword or `npx skills init` |
