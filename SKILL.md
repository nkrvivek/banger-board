---
name: banger-board
description: Speculative pop-hunter — surfaces ranked board of high-variance 5-7d candidates expected to pop 30%+, stacking 15 signals (float/SI/flow/catalyst/theme/sympathy/velocity/IVR/premium-tier). Ad-hoc + daily-cron setup mode, no auto-execute. Configurable to a single speculative-sleeve account; LLC/partnership accounts excluded by default. Triggered by /banger-board [N=10|20] [--theme <tag>] [--mode strict|loose|catalyst|smallcap|setup] [--min-signals N] [--tier-split].
---

# /banger-board — Speculative Pop Hunter

**Purpose.** Hunt 30%+ weekly pops by stacking 15 stackable signals (float · short-interest · options-flow · catalyst · theme · dark-pool · sentiment · borrow-cost · RVOL · activist/insider · news-velocity · theme-sympathy · float-velocity · IVR pre-position · premium-tier). Outputs a ranked board with thesis-fit rationale, suggested structure, sizing, and exit triggers. **No auto-execute** — user picks 3-5 manually for entry via `/nl-trade` or broker UI.

Binding: per `${SLEEVE_LEDGER}` (your sleeve thesis doc — see Configuration). 5% NAV ceiling. Speculative-sleeve account only (LLC/partnership accounts excluded). NO HALT regime. NO earnings-overnight holds. NO naked OTM weeklies.

---

## Configuration

This skill expects the following environment variables + paths configured in your shell rc (`~/.zshrc` / `~/.bashrc`):

```bash
# --- Required env (UnusualWhales + FMP API access) ---
export UW_TOKEN=<your-unusualwhales-api-token>      # https://unusualwhales.com/api
export FMP_API_KEY=<your-fmp-key>                   # https://financialmodelingprep.com/

# --- Optional env (sentiment cross-check) ---
export FINNHUB_API_KEY=<your-finnhub-key>           # signal #10 (sentiment); EXA fallback used if absent

# --- Required paths (companion repos) ---
export RADON_PATH=/path/to/your/radon               # IBKR-direct flow-discover + 7-milestone evaluate gate
export TRADE_REFRESH_PATH=/path/to/your/trade-refresh   # backtest harness

# --- Required vault docs (Obsidian or any markdown) ---
export SLEEVE_LEDGER=/path/to/your/sleeve-ledger.md     # 5% NAV ceiling, active picks, exit rules
export REGIME_DOC=/path/to/your/market-regime.md        # HALT regime gate (skill blocks under HALT)
```

If you don't have radon / traderkit companion repos: skip with `--no-radon-discover` / `--no-radon-evaluate` flags. The core 13-signal stack still functions (loses the convexity-gating edge but baseline scoring intact).

MCP servers required (configure in `~/.claude.json` under `mcpServers`):
- `unusualwhales` — `mcp__unusualwhales__uw_*` family
- `traderkit` — `mcp__traderkit__signal_rank`, `screen_options`, `earnings_calendar`, `combo_fillability`, `thesis_fit`, `check_concentration`
- `exa` (optional) — tier-1 source verification for signal #11

---

## Usage

```
/banger-board [N=10|20] [--theme <tag>] [--mode strict|loose|catalyst|smallcap] [--min-signals N] [--max-cap CAP] [--tier-split]
```

Args:
- `N` (default 10, max 20) — board size. With `--tier-split` AND `N=20`, output is split Tier-1 (core 1-10) + Tier-2 (small-cap moonshots 11-20).
- `--theme` — filter to one theme (`ai-memory`, `quantum`, `nuclear`, `glp1`, `defense`, `fusion`, `space`, `crypto-mining`, `photonics`, `biotech-pdufa`)
- `--mode` — one of five pop archetypes (defaults `strict`):
  - `strict` — squeeze-pop hunter (low-float + SI/CTB fuel + RVOL). Lowest false-positive rate.
  - `loose` — news-pop hunter (small/mid-cap catalysts). Broader funnel, more noise.
  - `catalyst` — macro/policy-pop hunter (INTC-class mega-cap turnarounds, foundry/AI/regulatory catalysts). Drops squeeze requirements entirely. **Now includes catalyst-tier theme fan-out (5/8) — sympathy-mega-caps surface even w/o own IVR≥80.**
  - `smallcap` — **small-cap moonshot hunter**. Tiny-cap (<$750M), tiny-float (<30M), low signal bar (2/15). Fans out via theme-adjacency map from a lead-theme name. Highest variance — sub-$1K per pick, max 5 active in tier.
  - `setup` — **pre-pop watchlist (NEW 5/8)**. Surfaces IVR-pre-positioned + premium-tier candidates BEFORE the move. Read-only — no entry recommendation, no sizing. Designed for daily cron (`/loop 1d /banger-board --mode setup`). Captures INTC/MU-class names 3-7d before the pop.
- `--min-signals` — explicit override (overrides mode default)
- `--max-cap` — market-cap ceiling (overrides mode default)
- `--tier-split` — emit 20-pick board split Tier-1 (signal-confirmed core) + Tier-2 (small-cap sympathy/moonshots). Implies `N=20`.
- `--no-radon-discover` — disable radon `discover.py` primary funnel (fallback to UW-only). NOT recommended — UW `short_screener` retired 2026-05-08 (stale 2021 cache, args ignored). Without radon, board falls back to UW per-ticker enrichment over a manual seed list.
- `--no-radon-evaluate` — skip radon `evaluate.py` 7-milestone gate at Phase D.5 (default ON for `catalyst` mode b/c M4 edge critical for option mispricing).

### Mode parameter table

| Mode | cap max | float max | SI min | RVOL min | min signals | scope |
|---|---|---|---|---|---|---|
| `strict` (default) | $1B | 50M | 15% | 3.0x | 5 of 15 | any signal |
| `loose` | $2B | 100M | 10% | 2.0x | 3 of 15 | any signal |
| `catalyst` | **$1T** (raised 5/8) | unlimited | — | 1.5x | 3 of 8 | from {#5 flow, #6 catalyst, #7 darkpool, #8 theme, #9 insider/13D, #11 news, **#14 IVR pre-position**, **#15 premium-tier**} |
| `smallcap` | **$750M** | **30M** | — | 2.0x (any RVOL counts via #13 float-velocity, threshold relaxed post-backtest) | **2 of 15** | from {#8 theme, #11 news, #12 theme-sympathy, #13 float-velocity} — squeeze gates DROPPED |
| `setup` (NEW 5/8) | $1T | unlimited | — | — | **2 of 3** | from {**#14 IVR pre-position**, **#15 premium-tier**, #5 options flow} — read-only watchlist, no entry sizing |

Catalyst mode rationale: mega-caps (INTC, AMD, MU, RKLB, BABA) can pop 30%+ on policy / foundry / antitrust / regulatory / capex news without ever showing squeeze fuel. The catalyst-only signal subset is more selective for macro pops and ignores the low-float / borrow-cost gates entirely.

**Catalyst cap raise 2026-05-08 ($200B → $1T):** post-mortem of a single-day mega-cap pop event showed every largest pop sat in the $230B-$842B band (INTC +14% @ $628B / MU +15% @ $842B / AMD +11% @ $742B / QCOM +8% @ $231B / SNDK +17% @ $231B). All blocked by prior $200B cap. Subset gating is selective enough on its own. RKLB +34% ($61B, space-launch) is the canonical mid-mega-cap catalyst-mode catch alongside INTC.

**MRAM mode rationale** (NEW 2026-05-08): small-cap pops (Everspin Technologies 5/8 +30% in week, sub-$300M micro-caps generally) are theme-sympathy + float-velocity driven, NOT squeeze-driven. They have:
- Float so tiny (<30M) that even modest flow creates 5x+ RVOL (so RVOL gate is redundant — captured by signal #13)
- Often ZERO direct news; pop b/c sector lead (MU/SNDK/IONQ/OKLO) rallied hard and bots/retail fan out to adjacencies
- Thin option chains → invisible to radon `discover.py` flow funnel → MUST be seeded via theme-adjacency map (see "Theme-adjacency map" below), NOT discovered via flow
- Can satisfy as few as 2 signals (theme + sympathy) and still pop. Anything higher gates them out.

Smallcap mode does NOT replace strict/loose — it's an **additive tier** invoked alongside via `--tier-split` to populate Tier-2 (picks 11-20).

### Per-position sizing by mode

| Mode | Per-pick $ | Max active in tier | Rationale |
|---|---|---|---|
| `strict` | $1-2K | 5 | Squeeze-pop variance — lottery math, expect 5+ wipeouts per winner |
| `loose` | $1-2K | 5 | News-pop variance still high — same lottery math |
| `catalyst` | $2-5K | 3 | Mega-cap variance lower per dollar — bigger size justified, but ceiling lower b/c upside also smaller (15-30% vs 30-100% for squeezes) |
| `smallcap` | **$500-1K** | 5 | Tiny-cap moonshot — highest variance of all tiers (extreme gap risk + delisting tail). Sub-$1K caps the loss when (not if) one delists / halts / dilutes. Tier-2 sleeve cap = 1.5% NAV (vs 5% sleeve total), to ensure Tier-1 core stays primary. |

Sleeve total stays **5% NAV** regardless of mode.

Examples:
- `/banger-board` — top 10, strict, all themes
- `/banger-board 15 --theme ai-memory` — 15 picks, AI memory theme only
- `/banger-board --mode loose --min-signals 3` — small/mid-cap news-pop funnel
- `/banger-board --mode catalyst --max-cap 100B --theme defense` — mega-cap catalyst hunt (INTC/RTX-class)
- `/banger-board --mode catalyst --theme ai-memory` — INTC/AMD/MU foundry-policy + AI capex pops
- `/banger-board --mode smallcap --theme ai-memory` — small-cap moonshots only (sub-$750M ai-memory adjacencies)
- `/banger-board --tier-split` — top-20 split: Tier-1 strict/loose core (1-10) + Tier-2 small-cap moonshots (11-20). **Recommended default for full market scan.**
- `/banger-board --mode setup` — pre-pop watchlist (read-only, no sizing). Surfaces IVR≥80 OR rising-IVR-trend candidates BEFORE the move.
- `/loop 1d /banger-board --mode setup` — daily cron for setup watchlist (catches mega-cap setups 3-7d before pop).

---

## Pre-flight gates (BLOCK if fail)

1. **Sleeve cap** — read `${SLEEVE_LEDGER}` ledger. If active picks total > 5% NAV → emit `[BLOCKED] sleeve cap reached`, list current picks + their exits.
2. **Regime** — read `${REGIME_DOC}`. If tier == HALT → emit `[BLOCKED] HALT regime, sleeve paused`. CAUTION → halve all suggested sizes + flag.
3. **Required env** — `UW_TOKEN` (UnusualWhales, used in Phase B enrichment), `FMP_API_KEY` (catalysts), `FINNHUB_API_KEY` (optional, sentiment). Fail loud if missing.
4. **Radon ready** — `mcp__radon__ib_check_connection` ready=true (Phase A primary funnel needs IB Gateway). If down → fall back to `--no-radon-discover` mode (degraded).

---

## Data-freshness rules (HARD — 2026-05-08 retirement)

**Retired endpoints — do NOT use:**
- `mcp__unusualwhales__uw_screener` w/ `short_screener` filter — returns stale 2021-10-15 records, filter args silently ignored. Surfaced AAPL @ 0.6% SI despite SI≥15% gate (smoke test 2026-05-08). **Cause of pre-2026-05-08 banger-board funnel breakage.**

**Verified-fresh UW endpoints (sanity-check timestamp before scoring):**
- `mcp__unusualwhales__uw_darkpool` ✓ current (2026-05-08 timestamps verified)
- `mcp__unusualwhales__uw_flow` ✓ current
- `mcp__unusualwhales__uw_news` ✓ current
- `mcp__unusualwhales__uw_insider` ✓ current
- `mcp__unusualwhales__uw_alerts` ✓ current
- `mcp__unusualwhales__uw_shorts` (per-ticker, NOT screener variant) — verify timestamp before signal scoring; degrade signal #2/#3 if stale

**Sanity check rule:** every UW response → check max timestamp ≥ today−1 trading day. If stale → log `[STALE-UW]` warning + drop that signal contribution from the score (don't block, but don't credit).

---

## Pipeline

### Phase A — Universe screen (radon `discover.py` primary funnel)

**Primary candidate source:** radon flow + dark-pool confluence scanner. Replaces retired UW `short_screener`. Proven 2026-05-08 (SNDK score 61.7 BULLISH+ACCUMULATION+CONFLUENCE 9 alerts $10.72M premium 2d sustained — ahead of +16.2% intraday).

```bash
cd "${RADON_PATH}" && .venv/bin/python3 scripts/discover.py \
  --min-premium {500000 if mode==catalyst else 250000 if mode==loose else 250000} \
  --min-alerts {3 if mode==catalyst else 2} \
  --dp-days 3 \
  --top {40 if mode==catalyst else 30 if mode==loose else 20}
```

Returns ranked `[{ticker, score (0-100), dp_direction, options_bias, confluence, alert_count, dp_premium, sustained_days, ...}]` — radon's edge score on options-flow + dark-pool confluence (not dollar size). Cache 4h at `data/cache/banger-board-radon-discover-{mode}.json`.

**Mode-driven primary-funnel params:**

| Mode | --min-premium | --min-alerts | --dp-days | --top |
|---|---|---|---|---|
| `strict` | $250K | 2 | 3 | 20 |
| `loose` | $250K | 2 | 3 | 30 |
| `catalyst` | $500K | 3 | 3 | 40 |

**Mode caps applied AFTER radon surfaces** (Phase B post-discovery filter, not Phase A primary):

```python
MODE_CAPS = {
    "strict":   {"market_cap_max": 1_000_000_000,    "float_max": 50_000_000,  "rvol_min": 3.0},
    "loose":    {"market_cap_max": 2_000_000_000,    "float_max": 100_000_000, "rvol_min": 2.0},
    "catalyst": {"market_cap_max": 1_000_000_000_000, "float_max": None,        "rvol_min": 1.5},  # raised 5/8: $200B → $1T (subset gating selective enough)
}
# Applied per-ticker after Phase A radon surface, using uw_stock + uw_shorts (per-ticker, fresh).
```

**Catalyst-mode pre-filter** (additional, runs after radon surface): drop any candidate w/o at least one of {next_earnings_date <14d, recent 13D filing <30d, recent insider cluster <30d, news count <7d ≥2}. Avoids surfacing mega-caps w/o any catalyst story — INTC w/o foundry news today is just IBM, not a banger.

**Skip / fallback conditions**:
- IB Gateway down or radon `discover.py` errors → emit `[DEGRADED:radon-down]`, fall back to **UW-only Phase B-driven funnel**: seed candidate list from prior cache + thesis-aligned manual seed list (curated per theme tag), run Phase B per-ticker enrichment over seed.
- `UW_TOKEN` missing → block (Phase B enrichment requires UW); emit `[BLOCKED:no-uw-token]`.

### Phase A.6 — Theme-adjacency fan-out (NEW 2026-05-08, small-cap + catalyst capture)

**When triggered**: `--mode smallcap` OR `--mode catalyst` OR `--tier-split` flag set. Skipped for pure `strict`/`loose` runs.

**Logic**:
1. From Phase A radon-surfaced candidates + verified-fresh `uw_stock` 5d % change, identify any `theme_adjacency_map` lead w/ ≥10% trailing-5d move.
2. For each hot lead → fan out: add ALL adjacency tickers for that theme to the candidate set (smallcap uses `adjacencies[]`, catalyst uses `peers[]` — see map below).
3. Tag each fan-out candidate w/ `source: "theme-adjacency:{lead}"` (audit trail) and `signal_12_eligible: true` (theme-sympathy candidate, gets +1 if Phase B confirms).
4. **Mode-specific cap gating**:
   - smallcap: cap <$750M AND float <30M (tiny moonshots only)
   - catalyst: cap ≤$1T AND lead theme is hot — captures sympathy-mega-caps where own IVR<80 (e.g. AMD pulled by MU/SNDK rally even when AMD's own pre-pop IVR was 70)
5. Cap raised from $500M → $750M post-backtest 2026-05-08 (MRAM ticker at $504M was borderline edge-case; $750M ceiling captures full small-cap range w/o leaking into mid-cap).

```python
# Pseudocode (mode-aware)
hot_leads = [t for t in all_lead_tickers
             if uw_stock(t).pct_change_5d >= 10.0]
peers_to_add = set()
for lead in hot_leads:
    theme = reverse_lookup_theme(lead)
    if mode in ("smallcap",) or tier_split:
        for adj in theme_adjacency_map[theme]["adjacencies"]:
            if uw_stock(adj).market_cap < 750_000_000 \
               and uw_stock(adj).float < 30_000_000:
                peers_to_add.add((adj, theme, lead))
    if mode == "catalyst":
        for peer in theme_adjacency_map[theme].get("peers", []):
            if uw_stock(peer).market_cap <= 1_000_000_000_000:
                peers_to_add.add((peer, theme, lead))
candidates += [{**c, "source": f"theme-adjacency:{lead}", ...}
               for (c, theme, lead) in peers_to_add]
```

**Why this matters**: small-cap names rarely show up in radon flow-discover (option chains too thin). W/o fan-out they're invisible to the funnel. With fan-out, when the lead theme is hot the adjacencies surface automatically. This is the structural fix for the "we missed Everspin pop" failure mode.

**Cost**: ~10 extra `uw_stock` calls per run (one per adjacency in active theme). Negligible vs total UW budget.

### Phase B — Layer signals (UW per-ticker enrichment, verified-fresh only)

For each Phase-A candidate (radon-surfaced or fallback seed), parallel-fetch from verified-fresh UW endpoints + traderkit + EXA:

```
# verified-fresh UW (per-ticker, not screener variant):
mcp__unusualwhales__uw_shorts(ticker)         # SI%, utilization, cost-to-borrow, days-to-cover -- VERIFY timestamp
mcp__unusualwhales__uw_flow(ticker)           # ✓ fresh — OTM call premium, sweep flag, call/put ratio
mcp__unusualwhales__uw_darkpool(ticker)       # ✓ fresh — 5d dark-pool prints + level breaks
mcp__unusualwhales__uw_insider(ticker)        # ✓ fresh — insider cluster <30d
mcp__unusualwhales__uw_news(ticker, days=7)   # ✓ fresh — signal #11: news velocity + tier-1 catalyst
mcp__unusualwhales__uw_alerts(ticker)         # ✓ fresh — UW-tagged event/anomaly alerts (recent)
mcp__unusualwhales__uw_stock(ticker)          # ✓ fresh — RVOL, market cap, float, price (mode caps)

# Catalyst + cross-source:
mcp__traderkit__earnings_calendar(ticker, days=7)
mcp__claude_ai_EXA__company_research_exa(ticker)  # signal #11 cross-check: tier-1 source verification
```

**Apply mode caps** (`MODE_CAPS[mode]`) using `uw_stock` (cap, float, RVOL) + `uw_shorts` (SI%) → drop candidate if outside cap/float/RVOL gate (catalyst skips float gate).

**`uw_shorts` freshness gate:** if response timestamp > 7 days stale → log `[STALE-UW:uw_shorts:{ticker}]`, drop signals #2 + #3 from score (don't credit). Do not silently use 2021-vintage cached data.

Score each ticker against the 15-signal stack (1 point per signal matched):

| # | Signal | Threshold |
|---|---|---|
| 1 | Float | < 50M shares |
| 2 | SI + util | SI > 15% AND util > 90% — source: per-ticker `uw_shorts` (NOT `short_screener`); drop if stale |
| 3 | Borrow cost | > 10% AND rising 7d — source: per-ticker `uw_shorts`; drop if stale |
| 4 | RVOL | > 3x 5d MA — source: `uw_stock` (verify timestamp ≥ today−1) |
| 5 | Options flow | call/put > 3 AND OTM weekly call premium > $500K |
| 6 | Catalyst | earnings/PDUFA/contract/court within 7d |
| 7 | Dark pool | sustained level break upside, 5d |
| 8 | Theme heat | matches active theme list (see themes) |
| 9 | Insider/activist | 13D <30d OR insider cluster buy <30d |
| 10 | Sentiment | X/Reddit mention velocity rising week-over-week (use exa search if Finnhub sentiment unavailable) |
| 11 | News velocity | ≥3 distinct news items <7d w/ ≥1 tier-1 source (Reuters/Bloomberg/WSJ/company PR/sector tier-1) OR single bullish tier-1 catalyst <48h (contract · partnership · regulatory approval · major customer). **Squeeze-substitute** — catches news-pops where SI/CTB fuel absent (MRAM 5/8 case). |
| **12** | **Theme-sympathy** | Lead theme ticker (see Theme-adjacency map) up ≥10% over trailing 5d AND candidate is in same theme bucket. Captures sympathy-pop pattern where smaller name rallies on lead-name strength w/o direct catalyst. Source: `uw_stock` 5d % change on lead ticker + theme-bucket lookup. **smallcap-mode core signal.** |
| **13** | **Float-velocity** | RVOL ≥2x on float <30M shares = +1 (smallcap-mode, post-backtest relaxed); ≥3x on float <50M for `loose` mode; ≥5x on float <30M for `strict` mode. Tiny floats move violently on small flow — heavier-weighted than vanilla RVOL signal #4. Counts ADDITIVE to #4. **smallcap-mode core signal.** Backtest 2026-05-08: MRAM popped +25% close-to-close on RVOL=2.6x (vs 30d avg) — tighter ≥5x threshold would have missed the event b/c thin float dislocates on normal volume. |
| **14** | **IVR pre-position** | EITHER (a) `iv_rank ≥ 80` (absolute current reading) OR (b) `iv_rank_delta_7d ≥ +25` percentile points (rising-trend half — captures RKLB-class names where IVR was 55 but accelerating fast). Source: `uw_stock.iv_rank` + 7d historical comparison. **catalyst-mode core signal** — added 2026-05-08 post-mortem: INTC/MU/QCOM/SNDK all hit IVR=100 BEFORE 5/8 pops. Backtest: 6/6 IVR≥80 candidates popped same day. |
| **15** | **Premium-tier (NEW 5/8)** | `net_call_premium_5d ≥ $100M` from `uw_flow.flow_alerts` aggregated 5d. Weights mega-flow over alert-count alone — catches mega-cap accumulation (INTC/AMD/MU all >$100M net call premium pre-pop) that thin alert counts miss. **catalyst-mode + setup-mode core signal.** |

→ Drop candidates with score < `--min-signals` (default per mode: strict=5, loose=3, catalyst=3, **smallcap=2**, **setup=2**). Cache scores in `data/cache/banger-board-{mode}.json`.

**Catalyst-mode scoring divergence:** in `--mode catalyst`, only the catalyst-side subset {#5, #6, #7, #8, #9, #11, #14, #15} counts toward the threshold (3 of 8). Squeeze-side signals {#1 float, #2 SI+util, #3 CTB, #4 RVOL, #10 sentiment-velocity} are **logged but not counted** (still surfaced in the board output for context). Rationale: mega-caps physically cannot satisfy #1/#2/#3, so requiring them blocks legitimate INTC/AMD/MU/RKLB pops on AI/foundry/policy/space news. Signals #14 + #15 added 5/8 — captures setups where vol market priced movement first AND mega-flow accumulation (INTC/MU/QCOM all hit IVR=100 + >$100M net call premium BEFORE pops).

**Setup-mode scoring divergence (NEW 5/8):** in `--mode setup`, only the pre-position subset {#14 IVR pre-position, #15 premium-tier, #5 options flow} counts toward the threshold (2 of 3). All other signals logged for context but not counted. Read-only watchlist — no entry sizing, no structure recommendation. Designed for daily cron via `/loop 1d /banger-board --mode setup`. Promote a setup-mode pick to default mode for full scoring + entry plan IF the setup matures into multi-signal confluence.

### Phase C — Multi-source rank

```
mcp__traderkit__signal_rank(candidates, sources=["uw_flow", "uw_shorts", "uw_darkpool", "exa_sentiment"])
```

→ Confluence-boosted ranking. Top N kept.

**Stale-aware:** if Phase-B flagged `uw_shorts` stale for a ticker, drop `"uw_shorts"` from that ticker's source list before calling `signal_rank` (don't double-credit dropped Phase-B signals via the consensus tool).

### Phase D — LLM thesis-fit pass (per pick)

For each top-N candidate, the LLM (this agent) writes:
- **Catalyst hypothesis** (1 sentence) — what triggers the pop
- **Invalidation trigger** (1 sentence) — what kills the thesis
- **Suggested structure** — long stock · debit call vertical · long call (DTE ≥14)
- **Suggested size** — $1-2K based on liquidity (lower size for tighter spreads)
- **Theme tag** — which theme bucket
- **Risk note** — gamma exposure · earnings-overnight risk · pump-and-dump red flags

### Phase D.5 — Radon 7-milestone evaluate gate (optional, catalyst-mode default ON)

**When enabled**: gate each Phase-D pick through radon `evaluate.py` (M1-M7 milestones) before emit. Picks failing M4 edge are surfaced read-only w/ `[RADON-DROP:M4]` tag, not in main board ranking.

```bash
cd "${RADON_PATH}" && .venv/bin/python3 scripts/evaluate.py {ticker} \
  --bankroll 40000 \
  --json
```

Returns `{milestones: {M1..M7}, decision, edge_score, structure_proposed, kelly_size, ...}`.

**Milestone interpretation for banger-board**:
| Milestone | Banger-board use |
|---|---|
| M1 (ticker validation) | Sanity check — reject delistings, halts, illiquid OTC names |
| M1B (seasonality) | Bonus only — log if unfavorable seasonal pattern |
| M1C (analyst ratings) | Skip (banger-board explicitly anti-consensus — analyst upgrades = mean-reversion fade risk) |
| M1D (news & catalysts) | **Cross-check w/ signal #11** — if M1D = no catalyst but signal #11 fired → re-verify news source tier |
| M2 (dark pool flow) | Confirms signal #7 from Phase B — if conflict, drop confidence |
| M3 (options chain + flow) | Confirms signal #5 from Phase B + provides liquidity check for structure decision |
| M3B (OI change) | Bonus — strong rising OI = institutional positioning conviction |
| **M4 (edge determination)** | **CRITICAL for `catalyst` mode**. M4 = "is the option chain mispriced relative to expected move?" Catalyst-mode picks are ALL options-based ($2-5K can't buy 30+ shares of INTC); if M4 says no edge, the structure has negative EV regardless of pop probability. Drop pick. |
| M5 (structure proposal) | Use as structure recommendation (overrides Phase D LLM-suggested structure if M5 specifies different DTE/strike) |
| M6 (Kelly sizing) | Cap at mode-defined per-pick $ (Kelly may suggest $5K but catalyst-mode ceiling is $5K and strict/loose ceiling is $2K — never over-size from Kelly) |
| M7 (final decision) | If M7 = REJECT, drop to read-only `[RADON-DROP:M7]` annotation |

**Mode-specific value**:
- `strict` / `loose` — useful for stock-buy or debit-spread structure check, but most signals already covered by Phase B + signal_rank. Optional.
- `catalyst` — **CRITICAL**. All catalyst picks are option structures (debit call spread or LEAP). M4 edge determination is the difference btw a 30% pop turning into a +200% return on capital (good IV setup) vs a +15% return (bad IV — paid up for theta). Without M4 gate, catalyst-mode is leaking edge to options market makers.

**Skip conditions**: IBKR Gateway not running → emit `[SKIP:radon-evaluate-no-gateway]`, continue w/o gate. radon `evaluate.py` errors per-ticker → log warning, mark `[RADON-EVAL-ERR]`, retain pick at lower confidence rank.

### Phase E — Emit board

**Standard emit (single tier, N=10 default):**

```
Banger Board · {date} · regime: {tier} · sleeve cap: ${used}/${max}

#  Tkr   Cap    Float  SI%   CTB%  RVOL  Flow$    Score  Theme         Catalyst       Structure         Size    Hypothesis (1-line)
1  XXX   $400M  18M    32%   25%   5.2x  $1.2M    9/13   ai-memory     Earnings 5/14  long_stock        $2.0K   "AI-DRAM tape squeeze + earnings"
2  ...
...
N

⚠ Pre-trade reminder:
- 5% sleeve cap NAV personal-only · $1-2K per pick · max 5 active
- 7d hard close · -50% stop · NO earnings-overnight holds
- Manual entry via /nl-trade or broker UI — this skill does not auto-execute
- Update banger-board.md ledger when you enter / exit
```

**Tier-split emit (`--tier-split` or N=20):**

```
Banger Board · {date} · regime: {tier} · sleeve cap: ${used}/${max}

═══ TIER-1 — high-conviction core (signal-confirmed, $1-2K/pick) ═══
#  Tkr   Cap    Float  SI%   CTB%  RVOL  Flow$    Score  Theme         Catalyst       Structure         Size    Hypothesis
1  HIMS  $14B   219M   32%   —     2.1x  $0.8M    7/13   glp1          insider clust  long_stock        $2.0K   "GLP-1 demand + 32% SI squeeze fuel"
...
10 ...

═══ TIER-2 — small-cap moonshots (theme-sympathy, $500-1K/pick, max 5 active) ═══
#  Tkr   Cap    Float  Theme         Lead↑5d  Adjacency-via    Sig-fired    Structure   Size   Hypothesis
11 MRAM  $200M  18M    ai-memory     MU+12%   MU               #8,#11,#12   long_stock  $750   "AI-DRAM sympathy via MU rally"
12 ARQQ  $250M  22M    quantum       IONQ+15% IONQ             #8,#12,#13   long_stock  $750   "Quantum theme spillover, tiny float"
...
20 ...

⚠ Pre-trade reminder (TIER-2 specific):
- Tier-2 sleeve cap: 1.5% NAV (subset of 5% total) · max 5 active
- Tier-2 entries are sympathy plays — IF lead-theme ticker reverses ≥5% in 1d → liquidate Tier-2 same-day
- Tier-2 names often have wider spreads / overnight gap risk → use limit orders, never market
- All standard rules still apply (7d hard close, -50% stop, no earnings overnight)

⚠ Tier discipline:
- Fill TIER-1 first. Only proceed to TIER-2 after TIER-1 is sized + entered.
- DO NOT skip TIER-1 to load up TIER-2 — moonshot allocations come from REMAINING sleeve, not primary.
- If sleeve constrained → TIER-1 only; skip TIER-2 entirely.
```

**Setup-mode emit (`--mode setup`, NEW 5/8):**

```
Banger Board · SETUP WATCHLIST · {date} · regime: {tier}
Pre-positioned candidates (IVR ≥80 OR rising-trend ≥+25 in 7d, premium-tier ≥$100M, or recent flow alert)

#  Tkr   Cap     IVR   IVR Δ7d   Net Call 5d   RVOL   ER Date   Setup score   Earliest pop window
1  XXXX  $250B   100   +35       $185M         1.8x   05-22     3/3           5-8 days
2  YYYY  $61B    78    +28       $112M         3.1x   none      2/3           3-5 days
...

⚠ Watchlist-only:
- NO entry recommendation. NO sizing. NO structure suggestion.
- Promote a name to /banger-board (default mode) for full scoring + entry plan IF setup matures.
- Demote / remove if IVR collapses ≥20pts WITHOUT price move → false positive.
- Designed for `/loop 1d` cron — re-runs daily, persists watchlist as `data/cache/banger-board-setup-{YYYY-MM-DD}.json`.
```

---

## Themes (sector heat tags)

Maintained in this skill, refreshed monthly:

| Tag | Description | Recent winners (rolling 90d) |
|---|---|---|
| `ai-memory` | DRAM/HBM/MRAM/specialty memory plays | MRAM, MU, WDC, SNDK |
| `quantum` | Quantum computing pure-plays | IONQ, RGTI, QBTS |
| `nuclear` | SMR + uranium + fusion | OKLO, SMR, NNE |
| `glp1` | GLP-1 weight-loss + adjacent | LLY, NVO, VKTX, HIMS |
| `defense` | Defense AI + drones | KTOS, AVAV, RKLB |
| `fusion` | Fusion energy moonshots | TLN, GEV |
| `space` | Space launch + sat | RKLB, ASTS, IRDM |
| `crypto-mining` | Bitcoin miner converts to AI | IREN, CIFR, CORZ |
| `photonics` | AI photonics + optical interconnect | LITE, CRWV, POET |
| `biotech-pdufa` | FDA decision-driven micro-caps | _rotates frequently_ |

Theme list reviewed monthly — sectors that produced no pops 60d+ get demoted; new heat sectors added based on UW sector-flow data.

---

## Theme-adjacency map (NEW 2026-05-08 — small-cap fan-out)

For each theme, the **lead ticker** is the >$1B name whose 5d % move drives signal #12 (theme-sympathy). The **adjacencies** are sub-$500M small-cap names that pop in sympathy. When smallcap-mode runs OR `--tier-split` is active, the universe fans out from leads to adjacencies:

```yaml
theme_adjacency_map:
  ai-memory:
    leads:        [MU, SNDK, WDC]                # >$1B drivers
    adjacencies:  [MRAM, NVTS, ALGM, KOPN]       # sub-$750M small-cap (smallcap mode)
    peers:        [INTC, AMD, QCOM, AVGO, ARM]   # mid/mega-cap fan-out (catalyst mode, NEW 5/8)
  quantum:
    leads:        [IONQ, RGTI, QBTS]
    adjacencies:  [ARQQ, QUBT]
    peers:        [IBM, GOOGL]                   # quantum-research mega-caps
  nuclear:
    leads:        [OKLO, SMR, NNE, CCJ]
    adjacencies:  [LEU, USAR, UEC]
    peers:        [VST, CEG, PWR]                # utility/grid mega-caps tied to SMR demand
  photonics:
    leads:        [LITE, CRWV, COHR]
    adjacencies:  [POET, AEVA, LASR, MVIS]
    peers:        [NVDA, AVGO]                   # photonics customers / interconnect plays
  defense-ai:
    leads:        [KTOS, AVAV, RTX]
    adjacencies:  [RDW, PL, RKLB]                # space-defense overlap
    peers:        [LMT, NOC, GD, BA]             # defense-prime mega-caps
  glp1:
    leads:        [LLY, NVO, HIMS]
    adjacencies:  [VKTX, ALT, REGN]
    peers:        [PFE, MRK, AMGN]               # pharma mega-cap fan-out
  crypto-mining:
    leads:        [IREN, CORZ, MARA, RIOT]
    adjacencies:  [CIFR, BTBT, BITF]
    peers:        [COIN, MSTR]                   # crypto-exposure mega-caps
  fusion:
    leads:        [TLN, GEV]
    adjacencies:  []                             # no sub-$750M fusion plays exist (capital-intensive)
    peers:        [VST, CEG]                     # utility-fusion overlap
  biotech-pdufa:
    leads:        []                             # rotates per FDA calendar — no permanent leads
    adjacencies:  []                             # populated dynamically via FMP earnings/PDUFA calendar w/in 30d
    peers:        []
```

**Authoring rule**: max 5 adjacencies per theme (curation > breadth). Reviewed monthly alongside themes table. New entries require: (a) sub-$500M cap, (b) clear thematic linkage, (c) at least one historical sympathy pop event vs lead.

**Lead-ticker liveness check**: at run-time, verify lead ticker still has >$1B cap + active flow (`uw_stock` cap pull). If lead has fallen below $1B → demote theme to dormant for that run + warn.

---

## Backtest validation (2026-05-08)

**Event:** MRAM (Everspin Technologies) +25% close-to-close 5/7→5/8, +30% intraday vs 5/8 high. Cap $504M, float 23.4M, beta 3.0, no direct news catalyst.

**Retro signal score against 13-stack on 5/7 close (PRE-pop, what would have been visible Wed evening):**

| Signal | Threshold | MRAM 5/7 reading | Fire |
|---|---|---|---|
| #1 Float | <50M | 23.4M | ✓ |
| #4 RVOL | >3x 5d MA | 0.8x (5/7 vol 2.75M vs 5d MA 3.44M) | ✗ |
| #8 Theme | match active tag | ai-memory ✓ | ✓ |
| #11 News velocity | ≥3 items <7d w/ tier-1 | 1-2 items only | ✗ |
| #12 Theme-sympathy | lead ≥10% trailing-5d in same bucket | MU +19.3% / SNDK +12.9% / WDC +7.5% — 2 of 3 leads fire | ✓✓ |
| #13 Float-velocity | ≥2x RVOL on float <30M (smallcap-mode, post-backtest) | 5/7: 1.45x ✗  ·  but 4/30 saw 7.4x as setup signal | ~ |

**Score: 3 of 13** (sufficient at smallcap-mode 2/13 threshold) → MRAM **WOULD HAVE SURFACED PRE-POP** on 5/7 evening run via signal #12 theme-sympathy (ai-memory bucket hot from MU+SNDK+WDC trailing-5d strength).

**Threshold tuning applied:**
1. Signal #13 RVOL threshold relaxed `5x → 2x` for smallcap-mode. Rationale: 5/8 actual pop happened on RVOL=2.6x (vs 30d) only — thin floats dislocate on normal volume, no surge required. Tight ≥5x would miss real events.
2. smallcap-mode cap raised `$500M → $750M`. Rationale: MRAM at $504M was borderline-pass; $750M ceiling captures full small-cap range w/o leaking into mid-cap.
3. Signal #12 (theme-sympathy) confirmed as **primary smallcap-mode firing signal**. 2-of-3 leads ≥10% on 5/7 close — strong confluence pre-event.

**Cache:** `data/cache/banger-board-backtest-MRAM-2026-05-08.json` (full retro score detail).

---

## Failure modes

- **UW screener returns empty** → loosen Phase A thresholds incrementally (cap to $2B, float to 100M, SI to 10%) and warn
- **All candidates score <5** → emit "no setups today, sleeve idle" + log lowest-scored 5 for monitoring
- **Sleeve cap reached** → list active picks w/ exit triggers, recommend close before opening new
- **HALT regime** → block, persist nothing, emit pause notice
- **smallcap-mode: no hot leads (all leads <10% 5d)** → no fan-out triggered; Tier-2 emits empty w/ note "no hot themes today, smallcap-tier dormant". Tier-1 still emits. Do NOT lower lead-strength threshold to manufacture picks.
- **smallcap-mode: lead hot but adjacency cap exceeded** ($750M ceiling exceeded by all candidates) → log `[SMALLCAP-CAP-OUT:{theme}]`, the theme contributes nothing this run; surface in next monthly review for cap-threshold revisit.
- **smallcap-mode: Tier-2 fired but Tier-1 has zero picks** → user discipline check: "Tier-1 empty — entering Tier-2 alone violates tier discipline rule. Continue? y/n". Force confirmation, do NOT auto-proceed.

---

## Persistence

After every run:
1. Append run summary to `${SLEEVE_LEDGER}` Performance ledger (date · candidates surfaced · top 5 picks list — even if not entered)
2. Cache full board JSON at `data/cache/banger-board-{YYYY-MM-DD-HHMM}.json` (90d retention)
3. Emit memory entity `banger-board-run:{date}` with picks list (for FOLLOWS link to next run)

---

## Backtest (validation before going live)

Before first live entry, run:

```bash
cd "${TRADE_REFRESH_PATH}" && .venv/bin/python3 -m src.banger_backtest --weeks 12 --min-signals 5
```

(Implementation TBD — uses `mcp__traderkit__backtest_signals` over 12 weeks of historical UW screener output. Validates ≥15% hit-rate floor before live deployment.)

If backtest hit-rate <15% → tighten to ≥6 signals or add new signals to stack before live trading.

---

## What this skill deliberately does NOT do

- **No auto-execute** — output is read-only board; user enters manually
- **No LLC/partnership-account picks** — partnership rules typically forbid speculation; configure exclusions per your accounts
- **No earnings-overnight structures** — gamma blow-up risk
- **No averaging down** — lottery tickets are one-shot
- **No naked OTM weekly calls** — defined-risk only

---

## See also

- Thesis: your `${SLEEVE_LEDGER}` (configure per Configuration section above)
- Companion skills (private — port to your setup): `/trade-refresh`, `/nl-trade`, `/screen-cc`
- Signal sources: your private signal-confluence + scanner-rules ledgers
