---
title: "Yield Curve Analysis Tool"
date: 2025-09-21 10:00:00 +0800
categories: [Projects, Finance]
tags: [yield curve, fixed income, data science, python]
description: "A Python framework that monitors yield curve inversions,
  credit spreads, and a logit-based recession probability signal, with a Streamlit dashboard
  and daily automated alerting."
math: true
---

The yield curve encodes the market's collective belief about growth, inflation, and monetary policy in a single snapshot. When it inverts, history says pay attention. This post walks through the design and implementation of YieldCurveWatch (YCW) — a Python framework that monitors the curve, tracks credit stress, and fires calibrated recession alerts automatically.

---

## Part 1: Why Yield Curves Matter

Under normal conditions, longer maturities carry more uncertainty, so lenders demand a premium — giving an upward-sloping curve. When that slope inverts (short rates exceed long rates), it signals either that the market expects the Fed to cut rates ahead, or that investors are piling into long-duration bonds as a recession hedge.

The empirical record is hard to ignore. Every U.S. recession since the 1960s was preceded by an inversion, typically 6–18 months ahead. The 10Y–3M spread, which the Fed itself monitors, has an even cleaner track record than the popular 10Y–2Y according to Estrella and Mishkin.

Credit spreads corroborate the signal. When high-yield OAS widens sharply or Baa spreads over Treasuries approach stress levels, markets are pricing in higher default risk. An inverted curve *plus* stressed credit is the classic hallmark of a system under genuine macro pressure.

---

## Part 2: Conceptual Background

### The Term Structure

The term structure encodes three factors: **level** (where are rates broadly?), **slope** (is the curve steep or inverted?), and **curvature** (is the belly rich or cheap relative to the wings?). YCW focuses on slope via two pairs:

- **10Y − 2Y:** The benchmark. Slow-moving, captures the medium-term growth outlook.
- **10Y − 3M:** The Fed's preferred signal. More reactive to policy and historically cleaner.

### Nelson-Siegel (Background)

The Nelson-Siegel model fits a smooth curve through observed tenors using three interpretable factors:

$$y(\tau) = \beta_0 + \beta_1 \cdot \frac{1 - e^{-\lambda\tau}}{\lambda\tau} + \beta_2 \cdot \left[\frac{1 - e^{-\lambda\tau}}{\lambda\tau} - e^{-\lambda\tau}\right]$$

YCW uses raw CMT spreads rather than a parametric fit today, but the framework is a natural extension — the raw 10Y–2Y slope is essentially a noisy proxy for β₁.

### Credit Spreads

YCW monitors two credit series:

- **Baa − 10Y:** Wide spreads (>250 bps) signal elevated investment-grade default or liquidity risk.
- **HY OAS:** Sustained readings above 500 bps have historically coincided with systemic credit stress.

---

## Part 3: System Architecture

YCW uses a **Registry/Plugin pattern** with four separated layers, chosen to enforce clean boundaries and allow extensions without touching core orchestration code.

```
YAML Config → Registry → Pipeline
                              ↓
                     [1] Fetch Data      (USFredFetcher — 11 CMT tenors via FRED)
                     [2] Indicators      (YieldCurveIndicators, USCreditIndicators)
                     [3] Signals         (CompositeSignal, LogitRecessionSignal)
                     [4] Notify          (ConsoleNotifier, SlackWebhookNotifier)
```

### Data Ingestion

The `USFredFetcher` pulls 11 Treasury CMT series (1M through 30Y) into a daily DataFrame over a configurable lookback window. A shared `FredCache` prevents redundant API calls — the credit indicator reuses the DGS10 column already fetched by the curve fetcher, injected automatically via `inspect.signature`. Missing values are forward-filled up to 10 consecutive days; beyond that, `NaN` is preserved rather than silently propagated.

### Indicators

Indicators transform raw rates into scalar features passed to the signal layer. `YieldCurveIndicators` produces spread levels, inversion flags, a 5-day net 10Y move, and a 200-day MA crossover. `USCreditIndicators` adds Baa−10Y spread, HY OAS level, and its month-over-month change. Signals never see raw DataFrames — only the computed feature dictionary. This boundary makes each layer independently testable.

### Signal Engine

**`CompositeSignal`** applies configurable threshold rules: inversion → `warning`; HY OAS > 500 bps → `watch`; large 10Y move or MA crossover → `watch`.

**`LogitRecessionSignal`** trains a logistic regression on 1962-to-present NBER recession data and outputs a 12-month recession probability based on the current 10Y–3M spread. The output maps to signal levels:

| Probability | Level   |
| ----------- | ------- |
| < 30%       | Info    |
| 30%–50%     | Watch   |
| ≥ 50%       | Warning |

A `SignalHistory` module deduplicates alerts within a configurable TTL (default: 7 days), so a sustained inversion doesn't flood Slack with daily duplicates.

### Dashboard and Scheduler

The Streamlit dashboard calls the same pipeline functions as the CLI, plots the curve with Altair, and renders signals as colored banners. A lightweight daily scheduler runs the pipeline at a configured time with exponential-backoff retry — no Airflow required.

---

## Part 4: Key Implementation Details

### Avoiding Naive MoM Calculations

`rolling(21).diff()` conflates calendar months with 21-day windows. YCW resamples to month-end instead:

```python
hy_monthly = mat["HY_OAS"].resample("ME").last()
hy_mom_monthly = hy_monthly.diff(1)
```

### Preventing Look-Ahead Bias in Training

NBER recession dates are published 6–18 months late. Including the trailing horizon in training silently labels an unannounced recession as "no recession," suppressing the probability estimate exactly when stress is highest. YCW excludes the last `horizon_months` from training:

```python
training_cutoff = s.index[-1] - pd.DateOffset(months=self.h)
for t, val in s.items():
    if t > training_cutoff:
        continue
    future_vals = rec.reindex([t + pd.DateOffset(months=self.h)])
    if future_vals.isna().all():
        continue
    X.append([val])
    y.append(int(future_vals.iloc[0] > 0))
```

### Walk-Forward Backtesting

The backtest uses an expanding-window approach: at each month *t*, a fresh model is trained only on months strictly before *t* with finalized labels, then predicts at *t*. Evaluation is restricted to months where a full 12-month forward window has elapsed, producing a clean classification report with no look-ahead contamination.

### Dependency Injection via `inspect`

The `Registry` stores plugin *classes*, not instances. The pipeline instantiates them at runtime, injecting the shared `FredCache` only when the class signature declares it:

```python
def _make_with_cache(cls, cache):
    if cache is not None and "cache" in inspect.signature(cls).parameters:
        return cls(cache=cache)
    return cls()
```

New plugins opt into caching simply by adding `cache=None` to their `__init__` — no pipeline changes needed.

---

## Part 5: Results and Historical Insights

The logit model produces strong signals at historical stress points:

- **Early 1980s:** Volcker-era inversions exceeded −300 bps on 10Y–3M; the model produces >80% recession probabilities consistent with the sharp contractions that followed.
- **2000–2001:** The curve inverted in mid-1999, well ahead of the March 2001 peak. HY OAS widened sharply through 2002, with the composite signal firing watch alerts across both channels.
- **2008:** The inversion began in late 2006; the logit probability crossed 50% in early 2007. HY OAS then moved from ~300 bps to over 1,800 bps at peak crisis — both signal layers firing simultaneously.
- **2022–2023:** The deepest inversion since the 1980s (>−100 bps on 10Y–2Y), yet no formal recession followed through early 2026 — the clearest example of the model's structural limitations.

---

## Part 6: Limitations and Model Risks

- **False positives in post-QE regimes.** Term premium suppression from QE compresses the long end independent of growth expectations, distorting the inversion signal. The model, trained on pre-QE history, cannot distinguish the two.
- **Single-factor model.** One feature (10Y–3M spread) ignores unemployment, ISM, lending standards, and forward rate expectations. Richer inputs would improve precision at the cost of overfitting risk.
- **NBER vintage bias.** Training on the current vintage of FRED data uses revised recession dates that wouldn't have been available in real time.
- **Non-stationarity.** A single logit trained across six decades treats the Volcker era as comparable to secular stagnation — a strong and probably wrong assumption.
- **Static thresholds.** HY OAS > 500 bps and Baa−10Y > 250 bps are fixed. Regime-conditional recalibration would be more robust.

---

## Part 7: Future Extensions

- **Term premium decomposition.** Use the NY Fed's ACM or Kim-Wright model (both on FRED) to separate the expectations component from the term premium. A logit trained on the expectations-only slope would be far more robust to QE distortions.
- **Regime modeling.** A Markov-switching regression could learn regime-specific slope-to-recession relationships, rather than pooling across structurally different monetary eras.
- **Multi-economy support.** The architecture already supports multiple economies via the `economies` config list. Adding ECB, Gilt, and JGB fetchers is a natural extension with no core changes needed.
- **Portfolio integration.** Publish signal outputs to a message bus (Kafka, Redis) for real-time position sizing or hedging overlays; or load historical signals into vectorbt for tactical allocation backtesting.
- **Nelson-Siegel fitting.** Fitting NS across all 11 tenors produces explicit level, slope, and curvature factors as model inputs — improving interpretability and potentially predictive power.
- **Probability calibration.** Apply isotonic regression or Platt scaling post-hoc so that a "30% probability" reflects a true 30% historical frequency of recession onset.

---

## Closing Thoughts

The interesting challenge in building YCW wasn't calling the FRED API — it was asking "what does a −150 bps 10Y–3M spread mean *right now*, given the monetary policy context?" The registry/plugin architecture keeps the system extensible as that question evolves. The logit model is deliberately humble: a one-feature summary of 60 years of history whose value lies in anchoring discussion with a calibrated number, not replacing judgment with a forecast.

The yield curve is the market's best guess about the future encoded in present prices. Building a system that listens carefully — and speaks clearly when it changes — is a worthwhile exercise regardless of how any particular episode resolves.

---

*Stack: Python 3.10+, pandas, scikit-learn, Streamlit, Altair, FRED API*
