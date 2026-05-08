# Shipping `banger-board` as a Claude Code Skill

How to package `banger-board` for distribution to other Claude Code users.

---

## What is a Claude Code skill?

A Claude Code skill is a single Markdown file (`SKILL.md`) with YAML frontmatter that defines a slash command (`/banger-board`). When a user types the command, Claude Code loads the skill body into context as a procedural prompt and executes it.

Anatomy:
```
~/.claude/skills/<skill-name>/
├── SKILL.md       # required: frontmatter + body
├── README.md      # optional: human-readable overview
├── SHIP.md        # optional: this file
└── data/          # optional: skill-specific cache/config
```

**Frontmatter contract** (top of `SKILL.md`):
```yaml
---
name: banger-board
description: <one-line summary including trigger pattern>
---
```

The `description` is what Claude Code uses to discover the skill — keep it specific and include the trigger pattern (e.g., `Triggered by /banger-board [args]`).

---

## Distribution options

### Option 1: Public GitHub repo (recommended for shareable skills)

```bash
# 1. Create a clean copy
cp -r ~/.claude/skills/banger-board /tmp/banger-board-public
cd /tmp/banger-board-public

# 2. Strip personal vault references from SKILL.md
# Search for these patterns and either generalize or remove:
#   - wiki/trading/theses/banger-board.md  → replace w/ generic "your sleeve ledger"
#   - /Users/Vivek/Development/...         → replace w/ relative paths or env vars
#   - personal book / Bildof exclusions    → keep as example, note "configure for your accounts"

# 3. Init git + push
git init
git add SKILL.md README.md SHIP.md
git commit -m "Initial release: banger-board skill"
gh repo create banger-board --public --source=. --push
```

Recipients install via:
```bash
mkdir -p ~/.claude/skills
git clone https://github.com/<you>/banger-board ~/.claude/skills/banger-board
```

### Option 2: Plugin marketplace (for skill bundles)

Claude Code supports plugin marketplaces — bundles of skills + agents + commands.

Structure:
```
my-trading-plugin/
├── plugin.json
└── skills/
    └── banger-board/
        ├── SKILL.md
        └── README.md
```

`plugin.json` schema:
```json
{
  "name": "trading-skills",
  "version": "0.1.0",
  "description": "Equity pop-hunting + risk gates",
  "skills": ["banger-board"]
}
```

Recipients install via:
```bash
claude plugin add github.com/<you>/trading-skills
```

### Option 3: Direct file share (simplest)

Just send `SKILL.md` + `README.md` + `SHIP.md` as a tarball or zip. Recipients drop into `~/.claude/skills/banger-board/`.

```bash
tar czf banger-board-skill.tar.gz -C ~/.claude/skills banger-board
# share banger-board-skill.tar.gz

# recipient:
mkdir -p ~/.claude/skills
tar xzf banger-board-skill.tar.gz -C ~/.claude/skills
```

---

## Pre-ship checklist

Before sharing publicly, audit `SKILL.md` for personal coupling:

### Hard requirements (must remove/parameterize)

- [ ] Remove all absolute paths (`/Users/Vivek/...`) — replace w/ env vars (`$VAULT_ROOT`, `$RADON_PATH`) or document as user-configured
- [ ] Remove personal vault binding (`wiki/trading/theses/banger-board.md`) — note "configure your own sleeve ledger doc"
- [ ] Remove personal account exclusions (`Bildof LLC`, `Innocore`, `Ally`) — generalize to "your speculative sleeve account"
- [ ] Strip API keys, tokens, account IDs from anywhere they accidentally appear (search `grep -i "key\|token\|secret\|11087763\|UW_TOKEN=[^$]"`)
- [ ] Remove personal NAV references (`$926K`, `$813K`) — show as `$<your_nav>`

### Soft cleanup (nice-to-have)

- [ ] Remove session-specific dates (`2026-05-08 backtest`) → keep concept, redact day-specific picks
- [ ] Generalize ticker examples beyond personal portfolio holdings
- [ ] Add a `## Configuration` section at top of SKILL.md noting env vars + vault path setup
- [ ] Add a `## Setup` section to README.md w/ install + first-run steps

### Companion-repo dependencies

`banger-board` depends on **two private companion repos** (`radon`, `traderkit`). Options:

1. **Open-source the deps** — fork + sanitize + publish each as separate repo
2. **Make deps optional** — gate radon-* phases behind feature flags so skill works w/o them (loses convexity edge-gate but core 13-signal still functions)
3. **Build minimal traderkit subset** — extract only the MCP tools banger-board needs (`signal_rank`, `screen_options`, `earnings_calendar`, `combo_fillability`, `thesis_fit`, `check_concentration`) into a small standalone npm/pip package

**Recommended path: option 2 + option 3.** Make radon truly optional (skip flags), and ship a minimal traderkit-lite that recipients can `npm install` for the required MCP tools.

---

## Recipient-side install steps

Document this in README.md so recipients know what to do:

```markdown
## Setup

### 1. Install Claude Code
https://claude.com/claude-code

### 2. Clone skill
git clone https://github.com/<you>/banger-board ~/.claude/skills/banger-board

### 3. Set env vars
Add to ~/.zshrc / ~/.bashrc:
export UW_TOKEN=...           # https://unusualwhales.com/api
export FMP_API_KEY=...         # https://financialmodelingprep.com/

### 4. Install MCP servers
Add to ~/.claude.json under "mcpServers":
- unusualwhales: uw API wrapper
- traderkit: trading toolkit (companion repo)
- exa (optional): web search for signal #11

### 5. Configure your speculative sleeve
Create a sleeve ledger doc at the path of your choice. Schema:
- 5% NAV ceiling
- max 5 active picks
- exit rules: 7d hard close · -50% stop · NO earnings holds

### 6. Run
/banger-board                       # default top-10 strict
/banger-board --mode smallcap       # small-cap moonshots
```

---

## Versioning

Add a `version:` field to `SKILL.md` frontmatter when releasing:

```yaml
---
name: banger-board
version: 1.0.0
description: ...
---
```

Bump on:
- **MAJOR** (1.0 → 2.0): breaking changes to args, mode names, output schema
- **MINOR** (1.0 → 1.1): new modes, new signals, additive flags
- **PATCH** (1.0.0 → 1.0.1): threshold tuning, doc fixes, bug fixes

Tag releases in git:
```bash
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
```

---

## Testing your shipped version

Before releasing, test in a clean Claude Code session:

```bash
# 1. Move your existing skill aside
mv ~/.claude/skills/banger-board ~/.claude/skills/banger-board.dev

# 2. Install the shipped version as a fresh recipient would
git clone https://github.com/<you>/banger-board ~/.claude/skills/banger-board

# 3. Open a new Claude Code session, run /banger-board
# Verify:
#  - Skill loads w/o errors
#  - All env vars + MCP servers wired correctly
#  - Sample run produces a coherent board
#  - No personal paths leak in output

# 4. Restore your dev version
rm -rf ~/.claude/skills/banger-board
mv ~/.claude/skills/banger-board.dev ~/.claude/skills/banger-board
```

---

## License

Recommend **MIT** for skills — minimal restrictions, allows commercial use, encourages community contribution.

Add `LICENSE` file:
```
MIT License

Copyright (c) 2026 <Your Name>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction...
[full MIT text]
```

---

## Contribution back

If recipients fork/improve, encourage upstream contribution:

```markdown
## Contributing

1. Fork
2. Add new signal/mode/theme on a feature branch
3. Backtest against ≥1 historical pop event (document in `backtests/`)
4. Open PR w/ backtest results

PRs without backtest validation will not be merged for new signals.
```

---

## See also

- Claude Code skill docs: https://docs.anthropic.com/claude-code/skills (or current path)
- Plugin marketplace docs: https://docs.anthropic.com/claude-code/plugins
- Companion skill examples in this repo: `/trade-refresh`, `/screen-cc`, `/nl-trade`
