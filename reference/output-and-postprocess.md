# SUMMA Output and Post-Processing

SUMMA writes two kinds of NetCDF files: history (time-series of model
variables) and restart (state at a single instant). This page covers
the file structure, the awkward layer-time dimension layout that exists
because NetCDF-3 lacked ragged arrays, and the standard ways to
post-process the output in Python with `xarray` and `pysumma`.

---

## Output file types

| Type | Cadence | Written by | Path |
|------|---------|-----------|------|
| History | Every forcing step (or aggregated per the output control file) | `netcdf/modelwrite.f90:writeData/writeBasin/writeTime/writeParm` | `<outputPath>/<outFilePrefix>_<run-suffix>.nc` |
| Restart | Cadence set by the `-r` command-line flag | `netcdf/modelwrite.f90:writeRestart` | `<outputPath>/<outFilePrefix>_restart_YYYYMMDDhhmm.nc` (or `<statePath>/...` if set) |

The history file name includes the `-s|--suffix` flag content if
provided. With `-g startGRU countGRU`, multiple ranks write to separate
files distinguished by GRU range.

---

## History file dimensions

SUMMA defines these dimensions in `netcdf/def_output.f90`. Not all of
them are populated in every file; those that have no associated variable
are still declared.

| Dimension | Long name | Use |
|-----------|-----------|-----|
| `hru` | HRU dimension | Variables and parameters that vary by HRU |
| `gru` | GRU dimension | Variables and parameters that vary by GRU |
| `depth` | Soil depth | Fixed-layer-count variables |
| `scalar` | Scalar | Degenerate dimension for true scalars |
| `spectral_bands` | Spectral bands | Albedo, radiation by band |
| `time` | Time | All time-varying variables |
| `timeDelayRouting` | Time delay routing | Internal routing arrays |
| `midSnowAndTime` | Mid-points of snow layers concatenated through time | Per-snow-layer time series |
| `midSoilAndTime` | Mid-points of soil layers concatenated through time | Per-soil-layer time series |
| `midTotoAndTime` | Mid-points of all layers concatenated through time | Per-layer time series across snow + soil |
| `ifcSnowAndTime` | Snow layer interfaces concatenated through time | Inter-layer flux time series in snow |
| `ifcSoilAndTime` | Soil layer interfaces concatenated through time | Inter-layer flux time series in soil |
| `ifcTotoAndTime` | All layer interfaces concatenated through time | Inter-layer flux time series across snow + soil |

The `mid*AndTime` and `ifc*AndTime` dimensions exist because the number
of snow layers changes over time as snow accumulates and melts. NetCDF-3
did not support ragged arrays, so SUMMA flattens the layer and time
dimensions: at each time step the layer values are appended end-to-end.
NetCDF-4 added variable-length arrays but SUMMA has not yet migrated.

---

## Slicing a layer-time variable

To extract one time step of a per-layer variable you must combine three
helpers that SUMMA writes alongside the data:

- `nLayers(time, hru)`: total number of layers (snow + soil) at this
  step.
- `nSnow(time, hru)`: number of snow layers at this step. Zero when
  there is no snowpack.
- `nSoil(time, hru)`: number of soil layers (constant across time in
  most setups).
- `midTotoStartIndex(time, hru)`, `ifcTotoStartIndex(time, hru)`,
  `midSnowStartIndex(time, hru)`, `midSoilStartIndex(time, hru)`,
  `ifcSnowStartIndex(time, hru)`, `ifcSoilStartIndex(time, hru)`: 1-based
  start indices into the corresponding `*AndTime` dimension.

SUMMA writes these indices as 1-based. In Python (0-based) you subtract
1 before slicing.

### Pseudocode for one time step at one HRU

```python
import xarray as xr

ds = xr.open_dataset("output/reynolds_constantDecayRate_1.nc")

hru = 0          # first HRU
t = 0            # first time step

n_layers   = int(ds["nLayers"].isel(time=t, hru=hru).values)
start      = int(ds["midTotoStartIndex"].isel(time=t, hru=hru).values) - 1
end        = start + n_layers

theta_t0   = ds["mLayerVolFracWat"].isel(midTotoAndTime=slice(start, end), hru=hru).values
height_t0  = ds["mLayerHeight"].isel(midTotoAndTime=slice(start, end), hru=hru).values
```

`height_t0[i]` is the depth (m) of the mid-point of layer `i` at time
`t`, with `0.0` at the top of the soil column. Negative values are
inside the soil; positive values are above ground in the snowpack.

To get a contour plot of temperature through time and depth you loop
this slicing over all time steps and assemble a 2D array, since
`midTotoAndTime` is concatenated along time.

### Convenience: flux through a layer interface

For an interface variable (say `iLayerLiqFluxSoil`):

```python
n_soil = int(ds["nSoil"].isel(time=t, hru=hru).values)
start  = int(ds["ifcSoilStartIndex"].isel(time=t, hru=hru).values) - 1
end    = start + n_soil + 1                    # interfaces = layers + 1

flux_t = ds["iLayerLiqFluxSoil"].isel(ifcSoilAndTime=slice(start, end), hru=hru).values
```

Layers indexed by `mid*` have one fewer entry per time step than
interfaces indexed by `ifc*`.

---

## Variable categories in the history file

The output controller pulls names from `dshare/var_lookup.f90`. The
following families are most often requested:

| Family | Examples | Where defined |
|--------|----------|---------------|
| Atmospheric forcing diagnostics | `scalarAirtemp`, `scalarPptrate`, `scalarRainfall`, `scalarSnowfall` | `iLook_force` |
| HRU attributes | `hruId`, `latitude`, `elevation` | `iLook_attr` |
| Surface fluxes | `scalarSenHeatTotal`, `scalarLatHeatTotal`, `scalarGroundEvaporation`, `scalarSnowSublimation` | `iLook_flux` |
| Energy state | `scalarCanairTemp`, `scalarCanopyTemp`, `mLayerTemp` | `iLook_prog` |
| Water state | `scalarSWE`, `scalarSnowDepth`, `scalarCanopyLiq`, `scalarCanopyIce`, `mLayerVolFracIce`, `mLayerVolFracLiq`, `mLayerMatricHead` | `iLook_prog` |
| Diagnostics | `scalarTotalET`, `scalarNetRadiation`, `scalarSurfaceTemp`, `mLayerHeight` | `iLook_diag` |
| Solver | `numFluxCalls`, `wallClockTime` (per HRU per step, added in v3.2.0) | `iLook_diag` |
| Indexing helpers | `nSnow`, `nSoil`, `nLayers`, `midTotoStartIndex`, ... | `iLook_index` |
| Basin diagnostics | `basin__SurfaceRunoff`, `basin__AquiferStorage`, `basin__AquiferTranspire` | `iLook_bvar` |
| Parameters echoed for reference | any variable in `iLook_param` or `iLook_bpar` | (echoed once at start) |

When you ask for a variable but the model decision turns it off (for
example `scalarRainPlusMelt` when there are no snow layers), SUMMA fills
the slot with `realMissing` (typically a large negative sentinel
defined in `dshare/globalData.f90`).

---

## Output statistics

The per-line statistics flags in `outputControl.txt` (sum, inst, mean,
var, min, max, mode) request the listed reductions over the
`<outFreq>` window. SUMMA stores each requested statistic as a separate
NetCDF variable. For example, with

```
scalarSWE | 24 | 0 | 0 | 1 | 0 | 1 | 1 | 0
```

and a 1-hour forcing step, three variables appear in the history file:

- `scalarSWE_mean`
- `scalarSWE_min`
- `scalarSWE_max`

The instantaneous variant has no suffix. The variance variant is
`<varname>_var`. The mode variant is `<varname>_mode`.

---

## Compression

Since v3.2.0 the output controller honors a `deflate` directive:

```
outputPrecision | float
deflate         | 4
```

Default is level 4 if not specified. Higher levels reduce file size at
the cost of CPU during write. For long climate runs this matters.

---

## Restart files

A restart file is the same NetCDF format as the initial conditions file
described in `running-summa.md`. The cadence is controlled by `-r`:

| Flag | Cadence |
|------|---------|
| `-r y` or `-r year` | Annual |
| `-r m` or `-r month` | Monthly |
| `-r d` or `-r day` | Daily |
| `-r n` or `-r never` | None |

Each restart file name includes the time stamp at which it is valid:
`<prefix>_restart_YYYYMMDDhhmm.nc`. To resume from a restart, set
`initConditionFile` in the file manager to that file path and rerun
`summa.exe`.

If `statePath` is set in the file manager, restarts are written there
and SUMMA expects the initial conditions file to be there too. If
`statePath` is omitted, restarts go under `outputPath` and the initial
conditions file is read from `settingsPath`.

---

## Post-processing in Python

### With `xarray` directly

```python
import xarray as xr
import matplotlib.pyplot as plt

ds = xr.open_dataset("output/reynolds_constantDecayRate_1.nc")

ds["scalarSWE"].isel(hru=0).plot()
plt.title("SWE at Reynolds Mountain East")
plt.savefig("swe.png")
```

For a layer-time variable you write the slicing helper from above as a
function and apply it to each time step.

### With `pysumma`

`pysumma` (https://github.com/UW-Hydro/pysumma,
https://pysumma.readthedocs.io) is the canonical Python wrapper. Install
via conda for the cleanest dependency setup:

```bash
conda install -c conda-forge pysumma
```

The package exposes:

- `Simulation`: configure and execute a single SUMMA run from Python.
  Edits the file manager, decisions, output control, and trial
  parameters in memory, writes them out, and calls `summa.exe` for you.
- `Ensemble`: run many `Simulation`s with parameter or decision sweeps.
  Useful for sensitivity analyses and the kinds of structured experiments
  SUMMA was designed to support.
- `Distributed`: spatially distributed simulation with a parallel job
  manager.
- `Ostrich`: wraps the OSTRICH calibration tool around a `Simulation`
  for parameter optimization.

A minimal session:

```python
import pysumma as ps

sim = ps.Simulation(
    executable="/path/to/summa.exe",
    filemanager="/path/to/case_study/reynolds/fileManager_reynoldsConstantDecayRate.txt",
)

sim.decisions["alb_method"] = "varDecay"
sim.run()                     # blocks until SUMMA exits

ds = sim.output               # xarray.Dataset of the history file
ds["scalarSWE"].isel(hru=0).plot()
```

`pysumma` also bundles plot helpers for layer-time profiles and
Hovmoeller diagrams that handle the `*AndTime` slicing for you. See
the readthedocs site for the full helper list.

### Layer-resolved Hovmoeller plot recipe

```python
import numpy as np
import xarray as xr
import matplotlib.pyplot as plt

ds = xr.open_dataset("output/reynolds_variableDecayRate_1.nc")
hru = 0

n_t = ds.sizes["time"]
profiles = []
heights  = []
for t in range(n_t):
    n_layers = int(ds["nLayers"].isel(time=t, hru=hru).values)
    start    = int(ds["midTotoStartIndex"].isel(time=t, hru=hru).values) - 1
    end      = start + n_layers
    profiles.append(ds["mLayerTemp"].isel(midTotoAndTime=slice(start, end), hru=hru).values)
    heights.append(ds["mLayerHeight"].isel(midTotoAndTime=slice(start, end), hru=hru).values)

# Pad to a common length for plotting
max_n = max(len(p) for p in profiles)
T = np.full((n_t, max_n), np.nan)
H = np.full((n_t, max_n), np.nan)
for t, (p, h) in enumerate(zip(profiles, heights)):
    T[t, :len(p)] = p
    H[t, :len(h)] = h

# scatter or pcolormesh with masking on NaN
plt.pcolormesh(np.arange(n_t), np.arange(max_n), T.T)
```

This is the pattern that produces the canonical SUMMA snow-temperature
plot in `docs/assets/img/SUMMA_temperature_profile_example.png`.

---

## Where to next

- Reproduce the experiments that drive these output files: `case-studies.md`
- Trace where each variable enters the output: `architecture.md`
- Diagnose missing-value floods or wrong indexing: `debugging.md`
- Compare results across model decisions in batch: `model-decisions.md`
