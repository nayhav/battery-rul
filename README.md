# Lithium-ion battery remaining useful life prediction

**Neha Varadharajan · 240962214 · CSE AIML · ML Mini Project**

A runnable implementation of the supplied proposal: NASA cycle-level health indicators,
Random Forest, XGBoost, a genuine PyTorch LSTM, 10/20/30-cycle history experiments,
full/reduced feature comparisons, battery-level evaluation, and Tree SHAP explanations.
Extra components include a cycle-only baseline, optional Ridge baseline, nested model
selection, a censoring audit, reusable model artifacts, an HTML results report, and
scientific-integrity tests. Code comments explain the non-obvious decisions.

**Start with the demo, then run the NASA experiment.** The bundled executed results
use synthetic fixtures and are labelled accordingly. They are not NASA findings.
No assignment grade can be guaranteed; the evidence, explanation, and your own analysis matter.

## 1. Windows + VS Code setup

1. Extract the entire ZIP. Open the **battery_rul** folder containing this README
   with **File → Open Folder** in VS Code (do not open only an individual script).
2. Install Python **3.11 or 3.12**, plus Microsoft's Python and Jupyter VS Code extensions.
3. In the VS Code terminal, run:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements-full.txt
```

Using the explicit Python path avoids PowerShell activation-policy problems. If you
installed Python 3.12, replace `-3.11` with `-3.12`. Select `.venv` via
**Ctrl+Shift+P → Python: Select Interpreter**. For notebooks, choose the same environment
using **Select Kernel**. A GPU is not required; LSTM runs on CPU.

For a smaller first install, use `requirements.txt`; the core demo needs no deep-learning packages.
Commands below use `python`. On Windows, replace it with `.\.venv\Scripts\python.exe`
unless you have activated `.venv`. VS Code's F5 configurations are included.

macOS/Linux:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements-full.txt
```

## 2. Run immediately: no dataset download needed

```bash
python -m battery_rul demo --output outputs/my_demo
python -m unittest discover -s tests -v
```

Open `outputs/my_demo/report.html` in your browser. This runs the cycle baseline,
Random Forest and Ridge on explicitly synthetic data, with 10/20/30 histories and
full/reduced features. It exercises real preprocessing, nested cell splits and reporting.
The demo uses small models for speed. `outputs/verified_demo/` contains the supplied run.

To smoke-test all proposed models and SHAP after installing full dependencies:

```bash
python -m battery_rul demo --full-models --output outputs/my_full_demo
```

That is still synthetic, and its four LSTM epochs are only an execution check.
**Each output directory must be new/empty.** Use another name for reruns; this prevents
accidentally mixing stale NASA and demo artifacts.

## 3. Obtain and prepare the NASA data

```bash
python -m battery_rul download
python -m battery_rul extract
python -m battery_rul audit --data data/processed/nasa_cycles.csv
```

The downloader uses the archive linked from [NASA's PCoE repository](https://www.nasa.gov/intelligent-systems-division/discovery-and-systems-health/pcoe/pcoe-data-set-repository/).
If a firewall or changed host blocks it, download **5. Battery Data Set** in a browser:

```bash
python -m battery_rul download --archive "C:/Users/YourName/Downloads/5. Battery Data Set.zip"
python -m battery_rul extract
```

Alternatively place the original `B0005.mat`, `B0006.mat`, `B0007.mat`, and `B0018.mat`
inside `data/raw/` and run `extract`. Keep original files unchanged. Nested ZIPs are
supported; only the four named cells are extracted. The parser expects NASA's original
MATLAB `Bxxxx.cycle` structure, not an unrelated Kaggle CSV schema. SHA-256 provenance
is written beside the processed CSV. No NASA raw files are bundled in this delivery.

**Read the audit before training.** NASA specifies 30% fade of nominal 2 Ah, i.e. 1.4 Ah.
A selected cell may not reach that threshold in its available records. The code treats
it as right-censored, keeps it in the audit, and excludes it from supervised RUL labels.
It does not call the final record EOL. As a result, the number of scored outer folds can
be less than four. The complete nested experiment requires at least three observed-EOL
cells. Do not change the threshold merely to obtain nicer results.

## 4. Full assignment experiment

```bash
python -m battery_rul run --data data/processed/nasa_cycles.csv --output outputs/nasa_full
```

This trains RF, XGBoost, LSTM and the cycle baseline for every feature/window/held-out-cell
combination, tunes model parameters on inner training-cell folds, and explains the tree
chosen by inner validation for each outer test cell. It can take substantially longer
than the demo; runtime depends on CPU and data. Progress is printed per cell/configuration.

For a stronger robustness analysis, average three seeds:

```bash
python -m battery_rul run --data data/processed/nasa_cycles.csv --output outputs/nasa_three_seeds --seeds 7 42 123
```

Optional capacity-only control and linear model:

```bash
python -m battery_rul run --data data/processed/nasa_cycles.csv --output outputs/nasa_controls --feature-sets full reduced capacity_only --models rf xgb lstm ridge
```

Core-only NASA run, if full dependencies are unavailable (does not fulfil the entire model comparison):

```bash
python -m battery_rul run --data data/processed/nasa_cycles.csv --output outputs/nasa_core --models rf ridge --no-shap
```

Do not present `--fast` results as the full experiment. Save the run directory alongside your submission.
`config.json`, `environment.json`, input hashes and `tuning.json` document exactly what ran.

## 5. Equivalent notebook

Open `notebooks/Battery_RUL_Complete.ipynb` and choose **Run All**.
The notebook is standalone: all implementation functions are included as readable code
cells; no import of the surrounding project is needed. It starts in the verified core
demo mode. The setup cell gives package installation instructions.

To run the full project, edit its configuration to `MODE = "nasa"`, set `RAW_DIR` to the
folder containing the four MATLAB files, and set `FULL_MODELS = True`. It uses the same
algorithms, model grids, splits and reporting as the package. The supplied notebook
contains outputs from the synthetic core run, not unexecuted NASA performance claims.
`tools/build_notebook.py` regenerates an unexecuted equivalent notebook after code changes.

## 6. Saved-model prediction

Use the historical measurements of exactly one battery, from cycle 1 through the current
completed cycle, with the cycle-level column names in `docs/DATA_CARD.md`.
There must be at least as many cycles as the saved model's history window.
Missing optional signal columns are treated as missing, and the training transform is reused.

```bash
python -m battery_rul predict --model outputs/verified_demo/models/rf_full_w20_seed42_SIM01.joblib --history data/processed/demo_history.csv
```

The bundled history contains only the first 50 cycles of SIM01, without targets or EOL.
The matching artifact was trained without SIM01. If any measured past capacity has
already reached the threshold, the deployment wrapper reports RUL=0. Benchmark metrics
use the clipped model prediction without this known-EOL override, so errors at EOL remain visible.
Load only trusted `.joblib` files. Keep package/library versions aligned with the saved run.
These tiny-cell models are educational, not validated battery-management controllers.

## 7. Output guide

| File | Purpose |
|---|---|
| `report.html` | Portable experiment summary, capacity plots, predictions and errors |
| `eol_audit.csv` | Threshold crossings, missing capacity, censoring and excluded post-EOL rows |
| `summary.csv` | Equal-cell macro metrics, after averaging seeds within each cell |
| `metrics_by_cell.csv` | Every model/window/feature/seed/test-cell metric |
| `predictions.csv` | Every held-out prediction and residual |
| `errors_by_life_stage.csv` | Early, middle and near-EOL errors; retrospective diagnostics only |
| `tuning.json` | Inner validation scores, candidates and selected hyperparameters |
| `inner_selected_tree_metrics.csv` | Outer results for trees chosen entirely with inner validation |
| `models/*.joblib` | Model + fold-fitted preprocessing + feature schema + provenance |
| `models/*.loss.json` | LSTM normalized training MSE by epoch |
| `shap/<cell>_seed<seed>/` | Held-out SHAP plots, signed values, feature rankings and base values |
| `config.json`, `environment.json` | Reproducibility record |
| `run_status.json` | Written only on successful completion |

## 8. Submission route

Read `docs/METHODOLOGY.md`, `docs/DATA_CARD.md`, `docs/VIVA_AND_SUBMISSION.md`, and
`docs/VALIDATION.md`. They explain the experimental choices and verification limits.
Run the full NASA experiment locally, inspect all cell errors, then write your discussion
from those measured results. Understand and adapt the implementation, and acknowledge
assistance according to your course's policy.

## Troubleshooting

- **Module not found:** wrong interpreter or terminal directory. Open the project folder,
  select `.venv`, and use that environment's Python. `python -m pip --version` shows its location.
- **No compatible SciPy wheel:** use Python 3.11/3.12, not an older Python; upgrade pip.
- **Download denied:** use the manual archive route. The app never substitutes synthetic data for NASA.
- **No EOL / too few cells:** inspect the audit, units and source files. A censored record has no exact RUL label.
- **All features missing/constant:** check the CSV units/schema and extraction output.
- **Output directory exists:** choose a new output name. No files are silently overwritten by experiment runs.
- **SHAP/backend error:** use a fresh environment and the full requirements. Errors remain visible;
  no different algorithm is silently substituted. Record resolved versions with `python -m pip freeze`.
- **Large download/storage:** the source archive contains more cells than this project uses; it is not included in the ZIP.

Notebook outputs show paths from the verification environment; rerunning creates paths on your own computer.
Notebook-created model files should be reloaded in the notebook after its definition cells; use package-created models with the CLI.
