---
name: summa
description: >
  Self-contained guide to SUMMA (Structure for Unifying Multiple Modeling
  Alternatives), the process-based hydrologic modeling framework from
  NCAR/UW/USask. Covers the Fortran source layout, NetCDF-driven file
  manager, the 40+ model decisions that select among physics options,
  installation via Makefile or CMake, the Reynolds Mountain East and
  related case studies, the SUNDIALS-based solver branch, output
  post-processing, and contributing through pull requests. Progressive
  disclosure: start in this file for routing, drill into reference/ for
  depth.
version: 0.1.0
tags:
  - earth-science
  - hydrology
  - land-surface-model
  - fortran
  - summa
  - netcdf
  - sundials
  - hydrologic-modeling
---

# SUMMA Complete Guide

> **SUMMA** = Structure for Unifying Multiple Modeling Alternatives
> Originating institution: NCAR / Univ. of Saskatchewan / Univ. of Washington
> Source (canonical): https://github.com/NCAR/summa
> Active development fork: https://github.com/CH-Earth/summa
> Online docs: https://summa.readthedocs.io
> Foundational papers: Clark et al. 2015a (WRR, doi:10.1002/2015WR017198), 2015b (WRR, doi:10.1002/2015WR017200), 2015c (NCAR/TN-514+STR), 2021 (J. Hydromet., doi:10.1175/JHM-D-20-0175.1)

**What SUMMA does.** SUMMA solves coupled land-surface energy and water
equations at a column (HRU) or over a network of columns (GRUs of HRUs).
Its defining feature is that the conservation equations and the numerical
solver form a fixed structural core, while the flux parameterizations are
selectable at runtime. The same executable can mimic Noah-MP, CLM, VIC,
PRMS, or other models by toggling 40+ named decisions.

**Who this skill is for.** Agents, students, and researchers who want to
install, compile, configure, run, modify, debug, and contribute to SUMMA.

---

## Quick Decision Tree

```
"What do I need?"
|
+-- First time. What is SUMMA and how do I install it?
|   Read: reference/getting-started.md
|   (clone, dependencies NetCDF/LAPACK/Sundials, Makefile vs CMake vs Docker)
|
+-- I want to understand how the source tree is organized
|   Read: reference/architecture.md
|   (driver, dshare, engine, hookup, netcdf, noah-mp; data structures, call chain)
|
+-- I want to set up and run a SUMMA simulation
|   Read: reference/running-summa.md
|   (file manager, attributes, parameters, forcing, output control, summa.exe CLI)
|
+-- I want to know which physics option to set for X
|   Read: reference/model-decisions.md
|   (the 40 numbered decisions, options table, how to mimic Noah-MP/CLM/VIC)
|
+-- I want to read or post-process SUMMA output
|   Read: reference/output-and-postprocess.md
|   (NetCDF dimensions, midToto/ifcToto layer indexing, pysumma helpers)
|
+-- I want to reproduce the published case studies
|   Read: reference/case-studies.md
|   (Reynolds Mountain East, Senatorial Snow, Wrightwood, Aspen)
|
+-- I want to use the SUNDIALS solver branch
|   Read: reference/sundials-and-solvers.md
|   (CH-Earth fork, IDA vs KINSOL, Jacobian options, build flags)
|
+-- The model won't compile, won't run, or gives implausible output
|   Read: reference/debugging.md
|   (dimension errors, missing values, snowpack instability, balance violations)
|
+-- I want to submit a pull request
    Read: reference/contributing-pr.md
    (NCAR vs CH-Earth fork choice, develop branch, testing, review)
```

---

## What's Inside SUMMA

```
summa/
|-- build/
|   |-- Makefile              <- traditional build (env vars: F_MASTER, FC, FC_EXE, INCLUDES, LIBRARIES)
|   |-- CMakeLists.txt        <- CMake build
|   |-- cmake/
|   |   |-- compile_script.sh
|   |   |-- FindNetCDF.cmake
|   |   `-- FindOpenBLAS.cmake
|   `-- source/
|       |-- driver/           <- top-level program (summa_driver.f90), runtime orchestration
|       |-- dshare/           <- shared data: data_types, var_lookup, multiconst, popMetadat
|       |-- engine/           <- physics + solver: coupled_em, computFlux, computJacob, snow*, soil*, vegNrgFlux, mDecisions
|       |-- hookup/           <- file/IO orchestration: summaFileManager, ascii_util
|       |-- netcdf/           <- NetCDF I/O: def_output, modelwrite, read_icond
|       |-- noah-mp/          <- inherited Noah-MP routines (module_sf_noahmplsm.F)
|       `-- lapack/           <- LAPACK shims when system lib not used
|-- case_study/
|   |-- base_settings/        <- VEGPARM.TBL, SOILPARM.TBL, GENPARM.TBL, MPTABLE.TBL, localParamInfo.txt, basinParamInfo.txt
|   `-- reynolds/             <- Reynolds Mountain East albedo decay experiment
|       |-- fileManager_reynoldsConstantDecayRate.txt
|       |-- fileManager_reynoldsVariableDecayRate.txt
|       |-- forcing/, site_settings/, output/, evaluation/
|-- ci/summa_install_utils/   <- Travis install scripts
|-- docs/                     <- mkdocs source for summa.readthedocs.io
|-- utils/convert_summa_config_v2_v3.py
|-- Dockerfile
|-- COPYING                   <- GPLv3
|-- LICENSE.txt
|-- header.license            <- file header template for new source
|-- mkdocs.yml
`-- readme.md -> docs/index.md
```

After a successful build, `summa.exe` lands in `summa/bin/`.

---

## Critical Rules

1. **The file manager is the single entry point.** Every SUMMA run is launched
   as `summa.exe -m <path/to/fileManager.txt>`. That ASCII file enumerates
   `settingsPath`, `forcingPath`, `outputPath`, the decisions file, the
   attributes file, the parameter files, the forcing list, the initial
   conditions file, and the output prefix. Get the paths wrong and SUMMA
   refuses to start.

2. **`controlVersion` must match the file manager parser.** Current code
   expects `'SUMMA_FILE_MANAGER_V3.0.0'`. V1 and V2 file managers are not
   backward-compatible. Use `utils/convert_summa_config_v2_v3.py` to migrate
   old setups.

3. **Forcing is period-ending and time-zone explicit.** Forcing time stamps
   reflect the average over the preceding `data_step` seconds. The
   `tmZoneInfo` setting (`utcTime`, `localTime`, or `ncTime`) must be
   consistent with the `units` attribute on the NetCDF `time` variable.
   Mismatches silently shift solar zenith and produce wrong fluxes.

4. **GRU and HRU order must be identical across all NetCDF inputs.** SUMMA
   relies on positional alignment among the attributes, forcing, cold-state,
   and trial-parameter files. When a GRU contains multiple HRUs, the HRUs
   for that GRU must occupy contiguous indices. The `hruId` and `gruId`
   values themselves can be arbitrary.

5. **Parameter files are layered, not exclusive.** The order is local
   parameters, then basin parameters, then Noah-MP tables (overwrite by
   soil/veg type), then trial parameters (HRU/GRU specific). Later files
   overwrite earlier ones for the same variable. Trial parameters always
   win.

6. **NetCDF Fortran is required, not just NetCDF C.** SUMMA links
   `-lnetcdff -lnetcdf`. Many systems install only `libnetcdf`. Build the
   Fortran bindings (`netcdf-fortran` package) with the same compiler used
   for SUMMA.

7. **LAPACK is required.** On macOS the standard route is the `lapack`
   Homebrew formula plus BLAS. On Linux pick `liblapack-dev` and
   `libblas-dev`, or load the vendor MKL/OpenBLAS module on HPC.

8. **Layer-resolved output uses concatenated dimensions
   (`midTotoAndTime`, `ifcSoilAndTime`, etc.).** To extract a per-time-step
   profile you must combine `nLayers`, `nSnow`, `nSoil`, and the
   `<dim>StartIndex` variables. SUMMA stores 1-based indices; subtract 1
   when slicing in Python.

---

## Quick Start

```bash
# 1. Get the code
git clone https://github.com/NCAR/summa.git
cd summa

# 2. Install dependencies (macOS Homebrew shown)
brew install gcc netcdf netcdf-fortran lapack

# 3. Pick a build system. CMake is the recommended modern path:
cd build/cmake
./compile_script.sh        # creates cmake_build/, builds parallel, drops summa.exe in summa/bin/

# Alternatively use the traditional Makefile
cd ../                     # back to summa/build/
export F_MASTER=$(pwd)/..
export FC=gfortran
export FC_EXE=gfortran
export INCLUDES="-I/opt/homebrew/include"
export LIBRARIES="-L/opt/homebrew/lib -lnetcdff -llapack -lblas"
make check                 # confirm variables look right
make                       # build; executable is summa/bin/summa.exe

# 4. Smoke-test the binary
../bin/summa.exe -v        # prints version banner

# 5. Run the Reynolds Mountain East case study
cd ../case_study/reynolds
../../bin/summa.exe -m fileManager_reynoldsConstantDecayRate.txt
../../bin/summa.exe -m fileManager_reynoldsVariableDecayRate.txt

# 6. Inspect output
ls output/
ncdump -h output/reynolds_constantDecayRate_*.nc | head -40
```

Full walkthrough in `reference/getting-started.md` and
`reference/running-summa.md`.

---

## Reference Documents

| Document | What's inside |
|----------|---------------|
| `reference/getting-started.md` | Repos, dependencies (NetCDF/LAPACK/Sundials), Makefile vs CMake vs Docker, build verification |
| `reference/architecture.md` | Directory map of `build/source`, derived data types in `dshare/data_types.f90`, the call chain through `coupled_em`, `systemSolv`, `computFlux`, `computJacob` |
| `reference/running-summa.md` | File manager fields, attributes/parameters/forcing/output-control/initial-condition file layouts, `summa.exe` CLI flags |
| `reference/model-decisions.md` | All 40 numbered decisions, allowed options, what each governs, how to assemble a Noah-MP / CLM / VIC mimic |
| `reference/output-and-postprocess.md` | NetCDF output dimensions, the `[mid|ifc][Snow|Soil|Toto]andTime` layout, restart vs history files, pysumma `Simulation` and `Ensemble` post-processing patterns |
| `reference/case-studies.md` | Reynolds Mountain East albedo experiment, plus the canonical Wrightwood, Senatorial Snow, and Aspen cases from Clark et al. 2015b |
| `reference/sundials-and-solvers.md` | The CH-Earth `summa-sundials` development line, IDA implicit DAE solver, KINSOL nonlinear solver, Jacobian and matrix-band options |
| `reference/debugging.md` | NetCDF dimension mismatches, missing-value floods, snowpack layer instability, energy/water-balance violations, NaN propagation |
| `reference/contributing-pr.md` | Choosing the right upstream (NCAR vs CH-Earth), develop-branch workflow, coding conventions, header.license, pull-request expectations |
