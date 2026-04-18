# CHAZ User Guide
## Columbia HAZard Model — Tropical Cyclone Downscaling

---

## Table of Contents

1. [Overview](#1-overview)
2. [Directory Structure](#2-directory-structure)
3. [Input Data Requirements](#3-input-data-requirements)
4. [Configuration: Namelist.py](#4-configuration-namelistpy)
5. [Running CHAZ: Step-by-Step](#5-running-chaz-step-by-step)
6. [Output Files](#6-output-files)
7. [Special Cases](#7-special-cases)
8. [Working with Outputs](#8-working-with-outputs)
9. [Dependencies](#9-dependencies)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Overview

CHAZ is a statistical-dynamical tropical cyclone (TC) downscaling model. Given large-scale atmospheric fields from a GCM (e.g., SPEAR, CMIP6 models), it generates synthetic TC tracks and intensities via three sequential steps:

1. **Genesis** — Seeds TCs probabilistically using a precomputed TC Genesis Index (TCGI)
2. **Track** — Advects each TC with a Beta-and-Advection Model (BAM)
3. **Intensity** — Evolves TC intensity via multi-variate regression against observed TC statistics

The model accounts for GCM mean-state biases via a bias-correction step during preprocessing.

**Key parameters (defaults):**
- CHAZ ensemble members per year: 10
- Intensity realizations per track: 40
- TC survival rate: 0.78
- Time step: 6 hours

---

## 2. Directory Structure

```
for_hiro/
├── code/
│   ├── chaz_src/              # Source code (do not modify directly)
│   │   ├── src/               # Preprocessing scripts (with daily wind data)
│   │   ├── src_nodaily/       # Preprocessing scripts (monthly-only data)
│   │   ├── src_nodaily_jzhuorevised/   # Modified version by J. Zhuo
│   │   ├── run/               # Workflow orchestration
│   │   │   ├── main.sh        # PRIMARY ENTRY POINT
│   │   │   ├── Namelist.py    # Configuration template
│   │   │   ├── run_CHAZ.py    # Multi-member runner
│   │   │   ├── run_pre.py     # Multi-member preprocessor
│   │   │   └── run_netcdf.py  # Batch pickle → NetCDF converter
│   │   ├── chaz_para/         # Static support data (landmask, obs TC stats)
│   │   ├── input_data_table/  # GCM data catalogs (CSV)
│   │   └── pyclee/            # TC utility library
│   │
│   └── chaz_work/             # Working directory (created at runtime)
│       └── {model}/{member}/
│           ├── pre/           # Preprocessing outputs
│           └── wdir/          # Downscaling outputs
│
└── data/
    └── output_tcgi/           # Precomputed TCGI and PI matrices
        ├── TCGI_CRH_PI/
        │   └── {model}/{forcing}/{member}/
        │       └── TCGI_CRH_PI_{model}_{forcing}_{member}_{YYYY}.mat
        └── {model}/{forcing}/{member}/
            └── PI_{model}_{forcing}_{member}_{YYYY}.mat
```

---

## 3. Input Data Requirements

### 3.1 TCGI Matrices (required)

Precomputed TC Genesis Index matrices. CHAZ does **not** compute these — they must exist before running CHAZ (use the `tcgi_src/` workflow to generate them).

- **Location:** `data/output_tcgi/TCGI_CRH_PI/{model}/{forcing}/{member}/`
- **Format:** MATLAB `.mat` files
- **Naming:** `TCGI_CRH_PI_{model}_{forcing}_{member}_{YYYY}.mat`
- **Content:** Monthly genesis probability (12 × nlat × nlon)

### 3.2 PI Matrices (required)

Potential Intensity matrices, also precomputed via `tcgi_src/`.

- **Location:** `data/output_tcgi/{model}/{forcing}/{member}/`
- **Naming:** `PI_{model}_{forcing}_{member}_{YYYY}.mat`

### 3.3 GCM Atmospheric Data (required)

Monthly mean fields used during preprocessing:

| Variable | Levels | Description |
|----------|--------|-------------|
| `ua` | 250, 850 hPa | Zonal wind |
| `va` | 250, 850 hPa | Meridional wind |
| `hur` | 500, 700, 850 hPa | Relative humidity |

Pointed to via a CSV catalog file (see Section 3.5).

### 3.4 Daily Wind Data (optional)

Daily `ua` and `va` at 250 and 850 hPa. Required only if you want CHAZ to compute wind covariance matrices (`calWind=True`). If unavailable, precomputed `A_YYYYMM.nc` files must be provided (see Section 7.1).

### 3.5 Data Catalog CSV Files

CHAZ reads GCM data locations from CSV catalog files in `code/chaz_src/input_data_table/`.

**Available catalogs:**

| File | Purpose |
|------|---------|
| `SPEAR_monthly.csv` | SPEAR model monthly data (local paths) |
| `SPEAR_daily.csv` | SPEAR model daily data (local paths) |
| `cmip6_historical_opendap.csv` | CMIP6 historical monthly (OPeNDAP URLs) |
| `cmip6_historical_opendap_daily.csv` | CMIP6 historical daily (OPeNDAP URLs) |

**CSV column format:**
```
activity_id, institution_id, source_id, experiment_id, member_id, table_id, variable_id, URL, ...
```

To add a new model, create a new CSV file with the appropriate local file paths or OPeNDAP URLs, then update `monthlycsv` (and `dailycsv`) in `Namelist.py`.

### 3.6 Static Support Data (chaz_para/)

These files ship with the code and should not be modified:

| File | Description |
|------|-------------|
| `landmask.nc` | 0.25° global land-sea mask (0=ocean) |
| `bt_atl.pik`, `bt_wnp.pik`, etc. | Observed TC statistics by basin |
| `bt_global_predictors.nc` | Global predictor climatology |
| `result_w.nc` | Intensity regression coefficients (over water) |
| `result_l.nc` | Intensity regression coefficients (over land) |

---

## 4. Configuration: Namelist.py

The `Namelist.py` file (template in `run/`) controls all CHAZ settings. `main.sh` copies this template to the working directories and modifies it automatically via `sed`. You can also edit it manually for custom runs.

### 4.1 Experiment Settings

```python
Model          = 'SPEAR'            # GCM name (must match CSV catalog source_id)
ENS            = 'r1i1'             # Ensemble member ID
TCGIinput      = 'TCGI_CRH_PI'     # TCGI variant to use
PImodelname    = 'SPEAR'            # Model name for PI lookup
Experiment_id  = 'historical'       # 'historical' or 'ssp585'
Year1          = 1965               # Start year
Year2          = 2014               # End year
```

### 4.2 Path Settings

```python
monthlycsv   = '/path/to/input_data_table/SPEAR_monthly.csv'
dailycsv     = '/path/to/input_data_table/SPEAR_daily.csv'
landmaskfile = '/path/to/chaz_para/landmask.nc'
ipath        = '/path/to/chaz_para/'
tcgipath     = '/path/to/data/output_tcgi/'
root         = '/path/to/for_hiro'
```

### 4.3 Model Parameters

```python
uBeta         = -1.5    # Zonal beta-drift parameter for track model
vBeta         =  2.0    # Meridional beta-drift parameter
survivalrate  =  0.78   # TC survival rate (fraction of seeded TCs that persist)
CHAZ_ENS_0   =  0       # Index of first CHAZ ensemble member
CHAZ_ENS     =  10      # Number of CHAZ track ensembles per year
CHAZ_Int_ENS =  40      # Number of stochastic intensity realizations per track
```

### 4.4 Processing Flags

These are set automatically by `main.sh`, but can be set manually:

```python
# --- Preprocessing phase (path_pre) ---
runPreprocess  = True   # Set True in path_pre, False in path_wdir
calWind        = True   # Compute wind covariance from daily data?
calpreProcess  = True   # Interpolate GCM predictors to TC grid?
calA           = True   # Compute Cholesky decomposition of covariance?

# --- Downscaling phase (path_wdir) ---
runCHAZ  = False        # Set False in path_pre, True in path_wdir
calGen   = True         # Run genesis module?
calBam   = True         # Run track module?
calInt   = True         # Run intensity module?
```

---

## 5. Running CHAZ: Step-by-Step

### Step 1: Verify Inputs

Before running, confirm that for each year in `[Year1, Year2]`:
- `TCGI_CRH_PI_{model}_{forcing}_{member}_{YYYY}.mat` exists in `data/output_tcgi/`
- `PI_{model}_{forcing}_{member}_{YYYY}.mat` exists
- GCM data is accessible via the CSV catalog (local path or OPeNDAP)

### Step 2: Configure main.sh

Open `code/chaz_src/run/main.sh` and set the top-of-file variables:

```bash
root="/scratch/gpfs/GEOCLIM/jzhuo/analysis/for_hiro"
root_work="${root}/code/chaz_work"
pth_chaz_src="${root}/code/chaz_src"
root_PI="${root}/data/output_tcgi"

# --- Experiment settings ---
model="SPEAR"           # Model name
imem="r1i1"             # Ensemble member
forcing="hist"          # "hist" or "ssps585"
year1=1965              # Start year
year2=2014              # End year
TCGI="TCGI_CRH_PI"     # TCGI variant

# --- Data resolution ---
reso="Amon"             # "Amon" (monthly only) or "day" (daily available)

# If reso="Amon", provide path to pre-computed A matrices:
path_windcov="/path/to/A_matrices"
```

### Step 3: Run the Workflow

```bash
cd /scratch/gpfs/GEOCLIM/jzhuo/analysis/for_hiro/code/chaz_src/run
bash main.sh
```

`main.sh` performs the following automatically:

#### Phase A — Setup
- Creates `chaz_work/{model}/{member}/pre/` and `.../wdir/`
- Copies the appropriate source directory (`src/` or `src_nodaily/`) into `pre/` and `wdir/`
- Creates symbolic links to TCGI, PI, and (if available) wind covariance files

#### Phase B — Preprocessing (`pre/`)
- Modifies `Namelist.py` for preprocessing (`runPreprocess=True`, `runCHAZ=False`)
- Runs `python CHAZ.py` → interpolates GCM data to TC grid, optionally computes wind covariance
- Runs `python getMeanStd.py` → computes bias-correction statistics

#### Phase C — Downscaling (`wdir/`)
- Copies/links preprocessing outputs to `wdir/`
- Modifies `Namelist.py` for downscaling (`runPreprocess=False`, `runCHAZ=True`)
- Runs `python CHAZ.py` → genesis, track, intensity
- Runs `python rev.pik2netcdf_merge_2100.py` → converts pickle files to NetCDF

### Step 4: Manual Run (without main.sh)

If you prefer to run each phase manually:

**Preprocessing:**
```bash
# Set up pre/ directory
mkdir -p $root_work/$model/$imem/pre
cp $pth_chaz_src/src_nodaily/* $root_work/$model/$imem/pre/
# Link TCGI and PI matrices, then:
cd $root_work/$model/$imem/pre
# Edit Namelist.py: set runPreprocess=True, runCHAZ=False, calWind=False, calA=False
python CHAZ.py
python getMeanStd.py
```

**Downscaling:**
```bash
mkdir -p $root_work/$model/$imem/wdir
cp $pth_chaz_src/src_nodaily/* $root_work/$model/$imem/wdir/
# Link outputs from pre/ to wdir/
cd $root_work/$model/$imem/wdir
# Edit Namelist.py: set runPreprocess=False, runCHAZ=True
python CHAZ.py
python rev.pik2netcdf_merge_2100.py
```

---

## 6. Output Files

### 6.1 Preprocessing Outputs (`pre/`)

| File | Description |
|------|-------------|
| `{YYYY}_{ENS}.nc` | Interpolated monthly predictors (ua250, va250, ua850, va850, hur, PI) for each year |
| `Cov_{YYYY}.nc` | Monthly wind covariance matrices (only if `calWind=True`) |
| `A_{YYYY}{MM}.nc` | Cholesky decomposition of wind covariance (one per month; only if `calA=True`) |
| `coefficient_meanstd.nc` | Mean and std for intensity bias correction |
| `trackPredictorsbt_obs.pik` | Observed TC statistics (copied/linked from `chaz_para/`) |

### 6.2 Downscaling Outputs (`wdir/`)

| File | Description |
|------|-------------|
| `bt_stochastic_det{YYYY}_ens{EEE}.pik` | TC tracks and intensities for year YYYY, ensemble EEE (binary pickle) |
| `netcdf/*.nc` | Human-readable NetCDF version of the above (created by `rev.pik2netcdf_merge_2100.py`) |

**One `.pik` file is produced per year per ensemble member.** With defaults (`CHAZ_ENS=10`, 50-year run), expect ~500 pickle files.

### 6.3 Pickle File Contents

```python
import pickle
with open('bt_stochastic_det1995_ens000.pik', 'rb') as f:
    bt = pickle.load(f)

bt.StormLon      # (ntime, nstorms) — longitude positions
bt.StormLat      # (ntime, nstorms) — latitude positions
bt.StormMwspd    # (ntime, nstorms) — maximum wind speed (m/s)
bt.StormMslp     # (ntime, nstorms) — minimum sea-level pressure (hPa)
bt.Time          # (ntime, nstorms) — datetime objects
bt.determin      # (ntime, nstorms) — deterministic intensity
bt.StormYear     # (nstorms,)       — genesis year of each storm
```

---

## 7. Special Cases

### 7.1 Monthly-Only Data (No Daily Wind)

When daily wind data are unavailable (e.g., SPEAR), set `reso="Amon"` in `main.sh`. This disables `calWind` and `calA`, but **precomputed `A_YYYYMM.nc` files must be provided** and linked into `pre/`.

```bash
reso="Amon"
path_windcov="/path/to/existing/A_matrices"
# main.sh will run: ln -sf $path_windcov/A*.nc $path_pre/
```

If you have no `A_YYYYMM.nc` files at all, contact the CHAZ developers for a climatological default.

### 7.2 Running Multiple Ensemble Members

Use `run_pre.py` and `run_CHAZ.py` in `run/` to loop over members:

```bash
python run_pre.py    # Preprocessing for all members listed in SearchMember.py
python run_CHAZ.py   # Downscaling for all preprocessed members
```

`SearchMember.py` queries the CSV catalogs to find ensemble members with complete data across all required variables and years.

### 7.3 Future Scenarios (SSP585)

Change `forcing` and `Experiment_id` in `main.sh` and `Namelist.py`:

```bash
forcing="ssps585"
# In Namelist.py:
Experiment_id = 'ssp585'
```

Ensure the CSV catalog contains the SSP data (e.g., a `cmip6_ssp585_opendap.csv`).

### 7.4 Converting All Pickles to NetCDF

```bash
cd $root_work/$model/$imem/wdir
python run_netcdf.py    # Batch-converts all *.pik in wdir to netcdf/
```

---

## 8. Working with Outputs

### 8.1 Read Pickle Files

```python
import pickle, numpy as np

with open('bt_stochastic_det1995_ens000.pik', 'rb') as f:
    bt = pickle.load(f)

# Filter to a specific year
yr = 1995
istorms = np.where(bt.StormYear == yr)[0]
lons = bt.StormLon[:, istorms]
lats = bt.StormLat[:, istorms]
wspd = bt.StormMwspd[:, istorms]
```

### 8.2 Assign Storms to Basins

```python
import sys
sys.path.insert(0, '/path/to/chaz_src/pyclee')
from pygplib3.util import getbasinMap, defineBasin

xbin, ybin, basinMap = getbasinMap()
basin = defineBasin(genesis_lon, genesis_lat, xbin, ybin, basinMap)
# basin: 1=ATL, 2=ENP, 3=WNP, 4=NI, 5=SI, 6=AUS, 7=SPC
```

### 8.3 Read NetCDF Outputs

```python
import xarray as xr
ds = xr.open_dataset('wdir/netcdf/chaz_tracks_1965-2014.nc')
print(ds)
```

---

## 9. Dependencies

**Python packages:**
```
numpy, pandas, scipy, xarray, netCDF4, dask, matplotlib, cartopy, statsmodels
```

**Internal library (no install needed):**
```
pyclee/  — added to PYTHONPATH automatically by main.sh
```

**External data tools:**
- MATLAB `.mat` file reading: `scipy.io.loadmat`
- OPeNDAP access: `netCDF4` or `xarray` with remote URL support

**PYTHONPATH (set in main.sh):**
```bash
export PYTHONPATH="${root}/code/chaz_src/pyclee:${PYTHONPATH}"
```

If running manually, add this export to your shell before calling `python CHAZ.py`.

---

## 10. Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| `FileNotFoundError: TCGI_*.mat` | TCGI not computed or wrong path | Run `tcgi_src/` workflow first; check `tcgipath` in Namelist.py |
| `FileNotFoundError: PI_*.mat` | PI not computed | Same as above |
| `KeyError: 'ua'` in preprocess | Variable name mismatch in GCM file | Check CSV catalog variable names match actual NetCDF variable names |
| `A_YYYYMM.nc not found` | Monthly-only run but no A files linked | Provide precomputed A matrices or use a model with daily data |
| Empty output pickle | `survivalrate` too low, or TCGI matrix all zeros | Check TCGI values; increase `survivalrate` for testing |
| OPeNDAP timeout | Network issue accessing remote data | Switch to local data; update CSV to local paths |
| `ImportError: pygplib3` | PYTHONPATH not set | `export PYTHONPATH=".../chaz_src/pyclee:$PYTHONPATH"` |

---

## Quick-Start Checklist

- [ ] TCGI matrices exist for all years in `data/output_tcgi/TCGI_CRH_PI/`
- [ ] PI matrices exist in `data/output_tcgi/`
- [ ] GCM data accessible; CSV catalog file updated
- [ ] If monthly-only: `A_YYYYMM.nc` files available at `path_windcov`
- [ ] `main.sh` top variables configured (`model`, `imem`, `forcing`, `year1`, `year2`, `reso`)
- [ ] PYTHONPATH includes `pyclee/`
- [ ] Run `bash main.sh`
- [ ] Check `chaz_work/{model}/{member}/wdir/` for `.pik` output files
- [ ] Run `rev.pik2netcdf_merge_2100.py` for NetCDF output
