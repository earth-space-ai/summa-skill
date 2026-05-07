# Debugging SUMMA

A field guide to the most common SUMMA failure modes: build errors,
runtime crashes, dimension mismatches, missing-value floods, snowpack
instability, and energy or water balance violations. Each entry lists
the symptom, the most likely cause, and the fastest diagnostic.

---

## 1. Build errors

### `cannot find -lnetcdff`

The NetCDF Fortran package is missing or not on the linker path. Check:

```bash
nf-config --prefix
nf-config --flibs        # should print -lnetcdff -lnetcdf ...
```

If `nf-config` itself is missing, install `netcdf-fortran` (a separate
package from `netcdf`). Update `LIBRARIES` in your environment or in
`build/Makefile` to include `-L$(nf-config --prefix)/lib`.

### `cannot find -llapack`

LAPACK is missing. macOS: `brew install lapack`. Ubuntu: `apt install
liblapack-dev libblas-dev`. HPC: `module load openblas` or use Intel MKL
with `-mkl`.

### `Symbol not found: __netcdf_MOD_*`

Mismatched compiler between the NetCDF Fortran build and the SUMMA
build. Rebuild `netcdf-fortran` with the same `gfortran` or `ifort` you
use for SUMMA, or load the matching module. Mixing gfortran-built
NetCDF with ifort-built SUMMA always fails this way.

### CMake reports "Could NOT find NetCDF"

NetCDF is outside the system search path.

```bash
export CMAKE_PREFIX_PATH="$CMAKE_PREFIX_PATH:/path/to/netcdf"
rm -rf build/cmake/cmake_build
./build/cmake/compile_script.sh
```

### Build fails inside `noah-mp/` files

The Noah-MP files are free-form Fortran with long lines. They need
`-ffree-line-length-none` (gfortran). The Makefile sets
`FLAGS_NOAH = -O3 -ffree-form -ffree-line-length-none -fmax-errors=0`
specifically for them. If you forked the build system, preserve that
flag block.

### Build succeeds but `summa.exe -v` reports the wrong version

Stale binary in `bin/`. Always start from clean:

```bash
make clean && make            # Makefile path
# or
rm -rf build/cmake/cmake_build && ./build/cmake/compile_script.sh
```

---

## 2. Startup errors

### `cannot open ... fileManager`

The `-m` argument is interpreted relative to the current working
directory. Either `cd` into the experiment directory or pass an
absolute path.

### `cannot find decisionsFile` (or any other input file)

Paths inside the file manager are relative to `settingsPath`,
`forcingPath`, or `outputPath` depending on the keyword. Re-read the
"Required keywords" table in `running-summa.md`.

### `controlVersion mismatch`

You are running V3 SUMMA against a V1 or V2 file manager. Convert with
`utils/convert_summa_config_v2_v3.py`.

### `requested decision not in registry`

Typo in `modelDecisions.txt` or you are using an option from a
development branch with stable SUMMA. The authoritative list is in
`build/source/engine/mDecisions.f90`.

### `requested variable not in metadata`

Typo in `outputControl.txt`. Search `build/source/dshare/var_lookup.f90`
for the intended name (case-sensitive). Variables are organized into
`iLook_force`, `iLook_attr`, `iLook_param`, `iLook_index`, `iLook_prog`,
`iLook_diag`, `iLook_flux`, `iLook_bpar`, `iLook_bvar`, `iLook_deriv`.

---

## 3. Input data errors

### Dimension mismatch between attribute and forcing files

```
ERROR: number of HRUs in forcing file does not match attributes file
```

The `hru` dimension in every NetCDF input must agree, and the `hruId`
values must appear in the same order. Use `ncdump -h` to verify both
sides:

```bash
ncdump -h settings/localAttributes.nc | grep 'hru ='
ncdump -h forcing/forcing_2010.nc     | grep 'hru ='
```

When a GRU contains multiple HRUs, the HRUs must occupy contiguous
indices in every file.

### `simStartTime not found in forcing`

Forcing time stamps are period-ending. If `simStartTime` is
`2005-09-01 00:00` and your hourly forcing has stamps on the hour, the
first usable stamp is `2005-09-01 01:00` (representing 00:00-01:00).
SUMMA expects `simStartTime` to equal one of the time stamps, so set
`simStartTime` to `2005-09-01 01:00` for hourly data starting that day,
or to `2005-09-01 03:00` for 3-hourly data.

### `forcing file time units malformed`

The `units` attribute on the `time` variable must be CF-compliant:

```
units = "seconds since 1970-01-01 00:00:00"
```

Optional zone offset is allowed:

```
units = "hours since 2005-09-01 00:00:00 -7:00"
```

The zone is read only when `tmZoneInfo = 'ncTime'`. With `utcTime` or
`localTime`, the zone in the units string is ignored.

### Forcing values look wrong by a factor

Common unit errors:

- Precipitation in mm/hr instead of kg m-2 s-1. SUMMA expects rate
  (kg m-2 s-1, equivalent to mm/s). 1 mm/hr = 0.0002777... kg m-2 s-1.
- Specific humidity in g/kg instead of g/g (kg/kg). Divide by 1000.
- Air pressure in hPa instead of Pa. Multiply by 100.
- Wind speed at 10 m when `mHeight` is 2 m. Either downscale the wind
  or set `mHeight` to 10.

---

## 4. Initial condition mismatches

### `restart variable X not in restart file`

You changed model decisions between the run that wrote the restart and
the run that reads it. The required state vector depends on which
decisions are active. Cold-start from a freshly built initial conditions
file instead.

### `nSnow > 0 in restart but mLayerTemp not provided for snow layers`

Your restart file is incomplete. `nSnow` says there are N snow layers,
but the per-layer arrays are sized for soil only. Regenerate the
restart file with consistent sizing, or set `nSnow = 0` and provide
`scalarSnowDepth = 0` for a clean cold start.

### `dt_init not in restart`

`dt_init` (the initial sub-step length) is required even for a cold
start. Set it to `60` seconds for a snow-dominated case or `3600` for a
warm catchment.

---

## 5. Runtime crashes

### Segfault inside `computJacob.f90` or `summaSolve.f90`

State vector size inconsistent with the layer counts. Check that
`nSnow + nSoil = nLayers` for every HRU at the failing time step. Often
a sign that `layerDivide.f90` or `layerMerge.f90` produced an
inconsistent layer state.

### `IEEE_INVALID_FLAG` warnings

NaNs in the state. Build with debug flags:

```bash
# in the Makefile, change the active FLAGS_* lines to the debug variants:
FLAGS_NOAH  = -p -g -ffree-form -ffree-line-length-none -fbacktrace
FLAGS_COMM  = -p -g -Wall -ffree-line-length-none -fbacktrace -fcheck=bounds
FLAGS_SUMMA = -p -g -Wall -ffree-line-length-none -fbacktrace -fcheck=bounds
```

Rebuild from clean. The first `IEEE_INVALID_FLAG` line points at the
file and routine that produced the NaN.

### Run hangs or extremely slow

Check whether the time step controller is stuck in a tight loop. SUMMA
prints a progress line based on `-p`. If progress lines stop:

- The implicit solver might be failing to converge and looping with
  ever-smaller sub-steps. Print the residual norm in `eval8summa.f90` to
  see.
- A snowpack might be alternating layer split and merge every sub-step.
  Check `layerDivide.f90` and `layerMerge.f90` thresholds in
  `localParamInfo.txt`.

### `convergence failure in systemSolv`

The Newton iteration ran out of attempts. First diagnostic: print the
state vector and the residual at the last iteration. Common causes:

- A flux is overshooting. Reduce the maximum sub-step:
  `maxstep` parameter in `localParamInfo.txt`.
- An analytic Jacobian entry is wrong (only relevant if you modified
  the physics). Switch to `fDerivMeth = numericl` and re-run; if the
  problem disappears, the analytic derivative is to blame.
- A boundary condition is inconsistent. For example,
  `bcLowrSoiH = drainage` with a saturated lowermost soil layer
  produces flux that the solver cannot balance.

---

## 6. Output looks wrong

### Variable is all `realMissing` (large negative sentinel)

The decision that turns the variable on is not active, or the variable
is masked because the relevant physics is off. For example,
`scalarRainPlusMelt` is missing when there are no snow layers in some
versions before v3.2.0. Check the `whats-new.md` in the repo for
version-specific behavior changes.

### Variable is constant year-round

Likely candidates:

- `scalarLatHeatTotal` zero when `computeVegFlux = .false.` and the
  ground is frozen.
- `scalarSWE` zero when forcing precipitation is zero or always falls
  as rain. Check the rain/snow partition decision and the air
  temperature forcing.
- LAI flat when `LAI_method = monTable` and the vegetation type maps to
  a row with the same LAI in every month.

### Layer-resolved variable plotted as garbage

You forgot to slice with `nLayers` and `*StartIndex`. The raw
`mLayerVolFracWat(midTotoAndTime, hru)` is a one-dimensional
concatenation, not `(time, layer)`. See the slicing recipe in
`output-and-postprocess.md`.

### Snow depth oscillates erratically time step to time step

Layer split/merge thrashing. Tune the `zminLayer*` and `zmaxLayer*`
parameters in `localParamInfo.txt` so adjacent layer thresholds are not
too close. With `snowLayers = CLM_2010` use the CLM-tuned defaults; with
`jrdn1991` use the wider defaults.

---

## 7. Energy and water balance violations

SUMMA includes balance diagnostics that print to stderr when the
per-time-step error exceeds a tolerance. Symptoms:

```
balance error in coupled_em: massBalance =  ...  energyBalance = ...
```

Most common causes, in order of frequency:

1. **A new state variable was added without a balance term.** If you
   added a new water reservoir, you must add it to the mass balance
   accounting in `coupled_em.f90`. If you added a new energy state, you
   must add it to the energy balance.

2. **Forcing has unphysical values.** Negative precipitation, negative
   shortwave, or specific humidity above saturation create balance
   errors that look like model bugs.

3. **A flux is being double-counted.** Check that any new flux is added
   to the receiving reservoir and subtracted from the donating one,
   never both.

4. **Layer merge transferred mass or energy incorrectly.** When you
   merge two snow layers, the new layer's water content must equal the
   sum of the two parents'. If you split or modified `layerMerge.f90`,
   re-derive the merge formulas.

5. **The tolerance is too tight for single precision.** SUMMA mostly
   runs in double precision, but if you flipped to single, balance
   tolerances may need loosening.

---

## 8. Diagnostics for the impatient

### Inspect a NetCDF file at the command line

```bash
ncdump -h output/run_1.nc                    # header only
ncdump -v scalarSWE output/run_1.nc | head   # variable contents
ncdump -t -v time output/run_1.nc | head     # human-readable time
```

### Compare two output files variable by variable

```bash
nccmp -df run_a.nc run_b.nc
cdo diff run_a.nc run_b.nc
```

### Snapshot the state at a specific time step

Add to `outputControl.txt` for the variable in question:

```
mLayerTemp        | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0
mLayerHeight      | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0
nSnow             | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0
nSoil             | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0
nLayers           | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0
midTotoStartIndex | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0
```

Without all five indexing helpers you cannot reconstruct the per-step
profile.

### Find every print statement in the engine

```bash
grep -nE 'print \*|write\(\*' build/source/engine/*.f90 | head -40
```

Most are gated by `globalPrintFlag` (in `dshare/globalData.f90`). Set it
to `.true.` temporarily for a chatty run.

### Use the build-time debug flag profile

Edit `build/Makefile` and uncomment the debug lines under `# Debug
runs`. They enable backtrace, bounds checking, and free-form length
relaxation. Strip them again before production: `-fcheck=bounds` is
several times slower than the production build.

---

## Where to next

- Inspect the file that the failure traces to: `architecture.md`
- Adjust the offending decision: `model-decisions.md`
- Re-check input file format: `running-summa.md`
- Switch to SUNDIALS if the legacy solver is the cause:
  `sundials-and-solvers.md`
