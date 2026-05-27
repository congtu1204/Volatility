# VN100 Volatility-Signal Portfolio Backtester

This repository contains a Python/Google Colab workflow for constructing and backtesting long-only volatility-sorted equity portfolios using stocks in the VN100 universe. The model divides the eligible stock universe into three sector-neutral portfolios from low risk to high risk, assigns market-cap-based weights inside each sector, and evaluates portfolio performance against VNINDEX.

The workflow is designed for research and educational use. It is not investment advice.

## Project summary

The script builds three volatility-signal portfolio models:

1. **Standard-deviation model**
   - Rebalanced monthly.
   - Uses 60-trading-day annualized standard deviation of daily stock returns.
   - Ranks stocks from low realized volatility to high realized volatility.

2. **Maximum-drawdown model**
   - Rebalanced monthly.
   - Uses the maximum drawdown over the previous 60 trading days.
   - Ranks stocks from low recent drawdown to high recent drawdown.

3. **Beta model**
   - Rebalanced monthly.
   - Uses 60-trading-day beta versus VNINDEX.
   - Ranks stocks from low market sensitivity to high market sensitivity.

For each signal, stocks are sorted within sectors into three portfolios:

| Portfolio | Meaning |
|---|---|
| `P1` | Lowest-risk stocks within each sector |
| `P2` | Middle-risk stocks within each sector |
| `P3` | Highest-risk stocks within each sector |

The market-cap-weighted portfolios use sector-replicated weighting. Sector weights are estimated from the full eligible VN100 universe, then stocks inside each sector bucket are weighted by market capitalization.

## Vietnamese market adaptations

The project includes several assumptions and adjustments for the Vietnamese equity market:

- Long-only portfolios only; no short selling.
- VN100-focused universe.
- Sector-aware sorting to reduce sector concentration in low-volatility and high-volatility portfolios.
- Monthly rebalancing using the last available trading date of each month.
- Market capitalization is forward-filled to the portfolio construction date to avoid look-ahead bias.
- VNINDEX is used for beta estimation and benchmark comparison.
- Trading costs include buy brokerage fee, sell brokerage fee, and sell-side personal income tax.
- Sectors with too few eligible stocks are assigned entirely to `P2` to avoid unstable tertile splits.

## Methodology

### Data loading

The code expects four types of input data.

#### 1. Daily stock price files

Daily stock price data should be uploaded as CSV files. Multiple files can be uploaded and concatenated.

Default expected columns:

| Field | Default column name |
|---|---|
| Date/time | `time` |
| Ticker | `ticker` |
| Close price | `close` |
| Volume | `volume` |

Default date format:

```python
%m/%d/%Y %H:%M
```

#### 2. Sector data file

The sector file can be the same valuation file used in the value/momentum project, as long as it contains ticker and sector columns.

Required default columns:

| Field | Default column name |
|---|---|
| Ticker | `Ticker` |
| Sector | `Industrial sector (ICB) L1` |

The script keeps the most recent available sector classification for each ticker.

#### 3. Market-capitalization file

Market capitalization can be uploaded as CSV or Excel files. Multiple files can be uploaded and concatenated.

Default expected columns:

| Field | Default column name |
|---|---|
| Ticker | `ticker` |
| Date | `Date` |
| Market capitalization | `Market cap` |

Default date format:

```python
%m/%d/%Y
```

Market cap is forward-filled to each monthly rebalance date.

#### 4. VNINDEX data file

VNINDEX data is used for two purposes:

1. Estimating 60-day stock beta versus VNINDEX.
2. Comparing portfolio performance against VNINDEX.

Default expected columns:

| Field | Default column name |
|---|---|
| Date | `date` |
| VNINDEX close price | `price` |

Default date format:

```python
%m/%d/%Y
```

If your VNINDEX file uses Vietnamese date format such as `29/04/2021`, change the format to:

```python
%d/%m/%Y
```

### Optional VN100 ticker filter

If the uploaded stock price files already contain only VN100 names, leave this variable unchanged:

```python
VN100_TICKERS = None
```

If the uploaded price files contain more than VN100 stocks, manually paste the VN100 ticker list:

```python
VN100_TICKERS = ["ACB", "BCM", "BID", "BVH", ...]
```

Only tickers in this list will be used for portfolio construction.

## Volatility signals

The strategy tests three risk signals separately. The signals are not combined in the default model.

### 1. Standard deviation

The standard-deviation signal is computed using trailing daily stock returns:

```text
std_60d = annualized standard deviation of daily returns over the previous 60 trading days
```

Formula:

```text
std_60d = sqrt(252) × std(daily_returns over 60 trading days)
```

Lower `std_60d` is treated as better and assigned toward `P1`.

### 2. Maximum drawdown

The maximum-drawdown signal is computed from the stock price path over the previous 60 trading days:

```text
mdd_60d = largest peak-to-trough loss over the previous 60 trading days
```

Lower drawdown is treated as better and assigned toward `P1`.

### 3. Beta versus VNINDEX

Beta is estimated using stock returns and VNINDEX returns over the previous 60 trading days:

```text
beta_60d = covariance(stock_return, VNINDEX_return) / variance(VNINDEX_return)
```

Lower beta is treated as better and assigned toward `P1`.

Because beta requires both stock return data and VNINDEX return data, the beta model may start later than the standard-deviation and maximum-drawdown models if VNINDEX data is missing or starts later.

## Signal window and minimum observations

Default configuration:

```python
VOL_WINDOW = 60
MIN_OBS_SIGNAL = 45
```

`VOL_WINDOW` is the target rolling window. `MIN_OBS_SIGNAL` allows the script to compute a signal when at least 45 observations are available inside the 60-trading-day window.

The backtest starts only after valid signals exist and after a monthly rebalance date is available. Therefore, the first portfolio date may be later than the first raw data date.

## Within-sector portfolio assignment

For each signal and each rebalance date:

1. Build the eligible universe.
2. Remove stocks with missing signal, sector, or market-cap data.
3. Rank stocks within each sector by the risk signal.
4. Assign stocks into `P1`, `P2`, and `P3`.

For all three risk signals:

```text
Lower signal = lower risk = better rank
```

The portfolio assignment is:

```text
P1 = lowest-risk third within the sector
P2 = middle-risk group within the sector
P3 = highest-risk third within the sector
```

The tertile sizes are calculated as:

```python
n_top = int(np.floor(n / 3))
n_bot = int(np.floor(n / 3))
```

where `n` is the number of eligible stocks in a sector. Any remainder goes to `P2`.

Example for a sector with 10 eligible stocks:

| Rank | Portfolio |
|---:|---|
| 1-3 | `P1` |
| 4-7 | `P2` |
| 8-10 | `P3` |

Sectors with fewer than `MIN_SECTOR_SIZE` eligible stocks are assigned entirely to `P2`.

## Market-cap weighting

The model uses a sector-replicated market-cap weighting method.

For each rebalance date and each signal:

1. Calculate each sector's market-cap weight in the full eligible universe.
2. For each portfolio, keep only sectors represented in that portfolio.
3. Redistribute unavailable sector weights across represented sectors.
4. Within each represented sector, weight stocks by market capitalization.

The final stock weight is:

```text
stock_weight = target_sector_weight × stock_market_cap / portfolio_sector_market_cap
```

Weights are normalized so each `date × signal_name × portfolio` sums to approximately 100%.

## Rebalancing and return measurement

The strategy is rebalanced monthly.

At each rebalance date:

1. Use the last available trading date of the month.
2. Compute trailing 60-day volatility signals using information available up to that date.
3. Construct `P1`, `P2`, and `P3` for each signal.
4. Assign sector-replicated market-cap weights.
5. Hold the portfolio until the next monthly rebalance date.

Holding-period return is calculated as:

```text
gross_ret = sum(stock_weight × stock_return)
net_ret   = gross_ret - fee_drag
```

`gross_ret` is the portfolio return before estimated trading costs. `net_ret` is the portfolio return after brokerage fees and sell tax.

## Turnover and transaction costs

The backtest calculates turnover using portfolio weights, not ticker counts.

At each rebalance, the code compares the new target weights against the previous portfolio's drifted pre-rebalance weights:

```text
turnover = (entering_weight + exiting_weight + holding_weight_adjustment) / 2
```

The first rebalance is assigned 100% turnover because the portfolio is assumed to be built from cash.

Transaction costs are calculated from active traded weights:

- Buy fee applies to entering stocks and weight increases in continuing holdings.
- Sell fee and sell tax apply to exiting stocks and weight decreases in continuing holdings.
- `fee_drag = buy_cost + sell_cost + rebal_cost`.

Default transaction-cost assumptions:

```python
BUY_FEE_PCT  = 0.15
SELL_FEE_PCT = 0.15
SELL_TAX_PCT = 0.10
```

The default round-trip cost is 0.40%.

## Benchmark comparison versus VNINDEX

The script computes VNINDEX holding-period returns over the same monthly periods as the model portfolios.

For each signal and portfolio, the benchmark report includes:

- Annualized gross return.
- Annualized net return.
- VNINDEX annualized return.
- Annualized active return.
- Model CAGR.
- VNINDEX CAGR.
- Model volatility.
- VNINDEX volatility.
- Tracking error.
- Sharpe ratio.
- Information ratio.
- Alpha versus VNINDEX.
- Beta versus VNINDEX.
- Alpha t-statistic.
- Model max drawdown.
- VNINDEX max drawdown.
- Win rate versus VNINDEX.

## Output files

The script creates an output folder:

```text
volatility_model_outputs/
```

It also creates a ZIP archive:

```text
volatility_model_outputs.zip
```

### Main CSV outputs

| File | Description |
|---|---|
| `volatility_model_monthly_market_cap_weighted_results.csv` | Monthly portfolio-level results for all signals and portfolios |
| `volatility_model_monthly_market_cap_weighted_stock_assignments.csv` | Stock-level assignments and weights for all signals and portfolios |
| `volatility_model_performance_summary.csv` | Standalone performance summary by signal and portfolio |
| `volatility_model_p1_minus_p3_spreads.csv` | Monthly `P1 - P3` spread returns by signal |
| `volatility_model_p1_minus_p3_summary.csv` | Summary statistics for the `P1 - P3` spread |
| `volatility_model_vs_vnindex_period_returns.csv` | Monthly portfolio returns with matched VNINDEX returns |
| `volatility_model_vs_vnindex_summary.csv` | Performance summary versus VNINDEX |
| `volatility_model_cumulative_returns.csv` | Cumulative model and VNINDEX return series |

### Per-signal stock assignment outputs

For each signal and portfolio, the script also exports a stock-level assignment file:

```text
volatility_model_std_60d_p1_stock_assignments.csv
volatility_model_std_60d_p2_stock_assignments.csv
volatility_model_std_60d_p3_stock_assignments.csv
volatility_model_mdd_60d_p1_stock_assignments.csv
volatility_model_mdd_60d_p2_stock_assignments.csv
volatility_model_mdd_60d_p3_stock_assignments.csv
volatility_model_beta_60d_p1_stock_assignments.csv
volatility_model_beta_60d_p2_stock_assignments.csv
volatility_model_beta_60d_p3_stock_assignments.csv
```

### Excel report

The script creates:

```text
volatility_model_report.xlsx
```

The Excel report includes:

| Sheet | Description |
|---|---|
| `Perf Summary` | Standalone performance summary |
| `VS VNINDEX` | Benchmark comparison versus VNINDEX |
| `P1-P3 Summary` | Summary statistics for low-risk minus high-risk spread |
| `P1-P3 Returns` | Monthly spread return series |
| `Cumulative` | Cumulative return series |
| `Latest Holdings` | Latest available stock-level holdings and weights |

### Cumulative return charts

The output folder includes one PNG chart for each `signal_name × portfolio` combination:

```text
cum_return_<signal_name>_<portfolio>_vs_vnindex.png
```

Each chart compares the portfolio's net cumulative return against VNINDEX.

## Key output columns

### Portfolio-level result files

| Column | Meaning |
|---|---|
| `signal_name` | Signal used for ranking: `std_60d`, `mdd_60d`, or `beta_60d` |
| `date` | Portfolio construction date |
| `period_end` | End date of the holding period |
| `portfolio` | Portfolio label: `P1`, `P2`, or `P3` |
| `n_stocks` | Number of stocks in the portfolio |
| `n_entering` | Number of stocks newly entering the portfolio |
| `n_exiting` | Number of stocks exiting the portfolio |
| `turnover` | Weight-based traded fraction of the portfolio |
| `buy_cost` | Estimated buy-side transaction cost |
| `sell_cost` | Estimated sell-side transaction cost and tax |
| `rebal_cost` | Cost from adjusting weights of continuing holdings |
| `fee_drag` | Total cost deducted from gross return |
| `gross_ret` | Holding-period return before trading costs |
| `net_ret` | Holding-period return after trading costs |
| `total_mcap` | Total market capitalization of stocks in the portfolio |
| `eligible_universe_size` | Number of stocks usable for portfolio construction after data filters |

### Stock-level assignment files

| Column | Meaning |
|---|---|
| `signal_name` | Volatility signal used for ranking |
| `date` | Portfolio construction date |
| `period_end` | End date of the holding period |
| `ticker` | Stock ticker |
| `sector` | Sector classification |
| `portfolio` | Assigned portfolio: `P1`, `P2`, or `P3` |
| `risk_signal` | Signal value used for ranking |
| `within_sector_rank` | Rank within the sector, where rank 1 is lowest risk |
| `within_sector_size` | Number of eligible stocks in the sector |
| `mcap` | Market capitalization used for weighting |
| `mcap_date` | Actual market-cap observation date |
| `sector_weight_target` | Target sector weight assigned to the portfolio |
| `weight_within_sector` | Stock weight inside its portfolio sector |
| `weight` | Final portfolio weight |
| `status` | `NEW` or `HOLD` status at the rebalance date |

### Performance-summary columns

| Column | Meaning |
|---|---|
| `n_periods` | Number of monthly return periods |
| `avg_n_stocks` | Average number of stocks in the portfolio |
| `ann_gross_return` | Arithmetic annualized gross return |
| `ann_net_return` | Arithmetic annualized net return |
| `cagr_net` | Compounded annual growth rate based on net returns |
| `ann_volatility` | Annualized volatility of monthly net returns |
| `sharpe_zero_rf` | Sharpe ratio assuming zero risk-free rate |
| `max_drawdown` | Maximum drawdown of the net return series |
| `avg_monthly_turnover` | Average monthly turnover |
| `ann_fee_drag` | Annualized fee drag |
| `win_rate_positive` | Fraction of months with positive net return |

### Benchmark-summary columns

| Column | Meaning |
|---|---|
| `vnindex_ann_return` | Arithmetic annualized VNINDEX return over matched periods |
| `ann_active_return` | Annualized portfolio return minus VNINDEX return |
| `model_cagr` | Portfolio CAGR |
| `vnindex_cagr` | VNINDEX CAGR |
| `tracking_error` | Annualized volatility of active return |
| `information_ratio` | Active return divided by tracking error |
| `alpha_vs_vnindex` | Annualized OLS alpha versus VNINDEX |
| `beta_vs_vnindex` | OLS beta versus VNINDEX |
| `alpha_t_stat` | t-statistic of monthly alpha estimate |
| `vnindex_max_drawdown` | VNINDEX maximum drawdown over matched periods |
| `win_rate_vs_vnindex` | Fraction of months where the portfolio beats VNINDEX |

## How to run

The workflow is written for Google Colab. It uses:

```python
from google.colab import files
files.upload()
files.download()
```

Recommended workflow:

1. Open `Volatility_Signal_VN100.ipynb` in Google Colab.
2. Edit the configuration section to match your data files.
3. Upload daily stock price CSV files when prompted.
4. Upload the sector file when prompted.
5. Upload market-cap files when prompted.
6. Upload the VNINDEX daily price file when prompted.
7. Run all notebook cells sequentially.
8. Download the generated `volatility_model_outputs.zip` file.

For local execution, replace Colab upload/download calls with local file paths, for example:

```python
prices = pd.read_csv("data/raw/prices.csv")
sector = pd.read_excel("data/raw/sector.xlsx")
mcap = pd.read_csv("data/raw/market_cap.csv")
vnindex = pd.read_csv("data/raw/vnindex.csv")
```

## Dependencies

Minimum Python dependencies:

```text
pandas
numpy
matplotlib
openpyxl
```

When running in Google Colab, `google.colab` is available by default. For local execution, remove or replace Colab-specific upload and download functions.

Suggested `requirements.txt`:

```text
pandas
numpy
matplotlib
openpyxl
```

## Suggested repository structure

```text
.
├── README.md
├── Volatility_Signal_VN100.ipynb
├── Volatility_Signal_VN100.py
├── requirements.txt
├── data/
│   ├── raw/              # not committed
│   └── processed/        # optional, not committed
└── outputs/              # generated files, usually not committed
```

Suggested `.gitignore`:

```text
__pycache__/
.ipynb_checkpoints/
*.csv
*.xlsx
*.zip
*.png
data/
outputs/
volatility_model_outputs/
volatility_model_outputs.zip
```

Do not commit proprietary raw financial data unless you have permission to redistribute it.

## Important assumptions and limitations

- The backtest depends entirely on the quality and coverage of the input data.
- The script does not automatically solve survivorship bias. If the universe is based only on current VN100 constituents, historical results may be overstated.
- Returns use the supplied close-price series. If prices are not adjusted for dividends or corporate actions, total return estimates may be inaccurate.
- The transaction-cost model is simplified and does not include bid-ask spread, market impact, board-lot constraints, liquidity limits, or foreign ownership limits.
- Market-cap data is forward-filled to rebalance dates. This avoids look-ahead bias but can still be stale if market-cap observations are sparse.
- VNINDEX date parsing must match the date format in the uploaded file. Incorrect parsing can delay or distort the beta signal.
- The model uses monthly rebalancing only. Weekly or quarterly variants require parameter changes.
- The model is long-only. It does not construct a long-short tradable factor portfolio.
- Standard deviation, maximum drawdown, and beta are tested separately. The default code does not combine them into a composite signal.
- The default annualized return in some summary fields uses arithmetic annualization. CAGR is also provided for compounded performance analysis.

## Research interpretation

The core research question is whether lower-risk stocks outperform higher-risk stocks in the VN100 universe after controlling for sector exposure and market-cap weighting.

The most important comparison is usually:

```text
P1 - P3 = low-risk portfolio return - high-risk portfolio return
```

A positive `P1 - P3` spread indicates that the volatility signal ranks stocks in the expected direction. A negative spread indicates that the high-risk portfolio outperformed the low-risk portfolio.

Interpretation by result pattern:

| Result | Interpretation |
|---|---|
| `P1 > P2 > P3` in raw return | Strong low-risk effect |
| `Sharpe(P1) > Sharpe(P2) > Sharpe(P3)` | Volatility signal improves risk-adjusted return |
| `P1 - P3 > 0` but weak t-stat | Directionally positive but statistically weak effect |
| `P3 > P1` | High-risk stocks were rewarded over the test period |
| `P1` beats VNINDEX with lower drawdown | Defensive low-volatility strategy is useful against benchmark |
| `P1` has lower return but much lower volatility | Signal may be useful for risk control rather than return prediction |

## Status

This project is a research backtesting workflow. It is not yet a production trading system. Before using it for investment decisions, add liquidity screens, data-vendor validation, corporate-action checks, out-of-sample testing, sensitivity tests for signal windows, and stricter transaction-cost assumptions.

