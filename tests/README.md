# Tests

```bash
cmake --build build
ctest --test-dir build --output-on-failure
```

Five cases, about six seconds, no dependencies beyond what the solver already
needs — `hy3s_check` is C against the vendored NetCDF, so it builds wherever
the solver does.

## The model

`resources/enzyme.nc` is the sample model that shipped with the original
sources: Michaelis–Menten enzyme kinetics,

```
S + E ⇌ ES → E + P
```

with 60200 molecules each of S and E, run to t = 5 over 51 timepoints.

It was chosen because its stoichiometry yields two **exact** conservation laws:
enzyme `E + ES` and substrate `S + ES + P` are each constant at every timepoint.
Neither depends on the trajectory, so both hold for any seed, on any platform,
under any of the three integrators — and because molecule counts are integers
held in doubles, they hold to the bit. `hy3s_check` treats any drift at all as a
failure rather than applying a tolerance.

`P` can only be produced, never consumed, so it must also be non-decreasing.

Every case runs with `epsilon` and `lambda` set high enough that no reaction is
approximated as continuous. The run is then an exact stochastic simulation and
the SDE integrator — the only thing that differs between the three binaries — is
never entered. That is why one baseline serves all three, and why
`euler_maruyama` and `milstein` produce identical output.

## What each case covers

| case | covers |
| --- | --- |
| `euler_maruyama`, `milstein`, `milstein_adaptive` | each binary satisfies the invariants and reproduces the committed baseline |
| `euler_maruyama_other_seed` | the invariants hold on a different trajectory, so they constrain the solver rather than one run |
| `reproducibility` | 20 runs at one seed must agree exactly |

## Why `--ran`, and why 20 runs

Both exist because of one defect, found while splitting this repo out and fixed
in `dataio.f90`: `CheckAndDefineVariables` left its `intent(out)` error flag
undefined on the success path, so about one optimised run in five reported a
failure that had not happened and stopped — **while still exiting 0 and leaving
the output file untouched**.

So `--ran` asserts the solution actually reaches `TEnd` and is not all zeros. A
zero-filled file satisfies both conservation laws perfectly; without `--ran` the
invariants would sign off on a solver that did nothing.

The repeat count is not redundancy either. With the fix reverted, the three
single-run cases all still passed — a one-in-five failure hides easily. Twenty
runs caught it in five suite runs out of five. Anything that intermittent needs
repetition to be tested at all, and repeating asserts the property the defect
violated: same seed, same answer.

## The baseline

`resources/enzyme_baseline.nc` is a committed solution, compared elementwise at
`rtol 1e-9`. It is the only check here that depends on the trajectory, and so
the only one that could legitimately differ between platforms. The trajectory
proved stable against floating-point reassociation locally — `-O0` and `-O2`
agree byte for byte — which is what suggested a single shared baseline would
hold. If a platform ever disagrees, the invariant checks are the ones to trust;
the baseline is a regression tripwire, not a statement about correctness.

To regenerate it after an intentional change:

```bash
cp tests/resources/enzyme.nc /tmp/base.nc
build/bin/Hybrid_EM_x64 /tmp/base.nc 100000 1000000 0.01 1e-5 -R 12345 -OV
cp /tmp/base.nc tests/resources/enzyme_baseline.nc
```
