# SUMMA Model Decisions

The single feature that defines SUMMA is the model decisions file: a flat
list of named choices that controls which flux parameterization, numerical
method, and boundary condition the solver uses. The conservation
equations and the adaptive time stepper are fixed; the decisions vary.

This page is the working reference for the 40 decisions, the options each
allows, and the practical implications of choosing one over another. The
master list of options lives in `build/source/engine/mDecisions.f90`. The
name-to-index lookup lives in
`build/source/dshare/get_ixname.f90:get_ixdecisions`.

---

## How decisions reach the code

```
modelDecisions.txt
  +--> read by mDecisions.f90 at startup
       +--> populates the model_decisions structure
            +--> consumed in select case branches inside the engine modules
                 (e.g., snowAlbedo.f90 picks based on alb_method)
```

To add a new option to an existing decision, you (1) add a named integer
in `mDecisions.f90`, (2) extend `get_ixdecisions` if needed, (3) extend
the `select case` in the relevant physics module, and (4) document it on
this page and the SUMMA Model Decisions readthedocs page. To add an
entirely new decision you also extend `iLookDECISIONS` in `var_lookup.f90`
and the metadata table in `popMetadat.f90`.

---

## The forty decisions, in order

The numbering follows the SUMMA Model Decisions documentation
(`docs/configuration/SUMMA_model_decisions.md` in the repo).

### 1. simulStart and 2. simulFinsh

Start and end times of the simulation as `'YYYY-MM-DD hh:mm'`. These also
appear in the file manager as `simStartTime` and `simEndTime`; the file
manager values take precedence in V3.0.0+.

### 3. tmZoneInfo

`ncTime`, `utcTime`, `localTime`. See `running-summa.md` for the offset
formula. `utcTime` is recommended for multi-time-zone domains.

### 4. soilCatTbl

Soil category dataset, names a row block in `SOILPARM.TBL`.

| Option | Notes |
|--------|-------|
| `STAS` | STATSGO soil categories |
| `STAS-RUC` | STATSGO with RUC modifications |
| `ROSETTA` | Rosetta-derived soil categories |

### 5. vegeParTbl

Vegetation category dataset, names a row block in `VEGPARM.TBL`.

| Option |
|--------|
| `USGS` |
| `MODIFIED_IGBP_MODIS_NOAH` |
| `plumberCABLE` |
| `plumberCHTESSEL` |
| `plumberSUMMA` |

### 6. soilStress

Function for the soil moisture control on stomatal resistance.

| Option | Description |
|--------|-------------|
| `NoahType` | Thresholded linear function of volumetric liquid water content |
| `CLM_Type` | Thresholded linear function of matric head |
| `SiB_Type` | Exponential of the log of matric head |

### 7. stomResist

Stomatal resistance scheme.

| Option | Notes |
|--------|-------|
| `BallBerry` | Standard Ball-Berry coupled with photosynthesis |
| `Jarvis` | Empirical Jarvis-style multiplicative stress functions |
| `simpleResistance` | Constant resistance for testing |
| `BallBerryFlex` | Flexible Ball-Berry variant with sub-options 8-13 |
| `BallBerryTest` | Development variant; not for production |

When `stomResist` is `BallBerry` or `BallBerryFlex`, decisions 8-13
become active.

### 8. bbTempFunc

Ball-Berry leaf temperature controls.

| Option |
|--------|
| `q10Func` |
| `Arrhenius` |

### 9. bbHumdFunc

Ball-Berry humidity controls on stomatal resistance.

| Option |
|--------|
| `humidLeafSurface` |
| `scaledHyperbolic` |

### 10. bbElecFunc

Ball-Berry dependence of photosynthesis on PAR.

| Option |
|--------|
| `linear` |
| `linearJmax` |
| `quadraticJmax` |

### 11. bbCO2point

Ball-Berry CO2 compensation point treatment.

| Option |
|--------|
| `origBWB` |
| `Leuning` |

### 12. bbNumerics

Ball-Berry iterative solver.

| Option |
|--------|
| `NoahMPsolution` |
| `newtonRaphson` |

### 13. bbAssimFnc

Ball-Berry carbon assimilation control.

| Option |
|--------|
| `colimitation` |
| `minFunc` |

### 14. bbCanIntg8

Ball-Berry leaf-to-canopy scaling.

| Option |
|--------|
| `constantScaling` |
| `laiScaling` |

### 15. num_method

Top-level numerical method.

| Option | Notes |
|--------|-------|
| `itertive` | Iterative implicit (default) |
| `non_iter` | Non-iterative |
| `itersurf` | Iterate the surface flux only |

### 16. fDerivMeth

Method to compute flux derivatives for the Jacobian.

| Option | Notes |
|--------|-------|
| `numericl` | Finite differences (slow but model-agnostic) |
| `analytic` | Hand-coded derivatives (faster, recommended for production) |

### 17. LAI_method

How LAI and SAI evolve.

| Option | Notes |
|--------|-------|
| `monTable` | Monthly table from `VEGPARM.TBL` (default) |
| `specified` | Read from forcing or trial parameters |

### 18. cIntercept

Canopy interception parameterization.

| Option |
|--------|
| `sparseCanopy` |
| `storageFunc` |
| `notPopulatedYet` |

### 19. f_Richards

Form of Richards' equation for soil water.

| Option | Notes |
|--------|-------|
| `moisture` | Theta-based; only stable in unsaturated soil |
| `mixdform` | Mixed form; handles unsaturated and saturated. Recommended. |

### 20. groundwatr

Groundwater representation.

| Option | Notes |
|--------|-------|
| `qTopmodl` | TOPMODEL-style baseflow |
| `bigBuckt` | Big bucket conceptual aquifer |
| `noXplict` | No explicit groundwater module |

### 21. hc_profile

Hydraulic conductivity profile with depth.

| Option |
|--------|
| `constant` |
| `pow_prof` |

### 22. bcUpprTdyn

Upper boundary condition for thermodynamics.

| Option | Notes |
|--------|-------|
| `presTemp` | Prescribed temperature |
| `nrg_flux` | Energy flux from atmosphere (recommended for coupled column) |
| `zeroFlux` | No flux |

### 23. bcLowrTdyn

Lower boundary condition for thermodynamics.

| Option |
|--------|
| `presTemp` |
| `zeroFlux` |

### 24. bcUpprSoiH

Upper boundary condition for soil hydrology.

| Option | Notes |
|--------|-------|
| `presHead` | Prescribed matric head |
| `liq_flux` | Liquid water flux from snowmelt or rainfall (recommended) |

### 25. bcLowrSoiH

Lower boundary condition for soil hydrology.

| Option | Notes |
|--------|-------|
| `presHead` | Prescribed head at the bottom |
| `bottmPsi` | Prescribed matric head |
| `drainage` | Free drainage (gravity flux only) |
| `zeroFlux` | No flux at the bottom |

### 26. veg_traits

Roughness length and displacement height for vegetation.

| Option | Reference |
|--------|-----------|
| `Raupach_BLM1994` | Raupach 1994 |
| `CM_QJRMS1988` | Choudhury and Monteith 1988 |
| `vegTypeTable` | Use the values from `VEGPARM.TBL` directly |

### 27. rootProfil

Vertical root profile.

| Option |
|--------|
| `powerLaw` |
| `doubleExp` |

### 28. canopyEmis

Canopy emissivity.

| Option |
|--------|
| `simplExp` |
| `difTrans` |

### 29. snowIncept

Snow interception by the canopy.

| Option | Reference |
|--------|-----------|
| `stickySnow` | Andreadis et al. 2009; sharp interception increase between -3 and 0 C |
| `lightSnow` | Hedstrom and Pomeroy 1998; slight decrease below -3 C |

### 30. windPrfile

Wind profile through the canopy.

| Option | Description |
|--------|-------------|
| `exponential` | Exponential decay |
| `logBelowCanopy` | Logarithmic decay |

### 31. astability

Atmospheric stability function.

| Option |
|--------|
| `standard` |
| `louisinv` |
| `mahrtexp` |

### 32. compaction

Snow compaction.

| Option |
|--------|
| `consettl` |
| `anderson` |

### 33. snowLayers

How snow layers are merged and split.

| Option | Notes |
|--------|-------|
| `jrdn1991` | Jordan 1991; up to 100 layers |
| `CLM_2010` | Community Land Model 2010; up to 5 layers, configurable |

### 34. thCondSnow

Thermal conductivity of snow.

| Option |
|--------|
| `tyen1965` |
| `melr1977` |
| `jrdn1991` |
| `smnv2000` |

### 35. thCondSoil

Thermal conductivity of soil.

| Option |
|--------|
| `funcSoilWet` |
| `mixConstit` |
| `hanssonVZJ` |

### 36. canopySrad

Method for canopy shortwave radiation.

| Option | Notes |
|--------|-------|
| `noah_mp` | Noah-MP two-stream |
| `CLM_2stream` | CLM two-stream |
| `UEB_2stream` | Utah Energy Balance two-stream |
| `NL_scatter` | Nijssen and Lettenmaier scattering |
| `BeersLaw` | Beers' law (simplest) |

### 37. alb_method

Snow albedo decay.

| Option | Notes |
|--------|-------|
| `conDecay` | Constant decay rate |
| `varDecay` | Variable decay rate that depends on snow temperature |

The Reynolds Mountain East case study compares these two; see
`case-studies.md`.

### 38. spatial_gw

Spatial representation of groundwater.

| Option |
|--------|
| `localColumn` |
| `singleBasin` |

`singleBasin` lets multiple HRUs in a GRU share a single conceptual
aquifer.

### 39. subRouting

Sub-grid routing.

| Option |
|--------|
| `timeDlay` |
| `qInstant` |

### 40. snowDenNew

Density of newly fallen snow.

| Option | Reference |
|--------|-----------|
| `hedAndPom` | Hedstrom and Pomeroy 1998; depends on air temperature |
| `anderson` | Anderson 1976 |
| `pahaut_76` | Pahaut 1976; depends on air temperature and wind speed |
| `constDens` | Constant 330 kg/m^3 |

---

## Mimicking other land surface models

A practical superpower of SUMMA is "model mimicry": pick the decisions
that reproduce the behavior of a published model. The Clark et al.
(2015b) paper documents the canonical mimics. A starting point:

### Noah-MP-like

```
stomResist       BallBerry
f_Richards       mixdform
groundwatr       bigBuckt
canopySrad       noah_mp
alb_method       conDecay
snowLayers       jrdn1991
thCondSnow       jrdn1991
thCondSoil       funcSoilWet
snowIncept       lightSnow
veg_traits       vegTypeTable
```

### CLM-like

```
stomResist       BallBerry
f_Richards       mixdform
groundwatr       bigBuckt
canopySrad       CLM_2stream
alb_method       varDecay
snowLayers       CLM_2010
thCondSoil       mixConstit
soilStress       CLM_Type
```

### VIC-like (column hydrology)

```
stomResist       Jarvis
f_Richards       mixdform
groundwatr       qTopmodl
canopySrad       BeersLaw
alb_method       conDecay
spatial_gw       singleBasin
subRouting       timeDlay
```

These are starting points, not exact reproductions. The fixed structural
core (operator splitting, adaptive time step, layer indexing) means a
SUMMA mimic does not bit-replicate the host model. It does, however,
isolate which physics options drive the differences when comparing to the
host model on the same forcing.

---

## Decision compatibility notes

- `stomResist = BallBerry` activates decisions 8 through 13. With
  `stomResist = Jarvis` those decisions are ignored but must still be
  syntactically present in the file.
- `groundwatr = noXplict` disables the aquifer; `spatial_gw` is then
  irrelevant. Provide a value anyway; it is read but not used.
- `f_Richards = moisture` is unstable when any soil layer reaches
  saturation. Use `mixdform` unless reproducing a moisture-form
  experiment.
- `snowLayers = CLM_2010` interacts with `zminLayer*` and `zmaxLayer*`
  parameters in `localParamInfo.txt`. Mixing `jrdn1991` decisions with
  CLM-tuned layer thresholds produces strange snowpack behavior.
- `bcUpprTdyn = nrg_flux` is required for a coupled energy-balance
  surface. Other choices are useful only for prescribed-temperature lab
  experiments.

---

## Where to next

- Set up the input files that consume these decisions: `running-summa.md`
- Reproduce the constant-vs-variable albedo experiment: `case-studies.md`
- Inspect what each option does in source: `architecture.md`
- Diagnose decision-related runtime failures: `debugging.md`
