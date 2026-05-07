# SUNDIALS and Solvers in SUMMA

The default SUMMA distributed by `NCAR/summa` uses an in-house implicit
solver built on LAPACK band linear systems. The CH-Earth fork
(`CH-Earth/summa`) has a development line that swaps that solver out for
SUNDIALS, exposing the IDA implicit DAE integrator and the KINSOL
nonlinear solver. This page covers both: what the default solver does,
where SUNDIALS plugs in, and how to build the SUNDIALS variant.

References:
- Clark et al. 2021, "The Numerical Implementation of Land Models", J.
  Hydrometeor., doi:10.1175/JHM-D-20-0175.1. The "laugh tests" paper.
- SUNDIALS: https://sundials.readthedocs.io
- CH-Earth fork: https://github.com/CH-Earth/summa

---

## The default in-house solver (NCAR/summa)

### Where it lives

| File | Role |
|------|------|
| `engine/coupled_em.f90` | Per-HRU driver of the adaptive time loop |
| `engine/opSplittin.f90` | Operator-splitting controller (full coupled, splitting by domain, scalar fallback) |
| `engine/systemSolv.f90` | Assembles one implicit time step |
| `engine/eval8summa.f90` | Evaluates the residual vector |
| `engine/computFlux.f90` | Computes every flux at the trial state |
| `engine/computResid.f90` | Computes the residual entry by entry |
| `engine/computJacob.f90` | Computes the Jacobian, analytic or numeric |
| `engine/summaSolve.f90` | Solves the linear system at each Newton step |
| `engine/matrixOper.f90` | Wraps the LAPACK band and dense paths |
| `engine/varSubstep.f90` | Adaptive sub-step controller |
| `engine/updateVars.f90` | Project state increments back into physical bounds |
| `engine/getVectorz.f90` | Pack and unpack the model state vector |

### The coupled state vector

For a single HRU at one time step, the unknowns SUMMA solves for are:

- Canopy air temperature (1 scalar, when `computeVegFlux = .true.`)
- Canopy temperature (1 scalar)
- Canopy total water (1 scalar)
- Layer temperatures (one per snow + soil layer)
- Layer total water (one per snow + soil layer)
- Aquifer storage (1 scalar, when groundwater is explicit)

Sizing varies with `nSnow`. SUMMA reassembles the state vector at the
start of each implicit step.

### The Jacobian structure

The Jacobian is banded with a small bandwidth driven by inter-layer
coupling: each layer is coupled to itself and its two neighbors plus the
canopy and surface fluxes that touch it. The band parameters live in
`globalData.f90`:

```fortran
ku = number of super-diagonal bands
kl = number of sub-diagonal bands
```

`summaSolve.f90` calls LAPACK's `dgbsv` for the band path and `dgesv`
for the dense path. The choice is driven by the operator-splitting
mode: full-coupled steps use the band solver; scalar fallback uses
direct algebra without LAPACK.

### Operator splitting modes

`opSplittin.f90` tries successively:

1. **Fully coupled**: solve canopy + soil + snow simultaneously. Lowest
   error, highest cost per step.
2. **Split by domain**: solve canopy first, then snow + soil. Drops
   cross-domain Jacobian entries.
3. **State-by-state scalar fallback**: pick off one prognostic variable
   at a time. Used as a last resort when the coupled and split paths
   fail to converge.

If the scalar fallback also fails, `varSubstep.f90` reduces the
sub-step length and retries from mode 1.

### Convergence and tolerances

SUMMA uses a dual relative + absolute tolerance on the residual norm.
The thresholds are hard-wired in `eval8summa.f90` and tuned for the
typical scale of land-surface fluxes (W m-2) and state variables
(K, m3 m-3). When you see "convergence failure" messages the first
diagnostic to add is the residual vector at the failing iteration; print
it from `computResid.f90`.

---

## The SUNDIALS variant (CH-Earth/summa)

### What changes

CH-Earth replaces the in-house Newton iteration with SUNDIALS solvers:

- **IDA**: an implicit DAE integrator. Suitable when the conservation
  equations are rewritten as a system of differential-algebraic
  equations and the time stepping itself is delegated to SUNDIALS.
- **KINSOL**: a Newton-Krylov nonlinear solver. Replaces the
  hand-coded Newton loop in `summaSolve.f90`.

The conservation equations and the flux parameterizations do not
change. Only the solver kernel does. This is exactly the separation that
SUMMA's design aims for: swap the numerical method without touching the
physics.

### Why use it

Three reasons documented in Clark et al. (2021):

1. Adaptive order and step-size control inside IDA reduces wasted
   work near rapid transitions (snow accumulation, freeze-thaw, soil
   wetting fronts).
2. KINSOL exposes Newton-Krylov methods that scale to bigger state
   vectors (relevant for high-resolution snowpack discretizations).
3. SUNDIALS is independently tested and maintained; bugs in the
   nonlinear solver are not SUMMA's problem.

### Where SUNDIALS plugs in

The CH-Earth source includes:

- `engine/summaSolve4ida.f90`, `summaSolveSundials.f90`: SUNDIALS-based
  replacements for `summaSolve.f90`.
- `engine/computJacobSundials.f90`, `computJacobSetup.f90`: Jacobian
  computation in SUNDIALS' band format.
- `engine/eval8summaSundials.f90`: residual evaluation in SUNDIALS form.
- `engine/tol4ida.f90`: per-component tolerance setup for IDA.
- `engine/type4ida.f90`, `type4kinsol.f90`: derived types that wrap
  SUMMA state for SUNDIALS callbacks.

A new model decision is added to choose between the legacy and SUNDIALS
solvers. In the CH-Earth `mDecisions.f90`, the `num_method` decision
gains options like `numrec`, `homegrown`, `kinsol`, and `ida`.

### Build prerequisites

In addition to the standard NetCDF + LAPACK dependencies you need:

- SUNDIALS 6.x (older versions lack the F2003 module interfaces SUMMA
  uses). Build with:
  ```bash
  cmake -B build_sundials -S sundials-6.5.0 \
    -DCMAKE_INSTALL_PREFIX=$HOME/sundials \
    -DBUILD_FORTRAN_MODULE_INTERFACE=ON \
    -DEXAMPLES_INSTALL=OFF \
    -DBUILD_SHARED_LIBS=ON
  cmake --build build_sundials -j
  cmake --install build_sundials
  ```
  This installs `libsundials_ida.so`, `libsundials_kinsol.so`,
  `libsundials_fida_mod.so`, and `libsundials_fkinsol_mod.so` plus
  Fortran module files.
- Sometimes a sparse linear solver library: SuiteSparse or KLU, depending
  on which IDA linear solver you choose.

### Build SUMMA against SUNDIALS (CMake)

```bash
git clone https://github.com/CH-Earth/summa.git
cd summa/build/cmake
export SUNDIALS_DIR=$HOME/sundials
cmake -B cmake_build -S ../. \
  -DSUNDIALS_DIR=$SUNDIALS_DIR \
  -DUSE_SUNDIALS=ON
cmake --build cmake_build -j
```

The CH-Earth CMakeLists adds a `USE_SUNDIALS` switch that conditionally
compiles the SUNDIALS solver files and links the SUNDIALS libraries.
Without `-DUSE_SUNDIALS=ON` the build falls back to the legacy LAPACK
solver and produces an executable identical in behavior to NCAR/summa.

### Build SUMMA against SUNDIALS (Makefile)

The CH-Earth Makefile sets `INCLUDES_SUNDIALS` and `LIBRARIES_SUNDIALS`
that you populate manually:

```
INCLUDES_SUNDIALS  = -I$(SUNDIALS_DIR)/include -I$(SUNDIALS_DIR)/fortran
LIBRARIES_SUNDIALS = -L$(SUNDIALS_DIR)/lib -lsundials_ida -lsundials_kinsol \
                     -lsundials_fida_mod -lsundials_fkinsol_mod \
                     -lsundials_fcore_mod -lsundials_core -lsundials_nvecserial
```

Set `USE_SUNDIALS = yes` near the top of `Makefile` and rebuild from
clean.

### Choosing IDA or KINSOL at runtime

In the model decisions file:

```
num_method  ida              # adaptive implicit DAE integrator from SUNDIALS
# or
num_method  kinsol           # Newton-Krylov nonlinear solver replacing SUMMA's Newton loop
# or
num_method  itertive         # legacy in-house iterative solver
```

(Decision names follow CH-Earth's `mDecisions.f90`. The exact strings
have evolved across CH-Earth releases; check the file in your checkout.)

Tolerance overrides can be passed via additional decisions (for IDA) or
hard-coded in `tol4ida.f90`. Per-component tolerances are essential for
mixed-magnitude state vectors: temperature variables have units of K,
volumetric water content is dimensionless, matric head is meters, all
in the same vector.

### Diagnostics specific to the SUNDIALS path

When IDA fails to converge it emits messages with codes like
`IDA_CONV_FAIL` or `IDA_LSETUP_FAIL`. The fastest first-line response:

1. Print the SUNDIALS error message verbatim, then the time step at
   which it occurred.
2. Check that `tol4ida.f90` is setting per-component tolerances at the
   right scale for every state. A common cause is a tolerance
   appropriate for temperature (1e-3 K) being applied to volumetric
   water content (where 1e-3 is a 0.1% absolute error, often too tight).
3. Reduce the maximum step size SUNDIALS is allowed to take. IDA
   defaults can be too aggressive for the snowpack-melt transition.
4. Switch to the legacy solver (`num_method itertive`) and confirm the
   problem is solver-specific rather than a physics or input issue.

---

## Choosing between solvers

| Situation | Recommendation |
|-----------|----------------|
| Standard catchment, daily output, single point | Legacy in-house solver (NCAR/summa default) |
| High-resolution snowpack, fine vertical grid, many freeze-thaw transitions | SUNDIALS IDA |
| Domain with thousands of GRUs, want adaptive load balance | Legacy solver, use `-g` to parallelize across GRUs |
| Reproducing Clark et al. 2021 numerics study | SUNDIALS variant from CH-Earth |
| Coupling SUMMA into a host that uses SUNDIALS already | SUNDIALS variant for solver consistency |

The default solver is well-tested and fast for typical hydrology
applications. SUNDIALS pays off when the physics is stiff and the cost
of adaptive step control beats the per-step overhead of SUNDIALS' more
elaborate machinery.

---

## Where to next

- Trace where the solver sits in the call chain: `architecture.md`
- Diagnose convergence failures: `debugging.md`
- Pick `num_method` and friends: `model-decisions.md`
- Build the CH-Earth fork from scratch: `getting-started.md` (start
  with the `CH-Earth/summa` repo instead of `NCAR/summa`)
