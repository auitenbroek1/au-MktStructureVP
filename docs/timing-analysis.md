# CRITICAL BUG ANALYSIS: Historical Rectangle Off-by-One Error

## Executive Summary

**ROOT CAUSE IDENTIFIED**: Historical rectangles are shifted forward by 1 profile because `startBar` calculation uses `resetBar` which points to the FIRST bar of the NEW profile, not the LAST bar of the OLD profile.

## Detailed Timing Sequence Trace

### Example Scenario (Swing Mode with pivRi=5):

```
Bar Numbers:  100  101  102  103  104  105  106  107  108  109  110  111  112
Anchor Event:                                   X (pivot detected at 110)
Profile:      [-------- Profile A ---------]    [-------- Profile B ---------]
Visual:       ^                             ^    ^
              Profile A Start               |    Profile B Start
                                    Profile A End
```

### Step-by-Step Execution:

#### **Bar 110: Anchor Detected**

**Line 476-485**: Pivot detection fires
```pinescript
series bool pivPrcHi = not na(ta.pivothigh(high, pivLe, pivRi))  // TRUE at bar 110
// This detects a pivot that occurred at bar 105 (110 - 5 = 105)

if pivPrcHi or pivPrcLo
    impulseStart := math.min(nz(impulseStart), math.max(CumO[pivRi], CumC[pivRi]))
    // Uses data from bar 105 (pivRi bars ago)

if impulseStart != impulseStart[1]
    prfReset := true  // TRIGGERS at bar 110
```

**Line 511-584**: Profile reset logic
```pinescript
var series int resetBar = na  // Currently holds bar index of PREVIOUS reset (e.g., 100)
var bool isFirstReset = true  // FALSE (not first profile)

if prfReset  // TRUE at bar 110
    if isFirstReset
        // Skip - not first
    else if not na(resetBar)  // TRUE (resetBar = 100)
        startBar := resetBar - delay  // 100 - 5 = 95 ⚠️
        endBar   := bar_index - 1 - delay  // 110 - 1 - 5 = 104 ⚠️

        // CAPTURE Profile A data with boundaries [95, 104]
        // BUT Profile A VISUALLY spans bars [100, 109]
```

**Line 584**: Update resetBar
```pinescript
resetBar := bar_index  // resetBar = 110 (FIRST bar of Profile B)
```

### The Problem Visualization:

```
EXPECTED (Visual Profile A):
  Bars:    [100, 101, 102, 103, 104, 105, 106, 107, 108, 109]
  Capture: startBar=100, endBar=109

ACTUAL (Captured Profile A):
  Bars:    [95, 96, 97, 98, 99, 100, 101, 102, 103, 104]
  Capture: startBar=95, endBar=104

SHIFT: 5 bars backward (= delay/pivRi)
```

### Historical Rectangle Rendering:

**Line 914-932**: When rendering historical profiles
```pinescript
int histStartBar = array.get(profileStartBars, profIdx)  // 95 ⚠️
int histEndBar = array.get(profileEndBars, profIdx)      // 104 ⚠️

int leftBar = histStartBar   // Rectangle starts at bar 95
int rightBar = histEndBar + peakExtensionBars  // Rectangle ends at bar 104+50=154
```

**Result**: Rectangle spans bars [95-154] instead of [100-159]

## Root Cause Analysis

### Issue 1: `resetBar` Semantic Confusion

**Line 511-584**: `resetBar` variable has dual meaning:
- **When SET** (line 584): `resetBar := bar_index` - Points to FIRST bar of NEW profile
- **When READ** (line 527): `startBar := resetBar - delay` - Treated as if it points to FIRST bar of OLD profile

### Issue 2: Off-by-One in Boundary Calculation

**Current Logic** (line 527-528):
```pinescript
startBar := resetBar - delay  // If resetBar=110, delay=5 → startBar=105
endBar   := bar_index - 1 - delay  // If bar_index=110 → endBar=104
```

**Problem**:
- `resetBar` (110) is the FIRST bar of the NEW profile
- Subtracting `delay` (5) gives 105, which is 5 bars AFTER the visual profile start (100)
- We need to go back by `delay` from the PREVIOUS `resetBar`, not the CURRENT one

### Issue 3: The `isFirstReset` Flag Was a Red Herring

The previous fix added `isFirstReset` to skip the first profile, but this doesn't address the fundamental timing error. Even with this flag, all SUBSEQUENT profiles are still shifted.

## The Correct Timing Logic

### What Should Happen:

```
Bar 100: First anchor fires
  - resetBar := 100 (start of Profile A)

Bar 110: Second anchor fires
  - CAPTURE Profile A: startBar = 100 - 5 = 95, endBar = 110 - 1 - 5 = 104
    ⚠️ WRONG! Should be: startBar = 100, endBar = 109
  - resetBar := 110 (start of Profile B)

Bar 120: Third anchor fires
  - CAPTURE Profile B: startBar = 110 - 5 = 105, endBar = 120 - 1 - 5 = 114
    ⚠️ WRONG! Should be: startBar = 110, endBar = 119
```

### The Fix:

**Option 1: Store Previous ResetBar**
```pinescript
var series int prevResetBar = na
var series int resetBar = na

if prfReset
    if not na(resetBar)  // Not first profile
        // Use CURRENT resetBar (not previous) for startBar
        startBar := resetBar
        endBar   := bar_index - 1

        // CAPTURE with correct boundaries

    prevResetBar := resetBar
    resetBar := bar_index
```

**Option 2: Use Previous Reset Directly**
```pinescript
if prfReset
    if not na(resetBar)  // Not first profile
        // resetBar already points to start of current profile
        startBar := resetBar
        endBar   := bar_index - 1

        // CAPTURE with correct boundaries (no delay subtraction needed!)
```

## Why This Bug Exists

1. **Swing Mode Complexity**: The `delay` variable was introduced to handle the retroactive nature of swing detection
2. **Conflation**: The code conflates "detection time" with "profile start time"
3. **resetBar Ambiguity**: Using `resetBar` to mean "where the new profile starts" while calculating historical boundaries as if it means "where the old profile started"

## Verification Test

To confirm this analysis, check the following in the chart:

1. Enable rectangle frame (`Show Rectangle Frame = true`)
2. Count bars manually:
   - Note the bar number where a profile visually starts (first bar after anchor)
   - Note the bar number where profile rectangle's left edge appears
   - Calculate the difference

**Expected**: Difference = `pivRi` bars (e.g., 5 bars backward shift)

## Conclusion

**The bug is NOT in the rendering logic** - rectangles are drawn correctly with the captured boundaries.

**The bug IS in the capture logic (lines 527-528)** - we're capturing the wrong time boundaries due to incorrect use of `resetBar`.

The fix requires:
1. Remove `- delay` from line 527 (startBar calculation)
2. Remove `- delay` from line 528 (endBar calculation) OR adjust to `bar_index - 1`
3. Remove the `isFirstReset` workaround (no longer needed)

This will align the captured boundaries with the visual profile boundaries.
