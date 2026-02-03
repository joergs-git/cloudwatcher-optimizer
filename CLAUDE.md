# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AAG CloudWatcher Solo K-Factor Optimizer - a Python tool for optimizing sky temperature correction parameters for the Lunatico Astronomia AAG CloudWatcher cloud detection system used in astronomical observation. It uses differential evolution to find optimal K-factor parameters (K1-K7) that minimize seasonal temperature drift in cloud detection.

## Commands

```bash
# Install dependencies
pip install numpy scipy matplotlib

# Run interactive GUI (default mode)
python cloudwatcher_optimizer

# Run CLI mode for batch processing
python cloudwatcher_optimizer --cli
```

Note: There are no formal test, lint, or build commands. This is a standalone Python script.

## Architecture

The codebase is a single monolithic Python script (~891 lines) organized into distinct sections:

### Data Models (Lines 28-80)
- `KFactors` dataclass: 7 parameters (K1-K7) with array conversion methods
- `ClimateProfile` dataclass: location/climate configuration (temp range, humidity)
- `CLIMATE_PRESETS` dict: 8 predefined climate profiles

### Physics Engine (Lines 83-172)
Pure functions implementing the Lunatico correction model:
- `calculate_T67()` - cold weather correction factor
- `calculate_Td()` - main temperature correction formula: `Td = (K1/100)×(Ta-K2/10) + (K3/100)×exp(K4/1000×Ta)^(K5/100) + T67`
- `get_ir_temperature()` - IR sensor simulation for sky conditions
- `calculate_Tsky()` - final corrected temperature: `Tsky = Ts - Td`

### Optimization (Lines 173-289)
- `objective_function()` - multi-objective: minimize variance + maintain -18°C target mean + penalize threshold violations
- `optimize_k_factors()` - scipy `differential_evolution` wrapper with 200-point sampling, 2000 iterations, seed=42

### Visualization (Lines 291-461)
- `create_analysis_plots()` - 4-subplot matplotlib figure (clear sky, all conditions, correction factor, comparison)
- `print_results_table()` - formatted terminal output

### Interactive GUI (Lines 464-773)
- `CloudWatcherOptimizer` class using matplotlib widgets (Slider, Button, TextBox)
- Real-time plot updates on slider changes
- Left panel: climate sliders, Right panel: K-factor sliders
- Buttons: OPTIMIZE, Reset, Save Plot

### CLI Interface (Lines 775-871)
- `run_cli()` - non-interactive mode with climate preset selection

### Entry Point (Lines 873-891)
- `main()` - parses `--cli` flag, defaults to GUI mode

## Key Implementation Details

- Optimization runs single-threaded (`workers=1`) for GUI thread safety
- Invalid objective function results return penalty value `1e10`
- Save Plot exports to user's Desktop as PNG
- All physics calculations are pure functions, suitable for unit testing

## Code Style

- All comments must be written in English
- Do not add "Co-Authored-By" or similar AI attribution to commits
