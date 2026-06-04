# Cross-Compilation from Windows

## Zig as cross-compiler

Zig bundles clang + cross-compilation sysroots. No separate toolchains needed.

### Single file

```bash
# Compile C++ for Linux from Windows
zig c++ hello.cpp -target x86_64-linux-gnu -o hello_linux

# Compile for macOS (experimental)
zig c++ hello.cpp -target x86_64-macos-none -o hello_macos

# Compile for ARM Linux
zig c++ hello.cpp -target aarch64-linux-gnu -o hello_arm

# List all targets
zig targets
```

### CMake integration

```bash
# 1. Create cross-compilation preset (see cmake-presets.md)
cmake --preset linux-cross

# 2. Build
cmake --build --preset linux-cross

# 3. Test with QEMU
qemu-x86_64 ./build/linux-cross/myapp
```

### CMake toolchain file (alternative to presets)

```cmake
# zig-toolchain.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR x86_64)

set(CMAKE_C_COMPILER "zig cc")
set(CMAKE_CXX_COMPILER "zig c++")

set(CMAKE_C_COMPILER_TARGET x86_64-linux-gnu)
set(CMAKE_CXX_COMPILER_TARGET x86_64-linux-gnu)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

```bash
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake
cmake --build build/linux
```

### Zig build system (no CMake)

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "myapp",
        .target = target,
        .optimize = optimize,
    });
    exe.addCSourceFiles(&.{"src/main.cpp", "src/utils.cpp"}, &.{});
    exe.linkLibCpp();
    b.installArtifact(exe);
}
```

```bash
zig build -Dtarget=x86_64-linux-gnu
```

## QEMU for testing

```bash
# Install
scoop install qemu

# Run Linux binary on Windows
qemu-x86_64 ./build/linux-cross/myapp

# With arguments
qemu-x86_64 ./build/linux-cross/myapp --help

# With strace equivalent
qemu-x86_64 -strace ./build/linux-cross/myapp

# Run ARM binary
qemu-aarch64 ./build/arm-cross/myapp
```

## MSYS2 for POSIX emulation

MSYS2 provides a POSIX compatibility layer. Useful for building Linux-targeted code that uses POSIX APIs.

```bash
# MSYS2 shell (installed via scoop)
# Start from: C:\Users\YOU\scoop\apps\msys2\current\msys2_shell.cmd

# Install build tools
pacman -S mingw-w64-x86_64-toolchain
pacman -S mingw-w64-x86_64-cmake
pacman -S mingw-w64-x86_64-ninja

# Build in MSYS2
cmake -B build -G Ninja
cmake --build build
```

## Feature detection (instead of OS detection)

```cmake
# BAD — fragile platform checks
if(WIN32)
    target_sources(myapp PRIVATE win_specific.cpp)
elseif(APPLE)
    target_sources(myapp PRIVATE mac_specific.cpp)
else()
    target_sources(myapp PRIVATE linux_specific.cpp)
endif()

# GOOD — feature detection
include(CheckIncludeFile)
include(CheckFunctionExists)

check_include_file("sys/epoll.h" HAS_EPOLL)
check_include_file("windows.h" HAS_WINDOWS)
check_function_exists("mmap" HAS_MMAP)

if(HAS_EPOLL)
    target_compile_definitions(myapp PRIVATE USE_EPOLL)
elseif(HAS_WINDOWS)
    target_compile_definitions(myapp PRIVATE USE_IOCP)
endif()
```

## Android NDK (via Android Studio CLT)

```bash
# Already installed: android-clt + android-studio
# Path to NDK: %LOCALAPPDATA%\Android\Sdk\ndk\

# CMake toolchain for Android
cmake -B build/android \
  -DCMAKE_TOOLCHAIN_FILE="%LOCALAPPDATA%/Android/Sdk/ndk/27.0.12077973/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24

cmake --build build/android
```

## CI matrix with presets

```yaml
# .github/workflows/build.yml
name: Build
on: [push, pull_request]

jobs:
  matrix:
    strategy:
      matrix:
        preset: [win-msvc, win-clang, gcc, linux-cross]
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - run: cmake --preset ${{ matrix.preset }}
      - run: cmake --build --preset ${{ matrix.preset }}
      - run: ctest --preset ${{ matrix.preset }}
```
