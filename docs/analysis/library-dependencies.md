# Library Dependencies Analysis
## Market Structure Volume Profile Indicator

**Document Version:** 1.0.0
**Analysis Date:** 2025-11-10
**Code Version:** 0.1.0

---

## Overview

The Market Structure Volume Profile indicator relies on **four specialized Pine Script libraries** that work together to provide event-based volume profiling anchored to market structure. This document provides a comprehensive analysis of each library, their interactions, and data flow.

---

## Library Import Structure

```pine
import AustrianTradingMachine/LibTmFr/1 as LibTmFr   // Line 138
import AustrianTradingMachine/LibBrSt/1 as LibBrSt   // Line 139
import AustrianTradingMachine/LibPvot/1 as LibPvot   // Line 140
import AustrianTradingMachine/LibVPrf/1 as LibVPrf   // Line 141
```

All libraries are version 1 and published by AustrianTradingMachine.

---

## 1. LibTmFr (Timeframe Library)

### Purpose
Manages timeframe selection and period detection for intra-bar analysis.

### Key Functions Used

#### `LibTmFr.autoLTF(chartTF)` - Line 290
**Purpose:** Automatically selects an appropriate lower timeframe (LTF) for intra-bar data collection.

**Usage:**
```pine
varip series string ibTF = ctmLTF ? manLTF : LibTmFr.autoLTF(chartTF)
```

**Function:** When custom timeframe is disabled, automatically determines optimal intra-bar timeframe based on current chart timeframe.

**Validation:** Lines 292-293 ensure `ibTF < chartTF`:
```pine
if timeframe.in_seconds(ibTF) >= timeframe.in_seconds(chartTF)
    runtime.error("Intra-Bar timeframe must be smaller than selected timeframe")
```

#### `LibTmFr.isNewPeriod(chartTF)` - Line 315
**Purpose:** Detects when a new period begins on the chart timeframe.

**Usage:**
```pine
series bool reset = LibTmFr.isNewPeriod(chartTF)
```

**Function:** Returns `true` on the first intra-bar of a new chart period, triggering volume profile reset.

#### `LibTmFr.isMultipleTF(chartTF)` - Line 316
**Purpose:** Validates that multiple intra-bars exist per chart bar.

**Usage:**
```pine
series bool multiple = LibTmFr.isMultipleTF(chartTF)
vldPrd := not multiple ? false : reset ? true : vldPrd
```

**Function:** Ensures timeframe relationship is valid for intra-bar calculations.

### Data Flow
```
Chart TF → autoLTF() → Intra-Bar TF
                ↓
         isNewPeriod() → Reset Signal
                ↓
       isMultipleTF() → Validation Flag
```

---

## 2. LibBrSt (Bar Statistics Library)

### Purpose
Provides statistical models for estimating intra-bar price distribution and volume allocation.

### Key Enums Used

#### `LibBrSt.PriceEst` - Line 207
**Purpose:** Defines statistical models for price density functions within a bar.

**User Input:**
```pine
LibBrSt.PriceEst prcEst = input.enum(
  title   = "Price Estimator",
  defval  = LibBrSt.PriceEst.t4Skew,
  tooltip = "Selects the statistical model (Price Density Function)...")
```

**Default:** `t4Skew` - T4 Skew distribution model

**Function:** Used by LibVPrf to distribute volume across price levels within each bar using probability density functions rather than uniform distribution.

**Integration Point:** Passed to `LibVPrf.create()` as the `estimator` parameter (line 306).

### Statistical Models
The library provides various distribution models:
- **T4-Skew:** Default skewed distribution model
- **Classic:** Simple close-price allocation
- **PDF-based models:** Probability density function approaches

### Data Flow
```
Bar OHLC → PriceEst Model → Volume Distribution Weights
                                       ↓
                              LibVPrf.create(estimator)
```

---

## 3. LibPvot (Pivot Library)

### Purpose
Analyzes market structure by identifying pivot points and tracking trend state.

### Key Functions Used

#### `LibPvot.marketStructure()` - Lines 367, 379
**Purpose:** Identifies price pivots and determines market structure state (HH, HL, LH, LL).

**Usage 1: Price Structure Anchoring (Lines 367-372)**
```pine
[_, _, _, _, _, _, prcTrendState] = LibPvot.marketStructure(
  highSrc  = high,
  lowSrc   = low,
  leftLen  = pivLe,
  rightLen = pivRi,
  srcTol   = pivTol)
```

**Usage 2: Delta Structure Anchoring (Lines 379-384)**
```pine
[_, _, _, _, _, _, cvdTrendState] = LibPvot.marketStructure(
  highSrc  = CumH,    // Cumulative delta high
  lowSrc   = CumL,    // Cumulative delta low
  leftLen  = pivLe,
  rightLen = pivRi,
  srcTol   = pivTol)
```

### Return Values
The function returns a 7-element tuple:
```
[pivHi, pivLo, pivHiPrice, pivLoPrice, pivHiBar, pivLoBar, trendState]
```

**Used:** Only `trendState` (7th element) is captured via `_` placeholders.

### Trend State Values
- **1:** Uptrend (Higher Highs, Higher Lows)
- **-1:** Downtrend (Lower Lows, Lower Highs)
- **0:** Ranging/Undefined

### Profile Reset Logic

#### Structure Mode (Lines 374-375)
```pine
if prcTrendState != prcTrendState[1]
    prfReset := true
```
**Trigger:** New profile begins when **price structure** changes (e.g., HH confirmed).

#### Delta Mode (Lines 386-387)
```pine
if cvdTrendState != cvdTrendState[1]
    prfReset := true
```
**Trigger:** New profile begins when **cumulative delta structure** changes.

### Data Flow
```
Price OHLC Data     Cumulative Delta (CumH, CumL)
       ↓                        ↓
   marketStructure()      marketStructure()
       ↓                        ↓
  prcTrendState           cvdTrendState
       ↓                        ↓
   Profile Reset Trigger (Structure Mode)
              ↓
   Profile Reset Trigger (Delta Mode)
```

---

## 4. LibVPrf (Volume Profile Library) - CORE ENGINE

### Purpose
**Primary calculation engine** for volume profile construction, bucket management, and statistical metrics.

### Key Types

#### `LibVPrf.VProf`
**Type:** Custom object type for volume profile data structure.

**Instances:**
1. **`ibProf`** (Line 319): Intra-bar profile builder
2. **`msProf`** (Line 340): Market structure profile accumulator
3. **`flashProf`** (Line 341): Clone for final rendering

### Key Enums

#### `LibVPrf.AllotMode` - Line 199
**Values:**
- `pdf`: Probability density function distribution
- `classic`: Simple close-price allocation

**Usage:**
```pine
LibVPrf.AllotMode alltMod = input.enum(
  title   = "Allot model",
  defval  = LibVPrf.AllotMode.pdf)
```

#### `LibVPrf.SplitMode` - Line 220
**Values:**
- `dynamic`: Complex wick/trend analysis for buy/sell split
- `classic`: Color-based classification (green = buy, red = sell)

**Usage:**
```pine
LibVPrf.SplitMode volEst = input.enum(
  title   = "Volume Estimator",
  defval  = LibVPrf.SplitMode.dynamic)
```

### Key Functions

#### `LibVPrf.create()` - Lines 298-309
**Purpose:** Factory function to create standardized volume profile objects.

**Signature:**
```pine
createProfile() =>
    LibVPrf.create(
      buckets   = rowNum,        // Number of price levels
      rangeUp   = 0,            // Initial upper bound
      rangeLo   = 0,            // Initial lower bound
      dynamic   = true,         // Dynamic range adjustment
      valueArea = valArea,      // VA percentage (default 70%)
      allot     = alltMod,      // PDF vs Classic allocation
      estimator = prcEst,       // Statistical model (T4-Skew)
      cdfSteps  = intStp,       // Integration steps (default 20)
      split     = volEst,       // Buy/sell split method
      trendLen  = trendLen)     // Trend lookback period
```

**Returns:** `LibVPrf.VProf` object

#### Profile Methods

##### `.clear()` - Lines 322, 398
**Purpose:** Resets profile data to empty state.

**Usage:**
```pine
ibProf.clear()   // Reset intra-bar profile
msProf.clear()   // Reset market structure profile
```

##### `.setRanges(rangeUp, rangeLo)` - Lines 323-325, 399-401
**Purpose:** Sets price boundaries for the profile.

**Usage:**
```pine
ibProf.setRanges(
  rangeUp = math.max(high, open + (syminfo.mintick/2)),
  rangeLo = math.min(low,  open - (syminfo.mintick/2)))
```

##### `.addBar()` - Line 328
**Purpose:** Processes the current intra-bar and distributes its volume across price buckets.

**Context:** Called within `intBarCalc()` on valid intra-bar periods.

**Returns:** Updates internal bucket arrays.

##### `.merge()` - Lines 409-416, 436-443
**Purpose:** Merges intra-bar volume data into the main profile.

**Signature:**
```pine
msProf.merge(
  srcABuy    = aBuy[delay],      // Buy volume array
  srcASell   = aSell[delay],     // Sell volume array
  srcRangeUp = rangeUp[delay],   // Upper price bound
  srcRangeLo = rangeLo[delay],   // Lower price bound
  srcCvd     = 0,               // Cumulative delta (not used)
  srcCvdHi   = 0,               // Delta high (not used)
  srcCvdLo   = 0)               // Delta low (not used)
```

**Function:** Accumulates volume from each intra-bar into the market structure profile buckets.

##### `.setBuckets(count)` - Lines 419, 446
**Purpose:** Dynamically adjusts the number of price buckets based on profile range.

**Formula:**
```pine
int(rowNum * ((msProf.rangeUp - msProf.rangeLo)/msProf.rangeUp) * 100)
```

**Result:** More buckets for larger price ranges, maintaining consistent visual density.

##### `.getPoc()` - Line 421
**Purpose:** Calculates Point of Control (price level with highest volume).

**Returns:** `[bucketIndex, priceLevel]`

**Usage:**
```pine
[_, _poc] = msProf.getPoc()
poc := _poc
```

##### `.getVA()` - Line 422
**Purpose:** Calculates Value Area (price range containing specified % of volume).

**Returns:** `[upperBucket, upperPrice, lowerBucket, lowerPrice]`

**Usage:**
```pine
[_, _vaUp, _, _vaLo] = msProf.getVA()
vaUp := _vaUp
vaLo := _vaLo
```

##### `.getVwap()` - Line 423
**Purpose:** Calculates Volume Weighted Average Price.

**Returns:** `[bucketIndex, vwapPrice]`

**Usage:**
```pine
[_, _vwap] = msProf.getVwap()
vwap := _vwap
```

##### `.getStdDev()` - Line 428
**Purpose:** Calculates standard deviation of volume distribution.

**Returns:** `float` (standard deviation value)

**Usage:**
```pine
stdDev := msProf.getStdDev()
```

##### `.getBktBuyVol(index)` - Lines 460, 493, 501, 520, 531
**Purpose:** Retrieves buy volume for a specific bucket.

**Returns:** `float` (buy volume)

##### `.getBktSellVol(index)` - Lines 461, 502, 520, 532
**Purpose:** Retrieves sell volume for a specific bucket.

**Returns:** `float` (sell volume)

##### `.getBktBnds(index)` - Lines 482, 485, 492, 500, 510, 519, 530
**Purpose:** Gets upper and lower price bounds for a bucket.

**Returns:** `[upperPrice, lowerPrice]`

**Usage:**
```pine
[bktUp, bktLo] = flashProf.getBktBnds(i)
```

##### `.clone()` - Line 341
**Purpose:** Creates a copy of the profile for rendering.

**Usage:**
```pine
LibVPrf.VProf flashProf = msProf.clone()
```

### Data Flow Through LibVPrf

```
1. INITIALIZATION (Lines 298-309)
   createProfile() → LibVPrf.create() → VProf object

2. INTRA-BAR COLLECTION (Lines 313-331)
   request.security(ibTF) → intBarCalc()
                ↓
   ibProf.clear() → ibProf.setRanges() → ibProf.addBar()
                ↓
   Returns: [aBuy, aSell, rangeUp, rangeLo, cvd, cvdHi, cvdLo]

3. PROFILE ACCUMULATION (Lines 408-428)
   For each chart bar with delay:
   msProf.merge(aBuy[delay], aSell[delay], ranges[delay])
                ↓
   Dynamic bucket adjustment: msProf.setBuckets()
                ↓
   Calculate metrics:
   - msProf.getPoc() → poc
   - msProf.getVA() → vaUp, vaLo
   - msProf.getVwap() → vwap
   - msProf.getStdDev() → stdDev

4. FINAL RENDERING (Lines 430-449)
   flashProf = msProf.clone()
   Merge remaining delayed bars into flashProf
                ↓
   Dynamic bucket adjustment if needed
                ↓
   Used for polygon rendering (lines 457-571)

5. VISUALIZATION (Lines 457-571)
   For each bucket in flashProf:
   - flashProf.getBktBnds(i) → price bounds
   - flashProf.getBktBuyVol(i) → buy volume
   - flashProf.getBktSellVol(i) → sell volume
                ↓
   Calculate polygon points based on volume
                ↓
   polyline.new() → Draw histogram bars
```

---

## Library Interaction Architecture

### High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER INPUTS & CONFIGURATION                   │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  TIMEFRAME MANAGEMENT (LibTmFr)                                 │
│  • autoLTF() → Select intra-bar timeframe                       │
│  • isNewPeriod() → Detect period resets                         │
│  • isMultipleTF() → Validate timeframe relationship             │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  INTRA-BAR DATA COLLECTION (request.security + LibVPrf)         │
│  • ibProf.clear() on new period                                 │
│  • ibProf.setRanges() with bar OHLC                             │
│  • ibProf.addBar() processes volume with:                       │
│    - LibBrSt.PriceEst for distribution model                    │
│    - LibVPrf.AllotMode for allocation method                    │
│    - LibVPrf.SplitMode for buy/sell split                       │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  PIVOT ANALYSIS (LibPvot)                                       │
│  • marketStructure(high, low) → prcTrendState                   │
│  • marketStructure(CumH, CumL) → cvdTrendState                  │
│  • Swing detection via ta.pivothigh/pivotlow                    │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  PROFILE RESET LOGIC (Anchoring)                                │
│  • Swing: Impulse baseline change from pivots                   │
│  • Structure: prcTrendState change                              │
│  • Delta: cvdTrendState change                                  │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  PROFILE ACCUMULATION (LibVPrf - msProf)                        │
│  • msProf.clear() on reset                                      │
│  • msProf.setRanges() with delayed bar                          │
│  • msProf.merge() accumulates intra-bar data                    │
│  • msProf.setBuckets() for dynamic sizing                       │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  METRIC CALCULATION (LibVPrf)                                   │
│  • msProf.getPoc() → Point of Control                           │
│  • msProf.getVA() → Value Area High/Low                         │
│  • msProf.getVwap() → Volume Weighted Average Price             │
│  • msProf.getStdDev() → Standard Deviation                      │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  FINAL RENDERING (LibVPrf - flashProf)                          │
│  • flashProf = msProf.clone()                                   │
│  • Merge remaining delayed bars                                 │
│  • flashProf.getBktBnds() for each bucket                       │
│  • flashProf.getBktBuyVol() / getBktSellVol()                   │
│  • Calculate polygon coordinates                                │
└────────────┬────────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────────┐
│  VISUALIZATION                                                   │
│  • polyline.new() with calculated points                        │
│  • plot() for developing lines (POC, VA, VWAP, StdDev)          │
│  • Histogram bars drawn as closed polylines                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Critical Integration Points

### 1. Timeframe Validation Chain
```pine
LibTmFr.autoLTF() → ibTF selection
       ↓
Validation: ibTF < chartTF (lines 292-293)
       ↓
request.security(ibTF, intBarCalc()) (line 336)
```

### 2. Statistical Model Pipeline
```pine
LibBrSt.PriceEst (user input)
       ↓
LibVPrf.create(estimator = prcEst)
       ↓
ibProf.addBar() applies distribution model
       ↓
Volume distributed across price buckets
```

### 3. Pivot-Based Reset System
```pine
LibPvot.marketStructure() → trendState
       ↓
trendState != trendState[1] → prfReset
       ↓
msProf.clear() + msProf.setRanges()
       ↓
New profile accumulation begins
```

### 4. Volume Profile Accumulation
```pine
ibProf.addBar() → [aBuy, aSell, ranges, cvd]
       ↓
msProf.merge(aBuy[delay], aSell[delay], ranges[delay])
       ↓
msProf.setBuckets() for dynamic sizing
       ↓
msProf.getPoc(), getVA(), getVwap(), getStdDev()
```

---

## Key Insights

### 1. No Factory Function for Histogram Drawing
The histogram bars are **explicitly constructed** using polylines (lines 454-571), not via a library factory function. The process:
- LibVPrf provides bucket data (volume, price bounds)
- Main code calculates polygon coordinates
- `polyline.new()` draws each histogram segment

### 2. Three-Stage Profile System
1. **Intra-bar collection** (`ibProf`): Collects high-resolution data
2. **Market structure accumulation** (`msProf`): Builds event-anchored profile
3. **Final rendering** (`flashProf`): Clone for visualization with delayed bars merged

### 3. Dual Market Structure Analysis
LibPvot is applied to **both**:
- **Price structure** (OHLC data) for Structure mode
- **Cumulative delta** (CumH, CumL) for Delta mode

### 4. Statistical Sophistication
The combination of LibBrSt.PriceEst and LibVPrf.AllotMode enables advanced distribution modeling:
- **T4-Skew distribution** vs uniform allocation
- **Dynamic buy/sell split** vs color-based classification
- **Simpson integration** for PDF calculations (20 steps default)

### 5. Delay Management
The `delay` variable (0 for Structure/Delta, pivRi for Swing) ensures:
- Consistent profile construction timing
- Aligned visualization with event anchors
- Proper alert triggering based on mode

---

## Dependency Summary

| Library | Primary Role | Key Functions | Integration Point |
|---------|-------------|---------------|-------------------|
| **LibTmFr** | Timeframe Management | `autoLTF()`, `isNewPeriod()`, `isMultipleTF()` | Lines 290, 315-316 |
| **LibBrSt** | Statistical Models | `PriceEst` enum | Line 207, passed to LibVPrf |
| **LibPvot** | Market Structure | `marketStructure()` | Lines 367, 379 for reset detection |
| **LibVPrf** | **Volume Profile Engine** | `create()`, `.merge()`, `.getPoc()`, `.getVA()`, `.getVwap()`, `.getStdDev()`, `.getBkt*()` | **Core calculations throughout** |

---

## Conclusion

The Market Structure Volume Profile indicator achieves its sophisticated functionality through a carefully orchestrated **four-library architecture**:

1. **LibTmFr** handles timeframe logic and period detection
2. **LibBrSt** provides statistical distribution models
3. **LibPvot** identifies market structure events for anchoring
4. **LibVPrf** performs the heavy lifting of volume profile calculations

The **histogram visualization is custom-built** using the bucket data from LibVPrf, demonstrating that while the libraries provide powerful calculation engines, the final rendering logic remains in the main indicator code for maximum flexibility.

This modular design enables:
- **Event-based anchoring** via LibPvot
- **High-resolution intra-bar data** via LibTmFr + request.security
- **Advanced statistical modeling** via LibBrSt + LibVPrf
- **Flexible visualization** via custom polyline construction

---

**Document Prepared By:** Research Agent (Researcher Specialist)
**Analysis Type:** Library Architecture & Data Flow Mapping
**Methodology:** Code trace analysis with function call mapping
