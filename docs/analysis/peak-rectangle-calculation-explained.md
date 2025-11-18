# Peak Rectangle Calculation Explained: The 50% Threshold

**Date:** 2025-11-18
**Version:** 1.0
**Status:** Technical Documentation

---

## 1. Quick Answer

**Rectangle tops and bottoms are determined by finding continuous zones in the volume profile where volume exceeds 50% of the profile's maximum volume.** The top and bottom prices of each rectangle correspond to the price boundaries of these high-volume "peak" zones, not arbitrary percentages of the profile's price range.

---

## 2. The 50% Threshold Explained

### What is the 50% Threshold?

The 50% threshold is a **volume concentration filter** that identifies "significant" high-volume zones within a volume profile. Here's what it means:

- **Maximum Volume**: First, find the highest volume in any single row of the profile
- **Threshold Calculation**: `volumeThreshold = maxVol × 0.5`
- **Peak Identification**: Any row with volume ≥ threshold is part of a "peak zone"

### Why 50%?

The 50% threshold strikes a balance between:
- **Too Low (e.g., 30%)**: Would highlight too many zones, creating visual clutter
- **Too High (e.g., 70%)**: Would miss important secondary concentration zones
- **50%**: Captures primary and significant secondary volume accumulation zones

**Trading Significance:**
- Peaks represent **price levels with sustained trading interest**
- They often act as **support/resistance zones**
- **Value Area** typically falls within or near these peaks
- Multiple peaks indicate **multiple zones of price acceptance**

---

## 3. Step-by-Step Calculation Process

### Phase 1: Volume Profile Construction

```pine
// From lines 680-708 in au-mktStructureVP-v105.pine

// STEP 1: Find maximum volume across all rows
series float maxVol = 0.0
for i = 0 to flashProf.buckets - 1
    series float bVol = flashProf.getBktBuyVol(i)
    series float sVol = flashProf.getBktSellVol(i)
    series float currentVol = bVol + sVol  // Total volume (can also use delta mode)

    maxVol := math.max(maxVol, currentVol)  // Track maximum
```

**Example:**
```
Row 0 (Price $100.00): Buy=50, Sell=30  → Total=80
Row 1 (Price $100.10): Buy=150, Sell=120 → Total=270 ← Max Volume
Row 2 (Price $100.20): Buy=100, Sell=90  → Total=190
Row 3 (Price $100.30): Buy=60, Sell=40   → Total=100
Row 4 (Price $100.40): Buy=30, Sell=20   → Total=50

maxVol = 270
```

### Phase 2: Threshold Calculation

```pine
// From line 699 in au-mktStructureVP-v105.pine

// STEP 2: Calculate volume threshold (50% of maximum)
float volumeThreshold = maxVol * 0.5
```

**Example:**
```
volumeThreshold = 270 × 0.5 = 135

This means: Any row with volume ≥ 135 is part of a peak
```

### Phase 3: Peak Detection (Continuous Zones)

```pine
// From lines 710-755 in au-mktStructureVP-v105.pine

// STEP 3: Identify continuous high-volume zones
bool inPeak = false
int peakStart = 0

for i = 0 to flashProf.buckets - 1
    float rowVol = array.get(volumeArray, i)
    bool isHighVol = rowVol >= volumeThreshold

    if isHighVol and not inPeak
        // Peak starts at this row
        peakStart := i
        inPeak := true

    else if not isHighVol and inPeak
        // Peak ends at previous row
        int startRow = peakStart
        int endRow = i - 1

        // Get price boundaries for this peak zone
        [startPrice, _] = flashProf.getBktBnds(startRow)
        [endPrice, _] = flashProf.getBktBnds(endRow)

        // Store peak boundaries (PRICES, not rows!)
        array.push(currentPeakStartPrices, startPrice)
        array.push(currentPeakEndPrices, endPrice)

        inPeak := false

// Handle peak extending to last row
if inPeak
    [startPrice, _] = flashProf.getBktBnds(peakStart)
    [endPrice, _] = flashProf.getBktBnds(flashProf.buckets - 1)
    array.push(currentPeakStartPrices, startPrice)
    array.push(currentPeakEndPrices, endPrice)
```

**Example:**
```
Row 0 (Price $100.00): Volume=80   → Below threshold (80 < 135)
Row 1 (Price $100.10): Volume=270  → ABOVE threshold (270 ≥ 135) ← Peak starts
Row 2 (Price $100.20): Volume=190  → ABOVE threshold (190 ≥ 135)
Row 3 (Price $100.30): Volume=100  → Below threshold (100 < 135) ← Peak ends
Row 4 (Price $100.40): Volume=50   → Below threshold (50 < 135)

RESULT: One peak detected
  - Start Price: $100.10 (bottom of rectangle)
  - End Price: $100.20 (top of rectangle)
```

### Phase 4: Rectangle Rendering

```pine
// From lines 909-928 in au-mktStructureVP-v105.pine

// STEP 4: Create rectangle with peak price boundaries
float bottomPrice = array.get(allPeakStartPrices, peakStartIndex + peakIdx)
float topPrice = array.get(allPeakEndPrices, peakStartIndex + peakIdx)

int leftBar = histStartBar  // Profile start time
int rightBar = histEndBar + peakExtensionBars  // Profile end + extension

box peakBox = box.new(
  left   = leftBar,
  top    = topPrice,      // ← Upper price boundary of peak zone
  right  = rightBar,
  bottom = bottomPrice,   // ← Lower price boundary of peak zone

  border_color = showPeakBorder ? peakRectColor : color.new(color.black, 100),
  bgcolor      = peakRectFillColor)
```

---

## 4. Practical Example with Real Numbers

### Scenario: SPY 5-minute chart with 25-row profile

```
Profile Range: $450.00 - $451.20 (1.2 points)
Row Height: $0.05 per row
Volume Distribution:

Row  Price      Buy   Sell  Total  Above Threshold?
---  --------   ---   ----  -----  ----------------
 0   $450.00    10    8     18     No
 1   $450.05    15    12    27     No
 2   $450.10    50    45    95     No
 3   $450.15    120   100   220    YES ← Peak starts (220 ≥ 150)
 4   $450.20    180   150   330    YES (330 ≥ 150) ← Max volume
 5   $450.25    140   130   270    YES (270 ≥ 150)
 6   $450.30    160   140   300    YES (300 ≥ 150)
 7   $450.35    110   100   210    YES (210 ≥ 150)
 8   $450.40    90    80    170    YES (170 ≥ 150)
 9   $450.45    80    70    150    YES (150 ≥ 150) ← Peak ends at boundary
10   $450.50    60    50    110    No
11   $450.55    40    35    75     No
...
24   $451.20    5     4     9      No

Max Volume: 330 (Row 4)
Threshold: 330 × 0.5 = 165... Wait, let's recalculate!
```

**Error in Example Above**: The threshold is 165, not 150. Let me correct:

```
Max Volume: 330 (Row 4)
Threshold: 330 × 0.5 = 165

Row  Price      Total  Above Threshold?
---  --------   -----  ----------------
 3   $450.15    220    YES (220 ≥ 165) ← Peak starts
 4   $450.20    330    YES (330 ≥ 165) ← Max volume
 5   $450.25    270    YES (270 ≥ 165)
 6   $450.30    300    YES (300 ≥ 165)
 7   $450.35    210    YES (210 ≥ 165)
 8   $450.40    170    YES (170 ≥ 165) ← Peak ends
 9   $450.45    150    No (150 < 165)
```

**Result:**
```
Peak Rectangle Created:
  - Bottom Price: $450.15 (Row 3 bottom boundary)
  - Top Price: $450.40 (Row 8 top boundary)
  - Time: Profile start bar → Profile end bar + 100 bars extension
  - Visual: Red semi-transparent fill highlighting the $450.15-$450.40 zone
```

**Trading Interpretation:**
- **Price spent most time between $450.15-$450.40** (value zone)
- This zone likely contains the **Point of Control (POC)**
- Expect **support at $450.15** and **resistance at $450.40**
- Break above $450.40 = bullish signal
- Break below $450.15 = bearish signal

---

## 5. Code References (Exact Line Numbers)

### Volume Profile Construction
- **Lines 680-691**: Calculate maximum volume across all rows
- **Line 699**: Calculate 50% threshold: `volumeThreshold = maxVol * 0.5`

### Peak Detection Algorithm
- **Lines 701-708**: Build volume array for analysis
- **Lines 716-755**: Peak detection loop (continuous zone identification)
  - **Lines 728-731**: Peak start detection
  - **Lines 732-743**: Peak end detection with price capture
  - **Lines 746-755**: Handle peak extending to last row

### Peak Storage Architecture
- **Lines 535-538**: Current profile peak arrays (working storage)
- **Lines 479-480**: Historical profile flattened peak arrays (persistent storage)
- **Lines 774-776**: Copy current peaks to historical storage

### Rectangle Rendering
- **Lines 864-930**: Peak rectangle rendering (two-phase: historical + current)
- **Lines 909-911**: Get peak price boundaries from flattened arrays
- **Lines 917-927**: Create box with price boundaries

---

## 6. Trading Context: Why This Matters

### Volume Profile Theory

Traditional volume profiles show **where price spent time**, but peak rectangles highlight **where the MOST trading occurred**. This is crucial because:

1. **High-Volume Nodes = Strong Price Memory**
   - Traders remember zones with heavy activity
   - Future price action respects these levels

2. **Multiple Peaks = Market Disagreement**
   - Single peak: Strong consensus on value
   - Multiple peaks: Buyers/sellers active at different levels
   - Gaps between peaks: Price rejection zones

3. **Peak Width (Vertical) = Price Acceptance Range**
   - Narrow peak (2-3 rows): Precise value agreement
   - Wide peak (8-10 rows): Broad value acceptance
   - Example above: $450.15-$450.40 = 5-row wide peak (moderate consensus)

4. **50% Threshold = Institutional Footprint**
   - Retail traders scatter volume randomly
   - Institutional orders cluster volume at specific prices
   - 50% threshold filters out noise, revealing institutional zones

### Comparison to Value Area

| Metric | Definition | Purpose |
|--------|------------|---------|
| **Value Area (VA)** | Price range containing 70% of total volume | Statistical distribution center |
| **Peak Rectangles** | Continuous zones with volume ≥50% of max | High-concentration trading zones |

**Key Difference:**
- VA is calculated by **accumulating volume until 70% threshold reached** (statistical)
- Peaks are identified by **row-by-row volume comparison to max** (threshold-based)

**Relationship:**
- Peaks usually fall **within or near** the Value Area
- VA can span multiple peaks (showing value accepted across wider range)
- Peaks can extend **outside VA** (showing secondary accumulation zones)

---

## 7. Advanced Details

### Can the 50% Threshold Be Changed?

**Currently: No** - The threshold is hardcoded at line 699:
```pine
float volumeThreshold = maxVol * 0.5  // Fixed at 50%
```

**Future Enhancement Possibility:**
Add user input parameter in the "rectangle frame" group:
```pine
float peakThresholdPct = input.float(
  title   = "Peak Threshold %",
  group   = gR,
  defval  = 50,
  minval  = 20,
  maxval  = 90,
  step    = 5,
  tooltip = "Percentage of maximum volume to identify peaks (50% = half of max volume)")

// Then change line 699 to:
float volumeThreshold = maxVol * (peakThresholdPct / 100.0)
```

**Impact of Different Thresholds:**
- **30-40%**: More peaks detected (shows secondary accumulation zones)
- **50%**: Balanced (default - shows primary concentration zones)
- **60-80%**: Fewer peaks (only highest concentration zones)
- **>80%**: Very few peaks (only extreme volume spikes - may miss important zones)

### What If Multiple Disconnected Peaks Exist?

**The algorithm handles this naturally:**

```
Row 0: Volume=50   → Below threshold
Row 1: Volume=200  → Peak 1 starts
Row 2: Volume=180  → Peak 1 continues
Row 3: Volume=120  → Below threshold → Peak 1 ends
Row 4: Volume=100  → Below threshold
Row 5: Volume=190  → Peak 2 starts
Row 6: Volume=210  → Peak 2 continues
Row 7: Volume=180  → Peak 2 continues
Row 8: Volume=100  → Below threshold → Peak 2 ends
```

**Result: Two separate rectangles:**
```
Rectangle 1: Rows 1-2 (continuous high-volume zone)
Rectangle 2: Rows 5-7 (second continuous high-volume zone)
```

**No Limit:** The code supports **unlimited peaks per profile** (limited only by Pine Script's 500-box constraint across ALL profiles)

### Edge Cases

#### Case 1: No Rows Exceed Threshold
```
Situation: maxVol = 100, threshold = 50, but all rows have volume < 50
Result: No peak rectangles drawn (no zones meet criteria)
Visual: Only profile boundary rectangle visible (if enabled)
```

#### Case 2: All Rows Exceed Threshold
```
Situation: Perfectly flat volume distribution where all rows ≥ threshold
Result: One giant peak rectangle spanning entire profile height
Interpretation: Uniform trading across entire price range (rare)
```

#### Case 3: Single-Row Peak
```
Situation: Only one row exceeds threshold
Result: Thin peak rectangle (height = 1 row)
Visual: Single horizontal band highlighting precise price level
```

### Performance Implications

**Peak Detection Complexity:**
```
Time Complexity: O(n) where n = number of rows (buckets)
Space Complexity: O(p) where p = number of peaks detected
Maximum Peaks: Limited by Pine Script box limit (500 total boxes)
```

**Optimization (Line 699-755):**
- Single-pass algorithm (no nested loops)
- Direct array operations (no sorting required)
- Runs only in `barstate.islast` (once per bar, not on every tick)

---

## 8. Visual Diagram

```
Volume Profile with 50% Threshold Peak Detection

Price
↑
$100.40 |────┐                          ← Top of Peak Rectangle
        |    │ ███████████████
$100.30 |    │ ████████████████ 300 vol (>165) ← In Peak
        |    │ ████████████ 270 vol (>165) ← In Peak
$100.20 |    │ ████████████████ 330 vol (MAX) ← In Peak
        |    │ █████████ 220 vol (>165) ← In Peak
$100.10 |    │ ████ 190 vol (>165) ← In Peak
        |────┘                          ← Bottom of Peak Rectangle
$100.00 |── 80 vol (below threshold)
        └────────────────────────────────────────
               Volume →

Legend:
█ = Volume bars
─ = Peak rectangle boundary
Threshold Line: 165 (50% of 330 max)
Peak Zone: All rows where volume ≥ 165
```

**Key Observations:**
1. Peak rectangle spans vertically across ALL consecutive rows exceeding threshold
2. Rectangle does NOT include row at $100.00 (volume=80 < 165)
3. Rectangle edges align with **price boundaries of threshold-exceeding rows**
4. Multiple peaks would create multiple separate rectangles

---

## 9. Common Questions Answered

### Q1: Does the 50% apply to the total profile volume?
**No.** The 50% applies to the **maximum volume in a single row**, not the sum of all volumes.

**Incorrect Interpretation:**
```
Total Profile Volume = 2000
50% of Total = 1000
❌ Wrong: Looking for rows with 1000 volume
```

**Correct Interpretation:**
```
Maximum Row Volume = 330
50% of Maximum = 165
✓ Correct: Looking for rows with volume ≥ 165
```

### Q2: Why use row-level maximum instead of total?
**Normalization:** Each profile has different total volumes (depends on profile duration). Using row maximum makes the threshold **relative to the profile's intensity**, not its absolute size.

**Example:**
- 5-minute profile: Max row = 500 → Threshold = 250
- 1-hour profile: Max row = 3000 → Threshold = 1500

Both use same 50% ratio, but absolute thresholds scale appropriately.

### Q3: Can I see the POC separately from peak rectangles?
**Yes.** These are independent features:
- **POC** (Point of Control): Single row with highest volume (line 637)
- **Peak Rectangles**: All continuous zones with volume ≥50% of max (lines 710-755)

**Relationship:**
- POC is **always within a peak rectangle** (by definition)
- A peak rectangle **may contain multiple rows** surrounding the POC
- POC = **precise level**, Peak = **zone around that level**

### Q4: What if I'm using "Delta" display mode?
**The threshold calculation adapts automatically** (line 686-689):

```pine
series float currentVol = switch volDisp
    VolDisp.upDn  => bVol + sVol        // Total volume
    VolDisp.total => bVol + sVol        // Total volume
    VolDisp.delta => math.abs(bVol - sVol)  // Absolute delta
```

**Delta Mode Behavior:**
- Threshold = 50% of **maximum absolute delta**
- Peaks highlight zones with **strongest buy/sell imbalance**
- Useful for spotting **directional conviction zones**

---

## 10. Conclusion

**Summary:**
1. **50% threshold identifies high-volume concentration zones**, not price range percentages
2. **Rectangle top/bottom = price boundaries of continuous rows exceeding threshold**
3. **Algorithm is dynamic** - adapts to each profile's unique volume distribution
4. **Multiple peaks are supported** - each gets its own rectangle
5. **Trading value**: Highlights institutional footprint and key support/resistance zones

**Code Location:**
- **Threshold Calculation**: Line 699
- **Peak Detection**: Lines 710-755
- **Rectangle Rendering**: Lines 864-930

**Next Steps:**
- Experiment with different threshold values (requires code modification)
- Compare peak rectangles to Value Area boundaries
- Observe how peaks align with price action (bounces, breakouts)

---

**Document Version History:**

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-18 | Initial comprehensive explanation |

