# CASA Spack Build Environment for Linux / aarch64

Spack environment and Makefile for building CASA6 from source on Linux/aarch64.
Currently tested on TACC Vista (NVIDIA Grace, Neoverse V2, Rocky 9).

## Prerequisites

- **Spack** v1.0+ (https://spack.io) — sourced into your shell:

```bash
source ~/path/to/spack/share/spack/setup-env.sh
```

- **System GCC** — Vista uses the system gcc@11.5.0 (`/usr/bin/gcc`). CASA
  requires gcc 4.9+ (9+ for GPU builds), so 11.5.0 is sufficient. The
  module gcc@13.2.0 has a broken `libisl.so.23` rpath that spack cannot work
  around, so we avoid it.

All other dependencies (cmake, grpc, protobuf, openmpi, python@3.12, etc.)
are managed by Spack.

## Directory layout

```
linux-aarch64/
  spack.yaml          # platform-independent spack env (specs, concretizer)
  site.yaml           # gitignored symlink to your site overlay
  sites/
    vista.yaml        # TACC Vista externals and compiler config
  patches/
    libsakura-aarch64.patch   # removes x86-only -m64 flag from libsakura
  Makefile            # CASA6 build recipe (libsakura -> casacore -> casatools -> ...)
```

## Setup

### 1. Clone and link site overlay

```bash
git clone <repo-url>
cd CASA-spack/linux-aarch64
ln -s sites/vista.yaml site.yaml
```

### 2. Create and install the spack environment

```bash
spack env create casa-dev spack.yaml
spack env activate casa-dev
spack concretize
spack install
```

After changes to `spack.yaml` or the site overlay, recreate the environment:

```bash
spack env rm casa-dev
spack env create casa-dev spack.yaml
spack env activate casa-dev
spack concretize
spack install
```

### 3. Build CASA

Copy (or symlink) the Makefile into a clean build directory, activate the
spack environment, and run:

```bash
mkdir -p ~/work/casa-build && cd ~/work/casa-build
cp /path/to/CASA-spack/linux-aarch64/Makefile .
spack env activate casa-dev
make PATCHDIR=/path/to/CASA-spack/linux-aarch64/patches firstcasa
```

`PATCHDIR` is required and validated at the start of the build. It must
point to the `patches/` directory containing the aarch64 patches.

For incremental rebuilds after the initial clone:

```bash
make PATCHDIR=/path/to/CASA-spack/linux-aarch64/patches casa
```

Individual targets are also available: `libsakura`, `casacore`,
`casacpp`, `casatools`, `casatasks`, `casashell`.

### Vista-specific notes

- Build on a **compute node** (`idev` or batch job), not the login node.
  Login nodes throttle process spawning, causing intermittent
  `/bin/sh: Operation not permitted` errors during parallel compilation.
- Do **not** `module load gcc/13.2.0` — it's unnecessary and its broken
  `libisl.so.23` rpath causes spack build failures.
- Run `module purge` before activating the spack env to avoid NVIDIA
  compilers (`nvc++`) being picked up by cmake.

## Adding a new site

Site overlays live in `sites/` and are included by `spack.yaml` via the
`site.yaml` symlink. Each site file defines cluster-specific externals
(compilers, MPI, etc.).

To add a new cluster:

```bash
cp sites/vista.yaml sites/my-cluster.yaml
```

Edit `sites/my-cluster.yaml` to match your cluster's compiler paths and
externals. The file contains two sections:

- **`packages:`** — external package definitions (compiler paths, prefixes)
- **`env_vars:`** (if needed) — environment variables to inject into builds

Then link it:

```bash
ln -sf sites/my-cluster.yaml site.yaml
```

## Patches

The `patches/` directory contains fixes applied automatically during the build:

- **`libsakura-aarch64.patch`** — removes the hardcoded `-m64` flag (x86-only)
  from libsakura's CMakeLists.txt and adds a `GENERIC` option to
  `SetArchFlags.cmake`. The libsakura source already has ARM NEON support via
  `sse2neon.h`, so only the cmake flags need fixing.

## grpc / protobuf version pinning

Spack's latest grpc/protobuf/abseil-cpp versions do not form a mutually
compatible set. The following versions are known to work:

| Package | Version |
|---------|---------|
| grpc | 1.67 (cxxstd=17) |
| protobuf | 26.1 |
| abseil-cpp | 20240116.1 |
| re2 | 2023-09-01 (grpc hard-pin) |

See comments in `spack.yaml` for details.

## Known issues

### `spack env deactivate` may corrupt PATH

`spack env deactivate` can strip entries from PATH and not restore them on
re-activate. This is a known Spack bug
([spack#48391](https://github.com/spack/spack/issues/48391)).

**Workaround:** Always activate from a fresh shell rather than deactivating
and re-activating.
