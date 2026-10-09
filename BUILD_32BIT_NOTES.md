# EPOCH 32-bit (single precision) build — status & how to rebuild

Date: 2026-10-09. Tree: `/media/dr/2605sn2/epochCurr32/epoch` (branch main, v4.20.1 + local diff).
Applies to **epoch1d, epoch2d and epoch3d**.
Reference 64-bit tree: `/media/dr/2605sn2/epochCurr64/epoch` (kept in sync for comparisons).
Published at: https://github.com/DanRRRR/epoch

## How to build

```bash
cd /media/dr/2605sn2/epochCurr32/epoch/epoch3d   # or epoch2d / epoch1d
make COMPILER=gfortran ARCH=native PRECISION=single -j16     # 32-bit float build
```

* `PRECISION=single`  -> `-DSINGLE_PRECISION`, `num = KIND(1.0)` (4-byte reals).
  Default (no PRECISION) reproduces the original 64-bit code exactly.
* `ARCH=native`       -> `-march=native -mtune=native` (AVX512 on this host).
* `FASTMATH=yes`      -> adds `-ffast-math`. Measured: **no gain** on the 32-bit
  build (17.75 s vs 17.62 s) and changes round-off. Leave it off.

Reference 64-bit build: same but without `PRECISION=single`
(`make COMPILER=gfortran ARCH=native -j16` in epochCurr64).

## What was changed (per dimension: 6 files + Makefile)

Same set of changes applied to epoch1d, epoch2d and epoch3d:

| File | Change |
|---|---|
| `Makefile` | `ARCH=`, `PRECISION=single`, `FASTMATH=yes` options |
| `src/constants.F90` | `num = KIND(1.0)` under `SINGLE_PRECISION`; constants whose intermediate values leave the 32-bit range (`(q0/c)**2 ~ 2.8e-55`, `1/mc0**2 ~ 1.3e43`, pair-production `b_s,e_s,tau_c`, bremsstrahlung logs) evaluated in `dbl` and exported as 64-bit |
| `src/housekeeping/redblack_module.f90` | `_r4` variants removed from generic `redblack` interface (ambiguous when `num == r4`); still callable directly |
| `src/physics_packages/background_collisions.F90` | `MAX(random(), 5e-9_dbl)` — kind must match `random()` |
| `src/physics_packages/injectors.F90` | explicit promote/demote around `random_box_muller` (double in, `num` out) |
| `src/user_interaction/particle_temperature.F90` | relativistic temperature sampler + Lorentz drift transform fully in `dbl` (constants `c^2/kb ~ 6.5e39`, `2.36e-80` out of 32-bit range; setup-only code, no core-loop cost) |

## Validation

### epoch3d (input.deck: 100^3, 1.23 M particles, laser 1e20 W/cm2)

* t_end = 20 fs (55 steps), 4 ranks, `runs/single32` vs `runs/base64`:
  * t = 0: fields bit-identical.
  * t = 20 fs: field rel. diffs 1e-5 .. 6e-4 (round-off level); particle/field
    energies agree to ~1e-7; charges exact.
* t_end = 100 fs (275 steps), `runs/bench32` vs `runs/bench64`:
  * pointwise fields decorrelate (normal chaotic amplification of round-off),
  * integral physics still agrees: particle energies ~1e-4, field energy ~1e-3,
    laser absorption 3e-5.

### epoch2d (Data/input.deck: 250^2, 0.5 M particles)

* 4 ranks, 20 fs (112 steps), `runs/single2d` vs `runs/base2d`:
  * `ey`, `ekbar`, temperature blocks bit-identical; laser-injected energy and
    absorption agree to ~1e-5.
  * number_density differs by ~1e-1 **only at the cone edge**: particle
    positions are drawn randomly per cell and cell indices are recomputed as
    `FLOOR((pos-x_min)/dx + 1.5)`; the ~1e-7 relative round-off in `pos` flips
    the cell of a particle sitting within ~1e-5 dx of a boundary, which then
    changes that particle's weight renormalisation (`npart_in_cell`).
    Affected: ~6 particles out of 250k. Uniform-density test deck matches to
    3e-6 everywhere, confirming this is a density-discontinuity boundary
    effect, not a code defect. After 20 fs of dynamics the total particle
    energies agree to ~1e-4.

### epoch1d (2000 cells, 40k particles, 50 fs, 1578 steps)

* Laser-injected energy locked to 1.8e-5 and absorption to 2.4e-7 over the
  whole run; particle energies agree to 1e-5 until the laser reaches the slab
  (t~33 fs), then grow ~10x/10fs to 8% by 50 fs.
* **Chaos control experiment**: 64-bit vs 64-bit at different rank counts
  (np=1 vs np=2, identical code, different round-off) diverges *more*
  (electron energy 0.2 rel at 50 fs, field energy 6.8e-3) than 32-bit vs
  64-bit (0.08, 2.6e-3). The exponential growth is therefore deterministic
  chaos amplifying round-off, not a single-precision defect.

## Performance (epoch3d, 4 ranks, 100 fs benchmark deck in `runs/bench{32,64}`)

| build | core runtime | max RSS (peak RAM) |
|---|---|---|
| 64-bit + AVX512 | 20.4 ± 0.3 s | 108 MB |
| 32-bit + AVX512 | 17.6 ± 0.1 s | 77 MB |

=> **1.16x faster, 28% less memory** on this small deck. Memory saving grows
towards 2x for particle-dominated production decks.
(epoch1d test deck: 1.44 s vs 1.86 s => 1.3x faster; epoch2d: 1.43 s vs 1.58 s.)

## Vectorization status (for future AVX512 work)

* Field solver: `update_b_field` (796 zmm), `update_e_field` (310 zmm) — already
  auto-vectorized well.
* `push_particles` in `src/particles.F90` (includes the charge-conserving
  current deposition): **scalar, 2 zmm** — gather/scatter pattern the compiler
  cannot vectorize. This is the main remaining hotspot and the natural target
  for manual AVX512 work (EPOCH's per-cell particle lists help here).
  Fast-math does NOT unlock it (measured).

## Known benign warning

`Unable to find ionisation_energies.table` — harmless unless average ionisation
models are used (not used by input.deck).

## Run layout (on /media/dr/2605sn2)

* `runs/single32`, `runs/base64` — epoch3d 20 fs validation runs.
* `runs/bench32`, `runs/bench64` — epoch3d 100 fs timing runs.
* `runs/single2d`, `runs/base2d` — epoch2d validation; `runs/t32`, `runs/t64`
  — epoch2d uniform-density control deck.
* `runs/single1d`, `runs/base1d` — epoch1d 50 fs validation (dumps every 5 fs);
  `runs/base1d_np1` — 64-bit np=1 chaos-control run.
* `runs/sdfcmp` — SDF comparison tool binary; `/tmp/readsdf.py` — minimal
  python SDF reader used for particle-level analysis.
* Note: `epoch*` reads `USE_DATA_DIRECTORY` file in cwd containing the data dir
  path (`.` = cwd). A 121-rank production run on `/media/dr/2509sn8` (different
  binary) was left untouched.
