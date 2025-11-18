# Changelog

All notable changes to the Market Structure Volume Profile (au-MSVP) indicator will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.7] - 2025-01-18

### Added
- **Peak Volume Threshold Configuration**: New user-configurable setting for peak detection sensitivity
  - Range: 10-90% of maximum volume (default: 50%)
  - Step size: 5%
  - Lower threshold = more peaks detected (increased sensitivity)
  - Higher threshold = fewer but stronger peaks (reduced sensitivity)
  - Allows traders to fine-tune peak rectangle highlighting based on:
    - Market volatility conditions
    - Trading timeframe
    - Personal analysis preferences
    - Institutional vs retail level focus

### Changed
- **Peak Detection Algorithm**: Replaced hardcoded 50% threshold with dynamic user parameter
  - Previous: `float volumeThreshold = maxVol * 0.5`
  - Updated: `float volumeThreshold = maxVol * (peakVolumeThreshold / 100.0)`
  - Added inline documentation explaining calculation method
  - Maintains backward compatibility with 50% default value

### Technical Details
- Lines 411-418: New `peakVolumeThreshold` input parameter in Rectangle Frame group
- Lines 702-709: Updated threshold calculation with explanatory comments
- Input type: `input.float()` with constraints (minval: 10.0, maxval: 90.0)
- Tooltip provides clear guidance on threshold behavior and trade-offs

### Use Cases
- **10-30%**: High sensitivity for scalping and intraday mean reversion
- **40-60%**: Balanced detection for swing trading and multi-timeframe analysis
- **70-90%**: Focus on dominant institutional levels and major support/resistance

### Notes
- Default 50% maintains identical behavior to v1.0.6 and earlier
- Threshold applies to total volume (buy + sell) regardless of Volume Display mode
- Peak detection remains tied to continuous high-volume zones (state machine logic)
- No impact on POC, Value Area, VWAP, or StdDev calculations

## [1.0.6] - 2025-01-17

### Added
- **Peak Border Toggle**: New `Show Peak Border` setting for peak rectangle border visibility
  - Allows showing peak rectangles with fill color only (no border)
  - Default: disabled (border hidden)
- **Enable Alerts Toggle**: New performance option to gate alert condition calculations
  - Disabling alerts improves performance when not using alert features
  - Default: disabled (alerts off)
- **Conditional Calculation Optimization**: Smart performance system
  - POC, VA, VWAP, and StdDev calculations only run when features are enabled
  - Saves approximately 320 operations per bar when developing indicators are hidden
  - Automatic dependency detection (e.g., VWAP needed for StdDev bands)

### Changed
- **Updated 14 Default Values** to optimized user preferences:
  - Profile Anchor: `swing` → `structure`
  - Row Number: `24` → `25`
  - Volume Estimator: `dynamic` → `classic`
  - Pivot Right Bars: `5` → `2`
  - Pivot Price Tolerance: `0.05` → `0.02`
  - Volume Display: `upDn` → `total`
  - Show Developing Value Area: `true` → `false`
  - Show Developing POC: `true` → `false`
  - Show Developing StdDev Bands: `true` → `false`
  - Show Developing VWAP: `true` → `false`
  - Rectangle Border Width: `1` → `2`
  - Show Peak Rectangles: `false` → `true`
  - Show Peak Border: `true` → `false`
  - Peak Extension Bars: `50` → `100`
- **Restructured Rendering Order**: Investigation into z-order behavior
  - Rectangle frame rendering moved before polyline rendering
  - Peak rectangle rendering positioned between frames and polylines
  - Note: Visual z-order unchanged due to Pine Script's type-based z-index hierarchy

### Performance
- **~320 operations per bar saved** when developing indicators disabled
- **Alert calculations gated** behind Enable Alerts toggle
- **Conditional library calls**: `getPoc()`, `getVA()`, `getVwap()`, `getStdDev()` only execute when needed

### Technical Details
- Lines 308-312: Enable Alerts toggle implementation
- Lines 396-400: Show Peak Border toggle implementation
- Lines 614-652: Conditional calculation logic with dependency tracking
- Lines 1178-1231: Alert conditions gated with `enableAlerts` flag
- Lines 778-1123: Z-order rendering restructure

### Notes
- Pine Script uses type-based z-index hierarchy (boxes always render above polylines)
- Z-order cannot be controlled via creation order in Pine Script v6
- Performance optimizations are backward-compatible with existing settings

## [1.0.4] - 2025-01-11

### Fixed
- Two-phase capture architecture for peak rectangle timing
- Resolved timing issues with historical peak rectangle rendering

## [1.0.3] - 2025-01-11

### Added
- Rectangle backward-shift feature for improved visual alignment
  - Historical rectangles now start at the previous profile's start position
  - Current rectangle starts at the last completed historical profile's start
  - Edge case handling: first profile and empty history fallback to original start positions

### Changed
- Modified historical rectangle positioning logic (line 962)
- Modified current rectangle positioning logic (line 1022)
- Improved rectangle-to-profile visual correlation

### Technical Details
- Implemented using Pine Script single-line ternary operators for syntax compliance
- Historical: `profIdx > 0 ? array.get(profileStartBars, profIdx - 1) : histStartBar`
- Current: `array.size(profileStartBars) > 0 ? array.get(profileStartBars, array.size(profileStartBars) - 1) : startBar`
- Zero impact on volume profile rendering (isolated to box.new() calls only)

### Notes
- This version represents incremental improvement toward exact rectangle alignment
- Further refinement planned based on user testing and feedback

## [1.0.2] - 2025-01-11

### Added
- Extended peak rectangles to historical volume profiles
- Peak rectangles now render across all past profiles (up to 50)
- Automatic capture of peak data on anchor changes
- FIFO memory management for efficient historical tracking

### Changed
- Improved peak rectangle memory management
- Optimized historical profile storage

## [1.0.1] - 2025-01-11

### Added
- Peak rectangle highlighting feature
- Rectangles automatically identify high-volume zones (≥50% of max volume)
- Fully customizable colors, border width, and extension distance
- Optimized threshold calculation for improved performance

### Changed
- Enhanced visual feedback for high-volume areas
- Improved peak detection algorithm

## [1.0.0] - 2025-01-10

### Added
- Initial release of Market Structure Volume Profile indicator
- Event-based profile anchoring (Swing/Structure/Delta modes)
- Statistical volume distribution engine with PDF models
- Dynamic buy/sell volume estimation
- Flexible display modes (Up/Down, Total, Delta)
- POC, VAH/VAL, VWAP, and StdDev calculations
- 13 integrated alerts for key level crosses
- Non-repainting structure/delta modes
- Dynamic row sizing based on price range

### Features
- High-resolution intra-bar data analysis
- T4-Skew probability density function for volume allocation
- Wick-based buy/sell pressure estimation
- Customizable visual settings
- Max 100 profiles for optimal performance
- Integration with Austrian Trading Machine Pine libraries

---

## Version References

- **[1.0.7]** - Configurable peak volume threshold (10-90%)
- **[1.0.6]** - Performance optimizations and user preference defaults
- **[1.0.4]** - Peak rectangle timing fixes
- **[1.0.3]** - Rectangle backward-shift alignment improvement
- **[1.0.2]** - Extended historical peak rectangles
- **[1.0.1]** - Added peak rectangle highlighting
- **[1.0.0]** - Initial release with core features
