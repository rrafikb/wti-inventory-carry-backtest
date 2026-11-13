# WTI Crude Oil: Inventory Surprises, Carry, and a Failed Holdout

A systematic backtest of a long-only weekly WTI crude oil strategy built on public EIA data, run under pre-declared testing rules with a locked out-of-sample holdout.

[![Open In Colab]([https://colab.research.google.com/assets/colab-badge.svg](https://colab.research.google.com/github/rrafikb/wti-inventory-carry-backtest/blob/main/wti_inventory_carry_backtest.ipynb))]

## TL;DR

- **Weekly EIA inventory surprises move the price, but only on release day.** Correlation with the release-day return is −0.138 (about 5.6 standard errors, 1990–2021). Correlation with the following week's return is −0.034, indistinguishable from zero. The information is priced in by Wednesday's close.
- **Time-series momentum on WTI alone is not robust.** All three lookbacks were positive over 1990–2021, but none cleared the pre-set significance bar, and the results were inconsistent over 2010–2021.
- **Carry with an inventory filter looked promising in-sample.** The rule "long when the curve is in backwardation and inventories drew at least as much as seasonally normal" returned a Sharpe of 0.59 vs 0.17 for buy-and-hold after costs (1990–2021). The block-bootstrap p-value was 0.026, one-sided.
- **It failed its locked holdout.** Over Jan 2022 – Mar 2024 its Sharpe was −0.05, against 0.42 for buy-and-hold. It was not re-tuned afterwards.

## Motivation

This started as a discretionary crude oil strategy traded on a simulated £100k prop-firm evaluation account (Jan–Mar 2026). That strategy combined Bloomberg-tracked tanker movements, RSI, realised volatility and news sentiment.

One instrument over one quarter cannot separate an edge from a favourable trend. So the idea was rebuilt as a rules-based backtest over decades of public data, to test whether physical oil flows actually predict weekly returns.

Tanker tracking was replaced by EIA weekly inventories, the official measure of the same physical flows, for two reasons. Bloomberg ship-tracking history is short, and it can't be published.

## Data

| Data | Source | Series | Coverage |
|---|---|---|---|
| US commercial crude stocks (excl. SPR) | EIA weekly | `WCESTUS1` | 1982 – present |
| Crude imports / exports / refinery inputs | EIA weekly | `WCRIMUS2`, `WCREXUS2`, `WCRRIUS2` | Used for diagnostics |
| NYMEX WTI futures, contracts 1 and 2 | EIA daily | `RCLC1`, `RCLC2` | 1985 – 5 Apr 2024 |

EIA stopped publishing futures prices after 5 April 2024, which sets the end of the usable sample.

## Method

### Roll-adjusted futures returns

Raw "contract 1" prices jump whenever the front contract expires. Those jumps are not returns anyone earned, and they bias results in the same direction a carry signal measures.

The backtest builds a rolled series instead. It holds contract 1, switches to contract 2 five trading days before expiry, and chains the daily returns of the contract actually held. Expiry dates follow the NYMEX rule: trading stops 3 business days before the 25th, or 4 if the 25th is not a business day.

Rolling early also avoids the May 2020 contract's negative settlement on 20 April 2020.

### Inventory surprise

The surprise is the weekly change in commercial crude stocks minus the seasonal norm. The seasonal norm is the average change for the same ISO week over the **previous five years only**.

Analyst consensus forecasts are not freely available historically, so this seasonal benchmark stands in for "expected".

### Timing

EIA data covers weeks ending Friday and is released the following Wednesday at 10:30 ET. Positions are set at the first close on or after the release date and held for one week. No data is used before its release.

### Testing rules

- **Sample split.** The exploration period was 2010–2021 at first and was later extended to 1990–2021 (see the research log). The holdout, Jan 2022 – Mar 2024, was run exactly once.
- **Pre-declaration.** Every signal, lookback, strategy variant, decision rule and holdout pass criterion was written down before the corresponding test was run.
- **Significance bar.** Several signals were tested, so a single-signal result needed t ≈ 2.5 or higher on the primary sample, pointing the same way in the 2010–2021 consistency check.
- **Costs.** 0.05% is charged on every entry, exit and monthly roll while in a position. Sensitivity was tested from 0% to 0.20%.
- **Bootstrap.** Strategy and benchmark weekly returns were resampled jointly in 26-week blocks, 10,000 times, to test the Sharpe difference. Blocks preserve the regime persistence (backwardation and contango spells last months) that a plain t-test ignores.

## Research log

Every test that was run, in order.

| # | Test | Sample | Result |
|---|---|---|---|
| 1 | Inventory surprise → next-week return | 2010–21 (unadjusted Yahoo `CL=F`) | No relationship; bucket means zig-zag |
| 2 | Inventory surprise → release-day return | 2010–21 (Yahoo) | Monotonic across buckets; corr −0.088 (≈2 s.e.) |
| 3 | Time-series momentum, 1/3/12-month lookbacks | 2010–21 (Yahoo) | t = 0.75 / −0.48 / 0.21; nothing |
| — | **Data fix:** Yahoo replaced with roll-adjusted EIA contract data; Tests 1–3 rerun | 2010–21 | Release-day corr −0.141; next-week corr −0.043; trend t = 0.64 / 0.33 / −0.31. Buy-and-hold Sharpe fell from 0.21 to 0.00 once roll yield was handled correctly |
| — | **Sample extension**, declared before Test 4: primary sample widened to 1990–2021 for power after Tests 1–3 were null; 2010–21 kept as a consistency check | — | — |
| 1–3 | Rerun on primary sample | 1990–2021 | Next-week corr −0.034; release-day corr −0.138; trend t = 2.23 / 1.57 / 1.05. None clears the bar |
| 4 | Carry: long if contract 1 > contract 2 (backwardation) | 1990–2021 | t = 2.36, Sharpe 0.48 vs 0.18 buy-and-hold. Just below the bar; same direction over 2010–21 (Sharpe 0.46 vs 0.00) |
| 5 | Strategy A (pure carry) vs B (carry + inventory surprise ≤ 0), with costs. Decision rule declared in advance: B goes forward only if its Sharpe beats A's by more than 0.1 | 1990–2021 | A: 0.46. B: 0.59. **B goes forward** |
| 6 | **Holdout**, run once, pass criteria declared in advance: Sharpe > 0 and max drawdown no worse than −26.9% | 2022-01 → 2024-03 | Sharpe −0.05, max drawdown −28.8%. **Fails both criteria** |

## Results

### In-sample, 1990–2021, after 0.05% costs

| | A: Pure carry | B: Carry + inventory | Buy-and-hold |
|---|---|---|---|
| Sharpe | 0.46 | **0.59** | 0.17 |
| Annual return | 7.63% | 8.05% | −0.98% |
| Annual volatility | 20.4% | 14.9% | 37.9% |
| Max drawdown | −49.7% | −26.9% | −98.4% |
| Longest drawdown | 677 weeks | 267 weeks | 703 weeks |
| % of weeks long | 39.0% | 20.7% | 100% |
| Hit rate (weeks long) | 56.9% | 61.1% | — |
| Trades | 136 | 358 | — |

**Cost sensitivity (Sharpe), by cost per trade:**

| Cost per trade | 0% | 0.05% | 0.10% | 0.20% |
|---|---|---|---|---|
| A: Pure carry | 0.49 | 0.46 | 0.44 | 0.40 |
| B: Carry + inventory | 0.64 | 0.59 | 0.55 | 0.46 |

**Block bootstrap, Sharpe difference vs buy-and-hold:**

| Strategy | Sharpe difference | p-value (one-sided) | 95% CI (two-sided) |
|---|---|---|---|
| A | +0.30 | 0.037 | [−0.03, 0.59] |
| B | +0.43 | 0.026 | [−0.01, 0.79] |

Both results are significant at 5% one-sided but borderline two-sided. Adjusting B's p-value for the two variants tested gives about 0.05. That is moderate evidence, not proof.

Buy-and-hold has a positive Sharpe but a negative compounded return. This is volatility drag: at roughly 38% annual volatility, compounding outweighs the small positive mean weekly return.

### Holdout, Jan 2022 – Mar 2024 (116 weeks)

| | B: Carry + inventory | Buy-and-hold |
|---|---|---|
| Sharpe | −0.05 | 0.42 |
| Annual return | −4.21% | 9.28% |
| Max drawdown | −28.8% | −40.7% |
| % of weeks long | 38.8% | 100% |

## Post-mortem

*Hypothesis, to be confirmed with a chart of positions against price over 2022–24.*

The strategy was long 39% of holdout weeks versus 21% in-sample. The curve stayed heavily backwardated through 2022 after Russia's invasion of Ukraine, while crude fell through much of the second half of the year.

This is the classic carry failure mode: backwardation signals a tight market, but it does not stop the price falling.

Over 2.3 years a Sharpe ratio has a standard error of roughly ±0.65. The holdout therefore cannot distinguish "no real edge" from "real edge, bad period". Under the pre-declared rules it is reported as a fail.

## Limitations

- **Single market.** The literature's evidence for carry and momentum comes from diversified multi-commodity portfolios. One market over 32 years has limited power.
- **Short holdout.** EIA futures data ends in April 2024, which limits the holdout to 2.3 years.
- **Release timing.** Releases are assumed to fall on Wednesday. Holiday weeks, when EIA publishes on Thursday, are not adjusted for.
- **Expiry dates.** These are derived from the NYMEX rule using the EIA trading calendar, not an official expiry list.
- **Same-close signals.** Carry is measured at the same close the trade is placed. In practice it would be observed shortly before the close.
- **Risk-free rate.** Sharpe ratios exclude it. This matters little for 2010–21 but slightly flatters earlier decades.
- **Proxy for expectations.** The seasonal norm stands in for analyst consensus, which is what the market actually reacts to.

## How to run

1. Get a free EIA API key at https://www.eia.gov/opendata/register.php.
2. Open the notebook in Google Colab using the badge above.
3. Add the key as a Colab secret named `EIA_KEY` (key icon in the left sidebar) and enable notebook access.
4. Runtime → Run all.

## Repository

```
├── README.md
└── wti_inventory_carry_backtest.ipynb
```
