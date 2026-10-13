# Forecasting S&P 500 Conditional Volatility with a GJR-GARCH Model

FIM 500 team project, NC State Master of Financial Mathematics.

We model daily S&P 500 (^GSPC) volatility with a GJR-GARCH(1,1,1) model to capture volatility clustering and the leverage effect, where negative shocks raise volatility more than positive ones. **Current stage:** dataset built and validated, and pre-modeling diagnostics done, for the midpoint check-in.

**Full documentation:** [`docs/project_guide.md`](docs/project_guide.md). It covers the file map, how the analysis works, results, findings, setup, and the work log.

## Headline numbers (2006-01-04 to 2026-09-18, all 5,209 daily log returns)

| | |
|---|---|
| Mean daily log return | 0.0345% |
| Daily std / annualized vol | 1.218% / 19.34% |
| Skewness / excess kurtosis | −0.475 / 12.95 |
| ADF stat (AIC) | −16.22 (p < 0.001): stationary |
| Ljung-Box Q(20), squared returns | 7,100.0 (p < 0.001): strong ARCH effects |

The data was checked against the NYSE trading calendar and official S&P closes from FRED. No observations are removed: the four days an earlier 8σ screen dropped are real 2008 and 2020 crash days ([Finding 1](docs/project_guide.md#1-no-days-are-removed)).

## Files

- `Presentations/SP500_data_slide_stats.ipynb`: the analysis notebook (data, validation, summary statistics)
- `Presentations/S&P500_datacleaned_Parin Sharda.ipynb`: the original dataset notebook

## Quickstart

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m ipykernel install --sys-prefix --name gjr-garch-venv --display-name "Python 3.11 (gjr_garch .venv)"
jupyter nbconvert --to notebook --execute --inplace Presentations/SP500_data_slide_stats.ipynb
```
