# SUMMA Source Architecture

This page is the map you need before you edit anything in the source tree.
SUMMA's organizing principle, stated by Clark et al. (2015a), is that the
conservation equations and the numerical solver form a fixed structural
core, while the flux parameterizations are pluggable. The source layout
mirrors that split.

---

## Top-level layout

```
summa/
|-- build/
|   |-- Makefile
|   |-- CMakeLists.txt
|   |-- cmake/
|   |   |-- compile_script.sh
|   |   |-- FindNetCDF.cmake
|   |   `-- FindOpenBLAS.cmake
|   `-- source/
|       |-- driver/      <- top-level program, run loop, setup, restart
|       |-- dshare/      <- shared data: types, var lookup tables, constants, metadata
|       |-- engine/      <- physics + numerics: flux/Jacobian/residual, solvers, snow, soil, veg, mDecisions
|       |-- hookup/      <- file IO orchestration: file manager, ASCII parser
|       |-- netcdf/      <- NetCDF read/write helpers and output definition
|       |-- noah-mp/     <- inherited Noah-MP routines (SOIL_VEG_GEN_PARM, NOAHMP_OPTIONS_GLACIER, etc.)
|       `-- lapack/      <- LAPACK shims (used when system LAPACK unavailable)
|-- bin/                 <- summa.exe lands here after a successful build
|-- case_study/          <- ready-to-run experiments (Reynolds Mountain East, plus base settings)
|-- ci/summa_install_utils/
|-- docs/                <- mkdocs source for summa.readthedocs.io
|-- utils/convert_summa_config_v2_v3.py
|-- Dockerfile
|-- COPYING              <- GPLv3
|-- LICENSE.txt
|-- header.license       <- per-file header to copy verbatim into new sources
|-- mkdocs.yml
`-- readme.md -> docs/index.md
```

Rule of thumb when locating a piece of code:

- File and IO orchestration: `hookup/`
- Read or write a NetCDF file: `netcdf/`
- Master data definitions, lookup tables, metadata: `dshare/`
- Per-HRU physics, fluxes, derivatives, solvers: `engine/`
- Top-level run loop, restart, output writeout: `driver/`
- Routines inherited from Noah-MP (vegetation, soil parameter table parsing): `noah-mp/`

---

## The driver layer (`build/source/driver/`)

`summa_driver.f90` is the program. Its run loop reduces to:

```fortran
call summa_initialize(...)        ! allocate, read decisions, attributes, params
call summa_paramSetup(...)        ! veg/soil tables and per-HRU params
call summa_readRestart(...)       ! cold start or warm start
do modelTimeStep = 1, numtim
   call summa_readForcing(...)    ! read one forcing step
   call summa_runPhysics(...)     ! call engine for every GRU/HRU
   call summa_writeOutputFiles(...) ! emit history; emit restart at the cadence asked on the command line
end do
call stop_program(0, 'finished')
```

Other driver-layer modules:

| File | Role |
|------|------|
| `summa_init.f90` | Allocate the master `summa1_type_dec` structure; read decisions, attributes, parameters |
| `summa_setup.f90` | Initialize parameter data structures (veg, soil) including Noah-MP table reads |
| `summa_restart.f90` | Read the initial conditions / restart NetCDF |
| `summa_forcing.f90` | Pull the next forcing time step from disk |
| `summa_modelRun.f90` | Per-time-step physics: call `coupled_em_module` for every GRU/HRU |
| `summa_writeOutput.f90` | Marshal output statistics and call the writer in `netcdf/modelwrite.f90` |
| `summa_defineOutput.f90` | First-call definition of the output history file |
| `summa_alarms.f90` | Decide when to write history vs restart |
| `summa_globalData.f90` | Initialization of `globalData` structures used everywhere |
| `summa_type.f90` | Defines `summa1_type_dec`, the master container that holds all state, fluxes, and parameters |
| `summa_util.f90` | `stop_program`, `handle_err`, error message helpers |

---

## Shared data (`build/source/dshare/`)

These files are the contract between the driver and the engine.

| File | Defines |
|------|---------|
| `nrtype.f90` (in `engine/`, but conceptually shared) | Numeric kinds: `i4b`, `i8b`, `sp`, `dp`, `rk` (the floating type used everywhere) |
| `multiconst.f90` | Physical constants: `Tfreeze`, `LH_fus`, `LH_sub`, `iden_water`, `iden_ice`, ... do not redefine elsewhere |
| `data_types.f90` | Compound types: `var_d`, `var_i`, `var_dlength`, `var_ilength`, `model_options` |
| `var_lookup.f90` | Named indices for every metadata structure: `iLook_time`, `iLook_force`, `iLook_attr`, `iLook_type`, `iLook_param`, `iLook_index`, `iLook_prog`, `iLook_diag`, `iLook_flux`, `iLook_bpar`, `iLook_bvar`, `iLook_deriv` |
| `get_ixname.f90` | Functions like `get_ixdecisions(varName)` and `get_ixparam(varName)` that look up an integer index from a string name |
| `popMetadat.f90` | Populates the metadata arrays read by the output controller; `read_output_file()` parses `outputControl.txt` here |
| `globalData.f90` | Module-level globals: `numtim`, `globalPrintFlag`, `realMissing`, `integerMissing`, `iname_snow`, `iname_soil`, etc. |
| `flxMapping.f90` | Maps fluxes to the state variables they update |
| `outpt_stat.f90` | Statistical aggregation across forcing time steps for output |

The cardinal rule: a new state, parameter, or output variable must be
declared (1) in the appropriate `iLook_*` named index in `var_lookup.f90`,
(2) in the `popMetadat.f90` table, (3) in any read/write helpers, and
(4) in `flxMapping.f90` if it is a flux. The compiler will not catch a
missing entry; the symptom at runtime is missing-value floods or silent
zero output.

---

## The engine (`build/source/engine/`)

This directory holds the physics and the numerics. It is where most users
who modify the code spend their time.

### Time integration entry point

`coupled_em.f90` (`coupled_em_module`) is the per-HRU driver. It receives
one forcing time step, splits it into adaptive sub-steps, and drives the
coupled energy and water solution to the end of the forcing step. It calls:

```
coupled_em
|-- vegPhenlgy        (LAI/SAI dynamics, optionally from monthly tables)
|-- volicePack        (snow/ice depth bookkeeping)
|-- layerDivide / layerMerge   (snowpack layer management)
|-- canopySnow        (canopy snow interception)
|-- opSplittin        (operator-splitting controller; calls systemSolv)
|   `-- systemSolv    (assembled implicit step)
|       |-- eval8summa            (evaluate the residual)
|       |   |-- computFlux        (compute every flux)
|       |   `-- computResid       (compute residual vector)
|       |-- computJacob           (compute Jacobian, analytic or numeric)
|       |-- summaSolve            (solve the linear system: dense or band)
|       |-- updateVars            (project state increments back to physical space)
|       `-- varSubstep            (step-size control)
`-- qTimeDelay        (time-delay routing for the GRU)
```

### Physics modules grouped by domain

| Group | Files |
|-------|-------|
| Numerical infrastructure | `nrtype.f90`, `nr_utility.f90`, `f2008funcs.f90`, `expIntegral.f90`, `spline_int.f90`, `time_utils.f90`, `sunGeomtry.f90` |
| Time stepping and operator splitting | `coupled_em.f90`, `opSplittin.f90`, `systemSolv.f90`, `varSubstep.f90`, `summaSolve.f90` |
| Residual and Jacobian | `eval8summa.f90`, `computResid.f90`, `computFlux.f90`, `computJacob.f90`, `matrixOper.f90` |
| State management | `getVectorz.f90`, `updateVars.f90`, `updatState.f90`, `var_derive.f90`, `indexState.f90`, `convE2Temp.f90`, `tempAdjust.f90`, `paramCheck.f90`, `pOverwrite.f90` |
| Forcing | `read_force.f90`, `derivforce.f90`, `ffile_info.f90` |
| Attributes and parameters | `read_attrb.f90`, `read_param.f90`, `read_pinit.f90`, `paramCheck.f90`, `checkStruc.f90`, `childStruc.f90`, `allocspace.f90` |
| Vegetation energy/water | `vegNrgFlux.f90`, `vegLiqFlux.f90`, `vegSWavRad.f90`, `vegPhenlgy.f90`, `stomResist.f90` |
| Snow physics | `canopySnow.f90`, `snowAlbedo.f90`, `snowLiqFlx.f90`, `snwCompact.f90`, `layerDivide.f90`, `layerMerge.f90`, `volicePack.f90`, `snow_utils.f90` |
| Soil physics | `soilLiqFlx.f90`, `soil_utils.f90`, `ssdNrgFlux.f90`, `diagn_evar.f90` |
| Groundwater and routing | `groundwatr.f90`, `bigAquifer.f90`, `qTimeDelay.f90` |
| Initial conditions | `check_icond.f90` |
| Per-HRU and per-GRU loops | `run_oneHRU.f90`, `run_oneGRU.f90` |
| Decision registry | `mDecisions.f90` |
| Conversion helpers | `conv_funcs.f90` |

### `mDecisions.f90` is the decision registry

Every model option (e.g., `f_Richards = mixdform`) maps to an integer
constant in `mDecisions.f90`. The integer is then used inside `select case`
branches in the physics modules to dispatch to the right parameterization.
When you add a new physics option, you (1) add a named integer here, (2)
extend the `select case` in the relevant physics module, (3) extend
`get_ixdecisions` in `dshare/get_ixname.f90`, and (4) document it on the
SUMMA Model Decisions page.

---

## File and IO orchestration (`build/source/hookup/`)

| File | Role |
|------|------|
| `summaFileManager.f90` | Parses the master configuration file passed via `-m`. Stores all paths and file names as module variables. |
| `ascii_util.f90` | Generic ASCII reader: line splitting, comment stripping (`!` to end of line), single-quoted string extraction. |

`summaFileManager.f90` is the single place where the master file format
lives. If you add a new file path keyword (say a new lookup table), you
add it here.

---

## NetCDF IO (`build/source/netcdf/`)

| File | Role |
|------|------|
| `def_output.f90` | Defines the output NetCDF file: dimensions, variables, attributes |
| `modelwrite.f90` | Writes data to the history file (`writeData`, `writeBasin`, `writeTime`, `writeParm`, `writeRestart`) |
| `read_icond.f90` | Reads the initial conditions / restart file |
| `netcdf_util.f90` | Thin wrappers around `nf90_*` calls with consistent error handling |

The output dimension layout is described in detail in
`output-and-postprocess.md`. The key complication: per-layer variables go
along concatenated dimensions like `midTotoAndTime` because NetCDF-3 did
not support ragged arrays.

---

## Inherited Noah-MP code (`build/source/noah-mp/`)

| File | Role |
|------|------|
| `module_sf_noahmplsm.F` | Subset of Noah-MP routines used for vegetation parameterization compatibility |
| `module_sf_noahlsm.F` | Older Noah LSM routines |
| `module_sf_noahutl.F` | Noah utilities (table parsing) |
| `module_model_constants.F` | Constants used inside the inherited Noah-MP code |

These files are compiled with their own flag profile (`FLAGS_NOAH` in the
Makefile) because they are free-form Fortran with long lines that need
`-ffree-line-length-none`.

The Noah-MP-style table parser (`SOIL_VEG_GEN_PARM`) lives in
`driver/multi_driver.f90` (older builds) or is invoked from
`summa_setup.f90`. It reads `VEGPARM.TBL`, `SOILPARM.TBL`, and
`GENPARM.TBL` to overwrite the spatially constant defaults loaded earlier.

---

## The master container `summa1_type_dec`

Defined in `driver/summa_type.f90`. It holds, per HRU and per GRU,
everything the model needs:

- `time_data`: the current model time stamp.
- `forc_data`: the current forcing values.
- `attr_data`: the time-invariant local attributes (`hruId`, `vegTypeIndex`, `mHeight`, etc.).
- `type_data`: integer indices (slope, soil, vegetation classifications).
- `prog_data`: prognostic (state) variables; the integration target.
- `diag_data`: diagnostics computed from prognostic variables.
- `flux_data`: per-time-step fluxes.
- `deriv_data`: derivatives used by the implicit solver.
- `mpar_data`: the HRU-level parameters.
- `bpar_data`, `bvar_data`: GRU-level parameters and basin diagnostics.
- `indx_data`: indexing structures (`nSnow`, `nSoil`, `nLayers`, etc.).
- `model_decisions`: the resolved decision registry.

Inside the engine modules you typically `associate` the slices you need:

```fortran
associate( &
   scalarCanopyTemp => prog_data%var(iLookPROG%scalarCanopyTemp)%dat(1), &
   mLayerVolFracIce => prog_data%var(iLookPROG%mLayerVolFracIce)%dat(:), &
   ... )
```

This pattern keeps subroutine signatures short while preserving explicit
intent.

---

## Flow of a single time step (illustrated)

```
summa_driver
 |
 +-- summa_runPhysics
       |
       +-- run_oneGRU                       (loop GRUs)
             |
             +-- run_oneHRU                 (loop HRUs in this GRU)
                   |
                   +-- coupled_em
                         |
                         +-- vegPhenlgy
                         +-- canopySnow
                         +-- volicePack
                         +-- layerDivide / layerMerge
                         +-- opSplittin
                               |
                               +-- systemSolv
                                     |
                                     +-- eval8summa
                                     |   +-- computFlux
                                     |   +-- computResid
                                     +-- computJacob
                                     +-- summaSolve  (LAPACK band or dense)
                                     +-- updateVars
                                     +-- varSubstep
                         +-- qTimeDelay
```

When you trace a bug, walk the chain top-down: at which module did the
state become wrong? Then drop into that module's `associate` block and
inspect.

---

## Where to next

- Configure and run an actual simulation: `running-summa.md`
- Choose physics options: `model-decisions.md`
- Slice the layer-resolved output: `output-and-postprocess.md`
- Replace the LAPACK solver with SUNDIALS: `sundials-and-solvers.md`
