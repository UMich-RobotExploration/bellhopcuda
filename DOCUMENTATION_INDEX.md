# BellhopCUDA/BellhopCXX Documentation Index

Welcome to the BellhopCUDA/BellhopCXX documentation! This index will help you find the information you need.

## 📖 Documentation Structure

### For New Users

**Start Here:**
1. **[README.md](README.md)** - Project overview and quick start (5 min)
2. **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Common tasks and API quick reference (10 min)
3. **[doc/compilation.md](doc/compilation.md)** - How to build the project (15 min)
4. **[examples/](examples/)** - Example programs showing library usage

**Then Move To:**
- **[DOCUMENTATION.md](DOCUMENTATION.md)** - Complete user guide covering:
  - What is BellhopCUDA/CXX and why use it
  - Architecture overview
  - Core concepts (SSP, boundaries, ray theory)
  - Complete API reference
  - Build system details
  - Usage examples
  - Advanced topics

### For Developers

**Architecture and Code:**
- **[DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)** - Deep dive into:
  - Code architecture and organization
  - Template system explanation
  - Data structures in detail
  - Module system design
  - Mode system design
  - Ray tracing engine internals
  - SSP interpolation methods
  - Influence functions
  - Memory management
  - CUDA implementation
  - Contributing guidelines

**Testing:**
- **[TESTING_GUIDE.md](TESTING_GUIDE.md)** - Testing framework:
  - Test categories and organization
  - How to run tests
  - Writing new tests
  - Debugging test failures
  - Performance benchmarking
  - Validation against BELLHOP

### Reference Materials

**Quick Lookups:**
- **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Cheat sheet for:
  - Common code patterns
  - Template parameter guide
  - RunType format
  - SSP and boundary types
  - API function signatures
  - Error solutions
  - Build options

**Detailed Topics:**
- **[doc/accuracy.md](doc/accuracy.md)** - Numerical accuracy and validation
- **[doc/performance.md](doc/performance.md)** - Performance characteristics and tuning
- **[doc/faq.md](doc/faq.md)** - Frequently asked questions

## 🎯 Find What You Need

### By Task

| I want to... | Read this... |
|--------------|--------------|
| Install and run my first simulation | [README.md](README.md), [doc/compilation.md](doc/compilation.md) |
| Use as library in my project | [DOCUMENTATION.md](DOCUMENTATION.md) § API Reference, [examples/](examples/) |
| Understand how it works | [DOCUMENTATION.md](DOCUMENTATION.md) § Architecture, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) |
| Modify parameters programmatically | [QUICK_REFERENCE.md](QUICK_REFERENCE.md), [DOCUMENTATION.md](DOCUMENTATION.md) § API Reference |
| Create custom SSP or boundaries | [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Data Structures |
| Improve performance | [doc/performance.md](doc/performance.md), [DOCUMENTATION.md](DOCUMENTATION.md) § Advanced Topics |
| Contribute code | [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Contributing Guidelines |
| Run or write tests | [TESTING_GUIDE.md](TESTING_GUIDE.md) |
| Debug a problem | [QUICK_REFERENCE.md](QUICK_REFERENCE.md) § Common Errors, [TESTING_GUIDE.md](TESTING_GUIDE.md) § Debugging |
| Understand accuracy / differences from BELLHOP | [doc/accuracy.md](doc/accuracy.md) |
| Build for specific platform or configuration | [doc/compilation.md](doc/compilation.md), [DOCUMENTATION.md](DOCUMENTATION.md) § Build System |

### By Topic

| Topic | Documentation |
|-------|---------------|
| **Installation** | [README.md](README.md), [doc/compilation.md](doc/compilation.md) |
| **Basic Usage** | [README.md](README.md), [QUICK_REFERENCE.md](QUICK_REFERENCE.md), [examples/](examples/) |
| **API Reference** | [DOCUMENTATION.md](DOCUMENTATION.md) § API Reference, [QUICK_REFERENCE.md](QUICK_REFERENCE.md) |
| **Core Concepts** | [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts |
| **Architecture** | [DOCUMENTATION.md](DOCUMENTATION.md) § Architecture, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) |
| **Data Structures** | [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Data Structures, [include/bhc/structs.hpp](include/bhc/structs.hpp) |
| **Ray Tracing** | [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Ray Tracing Engine |
| **SSP (Sound Speed)** | [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts § SSP, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § SSP Interpolation |
| **Boundaries** | [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts § Boundaries, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Boundary Handling |
| **Run Modes** | [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts § Run Types, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Mode System |
| **Performance** | [doc/performance.md](doc/performance.md), [DOCUMENTATION.md](DOCUMENTATION.md) § Advanced Topics |
| **Accuracy** | [doc/accuracy.md](doc/accuracy.md) |
| **Testing** | [TESTING_GUIDE.md](TESTING_GUIDE.md) |
| **CUDA/GPU** | [DOCUMENTATION.md](DOCUMENTATION.md) § Getting Started, [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § CUDA Implementation |
| **Build System** | [doc/compilation.md](doc/compilation.md), [DOCUMENTATION.md](DOCUMENTATION.md) § Build System |
| **Contributing** | [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Contributing Guidelines |

### By Experience Level

**Beginner** (New to BellhopCUDA/CXX):
1. [README.md](README.md) - Overview
2. [doc/compilation.md](doc/compilation.md) - Build it
3. [QUICK_REFERENCE.md](QUICK_REFERENCE.md) - Quick examples
4. [examples/defaults.cpp](examples/defaults.cpp) - Simple example
5. [doc/faq.md](doc/faq.md) - Common questions

**Intermediate** (Using the library):
1. [DOCUMENTATION.md](DOCUMENTATION.md) - Full user guide
2. [examples/](examples/) - All example programs
3. [QUICK_REFERENCE.md](QUICK_REFERENCE.md) - Quick API reference
4. [doc/performance.md](doc/performance.md) - Optimization
5. [TESTING_GUIDE.md](TESTING_GUIDE.md) - Validating results

**Advanced** (Developing / Contributing):
1. [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) - Architecture deep dive
2. [src/](src/) - Source code with comments
3. [TESTING_GUIDE.md](TESTING_GUIDE.md) - Testing framework
4. [doc/accuracy.md](doc/accuracy.md) - Numerical details
5. [Modified BELLHOP repo](https://github.com/A-New-BellHope/bellhop) - Reference implementation

## 📁 File Organization

```
bellhopcuda/
├── README.md                    # Project overview
├── DOCUMENTATION.md             # Complete user guide (★ START HERE)
├── DEVELOPER_GUIDE.md           # Architecture and internals
├── QUICK_REFERENCE.md           # Quick lookup reference
├── TESTING_GUIDE.md             # Testing documentation
│
├── doc/                         # Additional documentation
│   ├── compilation.md           # Build instructions
│   ├── accuracy.md              # Accuracy validation
│   ├── performance.md           # Performance benchmarks
│   └── faq.md                   # FAQ
│
├── include/bhc/                 # Public API headers
│   ├── bhc.hpp                  # Main API (heavily commented)
│   ├── structs.hpp              # Data structures (documented)
│   ├── math.hpp                 # Math utilities
│   └── platform.hpp             # Platform definitions
│
├── src/                         # Implementation (commented)
│   ├── api.cpp                  # API implementation
│   ├── module/                  # Parameter modules
│   ├── mode/                    # Run mode implementations
│   ├── *.hpp                    # Core algorithms
│   └── util/                    # Utilities
│
├── examples/                    # Example programs
│   ├── defaults.cpp             # Basic usage
│   ├── background.cpp           # Non-blocking execution
│   ├── province.cpp             # Custom SSP
│   └── ...
│
├── test/                        # Test environments
│   └── in/                      # Test .env files
│
└── config/                      # Build configuration
    └── CMakeLists.txt files
```

## 🔗 External Resources

- **Project Repository**: https://github.com/A-New-BellHope/bellhopcuda
- **Modified BELLHOP**: https://github.com/A-New-BellHope/bellhop (reference for validation)
- **Original BELLHOP**: http://oalib.hlsresearch.com/AcousticsToolbox/
- **CUDA Toolkit**: https://developer.nvidia.com/cuda-toolkit
- **GitHub Issues**: https://github.com/A-New-BellHope/bellhopcuda/issues
- **Contact**: Marine Physical Lab, jjaffe@ucsd.edu

## 🎓 Learning Path

### Path 1: Quick User (Just want to use it)

1. Build and run: [README.md](README.md) + [doc/compilation.md](doc/compilation.md) (20 min)
2. Try examples: [examples/](examples/) (30 min)
3. Use as needed: [QUICK_REFERENCE.md](QUICK_REFERENCE.md) for lookups

### Path 2: Power User (Integrate into project)

1. Quick start: [README.md](README.md) (10 min)
2. Understand concepts: [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts (1 hour)
3. Learn API: [DOCUMENTATION.md](DOCUMENTATION.md) § API Reference (2 hours)
4. Study examples: [examples/](examples/) (1 hour)
5. Optimize: [doc/performance.md](doc/performance.md) (30 min)
6. Reference: [QUICK_REFERENCE.md](QUICK_REFERENCE.md) for daily use

### Path 3: Developer (Contribute or extend)

1. Understand project: [DOCUMENTATION.md](DOCUMENTATION.md) (3 hours)
2. Learn architecture: [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) (4 hours)
3. Study source: [src/](src/) with guide (4 hours)
4. Learn testing: [TESTING_GUIDE.md](TESTING_GUIDE.md) (2 hours)
5. Read accuracy notes: [doc/accuracy.md](doc/accuracy.md) (1 hour)
6. Start contributing: [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) § Contributing

### Path 4: Researcher (Understand the physics)

1. Understand BELLHOP: Original BELLHOP documentation
2. Review ray theory: [DOCUMENTATION.md](DOCUMENTATION.md) § Core Concepts
3. Study accuracy: [doc/accuracy.md](doc/accuracy.md)
4. Run validations: [TESTING_GUIDE.md](TESTING_GUIDE.md)
5. Analyze differences: Modified BELLHOP documentation

## 💡 Tips for Reading

- **Start simple**: Don't try to read everything at once
- **Use the index**: Come back to find specific topics
- **Try examples**: Best way to learn the API
- **Check quick reference**: For common tasks while coding
- **Deep dive when needed**: Developer guide for implementation details

## 📝 Documentation Updates

This documentation is actively maintained. Last major update: January 2026.

If you find errors or areas needing clarification:
- Open an issue: https://github.com/A-New-BellHope/bellhopcuda/issues
- Submit a PR with improvements
- Contact us: jjaffe@ucsd.edu

## 🙏 Acknowledgments

- Original BELLHOP by Dr. Michael B. Porter
- Marine Physical Lab at Scripps Oceanography
- University of California
- All contributors and testers

---

**Happy coding! 🚀**
