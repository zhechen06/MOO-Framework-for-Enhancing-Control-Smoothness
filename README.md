# Optimal Chiller Loading Results Analysis

This document describes the key variables, data structures, and execution flow of the Jupyter Notebook `optimal_chiller_loading.ipynb`. The notebook focuses on optimizing chiller loading strategies to balance energy consumption and control smoothness, in support of the research manuscript.

In addition to the Jupyter Notebook, an HTML version (`optimal_chiller_loading.html`) is also provided. This allows you to conveniently view the full analysis workflow and results in any web browser, without requiring a Jupyter environment.

## 1. Overview

The notebook performs the following main tasks:

* Loads chiller performance data and historical cooling load profiles.
* Implements and runs various optimization algorithms (metaheuristics like PSO, GA, SA, DE; a multi-objective algorithm NSGA-II; and the Gurobi solver) to determine optimal chiller Part Load Ratios (PLRs).
* Evaluates the performance of these algorithms based on energy consumption and chiller switching frequency.
* Analyzes the impact of different parameters (e.g., random seeds, tolerance levels for MOO) on the results.
* Generates visualizations and summary tables, corresponding to figures and tables in the manuscript.

## 2. Environment Setup

[uv](https://github.com/astral-sh/uv) is a fast Python package installer and environment manager. To set up the environment and install the required libraries:

```bash
# Navigate to the project root directory (where pyproject.toml is located)
# Then run:

# Install uv if you don't have it
pip install uv

# Install dependencies from pyproject.toml
uv sync

# Activate the virtual environment
# On macOS/Linux:
source .venv/bin/activate

# On Windows:
# .venv\Scripts\activate
```

The notebook relies on several Python libraries that will be installed by `uv sync`:

* `pandas==2.0.0`
* `numpy==1.24.2`
* `matplotlib==3.7.1`
* `seaborn==0.12.2`
* `scikit-opt==0.6.6` (imported as `sko` for GA, DE, PSO, SA)
* `pymoo==0.6.1.1` (for NSGA-II)
* `gurobipy==11.0.1` (for Gurobi optimizer)

Note: Gurobi requires a license. Please refer to [Gurobi&#39;s documentation](https://www.gurobi.com/documentation/) for license setup.

## 3. Prerequisites and Notebook Execution

### 3.1 Notebook Execution and Pre-generated `.npy` Files

* **Computational Intensity:** Many optimization algorithms, especially when run over multiple days, random seeds, and parameter variations, are computationally intensive and can take a significant amount of time to complete.
* **Pre-generated Results:** To facilitate quicker review and analysis, this notebook is structured to **load pre-generated results** from `.npy` files for several key stages.
* **Generating Results:** The code cells responsible for *generating* these `.npy` files (e.g., `save_result()` function calls, loops running `eval_algorithms()`, or Gurobi optimization loops) are often **commented out**  in the provided notebook version.
  * If you wish to regenerate these results from scratch, you will need to:
    1. Uncomment the relevant code sections.
    2. Be prepared for potentially long execution times.
    3. Optimize multi-threaded execution
* **Data Directory:** Results are typically saved and loaded from a `./results/` subdirectory, often further organized by `seed` and `tolerance` levels. Ensure this directory structure is appropriate if you regenerate data.

## 4. Core Input Data

* **`df_chiller` (pandas.DataFrame):**

  * Loaded from `./chiller_info.csv`.
  * Contains performance characteristics of each chiller (coefficients `a, b, c, d()` for power calculation and `Capacity`).
  * Index: `Chiller` number/ID.
* **`abcd` (numpy.ndarray):**

  * Derived from `df_chiller`.
  * Array of performance coefficients `[a, b, c, d(=0))]` for each chiller. Shape: `(N, 4)`.
* **`cap_array` (numpy.ndarray):**

  * Derived from `df_chiller`.
  * Array of cooling capacities for each chiller. Shape: `(N,)`.
* **`N` (int):**

  * Total number of chillers.
* **`df_cl` (pandas.Series):**

  * Loaded from `./cooling_load.csv`.
  * Raw cooling load profile.
* **`cl_selected` (list of pandas.Series):**

  * Processed daily cooling load profiles (one Series per day), scaled by 2.

## 5. Key Result Variables (Loaded from `.npy` files)

These variables store the outputs of various optimization runs. Their names generally follow the convention:
`result_<metric>_<scope>_<seeds>_<tolerance>.npy`

* **`<metric>`**:
  * `PLR`: Part Load Ratio.
  * `Ntot`: Total Number of Switching Operations (often derived from PLR).
  * `eval`: Evaluated Metrics (e.g., daily energy consumption, total switching operations).
* **`<scope>`**:
  * `Gurobi`: Results from Gurobi optimizer.
  * `MOO`: Results from Multi-Objective Optimization (NSGA-II).
  * `all`: Results for all compared algorithms.
* **`<seeds>`**:
  * `seed0`: Results from a single random seed (0).
  * `seed10`: Results from 10 different random seeds (0-9).
* **`<tolerance>` (or `tol`)**:
  * `tol05`: Tolerance ⍺ = 0.05 (5%) for MOO.
  * `tolX`: tolerance levels (1%-10%) for MOO.

### 5.1 Specific Variable Explanations:

1. **`result_PLR_Gurobi`**

   * **`PLR`**: Part Load Ratios.
   * **`Gurobi`**: From the Gurobi optimizer.
   * **Content**: Time series of PLR values for each chiller, for each simulated day, by Gurobi.
   * **Typical Shape**: `(num_days, num_timesteps_per_day, num_chillers)`
2. **`result_Ntot_all_seed0_tol05`**

   * **`Ntot`**: Total switching operations per chiller.
   * **`all`**: For all algorithms (PSO, GA, SA, DE, Gurobi, MOO).
   * **`seed0`**: Run with random seed 0.
   * **`tol05`**: MOO used ⍺=0.05.
   * **Content**: Total on/off switches per chiller, per day, per algorithm (single seed).
   * **Typical Shape**: `(num_algorithms, num_days, num_chillers)`
3. **`result_PLR_all_seed10_tol05`**

   * **`PLR`**: Part Load Ratios.
   * **`all`**: For all algorithms.
   * **`seed10`**: Aggregated over 10 random seeds.
   * **`tol05`**: MOO used ⍺=0.05.
   * **Content**: Time series of PLRs per chiller, per time step, per day, per seed, per algorithm.
   * **Typical Shape**: `(num_algorithms, num_seeds, num_days, num_timesteps_per_day, num_chillers)`
4. **`result_eval_MOO_seed10_tolX`**

   * **`eval`**: Evaluated metrics (energy, switching).
   * **`MOO`**: For the Multi-Objective Optimization algorithm.
   * **`seed10`**: Aggregated over 10 random seeds.
   * **`tolX`**: For various tolerance levels (1% to 10%).
   * **Content**: Daily energy and switching for MOO, per day, per seed, per tolerance level.
   * **Typical Shape**: `(num_days, num_seeds, num_tolerance_levels, num_metrics, num_timesteps_per_day)`.
5. **`result_eval_all_seed10_tol05`**

   * **`eval`**: Evaluated metrics (energy, switching).
   * **`all`**: For all compared algorithms (PSO, GA, SA, DE, MOO).
   * **`seed10`**: Aggregated over 10 random seeds.
   * **`tol05`**: MOO component used ⍺=0.05.
   * **Content**: Daily energy and switching for each algorithm, per day, per seed.
   * **Typical Shape**: `(num_days, num_seeds, num_algorithms, num_metrics, num_timesteps_per_day)` (where the last dimension holds raw data before daily aggregation).

## 6. Workflow and Outputs

1. **Setup & Imports:** Load libraries, define common settings (colors, colormaps).
2. **Data Loading:** Load chiller data (`df_chiller`) and cooling load data (`df_cl`, `cl_selected`).
3. **Algorithm Implementation:** Define functions for each optimization algorithm (PSO, GA, SA, DE, Gurobi, NSGA-II).
4. **Optimization Runs (Speed up by Loading Pre-generated Results):**
   * The `eval_algorithms` function orchestrates runs for metaheuristics and MOO.
   * Gurobi optimizations are run separately.
   * Results (PLR arrays) are saved to `.npy` files via `save_result` or direct `np.save`.
5. **Result Loading & Preprocessing:**
   * Load the `.npy` files (as described in Section 4).
   * Reshape and combine data into pandas DataFrames (e.g., `df_tol`, `df_results`) for analysis.
   * Calculate derived metrics like daily/weekly totals for energy and switching.
6. **Analysis, Visualization, and Output Generation:**
   * Generate heatmaps, line plots, box plots, scatter plots to compare algorithm performance.
   * These plots likely correspond to figures in the associated manuscript (e.g., "Fig. X in the manuscript").
   * Create tables summarizing key performance indicators (e.g., "Table X in the manuscript").
   * All generated figures and tables are saved to the `./figures/` directory and are explicitly labeled in the notebook with comments like `# Fig. X in the manuscript` or `# Table X in the manuscript`.


## 7. General Notes on Data Processing

* The notebook often reshapes and aggregates raw result arrays into pandas DataFrames for easier analysis and plotting.
* "Energy consumption" is typically calculated from PLR values using the polynomial coefficients and then summed over a period.
* "Total switching number" (`Ntot`) is derived by counting changes in chiller operational state (on/off) based on PLR values (PLR > 0.01 or similar threshold means ON, PLR = 0 means OFF).
