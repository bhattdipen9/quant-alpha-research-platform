# Quant Alpha Research Platform: Progress and Plan

Last reviewed: 2026-10-12. This is a handoff and learning guide for the current local project, not a claim that the full research plan is complete. The saved price snapshot ends on **2026-05-20**.

## 1. What are we trying to answer?

The research question is: **Can signals derived from stock data help rank US stocks by their future returns, and can a portfolio built from those signals perform after realistic costs and risk controls?**

We are starting with one signal, momentum. The longer-term goal in the [research plan](reports/Quant_Research_Plan.pdf) is a multi-factor research platform with proper out-of-sample testing, backtesting, possible machine learning, and a report/dashboard. A positive result in the current ten-stock experiment is a reason to investigate further, **not** proof of a reliable or tradable strategy.

The basic workflow is:

`settings -> raw prices -> daily/monthly returns -> momentum -> next-month returns -> IC evaluation -> portfolio backtest -> risk and robustness analysis`

## 2. Where the project stands

The local Python environment, Git repository, data pipeline, momentum factor, IC evaluation, and a first cost-adjusted long-short backtest exist. The core backtest is working, but the broader multi-factor/ML project is still ahead. We have **not** yet built a performance report, benchmark comparison, robust investment universe, additional factors, ML model, optimizer, dashboard, or final research report.

Git has local commits (latest observed: `20c232c Add coding practice exercises`), but current work includes uncommitted changes. No GitHub remote was configured when this document was written. This document does not make a commit or publish anything.

## 3. Inputs and project setup

| Item | Current state |
| --- | --- |
| Settings | [config.yaml](config.yaml) holds the universe, dates, factor settings, paths, and backtest settings. [config_loader.py](src/utils/config_loader.py) reads it into a Python dictionary. |
| Python | Project-local `.venv` is used for scripts and tests. [health_check.py](src/utils/health_check.py) checks core packages. |
| Source | Yahoo Finance data downloaded through `yfinance` by [download_prices.py](src/data_pipeline/download_prices.py). |
| Universe | AAPL, MSFT, NVDA, AMZN, GOOGL, META, JPM, XOM, UNH, PG. These are ten hand-picked *current* stocks, not an historical index membership file. |
| Configured dates | `2015-01-01` through `2026-05-21` (the download API's end date is exclusive). Actual saved observations: `2015-01-02` through `2026-05-20`. |
| Raw data | [stock_prices_raw.parquet](data/raw/stock_prices_raw.parquet): 28,620 ticker-day rows; `date`, `ticker`, OHLC prices, adjusted close, and volume. |

`Parquet` is the file format used for compact data tables. Prices are stored in *long format*: each row is one ticker on one trading day. The table is sorted by ticker and then date, so many AAPL rows can appear before the MSFT rows.

## 4. What we implemented, step by step

### A. Daily and monthly returns

[calculate_returns.py](src/data_pipeline/calculate_returns.py) sorts each ticker's history by date and computes:

`daily_return = today's adjusted close / previous trading day's adjusted close - 1`

For each calendar month and ticker, it keeps the last available trading day's row. It then computes:

`monthly_return = this month's last adjusted close / previous month's last adjusted close - 1`

The calendar `month_end` label (for example, January 31) can differ from the actual last trading `date` (for example, January 30). Saved outputs are [daily_returns.parquet](data/processed/daily_returns.parquet), 28,620 rows, and [monthly_returns.parquet](data/processed/monthly_returns.parquet), 1,370 ticker-month rows. The first return for each ticker is missing because there is no earlier price in this dataset.

### B. The momentum signal

[momentum.py](src/factors/momentum.py) computes the configured **12-1 momentum** for each ticker at each signal month. In price terms:

`momentum_12_1(t) = adjusted_price(t-1 month) / adjusted_price(t-12 months) - 1`

In return terms, it compounds **11 monthly returns**, from month `t-11` through `t-1`, and skips the current signal month `t`. It is a compounded return, not an arithmetic average of monthly returns. For a March 2021 signal, it compounds April 2020 through February 2021 returns (equivalently, February 2021 price divided by March 2020 price, minus one). Each stock gets a *new* momentum value every month after enough history is available.

The saved [momentum_12_1.parquet](data/processed/momentum_12_1.parquet) has 1,370 rows; early factor values are missing until there is sufficient history. We corrected an earlier off-by-one version of this definition. Numbers calculated before that correction should not be mixed with the results below.

### C. Future return and Information Coefficient (IC)

[create_factor_dataset.py](src/factors/create_factor_dataset.py) adds each ticker's `forward_1m_return`, which is its **next** monthly return. At signal month `t`, we compare information known at `t` with what happened in `t+1`. This avoids using the future return to construct the signal.

The saved [factor_dataset.parquet](data/processed/factor_dataset.parquet) has 1,370 rows, of which 1,230 have both momentum and a complete next-month return. Because the saved price data stops on May 20, 2026, May 2026 is an incomplete holding month; its forward returns are not treated as a full-month outcome.

[evaluate_momentum.py](src/factors/evaluate_momentum.py) calculates a separate **Spearman rank IC for each month**: rank the ten stocks by momentum, rank the same stocks by next-month return, then correlate those two sets of ranks. IC near `+1` means the rankings agreed strongly; near `-1` means they were reversed; near zero means little rank agreement. It saves [momentum_monthly_ic.parquet](data/processed/momentum_monthly_ic.parquet) and [momentum_ic_summary.csv](data/processed/momentum_ic_summary.csv).

| IC result | Saved value |
| --- | ---: |
| Complete signal months evaluated | 123 |
| Average monthly IC | 0.0472 |
| Median monthly IC | 0.0909 |
| Monthly IC standard deviation | 0.3947 |
| Months with positive IC | 56.10% |
| IC information ratio (`mean IC / std. dev. of IC`) | 0.1195 |

This IC information ratio is about *signal-ranking consistency*. It is **not** the portfolio information ratio, which needs a return benchmark and tracking error.

### D. First momentum portfolio backtest

[momentum_portfolio.py](src/backtester/momentum_portfolio.py) tests a simple rule at every complete signal month:

1. Rank the ten stocks from highest to lowest momentum.
2. Hold the top three **long** at `+1/3` weight each, and the bottom three **short** at `-1/3` each, during the next month.
3. Calculate the gross monthly spread: `mean(next return of long stocks) - mean(next return of short stocks)`.
4. Compare target weights with the preceding month's targets. Charge 10 basis points (`0.10%`) per unit of traded notional, including the first month's entry trades.
5. Subtract that cost from the gross return and compound the monthly gross and net returns separately.

The target weights total zero net exposure, with 100% long and 100% short exposure (200% gross). The saved [momentum_portfolio_returns.parquet](data/processed/momentum_portfolio_returns.parquet) has one row per signal month, including the chosen tickers, component returns, traded notional, cost, net return, and cumulative gross/net results. `turnover` in that file equals **half** the sum of absolute target-weight changes; the cost uses the **full** traded notional.

| Backtest result | Saved value |
| --- | ---: |
| Complete signal months | 123, January 2016 through March 2026 |
| Corresponding final holding month | April 2026 |
| Compounded gross long-short spread | 129.02% |
| Compounded spread after modeled trading costs | 108.57% |
| One-way trading-cost assumption | 10 basis points |

These are **cumulative backtest returns over the entire period**, not annual returns or guaranteed profits. They are calculated on a normalized portfolio-return series; although `config.yaml` contains `initial_capital: 1000000`, the script does **not** yet create a dollar-valued equity curve from it. For orientation, the first portfolio row has traded notional `2.0` (opening both sides) and a cost of `0.002`, or `0.20%` of the normalized capital base.

## 5. Checks and coding practice

- Five automated project tests pass, covering the 12-1 momentum definition, incomplete final month, and transaction-cost behavior. Tests are in [tests/](tests/).
- Five learning exercises pass in [practice/](practice/): config loading, returns, momentum, forward returns, and IC. They are separate from the working production code in `src/`.
- The exercises are for understanding the mechanics. Passing their checker does not replace a full research validation or prove that the strategy works out of sample.

## 6. Important limitations and unfinished work

- **Small, selected universe:** Ten hand-picked stocks give unstable cross-sectional ICs and invite selection/survivorship bias. A credible historical study needs a broader, point-in-time universe including delisted names where possible.
- **No clean out-of-sample strategy test yet:** The configured train/test dates describe a future ML split, but there is no ML workflow or independently held-out portfolio evaluation yet. Looking at these backtest results while designing the next strategy can lead to overfitting.
- **Simplified execution:** Trading cost is a fixed rate on changes in *target* weights. The model ignores weight drift between rebalances, short borrow fees, financing, slippage differences, market impact, and trading restrictions.
- **Risk not yet measured:** There is no annualized return/volatility, Sharpe ratio, maximum drawdown, market/sector exposure, or appropriate benchmark comparison. A positive cumulative spread alone is not enough.
- **Data quality needs more work:** Yahoo Finance is convenient for learning but not institutional point-in-time data. The final-month completeness test uses a business-month-end approximation and should be made exchange-calendar aware. The local saved data is a May 2026 snapshot, not current October 2026 prices.
- **Configured does not mean implemented:** Volatility/liquidity parameters, ML train/test dates, and initial capital appear in `config.yaml`, but their associated modules or behaviors are not yet implemented. Value/quality factors from the wider research plan would require suitable fundamentals data.
- **Documentation and publication are unfinished:** The root `README.md` is still empty, there is no completed dashboard/report, and the repository has no GitHub remote.

## 7. Plan from here

The next task should be **a performance and risk report for the existing momentum portfolio**. We should first agree on what each measure means using a tiny worked example, then write and test the calculations: monthly net returns, annualized compounded return, annualized volatility, Sharpe ratio (with an explicit risk-free-rate assumption), maximum drawdown, and a simple plot/table. This gives us a clearer answer to whether the current positive cumulative return was worth its risk.

After that:

1. **Validate the baseline:** Add a sensible benchmark and market/sector exposure checks; verify signal/holding dates and realistic execution assumptions.
2. **Improve the dataset:** Refresh prices intentionally, audit missing observations/corporate actions, use a broader point-in-time universe, and handle exchange calendars and delistings where feasible.
3. **Add factors:** Implement volatility and liquidity, then consider value and quality when trustworthy fundamentals are available. Test each factor's IC before combining factors.
4. **Build a transparent multi-factor baseline:** Normalize signals across stocks, combine them using a stated rule, and compare its after-cost results with single-factor momentum.
5. **Test machine learning carefully:** Use past data only for training/validation, keep later dates for testing, prevent feature/label leakage, and compare ML with the simple baseline rather than assuming it helps.
6. **Improve portfolio construction:** Add position, sector, exposure, turnover, and cost constraints; evaluate robustness across universes, subperiods, factor definitions, and transaction-cost assumptions.
7. **Communicate the research:** Build the dashboard and final report, fill in the root README with reproducible instructions, and only then commit/publish the intended work to GitHub.

We will keep the learning style used so far: explain the financial idea on paper, show a small numerical example, write a few related lines of code at a time, inspect actual data/results, and run focused tests. The working `src/` modules remain separate from your `practice/` exercises.

## 8. Reproduce the current results

Run these from the **project root** with the project's Python interpreter. The first command uses the existing raw Parquet snapshot and does not download fresh data.

```bash
./.venv/bin/python src/data_pipeline/calculate_returns.py
./.venv/bin/python src/factors/momentum.py
./.venv/bin/python src/factors/create_factor_dataset.py
./.venv/bin/python src/factors/evaluate_momentum.py
./.venv/bin/python src/backtester/momentum_portfolio.py
./.venv/bin/python -m unittest discover -s tests -p 'test_*.py'
./.venv/bin/python practice/check_practice.py
```

To fetch fresh raw prices later, update the configured date and run `src/data_pipeline/download_prices.py` deliberately. That uses the network and overwrites the raw snapshot, so its outputs may differ from the figures recorded here.
