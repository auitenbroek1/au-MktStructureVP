# Rectangle Start Offset Bug - Test Scenario

## Executive Summary

**BUG**: Historical rectangles in Swing mode start 5 bars (delay) too late, creating a visual misalignment with the volume profile they're meant to frame.

**ROOT CAUSE**: Line 534 captures `resetBar` instead of the actual profile start `resetBar - delay`.

**IMPACT**: Swing mode only (Structure and Delta modes unaffected because delay = 0)

**SEVERITY**: High - Visual misalignment undermines indicator reliability

---

## Test Scenario Configuration

### Environment
- **Mode**: Swing (Profile Anchor = "Swing")
- **Pivot Settings**: Left = 10, Right = 5
- **Delay**: 5 bars (pivRi value)

### Test Timeline
```
Bar 0-49:    Previous profile data
Bar 50:      Profile reset (first anchor change)
Bar 50-99:   Profile accumulation phase
Bar 100:     Profile reset (second anchor change - CAPTURE EVENT)
Bar 100+:    New profile begins
```

---

## Detailed Bug Analysis

### Phase 1: Profile Accumulation (Bars 50-99)

**Profile Initialization at Bar 50:**
```pine
Line 595: resetBar := bar_index  // resetBar = 50
Line 478: delay := pivRi         // delay = 5 (in Swing mode)
```

**Current Profile Rendering:**
```pine
Line 648: startBar := resetBar - delay  // startBar = 50 - 5 = 45
Line 647: endBar   := bar_index          // endBar = 50-99 (developing)
```

**Profile draws from bar 45 to bar 99** ✓ (This is CORRECT)

### Phase 2: Profile Capture at Bar 100 (THE BUG OCCURS HERE)

**Capture Logic Executed:**
```pine
Line 528: if prfReset           // Triggered at bar 100
Line 534:     startBar := resetBar        // ❌ BUG: startBar = 50
Line 535:     endBar   := bar_index - 1   // ✓ CORRECT: endBar = 99
Line 543:     array.push(profileStartBars, startBar)  // ❌ Stores 50 (should be 45)
```

**What Should Have Been Captured:**
```pine
Line 534: startBar := resetBar - delay  // ✓ CORRECT: startBar = 45
```

### Phase 3: Historical Rectangle Rendering

**Rectangle Coordinates:**
```pine
Line 926: int histStartBar = array.get(profileStartBars, profIdx)  // Gets 50 ❌
Line 927: int histEndBar = array.get(profileEndBars, profIdx)      // Gets 99 ✓

Line 871-874: box.new(
    left   = histStartBar,  // 50 ❌ (should be 45)
    right  = histEndBar,    // 99 ✓
    top    = flashProf.rangeUp,
    bottom = flashProf.rangeLo)
```

**Visual Misalignment:**
```
Profile Drawing:     [===========PROFILE===========]
                      ↑                            ↑
                     Bar 45                      Bar 99

Rectangle Border:         [====RECTANGLE====]
                          ↑                 ↑
                         Bar 50           Bar 99

OFFSET: 5 bars gap ←←←←←
```

---

## Code Flow Analysis

### Current Flow (BUGGY)

```
Bar 100 Profile Reset Event:
  ↓
Line 528: if prfReset
  ↓
Line 534: startBar := resetBar (= 50) ❌ WRONG
Line 535: endBar := bar_index - 1 (= 99) ✓ CORRECT
  ↓
Line 543: array.push(profileStartBars, startBar) → Stores 50 ❌
  ↓
Later at barstate.islast:
  ↓
Line 926: histStartBar = array.get(profileStartBars, 0) → Retrieves 50 ❌
  ↓
Line 871-874: box.new(left = 50, ...) → Rectangle starts at bar 50 ❌
  ↓
RESULT: Rectangle offset by 5 bars (delay)
```

### Expected Flow (FIXED)

```
Bar 100 Profile Reset Event:
  ↓
Line 528: if prfReset
  ↓
Line 534: startBar := resetBar - delay (= 50 - 5 = 45) ✓ CORRECT
Line 535: endBar := bar_index - 1 (= 99) ✓ CORRECT
  ↓
Line 543: array.push(profileStartBars, startBar) → Stores 45 ✓
  ↓
Later at barstate.islast:
  ↓
Line 926: histStartBar = array.get(profileStartBars, 0) → Retrieves 45 ✓
  ↓
Line 871-874: box.new(left = 45, ...) → Rectangle starts at bar 45 ✓
  ↓
RESULT: Rectangle perfectly frames the profile
```

---

## Test Matrix: Bug Manifestation by Mode

| Mode      | delay | resetBar | Profile Start | Captured Start | Rectangle Start | Offset | Status |
|-----------|-------|----------|---------------|----------------|-----------------|--------|--------|
| Swing     | 5     | 50       | 45 ✓          | 50 ❌          | 50              | 5 bars | BUG    |
| Structure | 0     | 50       | 50 ✓          | 50 ✓           | 50              | 0 bars | OK     |
| Delta     | 0     | 50       | 50 ✓          | 50 ✓           | 50              | 0 bars | OK     |

### Conclusion
Bug **only affects Swing mode** where `delay = pivRi > 0`

---

## Expected vs Actual Behavior

### Expected Behavior (CORRECT)

**Profile Rendering:**
- Current profile draws: Bar 45 → Bar 99 ✓
- Historical rectangle: Bar 45 → Bar 99 ✓
- **Perfect alignment** ✓

**Code Logic:**
```pine
Line 534: startBar := resetBar - delay  // Captures actual profile start
Line 543: array.push(profileStartBars, resetBar - delay)  // Stores 45
```

### Actual Behavior (BUGGY)

**Profile Rendering:**
- Current profile draws: Bar 45 → Bar 99 ✓
- Historical rectangle: Bar 50 → Bar 99 ❌
- **5-bar offset misalignment** ❌

**Code Logic:**
```pine
Line 534: startBar := resetBar  // Captures reset point, not profile start
Line 543: array.push(profileStartBars, resetBar)  // Stores 50 (wrong)
```

---

## Visual Test Case

### Test Configuration
```pine
// Input Settings
profAnchor = ProfAnchor.swing
pivLe = 10
pivRi = 5  // delay = 5
```

### Timeline Visualization
```
Bar Index:  40   45   50   55   60   65   70   75   80   85   90   95  100
            |    |    |    |    |    |    |    |    |    |    |    |    |
Profile:         [================================================]
                 ↑ Profile Start (45)                            ↑
                 (resetBar - delay)                     Profile End (99)

Reset Bar:            ↑ Reset Point (50)
                      resetBar

Rectangle:                [====================================]
                          ↑ Rectangle Start (50) ❌ BUG        ↑
                          Should be at bar 45!        Rectangle End (99)

OFFSET:     ←←←←← 5 bars gap
```

### Expected Fix Result
```
Bar Index:  40   45   50   55   60   65   70   75   80   85   90   95  100
            |    |    |    |    |    |    |    |    |    |    |    |    |
Profile:         [================================================]
                 ↑ Profile Start (45)                            ↑
                                                        Profile End (99)

Rectangle:       [================================================]
                 ↑ Rectangle Start (45) ✓ FIXED                 ↑
                                                        Rectangle End (99)

OFFSET:     NONE - Perfect alignment ✓
```

---

## Verification Test Data

### Test Case 1: First Profile After Initialization

**Input:**
- Bar 0: Indicator starts
- Bar 50: First profile reset
- Bar 100: Second profile reset (capture event)

**Expected Captured Data:**
```
profileStartBars[0] = 45  (not 50)
profileEndBars[0] = 99
```

**Actual Captured Data (BUGGY):**
```
profileStartBars[0] = 50  ❌ BUG
profileEndBars[0] = 99
```

### Test Case 2: Multiple Profile Resets

**Input:**
- Bar 50: First reset (delay = 5)
- Bar 100: Second reset (captures first profile)
- Bar 150: Third reset (captures second profile)

**Expected Sequence:**
```
Profile 1: startBar = 45, endBar = 99
Profile 2: startBar = 95, endBar = 149
```

**Actual Sequence (BUGGY):**
```
Profile 1: startBar = 50 ❌, endBar = 99
Profile 2: startBar = 100 ❌, endBar = 149
```

---

## Fix Recommendation

### Minimal Fix (Single Line Change)

**File:** `/Users/aaronuitenbroek/hive-projects/i10iQ-projects/au-MktStructureVP/src/au-mktStructureVP-FULL.pine`

**Line 534 - BEFORE (BUGGY):**
```pine
startBar := resetBar        // ❌ Captures reset point
```

**Line 534 - AFTER (FIXED):**
```pine
startBar := resetBar - delay  // ✓ Captures actual profile start
```

### Complete Fix Context

**Lines 528-548 with Fix Applied:**
```pine
528: if prfReset
529:     if isFirstReset
530:         isFirstReset := false
531:         prevResetBar := 0
532:     else if not na(resetBar)
533:         // FIX: Capture actual profile start (accounting for delay)
534:         startBar := resetBar - delay  // ✓ FIXED
535:         endBar   := bar_index - 1
536:
537:         // ══════════════════════════════════════════════════════════════════
538:         // HISTORICAL PROFILE CAPTURE (Flattened Array Architecture)
539:         // ══════════════════════════════════════════════════════════════════
540:
541:         if array.size(currentPeakStarts) > 0
542:             // Store profile boundaries
543:             array.push(profileStartBars, startBar)  // Now stores correct value
544:             array.push(profileEndBars, endBar)
545:             // ... rest of capture logic
```

---

## Testing Validation Checklist

- [ ] Swing mode: Verify rectangle starts at `resetBar - delay`
- [ ] Swing mode: Verify no gap between profile and rectangle left edge
- [ ] Structure mode: Verify rectangle starts at `resetBar` (delay = 0)
- [ ] Delta mode: Verify rectangle starts at `resetBar` (delay = 0)
- [ ] Multi-profile: Verify all historical rectangles align correctly
- [ ] Edge case: First profile (resetBar = 0) handles correctly
- [ ] Edge case: Single-bar profiles (if possible) render correctly

---

## Related Code References

### Key Variables
- Line 477: `series int delay` - Delay value (0 for Structure/Delta, pivRi for Swing)
- Line 516: `var series int resetBar` - Current profile's reset point
- Line 519: `series int startBar` - Profile start bar (used for both current and capture)
- Line 520: `series int endBar` - Profile end bar

### Critical Sections
- Lines 478-514: Delay assignment and profile reset logic
- Lines 528-589: Historical profile capture (BUG LOCATION)
- Lines 647-648: Current profile rendering (CORRECT - uses `resetBar - delay`)
- Lines 871-891: Historical rectangle rendering (uses captured startBar)

---

## Impact Assessment

### Visual Impact
- **High**: Rectangles visibly misaligned with profiles they're meant to frame
- **User Confusion**: Undermines trust in indicator accuracy
- **Chart Clarity**: Reduces effectiveness of profile boundaries

### Technical Impact
- **Data Integrity**: No data corruption (profile calculation correct)
- **Rendering Only**: Bug affects visualization, not underlying calculations
- **Mode-Specific**: Only Swing mode affected (Structure/Delta work correctly)

### Performance Impact
- **None**: Fix adds no computational overhead
- **Single Line**: Minimal code change reduces regression risk

---

## Regression Testing

After applying fix, verify:

1. **Swing Mode Tests:**
   - Rectangle left edge aligns with profile start
   - Multiple profiles maintain alignment
   - Various pivRi values (1-20) work correctly

2. **Other Mode Tests:**
   - Structure mode still works (delay = 0 preserved)
   - Delta mode still works (delay = 0 preserved)
   - Mode switching maintains correct behavior

3. **Edge Cases:**
   - First profile after indicator load
   - Profile resets at bar 0
   - Rapid consecutive resets
   - Very short profiles (< 5 bars)

---

## Conclusion

This test scenario proves:

1. **Bug Exists**: Historical rectangles in Swing mode offset by `delay` bars
2. **Root Cause**: Line 534 captures `resetBar` instead of `resetBar - delay`
3. **Impact**: Visual misalignment in Swing mode only
4. **Fix**: Single-line change - `startBar := resetBar - delay`
5. **Testing**: Clear validation criteria and regression tests

The bug is **reproducible**, **well-documented**, and has a **straightforward fix** with **minimal risk**.
