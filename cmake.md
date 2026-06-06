# CMake CLI One-Liners

> Terminal commands only. No CMakeLists.txt. Copy-paste-run.
> Install: [scoop install cmake ninja](tools-install.md) (Windows) / `sudo apt install cmake ninja-build` (Linux) / `brew install cmake ninja` (macOS)

## Configure — the daily driver

```bash
# The one you'll type every day — debug + ninja + ccache + compile_commands
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

# Release with symbols (profiling + crash dumps)
cmake -B build-rel -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo

# Pick your compiler
cmake -B build -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
cmake -B build -DCMAKE_C_COMPILER=gcc-14 -DCMAKE_CXX_COMPILER=g++-14
cmake -B build -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl   # Windows clang

# C++ standard
cmake -B build -DCMAKE_CXX_STANDARD=20 -DCMAKE_CXX_STANDARD_REQUIRED=ON
cmake -B build -DCMAKE_CXX_STANDARD=23

# Sanitizers (clang/gcc)
cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"

# Thread sanitizer (separate build — conflicts with ASAN)
cmake -B build-tsan -DCMAKE_CXX_FLAGS="-fsanitize=thread" -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"

# Verbose — see exact compiler invocations
cmake -B build -DCMAKE_VERBOSE_MAKEFILE=ON

# Install prefix
cmake -B build -DCMAKE_INSTALL_PREFIX=/usr/local
cmake -B build -DCMAKE_INSTALL_PREFIX=./install
```

## Cross-platform configure

```bash
# Android NDK (install: scoop install android-clt / brew install --cask android-commandlinetools)
cmake -B build/android -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24

# iOS (macOS only, needs Xcode: xcode-select --install)
cmake -B build/ios -GXcode -DCMAKE_SYSTEM_NAME=iOS -DCMAKE_OSX_ARCHITECTURES=arm64

# macOS universal binary
cmake -B build -DCMAKE_OSX_ARCHITECTURES="arm64;x86_64"

# macOS minimum version
cmake -B build -DCMAKE_OSX_DEPLOYMENT_TARGET=14.0

# Zig cross-compile to Linux from Windows/macOS (install: scoop install zig / brew install zig)
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake -G Ninja

# Windows static CRT (no vcruntime.dll needed)
cmake -B build -DCMAKE_MSVC_RUNTIME_LIBRARY="MultiThreaded$<$<CONFIG:Debug>:Debug>"

# Windows PDB in release (crash dumps)
cmake -B build -DCMAKE_CXX_FLAGS_RELEASE="/Zi" -DCMAKE_EXE_LINKER_FLAGS_RELEASE="/DEBUG /OPT:REF /OPT:ICF"
```

## Build

```bash
# Build everything
cmake --build build

# Build specific target
cmake --build build --target myapp
cmake --build build --target mylib

# Clean rebuild one target
cmake --build build --target myapp --clean-first

# Parallel jobs
cmake --build build -j8                   # explicit
cmake --build build -j$(nproc)            # Linux: all cores
cmake --build build -j$(sysctl -n hw.ncpu)  # macOS: all cores
cmake --build build                        # Ninja auto-detects

# Verbose build (see compiler flags per file)
cmake --build build --verbose

# Build with specific config (multi-config generators like MSVC/Xcode)
cmake --build build --config Release
cmake --build build --config RelWithDebInfo

# Install
cmake --install build
cmake --install build --prefix ./install
cmake --install build --strip
cmake --install build --component Runtime
```

## Test

```bash
# Run all tests
ctest --test-dir build --output-on-failure

# Filter by name (regex)
ctest --test-dir build -R "unit" --output-on-failure
ctest --test-dir build -R "math|physics" --output-on-failure

# Exclude tests
ctest --test-dir build -E "slow|integration" --output-on-failure

# Parallel test execution
ctest --test-dir build -j$(nproc) --output-on-failure

# List all tests (don't run)
ctest --test-dir build -N

# Run specific test by index
ctest --test-dir build -I 3,3

# Run with timeout (kill stuck tests)
ctest --test-dir build --timeout 30 --output-on-failure

# Verbose output (see test stdout)
ctest --test-dir build --verbose --output-on-failure

# Repeat N times (find flaky tests)
ctest --test-dir build --repeat-until-fail 10
```

## Study a project — understand what's inside

```bash
# List ALL targets in the project (executables, libraries, custom targets)
cmake --build build --target help         # Ninja
cmake --build build --target help | rg -i "myapp|test|bench"  # filter

# List all cache variables (every -D option the project supports)
cmake -B build -LAH                        # LAH = List All Advanced with Help
cmake -B build -LAH | rg -i "feature|enable|option|with"  # find toggleable features

# System info (compiler, platform, paths)
cmake --system-info

# Generate dependency graph (needs graphviz: scoop install graphviz / apt install graphviz)
cmake -B build --graphviz=deps.dot
dot -Tpng deps.dot -o deps.png            # → visual dependency map
dot -Tsvg deps.dot -o deps.svg            # SVG for zooming

# Trace what CMake does during configure (finds bottlenecks, bad FindXXX)
cmake -B build --trace-expand 2>&1 | tee cmake-trace.log
cmake -B build --trace-expand --trace-source=CMakeLists.txt 2>&1 | tee trace.log

# Log-level for configure (see what find_package finds)
cmake -B build --log-level=VERBOSE
cmake -B build --log-level=DEBUG           # maximum output

# Print one specific variable
cmake -B build -LAH | rg CMAKE_INSTALL_PREFIX
cmake -B build -LAH | rg CMAKE_CXX_FLAGS

# What find_package found
cmake -B build --log-level=VERBOSE 2>&1 | rg "Found "
```

---

## 🔥 Pro Tricks

> **Did you know?** These are the CMake CLI tricks that separate juniors from seniors.

### Configure profiling — find slow FindXXX modules

```bash
# Time each configure step (CMake 3.28+)
cmake -B build --profiling-output=cmake-prof.json --profiling-format=google-trace
# Open cmake-prof.json at https://ui.perfetto.dev — see what's slow!

# Older CMake: use trace + timing
cmake -B build --trace-expand 2>&1 | awk '/^\/.*\.cmake/{print}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20
# → top 20 most-invoked .cmake files
```

### Export compile commands per-target

```bash
# compile_commands.json includes ALL targets. Filter to one:
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cat build/compile_commands.json | jq '.[] | select(.file | contains("renderer.cpp"))'

# Or use compdb tool:
pip install compdb
compdb -p build list                       # list all compilation units
compdb -p build list | rg renderer         # filter
```

### Compare two builds (diff compiler flags)

```bash
# Use process substitution ([shell-tricks.md](shell-tricks.md))
diff <(cmake -B build-a -LAH 2>/dev/null | sort) <(cmake -B build-b -LAH 2>/dev/null | sort)

# Compare compile commands between debug and release
diff <(jq -r '.[].command' build/compile_commands.json | sort) \
     <(jq -r '.[].command' build-rel/compile_commands.json | sort)
```

### Ninja build graph visualization

```bash
# See what Ninja will build and in what order
ninja -C build -t graph myapp | dot -Tpng -o build-graph.png
ninja -C build -t targets all | sort       # all targets
ninja -C build -t deps myapp               # dependencies of one target
ninja -C build -t commands myapp            # exact commands to build target
ninja -C build -t query myapp              # what depends on what
```

### Build timing — which .cpp files are slow to compile?

```bash
# Time each compilation (Ninja)
cmake --build build --verbose 2>&1 | ts -i '%.s' | tee build-timing.log

# Or: re-build with timing wrapper
cmake --build build -- -k0 2>&1 | while read line; do echo "$(date +%s.%N) $line"; done > timing.log

# Find the slowest files
cmake --build build --clean-first --verbose 2>&1 | \
  grep -oP '\[.*?\] Building .*? (.*?)$' | \
  awk '{print $NF}' | sort -t'.' -k2 -rn | head -20

# CMake 3.28+ built-in:
cmake -B build --profiling-output=prof.json --profiling-format=google-trace
# → open prof.json at https://ui.perfetto.dev
```

### Generator expressions from CLI

```bash
# Use $<CONFIG> for multi-config builds
cmake -B build -DCMAKE_CXX_FLAGS_RELEASE="-O3 -DNDEBUG"
cmake -B build -DCMAKE_CXX_FLAGS_DEBUG="-O0 -g -fsanitize=address"

# Conditional output directory
cmake -B build -DCMAKE_RUNTIME_OUTPUT_DIRECTORY="build/$<CONFIG>/bin"

# Per-config linker flags
cmake -B build \
  -DCMAKE_EXE_LINKER_FLAGS_DEBUG="-fsanitize=address" \
  -DCMAKE_EXE_LINKER_FLAGS_RELEASE="/DEBUG /OPT:REF /OPT:ICF"
```

### Quick reconfigure (without deleting build dir)

```bash
# Change one option on existing build
cmake -B build -DMY_FEATURE=ON            # just toggles one thing, fast

# Force re-detect of compiler
rm build/CMakeCache.txt
cmake -B build                             # reconfigure from scratch

# Check what changed between reconfigures
diff <(cmake -B build -LAH 2>/dev/null | sort) <(cmake -B build -LAH 2>/dev/null | sort)
```

### Package inspection from CLI

```bash
# vcpkg (install: scoop install vcpkg / brew install vcpkg)
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
vcpkg install fmt:x64-windows spdlog:x64-windows
vcpkg list                                 # what's installed
vcpkg search sqlite                        # find packages
vcpkg depend-info fmt                      # what does fmt depend on?
vcpkg x-history fmt --x-json               # version history

# Conan 2 (install: pip install conan)
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
conan profile detect                       # auto-detect compiler
conan list "*"                             # local cache
conan inspect fmt/11.0.2                   # package info
conan graph info .                         # dependency graph
```

### compile_commands.json

```bash
# Generate (add to any configure command)
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# Symlink to project root (clangd, clang-tidy, IDE need it here)
ln -sf build/compile_commands.json .                       # Linux/macOS
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build\compile_commands.json  # PowerShell
mklink compile_commands.json build\compile_commands.json   # Windows CMD
```

> Every tool that reads C++ needs this: [clangd](https://clangd.llvm.org), [clang-tidy](https://clang.llvm.org/extra/clang-tidy/), [compdb](https://github.com/Sarcasm/compdb)

## Presets — CLI usage

```bash
# List available presets
cmake --list-presets
cmake --list-presets build                 # build presets
cmake --list-presets test                  # test presets

# Configure with preset
cmake --preset dev
cmake --preset release
cmake --preset clang

# Build with preset
cmake --build --preset dev

# Test with preset
ctest --preset dev --output-on-failure

# Workflow preset (configure + build + test in one command, CMake 3.25+)
cmake --workflow --preset dev
```

> Presets defined in `CMakePresets.json` — see [Modern CMake presets guide](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)
> User overrides in `CMakeUserPresets.json` (gitignored)

## Unity builds

```bash
# Fast full rebuild (CI only — merges .cpp files)
cmake -B build -DCMAKE_UNITY_BUILD=ON -DCMAKE_UNITY_BUILD_BATCH_SIZE=16
cmake --build build

# Per-target (from CLI, override CMakeLists.txt)
cmake -B build -DCMAKE_UNITY_BUILD=ON -DCMAKE_UNITY_BUILD_BATCH_SIZE=8
```

> ⚠️ Unity hides ODR violations. Dev builds: use [ccache](build-acceleration.md) instead.

## Clean & reset

```bash
# Nuclear option
rm -rf build/

# Clean one preset
rm -rf build/dev/

# Reset cache (keep generated files)
rm build/CMakeCache.txt
cmake -B build                             # reconfigure from scratch

# Remove compile_commands symlink
rm compile_commands.json
```

## CMake version & info

```bash
cmake --version                            # e.g. 3.31.0
cmake --help                               # all options
cmake --help-variable CMAKE_BUILD_TYPE     # docs for one variable
cmake --help-property COMPILE_DEFINITIONS  # docs for one property
cmake --help-module FindPkgConfig          # docs for one module
cmake --help-command find_package          # docs for one command

# Compare CMake versions (if you have multiple)
cmake --help-module-list                   # all available modules
cmake --help-policy CMP0135                # specific policy docs
```

---

**Related**: [Project navigation](project-navigation.md) — git blame, history, diffs, AI tools for studying repos
**Related**: [Shell tricks](shell-tricks.md) — process substitution `<()`, pipes, parallel
**Related**: [Build acceleration](build-acceleration.md) — ccache, PCH, speedup table
**Related**: [Cross-compilation](cross-compilation.md) — Zig, Android NDK, iOS toolchains
**Related**: [Package managers](cmake-package-managers.md) — vcpkg, Conan, FetchContent deep dive
