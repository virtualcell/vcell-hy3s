# vcell-hy3s

Hy3S — Hybrid Stochastic Simulation for Supercomputers — as used by the
[Virtual Cell](https://github.com/virtualcell/vcell) framework. It simulates
chemical kinetics by partitioning reactions into a fast subset integrated as a
stochastic differential equation and a slow subset handled by discrete jumps.

Three executables come from one Fortran source set, differing only in the SDE
integrator compiled in:

| binary | integrator |
| --- | --- |
| `Hybrid_EM_x64` | Euler–Maruyama |
| `Hybrid_MIL_x64` | Milstein, fixed step |
| `Hybrid_MIL_Adaptive_x64` | Milstein, adaptive step |

Split out of
[virtualcell/vcell-solvers](https://github.com/virtualcell/vcell-solvers), where
it was switched off in every build recipe — and, it turns out, could not have
been built at all (see [Provenance](#provenance)).

## Layout

| path | what |
| --- | --- |
| `Hy3S/` | the solver: 13 Fortran 90 sources plus a C++ shim for messaging |
| `netcdf/` | vendored NetCDF 3.6.2 — C, F77 and F90 layers |
| `vcell-messaging/` | submodule — progress messaging over the JMS REST bridge |
| `cmake/` | `GetGitRevisionDescription` |

There is no expression parser here; Hy3S does not use one.

## Build

```bash
git submodule update --init --recursive     # vcell-messaging
CC=gcc CXX=g++ FC=gfortran conan install . --build=missing \
      -pr:a=conan-profiles/CI-CD/Linux-AMD64_profile.txt
source build/generators/conanbuild.sh
CC=gcc CXX=g++ FC=gfortran cmake -B build -S . -G Ninja \
      -DCMAKE_TOOLCHAIN_FILE="$PWD/build/generators/conan_toolchain.cmake" \
      -DCMAKE_BUILD_TYPE=Release \
      -DOPTION_TARGET_MESSAGING=ON
cmake --build build
```

Prerequisites are a GCC with **gfortran** — Clang has no Fortran compiler, and
gfortran's runtime is built against libstdc++, so the libc++ toolchain the ODE
repo uses is not an option either.

Conan carries very little here: **libcurl is the only dependency, and only when
messaging is enabled.** With `-DOPTION_TARGET_MESSAGING=OFF` the build needs
nothing external at all.

### Why NetCDF is vendored rather than a Conan package

Hy3S does `USE netcdf`, so it needs the Fortran 90 bindings. Conan Center
publishes `netcdf` as the **C library only** — no `.mod` files — and has no
`netcdf-fortran` recipe. Unidata's netcdf-fortran cannot be pulled in with
`FetchContent` either: it refers to `CMAKE_SOURCE_DIR` internally and so insists
on being the top-level project.

The vendored 3.6.2 tree carries its own C, F77 and F90 layers, is what this code
was written against, and builds cleanly with modern gcc/gfortran. It needed only
two changes: declaring C alongside Fortran, and dropping a hardcoded path to the
Intel compiler.

`netcdf/f90/netcdf_f90.lib` and `netcdf/libsrc/netcdf.lib` are committed 2007-era
MSVC binaries. Nothing in the build references them; they survive from the
original import and are worth deleting once the Windows question is settled.

## Platforms

Linux and macOS build with GCC. **Windows is not wired up yet**, but is more
tractable than it first appears — see below.

MSVC has no Fortran compiler, so it cannot build this alone. The original
Windows build paired **Intel Fortran with MSVC**, and the evidence is still in
the tree: `-DPowerStationFortran` and a `netcdf/win32/config.h` for the C layer,
Intel's `/iface:mixed_str_len_arg` flags for the F90 layer, and the uppercase
`__cdecl LOAD_JMS_INFO` entry points in `msgwrapper.cpp` that match Intel
Fortran's Windows name mangling.

The modern equivalent is Intel oneAPI's `ifx`, which still integrates with
Visual Studio. Nothing here needs MSYS2 or Cygwin: unlike Chombo, Hy3S has no
build system of its own — no GNU make, no perl, no custom preprocessor, just
CMake over plain Fortran. With messaging off, a Windows build would have **no
external dependencies whatsoever**.

## Provenance

Hy3S was an **Intel Fortran** project, and it had not been built in a long time.
Every recipe in vcell-solvers passes `-DOPTION_TARGET_HY3S_SOLVERS=OFF`, and it
would not have built if they had not: `mainprogram-HyJCMSS.f90` contains

```c
#ifndef SVNVERSION
#error "SVNVERSION required"
#endif
```

and nothing in that repository ever defined `SVNVERSION`. It is now stamped from
the git describe, which is what it was asking for.

Getting it building with gfortran turned up four more issues, three of them real
defects rather than configuration:

- **114 Intel `!DEC$ IF` directives.** gfortran does not implement these — it
  treats them as comments and compiles the guarded code regardless, so a
  messaging-free build would still try to link the messaging calls. All were
  converted to standard `#if`/`#else`/`#endif`. Intel's `Defined()` matched
  case-insensitively, so the sources said `Milstein` where the build defined
  `MILSTEIN`; the conversion normalises to upper case, since the C preprocessor
  draws no such equivalence. The result works under both compilers.
- **The version banner could not compile.** It stringified an unquoted macro
  with `#x`, and gfortran's `-cpp` preprocesses in *traditional* mode, which
  predates ANSI stringification. It is now supplied already quoted and
  concatenated with `//`.
- **A type bug in the messaging path.** `Call send_progress(percentile, i)`
  passed the `Integer` trial counter where the C side dereferences a `double*` —
  four bytes read as eight. Invisible until now, since Hy3S was never built and
  Intel's implicit interfaces would not have flagged it.
- **`f2kcli` declared `IARGC` `EXTERNAL`**, which defeats gfortran's intrinsic
  and leaves `iargc_` unresolved at link. The file's own comment says those
  declarations "should not really be necessary" and were added for PGI, so they
  are guarded out for gfortran and kept for everything else.

`msgwrapper` also drove the old in-tree `SimulationMessaging` — `create()`, an
explicit `start()`, and `new WorkerEvent(JOB_PROGRESS, …)`. None of that exists
in the current `vcell-messaging`, so it was ported to the lazy singleton and
`JobEvent` statuses, the same migration vcell-chombo needed.
