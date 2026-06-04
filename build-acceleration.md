# Build Acceleration

## ccache — compiler cache

```bash
# Install
scoop install ccache

# Check cache stats
ccache -s

# Clear cache
ccache -C

# Set cache size (default 5GB)
ccache --max-size 10G

# Show cache dir
ccache --show-config | rg cache_dir
```

### CMake integration

```cmake
# In CMakePresets.json cacheVariables:
"CMAKE_C_COMPILER_LAUNCHER": "ccache",
"CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
```

Or per-platform:

```bash
# Manual invocation (no CMake integration)
ccache g++ -c main.cpp -o main.o
```

### Verify it's working

```bash
# After first build:
ccache -s
# cache hit (direct) — should be 0

# After second build (no changes):
ccache -s
# cache hit (direct) — should be high
```

## sccache — distributed compiler cache

Like ccache but adds cloud storage backends (S3, GCS, Azure) and Rust support.

```bash
# Install
scoop install sccache

# Use as ccache replacement
sccache --start-server

# Same CMake variables as ccache:
# CMAKE_C_COMPILER_LAUNCHER=sccache
# CMAKE_CXX_COMPILER_LAUNCHER=sccache
```

## Ninja — fast build executor

```bash
# Install
scoop install ninja

# Use with CMake
cmake -B build -G Ninja
cmake --build build

# Ninja is default in all presets above
```

Why Ninja over Make:
- Better parallel job scheduling
- Faster dependency resolution
- No recursive make problems
- Can output `compile_commands.json` natively

**Tip:** Never use `-j` with Ninja — it auto-detects core count. CMake adds `-j` automatically.

## Unity builds (batch compilation)

For CI or when iterating on logic (not headers):

```cmake
# CMakeLists.txt
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16)  # 16 .cpp → 1 unity file
```

Benchmark: full rebuild of ~500 .cpp files:
- Without unity: 3:20
- With unity (batch=16): 0:45

Downside: ODR violations hidden, macros leak across files. Only for CI/release builds.

## precompiled headers

```cmake
# CMakeLists.txt
target_precompile_headers(myapp PRIVATE
    <vector>
    <string>
    <memory>
    <functional>
    "pch.hpp"
)
```

Especially effective with MSVC. Combine with ccache for ninja-level rebuild speeds.

## Measure your build

```bash
# Time a build
hyperfine --prepare "touch src/main.cpp" "cmake --build build" --warmup 2

# Profile build timing (GCC/Clang)
cmake --build build -- -d keeprsp
# Then inspect .rsp files to see full command lines
```

## Results to expect

| Technique | Cold build | Hot rebuild (1 file) |
|-----------|-----------|----------------------|
| Nothing | 100% (baseline) | 100% (baseline) |
| Ninja | ~80% | ~70% |
| + ccache | ~80% | **~15%** |
| + pch | ~60% | ~50% |
| + unity | ~25% | n/a |

Combine Ninja + ccache + pch for daily dev. Unity for CI.
