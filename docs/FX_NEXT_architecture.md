# FX NEXT — Research Stack Architecture (rev 2)

Nov 6, 2026 · @Kobla · rev 2 corrects rev 1; the corrections are listed first so nothing is silently changed.

## What this builds

A research pipeline that proves, out of sample and after the spread you will actually pay, which pairs and timeframes carry a directional edge — then trades the survivors as a portfolio, sized to a volatility target by calibrated probability. The TradingView indicator becomes a display layer; the learning moves to Python on years of data.

## What changed from rev 1, and why

Every item below is either an arithmetic error, a factual error about a data source or service, or a gate that a genuinely good model would fail by construction.

| # | Rev 1 said | Problem | Rev 2 |
|---|---|---|---|
| 1 | 15m clears spread | On OANDA retail spreads, 15m EURUSD costs ≈0.20 ATR per round trip; break-even hit rate with ±1 ATR barriers is ≈64%. No gate in this document is reachable at 15m. | Trade 1h, 4h, D1. 15m and 1m are feature sources only. The cost screen measures 15m once to document the wall. |
| 2 | Expectancy gate > 0.15 ATR after spread, alongside hit-rate ≥ 55% | Mutually exclusive. 55% with symmetric 1 ATR barriers and ~70% barrier-hit rate is 0.07 ATR gross. Net 0.15 would need ≈61% at 4h, ≈66% at 1h. | Net expectancy > 0 at t ≥ 3 (weekly-block bootstrap) and net ≥ 0.5 × round-trip cost. Realistic passing values are 0.02–0.07 ATR. |
| 3 | "Below N = 2,000, 58% is indistinguishable from luck" | 58% is significant at p < 0.01 from N ≈ 210 independent calls. The real problem is that calls are not independent (overlapping windows). | N ≥ 2,000 for a ±1-pt standard error on the rate itself; significance from a weekly-block bootstrap, not a binomial test. |
| 4 | Stability: ≥ 70% of test quarters positive | A true Sharpe-1.0 strategy has 69% positive quarters in expectation, ±10 pts over 22 quarters. The gate rejects the thing it is looking for about half the time. | ≥ 60% positive quarters and t-stat of quarterly P&L ≥ 2.5. |
| 5 | Kill switch: 20-day rolling Brier > 0.25 | At 4h, 20 days ≈ 30 calls; Brier SE ≈ 0.015, threshold sits 0.5 SE above a healthy model. It would halt constantly on noise. | CUSUM on per-call Brier excess, threshold set for ≤ 1 false halt per model-year on the test folds. |
| 6 | Quarter-Kelly, max 1% per position, sized by confidence | Quarter-Kelly at p = 0.56 on even barriers is 3%, so the 1% cap binds on every call and sizing is flat, not continuous. Separately, 1% per position at this call rate is ≈60–90% annualized vol, not 10%. | Kelly-after-cost is the shape; the 10% vol target sets the level. Per-position risk falls out at ≈0.1–0.3% of equity. |
| 7 | "New calls queue behind closes" | A directional call cannot be queued; the moment passes. | Calls at the position cap are skipped and logged as skipped so coverage stays honest. |
| 8 | DXY = … · USDCAD^0.042 · USDSEK^0.042 … "from the pairs themselves" | CAD exponent is 0.091. USDSEK is not among the 15 pairs. | Primary USD factor is a leave-one-out equal-weight basket of the 7 majors. ICE DXY is optional and needs a USDSEK pull. |
| 9 | FX week Sunday 22:00 → Friday 22:00 UTC | The FX day rolls at 17:00 New York, which is 21:00 UTC in US summer time and 22:00 UTC in winter. Half the year is misaligned. | Anchor to 17:00 America/New_York; align 4h and D1 bars to it (OANDA's default). Session flags in local London/New York time. |
| 10 | ForexFactory calendar export for 2019 → now | FF exports the current week only (ff_calendar_thisweek.xml/json); there is no historical export. | Agency-published release schedules for the ~12 series that move FX, plus the FF weekly feed going forward. |
| 11 | FRED daily 2-year yields per currency | FRED has daily US 2y (DGS2) and VIX (VIXCLS). It does not carry daily 2y for AUD, NZD, CHF, CAD, JPY, GBP, EUR. | BIS central bank policy rates (daily, all 8 currencies) for the rate differential; DGS2 and VIXCLS as-is. |
| 12 | Dukascopy "tick volume" and OANDA volume in one column | Dukascopy volume is order-book liquidity on its own venue in millions; OANDA volume is a tick count. Different quantities. Dukascopy spreads are also tighter than what you will trade on. | OANDA bid/ask candles are the price, spread and volume source for labels and features. Dukascopy is a QC cross-check on mids only. |
| 13 | Per-pair LightGBM, min_data_in_leaf 500 | A 24-month 4h training window is ≈1,900 usable rows per pair; a D1 window is ≈320. The model cannot grow past 3 leaves. | One pooled model per timeframe across all pairs, pair as categorical, features in ATR/rank units. Per-pair models only as a comparison. |
| 14 | Meta-model on primary score | Unstated nesting. Trained on in-sample primary scores it learns the primary's overconfidence. | Meta-model and calibrator fit only on out-of-fold primary scores. |
| 15 | 16+ test quarters | 2019 data, 24m train + 3m validate gives first test Q2 2021 → Q3 2026 = 22 quarters. | 22 quarters stated; pull from 2015 to add an out-of-time check. |
| 16 | ~20 GB Parquet for six years of 1m data on 15 pairs | ≈2.9M 1m bars per pair from 2019, 44M rows total, ≈3 GB uncompressed, 1–2 GB as Parquet. | ≈2 GB. |
| 17 | GitHub Actions every 15 min, cost zero | Private-repo free allowance is 2,000 min/month; 15-min cadence over FX hours is ≈2,000 runs/month and over the cap. Cron timing has no guarantee; 5–30 min delays are normal. | Hourly cadence (sufficient for 1h/4h/D1), triggered by an external cron via workflow_dispatch; backtest entry modelled at close + measured latency. |
| 18 | Sharpe 0.8–1.5, go-live at 1.0, kill at 0.5 | Over 5.5 test years the standard error of a Sharpe estimate is ≈0.5. The data can distinguish an edge from zero, not 1.0 from 0.5. | Expected 0.4–1.0 after costs. Gates restated with their standard errors. |
| 19 | 50 build hours | Ingest, QC, features, labels, walk-forward, CSCV, meta, calibration, portfolio sim, serving, dashboard, leakage tests. | 100–120 hours. 10–12 weeks at 10 h/week or 5–6 at 20. |
| 20 | "Rank windows fitted inside the training fold" | Rolling right-aligned ranks are causal by construction and need no fold fitting; the rule applies to any global standardization. | Clarified in the leakage checklist. |

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Pairs | 7 majors + 8 crosses: EURUSD, GBPUSD, USDJPY, USDCHF, USDCAD, AUDUSD, NZDUSD, EURGBP, EURJPY, GBPJPY, AUDJPY, EURAUD, EURCHF, CADJPY, AUDNZD | Breadth is the lever, but crosses are algebraic combinations of majors: 15 pairs hold 8 currencies and roughly 5–6 independent factors. AUDNZD and EURCHF are expected to fail the cost screen below D1. |
| Timeframes | 1h, 4h, D1 traded; 15m and 1m as feature sources | Where signal can clear spread (see cost table). 15m cannot at retail spreads; 1m and 5m are further off. |
| Horizons | 4 and 8 bars per TF | Drift grows with H, noise with √H, and the cost share falls. |
| Target | Triple-barrier: +k·ATR, −k·ATR, time stop at H bars | Matches what a stop/target trade experiences |
| Model | Pooled LightGBM per TF + meta-model gate + calibration | Interactions, honest probabilities, and enough rows at 4h and D1 |
| Validation | Walk-forward, purged, embargoed, OANDA spread charged, CSCV over every config tried | The only numbers that count |
| Stack | Python 3.11+, polars, LightGBM, DuckDB + Parquet, Supabase, Vercel, GitHub Actions | Free tiers; a laptop is enough compute |
| Execution | Out of scope for v1; OANDA v20 when paper trading passes | Robinhood has no FX |

**Realistic outcome, done well.** A passing pair/TF: 54–57% hit rate on calls at 20–30% coverage, net expectancy 0.02–0.07 ATR per call. Portfolio of 6–10 survivors: Sharpe 0.4–1.0 after costs, measured with a standard error of about 0.5 over the test window. Several pairs and most of 1h will fail; that is the information this stack produces.

## The cost wall (why the timeframes changed)

Round-trip cost in ATR units is c = spread / ATR(14) at that timeframe. With symmetric k·ATR barriers, a barrier-hit fraction q (≈0.7 for k = 1, H = 4) and time-stop outcomes treated as zero drift:

```
net expectancy per call  E = q · k · (2p − 1) − c
break-even hit rate      p* = 0.5 + c / (2 · k · q)
```

Indicative values on OANDA spread-only pricing, typical 2024–25 ATRs. The cost screen recomputes these from the store per pair, TF and session before any modelling.

| Pair | Spread (pips) | c at 15m | c at 1h | c at 4h | c at D1 | p* at 1h (k=1) | p* at 4h (k=1) | p* at D1 (k=1) |
|---|---|---|---|---|---|---|---|---|
| EURUSD | 1.2 | 0.20 | 0.086 | 0.040 | 0.017 | 56.1% | 52.9% | 51.2% |
| USDJPY | 1.2 | 0.17 | 0.075 | 0.034 | 0.015 | 55.4% | 52.4% | 51.1% |
| GBPUSD | 1.5 | 0.21 | 0.088 | 0.039 | 0.018 | 56.3% | 52.8% | 51.3% |
| GBPJPY | 3.0 | 0.20 | 0.079 | 0.035 | 0.016 | 55.6% | 52.5% | 51.1% |
| AUDUSD | 1.3 | 0.29 | 0.12 | 0.054 | 0.024 | 58.4% | 53.9% | 51.7% |
| USDCHF | 1.5 | 0.30 | 0.125 | 0.058 | 0.025 | 58.9% | 54.1% | 51.8% |
| NZDUSD | 1.8 | 0.40 | 0.18 | 0.082 | 0.036 | 62.9% | 55.8% | 52.6% |
| EURGBP | 1.3 | 0.37 | 0.16 | 0.072 | 0.033 | 61.6% | 55.2% | 52.3% |
| EURCHF | 1.8 | 0.60 | 0.23 | 0.10 | 0.045 | 66.1% | 57.1% | 53.2% |
| AUDNZD | 2.5 | 0.63 | 0.25 | 0.11 | 0.050 | 67.9% | 58.1% | 53.6% |

At k = 1.5 (the 8-bar horizon) multiply the excess over 50% by two-thirds. The 15m column is 59–68% break-even for every pair at k = 1; no realistic classifier reaches it. The 1h column is marginal for the five most liquid pairs and closed for the rest. 4h and D1 are where a 55% model has room.

**Cost screen (pre-gate, runs before any model):** a pair/TF enters the candidate set only if its median c over the test years is ≤ 0.06. This removes most of the 90 pair/TF/horizon combinations before they count against the overfitting budget.

## System overview

Bars flow from OANDA into one Parquet store; Dukascopy cross-checks it; the calendar and macro feeds join as-of. A model reaches the scoring job only after the cost screen, the walk-forward gates and the portfolio simulation. The serving layer never sees one that did not pass.

## Data layer

One Parquet store of 1-minute bid/ask bars per pair, 2015 to now, from which every timeframe is resampled. Bid/ask is non-negotiable: spread-blind labels overstate every edge. The spread must be the one you will trade on, so OANDA is the price source and Dukascopy is a check.

| Source | Gives | Role |
|---|---|---|
| OANDA v20 practice account (free API) | 1m candles with bid, ask and mid (price=BAM), max 5,000 per request, history to the mid-2000s for majors; volume = tick count | Backfill and ongoing pull. ≈2.9M 1m bars per pair from 2019 → ≈580 requests per pair, ≈9,000 for 15 pairs, under an hour at a polite rate. |
| Dukascopy (free historical download) | 1m bid/ask OHLC; volume = liquidity on Dukascopy's own book, in millions | QC cross-check on mid prices (flag |Δmid| > 2 pips). Not a label or volume source. |
| Agency release schedules | Exact UTC timestamps of the scheduled events that move FX: NFP and US CPI (BLS), FOMC (Fed), ECB, BoE, BoJ, RBA, RBNZ, BoC, SNB decisions, UK CPI (ONS), Eurozone flash CPI (Eurostat). Each agency publishes historical schedules. | Exclusion windows and minutes-to-event features back to 2015 without scraping |
| ForexFactory weekly JSON | This week's high-impact events | Fills the long tail from now on; not available historically |
| FRED (daily) | DGS2 (US 2y, 3pm close), VIXCLS | Risk sentiment and the USD leg of rate differentials |
| BIS central bank policy rates (daily) | Policy rate for USD, EUR, JPY, GBP, CHF, CAD, AUD, NZD | Rate differential per pair as a step function |

Storage: `data/bars/{pair}/{year}.parquet` with columns `ts_close_utc, bid_o/h/l/c, ask_o/h/l/c, tick_count`; DuckDB for queries. From 2019 that is ≈44M rows and ≈2 GB of Parquet; from 2015, ≈3.5 GB. Everything fits a 16 GB laptop.

Time handling: source candles are stamped at bar open; the store stamps them at bar close. The FX day rolls at 17:00 America/New_York, which is 21:00 UTC in US summer and 22:00 UTC in winter; 4h and D1 bars align to that roll, matching OANDA's default alignment so backtest bars and live bars are the same bars. Weekends dropped. Mid price is the modelling series; bid/ask is used only for labels and P&L.

USD factor: a leave-one-out equal-weight basket of the 7 majors' USD returns (the pair being modelled is excluded so the feature is not its own return). If the ICE DXY is wanted it needs a USDSEK pull and reads:

```
DXY = 50.14348112 · EURUSD^-0.576 · USDJPY^0.136 · GBPUSD^-0.119 · USDCAD^0.091 · USDSEK^0.042 · USDCHF^0.036
```

QC report per pair before anything downstream: gap list (bars missing inside sessions), duplicate timestamps, spread outliers (> 10× session median), mid disagreement with Dukascopy, bars per day, and the cost table above computed from the actual feed. A pair fails QC until the gap list is explained.

## Feature layer

About 60 features per pair and timeframe, every one computed from bars that had closed at the time. Five families; the last two are what Pine could not do. All price-scaled features are in ATR or rank units so the pooled model sees comparable inputs across pairs.

| Family | Features | Note |
|---|---|---|
| Price (the v4 set) | returns at lags 1, 2, 4, 8; momentum EMA at 3 windows; regression slope; distance from 20/50/100-bar mean; RSI; range position; AR(1) term; ATR / long ATR; Parkinson vol; distance to day high/low; HTF trend at 4× and 16× | HTF values from the last completed HTF bar only |
| Microstructure (from 1m sub-bars) | tick-count z-score; signed volume (sum of sub-bar tick counts signed by sub-bar close direction); effort vs result (tick z × |return|); intrabar reversal count; close position within the sub-bar distribution; spread z-score | The genuinely new information source. Tick count, not Dukascopy liquidity. |
| Clock | hour sin/cos in New York time; day of week; Asia / London / NY / overlap flags computed in Europe/London and America/New_York local time; minutes to next scheduled event; minutes since last | DST differs between London and New York for 2–3 weeks a year; local-time flags handle it |
| Cross-asset | leave-one-out USD factor return and trend; basket return of the quote and base currencies; policy-rate differential (daily); VIX level and change (daily) | Daily values broadcast forward, never backward |
| Cross-sectional | this pair's momentum rank across the 15; USD-strength factor; base-currency strength factor | Only possible with all pairs in one store |

Every continuous feature is stored twice: raw, and as a rolling 250-bar right-aligned percentile rank. Rank versions survive vol-regime shifts; LightGBM picks whichever splits better. Features with more than 1% missing values, or a correlation above 0.97 with another, are dropped before training.

Output: `data/features/{tf}.parquet`, one row per pair and bar close, labels joined by timestamp.

## Labels

The primary label is which barrier price touches first after the bar closes: +k·ATR (label +1), −k·ATR (−1), or the time stop at H bars (0, dropped from the direction model but kept for P&L). k = 1.0 ATR for 4-bar and 1.5 ATR for 8-bar horizons; both run and are compared. Expect 25–35% time stops.

Barriers are spread-aware: a long enters at the ask and its barriers are checked against the bid; a short the reverse. Entry is at the ask of the 1m bar `latency` minutes after bar close, where `latency` is the measured scheduler delay (start at 5 minutes), not at the closing mid. A move that only clears the barrier on the mid is a loss in practice and is labelled as one.

The v4 label (sign of the H-bar close-to-close move, minimum 0.3 ATR) is kept as a secondary target so Python can be checked against Pine on the same bars.

Overlapping samples: with H = 8 every bar shares its outcome window with up to seven neighbours. Each sample is weighted by the inverse of the number of concurrently open samples over its window, so a long trend does not count forty times. Rows with a scheduled high-impact event inside the window are excluded from training, and the live scorer suppresses calls under the same rule.

Usable rows per pair after time stops and event windows, and per 24-month training window:

| TF | Rows per pair (2019→) | Per 24-month window, one pair | Per window, pooled 15 pairs |
|---|---|---|---|
| 1h | ≈26,000 | ≈7,500 | ≈110,000 |
| 4h | ≈6,600 | ≈1,900 | ≈28,000 |
| D1 | ≈1,100 | ≈320 | ≈4,800 |

This is why the model is pooled across pairs.

## Models

Three models in sequence per timeframe: a direction model, a gate that predicts whether the direction call will be right, and a calibrator that turns scores into probabilities you can size on.

| Stage | Model | Input | Output | Purpose |
|---|---|---|---|---|
| 0 Baseline | Logistic regression on the v4 features | 18 features | P(up) | Proves the harness reproduces Pine; the number LightGBM must beat |
| 1 Primary | LightGBM, binary, pooled across pairs with pair as categorical; 300–800 trees, num_leaves 15–31, min_data_in_leaf 200 (1h/4h) or ridge logistic at D1, feature_fraction 0.7, lambda_l2 10, early stopping on the validation fold | ~60 features | P(up), direction call | Learns interactions (momentum × regime × session) a linear model averages away |
| 2 Meta-model | LightGBM on {out-of-fold primary score, regime, vol, session, HTF agreement, event distance, spread z, hit rate over the last 50 resolved calls} | 10 features | P(call is right) | Gates and sizes; the surgical version of the 56% threshold |
| 3 Calibration | Isotonic regression on out-of-fold meta scores where the validation fold has ≥ 1,000 calls (1h, 4h pooled); Platt scaling where it has fewer (D1) | meta score | Calibrated p | Reliability slope near 1 is a go/no-go gate |

The meta-model and calibrator see only primary scores produced out of fold, otherwise they learn the primary model's in-sample overconfidence. The hit-rate feature uses only outcomes resolved before bar t.

Ensembling is across targets, not seeds: a call needs the 4-bar and 8-bar models to agree, and the next timeframe's model (1h → 4h, 4h → D1) not to disagree. Agreement counts are also meta-model inputs.

Retrain cadence: full walk-forward refit monthly; the meta-model and calibrator weekly on the latest 6 months. Models are versioned by training-window end date and stored with their feature list and validation report.

## Validation protocol and go/no-go gates

Every number that matters comes from walk-forward out-of-sample folds with OANDA spread charged; nothing from the training window is ever reported. Train 24 months, validate 3 (early stopping, meta-model, calibrator), test 3, roll forward one quarter at a time. With data from 2019 the first test quarter is Q2 2021 and there are 22 test quarters through Q3 2026. Pulling from 2015 adds 2017–2020 as a secondary out-of-time check.

Between train and test, purge H bars (labels that overlap the boundary) and embargo one further day. Reported metrics are pooled across all test quarters, by pair/TF and by regime, session and quarter. Per-fold numbers are too noisy to gate on: a 3-month 4h test fold has ≈100 calls per pair.

| Gate | Pass threshold | Why this number |
|---|---|---|
| Cost screen | median spread / ATR ≤ 0.06 at that TF | Break-even hit rate ≤ 54.3% at k = 1; leaves a 55% model room to pay for a 50% spread widening |
| Hit rate on calls | ≥ 55% on N ≥ 2,000 calls; weekly-block-bootstrap 99% interval excludes 50% | N ≥ 2,000 makes the rate itself stable to about ±1 pt. Overlapping windows make calls dependent, so a binomial test overstates significance; the block bootstrap does not. |
| Brier score (calls) | < 0.245 | A calibrated model whose calls average p = 0.58 scores 0.244; coin flip is 0.250. Passing means calibrated and genuinely above threshold. |
| Calibration slope | 0.8–1.2, reliability curve monotone | Sizing on p requires p to mean what it says |
| Net expectancy per call | > 0 with t ≥ 3 (weekly blocks), and ≥ 0.5 × round-trip cost | Realistic passing values are 0.02–0.07 ATR. Gross edge at least 1.5× cost means a 50% wider spread leaves it positive. |
| Stability | ≥ 60% of test quarters positive; t-stat of quarterly P&L ≥ 2.5; no single quarter > 30% of total | A Sharpe-1.0 strategy has 69% positive quarters in expectation, ±10 pts over 22 quarters; 70% would reject it half the time |
| Shuffle test | AUC 0.50 ± 0.02 on randomized labels | Catches feature–label lookahead. It does not catch purge failures; the timestamp-overlap test does. |
| Overfit check | Probability of backtest overfitting (CSCV) < 0.2 across all configs tried; report the Deflated Sharpe Ratio with the trial count | Every config tried counts, including the ones thrown away and the ones the cost screen removed |

A pair/TF that passes every gate enters the portfolio. One that fails is reported, not tuned until it passes. The configuration count is logged from day one so the CSCV test is honest.

What the test window can and cannot tell you: over 5.5 years the standard error of an annualized Sharpe estimate is √((1 + SR²/2)/5.5) ≈ 0.5. The gates can establish that an edge is not zero. They cannot distinguish Sharpe 1.0 from Sharpe 0.5, and the plan below does not pretend they can.

## Portfolio and sizing

The gain lives here, not in the forecaster: the same weak signal spread across every pair/TF that passed its gate, sized to a vol target by calibrated probability, with currency exposure capped.

Expected scaling follows the fundamental law of active management, IR ≈ IC · √BR · TC, with BR the number of *independent* bets. Nominal bet count for 10 survivors (≈15 calls/week per 1h model, ≈4 per 4h, ≈1 per D1) is ≈4,000 a year. Same-pair, same-direction calls across horizons and timeframes, and the shared USD factor, cut the independent count to ≈500–1,000. The improvement over one pair on one timeframe is in √BR, so a 10× bet count is a ≈3× IR, not 10×. With IC 0.03–0.06 and TC 0.5–0.6:

```
IR ≈ 0.03 · √500 · 0.5 ≈ 0.34      to      0.06 · √1000 · 0.6 ≈ 1.1    (before costs)
```

Hence the 0.4–1.0 after-cost expectation.

Sizing per call: Kelly after cost on even barriers is f = (2p − 1 − c) / k; a quarter of it is the position's relative weight. At p = 0.56 and c = 0.04 that is 0.08 (×¼ = 2% of equity), at p = 0.62 it is 0.20 (×¼ = 5%), so without a level control the cap binds on nearly every call and sizing is flat. The level is therefore set by the vol target, not by Kelly:

| Rule | Setting | Derivation |
|---|---|---|
| Portfolio vol target | 10% annualized realized vol | Primary control. |
| Risk per position | derived: ≈0.1–0.3% of equity to the stop | ≈4,000 trades/year, per-trade P&L std ≈ 0.9 × risk, average pairwise correlation ≈ 0.3 among concurrent positions → 10% vol implies this range. The 1% cap from rev 1 would never bind and is dropped. |
| Shape within the level | quarter-Kelly-after-cost, zero below calibrated p = 0.56 | The selectivity dial made continuous |
| Per-currency exposure | net exposure to any currency ≤ 3× a single position's risk | Mechanical cap on the USD factor bet |
| Open positions | cap 12; expected average load ≈ 7; calls arriving at the cap are skipped and logged, never queued | Arrival rate × holding time: 60 × 5h + 16 × 20h + 2 × 120h per 120h week ≈ 7 open |
| Kill switch | CUSUM of per-call Brier excess over 0.25, reset at zero, threshold set on the test folds for ≤ 1 false halt per model-year; a halt stops that model until retrain | A fixed 20-day window has ≈30 calls at 4h and trips on noise |

The portfolio simulation runs on the same walk-forward test folds with the same spread model and the same entry latency, so its Sharpe is comparable to the per-model gates. Decision rule, with the ≈0.5 standard error in mind:

- Sharpe < 0.3 after costs, or fewer than 3 survivors: stop.
- 0.3–0.7: continue research; no paper trading.
- ≥ 0.7 with ≥ 3 survivors and a lower 1-SE bound above 0.2: paper trade.

## Serving

A GitHub Actions job runs hourly during FX hours (the traded timeframes are 1h, 4h and D1, so hourly is sufficient): pull the latest bars from OANDA, compute features, score every live model, write calls to Supabase, and let a database webhook push the ones with size > 0 to your phone. Supabase cannot run LightGBM, so Python runs in the Action and Supabase is the system of record.

Scheduling facts that shape this: GitHub's cron has no timing guarantee and delays of 5–30 minutes are normal under load, with occasional skips. A private repo on the free plan has 2,000 Actions minutes a month; hourly over FX hours is ≈520 runs a month, ≈1,000 minutes with dependency caching, under the cap (15-minute cadence would be ≈2,000 runs and over it). Trigger the workflow from an external free cron via `workflow_dispatch` rather than GitHub's `schedule`, and log the measured delay from bar close to scoring; that delay is the `latency` the backtest charges.

| Table | Columns | Written by |
|---|---|---|
| signals | pair, tf, ts, model_version, p_up, p_calibrated, call (up/down/none/skipped), size, regime, session, latency_s | Scoring job, every bar |
| outcomes | signal_id, resolved_ts, barrier_hit (+1/−1/0), pnl_atr, pnl_pips, entry_ask, exit_bid | Scoring job, when the window closes |
| model_health | pair, tf, model_version, rolling_hit, rolling_brier, cusum, coverage, feature_drift, status | Scoring job, daily |
| events | ts_utc, currency, impact, name, source | Calendar sync, weekly |

Alerts: a Supabase database webhook on `signals` insert where size > 0 calls an edge function that posts to ntfy or Telegram with pair, direction, calibrated %, size and the barrier levels. The same path can later post to OANDA for execution. The hourly writes keep the free Supabase project from pausing (it pauses after 7 idle days).

Dashboard: a Vercel Hobby page over signals, outcomes and model_health — open calls, per-model scorecards (rolling hit rate with its standard error, Brier, coverage, skipped-call count), the live reliability curve, and the kill-switch state. This is the calibration scoreboard from v3, moved off the chart and onto something that keeps history.

Monitoring: feature-drift score (population stability index per feature versus its training distribution) and the Brier CUSUM are written daily; PSI above 0.25 or a CUSUM breach sets status = halted and opens a retrain. Nothing retrains automatically.

The TradingView indicator stays as the on-chart view. v4 already works standalone; a v5 variant can take a pasted weight vector from the Python logistic baseline so the chart shows the same call the stack made.

## Repo layout

One Python package, one config per pair, one command per stage. Every stage reads Parquet and writes Parquet, so any step can be rerun alone.

```
fxnext/
  configs/
    pairs.yaml          # pair list, pip size, OANDA instrument names
    sessions.yaml       # roll time (17:00 America/New_York), session windows in local tz
    features.yaml       # windows, rank length, drop rules
    labels.yaml         # k per horizon, H, min move, entry latency
    model.yaml          # LightGBM params, walk-forward windows, gates, cost screen threshold
  fxnext/
    ingest/             # oanda.py  dukascopy.py  calendar.py  macro.py  qc.py
    features/           # price.py  micro.py  clock.py  cross.py  ranks.py  build.py
    labels/             # triple_barrier.py  horizon.py  weights.py
    models/             # baseline.py  primary.py  meta.py  calibrate.py  registry.py
    eval/               # cost_screen.py  walkforward.py  metrics.py  bootstrap.py  shuffle.py  cscv.py  report.py
    portfolio/          # sizing.py  exposure.py  voltarget.py  simulate.py
    serve/              # score_live.py  supabase.py  alerts.py  health.py
  data/                 # parquet, gitignored
  models/               # versioned artifacts + validation reports
  reports/              # per pair/TF HTML reports from eval.report
  dashboard/            # Next.js, deployed to Vercel
  .github/workflows/    # score.yml (workflow_dispatch, hourly from external cron), retrain.yml (monthly), calendar.yml (weekly)
  tests/                # leakage tests run on every build
```

Commands, in pipeline order: `fxnext ingest`, `fxnext qc`, `fxnext features`, `fxnext labels`, `fxnext screen`, `fxnext train`, `fxnext eval`, `fxnext portfolio`, `fxnext score`. Each takes `--pair` and `--tf` or runs the full matrix.

## Build plan

Each phase ends at a gate; a gate that fails stops the plan rather than loosening the gate. Honest effort: ingest and QC ≈ 12 h, features ≈ 12 h, labels and weights ≈ 6 h, walk-forward, gates, bootstrap and CSCV ≈ 18 h, meta and calibration ≈ 6 h, portfolio simulation ≈ 10 h, serving and alerts ≈ 12 h, dashboard ≈ 8 h, leakage tests ≈ 8 h, plus debugging. That is 100–120 hours: 10–12 weeks at 10 h/week, or 5–6 at 20. Then 8 weeks of paper trading, which at ≈80 calls a week pooled gives ≈640 calls, enough to detect a 5-pt hit-rate shortfall at about 2.5 standard errors.

Kill criteria, decided now so they are not renegotiated later:

- After the cost screen: fewer than 6 pair/TF combinations survive → the edge, if any, is at D1 only; evaluate D1 alone and stop if it fails.
- After eval: no pair/TF passes every validation gate → stop; the edge is not at these horizons.
- After the portfolio simulation: Sharpe below 0.3 after costs, or fewer than 3 survivors → stop; breadth did not rescue a thin signal.
- Paper trade: pooled live hit rate more than 2 standard errors below out-of-sample on ≥ 300 calls → halt and audit for leakage. Per-model comparisons at this sample size are noise and are reported, not acted on.

Cost: zero in cash if the repo is private and under 2,000 Actions minutes a month, or public. OANDA practice, Supabase, Vercel Hobby, ntfy and an external cron are free at this scale; the data is ≈2 GB. Time is the only budget.

## Leakage and realism checklist

Each item is a unit test in `tests/`, run on every build. A backtest showing 60%+ on 4h FX direction, or anything at all on 15m after spread, has failed one of these.

- [ ] Every rolling statistic is right-aligned: the value at bar t uses bars ≤ t only
- [ ] Rolling percentile ranks are causal by construction; any *global* standardization (a fixed mean/std or a quantile map) is fitted inside the training fold only and applied to test rows
- [ ] Higher-timeframe features come from the last HTF bar that closed before bar t, never the one containing it
- [ ] Daily macro values (policy rates, DGS2, VIX) are joined as-of the prior day's close in UTC, not the same calendar day
- [ ] Event timestamps come from the issuing agency's schedule, are converted to UTC from the agency's own timezone, and are checked against a known NFP release (first Friday, 08:30 America/New_York)
- [ ] Session flags and the daily roll use America/New_York and Europe/London local time, and a test covers a week where US and UK DST differ
- [ ] Resampled bar timestamps are the bar close; features at bar t cannot see bar t+1's open
- [ ] Labels use bid for long exits and ask for short exits; the barrier check never uses mid
- [ ] Entry is at the ask/bid of the 1m bar at close + configured latency, never the closing mid; the configured latency is ≥ the measured live median
- [ ] Purge and embargo are applied per fold and asserted with a timestamp-overlap test
- [ ] Meta-model and calibrator are trained only on out-of-fold primary scores; a test asserts no training-fold score reaches them
- [ ] The rolling hit-rate feature uses only outcomes whose resolution timestamp is < t
- [ ] The event exclusion rule in training is the same function the live scorer uses to suppress calls
- [ ] Shuffle test: training on randomized labels yields AUC 0.50 ± 0.02 on every pair/TF
- [ ] Configuration count is logged for the CSCV test; discarded configs and cost-screened configs count
- [ ] Live scoring and backtest scoring share one feature function and one label function, not two implementations
- [ ] Spread used in P&L is the actual OANDA bid/ask at entry and exit, not a flat assumption and not Dukascopy's
- [ ] Skipped calls (position cap, event window, halted model) are logged and included in coverage statistics
