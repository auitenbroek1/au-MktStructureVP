# Volume Profile Rendering Architecture Analysis

## Executive Summary

This document reverse-engineers the complete volume profile rendering system by comparing two architectural approaches:

- **WORKING version** (`au-mktStructureVP.pine`): Simple line-based visualization
- **FULL version** (`au-mktStructureVP-FULL.pine`): Polygon-based volume histogram rendering

The FULL version uses **TradingView's built-in polyline API** to draw actual volume profiles as filled polygons, not custom libraries or manual primitives.

---

## 1. WHAT DRAWS THE VOLUME HISTOGRAM/PROFILE?

### WORKING Version (Lines Only)
```pine
// Lines 165-175: Simple horizontal lines
line.new(bar_index - vpLookback, pocPrice, bar_index, pocPrice, ...)
label.new(bar_index, pocPrice, "POC", ...)
```

**Approach**: Draws 3 horizontal reference lines (POC, VAH, VAL)
- **No histogram visualization**
- **No volume distribution display**
- Only shows key price levels

### FULL Version (Actual Volume Profile)
```pine
// Lines 739-831: Polyline construction
array<chart.point> bPoints = array.new<chart.point>()
array<chart.point> sPoints = array.new<chart.point>()

// Build point arrays for buy/sell volumes
for i = 0 to flashProf.buckets - 1
    [bktUp, bktLo] = flashProf.getBktBnds(i)
    float buyVol = flashProf.getBktBuyVol(i)
    float buyWidth = barWidth * (buyVol / maxVol)
    float xEnd = xBase + direction * buyWidth
    array.push(bPoints, chart.point.from_index(int(xEnd), bktLo))
    array.push(bPoints, chart.point.from_index(int(xEnd), bktUp))

// Render as filled polygons
array.push(allPolylines, polyline.new(
  points     = bPoints,
  closed     = true,
  fill_color = buyClr))
```

**Key Discovery**: Uses TradingView's native **`polyline.new()`** function
- **NOT** a custom library
- **NOT** manual drawing primitives
- **Built-in TradingView feature** for polygon rendering
- Supports up to 100 polylines per indicator (line 145: `MAX_POLYLINES = 100`)
- Each polyline can have up to 10,000 points (line 146: `MAX_POINTS = 10000`)

---

## 2. PROFILE DATA STRUCTURE

### WORKING Version (Simple Arrays)
```pine
// Lines 74-76: Basic volume accumulation
var float[] vpPriceArray = array.new_float(vpRows, 0)
var float[] vpVolumeArray = array.new_float(vpRows, 0)

// Lines 88-109: Direct volume distribution
for i = 0 to vpLookback - 1
    barVolume = volume[i]
    startRow = math.floor((barLow - lowestPrice) / rowHeight)
    endRow = math.floor((barHigh - lowestPrice) / rowHeight)
    volumePerRow = barVolume / rowCount

    for row = startRow to endRow
        currentVolume = array.get(vpVolumeArray, row)
        array.set(vpVolumeArray, row, currentVolume + volumePerRow)
```

**Structure**:
- 2 parallel arrays (price + volume)
- Linear volume distribution
- No buy/sell separation
- Recalculated on `barstate.islast`

### FULL Version (Advanced Statistical Model)
```pine
// Lines 430-443: Intra-bar statistical engine
var LibVPrf.VProf ibProf = createProfile()

createProfile() =>
    LibVPrf.create(
      buckets   = rowNum,
      rangeUp   = 0,
      rangeLo   = 0,
      dynamic   = true,
      valueArea = valArea,
      allot     = alltMod,        // PDF distribution model
      estimator = prcEst,         // T4-Skew statistical model
      cdfSteps  = intStp,         // Integration steps
      split     = volEst,         // Dynamic buy/sell split
      trendLen  = trendLen)

// Lines 449-450: Lower timeframe data collection
[aBuy, aSell, rangeUp, rangeLo, cvd, cvdHi, cvdLo] =
    request.security(syminfo.tickerid, ibTF, intBarCalc(), ...)
```

**Key Differences**:
1. **Buy/Sell Separation**: Maintains separate `aBuy` and `aSell` arrays
2. **Statistical Distribution**: Uses Probability Density Functions (PDF) instead of linear allocation
3. **Intra-bar Resolution**: Samples lower timeframe for accurate volume distribution
4. **Dynamic Ranges**: Profile bounds adjust based on actual volume distribution
5. **Advanced Metrics**: POC, VA, VWAP, StdDev calculated from distributions

---

## 3. RENDERING PIPELINE

### WORKING Version Pipeline
```
┌─────────────────┐
│  Bar Updates    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Accumulate Vol  │ (Lines 88-109: Only on barstate.islast)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Find POC/VA    │ (Lines 114-148: Single pass calculation)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Draw Lines     │ (Lines 165-175: 3 horizontal lines)
└─────────────────┘

TOTAL: 1 active profile, no history
```

**Issues**:
- **No persistence**: Profile disappears when lookback window moves
- **No history**: Cannot see previous market structure phases
- **Visual limitation**: Only shows key levels, not distribution

### FULL Version Pipeline
```
┌─────────────────┐
│  Bar Updates    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Event Detection (Lines 476-514)                    │
│  ├── Swing: Detect pivot changes in price/delta     │
│  ├── Structure: Detect HH/LL breaks                 │
│  └── Delta: Detect cumulative delta breaks          │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Profile Reset Logic (Lines 528-595)                │
│  ├── Capture previous profile to historical storage │
│  ├── Store peak zones (flattened array)             │
│  ├── Clear working arrays                           │
│  └── Initialize new profile                         │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Volume Accumulation (Lines 607-627)                │
│  ├── Merge intra-bar data (aBuy, aSell)             │
│  ├── Update profile ranges dynamically              │
│  ├── Calculate POC, VA, VWAP, StdDev                │
│  └── Store developing metrics                       │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Flash Profile Creation (Lines 629-648)             │
│  ├── Clone current profile for rendering            │
│  ├── Merge delayed bars (Swing mode: pivRi delay)   │
│  └── Finalize bucket counts                         │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Polygon Construction (Lines 739-831)               │
│  ├── Build chart.point arrays for each bucket       │
│  ├── Calculate horizontal bar widths                │
│  ├── Handle 3 display modes (Up/Down, Total, Delta) │
│  └── Create polylines with fill colors              │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Rectangle Frames (Lines 851-898)                   │
│  ├── Draw border around profile                     │
│  ├── FIFO box management (MAX_BOXES = 500)          │
│  └── Store in allBoxes array                        │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Peak Rectangles (Lines 919-1061)                   │
│  ├── Phase 1: Render historical profile peaks       │
│  ├── Phase 2: Render current profile peaks          │
│  ├── Extend peaks forward (peakExtensionBars)       │
│  └── Clean up on new bars                           │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  Developing Lines (Lines 1063-1115)                 │
│  ├── Plot VA bands (fill between)                   │
│  ├── Plot POC line                                  │
│  ├── Plot StdDev bands (fill between)               │
│  └── Plot VWAP line                                 │
└─────────────────────────────────────────────────────┘

TOTAL: Up to 50 historical profiles + 1 current profile
```

---

## 4. KEY ARCHITECTURAL DIFFERENCES

### Profile Lifecycle

| Aspect | WORKING Version | FULL Version |
|--------|----------------|--------------|
| **Trigger** | Time-based (lookback window) | Event-based (market structure) |
| **Persistence** | None (recalculated each bar) | Historical storage (50 profiles) |
| **Visibility** | 1 active profile | 51 profiles (50 historical + 1 current) |
| **Data Structure** | 2 arrays (price, volume) | Complex object with buy/sell split |
| **Rendering** | 3 lines + labels | Polygons + rectangles + developing lines |

### Volume Distribution

| Feature | WORKING Version | FULL Version |
|---------|----------------|--------------|
| **Allocation** | Linear (volumePerRow = barVolume / rowCount) | PDF statistical model (T4-Skew) |
| **Buy/Sell Split** | None (total volume only) | Dynamic model using wick analysis |
| **Resolution** | Chart timeframe bars | Intra-bar (lower timeframe) |
| **Accuracy** | Basic approximation | High-fidelity statistical estimation |

### Rendering Strategy

| Component | WORKING Version | FULL Version |
|-----------|----------------|--------------|
| **Profile Shape** | N/A (lines only) | Polylines (closed polygons) |
| **Colors** | 3 colors (POC, VAH, VAL) | 2 colors (buy, sell) + configurable |
| **Display Modes** | Single mode | 3 modes (Up/Down, Total, Delta) |
| **Frames** | None | Rectangle borders + peak highlights |
| **Metrics** | Static lines | Dynamic developing lines with lag handling |

---

## 5. OVERLAP/COLLISION ISSUES IN CURRENT VERSION

### Root Cause Analysis

The WORKING version has **no overlap issues** because it only renders 3 lines. However, if you were to implement multiple profiles using the WORKING approach, you would encounter:

#### Problem 1: No Historical Storage
```pine
// WORKING version recalculates on every bar
if showVP and barstate.islast
    // Resets arrays
    array.fill(vpVolumeArray, 0)

    // Recalculates from scratch
    for i = 0 to vpLookback - 1
        // Volume distribution logic
```

**Issue**: Previous profiles are lost when window moves

#### Problem 2: No Separation Mechanism
```pine
// All lines drawn at same X-coordinate
line.new(bar_index - vpLookback, pocPrice, bar_index, pocPrice, ...)
line.new(bar_index - vpLookback, vahPrice, bar_index, vahPrice, ...)
line.new(bar_index - vpLookback, valPrice, bar_index, valPrice, ...)
```

**Issue**: If multiple profiles existed, they would overlap completely

#### Problem 3: No Profile Boundary Management
```pine
// No concept of profile start/end bars
// No tracking of which bars belong to which profile
// No separation between adjacent profiles
```

**Issue**: Cannot distinguish between different market structure phases

### How FULL Version Solves This

#### Solution 1: Historical Profile Storage (Lines 454-469)
```pine
// Flattened array architecture for historical profiles
var array<int> profileStartBars = array.new<int>()
var array<int> profileEndBars = array.new<int>()
var array<int> profilePeakCounts = array.new<int>()
var array<int> profilePeakStartIdx = array.new<int>()

// Flattened peak data (all peaks concatenated)
var array<float> allPeakStartPrices = array.new<float>()
var array<float> allPeakEndPrices = array.new<float>()
```

**Each profile knows**:
- Start bar index
- End bar index
- Number of peaks
- Peak price boundaries

#### Solution 2: Profile Anchoring (Lines 528-595)
```pine
if prfReset
    if not na(resetBar)
        startBar := prevResetBar    // Use PREVIOUS profile's start
        endBar   := bar_index - 1   // End at bar before current reset

        // Store profile to historical arrays
        array.push(profileStartBars, startBar)
        array.push(profileEndBars, endBar)
```

**Each profile gets**:
- Unique time boundaries
- No overlap with adjacent profiles
- Clear separation between market phases

#### Solution 3: Spatial Separation (Lines 731-733)
```pine
float barWidth  = (endBar - startBar + 1) * profWidth / 100.0
int   xBase     = profSide == ProfSide.left ? startBar : endBar
int   direction = profSide == ProfSide.left ? 1 : -1
```

**Each profile renders**:
- From its own start bar
- To its own end bar
- With configurable width (% of period)
- Left or right aligned

---

## 6. RENDERING LIMITS & CONSTRAINTS

### TradingView Platform Limits

| Resource | Limit | FULL Version Usage | Management Strategy |
|----------|-------|-------------------|---------------------|
| **Polylines** | 100 | Up to 2 per profile (buy + sell) | FIFO deletion (lines 736-737) |
| **Boxes** | 500 | Profile frames + peak rectangles | FIFO deletion (lines 858-859) |
| **Points per Polyline** | 10,000 | 2 points per bucket | Error if exceeded (lines 810-811) |
| **max_bars_back** | 5000 | Historical buffer access | Safety margin check (lines 939-942) |

### Memory Management

```pine
// Lines 736-737: Polyline FIFO
while array.size(allPolylines) > MAX_POLYLINES - reqPolyLn
    polyline.delete(array.shift(allPolylines))

// Lines 858-859: Box FIFO
if array.size(allBoxes) >= MAX_BOXES - 1
    box.delete(array.shift(allBoxes))

// Lines 557-583: Historical profile FIFO
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
    int oldestPeakCount = array.shift(profilePeakCounts)
    // Remove oldest profile data
    // Adjust all remaining indices
```

**Critical Innovation**: Flattened array architecture
- Instead of `array<array<float>>` (not supported in Pine)
- Uses `array<int>` for indices + `array<float>` for concatenated data
- Enables historical storage without nested arrays

---

## 7. ARCHITECTURAL COMPARISON DIAGRAM

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        WORKING VERSION                                     │
│                                                                            │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐        │
│  │   Bar    │────▶│ Recalc   │────▶│  Find    │────▶│   Draw   │        │
│  │ Updates  │     │ Volume   │     │ POC/VA   │     │  Lines   │        │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘        │
│                                                                            │
│  Features:                                                                 │
│  ✓ Simple implementation                                                   │
│  ✓ Low resource usage                                                      │
│  ✗ No historical profiles                                                  │
│  ✗ No volume distribution visualization                                    │
│  ✗ No buy/sell separation                                                  │
│  ✗ Limited market structure context                                        │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                         FULL VERSION                                       │
│                                                                            │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐        │
│  │   Bar    │────▶│  Event   │────▶│ Profile  │────▶│  Volume  │        │
│  │ Updates  │     │ Detection│     │  Reset   │     │Accumulate│        │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘        │
│                                            │                               │
│                                            ▼                               │
│                    ┌────────────────────────────────────────┐             │
│                    │  Historical Storage (50 profiles)      │             │
│                    │  ┌──────────┬──────────┬──────────┐   │             │
│                    │  │Profile 1 │Profile 2 │Profile N │   │             │
│                    │  │Start: 10 │Start: 50 │Start: 90 │   │             │
│                    │  │End:   49 │End:   89 │End:  129 │   │             │
│                    │  │Peaks: 3  │Peaks: 2  │Peaks: 4  │   │             │
│                    │  └──────────┴──────────┴──────────┘   │             │
│                    └────────────────────────────────────────┘             │
│                                            │                               │
│                                            ▼                               │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐        │
│  │  Flash   │────▶│ Polygon  │────▶│Rectangle │────▶│   Peak   │        │
│  │ Profile  │     │Construct │     │  Frames  │     │  Zones   │        │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘        │
│                                                                            │
│                    ┌────────────────────────────────────────┐             │
│                    │  Developing Lines (offset handling)   │             │
│                    │  ├── Value Area (fill)                │             │
│                    │  ├── POC                              │             │
│                    │  ├── StdDev Bands (fill)              │             │
│                    │  └── VWAP                             │             │
│                    └────────────────────────────────────────┘             │
│                                                                            │
│  Features:                                                                 │
│  ✓ Event-driven anchoring (Swing, Structure, Delta)                       │
│  ✓ 50 historical + 1 current profile                                      │
│  ✓ Full volume distribution visualization                                 │
│  ✓ Buy/sell separation with statistical models                            │
│  ✓ 3 display modes (Up/Down, Total, Delta)                                │
│  ✓ Peak zone highlighting with forward extension                          │
│  ✓ Rectangle frames for visual organization                               │
│  ✓ Intra-bar resolution for accuracy                                      │
│  ✓ FIFO resource management for 100 polylines, 500 boxes                  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 8. CRITICAL INSIGHTS

### Why Polylines, Not Lines?

**Lines** (`line.new()`):
- Connect two points
- Can only draw straight lines
- No fill capability
- Limited visual representation

**Polylines** (`polyline.new()`):
- Connect multiple points
- Can create complex shapes (histograms, polygons)
- Support fill colors
- Enable histogram visualization

### The "Flash Profile" Pattern

```pine
// Lines 452-453: Clone pattern
var LibVPrf.VProf msProf = createProfile()
LibVPrf.VProf flashProf  = msProf.clone()
```

**Purpose**:
1. `msProf`: Accumulates volume continuously (developing profile)
2. `flashProf`: Snapshot for rendering at `barstate.islast`
3. In Swing mode: `flashProf` merges delayed bars to align with retroactive plotting

This pattern ensures:
- Developing metrics stay synchronized with lag
- Rendering uses complete data
- Historical profiles capture correct state

### Peak Detection Architecture

```pine
// Lines 685-728: Continuous zone detection
bool inPeak = false
int peakStart = 0

for i = 0 to flashProf.buckets - 1
    float rowVol = array.get(volumeArray, i)
    bool isHighVol = rowVol >= volumeThreshold

    if isHighVol and not inPeak
        peakStart := i      // Peak starts
        inPeak := true
    else if not isHighVol and inPeak
        // Peak ends - store zone
        array.push(currentPeakStarts, peakStart)
        array.push(currentPeakEnds, i - 1)
        inPeak := false
```

**Innovation**: Detects continuous high-volume zones, not just POC
- Captures value clusters, not single price levels
- Extends forward to show persistent significance
- Stores price boundaries (not row indices) for stability across range changes

---

## 9. RECOMMENDATIONS FOR MIGRATION

### If Starting Fresh (Greenfield)

1. **Use FULL architecture** for actual volume profiles
2. **Skip WORKING approach** entirely (too limited)
3. **Leverage TradingView polylines** (native support)
4. **Implement historical storage** from day 1

### If Enhancing WORKING Version

1. **Add polyline rendering** (lines 739-831 from FULL)
2. **Implement event anchoring** (lines 476-514 from FULL)
3. **Add historical storage** (lines 454-469 from FULL)
4. **Use LibVPrf for distributions** (import on line 142)

### Critical Dependencies

```pine
import AustrianTradingMachine/LibTmFr/1 as LibTmFr  // Timeframe utilities
import AustrianTradingMachine/LibBrSt/1 as LibBrSt  // Bar statistics
import AustrianTradingMachine/LibPvot/1 as LibPvot  // Pivot detection
import AustrianTradingMachine/LibVPrf/1 as LibVPrf  // Volume profile engine
```

**These libraries provide**:
- Intra-bar timeframe handling
- Statistical distribution models (T4-Skew PDF)
- Market structure detection
- Volume profile calculations (POC, VA, VWAP, StdDev)

**Cannot replicate FULL functionality without these libraries** unless you reimplement:
- PDF integration (Simpson's rule)
- T4-Skew distribution model
- Dynamic buy/sell split estimation
- Market structure pivot detection

---

## 10. CONCLUSION

### The Fundamental Difference

| WORKING | FULL |
|---------|------|
| Shows **where** volume concentrated (3 lines) | Shows **how** volume distributed (histogram) |
| Time-based window (moving) | Event-based anchoring (persistent) |
| Single active profile | 51 total profiles (historical context) |
| Basic calculation | Statistical modeling |

### The Architectural Leap

The FULL version doesn't just "add features" to WORKING - it represents a **completely different paradigm**:

1. **Data Structure**: From 2 arrays → Complex object with buy/sell split
2. **Rendering**: From 3 lines → Polygon-based histograms
3. **Lifecycle**: From ephemeral → Persistent with historical storage
4. **Accuracy**: From linear approximation → PDF statistical modeling
5. **Resolution**: From chart timeframe → Intra-bar sampling

### Key Takeaway

**You cannot gradually evolve WORKING into FULL**. The architectural foundations are incompatible. To get true volume profile visualization:

1. Start with polyline rendering
2. Implement historical storage
3. Add statistical distribution models
4. Integrate event-based anchoring

The TradingView `polyline.new()` API is the **only way** to draw actual volume profiles with filled histograms. There are no alternatives or workarounds within Pine Script constraints.
