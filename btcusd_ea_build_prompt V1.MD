# MASTER PROMPT — BTCUSD Adaptive Multi-Strategy Scalping EA (MQL5)

> Paste this whole prompt into Claude/Gemini/etc. Use it in ONE conversation, and build the EA in the phased stages described at the bottom rather than asking for everything in one shot — see "Build Order" at the end for why.

---

## ROLE

You are a senior quantitative trading systems engineer with deep MQL5/MetaTrader 5 expertise, specializing in crypto CFD execution, volatility-adaptive risk management, and regime-based strategy routing. You write production-grade, defensively-coded MQL5. You do not write pseudocode, placeholders, or "TODO" sections. Every function you write must be complete and compile-ready.

## OBJECTIVE

Build a single MQL5 Expert Advisor for **BTCUSD** (CFD/crypto-margin instrument on MT5) that:

1. Classifies the current market regime using objective, falsifiable rules (not vague heuristics).
2. Routes execution to exactly one active strategy module at a time based on that regime — never runs conflicting strategies simultaneously.
3. Sizes every trade based on account size tier, current volatility, and current drawdown state.
4. Manages exits with ATR-based stops, structure-based stops, partial profit scaling, and trailing logic.
5. Enforces hard account-protection limits (daily/weekly loss limits, consecutive loss cooldown, kill switch).
6. Logs every decision (regime detected, confidence score, strategy selected, risk calculated) to the Experts log and optionally to a CSV file for later analysis.

## NON-NEGOTIABLE CONSTRAINTS

- Output must be **valid, complete MQL5 code that compiles in MetaEditor with zero errors and zero warnings**, targeting MT5 build 3800+.
- No external APIs, no DLLs, no WebRequest calls, no Python bridges. Fully self-contained in native MQL5.
- No martingale, no grid recovery, no "double down after loss" logic anywhere, under any label.
- No curve-fitted magic numbers without justification — every constant (e.g., ATR multiplier, RSI threshold) must be exposed as an `input` parameter with a sane default and an inline comment explaining its purpose.
- Do not invent indicator behavior. Only use indicators and functions that actually exist in MQL5 (iATR, iADX, iRSI, iBands, iMA, CopyTicks, CopyRates, etc.) with correct handle creation and `IndicatorRelease` cleanup in `OnDeinit`.
- Do not claim the EA is "machine learning" — it is a rules-based weighted scoring system. Name things accordingly in code comments (e.g., `CalculateRegimeConfidenceScore()`, not `MLPredict()`).
- All money/lot calculations must account for: `SYMBOL_VOLUME_MIN`, `SYMBOL_VOLUME_MAX`, `SYMBOL_VOLUME_STEP`, broker stop level (`SYMBOL_TRADE_STOPS_LEVEL`), and freeze level (`SYMBOL_TRADE_FREEZE_LEVEL`). If a calculated lot size or stop distance violates broker constraints, skip the trade — never force an invalid order.
- Handle `OrderSend`/`CTrade` failures explicitly (requotes, no money, invalid stops) with retry logic capped at a configurable max retry count, then abort and log.
- Use `CTrade`, `CPositionInfo`, `CSymbolInfo` from the Standard Library rather than raw low-level calls, unless there's a specific reason not to (state the reason in a comment if you deviate).

## ACCOUNT-ADAPTIVE RISK ENGINE (REQUIRED MODULE)

Implement risk scaling using account equity tiers, not a flat percentage:

```
input double InpSmallAccountThreshold   = 2000.0;   // below this = tight risk mode
input double InpLargeAccountThreshold    = 50000.0;  // above this = scale-mode
input double InpRiskPercent_Small        = 0.3;      // % risk per trade, small accounts
input double InpRiskPercent_Mid          = 0.75;     // % risk per trade, mid accounts
input double InpRiskPercent_Large        = 0.5;      // % risk per trade, large accounts (lower %, diversify instead)
input int    InpMaxConcurrentTrades_Small = 1;
input int    InpMaxConcurrentTrades_Mid   = 3;
input int    InpMaxConcurrentTrades_Large = 6;
```

Required logic:
- Lot size = `(EquityNow * RiskPercentForTier / 100) / (StopDistancePoints * TickValue)`, then normalized to `SYMBOL_VOLUME_STEP` and bounds-checked against min/max.
- If normalized lot size would represent **more than 1.5x the intended risk %** due to minimum-lot rounding (common on small accounts), **skip the trade** and log the reason — do not silently over-risk.
- Recalculate position size against **current equity**, not initial deposit, on every new trade (so the system de-risks after drawdown automatically).
- Volatility-adjust: when ATR(14) on the entry timeframe is above its 90-day average by a configurable multiple, reduce position size by a configurable factor (default 50%) regardless of account tier.
- Large accounts: implement order splitting — if calculated lot size exceeds a configurable `InpMaxSingleOrderLots`, split into N tranches submitted with a configurable delay (default 2–5 seconds apart) rather than one large market order.

## HARD ACCOUNT PROTECTION (REQUIRED MODULE)

```
input double InpDailyLossLimitPercent    = 3.0;
input double InpWeeklyLossLimitPercent   = 7.0;
input double InpMonthlyLossLimitPercent  = 15.0;
input int    InpMaxConsecutiveLosses     = 4;
input double InpMaxTotalOpenRiskPercent  = 4.0;  // sum across all open positions
input bool   InpKillSwitchEnabled        = true;
```

- Track realized daily/weekly/monthly P&L against starting equity for each period, reset at correct boundaries (broker server time).
- On limit breach: close no open positions forcibly (let existing stops manage them) but **block all new entries** until the next period resets, and log the breach clearly.
- Consecutive loss counter resets on any win; on breach, pause new entries for a configurable cooldown (default: rest of trading day).
- Kill switch: a manual `input bool` plus an automatic trigger if equity drawdown from peak exceeds a configurable hard ceiling (default 20%) — disables all new order placement for the remainder of the session, requires manual EA restart to resume.

## REGIME DETECTION (BUILD ONLY THESE TWO FIRST — see Build Order)

Implement as a function returning an enum and a 0–100 confidence score:

```cpp
enum ENUM_MARKET_REGIME
{
   REGIME_TREND_BULL,
   REGIME_TREND_BEAR,
   REGIME_RANGE,
   REGIME_HIGH_VOLATILITY,
   REGIME_UNDEFINED   // confidence too low to act — EA stays flat
};
```

Define **explicit, non-overlapping numeric criteria** for each regime (do not leave this vague):

- **Trend (bull/bear)**: ADX(14) > `InpADXTrendThreshold` (default 25) AND price > EMA(50) > EMA(200) for bull (reverse for bear) on the H1 timeframe, confirmed by at least 2 of the last 3 swing highs/lows being higher (bull) or lower (bear) on M15.
- **Range**: ADX(14) < `InpADXRangeThreshold` (default 18) AND price has touched both upper and lower Bollinger Band (20,2) at least once in the last `InpRangeLookbackBars` (default 40) bars without a sustained close beyond either band.
- **High volatility**: current ATR(14) > `InpVolatilityMultiplier` (default 1.8) × the 90-period average ATR. This regime **overrides** trend/range classification and routes to either the high-volatility module or a full stand-aside, per `InpTradeDuringHighVol` setting.
- If none of the above thresholds are clearly met, return `REGIME_UNDEFINED` and **do not trade**. Never force a classification.

Log the regime, the specific values that triggered it, and the confidence score on every new bar.

## STRATEGY MODULES (BUILD ONLY ONE FIRST)

### Module 1 — Trend Continuation Scalper (build this first, fully, before any other module)
- Active only when `REGIME_TREND_BULL` or `REGIME_TREND_BEAR` with confidence ≥ `InpMinRegimeConfidence` (default 65).
- Entry: pullback to EMA(21) on M5 in direction of H1 trend, confirmed by RSI(14) crossing back above 40 (bull) / below 60 (bear), with M1 bullish/bearish engulfing or pin bar at the EMA touch.
- Stop loss: `1.5 × ATR(14, M5)` beyond the pullback swing point.
- Take profit: first scale-out at `1.0R`, remainder trailed by `1.0 × ATR(14, M5)` chandelier-style.
- Do not add Module 2 (range) or any other module in the same build pass — confirm Module 1 compiles, backtests without errors, and produces sane trade logs first.

### Module 2 — Range Mean-Reversion Scalper (build only after Module 1 is validated)
- Active only during `REGIME_RANGE` with confidence ≥ threshold.
- Entry at Bollinger Band(20,2) extreme + RSI(14) divergence confirmation.
- Stop beyond the band extreme by `0.5 × ATR`. Target the opposite band or midline, whichever is configurable.

(Additional modules — breakout, liquidity sweep, session filters — are explicitly **out of scope for this build pass**. Only implement if requested in a follow-up after Modules 1–2 are proven.)

## EXECUTION QUALITY

- Before sending any order: check current spread against `InpMaxSpreadPoints`; skip trade if exceeded.
- Check for `TRADE_RETCODE_REQUOTE` and `TRADE_RETCODE_PRICE_CHANGED`; retry up to `InpMaxRetries` with fresh price quote each attempt.
- Skip trading during a configurable number of seconds around server-side abnormal tick gaps (price jump > `InpAbnormalGapATRMultiple × ATR` between consecutive ticks) — this is the EA's news-shock proxy, since no external news API is used.

## LOGGING & SELF-ANALYTICS

- On every closed trade, append to a CSV in `MQL5/Files/`: timestamp, regime at entry, strategy module, confidence score, lot size, entry/exit price, R-multiple result, account equity at time of trade.
- Implement a simple `OnTester()` function returning a custom optimization criterion (e.g., profit factor weighted by minimum trade count) so the Strategy Tester optimizer can be used meaningfully later — but do not auto-optimize live.

## WHAT TO OUTPUT

- A single `.mq5` file, fully commented, organized into clearly labeled `#region` blocks (Inputs, Globals, Regime Detection, Risk Engine, Strategy Modules, Order Execution, Logging, Event Handlers).
- No explanatory text outside the code block except a short header comment in the file itself describing scope and version.
- If any requested feature cannot be implemented validly in native MQL5 without external APIs or invented functions, **do not fake it** — implement the closest honest equivalent and say so in a comment, rather than silently omitting or hallucinating a working version.

---

## BUILD ORDER (do not request the whole EA in one prompt)

Send this master prompt once for context, then proceed in this sequence across separate follow-up messages, each one building on the last and asking for a compile-check:

1. **Skeleton + Risk Engine + Account Protection module only** (no trading logic yet) — verify it compiles and tracks equity/limits correctly in Strategy Tester with no entries.
2. **Add Regime Detection** (trend/range/high-vol only) — verify via logs that regime labeling on historical BTCUSD bars looks sane before adding any execution.
3. **Add Module 1 (Trend Continuation)** — backtest in isolation, review trade log CSV, sanity-check stop/target behavior.
4. **Add Execution Quality + Logging refinements** based on what backtest #3 reveals.
5. **Add Module 2 (Range)** only once Module 1 is stable, and verify the regime router never lets both modules fire on the same bar.
6. Everything else (breakout/liquidity-sweep modules, session intelligence, self-optimization) is a separate, later project — do not attempt it until 1–5 are proven on real historical data and ideally a forward-test demo period.

---

# PHASE 2 ADDENDUM — Institutional Upgrade Layer

**Gate condition: do not start Phase 2 until Phase 1 (steps 1–5 above) has run on a demo account for a minimum statistically meaningful sample — at least 50-100 closed trades — with results roughly consistent with backtest. If live results diverge significantly from backtest, fix Phase 1 before adding any of this.**

Append this section to the master prompt only once that gate is met. It upgrades the regime/strategy router into a multi-layer probability fusion system, replacing the simple `ENUM_MARKET_REGIME` with a richer state model — but every new subsystem must be added and validated **one at a time**, same discipline as Phase 1.

## Important calibration note for the AI before it writes any of this

Several requested components (Fair Value Gaps, breaker blocks, mitigation blocks, "liquidity hunt targeting") are retail ICT/SMC pattern concepts, not exchange-validated order-flow facts. MT5's BTCUSD feed is a broker's CFD price stream, not the actual Bitcoin exchange order book — there is no real bid/ask depth or executed-order data available. Implement these as clearly-labeled **heuristic pattern detectors and proxy scores**, not as claims of institutional order-flow truth. Name functions and comments accordingly (e.g., `DetectThreeBarGapPattern()` not `DetectInstitutionalFVG()`), so nobody mistakes a geometric pattern match for verified smart-money behavior. Any "0-100 quality score" must be described in a comment as an internal heuristic weighting, not a validated probability.

## 2A — Market State Engine (upgrade from Phase 1 regime enum)

```cpp
enum ENUM_MARKET_STATE
{
   STATE_TREND_EXPANSION,
   STATE_TREND_CONTINUATION,
   STATE_TREND_EXHAUSTION,
   STATE_RANGE_ACCUMULATION,
   STATE_RANGE_DISTRIBUTION,
   STATE_VOLATILITY_EXPANSION,
   STATE_VOLATILITY_CONTRACTION,
   STATE_TRANSITION,
   STATE_UNDEFINED
};

struct MarketStateResult
{
   ENUM_MARKET_STATE State;
   double            Confidence;   // 0-100, weighted-evidence based, never forced
};
```
Define explicit numeric criteria for each state (extend Phase 1's trend/range/vol rules — e.g., `TREND_EXHAUSTION` = trend state active AND momentum indicator (RSI or MACD histogram) diverging against price direction for N consecutive swings). `STATE_TRANSITION` is returned when two states score within a configurable margin of each other — do not force a tie-break.

*(Drop `STATE_LIQUIDITY_HUNT` from the original list as its own market state — liquidity sweep detection belongs in section 2B as an input signal to the state engine, not a state itself, since "a sweep happened" and "what regime is this" are different questions.)*

## 2B — Liquidity Mapping (heuristic, not true order-flow)

```cpp
struct LiquidityZone
{
   double   Price;
   double   Strength;     // 0-100 heuristic score, see weighting below
   datetime Created;
   bool     Swept;
};
```
Track: prior daily/weekly highs-lows, session highs-lows, equal highs/lows (price matches within a configurable tolerance in points), and untested swing points. Strength score weights: number of touches, age (older = lower weight, configurable decay), distance from current price (closer = higher relevance weight). Cap the liquidity zone array at a configurable max count (e.g., 50) and prune oldest/weakest when exceeded — unbounded growth will degrade performance over a long-running EA.

## 2C — Three-Bar Gap Pattern Detector (renamed from "FVG Engine" per calibration note above)

```cpp
struct PriceGapZone
{
   double UpperBound;
   double LowerBound;
   double Size;
   bool   Filled;
};
```
Bullish: `Low[bar-2] > High[bar]` (3-bar gap, indices per your bar-shift convention — verify against actual array direction before shipping). Bearish: mirrored. Quality heuristic (0-100) from: gap size relative to ATR, speed of formation, proximity to a tracked liquidity zone from 2B. This score is an internal filter weight only — validate empirically in backtest before trusting it; do not assume it has predictive power until tested.

## 2D — Tick-Based Pressure Proxy (renamed from "Order Flow Engine")

Using `CopyTicks`/`CopyTicksRange`: ticks-per-second, directional tick imbalance (count of upticks vs downticks in lookback window), micro-range expansion. Output a 0-100 `AggressionScore` and a bullish/bearish/neutral `PressureBias`. Label this explicitly in code comments as a **retail tick-count proxy**, since true exchange volume/order book data is not available through this feed.

## 2E — Volatility Composite Index

Replace single-ATR volatility checks with a weighted blend of: ATR, ATR percentile rank (vs trailing N-period history), realized range expansion, tick frequency, and spread expansion. Output discrete states: `VOL_SUPPRESSED / VOL_NORMAL / VOL_EXPANDING / VOL_EXTREME / VOL_CHAOTIC`, each with configurable thresholds. Each state modifies (via lookup, not hardcoded branches): stop distance multiplier, position size multiplier, target distance multiplier, and which strategy modules are eligible to fire.

## 2F — Structure Engine V2

Full swing-point tracker for HH/HL/LH/LL with a `StructureStrength` score from: swing distance (in ATR units), speed of move (bars elapsed), retracement depth (Fibonacci-style ratio of pullback to prior leg). Classify `STRUCTURE_BULL / STRUCTURE_BEAR / STRUCTURE_TRANSITION / STRUCTURE_RANGE`. This feeds into 2A's state engine as one weighted input, not a standalone trading signal.

## 2G — Probability Fusion Engine

```cpp
double FuseTradeProbability(double structureScore, double liquidityScore,
                             double volatilityScore, double pressureScore,
                             double sessionScore, double gapPatternScore)
{
   // Weights must be input parameters, not hardcoded — they need to be
   // tunable and walk-forward tested, not asserted as institutional truth.
   return structureScore   * InpWeight_Structure  +
          liquidityScore   * InpWeight_Liquidity   +
          volatilityScore  * InpWeight_Volatility  +
          pressureScore    * InpWeight_OrderFlow   +
          sessionScore     * InpWeight_Session     +
          gapPatternScore  * InpWeight_GapPattern;
}
```
Only allow trade evaluation to proceed past this gate when the fused score exceeds `InpMinTradeProbability` (start conservative — test at 80+ before lowering). **Treat the initial weight values as a hypothesis to be walk-forward tested, not a finished calibration** — these weights are the single highest overfitting risk in the whole system and must be validated out-of-sample before being trusted.

## 2H — Confidence-Tiered Dynamic Risk

```
// Risk-per-trade as a function of fused probability score, layered on top
// of the Phase 1 account-tier risk engine (multiply together, don't replace):
// <70   => 0R (no trade)
// 70-80 => 0.25R of the Phase-1 tier-based risk amount
// 80-90 => 0.50R
// 90+   => 1.00R
```

## 2I — Strategy Health Engine

Per strategy module, per market state, per session: track win rate, expectancy, profit factor, average R, drawdown, recovery factor over a rolling trade window. Auto-rules: if profit factor < 1.0 over the last 30 trades for a given module, halve its risk allocation. If profit factor < 0.8 over the last 50 trades, disable that module entirely until manually re-enabled. Persist this state across EA restarts (write/read from a file) so a restart doesn't erase a justified disablement.

## 2J — Meta-Router

```cpp
ENUM_STRATEGY_CHOICE SelectBestStrategy(MarketStateResult state, double fusedProbability,
                                          StrategyHealthSnapshot health[]);
```
The router — never the strategies themselves — decides which module (if any) gets control on a given bar. It must check: state compatibility, fused probability threshold, and that module's current health status (skip disabled modules even if otherwise eligible). Default return is `STRATEGY_STAND_ASIDE` — this must be the fallback, not an afterthought, whenever conditions are ambiguous, conflicting, or execution quality (spread/slippage per Phase 1) has degraded.

## Phase 2 build order (same incremental discipline as Phase 1)

1. Build 2E (Volatility Composite) and 2F (Structure V2) first — they extend Phase 1 logic most directly and are the most rigorously definable.
2. Build 2A (Market State engine) consuming 2E/2F outputs — validate via logs against your own visual chart review before trusting it.
3. Build 2I (Strategy Health) and 2H (tiered risk) — these protect capital and should exist before adding more signal sources.
4. Only then build 2B/2C/2D (liquidity zones, gap patterns, tick pressure) as additional probability-fusion inputs — treat each as an experimental filter to be backtested for actual predictive value before assuming it improves results. It's entirely possible some of these add noise rather than edge; be willing to set their fusion weight to zero if testing shows that.
5. Build 2G (Probability Fusion) and 2J (Meta-Router) last, once all component scores exist and have been individually sanity-checked.
6. Do not enable Phase 2 on a live or larger demo account until walk-forward testing shows the fused-probability threshold and weights are stable across multiple out-of-sample periods, not just the period they were tuned on.
