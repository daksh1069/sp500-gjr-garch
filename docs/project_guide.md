# S&P 500 GJR-GARCH: Project Guide

FIM 500 (NC State MFM) team project: **Forecasting S&P 500 Conditional Volatility with a GJR-GARCH Model**.

This guide covers what is in the repo, how the analysis works, the results, and how to rerun it.

**Status (2026-10-13):** data stage, prepared for the midpoint check-in. The dataset is built and validated, and the pre-modeling diagnostics are done. The final GJR-GARCH model has not been fit yet.

---

## Where to start

| If you want to... | Open |
|---|---|
| Get the numbers for the Data slide | [`Presentations/SP500_data_slide_stats.ipynb`](../Presentations/SP500_data_slide_stats.ipynb), §3, or [Results](#results) below |
| See why no days are removed | The same notebook, §2.5 and §2.6, or [Finding 1](#1-no-days-are-removed) |
| See the original dataset notebook | [`Presentations/S&P500_datacleaned_Parin Sharda.ipynb`](../Presentations/S&P500_datacleaned_Parin%20Sharda.ipynb) |
| Rerun everything | [Setup and running](#setup-and-running) |
| Read the intuition decks | `Presentations/Volatility_Clustering.pptx`, `Presentations/autocorrelation.pptx` |

---

## Repository layout

```
gjr_garch/
├── README.md                     Overview and headline numbers
├── requirements.txt              Pinned Python packages
├── docs/
│   └── project_guide.md          This file
└── Presentations/
    ├── SP500_data_slide_stats.ipynb             The analysis: data, validation, summary statistics
    ├── S&P500_datacleaned_Parin Sharda.ipynb    Original dataset notebook (Parin Sharda)
    ├── Volatility_Clustering.pptx               Explainer deck: volatility and volatility clustering
    └── autocorrelation.pptx                     Explainer deck: autocorrelation in regression residuals
```

All analysis code is in **one notebook**, `SP500_data_slide_stats.ipynb`. Parin's notebook is kept unchanged as the original record of the dataset. It applies an 8σ outlier screen, which the analysis notebook replaces (see [Finding 1](#1-no-days-are-removed)).

**Kept locally but not in the repo** (see `.gitignore`):

| Path | Why it's not committed |
|---|---|
| `.venv/` | Local virtual environment. Rebuild it from `requirements.txt`. |
| `Literature Review/` | Third-party papers (e.g. Engle, *GARCH 101*) and course material that we can't redistribute. |
| `Presentations/GJR Garch Mid Point Checkin.pptx` | The working midpoint deck. It includes team members' contact details. |
| `Presentations/Mid Point Checkin.pptx` | A deck from a different project (local volatility models), used as a template. |

---

## How the analysis notebook works

`SP500_data_slide_stats.ipynb` runs top to bottom and needs internet access, because it downloads from Yahoo Finance and FRED.

| Section | What it does |
|---|---|
| Setup | Imports; settings (ticker, window, Ljung-Box lags, 8σ threshold, EWMA λ, spike-check thresholds); helper functions `log_returns`, `summarize` (computes every statistic), and `slide_column` (formats them for the slide) |
| 1. Data | Downloads ^GSPC daily prices for 2006-01-01 to 2026-09-20 (end date exclusive, so the last close is Fri 2026-09-18) and computes daily log returns $r_t = \ln(P_t / P_{t-1})$. It stops with an error unless there are 5,210 closes and 5,209 returns. |
| 2.1 Trading calendar | Compares Yahoo's dates with the NYSE calendar, including special closures (`pandas_market_calendars`) |
| 2.2 Integrity | Counts missing values, duplicate dates, and non-positive closes. Checks whether unchanged closes are stale by looking at that day's intraday range. |
| 2.3 Spike-and-revert | Flags moves over 3% that are more than 90% undone the next day, the usual sign of a bad tick |
| 2.4 FRED cross-check | Compares Yahoo closes with the official S&P Dow Jones Indices closes on FRED (last 10 years). Then substitutes the official closes and lists every slide value that changes. |
| 2.5 Original 8σ screen | Reproduces Parin's screen (\|z\| > 8 using the full-sample mean and std): 4 flagged days, 5,205 kept. Shows each flagged day's previous and next close, and its z-score against the full sample and against the EWMA volatility at the time (λ = 0.94). |
| 2.6 GJR-GARCH robustness | Fits GJR-GARCH(1,1,1) (constant mean, skewed-t, returns in %) with and without the 4 days. Compares the parameters and the model's volatility on the day after each removed day. |
| 3. Summary statistics | The slide table. Columns: full sample (used) and 8σ-cleaned (robustness). |

**Estimator choices:**
- **Standard deviation:** the sample standard deviation (`ddof=1`).
- **Annualized vol:** daily std × √252.
- **Skewness and excess kurtosis:** pandas' bias-corrected `.skew()` and `.kurt()`.
- **Jarque-Bera:** `scipy.stats.jarque_bera`.
- **ADF:** `adfuller(autolag="AIC")`, constant only.
- **Ljung-Box:** `acorr_ljungbox` on $r_t$ and on $r_t^2$ (the ARCH-effects check).
- **Adjusted prices:** `yfinance` adjusts prices by default, but for an index `Close` equals `Adj Close` (checked: max difference 0.0). So "Adjusted Close" on the slide is accurate.

---

## Results

From the run on 2026-10-13. Yahoo Finance data for 2006-01-03 to 2026-09-18.

### Summary statistics (notebook §3)

| Statistic | Full sample (used) | 8σ-cleaned (robustness) |
|---|---|---|
| Price dates | 2006-01-03 to 2026-09-18 | 2006-01-03 to 2026-09-18 |
| Days removed | 0 | 4 |
| Return dates | 2006-01-04 to 2026-09-18 | 2006-01-04 to 2026-09-18 |
| Observations (daily log returns) | 5,209 | 5,205 |
| Mean daily log return (%) | 0.0345 | 0.0348 |
| Daily std (%) | 1.218 | 1.180 |
| Annualized vol (%) | 19.34 | 18.73 |
| Min daily return (%) | −12.77 (2020-03-16) | −9.47 (2008-10-15) |
| Max daily return (%) | 10.96 (2008-10-13) | 9.09 (2025-04-09) |
| Skewness | −0.475 | −0.438 |
| Excess kurtosis | 12.95 | 9.00 |
| Jarque-Bera | 36,523 (p < 0.001) | 17,679 (p < 0.001) |
| ADF stat (AIC lag selection) | −16.22 (p < 0.001) | −15.24 (p < 0.001) |
| ADF lags used / 1% critical value | 19 / −3.432 | 19 / −3.432 |
| Ljung-Box Q(10), returns | 119.0 (p < 0.001) | 85.4 (p < 0.001) |
| Ljung-Box Q(20), returns | 176.8 (p < 0.001) | 120.2 (p < 0.001) |
| Ljung-Box Q(10), squared returns | 5,054.7 (p < 0.001) | 4,727.9 (p < 0.001) |
| Ljung-Box Q(20), squared returns | 7,100.0 (p < 0.001) | 6,943.7 (p < 0.001) |

### Data validation (notebook §2)

| Check | Result |
|---|---|
| Trading calendar | All 5,210 dates match NYSE trading days exactly. None missing, none extra. |
| Integrity | No missing values, duplicates, or non-positive closes. Two closes were unchanged from the day before (2008-01-03, 2017-01-10), but both had intraday ranges of 13–14 points, so they are genuine. |
| Spike-and-revert | 7 candidates, all inside known crises (Sept 2008, Jan 2009, Aug 2011, Mar 2020). None is an isolated spike in a calm market. |
| FRED cross-check (2016-10-13 onward) | 2,496 common days. 8 closes differ by more than 0.01 points. Two are real Yahoo errors: 2019-08-12 (−1.05 pts) and 2021-08-11 (−5.29 pts). Substituting the official closes changes only Jarque-Bera (36,523 → 36,527) and Ljung-Box Q(20) on squared returns (7,100.0 → 7,100.1). The Yahoo series is used as is. |

### The four days flagged by the original 8σ screen (notebook §2.5)

| Date | Prev close | Close | Next close | Return (%) | z (full sample) | Vol going in (%) | z (conditional) |
|---|---|---|---|---|---|---|---|
| 2008-10-13 | 899.22 | 1003.35 | 998.01 | 10.96 | 8.96 | 3.72 | 2.94 |
| 2008-10-28 | 848.92 | 940.51 | 930.09 | 10.25 | 8.38 | 4.44 | 2.31 |
| 2020-03-12 | 2741.38 | 2480.64 | 2711.02 | −9.99 | −8.23 | 3.21 | −3.12 |
| 2020-03-16 | 2711.02 | 2386.13 | 2529.19 | −12.77 | −10.51 | 4.41 | −2.89 |

### GJR-GARCH(1,1,1), skewed-t: with and without the four days (notebook §2.6)

| Parameter | Full sample | 8σ-cleaned |
|---|---|---|
| ω | 0.0233 | 0.0227 |
| α | 0.0000 | 0.0000 |
| γ (t-stat) | 0.2432 (8.76) | 0.2397 (8.53) |
| β | 0.8626 | 0.8647 |
| Persistence α + γ/2 + β | 0.9842 | 0.9845 |
| Long-run annualized vol (%) | 19.28 | 19.22 |
| Skewed-t dof η / skew λ | 6.43 / −0.162 | 6.46 / −0.161 |

| Day after removed day | Model vol, full sample (%) | Model vol, 8σ-cleaned (%) | Actual \|return\| (%) |
|---|---|---|---|
| 2008-10-14 | 5.15 | 5.53 | 0.53 |
| 2008-10-29 | 4.72 | 5.18 | 1.11 |
| 2020-03-13 | 6.67 | 4.79 | 8.88 |
| 2020-03-17 | 8.54 | 4.46 | 5.82 |

---

## Findings

### 1. No days are removed

Parin's notebook dropped returns with |z| > 8 to catch data-entry errors, while intending to keep the 2008 and 2020 crashes. In practice the screen flagged exactly those crash days:

- **2008-10-13:** rally after the G7 and European bank-rescue announcements
- **2008-10-28:** rally during the October 2008 crisis
- **2020-03-12:** COVID crash; market-wide circuit breaker
- **2020-03-16:** COVID crash; circuit breaker after the Fed's emergency cut to zero

The screen measures each day against average volatility over 20 years. Against the volatility at the time, each of these is only a 2–3σ move. Removing them barely changes the GJR-GARCH parameters, because the skewed-t distribution already handles fat tails. It does distort the volatility path: in March 2020 the cleaned fit's next-day volatility is 28–48% lower, because the deleted shocks never enter the variance recursion.

The checks that target actual data errors (calendar, integrity, spike-and-revert, official closes) found nothing that needs removing. **The dataset is all 5,209 returns.** The 8σ-cleaned numbers are reported as a robustness check: "results are robust to excluding the four largest moves."

### 2. Strong ARCH effects, so a GARCH-type model is appropriate

Ljung-Box statistics on squared returns are 5,055 at lag 10 and 7,100 at lag 20 (p < 0.001), about 40 times the statistics on raw returns. The ADF statistic is −16.22 against a 1% critical value of −3.432, so returns are stationary. Excess kurtosis of 12.95 and negative skewness (−0.475) support a skewed-t innovation distribution.

### 3. Returns themselves are serially correlated

Ljung-Box Q(10) on returns is 119.0 (p < 0.001). When fitting the model, compare a constant-mean model with an AR(1) mean equation, and check the residual autocorrelation afterward.

### 4. Early sign of a leverage effect

In the robustness fit, α (the response to positive shocks) is 0.000 and γ (the extra response to negative shocks) is 0.243 with a t-statistic of 8.8. Volatility rises almost entirely after negative shocks. This is preliminary: it's a constant-mean model that hasn't been validated.

### 5. Keep the slides consistent with this sample

- **Intro slide:** the working midpoint deck's intro slide quotes μ = 0.024% and σ = 1.222% for "~25 years". This dataset covers about 20.7 years, with mean 0.0345% and std 1.218%. Update the intro slide to match.
- **Data slide:** its placeholders can be filled in from the full-sample column above.

---

## Setup and running

Tested with Python 3.11.0 on macOS. Package versions are pinned in [`requirements.txt`](../requirements.txt).

```bash
cd gjr_garch
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Register the venv as a Jupyter kernel. The analysis notebook's metadata points to this kernel name.
python -m ipykernel install --sys-prefix --name gjr-garch-venv --display-name "Python 3.11 (gjr_garch .venv)"
```

**Run interactively:** open the notebook in VS Code or JupyterLab and choose the **Python 3.11 (gjr_garch .venv)** kernel. In VS Code, you can also choose the `.venv` interpreter.

**Run from the command line** (runs every cell and saves the outputs back into the file):

```bash
.venv/bin/jupyter nbconvert --to notebook --execute --inplace Presentations/SP500_data_slide_stats.ipynb
```

### Reproducibility notes

- **Live data.** The notebook downloads prices each time it runs. If Yahoo revises history, the observation counts could change. The notebook checks for 5,210 closes, 5,209 returns, and 4 flagged days, and stops with an error if any differ.
- **FRED's rolling window.** FRED only provides the most recent 10 years, so the start of the overlap in §2.4 moves forward over time. The 2019 and 2021 mismatches stay inside the window until 2029.
- **statsmodels 0.16.** statsmodels 0.15 warns that `adfuller` will return a result object instead of a tuple in 0.16. `summarize()` silences that warning and takes the first five tuple elements. After upgrading past 0.16, pass `result_object=False`.

---

## Work log

**2026-09-20 (Parin Sharda):** built the dataset notebook: download, log returns, 8σ screen, plots.

**2026-10-13 (Daksh Kumar):**
1. **Environment:** created the project venv at `.venv` (Python 3.11.0 via pyenv) and registered it as the `gjr-garch-venv` Jupyter kernel.
2. **Slide statistics:** built `SP500_data_slide_stats.ipynb`, which reproduces Parin's pipeline exactly (5,209 → 4 flagged → 5,205) and computes the Data slide statistics.
3. **Outlier check:** found that the 4 flagged days are real 2008 and 2020 crash days, not data errors.
4. **Validation:** added checks that target actual errors (trading calendar, integrity, spike-and-revert, official closes from FRED). The data is clean.
5. **GJR-GARCH check:** fit GJR-GARCH with and without the 4 days. The parameters barely change, but deleting the days distorts the conditional volatility.
6. **Data decision:** switched to the full sample (5,209 returns), with the 8σ-cleaned series as a robustness check. Consolidated all analysis into the one notebook and removed the superseded draft notebook (`S&P500_data_stats_for_slides.ipynb`). It's still in git history.

## Next steps

- Fit GJR-GARCH(1,1,1) on the full sample with Student-t and skewed-t innovations, and compare them by AIC. Test a constant mean against an AR(1) mean.
- Validate the standardized residuals (Ljung-Box on residuals and squared residuals).
- Forecast conditional variance out of sample and evaluate the forecasts.
