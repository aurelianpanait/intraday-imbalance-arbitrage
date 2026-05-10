# Intraday vs Imbalance Arbitrage — Romanian Power Market

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-1.3+-orange.svg)](https://scikit-learn.org/)
[![Paper](https://img.shields.io/badge/paper-PDF-red.svg)](paper/main.pdf)
[![CI](https://github.com/USERNAME/intraday-imbalance-arbitrage/actions/workflows/ci.yml/badge.svg)](https://github.com/USERNAME/intraday-imbalance-arbitrage/actions/workflows/ci.yml)

A machine-learning pipeline that predicts the sign of grid imbalance on the Romanian power market 1h45min before delivery, then converts the prediction into a confidence-weighted intraday trading strategy.

Trained on 87,934 fifteen-minute intervals between July 2022 and January 2025. Backtested out-of-sample on the second half of 2024.

## Headline results

| Metric | Value |
|---|---|
| Direction accuracy (validation, May–Jun 2024) | 74.3% |
| Win rate (backtest, Jul–Dec 2024) | 67.8% |
| Cumulative PnL (backtest) | 46.66 M RON |
| Number of trades | 17,308 |
| Profit factor | 2.99 |
| Maximum drawdown | −695.8 kRON (1.5% of total PnL) |
| Capture ratio vs oracle | 31.2% |

Per-month consistency, sharpe disclaimer, and full discussion of the gap to live performance are in the [paper](paper/main.pdf).

## What's in the repo

```
.
├── notebooks/
│   └── intraday_arbitrage.ipynb     # End-to-end pipeline: data → features → models → backtest
├── src/                              # (Optional refactor location — currently logic lives in the notebook)
├── data/
│   ├── raw/                          # Input CSVs from OPCOM / Transelectrica
│   └── processed/
│       └── backtest_results_final.csv  # Output: one row per traded interval
├── figures/                          # PNG exports used in the paper
├── paper/
│   ├── main.tex                      # LaTeX source
│   └── main.pdf                      # Compiled paper (11 pages, IMRaD + Appendix)
├── requirements.txt
├── LICENSE
└── README.md
```

## Method, in three sentences

1. The target is binary direction of grid imbalance (surplus = 1, deficit = 0); the trading window closes 1h before delivery and the realised imbalance is published with a 25-minute delay, so all features must respect a 7-quarter lag (`REAL_LAG = 7`) on observed grid state and a 4-quarter lag (`FCST_LAG = 4`) on forecast-derived inputs.
2. A Gradient Boosting classifier is trained with class-balanced sample weights on July 2022 – April 2024 and beats logistic regression and random forest by a small margin on validation AUC-ROC; the dominant feature is the lagged imbalance volume itself (`imb_volume_t-7` carries roughly 0.58 of total importance).
3. The trading strategy converts the model probability into SELL/BUY/no-trade signals at a confidence-optimised threshold of 0.51, sizes volume linearly in `|p − 0.5|` capped at 10 MWh per interval, and is settled at the realised intraday VWAP — no transaction costs, no market-impact adjustment.

## Output schema (`data/processed/backtest_results_final.csv`)

One row per 15-minute interval, July 2024 – January 2025 (17,610 rows).

| Column | Description |
|---|---|
| `P_Surplus` | Model probability that the grid is in surplus |
| `Confidence` | Distance from 0.5: `\|p − 0.5\| / 0.5` |
| `Signal` | Trading decision: SELL / BUY / NO_TRADE |
| `Volume_MWh` | Volume traded (0–10 MWh, proportional to confidence) |
| `ID_Price_RON` | Intraday VWAP execution price (RON/MWh) |
| `Imb_Price_Pos` | Imbalance price for surplus participants (RON/MWh) |
| `Imb_Price_Neg` | Imbalance price for deficit participants (RON/MWh) |
| `PnL_RON` | PnL realised on this interval (RON) |
| `Cumulative_PnL_RON` | Running total PnL (RON) |
| `Actual_Imb_Volume` | Realised grid imbalance (MW) |
| `Actual_Sign` | Realised grid direction (+1 surplus / −1 deficit) |
| `Prediction_Correct` | Whether `Signal` matched `Actual_Sign` |

## Reproducing the results

```bash
# Clone and install
git clone https://github.com/<your-user>/intraday-imbalance-arbitrage.git
cd intraday-imbalance-arbitrage
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# Run the full pipeline
jupyter notebook notebooks/intraday_arbitrage.ipynb
```

Runtime is roughly 4–6 minutes on a laptop CPU, dominated by the gradient boosting fit on 63k rows.

## Limitations to read before quoting any number

The backtest does not model transaction costs, executes at realised intraday VWAP rather than against the order book, and assumes no market-impact for the 10 MWh order size. The 13.25 daily-Sharpe figure is an artefact of the wide imbalance spreads on the Romanian market combined with zero costs; a realistic live estimate, based on comparable European arbitrage strategies, sits between 1.5 and 3.0. A conservative live deployment estimate is 40–60% of the backtest PnL. Section 4.2 of the paper goes through this in detail.

## Data sources

Input CSVs are aggregated from public Romanian market operators:
- **OPCOM** — intraday VWAP prices, day-ahead prices
- **Transelectrica** — grid imbalance volume, imbalance settlement prices, aFRR / mFRR activations, day-ahead forecasts (load, solar, wind, renewable totals), realised generation per technology

The aggregated dataset is included in `data/raw/` for full reproducibility. If you adapt the pipeline to a different market (RTE, REE, REN, etc.), the lag constants `FCST_LAG` and `REAL_LAG` need to be re-derived from the local rules for forecast publication and post-delivery data release.

## License

MIT — see [LICENSE](LICENSE). Use the code freely; the data is included for reproducibility but is published by the Romanian TSO and OPCOM, please respect their terms of use if you redistribute it.

## Citation

If you build on this work, a link to the repo is enough. For an academic citation:

```bibtex
@techreport{panait2026intraday,
  author = {Panait, Aurelian Andrei},
  title  = {Intraday vs Imbalance Arbitrage on the Romanian Power Market},
  institution = {MINES Paris -- PSL},
  year   = {2026},
  type   = {Technical case study},
  url    = {https://github.com/aurelianpanait/intraday-imbalance-arbitrage}
}
```
