# Getting Started with SUMMA

This guide walks from zero to a working `summa.exe`. After this, see
`running-summa.md` to launch a real simulation, or `case-studies.md` to
reproduce the Reynolds Mountain East experiment that ships with the source.

---

## Step 1: Pick the right repository

There are two upstream copies of SUMMA you may encounter. Choose based on
what you need.

| Repo | When to use | URL |
|------|-------------|-----|
| `NCAR/summa` | Stable canonical source, GPLv3, most widely cited | https://github.com/NCAR/summa |
| `CH-Earth/summa` | Active development; required for the SUNDIALS-coupled implicit DAE solver, multilingual options, and recent numerics work by Clark, Spiteri, et al. | https://github.com/CH-Earth/summa |

For most users `NCAR/summa` master (currently v3.3.0) is the right choice.
If you need adaptive implicit time stepping with IDA or KINSOL, follow
`sundials-and-solvers.md` and use the CH-Earth fork.

Branches in `NCAR/summa`:
- `master`: latest tagged release (recommended for users).
- `develop`: bleeding-edge integration branch (recommended for contributors).

---

## Step 2: Clone

```bash
git clone https://github.com/NCAR/summa.git
cd summa
```

Tag-based clone if you want a specific release:

```bash
git clone --branch v3.3.0 --depth 1 https://github.com/NCAR/summa.git
```

---

## Step 3: Install dependencies

Three libraries plus a Fortran compiler are required.

### Fortran compiler

- `gfortran` version 6 or higher (free).
- `ifort` version 17.x or `ifx` (Intel) for performance.

SUMMA does not use compiler-specific extensions, so other Fortran 2003
compilers should work but are not regularly tested.

### NetCDF (C and Fortran)

SUMMA reads forcing, attributes, initial conditions, and trial parameters
from NetCDF-4, and writes both restart and history files in NetCDF-4. You
need:

- NetCDF C library (`libnetcdf`, version 4.x).
- NetCDF Fortran library (`libnetcdff`), built with the same compiler you
  will use for SUMMA. This is a separate package on most systems.

| Platform | Install command |
|----------|-----------------|
| macOS Homebrew | `brew install netcdf netcdf-fortran` |
| macOS MacPorts | `sudo port install netcdf netcdf-fortran +gcc12` |
| Ubuntu/Debian | `sudo apt-get install libnetcdf-dev libnetcdff-dev` |
| NCAR Derecho | `module load netcdf` |
| TACC | `module load netcdf` |

### LAPACK and BLAS

SUMMA uses LAPACK for the dense and band linear systems inside
`matrixOper.f90` and `summaSolve.f90`.

| Platform | Install command |
|----------|-----------------|
| macOS Homebrew | `brew install lapack` (BLAS comes along) |
| macOS MacPorts | `sudo port install atlas` (provides LAPACK + BLAS) |
| Ubuntu/Debian | `sudo apt-get install liblapack-dev libblas-dev` |
| Intel HPC | use MKL via `-mkl` (with `ifort`) |
| OpenBLAS HPC | `module load openblas` and link `-lopenblas` |

### SUNDIALS (optional, CH-Earth fork only)

The CH-Earth `summa-sundials` development line links SUNDIALS for IDA and
KINSOL. Build SUNDIALS 6.x with Fortran 2003 interfaces enabled. See
`sundials-and-solvers.md` for the full procedure. Skip this dependency if
you are using `NCAR/summa`.

---

## Step 4: Build

There are three supported routes. Pick one.

### Route A: CMake (recommended)

```bash
cd summa/build/cmake
./compile_script.sh
```

`compile_script.sh` calls `cmake -B cmake_build -S ../.` then
`cmake --build cmake_build --target all -j`. The result is
`summa/bin/summa.exe`.

If a dependency lives outside the system search path, set
`CMAKE_PREFIX_PATH`. Example for a user-built NetCDF in `/home/me/netcdf`:

```bash
export CMAKE_PREFIX_PATH="$CMAKE_PREFIX_PATH:/home/me/netcdf"
./compile_script.sh
```

The CMake build uses `FindNetCDF.cmake` and `FindOpenBLAS.cmake` (both
shipped under `build/cmake/`).

### Route B: Makefile (traditional)

Set five environment variables before invoking `make`:

```bash
cd summa/build
export F_MASTER=$(pwd)/..        # the parent directory of build/
export FC=gfortran               # compiler family for flag selection
export FC_EXE=gfortran           # compiler executable
export INCLUDES="-I/opt/homebrew/include"
export LIBRARIES="-L/opt/homebrew/lib -lnetcdff -llapack -lblas"
make check                       # confirm variables are reasonable
make                             # builds; result is summa/bin/summa.exe
```

Worked Homebrew example for an Apple-Silicon Mac (April 2024):

```bash
export FC=gfortran
export FC_EXE=gfortran
export INCLUDES="-I/opt/homebrew/Cellar/netcdf-fortran/4.6.1/include -I/opt/homebrew/Cellar/lapack/3.12.0/include"
export LIBRARIES="-L/opt/homebrew/Cellar/netcdf-fortran/4.6.1/lib -lnetcdff -L/opt/homebrew/Cellar/lapack/3.12.0/lib -lblas -llapack"
```

Replace the version numbers with `ls /opt/homebrew/Cellar/netcdf-fortran/`
output. The Makefile exposes `gfortran` and `ifort` flag profiles in
PART 0; if you use a different compiler, copy and edit one of those
blocks.

For the Digital Research Alliance of Canada Graham cluster with `ifort`:

```bash
module load intel/2019.3
export FC=ifort
export INCLUDES="-I/cvmfs/.../netcdf-fortran/4.4.4/include"
export LIBRARIES="-L/cvmfs/.../netcdf-fortran/4.4.4/lib -lnetcdff -mkl"
```

The MKL flag `-mkl` pulls in LAPACK and BLAS.

### Route C: Docker

Pre-built images are published under
`bartnijssen/summa` on Docker Hub.

```bash
docker pull bartnijssen/summa:latest      # tracks NCAR master
docker pull bartnijssen/summa:develop     # tracks NCAR develop
docker run bartnijssen/summa:latest -v    # version banner
```

For real runs you must mount the local settings directory into the
container:

```bash
docker run -v $(pwd):/work bartnijssen/summa:latest -m /work/fileManager.txt
```

The Dockerfile in the repo root rebuilds the image from source on Ubuntu
xenial with `gfortran-6`, `libnetcdff-dev`, and `liblapack-dev`. Use it as
a reference for minimal-system installs.

---

## Step 5: Smoke-test

```bash
$F_MASTER/bin/summa.exe -v
```

Expected output (text and version vary):

```
SUMMA - Structure for Unifying Multiple Modeling Alternatives
                      Version: v3.3.0
              Build Time: ...
                   Git Branch: master-...
         Git Hash: ...
```

The usage banner appears when you call `summa.exe` with no args:

```
Usage: summa.exe -m master_file [-s fileSuffix] [-g startGRU countGRU] [-h iHRU] [-r freqRestart] [-p freqProgress] [-c]
```

If you reach this point, SUMMA is built. Continue to
`running-summa.md` to actually launch a simulation, or
`case-studies.md` to run the included Reynolds Mountain East
experiment.

---

## Common install pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cannot find -lnetcdff` | NetCDF Fortran package missing or `LIBRARIES` path wrong | Install `netcdf-fortran` (separate from `netcdf`); confirm with `nf-config --prefix` |
| `Symbol not found: __netcdf_MOD_*` | NetCDF built with one compiler, SUMMA built with another | Rebuild netcdf-fortran with the same gfortran/ifort, or switch the SUMMA build to match |
| `cannot find -llapack` | LAPACK package missing | macOS: `brew install lapack`; Ubuntu: `apt install liblapack-dev libblas-dev`; HPC: `module load openblas` or use `-mkl` |
| CMake reports "Could NOT find NetCDF" | NetCDF outside default search path | `export CMAKE_PREFIX_PATH=$CMAKE_PREFIX_PATH:/your/netcdf` and rerun |
| Build succeeds but `summa.exe -v` reports the wrong version | Stale `bin/summa.exe` from a previous tag | `make clean` (Makefile) or `rm -rf build/cmake/cmake_build` (CMake), then rebuild |
| Segfault at startup | Linker mixed gfortran-built NetCDF with ifort-built SUMMA, or vice versa | Rebuild both with the same compiler |
| Build fails inside `noah-mp/` files | Free-form vs fixed-form flags wrong | Ensure `-ffree-form -ffree-line-length-none` is on for noah-mp objects (Makefile sets `FLAGS_NOAH` for this) |

---

## Where to next

- Tour of the source tree and the data structures: `architecture.md`
- Configure and run a simulation: `running-summa.md`
- Reproduce a published experiment: `case-studies.md`
- Build the SUNDIALS solver branch: `sundials-and-solvers.md`
