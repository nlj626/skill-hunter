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
> If empty (first use) → Ask for API keys below, save config.json, then start search
> If non-empty → Skip setup, go directly to search
>
> **No keyword?** If user invoked `/skill-hunter` without a keyword (e.g. just `/skill-hunter`), ask "What skill are you looking for?" before searching.
>
> **Language**: Match the user's language. If user speaks Chinese, reply in Chinese. Default: English. Do NOT mix languages in one response.

# Skill Hunter — Multi-Channel Skill Search

## First-Time Setup (config.json missing only)

Ask ONE question at a time. Show question 1, STOP and wait for user response, then show question 2. Do NOT show both questions in one response.

1. "Enter your SkillsMP API Key (free at skillsmp.com/docs/api), or type `s` to skip:" → **STOP HERE. Wait for user input.**
2. "Enter your ClawHub Token (via `clawhub` CLI login), or type `s` to skip:" → **STOP HERE. Wait for user input.**

Save with one Bash command (do NOT overthink, just run it):
```bash
mkdir -p ~/.claude/skills/skill-hunter && echo '{"asked":true,"skillsmp_key":"[user input or empty]","clawhub_token":"[user input or empty]"}' > ~/.claude/skills/skill-hunter/config.json
```

If user skipped both: `echo '{"asked":true}' > ~/.claude/skills/skill-hunter/config.json`

Then say: "Config saved. You can manually edit `~/.claude/skills/skill-hunter/config.json` later."

**After saving, if user provided a keyword → immediately start search. If no keyword → ask "What skill are you looking for?"**

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
| ClawHub | `curl -s --max-time 10 -H "Authorization: Bearer [token]" "https://clawhub.ai/api/v1/search?q=[keyword]&limit=10" 2>/dev/null` | WebSearch `"clawhub [keyword] claude skill"` |
| SkillsMP | `curl -s --max-time 10 -H "Authorization: Bearer [key]" "https://skillsmp.com/api/v1/skills/search?q=[keyword]&limit=10" 2>/dev/null` | WebSearch `"skillsmp [keyword] claude skill"` |
| awesome | WebSearch `"awesome claude skills [keyword] github 2026"` | same |
| Local | Glob `~/.claude/skills/*/SKILL.md` | same |

> curl timeout or error → skip that channel silently (do NOT show error to user). GitHub API fallback: WebSearch `"github SKILL.md claude skill [keyword]"`.

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

**Credibility** (MUST follow strictly, do not guess): Official 🟢🟢 (anthropics/vercel-labs/microsoft/openai/figma) > Trusted 🟢 (stars≥1K or installs≥10K) > Good 🟡 (stars≥100 or installs≥1K) > Average 🔵

0 results → relax to include 🔵 (max 3, mark "lower quality").

### 5. Output Format

Use **markdown table** format for BOTH sections — same columns, same alignment. Installed section MUST include all columns (Source, Installs, Stars, Trust, Description), do NOT simplify or omit columns for installed skills.

```
🔍 Skill Hunter: "[keyword]" (multi-channel)

## Installed

| Skill | Source | Installs | Stars | Trust | Description |
|-------|--------|----------|-------|-------|-------------|
| ✓ pptx | anthropics/skills | 75.7K | ⭐ 121886 | 🟢🟢 | PPTX generation |

## Not Installed

| # | Skill | Source | Installs | Stars | Trust | Description |
|---|-------|--------|----------|-------|-------|-------------|
| 1 | pptx | github/awesome-copilot | 15.4K | ⭐ 30639 | 🟢 | PPTX generation |
| 2 | easy-prd | instantX-research/... | 200 | ⭐ 11 | 🔵 | Simple PRD |

Enter number(s) to install (comma-separated, e.g. 1,3), or q to quit.
```

Rules: omit columns with no data (installs/stars). Trust badge always shown. Description max 30 chars, `-` if missing.

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
| curl API timeout/fail | Skip channel silently, no error output to user |
| All channels fail | Suggest checking network |
| No results | Suggest different keyword or `npx skills init` |
