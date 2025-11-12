# Rectangle Boundary Misplacement Analysis

## Executive Summary

The rectangle boundaries are **CORRECTLY aligned** with profile boundaries. The apparent "misplacement" is caused by the **peak extension feature** (`peakExtensionBars`), which is designed to extend rectangles beyond the profile to show future projection zones.

## Profile vs Rectangle Boundaries

### 1. HISTORICAL PROFILES (Completed Profiles)

**Profile Boundaries Stored:**
```pine
// Lines 543-544: Profile boundaries captured on anchor change
array.push(profileStartBars, startBar)  // startBar = resetBar (from line 534)
array.push(profileEndBars, endBar)      // endBar = bar_index - 1 (from line 535)
```

**Rectangle Boundaries Used:**
```pine
// Lines 962-963: Historical rectangles extend from profile start to end + extension
int leftBar = histStartBar              // Retrieved from profileStartBars[profIdx]
int rightBar = histEndBar + peakExtensionBars  // Retrieved from profileEndBars[profIdx] + extension
```

**Analysis:**
- `leftBar` = Profile start (correct)
- `rightBar` = Profile end + extension bars (intentional forward projection)
- Default extension: **50 bars** (line 393)
- Extension causes rectangles to extend beyond profile boundaries

### 2. CURRENT PROFILE (Developing Profile)

**Profile Boundaries Calculated:**
```pine
// Lines 647-648: Current profile boundaries updated every bar
endBar   := bar_index              // Current bar
startBar := resetBar - delay       // Profile start (accounting for pivot delay)
```

**Rectangle Boundaries Used:**
```pine
// Lines 1022-1023: Current rectangles use profile start to current bar + extension
int leftBar = startBar             // From line 648
int rightBar = bar_index + peakExtensionBars  // Current bar + extension
```

**Analysis:**
- `leftBar` = Profile start (correct)
- `rightBar` = Current bar + extension bars (intentional forward projection)
- Rectangles extend **50 bars into the future** by default

## Root Cause Analysis

### The Extension Is Intentional

The rectangles are **designed to extend beyond profile boundaries**:

1. **Purpose**: Show future projection zones where peaks may influence price
2. **User-Controlled**: `peakExtensionBars` input (lines 390-397)
3. **Range**: 10-200 bars (default: 50)
4. **Tooltip**: "Number of bars to extend peak rectangles forward from the profile end."

### Visual Comparison

```
PROFILE BOUNDARY:         [========Profile Data========]
RECTANGLE BOUNDARY:       [========Profile Data========]-------Extension------>
                          ^                             ^                      ^
                       startBar                       endBar            endBar+extension
```

### Why It Appears Wrong

1. **Profile visual**: Stops at `endBar`
2. **Rectangle visual**: Extends to `endBar + 50 bars`
3. **User expectation**: Rectangles should match profile boundaries exactly
4. **Design intent**: Rectangles should project peaks into the future

## Code Flow Verification

### Historical Profile Capture (Lines 528-595)

```pine
if prfReset
    if not isFirstReset
        startBar := resetBar        // Profile start
        endBar   := bar_index - 1   // Profile end (bar before reset)

        if array.size(currentPeakStarts) > 0
            // Store profile boundaries
            array.push(profileStartBars, startBar)  // ✓ Correct
            array.push(profileEndBars, endBar)      // ✓ Correct
```

### Historical Rectangle Drawing (Lines 925-977)

```pine
for profIdx = 0 to numHistoricalProfiles - 1
    int histStartBar = array.get(profileStartBars, profIdx)  // ✓ Correct retrieval
    int histEndBar = array.get(profileEndBars, profIdx)      // ✓ Correct retrieval

    // Rectangle extends from profile start to end + extension
    int leftBar = histStartBar                               // ✓ Uses profile start
    int rightBar = histEndBar + peakExtensionBars           // ✓ Adds extension
```

### Current Rectangle Drawing (Lines 1007-1050)

```pine
for peakIdx = 0 to numCurrentPeaks - 1
    // Time coordinates
    int leftBar = startBar                          // ✓ Uses profile start (line 648)
    int rightBar = bar_index + peakExtensionBars   // ✓ Adds extension to current bar
```

## Rectangle Cleanup Analysis

### Historical Rectangles (Lines 954-955)
```pine
// FIFO cleanup when approaching box limit
if array.size(allBoxes) >= MAX_BOXES - 10
    box.delete(array.shift(allBoxes))
```
- Rectangles are deleted only when box limit is reached
- No recreation/update of historical rectangles
- **Boundaries remain fixed at creation time** ✓

### Current Profile Rectangles (Lines 991-997)
```pine
// Delete previous current profile boxes on new bar
bool isNewBar = bar_index != lastCurrentProfileBar
if isNewBar and array.size(currentProfileBoxes) > 0
    for i = 0 to array.size(currentProfileBoxes) - 1
        box.delete(array.get(currentProfileBoxes, i))
    array.clear(currentProfileBoxes)
```
- Current profile rectangles are **deleted and recreated every bar**
- This ensures they use the **latest startBar/endBar values**
- **No stale boundary problem** ✓

## Conclusion

### No Code Bugs Found

1. **Historical rectangles**: Correctly use stored profile boundaries
2. **Current rectangles**: Correctly use dynamically updated boundaries
3. **Rectangle cleanup**: Properly manages stale rectangles
4. **Boundary alignment**: Perfect alignment between profiles and rectangle left edges

### The "Misplacement" Is a Feature

The rectangles intentionally extend beyond profile boundaries:
- **Left edge**: Matches profile start exactly
- **Right edge**: Extends beyond profile end by `peakExtensionBars`
- **Purpose**: Project peak zones into the future for trading analysis

### User Expectations vs Design

**If users expect exact alignment:**
```
User expectation: [====Profile====][====Rectangle====]  ← Same width
Actual behavior:  [====Profile====][====Rectangle=========>]  ← Extended
```

### Recommendation

If exact alignment is desired, users should:
1. Set `peakExtensionBars = 0` (requires changing minval in input)
2. Or the code could be modified to make extension optional

## Technical Verification

✓ Profile boundaries correctly captured and stored
✓ Rectangle boundaries correctly retrieved and applied
✓ No stale boundary values
✓ Extension is intentional and user-configurable
✓ Cleanup logic prevents orphaned rectangles
✓ Current profile updates dynamically every bar

**Status**: Working as designed. Extension feature causes the perceived misalignment.
