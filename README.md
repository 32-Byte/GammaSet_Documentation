# GammaSet Spectrum Analysis Workflow

## Table of Contents
1. [Overview](#overview)
2. [Components](#components)
3. [Workflow Steps](#workflow-steps)
4. [Best Practices](#best-practices)

---

## Overview

This document outlines the workflow for energy calibration and spectrum analysis using **py_calib_batch**, **GammaSet**, and **EliadeSorting**. The process converts raw ROOT files to energy-calibrated spectra through peak detection, fitting, and calibration using lookup tables (LUTs).

---

## Components

### Tools

**py_calib_batch**
- Python interface for GammaSet providing batch processing, plotting, and efficiency calculation
- Key command structure:
  ```python
  command_line = (
      f"{path}/gammaset -f selected_run_{my_params.runnbr}_{volnbr}_eliadeS{my_params.server}.root "
      f"-rp {lut_recall_fname} -sc {my_params.dom1} -ec {my_params.dom2} -s {src} -pd {my_params.pd} -fd 3 "
      f"-br {my_params.fitrange} -peakthresh {my_params.peakthresh} -rb 1 -hist {my_params.prefix} "
      f"-guideSigma {my_params.guideSigma} -fgf {Tail} -hough 0  -exclude_energy 1085.793,1112.070"
  )
  ```
- Important flags (can be used independently):
  - `--calib`: Enables GammaSet calibration analysis
  - `--plots` + `--processjson`: Generate calibration quality plots
  - `--enerplots`: Show energy shifts across volumes
  - `--update-lut`: Create new LUT_ELIADE files with calibration data

> **Note**: Each flag can be used independently - if calibration is already done, you can run just `--plots`, `--enerplots`, or other options without repeating the full analysis.

**GammaSet**
- Core analysis engine for peak detection, fitting, and polynomial coefficient calculation

**EliadeSorting**
- Converts raw ROOT files to selected spectra using LUT parameters

**Merging Scripts**
- `merge_selected.py`: Combines calibrated spectra from multiple volumes into a single ROOT file
- `file_merging.py`: Alternative merging tool from `EliadeSorting/EliadeTools/`

### Helper and Configuration Files

**start_me.C**
- Helper C++ macro used by both `run_me.sh` and `py_me.py` scripts for spectrum processing tasks
- Must be present in your working directory alongside `run_me.sh` and `py_me.py`

**LUT_ELIADE**
- Controls spectrum conversion parameters in EliadeSorting
- Symbolic link in `py_calib` directory pointing to `onlineEliade` repository

**LUT_RECALL**
- Defines GammaSet analysis parameters (bounds, fit limits, filtering)
- Symbolic link in `py_calib` directory pointing to `onlineEliade` repository

**Link Management Scripts**
- `run_me.sh`: Sets same LUT_ELIADE file for all volumes and executes EliadeSorting for each file
- `py_me.py`: Sets individual LUT_ELIADE files per volume and executes EliadeSorting for each file

> **Note**: EliadeSorting processes one file/volume at a time. The scripts loop through all files to complete batch processing.

**onlineEliade Repository**
- Git repository storing lookup tables and calibration data

---

## Working Directory Structure


The typical working environment follows this directory structure:

```
s7/                           # Server directory (e.g., s7, s9, etc.)
├── root_files/              # Contains raw ROOT files from experiment
│   ├── run170_0_eliadeS7.root
│   ├── run171_0_eliadeS7.root
│   └── ...
└── selector_yourname/       # Your working directory
   ├── run_me.sh           # Script for uniform LUT processing
   ├── py_me.py            # Script for individual LUT processing
   ├── start_me.C          # Helper macro used by run_me and py_me
   ├── selected_run_*_calib/  # Generated calibration directories
   └── ...                 # Analysis outputs and processed files
```

**Note:** The `LUT_ELIADE` and `LUT_RECALL` symbolic links are located in the `py_calib` directory, not in the analysis working directory. Ensure these links are set up in `~/py_calib` as described in the setup section.

### Working Process:
- **Work from**: `selector_yourname/` directory
- **Raw data location**: `../root_files/` (parent directory)
- **Processing scripts**: Execute `run_me.sh` or `py_me.py` from `selector_yourname/` (these require `start_me.C` to be present in the same directory)
- **Output location**: Selected files and analysis results appear in `selector_yourname/`

This structure keeps your analysis work separate from the raw data while maintaining easy access to source files.

---

## Setup and Prerequisites

### Initial Setup
All main components should be installed in your home directory:
- `py_calib_batch` 
- `gammaset`
- `EliadeSorting`
- `onlineEliade`

### Setting Up Symbolic Links
Before starting analysis, you need to configure the LUT symbolic links in the `py_calib` directory:

```bash
# Go to py_calib directory
cd ~/py_calib

# Remove existing links (if any)
rm LUT_ELIADE LUT_RECALL

# Create new symbolic links to onlineEliade repository
ln -s ~/onlineEliade/path/to/LUT_ELIADE.json LUT_ELIADE
ln -s ~/onlineEliade/path/to/LUT_RECALL.json LUT_RECALL
```

> **Note**: The LUT files exist in both locations - the working copies are in `py_calib` as symbolic links, while the master files are stored in the `onlineEliade` repository.

### Getting Scripts
- `run_me.sh`, `py_me.py`, and `start_me.C`: Obtain these from other servers/users or existing analysis directories
- `merge_selected.py`: Obtain from other servers/users or existing analysis directories
- `bulk_extract_times.py`: Located in the py_calib repository

---

## Workflow Steps

### Step 1: Convert Raw Files to Selected Spectra
1. Execute `run_me.sh` (requires `start_me.C` in the same directory) to set up uniform LUT_ELIADE symbolic links and process all files
   - Script automatically finds raw ROOT files in `../root_files/` directory
   - Script loops through each raw ROOT file
   - Applies same LUT_ELIADE configuration to all volumes
   - Executes EliadeSorting for each file individually
   
   **Example command:**
   ```bash
   ./run_me_s7_cebr.sh 191 191 0 169
   ```
   - Arguments: `start_run end_run start_volume end_volume`
   - This processes run 191 for volumes 0 through 169

2. **Output**: Selected ROOT files (e.g., `selected_run_187_0_eliadeS7.root`) containing `mDelila_raw` spectra appear in your `selector_yourname/` working directory

> **Note**: `mDelila_raw` spectra are independent of LUT_ELIADE calibration parameters

### Step 2: Analyze Spectra and Calibrate
1. Configure **LUT_RECALL** parameters:
   - Set detection thresholds and energy bounds (see `onlineEliade` repository for template examples)
   - Remove problematic reference peaks (e.g., merged peaks)
   - Inspect `mDelila_raw` spectra to optimize parameters
2. Run `py_calib_batch --calib` to analyze spectra and calculate polynomial coefficients
   - py_calib_batch handles source information and timing data using run tables and setup files in its directory
   
   **Example command:**
   ```bash
   py_b -s 7 -r 191 -d 101 140 -vol 0 169 -prefix mDelila_raw --calib
   ```
   - `-s 7`: Server number (S7)
   - `-r 191`: Run number
   - `-d 101 140`: Detector range from 101 to 140
   - `-vol 0 169`: Volume range from 0 to 169
   - `-prefix mDelila_raw`: Histogram prefix (raw spectra)
   - `--calib`: Perform calibration analysis

3. **Output**: `_calib` directories containing JSON files with polynomial coefficients

**Troubleshooting Resources:**
Each `_calib` directory contains a `troubleshooting/` folder with:
- **Fit plots**: Visual representations of individual peak fits
- **Detailed analysis text file**: Comprehensive analysis results and diagnostic information
- These resources are invaluable for debugging calibration issues and understanding peak fitting quality

> **Note**: If some volumes fail calibration, you can handle them manually, adjust LUT_RECALL parameters, or review your setup configuration. Check the `troubleshooting/` folder for detailed diagnostic information.

### Understanding GammaSet Terminal Output

During the calibration analysis, GammaSet provides detailed real-time feedback for each volume processed:

```
🏛️  Domain 112: 20 found, 9 filtered, 4 calibrated ✅ (4.70e+00, 2.10e-01), 📊 P/T=0.298
🏛️  Domain 113: 20 found, 8 filtered, 4 calibrated ✅ (5.14e+00, 2.40e-01), 📊 P/T=0.362
```

**Output Explanation:**
- **20 found**: Total peaks detected by the peak detection algorithm
- **9 filtered**: Peaks remaining after quality filtering (amplitude, width, etc.)
- **4 calibrated**: Peaks successfully matched to reference energies for calibration
- **(4.70e+00, 2.10e-01)**: Calibration polynomial coefficients
- **P/T=0.298**: Peak-to-total ratio (quality metric)

**Warning Messages:**
GammaSet issues warnings when peaks are matched but with poor quality. This commonly occurs with:
- **Combined/merged peaks** (especially in scintillator detectors)
- **Overlapping gamma lines**
- **Background artifacts**

**Recommended Action for Warnings:**
If you encounter poorly matched peaks, exclude problematic reference energies by modifying the py_calib configuration file:
```python
# In py_calib file, add exclude_energy parameter to the command line:
f"-exclude_energy 1085.793,1112.070"
```
Example energies to commonly exclude:
- `121.78`: Often problematic in scintillator detectors
- `344.28`: May overlap with other peaks
- `1085.793,1112.070`: High-energy peaks that may be poorly resolved

This removes unreliable reference peaks from the calibration process, improving overall calibration quality.

### Step 3: Update LUT_ELIADE Files
1. Run `py_calib_batch --update-lut` to generate volume-specific LUT files
   
   **Example command:**
   ```bash
   py_b -s 7 -r 191 -d 101 140 -vol 0 169 -prefix mDelila_raw --update-lut
   ```
   - `-s 7`: Server number (S7)
   - `-r 191`: Run number
   - `-d 101 140`: Detector range from 101 to 140
   - `-vol 0 169`: Volume range from 0 to 169
   - `-prefix mDelila_raw`: Histogram prefix (raw spectra)
   - `--update-lut`: Generate updated LUT files with calibration data

2. Move generated directory to `onlineEliade/Lookuptables/<server>/`
3. **Output**: Updated LUT files for production use

### Step 4: Generate Calibrated Spectra
1. Execute `py_me.py` (requires `start_me.C` in the same directory) to process files with volume-specific calibrations
   - Script sets volume-specific LUT_ELIADE symbolic links for each file
   - Executes EliadeSorting for each file with appropriate LUT configuration
   - Loops through all files to apply individual detector calibrations
   
   **Example command:**
   ```bash
   ./py_me_s7_cebr.py 191 191 0 169
   ```
   - Arguments: `start_run end_run start_volume end_volume`
   - This processes run 191 for volumes 0 through 169 with individual calibrations

2. **Output**: ROOT files containing calibrated `mDelila` spectra for each volume

### Step 5: Extract Measurement Times
1. Run `bulk_extract_times.py` in the directory containing raw ROOT files
   
   **Example command:**
   ```bash
   python3 ~/py_calib/tools/bulk_extract_times.py -s 7 -r 191 -vmin 0 -vmax 169
   ```
   - `-s 7`: Server number (S7)
   - `-r 191`: Run number
   - `-vmin 0 -vmax 169`: Volume range from 0 to 169

2. Copy generated JSON timing file to analysis directory
3. **Output**: JSON file with measurement duration data for each volume

> **Required for Step 6**

### Step 6: Verify Calibration Quality

**Important:**  
Before running `py_calib_batch --enerplots`, you must first analyze the calibrated spectra using the `--calib` flag. The `--enerplots` flag needs to read the `_calib` directories, and they need to have analyzed the calibrated plots. When you analyze a calibrated plot, you can see where the peak is and what energy it's supposed to be, which is why we need to run the `--calib` command on the calibrated `mDelila` spectra.

1. **Analyze calibrated spectra:**
   ```bash
   py_b -s 7 -r 191 -d 101 140 -vol 0 169 -prefix mDelila --calib
   ```
   - This command analyzes the calibrated `mDelila` spectra and creates `_calib` directories containing peak positions and expected energies.
   - **Note:** This will override the existing `_calib` directories from when you analyzed `mDelila_raw` spectra in Step 2.

2. **Generate energy shift plots:**
   ```bash
   py_b -s 7 -r 191 -d 101 140 -vol 0 169 -prefix mDelila --enerplots
   ```
   - `-s 7`: Server number (S7)
   - `-r 191`: Run number
   - `-d 101 140`: Detector range from 101 to 140
   - `-vol 0 169`: Volume range from 0 to 169
   - `-prefix mDelila`: Histogram prefix (calibrated spectra)
   - `--enerplots`: Generate energy shift plots (requires timing JSON file and `_calib` directories from previous step)

3. **Analyze plots to verify:**
   - Energy peak alignment across volumes (spread between energy points should be reasonably large)
   - Calibration consistency over time
   - Absence of systematic anomalies
3. If issues found, return to Steps 2-4 with adjusted parameters

### Step 7: Merge Calibrated Files
1. Run `file_merging.py` from `EliadeSorting/EliadeTools/`
2. **Output**: Single merged ROOT file containing summed spectra

---

## Best Practices


---

## Quick Guide: Analyzing a Spectrum

To analyze a spectrum (outside the full calibration workflow), use `py_calib_batch` with these flags:

1. Run with `--calib` to perform spectrum analysis and peak fitting:
   ```bash
   py_calib_batch --calib -f <selected_spectrum_file.root>
   ```
2. Use `--plots` and `--processjson` to generate and view analysis plots:
   ```bash
   py_calib_batch --plots --processjson -f <selected_spectrum_file.root>
   ```

This will produce peak detection results, fit parameters, and visualizations for the selected spectrum file.

---

## Frequently Asked Questions

**Q: Where can I find examples of LUT_RECALL parameters?**  
A: Check the `onlineEliade` repository for template files that show typical parameter configurations.

**Q: What if some volumes fail calibration in Step 2?**  
A: You have several options:
- Handle failed volumes manually
- Reconsider and adjust the LUT_RECALL parameters
- Review your overall setup and configuration

---

## Best Practices

### Before Analysis
- Verify symbolic link integrity for all LUT files
- Create backup copies of baseline LUT files
- Document any LUT_RECALL parameter modifications

### During Analysis
- Monitor `_calib` directory contents and JSON files for troubleshooting
- Use `--plots` and `--processjson` together for quality assessment
- Use `--enerplots` to check cross-volume consistency

### Quality Control
- Ensure timing data is present before running `--enerplots`
- Validate energy peak alignment within expected tolerances
- Check calibration polynomial coefficients for physical reasonableness

---

## Single File Analysis and Plotting

For detailed analysis of individual spectrum files, you can combine multiple flags to generate comprehensive plots and analysis results in a single command.

### Comprehensive Analysis Command

```bash
py_b -s 7 -r 191 -d 101 140 -vol 0 169 -prefix mDelila --calib --processjson --plots
```

This command simultaneously performs:
- `--calib`: Peak detection, fitting, and calibration analysis
- `--processjson`: Process analysis results and generate data files
- `--plots`: Create visualization plots

### Generated Plots

The combined analysis simultaneously generates comprehensive plots for:

1. **Efficiency Plots**: Detection efficiency across energy ranges
2. **Peak-to-Total Ratio Plots**: Background-to-peak signal quality metrics visualization
3. **Calibration Plots**: Polynomial coefficients and energy calibration visualization (if analyzing uncalibrated data)
4. **Resolution Plots**: Energy resolution as a function of energy

### Use Cases

- **Initial spectrum assessment**: Quick overview of data quality and calibration status
- **Quality control**: Verify analysis parameters before batch processing
- **Detailed investigation**: In-depth analysis of specific problematic spectra
- **Documentation**: Generate plots for reports and presentations

> **Note**: This approach is particularly useful for troubleshooting and parameter optimization before running full batch analyses.

