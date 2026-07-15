# HFT Autoresearch Audit

Investigation of the claim: "I let Claude run an autoresearch loop and it made €25k/month trading order book data" — LinkedIn post + repo [giacomo-ciro/autoresearch-hft](https://github.com/giacomo-ciro/autoresearch-hft).

Goal: determine whether this pattern is usable for real side income.

## The claim

Weekend project: Claude Code + Codex + Antigravity + conductor.build ran an autonomous agent loop (`AUTORESEARCH.md`) that implemented, backtested, kept/archived ~30 trading strategies over order-book data. Best strategy: train PnL 219,898.8, val (out-of-sample) PnL 25,785.9 — reported as "€25k/month."

## What the repo actually is

`README.seed.md` (still present in the repo) reveals the dataset is a **private prop-trading-firm interview take-home**, not live/public market data:

> "Task at a glance... please send a short free-form writeup on how you arrived at your solution... If you leaned heavily on an LLM, attaching chat transcripts is welcome but not required."

Key facts pulled from source:

- Two correlated futures, `SYMBOL_A` (standard) and `SYMBOL_B` (mini, 1/25 notional). Task: detect "pre-trigger moments" (lone sweep deal on A) and decide how many B lots to aggress.
- Rules explicitly state: **"No slippage, no latency constraint — the only trading cost is the per-lot fee"** (`fee = 0.04/lot`).
- Model can always take the full displayed top-of-book quantity, instantly, at the pre-trigger price. Real HFT execution never gets this for free.
- `download_data.sh` has `URL="REDACTED"` and `SHA256="REDACTED"` — dataset distributed to interview candidates only. Not obtainable independently, not live.
- No broker/exchange integration anywhere in the code. `scripts/main.py` only runs an offline backtest loop against the frozen historical dataset (Aug 2025–Feb 2026, val = Feb 2026).
- 30+ strategies were tuned repeatedly against the **same single held-out validation month** (Feb 2026). Train PnL / val PnL ratio ~8.5x raw (~1.4x when normalized for the 6-month-train vs 1-month-val period length) — consistent across nearly all strategies, which is the signature of implicit overfitting via repeated peeking at "out-of-sample" data, not proof of a stable edge.
- `AGENTS.md`/`AUTORESEARCH.md` are legit, well-designed autonomous-loop scaffolding (strict CAN/CANNOT rules, commit conventions, failure archive with mandatory postmortem docstrings) — the harness pattern itself is good practice for automated hypothesis generation. It is not evidence of a deployable strategy.

## Verdict

**Not usable for real side income, as marketed.** The €25k figure is selected simulated gross PnL under a frictionless, private-dataset, zero-latency interview objective — not verified trading income. Three things would need to change for a real attempt:

1. Treat the current "validation" month as development data; build genuinely hidden outer test folds (walk-forward, purged, embargoed).
2. Replace the toy PnL formula with a real execution simulator: post-latency order books, partial/zero fills, two-sided real fees, executable (not midpoint) exits.
3. Target short-horizon event trading, not latency-arb HFT, unless edge survives realistic retail decision-to-fill latency (50–200ms vs. the microsecond races that matter in real HFT).

## Deep research findings (numbers)

### Data sources (public, for a real attempt)

| Source | Granularity | Cost | Fit |
|---|---|---|---|
| Binance/Bybit/OKX WS | L2 deltas (market-by-price, no order IDs), 10–500ms cadence | Free live | Adequate for coarse taker-event research only |
| OKX historical L2 | Free L2 back to Mar 2023 | Free | **Cheapest way to falsify a crypto version of the signal** |
| Tardis.dev | Full historical L2 deltas/trades, multi-venue | ~$700/mo (Solo derivatives tier, verify) | Convenient but pay only after free-data study shows promise |
| Databento (CME MBO) | True order-by-order L3, nanosecond ts | Usage-based, ~$179/mo live plan + credits | **Closest public reproduction of the interview task** (ES/MES, NQ/MNQ pairs) |
| Polygon/Massive equities | Trades + NBBO, no full depth | ~$199/mo | Insufficient for depth-sweep strategies; top-of-book lead-lag only |

### Execution frictions (why the backtest PnL doesn't survive contact with reality)

- **Latency**: FCA/Budish-style research shows real latency races in liquid equities are won in ~5–10 microseconds, concentrated in a handful of firms with colocation. Retail decision-to-fill latency (50–200ms) destroys most/all edge for fast signals; crypto lead-lag (hundreds of ms–seconds) leaves more room from a co-located VPS (~1ms).
- **Fills**: guaranteed full top-of-book fill (repo's core assumption) is the single most flattering backtest simplification. Realistic partial-fill/queue modeling typically cuts naive PnL 50–95%.
- **Fees**: Binance/Bybit retail taker ~5–5.5bp per side (~10bp round trip). Most genuine microstructure edges are 2–5bp gross — retail taker fees alone flip the sign on most of them.
- **Impact/adverse selection**: square-root impact law + adverse selection on passive fills further erode edge; negligible only at very small size (which caps achievable income — see capital math below).
- Combined: a zero-friction, full-fill backtest overstating live PnL by 5–50x is normal; sign flips are common.

### Overfitting

- 30+ trials selected against one fixed validation month = textbook multiple comparisons. Expected max Sharpe of N noise strategies grows ~√(2·ln N).
- Required protocol before risking capital: Deflated Sharpe Ratio (Bailey & López de Prado 2014), Probability of Backtest Overfitting / CSCV (Bailey et al. 2015), purged k-fold CV with embargo + walk-forward with nested selection (López de Prado, *AFML* 2018), one final untouched holdout used exactly once.
- Given the repo's setup (no purging, no deflation, repeated val reuse), prior that surviving strategies have genuine post-cost edge: low.

### Capital math (why this can't be "side income" scale)

- €1,000/month at 1% net monthly return requires €100,000 capital.
- €1,000/month at 2% requires €50,000.
- €25,000/month at 2% requires €1.25M. Even at an unusually high 5%/mo, €25k/mo needs €500k.
- Turnover/leverage can lower nominal capital needs but raise fee exposure, liquidation, and tail risk.

### Regulatory (EU)

- Ordinary own-account trading via a broker generally falls under the MiFID II own-account exemption (Art. 2(1)(d)).
- Licensing triggers to avoid: direct venue membership, market making, DEA, "qualifying" high-frequency algorithmic trading technique as a venue member, managing/pooling others' money, discretionary trading of client accounts.
- No EU equivalent of the old US $25k PDT rule (and even that US rule changed — FINRA replaced it with a new intraday margin framework effective June 2026).

### Comparable public evidence

- Real evidence for the underlying phenomenon exists at the institutional level (Virtu S-1, FCA/Budish latency-race data) — concentrated among firms with colocation and scale.
- No independently audited retail/solo track record found for this "sweep/lead-lag" strategy type. Self-reported claims (like this LinkedIn post) are not verification.

### Probability-weighted outcome (solo operator, public data, 3–6mo part-time)

| Outcome | Probability | Monthly net |
|---|---:|---:|
| Negative or ~€0 | ~70% | −€300 to €0 |
| Breakeven / niche learning | ~24% | €0–€300 |
| Modest viable edge | ~5% | €300–€1,500 |
| Strong niche | ~0.9% | €1,500–€5,000 |
| Exceptional | ~0.1% | >€5,000 |

Expected value: roughly €0–€200/month before valuing own labor, likely negative after data/infra costs. Probability of reliably exceeding €1,000/mo within 6 months: ~1–5%. Probability of reaching the claimed €25k/mo under retail constraints: well under 0.1%.

## If still curious: minimum viable cheap experiment

~€0–200, 4–6 weeks, no capital risk:

1. Record Binance + Bybit L2 diffs + trades from a Tokyo-region VPS for 3–4 weeks (free data, ~€50 VPS), or use OKX's free historical L2 directly.
2. Define the sweep/lead-lag rule *before* looking at results. Re-test against latency stress (5/50/100/200ms), real taker fees, and a conservative fill-only-if-price-traded-through model — not midpoint exit.
3. Apply Deflated Sharpe / Probability of Backtest Overfitting corrections given true trial count.
4. If still positive after (2)–(3): 2 weeks shadow/paper trading against the live feed, then minimum-size live test with a hard loss cap. Kill it the moment live slippage-per-trade exceeds backtest assumptions.

Preferred venue for a closer reproduction of the original task: **CME ES→MES or NQ→MNQ via Databento MBO** (real order-by-order queue data, same standard/mini structure). Cheapest falsification path: **OKX free historical L2**.

## Bottom line

The autoresearch harness pattern (agent loop: implement → backtest → keep/archive → repeat, with strict rules and a failure-postmortem archive) is a legitimate, reusable technique for automated hypothesis generation — worth borrowing for other research problems (e.g. real Kaggle-style competitions). The specific HFT claim in this post is not a side-income opportunity: private non-live dataset, frictionless-fill assumption, unaudited/overfit result, and capital requirements (€50k–€1M+ for meaningful monthly income) that don't fit a "side hustle" framing.
