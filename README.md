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

## Probability calibration (Phase D.8 v1.1) — upside

After signals are stacked, banger-board emits a **conviction score (0-100)** per candidate, mapped through a **cap-aware logistic** to five anchor probabilities + an expected-pop magnitude.

**Five anchor thresholds** (deliberate ceiling — see anti-muddiness budget): `≥5%` (noise floor) · `≥10%` (any meaningful) · `≥20%` (mid-banger) · `≥30%` (banger target) · `≥50%` (extreme tail).

**Three cap tiers:** `small` ($200M-$1B) / `mid` ($1B-$50B) / `mega` (≥$50B). Small-caps hit ≥20% weekly at ~3x mega-cap base rate.

**E[pop] derivation:** piecewise mass-weighted midpoint integration over the 5 anchor buckets (midpoints 2.5/7.5/15/25/40/65%). Output is "expected max-of-week % move (intra-week high vs run-close)".

**Anti-muddiness budget (HARD design constraint):** 5 anchors max, 3 tiers max, no per-mode fits, cohort refit at n≥30, no hand-tuning between refits. Adding `≥15%` / `≥25%` / per-theme calibration = false precision the v1 logistic cannot support.

**Refit governance:** cohort gate n≥30 + per-tier n≥10 + Brier improvement ≥0.02 + monotone constraint + LLM council adversarial review. Bumps `v1.x → v2.0` atomically. Calibration tracker is append-only at `data/cache/banger-board-calibration.jsonl` (5 boolean outcome hits filled by t+7 retro-update job).

**MRAM-PROFILE flag** (NEW v1.1): structural pre-pop matcher. Any candidate satisfying ALL of:
1. Cap $200M-$1B
2. `pct_7d < 5%` (truly pre-pop)
3. Float < 30M shares
4. Lead theme ticker popped ≥10% in last 5d
5. Candidate is in same theme bucket via `theme_adjacency_map`

→ tagged `[MRAM-PROFILE]` and treated as **HIGH conviction by default** (override conv-score gate). Designed post-Everspin 5/8 +67% blow-off to catch the next analog before the move.

---

## Drawdown calibration (Phase D.9 v0.1) — downside

Mirror logistic for predicting probability of max-low drawdown over 7d at five negative anchors. **Critical separation:** drawdown drivers ≠ pop drivers, so a separate `drawdown_score` (0-100) is computed from structurally different components.

**Five anchor thresholds:** `≤−5%` · `≤−10%` · `≤−20%` · `≤−30%` · `≤−50%`. Same anti-muddiness budget as upside.

**Drawdown score components (0-100 total):**

| # | Component | Range | Trigger |
|---|---|---|---|
| 1 | already_popped_magnitude | 0-30 | Bucket B `pct_5d`: 0-10%=0, 10-20%=10, 20-30%=20, 30%+=30. Mean-reversion gravity ↑ w/ pop size. |
| 2 | iv_crush_risk | 0-20 | IVR<60=0, 60-80=5, 80-95=15, 95-100+ER<7d=20. Captures post-ER vol unwind (-10 to -25% typical). |
| 3 | theme_rotation_risk | 0-15 | cluster≥3 + leads already +20%/5d=15; cluster=2=8; cluster=1=0. Late-stage rotation. |
| 4 | cap_fragility | 0-15 | small+float<30M=15; small+float<60M=10; mid=5; mega=0. Halt/dilution/delisting tail. |
| 5 | rvol_exhaustion | 0-20 | RVOL≥3x sustained 3d+ AND pct_5d≥30%=20 (blow-off); RVOL≥3x 1-2d=10; RVOL 2-3x=5. |

**Combined EV (sleeve action gate):**

```
e_net_pct_7d = e_pop_pct_7d + e_drop_pct_7d
```

| E[net] | Action |
|---|---|
| ≥+15% | **FULL-SIZE** — banger-native; both tails favor upside |
| +5% to +15% | **MODE-TIER** — small +EV, mode-default sizing |
| −5% to +5% | **FLAT-EV** — wrong sleeve, banger structure not warranted |
| <−5% | **NEGATIVE-EV** — chase trap; skip entry |

**Regime sensitivity:** v0.1 anchors are calibrated at BULL regime baseline (current CRI 3.7). Drawdown is more regime-sensitive than upside — bear-market drawdown rates run 2-3x bull baseline. Refit gate flags `[REGIME-SHIFT]` if the regime tier changes mid-cohort.

**MRAM 5/8 validation:** drawdown_score=80 → E[net]=−29.6%, P(≤−30%)=73%. Quantitatively confirms the chase-trap AVOID call. Post-blow-off entry is a structural sink, not a probabilistic edge.

Calibration tracker: `data/cache/banger-board-downside-calibration.jsonl` (append-only, 5 boolean drop hits + `outcome_min_pct_7d` filled by t+7 retro-update job).

---

## Local visualization dashboard

A self-contained HTML dashboard reads the calibration JSONL files + emits a filterable, sortable table view w/ tooltips + per-row component breakdown. No build step, no framework — vanilla JS + native HTML.

**Build + open:**
```bash
python3 dashboard/banger-board/build.py
open dashboard/banger-board/index.html
```

**Features:**
- Summary cards: total candidates, FULL-SIZE count, NEGATIVE-EV count, avg E[net]
- Filters: ticker search, cap tier, EV tier, min E[net]
- Sort by any column (default: E[net] descending)
- Click a row to expand: shows 5-anchor probability ladders for both upside + downside, full component breakdown w/ per-component progress bars, and any calibration notes / preliminary outcomes
- Tooltips on every column header explaining the metric + thresholds
- Color coding: green for +EV tiers, red for NEGATIVE-EV / AVOID, neutral for FLAT

Refresh after each `/banger-board` run to pull the latest cache.

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
