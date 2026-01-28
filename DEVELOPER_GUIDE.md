# BellhopCUDA/BellhopCXX Developer Guide

## Table of Contents

1. [Code Architecture](#code-architecture)
2. [Template System Explained](#template-system-explained)
3. [Data Structures Deep Dive](#data-structures-deep-dive)
4. [Module System](#module-system)
5. [Mode System](#mode-system)
6. [Ray Tracing Engine](#ray-tracing-engine)
7. [Boundary Handling](#boundary-handling)
8. [SSP Interpolation](#ssp-interpolation)
9. [Influence Functions](#influence-functions)
10. [Memory Management](#memory-management)
11. [CUDA Implementation](#cuda-implementation)
12. [Contributing Guidelines](#contributing-guidelines)

---

## Code Architecture

### High-Level Design

BellhopCUDA/CXX uses a layered architecture:

```
┌───────────────────────────────────────────────────────┐
│                   Public API Layer                     │
│  (include/bhc/bhc.hpp, include/bhc/structs.hpp)       │
└────────────────────┬──────────────────────────────────┘
                     │
┌────────────────────┴──────────────────────────────────┐
│              API Implementation Layer                  │
│           (src/api.cpp, src/common_*.hpp)             │
└────────────────────┬──────────────────────────────────┘
                     │
         ┌───────────┴──────────┬────────────────────────┐
         │                      │                        │
┌────────┴────────┐  ┌─────────┴────────┐  ┌───────────┴─────────┐
│  Module System  │  │   Mode System    │  │  Utility Functions  │
│  (src/module/)  │  │  (src/mode/)     │  │    (src/util/)      │
│                 │  │                  │  │                     │
│ - Parameters    │  │ - Ray tracing    │  │ - Error handling    │
│ - I/O           │  │ - TL computation │  │ - Timing            │
│ - Validation    │  │ - Eigenrays      │  │ - File I/O          │
│                 │  │ - Arrivals       │  │ - Math utilities    │
└─────────────────┘  └──────────────────┘  └─────────────────────┘
         │                      │                        │
         └───────────┬──────────┴────────────────────────┘
                     │
┌────────────────────┴──────────────────────────────────┐
│                 Core Engine Layer                      │
│  (src/trace.hpp, src/step.hpp, src/ssp.hpp, etc.)    │
│                                                        │
│  - Ray stepping                                        │
│  - SSP evaluation                                      │
│  - Boundary detection                                  │
│  - Influence calculation                               │
└────────────────────────────────────────────────────────┘
```

### File Organization

**Public Headers** (`include/bhc/`):
- `bhc.hpp`: All public API function declarations
- `structs.hpp`: All public data structure definitions
- `math.hpp`: Mathematical types and utilities
- `platform.hpp`: Platform-specific macros and definitions

**Implementation** (`src/`):
- `api.cpp`: Implementation of public API functions
- `common.hpp`: Common includes, macros, internal types
- `common_setup.hpp`: Setup-related internal utilities
- `common_run.hpp`: Run-related internal utilities
- `cmdline.cpp`: Command-line executable entry point

**Modules** (`src/module/`):
Each module handles one aspect of parameter setup:
- `paramsmodule.hpp`: Base class for all modules
- `atten.cpp/hpp`: Attenuation (volume absorption)
- `ssp.cpp/hpp`: Sound speed profile
- `boundary.cpp/hpp`: Altimetry and bathymetry
- `szrz.cpp/hpp`: Source and receiver depths
- ... and many more

**Modes** (`src/mode/`):
Each mode implements a run type:
- `modemodule.hpp`: Base class for all modes
- `field.cpp/hpp`: Base class for field-computing modes
- `ray.cpp/hpp`: Ray tracing mode
- `tl.cpp/hpp`: Transmission loss mode
- `eigen.cpp/hpp`: Eigenray mode
- `arr.cpp/hpp`: Arrivals mode

**Core Components** (src/):
- `trace.hpp`: Main ray tracing loop
- `step.hpp`: Single step ray propagation
- `ssp.hpp`: SSP interpolation and evaluation
- `boundary.hpp`: Boundary detection and handling
- `reflect.hpp`: Reflection/refraction calculations
- `influence.hpp`: Acoustic influence/contribution to receivers
- `curves.hpp`: Curve-related utilities
- `eigenrays.hpp`: Eigenray-specific logic
- `arrivals.hpp`: Arrivals-specific logic

**Utilities** (`src/util/`):
- `errors.cpp/hpp`: Error handling and reporting
- `timing.cpp/hpp`: Performance timing
- `directio.hpp`: Direct (unformatted) file I/O
- `ldio.hpp`: List-directed (formatted) file I/O
- `UtilsCUDA.cuh`: CUDA-specific utilities

### Compilation Flow

1. **CMake Configuration**: Reads CMakeLists.txt and options, generates build files
2. **Template Instantiation**: Templates instantiated for enabled dimensionalities
3. **Separate Compilation**: CPU (.cpp) and GPU (.cu) code compiled separately
4. **Linking**: Object files linked into libraries and executables

For CUDA:
- Template files (`.cpp.in`, `.cu.in`) processed to generate `.cpp` and `.cu`
- NVCC compiles `.cu` files to device code
- Host compiler links everything together

---

## Template System Explained

### The O3D and R3D Parameters

The entire codebase is templated on two boolean parameters:

**`O3D` - Ocean is 3D**:
- `false`: Ocean environment is 2D (range-depth)
- `true`: Ocean environment is 3D (x-y-depth)

**`R3D` - Rays are 3D**:
- `false`: Rays propagate in 2D (vertical plane)
- `true`: Rays propagate in 3D (full 3D space)

**Valid combinations**:
```cpp
<false, false> // 2D:   2D ocean, 2D rays
<true, false>  // Nx2D: 3D ocean, 2D rays (multiple vertical planes)
<true, true>   // 3D:   3D ocean, 3D rays
```

### Type-Dependent Types

Using template specialization for dimension-dependent types:

```cpp
// VEC23: Vector type (vec2 or vec3)
template<bool X3D> using VEC23 = typename TmplVec23<X3D>::type;
// Usage:
VEC23<false> pos2d;  // vec2
VEC23<true> pos3d;   // vec3

// IORI2: Integer type (int32_t or int2)
template<bool X3D> using IORI2 = typename TmplInt12<X3D>::type;
// Usage:
IORI2<false> count2d;  // int32_t
IORI2<true> count3d;   // int2

// V2M2: Vector or matrix (vec2 or mat2x2)
template<bool X3D> using V2M2 = typename TmplVec2Mat2<X3D>::type;
// Usage:
V2M2<false> v;  // vec2 in 2D
V2M2<true> m;   // mat2x2 in 3D
```

### Structure Composition Pattern

Many structures use template specialization for dimension-specific members:

```cpp
// Base template with common members
template<bool X3D> struct MyStruct : public MyStructExtras<X3D> {
    int commonMember;
    VEC23<X3D> position;
};

// Empty in one dimension
template<> struct MyStructExtras<false> {};

// Additional members in other dimension
template<> struct MyStructExtras<true> {
    int extraMember3D;
};
```

Example in real code:

```cpp
template<bool R3D> struct rayPt : public rayPtExtras<R3D> {
    int32_t NumTopBnc, NumBotBnc;
    VEC23<R3D> x;    // vec2 in 2D, vec3 in 3D
    VEC23<R3D> t;    // Tangent vector
    real c;          // Sound speed
    real Amp, Phase;
    cpx tau;
};

template<> struct rayPtExtras<false> {
    vec2 p, q;  // 2D beam parameters
};

template<> struct rayPtExtras<true> {
    mat2x2 p, q;  // 3D beam parameters (matrices)
    real phi;     // Additional 3D parameter
    real dummy;   // Alignment padding
};
```

### Template Function Pattern

Functions templated on dimensionality:

```cpp
// Declaration in header
template<bool O3D, bool R3D> 
bool run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs);

// Explicit instantiation in source
template bool run<false, false>(...);  // 2D
template bool run<true, false>(...);   // Nx2D
template bool run<true, true>(...);    // 3D
```

Compile-time branching:

```cpp
template<bool O3D, bool R3D>
void ProcessRay(...) {
    // Common code for all dimensions
    
    if constexpr(R3D) {
        // Only in 3D ray mode
        // Compiler completely removes this branch in 2D/Nx2D builds
    } else {
        // Only in 2D/Nx2D ray mode
    }
    
    if constexpr(O3D && !R3D) {
        // Only in Nx2D mode
    }
}
```

### Why Templates?

**Advantages**:
1. **Type safety**: Compile-time errors for dimension mismatches
2. **Performance**: No runtime overhead for dimension checks
3. **Code reuse**: Single implementation for all dimensions
4. **Clarity**: Explicit about which code is dimension-specific

**Challenges**:
1. **Compile time**: Must instantiate all combinations
2. **Binary size**: Separate code for each instantiation
3. **Complexity**: Template metaprogramming can be hard to read

---

## Data Structures Deep Dive

### bhcParams Structure

The main parameter container:

```cpp
template<bool O3D> struct bhcParams {
    char Title[80];              // Simulation title
    real fT;                     // Transmission loss reference value
    BdryType *Bdry;              // Current boundary segment properties
    BdryInfo<O3D> *bdinfo;       // All boundary data
    ReflectionInfo *refl;        // Reflection coefficients
    SSPStructure *ssp;           // Sound speed profile
    AttenInfo *atten;            // Volume attenuation
    Position *Pos;               // Source/receiver positions
    AnglesStructure *Angles;     // Ray launch angles
    FreqInfo *freqinfo;          // Frequency information
    BeamStructure<O3D> *Beam;    // Beam/ray configuration
    SBPInfo *sbp;                // Source beam pattern
    void *internal;              // Internal state (opaque)
};
```

**Ownership**: 
- Struct owns all pointer members
- Allocated in `setup()`, freed in `finalize()`
- User should not allocate/free these manually

**Accessing**:
```cpp
// Read
int nRays = params.Beam->NBeams;
real depth = params.Pos->Sz[0];

// Write
params.Beam->deltas = 0.5;
params.Pos->Sz[0] = 100.0;

// Reallocate
extsetup_sz(params, 10);
for (int i = 0; i < 10; i++) {
    params.Pos->Sz[i] = i * 50.0;
}
```

### bhcOutputs Structure

Results container:

```cpp
template<bool O3D, bool R3D> struct bhcOutputs {
    RayInfo<O3D, R3D> *rayinfo;  // Ray trajectories (ray/eigenray runs)
    cpxf *uAllSources;           // Acoustic field (TL runs)
    EigenInfo *eigen;            // Eigenray hit information
    ArrInfo *arrinfo;            // Arrival data (arrivals runs)
};
```

**What's populated**:
- **Ray runs**: `rayinfo` contains all ray trajectories
- **TL runs**: `uAllSources` contains complex field values
- **Eigenray runs**: `rayinfo` (rays), `eigen` (which rays hit which receivers)
- **Arrivals runs**: `arrinfo` contains detailed arrival structure

**Accessing results**:

```cpp
// TL field
int nr = params.Pos->NRr;
int nz = params.Pos->NRz;
for (int iz = 0; iz < nz; iz++) {
    for (int ir = 0; ir < nr; ir++) {
        cpxf value = outputs.uAllSources[iz * nr + ir];
        float magnitude = abs(value);
        float phase_rad = arg(value);
    }
}

// Rays
int nRays = outputs.rayinfo->NRays;
for (int iray = 0; iray < nRays; iray++) {
    RayResult<O3D, R3D> &ray = outputs.rayinfo->results[iray];
    int nSteps = ray.Nsteps;
    for (int istep = 0; istep < nSteps; istep++) {
        rayPt<R3D> &pt = ray.ray[istep];
        VEC23<R3D> pos = pt.x;  // Position
        real amplitude = pt.Amp;
        // ...
    }
}

// Arrivals
int ir = 0, iz = 0, itheta = 0;  // Receiver indices
int iRecv = (itheta * nRr + ir) * nRz + iz;
int nArr = outputs.arrinfo->NArr[iRecv];
for (int iarr = 0; iarr < nArr; iarr++) {
    Arrival &arr = outputs.arrinfo->Arr[/* complex indexing */];
    float delay_sec = arr.delay.real();
    float amplitude = arr.a;
    int topBounces = arr.NTopBnc;
    // ...
}
```

### SSPStructure

Sound speed profile data:

```cpp
struct SSPStructure {
    // Profile data (1D profiles)
    cpx c[MaxSSP], cz[MaxSSP];           // Sound speed and derivative
    cpx n2[MaxSSP], n2z[MaxSSP];         // n² and derivative (N²-linear)
    cpx cSpline[4][MaxSSP];              // Cubic spline coefficients
    cpx cCoef[4][MaxSSP], CSWork[4][MaxSSP];  // PCHIP coefficients
    
    // Profile data (multi-D profiles)
    real *cMat, *czMat;                  // Sound speed grid
    
    // Grid coordinates
    rxyz_vector Seg;                     // .r, .x, .y, .z arrays
    real z[MaxSSP];                      // Depth points (1D profiles)
    real rho[MaxSSP];                    // Density
    real alphaR[MaxSSP], alphaI[MaxSSP]; // P-wave speed and attenuation
    real betaR[MaxSSP], betaI[MaxSSP];   // S-wave speed and attenuation
    
    // Metadata
    int32_t NPts, Nr, Nx, Ny, Nz;        // Dimensions
    char Type;                            // 'N', 'C', 'S', 'P', 'Q', 'H', 'A'
    char AttenUnit[2];                    // Attenuation units
    bool rangeInKm;                       // Coordinate units flag
    bool dirty;                           // Needs preprocessing flag
};
```

**Type meanings**:
- `'N'`: N²-linear (uses `n2`, `n2z`)
- `'C'`: C-linear (uses `c`, `cz`)
- `'S'`: Cubic spline (uses `cSpline`)
- `'P'`: PCHIP (uses `cCoef`)
- `'Q'`: Quadrilateral 2D (uses `cMat`, indexed by `[iz * Nr + ir]`)
- `'H'`: Hexahedral 3D (uses `cMat`, indexed by `[(ix * Ny + iy) * Nz + iz]`)
- `'A'`: Analytic (computed on-the-fly)

**Dirty flag**:
- Set to `true` whenever SSP data changes
- Causes preprocessing (computing derivatives, coefficients) on next run
- Automatically cleared after preprocessing

### BdryInfo and Related

Boundary information (altimetry/bathymetry):

```cpp
template<bool O3D> struct BdryInfo {
    BdryInfoTopBot<O3D> top, bot;
};

template<bool O3D> struct BdryInfoTopBot {
    IORI2<O3D> NPts;            // Number of boundary points
    char type[2];                // Boundary type and options
    bool dirty;                  // Needs preprocessing
    bool rangeInKm;              // Coordinate units
    BdryPtFull<O3D> *bd;        // Boundary point array
    int32_t NBotProvinces;       // Number of bottom provinces (3D)
    HSInfo *BotProv;             // Province properties (3D)
};

template<bool O3D> struct BdryPtFull : public BdryPtFullExtras<O3D>,
                                       public ReflCurvature<O3D> {
    VEC23<O3D> x;     // Boundary coordinate
    VEC23<O3D> t;     // Tangent vector
    VEC23<O3D> n;     // Normal vector (outward)
    VEC23<O3D> Noden; // Node normal (curvilinear)
    real Len;         // Segment/facet length
};

// 2D-specific extras
template<> struct BdryPtFullExtras<false> {
    vec2 Nodet;                  // Tangent at node
    real Dx, Dxx, Dss;          // Derivatives
    HSInfo hs;                   // Acoustic properties
};

// 3D-specific extras
template<> struct BdryPtFullExtras<true> {
    vec3 n1, n2;                 // Triangle pair normals
    vec3 Noden_unscaled;         // Unscaled node normal
    real phi_xx, phi_xy, phi_yy; // Second derivatives
    int32_t Province;            // Province index
};
```

**2D boundary**:
- Array of NPts points along range
- `bd[i].x = (range, depth)` for each point

**3D boundary**:
- 2D grid of NPts.x × NPts.y points
- `bd[ix * NPts.y + iy].x = (x, y, z)` for each point

**Boundary types** (type[0]):
- `'V'`: Vacuum (pressure release)
- `'R'`: Rigid
- `'A'`: Acousto-elastic (specify properties)
- `'F'`: File-based reflection coefficients
- `'*'`: Curvilinear

### Position Structure

Source and receiver positions:

```cpp
struct Position {
    // Counts
    int32_t NSx, NSy, NSz;  // Source counts
    int32_t NRz, NRr, Ntheta;  // Receiver counts
    int32_t NRz_per_range;  // For special receiver layouts
    
    // Flags
    bool SxSyInKm, RrInKm;  // Unit flags
    bool thetaDuplRemoved;  // Whether duplicate angle removed
    
    // Spacings (for uniformly-spaced receivers)
    real Delta_r, Delta_theta;
    
    // Source positions
    real *Sx, *Sy;   // X, Y coordinates
    float *Sz;       // Z coordinates (depths)
    
    // Receiver positions
    real *Rr;        // Ranges
    float *Rz;       // Depths
    real *theta;     // Bearings (degrees)
    vec2 *t_rcvr;    // Bearing direction vectors (cos, sin)
};
```

**Total sources**: NSx × NSy × NSz
**Total receivers**: NRr × Ntheta × NRz

**Indexing**:
```cpp
// Source index
int isx, isy, isz;
int isrc = (isy * NSx + isx) * NSz + isz;

// Receiver index
int ir, itheta, iz;
int irecv = (itheta * NRr + ir) * NRz + iz;
```

### RayInfo and Ray Results

Ray trajectory storage:

```cpp
template<bool O3D, bool R3D> struct RayInfo {
    RayResult<O3D, R3D> *results;  // Array of individual rays
    rayPt<R3D> *RayMem;             // Storage for all ray points
    rayPt<R3D> *WorkRayMem;         // Temporary storage
    size_t RayMemCapacity;          // Total points allocated
    size_t RayMemPoints;            // Points currently used
    int32_t MaxPointsPerRay;        // Max per individual ray
    int32_t NRays;                  // Number of rays
    bool isCopyMode;                // Memory management mode
};

template<bool O3D, bool R3D> struct RayResult {
    rayPt<R3D> *ray;         // Pointer into RayMem for this ray
    Origin<O3D, R3D> org;    // Origin (for Nx2D)
    real SrcDeclAngle;       // Launch angle
    int32_t Nsteps;          // Number of steps in this ray
};

template<bool R3D> struct rayPt {
    int32_t NumTopBnc, NumBotBnc;  // Bounce counts
    VEC23<R3D> x;                   // Position
    VEC23<R3D> t;                   // Tangent (scaled)
    real c;                         // Sound speed
    real Amp, Phase;                // Amplitude and phase
    cpx tau;                        // Travel time
    // + dimension-specific extras (p, q, etc.)
};
```

**Memory layout**:

```
RayMem: [Ray0_pt0][Ray0_pt1]...[Ray0_ptN0][Ray1_pt0]...[RayM_ptNM]
         ^                                ^
         |                                |
results[0].ray                      results[1].ray
```

---

## Module System

### Module Base Class

All parameter modules inherit from:

```cpp
template<bool O3D> class ParamsModule {
public:
    virtual ~ParamsModule() {}
    
    // Read from environment file
    virtual void Read(bhcParams<O3D> &params, LDIFile &ENVFile, HSInfo &topHS) = 0;
    
    // Validate and preprocess
    virtual void Preprocess(bhcParams<O3D> &params) {}
    
    // Write to environment file
    virtual void Write(const bhcParams<O3D> &params, LDOFile &ENVFile) const = 0;
    
    // Write human-readable summary to PRTFile
    virtual void Echo(const bhcParams<O3D> &params) const = 0;
    
    // Validate parameters
    virtual bool Validate(const bhcParams<O3D> &params) const { return true; }
    
    // Default values (when FileRoot is nullptr)
    virtual void Default(bhcParams<O3D> &params) const {}
    
    // Module name for error messages
    virtual const char *Name() const = 0;
};
```

### Module Lifecycle

1. **Construction**: Module objects created at program initialization
2. **Default** (optional): If no environment file, set defaults
3. **Read**: Parse environment file section
4. **Preprocess**: Convert units, compute derived values
5. **Validate**: Check for errors
6. **Echo**: Write summary to PRTFile
7. **Use**: Run simulation
8. **Write** (optional): Save to new environment file

### Example Module: Source Depths (SzRz)

```cpp
template<bool O3D> class SzRz : public ParamsModule<O3D> {
public:
    void Read(bhcParams<O3D> &params, LDIFile &ENVFile, HSInfo &topHS) override {
        LIST(ENVFile); // Read NSz
        int32_t NSz;
        ENVFile.Read(NSz);
        extsetup_sz(params, NSz);
        
        LIST(ENVFile); // Read Sz[:]
        for (int32_t i = 0; i < NSz; ++i) {
            ENVFile.Read(params.Pos->Sz[i]);
        }
        
        // Similar for receiver depths
        // ...
    }
    
    void Preprocess(bhcParams<O3D> &params) override {
        // Validate depths are within bounds
        real minZ = params.Bdry->Top.hs.Depth;
        real maxZ = params.Bdry->Bot.hs.Depth;
        
        for (int32_t i = 0; i < params.Pos->NSz; ++i) {
            if (params.Pos->Sz[i] < minZ || params.Pos->Sz[i] > maxZ) {
                EXTERR("Source depth out of bounds");
            }
        }
        // ...
    }
    
    void Echo(const bhcParams<O3D> &params) const override {
        PRTFILEWRITE("{:6d} source depths", params.Pos->NSz);
        // ...
    }
    
    void Write(const bhcParams<O3D> &params, LDOFile &ENVFile) const override {
        ENVFile.Write(params.Pos->NSz, "'NSD'");
        for (int32_t i = 0; i < params.Pos->NSz; ++i) {
            ENVFile.Write(params.Pos->Sz[i]);
        }
        ENVFile.Write("! Source depths (m)");
        // ...
    }
    
    const char *Name() const override { return "SzRz"; }
};
```

### Module Order

Modules executed in specific order (defined in `api.cpp`):

1. **Atten**: Attenuation info (needed by SSP for complex speeds)
2. **Title**: Simulation title
3. **Freq0**: Frequency
4. **NMedia**: Number of media layers
5. **TopOpt**: Top boundary options
6. **BoundaryCondTop**: Top boundary conditions
7. **SSP**: Sound speed profile
8. **BotOpt**: Bottom boundary options
9. **BoundaryCondBot**: Bottom boundary conditions
10. **SxSy**: Source X/Y positions
11. **SzRz**: Source/receiver depths
12. **RcvrRanges**: Receiver ranges
13. **RcvrBearings**: Receiver bearings
14. **FreqVec**: Frequency vector (broadband)
15. **RunType**: Run type selection
16. **RayAnglesElevation**: Ray elevation angles
17. **RayAnglesBearing**: Ray bearing angles
18. **BeamInfo**: Beam configuration
19. **Altimetry**: Top boundary geometry
20. **Bathymetry**: Bottom boundary geometry
21. **BRC**: Bottom reflection coefficients
22. **TRC**: Top reflection coefficients
23. **SBP**: Source beam pattern

Order matters because later modules may depend on earlier ones.

---

## Mode System

### Mode Base Class

All run modes inherit from:

```cpp
template<bool O3D, bool R3D> class ModeModule {
public:
    virtual ~ModeModule() {}
    
    // Check if this mode should handle the given RunType
    virtual bool Handle(const char RunType[7]) const = 0;
    
    // Execute the run
    virtual void Run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs) = 0;
    
    // Mode name
    virtual const char *Name() const = 0;
};
```

### Mode Selection

In `run()`:

```cpp
template<bool O3D, bool R3D>
bool run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs) {
    // Preprocess parameters
    // ...
    
    // Find mode to handle this RunType
    for (auto *mode : modes) {
        if (mode->Handle(params.Beam->RunType)) {
            mode->Run(params, outputs);
            return true;
        }
    }
    
    EXTERR("Unknown run type: %s", params.Beam->RunType);
    return false;
}
```

### Ray Mode

Traces individual rays, stores full trajectories:

```cpp
template<bool O3D, bool R3D> class Ray : public ModeModule<O3D, R3D> {
public:
    bool Handle(const char RunType[7]) const override {
        return RunType[0] == 'R';  // 'R*'
    }
    
    void Run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs) override {
        // Allocate ray storage
        SetupRayMemory(params, outputs);
        
        // Trace each ray
        RunRays(params, outputs, [](/* ray data */) {
            // Store full ray trajectory
            StoreRay(/* ... */);
        });
    }
};
```

### TL (Transmission Loss) Mode

Computes acoustic field at receivers:

```cpp
template<bool O3D, bool R3D> class TL : public Field<O3D, R3D> {
public:
    bool Handle(const char RunType[7]) const override {
        return RunType[0] == 'C' || RunType[0] == 'S' || RunType[0] == 'I';
    }
    
    void Run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs) override {
        // Allocate field storage
        AllocateField(params, outputs);
        
        // Trace rays and compute influence
        RunRays(params, outputs, [](/* ray data, receivers */) {
            // At each step, check if ray influences receivers
            for (each receiver near ray) {
                // Compute contribution
                cpxf contribution = ComputeInfluence(/* ... */);
                // Add to field
                outputs.uAllSources[irecv] += contribution;
            }
        });
    }
};
```

### Eigenray Mode

Finds rays connecting source to receivers:

```cpp
template<bool O3D, bool R3D> class Eigen : public ModeModule<O3D, R3D> {
public:
    bool Handle(const char RunType[7]) const override {
        return RunType[0] == 'E' || RunType[0] == 'V';  // 'E' or 'V' (combined)
    }
    
    void Run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs) override {
        // Phase 1: Trace all rays, record which hit receivers
        AllocateEigenHits(params, outputs);
        
        RunRays(params, outputs, [](/* ray, receivers */) {
            for (each receiver) {
                if (ray passes near receiver) {
                    // Record this as a hit
                    RecordEigenHit(/* ray info, receiver info */);
                }
            }
        });
        
        // Phase 2: Re-trace just the hits to get full trajectories
        for (each hit) {
            RetraceRay(hit.isx, hit.isy, hit.isz, hit.ialpha, hit.ibeta);
            // Store this ray in outputs.rayinfo
        }
    }
};
```

### Arrivals Mode

Computes detailed arrival structure:

```cpp
template<bool O3D, bool R3D> class Arr : public Field<O3D, R3D> {
public:
    bool Handle(const char RunType[7]) const override {
        return RunType[0] == 'A' || RunType[0] == 'a' || RunType[0] == 'V';
    }
    
    void Run(bhcParams<O3D> &params, bhcOutputs<O3D, R3D> &outputs) override {
        // Allocate arrivals storage
        AllocateArrivals(params, outputs);
        
        // Trace rays
        RunRays(params, outputs, [](/* ray data, receivers */) {
            for (each receiver influenced by ray) {
                // Create arrival record
                Arrival arr;
                arr.delay = /* travel time */;
                arr.a = /* amplitude */;
                arr.Phase = /* phase */;
                arr.NTopBnc = /* top bounces */;
                arr.NBotBnc = /* bottom bounces */;
                // ...
                
                // Add to arrivals for this receiver
                AddArrival(irecv, arr);
            }
        });
        
        // Post-process: sort, merge similar arrivals
        for (each receiver) {
            SortArrivals(irecv);
            if (params.arrinfo->AllowMerging) {
                MergeArrivals(irecv);
            }
        }
    }
};
```

---

## Ray Tracing Engine

### Main Tracing Loop (trace.hpp)

The core ray tracing function:

```cpp
template<bool O3D, bool R3D, bool CHECK_RECEIVERS>
void TraceRay(
    bhcParams<O3D> &params,
    const RayInitInfo &rinit,
    rayPt<R3D> *ray,
    int32_t &Nsteps,
    /* receiver influence callback */)
{
    // Initialize ray at source
    rayPt<R3D> ray0;
    InitializeRay(params, rinit, ray0);
    
    int32_t istep = 0;
    ray[istep++] = ray0;
    
    // Main loop
    while (istep < MaxSteps) {
        rayPt<R3D> &rayCurrent = ray[istep - 1];
        rayPt<R3D> &rayNext = ray[istep];
        
        // Take a step
        real stepSize = ComputeStepSize(params, rayCurrent);
        Step(params, rayCurrent, stepSize, rayNext);
        
        // Check boundaries
        BoundaryCheck(params, rayCurrent, rayNext, /* ... */);
        
        if constexpr(CHECK_RECEIVERS) {
            // Check if ray influences receivers
            CheckReceiverInfluence(params, rayCurrent, rayNext, /* ... */);
        }
        
        // Check termination conditions
        if (RayOutOfBounds(params, rayNext)) break;
        if (RayTooWeak(rayNext)) break;
        if (RayLooping(params, rayNext)) break;
        
        ++istep;
    }
    
    Nsteps = istep;
}
```

### Step Function (step.hpp)

Single step of ray propagation:

```cpp
template<bool O3D, bool R3D>
void Step(
    const bhcParams<O3D> &params,
    const rayPt<R3D> &ray,
    real h,  // step size
    rayPt<R3D> &rayOut)
{
    // Current state
    VEC23<R3D> x = ray.x;
    VEC23<R3D> t = ray.t;
    real c = ray.c;
    
    // Evaluate SSP and derivatives at current position
    SSPOutputs<R3D> ssp;
    EvaluateSSP(params, x, ssp);
    
    // Ray equations (2D example):
    // dx/ds = c² * t
    // dt/ds = ∇c
    // where s is arc length parameter
    
    // Runge-Kutta 4th order or similar integrator
    VEC23<R3D> k1_x = c * c * t;
    VEC23<R3D> k1_t = ssp.gradc;
    
    VEC23<R3D> x_mid = x + RL(0.5) * h * k1_x;
    VEC23<R3D> t_mid = t + RL(0.5) * h * k1_t;
    
    SSPOutputs<R3D> ssp_mid;
    EvaluateSSP(params, x_mid, ssp_mid);
    
    VEC23<R3D> k2_x = ssp_mid.ccpx.real() * ssp_mid.ccpx.real() * t_mid;
    VEC23<R3D> k2_t = ssp_mid.gradc;
    
    // ... k3, k4 ...
    
    // Update position and tangent
    rayOut.x = x + h * (k1_x + RL(2.0)*k2_x + RL(2.0)*k3_x + k4_x) / RL(6.0);
    rayOut.t = t + h * (k1_t + RL(2.0)*k2_t + RL(2.0)*k3_t + k4_t) / RL(6.0);
    
    // Evaluate SSP at new position
    EvaluateSSP(params, rayOut.x, ssp);
    rayOut.c = ssp.ccpx.real();
    
    // Update amplitude, phase, etc.
    UpdateBeamQuantities(params, ray, rayOut, h);
}
```

### Boundary Handling

When a ray approaches a boundary:

```cpp
template<bool O3D, bool R3D>
void HandleBoundary(
    const bhcParams<O3D> &params,
    const rayPt<R3D> &rayIn,
    rayPt<R3D> &rayOut,
    bool isTop)  // true = top boundary, false = bottom
{
    // Get boundary properties at intersection point
    BdryType *bdry = params.Bdry;
    HSInfo &hs = isTop ? bdry->Top.hs : bdry->Bot.hs;
    
    // Reflect or transmit based on boundary type
    switch(hs.bc) {
    case 'V':  // Vacuum (pressure release)
        ReflectPerfect(rayIn, rayOut, /* normal */, -1.0);
        break;
        
    case 'R':  // Rigid
        ReflectPerfect(rayIn, rayOut, /* normal */, +1.0);
        break;
        
    case 'A':  // Acousto-elastic
        ReflectAcoustoElastic(params, rayIn, rayOut, hs, /* ... */);
        break;
        
    case 'F':  // File-based reflection coefficient
        ReflectWithCoefficient(params, rayIn, rayOut, /* angle, refl coef */);
        break;
    }
    
    // Update bounce count
    if (isTop) {
        rayOut.NumTopBnc = rayIn.NumTopBnc + 1;
    } else {
        rayOut.NumBotBnc = rayIn.NumBotBnc + 1;
    }
}

void ReflectPerfect(
    const rayPt<R3D> &rayIn,
    rayPt<R3D> &rayOut,
    const VEC23<R3D> &normal,
    real reflCoef)
{
    // Specular reflection: t' = t - 2(t·n)n
    real tdotn = glm::dot(rayIn.t, normal);
    rayOut.t = rayIn.t - RL(2.0) * tdotn * normal;
    
    // Update amplitude with reflection coefficient
    rayOut.Amp = rayIn.Amp * STD::abs(reflCoef);
    
    // Phase change
    rayOut.Phase = rayIn.Phase + (reflCoef < 0 ? REAL_PI : RL(0.0));
}
```

---

## SSP Interpolation

### SSP Evaluation Dispatcher

```cpp
template<bool O3D, bool R3D>
void EvaluateSSP(
    const bhcParams<O3D> &params,
    const VEC23<R3D> &x,
    SSPOutputs<R3D> &ssp)
{
    switch(params.ssp->Type) {
    case 'N':
        EvaluateSSP_N2Linear<O3D, R3D>(params, x, ssp);
        break;
    case 'C':
        EvaluateSSP_CLinear<O3D, R3D>(params, x, ssp);
        break;
    case 'S':
        EvaluateSSP_CubicSpline<O3D, R3D>(params, x, ssp);
        break;
    case 'P':
        EvaluateSSP_PCHIP<O3D, R3D>(params, x, ssp);
        break;
    case 'Q':
        EvaluateSSP_Quad(params, x, ssp);  // 2D only
        break;
    case 'H':
        EvaluateSSP_Hexahedral<R3D>(params, x, ssp);  // 3D/Nx2D only
        break;
    case 'A':
        EvaluateSSP_Analytic<O3D, R3D>(params, x, ssp);
        break;
    }
}
```

### N²-Linear Interpolation

Most common and computationally efficient:

```cpp
template<bool O3D, bool R3D>
void EvaluateSSP_N2Linear(
    const bhcParams<O3D> &params,
    const VEC23<R3D> &x,
    SSPOutputs<R3D> &ssp)
{
    const SSPStructure *sspdata = params.ssp;
    
    // Find depth layer
    real z = x.y;  // or x.z in 3D
    int iz = FindLayer(sspdata->z, sspdata->NPts, z);
    
    // Interpolation weight
    real dz = sspdata->z[iz+1] - sspdata->z[iz];
    real w = (z - sspdata->z[iz]) / dz;
    
    // Interpolate n² linearly
    cpx n2 = (RL(1.0) - w) * sspdata->n2[iz] + w * sspdata->n2[iz+1];
    cpx n2z = sspdata->n2z[iz];  // Constant within layer
    
    // Convert to c
    ssp.ccpx = STD::sqrt(n2);
    
    // Gradient: ∇c = ∇(√n²) = (1/2√n²) ∇n²
    real c = ssp.ccpx.real();
    if constexpr(R3D) {
        ssp.gradc.x = 0;  // No horizontal variation in 1D profile
        ssp.gradc.y = 0;
        ssp.gradc.z = (n2z / (RL(2.0) * n2)).real();
    } else {
        ssp.gradc.x = 0;
        ssp.gradc.y = (n2z / (RL(2.0) * n2)).real();
    }
    
    // Second derivative
    ssp.czz = /* formula for d²c/dz² */;
}
```

### Quadrilateral (2D Range-Dependent)

Bilinear interpolation on (r, z) grid:

```cpp
void EvaluateSSP_Quad(
    const bhcParams<false> &params,
    const vec2 &x,
    SSPOutputs<false> &ssp)
{
    const SSPStructure *sspdata = params.ssp;
    
    // Find grid cell
    real r = x.x;
    real z = x.y;
    
    int ir = FindLayer(sspdata->Seg.r, sspdata->Nr, r);
    int iz = FindLayer(sspdata->z, sspdata->Nz, z);
    
    // Bilinear weights
    real wr = (r - sspdata->Seg.r[ir]) / (sspdata->Seg.r[ir+1] - sspdata->Seg.r[ir]);
    real wz = (z - sspdata->z[iz]) / (sspdata->z[iz+1] - sspdata->z[iz]);
    
    // Four corners of cell
    real c00 = sspdata->cMat[iz * sspdata->Nr + ir];
    real c10 = sspdata->cMat[iz * sspdata->Nr + (ir+1)];
    real c01 = sspdata->cMat[(iz+1) * sspdata->Nr + ir];
    real c11 = sspdata->cMat[(iz+1) * sspdata->Nr + (ir+1)];
    
    // Bilinear interpolation
    real c = (1-wr)*(1-wz)*c00 + wr*(1-wz)*c10 + (1-wr)*wz*c01 + wr*wz*c11;
    ssp.ccpx = c;
    
    // Gradients (finite differences)
    real dr = sspdata->Seg.r[ir+1] - sspdata->Seg.r[ir];
    real dz = sspdata->z[iz+1] - sspdata->z[iz];
    
    real c_r = ((1-wz)*c10 + wz*c11 - (1-wz)*c00 - wz*c01) / dr;
    real c_z = ((1-wr)*c01 + wr*c11 - (1-wr)*c00 - wr*c10) / dz;
    
    ssp.gradc = vec2(c_r, c_z);
    
    // Second derivatives
    ssp.crr = /* ... */;
    ssp.crz = /* ... */;
    ssp.czz = /* ... */;
}
```

---

## Influence Functions

### Overview

Influence functions compute how a ray contributes to the acoustic field at receivers. Different formulations for different beam types.

### Geometric Hat Beam (2D Ray-Centered)

Simplest influence function:

```cpp
template<bool O3D>
void InfluenceGeoHatRayCen2D(
    const bhcParams<O3D> &params,
    const InfluenceRayInfo<false> &rayinfo,
    const rayPt<false> &ray,
    const vec2 &rcvrPos,
    cpxf &contrib)
{
    // Distance from ray to receiver in ray-centered coordinates
    vec2 rayToRcvr = rcvrPos - ray.x;
    
    // Project onto ray normal
    vec2 normal = vec2(-ray.t.y, ray.t.x) / glm::length(ray.t);
    real xi = glm::dot(rayToRcvr, normal);
    
    // Hat function: 1 within beam width, 0 outside
    real beamWidth = ComputeBeamWidth(params, rayinfo, ray);
    
    if (STD::abs(xi) > beamWidth / RL(2.0)) {
        contrib = cpxf(0, 0);
        return;
    }
    
    // Contribution within beam
    real w = RL(1.0) / beamWidth;  // Normalize
    
    // Amplitude with geometric spreading
    real r = glm::length(ray.x);
    real spreading = RL(1.0) / STD::sqrt(r);
    
    // Phase
    real phase = rayinfo.omega * ray.tau.real() + ray.Phase;
    
    // Combine
    contrib = ray.Amp * spreading * w * STD::exp(cpxf(0, phase));
}
```

### Cerveny Ray-Centered (2D)

More accurate, accounts for beam spreading:

```cpp
template<bool O3D>
void InfluenceCervenyRayCen2D(
    const bhcParams<O3D> &params,
    const InfluenceRayInfo<false> &rayinfo,
    const rayPt<false> &ray,
    const vec2 &rcvrPos,
    cpxf &contrib)
{
    // Ray-centered coordinates at receiver
    vec2 rayToRcvr = rcvrPos - ray.x;
    vec2 tangent = ray.t / glm::length(ray.t);
    vec2 normal = vec2(-tangent.y, tangent.x);
    
    real s = glm::dot(rayToRcvr, tangent);  // Along ray
    real xi = glm::dot(rayToRcvr, normal);   // Across ray
    
    // Dynamic ray theory: beam spreads according to q
    // p and q satisfy transport equations
    real q = ray.q.x;  // or appropriate component
    
    if (STD::abs(q) < REAL_EPSILON) {
        // Caustic - special handling
        contrib = /* ... */;
        return;
    }
    
    // Beam width from q
    real width = ComputeDynamicWidth(params, rayinfo, q);
    
    // Check if receiver is within beam
    if (STD::abs(xi) > width) {
        contrib = cpxf(0, 0);
        return;
    }
    
    // Contribution
    real spreading = RL(1.0) / STD::sqrt(STD::abs(q));
    real phase = rayinfo.omega * (ray.tau.real() + xi*xi/(RL(2.0)*q));
    
    contrib = ray.Amp * spreading * STD::exp(cpxf(0, phase));
}
```

### Simple Gaussian Beam (3D)

```cpp
template<bool O3D>
void InfluenceSGB3D(
    const bhcParams<O3D> &params,
    const InfluenceRayInfo<true> &rayinfo,
    const rayPt<true> &ray,
    const vec3 &rcvrPos,
    cpxf &contrib)
{
    // Gaussian beam: amplitude ~ exp(-r²/w²)
    vec3 rayToRcvr = rcvrPos - ray.x;
    
    // Distance perpendicular to ray
    vec3 tangent = glm::normalize(ray.t);
    real alongRay = glm::dot(rayToRcvr, tangent);
    vec3 perp = rayToRcvr - alongRay * tangent;
    real r_perp = glm::length(perp);
    
    // Beam width (increases with distance)
    real beamWidth = ComputeGaussianWidth(params, rayinfo, ray);
    
    // Gaussian envelope
    real gauss = STD::exp(-r_perp * r_perp / (beamWidth * beamWidth));
    
    // Phase
    real phase = rayinfo.omega * ray.tau.real() + ray.Phase;
    
    // Combine
    contrib = ray.Amp * gauss * STD::exp(cpxf(0, phase));
}
```

---

## Memory Management

### Allocation Strategy

**Parameters**: Allocated once, reused
- Most arrays allocated in `extsetup_*` functions
- User calls `extsetup_*` to resize
- Freed in `finalize()`

**Outputs**: Allocated per run, may be reallocated
- Ray storage: Allocated in ray/eigenray modes
- Field storage: Allocated in TL modes
- Arrivals storage: Allocated in arrivals modes
- Freed and reallocated if dimensions change between runs

### Ray Memory Management

Two strategies:

**Standard mode** (`useRayCopyMode = false`):
```
Allocate: RayMemCapacity = min(NRays * EstimatedStepsPerRay, MaxMemory)
If not enough: Reduce EstimatedStepsPerRay (MaxPointsPerRay)
Store: All rays in single contiguous array
```

**Copy mode** (`useRayCopyMode = true`):
```
Allocate: WorkRayMem = one ray worth of space
For each ray:
    Trace into WorkRayMem
    Copy to permanent storage
    (May be slower due to copy, but uses less memory)
```

### Custom Allocators

For advanced users needing custom memory management:

```cpp
// Not directly supported in current API
// Would require modifying internal allocator calls
// Consider requesting feature if needed
```

---

## CUDA Implementation

### Unified Codebase

Same source code compiles to CPU and GPU:

```cpp
// In common.hpp
#ifdef BHC_BUILD_CUDA
#define HOST_DEVICE __host__ __device__
#define STD cuda::std
#else
#define HOST_DEVICE
#define STD std
#endif

// In implementation
HOST_DEVICE real SomeFunction(real x) {
    return STD::sqrt(x);  // Works on CPU and GPU
}
```

### Kernel Structure

Template instantiation generates `.cu` files from `.cu.in`:

```cpp
// fieldimpl.cu.in (template)
template<bool O3D, bool R3D>
__global__ void TraceRaysKernel(
    /* device pointers to params, outputs */)
{
    int iray = blockIdx.x * blockDim.x + threadIdx.x;
    if (iray >= nRays) return;
    
    // Each thread traces one ray
    TraceRay(/* ... */);
}
```

Generated:
```cpp
// fieldimpl_2D.cu
template<>
__global__ void TraceRaysKernel<false, false>(...) { /* ... */ }

// fieldimpl_3D.cu
template<>
__global__ void TraceRaysKernel<true, true>(...) { /* ... */ }
```

### Memory Transfer

Parameters copied to GPU at start of run:

```cpp
void SetupGPU(const bhcParams<O3D> &params, GPUParams &d_params) {
    // Allocate device memory
    cudaMalloc(&d_params.ssp, sizeof(SSPStructure));
    cudaMalloc(&d_params.Pos, sizeof(Position));
    // ...
    
    // Copy to device
    cudaMemcpy(d_params.ssp, params.ssp, sizeof(SSPStructure), 
               cudaMemcpyHostToDevice);
    // ... copy all structures
    
    // Copy arrays
    cudaMalloc(&d_params.ssp->z, params.ssp->NPts * sizeof(real));
    cudaMemcpy(d_params.ssp->z, params.ssp->z, 
               params.ssp->NPts * sizeof(real),
               cudaMemcpyHostToDevice);
    // ... copy all arrays
}
```

Results copied back at end:

```cpp
void RetrieveResults(const GPUOutputs &d_outputs, bhcOutputs<O3D, R3D> &outputs) {
    // Copy field data
    cudaMemcpy(outputs.uAllSources, d_outputs.uAllSources,
               nReceivers * sizeof(cpxf),
               cudaMemcpyDeviceToHost);
    // ... copy other outputs
}
```

### Synchronization

Kernels launched asynchronously:

```cpp
// Launch kernel
TraceRaysKernel<<<nBlocks, nThreads>>>(d_params, d_outputs);

// Synchronize before accessing results
cudaDeviceSynchronize();
checkCudaErrors(cudaGetLastError());
```

Non-blocking mode still synchronizes internally before returning results.

### Performance Considerations

**Optimize**:
- Minimize host-device transfers
- Coalesce memory accesses
- Use shared memory for frequently-accessed data
- Launch enough threads to saturate GPU

**Profile**:
```bash
nvprof ./bellhopcuda --3D TestCase
# Or use NVIDIA Nsight
```

---

## Contributing Guidelines

### Code Style

- **Indentation**: 4 spaces (no tabs)
- **Braces**: K&R style
  ```cpp
  if (condition) {
      // code
  } else {
      // code
  }
  ```
- **Naming**:
  - Types: `PascalCase` (e.g., `SSPStructure`)
  - Functions: `PascalCase` (e.g., `TraceRay`)
  - Variables: `camelCase` (e.g., `rayInfo`)
  - Constants: `UPPER_CASE` (e.g., `MAX_RAYS`)
  - Template parameters: `PascalCase` (e.g., `O3D`, `R3D`)

- **Comments**:
  - Public API: Doxygen-style documentation
  - Internal: C++ // style
  - Attribution: Mark "LP:" for new code, unmarked for BELLHOP original

### Testing

Before submitting PR:

1. **Build**: All build configurations (2D/3D/Nx2D, CPU/CUDA)
2. **Coverage tests**: Run `./run_all_gen.sh`
3. **OALIB tests**: Run standard test suite
4. **New tests**: Add test cases for new features
5. **Comparison**: Verify results match modified BELLHOP

### Pull Request Process

1. Fork repository
2. Create feature branch
3. Implement changes with tests
4. Ensure all tests pass
5. Update documentation
6. Submit PR with:
   - Clear description of changes
   - Motivation/use case
   - Test results
   - Any breaking changes noted

### Areas for Contribution

- **New SSP types**: Additional interpolation schemes
- **New boundary conditions**: Different reflection models
- **Optimization**: Performance improvements
- **Bug fixes**: Numerical stability, edge cases
- **Documentation**: Examples, tutorials, guides
- **Testing**: More comprehensive test coverage
- **MATLAB interface**: MEX file wrapper
- **Python bindings**: Pybind11 wrapper

---

**This guide is a living document. Contributions and corrections welcome!**
