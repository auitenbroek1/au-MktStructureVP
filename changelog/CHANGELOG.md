# Changelog

All notable changes to the Market Structure Volume Profile (au-MSVP) indicator will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.1] - 2025-01-11

### Added
- **Peak Rectangle Highlighting**: Automatic detection and visualization of high-volume zones
  - Rectangles drawn around volume peaks ≥50% of maximum volume
  - Continuous zone detection using state machine algorithm
  - User controls for colors, border width, and extension distance
  - Forward projection extends 50 bars beyond current bar (configurable 10-200)
  - Full customization: border color, fill color with transparency, border width (1-4px)

### Changed
- Simplified threshold calculation to use only 50% of max volume method
- Removed median volume calculation for improved performance
- Eliminated dropdown selector for threshold method (single optimized approach)

### Optimized
- Removed ~15 lines of unused median calculation code
- Eliminated O(n log n) array sorting operation
- Linear O(n) peak detection algorithm for efficient processing

### Documentation
- Updated README.md with peak rectangle feature description
- Added comprehensive release notes for v1.0.1
- Created technical architecture documentation for peak detection

## [1.0.0] - 2025-01-10

### Added
- **Initial Release** of Market Structure Volume Profile indicator
- **Event-Based Profile Anchoring**:
  - Swing mode: Profiles reset on impulse baseline changes
  - Structure mode: Profiles reset on market structure breaks
  - Delta mode: Profiles reset on cumulative delta structure breaks
- **Statistical Profile Engine**:
  - PDF (Probability Density Function) allocation using T4-Skew distribution
  - Dynamic volume estimator for buy/sell pressure analysis
  - Classic allocation and volume estimation modes
- **Flexible Display Modes**:
  - Up/Down volume visualization
  - Total volume display
  - Delta-based rendering
- **Key Level Plotting**:
  - POC (Point of Control)
  - VAH/VAL (Value Area High/Low)
  - VWAP (Volume-Weighted Average Price)
  - ±1 Standard Deviation bands
- **Rectangle Frame Feature**:
  - Customizable border around volume profile histograms
  - User-controlled colors, border width, and corners
- **13 Built-in Alerts**:
  - Profile reset notifications
  - POC crosses
  - Value Area boundary crosses
  - VWAP and StdDev band crosses
- **Dynamic Row Sizing**: Automatically adjusts profile resolution
- **Non-Repainting Design**: Structure/Delta modes use confirmed data
- **Performance Optimizations**:
  - Max 100 profiles displayed
  - FIFO memory management for 500 box limit
  - Efficient polyline rendering

### Dependencies
- AustrianTradingMachine/LibTmFr/1 - Timeframe utilities
- AustrianTradingMachine/LibBrSt/1 - Bar statistics
- AustrianTradingMachine/LibPvot/1 - Pivot detection
- AustrianTradingMachine/LibVPrf/1 - Volume profile calculations

### License
- Released under Mozilla Public License 2.0
- © AustrianTradingMachine

---

## Version Format

Versions follow semantic versioning: MAJOR.MINOR.PATCH

- **MAJOR**: Incompatible API changes
- **MINOR**: New functionality (backwards-compatible)
- **PATCH**: Backwards-compatible bug fixes

[1.0.1]: https://github.com/auitenbroek1/au-MktStructureVP/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/auitenbroek1/au-MktStructureVP/releases/tag/v1.0.0
