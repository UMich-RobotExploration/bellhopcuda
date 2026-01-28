# Testing and Validation Guide

## Overview

BellhopCUDA/BellhopCXX employs a comprehensive testing strategy to ensure correctness, accuracy, and performance. This guide explains the testing framework and how to use it.

## Test Categories

### 1. Coverage Tests

**Purpose**: Ensure all feature combinations compile and run without crashing

**Location**: Generated tests in test workspace

**How to run**:
```bash
# Generate test environments
python gen_tests.py

# Run all coverage tests
./run_all_gen.sh

# Or run specific subset
./run_tests.sh 2D ray N2linear  # Example pattern
```

**What's tested**:
- All dimensionalities (2D, 3D, Nx2D)
- All run types (ray, TL, eigenray, arrivals)
- All SSP types (N²-linear, C-linear, cubic, PCHIP, quad, hexahedral)
- All beam types (geometric, Cerveny, SGB)
- All boundary conditions
- Feature combinations that should be rejected (if BHC_LIMIT_FEATURES=ON)

**Expected results**:
- ~1500 tests total
- All should complete without errors
- Some combinations may be skipped if features disabled at compile time

### 2. OALIB Standard Tests

**Purpose**: Validate against canonical test cases from Acoustics Toolbox

**Location**: `test/in/` directory

**Test cases include**:
- **MunkB_***: Munk profile tests (deep ocean sound channel)
- **aet_***: Arctic environment tests
- **Beam_***: Beam pattern tests
- **Seamount_***: 3D bathymetry tests
- Many others covering various acoustic scenarios

**How to run**:
```bash
# Run individual test
./bellhopcxx --2D ../test/in/MunkB_ray
./bellhopcxx --3D ../test/in/Seamount

# Batch run
cd test/in
for env in *.env; do
    ../../build/bin/bellhopcxx --2D $(basename $env .env)
done
```

**Validation**:
- Visual comparison of ray plots
- Numerical comparison of TL fields
- Arrival structure comparison

### 3. Comparison Tests

**Purpose**: Verify results match modified BELLHOP

**Scripts**:
- `compare_arrivals.py`: Compare arrivals files
- `compare_ray.py`: Compare ray coordinates
- `compare_shdfil.py`: Compare transmission loss fields
- `compare_floats.py`: Helper for floating-point comparison

**How to use**:
```bash
# First, run reference BELLHOP
bellhop_modified MunkB_ray

# Then run our version
./bellhopcxx --2D MunkB_ray

# Compare
python compare_ray.py MunkB_ray
```

**What's compared**:
- **Rays**: Coordinates, bounce counts, amplitudes
- **TL**: Complex field values at each receiver
- **Arrivals**: Delays, amplitudes, phases, bounce counts

**Tolerances**:
- Absolute: ~1e-6 for positions, ~1e-4 for amplitudes
- Relative: ~1e-5 for most quantities
- Some tests have looser tolerances for known numerical sensitivity

### 4. Performance Tests

**Purpose**: Measure speedup and identify performance regressions

**Scripts**:
- `parse_timing_results.py`: Extract timing from print files

**How to run**:
```bash
# Run with timing
./bellhopcxx --2D MunkB_coherent > timing_cxx.txt 2>&1
./bellhopcuda --2D MunkB_coherent > timing_cuda.txt 2>&1

# Compare to original
bellhop MunkB_coherent > timing_orig.txt 2>&1

# Parse results
python parse_timing_results.py timing_*.txt
```

**Metrics**:
- Wall time
- Ray tracing time
- Field computation time
- I/O time
- Speedup vs. original BELLHOP

## Test Environment Structure

### Environment File (.env)

Defines simulation parameters:

```
'Test Title'                     ! Title
1500.0                           ! Frequency (Hz)
1                                ! NMEDIA
'SVT'                            ! SSP options
0   0.0   1500.0  0.0  1.0  0.0 0.0  ! Top boundary
100 1500.0 0.0 1.0 0.0 0.0       ! Depth points
5000 1500.0 0.0 1.0 0.0 0.0
'A'                              ! Bottom boundary
5000.0 1500.0 0.0 1.0 0.0 0.0   ! Bottom properties
1                                ! NSx
0.0                              ! Sx
1                                ! NSy  
0.0                              ! Sy
1                                ! NSz
1000.0                           ! Sz (m)
101                              ! NRz
0.0 5000.0 /                     ! Rz range
1001                             ! NRr
0.0 10.0 /                       ! Rr range (km)
'RG   2'                         ! Run type
201                              ! N angles
-90 90 /                         ! Angle range
0.0 5000.0 10000.0              ! Box limits
```

### Supporting Files

Depending on SSP type and boundary conditions:
- `.ssp`: Sound speed profile data (Type='Q' or 'H')
- `.bty`: Bathymetry data
- `.ati`: Altimetry data
- `.brc`/`.trc`: Reflection coefficients
- `.sbp`: Source beam pattern

### Output Files

After running:
- `.prt`: Print file (human-readable summary)
- `.shd`: Shade file (TL field) - binary
- `.ray`: Ray coordinates - binary
- `.arr`: Arrivals - binary

## Writing New Tests

### 1. Create Environment File

```bash
# Copy existing similar test
cp test/in/MunkB_ray.env test/in/MyTest.env

# Edit parameters
nano test/in/MyTest.env
```

### 2. Add Supporting Files (if needed)

For custom SSP:
```bash
# Create .ssp file for quadrilateral mode
cat > test/in/MyTest.ssp << EOF
51 21  ! NPts Nr
0.0
50.0
... (depths)
0.0 5000.0 ... (ranges)
1500.0 1500.1 ... (sound speeds)
EOF
```

### 3. Run Test

```bash
./bellhopcxx --2D test/in/MyTest
```

### 4. Validate Results

Check print file for errors:
```bash
cat test/in/MyTest.prt
```

Visualize (requires MATLAB or compatible):
```matlab
% Load and plot TL
plotshd('test/in/MyTest.shd')

% Load and plot rays
plotray('test/in/MyTest.ray')
```

### 5. Add to Test Suite

Add to coverage test generator:
```python
# In gen_tests.py
test_cases.append({
    'name': 'MyTest',
    'dim': '2D',
    'runtype': 'C',
    'ssp': 'N',
    # ...
})
```

## Debugging Test Failures

### Strategy

1. **Identify failing test**:
   ```bash
   ./run_all_gen.sh | grep FAIL
   ```

2. **Run individually with verbose output**:
   ```bash
   ./bellhopcxx --2D test/in/FailingTest
   cat test/in/FailingTest.prt
   ```

3. **Compare to reference**:
   ```bash
   # If you have reference output
   python compare_ray.py test/in/FailingTest
   ```

4. **Isolate issue**:
   - Check if it's setup (parameters) or run (computation)
   - Try simpler variant (fewer rays, receivers, etc.)
   - Enable debug output (if compiled with BHC_DEBUG=ON)

5. **Find diverging ray** (for accuracy issues):
   ```bash
   python find_failing_ray.py test/in/FailingTest
   ```

### Common Issues

#### Setup Failures

**Symptom**: Test fails immediately, setup() returns false

**Causes**:
- Missing or malformed input file
- Incompatible parameter combination
- Out-of-bounds values

**Debug**:
```bash
# Check print file
cat test/in/Test.prt | grep -i error

# Run with validation
./bellhopcxx --2D test/in/Test 2>&1 | less
```

#### Runtime Failures

**Symptom**: Test crashes or hangs during execution

**Causes**:
- Numerical instability (ray goes to infinity)
- Memory issues
- Boundary handling bug

**Debug**:
```bash
# Enable debug mode (requires recompile)
cmake .. -DBHC_DEBUG=ON
cmake --build .

# Run with debugger
gdb --args ./bellhopcxx --2D test/in/Test
```

#### Accuracy Failures

**Symptom**: Results don't match reference

**Causes**:
- Expected numerical differences
- Compiler optimization differences
- Bug in new code

**Debug**:
```bash
# Find first diverging ray
python find_failing_ray.py test/in/Test

# Compare step-by-step
# Add print statements at divergence point
```

### Debugging Tools

#### Print File Analysis

```bash
# Extract warnings
grep -i "warn" test/in/*.prt

# Extract errors
grep -i "error" test/in/*.prt

# Extract timing
grep -i "time\|elapsed" test/in/*.prt
```

#### Binary File Inspection

```python
# Read shade file
import struct
with open('test.shd', 'rb') as f:
    # Read header
    title = f.read(80).decode('utf-8').strip()
    freq = struct.unpack('f', f.read(4))[0]
    # ... etc
```

#### Memory Debugging

```bash
# Valgrind (Linux)
valgrind --leak-check=full ./bellhopcxx --2D test/in/Test

# CUDA memory check
cuda-memcheck ./bellhopcuda --2D test/in/Test
```

## Continuous Integration

### Automated Testing

GitHub Actions workflow runs on each commit:

1. Build all configurations (2D/3D/Nx2D, CPU/CUDA)
2. Run coverage tests
3. Run OALIB tests
4. Compare to reference
5. Report failures

### Local CI Simulation

```bash
# Full test suite
./run_all_gen.sh 2>&1 | tee test_results.txt

# Count passes/fails
grep -c "PASS" test_results.txt
grep -c "FAIL" test_results.txt

# List failures
grep "FAIL" test_results.txt
```

## Performance Benchmarking

### Timing Individual Tests

```bash
# Time execution
time ./bellhopcxx --2D test/in/MunkB_coherent

# Extract from print file
grep "Elapsed" test/in/MunkB_coherent.prt
```

### Systematic Benchmarking

```bash
# Run suite with timing
for test in MunkB_coherent MunkB_ray Seamount; do
    echo "=== $test ==="
    time ./bellhopcxx --2D test/in/$test
    time ./bellhopcuda --2D test/in/$test
done
```

### Performance Regression Testing

```bash
# Save baseline
./bellhopcxx --2D test/in/MunkB_coherent
grep "Elapsed" test/in/MunkB_coherent.prt > baseline_timing.txt

# After changes
./bellhopcxx --2D test/in/MunkB_coherent
grep "Elapsed" test/in/MunkB_coherent.prt > new_timing.txt

# Compare
diff baseline_timing.txt new_timing.txt
```

## Validation Against BELLHOP

### Setup Reference Version

```bash
# Download and compile modified BELLHOP
git clone https://github.com/A-New-BellHope/bellhop.git
cd bellhop
# Follow compilation instructions
```

### Run Comparison

```bash
# Run reference
cd /path/to/bellhop
./bellhop.exe /path/to/test/in/MunkB_coherent

# Run ours
cd /path/to/bellhopcuda
./bellhopcxx --2D test/in/MunkB_coherent

# Compare
python compare_shdfil.py test/in/MunkB_coherent
```

### Interpreting Comparison Results

**Exact match**: Rare, only for very simple cases

**Close match** (differences < 1e-4): Expected and acceptable
- Compiler differences
- Floating-point operation ordering
- Numerical stability improvements

**Significant differences**: May indicate bug
- Check if test is in known problematic category
- Verify environment files are identical
- Check for recent code changes
- Report issue if unexplained

## Test Data Management

### Organizing Test Results

```bash
# Create results directory
mkdir -p test_results/$(date +%Y%m%d)

# Run tests and save
./run_all_gen.sh > test_results/$(date +%Y%m%d)/coverage.log 2>&1

# Archive output files
tar -czf test_results/$(date +%Y%m%d)/outputs.tar.gz test/in/*.{prt,shd,ray,arr}
```

### Cleaning Test Outputs

```bash
# Remove output files
rm -f test/in/*.prt test/in/*.shd test/in/*.ray test/in/*.arr

# Clean build outputs
cd build && make clean
```

## Best Practices

### Before Committing Code

1. **Run coverage tests**: Ensure no regressions
   ```bash
   ./run_all_gen.sh | tee coverage_results.txt
   grep -c "FAIL" coverage_results.txt  # Should be 0
   ```

2. **Run OALIB tests**: Check standard cases
   ```bash
   for f in test/in/*.env; do ./bellhopcxx --2D $(basename $f .env); done
   ```

3. **Compare to reference**: Verify accuracy maintained
   ```bash
   python compare_shdfil.py test/in/MunkB_coherent
   ```

4. **Check performance**: No significant slowdown
   ```bash
   time ./bellhopcxx --2D test/in/MunkB_coherent
   ```

### When Adding Features

1. **Create specific test**: Demonstrate feature works
2. **Add to coverage**: Ensure feature tested going forward
3. **Document**: Update this guide and main docs
4. **Validate**: Compare to BELLHOP if analogous feature exists

### When Fixing Bugs

1. **Create reproducer**: Minimal test case showing bug
2. **Add to test suite**: Prevent regression
3. **Verify fix**: Test passes after fix
4. **Check side effects**: Run full test suite

## Troubleshooting

### Tests Fail After Update

```bash
# Rebuild completely
rm -rf build
mkdir build && cd build
cmake .. -DBHC_ENABLE_CUDA=OFF
cmake --build . --config Release

# Re-run tests
./run_all_gen.sh
```

### Inconsistent Results

```bash
# Check build configuration
cmake -L | grep BHC

# Ensure using same precision
grep BHC_USE_FLOATS CMakeCache.txt

# Check compiler optimization
grep CMAKE_BUILD_TYPE CMakeCache.txt  # Should be Release
```

### Can't Reproduce Reference Results

- Ensure using modified BELLHOP, not original
- Check environment files are byte-for-byte identical
- Verify same precision (double vs. float)
- Some tests have known sensitivity

## Reporting Test Failures

When reporting issues:

1. **Environment file**: Attach or link to .env file
2. **Command**: Exact command used
3. **Output**: Include .prt file
4. **Expected vs. actual**: What should happen vs. what does
5. **System info**: OS, compiler, GPU (if CUDA)
6. **Version**: Git commit hash

Submit to: https://github.com/A-New-BellHope/bellhopcuda/issues

---

**Remember**: Testing is ongoing! As the code evolves, tests must evolve too.
