# Structure-Based Profile Anchoring: Architectural Analysis
## Market Structure Volume Profile Indicator v1.0.5

**Document Type**: System Architecture Overview
**Date**: 2025-01-17
**Focus**: Structure Mode Design Philosophy & Implementation

---

## Executive Summary

This document provides a comprehensive architectural analysis of the **Structure-based profile anchoring system**, bridging technical implementation with trading concepts. It explains why market structure is used for profile anchoring, how components interact, what states exist, and the critical timing decisions that enable non-repainting behavior.

**Key Insight**: Structure mode treats volume profiles as **market-phase containers** rather than time-based intervals, creating profiles that adapt to the natural rhythm of market behavior rather than arbitrary time boundaries.

---

## 1. Conceptual Model: What is "Market Structure" in Trading?

### 1.1 Market Structure Fundamentals

**Market Structure** is the pattern of higher highs (HH), higher lows (HL), lower highs (LH), and lower lows (LL) that define market trends:

```
Bullish Structure:
HH → HL → HH → HL    (uptrend)
 ↑    ↑    ↑    ↑
Each swing point confirms trend

Bearish Structure:
LH → LL → LH → LL    (downtrend)
 ↑    ↑    ↑    ↑
Each swing point confirms trend
```

**Structure Break**: When the pattern changes (e.g., HH-HL sequence breaks into LH-LL), signaling a potential trend reversal.

### 1.2 Why Use Structure for Profile Anchoring?

**Trading Rationale:**
1. **Phase-Based Analysis**: Markets move in phases (accumulation, markup, distribution, markdown). Structure breaks mark phase transitions.
2. **Value Context**: Volume accumulated during a phase reveals where institutions established positions.
3. **Support/Resistance**: POC and Value Areas from structure-based profiles often act as future support/resistance.
4. **Non-Arbitrary Boundaries**: Time-based profiles (e.g., daily) cut through market moves arbitrarily. Structure profiles respect market flow.

**Example Scenario:**
```
Time-Based Profile:
|--Day 1--|--Day 2--|
    ↑ Cuts through middle of rally

Structure-Based Profile:
|--Accumulation--|--Markup--|--Distribution--|
       ↑ Profiles align with market phases
```

### 1.3 Business Logic: Why Traders Care

**Use Cases:**
1. **Trend Following**: Enter on pullbacks to POC from previous structure phase
2. **Reversal Trading**: Fade value areas when structure breaks
3. **Institutional Footprint**: Heavy volume at POC = institutional interest
4. **Dynamic Support/Resistance**: VAH/VAL update as structure evolves

**Edge**: Structure-based profiles provide **forward-looking context** by identifying where value was established during the last confirmed market phase.

---

## 2. High-Level Architecture: Structure Mode Design

### 2.1 System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STRUCTURE-BASED ANCHORING SYSTEM                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │ Pivot        │──▶│  Structure   │──▶│  Profile     │            │
│  │ Detector     │   │  Analyzer    │   │  Manager     │            │
│  │ (LibPvot)    │   │ (LibBrSt)    │   │ (LibVPrf)    │            │
│  └──────────────┘   └──────────────┘   └──────────────┘            │
│         │                   │                   │                    │
│         ▼                   ▼                   ▼                    │
│  Price Pivots         Structure State     Volume Profiles           │
│  (HH/HL/LH/LL)       (Bullish/Bearish)   (POC/VA/VWAP)             │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                EVENT FLOW (Structure Mode)                    │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │  1. Price forms swing high/low                               │  │
│  │  2. LibPvot confirms pivot (after rightLen bars)             │  │
│  │  3. LibBrSt analyzes pivot sequence → structure state        │  │
│  │  4. Structure state change detected → ANCHOR RESET           │  │
│  │  5. New volume profile begins accumulation                   │  │
│  │  6. Developing metrics (POC/VA/VWAP) update each bar         │  │
│  │  7. Process repeats on next structure break                  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Interactions

#### Phase 1: Pivot Detection (LibPvot)
```pine
// Lines 506-511
[_, _, _, _, _, _, prcTrendState] = LibPvot.marketStructure(
  highSrc  = high,
  lowSrc   = low,
  leftLen  = pivLe,   // Strength: bars to left
  rightLen = pivRi,   // Confirmation: bars to right
  srcTol   = pivTol)  // Price tolerance for "equal" pivots
```

**Inputs**: Raw price (high/low) + user parameters (pivot strength, confirmation bars)
**Output**: `prcTrendState` - Enum representing current market structure (HH, HL, LH, LL, or transition states)

#### Phase 2: Structure Analysis (LibBrSt)
```pine
// Lines 513-514
if prcTrendState != prcTrendState[1]
    prfReset := true
```

**Logic**: Compare current bar's structure state to previous bar
**Event**: State change triggers profile reset (new anchor)

**Key Insight**: Structure breaks are **instantaneous events** - no gradual transition. This is why Structure mode can plot in real-time without full pivot lag.

#### Phase 3: Profile Management (LibVPrf)
```pine
// Lines 602-607
if prfReset
    resetBar := bar_index
    msProf.clear()
    msProf.setRanges(
      rangeUp = nz(rangeUp[delay], ...),
      rangeLo = nz(rangeLo[delay], ...))
```

**Actions on Reset:**
1. Clear existing profile data
2. Set new price range (high/low bounds)
3. Begin accumulating volume from current bar forward

---

## 3. State Machine: Structure Mode States & Transitions

### 3.1 State Diagram

```
                    ┌──────────────────────────┐
                    │   ACCUMULATION PHASE     │
                    │  (Volume Building)       │
                    │  - Profile grows         │
                    │  - POC/VA update         │
                    │  - Developing lines plot │
                    └──────────┬───────────────┘
                               │
                               │ Structure state changes
                               │ (HH→LH or LL→HL)
                               ▼
                    ┌──────────────────────────┐
                    │   ANCHOR RESET EVENT     │
                    │  - Finalize old profile  │
                    │  - Clear buffers         │
                    │  - Set new range         │
                    └──────────┬───────────────┘
                               │
                               │ resetBar = bar_index
                               ▼
                    ┌──────────────────────────┐
                    │   NEW ACCUMULATION       │
                    │  (Cycle repeats)         │
                    └──────────────────────────┘
```

### 3.2 State Variables

```pine
// State tracking variables (Lines 488-533)
var series int resetBar = na         // Bar index of last anchor reset
var series int prevResetBar = 0      // Previous profile start (for capture)
var bool isFirstReset = true         // First profile flag
series bool prfReset = false         // Reset trigger (this bar)
series int delay = na                // Lag offset (0 for Structure mode)
```

### 3.3 State Transitions

#### Transition: Accumulation → Reset

**Trigger**: `prcTrendState != prcTrendState[1]` (Line 513)

**Conditions Met:**
1. Pivot confirmed (after `rightLen` bars)
2. Pivot sequence creates structure break
3. Structure state enum changes value

**Actions Executed:**
```pine
// Lines 540-607
1. Capture historical profile boundaries (startBar, endBar)
2. Store peak detection data (for rectangle rendering)
3. Update resetBar to current bar_index
4. Clear msProf buffers
5. Set new price range for incoming bars
6. Begin volume accumulation
```

#### Transition: Reset → Accumulation

**Trigger**: First bar after `prfReset = true`

**Actions**:
```pine
// Lines 620-631
1. Merge intra-bar volume data into profile
2. Update developing metrics (POC, VA, VWAP, StdDev)
3. Plot developing lines with offset
4. Continue until next structure break
```

---

## 4. Timing Architecture: When Events Occur

### 4.1 Critical Timing Constants

```pine
// Line 504 (Structure mode)
delay := 0  // No lag offset for real-time plotting
```

**vs Swing Mode:**
```pine
// Lines 490-491 (Swing mode)
delay := pivRi  // Full pivot confirmation lag (e.g., 5 bars)
```

### 4.2 Event Timeline (Structure Mode)

```
Bar Index:  100    101    102    103    104    105    106    107
            ─────────────────────────────────────────────────────
Pivot:                    [HH]                  [HL]
                           ↓                     ↓
Confirmed:                        ✓                    ✓
                                  (at 104)             (at 107)
Structure:  Bullish    Bullish   Bullish→Bearish   Bearish
                                      ↑
                                   BREAK (at 104)
Profile:    |──────Accumulating──────|  RESET  |──New Profile──
                                         ↑
                                      Line 513: prcTrendState changed
```

**Key Observations:**
1. Structure break detected **immediately** when pivot confirms (bar 104)
2. No additional lag - profile resets **on the confirmation bar**
3. Developing lines plot with `offset = -pivRi` for non-repainting (Lines 1125-1177)

### 4.3 Historical vs Real-Time Behavior

#### Historical Execution (Backtest)
```pine
// All bars are "closed" - no bar_index changes mid-bar
// Lines 620-652: Profile accumulation executes sequentially
// Structure breaks retroactively applied (already confirmed)
```

#### Real-Time Execution (Live Chart)
```pine
// Current bar updates continuously
// Lines 654-673: flashProf merges real-time data
// Structure state can change intra-bar if pivot forms
// Developing lines use [offset] to show non-repainting values
```

**Non-Repainting Mechanism:**
```pine
// Line 675
series int offset = profAnchor == ProfAnchor.swing ? 0 : pivRi

// Lines 1125-1177 (example: VA plot)
vaUpPl = plot(
  series = showDevVA ? vaUp[offset] : na,  // ← Uses historical value
  offset = -pivRi)                         // ← Shifts plot backward
```

**Effect**: On live bars, plotted line shows value from `pivRi` bars ago, preventing repainting as current bar updates.

---

## 5. Structure vs Swing vs Delta: Architectural Comparison

### 5.1 Anchoring Philosophy

| Mode      | Anchor Event                     | Trading Concept                  |
|-----------|----------------------------------|----------------------------------|
| Swing     | Impulse baseline shift (delta)   | Momentum change (buying/selling) |
| Structure | Price structure break (HH→LH)    | Trend phase transition           |
| Delta     | Cumulative delta structure break | Order flow trend change          |

### 5.2 Code Architecture Differences

#### Swing Mode (Lines 490-502)
```pine
delay := pivRi  // Full lag for retroactive plotting

series bool pivPrcHi = not na(ta.pivothigh(high, pivLe, pivRi))
series bool pivPrcLo = not na(ta.pivotlow (low,  pivLe, pivRi))

var series float impulseStart = na
if pivPrcHi or pivPrcLo
    impulseStart := switch pivPrcHi
        true  => math.min(nz(impulseStart), math.max(CumO[pivRi], CumC[pivRi]))
        false => math.max(nz(impulseStart), math.min(CumO[pivRi], CumC[pivRi]))

if impulseStart != impulseStart[1]
    prfReset := true
```

**Characteristics:**
- **Data Source**: Cumulative volume delta (CVD) pivots
- **Lag**: Full `pivRi` bars (retroactive to pivot bar)
- **Philosophy**: Profiles align with buying/selling pressure shifts
- **Use Case**: Scalping, order flow traders

#### Structure Mode (Lines 503-514)
```pine
delay := 0  // No lag offset

[_, _, _, _, _, _, prcTrendState] = LibPvot.marketStructure(
  highSrc  = high,
  lowSrc   = low,
  leftLen  = pivLe,
  rightLen = pivRi,
  srcTol   = pivTol)

if prcTrendState != prcTrendState[1]
    prfReset := true
```

**Characteristics:**
- **Data Source**: Price pivots (high/low)
- **Lag**: Partial (pivot confirms with `pivRi`, but plot is real-time)
- **Philosophy**: Profiles align with market structure phases
- **Use Case**: Swing trading, trend following, institutional analysis

#### Delta Mode (Lines 515-526)
```pine
delay := 0  // No lag offset

[_, _, _, _, _, _, cvdTrendState] = LibPvot.marketStructure(
  highSrc  = CumH,
  lowSrc   = CumL,
  leftLen  = pivLe,
  rightLen = pivRi,
  srcTol   = pivTol)

if cvdTrendState != cvdTrendState[1]
    prfReset := true
```

**Characteristics:**
- **Data Source**: Cumulative delta pivots (CumH/CumL)
- **Lag**: Partial (same as Structure mode)
- **Philosophy**: Profiles align with order flow structure breaks
- **Use Case**: Volume analysis, smart money tracking

### 5.3 Timing Comparison Table

| Aspect               | Swing                  | Structure              | Delta                  |
|----------------------|------------------------|------------------------|------------------------|
| **Anchor Detection** | Pivot + delta shift    | Pivot + state change   | Delta pivot + state    |
| **Profile Start**    | Retroactive to pivot   | Real-time confirmation | Real-time confirmation |
| **Developing Lines** | Plot with full lag     | Plot with offset       | Plot with offset       |
| **Alert Timing**     | Delayed to pivot bar   | Real-time on break     | Real-time on break     |
| **Repaint Behavior** | None (retroactive)     | Non-repainting offset  | Non-repainting offset  |

---

## 6. Library Abstraction: Encapsulation of Complexity

### 6.1 LibPvot: Pivot Detection & Structure Analysis

**Purpose**: Abstract pivot confirmation logic and structure state machine

**Key Functions**:
```pine
// marketStructure() - Main structure detection function
// Returns: [swingHi, swingLo, pivHi, pivLo, brStUp, brStLo, trendState]
//
// trendState enum values:
//   - Higher High (HH)
//   - Higher Low (HL)
//   - Lower High (LH)
//   - Lower Low (LL)
//   - Transition states
```

**Abstraction Benefits**:
1. Hides complex pivot buffering logic
2. Manages state transitions internally
3. Provides clean enum-based API
4. Reusable across modes (Structure/Delta)

### 6.2 LibBrSt: Breakout/Structure Detection

**Purpose**: Statistical models for price distribution and volume estimation

**Key Functions**:
```pine
// PriceEst enum: t4Skew, t3Skew, triangle, uniform
// Used in PDF allocation (Lines 216-218)
//
// Provides probability density functions for:
// - Volume distribution across candle range
// - Statistical models (T4 distribution, skewed triangular, etc.)
```

**Abstraction Benefits**:
1. Advanced statistical models encapsulated
2. User-selectable estimators via enum
3. Separation of concerns (stats vs rendering)

### 6.3 LibVPrf: Volume Profile Management

**Purpose**: Volume profile calculation and data structure management

**Key Functions**:
```pine
// createProfile() - Factory function (Lines 421-432)
// merge() - Add bar data to profile
// getPoc() - Calculate Point of Control
// getVA() - Calculate Value Area High/Low
// getVwap() - Calculate Volume Weighted Average Price
// getStdDev() - Calculate Standard Deviation
// getBktBnds() - Get bucket price boundaries
// getBktBuyVol() - Get bucket buy volume
// getBktSellVol() - Get bucket sell volume
```

**Data Structure**:
```pine
// LibVPrf.VProf object encapsulates:
// - Bucket arrays (buy/sell volume per price level)
// - Range values (rangeUp, rangeLo)
// - Cumulative metrics (CVD, CVDH, CVDL)
// - Configuration (allot mode, estimator, steps)
```

**Abstraction Benefits**:
1. Complex volume distribution hidden
2. Efficient bucket management (dynamic sizing)
3. Metric calculations optimized internally
4. Clean API for indicator logic

---

## 7. Event-Driven Architecture: Structure Breaks as Events

### 7.1 Event Model

**Event**: Structure state change (`prcTrendState != prcTrendState[1]`)

**Event Properties:**
- **Atomic**: State changes instantly (no partial transitions)
- **Observable**: Detected via comparison to previous bar
- **Actionable**: Triggers profile reset workflow

### 7.2 Event Handlers

```pine
// Event Handler: Profile Reset (Lines 540-607)
if prfReset
    // Handler logic:
    // 1. Capture historical data
    // 2. Update state variables
    // 3. Clear buffers
    // 4. Initialize new profile
```

**Handler Characteristics:**
- **Synchronous**: Executes immediately when event fires
- **Idempotent**: Multiple calls with same resetBar have no effect
- **Stateful**: Updates global state (resetBar, prevResetBar)

### 7.3 Event Flow Diagram

```
Price Action → Pivot Detection → Structure Analysis → Event Emission
                                        ↓
                                  State Change?
                                        ↓
                                  Yes: prfReset = true
                                        ↓
                               ┌────────┴────────┐
                               │                 │
                        Capture Handler    Reset Handler
                        (Historical)       (Profile Init)
                               │                 │
                               └────────┬────────┘
                                        ↓
                                  New Accumulation
```

---

## 8. Non-Repainting Implementation: Critical Design Decision

### 8.1 The Repainting Problem

**Challenge**: Real-time bars update continuously, causing plotted values to change.

**Example (Without Offset)**:
```
Bar 100: POC = 50,000 (plotted)
Bar 100 (5 seconds later): POC = 50,020 (repaints - confusing!)
Bar 100 (closes): POC = 50,015 (final value)
```

### 8.2 Solution: Offset-Based Historical Plotting

```pine
// Lines 675, 1125-1177
series int offset = profAnchor == ProfAnchor.swing ? 0 : pivRi

plot(
  series = showDevVA ? vaUp[offset] : na,  // Use historical value
  offset = -pivRi)                         // Shift plot backward
```

**Mechanism:**
1. `vaUp[offset]` = Value from `pivRi` bars ago (confirmed, won't change)
2. `offset = -pivRi` = Shift plot backward to align with historical position
3. Result: Plotted line shows **confirmed** value, not live updating value

**Visual Effect:**
```
Time:       100    101    102    103    104    105 (current bar)
                                              ↑
                                           Calculating live POC
Plot:       POC₁₀₀  POC₁₀₁  POC₁₀₂ (confirmed values)
                              ↑
                          Plotted 3 bars behind (pivRi=3)
```

### 8.3 Alert Timing Trade-Off

**Alert Logic** (Lines 1185-1238):
```pine
alertcondition(
  condition = enableAlerts and ta.crossover(close[delay], poc),
  ...)
```

**Trade-Off:**
- **Structure/Delta Modes**: Alerts fire in real-time (using current POC)
  - **Advantage**: Immediate notification
  - **Disadvantage**: Alert uses repainting value (may not match plotted line)
- **Swing Mode**: Alerts delayed to match retroactive plot
  - **Advantage**: Alert matches plotted value exactly
  - **Disadvantage**: Delayed by `pivRi` bars

**Design Decision**: Prioritize real-time alerts for Structure/Delta modes (traders value speed over exact plot alignment).

---

## 9. State Machine Detail: Profile Lifecycle

### 9.1 Profile Lifecycle States

```
┌─────────────────────────────────────────────────────────────────┐
│                      PROFILE STATE MACHINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐                                                   │
│  │  INIT    │  First bar or chart load                         │
│  └────┬─────┘                                                   │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────┐  Structure break                                 │
│  │  ACTIVE  │◄─────────────────┐                               │
│  │          │                   │                               │
│  │ - Accum  │  Each bar without │                               │
│  │ - Calc   │  structure change │                               │
│  │ - Plot   │                   │                               │
│  └────┬─────┘                   │                               │
│       │                         │                               │
│       │ Structure state changes │                               │
│       ▼                         │                               │
│  ┌──────────┐                   │                               │
│  │ FINALIZE │                   │                               │
│  │          │                   │                               │
│  │ - Capture│                   │                               │
│  │ - Store  │                   │                               │
│  └────┬─────┘                   │                               │
│       │                         │                               │
│       │ Clear & reset           │                               │
│       ▼                         │                               │
│  ┌──────────┐                   │                               │
│  │  RESET   │───────────────────┘                               │
│  │          │                                                   │
│  │ - Clear  │                                                   │
│  │ - Range  │                                                   │
│  └──────────┘                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 State Transition Code Mapping

**INIT → ACTIVE** (Lines 602-607):
```pine
if prfReset
    resetBar := bar_index
    msProf.clear()
    msProf.setRanges(rangeUp = ..., rangeLo = ...)
```

**ACTIVE (Accumulation)** (Lines 620-652):
```pine
if not na(aBuy[delay]) and not na(resetBar)
    msProf.merge(...)  // Add bar data
    if rowDyn
        msProf.setBuckets(...)  // Dynamic row sizing
    // Calculate metrics (POC, VA, VWAP, StdDev)
```

**ACTIVE → FINALIZE** (Lines 540-596):
```pine
if prfReset and not isFirstReset and not na(resetBar)
    startBar := resetBar - delay
    endBar := bar_index - 1
    // Capture historical data
    array.push(profileStartBars, startBar)
    array.push(profileEndBars, endBar)
    // Store peak data (two-phase capture)
```

**FINALIZE → RESET** (Lines 602-607):
```pine
msProf.clear()  // Clear buffers
msProf.setRanges(...)  // Set new range
// (Transitions back to ACTIVE)
```

---

## 10. Key Architectural Decisions & Rationale

### Decision 1: Structure Mode Uses `delay = 0`

**Rationale**: Structure breaks are **confirmed events**, not predictive. Once pivot confirms, structure state is known. No need to lag profile start.

**Trade-Off**: Alerts fire in real-time (may not match plotted line exactly), but traders get immediate notification.

### Decision 2: Two-Phase Peak Capture

**Rationale**: Peak detection requires full profile data (max volume calculation). But profile boundaries must be captured **immediately** on anchor change (before peaks are known).

**Solution**: Store boundaries in Phase 1 (Lines 555-560), attach peaks in Phase 2 (Lines 761-776 during `barstate.islast`).

### Decision 3: Flattened Array Architecture for Historical Peaks

**Rationale**: Pine Script array size limits prevent nested arrays. Flattened design allows efficient storage of variable peak counts per profile.

**Complexity**: Requires careful index management (FIFO cleanup bug was due to index drift - see architecture-analysis.md).

### Decision 4: Library Abstractions (LibPvot, LibBrSt, LibVPrf)

**Rationale**:
- **Reusability**: Same pivot logic used across Structure/Delta modes
- **Maintainability**: Complex stats in LibBrSt isolated from rendering
- **Performance**: Optimized volume calculations in LibVPrf

**Trade-Off**: Slightly less transparent (library internals hidden), but massive code organization benefit.

---

## 11. Performance Considerations

### 11.1 Computational Bottlenecks

**Expensive Operations:**
1. **Intra-bar data fetching** (Line 459): `request.security()` with lower timeframe
2. **PDF integration** (Lines 219-224): Simpson integration steps (20 default)
3. **Peak detection** (Lines 711-755): O(N) scan of volume array per profile
4. **Flattened array access** (Lines 884-929): O(M*K) where M = profiles, K = peaks/profile

### 11.2 Optimization Strategies

**Conditional Calculations** (Lines 614-652):
```pine
// Skip expensive metrics when features disabled
bool _needPOC = showDevPOC or enableAlerts
if _needPOC
    [_, _poc] = msProf.getPoc()
    poc := _poc
```

**Savings**: ~320 operations per bar when developing indicators hidden (v1.0.6 optimization).

**Dynamic Row Sizing** (Lines 630-631):
```pine
if rowDyn
    msProf.setBuckets(int(rowNum * ((msProf.rangeUp - msProf.rangeLo)/msProf.rangeUp) * 100))
```

**Benefit**: Consistent visual density across different price ranges (crypto vs forex).

### 11.3 Memory Management

**FIFO Cleanup** (Lines 544-590):
- Limit: 50 historical profiles (MAX_HISTORICAL_PROFILES)
- Cleanup: Remove oldest when limit exceeded
- **Critical Fix**: Adjust `profilePeakStartIdx` after flattened array shifts (Lines 578-584)

**Box Limits** (Lines 796-843):
- Limit: 500 boxes (MAX_BOXES)
- FIFO deletion when approaching limit
- Buffer: Reserve 10 boxes for peak rectangles

---

## 12. Trading Workflow: How Traders Use Structure Mode

### 12.1 Real-World Example

**Scenario**: Bitcoin (BTC/USD) 1H chart, swing trading strategy

**Setup:**
1. Enable Structure mode
2. Set Pivot Right Bars = 2 (faster confirmation)
3. Enable developing POC + VA plots
4. Show peak rectangles for high-volume zones
5. Configure alerts: "Price crosses above Upper VA Band"

**Trading Execution:**

**Bar 100-120 (Bullish Structure):**
- Profile accumulates during uptrend (HH-HL sequence)
- POC at $42,500 (heavy buying)
- VA: $42,300 - $42,700 (institutional range)

**Bar 121 (Structure Break):**
- Lower High forms, confirms bearish reversal
- **Alert fires**: "Profile was resetted"
- Old profile finalizes, new profile starts

**Bar 122-140 (Bearish Structure):**
- New profile shows distribution
- POC at $42,200 (selling pressure)
- Trader enters short on pullback to old VA High ($42,700)

**Bar 141 (Reversal):**
- Higher Low forms, bullish structure resumes
- **Alert fires**: "Profile was resetted"
- Trader exits short, flips long

### 12.2 Decision Support

**Structure-Based Profiles Provide:**
1. **Context**: Where was value during last confirmed phase?
2. **Levels**: POC/VA act as support/resistance
3. **Timing**: Structure breaks signal potential entry/exit
4. **Confirmation**: Heavy volume at level = institutional interest

---

## 13. Comparison to Traditional Volume Profile Indicators

### 13.1 Traditional Fixed-Period Profiles

**Common Approaches:**
- Daily Profile: Resets at midnight (arbitrary time boundary)
- Session Profile: Resets at market open/close
- Range Profile: User-defined start/end bars

**Limitations:**
1. **Arbitrary Boundaries**: Time doesn't respect market structure
2. **Carry-Over Volume**: Profile cuts through trends
3. **Manual Adjustment**: Trader must reposition manually
4. **No Adaptation**: Same reset regardless of market behavior

### 13.2 Structure-Based Profile Advantages

**Adaptive Behavior:**
1. **Market-Driven**: Boundaries align with structure breaks
2. **Phase Isolation**: Each profile captures a distinct market phase
3. **Automatic**: No manual repositioning needed
4. **Forward Context**: Previous profile's POC/VA remain relevant

**Example:**
```
Traditional (Daily Profile):
|--Mon--|--Tue--|--Wed--|
   ↑ Rally continues across days (profile fragmented)

Structure-Based:
|----Accumulation----|----Markup----|
         ↑ Single profile captures entire rally phase
```

---

## 14. Future Architectural Extensions

### 14.1 Potential Enhancements

**Multi-Timeframe Structure:**
- Analyze structure on higher TF (e.g., 4H)
- Anchor profiles on main TF (e.g., 1H)
- **Benefit**: Align with larger trend context

**Volume-Weighted Structure:**
- Weight pivot significance by volume
- Ignore low-volume pivots
- **Benefit**: Filter noise, focus on institutional moves

**Dynamic Pivot Parameters:**
- Adjust `leftLen`/`rightLen` based on ATR
- Volatile markets = wider pivots
- **Benefit**: Adapt to changing market conditions

**Profile Merging:**
- Combine profiles if structure quickly reverses
- **Use Case**: Whipsaw filtering
- **Benefit**: Reduce false signals

### 14.2 Architectural Challenges

**Complexity Management:**
- Additional modes increase code paths
- Library abstractions essential for maintainability

**Performance:**
- Multi-TF analysis = more `request.security()` calls
- Must balance features vs execution time

**User Experience:**
- More options = steeper learning curve
- Need clear documentation and defaults

---

## 15. Conclusion: Why Structure Mode Architecture Succeeds

### 15.1 Core Strengths

1. **Conceptual Clarity**: Market structure is a well-understood trading concept
2. **Library Abstractions**: Clean separation of concerns (pivot detection, volume calculation, rendering)
3. **Event-Driven Design**: Structure breaks as atomic events simplify state management
4. **Non-Repainting Integrity**: Offset-based plotting preserves historical accuracy
5. **Trader-Centric**: Design aligns with how professional traders analyze markets

### 15.2 Implementation Quality

**Robustness:**
- Comprehensive state machine (INIT → ACTIVE → FINALIZE → RESET)
- Edge case handling (first profile, empty history, buffer limits)
- FIFO memory management with index correction (v1.0.6 fix)

**Performance:**
- Conditional calculations (v1.0.6 optimization)
- Dynamic row sizing for visual consistency
- Efficient flattened array architecture

**Extensibility:**
- Library-based design enables easy feature addition
- Mode-agnostic profile management
- Clear separation between anchoring logic and rendering

### 15.3 Key Takeaway

**Structure mode transforms volume profiles from time-based snapshots into market-phase containers**, providing traders with context-aware support/resistance levels that adapt to natural market rhythm rather than arbitrary clock boundaries. The architecture achieves this through a clean event-driven design, leveraging market structure breaks as anchoring events and abstracting complexity into specialized libraries.

---

## Appendix: Code Navigation Guide

### Key Code Sections

| Section                  | Lines      | Purpose                              |
|--------------------------|------------|--------------------------------------|
| **User Inputs**          | 153-410    | Configuration parameters             |
| **Library Imports**      | 138-142    | LibTmFr, LibBrSt, LibPvot, LibVPrf   |
| **Intra-Bar Data**       | 436-461    | LTF volume fetching                  |
| **Structure Detection**  | 503-526    | Pivot analysis & anchor reset logic  |
| **Profile Management**   | 540-607    | State transitions & data capture     |
| **Developing Metrics**   | 614-652    | POC/VA/VWAP/StdDev calculations      |
| **Historical Storage**   | 466-481    | Flattened array declarations         |
| **FIFO Cleanup**         | 544-590    | Historical limit enforcement         |
| **Peak Detection**       | 693-756    | High-volume zone identification      |
| **Rendering (Rectangles)**| 778-1013   | Box creation for frames & peaks      |
| **Rendering (Polylines)**| 1015-1123  | Volume profile visualization         |
| **Plots**                | 1125-1177  | Developing lines (POC/VA/VWAP/StdDev)|
| **Alerts**               | 1180-1238  | Alert condition definitions          |

### Cross-Reference: Architecture → Code

**Pivot Detection:**
- Concept: Section 2.2 "Phase 1: Pivot Detection"
- Code: Lines 506-511 (`LibPvot.marketStructure()`)

**Structure Analysis:**
- Concept: Section 2.2 "Phase 2: Structure Analysis"
- Code: Lines 513-514 (state comparison)

**Profile Reset:**
- Concept: Section 3.3 "Transition: Accumulation → Reset"
- Code: Lines 540-607 (capture & reset logic)

**Non-Repainting:**
- Concept: Section 8 "Non-Repainting Implementation"
- Code: Lines 675, 1125-1177 (offset mechanism)

**Flattened Arrays:**
- Concept: Section 9 "Profile Lifecycle"
- Code: Lines 466-481 (array declarations), 528-596 (storage/cleanup)

---

**Document Metadata:**
- **Version**: 1.0
- **Author**: System Architecture Designer
- **Date**: 2025-01-17
- **Related Files**:
  - `/src/au-mktStructureVP-v105.pine`
  - `/docs/architecture-analysis.md` (FIFO bug analysis)
  - `/changelog/CHANGELOG.md` (version history)
