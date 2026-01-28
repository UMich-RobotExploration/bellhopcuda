# BellhopCUDA/BellhopCXX - Comprehensive Documentation

## Table of Contents

1. [Introduction](#introduction)
2. [Project Overview](#project-overview)
3. [Architecture](#architecture)
4. [Getting Started](#getting-started)
5. [Core Concepts](#core-concepts)
6. [API Reference](#api-reference)
7. [Build System](#build-system)
8. [Examples and Usage](#examples-and-usage)
9. [Advanced Topics](#advanced-topics)
10. [Testing and Validation](#testing-and-validation)

---

## Introduction

### What is BellhopCUDA/BellhopCXX?

BellhopCUDA/BellhopCXX is a high-performance C++/CUDA port of the BELLHOP and BELLHOP3D underwater acoustics simulators, originally developed by Dr. Michael B. Porter. This project brings modern parallel computing capabilities to underwater acoustic ray tracing, enabling:

- **Multi-threaded CPU execution** (bellhopcxx) with performance scaling roughly proportional to CPU core count
- **GPU-accelerated computation** (bellhopcuda) on NVIDIA GPUs for 10x-100x speedup
- **Library-based API** allowing integration into other programs without file I/O overhead
- **Enhanced numerical stability** and robustness compared to the original Fortran implementation

### Key Features

- **Three Dimensionality Modes**:
  - **2D**: Traditional range-depth plane acoustic propagation
  - **3D**: Full three-dimensional ocean environment and ray propagation
  - **Nx2D**: Hybrid mode with 3D ocean but 2D rays (multiple 2D slices)

- **Multiple Run Types**:
  - **Ray tracing**: Visualize individual acoustic ray paths
  - **Transmission Loss (TL)**: Compute acoustic field intensity throughout the ocean
  - **Eigenrays**: Find specific rays connecting source to receivers
  - **Arrivals**: Detailed arrival structure analysis

- **Flexible SSP (Sound Speed Profile) Models**:
  - N²-linear, C-linear, cubic spline, PCHIP interpolation
  - 2D/3D spatially-varying profiles (quadrilateral/hexahedral)
  - Analytic profiles for testing

- **Advanced Beam Models**:
  - Geometric ray-centered and Cartesian beams
  - Cerveny ray-centered and Cartesian beams
  - Simple Gaussian beams

### Performance Characteristics

**CPU (bellhopcxx)**:
- Multithreading scales with logical CPU cores
- Typical speedup: 10-30x over original BELLHOP on 12-core/24-thread systems
- No GPU required, works on all platforms (Linux, Windows, Mac)

**GPU (bellhopcuda)**:
- Consumer GPUs (RTX 3060): 10-50x speedup over BELLHOP
- Server GPUs (A100, GH200): 20-100x speedup over BELLHOP
- Best performance with tens of thousands of rays
- Requires NVIDIA GPU with CUDA support (Linux/Windows only)

### Who Should Use This?

- **Researchers** in underwater acoustics seeking faster simulation times
- **Software developers** integrating acoustic propagation into larger systems
- **Marine scientists** running parameter sweeps or Monte Carlo studies
- **Anyone** currently using BELLHOP/BELLHOP3D who wants better performance

---

## Project Overview

### Repository Structure

```
bellhopcuda/
├── include/bhc/          # Public API headers
│   ├── bhc.hpp          # Main API declarations
│   ├── structs.hpp      # Data structure definitions
│   ├── math.hpp         # Mathematical utilities
│   └── platform.hpp     # Platform-specific definitions
│
├── src/                 # Implementation source files
│   ├── api.cpp          # Main API implementation
│   ├── common*.hpp      # Shared internal headers
│   ├── module/          # Parameter reading/writing modules
│   │   ├── atten.cpp    # Attenuation handling
│   │   ├── ssp.cpp      # Sound speed profile
│   │   ├── boundary.cpp # Boundary conditions
│   │   └── ...          # Other modules
│   ├── mode/            # Run mode implementations
│   │   ├── ray.*        # Ray tracing mode
│   │   ├── field.*      # Transmission loss field
│   │   ├── eigen.*      # Eigenray mode
│   │   └── arr.*        # Arrivals mode
│   └── util/            # Utility functions
│       ├── errors.cpp   # Error handling
│       ├── timing.cpp   # Performance timing
│       └── ...
│
├── config/              # CMake build configuration
│   ├── cuda/            # CUDA-specific build rules
│   ├── cxx/             # C++-specific build rules
│   └── examples/        # Example build configuration
│
├── examples/            # Example programs demonstrating API usage
│   ├── defaults.cpp     # Basic usage with defaults
│   ├── background.cpp   # Non-blocking computation
│   ├── province.cpp     # Province-based SSP
│   ├── readout.cpp      # Reading saved results
│   └── writeenv.cpp     # Writing environment files
│
├── test/                # Test environments and input files
│   ├── in/              # Test .env files
│   └── individual/      # Individual test cases
│
├── doc/                 # Documentation
│   ├── compilation.md   # Build instructions
│   ├── accuracy.md      # Accuracy validation
│   ├── performance.md   # Performance benchmarks
│   └── faq.md          # Frequently asked questions
│
├── glm/                 # GLM vector math library (submodule)
├── CMakeLists.txt       # Top-level build configuration
└── README.md            # Project readme

```

### Development History

- **Original BELLHOP/BELLHOP3D**: Fortran code by Dr. Michael B. Porter (1983-2022)
- **BellhopCUDA/BellhopCXX**: C++/CUDA port by Marine Physical Lab at Scripps Oceanography
- **Current Version**: 1.5.0+ with ongoing improvements

### License

GNU General Public License v3.0 or later. This is free software - you can redistribute and/or modify it under the terms of the GPL as published by the Free Software Foundation.

---

## Architecture

### Design Philosophy

The codebase is designed around several key principles:

1. **Template-based dimensionality**: Uses C++ templates to share code between 2D, 3D, and Nx2D modes while maintaining type safety
2. **Unified CPU/GPU codebase**: Single source code compiles to both CPU and GPU versions using libcudacxx
3. **Modular design**: Separate modules for reading parameters, running simulations, and writing output
4. **Memory safety**: Careful memory management with clear ownership semantics
5. **Numerical stability**: Improved edge case handling compared to original Fortran

### Template Parameters

The codebase uses two primary template parameters:

- **`O3D`** (Ocean 3D): Whether the ocean environment (SSP, boundaries) is 3D
  - `false`: 2D mode
  - `true`: 3D or Nx2D mode

- **`R3D`** (Rays 3D): Whether rays propagate in 3D
  - `false`: 2D rays (used in 2D mode and Nx2D mode)
  - `true`: 3D rays (used in 3D mode)

**Mode combinations**:
```
O3D=false, R3D=false → 2D mode
O3D=true,  R3D=false → Nx2D mode  
O3D=true,  R3D=true  → 3D mode
```

### Core Components

#### 1. Setup System (modules/)

The setup system reads environment files and initializes parameters. Each module handles a specific aspect:

- **Atten**: Volume attenuation (absorption/scattering)
- **SSP**: Sound speed profile
- **Boundary**: Top/bottom boundary conditions
- **SxSy**: Source X/Y positions
- **SzRz**: Source/receiver Z positions (depths)
- **RcvrRanges**: Receiver range positions
- **RcvrBearings**: Receiver bearing angles
- **RayAngles**: Ray launch angles (elevation/bearing)
- **BeamInfo**: Beam/ray configuration
- **ReflCoef**: Reflection coefficients
- **SBP**: Source beam pattern

Each module:
- Reads from environment file
- Validates input data
- Preprocesses (e.g., converts km to m, degrees to radians)
- Stores in appropriate data structures

#### 2. Run System (mode/)

The run system executes the selected simulation mode:

- **Ray**: Traces individual rays and stores full trajectories
- **Field/TL**: Computes transmission loss field at receiver positions
- **Eigen**: Finds eigenrays connecting sources to receivers
- **Arr**: Computes detailed arrival structure

Each mode:
- Initializes ray starting conditions
- Steps rays through the ocean
- Applies boundary interactions
- Computes influence/contribution to receivers (TL/arrivals)
- Stores results in output structures

#### 3. Core Ray Tracing (trace.hpp, step.hpp)

The fundamental ray tracing engine:

- **Trace**: Main loop advancing ray through medium
- **Step**: Single step computation using ray equations
- **SSP evaluation**: Interpolates sound speed and derivatives
- **Boundary detection**: Determines when rays hit boundaries
- **Reflection/refraction**: Handles boundary interactions

#### 4. Influence Functions (influence.hpp)

Computes how rays contribute to acoustic field at receivers:

- **Geometric beams**: Hat and Gaussian beam shapes
- **Cerveny beams**: Dynamic ray theory with beam spreading
- **Simple Gaussian beams**: Simplified Gaussian model

Different formulations for ray-centered vs Cartesian coordinates, 2D vs 3D.

#### 5. Output System

Handles reading/writing standard BELLHOP file formats:

- **SHDFile** (.shd): Shade file with transmission loss field
- **ARRFile** (.arr): Arrivals file with detailed arrival structure
- **RAYFile** (.ray): Ray coordinates file
- **Environment files**: .env and associated .ssp, .bty, .ati, etc.

### Data Flow

```
┌─────────────────┐
│  Environment    │
│  Files (.env,   │
│  .ssp, .bty...) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Module System  │
│  (Parse/Validate│
│   Parameters)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   bhcParams     │◄─── User can modify parameters via API
│   Structure     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Run System    │
│  (Ray Tracing,  │
│   Field Comp.)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  bhcOutputs     │
│   Structure     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Output Files   │
│  (.shd, .arr,   │
│   .ray)         │
└─────────────────┘
```

### Parallelization Strategy

#### CPU Multithreading

- Work distributed across rays (each thread processes subset of rays)
- Thread pool initialized at startup, reused for multiple runs
- Lock-free for ray tracing itself
- Synchronization only for shared receiver contributions (TL/arrivals)
- Atomic operations for progress tracking

#### GPU (CUDA)

- Massive parallelism: one CUDA thread per ray (or groups of rays)
- Global memory for parameters and results
- Shared memory for receiver data in TL computations
- Efficient memory access patterns crucial for performance
- Kernel launches for different phases (initialize, trace, postprocess)

---

## Getting Started

### Prerequisites

**For bellhopcxx (CPU only)**:
- C++17 compatible compiler (GCC 7+, Clang 5+, MSVC 2017+, Intel C++ 19+)
- CMake 3.27 or later
- Git (for cloning repository and submodules)

**For bellhopcuda (GPU acceleration)**:
- All bellhopcxx prerequisites
- NVIDIA CUDA Toolkit 11.5 or later (12.4+ recommended)
- NVIDIA GPU with compute capability 3.5+ (7.0+ recommended)
- Appropriate GPU drivers

### Installation

#### 1. Clone the Repository

```bash
git clone https://github.com/A-New-BellHope/bellhopcuda.git
cd bellhopcuda
git submodule update --init --recursive
```

The `--recursive` flag is essential to fetch the GLM vector math library.

#### 2. Build CPU Version (bellhopcxx)

```bash
mkdir build
cd build
cmake .. -DBHC_ENABLE_CUDA=OFF
cmake --build . --config Release
```

**Build options**:
- `-DBHC_DEBUG=ON`: Enable debugging features (slower)
- `-DBHC_BUILD_EXAMPLES=ON`: Build example programs (default ON)
- `-DBHC_USE_FLOATS=ON`: Use 32-bit floats instead of 64-bit (faster but less accurate)

#### 3. Build GPU Version (bellhopcuda)

```bash
mkdir build
cd build
cmake ..  # CUDA enabled by default
cmake --build . --config Release
```

Additional CUDA options:
- `-DCUDA_ALL_ARCHES=ON`: Build for all GPUs in system (longer compile time)
- `-DBHC_ENABLE_CUDA=OFF`: Disable CUDA even if toolkit is available

#### 4. Verify Installation

After building, executables will be in `build/bin/`:
- `bellhopcxx` / `bellhopcuda`: Main multi-mode executables
- `bellhopcxx2d`, `bellhopcxx3d`, `bellhopcxxNx2D`: Mode-specific versions
- `bellhopcuda2d`, `bellhopcuda3d`, `bellhopcudaNx2D`: GPU mode-specific versions
- Example programs if enabled

Test with a sample environment:
```bash
cd build/bin
./bellhopcxx --2D ../../test/in/MunkB_ray
```

This should produce ray tracing output for the classic Munk profile test case.

### First Steps

#### Using as Drop-in BELLHOP Replacement

If you have existing BELLHOP/BELLHOP3D environment files:

1. Rename `bellhopcxx.exe` to `bellhop.exe` (or `bellhop3d.exe`)
2. Replace your existing BELLHOP executable
3. Run normally - it reads the same input files and produces the same output formats

The executables accept standard BELLHOP command-line arguments plus mode flags:
```bash
bellhopcxx --2D FileRoot    # 2D mode
bellhopcxx --3D FileRoot    # 3D mode
bellhopcxx --Nx2D FileRoot  # Nx2D mode
```

#### Using as a Library

See [Examples and Usage](#examples-and-usage) section for detailed examples.

Basic pattern:
```cpp
#define BHC_DLL_IMPORT 1  // If using DLL on Windows
#include <bhc/bhc.hpp>

int main() {
    // Initialize
    bhc::bhcParams<false> params;      // false = 2D mode
    bhc::bhcOutputs<false, false> outputs;
    bhc::bhcInit init;
    init.FileRoot = "path/to/environment";
    
    // Setup from environment file
    bhc::setup(init, params, outputs);
    
    // Run simulation
    bhc::run(params, outputs);
    
    // Access results in outputs structure
    // ...
    
    // Cleanup
    bhc::finalize(params, outputs);
    return 0;
}
```

---

## Core Concepts

### Sound Speed Profiles (SSP)

The sound speed profile describes how sound velocity varies throughout the ocean. This is the most important environmental parameter affecting acoustic propagation.

#### SSP Types

**1D Profiles** (depth-varying only):

- **N²-linear** (`Type = 'N'`): Linear variation in N² = c² where c is sound speed
  - Fast, simple, good for many oceanic profiles
  - Commonly used for deep ocean

- **C-linear** (`Type = 'C'`): Piecewise linear sound speed
  - Simple, intuitive
  - Can create discontinuities at layer boundaries

- **Cubic spline** (`Type = 'S'`): Smooth cubic interpolation
  - C² continuous (smooth second derivative)
  - Good for general use

- **PCHIP** (`Type = 'P'`): Piecewise Cubic Hermite Interpolating Polynomial
  - Shape-preserving (no oscillations)
  - Better for profiles with features like sound channels

**Multi-dimensional Profiles**:

- **Quadrilateral** (`Type = 'Q'`, 2D only): Range and depth varying
  - Define c(r, z) on a 2D grid
  - Bilinear interpolation

- **Hexahedral** (`Type = 'H'`, 3D/Nx2D): X, Y, and depth varying
  - Define c(x, y, z) on 3D grid
  - Trilinear interpolation

- **Analytic** (`Type = 'A'`): Mathematically-defined profiles
  - For testing and idealized scenarios
  - Examples: Munk profile, isovelocity

#### SSP in API

Reading from file:
```cpp
// Automatically loaded when setup() reads .env file
// SSP data in params.ssp
```

Setting programmatically (1D example):
```cpp
params.ssp->Type = 'S';  // Cubic spline
params.ssp->NPts = 101;  // Number of depth points

for (int i = 0; i < params.ssp->NPts; i++) {
    params.ssp->z[i] = i * 50.0;  // Depths: 0, 50, 100, ... 5000 m
    params.ssp->alphaR[i] = 1500.0 + 10.0 * i;  // Sound speed
    params.ssp->betaR[i] = 0.0;   // Shear speed (usually 0 in water)
    params.ssp->rho[i] = 1.0;     // Density
    params.ssp->alphaI[i] = 0.0;  // Attenuation
    params.ssp->betaI[i] = 0.0;
}
params.ssp->dirty = true;  // Mark as needing preprocessing
```

### Boundaries

#### Boundary Types

**Top (surface)**:
- `'V'`: Vacuum (perfect reflector, pressure release)
- `'R'`: Rigid (perfect reflector, rigid)
- `'A'`: Acousto-elastic (specify acoustic properties)
- `'F'`: File-based reflection coefficients
- `'*'`: Curvilinear (follows grid lines)

**Bottom**:
- Same options as top
- Most commonly use 'A' with sediment properties

#### Boundary Geometry

**2D**: Range-depth coordinates
```cpp
// Bathymetry example
int NPts = 100;
bhc::extsetup_bathymetry(params, NPts, 0);
for (int i = 0; i < NPts; i++) {
    params.bdinfo->bot.bd[i].x.x = i * 100.0;  // Range in meters
    params.bdinfo->bot.bd[i].x.y = 100.0 + 50.0 * sin(i * 0.1);  // Depth
    // Set acoustic properties if needed
}
params.bdinfo->bot.dirty = true;
```

**3D**: X-Y-Z coordinates on rectangular grid
```cpp
int2 NPts = {50, 50};  // 50x50 grid
bhc::extsetup_bathymetry(params, NPts, 0);
for (int ix = 0; ix < NPts.x; ix++) {
    for (int iy = 0; iy < NPts.y; iy++) {
        int idx = ix * NPts.y + iy;
        params.bdinfo->bot.bd[idx].x.x = ix * 200.0;  // X
        params.bdinfo->bot.bd[idx].x.y = iy * 200.0;  // Y  
        params.bdinfo->bot.bd[idx].x.z = 1000.0;      // Depth
    }
}
```

### Ray Theory Basics

Acoustic propagation is modeled using ray theory, a high-frequency approximation to the wave equation. Rays:

- Bend according to Snell's law in varying sound speed
- Reflect/refract at boundaries
- Carry amplitude that spreads geometrically
- Accumulate phase along path

#### Ray Equations

In 2D (range r, depth z):
```
dx/dτ = c² ∇c / |∇c|
```

Where:
- `x = (r, z)`: ray position
- `τ`: arc length parameter
- `c`: sound speed
- `∇c`: sound speed gradient

#### Beam Types

**Geometric beams**: Simple geometric spreading, fast
- Ray-centered: beam width in natural ray coordinates
- Cartesian: beam width in fixed Cartesian coordinates
- Hat function or Gaussian shape

**Cerveny beams**: Dynamic ray theory, more accurate
- Tracks beam width evolution via ray equations
- Accounts for curvature, focusing, defocusing
- Required for quantitative transmission loss

**Simple Gaussian beams**: Compromise between geometric and Cerveny
- Gaussian envelope
- Simpler than full Cerveny
- Good for many applications

### Sources and Receivers

#### Source Positions

**2D mode**: 
- One source at (r=0, z=Sz[0])
- Multiple depths Sz[] for different source depths

**3D/Nx2D**:
- Multiple X/Y positions: Sx[], Sy[]
- Multiple depths: Sz[]
- Total sources: NSx × NSy × NSz

#### Receiver Positions

Receivers defined by:
- **Ranges** Rr[]: Radial distances from source (2D/3D)
- **Bearings** theta[]: Azimuthal angles (3D/Nx2D only; single value 0° in 2D)
- **Depths** Rz[]: Receiver depths

Total receivers: NRr × Ntheta × NRz

#### Ray Launch Angles

**Elevation angles** (alpha): Vertical launch angle
- Measured from horizontal
- Positive = upward, negative = downward

**Bearing angles** (beta): Horizontal launch angle (3D only)
- Azimuthal direction
- In Nx2D, replaced by receiver bearings

### Run Types

#### Ray Tracing ('R')

Traces and stores full ray trajectories.

**Use cases**:
- Visualizing acoustic paths
- Understanding propagation mechanisms
- Debugging environment setup

**Output**: Ray file (.ray) with coordinates along each ray

**Performance**: Moderate - limited by I/O for writing rays

#### Transmission Loss ('C', 'S', 'I')

Computes acoustic intensity at receiver positions.

**Types**:
- `'C'`: Coherent TL (preserves phase)
- `'S'`: Semi-coherent TL
- `'I'`: Incoherent TL (intensity only)

**Use cases**:
- Sound field prediction
- Coverage area analysis
- Sonar performance estimation

**Output**: Shade file (.shd) with complex field values

**Performance**: Best speedups, especially with many rays

#### Eigenrays ('E')

Finds specific rays connecting source to each receiver.

**Use cases**:
- Identifying propagation paths
- Multipath analysis
- Time-of-arrival prediction

**Output**: Ray file (.ray) with only eigenrays

**Performance**: Good with few receivers, poor with many

#### Arrivals ('A', 'a')

Detailed arrival structure at each receiver.

**Types**:
- `'A'`: Standard arrivals
- `'a'`: Arrivals with amplitude/phase

**Use cases**:
- Multipath time series
- Impulse response estimation
- Matched field processing

**Output**: Arrivals file (.arr) with arrival details

**Performance**: Moderate, depends on receiver layout

---

## API Reference

### Initialization and Setup

#### `bhcInit` Structure

Configuration for initializing a BellhopCUDA/CXX instance.

```cpp
struct bhcInit {
    int32_t numThreads = -1;
    size_t maxMemory = 4ull * 1024ull * 1024ull * 1024ull;
    bool useRayCopyMode = false;
    int gpuIndex = 0;
    const char *FileRoot = nullptr;
    void (*prtCallback)(const char *message) = nullptr;
    void (*outputCallback)(const char *message) = nullptr;
    void (*completedCallback)() = nullptr;
};
```

**Members**:
- `numThreads`: Number of worker threads. `-1` = use all logical cores
- `maxMemory`: Maximum memory in bytes (default 4 GiB)
- `useRayCopyMode`: Memory management strategy for ray storage
- `gpuIndex`: CUDA device index (0 = first/fastest GPU)
- `FileRoot`: Path to environment file (without .env extension), or `nullptr` for defaults
- `prtCallback`: Callback for print file messages, or `nullptr` to write .prt file
- `outputCallback`: Callback for terminal output, or `nullptr` for stdout
- `completedCallback`: Called when computation completes (useful for non-blocking mode)

#### `setup()` - Initialize from Environment File

```cpp
template<bool O3D, bool R3D>
bool setup(const bhcInit &init, bhcParams<O3D> &params, 
           bhcOutputs<O3D, R3D> &outputs);
```

**Purpose**: Initialize parameters from environment file or defaults.

**Parameters**:
- `init`: Initialization configuration
- `params`: [out] Parameter structure to initialize
- `outputs`: [out] Output structure to initialize

**Returns**: `true` on success, `false` on error

**Example**:
```cpp
bhc::bhcInit init;
init.FileRoot = "MunkB_ray";
init.numThreads = 8;

bhc::bhcParams<false> params;        // 2D
bhc::bhcOutputs<false, false> outputs;

if (!bhc::setup(init, params, outputs)) {
    // Handle error
}
```

### Parameter Modification

After `setup()`, you can modify parameters before calling `run()`. Use these functions to reallocate arrays:

#### `extsetup_sxsy()` - Set Source X/Y Positions

```cpp
template<bool O3D>
void extsetup_sxsy(bhcParams<O3D> &params, int32_t NSx, int32_t NSy);
```

**Purpose**: Reallocate source X/Y position arrays.

**Notes**: 
- In 2D, must have NSx=1, NSy=1 at (0,0)
- Set `params.Pos->SxSyInKm` based on your units
- Fill `params.Pos->Sx[i]` and `Sy[i]` after calling

#### `extsetup_sz()` - Set Source Depths

```cpp
template<bool O3D>
void extsetup_sz(bhcParams<O3D> &params, int32_t NSz);
```

**Purpose**: Reallocate source depth array.

**Notes**: Fill `params.Pos->Sz[i]` with depths in meters

#### `extsetup_rcvrranges()` - Set Receiver Ranges

```cpp
template<bool O3D>
void extsetup_rcvrranges(bhcParams<O3D> &params, int32_t NRr);
```

**Purpose**: Reallocate receiver range array.

**Notes**: 
- Set `params.Pos->RrInKm` based on units
- Fill `params.Pos->Rr[i]` after calling

#### `extsetup_rcvrdepths()` - Set Receiver Depths

```cpp
template<bool O3D>
void extsetup_rcvrdepths(bhcParams<O3D> &params, int32_t NRz);
```

**Purpose**: Reallocate receiver depth array.

**Notes**: Fill `params.Pos->Rz[i]` with depths in meters

#### `extsetup_rcvrbearings()` - Set Receiver Bearings

```cpp
template<bool O3D>
void extsetup_rcvrbearings(bhcParams<O3D> &params, int32_t Ntheta);
```

**Purpose**: Reallocate receiver bearing array.

**Notes**:
- In 2D, must have Ntheta=1 at 0°
- Fill `params.Pos->theta[i]` in degrees after calling

#### `extsetup_rayelevations()` - Set Ray Elevation Angles

```cpp
template<bool O3D>
void extsetup_rayelevations(bhcParams<O3D> &params, int32_t n);
```

**Purpose**: Reallocate ray elevation angle array.

**Notes**:
- Set `params.Angles->alpha.inDegrees`
- Fill `params.Angles->alpha.angles[i]` after calling

#### `extsetup_raybearings()` - Set Ray Bearing Angles

```cpp
template<bool O3D>
void extsetup_raybearings(bhcParams<O3D> &params, int32_t n);
```

**Purpose**: Reallocate ray bearing angle array.

**Notes**:
- In 2D, must have n=1 at 0°
- In Nx2D, these are replaced by receiver bearings
- Set `params.Angles->beta.inDegrees`

#### `extsetup_ssp_quad()` - Set 2D Quadrilateral SSP

```cpp
void extsetup_ssp_quad(bhcParams<false> &params, int32_t NPts, int32_t Nr);
```

**Purpose**: Setup range-dependent SSP (2D only).

**After calling**:
- Fill `params.ssp->z[iz]` with depths
- Fill `params.ssp->Seg.r[ir]` with ranges
- Fill `params.ssp->cMat[iz * Nr + ir]` with sound speeds
- Set `params.ssp->rangeInKm`
- Set `params.Bdry->Top.hs.Depth = ssp->z[0]`
- Set `params.Bdry->Bot.hs.Depth = ssp->z[NPts-1]`

#### `extsetup_ssp_hexahedral()` - Set 3D Hexahedral SSP

```cpp
void extsetup_ssp_hexahedral(bhcParams<true> &params, 
                             int32_t Nx, int32_t Ny, int32_t Nz);
```

**Purpose**: Setup spatially-varying SSP (3D/Nx2D).

**After calling**:
- Fill `params.ssp->Seg.x[ix]`, `.y[iy]`, `.z[iz]`
- Fill `params.ssp->cMat[(ix*Ny+iy)*Nz+iz]` with sound speeds
- Set `params.ssp->rangeInKm`
- Set top/bottom depths as with quad

#### `extsetup_altimetry()` - Set Surface Elevation

```cpp
template<bool O3D>
void extsetup_altimetry(bhcParams<O3D> &params, const IORI2<O3D> &NPts);
```

**Purpose**: Setup spatially-varying altimetry.

**After calling**:
- 2D: Fill `params.bdinfo->top.bd[i].x` with (range, depth)
- 3D: Fill on X-Y grid
- Set `params.bdinfo->top.dirty = true`

#### `extsetup_bathymetry()` - Set Bottom Depth

```cpp
template<bool O3D>
void extsetup_bathymetry(bhcParams<O3D> &params, const IORI2<O3D> &NPts,
                         const int32_t &NBotProvinces);
```

**Purpose**: Setup spatially-varying bathymetry.

**Parameters**:
- `NPts`: int32_t (2D) or int2 (3D) with number of points
- `NBotProvinces`: Number of bottom provinces (usually 0 for uniform bottom)

**After calling**: Same as altimetry, for bottom

#### `extsetup_blocking()` - Set Blocking Mode

```cpp
template<bool O3D>
void extsetup_blocking(bhcParams<O3D> &params, const bool &blocking);
```

**Purpose**: Request non-blocking (background) execution.

**Usage**:
```cpp
extsetup_blocking(params, false);  // Non-blocking
run(params, outputs);               // Returns immediately
// ... do other work ...
while (!done) {
    int progress = get_percent_progress(params);
    // Update UI, etc.
}
postprocess(params, outputs);       // Get results
```

### Execution

#### `echo()` - Validate and Print Parameters

```cpp
template<bool O3D>
bool echo(bhcParams<O3D> &params);
```

**Purpose**: Validate parameters and write summary to PRTFile.

**Returns**: `true` if valid, `false` if errors detected

**Notes**: Automatically called by `run()`, but useful for checking before running

#### `run()` - Execute Simulation

```cpp
template<bool O3D, bool R3D>
bool run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);
```

**Purpose**: Run the simulation with current parameters.

**Returns**: `true` on success, `false` on error

**Notes**:
- Preprocesses parameters if marked dirty
- Selects run mode based on `params.Beam->RunType`
- In blocking mode, returns when complete
- In non-blocking mode, returns immediately; call `postprocess()` when done

#### `postprocess()` - Finalize Non-Blocking Run

```cpp
template<bool O3D, bool R3D>
bool postprocess(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);
```

**Purpose**: Wait for non-blocking run to complete and finalize results.

**Returns**: `true` on success, `false` on error

**Notes**: Only needed if you called `extsetup_blocking(params, false)`

#### `get_percent_progress()` - Check Progress

```cpp
template<bool O3D>
int get_percent_progress(bhcParams<O3D> &params);
```

**Purpose**: Get progress percentage (0-100) of current run.

**Returns**: Integer 0-100

**Notes**: Thread-safe, can call while run() is in progress

### Output

#### `writeout()` - Write Results to Files

```cpp
template<bool O3D, bool R3D>
bool writeout(const bhcParams<O3D> &params, const bhcOutputs<O3D, R3D> &outputs,
              const char *FileRoot);
```

**Purpose**: Write results in BELLHOP-compatible format.

**Parameters**:
- `FileRoot`: Output file base name, or `nullptr` to use original FileRoot

**Output files** (depending on run type):
- `.ray`: Ray coordinates (ray runs)
- `.shd`: Shade file with TL field (TL runs)
- `.arr`: Arrivals data (arrivals runs)

#### `readout()` - Read Results from Files

```cpp
template<bool O3D, bool R3D>
bool readout(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs,
             const char *FileRoot);
```

**Purpose**: Load previously-saved results.

**Parameters**:
- `FileRoot`: Input file base name, or `nullptr` to use original FileRoot

**Notes**: `params` should already be initialized; may be updated to match file contents

#### `writeenv()` - Write Environment File

```cpp
template<bool O3D>
bool writeenv(bhcParams<O3D> &params, const char *FileRoot);
```

**Purpose**: Save current parameters to environment file(s).

**Output**: `.env` file and associated `.ssp`, `.bty`, etc. as needed

### Cleanup

#### `finalize()` - Free Memory

```cpp
template<bool O3D, bool R3D>
void finalize(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);
```

**Purpose**: Release all allocated memory.

**Notes**: 
- Can call `run()` multiple times before `finalize()`
- Must call `finalize()` before program exit to avoid leaks

### Utility

#### `get_ssp()` - Query Sound Speed

```cpp
template<bool O3D, bool R3D>
bool get_ssp(bhcParams<O3D> &params, const VEC23<R3D> &x, float &sound_speed);
```

**Purpose**: Get interpolated sound speed at a point.

**Parameters**:
- `x`: Position (vec2 for 2D/Nx2D, vec3 for 3D)
- `sound_speed`: [out] Sound speed in m/s

**Returns**: `true` on success, `false` if point outside domain

---

## Build System

### CMake Configuration

The build system uses CMake with a modular configuration. Top-level options:

#### Core Options

```cmake
# Enable/disable CUDA
option(BHC_ENABLE_CUDA "Build CUDA version in addition to C++ version" ON)

# Enable/disable example programs
option(BHC_BUILD_EXAMPLES "Build example programs" ON)

# Floating point precision
option(BHC_USE_FLOATS "Use 32-bit floats instead of 64-bit" OFF)

# Debug mode
option(BHC_DEBUG "Enable debugging features, reduces performance" OFF)
```

#### Dimensionality Options

```cmake
option(BHC_DIM_ENABLE_2D   "Enable 2D runs" ON)
option(BHC_DIM_ENABLE_3D   "Enable 3D runs" ON)
option(BHC_DIM_ENABLE_NX2D "Enable Nx2D runs" ON)
```

Disabling unused dimensionalities reduces compile time and binary size.

#### Run Type Options

```cmake
option(BHC_RUN_ENABLE_TL        "Enable TL runs" ON)
option(BHC_RUN_ENABLE_EIGENRAYS "Enable eigenrays runs" ON)
option(BHC_RUN_ENABLE_ARRIVALS  "Enable arrivals runs" ON)
```

#### Influence Function Options

```cmake
option(BHC_INFL_ENABLE_CERVENY_RAYCEN "Enable Cerveny ray-centered" ON)
option(BHC_INFL_ENABLE_CERVENY_CART   "Enable Cerveny Cartesian" ON)
option(BHC_INFL_ENABLE_GEOM_RAYCEN    "Enable geometric ray-centered" ON)
option(BHC_INFL_ENABLE_GEOM_CART      "Enable geometric Cartesian" ON)
option(BHC_INFL_ENABLE_SGB            "Enable simple Gaussian beams" ON)
```

#### SSP Type Options

```cmake
option(BHC_SSP_ENABLE_N2LINEAR   "Enable N²-linear SSP" ON)
option(BHC_SSP_ENABLE_CLINEAR    "Enable C-linear SSP" ON)
option(BHC_SSP_ENABLE_CUBIC      "Enable cubic spline SSP" ON)
option(BHC_SSP_ENABLE_PCHIP      "Enable PCHIP SSP" ON)
option(BHC_SSP_ENABLE_QUAD       "Enable quadrilateral 2D SSP" ON)
option(BHC_SSP_ENABLE_HEXAHEDRAL "Enable hexahedral 3D SSP" ON)
option(BHC_SSP_ENABLE_ANALYTIC   "Enable analytic SSP" ON)
```

### Build Targets

The build produces multiple targets:

**Executables**:
- `bellhopcxx` / `bellhopcuda`: Main multi-mode executables
- `bellhopcxx2d`, `bellhopcxx3d`, `bellhopcxxNx2D`: Single-mode versions
- `bellhopcuda2d`, `bellhopcuda3d`, `bellhopcudaNx2D`: GPU single-mode

**Libraries**:
- `libbhc2d.a` / `libbhc2d.so` / `bhc2d.dll`: 2D library
- `libbhc3d.a` / `libbhc3d.so` / `bhc3d.dll`: 3D library
- `libbhcNx2D.a` / `libbhcNx2D.so` / `bhcNx2D.dll`: Nx2D library

**Examples** (if enabled):
- `bhc_defaults`, `bhc_background`, `bhc_province`, etc.

### Platform-Specific Notes

#### Linux

Standard CMake workflow:
```bash
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

#### Windows (Visual Studio)

```bash
# Command line
cmake .. -G "Visual Studio 17 2022" -A x64
cmake --build . --config Release

# Or open folder in Visual Studio
# It auto-detects CMakeLists.txt
```

#### macOS

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release -DBHC_ENABLE_CUDA=OFF
make -j$(sysctl -n hw.ncpu)
```

Note: CUDA not available on macOS

### Linking Against the Library

In your CMakeLists.txt:

```cmake
find_package(bhc REQUIRED)

add_executable(myprogram main.cpp)
target_link_libraries(myprogram bhc::bhc2d)  # or bhc::bhc3d, bhc::bhcNx2D
```

Or manually:

```cmake
target_include_directories(myprogram PRIVATE /path/to/bellhopcuda/include)
target_link_libraries(myprogram /path/to/bellhopcuda/build/bin/libbhc2d.so)
```

---

## Examples and Usage

### Example 1: Basic Usage with Defaults

From `examples/defaults.cpp`:

```cpp
#define BHC_DLL_IMPORT 1  // For Windows DLL
#include <bhc/bhc.hpp>
#include <iostream>

// Callbacks for output
void OutputCallback(const char *message) {
    std::cout << "Out: " << message << std::endl;
}

void PrtCallback(const char *message) {
    std::cout << message << std::flush;
}

int main() {
    // Create parameter and output structures
    bhc::bhcParams<false> params;           // false = 2D mode
    bhc::bhcOutputs<false, false> outputs;  // 2D rays
    
    // Setup initialization
    bhc::bhcInit init;
    init.FileRoot = nullptr;  // Use defaults
    init.outputCallback = OutputCallback;
    init.prtCallback = PrtCallback;
    
    // Initialize with defaults
    bhc::setup(init, params, outputs);
    
    // Validate and print configuration
    bhc::echo(params);
    
    // Run simulation
    bhc::run(params, outputs);
    
    // Access results
    std::cout << "Field:\\n";
    for (int iz = 0; iz < params.Pos->NRz; ++iz) {
        for (int ir = 0; ir < params.Pos->NRr; ++ir) {
            // Complex field value at this receiver
            bhc::cpxf value = outputs.uAllSources[iz * params.Pos->NRr + ir];
            std::cout << "(" << value.real() << ", " << value.imag() << ") ";
        }
        std::cout << "\\n";
    }
    
    // Cleanup
    bhc::finalize(params, outputs);
    return 0;
}
```

**Key points**:
- Template parameters specify dimensionality
- Callbacks handle program output
- `setup()` with `FileRoot=nullptr` creates minimal defaults
- Results accessed directly from `outputs` structure

### Example 2: Reading from Environment File

```cpp
#define BHC_DLL_IMPORT 1
#include <bhc/bhc.hpp>

int main() {
    bhc::bhcParams<false> params;
    bhc::bhcOutputs<false, false> outputs;
    
    bhc::bhcInit init;
    init.FileRoot = "test/in/MunkB_ray";  // Without .env extension
    init.numThreads = 8;  // Use 8 threads
    
    if (!bhc::setup(init, params, outputs)) {
        std::cerr << "Setup failed\\n";
        return 1;
    }
    
    if (!bhc::run(params, outputs)) {
        std::cerr << "Run failed\\n";
        return 1;
    }
    
    // Write results
    bhc::writeout(params, outputs, "MunkB_output");
    
    bhc::finalize(params, outputs);
    return 0;
}
```

### Example 3: Modifying Parameters

```cpp
#define BHC_DLL_IMPORT 1
#include <bhc/bhc.hpp>
#include <cstring>

int main() {
    bhc::bhcParams<true> params;           // 3D mode
    bhc::bhcOutputs<true, true> outputs;
    
    bhc::bhcInit init;
    init.FileRoot = nullptr;
    init.outputCallback = [](const char* msg) { std::cout << msg; };
    init.prtCallback = [](const char* msg) { std::cout << msg; };
    
    bhc::setup(init, params, outputs);
    
    // Modify run type to transmission loss
    strcpy(params.Beam->RunType, "CG   3");  // Coherent, Gaussian beams, 3D
    params.Beam->rangeInKm = true;
    params.Beam->Box.x = 20.0;  // 20 km X range
    params.Beam->Box.y = 20.0;  // 20 km Y range
    params.Beam->Box.z = 2000.0;  // 2000 m depth
    
    // Set source position
    bhc::extsetup_sxsy(params, 1, 1);
    bhc::extsetup_sz(params, 1);
    params.Pos->Sx[0] = 10.0;  // X position
    params.Pos->Sy[0] = 10.0;  // Y position
    params.Pos->SxSyInKm = true;
    params.Pos->Sz[0] = 100.0;  // 100 m depth
    
    // Set receivers
    bhc::extsetup_rcvrranges(params, 100);
    params.Pos->RrInKm = true;
    for (int i = 0; i < 100; ++i) {
        params.Pos->Rr[i] = i * 0.2;  // Every 200 m
    }
    
    bhc::extsetup_rcvrdepths(params, 50);
    for (int i = 0; i < 50; ++i) {
        params.Pos->Rz[i] = i * 40.0;  // Every 40 m
    }
    
    bhc::extsetup_rcvrbearings(params, 36);
    for (int i = 0; i < 36; ++i) {
        params.Pos->theta[i] = i * 10.0;  // Every 10 degrees
    }
    
    // Set ray angles
    int nAngles = 100;
    bhc::extsetup_rayelevations(params, nAngles);
    bhc::extsetup_raybearings(params, nAngles);
    params.Angles->alpha.inDegrees = true;
    params.Angles->beta.inDegrees = true;
    
    for (int i = 0; i < nAngles; ++i) {
        params.Angles->alpha.angles[i] = -45.0 + 90.0 * i / (nAngles - 1);
        params.Angles->beta.angles[i] = -15.0 + 30.0 * i / (nAngles - 1);
    }
    
    // Run simulation
    bhc::run(params, outputs);
    bhc::writeout(params, outputs, nullptr);
    
    bhc::finalize(params, outputs);
    return 0;
}
```

### Example 4: Non-Blocking Execution

From `examples/background.cpp`:

```cpp
#define BHC_DLL_IMPORT 1
#include <bhc/bhc.hpp>
#include <atomic>

std::atomic<bool> going;

void CompletedCallback() {
    std::cout << "Computation complete!\\n";
    going = false;
}

int main() {
    bhc::bhcParams<true> params;
    bhc::bhcOutputs<true, true> outputs;
    
    bhc::bhcInit init;
    init.FileRoot = nullptr;
    init.completedCallback = CompletedCallback;
    init.outputCallback = [](const char* msg) { std::cout << msg; };
    init.prtCallback = [](const char* msg) { std::cout << msg; };
    
    bhc::setup(init, params, outputs);
    
    // Request non-blocking execution
    bhc::extsetup_blocking(params, false);
    
    // Setup parameters
    strcpy(params.Beam->RunType, "CG   3");
    // ... set other parameters ...
    
    // Start computation (returns immediately)
    bhc::run(params, outputs);
    
    going = true;
    while (going) {
        int progress = bhc::get_percent_progress(params);
        std::cout << progress << "% complete\\r" << std::flush;
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    
    // Finalize results
    bhc::postprocess(params, outputs);
    
    bhc::writeout(params, outputs, nullptr);
    bhc::finalize(params, outputs);
    return 0;
}
```

**Key points**:
- `extsetup_blocking(params, false)` enables non-blocking mode
- `run()` returns immediately, computation continues in background
- Poll progress with `get_percent_progress()`
- Must call `postprocess()` to finalize when complete

### Example 5: Custom Sound Speed Profile

```cpp
#define BHC_DLL_IMPORT 1
#include <bhc/bhc.hpp>
#include <cmath>

int main() {
    bhc::bhcParams<false> params;
    bhc::bhcOutputs<false, false> outputs;
    
    bhc::bhcInit init;
    init.FileRoot = nullptr;
    init.outputCallback = [](const char* msg) { std::cout << msg; };
    init.prtCallback = [](const char* msg) { std::cout << msg; };
    
    bhc::setup(init, params, outputs);
    
    // Define custom Munk profile
    params.ssp->Type = 'N';  // N²-linear
    params.ssp->NPts = 101;
    
    const double z_axis = 1300.0;  // Sound channel axis depth
    const double epsilon = 0.00737;  // Scale parameter
    
    for (int i = 0; i < params.ssp->NPts; i++) {
        double z = i * 50.0;  // 0 to 5000 m
        params.ssp->z[i] = z;
        
        // Munk profile
        double eta = 2.0 * (z - z_axis) / z_axis;
        double c = 1500.0 * (1.0 + epsilon * (eta - 1.0 + exp(-eta)));
        
        params.ssp->alphaR[i] = c;
        params.ssp->betaR[i] = 0.0;
        params.ssp->rho[i] = 1.0;
        params.ssp->alphaI[i] = 0.0;
        params.ssp->betaI[i] = 0.0;
    }
    
    params.ssp->dirty = true;
    
    // Set boundary depths to match SSP
    params.Bdry->Top.hs.Depth = params.ssp->z[0];
    params.Bdry->Bot.hs.Depth = params.ssp->z[params.ssp->NPts - 1];
    
    // Run with custom profile
    strcpy(params.Beam->RunType, "RG   2");
    bhc::run(params, outputs);
    
    bhc::writeout(params, outputs, "custom_munk");
    bhc::finalize(params, outputs);
    return 0;
}
```

### Example 6: Range-Dependent SSP (Quad Mode)

```cpp
#define BHC_DLL_IMPORT 1
#include <bhc/bhc.hpp>

int main() {
    bhc::bhcParams<false> params;
    bhc::bhcOutputs<false, false> outputs;
    
    bhc::bhcInit init;
    init.FileRoot = nullptr;
    // ... set callbacks ...
    
    bhc::setup(init, params, outputs);
    
    // Setup quadrilateral SSP (range-dependent)
    int NPts = 51;   // Depth points
    int Nr = 21;     // Range points
    
    bhc::extsetup_ssp_quad(params, NPts, Nr);
    
    // Fill depth values
    for (int iz = 0; iz < NPts; iz++) {
        params.ssp->z[iz] = iz * 100.0;  // 0 to 5000 m
    }
    
    // Fill range values
    params.ssp->rangeInKm = true;
    for (int ir = 0; ir < Nr; ir++) {
        params.ssp->Seg.r[ir] = ir * 1.0;  // 0 to 20 km
    }
    
    // Fill sound speed grid
    for (int iz = 0; iz < NPts; iz++) {
        for (int ir = 0; ir < Nr; ir++) {
            double z = params.ssp->z[iz];
            double r = params.ssp->Seg.r[ir] * 1000.0;  // Convert to meters
            
            // Example: Sound speed varies with range and depth
            double c = 1500.0 + 0.01 * z - 0.0001 * r;
            
            params.ssp->cMat[iz * Nr + ir] = c;
        }
    }
    
    params.ssp->dirty = true;
    params.Bdry->Top.hs.Depth = params.ssp->z[0];
    params.Bdry->Bot.hs.Depth = params.ssp->z[NPts - 1];
    
    bhc::run(params, outputs);
    bhc::finalize(params, outputs);
    return 0;
}
```

---

## Advanced Topics

### Memory Management

#### Memory Layout

**Parameters** (`bhcParams`): Allocated once, reused across multiple runs
- Small metadata in main struct
- Large arrays (SSP, positions, etc.) allocated separately
- Manual allocation via `extsetup_*` functions

**Outputs** (`bhcOutputs`): Allocated/reallocated as needed
- Ray data: Can be very large, managed by copy mode setting
- TL field: Size depends on receiver count
- Arrivals: Dynamic size, managed by `MaxNArr` parameter

#### Copy Mode vs. Non-Copy Mode (Ray Runs)

**Non-copy mode** (default):
- All rays stored in single contiguous array
- Fast access
- Limited by available memory
- May reduce `MaxPointsPerRay` to fit

**Copy mode** (`useRayCopyMode = true`):
- Rays stored in temporary buffer, copied when complete
- Slower but uses less memory
- Can handle longer rays
- Useful for large ray counts

#### GPU Memory

CUDA version manages device memory automatically:
- Parameters copied to GPU at start
- Results copied back at end
- Intermediate data stays on GPU
- Monitor with `nvidia-smi` for memory usage

### Numerical Precision

#### Double vs. Single Precision

**Double precision** (default):
- Matches original BELLHOP
- Required for accurate results in most cases
- Some rays numerically unstable in single precision

**Single precision** (`BHC_USE_FLOATS=ON`):
- ~2x faster on CPU
- Much faster on consumer GPUs (64x faster FP32 vs FP64)
- Use with caution - validate results
- Suitable for initial exploration, visualization

#### Numerical Stability Improvements

This version includes fixes over original BELLHOP:
- Consistent boundary stepping
- Improved edge case handling
- Better floating-point ordering
- Reduced sensitivity to compiler optimizations

Details in `doc/accuracy.md` and [modified BELLHOP repo](https://github.com/A-New-BellHope/bellhop).

### Thread Safety

#### Multiple Instances

Can run multiple independent simulations in parallel:

```cpp
// Thread 1
bhc::bhcParams<false> params1;
bhc::bhcOutputs<false, false> outputs1;
// ... setup params1 ...
bhc::run(params1, outputs1);

// Thread 2 (simultaneously)
bhc::bhcParams<false> params2;
bhc::bhcOutputs<false, false> outputs2;
// ... setup params2 ...
bhc::run(params2, outputs2);
```

**Requirements**:
- Each thread uses separate `params` and `outputs`
- Callbacks must be thread-safe if shared
- Different `FileRoot` if using PRTFile output

#### Callback Thread Safety

If using same callback across multiple instances:

```cpp
std::mutex output_mutex;

void ThreadSafeOutput(const char* message) {
    std::lock_guard<std::mutex> lock(output_mutex);
    std::cout << message << std::flush;
}
```

### Performance Tuning

#### Optimal Thread Count

CPU version:
```cpp
init.numThreads = -1;  // Auto-detect (all logical cores)
init.numThreads = 8;   // Manual (e.g., for shared machine)
```

More threads ≈ more speedup, up to logical core count. Diminishing returns beyond.

#### GPU Selection

Multiple GPUs:
```cpp
init.gpuIndex = 0;  // Usually fastest GPU
init.gpuIndex = 1;  // Second GPU
```

Use `nvidia-smi` to see available GPUs and their capabilities.

#### Ray Count

For GPU, more rays = better utilization:
- Consumer GPU: 10,000+ rays for good speedup
- Server GPU: 100,000+ rays for best performance
- CPU: Even small ray counts benefit from multithreading

#### Receiver Layout

For best performance:
- Concentrate receivers in areas of interest
- Avoid spreading receivers everywhere
- Especially important for arrivals/eigenrays

See `doc/performance.md` for detailed benchmarks.

### Customization and Extension

#### Adding New SSP Types

1. Define new type character in `SSPStructure`
2. Implement interpolation in `ssp.hpp`
3. Add reading/writing in `module/ssp.cpp`
4. Enable with CMake option

#### Adding New Influence Functions

1. Implement in `influence.hpp`
2. Add selection logic in `field.cpp`
3. Test thoroughly for numerical stability
4. Enable with CMake option

#### Modifying Ray Equations

Core ray stepping in `step.hpp`:
- `Step()`: Single step advance
- `EvaluateSSP()`: SSP at current position
- `ApplyBoundary()`: Boundary interaction

Modify with extreme caution - small changes can have large effects.

---

## Testing and Validation

### Test Suite Organization

#### Coverage Tests

Auto-generated tests for all feature combinations:

```bash
# Generate test environments
python gen_tests.py

# Run all tests
./run_tests.sh
```

Tests every combination of:
- Dimensionality (2D/3D/Nx2D)
- Run type (ray/TL/eigenray/arrivals)
- SSP type
- Beam type
- Boundary conditions

~1500 test cases total.

#### OALIB Tests

Standard test cases from Acoustics Toolbox:

```bash
# Located in test/in/
# Run individual test:
./bellhopcxx --2D test/in/MunkB_ray
```

Compare results visually with MATLAB plotting scripts.

#### Validation Against Original BELLHOP

Comparison scripts:

```bash
# Compare arrivals
python compare_arrivals.py test/in/MunkB_arr

# Compare rays
python compare_ray.py test/in/MunkB_ray

# Compare transmission loss
python compare_shdfil.py test/in/MunkB_coherent
```

These compare:
- Floating-point values within tolerance
- Ray paths and arrivals
- Transmission loss fields

### Running Tests

#### Individual Test

```bash
./bellhopcxx --2D test/in/TestCase
# Produces TestCase.prt, TestCase.shd (or .ray, .arr)
```

#### Batch Testing

```bash
# Run all generated tests
./run_all_gen.sh

# Find failing tests
./find_tests.sh
```

#### Comparing to Reference

```bash
# Requires reference BELLHOP output
# Run original BELLHOP first to generate .shd/.arr/.ray
# Then run comparison script
python compare_shdfil.py test/in/TestCase
```

### Common Issues and Debugging

#### Setup Failures

**Problem**: `setup()` returns false

**Solutions**:
- Check FileRoot path is correct (without .env extension)
- Verify .env file is well-formed
- Check associated files (.ssp, .bty, etc.) exist
- Review PRTFile or callback output for specific error

#### Run Failures

**Problem**: `run()` returns false

**Solutions**:
- Validate parameters with `echo()` first
- Check ray/receiver positions are within bounds
- Verify SSP and boundary depths are consistent
- Review error messages in PRTFile

#### GPU Errors

**Problem**: CUDA errors during execution

**Solutions**:
- Verify GPU memory not exhausted (`nvidia-smi`)
- Try reducing ray count or receiver count
- Check CUDA drivers are up to date
- Try different GPU with `gpuIndex`

#### Accuracy Issues

**Problem**: Results don't match BELLHOP

**Solutions**:
- Verify environment file is identical
- Check for numerical precision issues (use double, not float)
- Compare to modified BELLHOP (see repo), not original
- Some divergence expected for complex environments
- Review `doc/accuracy.md` for known limitations

#### Performance Issues

**Problem**: Slower than expected

**Solutions**:
- Ensure Release build, not Debug
- Check thread count (CPU)
- Verify GPU is being used (CUDA)
- Profile with timing tools
- Review receiver layout (fewer/concentrated is faster)

### Reporting Issues

When reporting bugs:

1. **Minimal test case**: Smallest environment file that reproduces issue
2. **Version info**: Git commit hash, build options
3. **Platform**: OS, compiler version, GPU model (if CUDA)
4. **Error messages**: Full output from PRTFile or callbacks
5. **Expected vs. actual**: What you expected to happen

Submit to: [GitHub Issues](https://github.com/A-New-BellHope/bellhopcuda/issues)

---

## Additional Resources

### Documentation Files

- `README.md`: Quick start and overview
- `doc/compilation.md`: Detailed build instructions
- `doc/accuracy.md`: Accuracy validation and comparison
- `doc/performance.md`: Performance benchmarks and tuning
- `doc/faq.md`: Frequently asked questions

### External Resources

- **Original BELLHOP**: [Acoustics Toolbox](http://oalib.hlsresearch.com/AcousticsToolbox/)
- **Modified BELLHOP**: [GitHub](https://github.com/A-New-BellHope/bellhop)
- **Project Repository**: [GitHub](https://github.com/A-New-BellHope/bellhopcuda)
- **GLM Library**: [GitHub](https://github.com/g-truc/glm)
- **CUDA Toolkit**: [NVIDIA](https://developer.nvidia.com/cuda-toolkit)

### Academic References

For understanding underwater acoustics and ray theory:
- Porter, M.B. "The BELLHOP Manual and User's Guide"
- Jensen, F.B., et al. "Computational Ocean Acoustics"
- Etter, P.C. "Underwater Acoustic Modeling and Simulation"

### Community and Support

- GitHub Discussions: For questions and community interaction
- GitHub Issues: For bug reports and feature requests
- Email: Marine Physical Lab at jjaffe@ucsd.edu

---

**Last Updated**: January 2026  
**Version**: 1.5.0+  
**Authors**: Marine Physical Lab at Scripps Oceanography, University of California

