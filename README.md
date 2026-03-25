# Options Backtest & GEX Analysis

**Systematic backtest of an options flow strategy using real OPRA market data, with GEX (Gamma Exposure) zone analysis as the primary alpha signal.**

---

## Summary

This project answers one question: **does GEX wall positioning predict options trade outcomes?**

Using 79 institutional flow signals from Unusual Whales, backtested against real tick-level data from Databento's OPRA.PILLAR feed, the answer is yes — and the effect is large enough to be tradeable.

| | KNOWN GEX Zone | UNKNOWN Zone |
|---|---|---|
| Trades | 38 | 27 |
| Win Rate | 32% | 19% |
| P&L | +$9,324 | -$4,650 |

The best-performing zone (`BETWEEN_WALLS`) hits a 50% win rate. The worst (`NEAR_RESIST`) has 0 take-profit hits and a 67% stop-loss rate. Skipping `NEAR_RESIST` and `UNKNOWN` zones alone would have turned a marginal strategy into a consistently profitable one.

---

## Data

**Source:** [Databento](https://databento.com/) OPRA.PILLAR dataset

- Real OPRA options tick data (not synthetic or delayed)
- ~$3.25 for the 79-signal backtest window (using $125 free credit)
- Covers the exact timestamps of each UW flow signal

**Critical lesson learned:** Databento's `to_df()` auto-converts fixed-point prices — never manually divide by 1e9. This caused phantom price discrepancies in early iterations until identified.

---

## Methodology

### Signal Generation
1. Scan Unusual Whales for institutional flow: large sweeps, AGG* signals
2. Apply quality filter:
   - Chain Bid/Ask spread ≥ 95%
   - Premium ≥ $50K
   - Open Interest > 0
3. Record signal timestamp, underlying, strike, expiry, direction

### GEX Zone Classification
For each signal, classify the underlying's price position relative to GEX walls:

| Zone | Definition |
|------|-----------|
| `BETWEEN_WALLS` | Price sits between two GEX walls (support below, resistance above) |
| `NEAR_SUPPORT` | Price is near a GEX support wall |
| `NEAR_RESIST` | Price is near a GEX resistance wall |
| `UNKNOWN` | No clear GEX wall structure at time of signal |

### Backtest Execution
For each signal:
1. Look up the actual OPRA tick at signal timestamp via Databento
2. Enter at the recorded price
3. Set SL/TP based on GEX wall levels (with offset to avoid round numbers)
4. Apply T-5 DTE time stop
5. Record outcome: TP hit, SL hit, or time stop exit

### Analysis
- Segment results by GEX zone
- Compare win rates, P&L, and SL/TP hit rates across zones
- Derive trading rules from statistical patterns

---

## Results

### Overall Performance

| Metric | Value |
|--------|-------|
| Total signals evaluated | 79 |
| Clean trades (after filtering) | 65 |
| Notional capital | $10,000 |
| Net P&L | +$4,674 |
| Profit Factor | 1.27 |
| Win Rate | 26.2% |

### GEX Zone Breakdown

| Zone | Trades | Win Rate | Net P&L | Avg P&L/Trade |
|------|--------|----------|---------|---------------|
| KNOWN GEX (all) | 38 | 32% | +$9,324 | +$245 |
| — BETWEEN_WALLS | best | 50% | — | — |
| — NEAR_SUPPORT | — | — | — | — |
| — NEAR_RESIST | worst | 0% TP | — | 67% SL rate |
| UNKNOWN | 27 | 19% | -$4,650 | -$172 |

### Phase 3: Filtered Results (Zone Rules Applied)

After removing `NEAR_RESIST` and `UNKNOWN` trades, a scenario matrix over different TP/SL configurations shows consistent improvement:

| Scenario | Trades | Win Rate | Total P&L | Profit Factor | $/Trade |
|----------|--------|----------|-----------|---------------|---------|
| ScenE_TP100 | 29 | 41.4% | $1,696 | 1.36 | $58 |
| NoTP_T30_G30 | 29 | 41.4% | $10,495 | 3.22 | $362 |
| NoTP_T30_G20 | 29 | 51.7% | $7,818 | 2.66 | $270 |
| NoTP_T50_G30 | 29 | 48.3% | $12,291 | 3.41 | $424 |
| NoTP_T50_G40 | 29 | 44.8% | $11,675 | 3.29 | $403 |
| TP200_T30_G30 | 29 | 41.4% | $5,324 | 2.13 | $184 |
| TP300_T50_G25 | 29 | 48.3% | $9,377 | 2.84 | $323 |

*Scenario naming: `NoTP` = let winners run, `T30/T50` = time stop at 30/50% DTE remaining, `G20-G40` = GEX wall proximity threshold for SL placement.*

**The core research result:** GEX zone filtering transforms a marginal strategy (PF 1.27) into a robust one (PF 2.6–3.4). The improvement is not from curve-fitting exit parameters — it's from *removing structurally negative-EV trades* identified through zone analysis. Every scenario in the matrix outperforms the unfiltered baseline.

### Derived Trading Rules

Based on the backtest:

1. **Skip NEAR_RESIST zone entirely** — zero take-profit hits, 67% stop-loss rate
2. **Skip UNKNOWN zone** — negative expected value, win rate below breakeven threshold
3. **Prefer BETWEEN_WALLS** — highest win rate (50%), best risk/reward
4. **Let winners run** — removing the TP100 cap and using time-based exits dramatically improves PF (1.36 → 3.0+)
5. **Puts outperformed calls** in the sample period (noted as potentially period-specific, not a permanent rule)

### Position Sizing

Live deployment uses **half-Kelly sizing** derived from each zone's win rate and payoff ratio. Half-Kelly sacrifices ~25% of theoretical growth for ~50% reduction in variance — appropriate for a concentrated options strategy with fat-tailed outcomes.

---

## Project Structure

```
options-backtest-gex/
├── README.md
├── backtest/
│   ├── databento_loader.py      # OPRA.PILLAR data ingestion
│   ├── signal_parser.py         # UW signal processing & quality filter
│   ├── gex_zone_classifier.py   # GEX wall zone classification
│   ├── backtest_engine.py       # Trade simulation engine
│   └── results_analyzer.py      # Zone-segmented performance analysis
├── gex_log/
│   └── gex_log.py               # CLI tool (log/outcome/summary/show)
├── data/
│   └── signals.csv              # Signal list (anonymized)
├── results/
│   ├── trade_log.csv            # Full trade log
│   └── zone_summary.csv         # Zone-level statistics
└── notebooks/
    └── analysis.ipynb           # Exploratory analysis & charts
```

> **Note:** Signal parameters, specific GEX wall values, and entry/exit thresholds are anonymized. The framework, methodology, and statistical results are fully reproducible with a Databento account.

---

## GEX Log CLI Tool

Standalone Python CLI for tracking GEX wall observations over time:

```bash
# Log today's GEX walls for a ticker
python gex_log.py log --ticker SPY --support 580 --resist 595

# Backfill outcome after T+4
python gex_log.py outcome --date 2025-01-15 --result TP

# View zone win-rate summary
python gex_log.py summary

# Show observation history
python gex_log.py show --ticker SPY
```

---

## Phases

| Phase | Status | Description |
|-------|--------|-------------|
| **Phase 1** | ✅ Done | Signal collection, quality filtering, initial backtest |
| **Phase 2** | ✅ Done | GEX zone analysis, zone-segmented performance, trading rule derivation |
| **Phase 3** | ✅ Done | Zone-filtered scenario matrix, TP/SL optimization, half-Kelly sizing |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Market Data | Databento OPRA.PILLAR (real OPRA tick data) |
| Flow Scanner | Unusual Whales (Basic plan) |
| Language | Python 3.10+ |
| Data Processing | pandas, numpy |
| Visualization | matplotlib, Jupyter notebooks |

---

## Related

- [**MAD DOG Terminal**](https://github.com/HaodongYan111/mad-dog-terminal) — The live trading system that uses these backtest-derived rules
- [**FinTel**](https://github.com/HaodongYan111/fintel) — Multi-agent financial intelligence platform (ESG + macro research)
