# landsat-plume-proxy

Code for simulating subglacial meltwater plumes at Greenland tidewater glaciers using OMG CTD/AXCTD profiles, and evaluating Landsat SST as a plume proxy.

## Setup

### 1. Install dependencies

```bash
pip install numpy scipy xarray pandas geopandas shapely dask distributed pyDOE scikit-learn matplotlib seaborn pyyaml
```

### 2. Configure external data paths

All paths to datasets that live outside this repository are declared in `config.yaml`. Open it and fill in the paths for your system — every entry marked `/path/to/...` needs to be set before running the notebooks.

```bash
cp config.yaml config.yaml   # already committed — just edit in place
```

In-repo data (inside `data/`) is referenced by relative path and requires no configuration.

---

## Overview

The workflow:
1. Filters and pre-processes OMG CTD/AXCTD profiles for a target fjord
2. Runs a buoyant plume model (Slater 2022) in ensemble using Latin Hypercube Sampling
3. Analyses plume surface temperature and surfacing probability as a function of ocean forcing and discharge
4. Compares modelled plume surface temperature against Landsat-derived SST and reanalysis ocean data

## Repository structure

```
config.yaml                           # External data paths — fill in before running

scripts/plume_modelling/
  run_PLUME.py                        # Buoyant plume model (from Slater 2022, GRL)
  get_runoff.py                       # Reads Mankoff RACMO/MAR runoff, saves data/runoff.nc
  run_CTD-plume-ensembles.ipynb       # Main pipeline: CTD processing + ensemble plume runs
  run_CTD-plume-analysis.ipynb        # Ensemble analysis and Random Forest sensitivity
  run_and_plot_STORE-CTD-plume-ensemble.ipynb  # Store CTD plume ensemble + comparison plots

data/
  fjords_gl_sill_depths_reduced.csv   # Fjord catalogue (Mas e Braga et al. 2025)
  shapefiles/                         # Fjord boundary and model coordinate shapefiles
  reanalysis/                         # ASTE, TOPAZ4B, ORAS5 ocean reanalysis (NetCDF/GeoPackage)
  OMG_AXCTD_CTD_filtered-main.nc      # Pre-processed CTD profiles (pipeline output)
  OMG_plume_ensemble_results-main.nc  # Plume ensemble results (pipeline output)

figures/                              # All saved figures (auto-created at runtime)

plot_PlumeSST.ipynb                   # Landsat SST vs reanalysis ocean temperature
plot_Decadal_store_ctd_pst_comp.ipynb # CTD-forced plume temps vs Landsat SST time series
plot_CUBO_landsat_scenes.ipynb        # Landsat RGB scene visualisation via cubo/GEE
plot_LAKEheating.ipynb                # Solar/lake heating analysis via GEE
get_ASTEdata.ipynb                    # ASTE reanalysis point-extraction
extract_EN4.ipynb                     # EN4 depth-averaged temperature extraction
ASTE_gridcheck.ipynb                  # Grid alignment checks (ASTE, TOPAZ4B, ORAS5)
```

## Main pipeline

### 1. CTD pre-processing (`scripts/plume_modelling/run_CTD-plume-ensembles.ipynb`, sections 1–3)

- Loads all OMG CTD and AXCTD NetCDF profiles from the directory set in `config.yaml → omg_ctd_dir`
- Filters by date using the Mas e Braga et al. (2025) fjord classification CSV
- Interpolates profiles to a common 1 m depth grid and applies QC (gross range, z-score spike check)
- Saves to `data/OMG_AXCTD_CTD_filtered-main.nc`

### 2. Plume ensemble (section 4)

- Generates 40 parameter sets per CTD profile via Latin Hypercube Sampling:
  - Q₀ (subglacial discharge): 0.005–5 m³/s
  - α (entrainment coefficient): 0.05–0.15
  - z_max (grounding line depth factor): 0.6–1.4 × observed fjord depth
- Runs the buoyant plume model in parallel using Dask
- Saves results to `data/OMG_plume_ensemble_results-main.nc`

### 3. Sensitivity analysis (`run_CTD-plume-analysis.ipynb`)

- Random Forest models assess which factors control plume surfacing probability and surface temperature
- Features: discharge, entrainment, depth, ocean layer temperatures, stratification (N²)
- Figures saved to `figures/`

## Key dependencies

- `numpy`, `scipy`, `xarray`, `pandas`, `pyyaml`
- `geopandas`, `shapely`, `cartopy`
- `dask`, `distributed`
- `pyDOE` (Latin Hypercube Sampling)
- `scikit-learn` (Random Forest, GPR)
- `matplotlib`, `seaborn`

## External data requirements

Set paths for these in `config.yaml` before running:

| Key in config.yaml | Dataset | Source |
|---|---|---|
| `omg_ctd_dir` | OMG CTD/AXCTD NetCDF files | [NASA OMG](https://omg.jpl.nasa.gov/) |
| `mankoff_runoff_dir` | RACMO.nc + MAR.nc | [Mankoff et al. 2020](https://doi.org/10.5194/essd-12-2361-2020) |
| `rignot_ctd_file` | Store Glacier CTD (Aug 2010) | Rignot et al. 2010 |
| `landsat_plume_sst_dir` | Landsat plume-front SST CSVs | (contact authors) |
| `store_plume_ensemble_nc` | Pre-computed Store ensemble | (contact authors) |
| `en4_dir` | EN4 hydrographic profiles | [Met Office](https://www.metoffice.gov.uk/hadobs/en4/) |
| `topaz4_store_file` | TOPAZ4B Store-region extract | [Copernicus Marine](https://marine.copernicus.eu/) |
| `oras5_store_file` | ORAS5 Store-region extract | [ECMWF](https://www.ecmwf.int/en/research/climate-reanalysis/ocean-reanalysis) |
| `aste_grid_file` | ASTE MITgcm grid (GRID.0014.nc) | [CRIOS Portal](https://crios-ut.github.io/ASTE/) |

## References

- Slater, D. A. et al. (2022). Localised plume forcing of Greenland outlet glaciers. *GRL*. — plume model basis
- Mas e Braga, M. et al. (2025). — fjord classification CSV
- Mankoff, K. D. et al. (2020). Greenland liquid water discharge. *ESSD*.
