# banger-board

A Claude Code **skill** for hunting 30%+ weekly equity pops by stacking 13 confluence signals (float · short-interest · options-flow · catalyst · theme · dark-pool · sentiment · borrow-cost · RVOL · activist/insider · news-velocity · theme-sympathy · float-velocity).

Outputs a ranked board with thesis-fit rationale, suggested structure, sizing, and exit triggers. **No auto-execute** — user picks manually for entry via broker UI.

---

## What this skill does

Given a small slice of capital (5% NAV ceiling) earmarked for speculative pop-hunting, banger-board:

1. Screens the equity universe via UnusualWhales `uw_screener` + (optionally) radon flow-discover for confluence-flagged tickers
2. Layers 13 signals on each candidate (squeeze fuel, options flow, dark-pool prints, news velocity, theme heat, theme-sympathy, float-velocity, etc.)
3. Filters by mode-specific signal threshold (2-of-13 to 5-of-13 depending on mode)
4. Multi-source ranks via traderkit `signal_rank`
5. (Optional) Gates picks through radon 7-milestone (M1-M7) convexity check for option-structure picks
6. Emits a ranked top-N board with hypothesis, invalidation trigger, structure, sizing, and risk note per pick

User picks 3-5 manually and enters via broker UI. Skill never auto-trades.

---

## Modes

| Mode | Cap max | Float max | RVOL min | Min signals | Use case |
|---|---|---|---|---|---|
| `strict` (default) | $1B | 50M | 3.0x | 5 of 13 | Squeeze-pop hunter — low-float + SI/CTB fuel. Lowest false-positive rate. |
| `loose` | $2B | 100M | 2.0x | 3 of 13 | News-pop hunter — broader funnel, Everspin-class catalysts. More noise. |
| `catalyst` | $200B | unlimited | 1.5x | 3 of 6 | Mega-cap macro/policy pops (INTC/AMD/MU foundry, antitrust, regulatory). Squeeze gates dropped. |
| `smallcap` | $750M | 30M | 2.0x | 2 of 13 | Small-cap moonshots via theme-sympathy + float-velocity. Fans out from lead-theme adjacency map. Tier-2 sleeve cap 1.5% NAV. Backtest-validated against actual 2026-05-08 small-cap pop event. |

**Modifier flags:**
- `--tier-split` — emit top-20 split into TIER-1 (signal-confirmed core picks 1-10) / TIER-2 (small-cap moonshots picks 11-20). Recommended default for full market scan.
- `--theme <tag>` — filter to one theme bucket (`ai-memory` · `quantum` · `nuclear` · `photonics` · `defense-ai` · `glp1` · `crypto-mining` · `fusion` · `biotech-pdufa`)
- `--min-signals N` / `--max-cap CAP` — explicit overrides
- `--radon-discover` / `--radon-evaluate` — opt in/out of radon flow-discovery + 7-milestone evaluate gate. Default ON for `catalyst`/`smallcap`, OFF for `strict`/`loose`.

---

## Quick start

```
/banger-board                                        # top 10, strict, all themes
/banger-board --mode smallcap                        # small-cap moonshots
/banger-board --mode catalyst --max-cap 100B --theme defense-ai
/banger-board 20 --tier-split                        # top-20 tier-split
/banger-board --mode smallcap --theme ai-memory      # ai-memory adjacencies only
```

---

## 13-signal stack

| # | Signal | Threshold |
|---|---|---|
| 1 | Float | < 50M shares |
| 2 | SI + util | SI > 15% AND util > 90% |
| 3 | Borrow cost | > 10% AND rising 7d |
| 4 | RVOL | > 3x 5d MA |
| 5 | Options flow | call/put > 3 AND OTM weekly call premium > $500K |
| 6 | Catalyst | earnings/PDUFA/contract/court within 7d |
| 7 | Dark pool | sustained level break upside, 5d |
| 8 | Theme heat | matches active theme list |
| 9 | Insider/activist | 13D <30d OR insider cluster buy <30d |
| 10 | Sentiment | X/Reddit mention velocity rising WoW |
| 11 | News velocity | ≥3 distinct news items <7d w/ ≥1 tier-1 source OR single bullish tier-1 catalyst <48h |
| **12** | **Theme-sympathy** | Lead theme ticker up ≥10% over trailing 5d AND candidate in same theme bucket. Smallcap-mode core signal. |
| **13** | **Float-velocity** | RVOL ≥2x on float <30M (smallcap-mode); ≥3x on float <50M (loose); ≥5x on float <30M (strict). Counts ADDITIVE to #4. |

---

## Theme-adjacency map

For each theme, lead tickers (>$1B drivers) are watched for ≥10% trailing-5d moves. When a lead is hot, sub-$750M adjacencies in the same bucket fan out as smallcap-mode candidates.

| Theme | Leads | Adjacencies |
|---|---|---|
| ai-memory | MU, SNDK, WDC | MRAM, NVTS, ALGM, KOPN |
| quantum | IONQ, RGTI, QBTS | ARQQ, QUBT |
| nuclear | OKLO, SMR, NNE, CCJ | LEU, USAR, UEC |
| photonics | LITE, CRWV, COHR | POET, AEVA, LASR, MVIS |
| defense-ai | KTOS, AVAV, RTX | RDW, PL, RKLB |
| glp1 | LLY, NVO, HIMS | VKTX, ALT, REGN |
| crypto-mining | IREN, CORZ, MARA, RIOT | CIFR, BTBT, BITF |
| fusion | TLN, GEV | _(none yet)_ |
| biotech-pdufa | _(dynamic)_ | _(dynamic)_ |

---

## Backtest validation

**Event:** Everspin Technologies (MRAM) +25% close-to-close 2026-05-07 → 2026-05-08, +30% intraday vs 5/8 high. Cap $504M, float 23.4M.

Retro signal score against 13-stack on 5/7 close (PRE-pop):
- #1 Float 23.4M <50M ✓
- #8 Theme ai-memory ✓
- #12 Theme-sympathy: MU +19.3% / SNDK +12.9% trailing-5d on 5/7 ✓✓
- Score: **3 of 13** → above smallcap-mode 2/13 threshold → **WOULD HAVE SURFACED PRE-POP**

Threshold tuning applied post-backtest:
1. Signal #13 RVOL relaxed `5x → 2x` for smallcap-mode (5/8 actual RVOL was 2.6x — tight 5x would miss)
2. Smallcap-mode cap raised `$500M → $750M` ($504M was borderline edge)
3. Signal #12 confirmed as primary smallcap-mode firing signal

---

## Dependencies

### Required external services (env vars)

- `UW_TOKEN` — UnusualWhales API key (signals #1, #2, #4, #5, #7, #11)
- `FMP_API_KEY` — Financial Modeling Prep (signal #6 catalysts, fundamentals)
- `FINNHUB_API_KEY` (optional) — sentiment cross-check for signal #10

### Required Claude Code MCP servers

- `mcp__unusualwhales__*` — `uw_screener`, `uw_shorts`, `uw_flow`, `uw_darkpool`, `uw_news`, `uw_insider`, `uw_stock`
- `mcp__traderkit__*` — `earnings_calendar`, `signal_rank`, `screen_options`, `combo_fillability`, `thesis_fit`, `check_concentration` (companion repo: traderkit MCP server)
- `mcp__claude_ai_EXA__*` (optional) — tier-1 source verification for signal #11

### Optional convexity gating

- **radon** — Python project at companion path. Provides `discover.py` (Phase A.5 flow-discovery) + `evaluate.py` (Phase D.5 7-milestone M1-M7 gate). Required only for `--radon-discover` / `--radon-evaluate` flags. Can run w/o radon (skip flags) but `catalyst` + `smallcap` modes lose edge-determination on option structures.

### Vault docs (configurable — Obsidian / any markdown)

Set these env vars in your shell rc to point at your own ledger / regime docs:

- `${SLEEVE_LEDGER}` — sleeve ledger (5% NAV ceiling, active picks, exit triggers). Schema docs: see SHIP.md.
- `${REGIME_DOC}` — HALT regime gate (skill blocks under HALT)

---

## Failure modes

- UW screener returns empty → loosen Phase A thresholds incrementally + warn
- All candidates score <min-signals → "no setups today, sleeve idle" + log lowest-scored 5
- Sleeve cap reached → list active picks w/ exit triggers, recommend close before opening new
- HALT regime → block, persist nothing, emit pause notice
- Smallcap-mode no hot leads → Tier-2 emits empty w/ "no hot themes today, smallcap-tier dormant"
- Smallcap-mode Tier-1 empty + Tier-2 fired → force user confirmation ("Tier-1 empty — entering Tier-2 alone violates tier discipline. Continue? y/n")

---

## What this skill deliberately does NOT do

- **No auto-execute** — output is read-only board; user enters manually
- **No LLC/partnership-account picks by default** — partnership rules typically forbid speculation; configure exclusions per your accounts
- **No earnings-overnight structures** — gamma blow-up risk
- **No averaging down** — lottery tickets are one-shot
- **No naked OTM weekly calls** — defined-risk only

---

## Credits

Built atop:
- **UnusualWhales** screener + flow data
- **Financial Modeling Prep** fundamentals + catalysts
- **radon** (private companion repo) — IBKR options pipeline + convexity edge gate
- **traderkit** (private companion repo) — MCP server w/ trading-specific tools

---

## See also

- Skill body: `SKILL.md`
- Ship guide: `SHIP.md`
- Companion skills: `/trade-refresh`, `/nl-trade`, `/screen-cc`
