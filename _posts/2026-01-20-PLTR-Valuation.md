---
title: "What I Learned Valuing Palantir for a School Finance Paper"
date: 2026-01-20
categories: [Finance, Investing, Valuation]
tags: [palantir, dcf, python, equity research]
description: A breakdown of my school equity research project valuing Palantir Technologies — covering DCF modeling, relative valuation, and sensitivity analysis — and what building the entire pipeline in Python taught me about high-growth tech investing.
---

A few weeks ago I finished a valuation paper on **Palantir Technologies (PLTR)** for school. The assignment was to pick a publicly traded company, dig into the financials, and arrive at a well-reasoned estimate of intrinsic value. I chose Palantir because it sits right at the intersection of two things I find genuinely interesting: AI infrastructure and controversial market valuations. The stock trades at eye-watering multiples, and I wanted to understand whether any of that is defensible.

This post is a writeup of what the paper covered and, more importantly, what I actually learned from building it from the ground up in Python.

---

## What the Paper Is About

The paper is a full-stack equity research writeup covering:

1. **Business model and revenue drivers** — how Palantir makes money across Gotham (government), Foundry (commercial), and AIP (its AI platform layer)
2. **Industry and competitive landscape** — where Palantir sits relative to Snowflake, Datadog, Salesforce, and the big cloud vendors
3. **Historical financial analysis** — revenue CAGR, gross margin trajectory, operating leverage, and free cash flow quality from 2019–2024
4. **Three-scenario forecast** — bear, base, and bull cases projecting revenue growth, operating margins, and FCF margins out to 2029
5. **DCF valuation** — discounting projected FCFs back to fiscal year-end 2024 with a Gordon Growth terminal value
6. **Relative valuation** — EV/Revenue comps against peers pulled live from Yahoo Finance
7. **Sensitivity analysis** — a WACC × terminal growth rate grid across all three scenarios
8. **Risk assessment and investment recommendation**

The headline finding: **Palantir appears materially overvalued relative to fundamentals.** The base-case DCF puts intrinsic value around $28/share, the relative valuation implies roughly $12–$24/share using peer multiples, and the stock was trading around $165 at the time of writing. Even the most optimistic bull-case sensitivity run (6% WACC, 5% terminal growth) only gets to about $73/share. To justify the actual market price you'd need assumptions that stretch well beyond any reasonable scenario.

---

## The Tech Stack

I built the entire analysis pipeline in Python rather than doing it in Excel, which turned out to be one of the better learning decisions of the project. The codebase had a clean separation of concerns:

```
pltr-valuation/
├── data_raw/            # CSVs manually populated from SEC filings
├── data_clean/          # Validated, derived outputs
├── outputs/             # Final tables, sensitivity grids, heatmaps
├── validate_and_clean.py
├── historical_analysis.py
├── forecast_build.py
├── dcf_valuation.py
├── sensitivity_analysis.py
└── relative_valuation.py
```

Each script is a standalone module you run in sequence. No Jupyter notebooks, no hidden state — just plain Python and pandas.

---

## Building the DCF in Python

The core valuation logic is pretty clean. Here's the DCF engine:

```python
def run_dcf(df, scenario):
    df = df[df["scenario"] == scenario].copy()
    df = df.sort_values("fiscal_year").reset_index(drop=True)

    df["year_index"] = np.arange(1, len(df) + 1)
    df["discount_factor"] = 1 / (1 + WACC) ** df["year_index"]
    df["pv_fcf"] = df["fcf"] * df["discount_factor"]

    pv_fcf_sum = df["pv_fcf"].sum()

    terminal_fcf = df.iloc[-1]["fcf"]
    terminal_value = terminal_fcf * (1 + TERMINAL_GROWTH) / (WACC - TERMINAL_GROWTH)
    pv_terminal = terminal_value * df.iloc[-1]["discount_factor"]

    enterprise_value = pv_fcf_sum + pv_terminal
    equity_value = enterprise_value + NET_CASH
    value_per_share = equity_value / DILUTED_SHARES / 1000

    return {
        "scenario": scenario,
        "pv_fcf": pv_fcf_sum,
        "pv_terminal": pv_terminal,
        "enterprise_value": enterprise_value,
        "equity_value": equity_value,
        "value_per_share": value_per_share,
    }
```

Plugging in the base-case forecast (7% WACC, 5% terminal growth, ~$3.7B net cash, 2.3B diluted shares) gave:

| Scenario | PV of FCFs | PV of Terminal Value | Equity Value | Value/Share |
| -------- | ---------- | -------------------- | ------------ | ----------- |
| Bear     | $3.2B      | $38.7B               | $45.6B       | **$19.82**  |
| Base     | $4.3B      | $56.8B               | $64.7B       | **$28.15**  |
| Bull     | $5.4B      | $75.8B               | $84.9B       | **$36.90**  |

One thing that immediately jumps out: in every scenario, the terminal value represents **90%+ of total enterprise value**. That's not unusual for a growth software company, but it forces you to be really honest about your long-run assumptions, because they're driving almost everything.

---

## The Sensitivity Analysis

The most revealing part of the project was the WACC × terminal growth sensitivity grid. I built it as a vectorized loop across all three scenarios:

```python
WACC_GRID = np.array([0.06, 0.07, 0.08, 0.09, 0.10, 0.11])
G_GRID    = np.array([0.02, 0.03, 0.04, 0.05])

def dcf_value_per_share(df, scenario, wacc, g):
    if wacc <= g:
        return np.nan

    sdf = df[df["scenario"] == scenario].copy().sort_values("fiscal_year")
    n = len(sdf)
    discount = 1 / (1 + wacc) ** np.arange(1, n + 1)

    pv_fcf = float((sdf["fcf"].values * discount).sum())
    terminal_value = float(sdf.iloc[-1]["fcf"]) * (1 + g) / (wacc - g)
    pv_terminal = terminal_value * discount[-1]

    equity_value = pv_fcf + pv_terminal + NET_CASH
    return (equity_value / DILUTED_SHARES_M) / UNIT_DIVISOR
```

Here's what the base-case grid looks like across WACC and terminal growth rate `g`:

```
g →        2%     3%     4%     5%
WACC ↓
6%       16.09  20.44  29.15  55.26
7%       13.06  15.57  19.76  28.15
8%       11.04  12.66  15.08  19.12
9%        9.60  10.72  12.27  14.61
10%       8.53   9.33  10.40  11.91
11%       7.69   8.30   9.07  10.11
```

No combination of WACC and terminal growth gets you anywhere close to $165. To even get into the $50–$70 range, you'd need to assume a very low cost of capital *and* a terminal growth rate at or above 5%, which implies Palantir grows in perpetuity at a rate rivaling long-run nominal GDP — a heroic assumption for any company.

---

## Relative Valuation

I used `yfinance` to pull live market data and compute EV/Revenue multiples for a peer group: Snowflake, Datadog, ServiceNow, Salesforce, MongoDB, and C3.ai.

```python
import yfinance as yf

def fetch_snapshot(ticker):
    info = yf.Ticker(ticker).info
    ev = float(info.get("enterpriseValue", 0))
    rev = float(info.get("totalRevenue", 1))
    return {
        "ticker": ticker,
        "ev_to_revenue": ev / rev if rev else None,
        ...
    }
```

The peer group traded at a **median EV/Revenue of ~11.8×**, with a 25th–75th percentile range of roughly **6.6×–13.4×**. Applying those multiples to Palantir's 2024 revenue implied a share price of **$12.71–$24.29**, with a midpoint around $21.55. Palantir's actual EV/Revenue at the time? **~100×**. That's not a small premium — that's a completely different conversation about optionality and AI narrative.

---

## What I Actually Learned

**1. Terminal value assumptions are everything for growth companies.**
When 90% of your DCF value comes from the terminal value, you're not really valuing cash flows — you're valuing your assumptions about the far future, discounted back. This is both the strength and the fragility of DCF for high-growth tech.

**2. The market price is a forecast, not a fact.**
Seeing Palantir trade at 100× revenue while every reasonable DCF scenario puts fair value at $20–$37 was jarring at first. But the market price isn't wrong — it's just embedding different assumptions. The question is whether *those* assumptions are reasonable, and that's where the analysis gets interesting.

**3. Structuring the code matters as much as the finance.**
By keeping each step (clean → forecast → DCF → sensitivity → relative) as a separate, runnable script, I could iterate on assumptions quickly without breaking anything else. That discipline forced clearer thinking about what each step was actually computing.

**4. High gross margins don't automatically justify high valuations.**
Palantir has ~80% gross margins, which is great. But gross margins alone don't tell you what the business is worth — you need to know how much of that gross profit actually survives as free cash flow after SBC, R&D, and sales spend. Palantir's SBC has historically been very high relative to revenue, which dilutes shareholders even when the income statement looks improving.

**5. Relative valuation has real limits for category-defining companies.**
Comparing Palantir to Snowflake and Salesforce on EV/Revenue is useful context, but Palantir's bull case isn't "it trades like its peers" — it's "it becomes the operating system for AI-driven decision-making at a scale none of its peers have achieved." That's a narrative, not a financial statement. Learning to hold both the quantitative model and the qualitative narrative in mind at the same time is a skill in itself.

---

## Final Thoughts

The paper came to a **Hold / Underperform** recommendation: Palantir is a high-quality company with real competitive advantages, but the market price at the time of writing embeds a level of optimism that leaves almost no margin for error. Any stumble in commercial growth, any compression in AI multiples, or any rise in discount rates would hit the stock hard.

Building this from scratch in Python — pulling real SEC data, writing a modular pipeline, generating sensitivity tables and heatmaps programmatically — made the finance much more concrete than working through a pre-built template ever would have. I'd recommend this approach to anyone doing similar work. It's more work upfront, but you understand every number because you had to write the code that produced it.

---

*All data sourced from Palantir SEC filings (10-K, 2019–2024) and Yahoo Finance as of early 2025. This is a school project — not investment advice.*

