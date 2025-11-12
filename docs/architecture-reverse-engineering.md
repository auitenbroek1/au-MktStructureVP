# Market Structure Volume Profile - Architecture Reverse Engineering

## Executive Summary

This document provides a complete reverse-engineered architecture of the Market Structure Volume Profile indicator, mapping the data flow from raw price/volume inputs through statistical processing to final chart rendering.

---

## 1. ANCHORING EVENT DETECTION

### 1.1 Swing Anchoring (Lines 351-363, 478-490)

**Detection Engine:**
- **Library:** Pine Script built-in `ta.pivothigh()` and `ta.pivotlow()`
- **Trigger Logic:** Changes in cumulative delta impulse baseline
- **Confirmation:** Requires `pivRi` bars (default: 5) for pivot confirmation

**Algorithm:**
```pine
pivPrcHi = not na(ta.pivothigh(high, pivLe, pivRi))
pivPrcLo = not na(ta.pivotlow(low, pivLe, pivRi))

if pivPrcHi or pivPrcLo
    impulseStart := switch pivPrcHi
        true  => math.min(nz(impulseStart), math.max(CumO[pivRi], CumC[pivRi]))
        false => math.max(nz(impulseStart), math.min(CumO[pivRi], CumC[pivRi]))

if impulseStart != impulseStart[1]
    prfReset := true  // Triggers profile reset
```

**Key Variables:**
- `impulseStart`: Cumulative delta baseline (varip)
- `pivLe`: Left bars for pivot strength (default: 10)
- `pivRi`: Right bars for confirmation lag (default: 5)
- `delay`: Set to `pivRi` in Swing mode for retroactive plotting

### 1.2 Structure Anchoring (Lines 364-375, 491-502)

**Detection Engine:**
- **Library:** `LibPvot.marketStructure()` (AustrianTradingMachine/LibPvot/1)
- **Source Data:** Price highs and lows
- **Trigger Logic:** Market structure breaks (HH/LL sequences)

**Algorithm:**
```pine
[_, _, _, _, _, _, prcTrendState] = LibPvot.marketStructure(
  highSrc  = high,
  lowSrc   = low,
  leftLen  = pivLe,
  rightLen = pivRi,
  srcTol   = pivTol)

if prcTrendState != prcTrendState[1]
    prfReset := true
```

**Key Variables:**
- `prcTrendState`: Market structure state machine output
- `pivTol`: Price tolerance percentage (default: 0.05%)
- `delay`: Set to 0 (real-time) in Structure mode

### 1.3 Delta Anchoring (Lines 376-387, 503-514)

**Detection Engine:**
- **Library:** `LibPvot.marketStructure()` (same as Structure)
- **Source Data:** Cumulative delta (CumH, CumL)
- **Trigger Logic:** Delta structure breaks (HH/LL in cumulative delta)

**Algorithm:**
```pine
[_, _, _, _, _, _, cvdTrendState] = LibPvot.marketStructure(
  highSrc  = CumH,
  lowSrc   = CumL,
  leftLen  = pivLe,
  rightLen = pivRi,
  srcTol   = pivTol)

if cvdTrendState != cvdTrendState[1]
    prfReset := true
```

**Key Variables:**
- `cvdTrendState`: Delta structure state machine output
- `CumH/CumL`: Cumulative delta high/low (continuous, non-resetting)
- `delay`: Set to 0 (real-time) in Delta mode

---

## 2. VOLUME DATA COLLECTION

### 2.1 Lower Timeframe Selection (Lines 289-293, 400-404)

**Timeframe Engine:**
- **Library:** `LibTmFr.autoLTF()` (AustrianTradingMachine/LibTmFr/1)
- **Method:** Automatic selection or manual override

**Algorithm:**
```pine
varip series string chartTF = timeframe.main_period
varip series string ibTF = ctmLTF ? manLTF : LibTmFr.autoLTF(chartTF)

if timeframe.in_seconds(ibTF) >= timeframe.in_seconds(chartTF)
    runtime.error("Intra-Bar timeframe must be smaller than selected timeframe")
```

**Configuration:**
- `ctmLTF`: Boolean flag for custom timeframe
- `manLTF`: User-specified intra-bar timeframe
- Auto-selection ensures LTF < Main TF

### 2.2 Intra-Bar Profile Construction (Lines 313-331, 424-442)

**Processing Engine:**
- **Function:** `intBarCalc()` - Core intra-bar aggregation
- **Library:** `LibVPrf.VProf` object (AustrianTradingMachine/LibVPrf/1)
- **Method:** Statistical volume distribution per bar

**Algorithm:**
```pine
intBarCalc() =>
    var series bool vldPrd = false
    series bool reset = LibTmFr.isNewPeriod(chartTF)
    series bool multiple = LibTmFr.isMultipleTF(chartTF)
    vldPrd := not multiple ? false : reset ? true : vldPrd

    var LibVPrf.VProf ibProf = createProfile()

    if reset
        ibProf.clear()
        ibProf.setRanges(
          rangeUp = math.max(high, open + (syminfo.mintick/2)),
          rangeLo = math.min(low, open - (syminfo.mintick/2)))

    if vldPrd
        ibProf.addBar()
        [ibProf.aBuy, ibProf.aSell, ibProf.rangeUp, ibProf.rangeLo,
         ibProf.cvd, ibProf.cvdHi, ibProf.cvdLo]
    else
        [na, na, na, na, na, na, na]
```

**Output Arrays:**
- `aBuy`: Buy volume array per price level
- `aSell`: Sell volume array per price level
- `rangeUp/rangeLo`: Price range boundaries
- `cvd/cvdHi/cvdLo`: Cumulative volume delta metrics

### 2.3 Timeframe Conversion (Lines 335-336, 446-449)

**Security Function:**
- **Method:** `request.security()` - Pine Script MTF aggregation
- **Parameters:** No lookahead, no gaps filling

**Algorithm:**
```pine
ibTFCnvrt() =>
    request.security(syminfo.tickerid, ibTF, intBarCalc(),
                    barmerge.gaps_off, barmerge.lookahead_off)

[aBuy, aSell, rangeUp, rangeLo, cvd, cvdHi, cvdLo] = ibTFCnvrt()
```

**Key Behavior:**
- Synchronizes intra-bar data with chart timeframe bars
- Preserves statistical array structures across timeframe boundaries
- Prevents repainting with `lookahead_off`

---

## 3. STATISTICAL DISTRIBUTION ENGINE

### 3.1 Profile Factory (Lines 298-309, 409-420)

**Profile Object Creation:**
- **Library:** `LibVPrf.create()` (AustrianTradingMachine/LibVPrf/1)
- **Type:** `LibVPrf.VProf` user-defined type (UDT)

**Configuration:**
```pine
createProfile() =>
    LibVPrf.create(
      buckets   = rowNum,              // Number of price levels (default: 24)
      rangeUp   = 0,                   // Initial upper range
      rangeLo   = 0,                   // Initial lower range
      dynamic   = true,                // Enable dynamic range adjustment
      valueArea = valArea,             // VA percentage (default: 70%)
      allot     = alltMod,             // Volume allocation model
      estimator = prcEst,              // PDF statistical model
      cdfSteps  = intStp,              // Integration precision (default: 20)
      split     = volEst,              // Buy/Sell split method
      trendLen  = trendLen)            // Trend lookback (default: 3)
```

**Allocation Models (`alltMod`):**
1. **Classic:** All volume assigned to close price level
2. **PDF:** Statistical distribution via Probability Density Function

**Volume Estimators (`prcEst`):**
- **t4Skew:** T4 distribution with skewness adjustment (default)
- Other LibBrSt.PriceEst enum options

**Split Methods (`volEst`):**
1. **Classic:** Volume classified by candle color (green=buy, red=sell)
2. **Dynamic:** Wick analysis + trend-based buy/sell pressure estimation

### 3.2 Volume Merging (Lines 408-428, 607-627)

**Merge Engine:**
- **Method:** `msProf.merge()` - Accumulates intra-bar data into profile
- **Frequency:** Every bar where `not na(aBuy[delay])`

**Algorithm:**
```pine
if not na(aBuy[delay]) and not na(resetBar)
    msProf.merge(
      srcABuy    = aBuy[delay],        // Buy volume array
      srcASell   = aSell[delay],       // Sell volume array
      srcRangeUp = rangeUp[delay],     // Price range upper
      srcRangeLo = rangeLo[delay],     // Price range lower
      srcCvd     = 0,                  // CVD delta (not used)
      srcCvdHi   = 0,                  // CVD high (not used)
      srcCvdLo   = 0)                  // CVD low (not used)

    if rowDyn
        // Dynamic bucket adjustment based on price range percentage
        msProf.setBuckets(int(rowNum * ((msProf.rangeUp - msProf.rangeLo)/msProf.rangeUp) * 100))
```

**Key Features:**
- **Delay Handling:** Uses `aBuy[delay]` for Swing mode retroactive alignment
- **Dynamic Rows:** Adjusts bucket count based on profile height percentage
- **Continuous Accumulation:** Merges every bar within profile period

### 3.3 Statistical Metrics Calculation (Lines 420-428, 619-627)

**Metrics Engine:**
- **Methods:** LibVPrf getter functions (POC, VA, VWAP, StdDev)

**Algorithm:**
```pine
[_, _poc] = msProf.getPoc()                    // Point of Control
[_, _vaUp, _, _vaLo] = msProf.getVA()         // Value Area High/Low
[_, _vwap] = msProf.getVwap()                 // Volume-Weighted Average Price
stdDev := msProf.getStdDev()                  // Standard Deviation

poc    := _poc
vaUp   := _vaUp
vaLo   := _vaLo
vwap   := _vwap
```

**Metric Definitions:**
- **POC:** Price level with maximum volume
- **VA:** Price range containing `valArea`% of total volume (default: 70%)
- **VWAP:** Volume-weighted average price across all levels
- **StdDev:** Statistical spread around VWAP

---

## 4. PROFILE OBJECT ARCHITECTURE

### 4.1 Profile Object Types

**Primary Objects:**
1. **ibProf** (Lines 319, 430)
   - Type: `LibVPrf.VProf`
   - Scope: `var` (persistent across bars, intra-bar timeframe)
   - Purpose: Accumulates intra-bar volume for current chart bar

2. **msProf** (Lines 340, 451)
   - Type: `LibVPrf.VProf`
   - Scope: `var` (persistent across bars, chart timeframe)
   - Purpose: Main profile accumulator for current anchored period

3. **flashProf** (Lines 341, 452)
   - Type: `LibVPrf.VProf`
   - Scope: Series (recalculated each bar)
   - Purpose: Final profile snapshot for rendering (includes delay projection)

### 4.2 Profile Lifecycle

**State Machine:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. ANCHOR EVENT DETECTED (prfReset = true)                 │
│    - Clear msProf                                           │
│    - Initialize ranges (rangeUp, rangeLo)                  │
│    - Store previous profile boundaries (startBar, endBar)  │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. ACCUMULATION PHASE (every bar until next reset)         │
│    - Merge aBuy[delay] and aSell[delay] arrays             │
│    - Dynamically adjust bucket count if rowDyn=true        │
│    - Calculate metrics (POC, VA, VWAP, StdDev)             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. RENDERING PHASE (barstate.islast)                       │
│    - Clone msProf to flashProf                             │
│    - Apply delay projection if needed (Swing mode)         │
│    - Generate polyline points for histogram                │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Flash Profile Projection (Lines 430-447, 629-648)

**Purpose:** Project profile forward for Swing mode retroactive alignment

**Algorithm:**
```pine
if barstate.islast
    if resetBar == bar_index
        flashProf := msProf  // Direct clone (profile just started)
    else
        // Merge forward-looking bars for Swing mode delay
        for i = (delay - 1) to 0
            if i >= 0
                flashProf.merge(
                  srcABuy    = aBuy[i],
                  srcASell   = aSell[i],
                  srcRangeUp = rangeUp[i],
                  srcRangeLo = rangeLo[i],
                  srcCvd     = 0,
                  srcCvdHi   = 0,
                  srcCvdLo   = 0)

        if rowDyn
            flashProf.setBuckets(int(rowNum * ((flashProf.rangeUp - flashProf.rangeLo)/flashProf.rangeUp) * 100))

    endBar   := bar_index
    startBar := resetBar - delay
```

**Key Behavior:**
- **Swing Mode (delay = pivRi):** Merges past `pivRi` bars to align profile with historical pivot
- **Structure/Delta Mode (delay = 0):** Direct clone without projection

---

## 5. CHART RENDERING ENGINE

### 5.1 Polyline Construction (Lines 454-571, 655-831)

**Rendering Library:**
- **Type:** `polyline` (Pine Script v6 drawing object)
- **Method:** Point array construction with `chart.point.from_index()`

**Algorithm Overview:**
```
1. Calculate maximum volume across all buckets
2. Initialize point arrays (bPoints for buy, sPoints for sell)
3. Iterate through buckets (0 to flashProf.buckets - 1)
4. Generate polyline vertices based on volDisp mode
5. Create closed polyline objects with fill colors
6. FIFO cleanup if approaching MAX_POLYLINES limit
```

**Point Array Construction (Up/Down Mode, Lines 489-515):**
```pine
switch volDisp
    VolDisp.upDn =>
        // Buy volume polygon (baseline to buy extent)
        array.push(bPoints, chart.point.from_index(xBase, yFirst))
        for i = 0 to flashProf.buckets - 1
            [bktUp, bktLo] = flashProf.getBktBnds(i)
            float buyVol   = flashProf.getBktBuyVol(i)
            float buyWidth = barWidth * (buyVol / maxVol)
            float xEnd     = xBase + direction * buyWidth
            array.push(bPoints, chart.point.from_index(int(xEnd), bktLo))
            array.push(bPoints, chart.point.from_index(int(xEnd), bktUp))
        array.push(bPoints, chart.point.from_index(xBase, yLast))

        // Sell volume polygon (buy extent to total extent)
        for i = 0 to flashProf.buckets - 1
            [bktUp, bktLo] = flashProf.getBktBnds(i)
            float buyVol     = flashProf.getBktBuyVol(i)
            float sellVol    = flashProf.getBktSellVol(i)
            float buyWidth   = barWidth * (buyVol / maxVol)
            float sellWidth  = barWidth * (sellVol / maxVol)
            float sellXStart = xBase + direction * buyWidth
            float sellXEnd   = sellXStart + direction * sellWidth
            array.push(sPoints, chart.point.from_index(int(sellXEnd), bktLo))
            array.push(sPoints, chart.point.from_index(int(sellXEnd), bktUp))

        // Close sell polygon
        for i = flashProf.buckets - 1 to 0
            [bktUp, bktLo] = flashProf.getBktBnds(i)
            float buyVol     = flashProf.getBktBuyVol(i)
            float buyWidth   = barWidth * (buyVol / maxVol)
            float sellXStart = xBase + direction * buyWidth
            array.push(sPoints, chart.point.from_index(int(sellXStart), bktUp))
            array.push(sPoints, chart.point.from_index(int(sellXStart), bktLo))
```

**Coordinate System:**
- **X-Axis:** Bar index (`startBar` to `endBar`)
- **Y-Axis:** Price levels from `flashProf.getBktBnds()`
- **Width Scaling:** Proportional to volume (`barWidth * vol/maxVol`)
- **Direction:** Left (`direction = 1`) or Right (`direction = -1`)

### 5.2 Polyline Object Creation (Lines 553-571, 813-831)

**Polyline API:**
```pine
if array.size(bPoints) > 2
    array.push(allPolylines, polyline.new(
      points     = bPoints,
      closed     = true,
      xloc       = xloc.bar_index,
      line_color = LINE_CLR,            // Transparent (#000000, transp=100)
      fill_color = buyClr,              // User-defined (#00332a)
      line_style = line.style_solid,
      line_width = 1))

if array.size(sPoints) > 2
    array.push(allPolylines, polyline.new(
      points     = sPoints,
      closed     = true,
      xloc       = xloc.bar_index,
      line_color = LINE_CLR,
      fill_color = sellClr,             // User-defined (#801922)
      line_style = line.style_solid,
      line_width = 1))
```

**Key Features:**
- **Closed Polygons:** `closed = true` for filled histograms
- **Transparent Border:** `LINE_CLR` with 100% transparency
- **User Colors:** `buyClr` and `sellClr` from input configuration
- **FIFO Management:** Oldest polylines deleted when approaching limit

### 5.3 Developing Lines (Lines 573-625, 1063-1115)

**Plot Method:**
- **Type:** `plot()` with `plot.style_stepline` or `plot.style_line`
- **Offset:** `-pivRi` for visual alignment with historical data

**Metrics Plotted:**
```pine
// Value Area
vaUpPl = plot(vaUp[offset], title = "Developing VA Up",
              color = color.new(#86549b, transp = 50),
              style = plot.style_stepline, linewidth = 1, offset = -pivRi)
vaLoPl = plot(vaLo[offset], title = "Developing VA Low",
              color = color.new(#86549b, transp = 50),
              style = plot.style_stepline, linewidth = 1, offset = -pivRi)
fill(vaUpPl, vaLoPl, color = color.new(#86549b, transp = 75))

// Point of Control
plot(poc[offset], title = "Developing POC",
     color = color.new(#b8b8b8, transp = 50),
     style = plot.style_stepline, linewidth = 1, offset = -pivRi)

// Standard Deviation Bands
sgmUpPl = plot(vwap[offset] + stdDev[offset], title = "Developing +StdDev",
               color = color.new(#b6a72b, transp = 50),
               style = plot.style_line, linewidth = 1, offset = -pivRi)
sgmLoPl = plot(vwap[offset] - stdDev[offset], title = "Developing -StdDev",
               color = color.new(#b6a72b, transp = 50),
               style = plot.style_line, linewidth = 1, offset = -pivRi)
fill(sgmLoPl, sgmUpPl, color = color.new(#b6a72b, transp = 75))

// Volume-Weighted Average Price
plot(vwap[offset], title = "Developing VWAP",
     color = color.new(#c67600, transp = 50),
     style = plot.style_line, linewidth = 1, offset = -pivRi)
```

**Offset Logic:**
- **Swing Mode:** `offset = 0` (retroactive, already aligned)
- **Structure/Delta Mode:** `offset = pivRi` (non-repainting, shift back)

---

## 6. HISTORICAL PROFILE MANAGEMENT

### 6.1 Storage Architecture (ORIGINAL)

**ORIGINAL Implementation:** None
- No historical profile persistence
- Only current/developing profile rendered
- Previous profiles discarded on anchor change

### 6.2 Storage Architecture (CURRENT - Lines 454-469)

**Arrays for Historical Storage:**
```pine
var array<polyline> allPolylines = array.new<polyline>()
var array<box> allBoxes = array.new<box>()
var array<box> currentProfileBoxes = array.new<box>()

// Historical profile metadata (flattened arrays)
var array<int> profileStartBars = array.new<int>()
var array<int> profileEndBars = array.new<int>()
var array<int> profilePeakCounts = array.new<int>()
var array<int> profilePeakStartIdx = array.new<int>()

// Flattened peak data (prices, not row indices)
var array<float> allPeakStartPrices = array.new<float>()
var array<float> allPeakEndPrices = array.new<float>()
```

**FIFO Management (Lines 557-583):**
```pine
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
    // Get oldest profile's peak count
    int oldestPeakCount = array.shift(profilePeakCounts)

    // Remove oldest profile data
    array.shift(profileStartBars)
    array.shift(profileEndBars)
    array.shift(profilePeakStartIdx)

    // Remove oldest peaks from flattened arrays
    for i = 0 to oldestPeakCount - 1
        if array.size(allPeakStartPrices) > 0
            array.shift(allPeakStartPrices)
            array.shift(allPeakEndPrices)

    // CRITICAL FIX: Adjust remaining indices
    int remainingProfiles = array.size(profilePeakStartIdx)
    for i = 0 to remainingProfiles - 1
        int currentIdx = array.get(profilePeakStartIdx, i)
        array.set(profilePeakStartIdx, i, currentIdx - oldestPeakCount)

    // Delete associated boxes
    for i = 0 to math.min(oldestPeakCount, array.size(allBoxes)) - 1
        if array.size(allBoxes) > 0
            box.delete(array.shift(allBoxes))
```

### 6.3 Rectangle Frame Rendering (CURRENT ONLY - Lines 851-898)

**Box Creation:**
```pine
if showRect
    // FIFO cleanup
    if array.size(allBoxes) >= MAX_BOXES - 1
        box.delete(array.shift(allBoxes))

    // Create profile border rectangle
    box newBox = box.new(
      left   = startBar,
      top    = flashProf.rangeUp,
      right  = endBar,
      bottom = flashProf.rangeLo,
      border_color = rectBorderClr,
      border_width = rectBorderWidth,
      border_style = rectBorderStyle,
      bgcolor      = color.new(rectFillClr, rectTransp),
      xloc         = xloc.bar_index,
      extend       = extend.none)

    array.push(allBoxes, newBox)
```

**Key Features:**
- **Coordinates:** Uses `startBar`/`endBar` for time, `rangeLo`/`rangeUp` for price
- **FIFO Limit:** MAX_BOXES = 500 (Pine Script limit)
- **Visual:** Border + semi-transparent fill

### 6.4 Peak Rectangle Rendering (CURRENT ONLY - Lines 919-1061)

**Two-Phase Architecture:**

**Phase 1: Historical Profiles (Lines 923-977)**
```pine
int numHistoricalProfiles = array.size(profileStartBars)

for profIdx = 0 to numHistoricalProfiles - 1
    int histStartBar = array.get(profileStartBars, profIdx)
    int histEndBar = array.get(profileEndBars, profIdx)
    int peakCount = array.get(profilePeakCounts, profIdx)
    int peakStartIndex = array.get(profilePeakStartIdx, profIdx)

    // Safety check: Skip if beyond buffer limit
    int barsBack = bar_index - histStartBar
    if barsBack > MAX_BARS_BACK_LIMIT - SAFETY_MARGIN
        continue

    // Safety check: Skip if profile < 5 bars
    int profileBarCount = histEndBar - histStartBar + 1
    if profileBarCount < MIN_PROFILE_BARS
        continue

    // Render each peak
    for peakIdx = 0 to peakCount - 1
        // Get prices directly from flattened arrays
        float bottomPrice = array.get(allPeakStartPrices, peakStartIndex + peakIdx)
        float topPrice = array.get(allPeakEndPrices, peakStartIndex + peakIdx)

        // Create peak rectangle
        box peakBox = box.new(
          left   = histStartBar,
          top    = topPrice,
          right  = histEndBar + peakExtensionBars,
          bottom = bottomPrice,
          border_color = peakRectColor,
          border_width = peakRectBorderWidth,
          border_style = line.style_solid,
          bgcolor      = peakRectFillColor,
          xloc         = xloc.bar_index,
          extend       = extend.none)

        array.push(allBoxes, peakBox)
```

**Phase 2: Current Profile (Lines 982-1061)**
```pine
int numCurrentPeaks = array.size(currentPeakStarts)

// Cleanup previous bars' current profile boxes
bool isNewBar = bar_index != lastCurrentProfileBar
if isNewBar and array.size(currentProfileBoxes) > 0
    for i = 0 to array.size(currentProfileBoxes) - 1
        box.delete(array.get(currentProfileBoxes, i))
    array.clear(currentProfileBoxes)

// Safety check: Only render if current profile >= 5 bars
int currentProfileBarCount = not na(resetBar) ? bar_index - resetBar + 1 : bar_index + 1

if currentProfileBarCount >= MIN_PROFILE_BARS
    for peakIdx = 0 to numCurrentPeaks - 1
        int startRow = array.get(currentPeakStarts, peakIdx)
        int endRow = array.get(currentPeakEnds, peakIdx)

        // Get prices from flashProf bucket bounds
        [topPrice, _] = flashProf.getBktBnds(endRow)
        [_, bottomPrice] = flashProf.getBktBnds(startRow)

        // Create current profile peak rectangle
        box peakBox = box.new(
          left   = startBar,
          top    = topPrice,
          right  = bar_index + peakExtensionBars,
          bottom = bottomPrice,
          border_color = peakRectColor,
          border_width = peakRectBorderWidth,
          border_style = line.style_solid,
          bgcolor      = peakRectFillColor,
          xloc         = xloc.bar_index,
          extend       = extend.none)

        array.push(currentProfileBoxes, peakBox)

lastCurrentProfileBar := bar_index
```

---

## 7. CRITICAL DEVIATIONS: ORIGINAL vs CURRENT

### 7.1 Historical Profile Storage

**ORIGINAL:**
- ✅ No historical storage - single profile only
- ✅ Polylines naturally deleted by FIFO on new profile
- ✅ Simple memory footprint
- ❌ No historical context
- ❌ No peak zone visualization

**CURRENT:**
- ✅ MAX_HISTORICAL_PROFILES = 50 profiles stored
- ✅ Flattened array architecture for peaks
- ✅ Peak zone rectangles with forward extension
- ❌ Complex FIFO management with index adjustment
- ❌ Higher memory usage
- ⚠️ **CRITICAL BUG:** Index adjustment logic (lines 572-578) - FIXED in current

### 7.2 Rectangle Frame Feature

**ORIGINAL:**
- ❌ No rectangle frame
- ✅ Clean polyline-only visualization

**CURRENT:**
- ✅ Optional rectangle frame (showRect = true by default)
- ✅ Configurable border color/width/style
- ✅ Semi-transparent background fill
- ⚠️ Additional box object overhead (1 per profile)

### 7.3 Peak Detection & Rendering

**ORIGINAL:**
- ❌ No peak detection
- ❌ No peak rectangles

**CURRENT:**
- ✅ Continuous high-volume zone detection (50% threshold)
- ✅ Peak rectangles with forward extension (default: 50 bars)
- ✅ Two-phase rendering (historical + current)
- ✅ Safety checks: buffer limit (Issue #2), 5-bar minimum (Issue #6)
- ⚠️ Complex peak storage in flattened arrays
- ⚠️ Price-based storage (not row indices) for profile range changes

### 7.4 Display Toggle Controls

**ORIGINAL:**
- ❌ All developing lines always visible

**CURRENT:**
- ✅ Independent toggles for:
  - Value Area (showDevVA)
  - POC (showDevPOC)
  - StdDev Bands (showDevStdDev)
  - VWAP (showDevVWAP)
- ✅ Conditional plotting with `na` when disabled

### 7.5 Safety Mechanisms

**ORIGINAL:**
- ✅ MAX_POLYLINES = 100 (Pine Script limit)
- ✅ MAX_POINTS = 10000 check with runtime error

**CURRENT:**
- ✅ MAX_POLYLINES = 100 (unchanged)
- ✅ MAX_BOXES = 500 (Pine Script limit)
- ✅ MAX_HISTORICAL_PROFILES = 50 (custom limit)
- ✅ MIN_PROFILE_BARS = 5 (Issue #6 fix)
- ✅ SAFETY_MARGIN = 100 (Issue #2 fix)
- ✅ MAX_BARS_BACK_LIMIT = 5000 (matches indicator parameter)

### 7.6 Constants Added

**ORIGINAL:**
```pine
const int MAX_POLYLINES = 100
const int MAX_POINTS    = 10000
```

**CURRENT:**
```pine
const int MAX_POLYLINES = 100
const int MAX_POINTS    = 10000
const int MAX_BOXES     = 500
const int MAX_HISTORICAL_PROFILES = 50
const int MIN_PROFILE_BARS = 5
const int MAX_BARS_BACK_LIMIT = 5000
const int SAFETY_MARGIN = 100
```

---

## 8. COMPLETE DATA FLOW DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INPUT: PRICE & VOLUME DATA                          │
│                    (OHLCV from TradingView chart bars)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAGE 1: ANCHORING EVENT DETECTION                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌───────────────────┐  ┌──────────────────────┐      │
│  │  Swing Mode    │  │  Structure Mode   │  │  Delta Mode          │      │
│  ├────────────────┤  ├───────────────────┤  ├──────────────────────┤      │
│  │ ta.pivothigh() │  │ LibPvot.market    │  │ LibPvot.market       │      │
│  │ ta.pivotlow()  │  │ Structure(price)  │  │ Structure(CumDelta)  │      │
│  │ Impulse Delta  │  │ Pivot Detection   │  │ Delta Pivot          │      │
│  │ Baseline Track │  │ HH/LL Sequence    │  │ HH/LL Sequence       │      │
│  │ delay = pivRi  │  │ delay = 0         │  │ delay = 0            │      │
│  └────────────────┘  └───────────────────┘  └──────────────────────┘      │
│                                      │                                      │
│                                      ▼                                      │
│                        prfReset = true (trigger)                            │
│                                      │                                      │
│                      ┌───────────────┴───────────────┐                     │
│                      │ Profile Lifecycle Management  │                     │
│                      ├───────────────────────────────┤                     │
│                      │ if prfReset:                  │                     │
│                      │   - Clear msProf              │                     │
│                      │   - Initialize ranges         │                     │
│                      │   - Store prev boundaries     │                     │
│                      │   - Capture historical peaks  │                     │
│                      │   resetBar := bar_index       │                     │
│                      └───────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 2: LOWER TIMEFRAME DATA COLLECTION                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Timeframe Selection (LibTmFr.autoLTF)                         │         │
│  │ - Auto: Optimal LTF < Chart TF                                │         │
│  │ - Manual: User-specified ibTF                                 │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Intra-Bar Profile Construction (intBarCalc)                   │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ var LibVPrf.VProf ibProf = createProfile()              │  │         │
│  │ │                                                          │  │         │
│  │ │ if LibTmFr.isNewPeriod(chartTF):                        │  │         │
│  │ │     ibProf.clear()                                      │  │         │
│  │ │     ibProf.setRanges(rangeUp, rangeLo)                 │  │         │
│  │ │                                                          │  │         │
│  │ │ if vldPrd:                                              │  │         │
│  │ │     ibProf.addBar() // Statistical volume distribution  │  │         │
│  │ │                                                          │  │         │
│  │ │ return [aBuy, aSell, rangeUp, rangeLo,                 │  │         │
│  │ │         cvd, cvdHi, cvdLo]                              │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Timeframe Conversion (request.security)                       │         │
│  │ - Synchronize ibTF data with chartTF bars                     │         │
│  │ - barmerge.gaps_off, barmerge.lookahead_off                   │         │
│  │ - Preserve array structures across timeframes                 │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│            [aBuy, aSell, rangeUp, rangeLo, cvd, cvdHi, cvdLo]              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAGE 3: STATISTICAL DISTRIBUTION ENGINE                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Profile Object Creation (createProfile)                       │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ LibVPrf.create(                                          │  │         │
│  │ │   buckets   = rowNum        // Default: 24              │  │         │
│  │ │   dynamic   = true           // Auto-adjust ranges      │  │         │
│  │ │   valueArea = valArea        // Default: 70%            │  │         │
│  │ │   allot     = alltMod        // PDF or Classic          │  │         │
│  │ │   estimator = prcEst         // t4Skew (default)        │  │         │
│  │ │   cdfSteps  = intStp         // Default: 20 steps       │  │         │
│  │ │   split     = volEst         // Dynamic or Classic      │  │         │
│  │ │   trendLen  = trendLen       // Default: 3 bars         │  │         │
│  │ │ )                                                        │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  │ Returns: LibVPrf.VProf UDT                                    │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Volume Merging Loop (every bar if not na(aBuy[delay]))       │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ msProf.merge(                                            │  │         │
│  │ │   srcABuy    = aBuy[delay]    // Buy volume array       │  │         │
│  │ │   srcASell   = aSell[delay]   // Sell volume array      │  │         │
│  │ │   srcRangeUp = rangeUp[delay] // Upper price bound      │  │         │
│  │ │   srcRangeLo = rangeLo[delay] // Lower price bound      │  │         │
│  │ │ )                                                        │  │         │
│  │ │                                                          │  │         │
│  │ │ if rowDyn:                                               │  │         │
│  │ │   // Dynamic bucket adjustment per price range %        │  │         │
│  │ │   msProf.setBuckets(                                    │  │         │
│  │ │     int(rowNum * priceRangePercent * 100)               │  │         │
│  │ │   )                                                      │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Statistical Metrics Calculation (LibVPrf getters)             │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ [_, poc]    = msProf.getPoc()     // Max volume level   │  │         │
│  │ │ [_, vaUp, _, vaLo] = msProf.getVA() // 70% vol range   │  │         │
│  │ │ [_, vwap]   = msProf.getVwap()    // Vol-weighted avg   │  │         │
│  │ │ stdDev      = msProf.getStdDev()  // Statistical spread │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ Flash Profile Projection (barstate.islast)                    │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ if resetBar == bar_index:                                │  │         │
│  │ │   flashProf := msProf  // Direct clone                  │  │         │
│  │ │ else:                                                    │  │         │
│  │ │   // Swing mode: Merge past delay bars                  │  │         │
│  │ │   for i = (delay-1) to 0:                               │  │         │
│  │ │     flashProf.merge(aBuy[i], aSell[i], ...)             │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 4: PROFILE RENDERING                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ 4.1: POLYLINE HISTOGRAM CONSTRUCTION                          │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ 1. Calculate maxVol across all buckets                  │  │         │
│  │ │ 2. Initialize point arrays (bPoints, sPoints)           │  │         │
│  │ │ 3. For i = 0 to flashProf.buckets - 1:                 │  │         │
│  │ │      [bktUp, bktLo] = flashProf.getBktBnds(i)          │  │         │
│  │ │      buyVol  = flashProf.getBktBuyVol(i)               │  │         │
│  │ │      sellVol = flashProf.getBktSellVol(i)              │  │         │
│  │ │                                                         │  │         │
│  │ │      // Generate vertices based on volDisp mode        │  │         │
│  │ │      switch volDisp:                                   │  │         │
│  │ │        UpDn:  Stacked buy+sell polygons                │  │         │
│  │ │        Total: Single combined polygon                  │  │         │
│  │ │        Delta: Positive/negative delta polygons         │  │         │
│  │ │                                                         │  │         │
│  │ │ 4. Create polyline objects:                            │  │         │
│  │ │    polyline.new(bPoints, closed=true, fill=buyClr)     │  │         │
│  │ │    polyline.new(sPoints, closed=true, fill=sellClr)    │  │         │
│  │ │                                                         │  │         │
│  │ │ 5. FIFO cleanup if approaching MAX_POLYLINES           │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ 4.2: RECTANGLE FRAME (CURRENT ONLY)                           │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ if showRect:                                             │  │         │
│  │ │   box.new(                                               │  │         │
│  │ │     left   = startBar                                    │  │         │
│  │ │     right  = endBar                                      │  │         │
│  │ │     top    = flashProf.rangeUp                           │  │         │
│  │ │     bottom = flashProf.rangeLo                           │  │         │
│  │ │     border_color = rectBorderClr                         │  │         │
│  │ │     bgcolor = color.new(rectFillClr, rectTransp)         │  │         │
│  │ │   )                                                      │  │         │
│  │ │   array.push(allBoxes, newBox)                           │  │         │
│  │ │   FIFO cleanup if >= MAX_BOXES                           │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ 4.3: PEAK DETECTION & RECTANGLES (CURRENT ONLY)              │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ // Peak Detection:                                       │  │         │
│  │ │ 1. Calculate volumeThreshold = maxVol * 0.5              │  │         │
│  │ │ 2. Iterate buckets, find continuous zones >= threshold  │  │         │
│  │ │ 3. Store peaks in:                                       │  │         │
│  │ │    - currentPeakStarts/Ends (row indices)                │  │         │
│  │ │    - currentPeakStartPrices/EndPrices (prices)           │  │         │
│  │ │                                                          │  │         │
│  │ │ // Phase 1: Historical Profiles                         │  │         │
│  │ │ for profIdx in profileStartBars:                        │  │         │
│  │ │   // Safety checks (buffer limit, 5-bar minimum)        │  │         │
│  │ │   for peakIdx in profile peaks:                         │  │         │
│  │ │     Get prices from flattened arrays                    │  │         │
│  │ │     Create box extending to histEndBar + extension      │  │         │
│  │ │                                                          │  │         │
│  │ │ // Phase 2: Current Profile                             │  │         │
│  │ │ Delete previous bar's current profile boxes             │  │         │
│  │ │ if currentProfileBarCount >= MIN_PROFILE_BARS:          │  │         │
│  │ │   for peakIdx in currentPeakStarts:                     │  │         │
│  │ │     Get prices from flashProf.getBktBnds()              │  │         │
│  │ │     Create box extending to bar_index + extension       │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                      │                                      │
│                                      ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ 4.4: DEVELOPING STATISTICAL LINES                             │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ // Conditional plotting based on toggles:                │  │         │
│  │ │                                                          │  │         │
│  │ │ if showDevVA:                                            │  │         │
│  │ │   plot(vaUp[offset], style=stepline, offset=-pivRi)     │  │         │
│  │ │   plot(vaLo[offset], style=stepline, offset=-pivRi)     │  │         │
│  │ │   fill(vaUp, vaLo, color=#86549b)                       │  │         │
│  │ │                                                          │  │         │
│  │ │ if showDevPOC:                                           │  │         │
│  │ │   plot(poc[offset], style=stepline, offset=-pivRi)      │  │         │
│  │ │                                                          │  │         │
│  │ │ if showDevStdDev:                                        │  │         │
│  │ │   plot(vwap[offset] ± stdDev[offset], offset=-pivRi)    │  │         │
│  │ │   fill(stdUp, stdLo, color=#b6a72b)                     │  │         │
│  │ │                                                          │  │         │
│  │ │ if showDevVWAP:                                          │  │         │
│  │ │   plot(vwap[offset], style=line, offset=-pivRi)         │  │         │
│  │ │                                                          │  │         │
│  │ │ // Offset logic:                                         │  │         │
│  │ │ // Swing:           offset = 0    (retroactive)          │  │         │
│  │ │ // Structure/Delta: offset = pivRi (non-repainting)      │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STAGE 5: HISTORICAL PROFILE CAPTURE                      │
│                              (CURRENT ONLY)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────┐         │
│  │ On prfReset (anchor event):                                   │         │
│  │ ┌──────────────────────────────────────────────────────────┐  │         │
│  │ │ if not isFirstReset and not na(resetBar):                │  │         │
│  │ │   // Store completed profile metadata                    │  │         │
│  │ │   array.push(profileStartBars, prevResetBar)             │  │         │
│  │ │   array.push(profileEndBars, bar_index - 1)              │  │         │
│  │ │                                                          │  │         │
│  │ │   // Store peak data                                     │  │         │
│  │ │   peakCount = array.size(currentPeakStarts)              │  │         │
│  │ │   array.push(profilePeakCounts, peakCount)               │  │         │
│  │ │   array.push(profilePeakStartIdx, startIdx)              │  │         │
│  │ │                                                          │  │         │
│  │ │   // Flatten peaks to global arrays (PRICES!)            │  │         │
│  │ │   for i = 0 to peakCount - 1:                           │  │         │
│  │ │     array.push(allPeakStartPrices, startPrice)           │  │         │
│  │ │     array.push(allPeakEndPrices, endPrice)               │  │         │
│  │ │                                                          │  │         │
│  │ │   // FIFO cleanup if > MAX_HISTORICAL_PROFILES           │  │         │
│  │ │   if array.size(profileStartBars) > MAX:                 │  │         │
│  │ │     Shift oldest profile metadata                        │  │         │
│  │ │     Shift oldest peaks from flattened arrays             │  │         │
│  │ │     Adjust remaining peak indices                        │  │         │
│  │ │     Delete oldest boxes                                  │  │         │
│  │ │                                                          │  │         │
│  │ │ // Clear working arrays for new profile                 │  │         │
│  │ │ array.clear(currentPeakStarts/Ends/Prices)               │  │         │
│  │ └──────────────────────────────────────────────────────────┘  │         │
│  └────────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STAGE 6: ALERT CONDITIONS                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  alertcondition(prfReset, "Profile was resetted")                          │
│  alertcondition(ta.crossover(close[delay], vaUp), "Price crosses above VA")│
│  alertcondition(ta.crossunder(close[delay], vaUp), "Price crosses below VA")│
│  alertcondition(ta.crossover(close[delay], poc), "Price crosses above POC") │
│  alertcondition(ta.crossunder(close[delay], poc), "Price crosses below POC")│
│  alertcondition(ta.crossover(close[delay], vaLo), "Price crosses above VA") │
│  alertcondition(ta.crossunder(close[delay], vaLo), "Price crosses below VA")│
│  alertcondition(ta.crossover(close[delay], vwap±stdDev), "StdDev bands")   │
│  alertcondition(ta.crossover(close[delay], vwap), "VWAP cross")            │
│                                                                             │
│  Note: delay = pivRi for Swing mode (retroactive), 0 for Structure/Delta   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. LIBRARY DEPENDENCIES

### 9.1 External Libraries (lines 138-141)

**Import Statements:**
```pine
import AustrianTradingMachine/LibTmFr/1 as LibTmFr
import AustrianTradingMachine/LibBrSt/1 as LibBrSt
import AustrianTradingMachine/LibPvot/1 as LibPvot
import AustrianTradingMachine/LibVPrf/1 as LibVPrf
```

**Library Purposes:**

1. **LibTmFr (Timeframe Library):**
   - `autoLTF(chartTF)`: Auto-select optimal intra-bar timeframe
   - `isNewPeriod(chartTF)`: Detect chart timeframe bar boundaries
   - `isMultipleTF(chartTF)`: Validate multi-timeframe compatibility

2. **LibBrSt (Bar Statistics Library):**
   - `PriceEst` enum: Statistical models (t4Skew, etc.)
   - Used by LibVPrf for PDF-based volume allocation

3. **LibPvot (Pivot Library):**
   - `marketStructure()`: Detect HH/LL sequences for Structure/Delta anchoring
   - Returns: pivot arrays and trend state

4. **LibVPrf (Volume Profile Library):**
   - **Core Type:** `VProf` UDT (User-Defined Type)
   - **Factory:** `create()` - Initialize profile object
   - **Methods:**
     - `clear()`: Reset profile data
     - `setRanges(rangeUp, rangeLo)`: Set price boundaries
     - `setBuckets(count)`: Adjust bucket count dynamically
     - `addBar()`: Accumulate current bar volume (intra-bar)
     - `merge(srcABuy, srcASell, ...)`: Merge intra-bar data
     - `getPoc()`: Get Point of Control
     - `getVA()`: Get Value Area High/Low
     - `getVwap()`: Get Volume-Weighted Average Price
     - `getStdDev()`: Get Standard Deviation
     - `getBktBnds(i)`: Get bucket i's price boundaries
     - `getBktBuyVol(i)`: Get buy volume in bucket i
     - `getBktSellVol(i)`: Get sell volume in bucket i
     - `clone()`: Create deep copy of profile
   - **Enums:**
     - `AllotMode`: pdf, classic
     - `SplitMode`: dynamic, classic

---

## 10. KEY ARCHITECTURAL DECISIONS

### 10.1 Event-Based vs Time-Based Anchoring

**Decision:** Market structure events (pivots, structure breaks, delta breaks)
**Rationale:** Aligns profiles with meaningful market phases vs arbitrary time windows
**Trade-offs:**
- ✅ Profiles capture structural turning points
- ✅ More relevant to market dynamics
- ❌ Requires confirmation lag (pivRi bars)
- ❌ Profile durations vary (not fixed time periods)

### 10.2 Statistical vs Simple Volume Distribution

**Decision:** PDF-based allocation with T4-Skew estimator (default)
**Rationale:** Better approximates intra-bar price-time distribution
**Trade-offs:**
- ✅ More accurate representation of actual volume distribution
- ✅ Smoother profiles vs spike artifacts
- ❌ Computational overhead (Simpson integration)
- ❌ Requires cdfSteps parameter tuning

### 10.3 Dynamic vs Fixed Bucket Count

**Decision:** Dynamic row sizing based on price range percentage (default: enabled)
**Rationale:** Maintains consistent visual resolution across different profile heights
**Trade-offs:**
- ✅ Adaptive to market volatility
- ✅ Prevents over/under-sampling
- ❌ Profile resolution varies per profile
- ❌ Bucket count recalculated every merge

### 10.4 Swing Mode Retroactive Plotting

**Decision:** Delay profile start to historical pivot bar (delay = pivRi)
**Rationale:** Aligns profile with actual swing origin, not confirmation bar
**Trade-offs:**
- ✅ Profiles visually aligned with pivot points
- ✅ Historical accuracy
- ❌ Full pivRi bar lag for all data
- ❌ Requires flash profile projection logic

### 10.5 CURRENT: Historical Profile Storage

**Decision:** Flattened array architecture with FIFO management
**Rationale:** Preserve completed profiles for context, extend peaks forward
**Trade-offs:**
- ✅ Up to 50 historical profiles visible
- ✅ Peak zones extended for resistance/support tracking
- ✅ Efficient peak storage (prices, not row indices)
- ❌ Complex index management on FIFO deletion
- ❌ Higher memory footprint
- ❌ Additional box object overhead

### 10.6 CURRENT: Peak Detection Threshold

**Decision:** 50% of maximum volume defines "high-volume zones"
**Rationale:** Captures significant accumulation areas, filters noise
**Trade-offs:**
- ✅ Clear visual identification of key zones
- ✅ Simple, interpretable threshold
- ❌ Fixed percentage may not suit all market conditions
- ❌ No adaptive threshold based on profile characteristics

---

## 11. PERFORMANCE CHARACTERISTICS

### 11.1 Computational Complexity

**Per-Bar Costs:**
1. **Anchoring Detection:** O(pivLe + pivRi) for pivot calculation
2. **Intra-Bar Processing:** O(buckets * cdfSteps) for PDF integration
3. **Profile Merging:** O(buckets) array operations
4. **Metric Calculation:** O(buckets) for POC/VA/VWAP
5. **Polyline Rendering:** O(buckets) point generation (barstate.islast only)
6. **CURRENT: Peak Detection:** O(buckets) per profile (barstate.islast only)
7. **CURRENT: Peak Rectangles:** O(MAX_HISTORICAL_PROFILES * peaks) box creation

**Memory Footprint:**
- **ORIGINAL:** 2 profile objects (msProf, flashProf)
- **CURRENT:** 2 profile objects + flattened peak arrays + box/polyline arrays
- **CURRENT Peak Storage:** ~(50 profiles * avg 2-3 peaks * 2 prices) = ~200-300 floats

### 11.2 Pine Script Limits

**Constants:**
```pine
max_bars_back       = 5000  (indicator parameter)
max_polylines_count = 100   (indicator parameter)
MAX_BOXES           = 500   (Pine Script runtime limit)
MAX_POINTS          = 10000 (custom polyline vertex limit)
```

**Safety Mechanisms:**
- **Polylines:** FIFO deletion when approaching 100
- **Boxes:** FIFO deletion when approaching 500
- **Points:** Runtime error if exceeding 10000 vertices
- **CURRENT: Historical Buffer:** Skip profiles beyond MAX_BARS_BACK_LIMIT - SAFETY_MARGIN
- **CURRENT: Profile Minimum:** Skip profiles with < MIN_PROFILE_BARS (5 bars)

---

## 12. CONCLUSION

### 12.1 Proven Architecture Strengths (ORIGINAL)

1. **Clean Separation of Concerns:**
   - Anchoring logic isolated per mode
   - Statistical engine encapsulated in LibVPrf
   - Rendering logic separate from calculation

2. **Robust Statistical Foundation:**
   - PDF-based allocation with configurable models
   - Dynamic buy/sell splitting
   - Comprehensive metrics (POC, VA, VWAP, StdDev)

3. **Efficient Multi-Timeframe Integration:**
   - request.security for seamless LTF data access
   - No repainting with lookahead_off
   - Automatic timeframe validation

4. **Simple Memory Management:**
   - FIFO polyline cleanup
   - No complex historical storage
   - Predictable resource usage

### 12.2 CURRENT Implementation Enhancements

1. **Historical Context Visualization:**
   - Up to 50 profiles preserved
   - Peak zones extended forward for S/R tracking
   - Rectangle frames for profile boundaries

2. **Improved User Control:**
   - Independent toggles for developing lines
   - Peak rectangle configuration
   - Rectangle frame styling options

3. **Safety Improvements:**
   - Buffer limit checks (Issue #2 fix)
   - 5-bar minimum filter (Issue #6 fix)
   - Price-based peak storage (robust to profile range changes)

### 12.3 CURRENT Implementation Complexities

1. **Flattened Array Architecture:**
   - Index management on FIFO cleanup
   - Separate tracking of peaks per profile
   - Price vs row index considerations

2. **Two-Phase Peak Rendering:**
   - Historical vs current profile handling
   - Separate box arrays (allBoxes vs currentProfileBoxes)
   - Bar-by-bar cleanup logic

3. **Increased Memory Overhead:**
   - 50x historical profile metadata
   - Flattened peak arrays
   - Additional box objects

### 12.4 Recommendations

**For ORIGINAL-like Simplicity:**
- Remove historical storage
- Remove rectangle features
- Keep single-profile rendering

**For CURRENT Enhancements:**
- ✅ Keep flattened array architecture (robust design)
- ✅ Keep safety checks (buffer limit, 5-bar minimum)
- ✅ Keep price-based peak storage (not row indices)
- Consider: Add adaptive peak threshold (vs fixed 50%)
- Consider: Add historical profile count limit control
- Consider: Add peak extension bars as user input

---

## DOCUMENT VERSION

- **Version:** 1.0.0
- **Date:** 2025-01-11
- **Author:** System Architecture Designer
- **Files Analyzed:**
  - ORIGINAL: au-mktStructureVP-original.txt
  - CURRENT: au-mktStructureVP-FULL.pine (v1.0.2)
