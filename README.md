# CASA-spack

Spack environment configurations and Makefiles for building CASA6 from source on three platforms:

| Directory | Platform |
|-----------|----------|
| `linux-x86/` | RHEL 8 / x86_64 (e.g. NRAO clusters) |
| `linux-aarch64/` | Linux / aarch64 (e.g. TACC Vista, NVIDIA Grace) |
| `mac26/` | macOS ARM64 (Apple Silicon) |

Each directory contains a `spack.yaml`, a `Makefile`, and a platform-specific `README.md`. Read the per-platform README for full details. This document covers the common workflow.

## Prerequisites

### All platforms

- **Spack** — https://spack.io
- Source the Spack setup script before any `spack` commands:

```bash
source ~/path/to/spack/share/spack/setup-env.sh
```

Add that line to your `~/.bashrc` (Linux) or `~/.zshrc` (macOS).

### Linux x86\_64 (RHEL 8)

- System GCC 8.5.0 (gcc, g++, gfortran). Register it once:

```bash
spack compiler find
```

### Linux aarch64 (TACC Vista)

- GCC 13.2.0 via the system module. Register it once:

```bash
module load gcc/13.2.0
spack compiler find
```

### macOS ARM64

- Xcode command line tools
- GCC 15 and system grpc/protobuf (declared as Spack externals):

```bash
# Homebrew
brew install gcc@15 grpc protobuf

# MacPorts
sudo port install gcc15 grpc protobuf3-cpp
```

- Register the custom Spack package repo (one-time, needed for the patched `wcslib` recipe):

```bash
spack repo add /path/to/this/repo/mac26/repo
```

## Spack environment setup

Pick the directory matching your platform. The workflow is the same on all three:

```bash
cd linux-x86          # or linux-aarch64, mac26

spack env create casa-dev spack.yaml
spack env activate casa-dev
spack concretize
spack install
```

To recreate the environment after editing `spack.yaml`:

```bash
spack env rm casa-dev
spack env create casa-dev spack.yaml
spack env activate casa-dev
spack concretize
spack install
```

## Building CASA

With the Spack environment active, copy (or symlink) the platform Makefile into a clean build directory and run:

```bash
# Linux (x86 or aarch64)
spack env activate casa-dev
cp linux-x86/Makefile /path/to/build/
cd /path/to/build/
make firstcasa

# macOS — PATCHDIR is required
spack env activate casa-dev
cp mac26/Makefile /path/to/build/
cd /path/to/build/
make PATCHDIR=/path/to/this/repo/mac26/patches firstcasa
```

`make firstcasa` clones the CASA6 source tree and builds libsakura → casacore → casacpp →
casatools → casatasks → casashell, installing everything under `./install/`. On completion:

```bash
source ./venv/bin/activate
python -c "import casatasks; print('OK')"
```

Individual stages can be re-run after the first build (`make casacore`, `make casatools`, etc.).

## Dependency notes

### grpc / protobuf (Linux)

Spack's latest grpc/abseil-cpp/re2 versions are not mutually compatible. The `spack.yaml` pins
known-good versions:

| Package | Version |
|---------|---------|
| grpc | 1.67 (cxxstd=17) |
| protobuf | 26.1 |
| abseil-cpp | 20240116.1 (grpc constraint) |
| re2 | 2023-09-01 (grpc hard-pin) |

### grpc / protobuf (macOS)

grpc and protobuf are managed externally (Homebrew/MacPorts) and declared as external packages
in `spack.yaml`. Spack-built abseil-cpp generates `.pc` files with CMake `SHELL:` syntax that
leaks x86 SIMD flags on ARM64 — using system packages avoids this entirely.

## Known issues

### `spack env deactivate` corrupts PATH

`spack env deactivate` can strip entries from PATH that are not restored on re-activate
([spack#48391](https://github.com/spack/spack/issues/48391)). On macOS this breaks `gfortran`
discovery; on aarch64 it drops the module-loaded GCC.

**Workaround:** Always activate from a fresh shell rather than deactivating and re-activating.

```bash
# Open a new terminal, source Spack, then activate:
source ~/src/spack/share/spack/setup-env.sh
module load gcc/13.2.0   # aarch64 only
spack env activate casa-dev
```
