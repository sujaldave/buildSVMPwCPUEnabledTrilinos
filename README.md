# buildSVMPwCPUEnabledTrilinos
This repository documents a reproducible workflow to build svMultiPhysics on a local CPU-based Unix system using the Trilinos linear algebra package.

This guide is intended for:

- Linux systems
- macOS systems
- Windows users using WSL

CUDA is **not** required for this build.

> **Important note**
> This guide is based on one working CPU-based installation workflow. Some URLs, library versions, CMake flags, and compiler paths may change over time. Before running `wget`, `git clone`, or CMake commands, please check the latest installation instructions from each software package’s official website.

---

## 1. Recommended Directory Structure

Create one parent directory for all source codes, builds, and installed libraries.

For example:

```bash
mkdir -p /path/to/library/installation/directory
cd /path/to/library/installation/directory
```

This guide assumes the following structure:

```bash
/path/to/library/installation/directory/
├── lib/
│   ├── boost/
│   ├── lapack/
│   ├── hdf5/
│   ├── hypre/
│   └── trilinos/
└── svMultiPhysics/
```
Throughout this guide, replace: /path/to/library/installation/directory
with your actual installation directory.

It is helpful to define:

```bash
export INSTALL_ROOT=/path/to/library/installation/directory
cd $INSTALL_ROOT
```

## 2. Basic Prerequisites

Install the following before building:

* C compiler
* C++ compiler
* Fortran compiler
* CMake
* Make
* Git
* Wget
* OpenMPI
* BLAS/LAPACK support

On macOS, one possible setup is:
```bash
brew install gcc open-mpi cmake wget git
```

On Ubuntu/WSL, one possible setup is:
```bash
sudo apt update
sudo apt install -y build-essential gfortran cmake make git wget openmpi-bin libopenmpi-dev
```

Check MPI compilers:
```bash
which mpicc
which mpicxx
which mpifort
mpicc --version
```

> **Note**

If you are using macOS with Homebrew, MPI compilers may be located in:
```bash
/opt/homebrew/bin/mpicc
/opt/homebrew/bin/mpicxx
/opt/homebrew/bin/mpif90
```

On Intel-based macOS, they may instead be under:
```bash
/usr/local/bin
```

Before building, please check:

* Whether all git clone URLs are still valid
* Whether all wget URLs still point to the desired software version
* Whether newer versions of Boost, HDF5, HYPRE, LAPACK, Trilinos, or svMultiPhysics require different flags
* Whether your MPI compiler paths are correct
* Whether your system uses mpifort, mpif90, or another Fortran MPI wrapper
* Whether your shell is bash, zsh, or another shell
* Whether LD_LIBRARY_PATH or DYLD_LIBRARY_PATH is needed on your system
* Whether CMake correctly finds Trilinos when building svMultiPhysics

## 3. Install LAPACK

```bash
cd $INSTALL_ROOT

mkdir -p lapack
cd lapack

# Double check the repository URL before cloning.
git clone https://github.com/Reference-LAPACK/lapack.git

mkdir -p build
cd build

cmake \
  -DBUILD_SHARED_LIBS=ON \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_ROOT/lib/lapack \
  ../lapack

cmake --build . -j4
cmake --install . --config Release
```

## 4. Install HDF5 with Parallel Support

```bash
cd $INSTALL_ROOT

mkdir -p hdf5build
cd hdf5build

# Double check the repository URL before cloning.
git clone https://github.com/HDFGroup/hdf5.git

mkdir -p build
cd build

mkdir -p $INSTALL_ROOT/lib/hdf5
```

Set MPI compilers.

For macOS/Homebrew:
```bash
export CC=$(brew --prefix)/bin/mpicc
export CXX=$(brew --prefix)/bin/mpicxx
export FC=$(brew --prefix)/bin/mpifort
```

For Linux/WSL, this is usually enough:
```bash
export CC=mpicc
export CXX=mpicxx
export FC=mpifort
```

Configure and install HDF5:

```bash
cmake \
  -C ../hdf5/config/cmake/cacheinit.cmake \
  -G "Unix Makefiles" \
  -DHDF5_ALLOW_UNSUPPORTED=ON \
  -DHDF5_ENABLE_NONSTANDARD_FEATURE_FLOAT16:BOOL=OFF \
  -DHDF5_BUILD_JAVA:BOOL=OFF \
  -DHDF5_ENABLE_PARALLEL:BOOL=ON \
  -DALLOW_UNSUPPORTED:BOOL=ON \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_ROOT/lib/hdf5 \
  ../hdf5

cmake --build . -j4
make install
```

## 5. Install Boost

> Version used here: Boost 1.89.0
> Check before downloading
> The Boost archive URL may change if a newer version is released. Please verify the latest available version from the Boost website before running wget.

```bash
cd $INSTALL_ROOT

mkdir -p boost_1_89_0
cd boost_1_89_0

# Double check the URL/version before downloading.
wget https://archives.boost.io/release/1.89.0/source/boost_1_89_0.tar.gz

tar -xzvf boost_1_89_0.tar.gz
rm boost_*.tar.gz

cd boost_1_89_0

mkdir -p $INSTALL_ROOT/lib/boost

./bootstrap.sh --prefix=$INSTALL_ROOT/lib/boost
./b2 install
```

## 6. Install HYPRE

```bash
cd $INSTALL_ROOT

# Double check the repository URL before cloning.
git clone https://github.com/hypre-space/hypre.git

cd hypre/src

./configure --prefix=$INSTALL_ROOT/lib/hypre

make -j4
make install
```

> Note
> Avoid using sudo unless absolutely necessary. If $INSTALL_ROOT is inside your home directory or another user-owned directory, sudo should not be needed.

## 7. Install Trilinos with CPU-Based Kokkos and Tpetra Support

```bash
cd $INSTALL_ROOT

# Double check the repository URL before cloning.
git clone https://github.com/trilinos/Trilinos.git
```

## 7.1 HYPRE Header Compatibility Workaround

Some newer HYPRE versions have filename changes that may not be recognized by Trilinos build files.

If needed, run:
```bash
cd $INSTALL_ROOT/lib/hypre/include
printf '#include "HYPRE_krylov.h"\n' > krylov.h
```
Then return to the Trilinos directory:
```bash
cd $INSTALL_ROOT/Trilinos
mkdir -p build
cd build
```

## 8. Configure Trilinos

Create a file named install inside:
```bash
$INSTALL_ROOT/Trilinos/build
```

with the following contents:
```bash
cmake \
  -DCMAKE_INSTALL_PREFIX=$INSTALL_ROOT/lib/trilinos \
  -DTPL_ENABLE_MPI=ON \
  -DTPL_ENABLE_Boost=ON \
  -DBoost_LIBRARY_DIRS=$INSTALL_ROOT/lib/boost/lib \
  -DBoost_INCLUDE_DIRS=$INSTALL_ROOT/lib/boost/include \
  -DTPL_ENABLE_BLAS=ON \
  -DBLAS_LIBRARY_DIRS=$INSTALL_ROOT/lib/lapack/lib \
  -DTPL_ENABLE_HDF5=ON \
  -DHDF5_LIBRARY_DIRS=$INSTALL_ROOT/lib/hdf5/lib \
  -DHDF5_INCLUDE_DIRS=$INSTALL_ROOT/lib/hdf5/include \
  -DTPL_ENABLE_HYPRE=ON \
  -DHYPRE_LIBRARY_DIRS=$INSTALL_ROOT/lib/hypre/lib \
  -DHYPRE_INCLUDE_DIRS=$INSTALL_ROOT/lib/hypre/include \
  -DTPL_ENABLE_LAPACK=ON \
  -DLAPACK_LIBRARY_DIRS=$INSTALL_ROOT/lib/lapack/lib \
  -DCMAKE_C_COMPILER=mpicc \
  -DCMAKE_CXX_COMPILER=mpicxx \
  -DCMAKE_Fortran_COMPILER=mpifort \
  -DTrilinos_ENABLE_MueLu=ON \
  -DTrilinos_ENABLE_ROL=ON \
  -DTrilinos_ENABLE_Sacado=ON \
  -DTrilinos_ENABLE_Teuchos=ON \
  -DTrilinos_ENABLE_Zoltan=ON \
  -DTrilinos_ENABLE_Tpetra=ON \
  -DTrilinos_ENABLE_Belos=ON \
  -DTrilinos_ENABLE_Ifpack2=ON \
  -DTrilinos_ENABLE_Amesos2=ON \
  -DTrilinos_ENABLE_Zoltan2=ON \
  -DTrilinos_ENABLE_Kokkos=ON \
  -DKokkos_ENABLE_SERIAL=ON \
  -DTrilinos_ENABLE_KokkosKernels=ON \
  -DTrilinos_ENABLE_Xpetra=ON \
  -DXpetra_ENABLE_Kokkos_compat=ON \
  -DTrilinos_ENABLE_EXPLICIT_INSTANTIATION=ON \
  -DTpetra_INST_SERIAL=ON \
  -DTpetra_INST_DOUBLE=ON \
  -DTpetra_INST_INT_INT=ON \
  -DMueLu_ENABLE_EXPLICIT_INSTANTIATION=ON \
  -DTrilinos_ENABLE_Gtest=OFF \
  ..
```
Then run:
```bash
chmod +x install
./install
make -j6 install
```

**Important**

> If you are on macOS/Homebrew and mpicc, mpicxx, or mpifort are not found, replace:
```bash
-DCMAKE_C_COMPILER=mpicc
-DCMAKE_CXX_COMPILER=mpicxx
-DCMAKE_Fortran_COMPILER=mpifort
```
> with the full paths, for example:
```bash
-DCMAKE_C_COMPILER=/opt/homebrew/bin/mpicc
-DCMAKE_CXX_COMPILER=/opt/homebrew/bin/mpicxx
-DCMAKE_Fortran_COMPILER=/opt/homebrew/bin/mpif90
```
> Always check the correct paths using:
```bash
which mpicc
which mpicxx
which mpifort
```

## 9. Add Trilinos to Your Environment

Add the following to your ~/.bashrc, ~/.zshrc, or equivalent shell configuration file:

```bash
export TRILINOS_DIR=$INSTALL_ROOT/lib/trilinos
export PATH=$TRILINOS_DIR/lib/cmake/Trilinos:$PATH
export LD_LIBRARY_PATH=$TRILINOS_DIR/lib:$LD_LIBRARY_PATH
```

For macOS, use DYLD_LIBRARY_PATH if needed:
```bash
export DYLD_LIBRARY_PATH=$TRILINOS_DIR/lib:$DYLD_LIBRARY_PATH
```

Reload your shell:
```bash
source ~/.bashrc
```
or, for zsh:
```bash
source ~/.zshrc
```

## 10. Build svMultiPhysics with Trilinos Enabled

```bash
cd $INSTALL_ROOT

# Double check the repository URL before cloning.
git clone https://github.com/SimVascular/svMultiPhysics.git

cd svMultiPhysics

mkdir -p build-trilinos
cd build-trilinos

cmake \
  -DSV_USE_TRILINOS:BOOL=ON \
  -DTrilinos_DIR=$INSTALL_ROOT/lib/trilinos/lib/cmake/Trilinos \
  ..

make -j4
```

The executable should be created under the build directory, commonly in:

```bash
$INSTALL_ROOT/svMultiPhysics/build-trilinos/bin/
```

## Final Note

This CPU-based workflow does not use CUDA. If you want GPU acceleration, you need a separate CUDA-aware OpenMPI installation and a CUDA/Kokkos-enabled Trilinos build -> Check out another repository that I have for the GPU build. 

