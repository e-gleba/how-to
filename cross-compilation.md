# Cross-Compilation from Windows

## Zig — one-command cross-compiler

```bash
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o app_linux
zig c++ -target aarch64-linux-gnu -O2 main.cpp -o app_arm
zig c++ -target x86_64-macos-none -O2 main.cpp -o app_mac

zig targets   # list all supported targets
```

## CMake + Zig

```cmake
# zig-toolchain.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_C_COMPILER "zig cc")
set(CMAKE_CXX_COMPILER "zig c++")
set(CMAKE_C_COMPILER_TARGET x86_64-linux-gnu)
set(CMAKE_CXX_COMPILER_TARGET x86_64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

```bash
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake -G Ninja
cmake --build build/linux
```

## QEMU — test without VM

```bash
scoop install qemu

qemu-x86_64 ./app_linux
qemu-x86_64 -strace ./app_linux     # trace syscalls
qemu-aarch64 ./app_arm
```

## Android NDK

```bash
# Via android-studio + android-clt (already installed)
cmake -B build/android \
  -DCMAKE_TOOLCHAIN_FILE="$env:LOCALAPPDATA/Android/Sdk/ndk/27.0.12077973/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24

cmake --build build/android
```

## MSYS2 (POSIX on Windows)

```bash
# Start MSYS2 shell
C:\Users\YOU\scoop\apps\msys2\current\msys2_shell.cmd

# Inside MSYS2:
pacman -S mingw-w64-x86_64-toolchain mingw-w64-x86_64-cmake mingw-w64-x86_64-ninja
cmake -B build -G Ninja && cmake --build build
```

## Feature detection (over OS detection)

```cmake
# BAD
if(WIN32) ... elseif(APPLE) ... else() ... endif()

# GOOD
include(CheckIncludeFile)
check_include_file("sys/epoll.h" HAS_EPOLL)
if(HAS_EPOLL)
    target_compile_definitions(myapp PRIVATE USE_EPOLL)
endif()
```

## CI matrix

```yaml
# .github/workflows/build.yml
strategy:
  matrix:
    preset: [win-msvc, win-clang, gcc, linux-cross]
steps:
  - run: cmake --preset ${{ matrix.preset }}
  - run: cmake --build --preset ${{ matrix.preset }}
  - run: ctest --preset ${{ matrix.preset }}
```

> 💡 **Tip:** Zig + QEMU = full Linux dev loop on Windows. No VM, no WSL, no Docker.
