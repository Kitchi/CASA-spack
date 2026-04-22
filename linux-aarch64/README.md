# CASA Spack Build Environment for Linux / aarch64

Spack environment for CASA6 build dependencies on TACC Vista (aarch64 / NVIDIA Grace,
Neoverse V2). Make sure to have the spack `setup-env.sh` sourced into your shell before
running any of the below commands.

```bash
source ~/path/to/spack_install/share/spack/setup-env.sh
```

or add that line into your `~/.bashrc`.

> **Note:** This build recipe is adapted from the verified x86_64 Linux recipe
> (`casa-spack-linux`). The aarch64 build has not yet been verified end-to-end.

## Prerequisites

- **Spack** (https://spack.io)
- **GCC 13.2.0** via module (`module load gcc/13.2.0`) — system gcc@11.5.0 is the
  fallback if 13.2 causes issues
- **`spack compiler find`** run once after loading the gcc module to register it

```bash
module load gcc/13.2.0
spack compiler find
```

All other dependencies (cmake, grpc, protobuf, openmpi, python@3.12, etc.) are
managed by Spack.

## Setup

Clone this repository:

```bash
git clone <repo-url> /path/to/casa-spack-aarch64
```

Create and install the environment:

```bash
module load gcc/13.2.0
spack env create casa-dev /path/to/casa-spack-aarch64/spack.yaml
spack env activate casa-dev
spack concretize
spack install
```

After any changes to `spack.yaml`, recreate the environment:

```bash
spack env rm casa-dev
spack env create casa-dev /path/to/casa-spack-aarch64/spack.yaml
spack env activate casa-dev
spack concretize
spack install
```

## Usage

Activate the environment before building CASA:

```bash
spack env activate casa-dev
```

Python packages (numpy, pip, build) are not managed by Spack. They are installed
into a venv by the Makefile automatically.

## Makefile

The included `Makefile` is identical to the x86_64 Linux variant — no aarch64-specific
changes are needed:

- `SIMD_ARCH=GENERIC` on libsakura (no x86 SIMD assumptions)
- `PORTABLE=ON` on casacore
- GCC used for C, C++, and Fortran throughout (no compiler pinning)
- ccache launcher flags on all cmake targets

Copy the Makefile into a clean build directory with the spack env active, then:

```bash
make firstcasa
```

## grpc / protobuf version pinning

Spack's latest grpc/protobuf/abseil-cpp versions do not form a mutually compatible
set with the re2 version that grpc hard-pins. The following versions are known to
work together (same constraint as x86_64):

| Package | Version |
|---------|---------|
| grpc | 1.67 (cxxstd=17) |
| protobuf | 26.1 |
| abseil-cpp | 20240116.1 |
| re2 | 2023-09-01 (grpc hard-pin) |

See comments in `spack.yaml` for details.

## Known Issues

### `spack env deactivate` may corrupt PATH

`spack env deactivate` can strip entries from PATH and not restore them on
re-activate. This is a known Spack bug
([spack#48391](https://github.com/spack/spack/issues/48391)).

**Workaround:** Always activate from a fresh shell rather than deactivating and
re-activating:

```bash
# Open a new terminal, then:
module load gcc/13.2.0
source ~/src/spack/share/spack/setup-env.sh
spack env activate casa-dev
```
