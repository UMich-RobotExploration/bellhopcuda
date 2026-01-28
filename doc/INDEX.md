# BellhopCUDA Documentation Index

Complete guide to all documentation in the bellhopcuda repository.

## 📁 Documentation Structure

```
doc/
├── INDEX.md              ← You are here
├── OVERVIEW.md           ← Documentation summary
├── QUICK_START.md        ← 5-minute getting started guide
├── USER_GUIDE.md         ← Complete user guide & API reference
├── compilation.md        ← Build instructions
├── accuracy.md           ← Numerical accuracy discussion  
├── performance.md        ← Performance benchmarks
├── faq.md                ← Frequently asked questions
└── guides/
    ├── DEVELOPER_GUIDE.md    ← Architecture & internals
    └── TESTING_GUIDE.md      ← Testing workflows
```

## 🚀 Quick Navigation

### I want to...

| Goal | Document | Time |
|------|----------|------|
| Get started in 5 minutes | [QUICK_START.md](QUICK_START.md) | 5 min |
| Build the project | [compilation.md](compilation.md) | 10 min |
| Use the library API | [USER_GUIDE.md](USER_GUIDE.md) §6-7 | 20 min |
| Understand core concepts | [USER_GUIDE.md](USER_GUIDE.md) §4 | 30 min |
| See code examples | [USER_GUIDE.md](USER_GUIDE.md) §7 | 15 min |
| Understand template system | [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §2 | 20 min |
| Learn the architecture | [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §3-6 | 60 min |
| Run tests | [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §2 | 10 min |
| Debug failing rays | [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §4 | 30 min |
| Optimize performance | [performance.md](performance.md) | 15 min |
| Troubleshoot build issues | [faq.md](faq.md) | 5 min |
| Validate accuracy | [accuracy.md](accuracy.md) | 20 min |

## 📚 Document Summaries

### [QUICK_START.md](QUICK_START.md) (~4,000 words)
**Purpose:** Get running quickly  
**Audience:** First-time users  
**Content:**
- 5-minute quick start
- Common task recipes
- Template parameters guide
- RunType format reference
- Error solutions
- Performance tips

### [USER_GUIDE.md](USER_GUIDE.md) (~15,000 words)
**Purpose:** Complete user reference  
**Audience:** Library users  
**Content:**
- Introduction and overview
- Architecture explanation
- Getting started tutorial
- Core concepts (SSP, boundaries, ray theory)
- Complete API reference
- Build system guide
- Working examples
- Advanced topics
- Troubleshooting

### [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) (~12,000 words)
**Purpose:** Understand internals  
**Audience:** Contributors and advanced users  
**Content:**
- Code architecture
- Template metaprogramming system
- Data structures deep dive
- Module and mode systems
- Ray tracing engine internals
- SSP interpolation algorithms
- Influence functions
- Memory management
- CUDA implementation
- Contributing guidelines

### [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) (~6,000 words)
**Purpose:** Testing and validation  
**Audience:** Developers and testers  
**Content:**
- Test categories (coverage, OALIB, comparison)
- Running tests
- Writing new tests
- Debugging workflows
- Finding failing rays
- Performance benchmarking
- Validation methodology

### [compilation.md](compilation.md) (~250 words)
**Purpose:** Build instructions  
**Audience:** All users  
**Content:**
- CMake build steps
- CPU-only builds
- CUDA builds
- Platform support

### [accuracy.md](accuracy.md) (~1,000 words)
**Purpose:** Numerical precision  
**Audience:** Research users  
**Content:**
- Numerical stability improvements
- Test methodology
- Comparison with BELLHOP
- Coverage test results
- OALIB validation

### [performance.md](performance.md) (~900 words)
**Purpose:** Performance analysis  
**Audience:** Performance-conscious users  
**Content:**
- Speedup factors
- Ray count effects
- Receiver layout impact
- Run type performance
- Precision trade-offs

### [faq.md](faq.md) (~1,300 words)
**Purpose:** Common questions  
**Audience:** All users  
**Content:**
- Platform support
- MATLAB integration
- Library usage
- Build troubleshooting
- Bug reporting

### [OVERVIEW.md](OVERVIEW.md) (~1,000 words)
**Purpose:** Documentation summary  
**Audience:** Project maintainers  
**Content:**
- What was documented
- Coverage metrics
- Key features
- Value delivered

## 🎯 Learning Paths

### Beginner Path (Total: ~1 hour)
1. [QUICK_START.md](QUICK_START.md) - Get oriented (5 min)
2. [compilation.md](compilation.md) - Build the project (10 min)
3. [USER_GUIDE.md](USER_GUIDE.md) §1-3 - Understand basics (20 min)
4. [USER_GUIDE.md](USER_GUIDE.md) §7 - Try examples (15 min)
5. [faq.md](faq.md) - Common issues (10 min)

### Intermediate Path (Total: ~2 hours)
1. [USER_GUIDE.md](USER_GUIDE.md) §4-5 - Core concepts (45 min)
2. [USER_GUIDE.md](USER_GUIDE.md) §6 - Full API (30 min)
3. [performance.md](performance.md) - Optimization (15 min)
4. [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §2-3 - Testing basics (20 min)
5. [accuracy.md](accuracy.md) - Validation (10 min)

### Advanced/Contributor Path (Total: ~3 hours)
1. [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §1-2 - Architecture (30 min)
2. [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §3-5 - Data & systems (60 min)
3. [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §6-8 - Algorithms (45 min)
4. [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §4-6 - Advanced testing (30 min)
5. [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §11 - Contributing (15 min)

## 🔍 Topic Index

### API & Usage
- API Reference → [USER_GUIDE.md](USER_GUIDE.md) §6
- Library setup → [USER_GUIDE.md](USER_GUIDE.md) §6.1
- Running simulations → [USER_GUIDE.md](USER_GUIDE.md) §6.2
- Code examples → [USER_GUIDE.md](USER_GUIDE.md) §7
- Common tasks → [QUICK_START.md](QUICK_START.md) §2

### Building & Configuration
- Build instructions → [compilation.md](compilation.md)
- CMake options → [USER_GUIDE.md](USER_GUIDE.md) §5
- Template parameters → [QUICK_START.md](QUICK_START.md) §3
- Build troubleshooting → [faq.md](faq.md)

### Concepts & Theory
- SSP (Sound Speed Profile) → [USER_GUIDE.md](USER_GUIDE.md) §4.1
- Boundaries → [USER_GUIDE.md](USER_GUIDE.md) §4.2
- Ray theory → [USER_GUIDE.md](USER_GUIDE.md) §4.3
- Influence functions → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §7

### Architecture & Internals
- Code organization → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §1
- Template system → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §2
- Data structures → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §3
- Module system → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §4
- Mode system → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §5
- Ray tracing engine → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §6

### Testing & Validation
- Running tests → [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §2
- Writing tests → [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §3
- Debugging rays → [guides/TESTING_GUIDE.md](guides/TESTING_GUIDE.md) §4
- Accuracy validation → [accuracy.md](accuracy.md)
- Performance benchmarks → [performance.md](performance.md)

### Performance
- Speedup analysis → [performance.md](performance.md)
- Optimization tips → [QUICK_START.md](QUICK_START.md) §7
- CUDA implementation → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §9
- Memory management → [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §8

## 📞 Getting Help

- **Quick answers:** Check [faq.md](faq.md)
- **Build issues:** See [compilation.md](compilation.md) and [faq.md](faq.md)
- **API questions:** See [USER_GUIDE.md](USER_GUIDE.md) §6
- **Bug reports:** Follow [guides/DEVELOPER_GUIDE.md](guides/DEVELOPER_GUIDE.md) §11.6
- **GitHub Issues:** https://github.com/A-New-BellHope/bellhopcuda/issues

## 📊 Documentation Statistics

- **Total words:** ~40,000
- **Total documents:** 10 files
- **Coverage:** API, Architecture, Testing, Performance, Accuracy
- **Examples:** 20+ code samples
- **Time to read all:** ~8 hours
