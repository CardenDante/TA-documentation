# Trade Assistant Pro — EA Documentation

**Version:** 5.00
**Platform:** MetaTrader 5 (MQL5)
**Primary Instrument:** XAUUSD (Gold), multi-symbol capable
**Default Timeframe:** H1 analysis, H4 trend filter

---

## 1. Overview

Trade Assistant Pro is a fully self-sufficient Expert Advisor that analyzes the market using an 8-factor weighted confluence engine, opens and manages trades automatically, and streams its activity to a cloud dashboard for monitoring. The EA does **not** depend on the server to decide — it computes signals locally on the MT5 terminal. The dashboard is read-only telemetry.

Two engines run in parallel:

- **Swing engine** — H1 analysis, 3:1 R:R, held for hours.
- **Scalp engine** — M15 analysis, 1.8:1 R:R, held for minutes.

---

## 2. The 8-Factor Confluence Engine

Every bar, the EA scores the market in 8 categories. Total score (0–100) determines signal grade.

| Category | Weight | What it measures |
|---|---|---|
| **Trend** | 20 | EMA ribbon alignment (EMA 9/21/50/200) + HTF trend bias |
| **S&R** | 18 | Price position relative to pivot-based support/resistance zones |
| **Momentum** | 15 | RSI, StochRSI direction and extremes |
| **SMC** | 15 | Smart Money Concepts — Order Blocks, Fair Value Gaps, Liquidity Sweeps |
| **MACD** | 12 | Line vs signal crossover, histogram direction |
| **Volume** | 8 | Current volume vs 20-bar MA, spike detection |
| **Candle** | 7 | Pattern recognition (pin bar, engulfing, hammer, doji) |
| **Session** | 5 | London/NY overlap preference |

### Grade thresholds

| Grade | Score | Action |
|---|---|---|
| A+ | ≥80 | Highest-conviction — always trades |
| A | ≥65 | Strong — always trades |
| B+ | ≥55 | Default minimum — trades if score ≥ MinSignalScore |
| B | ≥45 | Weak — skipped by default |

---

## 3. Signal Computation — Step by Step

1. **Pull candles** — 300 bars of AnalysisTF + 300 bars of TrendTF + M15 scalp data.
2. **Calculate indicators** — EMA ribbon, RSI, StochRSI, MACD, Bollinger Bands, ATR, ADX, Volume MA.
3. **Detect structure** — Pivot-based S&R levels (min 2 touches), Order Blocks, FVGs, Liquidity Sweeps.
4. **Score each category** for BUY and SELL independently.
5. **Pick dominant direction** — whichever side scores higher.
6. **Apply HTF trend filter** — block counter-trend signals if `UseTrendFilter=true`.
7. **Apply ADX gate** — require minimum trend strength.
8. **Compute entry, SL, TP** — entry at market, SL at ATR × 1.5, TP at 3R.
9. **Fire if score ≥ MinSignalScore**.

---

## 4. Risk Management

Built-in guardrails — designed to preserve capital over chasing trades.

| Control | Default | Purpose |
|---|---|---|
| `MaxRiskPerTradePct` | 1.25% | Auto-sizes lot so SL distance × lot ≤ 1.25% of balance |
| `DailyMaxLossPct` | 4.5% | Halts new trades when today's loss crosses this threshold |
| `DailyProfitTargetPct` | 6.0% | Halts new trades when target hit (locks in the day) |
| `LossStreakSoftStop` | 3 losses | Halves lot size after 3 consecutive losses |
| `LossStreakHardStop` | 5 losses | Halts EA for the rest of the day |
| `CooldownAfterLossSec` | 60s | Forced wait after any loss before re-entering |
| `MaxPositions` | 8 | Hard cap on concurrent open positions |
| `MarginSafetyPct` | 70% | Leaves 30% free margin buffer |
| `MaxSpread` | 50 pts | Blocks entry if spread too wide |

### Position sizing

Lot size is auto-calculated each trade:
```
riskMoney = balance × MaxRiskPerTradePct / 100
lot       = riskMoney / (SL_distance × pointValue)
```
Then clamped to `BaseLotSize` as a floor and broker min/max as hard limits.

---

## 5. Trade Management

Once a trade is open, the EA actively manages it:

### Scaled Take Profit
- 60% of lots close at **TP1** (3R).
- 40% run to **TP2** (further extension).
- Controlled by `UseScaledTP` + `ScaledTPFirstPct`.

### Breakeven Shift
- When floating profit ≥ `BreakevenProfitUSD` (default $7), SL moves to entry + `BreakevenBufferPts`.
- Protects the trade from turning into a loss.

### Trailing Stop
- Arms when profit reaches `TrailStartPoints` (default 200 pts).
- Trails `TrailDistancePoints` (150 pts) behind price.
- Only re-writes SL when price advances by `TrailStepPoints` (20 pts).

### Re-entry
- When a position closes in profit, the EA can reopen on the next qualifying signal.
- `ReentryCooldownSec` (5 min default) prevents spam.
- `ReentryOnlyWhenFlat=true` requires no open positions before re-entering.

---

## 6. Scalp Engine

A parallel lightweight engine that runs on M15 for fast in-out trades.

| Setting | Default | Notes |
|---|---|---|
| `EnableScalp` | true | Master switch |
| `ScalpTF` | M15 | Analysis timeframe |
| `ScalpMinScore` | 50 | Lower bar than swing (more signals) |
| `ScalpCooldownMin` | 3 min | Faster re-fire than swing |
| `ScalpSL_ATR_Mult` | 0.8 | Tighter SL than swing (1.5) |
| `ScalpRR_Ratio` | 1.8 | Lower R:R — easier to hit TP |
| `ScalpMaxSpreadPts` | 25 | Stricter spread filter |
| `ScalpBE_USD` | $1.50 | Fast breakeven shift |
| `ScalpADX_Min` | 20 | Avoids range scalps |

Scalp trades are tagged separately in the dashboard and counted under the "scalp" section.

---

## 7. Auto-tune on Attach

When `AutoTuneOnAttach=true` (default), EA adapts parameters to the symbol's current volatility:

- `TrailStartPoints = ATR × 1.3`
- `TrailDistancePoints = ATR × 0.9`
- `MaxSpread = max(10, currentSpread × 3)`

This lets the same EA settings work on XAUUSD, EURUSD, GBPJPY without manual re-tuning.

---

## 8. Safe Attach Behavior

- `AdoptExistingOnAttach=false` (default): EA refuses to attach if open positions with the same magic number already exist. Protects against accidental double-management.
- Set `true` only when intentionally reattaching after disconnect.

---

## 9. Safety & Design Philosophy

- **Never trades without a valid license** — hard block, not warning.
- **Daily loss cap** stops the EA before a bad day becomes a disaster.
- **Loss-streak circuit breakers** halve size at 3 losses, halt at 5.
- **Breakeven + trailing** protect open winners.
- **Scaled TP** lets half the book book early, half run.
- **HTF trend filter** blocks counter-trend noise.
- **Auto-tune** adapts risk to each symbol's volatility.
- **Magic number guard** prevents accidental double-management.

---

*© Trade Assistant Pro — v5.00*
