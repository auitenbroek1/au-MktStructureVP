# Peak Volume Threshold Feature - Test Plan
**Version:** 1.0.6 (Proposed)
**Feature:** Configurable peak volume threshold parameter
**Current Implementation:** Fixed 50% threshold (line 699)
**Date:** 2025-11-18

---

## Executive Summary

This test plan validates the implementation of a user-configurable peak volume threshold parameter. The feature allows users to adjust the volume threshold (currently hardcoded at 50%) that determines which volume profile rows qualify as "peak" zones for rectangle highlighting.

**Current State:** Line 699 uses hardcoded `float volumeThreshold = maxVol * 0.5`
**Proposed State:** User input parameter with range 10-90%, default 50%

---

## 1. Feature Specification

### 1.1 Input Parameter Design

```pine
// Location: User Inputs section (after line 410)
int peakThresholdPct = input.int(
  title   = "Peak Volume Threshold (%)",
  group   = gR,
  defval  = 50,
  minval  = 10,
  maxval  = 90,
  step    = 5,
  tooltip = "Volume threshold for peak detection. Rows with volume >= this percentage of maximum volume will be highlighted as peaks. Lower values (10-30%) show more peaks, higher values (70-90%) show only the most significant peaks.")
```

### 1.2 Implementation Change

```pine
// Line 699 - Current implementation
float volumeThreshold = maxVol * 0.5

// Line 699 - Proposed implementation
float volumeThreshold = maxVol * (peakThresholdPct / 100.0)
```

**Critical:** Use `100.0` (not `100`) to ensure floating-point division in Pine Script v6.

---

## 2. Test Cases

### Test Case 1: Minimum Threshold (10%)
**Objective:** Verify that low threshold values capture many peak zones

**Test Data Requirements:**
- Profile with 10 rows
- Maximum volume in profile: 1000 units
- Volume distribution: Gradually declining from top (1000, 900, 800, 700, 600, 500, 400, 300, 200, 100)

**Expected Threshold Calculation:**
```
volumeThreshold = 1000 * (10 / 100.0) = 100.0
```

**Expected Peak Detection:**
- **Rows qualifying as peaks:** 9 out of 10 (all rows with volume >= 100)
- **Only excluded:** Row with 100 volume (edge case - equals threshold, may depend on >= vs > operator)
- **Rectangle count:** 1 continuous peak zone (assuming continuous high-volume zone)

**Expected Visual Outcome:**
- Most of the profile should be highlighted with peak rectangles
- Peak zones cover approximately 90% of vertical profile space

**Pass Criteria:**
- ✅ Peak rectangles extend across nearly entire vertical range
- ✅ 6-9 rows highlighted (accounting for volume distribution variance)
- ✅ No Pine Script compilation errors
- ✅ POC and Value Area calculations unchanged

---

### Test Case 2: Default Threshold (50%)
**Objective:** Verify backward compatibility with current behavior

**Test Data Requirements:**
- Profile with 10 rows
- Maximum volume: 1000 units
- Volume distribution: Exponential decay (1000, 750, 500, 400, 300, 200, 150, 100, 50, 25)

**Expected Threshold Calculation:**
```
volumeThreshold = 1000 * (50 / 100.0) = 500.0
```

**Expected Peak Detection:**
- **Rows qualifying:** 3 rows (volumes 1000, 750, 500)
- **Rectangle count:** 1 continuous peak zone (if rows are adjacent) OR 2-3 separate zones
- **Moderate filtering** - only upper portion highlighted

**Expected Visual Outcome:**
- Approximately 30-40% of vertical profile highlighted
- Clear distinction between peak and non-peak zones

**Pass Criteria:**
- ✅ Behavior matches v1.0.5 exactly when threshold = 50%
- ✅ 2-4 rows highlighted
- ✅ Peak rectangles in upper third of profile
- ✅ POC/VA calculations identical to previous version

---

### Test Case 3: High Threshold (90%)
**Objective:** Verify selective peak detection at extreme thresholds

**Test Data Requirements:**
- Profile with 10 rows
- Maximum volume: 1000 units
- Volume distribution: Sharp peak (1000, 950, 700, 500, 300, 250, 200, 150, 100, 50)

**Expected Threshold Calculation:**
```
volumeThreshold = 1000 * (90 / 100.0) = 900.0
```

**Expected Peak Detection:**
- **Rows qualifying:** 2 rows maximum (volumes 1000, 950)
- **Rectangle count:** 1 narrow peak zone
- **Very selective** - only highest volume rows

**Expected Visual Outcome:**
- Small highlighted region at absolute peak
- Approximately 10-20% of vertical profile highlighted
- Most of profile remains unhighlighted

**Pass Criteria:**
- ✅ Only 1-2 rows highlighted
- ✅ Peak rectangles only at very top of volume profile
- ✅ Clear visual distinction from lower threshold settings
- ✅ No performance degradation

---

### Test Case 4: Edge Case - All Peaks (Threshold = 10%, Uniform Volume)
**Objective:** Test boundary condition where all rows qualify

**Test Data Requirements:**
- Profile with 10 rows
- All rows have identical volume: 1000 units each (uniform distribution)

**Expected Threshold Calculation:**
```
volumeThreshold = 1000 * (10 / 100.0) = 100.0
All rows: 1000 >= 100 = TRUE
```

**Expected Peak Detection:**
- **Rows qualifying:** All 10 rows
- **Rectangle count:** 1 continuous peak zone spanning entire profile

**Expected Visual Outcome:**
- Entire profile highlighted as one large peak
- Peak rectangle from profile bottom to top

**Pass Criteria:**
- ✅ Single continuous peak rectangle
- ✅ Peak spans flashProf.rangeLo to flashProf.rangeUp
- ✅ No visual gaps in peak highlighting
- ✅ Rectangle rendering does not exceed MAX_BOXES limit

---

### Test Case 5: Edge Case - No Peaks (Threshold = 90%, Low Volume)
**Objective:** Test boundary condition where no rows qualify

**Test Data Requirements:**
- Profile with 10 rows
- Maximum volume: 100 units
- Volume distribution: All rows below threshold (100, 80, 70, 60, 50, 40, 30, 20, 10, 5)

**Expected Threshold Calculation:**
```
volumeThreshold = 100 * (90 / 100.0) = 90.0
Highest row: 100 >= 90 = TRUE (1 row qualifies)
```

**Note:** True "no peaks" scenario requires all volumes < threshold. This is unlikely in real data but theoretically possible with custom test data where max volume = 89 and threshold = 90%.

**Expected Peak Detection:**
- **Rows qualifying:** 1 row (the maximum at 100)
- **Rectangle count:** 1 small peak zone

**Expected Visual Outcome:**
- Minimal or no peak highlighting
- If 1 row qualifies: single small rectangle at peak

**Pass Criteria:**
- ✅ 0-1 peak rectangles rendered
- ✅ No runtime errors when array.size(currentPeakStarts) = 0
- ✅ Profile polylines render normally without peaks
- ✅ No impact on non-peak visual elements

---

### Test Case 6: Multiple Disconnected Peaks
**Objective:** Test peak detection with non-continuous high-volume zones

**Test Data Requirements:**
- Profile with 15 rows
- Volume distribution with 3 separate peaks:
  - Rows 0-2: (1000, 950, 900) - Peak 1
  - Rows 3-7: (300, 250, 200, 150, 100) - Valley
  - Rows 8-10: (850, 800, 750) - Peak 2
  - Rows 11-14: (200, 150, 100, 50) - Valley
- Threshold: 50% (threshold = 500)

**Expected Peak Detection:**
- **Peak 1:** Rows 0-2 (volumes 1000, 950, 900 all >= 500)
- **Peak 2:** Rows 8-10 (volumes 850, 800, 750 all >= 500)
- **Rectangle count:** 2 separate peak zones

**Expected Visual Outcome:**
- Two distinct peak rectangles with gap between them
- Peak 1 at top of profile
- Peak 2 in middle-upper section
- Valleys remain unhighlighted

**Pass Criteria:**
- ✅ Exactly 2 peak rectangles rendered
- ✅ Clear visual gap between peaks
- ✅ Peak detection logic correctly identifies discontinuous zones
- ✅ Both peaks extend correctly with peakExtensionBars

---

### Test Case 7: Threshold Boundary (Volume = Threshold Exactly)
**Objective:** Verify >= operator behavior at exact threshold match

**Test Data Requirements:**
- Profile with 5 rows
- Maximum volume: 1000 units
- Threshold: 50% (threshold = 500.0)
- Volume distribution: (1000, 750, 500, 250, 100)

**Expected Threshold Calculation:**
```
volumeThreshold = 1000 * (50 / 100.0) = 500.0
Row 2 volume: 500.0
Condition: 500.0 >= 500.0 = TRUE
```

**Expected Peak Detection:**
- **Rows qualifying:** Rows 0-2 (volumes 1000, 750, 500)
- Row with volume = 500 SHOULD qualify (using >= not >)

**Expected Visual Outcome:**
- 3 rows highlighted in continuous peak
- Row with exactly 500 volume included in peak

**Pass Criteria:**
- ✅ Row with volume exactly equal to threshold IS included
- ✅ Confirms >= operator (not >) in line 726: `bool isHighVol = rowVol >= volumeThreshold`
- ✅ 3 continuous rows highlighted

---

## 3. Syntax Validation Checklist

### 3.1 Pine Script v6 Compliance

- [ ] **Input Parameter:**
  - [ ] Type: `input.int()` (correct for percentage input)
  - [ ] Range constraints: `minval = 10, maxval = 90` (reasonable bounds)
  - [ ] Step: `step = 5` (usability - allows 10, 15, 20...90)
  - [ ] Default: `defval = 50` (backward compatible)
  - [ ] Tooltip: Clear explanation of behavior

- [ ] **Calculation:**
  - [ ] Division: `(peakThresholdPct / 100.0)` uses `100.0` NOT `100`
  - [ ] Reason: Integer division would return 0 for all values < 100
  - [ ] Float result: `50 / 100.0 = 0.5` (correct)
  - [ ] Integer error: `50 / 100 = 0` (WRONG - would break feature)

- [ ] **Type Safety:**
  - [ ] `peakThresholdPct` is `int` type
  - [ ] Division result is `float` (due to 100.0)
  - [ ] `maxVol` is `float` type
  - [ ] Final `volumeThreshold` is `float` (correct for comparison)

### 3.2 Integration Points

- [ ] **No changes required to:**
  - [ ] Peak detection loop (lines 724-755) - uses volumeThreshold variable
  - [ ] Peak rectangle rendering (lines 864-1013) - uses detected peaks
  - [ ] Historical profile storage (lines 556-595) - stores peak data
  - [ ] POC calculation (lines 636-638) - independent of peaks
  - [ ] Value Area calculation (lines 640-643) - independent of peaks

- [ ] **Variable scope:**
  - [ ] `peakThresholdPct` declared in user inputs (global scope)
  - [ ] `volumeThreshold` calculated in `barstate.islast` block (line 699)
  - [ ] Accessible to peak detection logic (line 726)

### 3.3 Compilation Test

**Command to verify syntax:**
```bash
# Pine Script editor will show compilation errors
# No command-line compiler available for Pine Script
# Must test in TradingView editor directly
```

**Expected result:** No compilation errors

---

## 4. Manual Testing Instructions

### 4.1 Test Environment Setup

1. **Platform:** TradingView (https://www.tradingview.com/)
2. **Indicator:** Market Structure Volume Profile v1.0.6 (with changes)
3. **Symbol:** SPY (high liquidity, consistent volume distribution)
4. **Timeframe:** 1H (hourly chart for clear profile visualization)
5. **Date Range:** Last 30 days of data

### 4.2 Pre-Test Configuration

```pine
// Recommended settings for testing:
Profile Anchor = "Structure"        // Consistent profile resets
Row Number = 25                     // Medium resolution
Volume = "Total"                    // Simplest visualization
Show Peak Rectangles = TRUE         // Feature under test
Show Rectangle Frame = TRUE         // Profile boundaries visible
Peak Extension Bars = 50            // Short extension for clarity
```

### 4.3 Test Execution Steps

#### Step 1: Baseline Test (50% Threshold)
1. Apply indicator to chart
2. Set **Peak Volume Threshold (%)** = 50
3. Zoom into a completed profile (5-10 bars)
4. **Observe:**
   - Count number of peak rectangles
   - Note vertical coverage of peaks (% of profile height)
   - Take screenshot: `test-50pct-baseline.png`

**Expected:** 2-4 peak rectangles, covering ~30-50% of profile height

---

#### Step 2: Low Threshold Test (10%)
1. Change **Peak Volume Threshold (%)** to 10
2. **Observe same profile:**
   - Peak rectangles should increase significantly
   - Most rows should now be highlighted
   - Take screenshot: `test-10pct-many-peaks.png`

**Expected:** 6-9 peak rectangles, covering ~70-90% of profile height

**Visual Comparison:**
```
50% threshold:  ████░░░░░░  (sparse peaks)
10% threshold:  ██████████  (dense peaks)
```

---

#### Step 3: High Threshold Test (90%)
1. Change **Peak Volume Threshold (%)** to 90
2. **Observe same profile:**
   - Peak rectangles should decrease significantly
   - Only absolute maximum volume rows highlighted
   - Take screenshot: `test-90pct-few-peaks.png`

**Expected:** 1-2 peak rectangles, covering ~10-20% of profile height

**Visual Comparison:**
```
50% threshold:  ████░░░░░░  (moderate peaks)
90% threshold:  █░░░░░░░░░  (minimal peaks)
```

---

#### Step 4: Boundary Value Test
1. Set threshold to 10% (minimum)
2. Set threshold to 90% (maximum)
3. Attempt to set threshold to 5% (below minimum)
4. Attempt to set threshold to 95% (above maximum)

**Expected:**
- ✅ 10% and 90% accepted without error
- ✅ 5% rejected or clamped to 10%
- ✅ 95% rejected or clamped to 90%

---

#### Step 5: Profile Change Test
1. Set threshold to 50%
2. Wait for new profile anchor (structure break)
3. **Observe:**
   - New profile uses current threshold setting
   - Old profiles retain their original peak rectangles
   - No visual glitches during transition

**Expected:** Smooth transition, no rectangle disappearance/reappearance

---

#### Step 6: Multiple Profile Test
1. Zoom out to show 5-10 historical profiles
2. Change threshold from 50% to 20%
3. **Observe:**
   - ALL visible profiles update immediately
   - Historical peak rectangles re-render with new threshold
   - No performance lag or visual artifacts

**Expected:** All profiles respond to threshold change in <1 second

---

#### Step 7: Edge Case - Uniform Volume
1. Find or create profile with very uniform volume distribution
2. Set threshold to 10%
3. **Observe:**
   - Nearly entire profile highlighted as one peak
   - Single continuous rectangle from top to bottom

**Expected:** 1 peak rectangle spanning >80% of profile height

---

#### Step 8: Edge Case - Extreme Skew
1. Find profile with one dominant peak (e.g., after major news event)
2. Set threshold to 90%
3. **Observe:**
   - Only single row or very narrow zone highlighted
   - Peak rectangle is visually small

**Expected:** 1 tiny peak rectangle at absolute volume maximum

---

#### Step 9: Performance Test
1. Load chart with 50+ historical profiles visible
2. Rapidly change threshold: 10% → 50% → 90% → 10%
3. **Observe:**
   - No lag or freezing
   - All profiles update smoothly
   - Chart remains responsive

**Expected:** No performance degradation, <1 second update time

---

#### Step 10: Integration Test
1. Enable **Show Developing POC** = TRUE
2. Enable **Show Developing Value Area** = TRUE
3. Change threshold to 20%
4. **Observe:**
   - POC line position unchanged
   - Value Area bands unchanged
   - Only peak rectangles affected

**Expected:** POC and VA completely independent of threshold parameter

---

### 4.4 Visual Inspection Criteria

#### What to Look For:

**✅ PASS Criteria:**
- Peak rectangle count changes inversely with threshold (lower threshold = more peaks)
- Peak rectangle vertical coverage changes with threshold
- All rectangles properly bounded by profile start/end bars
- Peak rectangles extend forward correctly (peakExtensionBars)
- No rectangles with zero height (topPrice = bottomPrice)
- No visual gaps in continuous peak zones
- POC and Value Area lines unaffected

**❌ FAIL Criteria:**
- Peak rectangles disappear entirely (unless threshold = 100% equivalent)
- Rectangles extend outside profile price range
- Rectangles start before profile start bar
- Visual artifacts (flickering, overlapping incorrectly)
- Compilation errors when changing threshold
- POC or Value Area changes when only threshold changed

---

### 4.5 Expected Behavior Summary Table

| Threshold | Typical Peak Count | Vertical Coverage | Use Case |
|-----------|-------------------|-------------------|----------|
| 10%       | 6-9 rows          | 70-90% of profile | Show all significant volume zones |
| 30%       | 4-6 rows          | 50-70% of profile | Highlight above-average volume |
| 50%       | 2-4 rows          | 30-50% of profile | Default balanced view (current behavior) |
| 70%       | 1-3 rows          | 20-40% of profile | Focus on high-volume zones |
| 90%       | 1-2 rows          | 10-20% of profile | Only show absolute peaks |

---

## 5. Automated Testing (Conceptual)

**Note:** Pine Script does not support unit testing frameworks. The following outlines a conceptual test suite for documentation purposes.

### 5.1 Unit Test: Threshold Calculation

```pine
// Conceptual test - not executable in Pine Script
TEST "threshold_calculation_50_percent"
  INPUT: peakThresholdPct = 50, maxVol = 1000.0
  EXPECTED: volumeThreshold = 500.0
  ACTUAL: volumeThreshold = maxVol * (peakThresholdPct / 100.0)
  ASSERT: ACTUAL == EXPECTED
END TEST

TEST "threshold_calculation_10_percent"
  INPUT: peakThresholdPct = 10, maxVol = 2000.0
  EXPECTED: volumeThreshold = 200.0
  ACTUAL: volumeThreshold = maxVol * (peakThresholdPct / 100.0)
  ASSERT: ACTUAL == EXPECTED
END TEST

TEST "threshold_calculation_90_percent"
  INPUT: peakThresholdPct = 90, maxVol = 500.0
  EXPECTED: volumeThreshold = 450.0
  ACTUAL: volumeThreshold = maxVol * (peakThresholdPct / 100.0)
  ASSERT: ACTUAL == EXPECTED
END TEST
```

### 5.2 Unit Test: Peak Detection Logic

```pine
// Conceptual test - not executable in Pine Script
TEST "peak_detection_above_threshold"
  INPUT: volumeArray = [1000, 800, 600, 400], threshold = 500
  EXPECTED: peaks = [{start: 0, end: 2}]  // Rows 0,1,2 qualify
  ACTUAL: Run peak detection loop (lines 724-755)
  ASSERT: array.size(currentPeakStarts) == 1
  ASSERT: array.get(currentPeakStarts, 0) == 0
  ASSERT: array.get(currentPeakEnds, 0) == 2
END TEST

TEST "peak_detection_multiple_zones"
  INPUT: volumeArray = [1000, 900, 300, 800, 700, 200], threshold = 500
  EXPECTED: peaks = [{start: 0, end: 1}, {start: 3, end: 4}]
  ACTUAL: Run peak detection loop
  ASSERT: array.size(currentPeakStarts) == 2
END TEST

TEST "peak_detection_boundary_value"
  INPUT: volumeArray = [1000, 500, 499], threshold = 500
  EXPECTED: peaks = [{start: 0, end: 1}]  // 500 should qualify with >=
  ACTUAL: Run peak detection loop
  ASSERT: array.get(currentPeakEnds, 0) == 1  // Includes row with vol=500
END TEST
```

### 5.3 Integration Test: Historical Profile Storage

```pine
// Conceptual test - not executable in Pine Script
TEST "historical_peaks_persist_after_threshold_change"
  SETUP: Create profile with peakThresholdPct = 50
  ACTION 1: Store profile to historical arrays
  ACTION 2: Change peakThresholdPct to 20
  EXPECTED: Historical peaks re-rendered with new threshold
  VERIFY: array.size(allPeakStartPrices) updated
END TEST
```

---

## 6. Regression Testing

### 6.1 Unchanged Functionality Checklist

These features MUST remain unchanged after implementing the threshold parameter:

- [ ] **Volume Profile Rendering:**
  - [ ] Polyline shapes unchanged
  - [ ] Buy/Sell volume colors unchanged
  - [ ] Profile width and positioning unchanged

- [ ] **Statistical Metrics:**
  - [ ] POC (Point of Control) calculation unchanged
  - [ ] Value Area (VA) calculation unchanged
  - [ ] VWAP calculation unchanged
  - [ ] StdDev calculation unchanged

- [ ] **Profile Anchoring:**
  - [ ] Swing anchor behavior unchanged
  - [ ] Structure anchor behavior unchanged
  - [ ] Delta anchor behavior unchanged
  - [ ] Profile reset timing unchanged

- [ ] **Rectangle Frame:**
  - [ ] Main rectangle border unchanged
  - [ ] Rectangle fill color unchanged
  - [ ] Rectangle positioning unchanged

- [ ] **Peak Rectangle Styling:**
  - [ ] Peak border color unchanged (user input)
  - [ ] Peak fill color unchanged (user input)
  - [ ] Peak extension bars unchanged
  - [ ] Peak border width unchanged

- [ ] **Performance:**
  - [ ] No new runtime errors
  - [ ] No memory leaks (box cleanup still works)
  - [ ] No increase in compilation time

### 6.2 Regression Test Scenarios

#### Scenario 1: Default Behavior Match
**Test:** Set threshold to 50% and compare with v1.0.5
**Expected:** Identical visual output to previous version

#### Scenario 2: Profile Anchor Compatibility
**Test:** Change profile anchor (Swing/Structure/Delta) with threshold at 50%
**Expected:** Profiles reset correctly, peaks render at appropriate times

#### Scenario 3: Multiple Timeframes
**Test:** Apply indicator to 5m, 15m, 1H, 4H, 1D charts with threshold at 50%
**Expected:** Peak detection works correctly at all timeframes

---

## 7. Known Limitations and Constraints

### 7.1 Technical Constraints

1. **Pine Script Box Limit:**
   - Maximum 500 boxes per indicator instance
   - FIFO deletion implemented (lines 803-804, 906-907)
   - Lower thresholds create more peaks = more boxes
   - Risk: Threshold of 10% on 50 profiles could exceed limit

2. **Historical Buffer Limit:**
   - max_bars_back = 5000 (line 119)
   - Safety check prevents "offset beyond buffer" errors (lines 884-887)
   - Very old profiles may not render if beyond buffer

3. **Profile Size Constraint:**
   - MIN_PROFILE_BARS = 5 (line 149)
   - Profiles with <5 bars don't render rectangles (lines 893-894)
   - Prevents visual clutter from tiny profiles

### 7.2 Threshold Range Rationale

**Why 10-90% (not 1-100%)?**

- **Minimum 10%:**
  - Below 10% would highlight almost entire profile
  - Defeats purpose of "peak" detection
  - Would create excessive boxes (performance issue)

- **Maximum 90%:**
  - Above 90% often results in zero peaks
  - 90-100% range has diminishing utility
  - Users wanting >90% likely want no highlighting (toggle feature off)

### 7.3 Edge Cases

1. **All Rows Qualify (Threshold Too Low):**
   - **Scenario:** Uniform volume distribution + 10% threshold
   - **Result:** Single peak covering entire profile
   - **Impact:** Acceptable - still functionally correct

2. **No Rows Qualify (Threshold Too High):**
   - **Scenario:** Skewed distribution + 90% threshold + low peak
   - **Result:** Zero peak rectangles
   - **Impact:** Acceptable - user intentionally set high threshold

3. **Threshold = 50.001% on Integer Input:**
   - **Non-issue:** Input is integer (50%), calculation uses float division
   - **Result:** Exact 50.0% threshold, no floating-point precision errors

---

## 8. Test Data Generation Guide

### 8.1 Synthetic Test Profiles

For controlled testing, create synthetic volume distributions:

#### Gradual Decline Profile
```
Row 0:  1000 volume (100% of max) ← Always peak
Row 1:   900 volume (90%)
Row 2:   800 volume (80%)
Row 3:   700 volume (70%)
Row 4:   600 volume (60%)
Row 5:   500 volume (50%)  ← Threshold boundary at 50%
Row 6:   400 volume (40%)
Row 7:   300 volume (30%)
Row 8:   200 volume (20%)
Row 9:   100 volume (10%)  ← Threshold boundary at 10%
```

**Test with:**
- 10% threshold → Rows 0-9 qualify (9 rows)
- 50% threshold → Rows 0-5 qualify (6 rows)
- 90% threshold → Rows 0-1 qualify (2 rows)

---

#### Bimodal Distribution (Two Peaks)
```
Row 0:  1000 volume ← Peak 1
Row 1:   950 volume ← Peak 1
Row 2:   200 volume ← Valley
Row 3:   150 volume ← Valley
Row 4:   800 volume ← Peak 2
Row 5:   750 volume ← Peak 2
Row 6:   100 volume ← Valley
```

**Test with 50% threshold (500):**
- Peak 1: Rows 0-1 (continuous)
- Peak 2: Rows 4-5 (continuous)
- Result: 2 separate peak rectangles

---

#### Uniform Distribution
```
All rows: 1000 volume (identical)
```

**Test with:**
- Any threshold <100% → All rows qualify
- Result: 1 continuous peak spanning entire profile

---

### 8.2 Real Market Data Testing

**Recommended symbols for natural volume patterns:**

1. **SPY (S&P 500 ETF):**
   - High liquidity
   - Consistent volume distribution
   - Good for baseline testing

2. **AAPL (Apple Inc.):**
   - Variable volume patterns
   - Clear peaks during earnings
   - Tests edge cases naturally

3. **BTC/USD (Bitcoin):**
   - Extreme volume spikes
   - Tests high-threshold behavior
   - Variable profile shapes

**Recommended timeframes:**
- **5m:** Many small profiles (tests performance)
- **1H:** Balanced profile size (recommended for screenshots)
- **4H:** Large profiles (tests peak merging logic)

---

## 9. Documentation Requirements

### 9.1 Code Comments

Add detailed comment block at line 699:

```pine
// ══════════════════════════════════════════════════════════════════
// PEAK DETECTION THRESHOLD CALCULATION (v1.0.6)
// ══════════════════════════════════════════════════════════════════
// Calculate volume threshold for identifying high-volume peak zones.
//
// User Parameter: peakThresholdPct (10-90%, default 50%)
// - Lower values (10-30%): More peaks, broad highlighting
// - Medium values (40-60%): Balanced peak detection
// - Higher values (70-90%): Only most significant peaks
//
// Formula: volumeThreshold = maxVol * (peakThresholdPct / 100.0)
// Note: Division by 100.0 (not 100) ensures float arithmetic
// ══════════════════════════════════════════════════════════════════

float volumeThreshold = maxVol * (peakThresholdPct / 100.0)
```

### 9.2 Changelog Entry

```markdown
# CHANGELOG.md

## [1.0.6] - 2025-11-18

### Added
- Configurable peak volume threshold parameter (`Peak Volume Threshold (%)`)
  - Range: 10-90% (default: 50%)
  - Location: Rectangle Frame settings group
  - Purpose: Control sensitivity of peak rectangle highlighting
  - Lower threshold → more peaks visible
  - Higher threshold → only strongest peaks visible

### Changed
- Peak detection calculation (line 699) now uses user input instead of hardcoded 50%
- Tooltip enhanced to explain threshold behavior

### Backward Compatibility
- Default value (50%) preserves v1.0.5 behavior exactly
- No changes to profile calculation, POC, or Value Area
```

### 9.3 User Guide Addition

```markdown
## Peak Rectangle Highlighting

Peak rectangles highlight continuous high-volume zones in your volume profiles.

**Peak Volume Threshold (%)** - Default: 50%

This parameter controls which volume rows qualify as "peaks":

- **10-30% (Low):** Shows many peaks - useful for identifying all significant volume zones
- **40-60% (Balanced):** Default behavior - highlights moderate to high volume areas
- **70-90% (High):** Only shows absolute peaks - useful for focusing on maximum volume

**Example:**
- With threshold = 50%, only rows with volume ≥ 50% of profile maximum are highlighted
- If max volume = 1000, threshold = 500, rows with <500 volume are not highlighted
- Lower the threshold to see more highlighted zones

**Tip:** Adjust threshold based on your trading style:
- Day traders: 30-50% to catch multiple support/resistance zones
- Swing traders: 60-80% to focus on strongest institutional levels
```

---

## 10. Test Sign-Off Checklist

### 10.1 Pre-Release Validation

Before releasing v1.0.6, verify:

- [ ] All 10 manual test cases passed
- [ ] Screenshots captured for documentation
- [ ] Syntax validation completed (no compilation errors)
- [ ] Regression tests passed (unchanged functionality verified)
- [ ] Performance testing completed (no lag with 50+ profiles)
- [ ] Edge cases tested (uniform volume, extreme thresholds)
- [ ] Backward compatibility confirmed (50% threshold = v1.0.5 behavior)

### 10.2 Code Review Checklist

- [ ] Input parameter correctly defined with constraints
- [ ] Float division used (100.0 not 100)
- [ ] No magic numbers (50% replaced with parameter)
- [ ] Tooltip provides clear user guidance
- [ ] Code comments added at calculation point
- [ ] CHANGELOG.md updated with feature description
- [ ] No impact on POC/VA calculations verified
- [ ] FIFO box cleanup still functions correctly

### 10.3 User Acceptance Criteria

- [ ] Feature is intuitive (no user confusion expected)
- [ ] Default behavior matches v1.0.5 (transparent upgrade)
- [ ] Parameter range prevents misuse (10-90% logical bounds)
- [ ] Visual feedback immediate (threshold changes apply instantly)
- [ ] No performance degradation reported

---

## 11. Test Results Log Template

### Test Execution Record

**Tester:** ________________
**Date:** __________________
**Version:** v1.0.6
**Environment:** TradingView / Symbol: ______ / Timeframe: ______

| Test Case | Threshold | Expected Peaks | Actual Peaks | Pass/Fail | Notes |
|-----------|-----------|----------------|--------------|-----------|-------|
| TC1: Min  | 10%       | 6-9 rows       |              | [ ]       |       |
| TC2: Default | 50%    | 2-4 rows       |              | [ ]       |       |
| TC3: High | 90%       | 1-2 rows       |              | [ ]       |       |
| TC4: All Peaks | 10% (uniform) | All rows |        | [ ]       |       |
| TC5: No Peaks | 90% (low vol) | 0-1 rows |        | [ ]       |       |
| TC6: Multi-Peak | 50% | 2 zones      |              | [ ]       |       |
| TC7: Boundary | 50%    | Includes 500  |              | [ ]       |       |

**Regression Tests:**
- [ ] POC unchanged
- [ ] Value Area unchanged
- [ ] Profile anchoring unchanged
- [ ] Performance acceptable

**Issues Found:**
1. _______________________________________________________________
2. _______________________________________________________________
3. _______________________________________________________________

**Overall Assessment:** [ ] Pass  [ ] Fail  [ ] Pass with Issues

**Sign-Off:** _______________________ Date: ___________________

---

## 12. Troubleshooting Guide

### Issue 1: No Peak Rectangles Visible
**Symptoms:** Peak rectangles enabled but not showing

**Diagnosis:**
1. Check `Show Peak Rectangles` = TRUE (line 369)
2. Verify profile has ≥5 bars (MIN_PROFILE_BARS check)
3. Confirm threshold not set to extreme value (try 50%)
4. Check if all volumes below threshold (unlikely but possible)

**Resolution:**
- Lower threshold to 20-30%
- Verify profile is complete (wait for anchor change)
- Check Pine Script console for errors

---

### Issue 2: Too Many Peak Rectangles (Visual Clutter)
**Symptoms:** Entire profile highlighted, no clear distinction

**Diagnosis:**
1. Threshold likely too low (<20%)
2. Volume distribution very uniform
3. Many small profiles (each creates rectangles)

**Resolution:**
- Increase threshold to 50-70%
- Consider toggling `Show Peak Rectangles` off if not needed
- Zoom into fewer profiles for clarity

---

### Issue 3: Peak Rectangles Not Updating
**Symptoms:** Threshold change doesn't affect visual output

**Diagnosis:**
1. Chart not refreshing (Pine Script caching)
2. Looking at historical profile (peaks fixed at creation time)

**Resolution:**
- Remove and re-add indicator to force recalculation
- Wait for new profile to form (current profile will use new threshold)
- F5 to refresh entire chart

---

### Issue 4: Compilation Error "Cannot call 'operator /' with arguments..."
**Symptoms:** Script fails to compile after adding threshold

**Diagnosis:**
- Used integer division `peakThresholdPct / 100` instead of float

**Resolution:**
- Change to `peakThresholdPct / 100.0` (note the .0)
- Ensures result is float type, not integer 0

---

## 13. Future Enhancements (Out of Scope)

Potential features for future versions:

1. **Adaptive Threshold:**
   - Automatically adjust threshold based on volume distribution
   - Use standard deviation to set dynamic threshold

2. **Multi-Tier Peaks:**
   - Define multiple threshold levels (e.g., 30%, 60%, 90%)
   - Different colors for different peak tiers

3. **Peak Count Limit:**
   - `Max Peak Rectangles` parameter to prevent excessive rendering
   - Prioritize largest peaks if limit exceeded

4. **Peak Statistics Display:**
   - Show peak count on chart (e.g., "3 peaks detected")
   - Display threshold value used for historical profiles

---

## 14. Conclusion

This test plan provides comprehensive validation for the peak volume threshold feature. The proposed implementation:

1. **Enhances flexibility** - Users can adjust peak sensitivity
2. **Maintains backward compatibility** - Default 50% preserves current behavior
3. **Minimal code changes** - Single line modification (line 699) + input parameter
4. **No performance impact** - Peak detection logic unchanged
5. **Intuitive UX** - Clear parameter naming and tooltip

**Recommendation:** Proceed with implementation and execute manual test cases 1-10 before release.

**Version Increment:** v1.0.5 → v1.0.6 (minor feature addition, no breaking changes)

---

**Document Version:** 1.0
**Last Updated:** 2025-11-18
**Next Review:** After v1.0.6 release
