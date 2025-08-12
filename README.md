# Deneb_mac
Deneb for mac os (Apple Silicon M1/M2/M3 etc)

NOTE: Not for Intel mac os

I use vscode for modifying this code. 

# Overview
This repository provides a complete(maybe..) guide and setup for the DENEB on macOS (Apple Silicon; M1, M2, M3). 
DENEB is a high-order finite element CFD solver that requires complex scientific computing libraries including MPI, PETSc, and OpenBLAS.

I used 'conda' to install external libraries on my Mac by configuring environment variables.
(I attempted to create a Windows virtual machine using VMware for external library installation. 
However, due to Apple Silicon's ARM architecture, I encountered significant compatibility issues with Windows for ARM, specifically failing to install msmpi. While Parallels could have been an alternative solution, I chose not to test it. Consequently, I decided to use conda for managing my external library dependencies instead.)

# Installation

### 0. Install Mini-forge (ARM64)

```
curl -L https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh -o miniforge.sh
chmod +x miniforge.sh
bash miniforge.sh
```
Install Path: default(~/miniforge3) recommended
Path added: choose 'yes'
conda init: choose 'yes'

Then check the version
```
conda --version

uname -m
```

You should re-open the terminal or use this code before step 2
```
source ~/.zshrc 
```

### 1. Create Conda Environment

create conda envirnment for external libraries
```
# Create dedicated conda environment for DENEB
conda create -n Deneb python=3.11 -y
conda activate Deneb
```

### 2. Install Required External Libraries
```
# Install all required libraries from conda-forge
conda install -c conda-forge \
    petsc=3.23.5 \
    openmpi=5.0.8 \
    openblas \
    parmetis \
    metis \
    hdf5 \
    fftw \
    hypre \
    mumps-mpi \
    openmp \
    gfortran_impl_osx-arm64
```

check
```
mpicc --version
mpicxx --version

mpirun --version
```

Then install 'IDEA' library
```
git clone https://github.com/HojunYouKr/IDEA.git
cd IDEA
```
Modify the Makefile of 'IDEA' (I modified it already)
Then compile IDEA 

```
make clean
make all
```

```
# install in conda env
cp library/libidea.a $CONDA_PREFIX/lib/
cp include/*.h $CONDA_PREFIX/include/

# check
ls $CONDA_PREFIX/lib/libidea.a
ls $CONDA_PREFIX/include/ | grep -i idea
```


### 3. Verify Installation
```
# Check installed packages
conda list | grep -E "(petsc|mpi|blas|parmetis|metis)"
echo $CONDA_PREFIX  # Verify conda environment path
```

if you install correctly, you may get the output like this.

```
echo $CONDA_PREFIX
```
output
```
/Users/User_name/miniforge3/envs/Deneb
```

### 4. Environment Setup

Following this way
```
# Activate conda environment
conda activate Deneb

# Set header file paths
export CPLUS_INCLUDE_PATH="$CONDA_PREFIX/include:$CPLUS_INCLUDE_PATH"
export C_INCLUDE_PATH="$CONDA_PREFIX/include:$C_INCLUDE_PATH"

# Optional PETSc variables
export PETSC_DIR=$CONDA_PREFIX
export PETSC_ARCH=""
```

You should also set up the c_cpp_properties.json, settings.json (uploaded in repository)

### Optional: Permanent Setup

Add to your ~/.zshrc or ~/.bash_profile:
```
alias deneb-env="conda activate Deneb && export CPLUS_INCLUDE_PATH=\"\$CONDA_PREFIX/include:\$CPLUS_INCLUDE_PATH\" && export C_INCLUDE_PATH=\"\$CONDA_PREFIX/include:\$C_INCLUDE_PATH\""
```

## Header File Resolution Strategy

### MPI Headers: Symbolic Link Approach
Problem: CMake cannot properly locate MPI headers in conda environment.

Solution: Create symbolic links to conda include directory.

```
# Create symbolic links for Avocado library
cd /path/to/Deneb-master/source/Avocado/inc
ln -sf $CONDA_PREFIX/include conda_include

# Create symbolic links for Deneb library  
cd /path/to/Deneb-master/source/Deneb/inc
ln -sf $CONDA_PREFIX/include conda_include
```

### Header file modification: (Already modified in my repository)
```
// In avocado_mpi.h
// Change: #include <mpi.h>
// To:     #include "conda_include/mpi.h"
```

### PETSc Headers: Environment Variable Approach
Problem: PETSc has complex internal header structure (petsc/private/, petsc/finclude/) that symbolic links cannot handle.

Solution: Use environment variables + CMake target_include_directories.

CMakeLists.txt modification(Already modified)

```
# Add at the end of CMakeLists.txt
if(DEFINED ENV{CONDA_PREFIX})
    target_include_directories(${PROJECT_NAME} PRIVATE "$ENV{CONDA_PREFIX}/include")
    target_compile_definitions(${PROJECT_NAME} PRIVATE CONDA_HEADERS_AVAILABLE)
    message(STATUS "TARGET: Added conda include directory: $ENV{CONDA_PREFIX}/include")
endif()
```

Header file usage:

```
// In deneb_equation.h - keep original format
#include <petsc.h>  // No symbolic link needed
```

## OpenBLAS Linking Fix

### Problem: Undefined Symbols
```
Undefined symbols for architecture arm64:
"___kmpc_for_static_fini"     ← OpenMP functions
"omp_get_max_threads"         ← OpenMP functions  
"__gfortran_concat_string"    ← Fortran runtime functions
```

Solution: Auto-detect OpenMP and Fortran Libraries
CMakeLists.txt modification in OpenBLAS section(already modified):

```
if(${BLASLAPACK} STREQUAL OpenBLAS)
    # ... existing code ...
    else()
        include_directories(${OPENBLAS_INCLUDE_DIR})
        set(EXTRA_LIBS ${EXTRA_LIBS} ${OPENBLAS_LIBRARY})
        
        # Auto-detect and add OpenMP and Fortran libraries
        if(APPLE AND DEFINED ENV{CONDA_PREFIX})
            find_library(OMP_LIBRARY NAMES omp libomp PATHS ${OPENBLAS_LIB} $ENV{CONDA_PREFIX}/lib)
            find_library(GFORTRAN_LIBRARY NAMES gfortran libgfortran PATHS ${OPENBLAS_LIB} $ENV{CONDA_PREFIX}/lib)
            
            if(OMP_LIBRARY)
                set(EXTRA_LIBS ${EXTRA_LIBS} ${OMP_LIBRARY})
                message(STATUS "Added OpenMP library: ${OMP_LIBRARY}")
            endif()
            
            if(GFORTRAN_LIBRARY)
                set(EXTRA_LIBS ${EXTRA_LIBS} ${GFORTRAN_LIBRARY})
                message(STATUS "Added GFortran library: ${GFORTRAN_LIBRARY}")
            endif()
        endif()
        
        set(EXTRA_LIBS ${EXTRA_LIBS} ${OPENBLAS_FC_LIBRARY})
    endif()
```

## Complete Build Process
```
# 1. Prepare environment
conda activate Deneb
export CPLUS_INCLUDE_PATH="$CONDA_PREFIX/include:$CPLUS_INCLUDE_PATH"
export C_INCLUDE_PATH="$CONDA_PREFIX/include:$C_INCLUDE_PATH"

# 2. Navigate to project directory
cd /path/to/Deneb-master

# 3. Clean build directory
cd build && rm -rf *

# 4. Run CMake
cmake .. \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_BUILD_TYPE=Release \
  -DBLASLAPACK=OpenBLAS

# 5. Compile
make -j8
```

## Troubleshooting

Common Warnings (Safe to Ignore)
```
ld: warning: could not create compact unwind for *: does not use standard frame
```

## Debug Tips

```
# Check conda environment
conda info --envs
echo $CONDA_PREFIX

# Verify library installation
conda list | grep -i petsc
ls $CONDA_PREFIX/include/petsc.h

# Check environment variables
echo $CPLUS_INCLUDE_PATH
echo $C_INCLUDE_PATH
```

Note: This guide is specifically tested on Apple Silicon Macs (M1/M2/M3). For Intel-based Macs, some modifications might be necessary.








