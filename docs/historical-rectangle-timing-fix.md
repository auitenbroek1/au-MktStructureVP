# Historical Rectangle Timing Fix - Architecture Design

## Problem Statement

Historical rectangles start ONE PROFILE TOO LATE (X-coordinate shifted forward in time). The rectangles should start at the bar where the ENDING profile began, but currently start at the bar where the NEW profile begins.

**Visual Evidence:** User's screenshot shows rectangles incorrectly aligned - they start at the NEXT profile's beginning instead of the CURRENT profile's beginning.

## Root Cause Analysis

### Variable Lifecycle

```pine
Line 511: var series int resetBar = na
Line 512: series int     startBar = na
Line 513: series int       endBar = na

Line 521: if prfReset
Line 522:     if not na(resetBar)
Line 523:         startBar := resetBar - delay      // ← CAPTURES OLD resetBar
Line 524:         endBar   := bar_index - 1 - delay
Line 525:
Line 532:             array.push(profileStartBars, startBar)  // ← STORES TO HISTORY
Line 533:             array.push(profileEndBars, endBar)
Line 580:     resetBar := bar_index                // ← UPDATES resetBar AFTER CAPTURE
```

### Timing Sequence

**Current Flow (INCORRECT):**
1. Bar N: `prfReset = true` (profile change detected)
2. Line 522: Check `if not na(resetBar)` → TRUE (contains OLD profile's start)
3. Line 523: **`startBar := resetBar - delay`** → Uses OLD resetBar (profile that JUST ended)
4. Line 532-533: Push `startBar`, `endBar` to history arrays
5. Line 580: **`resetBar := bar_index`** → Updates to NEW profile's start

**Key Insight:** The code actually WORKS CORRECTLY because:
- `resetBar` holds the OLD profile's start bar throughout lines 522-533
- Only AFTER historical capture (line 580) does `resetBar` update to the NEW profile

### Wait... Let me re-analyze

Looking at the flow more carefully:

```
Profile 1 Active:
├─ resetBar = 100 (Profile 1 started at bar 100)
├─ ... profile accumulates data ...
│
Bar 150: NEW PIVOT DETECTED
├─ prfReset = true
├─ Line 522: if not na(resetBar)  → TRUE (resetBar = 100)
├─ Line 523: startBar := 100 - delay  → Profile 1's START
├─ Line 524: endBar := 149 - delay    → Profile 1's END
├─ Line 532-533: STORE Profile 1 (bars 100-delay to 149-delay)
├─ Line 580: resetBar := 150          → NEW Profile 2 starts
│
Profile 2 Active:
├─ resetBar = 150
├─ ... continues ...
```

**THIS IS CORRECT!** So where's the bug?

### The Real Issue - Developing Profile Coordinates

Let me check lines 632-633 (in `barstate.islast` block):

```pine
Line 614: if barstate.islast
Line 615:     if resetBar == bar_index
...
Line 632:     endBar   := bar_index
Line 633:     startBar := resetBar - delay
```

**AH-HA! The bug is in the DISPLAY coordinates, not the HISTORICAL storage!**

The `startBar` and `endBar` used for RENDERING (lines 857, 859) are calculated in the `barstate.islast` block at lines 632-633. These use the CURRENT `resetBar` value.

### The Actual Bug

When rendering historical rectangles (Phase 1, lines 910-943):

```pine
Line 911: int histStartBar = array.get(profileStartBars, profIdx)
Line 912: int histEndBar = array.get(profileEndBars, profIdx)
...
Line 927: int leftBar = histStartBar   // ← Uses stored value
Line 928: int rightBar = histEndBar + peakExtensionBars
```

**These coordinates ARE correct!** They use the stored values from lines 532-533.

## Wait... Let me trace through a concrete example

### Scenario: Structure Anchor Mode (delay = 0)

**Timeline:**
- Bar 100: Profile 1 starts, `resetBar = 100`
- Bar 150: Structure break detected, `prfReset = true`
  - Line 522: `if not na(resetBar)` → TRUE (resetBar = 100)
  - Line 523: `startBar := 100 - 0 = 100` ✓ CORRECT
  - Line 524: `endBar := 149 - 0 = 149` ✓ CORRECT
  - Line 532-533: Store (100, 149) to history ✓ CORRECT
  - Line 580: `resetBar := 150` (NEW profile starts)

**Historical Rendering (Phase 1):**
- Line 911: `histStartBar = 100` ✓ CORRECT
- Line 912: `histEndBar = 149` ✓ CORRECT
- Line 927: Rectangle drawn from bar 100 to 149+extension ✓ CORRECT

### Scenario: Swing Anchor Mode (delay = 5)

**Timeline:**
- Bar 100: Profile 1 starts, `resetBar = 100`
- Bar 150: Pivot confirmed (5 bars after bar 145), `prfReset = true`
  - Line 522: `if not na(resetBar)` → TRUE (resetBar = 100)
  - Line 523: `startBar := 100 - 5 = 95` → **RETROACTIVE START**
  - Line 524: `endBar := 149 - 5 = 144` → **RETROACTIVE END**
  - Line 532-533: Store (95, 144) to history
  - Line 580: `resetBar := 150`

**Historical Rendering:**
- Line 911: `histStartBar = 95`
- Line 912: `histEndBar = 144`
- Rectangle drawn from bar 95 to 144+extension

## Re-examining the Bug Report

User says: **"rectangles start ONE PROFILE TOO LATE"**

If the rectangles appear to start at bar 150 instead of bar 95 (in Swing mode) or bar 100 (in Structure mode), the stored values must be WRONG.

### Hypothesis: The Bug Happens During Profile Transition

Let me check what happens BEFORE the first profile is stored...

```pine
Line 521: if prfReset
Line 522:     if not na(resetBar)  // ← ONLY processes if resetBar EXISTS
```

**BINGO! First Profile Never Gets Stored!**

When the indicator first loads:
1. Bar 0: `resetBar = na` (var initialization)
2. Bar 10: First pivot detected, `prfReset = true`
3. Line 522: `if not na(resetBar)` → **FALSE** (resetBar is still na!)
4. Lines 523-578: **SKIPPED**
5. Line 580: `resetBar := 10` (first profile "starts")
6. Bar 20: Second pivot detected, `prfReset = true`
7. Line 522: `if not na(resetBar)` → TRUE (resetBar = 10)
8. Line 523: `startBar := 10 - delay` → Second profile's coordinates stored as FIRST historical

**THIS IS THE BUG!** The first profile coordinates are never stored, so historical arrays are offset by one profile.

## The Correct Fix

### Option 1: Store First Profile Differently (RECOMMENDED)

Track whether it's the first reset and handle it specially:

```pine
var series bool firstReset = true

if prfReset
    if firstReset
        // Don't store historical - there's no previous profile
        firstReset := false
    else if not na(resetBar)
        // Normal historical storage
        startBar := resetBar - delay
        endBar   := bar_index - 1 - delay
        // ... store to history arrays ...

    resetBar := bar_index
```

### Option 2: Store Coordinates BEFORE Updating resetBar (SIMPLER)

Capture the OLD resetBar before it gets updated:

```pine
if prfReset
    series int oldResetBar = resetBar  // ← Capture OLD value

    if not na(oldResetBar)
        startBar := oldResetBar - delay
        endBar   := bar_index - 1 - delay
        // ... store to history arrays ...

    resetBar := bar_index  // ← Update to NEW value
```

### Option 3: Initialize resetBar to bar_index (CLEANEST)

Change line 511 to initialize `resetBar` properly:

```pine
var series int resetBar = bar_index  // Instead of na
```

This ensures the first profile has valid coordinates from the start.

## Recommended Solution

**Use Option 1** because:
1. Explicitly handles the first profile case
2. Avoids storing invalid/incomplete first profile
3. Clear intent - first profile has no "previous" to store
4. Prevents potential edge cases with early bar indices

## Implementation Plan

### Step 1: Add First Reset Tracking

```pine
Line 511: var series int resetBar = na
Line 512: var series bool firstReset = true  // ← ADD THIS
Line 513: series int     startBar = na
```

### Step 2: Modify Reset Logic

```pine
Line 521: if prfReset
Line 522:     if firstReset
Line 523:         // Skip historical storage - no previous profile exists
Line 524:         firstReset := false
Line 525:     else if not na(resetBar)
Line 526:         startBar := resetBar - delay
Line 527:         endBar   := bar_index - 1 - delay
Line 528:
Line 529:         // Historical profile capture
Line 530:         if array.size(currentPeakStarts) > 0
Line 531:             array.push(profileStartBars, startBar)
Line 532:             array.push(profileEndBars, endBar)
Line 533:             // ... rest of historical storage ...
Line 574:
Line 575:     resetBar := bar_index
```

### Step 3: Testing Checklist

Test scenarios:
1. First profile on chart load → No rectangle (correct)
2. Second profile → Rectangle at correct first profile coordinates
3. Structure mode (delay=0) → Rectangles at exact profile boundaries
4. Swing mode (delay=5) → Rectangles retroactively positioned 5 bars back
5. Multiple profiles → Each rectangle starts at correct historical position

## Technical Constraints

- Pine Script version: 6
- No lookback beyond initialization
- `resetBar` tracks the START of each new profile
- Historical storage happens BEFORE updating to new profile

## Success Criteria

1. Historical rectangles start at the CORRECT profile's start bar
2. No off-by-one error in X-coordinates
3. First profile handled gracefully (no invalid storage)
4. All anchor modes (Swing, Structure, Delta) render correctly

## Code Locations

- **Variable Declarations:** Lines 511-513
- **Reset Logic:** Lines 521-580
- **Historical Capture:** Lines 530-578
- **Rectangle Rendering:** Lines 910-943

## Dependencies

- Requires modification to reset logic only
- No changes to peak detection or price calculations
- Historical storage arrays remain unchanged
- Rendering logic remains unchanged
