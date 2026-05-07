# Running SUMMA

This page covers everything between a built `summa.exe` and a finished
NetCDF history file: the file manager, the eight required input files,
the command-line flags, and what to verify at each step.

---

## The single command

```
summa.exe -m <path/to/fileManager.txt> [options]
```

Everything else is configured inside the file manager and the files it
points to. The `-m` flag is required.

### Command-line flags

| Flag | Long form | Meaning |
|------|-----------|---------|
| `-m` | `--master` | Path to the file manager (required) |
| `-s` | `--suffix` | Append this string to every output file name |
| `-g` | `--gru` | Run a contiguous slice: `startGRU countGRU` |
| `-h` | `--hru` | Run a single HRU by 1-based index |
| `-r` | `--restart` | Restart cadence: `y`, `m`, `d`, or `never` |
| `-p` | `--progress` | Progress-message cadence: `m`, `d`, `h`, or `never` |
| `-c` | (none) | Constant output (no statistical aggregation) |
| `-v` | `--version` | Print version banner and exit |

The `-g` flag is the standard way to parallelize across HRUs by launching
many `summa.exe` processes, each handling a contiguous slice.

---

## The file manager (master configuration)

ASCII format. Comments start with `!` and run to end of line. Each entry
has the form `<keyword>  '<value>'` with the value in single quotes.
Order is irrelevant; pairing is by keyword.

The current parser version is `'SUMMA_FILE_MANAGER_V3.0.0'`. Older V1 and
V2 files must be migrated with `utils/convert_summa_config_v2_v3.py`.

### Required keywords (V3.0.0)

| Keyword | What it points to |
|---------|-------------------|
| `controlVersion` | Always `'SUMMA_FILE_MANAGER_V3.0.0'` for current code |
| `simStartTime` | `'YYYY-MM-DD hh:mm'`, end of the first time step |
| `simEndTime` | `'YYYY-MM-DD hh:mm'`, end of the last time step |
| `tmZoneInfo` | `'utcTime'`, `'localTime'`, or `'ncTime'` |
| `settingsPath` | Base path for ASCII config files |
| `forcingPath` | Base path for forcing NetCDF files |
| `outputPath` | Base path for written output |
| `decisionsFile` | Model decisions file (relative to `settingsPath`) |
| `outputControlFile` | Output control file (relative to `settingsPath`) |
| `attributeFile` | Local attributes NetCDF (relative to `settingsPath`) |
| `globalHruParamFile` | HRU-level parameter defaults ASCII (relative to `settingsPath`) |
| `globalGruParamFile` | GRU-level parameter defaults ASCII (relative to `settingsPath`) |
| `forcingListFile` | List of forcing files ASCII (relative to `settingsPath`) |
| `initConditionFile` | Initial conditions NetCDF (relative to `settingsPath`) |
| `trialParamFile` | Trial parameters NetCDF (relative to `settingsPath`) |
| `vegTableFile` | Vegetation parameter table (defaults to `VEGPARM.TBL`) |
| `soilTableFile` | Soil parameter table (defaults to `SOILPARM.TBL`) |
| `generalTableFile` | General parameter table (defaults to `GENPARM.TBL`) |
| `noahmpTableFile` | Noah-MP parameter table (defaults to `MPTABLE.TBL`) |
| `outFilePrefix` | Prefix for every output file name |

Optional:
- `statePath`: separate base path for state files (initial conditions on
  read, restart files on write). If absent, the initial conditions file is
  read from `settingsPath` and restart files are written under
  `outputPath`.

### Worked example (Reynolds Mountain East, constant decay rate)

```
controlVersion       'SUMMA_FILE_MANAGER_V3.0.0'
simStartTime         '2005-09-01 00:00'
simEndTime           '2006-09-01 00:00'
tmZoneInfo           'localTime'
settingsPath         './site_settings/'
forcingPath          './forcing/'
outputPath           './output/'
forcingListFile      'forcingFileList.txt'
outFilePrefix        'reynolds_constantDecayRate'
decisionsFile        'modelDecisions_reynoldsConstantDecayRate.txt'
outputControlFile    'outputControl.txt'
initConditionFile    'initialState.nc'
attributeFile        'localAttributes.nc'
globalHruParamFile   '../../base_settings/localParamInfo.txt'
globalGruParamFile   '../../base_settings/basinParamInfo.txt'
vegTableFile         '../../base_settings/VEGPARM.TBL'
soilTableFile        '../../base_settings/SOILPARM.TBL'
generalTableFile     '../../base_settings/GENPARM.TBL'
noahmpTableFile      '../../base_settings/MPTABLE.TBL'
trialParamFile       'trialParams_reynoldsConstantDecayRate.nc'
```

This is `case_study/reynolds/fileManager_reynoldsConstantDecayRate.txt`
verbatim. Note the relative paths into `case_study/base_settings/` for the
parameter tables.

---

## Time conventions

- All time stamps are **period-ending**. A forcing record stamped
  `2005-09-01 03:00` represents the average over the three preceding
  hours (when `data_step = 10800`).
- `simStartTime` is the end of the first time step, not the beginning.
- `tmZoneInfo` controls the offset between the time stamps in the forcing
  file and local solar time:

| Option | Offset formula |
|--------|---------------|
| `ncTime` | `timeOffset = longitude/15 - ncTimeOffset`, where `ncTimeOffset` is parsed from the `units` attribute on the `time` variable |
| `utcTime` | `timeOffset = longitude/15`. Forcing must be in UTC. Recommended for large multi-time-zone domains |
| `localTime` | `timeOffset = 0`. Forcing is already in local solar time |

Mismatches silently shift solar zenith and ruin the radiation forcing.
For a multi-decade run on a single point, this manifests as a small
seasonal bias that is easy to miss.

---

## The eight input files

```
fileManager.txt   <- master configuration
+-- decisionsFile          (ASCII): which physics options
+-- outputControlFile      (ASCII): which variables to write, at what frequency, with which statistics
+-- attributeFile          (NetCDF): time-invariant per-HRU descriptors
+-- globalHruParamFile     (ASCII): default HRU-level parameter values
+-- globalGruParamFile     (ASCII): default GRU-level parameter values
+-- vegTableFile, soilTableFile, generalTableFile, noahmpTableFile  (ASCII): Noah-MP parameter tables
+-- trialParamFile         (NetCDF): per-HRU/GRU parameter overrides
+-- initConditionFile      (NetCDF): cold-start or warm-start state
+-- forcingListFile        (ASCII): list of NetCDF forcing files
       +-- forcing files   (NetCDF, one or many): T, q, p, SW, LW, precip, wind
```

The order in which SUMMA processes parameters matters. It is:

1. `globalHruParamFile` and `globalGruParamFile` (spatially constant
   defaults).
2. The Noah-MP tables, which overwrite the defaults based on
   `vegTypeIndex` and `soilTypeIndex` per HRU.
3. The `trialParamFile`, which overwrites everything for the listed
   variables on the listed HRUs / GRUs.

Trial parameters always win.

---

## File 1: model decisions (ASCII)

Each line is `<decisionName>  <option>`. Names and options are
case-sensitive. The full registry is in `build/source/engine/mDecisions.f90`.

Minimum example for a snow-dominated point simulation:

```
soilCatTbl       STAS
vegeParTbl       MODIFIED_IGBP_MODIS_NOAH
soilStress       NoahType
stomResist       BallBerry
bbTempFunc       q10Func
bbHumdFunc       humidLeafSurface
bbElecFunc       linearJmax
bbCO2point       origBWB
bbNumerics       NoahMPsolution
bbAssimFnc       colimitation
bbCanIntg8       constantScaling
num_method       itertive
fDerivMeth       analytic
LAI_method       monTable
cIntercept       sparseCanopy
f_Richards       mixdform
groundwatr       noXplict
hc_profile       constant
bcUpprTdyn       nrg_flux
bcLowrTdyn       zeroFlux
bcUpprSoiH       liq_flux
bcLowrSoiH       drainage
veg_traits       Raupach_BLM1994
rootProfil       powerLaw
canopyEmis       difTrans
snowIncept       lightSnow
windPrfile       logBelowCanopy
astability       louisinv
compaction       anderson
snowLayers       jrdn1991
thCondSnow       jrdn1991
thCondSoil       mixConstit
canopySrad       BeersLaw
alb_method       varDecay
spatial_gw       localColumn
subRouting       qInstant
snowDenNew       hedAndPom
```

Detailed semantics for each decision are in `model-decisions.md`.

---

## File 2: output control (ASCII)

Each non-comment line specifies one variable. Format:

```
<varName> | <outFreq> | <sum> | <inst> | <mean> | <var> | <min> | <max> | <mode>
```

- `<varName>`: any variable from `iLook_*` in `dshare/var_lookup.f90`.
- `<outFreq>`: how many forcing time steps per output write. `1` writes
  every forcing step.
- The seven trailing 0/1 flags request which statistics to keep. With
  `<outFreq> > 1` you can ask for `mean`, `min`, `max`, `sum`, etc.
  computed across the aggregation window. With `<outFreq> = 1` only
  `inst` is meaningful.

Example fragment:

```
! varName              | freq | sum | inst | mean | var | min | max | mode
scalarSenHeatTotal     | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
scalarLatHeatTotal     | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
scalarSWE              | 24   | 0   | 0    | 1    | 0   | 1   | 1   | 0
mLayerVolFracWat       | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
mLayerHeight           | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
nSnow                  | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
nSoil                  | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
midTotoStartIndex      | 1    | 0   | 1    | 0    | 0   | 0   | 0   | 0
```

Optional global directive:

```
outputPrecision | float    ! or 'single', or 'double' (default)
```

The output controller errors out on unknown variable names. The fastest
way to discover what is available is to grep `var_lookup.f90` for the
relevant `iLook_*` block.

---

## File 3: attributes (NetCDF)

Two dimensions, `gru` and `hru`. The variables below are all required.

| Variable | Dim | Type | Units | Notes |
|----------|-----|------|-------|-------|
| `hruId` | `hru` | int | - | Unique numeric ID |
| `gruId` | `gru` | int | - | Unique numeric ID |
| `hru2gruId` | `hru` | int | - | The `gruId` of the parent GRU |
| `downHRUindex` | `hru` | int | - | Index of downslope HRU in the same GRU; 0 means basin outlet |
| `longitude` | `hru` | double | decimal degrees east | West is negative |
| `latitude` | `hru` | double | decimal degrees north | South is negative |
| `elevation` | `hru` | double | m | |
| `HRUarea` | `hru` | double | m^2 | |
| `tan_slope` | `hru` | double | m m-1 | |
| `contourLength` | `hru` | double | m | Width of hillslope parallel to a stream; used in `groundwatr.f90` |
| `slopeTypeIndex` | `hru` | int | - | |
| `soilTypeIndex` | `hru` | int | - | Index into `SOILPARM.TBL` |
| `vegTypeIndex` | `hru` | int | - | Index into `VEGPARM.TBL` |
| `mHeight` | `hru` | double | m | Measurement height for forcing data, above bare ground |

The `hruId` and `gruId` orders in this file must match the orders in the
forcing, initial conditions, and trial parameter files. When a GRU
contains multiple HRUs, the HRUs for that GRU must occupy contiguous
indices.

---

## File 4 and 5: HRU and GRU parameter defaults (ASCII)

The first non-comment line is a Fortran format string in single quotes.
Every subsequent line follows it. Example format:

```
'(a25,1x,3(a1,1x,f12.4,1x))'        ! format string
```

Then four columns per row: parameter name, default value, lower bound,
upper bound.

```
upperBoundHead            |      -0.7500 |    -100.0000 |      -0.0100
fieldCapacity             |       0.2000 |       0.0000 |       1.0000
heightCanopyTop           |      20.0000 |       0.0500 |     100.0000
```

The HRU parameter file lists every variable in
`var_lookup.f90:iLook_param`. The GRU parameter file lists every variable
in `var_lookup.f90:iLook_bpar` (basin parameters).

The lower and upper bounds are not currently enforced, but you must
specify them.

---

## File 6: trial parameters (NetCDF)

Per-HRU or per-GRU overrides. Optional but almost always used in real
applications.

- Has a `gru` and/or `hru` dimension.
- Includes `hruId` and/or `gruId` for mapping.
- Includes any subset of the variables in `iLook_param` (HRU dimension)
  or `iLook_bpar` (GRU dimension).

These values overwrite both the spatially constant defaults and the
Noah-MP table values for the named variables.

---

## File 7: initial conditions / restart (NetCDF)

Required even for a cold start. Dimensions: `hru`, `scalarv`, `spectral`,
`ifcSoil`, `ifcToto`, `midSoil`, `midToto`. No `time` dimension.

Required variables (subset of `iLook_prog`):

| Variable | Dim | Type | Units |
|----------|-----|------|-------|
| `dt_init` | `scalarv, hru` | double | s |
| `nSoil`, `nSnow` | `scalarv, hru` | int | - |
| `scalarCanopyIce`, `scalarCanopyLiq` | `scalarv, hru` | double | kg m-2 |
| `scalarCanairTemp`, `scalarCanopyTemp` | `scalarv, hru` | double | K |
| `scalarSnowAlbedo` | `scalarv, hru` | double | - |
| `scalarSnowDepth`, `scalarSWE`, `scalarSfcMeltPond` | `scalarv, hru` | double | m, kg m-2 |
| `scalarAquiferStorage` | `scalarv, hru` | double | m |
| `iLayerHeight` | `ifcToto, hru` | double | m |
| `mLayerDepth`, `mLayerTemp`, `mLayerVolFracIce`, `mLayerVolFracLiq` | `midToto, hru` | double | (varies) |
| `mLayerMatricHead` | `midSoil, hru` | double | m |

SUMMA computes `mLayerHeight`, `scalarCanopyWat`, `spectralSnowAlbedoDiffuse`,
`scalarSurfaceTemp`, and `mLayerVolFracWat` internally; do not include
them in the restart file or they will be ignored.

The restart cadence is set on the command line via `-r`. With `-r m` the
file `<prefix>_restart_YYYYMMDDhhmm.nc` is written at the start of each
month.

---

## File 8: forcing list (ASCII) and forcing files (NetCDF)

The forcing list has one quoted file name per line:

```
'wrr_pluvial_sf_flathead_1980-1989.nc'
'wrr_pluvial_sf_flathead_1990-1999.nc'
'wrr_pluvial_sf_flathead_2000-2009.nc'
```

Each NetCDF forcing file must contain dimensions `hru` and `time`, plus:

| Variable | Dim | Type | Units | Long name |
|----------|-----|------|-------|-----------|
| `data_step` | scalar | double | s | Length of one forcing time step (must match across files in the list) |
| `hruId` | `hru` | int | - | HRU IDs in the same order as the attributes file |
| `time` | `time` | double | `<units> since YYYY-MM-DD hh:mm` | Period-ending |
| `pptrate` | `time, hru` | double | kg m-2 s-1 | Precipitation rate |
| `SWRadAtm` | `time, hru` | double | W m-2 | Downward shortwave at the upper boundary |
| `LWRadAtm` | `time, hru` | double | W m-2 | Downward longwave at the upper boundary |
| `airtemp` | `time, hru` | double | K | At measurement height |
| `windspd` | `time, hru` | double | m s-1 | At measurement height |
| `airpres` | `time, hru` | double | Pa | At measurement height |
| `spechum` | `time, hru` | double | g g-1 | At measurement height |

The CF time units string format is critical. SUMMA parses
`<units> since YYYY-MM-DD hh:mm[ +HH:MM]`. The optional zone offset is
used only when `tmZoneInfo = 'ncTime'`.

A note on dimension order: SUMMA expects `(hru, time)` as written. NetCDF
files written by Python's `xarray` or `netCDF4` are typically `(time, hru)`
in Python's row-major convention, which Fortran reads as `(hru, time)`.
This is normally fine. If you write the file from Fortran directly, you
get the order Fortran sees.

---

## A typical run cycle

```bash
# 1. Configure
$EDITOR fileManager.txt           # set paths and dates
$EDITOR settings/modelDecisions.txt
$EDITOR settings/outputControl.txt

# 2. Validate
ncdump -h settings/localAttributes.nc | head -40
ncdump -h forcing/forcing_2010.nc | head -40
ls -la settings/localParamInfo.txt settings/basinParamInfo.txt

# 3. Cold-start the simulation
summa.exe -m fileManager.txt -r m -p d

# 4. Inspect first output
ls output/
ncdump -h output/<prefix>_1.nc | head -40

# 5. Restart from a written state file
$EDITOR fileManager.txt           # change initConditionFile to the restart file
summa.exe -m fileManager.txt -r m
```

---

## Common runtime failures and the file they point at

| Error message contains | First place to look |
|------------------------|---------------------|
| `cannot open ... fileManager` | The path to `-m`; remember it is interpreted relative to the current working directory |
| `cannot find decisionsFile` | `decisionsFile` keyword in the file manager; path is relative to `settingsPath` |
| `requested decision not in registry` | Typo or unsupported option in `modelDecisions.txt` |
| `requested variable not in metadata` | Typo or removed variable in `outputControl.txt` |
| `attribute file dimension mismatch` | `gru` or `hru` count in `localAttributes.nc` does not match what `forcingFileList.txt` files contain |
| `forcing file time units malformed` | `units` string on the `time` variable; use `seconds since YYYY-MM-DD hh:mm` |
| `simStartTime not found in forcing` | Forcing time stamps do not include `simStartTime`; remember they are period-ending |
| `restart variable X not in restart file` | You changed decisions between the run that wrote the restart and the run that reads it; cold-start instead |

For a richer guide to runtime failures see `debugging.md`.

---

## Where to next

- Pick the right physics options: `model-decisions.md`
- Slice the layer-resolved output: `output-and-postprocess.md`
- Reproduce the published case studies: `case-studies.md`
- Diagnose runtime failures: `debugging.md`
