# EventEdge: Iron Condor Backtester — RBI Event Volatility Study

A systematic backtester for delta-based Iron Condor strategies around Indian market catalysts, built on real NSE historical options data. This project tests whether a strategy that looks profitable on scheduled events still holds up when stress-tested against real market shocks.

## What this is

A from-scratch options backtesting engine in Python (Google Colab), using:
- **Historical data:** NSE's official F&O Bhavcopy archives (UDiFF format) — this is what the backtester actually runs on
- **Pricing:** A custom Black-Scholes module with an implied volatility solver (Brent's method) and full Greeks (delta, gamma, theta, vega)
- **Spot estimation:** Put-call parity, derived directly from the option chain itself (NSE's daily F&O bhavcopy doesn't publish the underlying spot price)
- **Strategy:** A 4-leg Iron Condor, with strikes selected by target delta rather than fixed point distance — this keeps the position's probability profile consistent across different volatility regimes, which a fixed-point-width condor doesn't.

## The question this project actually asks

Most retail backtests report a win rate and stop there. The more useful question is: **what does this strategy look like on the days nobody planned for?**

This project deliberately tests the strategy on two different kinds of days:
1. **RBI Monetary Policy Committee announcement days** — scheduled, well-anticipated events, where elevated pre-event implied volatility usually collapses once the outcome is known (the "IV crush" premium sellers are trying to harvest).
2. **Real historical market shock days** — the June 2024 election result crash and a 2026 geopolitical/Fed-driven selloff — unscheduled moves where nobody had advance warning.

## Strategy rules (fully mechanical, no discretion)

- **Entry:** 2 trading days before a scheduled RBI event (captures elevated pre-event IV); 1 day before for unscheduled shock-day tests
- **Strikes:** Short call/put at a target delta (tested at both 0.20 and 0.10); long call/put hedges at half that delta
- **Target:** Close at 50% of credit received
- **Stop-loss:** Close at 2x credit received (i.e., max intended loss = credit collected)
- **Forced exit:** Expiry, if neither target nor stop-loss triggers first

## Results

### 0.20-delta Condor (tighter, more premium per trade)

| Sample | Trades | Win Rate | Total P&L | Worst Single Trade |
|---|---|---|---|---|
| 10 RBI events | 10/10 | 100% | +380.90 pts | — |
| 2 shock events | 2/2 | 0% | -531.30 pts | -317.10 pts |
| **Combined** | **12/12** | **83.3%** | **-69.30 pts** | **-317.10 pts** |

### 0.10-delta Condor (wider, less premium, more room before strikes are breached)

| Sample | Trades | Win Rate | Total P&L | Worst Single Trade |
|---|---|---|---|---|
| 10 RBI events | 10/10 | 100% | +166.30 pts | — |
| 2 shock events | 1/2* | 50% | -73.55 pts | -73.55 pts |
| **Combined** | **11/12** | **90.9%** | **+72.75 pts** | **-73.55 pts** |

*One shock date (June 3, 2024) could not be tested at 0.10 delta — NSE's listed option chain that day had no liquid strikes far enough out-of-the-money to construct the wider hedge leg. This is a genuine market-data constraint (low pre-shock IV meant far-OTM strikes had no open interest), not a code limitation, and is worth flagging rather than working around.

## The actual finding

**A 0.20-delta Iron Condor shows a perfect 10/10 win rate on RBI announcement days — and would have looked like a "solved" strategy if that were the only test.** It isn't. Once two real historical shock days are added to the sample, the strategy is net negative overall. The two shock-day losses are large enough (-317 and -214 points) that they more than erase ten winning trades' average gain of +38 points each.

**Widening the short strikes from 0.20 to 0.10 delta flips the outcome.** Per-trade income on calm RBI days drops roughly in half (avg +38 → +17 points), but the wider strikes give the position enough room that the same shock events cause proportionally smaller damage. Net result: -69.30 becomes +72.75.

This mirrors a finding from an earlier project in this series — a short ATM straddle backtest across 8 RBI events showed a 75% win rate but a net **loss**, because the few losing trades were disproportionately large relative to the many small wins. The Iron Condor result here is the same structural pattern (asymmetric tail risk in short-volatility strategies), now confirmed on a second, independently-built strategy.

## A specific limitation worth being upfront about

The stop-loss logic checks the position's value once per trading day, using end-of-day closing prices. On the June 2024 election shock, Nifty fell 5.93% in a single session — the position's actual cost-to-close (₹420.50) was more than double the intended stop-loss trigger (₹204.20) by the time the next close was checked. **A stop-loss only protects you if the market gives you a price to exit at near your trigger.** On a genuine overnight gap, it doesn't — the realized loss exceeded the theoretical maximum loss by roughly 2x. This is a real execution risk that a backtest like this can surface but can't fully solve; live trading would need either tighter intraday monitoring or accepting that worst-case losses can exceed the "designed" stop-loss level.

## Honest caveats on sample size

- 12 total events is still a small sample. Two shock days is enough to demonstrate the *pattern* of tail risk, not enough to estimate its true frequency or size with confidence.
- All historical data is from NSE's official Bhavcopy archives, which only cover the UDiFF format from July 2024 onward — six earlier RBI dates from a prior related study could not be included here due to this format change.
- Spot price is estimated via put-call parity rather than pulled from an official index close, since NSE's options bhavcopy doesn't publish the underlying spot directly. This was sanity-checked against known historical Nifty closing levels for the same dates and found accurate to within a few points.
- This is a backtest on historical closing prices, not a simulation of live execution — real slippage, bid-ask spread costs, and margin requirements are not modeled.

## Stack

Python · pandas · NumPy · SciPy (Black-Scholes, Brent's method) · NSE Bhavcopy archives · Google Colab

## Status / Next steps

- [x] Historical data engine (NSE Bhavcopy)
- [x] Black-Scholes Greeks module with IV solver
- [x] Put-call parity spot estimation
- [x] Delta-based Iron Condor with configurable strike targets
- [x] Multi-day P&L simulator with target/stop-loss/expiry logic
- [x] Stress-tested against real historical shock events
- [ ] Straddle/Strangle strategies on the same engine, for direct comparison
- [ ] Intraday (rather than EOD) stop-loss checking, to address the gap-risk finding above
- [ ] Old-format Bhavcopy parser, to extend historical coverage to pre-July-2024 dates
