# CMake One-Liners Cookbook

> Daily-use CMake commands. Copy, paste, adapt. Cross-platform.

## Configure

```bash
# Basic configure — Debug build with Ninja
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug

# Release with symbols (best for profiling)
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo

# Export compile_commands.json (needed by clangd, clang-tidy, IDE)
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# Enable compiler cache (ccache/sccache) — 5-10× faster rebuilds
cmake -B build -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

# Pick compiler explicitly
cmake -B build -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
cmake -B build -DCMAKE_C_COMPILER=gcc-14 -DCMAKE_CXX_COMPILER=g++-14

# Set C++ standard
cmake -B build -DCMAKE_CXX_STANDARD=20 -DCMAKE_CXX_STANDARD_REQUIRED=ON

# Enable sanitizers (clang/gcc)
cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" \
               -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"

# Static CRT on MSVC (no vcruntime.dll dependency)
cmake -B build -DCMAKE_MSVC_RUNTIME_LIBRARY="MultiThreaded$<$<CONFIG:Debug>:Debug>"

# PDB in release (Windows crash dumps)
cmake -B build -DCMAKE_CXX_FLAGS_RELEASE="/Zi" \
               -DCMAKE_EXE_LINKER_FLAGS_RELEASE="/DEBUG /OPT:REF /OPT:ICF"

# Cross-compile with toolchain file
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake

# Install prefix
cmake -B build -DCMAKE_INSTALL_PREFIX=./install

# Android NDK
cmake -B build/android \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24

# iOS
cmake -B build/ios -GXcode \
  -DCMAKE_SYSTEM_NAME=iOS -DCMAKE_OSX_ARCHITECTURES=arm64

# macOS universal binary
cmake -B build -DCMAKE_OSX_ARCHITECTURES="arm64;x86_64"

# Unity build (fast full rebuild, CI only)
cmake -B build -DCMAKE_UNITY_BUILD=ON -DCMAKE_UNITY_BUILD_BATCH_SIZE=16

# Verbose build output (see exact compiler commands)
cmake -B build -DCMAKE_VERBOSE_MAKEFILE=ON
```

## Build

```bash
# Build everything
cmake --build build

# Build specific target
cmake --build build --target myapp

# Parallel jobs (Ninja auto-detects, Make needs -j)
cmake --build build -j$(nproc)          # Linux
cmake --build build -j$(sysctl -n hw.ncpu)  # macOS
cmake --build build -j%NUMBER_OF_PROCESSORS%  # Windows CMD
cmake --build build                      # Ninja: no -j needed

# Clean rebuild of one target
cmake --build build --target myapp --clean-first

# Build with verbose output
cmake --build build --verbose

# Install
cmake --install build
cmake --install build --prefix ./install
cmake --install build --component Runtime
```

## Test

```bash
# Run all tests
cd build && ctest --output-on-failure

# Run tests matching pattern
ctest -R "unit" --output-on-failure

# Run tests in parallel
ctest -j$(nproc) --output-on-failure

# List available tests
ctest --test-dir build -N

# Run specific test by number
ctest --test-dir build -I 3,3
```

## Inspect

```bash
# List all cache variables
cmake -B build -LAH

# Print system info
cmake --system-info

# Generate dependency graph
cmake -B build --graphviz=deps.dot && dot -Tpng deps.dot -o deps.png

# List all targets
cmake --build build --target help

# Print all properties of a target (from CMakeLists.txt)
# get_target_property(ALL_PROPS myapp PROPERTY)
```

## compile_commands.json

> **Why?** Every tool (clangd, clang-tidy, IDE completion) needs this file.

```bash
# Generate (add to configure step)
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# Symlink to project root
ln -sf build/compile_commands.json .              # Linux/macOS
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build\compile_commands.json  # PowerShell
mklink compile_commands.json build\compile_commands.json  # CMD (no admin)
```

## Presets (the good way)

> Presets save typing. Define once, use forever.

```bash
# List available presets
cmake --list-presets

# Configure + build + test with preset
cmake --preset dev
cmake --build --preset dev
ctest --preset dev --output-on-failure
```

Minimal `CMakePresets.json`:
```json
{
  "version": 6,
  "configurePresets": [{
    "name": "dev",
    "generator": "Ninja",
    "binaryDir": "${sourceDir}/build/${presetName}",
    "cacheVariables": {
      "CMAKE_BUILD_TYPE": "Debug",
      "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
      "CMAKE_C_COMPILER_LAUNCHER": "ccache",
      "CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
    }
  }],
  "buildPresets": [{"name": "dev", "configurePreset": "dev"}],
  "testPresets": [{"name": "dev", "configurePreset": "dev", "output": {"outputOnFailure": true}}]
}
```

User overrides → `CMakeUserPresets.json` (gitignored):
```json
{
  "version": 6,
  "configurePresets": [{
    "name": "my-debug",
    "inherits": "dev",
    "cacheVariables": { "MY_FEATURE": "ON" }
  }]
}
```

## Useful CMake Variables Quick Reference

| Variable | Value | Effect |
|----------|-------|--------|
| `CMAKE_BUILD_TYPE` | `Debug` / `Release` / `RelWithDebInfo` / `MinSizeRel` | Build configuration |
| `CMAKE_CXX_STANDARD` | `17` / `20` / `23` | C++ standard version |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | `ON` | Generate compile_commands.json |
| `CMAKE_C_COMPILER_LAUNCHER` | `ccache` | Compiler cache for faster rebuilds |
| `CMAKE_VERBOSE_MAKEFILE` | `ON` | Show full compiler commands |
| `CMAKE_UNITY_BUILD` | `ON` | Combine .cpp files (fast full rebuild) |
| `CMAKE_INSTALL_PREFIX` | `/usr/local` or `./install` | Install destination |
| `CMAKE_MSVC_RUNTIME_LIBRARY` | `MultiThreaded` / `MultiThreadedDLL` | Static/dynamic CRT |
| `CMAKE_OSX_ARCHITECTURES` | `arm64;x86_64` | macOS universal binary |
| `CMAKE_OSX_DEPLOYMENT_TARGET` | `14.0` | Minimum macOS version |
| `CMAKE_ANDROID_ARCH_ABI` | `arm64-v8a` | Android target arch |

## Common Patterns

```cmake
# FetchContent — grab a library from GitHub
include(FetchContent)
FetchContent_Declare(fmt GIT_REPOSITORY https://github.com/fmtlib/fmt.git GIT_TAG 11.0.2)
FetchContent_MakeAvailable(fmt)
target_link_libraries(myapp PRIVATE fmt::fmt)

# Precompiled headers — speed up compilation
target_precompile_headers(myapp PRIVATE <vector> <string> <memory> "pch.hpp")

# Feature detection (better than OS detection)
include(CheckIncludeFile)
check_include_file("sys/epoll.h" HAS_EPOLL)
if(HAS_EPOLL)
    target_compile_definitions(myapp PRIVATE USE_EPOLL)
endif()

# Set warnings
target_compile_options(myapp PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /WX>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall -Wextra -Wpedantic -Werror>
)
```

---

→ **Install**: [tools-install.md](tools-install.md) — how to install cmake + ninja + ccache
→ **Build speed**: [build-acceleration.md](build-acceleration.md) — ccache, PCH, unity builds
→ **Cross-compile**: [cross-compilation.md](cross-compilation.md) — Zig, Android, iOS toolchains
→ **Packages**: [cmake-package-managers.md](cmake-package-managers.md) — vcpkg, Conan, FetchContent
