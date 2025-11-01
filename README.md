# Systematic Equity Trading from Volatility Surface Skew

**Authors:** Julius Gruber & Karan Jeswani
**Course:** MTH 9897 — Systematic Trading
**Date:** October 2024

---

## 🎯 Project Overview

This project investigates whether **volatility surface skew**—derived from single-stock option prices—can be used to **predict short-term equity returns**.
We build a complete research and trading pipeline: from option-data processing to skew computation, signal generation, portfolio construction, and mean-variance optimization.

Our findings show that **changes in skew contain predictive information**, especially on the **short side**, confirming the hypothesis that **option-implied skewness reflects investor sentiment and misvaluation**.

---

## 📂 Repository Structure

| File                                     | Description                                                                                                                               |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **`filesplitup.ipynb`**                  | Data preparation using `polars`: loads the multi-ticker OptionMetrics file, cleans it, and writes one CSV per ticker.                     |
| **`spconstituentsquery.ipynb`**          | Retrieves the S&P 500 constituents from CRSP to align the stock universe.                                                                 |
| **`SkewGeneration.ipynb`**               | Core skew-feature script: constructs 30-day volatility smiles via SABR calibration and computes 25Δ Call – 25Δ Put skews.                 |
| **`SystematicTrading.ipynb`**            | Implements the **directional trading strategy** on equities based on daily changes in skew.                                               |
| **`SystematicTrading-LedoitWolf.ipynb`** | Extends the baseline by performing **mean-variance optimization** with both empirical and **Ledoit–Wolf shrinkage** covariance estimates. |
| **`SysTrading-2.pdf`**                   | Presentation summarizing the motivation, methodology, and quantitative backtesting results.                                               |
| **`requirements.txt`**                   | Full environment specification for reproducibility.                                                                                       |

---

## 🧠 Research Motivation

Previous academic work (e.g. *Rehman & Vilkov, “Risk-Neutral Skewness: Return Predictability and Its Sources”*) and the reference paper *“Implied Skewness and Stock Returns”* find that model-free implied skewness (MFIS) predicts future returns.

Our research replicates and extends these findings using a **daily, delta-based skew measure** instead of the full MFIS computation.

Key economic intuition:

* Option skew reflects **investor demand for downside protection**.
* When skew **flattens**, traders are shedding protection → **bullish signal**.
* When skew **steepens**, traders seek protection → **bearish signal**.

---

## ⚙️ Methodology

### 1. Data Pipeline

| Source               | Description                                      |
| -------------------- | ------------------------------------------------ |
| **Equities**         | CRSP daily prices for S&P 500 constituents.      |
| **Options**          | OptionMetrics implied vols, multiple maturities. |
| **Processing Tools** | `pandas`, `numpy`, `sklearn`, `polars`.          |

Workflow:

1. Use `filesplitup.ipynb` to separate the raw multi-GB option dataset.
2. Filter expiries around 30 days.
3. Calibrate a SABR model to each expiry.
4. Interpolate cumulative variance to obtain a 30-day smile.
5. Compute:
   [
   \text{Skew} = IV_{25\Delta,\text{Call}} - IV_{25\Delta,\text{Put}}
   ]

---

### 2. Signal Construction

We define the daily **change in skew**:

[
\Delta\text{Skew}_t = \text{Skew}*t - \text{Skew}*{t-1}
]

Standardize it via rolling z-scores, then map to trading actions:

| Condition | Interpretation                      | Position |
| --------- | ----------------------------------- | -------- |
| ΔSkew > 0 | Skew steepens → protection demand ↑ | Short    |
| ΔSkew < 0 | Skew flattens → optimism ↑          | Long     |

Signals are held 1–5 days and neutralized to the market or industry level.
Exposure clipping (winsorization) reduces the influence of outliers.

---

### 3. Portfolio Construction

1. **Universe:** S&P 500 stocks with valid option data.
2. **Neutralization:** Market- and industry-neutral versions tested; industry neutralization under-performed.
3. **Weighting:** Directly proportional to standardized factor exposures.
4. **Rebalancing:** Daily with transaction-cost assumptions.
5. **Evaluation Metrics:** Expected return E, standard deviation σ, Sharpe ratio, hit rate.

---

## 📊 Empirical Results (From SysTrading-2 Presentation)

| Metric                  | Value                               |
| ----------------------- | ----------------------------------- |
| **Expected Return (E)** | 12.02 %                             |
| **Volatility (σ)**      | 18.45 %                             |
| **Sharpe Ratio**        | 0.65 overall (> 1 in several years) |

### Yearly Performance Highlights

| Year        | Return                                                   | Volatility | Sharpe |
| ----------- | -------------------------------------------------------- | ---------- | ------ |
| 2011        | 74.9 %                                                   | 17 %       | 4.3    |
| 2014        | 157.7 %                                                  | 22 %       | 6.96   |
| 2017        | 42.5 %                                                   | 12 %       | 3.37   |
| 2010 & 2016 | Negative Sharpe due to macro volatility and signal noise |            |        |

### Decile Analysis (γ = 21)

* Strong monotonic relationship between skew deciles and returns.
* Top deciles (8–10) significantly outperformed bottom deciles, with t-stats > 10 in later periods.

### Portfolio Distribution

| Statistic | Value  |
| --------- | ------ |
| Skewness  | −0.294 |
| Kurtosis  | 7.11   |
| Hit Rate  | 55.7 % |

> **Observation:** Predictive power strong on the short side (skew steepening → negative returns); long signals weaker but add diversification.

---

## 🧮 Optimization Extensions ( `SystematicTrading-LedoitWolf.ipynb` )

To enhance risk-adjusted performance, we applied **mean-variance optimization** using the daily signal returns:

| Method                | Description                           | Outcome                           |
| --------------------- | ------------------------------------- | --------------------------------- |
| Empirical Covariance  | Classic in-sample Σ                   | Baseline                          |
| Ledoit–Wolf Shrinkage | Well-conditioned covariance estimator | Smoother weights, similar returns |

* Both methods produced nearly identical efficient frontiers.
* **Max-Sharpe portfolios** were implemented but computationally intensive (non-convex).
* The optimization confirmed that the skew signal adds incremental value when combined cross-sectionally.

---

## 🧩 Conclusions & Key Insights

1. Volatility surface skews contain **forward-looking information** useful for predicting equity returns.
2. The effect is **asymmetric** — stronger for short positions.
3. **Industry neutralization** reduces signal efficacy due to cross-sector skew structure.
4. **Ledoit–Wolf** covariance stabilizes portfolio weights without hurting performance.
5. **Daily skew dynamics** can serve as input to broader systematic risk-sentiment frameworks.

---

## 🚀 Next Steps

* Implement faster covariance estimation for full-period runs.
* Analyze shorter-dated skews (< 15 days) to capture market-maker hedging activity.
* Integrate term-structure features and realized volatility filters.
* Express signals via option spreads instead of underlying stocks for delta-hedged strategies.

---

## 🧱 Environment Setup

```bash
conda create -n volskew python=3.11
conda activate volskew
pip install -r requirements.txt
```


---

## 📘 References

* Rehman & Vilkov (2021) *Risk-Neutral Skewness: Return Predictability and Its Sources*
* “Implied Skewness and Stock Returns” (Paper replicated in our study)
* Ledoit & Wolf (2003) *A Well-Conditioned Estimator for Large-Dimensional Covariance Matrices*
* OptionMetrics & CRSP data feeds


