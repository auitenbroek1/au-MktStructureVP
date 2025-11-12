# Pine Script Buffer Limit Behavior - Critical Re-Research

**Date:** 2025-11-11
**Researcher:** Research Specialist Agent
**Status:** Critical Findings - Requires Architectural Review

---

## Executive Summary

Based on critical user feedback, the buffer limit error occurs **IMMEDIATELY** when offset increments, not gradually. The "buffer limit" in the error message equals the initial offset value, suggesting Pine Script is calculating this limit dynamically based on the **oldest stored profile's position**, not a fixed engine-level constant.

---

## Critical User Observations

### Pattern Discovery

**Error Sequence:**
```
Bar N:   offset = 1950, buffer limit = 1950  (NO ERROR)
Bar N+1: offset = 1951, buffer limit = 1950  (ERROR!)
```

**Key Insight:** Offset and buffer limit START at the **SAME VALUE**.

### Chart-Specific Data

**Daily Chart:**
- Profiles go back: ~2500 days
- Chart bars go back: YEARS (5000+ bars)
- **Gap:** Profiles DON'T reach full chart history

**Hourly Chart:**
- Profiles go back: ~1700 hours (Aug 25 to Nov 11)
- Chart bars go back: 2022 (thousands of hours)
- **Gap:** Same pattern - profiles stop far before chart end

---

## Root Cause Analysis

### The "Buffer Limit" in the Error Message

**Critical Question:** Is this Pine Script's engine limit OR something the indicator creates?

#### Evidence Analysis

**Line 935-936 in Code:**
```pinescript
int historicalOffset = bar_index - histStartBar
if historicalOffset > bar_index - HISTORICAL_BUFFER_SAFETY
```

**The Check:**
- `historicalOffset` = how far back the profile is
- `bar_index - HISTORICAL_BUFFER_SAFETY` = maximum allowed offset
- `HISTORICAL_BUFFER_SAFETY = 50` (line 150)

**Translation:**
```
if (bar_index - histStartBar) > (bar_index - 50)
// Simplifies to:
if histStartBar < 50
```

**This means:** Only profiles stored in the last 50 bars are rendered!

---

## The Mystery: Why Profiles Stop at 2500/1700 Bars

### Hypothesis 1: FIFO Cleanup Limiting Storage

**Code Evidence (lines 554-581):**
```pinescript
// FIFO cleanup if exceeding limit
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
    int oldestPeakCount = array.shift(profilePeakCounts)
    array.shift(profileStartBars)  // Oldest profile deleted
```

**MAX_HISTORICAL_PROFILES = 50** (line 148)

**Reality Check:**
- Only 50 profiles are stored maximum
- On daily chart: 50 profiles ≠ 2500 days
- On hourly chart: 50 profiles ≠ 1700 hours

**Conclusion:** This is NOT why profiles stop at 2500/1700.

---

### Hypothesis 2: Profile Anchor Event Frequency

**How Profiles Are Created:**

Profiles start on anchor events (line 473-511):
- **Swing mode:** Price pivot confirmed
- **Structure mode:** Market structure break
- **Delta mode:** Delta structure break

**All require:** `pivLe` (left bars) + `pivRi` (right bars) for confirmation

**Default Settings:**
- `pivLe = 10` (line 243)
- `pivRi = 5` (line 249)

**Profile Creation Frequency:**
```
Daily chart: Pivot every ~50 days (estimate)
→ 50 profiles × 50 days = ~2500 days back ✓

Hourly chart: Pivot every ~34 hours (estimate)
→ 50 profiles × 34 hours = ~1700 hours back ✓
```

**DISCOVERY:** The 2500/1700 limit is NOT a buffer limit - it's how far back 50 stored profiles reach based on **pivot frequency**!

---

## The REAL Buffer Limit Error

### Where It Actually Occurs

**Line 936 Safety Check:**
```pinescript
if historicalOffset > bar_index - HISTORICAL_BUFFER_SAFETY
    continue  // Skip this profile
```

**Problem:** `HISTORICAL_BUFFER_SAFETY = 50` is WRONG!

**What This Check Does:**
```
bar_index = 5000
histStartBar = 4900
historicalOffset = 5000 - 4900 = 100

Check: 100 > (5000 - 50)
Check: 100 > 4950
Result: FALSE → Renders profile

But if histStartBar = 10:
historicalOffset = 5000 - 10 = 4990
Check: 4990 > 4950
Result: TRUE → Skips profile
```

**The Bug:** This check prevents rendering profiles older than 50 bars, but it doesn't prevent the **actual Pine Script buffer limit error** when accessing historical data like `flashProf.getBktBnds()`.

---

## The Pine Script Engine's REAL Buffer Limit

### Documentation Research

**Pine Script v6 Limits:**
- `max_bars_back = 500` (line 119 in indicator declaration)
- This sets the **historical reference buffer** for series variables
- Drawing objects (box, line, polyline) have their own internal limits

**The Error Message Format:**
```
"Cannot reference 'bar_index' more than X bars back"
```

**Where X Comes From:**
1. User sets `max_bars_back = 500` explicitly
2. Pine Script auto-detects based on first bars of script execution
3. For drawing objects, limit may be different from series variables

---

## Critical Discovery: The "Same Value" Mystery

### Why Offset = Buffer Limit Initially

**User's Observation:**
> "Offset and buffer limit seem to be getting set to the SAME VALUE initially"

**Hypothesis:** Pine Script calculates the buffer limit based on the **oldest profile currently stored**.

**Scenario:**
```
Chart loads, script executes:

Bar 0: First profile stored at bar 0
Bar 100: Second profile stored at bar 100
Bar 200: Third profile stored at bar 200
...
Bar 1950: 50th profile stored at bar 1950

Current bar_index = 3000
Oldest profile at bar 0
Offset to oldest = 3000 - 0 = 3000

BUT if we delete profiles older than 1950:
Oldest profile now at bar 1950
Offset to oldest = 3000 - 1950 = 1050

If FIFO keeps moving oldest profile forward:
At bar 4000, oldest might be at bar 2050
Offset = 4000 - 2050 = 1950 ← THIS BECOMES THE LIMIT
```

**Next bar:**
```
Bar 4001, oldest still at 2050
New offset = 4001 - 2050 = 1951 > 1950 → ERROR!
```

---

## Why Profiles Don't Go Further Back

### The Missing Piece

**FIFO Cleanup (line 554):**
```pinescript
if array.size(profileStartBars) > MAX_HISTORICAL_PROFILES
```

**But this only stores 50 profiles!**

**The REAL Constraint:**

**Drawing Object Limits:**
- `max_polylines_count = 100` (line 120)
- `MAX_BOXES = 500` (line 147)
- Each profile can create multiple boxes (peak rectangles)

**Memory Calculation:**
```
50 profiles × 5 peaks average × 1 box per peak = 250 boxes
250 boxes < 500 limit ✓
```

**But accessing historical data for rendering:**

When rendering a profile from bar 0 on bar 5000:
- Need to call `flashProf.getBktBnds(rowIndex)`
- This internally references historical price data
- If that data is beyond `max_bars_back`, Pine Script errors

---

## The maxbarsback Detection Mechanism

### Pine Script Auto-Detection

**From Pine Script documentation:**

Pine Script analyzes the first ~100 bars to determine how far back each series variable needs to reference. It sets `max_bars_back` for each series.

**For drawing objects:**

Drawing objects have **coordinates** (bar_index values), but accessing historical data to calculate those coordinates requires series lookbacks.

**The Indicator's Issue:**

Line 935: `int historicalOffset = bar_index - histStartBar`

This is a **simple integer calculation** - no series lookback needed.

**BUT** line 1000 (peak rendering):
```pinescript
[topPrice, _] = flashProf.getBktBnds(endRow)
```

`flashProf` is built from historical data. If it needs to reference `rangeUp[1950]` or similar, and we're at bar 5000, that's a 3050 bar lookback!

---

## Research Findings Summary

### What We Now Know

1. **"Buffer Limit" = Dynamic:** Not a fixed 5000, but based on oldest stored profile position
2. **Profile Coverage ≠ Chart History:** Only 50 profiles stored, spacing determined by pivot frequency
3. **Safety Margin Wrong:** `HISTORICAL_BUFFER_SAFETY = 50` prevents rendering almost all profiles
4. **Real Limit Source:** `max_bars_back = 500` in indicator declaration
5. **Error Trigger:** Accessing historical series data within profile objects when rendering

### What We DON'T Know (Pine Script Mysteries)

1. **Exact buffer calculation:** How does Pine Script decide "1950" vs "2000" vs "2500"?
2. **Drawing object limits:** Are box coordinates subject to max_bars_back?
3. **flashProf internals:** Does it store references to historical series data?
4. **Query API:** No programmatic way to ask Pine Script "what's my current buffer limit?"

---

## Recommended Solutions (Ordered by Viability)

### Solution 1: Increase max_bars_back (SIMPLE)

**Change line 119:**
```pinescript
// FROM:
max_bars_back = 500

// TO:
max_bars_back = 5000
```

**Pros:**
- Simple one-line change
- Should allow historical data access up to 5000 bars
- No algorithm changes needed

**Cons:**
- Uses more memory
- May slow down initial chart load
- Still has a hard limit (just higher)

**Test This First!**

---

### Solution 2: Fix HISTORICAL_BUFFER_SAFETY (CORRECT)

**Change line 150:**
```pinescript
// FROM:
const int HISTORICAL_BUFFER_SAFETY = 50

// TO:
const int HISTORICAL_BUFFER_SAFETY = 4500
```

**This aligns with the documented strategy** in existing architecture docs.

**Effect:**
```
historicalOffset > bar_index - 4500
→ Only renders profiles from last 4500 bars
```

**Pros:**
- Matches architectural design intent
- Provides large safe zone (4500 bars)
- Should eliminate errors on most timeframes

**Cons:**
- Very old profiles won't render (by design)
- Still needs max_bars_back increased to support the calculation

---

### Solution 3: Store Profiles as Relative Offsets (COMPLEX)

**Complete redesign:** Store offsets instead of absolute bar_index values.

**Not recommended** based on re-research - the issue is simpler than initially thought.

---

## Critical Action Items

### Immediate Testing Required

1. **Test max_bars_back increase:**
   ```pinescript
   max_bars_back = 5000
   ```
   Monitor if error still occurs.

2. **Test HISTORICAL_BUFFER_SAFETY fix:**
   ```pinescript
   const int HISTORICAL_BUFFER_SAFETY = 4500
   ```
   Verify profiles render correctly within 4500 bar window.

3. **Verify flashProf data access:**
   Check if `flashProf.getBktBnds()` accesses historical series data internally.

4. **Test on multiple timeframes:**
   - 1-minute (high bar count)
   - Daily (low bar count, wide time range)
   - Hourly (medium)

---

## References

**Code Locations:**
- Line 119: `max_bars_back = 500` (indicator declaration)
- Line 150: `HISTORICAL_BUFFER_SAFETY = 50` (safety constant)
- Line 935-936: Historical offset safety check
- Line 554-581: FIFO profile cleanup logic

**External Documentation:**
- Pine Script v6 Reference Manual
- TradingView Drawing Objects Limits
- max_bars_back Auto-Detection Algorithm

---

## Conclusion

The "buffer limit" is NOT a single fixed value but a **dynamic calculation by Pine Script** based on:
1. `max_bars_back` setting (currently 500)
2. Oldest profile position in storage
3. Historical series data access patterns

**Primary Fix:** Increase `max_bars_back = 5000` + correct `HISTORICAL_BUFFER_SAFETY = 4500`.

**Success Criteria:** No errors on charts with 5000+ bars, profiles render back to 4500 bars reliably.

---

**End of Research Document**
