# SF Banked Rent Increase Calculator

This tool was built for calculating unrealized property value in properties subject to the San Francisco Rent Ordinance. In essence, it solves a missing paperwork problem - when a real estate company is interested in acquiring a rent-controlled property, they could ideally calculate this value simply based on information in the paperwork. However, more often than not, this paperwork is missing or was never generated. To calculate this value instead, an algorithm that calculates possible rent increase schedules is necessary - this calculator accomplishes this endeavor.

Given a property's starting rent, current rent, and lease start date, this calculator determines which annual allowable increases were likely used and which remain "banked" (available for future use i.e. unrealized value).

---

## Table of Contents

1. [Background: SF Rent Ordinance Banking](#background-sf-rent-ordinance-banking)
2. [Core Algorithm](#core-algorithm)
   - [The Problem](#the-problem)
   - [Historical Rate Data](#historical-rate-data)
   - [Search Strategy (Generalized)](#search-strategy-generalized)
   - [Pruning Optimizations](#pruning-optimizations)
3. [GUI Application](#gui-application)
   - [Single Property Tab](#single-property-tab)
   - [Multiple Properties Tab](#multiple-properties-tab)
   - [Output Configuration](#output-configuration)
4. [Installation & Usage](#installation--usage)
5. [File Structure](#file-structure)

---

## Background: SF Rent Ordinance Banking

Under San Francisco's Rent Ordinance, landlords of rent-controlled units are allowed to raise rent by a specific percentage each year (the rate is tied to inflation and hence different every year). If a landlord doesn't take the full allowable increase in a given year, that unused portion can be "banked" and applied in future years in an uncompounded manner.

Only increases from April 1, 1982 onward can be banked. Earlier increases (1979-1982) count toward rent growth calculations but cannot be saved for later use, which the calculator accounts for when producing possible rent increase schedules.

This calculator reverse-engineers a property's rent history to determine:
- Which annual increases were likely applied
- Which increases were not used and hence become banked rates 
- The maximum legal rent if all banked increases were applied today (in an uncompounded manner)

---

## Core Algorithm

**Note:** There are multiprocessing capabilities in the code that worked well when bundled into a Windows executable, but were problematic when bundled into a macOS .app, so they remain in the code but are unused and can be ignored.

### The Problem

Given:
- Lease Start Date (e.g., June 15, 1985)
- Starting Rent (e.g., $500)
- Current Rent (e.g., $1000)

Find: A valid combination of annual rent increases that explains how the rent grew from the starting amount to the current amount.

Solving this problem requires combinatorial search. With 40+ years of rate periods, there are potentially billions of combinations to evaluate. The algorithm efficiently searches a generated space based on the input while respecting other constraints (tolerance and a potential minimum gap between rent increases).



### Historical Rate Data

The calculator contains a complete table of SF Rent Ordinance allowable increases from 1979 to 2026:

```python
RATES = {
    '1979-06-13_to_1980-02-29': 0.07,  # 7% - NOT BANKABLE
    '1980-03-01_to_1981-02-28': 0.07,  # 7% - NOT BANKABLE
    # ... early periods at 7% ...
    '1984-03-01_to_1985-02-28': 0.04,  # 4%
    # ... rates vary by year based on CPI ...
    '2023-03-01_to_2024-02-29': 0.036, # 3.6%
    '2024-03-01_to_2025-02-28': 0.017, # 1.7%
    '2025-03-01_to_2026-02-28': 0.017, # 1.7%
}
```

Each period is marked as bankable or non-bankable based on whether it starts on or after April 1, 1982.



### Search Strategy (generalized)

The algorithm uses a bidirectional search approach:

1. Estimate the number of increases needed using a logarithmic approximation (rent increases compound exponentially over the uses, so logarithms can "undo" the compounding growth).

2. First search downward from the estimate - the downward search is a simple optimization to calculate the maximum banked rent increase, less increases means more banked rate periods.
  - try `n`, if one valid solution is found, try `n-1`, if a valid solution is found, try `n-2`, etc.
  - a) For each "level" or number of increases in the current potential schedul, generate combinations of that many periods and evaluate whether they produce a rent within the acceptable tolerance of the current rent.
  - b) If progressively lower levels fail 3 times in a row, return to the `n`th level and search upwards (`n+1 `) to find a solution.

3. **Stop when a valid solution is found** or when the search space is exhausted.


### Combination Evaluation

For each combination of periods, the algorithm:

1. **Checks the gap constraint** — ensures minimum years between consecutive increases (configurable, default 1 year)

2. **Calculates the resulting rent** using compound growth:
   ```python
   calculated_rent = starting_rent
   for rate in selected_rates:
       calculated_rent *= (1 + rate)
   ```

3. **Checks if within tolerance** — the calculated rent must be between `current_rent` and `current_rent * (1 + tolerance)`
   
   Note: it will not allow a rent schedule that gets within the tolerance level BELOW the current rent, as this would overcalculate the banked rate and thus not be in accordance with SF Rent Ordinance

4. **Calculates banked amount** — sum of rates for bankable periods that were NOT used - this SUMMED amount is used to calculate what the legal maximum rent is if applied today (the summed rate does not allow a landlord to take advantage of the compounding effect with banked increases)



### Pruning Optimizations

To handle the combinatorial explosion, the algorithm employs several pruning strategies:

#### 1. Pre-parsing
All period data is parsed once upfront into `PeriodInfo` objects, avoiding repeated string parsing during the search.

#### 2. Preliminary Check
Before the program begins intensive calculation, it first checks if the target (current rent) is even possible to reach with the conditions. It quickly calculates what the maximum rent possible is if ALL subsequent rent increases were applied, and if the current rent is not reachable, then it performs no further calculation and reports the error to the user (this happens if a user puts in values for a property that is potentially not in compliance with SF Rent Ordinance).

#### 3. Early Termination (Bounds Checking)
During combination generation, the algorithm tracks:
- **Current accumulated value** — rent if we stopped here
- **Maximum possible value** — rent if we used all remaining highest-rate periods

If the current value already exceeds the target (with tolerance), or if the maximum possible value can't reach the target, that branch is pruned.

```python
# Pruning thresholds
PRUNE_LOWER_FACTOR = 0.92  # Stop if max_possible < target * 0.92
PRUNE_UPPER_FACTOR = 1.10  # Stop if current_value > target * 1.10
```

#### 4. Multi-Stage Pruning
Pruning checks happen at multiple stages:
- During initial combination generation
- At periodic intervals
- With progressively tighter bounds as the search continues

#### 5. The Chronological Constraint
While many combinations could be checked that apply different increases from different years (i.e. applying the rent increase from 1995 and then applying a rent increase from 1985), these rent schedules are useless and hence are not calculated.

---

## GUI Application

The GUI (`rent_calculator_gui_tabbed.py`) provides a user-friendly interface with two modes of operation.

### Single Property Tab

For calculating a single property's banked rent increases.

**Input Fields:**

| Field | Description | Required |
|-------|-------------|----------|
| Address | Property street address | Optional |
| Unit | Unit number/letter | Optional |
| Lease Start Date | When the tenancy began (Year/Month/Day) | Required |
| Starting Rent | Initial rent amount | Required |
| Current Rent | Present rent amount | Required |
| Tolerance (%) | How far above current rent a solution can be (default: 5%) | Optional |
| Min Gap Between Increases | Minimum years between rent increases (default: 1) | Optional |

**Output:**
- Single-row CSV with columns: Address, Unit, Lease Start Date, Starting Rent, Current Rent, Max Rent, Total Banked Rate, [Error Message], and all period columns
- Each period column shows either `0.0%` (used/non-bankable) or the rate percentage (banked)


### Multiple Properties Tab

For batch processing multiple properties from a CSV file.

**Workflow:**
1. **Load CSV** — Select a CSV file containing property data
2. **Map Columns** — Specify which columns contain Lease Start Date, Starting Rent, and Current Rent
3. **Configure Settings** — Set tolerance and minimum gap
4. **Run Calculation** — Process all properties
5. **Save Results** — Export to a new CSV with calculated columns appended

**Supported Date Formats:**
- `M/D/YYYY` (e.g., 6/15/1985)
- `MM/DD/YYYY` (e.g., 06/15/1985)
- `YYYY-MM-DD` (e.g., 1985-06-15)


### Output Configuration

Both tabs offer live-updating output configuration:

#### Period Column Order
- **Newest First** — Most recent periods appear first (left to right)
- **Oldest First** — Oldest periods appear first

#### Error Message Placement
- **Before Rental Periods** — Error column appears before period columns
- **After Rental Periods** — Error column appears after period columns
- **Do not include** — Omit the error message column entirely

**Live Preview:** Changing these options instantly updates the results preview without recalculating. This works by pre-computing all 6 variants (3 error placements × 2 period orders) during calculation.


### Additional Features

- **Open in Larger View** — Popup window for easier horizontal scrolling through many columns
- **Unsaved Results Warning** — Prompts before clearing if results haven't been saved
- **Progress Tracking** — Shows calculation progress and elapsed time
- **Cancel and Restart** — Stop a running calculation and reset

---

## Installation & Usage

### Requirements
- Python 3.8+
- All other libraries used are part of the Python standard library

### Running from Source
```bash
# Navigate to the directory containing the files
cd /path/to/calculator

# Run the GUI
python3 rent_calculator_gui_tabbed.py
```

### Command Line Mode
The core module can also be run directly for command-line usage:
```bash
python3 rent_calculator_core_multiprocess.py
```

---

## File Structure

```
├── rent_calculator_core_multiprocess.py   # Core algorithm and calculation logic
├── rent_calculator_gui_tabbed.py          # Tkinter GUI application
└── README.md                               # This file
```

### Module Dependencies

```
rent_calculator_gui_tabbed.py
    └── imports from: rent_calculator_core_multiprocess.py
            ├── find_rent_increases()
            ├── find_rent_increases_with_retry()
            ├── validate_date()
            ├── RATES (dict)
            ├── BANKING_START_DATE
            └── CalculationCancelled (exception)
```

---

## License

This software is provided as-is for calculating SF Rent Ordinance banked increases. Always verify calculations with official Rent Board resources for legal purposes.
