# how-to

> C++ dev on Windows — tool by tool, daily drivers first, advanced linked

---

## 📋 Index

| # | Tool | What |
|---|------|------|
| 1 | [ripgrep](#1-ripgrep) | Search code, file types, daily patterns |
| 2 | [fd](#2-fd) | Find files |
| 3 | [fzf](#3-fzf) | Fuzzy everything |
| 4 | [bat](#4-bat) | `cat` with syntax |
| 5 | [CMake](#5-cmake) | Presets, build, compile_commands |
| 6 | [just](#6-just) | Task runner — never type cmake again |
| 7 | [ccache](#7-ccache) | Compiler cache |
| 8 | [watchexec](#8-watchexec) | Auto-rebuild on save |
| 9 | [cppcheck](#9-cppcheck) | Static analysis |
| 10 | [clang-tidy](#10-clang-tidy) | Linting + auto-fix |
| 11 | [clang-format](#11-clang-format) | Formatting |
| 12 | [ast-grep](#12-ast-grep) | Structural search |
| 13 | [GDB](#13-gdb) | Debugger |
| 14 | [Tracy](#14-tracy) | Profiler |
| 15 | [hyperfine](#15-hyperfine) | Benchmarking |
| 16 | [vcpkg / Conan](#16-vcpkg--conan) | Package managers |
| 17 | [Windows](#17-windows) | CRT, PDB, Defender, long paths |
| 18 | [Binary tools](#18-binary-tools) | UPX, pe-bear, depends, ImHex |
| 19 | [Tracing](#19-tracing) | Trace what a command actually does |
| 20 | [Cross-compile](#20-cross-compile) | Zig + QEMU (optional) |
| 21 | [Steam Deck devkit](#21-steam-deck-devkit) | Remote deploy and debug on Deck |
| 22 | [Radeon GPU profiling](#22-radeon-gpu-profiling) | RGP, RGA, RRA, RGD, shader analysis |
| 23 | [Best practices & polyfills](#23-best-practices--polyfills) | cppreference macros, stdx, BackportCpp |
| 24 | [Shortcuts](#24-shortcuts) | VS, VS Code, terminal, Windows |
| 25 | [Where to watch](#25-where-to-watch) | Channels, podcasts, conference talks |

---

## 1. ripgrep

Search code. Respects `.gitignore`. 10–50× faster than `grep`.

### File types

```bash
rg --type-list                 # list all 100+ built-in types

rg "ptrn" --type cpp           # .cpp .hpp .h .c .cc .cxx .hxx .hh
rg "ptrn" --type cmake         # CMakeLists.txt .cmake
rg "ptrn" --type c             # .c .h only
rg "ptrn" --type python
rg "ptrn" --type json
rg "ptrn" --type markdown
rg "ptrn" --type sh
rg "ptrn" -t cpp -t cmake      # multi-type (short form)
```

### Custom types — `~/.ripgreprc`

```bash
--type-add glsl:*.vert,*.frag,*.geom,*.comp,*.glsl
--type-add hlsl:*.hlsl,*.hlsli,*.fx,*.fxh
--type-add premake:*.lua,premake*.lua
--type-add just:justfile,.justfile,*.just
--type-add cmake_extra:CMakePresets.json,CMakeUserPresets.json
```

### Daily patterns

```bash
# TODOs
rg "TODO|FIXME|HACK|XXX|BUG" --type cpp -C 3

# Raw owning pointers (smart ptr candidates)
rg "new\s+\w+" --type cpp -n

# C-style casts
rg "\(\w+\*\)" --type cpp

# All #includes (deduplicated)
rg "^#include" --type cpp --no-filename --sort path | sort -u

# Empty bodies (potential bugs)
rg "\{\s*\}" --type cpp -n

# Files missing #pragma once
rg -L "^#pragma once" --type cpp -g "*.hpp"

# Count matches per file (sorted)
rg -c "std::vector" --type cpp | rg -v ":0$" | sort -t: -k2 -nr

# Replace (dry-run first!)
rg "old" --type cpp -l
rg "old" --type cpp -l | xargs sed -i 's/old/new/g'

# What commit added/removed this?
git log -S "function_name" --oneline
```

### Flags

```
-n          line numbers        -C 3        context lines
-l          filenames only      -c          count matches
-i          ignore case         -w          whole word
-v          invert match        --no-ignore skip .gitignore
-g "*.hpp"  glob filter         --json      machine output
```

→ **Advanced**: [search-navigation.md](search-navigation.md) — ugrep, tree-sitter, git history combos

---

## 2. fd

Find files. Faster than `find`. Respects `.gitignore`.

```bash
fd -e cpp -e hpp               # all C++ files
fd "test.*\.cpp"               # pattern match
fd --changed-within 1day       # modified today
fd -e cpp -x clang-format -i {}  # find + execute
fd -e cpp --size -1b           # empty files
```

→ **Advanced**: [search-navigation.md](search-navigation.md)

---

## 3. fzf

Fuzzy finder. Pipe anything into it.

```bash
# Fuzzy open file
fd -e cpp | fzf --preview 'bat --style=numbers {}' | xargs nvim

# Git branch checkout
git branch | fzf | xargs git checkout

# Search code → preview → open
rg --line-number --no-heading . | fzf --delimiter : \
  --preview 'bat --style=numbers {1}' --preview-window +{2}
```

→ **Advanced**: [search-navigation.md](search-navigation.md)

---

## 4. bat

`cat` with syntax highlighting, line numbers, git gutter.

```bash
bat src/main.cpp               # view
bat --diff src/main.cpp         # show git changes in gutter
bat --list-themes               # pick a theme
```

---

## 5. CMake

### Presets (drop-in)

```json
// CMakePresets.json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "dev",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/dev",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_C_COMPILER_LAUNCHER": "ccache",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
      }
    },
    {
      "name": "clang",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/clang",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "clang-cl",
        "CMAKE_CXX_COMPILER": "clang-cl",
        "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer",
        "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined",
        "CMAKE_CXX_CLANG_TIDY": "clang-tidy",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      }
    }
  ],
  "buildPresets": [
    { "name": "dev", "configurePreset": "dev" },
    { "name": "clang", "configurePreset": "clang" }
  ],
  "testPresets": [
    { "name": "dev", "configurePreset": "dev", "output": { "outputOnFailure": true } }
  ]
}
```

```bash
cmake --list-presets
cmake --preset dev
cmake --build --preset dev
ctest --preset dev

# Symlink compile_commands.json for clangd
# PowerShell:
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build/dev/compile_commands.json
```

→ **Advanced**: [cmake-presets.md](cmake-presets.md) — multi-compiler, release+PDB, cross-compile presets, unity builds

---

## 6. just

Command runner. Like Make but without the pain.

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
    rg "TODO|FIXME|HACK|XXX" --type cpp -C 2

bench preset="dev" app="myapp":
    cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 --min-runs 20 "./build/{{preset}}/{{app}}"

loc:
    scc src/

clean:
    rm -rf build/
```

```bash
just build preset=clang
just watch
just todos
```

→ **Advanced**: [justfile.md](justfile.md) — watch+test, bench-compare, docs generation

---

## 7. ccache

Compiler cache. Rebuild after touching 1 file → near-instant.

```bash
ccache -s                # stats
ccache -C                # clear
ccache --max-size 10G    # set limit

# In CMake presets (already in the preset above):
# "CMAKE_C_COMPILER_LAUNCHER": "ccache",
# "CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
```

→ **Advanced**: [build-acceleration.md](build-acceleration.md) — sccache, precompiled headers, unity builds, speedup table

---

## 8. watchexec

Watch files → run command. Instant feedback loop.

```bash
# Rebuild on save
watchexec -e cpp,hpp -- cmake --build build/dev

# Rebuild + test
watchexec -e cpp,hpp -- 'cmake --build build/dev && ctest --preset dev'

# Use via justfile instead:
just watch
```

---

## 9. cppcheck

Static analysis. Catch bugs without compiling.

```bash
cppcheck --enable=all --suppress=missingIncludeSystem src/
cppcheck --enable=warning,performance,portability src/
cppcheck --enable=all --xml src/ 2> cppcheck.xml    # CI
```

→ **Advanced**: [static-analysis.md](static-analysis.md) — suppressions, CMake integration, pre-commit hook

---

## 10. clang-tidy

Linting + auto-fix. Needs `compile_commands.json`.

```bash
clang-tidy -p build/dev src/main.cpp
clang-tidy -p build/dev --fix src/main.cpp
run-clang-tidy -p build/dev

# Best checks starter:
clang-tidy -p build/dev src/main.cpp \
  -checks="-*,clang-analyzer-*,modernize-*,performance-*,readability-*"
```

→ **Advanced**: [static-analysis.md](static-analysis.md) — check group table, per-target CMake config

---

## 11. clang-format

Formatting. One style, no arguments.

```bash
clang-format -style=llvm -dump-config > .clang-format
clang-format --dry-run -Werror src/main.cpp     # CI check
clang-format -i src/**/*.cpp src/**/*.hpp       # apply
```

---

## 12. ast-grep

Structural search — AST-aware, not text.

```bash
sg -p 'new $$$' --lang cpp              # all new expressions
sg -p 'delete $PTR' --lang cpp          # all raw deletes
sg -p '($TYPE)$EXPR' --lang cpp         # C-style casts
sg -p 'NULL' -r 'nullptr' --lang cpp -i  # replace NULLs
```

→ **Advanced**: [static-analysis.md](static-analysis.md) — custom rules, `sg scan`

---

## 13. GDB

```bash
gdb -tui ./myapp

# Inside:
b main              # breakpoint
b file.cpp:42       # line breakpoint
r                   # run
n                   # next
s                   # step into
c                   # continue
p var               # print
bt                  # backtrace
bt full             # backtrace + locals
info locals         # all locals
watch var           # break on change
layout split        # source + asm
thread apply all bt # all threads
```

→ **Advanced**: [debugging-profiling.md](debugging-profiling.md) — raddebugger, x64dbg, sanitizers, RenderDoc

---

## 14. Tracy

Profiler. Nanosecond overhead. Always-on in dev.

```cpp
#include <Tracy.hpp>

void heavy() {
    ZoneScoped;                    // auto-named zone
    ZoneScopedN("Physics");        // named zone
    TracyPlot("dt", dt * 1000.0);  // plot value
    FrameMark;                     // frame boundary
}
```

```cmake
FetchContent_Declare(tracy GIT_REPOSITORY https://github.com/wolfpld/tracy.git GIT_TAG v0.11.1)
FetchContent_MakeAvailable(tracy)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
```

```bash
tracy-profiler & ./build/dev/myapp
```

→ **Advanced**: [debugging-profiling.md](debugging-profiling.md) — memory tracking, mutex profiling, conditional compilation

---

## 15. hyperfine

Statistical benchmarking. Warmup, outliers, comparison.

```bash
hyperfine --warmup 5 --min-runs 20 "./myapp"
hyperfine "./old_app" "./new_app"                  # compare
hyperfine -P size 1024 65536 "./myapp --buf {size}"  # sweep
hyperfine --export-markdown results.md "./myapp"
```

→ **Advanced**: [benchmarking.md](benchmarking.md) — Google Benchmark, compilation benchmarks

---

## 16. vcpkg / Conan

### vcpkg (manifest mode)

```json
// vcpkg.json
{ "name": "myapp", "version": "1.0.0", "dependencies": ["fmt", "spdlog"] }
```

```bash
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```

### Conan 2

```python
# conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMakeToolchain, CMakeDeps, cmake_layout

class MyApp(ConanFile):
    name = "myapp"; version = "1.0"
    settings = "os", "compiler", "build_type", "arch"
    def requirements(self): self.requires("fmt/11.0.2")
    def layout(self): cmake_layout(self)
    def generate(self):
        CMakeToolchain(self).generate()
        CMakeDeps(self).generate()
```

```bash
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
```

→ **Advanced**: [cmake-package-managers.md](cmake-package-managers.md) — vcpkg vs Conan comparison, FetchContent, CPM.cmake

---

## 17. Windows

### Toolchain pick

| Compiler | Best for |
|----------|----------|
| **MSVC** (`cl`) | Production, best PDB |
| **Clang-cl** | Sanitizers, clang-tidy, cross-platform parity |
| **GCC** (MSYS2) | POSIX APIs, autotools |

### Essentials

```cmake
# Static CRT (no DLL dep, larger binary)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# PDB in release (crash dumps)
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
```

```powershell
# Defender exclusion for build dirs (admin)
Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"

# Enable long paths (admin)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" \
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

→ **Advanced**: [windows-cpp.md](windows-cpp.md) — Winsock, memory-mapped files, console setup, env vars

---

## 18. Binary tools

```bash
# Compress
strip myapp.exe && upx --best myapp.exe

# DLL dependencies
depends myapp.exe

# PE inspection (sections, imports, security flags)
pe-bear myapp.exe

# Strings
strings myapp.exe | rg "password|secret|token|key"

# Hex editor
imhex myapp.exe

# Reverse engineering
ghidra                     # decompiler
cutter myapp.exe           # radare2 GUI
x64dbg myapp.exe           # runtime debugger
```

### Pre-ship checklist

```bash
strings myapp.exe | rg "\.pdb"                       # debug symbols leaked?
pe-bear myapp.exe                                     # ASLR + DEP on?
strings myapp.exe | rg -i "password|secret|token"     # secrets?
depends myapp.exe                                     # missing DLLs?
ls -lh myapp.exe                                      # size ok?
```

→ **Advanced**: [binary-tools.md](binary-tools.md) — radare2 CLI, Ghidra scripting, x64dbg scripting

---

## 19. Tracing

> "What did that command actually do?" — files opened, registry touched, syscalls made, network connections, pipes, child processes.

### What traces what

| Tool | Traces | Installed? | Cross-platform? |
|------|--------|-----------|-----------------|
| **procmon** | Files, registry, network, process/thread (GUI + CLI) | ✅ sysinternals | Windows only |
| **handle** | Open handles snapshot (files, pipes, mutexes, etc.) | ✅ sysinternals | Windows only |
| **listdlls** | DLLs loaded by a process | ✅ sysinternals | Windows only |
| **drstrace** | System calls (arguments + return values) | ❌ `scoop install drmemory` | Windows only |
| **QEMU -strace** | Syscalls of a Linux binary | ✅ qemu | Linux binaries on any host |
| **perfetto** | System-wide kernel + userspace tracing | ✅ perfetto | Linux + Android + Windows (limited) |
| **ETW / logman** | Kernel events (file I/O, network, process) | ✅ built-in | Windows only |

### procmon — the big one (GUI + CLI)

```bash
# GUI mode — filter by process name, see file/registry/network activity in real time
procmon

# CLI mode — capture to file, no GUI
procmon /Quiet /Minimized /BackingFile trace.pml /AcceptEula

# Stop capture
procmon /Terminate

# Open saved trace
procmon /OpenLog trace.pml

# Convert trace to CSV/XML for scripting
procmon /OpenLog trace.pml /SaveAs trace.csv
procmon /OpenLog trace.pml /SaveAs trace.xml
```

**Filter examples** (set in GUI, or pre-configure with `.pmf` config):

```
Process Name  is        myapp.exe      → only my app
Operation     is        WriteFile      → only file writes
Operation     contains  Delete         → file/registry deletes
Path          contains  \\build\\        → only in build folder
Result        is not    SUCCESS        → only failures
```

### handle — what's open right now?

```bash
# All handles for a process
handle -p myapp.exe

# Only file handles
handle -p myapp.exe -a | rg "File"

# Find which process has a file locked
handle some_file.dll

# Find all pipes
handle -a | rg "Pipe"

# Find all network sockets
handle -a | rg "Socket|Tcp|Udp"
```

### listdlls — what DLLs are loaded?

```bash
listdlls myapp.exe
listdlls -v myapp.exe           # verbose (version numbers)
listdlls -d some.dll            # which processes loaded this DLL?
```

### drstrace — "strace for Windows"

```bash
# Install
scoop install drmemory

# Trace all syscalls of a command
drstrace -- myapp.exe arg1 arg2
# Output → drstrace.myapp.exe.PID.0000.log

# Trace to specific directory
drstrace -logdir ./traces -- myapp.exe

# Don't follow child processes
drstrace -no_follow_children -- myapp.exe

# Example output line:
# NtCreateFile(0x..., 0x80100000, "\Device\HarddiskVolume4\...\config.json", ...) => 0x0 (STATUS_SUCCESS)
```

### QEMU -strace — syscalls of a Linux binary

```bash
# Trace all syscalls
qemu-x86_64 -strace ./myapp_linux

# Trace specific syscalls (grep the output)
qemu-x86_64 -strace ./myapp_linux 2>&1 | rg "openat|read|write|connect"
```

### ETW / logman — built-in kernel tracing

```bash
# Create a trace session (admin)
logman create trace KernelTrace -p "Microsoft-Windows-Kernel-File" -o trace.etl

# Start
logman start KernelTrace

# Run your app...

# Stop
logman stop KernelTrace

# View with PerfView or Windows Performance Analyzer
```

### When to use which

```
"Which files did my app touch?"           → procmon (filter Process Name)
"Who's locking this DLL?"                 → handle
"What DLLs are loaded at runtime?"        → listdlls
"What syscalls did this command make?"    → drstrace (Windows) / qemu -strace (Linux)
"Why is this slow? (system-level)"        → perfetto / ETW
"Network connections made?"               → procmon (filter Operation = TCP Connect)
"Registry keys modified?"                 → procmon (filter Operation contains RegSet)
"Child processes spawned?"                → procmon (filter Operation = Process Create)
```

→ **Advanced**: [binary-tools.md](binary-tools.md) — Ghidra, radare2, x64dbg for deeper reverse engineering

---

## 20. Cross-compile

```bash
# Zig — one command to Linux binary from Windows
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux

# Test with QEMU
qemu-x86_64 ./main_linux

# MSYS2 POSIX env:
# C:\Users\YOU\scoop\apps\msys2\current\msys2_shell.cmd
```

→ **Advanced**: [cross-compilation.md](cross-compilation.md) — CMake toolchain, Android NDK, feature detection

---

## 21. Steam Deck devkit

### Setup

Install the [Steamworks partner devkit](https://partner.steamgames.com/doc/sdk) on your Deck. The `SteamworksContentKit` package includes the Steamship Remote Play Together and SSH servers.

```bash
# On the Deck (SteamOS):
# 1. Enable developer mode: Settings → System → "Developer Mode"
# 2. Install SteamworksContentKit from Steam (search "devkit")
# 3. Start the SSH service:
sudo systemctl start steamdevkit-ssh
sudo systemctl enable steamdevkit-ssh
```

### Remote deploy

```bash
# SCP your build directly to the Deck
scp -P 22 -r build/dev/ user@192.168.X.X:/home/deck/.local/share/Steam/steamapps/compatdata/0/pfx/drive_c/users/steamuser/Documents/

# Or use Steam Remote Play Together for local testing:
# Build with Steam Networking Sockets enabled, then stream to the Deck.
```

### Remote SSH (no SCP)

```bash
# SSH into the Deck from Windows
ssh deck@192.168.X.X

# Forward the GPU port for RenderDoc (Windows host → Deck GPU)
ssh -L 48437:127.0.0.1:48437 deck@192.168.X.X

# Or use Visual Studio "Remote Debugging over SSH"
# Set Remote Debugger to "Native Only, No Authentication" on the Deck.
```

### Proton vs native

```bash
# Steam's Remote Play feature will auto-wrap your Linux binary in Proton if needed.
# Prefer native Linux for best performance, unless the project depends on
# Windows-specific APIs (Win32, COM, Direct3D).
#
# For Vulkan: both paths work identically — the Steam Deck has a Vulkan
# driver that works the same way on native Linux as it would through Proton.
```

→ **Advanced**: [debugging-profiling.md](debugging-profiling.md) — RenderDoc, Tracy remote profiling over network

---

## 22. Radeon GPU profiling

### Tools overview

| Tool | Purpose | Installed? | Docs |
|------|---------|-----------|------|
| **RGP (Render GPU Profiler)** | Per-frame GPU capture — shader stalls, draw call breakdown, memory bandwidth | ❌ `scoop install` from [GPUOpen](https://gpuopen.com/developer-toolSuite/radeon-developer-tool-suite/) | [RGP User Guide](https://gpuopen.com/learn/radeon_gpu_profiler/) |
| **RGA (Render Graphics Analyzer)** | Offline shader analysis — instruction count, registers, wavefront stats | ❌ same suite | [RGA Docs](https://gpuopen.com/learn/radeon_graphics_analyzer/) |
| **RRA (Render Results Analyzer)** | Reactive rendering — frame-by-frame GPU replay, graphics pipe state | ❌ same suite | [RRA Docs](https://gpuopen.com/learn/radeon_reactive_rendering/) |
| **RGD (Render Graphics Driver)** | Driver-level GPU profiling — low-level command buffer analysis | ❌ same suite | [RGD Docs](https://gpuopen.com/learn/radeon_graphics_driver/) |
| **RDP (Render Debug)** | Validation layer for Vulkan — detect driver bugs, incorrect usage | ❌ same suite | [RDP Docs](https://gpuopen.com/learn/radeon_debug/) |

### RGP — daily use

```bash
# CLI — capture frame N of a Vulkan app
rgp capture --output ./frames.rgp --app ./myapp --frame 42

# CLI — parse specific metrics
rgp info ./frames.rgp --metrics fps,draw_calls,triangles,vertex_shading,fragment_shading

# CLI — compare two captures
rgp compare ./old.rgp ./new.rgp --output ./diff.html
```

### RGA — offline shader analysis

```bash
# Analyze compiled SPIR-V shaders
rga analyze --spirv ./shaders/scene.comp --output ./shader_report.html

# Check register usage, wavefront occupancy, instruction throughput
rga analyze --spirv ./shaders/scene.comp --check-registers --check-occupancy
```

### RRA — reactive rendering

```bash
# Record a GPU trace (Vulkan apps only)
rra record --app ./myapp --output ./trace.rra

# Open in RRA GUI to step through the graphics pipeline frame-by-frame
# Inspect descriptor sets, pipeline state, resource bindings per draw call.
```

### VS Code — RGA extension

```bash
# Install "Radeon GPU Analyzer" from the VS Code marketplace.
# It provides shader analysis integration — drop a .vert/.frag/.comp file,
# view instruction counts and register usage inline.
```

### When to use which

```
"Where's my frame time going?"            → RGP
"Is this shader the bottleneck?"           → RGA (offline SPIR-V analysis)
"Let me replay the GPU commands frame-by-frame" → RRA
"Vulkan driver bug suspected"              → RDP (validation)
"Command buffer layout / pipeline state"   → RGD (driver-level)
```

→ **Advanced**: [debugging-profiling.md](debugging-profiling.md) — RenderDoc as alternative, Tracy GPU zones, NVIDIA Nsight if you switch to RTX

---

## 23. Best practices & polyfills

### Feature testing

Check compiler support before using a C++20/23/26 feature.

```cpp
#include <version>   // stdc_version, __cpp_concepts, etc. — see cppreference

// C++26 — future
#if __has_include(<concepts>) && __cplusplus >= 202300L
// safe to use concepts
#endif

// C++23 — std::span (guarded)
#if __cpp_lib_span >= 202002L
#include <span>
#endif
```

> More macros: [cppreference — Compiler feature support](https://en.cppreference.com/w/cpp/compiler_support)

### Polyfill libraries

| Library | Scope | License | Link |
|---------|-------|---------|------|
| **std::span polyfill** | `std::span` | MIT | [tcbrindle/cpp17_headers](https://github.com/tcbrindle/spconv) |
| **stdx** | Most C++20 types (bytes, string_view32, formatting, ranges) | Apache-2.0 | [std-lite/stdx](https://github.com/std-lite/stdx) |
| **BackportCpp** | C++17/20/23 features backported to C++11 | MIT | [BackportCpp/BackportCpp](https://github.com/BackportCpp/BackportCpp) |
| **Boost.Compat** | Boost-powered polyfills (algorithms, containers, string_view) | BSL-1.0 | [boostorg/compat](https://github.com/boostorg/compat) |

### Polyfill example — `std::span`

```cmake
# CMake
FetchContent_Declare(
  span_polyfill
  GIT_REPOSITORY https://github.com/tcbrindle/spconv.git
  GIT_TAG main
)
FetchContent_MakeAvailable(span_polyfill)
target_link_libraries(myapp PRIVATE span::span)
```

```cpp
// Now works on C++17 and earlier
#include <span>
void process(std::span<int> data) {
    for (auto v : data) { /* ... */ }
}
```

### Jason Turner — C++ Weekly

> [JasonT Turner C++ Weekly](https://cpp.libhunt.com/cpp-weekly) — 600+ weekly episodes on tooling, safety, performance, best practices.

| Category | Top episodes |
|----------|-------------|
| **Tooling** | #111 (CMake and You), #113 (Clang-Tidy), #127 (LLDB) |
| **Safety** | #105 (Assertions vs Exceptions), #108 (Constexpr Safety), #114 (RAII) |
| **Performance** | #118 (Cache Friendly Code), #120 (SIMD Primer), #124 (Inlining) |
| **Best practices** | #121 (Move Semantics), #123 (Rule of Five), #126 (SFINAE vs Concepts) |

**Book**: [The Complete Overview of C++ 20](https://www.youtube.com/playlist?list=PLrzuiR0xo2OqMPl5ZZ3YuqGzM6yZbhJJU)

→ **Advanced**: [best-practices.md](best-practices.md) — cppreference search, cppbestpractices.com, section 24 & 25 below

---

## 24. Shortcuts

### Visual Studio

| Shortcut | Action |
|----------|--------|
| `Ctrl+Shift+B` | Build |
| `F5` / `Ctrl+F5` | Debug / Run |
| `F10` | Step over |
| `F11` | Step into |
| `Shift+F11` | Step out |
| `Ctrl+Shift+F10` | Current line execute (Ctrl+Shift+F5) |
| `Ctrl+K, Ctrl+D` | Format document |
| `Ctrl+K, Ctrl+F` | Format selection |
| `Ctrl+M, Ctrl+M` | Toggle outlining |
| `Ctrl+M, L` | Outline all |
| `Ctrl+M, O` | Collapse all |
| `Ctrl+.` | Quick actions |
| `F12` | Go to definition |
| `Ctrl+F12` | Go to implementation |
| `Alt+F12` | Peek definition |

> Microsoft docs: [Visual Studio keyboard shortcuts](https://docs.microsoft.com/en-us/visualstudio/ide/reference/keyboard-shortcuts)

### Visual Studio Code

| Shortcut | Action |
|----------|--------|
| `Ctrl+Shift+P` | Command palette |
| `Ctrl+P` | Quick open |
| `Ctrl+` ` ` | Open integrated terminal |
| `Alt+Z` | Toggle word wrap |
| `Ctrl+Shift+X` | Extensions |
| `Ctrl+\`` | Toggle terminal |
| `Ctrl+Shift+\\` | Diff two files |
| `F12` | Go to definition |
| `Alt+F12` | Peek definition |
| `Ctrl+Shift+O` | Go to symbol |
| `Ctrl+Shift+T` | Reopen closed tab |
| `Ctrl+Shift+K` | Delete line |
| `Alt+Up/Down` | Move line |
| `Ctrl+Shift+L` | Select all occurrences |
| `Ctrl+D` | Select next occurrence |
| `Ctrl+/` | Toggle line comment |
| `Ctrl+Shift+\\` | Find and replace |

> Microsoft docs: [VS Code keyboard shortcuts](https://code.visualstudio.com/shortcuts/keyboard-shortcuts-windows.pdf)

### Terminal (PowerShell + Windows)

| Shortcut | Action |
|----------|--------|
| `Ctrl+Shift+Enter` | Open as admin |
| `Ctrl+Shift+N` | New folder |
| `Ctrl+L` | Clear screen |
| `Alt+Enter` | Fullscreen window |
| `Ctrl+Shift+V` | Paste from clipboard |
| `Ctrl+Shift+S` | Save output to file |

### Windows Explorer

| Shortcut | Action |
|----------|--------|
| `Win+D` | Show desktop |
| `Win+E` | Open Explorer |
| `Win+.` | Emoji picker (includes code symbols: © ™ « » ² ³ ° ± ≈ ≤ ≥ ∈ √ ∞ ∑) |
| `Win+Shift+S` | Snip screenshot |
| `Win+V` | Clipboard history |

> Microsoft docs: [Windows keyboard shortcuts](https://support.microsoft.com/en-us/windows/keyboard-shortcuts-in-windows-dcc61a57-8ff0-cffe-679a-bb7c2082b9ab)

---

## 25. Where to watch

### YouTube channels

| Channel | What | Link |
|---------|------|------|
| **C++ Weekly (Jason Turner)** | Weekly episodes, tooling, C++20/23 features | [cpp.libhunt.com/c++weekly](https://cpp.libhunt.com/cpp-weekly) |
| **The Cherno** | Game engine from scratch (C++), Yaretzi engine series | [youtube.com/thecherno](https://www.youtube.com/user/TheCherno) |
| **CodeBeautiful** | C++ performance, interviews, best practices | [youtube.com/@CodeBeautiful](https://www.youtube.com/@CodeBeautiful) |
| **Catch2** | Testing framework deep dives, unit testing | [youtube.com/@Catchorg](https://www.youtube.com/@CatchOrg) |
| **C++ Insights** | Compiler output — see what C++ really compiles to | [youtube.com/@CPlusPlusInsights](https://www.youtube.com/@CppInsights) |
| **CppCon talks** | Annual C++ conference — 300+ talks/year | [youtube.com/@CppCon](https://www.youtube.com/@CppCon) |
| **GDC / Game Developers Conference** | GPU, engine architecture, AI, rendering | [youtube.com/@GameDevelopersConference](https://www.youtube.com/@GameDevelopersConference) |
| **GPUOpen** | AMD GPU drivers, Radeon developer tools | [youtube.com/@GPUOpen](https://www.youtube.com/@GPUOpen) |
| **Breaking The Sim** | Game dev, VFX, C++ tools, performance | [youtube.com/@BreakingTheSim](https://www.youtube.com/@BreakingTheSim) |

### Podcasts

| Podcast | What | Link |
|---------|------|------|
| **CppCast** | C++ weekly interviews, news | [cppcast.com](https://cppcast.com) |
| **Speaker's Voice** | Behind the scenes of C++ conferences | [speakersvoice.com](https://speakersvoice.com) |
| **Naked C++** | Language features, library changes | [nakedcpp.com](https://nakedcpp.com) |

### Conference schedule

| Event | When | Where | Talks |
|-------|------|-------|-------|
| **CppCon** | August | Colorado Springs | 300+ talks |
| **CppCast Live** | Autumn | Various | Workshops |
| **CppNow** | July | Utah | 150+ talks |
| **Meeting C++** | November | Berlin | 200+ talks |
| **ZetCon** | Spring | Online | C++ weekly + C++ Core Guidelines |
| **GDC** | March | San Francisco | Engine/GPU/game-specific talks |

### Recommended talks to start

| Talk | Speaker | Why |
|------|---------|-----|
| "Is Parallel Programming Hard, And If So, What Can You Do About It?" | Paul Khuong | Parallelism reality check |
| "A Brief History of Everything" | Herb Sutter | C++ history that shaped the language |
| "Custom Allocation in C++" | Matt kalk | Game engine memory management |
| "Vulkan Programming" | Kenneth Russel | GPU fundamentals |
| "What Every C++ Programmer Should Know About NaN" | Richard Smith | Floating-point correctness |

→ **Advanced**: [best-practices.md](best-practices.md) — cppreference, cppbestpractices.com, section 23 above

---

## ⚡ Quick-tip cards

```
BUILD:       cmake --preset dev && cmake --build --preset dev
WATCH:       watchexec -e cpp,hpp -- cmake --build build/dev
SEARCH:      rg "pattern" --type cpp -n -C 3
FIND:        fd -e cpp -e hpp
TODOS:       rg "TODO|FIXME|HACK" --type cpp -C 2
BENCH:       hyperfine --warmup 5 --min-runs 20 "./myapp"
LINT:        cppcheck --enable=all --suppress=missingIncludeSystem src/
FORMAT:      clang-format -i src/**/*.cpp src/**/*.hpp
LOC:         scc src/
DEPS:        depends myapp.exe
COMPRESS:    strip myapp.exe && upx --best myapp.exe
CACHE:       ccache -s
PROFILE:     tracy-profiler & ./myapp
COMPILE_DB:  cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
LINK_DB:     ln -s build/dev/compile_commands.json .
TRACE:       procmon /BackingFile trace.pml & myapp.exe & procmon /Terminate
STRACE:      drstrace -- myapp.exe        (scoop install drmemory)
SOCKETS:     handle -a | rg "Socket|Tcp"
DLLS:        listdlls myapp.exe
GPU:         rgp capture --output frames.rgp --app myapp --frame 42
STEAM:       scp -P 22 -r build/ user@DECK_IP:~/...
POLYFILL:    FetchContent stdx or BackportCpp for C++20 types on C++17
SHORTCUT:    F5 debug | F10 step-over | F12 go-to-def | Ctrl+Shift+P VSCode cmd palette
CPP_WEEKLY:  cpp.libhunt.com/cpp-weekly — 600+ episodes, tooling/safety/performance
```

---

## 🗺️ Flow

```
WRITE CODE ──► watchexec rebuilds ──► cppcheck + clang-tidy
                    │                       │
                    ▼                       ▼
              gdb/raddebugger          sanitizers
                    │                       │
                    └──────────┬────────────┘
                               ▼
                          tracy profile
                               │
                               ▼
                          hyperfine bench
                               │
                     ┌─────────┴─────────┐
                     ▼                   ▼
              procmon/drstrace        ship
              (trace syscalls,        (UPX/strip,
               files, network)         polyfill compat)
                               │
                               ▼
                          GPU profile (RGP)
                               │
                               ▼
                          Steam Deck remote deploy
```

---

Tools via [scoop](https://scoop.sh). Companion deep-dives linked throughout.
