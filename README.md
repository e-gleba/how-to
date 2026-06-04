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
| 19 | [Cross-compile](#19-cross-compile) | Zig + QEMU (optional) |

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
--type-add
glsl:*.vert,*.frag,*.geom,*.comp,*.glsl
--type-add
hlsl:*.hlsl,*.hlsli,*.fx,*.fxh
--type-add
premake:*.lua,premake*.lua
--type-add
just:justfile,.justfile,*.just
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
    {"name": "dev", "configurePreset": "dev"},
    {"name": "clang", "configurePreset": "clang"}
  ],
  "testPresets": [
    {"name": "dev", "configurePreset": "dev", "output": {"outputOnFailure": true}}
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

## 19. Cross-compile

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
                               ▼
                             ship
```

---

Tools via [scoop](https://scoop.sh). Companion deep-dives linked throughout.
