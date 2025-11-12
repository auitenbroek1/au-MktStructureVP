# Rectangle Timing Analysis - Profile State Machine

## Executive Summary

**ROOT CAUSE IDENTIFIED**: Rectangles are shifted forward by 1 profile because of a **CAPTURE TIMING MISMATCH**:
- Historical capture occurs AFTER `resetBar` is reassigned (line 584)
- This causes `startBar = resetBar - delay` to use the NEW resetBar instead of OLD resetBar
- Result: Rectangles use indices from profile N+1 instead of profile N

---

## State Machine Analysis

### Profile Lifecycle States

```
STATE 1: PROFILE ACCUMULATION (resetBar → prfReset)
├─ resetBar assigned (line 584)
├─ Profile data accumulates in msProf
└─ Developing lines plotted with offset

STATE 2: ANCHOR DETECTION (prfReset = true)
├─ Detected at lines 485/497/509
├─ isFirstReset check (line 523)
└─ Triggers transition to STATE 3

STATE 3: HISTORICAL CAPTURE (lines 526-583)
├─ Calculate startBar/endBar boundaries
├─ Store peak data to flattened arrays
├─ FIFO cleanup if exceeding limits
└─ Clear working arrays

STATE 4: RESET FOR NEW PROFILE (line 584)
├─ resetBar := bar_index  ← CRITICAL LINE
├─ msProf.clear()
└─ msProf.setRanges()
```

### The Critical Timing Bug

**Lines 522-584 order of operations:**
```pine
if prfReset
    if isFirstReset
        // Skip historical capture
        isFirstReset := false
    else if not na(resetBar)
        // ═══════════════════════════════════════════════════════
        // BUG: These calculations happen BEFORE resetBar update
        // ═══════════════════════════════════════════════════════
        startBar := resetBar - delay         // Line 527 - Uses OLD resetBar
        endBar   := bar_index - 1 - delay    // Line 528

        // Store historical data (lines 534-582)
        // ...peak capture logic...

    // ═══════════════════════════════════════════════════════════
    // PROBLEM: resetBar reassignment happens AFTER capture
    // ═══════════════════════════════════════════════════════════
    resetBar := bar_index                    // Line 584

    msProf.clear()                           // Lines 586-589
    msProf.setRanges(...)
```

**Visual Timeline of What Should Happen:**

```
Profile N Lifecycle:
═══════════════════════════════════════════════════════════════════════

Bar:        100          110          120          130 (prfReset)
            │            │            │            │
Profile N:  ├────────────┼────────────┼────────────┤ (should end at 129)
            │            │            │            │
resetBar:   100          100          100          100 (OLD value)

At bar 130 (when prfReset = true):
  CORRECT SEQUENCE:
    1. startBar = 100 - delay (OLD resetBar)
    2. endBar = 129 - delay   (bar_index - 1)
    3. STORE rectangle for Profile N at [startBar, endBar]
    4. resetBar = 130          (NEW value for Profile N+1)

Profile N+1 starts at bar 130 with resetBar = 130
```

**What Actually Happens:**

```
CURRENT BUGGY SEQUENCE:
═══════════════════════════════════════════════════════════════════════

At bar 130 (when prfReset = true):
    1. startBar = resetBar - delay        (uses OLD resetBar = 100) ✓
    2. endBar = bar_index - 1 - delay     (= 129 - delay) ✓
    3. STORE peaks to flattened arrays    ✓
    4. resetBar = 130                     (reassignment happens)
    5. Next profile capture will see resetBar = 130

At bar 140 (when next prfReset = true):
    1. startBar = resetBar - delay        (uses resetBar = 130) ✗ WRONG!
       Should be 130, but Profile N+1 actually started at 130
    2. endBar = 139 - delay
    3. Rectangle drawn at [130-delay, 139-delay] for Profile N+1

The problem: Profile N+1's startBar calculation uses its own resetBar,
which was assigned at the START of Profile N+1, not the END of Profile N.
```

---

## The Fundamental Issue

### Current Logic Flow (BROKEN)

```
prfReset = true (bar 130)
│
├─ isFirstReset check
│
├─ Calculate bounds for Profile N:
│  ├─ startBar = resetBar - delay     ← Uses resetBar from Profile N-1!
│  └─ endBar = bar_index - 1 - delay  ← Correct
│
├─ Store peaks for Profile N
│
└─ Update resetBar = bar_index        ← Too late! Already captured Profile N
```

### Root Cause Analysis

**The issue is NOT in rendering (lines 900-930)** - rendering correctly uses stored indices.

**The issue IS in capture timing (lines 527-528):**
```pine
startBar := resetBar - delay              // Line 527
```

At the moment of capture:
- `resetBar` still holds the value from when **THIS** profile started
- But we're trying to capture the **PREVIOUS** profile's boundaries
- This causes startBar to point to the current profile's start, not the previous one

**Why this creates the 1-profile forward shift:**

```
Profile 0: bars 0-50    (resetBar = 0)
Profile 1: bars 50-100  (resetBar = 50)
Profile 2: bars 100-150 (resetBar = 100)

When prfReset = true at bar 100:
  - We want to capture Profile 1: [50-delay, 99-delay]
  - But startBar = resetBar - delay = 100 - delay
  - This is Profile 2's start, not Profile 1's!
  - Result: Rectangle drawn at [100-delay, 99-delay] ✗ INVALID RANGE
```

---

## The Solution

### Required Fix: Track Previous resetBar

We need to store the PREVIOUS profile's resetBar before updating it:

**BEFORE (BROKEN):**
```pine
if prfReset
    if isFirstReset
        isFirstReset := false
    else if not na(resetBar)
        startBar := resetBar - delay              // ✗ Uses current profile's resetBar
        endBar   := bar_index - 1 - delay

        // ...store peaks...

    resetBar := bar_index                         // Too late
```

**AFTER (FIXED):**
```pine
var series int prevResetBar = na                  // NEW: Track previous profile start

if prfReset
    if isFirstReset
        isFirstReset := false
    else if not na(resetBar)
        // Use PREVIOUS profile's resetBar for historical capture
        startBar := prevResetBar - delay          // ✓ Uses Profile N-1's resetBar
        endBar   := bar_index - 1 - delay

        // ...store peaks...

    // Store current resetBar before updating
    prevResetBar := resetBar                      // Save Profile N's resetBar
    resetBar := bar_index                         // Update for Profile N+1
```

### Timing Diagram (FIXED)

```
Bar:        0            50           100          150
            │            │            │            │
Profile 0:  ├────────────┤            │            │
Profile 1:               ├────────────┤            │
Profile 2:                            ├────────────┤
            │            │            │            │
resetBar:   0            50           100          150
prevResetBar: na         0            50           100

At bar 50 (Profile 0 ends, Profile 1 starts):
  - prevResetBar = 0 (Profile 0's start)
  - startBar = 0 - delay      ✓ Correct for Profile 0
  - endBar = 49 - delay       ✓ Correct for Profile 0
  - Store rectangle [startBar, endBar] for Profile 0
  - resetBar = 50             ✓ Profile 1 starts

At bar 100 (Profile 1 ends, Profile 2 starts):
  - prevResetBar = 50 (Profile 1's start)
  - startBar = 50 - delay     ✓ Correct for Profile 1
  - endBar = 99 - delay       ✓ Correct for Profile 1
  - Store rectangle [startBar, endBar] for Profile 1
  - resetBar = 100            ✓ Profile 2 starts
```

---

## Implementation Details

### Change Summary

1. **Add variable** (after line 511):
   ```pine
   var series int prevResetBar = na
   ```

2. **Update capture logic** (line 527):
   ```pine
   startBar := prevResetBar - delay  // Was: resetBar - delay
   ```

3. **Store previous before update** (before line 584):
   ```pine
   prevResetBar := resetBar
   resetBar := bar_index
   ```

### Edge Cases Handled

1. **First profile**: `prevResetBar = na` skips historical capture (isFirstReset)
2. **Swing mode delay**: `delay = pivRi` correctly offsets boundaries
3. **Structure/Delta mode**: `delay = 0` uses real-time indices
4. **FIFO cleanup**: No impact - indices remain correct relative to bar_index

---

## Verification Strategy

### Test Scenarios

1. **Basic sequence** (3+ profiles):
   - Verify rectangle N aligns with profile N data
   - Check startBar/endBar match visual profile boundaries
   - Confirm no overlap or gaps

2. **Swing mode** (delay = pivRi):
   - Verify rectangles are retroactively positioned
   - Check alignment with lagged developing lines

3. **Structure/Delta mode** (delay = 0):
   - Verify real-time positioning
   - Confirm no lag in rectangle appearance

### Expected Outcome

**BEFORE FIX:**
```
Profile 0 data → Rectangle at Profile 1 position ✗
Profile 1 data → Rectangle at Profile 2 position ✗
Profile 2 data → Rectangle at Profile 3 position ✗
```

**AFTER FIX:**
```
Profile 0 data → Rectangle at Profile 0 position ✓
Profile 1 data → Rectangle at Profile 1 position ✓
Profile 2 data → Rectangle at Profile 2 position ✓
```

---

## Architecture Decision

**Why prevResetBar instead of modifying resetBar assignment order?**

1. **Clarity**: Explicit variable name shows intent (previous vs current)
2. **Safety**: Preserves resetBar's role as "current profile start"
3. **Minimal impact**: Only affects historical capture logic
4. **Debugging**: Easy to inspect prevResetBar vs resetBar values

**Alternative approaches considered:**

1. ❌ **Move resetBar assignment before capture**: Would break profile accumulation
2. ❌ **Calculate offset dynamically**: More complex, harder to verify
3. ✓ **Add prevResetBar variable**: Clean, explicit, minimal changes

---

## Conclusion

The rectangle shift is caused by a **state machine timing bug** where historical capture uses the NEW profile's resetBar instead of the PREVIOUS profile's resetBar. The fix is a 3-line change:

1. Add `var series int prevResetBar = na`
2. Change `startBar := resetBar - delay` to `startBar := prevResetBar - delay`
3. Add `prevResetBar := resetBar` before `resetBar := bar_index`

This preserves the profile start bar from Profile N for use when capturing Profile N's rectangle at the start of Profile N+1.
