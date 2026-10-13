# Forecasting S&P 500 Conditional Volatility with a GJR-GARCH Model

FIM 500 team project, NC State Master of Financial Mathematics.

We model daily S&P 500 (^GSPC) volatility with a GJR-GARCH(1,1,1) model to capture volatility clustering and the leverage effect, where negative shocks raise volatility more than positive ones. **Current stage:** dataset built and pre-modeling diagnostics done, for the midpoint check-in. The model has not been fit yet.

**Full documentation:** [`docs/project_guide.md`](docs/project_guide.md). It covers the file map, how the pipeline works, results, findings, setup, and the work log.

## Headline numbers (cleaned sample, 2006-01-04 to 2026-09-18, n = 5,205)

| | |
|---|---|
| Mean daily log return | 0.0348% |
| Daily std / annualized vol | 1.180% / 18.73% |
| Skewness / excess kurtosis | −0.438 / 9.00 |
| ADF stat (AIC) | −15.24 (p < 0.001): stationary |
| Ljung-Box Q(20), squared returns | 6,943.7 (p < 0.001): strong ARCH effects |

The 4 days the 8σ cleaning step removes are real 2008 and 2020 crash days, not data errors. See [Finding 1](docs/project_guide.md#1-the-8σ-filter-removes-real-crash-days-not-data-errors).

## Quickstart

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m ipykernel install --sys-prefix --name gjr-garch-venv --display-name "Python 3.11 (gjr_garch .venv)"
jupyter nbconvert --to notebook --execute --inplace Presentations/SP500_data_slide_stats.ipynb
```

## Key files

- `Presentations/S&P500_datacleaned_Parin Sharda.ipynb`: builds the dataset (download, log returns, 8σ cleaning)
- `Presentations/SP500_data_slide_stats.ipynb`: summary statistics and diagnostics for the Data slide
