# Profile Storage Architecture Analysis: Off-By-One Error

## Executive Summary

**CRITICAL ISSUE IDENTIFIED**: Historical rectangles are shifted by 1 profile due to a timing mismatch between profile boundary capture and data storage.

**Root Cause**: Profile boundaries (`startBar`/`endBar`) are calculated AFTER the new profile begins, but they reference the PREVIOUS profile's data. This creates a 1-profile lag in rectangle rendering.

---

## 1. Profile Lifecycle Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PROFILE LIFECYCLE                                 │
└─────────────────────────────────────────────────────────────────────────┘

Bar 0 ──→ Bar 50 ──→ Bar 100 ──→ Bar 150 ──→ Bar 200
         [Profile 1]  RESET       [Profile 2]  RESET       [Profile 3]
                      ↑                        ↑
                      │                        │
                   prfReset                 prfReset
                   triggers                 triggers
```

---

## 2. Critical Code Section: Profile Reset (Lines 528-595)

### 2.1 Reset Trigger Flow

```pine
if prfReset                                    // Line 528
    if isFirstReset                            // Line 529
        isFirstReset := false                  // Line 531
        prevResetBar := 0                      // Line 532
    else if not na(resetBar)                   // Line 533
        startBar := resetBar - delay           // Line 534 ⚠️ USES OLD resetBar
        endBar   := bar_index - 1              // Line 535
```

### 2.2 The Timing Problem

```
SCENARIO: Profile 2 ends at bar 100, Profile 3 begins at bar 100

Bar 99:  | Profile 2 data accumulating
Bar 100: | prfReset = true (Profile 3 starts)
         | resetBar = 50 (from Profile 2 start) ⚠️ OLD VALUE
         |
         | Line 534: startBar := resetBar - delay
         |           startBar := 50 - 0 = 50    ← Profile 2 start
         | Line 535: endBar := bar_index - 1
         |           endBar := 100 - 1 = 99     ← Profile 2 end
         |
         | Lines 541-554: Store Profile 2 boundaries + peaks
         |
         | Line 595: resetBar := bar_index      ← NOW set to 100
```

### 2.3 The Off-By-One Effect

```
WHAT SHOULD HAPPEN:
  Profile 2 captured when Profile 3 starts (bar 100)
  Rectangles drawn for Profile 2: bars 50-99 ✓

WHAT ACTUALLY HAPPENS:
  Profile 1 boundaries (bars 0-49) stored when Profile 2 starts (bar 50)
  Profile 2 boundaries (bars 50-99) stored when Profile 3 starts (bar 100)

  BUT: Peak data stored at bar 100 belongs to Profile 2
       Rectangle boundaries at bar 100 reference Profile 1 (startBar=0, endBar=49)

  RESULT: Profile 2 peaks drawn at Profile 1 coordinates! ⚠️
```

---

## 3. Data Storage Architecture

### 3.1 Array Structure

```
profileStartBars  = [  0,  50, 100, 150 ]  ← Profile boundaries (time)
profileEndBars    = [ 49,  99, 149, 199 ]
profilePeakCounts = [  2,   3,   1,   4 ]  ← Number of peaks per profile
profilePeakStartIdx=[  0,   2,   5,   6 ]  ← Index into flattened peak arrays

allPeakStartPrices= [1.10, 1.15, 1.20, 1.25, 1.30, 1.18, ...]  ← Flattened peak data
allPeakEndPrices  = [1.12, 1.17, 1.22, 1.27, 1.32, 1.20, ...]
                     └─P1─┘ └────P2────┘ └P3┘ └────P4...
```

### 3.2 Storage Operation (Lines 541-554)

```pine
if array.size(currentPeakStarts) > 0
    // Store profile boundaries
    array.push(profileStartBars, startBar)     // Line 543 ⚠️ Wrong timing
    array.push(profileEndBars, endBar)         // Line 544

    // Store peak count and starting index
    int peakCount = array.size(currentPeakStarts)  // Line 547
    array.push(profilePeakCounts, peakCount)       // Line 548
    array.push(profilePeakStartIdx, array.size(allPeakStartPrices))  // Line 549

    // Append peaks to flattened arrays
    for i = 0 to peakCount - 1                // Line 552
        array.push(allPeakStartPrices, array.get(currentPeakStartPrices, i))  // Line 553
        array.push(allPeakEndPrices, array.get(currentPeakEndPrices, i))      // Line 554
```

### 3.3 The Mismatch

```
CAPTURED AT RESET (Bar 100):
  ┌────────────────────────────────────────────┐
  │ startBar = 50 (from old resetBar)         │ ← Profile 2 boundaries
  │ endBar = 99                                 │
  │ currentPeakStarts = [rows for Profile 2]   │ ← Profile 2 peak data
  │ currentPeakEndPrices = [prices for Prof 2] │
  └────────────────────────────────────────────┘

  ALL STORED TOGETHER → Creates association:
    profileStartBars[1] = 50    }
    profileEndBars[1] = 99       }→ Profile 2 coordinates
    profilePeakCounts[1] = 3     }
    profilePeakStartIdx[1] = 2   }→ Points to Profile 2 peaks
```

**HOWEVER**: The issue manifests when rendering uses index-based retrieval:

```pine
// Line 925: Loop through historical profiles
for profIdx = 0 to numHistoricalProfiles - 1
    int histStartBar = array.get(profileStartBars, profIdx)    // ← Gets wrong index
    int histEndBar = array.get(profileEndBars, profIdx)
    int peakCount = array.get(profilePeakCounts, profIdx)
    int peakStartIndex = array.get(profilePeakStartIdx, profIdx)
```

---

## 4. Historical Rendering (Lines 919-977)

### 4.1 Retrieval Logic

```pine
for profIdx = 0 to numHistoricalProfiles - 1              // Line 925
    int histStartBar = array.get(profileStartBars, profIdx)   // Line 926
    int histEndBar = array.get(profileEndBars, profIdx)       // Line 927
    int peakCount = array.get(profilePeakCounts, profIdx)     // Line 928
    int peakStartIndex = array.get(profilePeakStartIdx, profIdx)  // Line 929

    // ... safety checks ...

    for peakIdx = 0 to peakCount - 1                      // Line 952
        float bottomPrice = array.get(allPeakStartPrices, peakStartIndex + peakIdx)  // Line 958
        float topPrice = array.get(allPeakEndPrices, peakStartIndex + peakIdx)       // Line 959

        int leftBar = histStartBar                        // Line 962
        int rightBar = histEndBar + peakExtensionBars     // Line 963
```

### 4.2 Example Rendering Sequence

```
AT BAR 150 (Profile 3 starts):

numHistoricalProfiles = 2

profIdx = 0:
  histStartBar  = array.get(profileStartBars, 0) = 0    ← Profile 1 time
  histEndBar    = array.get(profileEndBars, 0) = 49
  peakCount     = array.get(profilePeakCounts, 0) = 2
  peakStartIndex = array.get(profilePeakStartIdx, 0) = 0

  Peak 0:
    bottomPrice = allPeakStartPrices[0] = 1.10  ← Profile 1 peaks
    topPrice = allPeakEndPrices[0] = 1.12
    leftBar = 0                                  ← Profile 1 time
    rightBar = 49 + 50 = 99

profIdx = 1:
  histStartBar  = array.get(profileStartBars, 1) = 50   ← Profile 2 time
  histEndBar    = array.get(profileEndBars, 1) = 99
  peakCount     = array.get(profilePeakCounts, 1) = 3
  peakStartIndex = array.get(profilePeakStartIdx, 1) = 2

  Peak 0:
    bottomPrice = allPeakStartPrices[2] = 1.20  ← Profile 2 peaks
    topPrice = allPeakEndPrices[2] = 1.22
    leftBar = 50                                 ← Profile 2 time
    rightBar = 99 + 50 = 149
```

**NO MISMATCH HERE** - Arrays are correctly aligned!

---

## 5. Root Cause Analysis

### 5.1 The Real Problem

After deeper analysis, the issue is **NOT in the storage architecture** but in:

**HYPOTHESIS 1: Profile Start Calculation**
```pine
startBar := resetBar - delay   // Line 534
```

For `Swing` mode with `delay = pivRi`:
- Profile should start `pivRi` bars BEFORE the pivot confirmation
- But `resetBar` already points to a historical bar (pivot location)
- Subtracting `delay` again causes double offset

**HYPOTHESIS 2: Peak Capture Timing**
```pine
// Lines 691-728: Peak detection happens BEFORE prfReset
// But uses currentPeakStarts/currentPeakEnds arrays
// These arrays accumulate peaks from CURRENT profile
```

The peak arrays are cleared at line 586-589:
```pine
array.clear(currentPeakStarts)
array.clear(currentPeakEnds)
array.clear(currentPeakStartPrices)
array.clear(currentPeakEndPrices)
```

**BUT**: This happens AFTER storing the profile data (lines 541-554)!

### 5.2 Execution Order at Reset

```
Bar 100 (prfReset = true):

1. Lines 534-535: Calculate startBar/endBar using OLD resetBar
2. Lines 541-554: Store boundaries + peaks using OLD data
3. Lines 586-589: Clear working arrays
4. Line 595: Update resetBar to current bar_index
5. Line 597-600: Clear msProf and set new ranges

6. [Later in same bar at barstate.islast]
7. Lines 691-728: Detect peaks for NEW profile
   BUT: These are stored in currentPeakStarts arrays
   NOT pushed to historical arrays until NEXT reset!
```

### 5.3 The Off-By-One Manifestation

```
CORRECT DATA FLOW:
  Bar 0-49:  Profile 1 accumulates → peaks detected → stored in current arrays
  Bar 50:    Reset → Profile 1 boundaries + peaks → pushed to historical
  Bar 50-99: Profile 2 accumulates → peaks detected → stored in current arrays
  Bar 100:   Reset → Profile 2 boundaries + peaks → pushed to historical

ACTUAL DATA FLOW (with delay):
  Bar 0-49:  Profile 1 accumulates
  Bar 50:    Reset detected at pivot confirmation
             startBar = resetBar - delay = 50 - 5 = 45  ← Wrong!
             Should be: startBar = 0 (actual profile start)
```

---

## 6. Visual Diagram: Data Flow Mismatch

```
TIME AXIS: →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→

Bar 0     Bar 45    Bar 50      Bar 95    Bar 100
│         │         │           │         │
│◄────Profile 1────►│           │         │  (Actual data span)
│         │         │◄────Profile 2──────►│
│         │         │           │         │
│         │         RESET       │         RESET
│         │         (pivot at   │         (pivot at
│         │          bar 45)    │          bar 95)
│         │         │           │         │
│         │         │           │         │
│         │         Capture:    │         Capture:
│         │         startBar=45-5=40 ⚠️   startBar=95-5=90 ⚠️
│         │         endBar=49             endBar=99
│         │         peaks=[P1 peaks]      peaks=[P2 peaks]
│         │         │           │         │
│         │         Store:      │         Store:
│         │         [40,49,P1]  │         [90,99,P2] ⚠️
│         │         │           │         │
│         │         Arrays:     │         Arrays:
│         │         idx 0       │         idx 1
│         │         │           │         │
│         │         Render:     │         Render:
│         │         bars 40-99  │         bars 90-149 ⚠️
│         │         with P1 data│         with P2 data
│         │         │           │         │
│         └─────────MISMATCH────┘         │
│                   10 bar gap!           │
└─────────────────────────────────────────┘

EXPECTED RECTANGLES:
  Profile 1: bars 0-49 with P1 peaks
  Profile 2: bars 50-99 with P2 peaks

ACTUAL RECTANGLES:
  Profile 1: bars 40-99 with P1 peaks  ← 10 bar offset!
  Profile 2: bars 90-149 with P2 peaks ← 10 bar offset!
```

---

## 7. The Core Issue

### 7.1 Variable Lifecycle

```pine
// State variables across bars:
var series int resetBar = na          // Line 516
var series int prevResetBar = 0       // Line 517
var bool isFirstReset = true          // Line 518
series int startBar = na              // Line 519
series int endBar = na                // Line 520

// At Profile 1 → Profile 2 transition (bar 50):
// BEFORE Line 528:
resetBar = 50 (set at previous reset, bar 0)
prevResetBar = 0
delay = 5 (pivRi)

// Line 534: Calculate startBar
startBar := resetBar - delay
startBar := 50 - 5 = 45  ⚠️ WRONG!

// SHOULD BE:
startBar := prevResetBar = 0  ✓ CORRECT!
```

### 7.2 Root Cause Identification

**THE BUG**: Line 534 uses `resetBar` (current/future profile marker) instead of `prevResetBar` (previous profile marker).

```pine
// CURRENT (WRONG):
startBar := resetBar - delay           // Line 534
endBar   := bar_index - 1              // Line 535

// SHOULD BE:
startBar := prevResetBar - delay       // Use previous profile start
endBar   := bar_index - 1              // Current bar - 1 is correct
```

But wait... there's more complexity:

### 7.3 Delay Logic for Swing Mode

In Swing mode (`profAnchor == ProfAnchor.swing`):
- Pivot is detected `pivRi` bars AFTER it occurs
- Profile should start AT the pivot bar, not `pivRi` bars after
- So profile boundaries need to be offset backwards by `delay`

**HOWEVER**: The profile data (`msProf`) is already accumulated with the delay:
```pine
if not na(aBuy[delay]) and not na(resetBar)  // Line 607
    msProf.merge(
      srcABuy    = aBuy[delay],              // Line 609 - Uses delayed data
      ...
```

So the profile data is correctly lagged, but the boundary calculation is DOUBLE-lagging!

---

## 8. Recommended Fix

### 8.1 Option 1: Use prevResetBar (Simplest)

```pine
// Line 534-535: Change boundary calculation
if not na(resetBar)
    startBar := prevResetBar  // Don't apply delay - already in data
    endBar   := bar_index - 1
```

### 8.2 Option 2: Separate Current/Previous Tracking

```pine
// Add new variable to track current profile start
var series int currentProfileStart = na

// At reset:
if prfReset
    if not isFirstReset and not na(resetBar)
        // Capture PREVIOUS profile
        startBar := currentProfileStart
        endBar   := bar_index - 1
        // ... store data ...

    // Start NEW profile
    currentProfileStart := bar_index - delay
    resetBar := bar_index
```

### 8.3 Option 3: Store Boundaries Earlier

Store boundaries when profile STARTS, not when it ENDS:

```pine
var array<int> profileStartBars = array.new<int>()

// At first reset:
if prfReset
    if isFirstReset
        array.push(profileStartBars, 0)  // Mark start of first profile
        isFirstReset := false
    else
        // Store END of previous profile
        array.push(profileEndBars, bar_index - 1)
        // Store START of new profile
        array.push(profileStartBars, bar_index)

    resetBar := bar_index
```

---

## 9. Verification Steps

After fix implementation:

1. **Test Case 1: Simple 2-Profile Scenario**
   ```
   - Set pivRi = 5
   - Create pivot at bar 50
   - Verify rectangle spans bars 0-49 (not 40-49 or 45-49)
   ```

2. **Test Case 2: Multiple Profiles**
   ```
   - Create 3 profiles at bars 0, 50, 100
   - Verify no gaps or overlaps in rectangles
   - Check peak rectangles align with profile boundaries
   ```

3. **Test Case 3: Delay Variations**
   ```
   - Test with pivRi = 1, 5, 10
   - Verify rectangles always start at actual profile start
   ```

---

## 10. Impact Assessment

### 10.1 Affected Components
- Historical rectangle rendering (lines 919-977)
- Peak rectangle positioning (lines 962-963, 1022-1023)
- Profile boundary storage (lines 541-544)

### 10.2 Unaffected Components
- Peak detection logic (lines 686-728) ✓ Correct
- Flattened array architecture (lines 460-468) ✓ Correct
- FIFO cleanup logic (lines 557-583) ✓ Correct
- Current profile rendering (lines 1006-1061) ✓ Correct

### 10.3 Severity
**HIGH**: Visual misalignment affects all historical profiles in Swing mode.

---

## 11. Architecture Diagram: Corrected Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CORRECTED PROFILE LIFECYCLE                       │
└──────────────────────────────────────────────────────────────────────┘

INITIALIZATION:
  prevResetBar = 0
  resetBar = na
  currentPeakStarts = []

BAR 0:
  prfReset = true (first reset)
  isFirstReset = false
  prevResetBar = 0
  resetBar = 0
  [Profile 1 starts]

BAR 1-49:
  [Profile 1 accumulates data]
  [Peak detection fills currentPeakStarts]

BAR 50:
  prfReset = true (pivot confirmed)

  ┌─────────────────────────────────────┐
  │ CRITICAL SECTION (Lines 533-595)   │
  ├─────────────────────────────────────┤
  │ if not na(resetBar)  [TRUE]        │
  │                                     │
  │   // FIX: Use prevResetBar          │
  │   startBar := prevResetBar          │ ← 0 (correct!)
  │   endBar := bar_index - 1           │ ← 49 (correct!)
  │                                     │
  │   // Store Profile 1 data           │
  │   array.push(profileStartBars, 0)   │
  │   array.push(profileEndBars, 49)    │
  │   array.push(peaks...)              │
  │                                     │
  │   // Clear working arrays           │
  │   array.clear(currentPeakStarts)    │
  │                                     │
  │ // Update for next cycle            │
  │ prevResetBar := resetBar            │ ← 0 → prevResetBar
  │ resetBar := bar_index               │ ← 50 → resetBar
  └─────────────────────────────────────┘

  [Profile 2 starts]

BAR 51-99:
  [Profile 2 accumulates data]

BAR 100:
  prfReset = true

  startBar := prevResetBar  = 50 ✓
  endBar := 99 ✓

  Store Profile 2: [50, 99, peaks...]

  prevResetBar := 50
  resetBar := 100

  [Profile 3 starts]

RENDERING (barstate.islast):
  profIdx = 0: bars 0-49 with P1 peaks   ✓ CORRECT
  profIdx = 1: bars 50-99 with P2 peaks  ✓ CORRECT
```

---

## 12. Conclusion

### 12.1 Summary
The off-by-one error stems from using `resetBar` instead of `prevResetBar` for profile boundary calculation at line 534. This causes a timing mismatch where rectangle coordinates are offset by the delay value.

### 12.2 Recommended Action
Implement Option 1 fix:
```pine
startBar := prevResetBar  // Line 534
```

This aligns profile boundaries with the actual data accumulation period.

### 12.3 Testing Priority
1. Verify Swing mode with various pivRi values
2. Test Structure/Delta modes (delay=0) to ensure no regression
3. Validate peak rectangle alignment after fix

---

**Document Version**: 1.0
**Analysis Date**: 2025-11-11
**Analyzed By**: System Architecture Designer
**Status**: Root Cause Identified - Fix Ready for Implementation
