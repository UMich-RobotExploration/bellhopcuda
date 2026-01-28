# BellhopCUDA/BellhopCXX Quick Reference Guide

## Quick Start (5 Minutes)

### Using Existing Environment Files

```bash
# Build (CPU only)
git clone https://github.com/A-New-BellHope/bellhopcuda.git
cd bellhopcuda
git submodule update --init --recursive
mkdir build && cd build
cmake .. -DBHC_ENABLE_CUDA=OFF
cmake --build . --config Release

# Run a test
cd bin
./bellhopcxx --2D ../../test/in/MunkB_ray
```

### Basic Library Usage

```cpp
#define BHC_DLL_IMPORT 1  // For Windows DLL
#include <bhc/bhc.hpp>

int main() {
    // Setup
    bhc::bhcParams<false> params;           // false = 2D
    bhc::bhcOutputs<false, false> outputs;
    bhc::bhcInit init;
    init.FileRoot = "path/to/environment";  // Without .env
    bhc::setup(init, params, outputs);
    
    // Run
    bhc::run(params, outputs);
    
    // Cleanup
    bhc::finalize(params, outputs);
}
```

## Common Tasks

### Switch to GPU Mode

```bash
cmake .. -DBHC_ENABLE_CUDA=ON  # Default if CUDA available
cmake --build . --config Release
./bellhopcuda --3D TestCase
```

### Modify Parameters Before Running

```cpp
// After setup(), before run()
strcpy(params.Beam->RunType, "CG   3");  // Coherent TL, Gaussian, 3D
params.Beam->Box.z = 5000.0;             // Depth range in meters

// Change source depth
bhc::extsetup_sz(params, 1);
params.Pos->Sz[0] = 200.0;  // 200m depth

// Change receiver grid
bhc::extsetup_rcvrranges(params, 100);
params.Pos->RrInKm = true;
for(int i = 0; i < 100; i++) {
    params.Pos->Rr[i] = i * 0.1;  // Every 100m to 10km
}
```

### Access Results

```cpp
// Transmission Loss Field
int nr = params.Pos->NRr;
int nz = params.Pos->NRz;
for(int iz = 0; iz < nz; iz++) {
    for(int ir = 0; ir < nr; ir++) {
        bhc::cpxf value = outputs.uAllSources[iz * nr + ir];
        float tl_db = -20.0f * log10f(abs(value));
    }
}

// Ray Coordinates
for(int iray = 0; iray < outputs.rayinfo->NRays; iray++) {
    auto &ray = outputs.rayinfo->results[iray];
    for(int istep = 0; istep < ray.Nsteps; istep++) {
        auto &pt = ray.ray[istep];
        bhc::VEC23<false> pos = pt.x;  // vec2 in 2D
        // pos.x = range, pos.y = depth
    }
}
```

## Template Parameters Quick Reference

```cpp
// O3D: Ocean is 3D
// R3D: Rays are 3D

bhcParams<false>         // 2D ocean
bhcParams<true>          // 3D or Nx2D ocean

bhcOutputs<false, false> // 2D mode (2D ocean, 2D rays)
bhcOutputs<true, false>  // Nx2D mode (3D ocean, 2D rays)
bhcOutputs<true, true>   // 3D mode (3D ocean, 3D rays)
```

## RunType Quick Reference

### Format: `"XY   Z"`
- **X** (Run type):
  - `'R'`: Ray tracing
  - `'C'`: Coherent TL
  - `'S'`: Semi-coherent TL
  - `'I'`: Incoherent TL
  - `'E'`: Eigenrays
  - `'A'/'a'`: Arrivals

- **Y** (Beam type):
  - `'G'`: Geometric hat beam
  - `'g'`: Geometric Gaussian beam
  - `'B'`: Geometric hat beam (Cartesian)
  - `'b'`: Geometric Gaussian beam (Cartesian)
  - `'R'`: Cerveny ray-centered beam
  - `'C'`: Cerveny Cartesian beam
  - `'S'`: Simple Gaussian beam

- **Z** (Dimensionality):
  - `'2'`: 2D
  - `'3'`: 3D
  - `'N'`: Nx2D

Examples:
```cpp
strcpy(params.Beam->RunType, "RG   2");  // 2D ray tracing, geometric
strcpy(params.Beam->RunType, "CR   3");  // 3D coherent TL, Cerveny ray-centered
strcpy(params.Beam->RunType, "ES   N");  // Nx2D eigenrays, simple Gaussian
```

## SSP Types

```cpp
params.ssp->Type = 'N';  // N²-linear (most common)
params.ssp->Type = 'C';  // C-linear
params.ssp->Type = 'S';  // Cubic spline
params.ssp->Type = 'P';  // PCHIP
params.ssp->Type = 'Q';  // Quadrilateral (2D range-dependent)
params.ssp->Type = 'H';  // Hexahedral (3D)
params.ssp->Type = 'A';  // Analytic
```

## Boundary Types

```cpp
// Top/bottom boundary type
params.Bdry->Top.hs.bc = 'V';  // Vacuum (pressure release)
params.Bdry->Bot.hs.bc = 'R';  // Rigid
params.Bdry->Bot.hs.bc = 'A';  // Acousto-elastic
```

## Common Errors and Fixes

### "Setup failed"
- Check FileRoot path is correct (without .env extension)
- Verify .env file and associated files (.ssp, .bty) exist
- Check file is well-formed

### "CUDA out of memory"
- Reduce ray count
- Reduce receiver count
- Try smaller environment

### "Results don't match BELLHOP"
- Use double precision (BHC_USE_FLOATS=OFF)
- Compare to [modified BELLHOP](https://github.com/A-New-BellHope/bellhop), not original
- Some divergence expected for complex environments

### "Slow performance"
- Ensure Release build: `cmake --build . --config Release`
- Check thread count: `init.numThreads = -1;` (all cores)
- For GPU: Verify CUDA is being used
- Reduce receiver density

## File Formats

### Input Files
- **`.env`**: Environment parameters
- **`.ssp`**: Sound speed profile (if Type='Q' or 'H')
- **`.bty`**: Bathymetry (if spatially varying)
- **`.ati`**: Altimetry (if spatially varying)
- **`.brc`**: Bottom reflection coefficients (if bc='F')
- **`.trc`**: Top reflection coefficients (if bc='F')
- **`.sbp`**: Source beam pattern (if SBPFlag='*')

### Output Files
- **`.prt`**: Print file (human-readable summary)
- **`.shd`**: Shade file (TL field data) - binary
- **`.ray`**: Ray coordinates - binary
- **`.arr`**: Arrivals - binary

## Build Options Reference

```bash
# Core options
-DBHC_ENABLE_CUDA=ON/OFF      # Enable CUDA (default ON if available)
-DBHC_BUILD_EXAMPLES=ON/OFF   # Build examples (default ON)
-DBHC_USE_FLOATS=ON/OFF       # 32-bit floats (default OFF - use 64-bit)
-DBHC_DEBUG=ON/OFF            # Debug mode (default OFF)

# Dimensionality
-DBHC_DIM_ENABLE_2D=ON/OFF    # Enable 2D (default ON)
-DBHC_DIM_ENABLE_3D=ON/OFF    # Enable 3D (default ON)
-DBHC_DIM_ENABLE_NX2D=ON/OFF  # Enable Nx2D (default ON)

# Feature limiting (for compatibility testing)
-DBHC_LIMIT_FEATURES=ON/OFF   # Limit to BELLHOP features (default OFF)
```

## API Function Reference

### Initialization
```cpp
bool setup(const bhcInit &init, bhcParams<O3D> &params, 
           bhcOutputs<O3D, R3D> &outputs);
```

### Parameter Modification
```cpp
void extsetup_sxsy(bhcParams<O3D> &params, int32_t NSx, int32_t NSy);
void extsetup_sz(bhcParams<O3D> &params, int32_t NSz);
void extsetup_rcvrranges(bhcParams<O3D> &params, int32_t NRr);
void extsetup_rcvrdepths(bhcParams<O3D> &params, int32_t NRz);
void extsetup_rcvrbearings(bhcParams<O3D> &params, int32_t Ntheta);
void extsetup_rayelevations(bhcParams<O3D> &params, int32_t n);
void extsetup_raybearings(bhcParams<O3D> &params, int32_t n);
void extsetup_ssp_quad(bhcParams<false> &params, int32_t NPts, int32_t Nr);
void extsetup_ssp_hexahedral(bhcParams<true> &params, int32_t Nx, int32_t Ny, int32_t Nz);
void extsetup_altimetry(bhcParams<O3D> &params, const IORI2<O3D> &NPts);
void extsetup_bathymetry(bhcParams<O3D> &params, const IORI2<O3D> &NPts, int32_t NBotProvinces);
void extsetup_blocking(bhcParams<O3D> &params, bool blocking);
```

### Execution
```cpp
bool echo(bhcParams<O3D> &params);
bool run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);
bool postprocess(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);
int get_percent_progress(bhcParams<O3D> &params);
```

### Output
```cpp
bool writeout(const bhcParams<O3D> &params, const bhcOutputs<O3D, R3D> &outputs, const char *FileRoot);
bool readout(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs, const char *FileRoot);
bool writeenv(bhcParams<O3D> &params, const char *FileRoot);
```

### Cleanup
```cpp
void finalize(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);
```

### Utilities
```cpp
bool get_ssp(bhcParams<O3D> &params, const VEC23<R3D> &x, float &sound_speed);
```

## Performance Tips

### CPU Optimization
- Use all cores: `init.numThreads = -1;`
- Release build: Essential for performance
- Many rays (100+): Better parallelization

### GPU Optimization
- Many rays (10,000+): Saturate GPU
- Minimize receivers: Less memory contention
- Server GPU: Better for many receivers
- Check memory: `nvidia-smi`

### General
- Reduce receiver density in areas not of interest
- TL runs faster than arrivals/eigenrays
- File I/O can dominate - use library mode to avoid

## Coordinate Systems

### 2D
- X-axis: Range (horizontal distance from source)
- Y-axis: Depth (positive downward)
- Source at (0, Sz)
- Receivers at (Rr, Rz)

### 3D
- X-axis: East
- Y-axis: North
- Z-axis: Depth (positive downward)
- Source at (Sx, Sy, Sz)
- Receivers at polar (Rr, theta) or Cartesian from source

### Units
- Distances: Meters (after preprocessing)
- Angles: Radians (after preprocessing)
- Sound speed: m/s
- Frequency: Hz
- Time: seconds

## Key Differences from Original BELLHOP

### Improvements
- Parallel execution (CPU multithreading, GPU)
- Library API (no file I/O required)
- Better numerical stability
- More feature combinations supported

### Compatibility
- Reads same .env files
- Produces same output formats
- Drop-in replacement for executables
- Some results differ slightly due to numerical stability fixes

### Breaking Changes (v1.5.0+)
- Binary output format updated to match BELLHOP 2024
- CMake 3.27+ required
- Some API changes for extsetup functions

## Getting Help

- **Documentation**: `DOCUMENTATION.md`, `DEVELOPER_GUIDE.md`
- **FAQs**: `doc/faq.md`
- **Examples**: `examples/` directory
- **GitHub Issues**: Bug reports and feature requests
- **Email**: jjaffe@ucsd.edu (Marine Physical Lab)

## Links

- **Repository**: https://github.com/A-New-BellHope/bellhopcuda
- **Modified BELLHOP**: https://github.com/A-New-BellHope/bellhop
- **Original BELLHOP**: http://oalib.hlsresearch.com/AcousticsToolbox/
- **Releases**: https://github.com/A-New-BellHope/bellhopcuda/releases
