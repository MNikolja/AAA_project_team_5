# Chicago Taxi Demand & Smart Charging — Run Guide

Team 5 · Advanced Analytics & Applications

This guide explains how to set up the `uv` environment, where to get the raw data, in
which order to run the notebooks, and how the **sample / quick-run modes** work.

---

## 1. Prerequisites

| Tool | Needed for | Install |
|---|---|---|
| [uv](https://github.com/astral-sh/uv) | Python environment & dependencies | `powershell -c "irm https://astral.sh/uv/install.ps1 \| iex"` (Windows) · `curl -LsSf https://astral.sh/uv/install.sh \| sh` (macOS/Linux) |
| [Quarto](https://quarto.org/docs/get-started/) | Rendering `report.qmd` (optional for notebooks) | Installer from quarto.org |
| LaTeX (TinyTeX) | PDF output (optional) | `quarto install tinytex` |

Python itself does **not** need to be pre-installed — `uv` fetches the interpreter
(`requires-python >= 3.10`) automatically.

---

## 2. Set up the environment

From the repository root:

```bash
uv sync
```

This creates `.venv/` and installs the exact pinned versions from `uv.lock`
(polars, pandas, geopandas, h3, osmnx, scikit-learn, torch, stable-baselines3, …).

Register the Jupyter kernel so notebooks and Quarto can find the environment:

```bash
uv run python -m ipykernel install --user --name aaa-team-5 --display-name "Python (AAA Team 5)"
```

Anything you run outside the notebooks should be prefixed with `uv run`, e.g.
`uv run quarto render report.qmd`.

---

## 3. Get the raw data (`data/00`)

The raw input folder is **not** in the repository due to the size.
Download it from the onedrive link below and unpack it into `data/00/` so that the paths below exist:

> **OneDrive:** `https://1drv.ms/u/c/2b600f51f0f0fd46/IQDTGwJbPJALR6alcWcGyUHpAThhNn4_WZTpIjlnzZfcBnM?e=9Iketp`

```
data/00/
├── Taxi_Trips_(2024-).csv           # raw Chicago taxi trips, ~15.4 M rows  -> nb 01
├── Census_Tracts.csv                # tract geometries/ids                  -> nb 01, 02
├── Community_Areas.csv              # community area geometries/ids         -> nb 02
├── chicago_census_tracts.geojson    # tract boundaries for maps             -> nb 05, 06b
├── chicago_community_boundaries.json# community area boundaries for maps    -> nb 05, 06a, 06c, 07
├── chicago_holidays.xlsx            # holiday calendar (sheet "Calendar")   -> nb 02
├── chicago_events_20260515.csv      # city events export                    -> nb 02
└── tl_2025_17_tract/                # TIGER/Line shapefile (reference only)
```

All notebooks use **relative paths** (`../data/...`) and therefore must be executed with
`notebooks/` as the working directory — which is what VS Code / Jupyter do by default when
the notebook is opened from that folder.

---

## 4. Run order

Notebooks are **chronological** — run them top to bottom, in numeric order. Each one
writes files that later notebooks read.

| # | Notebook | What it does | Toggle |
|---|---|---|---|
| 00 | [00_create_mini_dataset.ipynb](notebooks/00_create_mini_dataset.ipynb) | Cuts one week out of the raw CSV (optional, see §6) | — |
| 01 | [01_data_cleaning.ipynb](notebooks/01_data_cleaning.ipynb) | Cleaning: types, missing values, duplicates, outliers, time features | **`SAMPLE_MODE`** |
| 02 | [02_other_data_sources.ipynb](notebooks/02_other_data_sources.ipynb) | Adds holidays, events, weather, POIs and centroids (needs internet) | — |
| 03 | [03_data_preparation_for_nn_and_svm.ipynb](notebooks/03_data_preparation_for_nn_and_svm.ipynb) | Aggregates demand per area and time bin, splits and scales it | — |
| 04 | [04_hexagon_segmentation.ipynb](notebooks/04_hexagon_segmentation.ipynb) | Assigns H3 hexagon ids to pickup/dropoff coordinates | — |
| 05 | [05_spatial_data_exploration.ipynb](notebooks/05_spatial_data_exploration.ipynb) | Explores the spatial structure of demand | — |
| 06a | [06a_community_feature_visualization.ipynb](notebooks/06a_community_feature_visualization.ipynb) | Trip counts and features per community area | — |
| 06b | [06b_census_tract_feature_visualization.ipynb](notebooks/06b_census_tract_feature_visualization.ipynb) | Same, per census tract | — |
| 06c | [06c_hexagon_feature_visualization.ipynb](notebooks/06c_hexagon_feature_visualization.ipynb) | Same, per H3 hexagon | — |
| 07 | [07_spatial_hotspot_analysis.ipynb](notebooks/07_spatial_hotspot_analysis.ipynb) | Hotspot detection with KDE and GMM | — |
| 08a | [08a_SVR.ipynb](notebooks/08a_SVR.ipynb) | Demand forecasting with SVR | **`QUICK_RUN`** |
| 08b | [08b_NN.ipynb](notebooks/08b_NN.ipynb) | Demand forecasting with a neural network | **`QUICK_RUN`** |
| 09 | [09_smart_charging_rl.ipynb](notebooks/09_smart_charging_rl.ipynb) | EV smart charging: simulation, MDP, Monte Carlo vs DQN | **`SAMPLE_MODE`** |

The files each notebook reads and writes are listed in the overwrite chain in §5.

Dependency notes:

* **01 → 02 → 03 → 08a / 08b** is the modelling chain; skipping a step leaves the next
  one reading stale files.
* **02 → 04 → 05 / 06c** is the hexagon chain.
* **06a, 06b, 07** only need notebook 02.
* **09 is fully independent** of the data pipeline — it can be run on its own at any time.
* Notebook **02 requires an internet connection**: it pulls hourly weather from the
  Open-Meteo API and POIs from OpenStreetMap via `osmnx`. Responses are cached in
  `notebooks/.cache.sqlite` / `notebooks/cache/`, so re-runs are much faster. If OSM
  Overpass is rate-limiting, just re-run the cell.

---

## 5. Sample mode / quick run

There are **three independent switches**. Only four notebooks have to be edited as shown in the table below. 
> <b>The change in 01 affects all following notebooks except 09!</b> 

Notebooks **02–07 have no switch** — they simply inherit whatever size of dataset
notebook 01 produced.

| Notebook | Variable | Committed value | Effect when on |
|---|---|---|---|
| 01 | `SAMPLE_MODE` (+ `SAMPLE_FRACTION = 0.10`, `SAMPLE_SEED = 42`) | `True` | Randomly keeps **10 % of the raw trip rows** right after loading the CSV. This 10% is then applied for the whole pipeline. |
| 08a | `QUICK_RUN` | `True` | All 18 splits trimmed to 500 rows; grid/refit subsamples 20 000/20 000/50 000 → 200/200/200; RBF & poly grids collapsed to a single parameter combination |
| 08b | `QUICK_RUN` | `True` | Train/val/test trimmed to 500 rows; hyperparameter grid cut from 32 combos to 1; `MAX_EPOCHS_PER_COMBO = 2` |
| 09 | `SAMPLE_MODE` | `False` | Reduced episode/timestep budgets for the Monte-Carlo and DQN agents |



### ⚠️ Sample mode in notebook 01 overwrites the full-data artifacts

Notebook 01's `SAMPLE_MODE` does **not** write to separate files. It only reduces the
number of records that flow through the rest of the pipeline, and every downstream
notebook keeps writing to the **same filenames**.

Consequences:

* After a sample run, **the full-data results are gone** unless you saved them. To get
  them back, set `SAMPLE_MODE = False` and re-run 01 → 02 → 03 → 04, or restore
  `data/01`, `data/02`, `data/03` from a backup.
* Numbers, figures and tables printed in the notebooks' markdown cells refer to the
  **full run**. In sample mode the reported metrics will not match the report.
* If you want to keep both, copy the `data/` and `models/` folders aside before flipping
  the switch.
---

## 7. Rendering the report

```bash
uv run quarto render report.qmd --to pdf
```

Output goes to [docs/report.pdf](docs/report.pdf). The report **embeds stored notebook
outputs** (`{{< embed notebooks/xx.ipynb#cell-label >}}`) — it does not re-execute the
notebooks. So whatever state the notebooks were last saved in (full or sample) is what
ends up in the PDF. Render the report only after a **full** run.

---

## 8. Repository layout

| Path | Contents |
|---|---|
| [notebooks/](notebooks/) | The analysis pipeline, chronological |
| [data/00](data/00/) | Raw inputs (from OneDrive, not in git) |
| `data/01` – `data/03` | Generated intermediate datasets |
| `data/08` | Model comparison figures and result csvs |
| [models/](models/) | Fitted scalers (`joblib`) |
| [sections/](sections/) | Report sections (`.qmd`) |
| [report.qmd](report.qmd) · [_quarto.yml](_quarto.yml) | Report source and Quarto config |
| [archive/](archive/) | Superseded exploratory notebooks, not part of the pipeline |
| [pyproject.toml](pyproject.toml) · [uv.lock](uv.lock) | Dependency definition and lock file |

---

## 9. Troubleshooting

* **`FileNotFoundError: ../data/00/...`** — the notebook was not started from
  `notebooks/`, or `data/00` was not unpacked from OneDrive.
* **`FileNotFoundError: ../data/02/...` or `../data/03/...`** — an earlier notebook in
  the chain has not been run yet (see the dependency notes in §4).
* **Kernel not listed / wrong packages** — re-run the `ipykernel install` command from §2
  and pick the `Python (AAA Team 5)` kernel.
* **Notebook 02 hangs or errors on OSM/weather** — network issue or Overpass rate limit;
  re-run the cell, cached responses in `notebooks/.cache.sqlite` are reused.
* **Out of memory in notebook 01** — the full CSV is ~8 GB; use `SAMPLE_MODE = True`
  (noting §5) or run on a machine with more RAM.
