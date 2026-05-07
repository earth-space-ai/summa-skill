# SUMMA Case Studies

The `case_study/` directory inside the SUMMA source tree contains a
ready-to-run reference experiment: Reynolds Mountain East. Beyond that,
the canonical SUMMA case studies from Clark et al. (2015b) (Wrightwood,
Senatorial Snow, Aspen) live in a separate test-cases archive
distributed from the NCAR project page. This page documents both.

---

## What ships in the source tree

```
case_study/
|-- base_settings/                 <- defaults shared by every experiment
|   |-- VEGPARM.TBL
|   |-- SOILPARM.TBL
|   |-- GENPARM.TBL
|   |-- MPTABLE.TBL
|   |-- localParamInfo.txt         <- HRU-level parameter defaults
|   |-- basinParamInfo.txt         <- GRU-level parameter defaults
|   `-- readme.md
|-- reynolds/                      <- Reynolds Mountain East albedo experiment
|   |-- fileManager_reynoldsConstantDecayRate.txt
|   |-- fileManager_reynoldsVariableDecayRate.txt
|   |-- forcing/
|   |-- site_settings/
|   |   |-- modelDecisions_reynoldsConstantDecayRate.txt
|   |   |-- modelDecisions_reynoldsVariableDecayRate.txt
|   |   |-- outputControl.txt
|   |   |-- localAttributes.nc
|   |   |-- initialState.nc
|   |   |-- forcingFileList.txt
|   |   |-- trialParams_reynoldsConstantDecayRate.nc
|   |   `-- trialParams_reynoldsVariableDecayRate.nc
|   |-- output/                    <- empty until you run the model
|   `-- evaluation/                <- observation data and a sample notebook
`-- readme.md
```

The split between `base_settings/` (shared) and `reynolds/site_settings/`
(experiment-specific) mirrors the standard SUMMA project layout.

---

## Reynolds Mountain East: the snow-albedo decay experiment

### What it tests

Reynolds Mountain East is a snow-dominated site in southwestern Idaho
(43.06 N, -116.78 E, 2093 m) that has been instrumented since 1966 by
the USDA-ARS Northwest Watershed Research Center. The case study compares
two snow-albedo decay parameterizations from the SUMMA model decisions
table:

- `alb_method = conDecay`: constant decay rate (the simpler scheme).
- `alb_method = varDecay`: variable decay rate that depends on snow
  surface temperature (faster decay when the snow is warm).

Everything else is held identical between the two runs. The expected
finding is that the variable-decay scheme produces a faster spring
melt-out and matches observed snow depth more closely. This reproduces
Figure 6a in Clark et al. (2015b).

### How to run it

```bash
cd summa/case_study/reynolds
../../bin/summa.exe -m fileManager_reynoldsConstantDecayRate.txt
../../bin/summa.exe -m fileManager_reynoldsVariableDecayRate.txt
```

Each run writes one history NetCDF in `output/`. Run time is well under
a minute on a laptop.

### File manager (constant decay)

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

The variable-decay file manager differs only in three lines:
`outFilePrefix`, `decisionsFile`, and `trialParamFile`.

### Evaluation

`case_study/reynolds/evaluation/` contains daily observed snow depth
and a sample Python notebook. The notebook opens both output files,
slices the `scalarSnowDepth` variable, and plots both simulations
against the observations. The variable-decay run melts out about a week
earlier and tracks observations more closely through April-May.

### What this experiment teaches

1. The mechanics of running two SUMMA simulations that differ only in
   one decision.
2. The mechanics of using a `trialParamFile` to perturb specific
   parameters.
3. How a single line in `modelDecisions.txt` can change a downstream
   physical signal (snow melt-out date).
4. The output post-processing pattern in `output-and-postprocess.md`.

---

## The published Clark et al. (2015b) test cases

The original SUMMA publication ran four flagship cases. These are
distributed separately from the source tree. Get them from the NCAR
SUMMA project page (https://ral.ucar.edu/projects/summa) under "Test
Cases". Versioning is important: the test-case archive must match your
SUMMA version (file manager format and parameter names change between
major versions).

| Site | Climate | What it stresses | Key reference |
|------|---------|------------------|---------------|
| **Reynolds Mountain East** (RME), Idaho | Snow-dominated forest | Snow albedo decay, canopy snow interception, melt timing | Reba et al. 2011, Storck et al. 2002 |
| **Senatorial Snow Field**, Wyoming | Open snow field | Snow accumulation in absence of canopy; snowmelt without vegetation; effects of new-snow density choice | Pomeroy et al. 1998 |
| **Wrightwood**, California | Snow-dominated chaparral, dry summer | Energy-balance snowmelt; soil-moisture drawdown after snowmelt; Mediterranean climate | Bales et al. 2006 |
| **Aspen**, Colorado | Aspen forest with subcanopy snow | Canopy radiation transfer below sparse canopy; subcanopy snow energy budget | Mahat and Tarboton 2012 |

Each test case ships with:

```
testCases_data/
+-- settings/<siteName>/
|   |-- summa_zFileManager_<siteName>.txt
|   |-- summa_zParamTrial_<experiment>.nc
|   |-- summa_zLocalAttributes_<siteName>.nc
|   |-- summa_zInitialCond_<siteName>.nc
|   |-- summa_zModelDecisions_<experiment>.txt
|   `-- summa_zOutputControl.txt
+-- forcing/<siteName>/
|   `-- *.nc
+-- verification/<siteName>/
|   `-- expected output for diff comparison
+-- install_local_setup.bash       (rewrites paths inside file managers to the local install)
`-- run_local_setup.bash           (loops summa.exe over every experiment)
```

### Standard workflow for the test cases

```bash
# 1. Get the archive
wget https://ral.ucar.edu/sites/default/files/public/projects/structure-unifying-multiple-modeling-alternatives-summa/summatestcases-<version>.tar.gz
tar xzf summatestcases-<version>.tar.gz
cd summaTestCases_<version>

# 2. Edit install_local_setup.bash to point at your built summa.exe and your data root
$EDITOR install_local_setup.bash
./install_local_setup.bash

# 3. Run all experiments
./run_local_setup.bash

# 4. Compare to verification output
ls verification/
ncdump -h verification/wrightwood/<...>.nc | head
```

The run script loops over every site and every experiment. A full sweep
on a laptop takes minutes for the point cases.

---

## Adding your own case study

The minimal recipe to wedge in a new site:

1. Create `case_study/<mysite>/` mirroring `case_study/reynolds/`.
2. Copy the `base_settings/` tables. Most of the time you do not need
   to modify them.
3. Build `localAttributes.nc` with one HRU and one GRU for a point
   simulation.
4. Build `initialState.nc` with sensible cold-start values for soil
   moisture, soil temperature, and zero snow.
5. Build the forcing NetCDFs from your meteorology data, observing the
   period-ending convention and the time zone.
6. Write `modelDecisions.txt` with the physics options you want.
7. Write `outputControl.txt` listing the variables you care about.
8. Write the file manager `fileManager.txt` with paths into your
   directories.
9. Run `summa.exe -m fileManager.txt`.

The Reynolds Mountain East directory is a good template; copy it and
modify in place rather than starting from a blank directory. The
boilerplate of `forcingFileList.txt` and `trialParams.nc` is easier to
understand by example than to write from spec.

---

## Where to next

- Choose physics options for your case: `model-decisions.md`
- Set up the input files for a new case: `running-summa.md`
- Inspect and visualize the output: `output-and-postprocess.md`
- Diagnose case-specific failures: `debugging.md`
