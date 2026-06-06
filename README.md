# how-to

> **C++ Cross-Platform Dev Cookbook** — one-liners, tools, and patterns for game dev, research, and reverse engineering.
>
> Install once. Use everywhere. Copy-paste friendly.

---

## 📋 Index

| # | Topic | What | Deep Dive |
|---|-------|------|----------|
| 0 | [Install](#0-install-tools) | One-command install per platform | [tools-install.md](tools-install.md) |
| 1 | [CMake](#1-cmake) | Configure, build, one-liners | [cmake.md](cmake.md) |
| 2 | [Build Speed](#2-build-speed) | ccache, Ninja, PCH, unity | [build-acceleration.md](build-acceleration.md) |
| 3 | [Search](#3-search) | ripgrep, fd, fzf, bat | [search-navigation.md](search-navigation.md) |
| 4 | [Static Analysis](#4-static-analysis) | cppcheck, clang-tidy, ast-grep | [static-analysis.md](static-analysis.md) |
| 5 | [Debug & Profile](#5-debug--profile) | GDB, LLDB, Tracy, sanitizers | [debugging-profiling.md](debugging-profiling.md) |
| 6 | [Cross-Compile](#6-cross-compile) | Zig, Android, iOS | [cross-compilation.md](cross-compilation.md) |
| 7 | [Packages](#7-packages) | vcpkg, Conan, FetchContent | [cmake-package-managers.md](cmake-package-managers.md) |
| 8 | [Task Runner](#8-task-runner) | just — never type cmake again | [justfile.md](justfile.md) |
| 9 | [Benchmarking](#9-benchmarking) | hyperfine, Google Benchmark | [benchmarking.md](benchmarking.md) |
| 10 | [File Tracking](#10-file-tracking) | What did that command produce? | ↓ |
| 11 | [Mobile Dev](#11-mobile-dev) | ADB, root, Xcode, devicectl | [mobile.md](mobile.md) |
| 12 | [Reverse Engineering](#12-reverse-engineering) | Ghidra, r2, x64dbg, WinDbg | [reverse-engineering.md](reverse-engineering.md) |
| 13 | [Binary Tools](#13-binary-tools) | UPX, hex, PE, shipping | [binary-tools.md](binary-tools.md) |
| 14 | [Controls & Input](#14-controls--input) | Steam Golden Rules, gamepad | [controls.md](controls.md) |
| 15 | [Windows](#15-windows-cpp) | MSVC, CRT, PDB, Defender | [windows-cpp.md](windows-cpp.md) |
| 16 | [Resources](#16-resources) | Books, talks, channels | [resources.md](resources.md) |

---

## 0. Install Tools

> **Scoop preferred on Windows.** Full table: [tools-install.md](tools-install.md)

```bash
# 🪟 Windows (scoop)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
scoop install git cmake ninja ccache ripgrep fd bat fzf just llvm zig

# 🐧 Linux (Ubuntu/Debian)
sudo apt update && sudo apt install -y \
  build-essential cmake ninja-build ccache gdb lldb clang clang-tidy \
  ripgrep fd-find bat fzf strace

# 🍎 macOS
xcode-select --install
brew install cmake ninja ccache ripgrep fd bat fzf just
```

---

## 1. CMake

> Full cookbook: [cmake.md](cmake.md)

```bash
# Configure (debug, with compile_commands.json, with ccache)
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

# Build
cmake --build build

# Build target
cmake --build build --target myapp

# Test
cd build && ctest --output-on-failure

# Symlink compile_commands.json
ln -sf build/compile_commands.json .        # Linux/macOS
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build\compile_commands.json  # PS

# Presets (the good way)
cmake --preset dev && cmake --build --preset dev && ctest --preset dev

# Cross-compile
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake

# Android
cmake -B build/android -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24

# iOS
cmake -B build/ios -GXcode -DCMAKE_SYSTEM_NAME=iOS -DCMAKE_OSX_ARCHITECTURES=arm64
```

---

## 2. Build Speed

> Full reference: [build-acceleration.md](build-acceleration.md)

```bash
# ccache — 5-10× faster rebuilds
ccache -s                    # stats
ccache -C                    # clear

# In CMake:
# -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

# Precompiled headers (in CMakeLists.txt):
# target_precompile_headers(myapp PRIVATE <vector> <string> <memory>)

# Unity builds (CI only!):
# cmake -B build -DCMAKE_UNITY_BUILD=ON
```

| Technique | Cold build | Hot rebuild |
|-----------|-----------|-------------|
| + Ninja | ~80% | ~70% |
| + ccache | ~80% | **~15%** |
| + PCH | ~60% | ~50% |
| + Unity | ~25% | n/a |

---

## 3. Search

> Full reference: [search-navigation.md](search-navigation.md)

```bash
# ripgrep — search code
rg "pattern" --type cpp -n -C 3         # with context
rg "TODO|FIXME|HACK" --type cpp         # todos
rg "new\s+\w+" --type cpp               # raw owning pointers
rg "^#include" --type cpp --no-filename | sort -u  # all includes

# fd — find files
fd -e cpp -e hpp                         # all C++ files
fd --changed-within 1day                 # modified today

# fzf — fuzzy find
fd -e cpp | fzf --preview 'bat {}'       # fuzzy open with preview
git branch | fzf | xargs git checkout    # fuzzy branch switch

# bat — cat with syntax
bat src/main.cpp --diff                   # show git changes
```

---

## 4. Static Analysis

> Full reference: [static-analysis.md](static-analysis.md)

```bash
# cppcheck
cppcheck --enable=all --suppress=missingIncludeSystem src/

# clang-tidy (needs compile_commands.json)
clang-tidy -p build src/main.cpp
clang-tidy -p build --fix src/main.cpp

# clang-format
clang-format -style=llvm -dump-config > .clang-format
clang-format -i src/**/*.cpp src/**/*.hpp

# ast-grep — structural search
sg -p 'new $$$' --lang cpp               # all raw new
sg -p 'NULL' -r 'nullptr' --lang cpp -i  # replace NULLs
```

---

## 5. Debug & Profile

> Full reference: [debugging-profiling.md](debugging-profiling.md)

### GDB
```bash
gdb -tui ./myapp
# b main | r | n | s | c | p var | bt | bt full | watch var | layout split
```

### LLDB
```bash
lldb ./myapp
# b main | run | next | step | continue | print var | po obj | bt | frame variable
```

### Sanitizers
```cmake
# In CMake:
-DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
-DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
```

### Tracy (always-on profiler)
```cpp
#include <Tracy.hpp>
void heavy() { ZoneScoped; TracyPlot("dt", dt); FrameMark; }
```

```bash
tracy-profiler & ./build/dev/myapp
```

### Advanced debugging
- **LLDB** — iOS/macOS/NDK debugging → [mobile.md](mobile.md)
- **WinDbg** — kernel debugging, crash dumps → [reverse-engineering.md](reverse-engineering.md)
- **raddebugger** — fast native Windows debugger
- **x64dbg** — runtime debugger for stripped binaries

---

## 6. Cross-Compile

> Full reference: [cross-compilation.md](cross-compilation.md)

```bash
# Zig — one-command cross-compile
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux
zig c++ -target aarch64-linux-gnu -O2 main.cpp -o main_arm
zig c++ -target x86_64-macos-none -O2 main.cpp -o main_mac

# Test with QEMU
qemu-x86_64 ./main_linux
qemu-x86_64 -strace ./main_linux         # trace syscalls

# CMake + Zig toolchain
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake -G Ninja
```

---

## 7. Packages

> Full reference: [cmake-package-managers.md](cmake-package-managers.md)

```cmake
# FetchContent (simplest)
include(FetchContent)
FetchContent_Declare(fmt GIT_REPOSITORY https://github.com/fmtlib/fmt.git GIT_TAG 11.0.2)
FetchContent_MakeAvailable(fmt)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

```bash
# vcpkg (manifest mode)
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"

# Conan 2
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
```

---

## 8. Task Runner

> Full reference: [justfile.md](justfile.md)

```makefile
# justfile
default:
    @just --list

build preset="dev":
    cmake --preset {{preset}}
    cmake --build --preset {{preset}}

test preset="dev":
    ctest --preset {{preset}} --output-on-failure

watch preset="dev":
    watchexec -e cpp,hpp,h,c,cmake,just -- just build {{preset}}

lint:
    cppcheck --enable=all --suppress=missingIncludeSystem src/

format:
    clang-format -i src/**/*.cpp src/**/*.hpp

todos:
    rg "TODO|FIXME|HACK|XXX" --type cpp -n

bench app="myapp" preset="dev":
    hyperfine --warmup 5 --min-runs 20 "./build/{{preset}}/{{app}}"
```

```bash
just build preset=clang
just watch
just todos
```

---

## 9. Benchmarking

> Full reference: [benchmarking.md](benchmarking.md)

```bash
# hyperfine — CLI benchmarking
hyperfine --warmup 5 --min-runs 20 "./myapp"
hyperfine "./old" "./new"                              # compare
hyperfine -P size 1024 65536 "./myapp --buf {size}"    # parameter sweep

# Compilation benchmark
hyperfine --prepare 'rm -rf build && cmake -B build -G Ninja' 'cmake --build build'
```

---

## 10. File Tracking

> What files did that command/exe actually produce?

### strace / dtrace — trace file operations

```bash
# Linux — trace all file opens/writes
strace -f -e trace=open,openat,write -o trace.log ./myapp

# Linux — just files opened for writing
strace -f -e trace=open,openat ./myapp 2>&1 | grep "O_WRONLY\|O_RDWR\|O_CREAT"

# macOS — trace file operations
dtrace -qn 'syscall::open*:entry { printf("%s", copyinstr(arg0)); }' -c "./myapp"

# macOS — fs_usage (built-in, comprehensive)
fs_usage -f filesys ./myapp
```

### Windows — procmon + drstrace

```powershell
# procmon — GUI file tracking
procmon /Quiet /Minimized /BackingFile trace.pml /AcceptEula
./myapp.exe
procmon /Terminate
procmon /OpenLog trace.pml /SaveAs trace.csv

# drstrace — CLI syscall tracing
scoop install drmemory
drstrace -- myapp.exe                  # → drstrace.myapp.exe.PID.log
drstrace -logdir ./traces -- myapp.exe
```

### QEMU strace — cross-platform syscall tracing

```bash
qemu-x86_64 -strace ./myapp_linux 2>&1 | rg "openat|creat|write"
qemu-aarch64 -strace ./myapp_arm 2>&1 | rg "openat|creat"
```

### Track produced files — practical patterns

```bash
# What files does cmake --build produce?
strace -f -e trace=openat cmake --build build 2>&1 | \
  grep -oP '"[^"]*"' | sort -u | grep -v "^\"/usr" | grep -v "^\"/proc"

# What files does my app create at runtime?
strace -f -e trace=openat,creat ./myapp 2>&1 | grep "O_CREAT" | \
  grep -oP '"[^"]*"' | sort -u

# Compare filesystem before/after
find . -newer marker_file -type f       # files newer than marker
touch marker && ./myapp && find . -newer marker -type f

# What DLLs does my app load at runtime?
handle -p myapp.exe -a | rg "File"
listdlls myapp.exe

# What network connections does it make?
handle -a | rg "Socket|Tcp|Udp"
# Or procmon with Operation = TCP Connect
```

### Output file tracking table

| Command | Produces | Where |
|---------|----------|-------|
| `cmake -B build` | `build/CMakeCache.txt`, `build/build.ninja` | `build/` |
| `cmake --build build` | `.o`, `.a`, `.exe`, `.so`, `.dylib` | `build/` |
| `clang-tidy --fix` | Modified `.cpp` files (in-place) | source tree |
| `clang-format -i` | Modified `.cpp` files (in-place) | source tree |
| `upx --best app` | Compressed `app` (in-place) | same path |
| `strip app` | Stripped `app` (in-place) | same path |
| `rgp capture` | `*.rgp` GPU trace file | specified path |
| `tracy-profiler` | `*.tracy` profiling capture | current dir |
| `hyperfine --export-markdown` | `results.md` | current dir |
| `doxygen` | `html/`, `latex/` | `docs/` or configured |

---

## 11. Mobile Dev

> Full reference: [mobile.md](mobile.md)

### Android

```bash
# Device management
adb devices -l                    # list devices
adb connect 192.168.1.100:5555    # wireless
adb install -r app.apk            # install/reinstall

# Logs
adb logcat -s MyTag *:E           # filtered errors
adb logcat --pid=$(adb shell pidof com.example.app)

# Root
adb root                          # restart as root
adb shell setenforce 0            # disable SELinux
adb shell getenforce              # check status

# Google APIs image (no Play Services)
sdkmanager "system-images;android-34;google_apis;arm64-v8a"
emulator -avd pixel_test -writable-system -no-snapshot
```

### iOS

```bash
# Simulator
xcrun simctl list devices
xcrun simctl boot "iPhone 15"
xcrun simctl install booted MyApp.app
xcrun simctl io booted screenshot screen.png

# Physical device (Xcode 15+)
xcrun devicectl list devices
xcrun devicectl device install app --device <id> MyApp.app

# Build
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -sdk iphonesimulator build

# LLDB remote debug
lldb -n MyApp
```

---

## 12. Reverse Engineering

> Full reference: [reverse-engineering.md](reverse-engineering.md)

```bash
# Quick recon
file myapp                         # file type
checksec --file=myapp              # security flags
strings myapp | rg -i "password|secret|key|token"

# Ghidra — decompile
ghidra                             # launch GUI
# F5 = decompile, L = rename, X = cross-refs, G = go to address

# radare2
r2 -A myapp                        # open + analyze
# afl = functions, iz = strings, pdf = disasm, VV = graph, s main = seek

# x64dbg (Windows)
x64dbg myapp.exe
# F2 = breakpoint, F7 = step in, F8 = step over, F9 = run

# WinDbg (kernel)
windbg -z crash.dmp
# !analyze -v | kb | .sympath srv*https://msdl.microsoft.com/download/symbols

# Binary patching (r2)
oo+                               # write mode
wa nop @ 0x401000                 # NOP out instruction
```

---

## 13. Binary Tools

> Full reference: [binary-tools.md](binary-tools.md)

```bash
# Compress
strip myapp.exe && upx --best myapp.exe

# DLL dependencies
depends myapp.exe                  # Windows
ldd myapp                          # Linux
otool -L myapp                     # macOS

# PE/ELF inspection
pe-bear myapp.exe                  # Windows PE viewer
readelf -h myapp                   # ELF header
objdump -h myapp.exe               # PE sections

# Strings
strings myapp.exe | rg -i "password|secret|pdb"

# Pre-ship checklist
checksec --file=myapp              # security flags
strings myapp | rg "\.pdb"         # debug symbols leaked?
strings myapp | rg -i "password"   # secrets?
depends myapp.exe                  # missing DLLs?
```

---

## 14. Controls & Input

> Full reference: [controls.md](controls.md)

### Steam's 5 Golden Rules

1. **On-screen icons match device** — detect controller type, swap glyphs
2. **Cursor matches device** — visible for mouse, hidden for gamepad
3. **All devices work 100%** — test disconnect/reconnect, Remote Play
4. **All inputs navigate menus** — d-pad, stick, mouse
5. **Disconnect → pause** — single player pause, multiplayer mark disconnected

**Bonus**: Gamepad + mouse must work **simultaneously**.

```cpp
// Steam Input API
SteamInput()->Init();
SteamInput()->RunFrame();
InputHandle_t ctrl = SteamInput()->GetControllerForGamepadIndex(0);
EInputActionOrigin origin = SteamInput()->GetActionOriginFromXboxOrigin(ctrl, k_EXboxOrigin_A);
const char* glyph = SteamInput()->GetGlyphForActionOrigin(origin);
// Load glyph as texture → future-proof for new controllers
```

> **Critical**: Camera/aim action type MUST be `absolute_mouse` in IGA file. Steam Input converts gyro/trackpad/stick → absolute_mouse. Cannot convert back from `joystick_move`.

---

## 15. Windows C++

> Full reference: [windows-cpp.md](windows-cpp.md)

```cmake
# Static CRT (no DLL dependency)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# PDB in release (crash dumps)
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
```

```powershell
# Defender exclusion (admin)
Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"

# Long paths (admin)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

| Compiler | Best for |
|----------|----------|
| **MSVC** (`cl`) | Production, best PDB |
| **Clang-cl** | Sanitizers, clang-tidy, cross-platform |
| **GCC** (MSYS2) | POSIX APIs, autotools |

---

## 16. Resources

> Full reference: [resources.md](resources.md)

### Must-Read Books

| Book | Author | Focus |
|------|--------|-------|
| **Professional CMake** | Craig Scott | CMake definitive guide |
| **C++ Best Practices** | Jason Turner | 45+ actionable rules |
| **Game Engine Architecture** | Jason Gregory | Engine design (Naughty Dog) |
| **Practical Reverse Engineering** | Dang et al. | RE for x86/ARM/Windows |
| **Optimized C++** | Kurt Guntheroth | Performance |

### Must-Watch

| Channel/Talk | Focus |
|--------------|-------|
| **C++ Weekly** (Jason Turner) | 600+ weekly C++ episodes |
| **The Cherno** | Game engine dev, C++ deep dives |
| **CppCon** | Annual conference, 300+ talks |
| **LiveOverflow** | Binary exploitation, RE |
| **GDC** | Game dev conference talks |

### Essential Online

| Resource | Link |
|----------|------|
| **cppreference.com** | [en.cppreference.com](https://en.cppreference.com) |
| **Compiler Explorer** | [godbolt.org](https://godbolt.org) |
| **C++ Core Guidelines** | [isocpp.github.io/CppCoreGuidelines](https://isocpp.github.io/CppCoreGuidelines/) |
| **Modern CMake** | [cliutils.gitlab.io/modern-cmake](https://cliutils.gitlab.io/modern-cmake/) |
| **Steamworks Docs** | [partner.steamgames.com/doc](https://partner.steamgames.com/doc) |

---

## ⚡ Quick Reference Card

```
INSTALL:   scoop install ripgrep fd bat fzf just cmake ninja ccache llvm zig
BUILD:     cmake -B build -G Ninja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON && cmake --build build
WATCH:     watchexec -e cpp,hpp -- cmake --build build
SEARCH:    rg "pattern" --type cpp -n -C 3
FIND:      fd -e cpp -e hpp
TODOS:     rg "TODO|FIXME|HACK" --type cpp -C 2
BENCH:     hyperfine --warmup 5 --min-runs 20 "./myapp"
LINT:      cppcheck --enable=all --suppress=missingIncludeSystem src/
FORMAT:    clang-format -i src/**/*.cpp src/**/*.hpp
PROFILE:   tracy-profiler & ./myapp
DEBUG:     gdb -tui ./myapp  |  lldb ./myapp
CACHE:     ccache -s
COMPILEDB: cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON && ln -sf build/compile_commands.json .
TRACE:     strace -f -e trace=openat ./myapp  |  procmon
STRACE:    drstrace -- myapp.exe  |  qemu-x86_64 -strace ./myapp
TRACK:     touch marker && ./myapp && find . -newer marker -type f
GPU:       rgp capture --output frames.rgp --app myapp --frame 42
ANDROID:   adb devices -l && adb install -r app.apk && adb logcat -s Tag
IOS:       xcrun simctl boot "iPhone 15" && xcrun simctl install booted App.app
RE:        r2 -A myapp  |  ghidra  |  x64dbg myapp.exe
CHECKSEC:  checksec --file=myapp
SHIP:      strip app && upx --best app && depends app && strings app | rg pdb
STEAM:     SteamInput()->Init() + GetGlyphForActionOrigin() → future-proof glyphs
CROSS:     zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux
LOC:       scc src/
```

---

## 🗺️ Dev Flow

```
WRITE CODE
  │
  ├──► watchexec auto-rebuilds ──► cppcheck + clang-tidy (static analysis)
  │                                        │
  │                                        ▼
  │                              gdb / lldb / raddebugger (debug)
  │                              sanitizers (ASAN/UBSAN)
  │                                        │
  │                                        ▼
  │                              tracy profile (CPU)
  │                              RGP/RGA profile (GPU)
  │                                        │
  │                                        ▼
  │                              hyperfine benchmark
  │                                        │
  ├──► cross-compile ──► Zig/QEMU test ──► ADB deploy (Android)
  │                                       xcrun deploy (iOS)
  │                                       Steam Deck SSH (SteamOS)
  │                                        │
  ├──► reverse engineer ──► Ghidra decompile
  │                         r2 analyze
  │                         x64dbg runtime
  │                         WinDbg kernel
  │                                        │
  └──► ship ──► checksec + strip + UPX
               depends/ldd (DLL check)
               strings (secret scan)
               Steam Input API (IGA file)
               Big Picture / SteamTenfoot
```

---

Companion deep-dives linked throughout. Install via [scoop](https://scoop.sh), apt, or brew.
