# Quant Alpha Research Platform Source Code

This file consolidates the current publishable source files because command-line Git push is not authenticated on this machine.


## `requirements.txt`

```text
beautifulsoup4==4.14.3
certifi==2026.4.22
cffi==2.0.0
charset-normalizer==3.4.7
contourpy==1.3.3
curl_cffi==0.15.0
cycler==0.12.1
duckdb==1.5.2
fonttools==4.63.0
frozendict==2.4.7
idna==3.15
kiwisolver==1.5.0
markdown-it-py==4.2.0
matplotlib==3.10.9
mdurl==0.1.2
multitasking==0.0.13
numpy==2.4.6
packaging==26.2
pandas==3.0.3
peewee==4.0.5
pillow==12.2.0
platformdirs==4.9.6
protobuf==7.35.0
pyarrow==24.0.0
pycparser==3.0
Pygments==2.20.0
pyparsing==3.3.2
python-dateutil==2.9.0.post0
pytz==2026.2
PyYAML==6.0.3
requests==2.34.2
rich==15.0.0
six==1.17.0
soupsieve==2.8.3
typing_extensions==4.15.0
urllib3==2.7.0
websockets==16.0
yfinance==1.3.0

```

## `config.yaml`

```yaml
project:
  name: "quant-alpha-research-platform"
  description: "Institutional-style multi-factor equity alpha research platform with machine learning and backtesting"
  author: "Dipen Bhatt"

data:
  start_date: "2015-01-01"
  end_date: "2026-05-21"
  frequency: "monthly"

  universe:
    name: "starter_us_equity_universe"
    tickers:
      - AAPL
      - MSFT
      - NVDA
      - AMZN
      - GOOGL
      - META
      - JPM
      - XOM
      - UNH
      - PG

  paths:
    raw: "data/raw"
    interim: "data/interim"
    processed: "data/processed"

backtest:
  rebalance_frequency: "monthly"
  initial_capital: 1000000
  transaction_cost_bps: 10
  long_count: 3
  short_count: 3

factors:
  momentum:
    enabled: true
    lookback_months: 12
    skip_recent_months: 1

  volatility:
    enabled: true
    lookback_days: 63

  liquidity:
    enabled: true
    lookback_days: 63

machine_learning:
  target: "forward_1m_return"
  train_start: "2015-01-01"
  train_end: "2020-12-31"
  test_start: "2021-01-01"
  test_end: "2026-05-21"

```

## `src/utils/config_loader.py`

```python
"""
Configuration loader for the quant-alpha-research-platform.

This module reads the project's config.yaml file and returns the settings
as a Python dictionary.
"""

from pathlib import Path
from typing import Any

import yaml


def get_project_root() -> Path:
    """
    Return the root directory of the project.

    This file lives at src/utils/config_loader.py, so the project root is
    two parent directories above it.
    """
    return Path(__file__).resolve().parents[2]


def load_config(config_path: str | Path | None = None) -> dict[str, Any]:
    """
    Load the YAML configuration file.

    Parameters
    ----------
    config_path:
        Optional path to a config file. If not provided, this function loads
        config.yaml from the project root.

    Returns
    -------
    dict[str, Any]
        Project configuration as a Python dictionary.
    """
    if config_path is None:
        config_path = get_project_root() / "config.yaml"
    else:
        config_path = Path(config_path)

    if not config_path.exists():
        raise FileNotFoundError(f"Config file not found: {config_path}")

    with config_path.open("r", encoding="utf-8") as file:
        config = yaml.safe_load(file)

    if not isinstance(config, dict):
        raise ValueError("Config file must contain a YAML dictionary at the top level.")

    return config


def main() -> None:
    """Load and print key configuration values for verification."""
    config = load_config()

    project_name = config["project"]["name"]
    start_date = config["data"]["start_date"]
    end_date = config["data"]["end_date"]
    tickers = config["data"]["universe"]["tickers"]

    print("Configuration loaded successfully.")
    print("-" * 60)
    print(f"Project name: {project_name}")
    print(f"Start date: {start_date}")
    print(f"End date: {end_date}")
    print(f"Number of tickers: {len(tickers)}")
    print(f"Tickers: {', '.join(tickers)}")


if __name__ == "__main__":
    main()

```

## `src/utils/health_check.py`

```python
"""
Project health check script.

This script verifies that the Python environment and core packages
for the quant-alpha-research-platform are working correctly.
"""

import sys

import duckdb
import matplotlib
import numpy as np
import pandas as pd
import pyarrow
import yfinance as yf


def main() -> None:
    """Run a basic environment health check."""

    print("Quant Alpha Research Platform - Environment Check")
    print("-" * 60)

    print(f"Python version: {sys.version.split()[0]}")
    print(f"pandas version: {pd.__version__}")
    print(f"numpy version: {np.__version__}")
    print(f"matplotlib version: {matplotlib.__version__}")
    print(f"pyarrow version: {pyarrow.__version__}")
    print(f"duckdb version: {duckdb.__version__}")
    print(f"yfinance version: {yf.__version__}")

    sample_data = pd.DataFrame(
        {
            "ticker": ["AAPL", "MSFT", "NVDA"],
            "price": [190.0, 420.0, 850.0],
            "weight": [0.30, 0.40, 0.30],
        }
    )

    weighted_average_price = np.sum(sample_data["price"] * sample_data["weight"])

    print("-" * 60)
    print("Sample portfolio data:")
    print(sample_data)
    print("-" * 60)
    print(f"Weighted average price: {weighted_average_price:.2f}")

    duckdb_result = duckdb.query(
        """
        SELECT
            ticker,
            price,
            weight
        FROM sample_data
        WHERE weight >= 0.30
        """
    ).to_df()

    print("-" * 60)
    print("DuckDB query result:")
    print(duckdb_result)

    print("-" * 60)
    print("Environment health check completed successfully.")


if __name__ == "__main__":
    main()
```

## `src/data_pipeline/download_prices.py`

```python
"""
Download raw stock price data for the configured equity universe.

The script reads tickers, dates, and storage paths from config.yaml, downloads
historical prices with yfinance, and saves the raw table as a Parquet file.
"""

from pathlib import Path
import sys

import pandas as pd
import yfinance as yf


PROJECT_ROOT = Path(__file__).resolve().parents[2]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from src.utils.config_loader import load_config


def download_price_data(
    tickers: list[str],
    start_date: str,
    end_date: str,
) -> pd.DataFrame:
    """Download daily OHLCV price data and return it in long format."""
    prices = yf.download(
        tickers=tickers,
        start=start_date,
        end=end_date,
        auto_adjust=False,
        progress=False,
        group_by="ticker",
        threads=True,
    )

    if prices.empty:
        raise ValueError("No price data was downloaded. Check tickers and date range.")

    prices.index.name = "date"

    if isinstance(prices.columns, pd.MultiIndex):
        prices = (
            prices.stack(level=0, future_stack=True)
            .rename_axis(index=["date", "ticker"])
            .reset_index()
        )
    else:
        prices = prices.reset_index()
        prices["ticker"] = tickers[0]

    prices.columns = [
        str(column).strip().lower().replace(" ", "_") for column in prices.columns
    ]

    expected_columns = [
        "date",
        "ticker",
        "open",
        "high",
        "low",
        "close",
        "adj_close",
        "volume",
    ]
    available_columns = [column for column in expected_columns if column in prices.columns]
    prices = prices[available_columns].sort_values(["ticker", "date"]).reset_index(
        drop=True
    )

    return prices


def main() -> None:
    """Download configured price data and save it to the raw data directory."""
    config = load_config()

    tickers = config["data"]["universe"]["tickers"]
    start_date = config["data"]["start_date"]
    end_date = config["data"]["end_date"]
    raw_data_dir = PROJECT_ROOT / config["data"]["paths"]["raw"]
    output_path = raw_data_dir / "stock_prices_raw.parquet"

    raw_data_dir.mkdir(parents=True, exist_ok=True)

    print("Downloading raw stock price data...")
    print("-" * 60)
    print(f"Tickers: {', '.join(tickers)}")
    print(f"Start date: {start_date}")
    print(f"End date: {end_date}")

    prices = download_price_data(
        tickers=tickers,
        start_date=start_date,
        end_date=end_date,
    )

    prices.to_parquet(output_path, index=False)

    print("-" * 60)
    print("Download completed successfully.")
    print(f"Rows saved: {len(prices):,}")
    print(f"Columns: {', '.join(prices.columns)}")
    print(f"Output file: {output_path}")


if __name__ == "__main__":
    main()

```

## `src/data_pipeline/calculate_returns.py`

```python
"""
Create daily and monthly return datasets from raw stock prices.

The script reads the raw price Parquet file, calculates returns from adjusted
close prices, and saves processed datasets for later factor research.
"""

from pathlib import Path
import sys

import pandas as pd


PROJECT_ROOT = Path(__file__).resolve().parents[2]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from src.utils.config_loader import load_config


def calculate_daily_returns(prices: pd.DataFrame) -> pd.DataFrame:
    """Calculate daily returns for each ticker using adjusted close prices."""
    prices = prices.copy()
    prices["date"] = pd.to_datetime(prices["date"])
    prices = prices.sort_values(["ticker", "date"]).reset_index(drop=True)

    prices["daily_return"] = prices.groupby("ticker")["adj_close"].pct_change()

    return prices


def calculate_monthly_returns(prices: pd.DataFrame) -> pd.DataFrame:
    """Calculate monthly returns from last trading-day adjusted close prices."""
    prices = prices.copy()
    prices["date"] = pd.to_datetime(prices["date"])
    prices = prices.sort_values(["ticker", "date"]).reset_index(drop=True)
    prices["month_end"] = prices["date"].dt.to_period("M").dt.to_timestamp("M")

    monthly_prices = (
        prices.groupby(["ticker", "month_end"], as_index=False)
        .last()
        .sort_values(["ticker", "month_end"])
        .reset_index(drop=True)
    )

    monthly_prices["monthly_return"] = monthly_prices.groupby("ticker")[
        "adj_close"
    ].pct_change()

    return monthly_prices


def main() -> None:
    """Read raw prices, calculate returns, and save processed outputs."""
    config = load_config()

    raw_data_dir = PROJECT_ROOT / config["data"]["paths"]["raw"]
    processed_data_dir = PROJECT_ROOT / config["data"]["paths"]["processed"]

    input_path = raw_data_dir / "stock_prices_raw.parquet"
    daily_output_path = processed_data_dir / "daily_returns.parquet"
    monthly_output_path = processed_data_dir / "monthly_returns.parquet"

    if not input_path.exists():
        raise FileNotFoundError(
            f"Raw price file not found: {input_path}. "
            "Run src/data_pipeline/download_prices.py first."
        )

    processed_data_dir.mkdir(parents=True, exist_ok=True)

    print("Creating return datasets from raw stock prices...")
    print("-" * 60)
    print(f"Input file: {input_path}")

    prices = pd.read_parquet(input_path)
    daily_returns = calculate_daily_returns(prices)
    monthly_returns = calculate_monthly_returns(prices)

    daily_returns.to_parquet(daily_output_path, index=False)
    monthly_returns.to_parquet(monthly_output_path, index=False)

    print("-" * 60)
    print("Return datasets created successfully.")
    print(f"Daily rows saved: {len(daily_returns):,}")
    print(f"Monthly rows saved: {len(monthly_returns):,}")
    print(f"Daily output: {daily_output_path}")
    print(f"Monthly output: {monthly_output_path}")


if __name__ == "__main__":
    main()

```

## `src/factors/momentum.py`

```python
"""
Calculate the momentum alpha factor from monthly returns.

Momentum at month t is P_(t-skip) / P_(t-lookback) - 1. For 12-1,
this compounds 11 monthly returns and excludes the current month.
"""

from pathlib import Path
import sys

import pandas as pd


PROJECT_ROOT = Path(__file__).resolve().parents[2]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from src.utils.config_loader import load_config


def calculate_momentum_factor(
    monthly_returns: pd.DataFrame,
    lookback_months: int,
    skip_recent_months: int,
) -> pd.DataFrame:
    """Calculate compounded momentum for each ticker and month."""
    if not 0 <= skip_recent_months < lookback_months:
        raise ValueError("Require 0 <= skip_recent_months < lookback_months.")

    factor_name = f"momentum_{lookback_months}_{skip_recent_months}"
    compounding_months = lookback_months - skip_recent_months

    factor_data = monthly_returns.copy()
    factor_data["month_end"] = pd.to_datetime(factor_data["month_end"])
    factor_data["date"] = pd.to_datetime(factor_data["date"])
    factor_data = factor_data.sort_values(["ticker", "month_end"]).reset_index(
        drop=True
    )

    shifted_returns = factor_data.groupby("ticker")["monthly_return"].shift(
        skip_recent_months
    )

    factor_data[factor_name] = (
        (1 + shifted_returns)
        .groupby(factor_data["ticker"])
        .rolling(window=compounding_months, min_periods=compounding_months)
        .apply(lambda returns: returns.prod() - 1, raw=True)
        .reset_index(level=0, drop=True)
    )

    return factor_data[
        [
            "ticker",
            "month_end",
            "date",
            "monthly_return",
            factor_name,
        ]
    ]


def main() -> None:
    """Read monthly returns, calculate momentum, and save the factor dataset."""
    config = load_config()

    lookback_months = config["factors"]["momentum"]["lookback_months"]
    skip_recent_months = config["factors"]["momentum"]["skip_recent_months"]

    processed_data_dir = PROJECT_ROOT / config["data"]["paths"]["processed"]
    input_path = processed_data_dir / "monthly_returns.parquet"
    output_path = processed_data_dir / f"momentum_{lookback_months}_{skip_recent_months}.parquet"

    if not input_path.exists():
        raise FileNotFoundError(
            f"Monthly returns file not found: {input_path}. "
            "Run src/data_pipeline/calculate_returns.py first."
        )

    print("Calculating momentum factor...")
    print("-" * 60)
    print(f"Input file: {input_path}")
    print(f"Lookback months: {lookback_months}")
    print(f"Skip recent months: {skip_recent_months}")
    print(f"Monthly returns compounded: {lookback_months - skip_recent_months}")

    monthly_returns = pd.read_parquet(input_path)
    momentum = calculate_momentum_factor(
        monthly_returns=monthly_returns,
        lookback_months=lookback_months,
        skip_recent_months=skip_recent_months,
    )

    momentum.to_parquet(output_path, index=False)

    factor_name = f"momentum_{lookback_months}_{skip_recent_months}"
    non_null_count = momentum[factor_name].notna().sum()

    print("-" * 60)
    print("Momentum factor created successfully.")
    print(f"Rows saved: {len(momentum):,}")
    print(f"Non-null factor values: {non_null_count:,}")
    print(f"Output file: {output_path}")


if __name__ == "__main__":
    main()

```

## `src/factors/create_factor_dataset.py`

```python
"""
Create a research dataset that links factor signals to future returns.

The script reads the configured momentum factor data, adds each stock's
forward one-month return, and saves the result for factor evaluation.
"""

from pathlib import Path
import sys

import pandas as pd


PROJECT_ROOT = Path(__file__).resolve().parents[2]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from src.utils.config_loader import load_config


def create_factor_dataset(
    factor_data: pd.DataFrame,
    factor_name: str,
) -> pd.DataFrame:
    """Add forward one-month returns to a monthly factor dataset."""
    required_columns = {
        "ticker",
        "month_end",
        "date",
        "monthly_return",
        factor_name,
    }
    missing_columns = required_columns.difference(factor_data.columns)
    if missing_columns:
        missing = ", ".join(sorted(missing_columns))
        raise ValueError(f"Factor data is missing required columns: {missing}")

    result = factor_data.copy()
    result["month_end"] = pd.to_datetime(result["month_end"])
    result["date"] = pd.to_datetime(result["date"])
    result = result.sort_values(["ticker", "month_end"]).reset_index(drop=True)

    result["forward_1m_return"] = result.groupby("ticker")[
        "monthly_return"
    ].shift(-1)

    last_observation_date = result["date"].max()
    expected_month_end = last_observation_date + pd.offsets.BMonthEnd(0)
    if last_observation_date < expected_month_end:
        incomplete_signal_month = result["month_end"].max() - pd.offsets.MonthEnd(1)
        result.loc[
            result["month_end"].eq(incomplete_signal_month), "forward_1m_return"
        ] = float("nan")

    return result[
        [
            "ticker",
            "month_end",
            "date",
            "monthly_return",
            factor_name,
            "forward_1m_return",
        ]
    ]


def main() -> None:
    """Read momentum data, add forward returns, and save the factor dataset."""
    config = load_config()

    lookback_months = config["factors"]["momentum"]["lookback_months"]
    skip_recent_months = config["factors"]["momentum"]["skip_recent_months"]
    factor_name = f"momentum_{lookback_months}_{skip_recent_months}"

    processed_data_dir = PROJECT_ROOT / config["data"]["paths"]["processed"]
    input_path = processed_data_dir / f"{factor_name}.parquet"
    output_path = processed_data_dir / "factor_dataset.parquet"

    if not input_path.exists():
        raise FileNotFoundError(
            f"Momentum factor file not found: {input_path}. "
            "Run src/factors/momentum.py first."
        )

    print("Creating factor research dataset...")
    print("-" * 60)
    print(f"Input file: {input_path}")
    print(f"Factor: {factor_name}")

    momentum_data = pd.read_parquet(input_path)
    factor_dataset = create_factor_dataset(
        factor_data=momentum_data,
        factor_name=factor_name,
    )

    factor_dataset.to_parquet(output_path, index=False)

    usable_rows = factor_dataset[
        [factor_name, "forward_1m_return"]
    ].notna().all(axis=1).sum()

    print("-" * 60)
    print("Factor research dataset created successfully.")
    print(f"Rows saved: {len(factor_dataset):,}")
    print(f"Rows usable for momentum testing: {usable_rows:,}")
    print(f"Output file: {output_path}")


if __name__ == "__main__":
    main()

```

## `src/factors/evaluate_momentum.py`

```python
"""Evaluate whether momentum predicts next-month stock returns."""

from pathlib import Path
import sys

import pandas as pd


PROJECT_ROOT = Path(__file__).resolve().parents[2]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from src.utils.config_loader import load_config


def calculate_monthly_ic(
    factor_data: pd.DataFrame,
    factor_name: str,
) -> pd.DataFrame:
    """Calculate the momentum Information Coefficient for each month."""
    required_columns = {
        "ticker",
        "month_end",
        factor_name,
        "forward_1m_return",
    }
    missing_columns = required_columns.difference(factor_data.columns)
    if missing_columns:
        missing = ", ".join(sorted(missing_columns))
        raise ValueError(f"Factor data is missing required columns: {missing}")

    result = factor_data.copy()
    result["month_end"] = pd.to_datetime(result["month_end"])
    result = result.dropna(subset=[factor_name, "forward_1m_return"])

    def calculate_one_month_ic(month_data: pd.DataFrame) -> pd.Series:
        factor_ranks = month_data[factor_name].rank()
        return_ranks = month_data["forward_1m_return"].rank()
        ic = factor_ranks.corr(return_ranks)

        return pd.Series(
            {
                "ic": ic,
                "n_stocks": len(month_data),
            }
        )

    monthly_ic = (
        result.groupby("month_end")
        .apply(calculate_one_month_ic, include_groups=False)
        .reset_index()
        .sort_values("month_end")
        .reset_index(drop=True)
    )

    return monthly_ic


def summarize_ic(monthly_ic: pd.DataFrame) -> pd.Series:
    """Summarize the strength and consistency of monthly IC values."""
    if "ic" not in monthly_ic.columns:
        raise ValueError("Monthly IC data must contain an 'ic' column.")

    valid_ic = monthly_ic["ic"].dropna()
    if valid_ic.empty:
        raise ValueError("No valid IC values are available to summarize.")

    ic_standard_deviation = valid_ic.std()
    information_ratio = (
        valid_ic.mean() / ic_standard_deviation
        if ic_standard_deviation != 0
        else float("nan")
    )

    return pd.Series(
        {
            "months_evaluated": int(valid_ic.size),
            "average_ic": valid_ic.mean(),
            "median_ic": valid_ic.median(),
            "ic_standard_deviation": ic_standard_deviation,
            "positive_ic_rate": (valid_ic > 0).mean(),
            "ic_information_ratio": information_ratio,
        }
    )


def main() -> None:
    """Read the factor dataset, evaluate momentum, and save the results."""
    config = load_config()

    lookback_months = config["factors"]["momentum"]["lookback_months"]
    skip_recent_months = config["factors"]["momentum"]["skip_recent_months"]
    factor_name = f"momentum_{lookback_months}_{skip_recent_months}"

    processed_data_dir = PROJECT_ROOT / config["data"]["paths"]["processed"]
    input_path = processed_data_dir / "factor_dataset.parquet"
    monthly_output_path = processed_data_dir / "momentum_monthly_ic.parquet"
    summary_output_path = processed_data_dir / "momentum_ic_summary.csv"

    if not input_path.exists():
        raise FileNotFoundError(
            f"Factor dataset not found: {input_path}. "
            "Run src/factors/create_factor_dataset.py first."
        )

    print("Evaluating momentum factor...")
    print("-" * 60)
    print(f"Input file: {input_path}")
    print(f"Factor: {factor_name}")

    factor_data = pd.read_parquet(input_path)
    monthly_ic = calculate_monthly_ic(factor_data, factor_name)
    ic_summary = summarize_ic(monthly_ic)

    monthly_ic.to_parquet(monthly_output_path, index=False)
    ic_summary.rename("value").to_csv(summary_output_path, header=True)

    print("-" * 60)
    print("Momentum evaluation completed successfully.")
    print(f"Months evaluated: {int(ic_summary['months_evaluated'])}")
    print(f"Average monthly IC: {ic_summary['average_ic']:.4f}")
    print(f"Median monthly IC: {ic_summary['median_ic']:.4f}")
    print(f"IC standard deviation: {ic_summary['ic_standard_deviation']:.4f}")
    print(f"Positive IC rate: {ic_summary['positive_ic_rate']:.2%}")
    print(f"IC information ratio: {ic_summary['ic_information_ratio']:.4f}")
    print(f"Monthly results: {monthly_output_path}")
    print(f"Summary results: {summary_output_path}")


if __name__ == "__main__":
    main()

```

## `src/backtester/momentum_portfolio.py`

```python
"""Build and save a monthly long-short momentum portfolio backtest."""

from pathlib import Path
import sys

import pandas as pd


PROJECT_ROOT = Path(__file__).resolve().parents[2]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

from src.utils.config_loader import load_config


def build_momentum_portfolio(
    factor_data: pd.DataFrame,
    factor_name: str,
    long_count: int = 3,
    short_count: int = 3,
    transaction_cost_bps: float = 0.0,
) -> pd.DataFrame:
    """Build monthly returns and charge for changes in target weights.

    Each side has 100% target exposure. Turnover is half the absolute target
    weight change; costs apply to the full amount bought or sold, including
    opening positions. This simple model does not include weight drift,
    financing, or short-borrow costs.
    """
    if long_count < 1 or short_count < 1:
        raise ValueError("long_count and short_count must both be positive.")
    if transaction_cost_bps < 0:
        raise ValueError("transaction_cost_bps must be non-negative.")

    required_columns = {
        "ticker",
        "month_end",
        factor_name,
        "forward_1m_return",
    }
    missing_columns = required_columns.difference(factor_data.columns)
    if missing_columns:
        missing = ", ".join(sorted(missing_columns))
        raise ValueError(f"Factor data is missing required columns: {missing}")

    result = factor_data.copy()
    result["month_end"] = pd.to_datetime(result["month_end"])
    result = result.dropna(subset=[factor_name, "forward_1m_return"])

    duplicate_rows = result.duplicated(subset=["month_end", "ticker"])
    if duplicate_rows.any():
        raise ValueError("Factor data contains duplicate ticker-month rows.")

    def calculate_one_month_portfolio(month_data: pd.DataFrame) -> pd.Series:
        ranked = month_data.sort_values(
            [factor_name, "ticker"],
            ascending=[False, True],
        ).reset_index(drop=True)

        minimum_stocks = long_count + short_count
        if len(ranked) < minimum_stocks:
            raise ValueError(
                f"Each month must contain at least {minimum_stocks} stocks."
            )

        long_stocks = ranked.head(long_count)
        short_stocks = ranked.tail(short_count)
        long_weight = 1.0 / long_count
        short_weight = -1.0 / short_count
        target_weights = dict.fromkeys(long_stocks["ticker"], long_weight)
        target_weights.update(dict.fromkeys(short_stocks["ticker"], short_weight))

        long_return = long_stocks["forward_1m_return"].mean()
        short_basket_return = short_stocks["forward_1m_return"].mean()
        short_position_return = -short_basket_return
        long_short_return = long_return + short_position_return

        return pd.Series(
            {
                "n_stocks": len(ranked),
                "long_return": long_return,
                "short_basket_return": short_basket_return,
                "short_position_return": short_position_return,
                "long_short_return": long_short_return,
                "long_tickers": ", ".join(long_stocks["ticker"]),
                "short_tickers": ", ".join(short_stocks["ticker"]),
                "target_weights": target_weights,
            }
        )

    monthly_portfolio = (
        result.groupby("month_end")
        .apply(calculate_one_month_portfolio, include_groups=False)
        .reset_index()
        .sort_values("month_end")
        .reset_index(drop=True)
    )

    monthly_portfolio["cumulative_long_short_return"] = (
        1 + monthly_portfolio["long_short_return"]
    ).cumprod() - 1

    previous_weights: dict[str, float] = {}
    traded_notional = []
    for target_weights in monthly_portfolio["target_weights"]:
        tickers = target_weights.keys() | previous_weights.keys()
        traded_notional.append(
            sum(
                abs(target_weights.get(ticker, 0.0) - previous_weights.get(ticker, 0.0))
                for ticker in tickers
            )
        )
        previous_weights = target_weights

    monthly_portfolio["traded_notional"] = traded_notional
    monthly_portfolio["turnover"] = monthly_portfolio["traded_notional"] / 2
    monthly_portfolio["transaction_cost"] = (
        monthly_portfolio["traded_notional"] * transaction_cost_bps / 10_000
    )
    monthly_portfolio["net_long_short_return"] = (
        monthly_portfolio["long_short_return"] - monthly_portfolio["transaction_cost"]
    )
    monthly_portfolio["cumulative_net_return"] = (
        1 + monthly_portfolio["net_long_short_return"]
    ).cumprod() - 1

    return monthly_portfolio.drop(columns="target_weights")


def main() -> None:
    """Load factor data, run the momentum backtest, and save monthly results."""
    config = load_config()

    lookback_months = config["factors"]["momentum"]["lookback_months"]
    skip_recent_months = config["factors"]["momentum"]["skip_recent_months"]
    factor_name = f"momentum_{lookback_months}_{skip_recent_months}"

    long_count = config["backtest"].get("long_count", 3)
    short_count = config["backtest"].get("short_count", 3)
    transaction_cost_bps = config["backtest"]["transaction_cost_bps"]

    processed_data_dir = PROJECT_ROOT / config["data"]["paths"]["processed"]
    input_path = processed_data_dir / "factor_dataset.parquet"
    output_path = processed_data_dir / "momentum_portfolio_returns.parquet"

    if not input_path.exists():
        raise FileNotFoundError(
            f"Factor dataset not found: {input_path}. "
            "Run src/factors/create_factor_dataset.py first."
        )

    print("Running momentum portfolio backtest...")
    print("-" * 60)
    print(f"Input file: {input_path}")
    print(f"Factor: {factor_name}")
    print(f"Long stocks per month: {long_count}")
    print(f"Short stocks per month: {short_count}")
    print(f"One-way transaction cost: {transaction_cost_bps} bps per trade")

    factor_data = pd.read_parquet(input_path)
    factor_data["month_end"] = pd.to_datetime(factor_data["month_end"])
    factor_data["date"] = pd.to_datetime(factor_data["date"])

    last_observation_date = factor_data["date"].max()
    expected_month_end = last_observation_date + pd.offsets.BMonthEnd(0)
    if last_observation_date < expected_month_end:
        incomplete_holding_month = last_observation_date.to_period("M").to_timestamp(
            "M"
        )
        incomplete_signal_month = incomplete_holding_month - pd.offsets.MonthEnd(1)
        factor_data = factor_data[
            factor_data["month_end"] < incomplete_signal_month
        ].copy()
        print(
            "Excluded incomplete holding month ending "
            f"{incomplete_holding_month.date()}."
        )

    portfolio_returns = build_momentum_portfolio(
        factor_data=factor_data,
        factor_name=factor_name,
        long_count=long_count,
        short_count=short_count,
        transaction_cost_bps=transaction_cost_bps,
    )

    portfolio_returns.to_parquet(output_path, index=False)

    print("-" * 60)
    print("Momentum portfolio backtest completed successfully.")
    print(f"Months tested: {len(portfolio_returns):,}")
    print(
        "First portfolio month: "
        f"{portfolio_returns['month_end'].min().date()}"
    )
    print(
        "Last portfolio month: "
        f"{portfolio_returns['month_end'].max().date()}"
    )
    print(
        "Compounded long-short return: "
        f"{portfolio_returns['cumulative_long_short_return'].iloc[-1]:.2%}"
    )
    print(
        "Compounded return after trading costs: "
        f"{portfolio_returns['cumulative_net_return'].iloc[-1]:.2%}"
    )
    print(f"Output file: {output_path}")


if __name__ == "__main__":
    main()

```

## `tests/test_momentum_definition.py`

```python
"""Checks for the price-ratio definition of 12-1 momentum."""

import math
import unittest

import pandas as pd

from src.factors.momentum import calculate_momentum_factor


class MomentumDefinitionTests(unittest.TestCase):
    def test_twelve_one_uses_eleven_returns_and_skips_signal_month(self) -> None:
        month_ends = pd.date_range("2020-03-31", periods=13, freq="ME")
        rows = []
        for ticker, earlier_return, current_return in (
            ("AAA", 0.01, 1.00),
            ("BBB", -0.02, -0.50),
        ):
            monthly_returns = [float("nan")] + [earlier_return] * 11 + [
                current_return
            ]
            for month_end, monthly_return in zip(
                month_ends, monthly_returns, strict=True
            ):
                rows.append(
                    {
                        "ticker": ticker,
                        "month_end": month_end,
                        "date": month_end,
                        "monthly_return": monthly_return,
                    }
                )

        result = calculate_momentum_factor(
            pd.DataFrame(rows).sample(frac=1, random_state=7),
            lookback_months=12,
            skip_recent_months=1,
        )

        for ticker, monthly_return in (("AAA", 0.01), ("BBB", -0.02)):
            stock = result[result["ticker"].eq(ticker)]
            self.assertTrue(stock.iloc[:-1]["momentum_12_1"].isna().all())
            self.assertTrue(
                math.isclose(
                    stock.iloc[-1]["momentum_12_1"],
                    (1 + monthly_return) ** 11 - 1,
                    rel_tol=1e-12,
                )
            )

    def test_skip_must_be_less_than_lookback(self) -> None:
        with self.assertRaises(ValueError):
            calculate_momentum_factor(pd.DataFrame(), 1, 1)


if __name__ == "__main__":
    unittest.main()

```

## `tests/test_forward_month_completeness.py`

```python
"""Checks that partial final months are not used as forward-return targets."""

import math
import unittest

import pandas as pd

from src.factors.create_factor_dataset import create_factor_dataset


class ForwardMonthCompletenessTests(unittest.TestCase):
    def test_partial_final_month_has_no_forward_return(self) -> None:
        factor_data = pd.DataFrame(
            {
                "ticker": ["AAA", "AAA", "AAA"],
                "month_end": pd.to_datetime(
                    ["2024-01-31", "2024-02-29", "2024-03-31"]
                ),
                "date": pd.to_datetime(
                    ["2024-01-31", "2024-02-29", "2024-03-20"]
                ),
                "monthly_return": [0.10, 0.20, 0.30],
                "momentum_12_1": [0.40, 0.50, 0.60],
            }
        )

        result = create_factor_dataset(factor_data, "momentum_12_1")

        self.assertTrue(math.isclose(result.iloc[0]["forward_1m_return"], 0.20))
        self.assertTrue(result.iloc[1:]["forward_1m_return"].isna().all())


if __name__ == "__main__":
    unittest.main()

```

## `tests/test_momentum_transaction_costs.py`

```python
"""Checks turnover and costs for a small two-month portfolio."""

import math
import unittest

import pandas as pd

from src.backtester.momentum_portfolio import build_momentum_portfolio


class MomentumTransactionCostTests(unittest.TestCase):
    def test_initial_entry_and_one_long_replacement(self) -> None:
        rows = []
        months = (
            (
                "2024-01-31",
                {"A": 0.6, "B": 0.5, "C": 0.4, "D": 0.3, "E": 0.2, "F": 0.1},
                {"A": 0.10, "B": 0.20, "C": 0.0, "D": 0.0, "E": -0.05, "F": 0.05},
            ),
            (
                "2024-02-29",
                {"A": 0.6, "C": 0.5, "B": 0.4, "D": 0.3, "E": 0.2, "F": 0.1},
                {"A": 0.01, "B": 0.0, "C": 0.03, "D": 0.0, "E": 0.01, "F": -0.01},
            ),
        )
        for month_end, scores, future_returns in months:
            for ticker in scores:
                rows.append(
                    {
                        "ticker": ticker,
                        "month_end": month_end,
                        "momentum_12_1": scores[ticker],
                        "forward_1m_return": future_returns[ticker],
                    }
                )

        result = build_momentum_portfolio(
            pd.DataFrame(rows).sample(frac=1, random_state=7),
            factor_name="momentum_12_1",
            long_count=2,
            short_count=2,
            transaction_cost_bps=10,
        )

        self.assertEqual(result["long_tickers"].tolist(), ["A, B", "A, C"])
        self.assertEqual(result["short_tickers"].tolist(), ["E, F", "E, F"])
        self.assertEqual(result["traded_notional"].tolist(), [2.0, 1.0])
        self.assertEqual(result["turnover"].tolist(), [1.0, 0.5])
        self.assertTrue(math.isclose(result.loc[0, "long_short_return"], 0.15))
        self.assertTrue(math.isclose(result.loc[1, "long_short_return"], 0.02))
        self.assertTrue(math.isclose(result.loc[0, "transaction_cost"], 0.002))
        self.assertTrue(math.isclose(result.loc[1, "transaction_cost"], 0.001))
        self.assertTrue(math.isclose(result.loc[0, "net_long_short_return"], 0.148))
        self.assertTrue(math.isclose(result.loc[1, "net_long_short_return"], 0.019))
        self.assertTrue(math.isclose(result.loc[1, "cumulative_net_return"], 0.169812))

    def test_negative_cost_is_rejected(self) -> None:
        with self.assertRaises(ValueError):
            build_momentum_portfolio(pd.DataFrame(), "momentum_12_1", transaction_cost_bps=-1)


if __name__ == "__main__":
    unittest.main()

```

## `practice/README.md`

```markdown
# Coding Practice Area

This folder is for learning by rewriting the project logic yourself.

The real project code stays in `src/`. Practice code stays here, so you can
experiment freely without breaking the working pipeline.

## How to Practice

Work through the files in this order:

1. `exercises/01_config_loader_practice.py`
2. `exercises/02_returns_practice.py`
3. `exercises/03_momentum_practice.py`
4. `exercises/04_forward_returns_practice.py`
5. `exercises/05_information_coefficient_practice.py`

Each file has `TODO` comments and unfinished functions. Your job is to replace
the unfinished parts with working code.

After editing a file, run:

` ` `bash
python practice/check_practice.py
` ` `

At first, the checker will fail. That is expected. It will pass gradually as
you complete the exercises.

## Rule For Learning

Try to write the code yourself first. If stuck, compare the idea with the real
project files in `src/`, then come back and rewrite it in your own hands.

```

## `practice/check_practice.py`

```python
"""
Check the practice exercises.

Run from project root:

python practice/check_practice.py
"""

from pathlib import Path
import importlib.util
import math
import sys

import pandas as pd


PROJECT_ROOT = Path(__file__).resolve().parents[1]
EXERCISES_DIR = PROJECT_ROOT / "practice" / "exercises"


def load_module(file_name: str, module_name: str):
    """Load a practice file whose name starts with a number."""
    path = EXERCISES_DIR / file_name
    spec = importlib.util.spec_from_file_location(module_name, path)
    if spec is None or spec.loader is None:
        raise ImportError(f"Could not import {path}")
    module = importlib.util.module_from_spec(spec)
    sys.modules[module_name] = module
    spec.loader.exec_module(module)
    return module


def sample_prices() -> pd.DataFrame:
    """Create tiny price data for practice checks."""
    return pd.DataFrame(
        {
            "date": [
                "2024-01-30",
                "2024-01-31",
                "2024-02-29",
                "2024-03-29",
                "2024-01-31",
                "2024-02-29",
                "2024-03-29",
            ],
            "ticker": ["AAA", "AAA", "AAA", "AAA", "BBB", "BBB", "BBB"],
            "adj_close": [100.0, 110.0, 121.0, 108.9, 50.0, 55.0, 60.5],
            "open": [0.0] * 7,
            "high": [0.0] * 7,
            "low": [0.0] * 7,
            "close": [0.0] * 7,
            "volume": [100] * 7,
        }
    )


def sample_monthly_returns() -> pd.DataFrame:
    """Create monthly returns with enough rows for a 3_1 momentum check."""
    rows = []
    month_ends = pd.date_range("2024-01-31", periods=6, freq="ME")
    returns = [0.02, 0.03, -0.01, 0.04, 0.05, -0.02]
    for month_end, monthly_return in zip(month_ends, returns, strict=True):
        rows.append(
            {
                "ticker": "AAA",
                "month_end": month_end,
                "date": month_end,
                "monthly_return": monthly_return,
            }
        )
    return pd.DataFrame(rows)


def check_config_loader() -> None:
    module = load_module("01_config_loader_practice.py", "config_loader_practice")
    root = module.get_project_root()
    assert root == PROJECT_ROOT, f"Wrong project root: {root}"

    config = module.load_config_practice()
    assert config["project"]["name"] == "quant-alpha-research-platform"
    assert "AAPL" in config["data"]["universe"]["tickers"]
    print("PASS: config loader practice")


def check_returns() -> None:
    module = load_module("02_returns_practice.py", "returns_practice")
    prices = sample_prices()

    daily = module.calculate_daily_returns_practice(prices)
    aaa_second_return = daily[daily["ticker"].eq("AAA")]["daily_return"].iloc[1]
    assert math.isclose(aaa_second_return, 0.10, rel_tol=1e-9)

    monthly = module.calculate_monthly_returns_practice(prices)
    aaa_feb_return = monthly[
        monthly["ticker"].eq("AAA") & monthly["month_end"].eq(pd.Timestamp("2024-02-29"))
    ]["monthly_return"].iloc[0]
    assert math.isclose(aaa_feb_return, 0.10, rel_tol=1e-9)
    print("PASS: returns practice")


def check_momentum() -> None:
    module = load_module("03_momentum_practice.py", "momentum_practice")
    monthly = sample_monthly_returns()

    momentum = module.calculate_momentum_practice(
        monthly,
        lookback_months=3,
        skip_recent_months=1,
    )

    factor_column = "momentum_3_1"
    june_value = momentum[momentum["month_end"].eq(pd.Timestamp("2024-06-30"))][
        factor_column
    ].iloc[0]
    expected = (1 + 0.04) * (1 + 0.05) - 1
    assert math.isclose(june_value, expected, rel_tol=1e-9)
    print("PASS: momentum practice")


def check_forward_returns() -> None:
    module = load_module(
        "04_forward_returns_practice.py",
        "forward_returns_practice",
    )

    factor_data = pd.DataFrame(
        {
            "ticker": ["AAA", "AAA", "AAA", "BBB", "BBB", "BBB"],
            "month_end": pd.to_datetime(
                [
                    "2024-01-31",
                    "2024-02-29",
                    "2024-03-31",
                    "2024-01-31",
                    "2024-02-29",
                    "2024-03-31",
                ]
            ),
            "date": pd.to_datetime(
                [
                    "2024-01-31",
                    "2024-02-29",
                    "2024-03-29",
                    "2024-01-31",
                    "2024-02-29",
                    "2024-03-29",
                ]
            ),
            "monthly_return": [0.02, 0.05, -0.03, -0.01, 0.04, 0.06],
            "momentum_12_1": [0.10, 0.12, 0.15, -0.05, -0.02, 0.01],
        }
    )

    result = module.create_forward_returns_practice(factor_data)

    aaa = result[result["ticker"].eq("AAA")].sort_values("month_end")
    bbb = result[result["ticker"].eq("BBB")].sort_values("month_end")

    assert math.isclose(aaa["forward_1m_return"].iloc[0], 0.05, rel_tol=1e-9)
    assert math.isclose(aaa["forward_1m_return"].iloc[1], -0.03, rel_tol=1e-9)
    assert pd.isna(aaa["forward_1m_return"].iloc[2])
    assert math.isclose(bbb["forward_1m_return"].iloc[0], 0.04, rel_tol=1e-9)
    assert pd.isna(bbb["forward_1m_return"].iloc[2])
    print("PASS: forward returns practice")


def check_information_coefficient() -> None:
    module = load_module(
        "05_information_coefficient_practice.py",
        "information_coefficient_practice",
    )

    factor_data = pd.DataFrame(
        {
            "month_end": pd.to_datetime(
                [
                    "2024-01-31",
                    "2024-01-31",
                    "2024-01-31",
                    "2024-02-29",
                    "2024-02-29",
                    "2024-02-29",
                ]
            ),
            "ticker": ["AAA", "BBB", "CCC", "AAA", "BBB", "CCC"],
            "momentum_12_1": [0.10, 0.20, 0.30, 0.10, 0.20, 0.30],
            "forward_1m_return": [0.01, 0.02, 0.03, 0.03, 0.02, 0.01],
        }
    )

    result = module.calculate_monthly_ic_practice(factor_data)
    result = result.sort_values("month_end").reset_index(drop=True)

    assert math.isclose(result["ic"].iloc[0], 1.0, rel_tol=1e-9)
    assert math.isclose(result["ic"].iloc[1], -1.0, rel_tol=1e-9)
    assert result["n_stocks"].tolist() == [3, 3]
    print("PASS: information coefficient practice")


def main() -> None:
    """Run all practice checks."""
    checks = [
        check_config_loader,
        check_returns,
        check_momentum,
        check_forward_returns,
        check_information_coefficient,
    ]
    failures = 0

    for check in checks:
        try:
            check()
        except Exception as error:
            failures += 1
            print(f"FAIL: {check.__name__}: {type(error).__name__}: {error}")

    print("-" * 60)
    if failures:
        print(f"{failures} practice check(s) still need work.")
    else:
        print("All practice checks passed.")


if __name__ == "__main__":
    main()

```

## `practice/exercises/01_config_loader_practice.py`

```python
"""
PRACTICE 1: Load a configuration file using the `configparser` module.
GOAL
Write Your own small version of the `load_config` function that reads a configuration file and returns a dictionary of the settings.
"""

from pathlib import Path
from typing import Any

import yaml

def get_project_root() -> Path:
    """
    Return the project root folder
    This file is inside practice/exercises/.
    The project root is two parent folders above this file.
    """
    return Path(__file__).resolve().parents[2]

def load_config_practice(config_path: str | Path | None=None) -> dict[str, Any]:
    """
    load config.yaml and return it to python dictionary>

    If no config path is provided, use the project's default config.yaml file.
    """
    if config_path is None:
        config_path = get_project_root() / "config.yaml"
    else:
        config_path = Path(config_path)

    if not config_path.exists():
        raise FileNotFoundError(f"Config file not found at {config_path}")
    
    with config_path.open("r", encoding="utf-8") as file:
        config= yaml.safe_load(file)
    if not isinstance(config, dict):
        raise ValueError(f"Config file at {config_path} is not a valid dictionary.")

    return config   


if __name__ == "__main__":    
        config = load_config_practice()  
        print(config["project"]["name"]) 
        print(config["data"]["universe"]["tickers"]) 


    







```

## `practice/exercises/02_returns_practice.py`

```python
"""
Practice 2: Calculate daily and monthly returns.

Goal:
Rewrite the return calculations yourself using pandas.
"""

import pandas as pd


def calculate_daily_returns_practice(prices: pd.DataFrame) -> pd.DataFrame:
    """
    Add a daily_return column using adjusted close prices.

    Steps:
    1. Copy the input dataframe.
    2. Convert date to datetime.
    3. Sort by ticker and date.
    4. For each ticker, calculate pct_change() of adj_close.
    5. Return the new dataframe.
    """
    prices=prices.copy()
    prices["date"]=pd.to_datetime(prices["date"])
    prices=prices.sort_values(["ticker","date"])
    prices["daily_return"]=prices.groupby("ticker")["adj_close"].pct_change()
    return prices




def calculate_monthly_returns_practice(prices: pd.DataFrame) -> pd.DataFrame:
    """
    Create monthly returns from last trading-day adjusted close prices.

    Steps:
    1. Copy the input dataframe.
    2. Convert date to datetime.
    3. Create month_end using date.dt.to_period("M").dt.to_timestamp("M").
    4. Group by ticker and month_end.
    5. Keep the last row in each group.
    6. Calculate monthly_return using pct_change() by ticker.
    7. Return the monthly dataframe.
    """
    prices = prices.copy()
    prices["date"] = pd.to_datetime(prices["date"])
    prices = prices.sort_values(["ticker", "date"]).reset_index(drop=True)
    prices["month_end"] = prices["date"].dt.to_period("M").dt.to_timestamp("M")

    monthly_prices = (
        prices.groupby(["ticker", "month_end"], as_index=False)
        .last()
        .sort_values(["ticker", "month_end"])
        .reset_index(drop=True)
    )

    monthly_prices["monthly_return"] = monthly_prices.groupby("ticker")[
        "adj_close"
    ].pct_change()

    return monthly_prices

```

## `practice/exercises/03_momentum_practice.py`

```python
"""
Practice 3: Calculate the momentum factor.

Goal:
Rewrite momentum_12_1 yourself.
"""

import pandas as pd


def calculate_momentum_practice(
    monthly_returns: pd.DataFrame,
    lookback_months: int = 12,
    skip_recent_months: int = 1,
) -> pd.DataFrame:
    """
    Calculate compounded momentum for each ticker.

    Momentum formula:
    (1 + r1) * (1 + r2) * ... * (1 + rn) - 1

    Steps:
    1. Copy the input dataframe.
    2. Convert month_end and date to datetime.
    3. Sort by ticker and month_end.
    4. Shift monthly_return by skip_recent_months within each ticker.
    5. Use rolling(window=lookback_months - skip_recent_months) within each ticker.
    6. Compound the shifted returns.
    7. Save the factor as momentum_12_1.
    8. Return ticker, month_end, date, monthly_return, momentum_12_1.
    """
    factor_data=monthly_returns.copy()
    factor_data["month_end"]=pd.to_datetime(factor_data["month_end"])
    factor_data["date"]=pd.to_datetime(factor_data["date"])
    factor_data=factor_data.sort_values(["ticker","month_end"]).reset_index(drop=True)
    factor_name=f"momentum_{lookback_months}_{skip_recent_months}"
    shifted_returns=factor_data.groupby("ticker")["monthly_return"].shift(skip_recent_months)
    growth_factors=1+shifted_returns
    factor_data[factor_name]=(growth_factors.groupby(factor_data["ticker"]).rolling(window=lookback_months - skip_recent_months,min_periods=lookback_months - skip_recent_months).apply(lambda returns:returns.prod()-1,raw=True).reset_index(level=0,drop=True))
    return factor_data[["date","month_end","ticker","monthly_return",factor_name]]







```

## `practice/exercises/04_forward_returns_practice.py`

```python
"""
Practice 4: Create forward one-month returns.

Goal:
Connect each month's factor signal to the return earned in the next month.
"""

import pandas as pd


def create_forward_returns_practice(
    factor_data: pd.DataFrame,
) -> pd.DataFrame:
    """
    Add a forward_1m_return column to a monthly factor dataset.

    Steps:
    1. Copy the input dataframe.
    2. Convert month_end and date to datetime.
    3. Sort by ticker and month_end.
    4. Within each ticker, shift monthly_return upward by one row.
    5. Save the shifted values as forward_1m_return.
    6. Return the completed dataframe.
    """
    df=factor_data.copy()
    df["month_end"]=pd.to_datetime(df["month_end"])
    df["date"]=pd.to_datetime(df["date"])
    df=df.sort_values(["ticker","month_end"]).reset_index(drop=True)
    df["forward_1m_return"]=df.groupby("ticker")["monthly_return"].shift(-1)
    return df[
        [
            "date",
            "month_end",
            "ticker",
            "monthly_return",
            "momentum_12_1",
            "forward_1m_return",
        ]
    ]


```

## `practice/exercises/05_information_coefficient_practice.py`

```python
"""
Practice 5: Calculate monthly Information Coefficient values.

Goal:
Measure whether higher momentum ranks predict higher next-month return ranks.
"""

import pandas as pd


def calculate_monthly_ic_practice(
    factor_data: pd.DataFrame,
    factor_name: str = "momentum_12_1",
) -> pd.DataFrame:
    """
    Calculate Spearman rank correlation for every month.

    Steps:
    1. Copy the input dataframe.
    2. Convert month_end to datetime.
    3. Remove rows missing the factor or forward return.
    4. Group rows by month_end.
    5. Correlate factor values with forward returns using method="spearman".
    6. Return month_end, ic, and number of stocks.
    """
    result = factor_data.copy()
    result["month_end"] = pd.to_datetime(result["month_end"])

    result = result.dropna(
        subset=[factor_name, "forward_1m_return"]
    )

    def calculate_one_month_ic(month_data: pd.DataFrame) -> pd.Series:
        factor_ranks = month_data[factor_name].rank()
        return_ranks = month_data["forward_1m_return"].rank()
        ic = factor_ranks.corr(return_ranks)

        return pd.Series(
            {
                "ic": ic,
                "n_stocks": len(month_data),
            }
        )

    monthly_ic = (
        result.groupby("month_end")
        .apply(calculate_one_month_ic, include_groups=False)
        .reset_index()
    )

    return monthly_ic

```
