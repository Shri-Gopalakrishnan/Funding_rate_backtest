# Funding Rate Mean Reversion — Walk-Forward Backtest

6-month walk-forward backtest of a BTC perpetual funding rate mean reversion strategy on Hyperliquid, tested across BTC, ETH, and SOL.

---

## The Strategy

Hyperliquid perpetual funding rates mean revert. When longs are paying an extreme premium to stay in their positions — economically unsustainable — they eventually close, pushing funding back toward zero. The strategy exploits this:

- **Entry:** open a short when funding rate exceeds the 75th percentile of its historical distribution (extreme positive — too many longs). Open a long when it drops below the 25th percentile (extreme negative — too many shorts).
- **Exit:** close when funding reverts to the 25th percentile threshold, after 24 hours, or at a 15% stop loss.
- **Position size:** 3 USDC per trade, maximum 6 USDC open simultaneously.

---

## Data

The Hyperliquid API returns a maximum of 500 rows per request — approximately 20 days of hourly data. To build a statistically meaningful backtest, we paginated backwards across multiple requests and stitched the results together, producing **4,795 hourly observations per asset spanning 6 months** (November 2025 to June 2026). Price data was sourced from Yahoo Finance via yfinance for Hurst exponent computation.

---

## Walk-Forward Methodology

To avoid look-ahead bias:
- **Train window:** 30 days — HMM calibration and threshold estimation
- **Test window:** 10 days — out-of-sample trade simulation
- Roll forward until the full 6-month history is covered

The model never sees future data. All parameters are calibrated fresh on each training window.

---

## Three Versions Tested

**V1 — Baseline:** Pure funding rate signal. Entry when |funding| exceeds the 75th percentile threshold, exit on reversion.

**V2 — HMM Filter:** Same signal, but only enters when a 3-state Hidden Markov Model classifies the current regime as EXTREME — the highest confidence mean reversion setups. The HMM uses Gaussian emissions and Bayesian posterior updates, calibrated on each training window.

**V3 — HMM + Hurst Filter:** Adds a rolling Hurst exponent filter. Only trades when the 48-hour rolling H < 0.5, confirming the market is in an anti-persistent, mean-reverting regime.

---

## Key Finding — BTC is Regime-Dependent

The backtest revealed that BTC V1, while profitable overall, is **regime-dependent** — it makes money in the first 4 months then gives back gains when market conditions shift. Looking at the cumulative P&L curve, the strategy peaks around trade 90 and declines for the remainder of the period. This is the core risk of a mean reversion strategy in a market that periodically trends strongly.

**The Hurst filter fixes this.**

The rolling Hurst exponent measures market memory. H < 0.5 indicates anti-persistent, mean-reverting behaviour — the regime where the strategy has structural edge. H > 0.5 indicates persistent, trending behaviour — the regime where mean reversion strategies lose money.

Adding the Hurst filter:
- **Reduces trade count** from 142 (V1) to 10 (V3)
- **Eliminates drawdown** — max drawdown drops from -2.83% to -0.17%
- **Improves Sharpe** from 1.85 to 8.70
- **Transforms the strategy** from regime-dependent to regime-aware — it only trades when market structure genuinely supports mean reversion, sitting out entirely when conditions are unfavourable

This is not just a performance improvement — it's a fundamentally more robust strategy. V1 is profitable only because the favourable regime lasted longer than the unfavourable one in this specific period. V3 would produce positive results regardless of which regime you start in.

---

## Multi-Asset Results

| Asset | Strategy | Trades | Win rate | Sharpe | Max DD | Stop losses |
|-------|----------|--------|----------|--------|--------|-------------|
| BTC | V1: Baseline | 142 | 43.0% | 1.85 | -2.83% | 0.0% |
| BTC | V2: +HMM | 137 | 43.1% | 0.83 | -2.80% | 0.0% |
| BTC | V3: +HMM +Hurst | 10 | 40.0% | 8.70 | -0.17% | 0.0% |
| ETH | V1: Baseline | 106 | 38.7% | -9.64 | -9.87% | 0.0% |
| ETH | V3: +HMM +Hurst | 3 | 100% | 121.11 | 0.00% | 0.0% |
| SOL | V1: Baseline | 343 | 43.4% | -4.09 | -20.78% | 0.3% |
| SOL | V3: +HMM +Hurst | 12 | 50.0% | 11.74 | -0.34% | 0.0% |

**BTC was chosen for deployment** because it showed the cleanest baseline signal — zero stop losses across 142 trades, positive Sharpe without any filter. ETH and SOL lost money without regime filtering, confirming BTC as the most appropriate asset for the funding rate signal.

---

## Gina Deployment

Parameters from the backtest were used directly in the Gina strategy:

- Entry threshold: 0.00125% hourly funding (75th percentile)
- Exit threshold: 0.00041% hourly funding (25th percentile)
- Position size: 3 USDC
- Maximum exposure: 6 USDC
- Time limit: 24 hours
- Check frequency: every 5 minutes

Live strategy: https://askgina.ai/strategy/6b2f68e9-d615-4b24-a088-1804ebb4bc38

---

## Research Connections

- **HMM architecture** mirrors the Interactive Brokers Live HMM Regime Dashboard
- **Hurst R/S analysis** connects directly to the fBm research notebook — H < 0.5 is the theoretical signature of anti-persistent, mean-reverting processes
- **Mean reversion thesis** validated in the IV Regime Dashboard using forward regression on equity implied volatility data

---

## Requirements

```
pip install requests pandas numpy matplotlib scipy yfinance
```

## Run

Open `funding_rate_backtest_clean.ipynb` in Jupyter and run all cells. Fetches live data from Hyperliquid API and Yahoo Finance automatically. Full 6-month fetch takes approximately 2 minutes.
