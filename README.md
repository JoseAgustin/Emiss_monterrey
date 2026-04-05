# Emiss_monterrey — WRF-Chem Emission Inventory for the Monterrey Metropolitan Area

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Language: Fortran](https://img.shields.io/badge/Language-Fortran%2090-orange.svg)]()
[![WRF-Chem](https://img.shields.io/badge/Model-WRF--Chem-lightblue.svg)](https://ruc.noaa.gov/wrf/wrf-chem/)
[![Region](https://img.shields.io/badge/Region-Monterrey%20%7C%20Saltillo-green.svg)]()
[![Institution](https://img.shields.io/badge/Institution-CCA%20UNAM-red.svg)](https://www.atmosfera.unam.mx/)

> A modular Fortran 90 pipeline that converts the **Monterrey Metropolitan Area** (Zona Metropolitana de Monterrey, ZMM) emission inventory — including Saltillo — into **WRF-Chem `wrfchemi_*` NetCDF files**. It covers area, mobile, and point sources; applies hourly temporal profiles; speciates VOC into multiple chemical mechanisms (RADM2, CBM05, SAPRC99, RACM2); speciates PM₂.₅; and writes the final gridded emission files ready for WRF-Chem simulations.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Repository Structure](#repository-structure)
- [Pipeline Description](#pipeline-description)
  - [Step 01 — datos: Input inventory data](#step-01--datos-input-inventory-data)
  - [Step 02 — aemis: Area source spatial disaggregation](#step-02--aemis-area-source-spatial-disaggregation)
  - [Step 03 — movilspatial: Mobile source spatial disaggregation](#step-03--movilspatial-mobile-source-spatial-disaggregation)
  - [Step 04 — temis: Area source temporal profiles](#step-04--temis-area-source-temporal-profiles)
  - [Step 05 — semisM: Mobile source spatial emission](#step-05--semism-mobile-source-spatial-emission)
  - [Step 06 — temisM: Mobile source temporal profiles](#step-06--temism-mobile-source-temporal-profiles)
  - [Step 07 — puntual: Point source temporal profiles](#step-07--puntual-point-source-temporal-profiles)
  - [Step 08 — spec: VOC speciation](#step-08--spec-voc-speciation)
  - [Step 09 — pm25spec: PM₂.₅ speciation](#step-09--pm25spec-pm₂₅-speciation)
  - [Step 10 — storage: WRF-Chem NetCDF output](#step-10--storage-wrf-chem-netcdf-output)
  - [Step 12 — cmaq: CMAQ output (optional)](#step-12--cmaq-cmaq-output-optional)
- [Compilation](#compilation)
- [Running the Pipeline](#running-the-pipeline)
- [Chemical Mechanisms Supported](#chemical-mechanisms-supported)
- [Output Files](#output-files)
- [References](#references)

---

## Overview

The pipeline processes the official emission inventory of the **Zona Metropolitana de Monterrey (ZMM)** — the second-largest metropolitan area in Mexico — together with **Saltillo**, included as an additional city within the model domain. The inventory covers three source categories:

- **Area sources** — residential, commercial, and service-sector fugitive emissions disaggregated spatially over the model grid using surrogate fields.
- **Mobile sources** — vehicular emissions on highways and urban streets, with speed-dependent emission factors and cold-start corrections.
- **Point sources** — industrial stacks with individual coordinates, stack parameters, and hourly temporal profiles.

The pipeline applies:

1. **Spatial disaggregation** — distributes annual municipal totals onto the WRF-Chem grid using spatial surrogates.
2. **Temporal disaggregation** — applies day-of-week and hour-of-day profiles per source category and compound.
3. **VOC speciation** — maps total VOC to individual species for four chemical mechanisms.
4. **PM₂.₅ speciation** — distributes total PM₂.₅ into aerosol species (EC, OC, sulphate, nitrate, other).
5. **NetCDF output** — writes WRF-Chem `wrfchemi_d01_<date>_<HH>:00:00` files per mechanism.

---

## Requirements

| Component | Notes |
|---|---|
| Fortran compiler | Intel `ifort` ≥ 17 (used in `compila.sh`); `gfortran` ≥ 8 also supported via the Makefile |
| NetCDF-Fortran | Required only by the `10_storage` programs that write NetCDF output; set `$NETCDF` |
| GNU Make | For Makefile-based compilation across all subdirectories |
| Bash | For the orchestration scripts (`emis_2014.sh`, `feb_2019.sh`, `abril_2014.sh`) |

---

## Repository Structure

```
Emiss_monterrey/
│
├── 01_datos/                   # Input inventory data (annual totals, surrogate files)
├── 02_aemis/                   # Area source spatial disaggregation
│   └── area_espacial.f90       →  ASpatial.exe
├── 03_movilspatial/            # Mobile source spatial disaggregation
│   ├── suma_carretera.f90      →  carr.exe     (highway aggregation)
│   ├── suma_vialidades.f90     →  vial.exe     (urban street aggregation)
│   └── agrega.f90              →  agrega.exe   (merge highway + street)
├── 04_temis/                   # Area source hourly temporal profiles
│   └── atemporal.f90           →  Atemporal.exe
├── 05_semisM/                  # Mobile source spatial emission totals
│   └── movil_spatial.f90       →  MSpatial.exe
├── 06_temisM/                  # Mobile source hourly temporal profiles
│   └── movil_temp.f90          →  Mtemporal.exe
├── 07_puntual/                 # Point source temporal profiles
│   └── t_puntal.f90            →  Puntual.exe
├── 08_spec/                    # VOC speciation (area, mobile, point)
│   ├── agg_a.f90               →  spa.exe      (area speciation)
│   ├── agg_m.f90               →  spm.exe      (mobile speciation)
│   └── agg_p.f90               →  spp.exe      (point speciation)
├── 09_pm25spec/                # PM₂.₅ speciation (area, mobile, point)
│   ├── pm25_speci_a.f90        →  spm25a.exe
│   ├── pm25_speci_m.f90        →  spm25m.exe
│   └── pm25_speci_p.f90        →  spm25p.exe
├── 10_storage/                 # WRF-Chem NetCDF wrfchemi output
│   ├── g_radm_2014.f90         →  radm2.exe    (RADM2 mechanism, 2014 IE)
│   ├── g_2014_racm.f90         →  racm2.exe    (RACM2 mechanism)
│   ├── g_cbm5_2014.f90         →  cbm5.exe     (CBM05 mechanism)
│   ├── g_saprc_2014.f90        →  saprc.exe    (SAPRC99 mechanism)
│   └── g_radm_2019.f90         →  radm2019.exe (RADM2, 2019 IE update)
├── 12_cmaq/                    # CMAQ-format output (optional)
│
├── Makefile                    # Top-level Makefile — compiles all subdirectories
├── compila.sh                  # Alternative compilation script using ifort
├── emis_2014.sh                # Main run script for the 2014 emission inventory
├── feb_2019.sh                 # Run script for February 2019 episode
├── abril_2014.sh               # Run script for April 2014 episode
└── README.md                   # This file
```

---

## Pipeline Description

The emission processing pipeline follows a fixed sequence of steps. Each step corresponds to a numbered subdirectory and produces intermediate files consumed by the next step.

```
01_datos          ← Annual emission inventory (area, mobile, point)
     │
     ▼
02_aemis          ASpatial.exe   → Area spatial totals per grid cell
03_movilspatial   carr.exe
                  vial.exe       → Mobile spatial totals per grid cell
                  agrega.exe
     │
     ▼
04_temis          Atemporal.exe  → Area hourly emissions (area × temporal profile)
05_semisM         MSpatial.exe   → Mobile spatial emission totals
06_temisM         Mtemporal.exe  → Mobile hourly emissions (mobile × temporal profile)
07_puntual        Puntual.exe    → Point source hourly emissions
     │
     ▼
08_spec           spa.exe  spm.exe  spp.exe   → VOC speciation per mechanism
09_pm25spec       spm25a.exe  spm25m.exe  spm25p.exe  → PM₂.₅ speciation
     │
     ▼
10_storage        radm2.exe / racm2.exe / cbm5.exe / saprc.exe
                  → wrfchemi_d01_YYYY-MM-DD_HH:00:00 (NetCDF, WRF-Chem ready)
```

### Step 01 — datos: Input inventory data

The `01_datos/` directory holds the annual emission inventory tables for the ZMM, including:

- Annual municipal totals by source category and pollutant (CO, NOx, SO₂, PM₂.₅, PM₁₀, VOC, NH₃).
- Spatial surrogate files relating census or land-use units to the WRF-Chem grid cells.
- Hourly and day-of-week temporal profile tables per source category.
- Highway and urban street segment activity data for mobile sources.
- Point source catalogue with stack coordinates, heights, diameters, exit velocities, and temperatures.

### Step 02 — aemis: Area source spatial disaggregation

**Program:** `area_espacial.f90` → `ASpatial.exe`

Reads the annual area-source emission totals by municipality and disaggregates them spatially onto the WRF-Chem model grid using surrogate fields (population density, road length, or land-use fraction, depending on the source category). Produces gridded annual area-source emission totals per compound.

### Step 03 — movilspatial: Mobile source spatial disaggregation

**Programs:**
- `suma_carretera.f90` → `carr.exe`: Aggregates emissions from highway segments onto the grid.
- `suma_vialidades.f90` → `vial.exe`: Aggregates emissions from urban street segments onto the grid.
- `agrega.f90` → `agrega.exe`: Merges highway and urban mobile emission grids into a single mobile-source total per cell.

### Step 04 — temis: Area source temporal profiles

**Program:** `atemporal.f90` → `Atemporal.exe`

Reads the date from `fecha.txt` (month and day), selects the appropriate day-type temporal profile (working day, Saturday, or Sunday), and applies it to the gridded area-source annual totals to produce **hourly area-source emission fields** for that date. Reads the annual data file (e.g., `anio2014.csv`) linked via `ln -sf`.

**Input `fecha.txt` format:**
```
4   ! month  (1 = January ... 12 = December)
22  ! day in the month
```

### Step 05 — semisM: Mobile source spatial emission

**Program:** `movil_spatial.f90` → `MSpatial.exe`

Computes gridded daily mobile-source emission totals per compound by applying speed-dependent EPA emission factors to the vehicle activity data (number of vehicles, speed, road type) for each grid cell.

### Step 06 — temisM: Mobile source temporal profiles

**Program:** `movil_temp.f90` → `Mtemporal.exe`

Applies hourly temporal profiles to the mobile-source daily emission totals, producing **24-hour mobile emission fields** for each compound for the date specified in `fecha.txt`.

### Step 07 — puntual: Point source temporal profiles

**Program:** `t_puntal.f90` → `Puntual.exe`

Reads the point source catalogue (individual stack locations, annual emission rates) and applies hourly temporal profiles to produce **24-hour point-source emission fields** on the model grid.

### Step 08 — spec: VOC speciation

**Programs:**
- `agg_a.f90` → `spa.exe` (area sources)
- `agg_m.f90` → `spm.exe` (mobile sources)
- `agg_p.f90` → `spp.exe` (point sources)

Distributes total VOC emissions into individual chemical species using the speciation profile selected via a symbolic link:

```bash
ln -sf profile_radm2.csv   profile_mech.csv   # RADM2
ln -sf profile_cbm05.csv   profile_mech.csv   # CBM05
ln -sf profile_saprc99.csv profile_mech.csv   # SAPRC99
ln -sf profile_racm2.csv   profile_mech.csv   # RACM2
```

Each mechanism's speciation is run in sequence for all three source types, so speciated outputs exist for all mechanisms before the storage step.

### Step 09 — pm25spec: PM₂.₅ speciation

**Programs:**
- `pm25_speci_a.f90` → `spm25a.exe` (area)
- `pm25_speci_m.f90` → `spm25m.exe` (mobile)
- `pm25_speci_p.f90` → `spm25p.exe` (point)

Distributes total PM₂.₅ into aerosol component species (elemental carbon EC, organic carbon OC, sulphate, nitrate, and unspeciated PM₂.₅) required by WRF-Chem aerosol modules.

### Step 10 — storage: WRF-Chem NetCDF output

**Programs:**

| Executable | Source | Chemical mechanism | Inventory year |
|---|---|---|---|
| `radm2.exe` | `g_radm_2014.f90` | RADM2 | 2014 |
| `racm2.exe` | `g_2014_racm.f90` | RACM2 | 2014 |
| `cbm5.exe` | `g_cbm5_2014.f90` | CBM05 | 2014 |
| `saprc.exe` | `g_saprc_2014.f90` | SAPRC99 | 2014 |
| `radm2019.exe` | `g_radm_2019.f90` | RADM2 | 2019 |

Each program assembles the hourly speciated emission fields (from steps 08 and 09) for all three source types and writes the final **`wrfchemi_d01_YYYY-MM-DD_HH:00:00`** NetCDF file, one per mechanism per hour, in the format expected by WRF-Chem (`io_style_emissions = 1`).

The NetCDF files include WRF-Chem global attributes (domain dimensions, projection, chemical mechanism) and individual emission variables in units of mol km⁻² hr⁻¹ (gases) and µg m⁻² s⁻¹ (aerosols).

### Step 12 — cmaq: CMAQ output (optional)

The `12_cmaq/` directory contains programs to write emission output in CMAQ I/O API format, for use with the CMAQ air quality model instead of WRF-Chem.

---

## Compilation

### Option A — Makefile (gfortran, recommended)

Compiles all subdirectories in sequence using the local Makefiles:

```bash
git clone https://github.com/JoseAgustin/Emiss_monterrey.git
cd Emiss_monterrey
make
```

For the `10_storage` programs, set the NetCDF environment variable before compiling:

```bash
export NETCDF=/path/to/netcdf   # e.g. /usr/local or /opt/netcdf
make
```

To rebuild from scratch:

```bash
make clean
make
```

### Option B — compila.sh (ifort, AVX)

The script `compila.sh` uses Intel Fortran with `-axAVX` vectorisation for maximum performance on modern CPUs:

```bash
export NETCDF=/path/to/netcdf
bash compila.sh
```

This compiles and immediately runs the spatial disaggregation steps (`02_aemis`, `03_movilspatial`, `05_semisM`) so that their outputs are ready when the temporal and speciation steps are executed later.

---

## Running the Pipeline

After compilation, use one of the provided run scripts to process a specific date or episode. The scripts loop over a range of days, executing all processing steps in the correct parallel/sequential order.

### Single date or episode — `emis_2014.sh`

Edit the script to set the target month, day range, and inventory year link:

```bash
# Inside emis_2014.sh:
mes=4              # April
dia=22             # Start day
# while [ $dia -le 22 ]  →  process only day 22; increase upper bound for multi-day

ln -sf anio2014.csv.org anio2014.csv   # Select 2014 inventory
```

Then run:

```bash
bash emis_2014.sh
```

**Execution sequence per day (from `emis_2014.sh`):**

```
1. Write fecha.txt (month + day)
2. Atemporal.exe    → area hourly emissions          (background)
3. Puntual.exe      → point hourly emissions          (foreground, blocks)
4. Mtemporal.exe    → mobile hourly emissions         (background)
   [wait for all three]
5. spm25p.exe  spm25m.exe  spm25a.exe  → PM₂.₅ speciation  (parallel)
6. For each mechanism (RADM2, CBM05, SAPRC99, RACM2):
   ln -sf profile_<mech>.csv  profile_mech.csv
   spm.exe   spp.exe   spa.exe              → VOC speciation (parallel)
7. radm2.exe  saprc.exe  cbm5.exe  racm2.exe → write wrfchemi NetCDF files
```

### 2019 episode — `feb_2019.sh`

```bash
bash feb_2019.sh
```

Uses the 2019 emission inventory and produces `wrfchemi` files using the updated RADM2 speciation profiles for that year.

### April 2014 episode — `abril_2014.sh`

```bash
bash abril_2014.sh
```

---

## Chemical Mechanisms Supported

| Mechanism | `chem_opt` (WRF-Chem) | Executable | VOC profile file |
|---|---|---|---|
| RADM2 | 2 | `radm2.exe` / `radm2019.exe` | `profile_radm2.csv` |
| CBM05 | 5 | `cbm5.exe` | `profile_cbm05.csv` |
| SAPRC99 | 6 | `saprc.exe` | `profile_saprc99.csv` |
| RACM2 | 11 | `racm2.exe` | `profile_racm2.csv` |

To use the generated files in a WRF-Chem simulation, set in `namelist.input`:

```fortran
&chem
  chem_opt           = 2           ! or 5, 6, 11 depending on mechanism
  emiss_opt          = 3           ! use wrfchemi files
  io_style_emissions = 1           ! hourly files (00z and 12z convention)
/
```

Then link the output files to the WRF run directory:

```bash
ln -sf wrfchemi_d01_YYYY-MM-DD_HH:00:00 wrfchemi_d01_YYYY-MM-DD_HH:00:00
```

---

## Output Files

Each run of the `10_storage` programs produces hourly NetCDF emission files:

```
wrfchemi_d01_YYYY-MM-DD_00:00:00   (00z–12z file, or hour 00)
wrfchemi_d01_YYYY-MM-DD_12:00:00   (12z–24z file, or hour 12)
```

The files contain gridded emission variables in the WRF-Chem `wrfchemi` format, including at minimum:

| Variable group | Examples | Units |
|---|---|---|
| Gaseous species | `E_CO`, `E_NO`, `E_SO2`, `E_NH3` | mol km⁻² hr⁻¹ |
| VOC species | `E_ETH`, `E_HC3`, `E_HC5`, `E_HC8`, `E_OL2`, `E_OLT`, `E_OLI`, `E_TOL`, `E_XYL`, `E_KET`, `E_ALD`, `E_HCHO`, `E_ORA2` (RADM2) | mol km⁻² hr⁻¹ |
| Aerosol species | `E_PM25I`, `E_PM25J`, `E_ECI`, `E_ECJ`, `E_ORGI`, `E_ORGJ`, `E_SO4I`, `E_SO4J`, `E_NO3I`, `E_NO3J` | µg m⁻² s⁻¹ |

---

## References

- Grell, G. A., Peckham, S. E., Schmitz, R., McKeen, S. A., Frost, G. J., Skamarock, W. C., & Eder, B. K. (2005). Fully coupled "online" chemistry within the WRF model. *Atmospheric Environment*, **39**, 6957–6975. https://doi.org/10.1016/j.atmosenv.2005.04.027

- Stockwell, W. R., Middleton, P., Chang, J. S., & Tang, X. (1990). The second generation regional acid deposition model chemical mechanism for regional air quality modeling. *Journal of Geophysical Research*, **95**(D10), 16343–16367. https://doi.org/10.1029/JD095iD10p16343

- Zavala, M., Herndon, S. C., Wood, E. C., Onasch, T. B., Knighton, W. B., Kolb, C. E., Molina, L. T., & Molina, M. J. (2006). Evaluation of mobile emissions contributions to Mexico City's emissions inventory using on-road and cross-road emission measurements and ambient data. *Atmospheric Chemistry and Physics*, **6**, 3691–3710. https://doi.org/10.5194/acp-6-3691-2006

- Secretaría de Medio Ambiente y Recursos Naturales (SEMARNAT) / INECC (2014). *Inventario Nacional de Emisiones de México*. https://www.gob.mx/inecc/acciones-y-programas/inventario-nacional-de-emisiones-de-gases-y-compuestos-de-efecto-invernadero

---

*README last updated: March 2026*
