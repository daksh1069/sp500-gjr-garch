# S&P 500 GJR-GARCH: Project Guide

FIM 500 (NC State MFM) team project: **Forecasting S&P 500 Conditional Volatility with a GJR-GARCH Model**.

This guide covers what is in the repo, how the data pipeline works, how to rerun it, and the current results for the midpoint check-in.

**Status (2026-10-13):** data stage. The dataset is built and the pre-modeling diagnostics are done. The GJR-GARCH model has not been fit yet.

---

## Where to start

| If you want to... | Open |
|---|---|
| Get the numbers for the Data slide | [`Presentations/SP500_data_slide_stats.ipynb`](../Presentations/SP500_data_slide_stats.ipynb), section 8, or [Results](#results) below |
| See how the dataset is built | [`Presentations/S&P500_datacleaned_Parin Sharda.ipynb`](../Presentations/S&P500_datacleaned_Parin%20Sharda.ipynb) |
| Check whether the dropped outliers are real | `SP500_data_slide_stats.ipynb`, section 2, or [Finding 1](#1-the-8σ-filter-removes-real-crash-days-not-data-errors) |
| Rerun everything on your machine | [Setup and running](#setup-and-running) |
| Read the intuition decks on volatility clustering and autocorrelation | `Presentations/Volatility_Clustering.pptx`, `Presentations/autocorrelation.pptx` |

---

## Repository layout

```
gjr_garch/
├── README.md                          Short overview; links here
├── requirements.txt                   Pinned Python packages
├── docs/
│   └── project_guide.md               This file
└── Presentations/
    ├── S&P500_datacleaned_Parin Sharda.ipynb   Source dataset: download, log returns, 8σ cleaning, plots (Parin Sharda)
    ├── SP500_data_slide_stats.ipynb            Data slide stats: reproduces Parin's pipeline, then runs all diagnostics
    ├── S&P500_data_stats_for_slides.ipynb      Earlier draft (Parin's notebook plus one unexecuted stats cell); superseded
    ├── Volatility_Clustering.pptx              Explainer deck: what volatility and volatility clustering are
    └── autocorrelation.pptx                    Explainer deck: autocorrelation in regression residuals
```

**Kept locally but not in the repo** (see `.gitignore`):

| Path | Why it's not committed |
|---|---|
| `.venv/` | Local virtual environment. Rebuild it from `requirements.txt`. |
| `Literature Review/` | Third-party papers (e.g. Engle, *GARCH 101*) and course material that we can't redistribute. |
| `Presentations/GJR Garch Mid Point Checkin.pptx` | The working midpoint deck. It includes team members' contact details. |
| `Presentations/Mid Point Checkin.pptx` | A deck from a different project (local volatility models), used as a template. |

**Do not edit Parin's notebook.** It defines the dataset. Any analysis that needs the cleaned series should copy its steps exactly, the way `SP500_data_slide_stats.ipynb` does.

---

## How the pipeline works

### Dataset (Parin's notebook)

| Step | Code | Detail |
|---|---|---|
| Download | `yf.download("^GSPC", "2006-01-01", "2026-09-20")['Close']` | Yahoo Finance via `yfinance`. The end date is exclusive, so the last bar is Fri 2026-09-18. `yfinance` auto-adjusts `Close` by default. For an index like ^GSPC, `Close` equals `Adj Close` (checked: max difference 0.0), so "Adjusted Close" on the slide is accurate. |
| Log returns | `np.log(data / data.shift(1)).dropna()` | 5,210 prices give 5,209 daily log returns, from 2006-01-04 to 2026-09-18. |
| Outlier cleaning | `z = (r - r.mean()) / r.std()`, keep `\|z\| < 8` | Uses the full-sample mean and standard deviation. The 8σ cutoff was chosen so the 2008 and 2020 crashes would stay in. In return terms, a day is flagged if its log return is outside [−9.71%, +9.78%]. 4 days are flagged and 5,205 returns remain. |

The notebook also plots the return series with the ±8σ band and a histogram against a normal curve with the same mean and standard deviation.

### Slide stats notebook (`SP500_data_slide_stats.ipynb`)

The notebook copies Parin's code word for word, then adds the following. It stops with an error unless it gets the same result as Parin's run (4 outliers, 5,205 returns).

| Section | What it computes | How |
|---|---|---|
| 1 | Download, log returns, 8σ cleaning | Parin's code, unchanged |
| 2 | The 4 flagged days, with previous, current, and next close, log and simple return, and z-score | Checks whether each day is a data error or a real move |
| 3 | First and last price and return dates; observation counts (raw and cleaned) | |
| 4 | Mean, daily std, annualized vol, min and max with dates, skewness, excess kurtosis, Jarque-Bera | On the **cleaned** series |
| 5 | ADF test: stat, p-value, lags used, critical values | `statsmodels.tsa.stattools.adfuller(r, autolag="AIC")`, constant only (default) |
| 6 | Ljung-Box at lags 10 and 20 on returns **and** squared returns | `statsmodels.stats.diagnostic.acorr_ljungbox`. The squared-returns test is the ARCH-effects check. |
| 7 | Mean, std, skewness, and excess kurtosis on the **uncleaned** series | For comparison with section 4 |
| 8 | One summary DataFrame with everything above, formatted for the slide | Cleaned and uncleaned columns side by side |

**Estimator choices:**
- **Standard deviation:** uses the sample std (`ddof=1`, the pandas default). This matches the std used in Parin's z-scores.
- **Annualized vol:** daily std × √252.
- **Skewness and excess kurtosis:** pandas' bias-corrected `.skew()` and `.kurt()`. `.kurt()` already returns *excess* kurtosis.
- **Jarque-Bera:** `scipy.stats.jarque_bera`.
- **Ljung-Box input:** uses raw squared returns $r_t^2$, not demeaned returns. With a mean of about 0.03% a day, the difference is negligible.

The draft notebook (`S&P500_data_stats_for_slides.ipynb`) uses scipy's biased skewness and kurtosis, so its numbers can differ in the third decimal place. If it's run, it also writes `slide_stats_output.txt` to the working directory, which is gitignored.

---

## Results

From the run on 2026-10-13. Yahoo Finance data for 2006-01-03 to 2026-09-18.

### Flagged outliers (|z| > 8)

| Date | Prev close | Close | Next close | Log ret (%) | Simple ret (%) | z |
|---|---|---|---|---|---|---|
| 2008-10-13 | 899.22 | 1003.35 | 998.01 | 10.96 | 11.58 | 8.96 |
| 2008-10-28 | 848.92 | 940.51 | 930.09 | 10.25 | 10.79 | 8.38 |
| 2020-03-12 | 2741.38 | 2480.64 | 2711.02 | −9.99 | −9.51 | −8.23 |
| 2020-03-16 | 2711.02 | 2386.13 | 2529.19 | −12.77 | −11.98 | −10.51 |

### Summary table

| Statistic | Cleaned (8σ) | Uncleaned |
|---|---|---|
| Price dates | 2006-01-03 to 2026-09-18 | 2006-01-03 to 2026-09-18 |
| Return dates | 2006-01-04 to 2026-09-18 | 2006-01-04 to 2026-09-18 |
| Observations (daily log returns) | 5,205 | 5,209 |
| Outliers removed (\|z\| > 8) | 4 | — |
| Mean daily return (%) | 0.0348 | 0.0345 |
| Daily std (%) | 1.180 | 1.218 |
| Annualized vol (%) | 18.73 | 19.34 |
| Min daily return (%) | −9.47 (2008-10-15) | −12.77 (2020-03-16) |
| Max daily return (%) | 9.09 (2025-04-09) | 10.96 (2008-10-13) |
| Skewness | −0.438 | −0.475 |
| Excess kurtosis | 9.00 | 12.95 |
| Jarque-Bera stat | 17,679 | — |
| Jarque-Bera p-value | < 0.001 | — |
| ADF stat (AIC lag selection) | −15.24 | — |
| ADF p-value | < 0.001 | — |
| ADF lags used | 19 | — |
| Ljung-Box Q(10), returns | 85.4 (p < 0.001) | — |
| Ljung-Box Q(20), returns | 120.2 (p < 0.001) | — |
| Ljung-Box Q(10), squared returns | 4,727.9 (p < 0.001) | — |
| Ljung-Box Q(20), squared returns | 6,943.7 (p < 0.001) | — |

ADF critical values: −3.432 (1%), −2.862 (5%), −2.567 (10%). Jarque-Bera, ADF, and Ljung-Box were run on the cleaned series only.

---

## Findings and open questions

### 1. The 8σ filter removes real crash days, not data errors

The filter was meant to catch data-entry errors while keeping the 2008 and 2020 crashes. All four days it flags are documented market moves. In each case the price stays at its new level the next day instead of snapping back:

- **2008-10-13** (+11.6%): rally after the G7 and European bank-rescue announcements
- **2008-10-28** (+10.8%): rally during the October 2008 crisis
- **2020-03-12** (−9.5%): COVID crash; market-wide circuit breaker
- **2020-03-16** (−12.0%): COVID crash; circuit breaker after the Fed's emergency rate cut to zero

So the filter drops the single largest days of both crises, while the surrounding crisis days stay in. Removing those 4 days lowers excess kurtosis from 12.95 to 9.00 and daily std from 1.218% to 1.180%.

**Open question for the team:** should we keep the filter? One option is to keep all 5,209 returns. A GJR-GARCH model with Student-t or skewed-t innovations is designed to handle tail days like these, and none of the four is a data error. That's what the filter was meant to catch. The other option is to keep the filter and say on the slide which days it removes. Either way, decide before fitting the model.

### 2. Strong ARCH effects, so a GARCH-type model is appropriate

The Ljung-Box statistics on squared returns are 4,728 at lag 10 and 6,944 at lag 20, with p < 0.001. That's 55 to 58 times the statistics on raw returns, which is clear evidence of volatility clustering. The ADF statistic is −15.24 against a 1% critical value of −3.432, so returns are stationary. Negative skewness (−0.44) is consistent with the leverage effect that the GJR term (γ) is meant to capture. High excess kurtosis (9.0) supports fat-tailed innovations (Student-t or skewed-t) over a normal distribution.

### 3. Returns themselves are serially correlated

The Ljung-Box statistic on raw returns is 85.4 at lag 10, with p < 0.001. When fitting the model, compare a constant-mean model with an AR(1) mean equation, and check the residual autocorrelation afterward.

### 4. Keep the slides consistent with this sample

- **Intro slide figures:** the working midpoint deck's intro slide quotes μ = 0.024% and σ = 1.222% for "~25 years". This dataset covers about 20.7 years (2006-01-04 to 2026-09-18), with mean 0.0348% and std 1.180% cleaned (0.0345% and 1.218% uncleaned). Update the intro slide to match whichever series the team uses.
- **Data slide placeholders:** the placeholders (mean, std, skewness, kurtosis, period) can be filled in from the summary table above.

---

## Setup and running

Tested with Python 3.11.0 on macOS. Package versions are pinned in [`requirements.txt`](../requirements.txt).

```bash
cd gjr_garch
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Register the venv as a Jupyter kernel. The slide-stats notebook's metadata points to this kernel name.
python -m ipykernel install --sys-prefix --name gjr-garch-venv --display-name "Python 3.11 (gjr_garch .venv)"
```

**Run interactively:** open the notebook in VS Code or JupyterLab and choose the **Python 3.11 (gjr_garch .venv)** kernel. In VS Code, you can also choose the `.venv` interpreter.

**Run from the command line** (runs every cell and saves the outputs back into the file):

```bash
.venv/bin/jupyter nbconvert --to notebook --execute --inplace Presentations/SP500_data_slide_stats.ipynb
```

**Conda users:** Parin's notebook was saved with a `conda base` kernel. Both notebooks run in any environment that has the packages in `requirements.txt`.

### Reproducibility notes

- **Live data.** The notebooks download prices from Yahoo Finance each time they run. If Yahoo ever revises history, the counts could change. The slide-stats notebook checks for 4 outliers and 5,205 returns and stops with an error if they don't match, so a changed dataset is caught instead of slipping through.
- **statsmodels 0.16.** statsmodels 0.15 warns that `adfuller` will return a result object instead of a tuple in 0.16. The notebook silences that warning and takes the first five tuple elements. After upgrading past 0.16, pass `result_object=False` or switch to the result object's attributes.
- **Generated notebook.** `SP500_data_slide_stats.ipynb` was generated and run in a script, then saved with outputs. It's safe to edit directly in Jupyter.

---

## Work log

**2026-09-20 (Parin Sharda):** built the dataset notebook: download, log returns, 8σ cleaning, plots.

**2026-10-13 (Daksh Kumar):**
1. **Environment:** created the project venv at `.venv` (Python 3.11.0 via pyenv), since none existed. Installed `yfinance`, `statsmodels`, `scipy`, `pandas`, `numpy`, `matplotlib`, and the Jupyter tooling. Registered the venv as the `gjr-garch-venv` kernel, inside the venv.
2. **Slide stats notebook:** created `Presentations/SP500_data_slide_stats.ipynb`. It reproduces Parin's pipeline exactly (matching 5,209 → 4 outliers → 5,205) and computes the Data slide statistics. All cells were run and the outputs saved, with no errors or warnings.
3. **Outlier check:** checked the 4 flagged outliers and found that they are real crash days. See Finding 1.
4. **Data adjustment check:** confirmed that `Close` equals `Adj Close` for ^GSPC, so the auto-adjust setting doesn't affect the data.
5. **Docs and repo:** wrote this guide, the README, `requirements.txt`, and `.gitignore`, and published the repo.

## Next steps

- Decide on the outlier filter (Finding 1).
- Fit GJR-GARCH(1,1,1) with Student-t and skewed-t innovations; compare them by AIC. A likely package is `arch`, which isn't installed yet.
- Validate the standardized residuals (Ljung-Box on residuals and squared residuals) and forecast conditional variance.
