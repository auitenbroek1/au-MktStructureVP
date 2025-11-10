# Changelog

All notable changes to the Market Structure & Volume Profile indicator will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-11-10

### Added
- Initial release of Market Structure & Volume Profile indicator
- Market structure detection with swing highs/lows
- Break of Structure (BOS) identification
- Change of Character (ChoCh) detection
- Volume Profile calculation with configurable rows
- Point of Control (POC) visualization
- Value Area High (VAH) and Value Area Low (VAL) display
- Customizable lookback periods for both market structure and volume profile
- Color customization for all visual elements
- Alert conditions for BOS and ChoCh events (bullish and bearish)
- Toggle controls for all major features
- Value Area percentage configuration (50-95%)

### Features
- **Market Structure Analysis**
  - Automatic trend detection (uptrend/downtrend)
  - Higher highs/lows and lower highs/lows identification
  - Visual markers for swing points
  - BOS and ChoCh labeling

- **Volume Profile**
  - Histogram-style volume distribution across price levels
  - Dynamic POC calculation
  - Value Area calculation (default 70%)
  - Configurable number of price rows (10-100)
  - Adjustable lookback period (20-500 bars)

- **Customization**
  - Independent toggle for each feature
  - Color customization for all elements
  - Adjustable lookback periods
  - Configurable value area percentage

### Technical Details
- Pine Script v5
- Overlay indicator (displays on price chart)
- Real-time calculation updates
- Alert-ready conditions
