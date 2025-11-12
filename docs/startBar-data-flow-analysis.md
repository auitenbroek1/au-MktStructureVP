# Critical Data Flow Analysis: `startBar` Variable

## Executive Summary

**Root Cause**: Line 534 change from `startBar := resetBar - delay` to `startBar := prevResetBar` broke volume profile rendering by creating a **temporal misalignment** between historical profile capture and current profile rendering.

**Impact**: Volume profiles fail to render rectangles because the profile loop at line 667 receives an incorrect start boundary.

---

## Variable Declaration & Scope

### Line 519: Variable Declaration
```pine
series int startBar = na
```
- **Type**: `series int` (value changes bar-to-bar)
- **Initial State**: `na` (undefined)
- **Scope**: Global within indicator
- **Mutability**: Reassigned at lines 534, 648

---

## Execution Timeline & Data Flow

### Phase 1: Profile Reset Event (`prfReset = true`)

```
TIME: Bar where anchor event is confirmed
CONDITION: prfReset = true (lines 490, 502, 514)
```

**Execution Sequence:**

1. **Line 528-533: First Reset Handling**
   ```pine
   if prfReset
       if isFirstReset
           isFirstReset := false
           prevResetBar := 0  // Initialize for FIRST profile
   ```
   - **Purpose**: Bootstrap first profile's start boundary
   - **Sets**: `prevResetBar = 0` (start at bar 0)
   - **Does NOT execute line 534** on first reset

2. **Line 533-535: Subsequent Reset Handling (THE BREAKING CHANGE)**
   ```pine
   else if not na(resetBar)
       startBar := prevResetBar      // ⚠️ CHANGED FROM: resetBar - delay
       endBar   := bar_index - 1
   ```
   - **BEFORE (Working)**: `startBar := resetBar - delay`
     - Used the **current profile's reset bar** minus delay
     - Created **temporal alignment** with profile data
   - **AFTER (Broken)**: `startBar := prevResetBar`
     - Uses **previous profile's reset bar**
     - Creates **temporal misalignment** - off by one profile period
   - **Executes**: Only when capturing PREVIOUS profile
   - **Timing**: Runs during NEW reset, captures OLD profile

3. **Line 543: Historical Storage**
   ```pine
   array.push(profileStartBars, startBar)
   ```
   - **Stores**: The INCORRECT `startBar` value from line 534
   - **Used Later**: By historical rendering at line 926

4. **Line 591-595: State Update**
   ```pine
   if not na(resetBar)
       prevResetBar := resetBar  // Save current as "previous" for next cycle
   resetBar := bar_index          // Update to new reset point
   ```
   - **Critical State Transition**: Current reset becomes previous
   - **Sequence**: `prevResetBar` update happens BEFORE `resetBar` update
   - **Implication**: Next reset will use THIS bar as `prevResetBar`

### Phase 2: Real-Time Profile Development

```
TIME: Every bar AFTER reset until next reset
CONDITION: barstate.islast = true, not at reset bar
```

**Line 629-648: Live Profile Calculation**
```pine
if barstate.islast
    if resetBar == bar_index
        flashProf := msProf  // Use accumulated profile
    else
        for i = (delay - 1) to 0
            flashProf.merge(...)  // Add delayed bars

    endBar   := bar_index
    startBar := resetBar - delay  // ✅ CORRECT VALUE for current profile
```

**Key Points:**
- **Line 648**: Uses `resetBar - delay` (CORRECT formula)
- **Purpose**: Calculate CURRENT profile boundaries for rendering
- **Timing**: Executes EVERY bar in real-time
- **Does NOT Store**: This value is NOT saved to historical arrays

### Phase 3: Profile Rendering (Lines 655-831)

```
TIME: barstate.islast = true (every bar)
CONDITION: endBar >= startBar
```

**Line 667: Profile Rendering Loop**
```pine
if endBar >= startBar
    for i = startBar to endBar  // ⚠️ Uses startBar from Phase 1 OR Phase 2
```

**Critical Question: WHICH `startBar` value is used?**

**Answer**: It depends on execution path:

| Scenario | startBar Source | Value | Result |
|----------|-----------------|-------|--------|
| **During Reset Bar** | Line 534 (broken) | `prevResetBar` | ❌ WRONG - off by 1 profile |
| **After Reset Bar** | Line 648 (correct) | `resetBar - delay` | ✅ CORRECT |
| **Historical Render** | Line 926 array | Stored from line 534 | ❌ WRONG - stored bad value |

---

## The Breaking Mechanism

### Before Change (Working):
```
Profile 1: Bars 0-50
  - Reset at bar 50
  - Line 534: startBar := resetBar - delay = 50 - 5 = 45 ✅
  - Captures: Bars 45-49 (correct historical boundary)
  - Stores: array.push(profileStartBars, 45) ✅

Profile 2: Bars 50-100
  - Reset at bar 100
  - Line 648: startBar := resetBar - delay = 100 - 5 = 95 ✅
  - Renders: Bars 95-100 (correct current boundary) ✅
```

### After Change (Broken):
```
Profile 1: Bars 0-50
  - Reset at bar 50
  - Line 534: startBar := prevResetBar = 0 ✅ (works on first profile)
  - Captures: Bars 0-49 ✅
  - Stores: array.push(profileStartBars, 0) ✅
  - prevResetBar := 50 (save for next cycle)

Profile 2: Bars 50-100
  - Reset at bar 100
  - Line 534: startBar := prevResetBar = 50 ❌ WRONG!
    (Should be 95, but uses Profile 1's reset bar)
  - Captures: Bars 50-99 ❌ WRONG RANGE
  - Stores: array.push(profileStartBars, 50) ❌ BAD VALUE
  - prevResetBar := 100 (save for next cycle)

Profile 3: Bars 100-150
  - Reset at bar 150
  - Line 534: startBar := prevResetBar = 100 ❌ WRONG!
    (Should be 145, but uses Profile 2's reset bar)
  - Captures: Bars 100-149 ❌ ENTIRE PREVIOUS PROFILE DUPLICATED
```

**Cascading Failure Pattern:**
Each reset uses the ENTIRE previous profile's range instead of the DELAYED current profile's range.

---

## Architectural Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROFILE LIFECYCLE                            │
└─────────────────────────────────────────────────────────────────┘

TIME: Bar 0 ──────────► Bar 50 ──────────► Bar 100 ──────────► Bar 150
      │                  │                   │                    │
      │                  │                   │                    │
PROFILE 1              RESET               RESET                RESET
(developing)         (capture P1)        (capture P2)         (capture P3)

┌─────────────────────────────────────────────────────────────────┐
│                    VARIABLE STATE FLOW                          │
└─────────────────────────────────────────────────────────────────┘

At Bar 50 (First Reset):
  ┌──────────────────┐
  │ prfReset = true  │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ if isFirstReset                  │
  │   prevResetBar := 0              │ Initialize
  └────────┬─────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ resetBar := 50                   │ Set new reset point
  └────────┬─────────────────────────┘

At Bar 100 (Second Reset - THE BREAK):
  ┌──────────────────┐
  │ prfReset = true  │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ else if not na(resetBar)         │
  │   startBar := prevResetBar       │ ❌ USES 50, should use 95!
  │   endBar := 99                   │
  └────────┬─────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ array.push(startBars, 50)        │ ❌ STORES WRONG VALUE
  └────────┬─────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ prevResetBar := 50               │ Update state
  │ resetBar := 100                  │ Set new reset
  └──────────────────────────────────┘

During Bars 100-149 (Development):
  ┌──────────────────┐
  │ barstate.islast  │ Every bar
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ startBar := resetBar - delay     │ ✅ CORRECT: 100 - 5 = 95
  │ endBar := bar_index              │
  └────────┬─────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ for i = startBar to endBar       │ ✅ Renders 95-100 correctly
  │   flashProf.merge(...)           │
  └──────────────────────────────────┘

At Bar 926 (Historical Render):
  ┌──────────────────────────────────┐
  │ histStartBar = array.get(        │
  │   profileStartBars, profIdx)     │ ❌ Gets 50 (wrong value)
  └────────┬─────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────┐
  │ for peakIdx = 0 to peakCount     │ ❌ Wrong time coordinates
  │   leftBar = histStartBar         │    uses bar 50 instead of 95
  └──────────────────────────────────┘
```

---

## Critical Dependencies

### Line 534 Dependencies (Historical Capture)
**INPUTS:**
- `prevResetBar` (state variable from line 594)
- `resetBar` (current reset point from line 595)
- `bar_index` (current bar number)

**OUTPUTS:**
- `startBar` → stored to `profileStartBars` array (line 543)
- Used by historical rendering (line 926)

**EXECUTION CONDITION:**
- `prfReset = true` (anchor event detected)
- `not isFirstReset` (skip first profile)
- `not na(resetBar)` (reset bar exists)

### Line 648 Dependencies (Real-Time Rendering)
**INPUTS:**
- `resetBar` (last reset point)
- `delay` (pivot confirmation bars)
- `bar_index` (current bar)

**OUTPUTS:**
- `startBar` → used by profile rendering loop (line 667)

**EXECUTION CONDITION:**
- `barstate.islast = true` (on every real-time bar)

### Line 667 Dependencies (Profile Loop)
**INPUTS:**
- `startBar` (from line 534 OR line 648)
- `endBar` (from line 535 or line 647)
- `flashProf` (volume profile data)

**OUTPUTS:**
- Rendered polylines (volume profile visualization)
- Rectangle coordinates

**EXECUTION CONDITION:**
- `endBar >= startBar` (valid profile range)

---

## The Fix

### Required Change
**Line 534** must revert to original formula:
```pine
startBar := resetBar - delay  // Use CURRENT profile's delayed start
```

### Why This Works
1. **Temporal Alignment**: Uses the SAME formula as line 648
2. **Correct Historical Range**: Captures the profile's actual time window
3. **State Independence**: Doesn't rely on stale `prevResetBar` state
4. **Consistent Logic**: Both historical and real-time use identical calculation

### Impact of Fix
- Historical rendering (line 926) receives correct start boundaries
- Peak rectangles align with actual profile data
- No temporal offset between capture and render

---

## Root Cause Analysis

**Why was the change made?**
- Likely attempted to use "previous profile start" for clarity
- Misunderstood that historical capture happens DURING new reset
- Didn't account for temporal offset in profile rendering

**Why did it break?**
- `prevResetBar` represents Profile N-1's reset point
- Line 534 captures Profile N-1's data during Profile N's reset
- Using `prevResetBar` creates N-1 boundary with N-1 data ✅
- BUT Profile N-1's data spans from `prevResetBar` to `resetBar - 1`
- The DISPLAYED range should be `(resetBar - delay) to (resetBar - 1)`
- Using `prevResetBar` ignores the delay compensation

**Fundamental Issue:**
The code conflated two concepts:
1. **Profile Period**: Full time range from one reset to next (`prevResetBar` to `resetBar`)
2. **Rendered Range**: Delayed time window matching actual profile data (`resetBar - delay` to `resetBar`)

---

## Validation Tests

To verify the fix:
1. **Visual Test**: Check if profile rectangles align with volume bars
2. **Coordinate Test**: Verify `leftBar` in peak rendering matches expected `startBar`
3. **Historical Test**: Confirm old profiles render at correct positions
4. **Delay Test**: Test with different `pivRi` values (delay lengths)

---

## Conclusion

The line 534 change broke volume profiles by introducing a **systematic one-profile offset** in the historical capture mechanism. The fix is straightforward: revert to the original formula that correctly compensates for the pivot confirmation delay.

**Key Insight**: In a delayed rendering system, historical boundaries must be calculated using the SAME delay compensation as real-time boundaries, not raw state variables.
