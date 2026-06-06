# Cross-Compilation from Windows

## Zig — One-Command Cross-Compiler

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

## QEMU — Test Without VM

```bash
scoop install qemu     # Windows
sudo apt install qemu-user  # Linux
brew install qemu      # macOS

qemu-x86_64 ./app_linux
qemu-x86_64 -strace ./app_linux     # trace syscalls
qemu-aarch64 ./app_arm
```

## Android NDK

```bash
# Via android-studio or android-clt
# Install:
scoop install android-clt                    # Windows (scoop preferred)
brew install --cask android-commandlinetools # macOS
sudo apt install android-sdk                 # Linux

# Configure
cmake -B build/android \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24

cmake --build build/android

# Deploy + run
adb push build/android/myapp /data/local/tmp/
adb shell chmod +x /data/local/tmp/myapp
adb shell /data/local/tmp/myapp
```

## iOS

```bash
# macOS only — requires Xcode
cmake -B build/ios -GXcode \
  -DCMAKE_SYSTEM_NAME=iOS \
  -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=16.0

cmake --build build/ios

# Or xcodebuild directly
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -sdk iphoneos -destination 'generic/platform=iOS' build

# Deploy to device
xcrun devicectl device install app --device <device-id> build/ios/MyApp.app
```

## macOS Universal Binary

```bash
cmake -B build -DCMAKE_OSX_ARCHITECTURES="arm64;x86_64"
```

## MSYS2 (POSIX on Windows)

```bash
# Start MSYS2 shell
C:\Users\YOU\scoop\apps\msys2\current\msys2_shell.cmd

# Inside MSYS2:
pacman -S mingw-w64-x86_64-toolchain mingw-w64-x86_64-cmake mingw-w64-x86_64-ninja
cmake -B build -G Ninja && cmake --build build
```

## Feature Detection (over OS Detection)

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

## CI Matrix

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

> 💡 Zig + QEMU = full Linux dev loop on Windows. No VM, no WSL, no Docker.

→ **CMake one-liners**: [cmake.md](cmake.md) — configure/build/test shortcuts
→ **Mobile deploy**: [mobile.md](mobile.md) — ADB, devicectl, xcrun
→ **Install**: [tools-install.md](tools-install.md) — install zig, qemu, NDK
