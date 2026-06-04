# 🛠️ C++ Dev Handbook

> Windows · CMake · daily-driver reference · grab-and-go

Single-file reference. Every section: **what to type** + **why**.

---

## 🔎 ripgrep — daily driver

### File types (use these, not `grep -r`)

```bash
# List all 100+ built-in types
rg --type-list

# C++ family
rg "pattern" --type cpp          # .cpp .hpp .h .c .cc .cxx .hxx .hh
rg "pattern" --type cmake        # CMakeLists.txt .cmake
rg "pattern" --type c            # .c .h only
rg "pattern" --type cpp -tc      # cpp OR c

# Scripts & config
rg "pattern" --type python
rg "pattern" --type json
rg "pattern" --type yaml
rg "pattern" --type markdown
rg "pattern" --type sh

# Build & tooling
rg "pattern" --type make         # Makefile *.mk *.mak
rg "pattern" --type bazel        # BUILD *.bzl
rg "pattern" --type meson        # meson.build
rg "pattern" --type docker
rg "pattern" --type js
rg "pattern" --type rust
rg "pattern" --type go
rg "pattern" --type java

# Multi-type
rg "TODO" --type cpp --type cmake --type python
rg "TODO" -t cpp -t cmake -t py     # short form
```

### Add custom file types

```bash
# ~/.ripgreprc   (or set RIPGREP_CONFIG_PATH)
--type-add
glsl:*.vert,*.frag,*.geom,*.comp,*.glsl
--type-add
hlsl:*.hlsl,*.hlsli,*.fx,*.fxh
--type-add
csharp:*.cs,*.csx,*.cshtml
--type-add
premake:*.lua,premake*.lua
--type-add
just:justfile,.justfile,*.just

# Now use:
rg "MVP" --type glsl
rg "StructuredBuffer" --type hlsl
rg "workspace" --type premake
rg "build:" --type just
```

### Daily patterns

```bash
# ── Find stuff ──────────────────────────────

# TODOs with 3 lines context
rg "TODO|FIXME|HACK|XXX|BUG" --type cpp -C 3

# Find raw owning pointers (smart ptr candidates)
rg "new\s+\w+" --type cpp -n

# Find C-style casts
rg "\(\w+\*\)" --type cpp

# Find #include directives
rg "^#include" --type cpp --no-filename --sort path | sort -u

# Find functions with many params (>5)
rg "^\s*(?:static\s+)?(?:inline\s+)?(?:virtual\s+)?\w+\s+\w+\([^)]{80,}" --type cpp

# Find empty if/for/while bodies (potential bugs)
rg "\{\s*\}" --type cpp -n

# Find noexcept(false) or missing noexcept
rg "\bthrow\b" --type cpp

# Find recursive includes
rg '^#include\s+"' --type cpp -r '$0' | sort | uniq -d

# ── Count / stats ───────────────────────────

# Files containing a pattern
rg -l "singleton" --type cpp | wc -l

# Count matches per file (sorted)
rg -c "std::vector" --type cpp | rg -v ":0$" | sort -t: -k2 -nr

# Count total matches in project
rg -c "std::shared_ptr" --type cpp | awk -F: '{s+=$2} END {print s}'

# ── Replace ─────────────────────────────────

# Dry-run first (always)
rg "old_name" --type cpp -l
# Then replace
rg "old_name" --type cpp -l | xargs sed -i 's/old_name/new_name/g'

# ── Files that DON'T contain ────────────────

# Headers missing pragma once
rg -L "^#pragma once" --type cpp -g "*.hpp"

# Files without copyright header
rg -L "Copyright" --type cpp

# ── Git-aware ───────────────────────────────

# Only in tracked files (default — rg ignores .gitignore)
rg "pattern" --type cpp

# Include untracked files too
rg "pattern" --type cpp --no-ignore

# Search in a specific git revision
rg "pattern" --type cpp $(git ls-tree -r HEAD --name-only)

# Find what commit added/removed a pattern
git log -S "pattern" --oneline
git log -G "regex" --oneline
```

### rg flags you actually need

```bash
-n           # line numbers
-C 3         # 3 lines context
-l           # only filenames
-c           # count matches
-i           # ignore case
-w           # whole word
-v           # invert match
--no-ignore  # don't respect .gitignore
--hidden     # search hidden files too
--sort path  # sort output by path
--json       # JSON output (for scripts)
-g "*.hpp"   # glob filter (can also use --type)
--max-depth 3
--no-heading
--color always
```

---

## ⚡ Build

### CMake presets (drop-in)

```json
// CMakePresets.json — drop in project root
{
  "version": 6,
  "configurePresets": [
    {
      "name": "dev",
      "displayName": "Ninja + ccache",
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
      "name": "release",
      "displayName": "Release + ccache",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "RelWithDebInfo",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_CXX_FLAGS_RELEASE": "/Zi",
        "CMAKE_EXE_LINKER_FLAGS_RELEASE": "/DEBUG /OPT:REF /OPT:ICF",
        "CMAKE_C_COMPILER_LAUNCHER": "ccache",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
      }
    },
    {
      "name": "clang",
      "displayName": "Clang + sanitizers",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/clang",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "clang-cl",
        "CMAKE_CXX_COMPILER": "clang-cl",
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer",
        "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined",
        "CMAKE_CXX_CLANG_TIDY": "clang-tidy"
      }
    }
  ],
  "buildPresets": [
    {"name": "dev", "configurePreset": "dev"},
    {"name": "release", "configurePreset": "release"},
    {"name": "clang", "configurePreset": "clang"}
  ],
  "testPresets": [
    {"name": "dev", "configurePreset": "dev", "output": {"outputOnFailure": true}},
    {"name": "clang", "configurePreset": "clang", "output": {"outputOnFailure": true}}
  ]
}
```

```bash
# Use
cmake --preset dev
cmake --build --preset dev
ctest --preset dev

# Symlink compile_commands.json so clangd finds it
# PowerShell:
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build/dev/compile_commands.json
```

### justfile (no more typing cmake commands)

```makefile
# justfile — drop in project root

# List all recipes
default:
    @just --list

# Build
build preset="dev":
    cmake --preset {{preset}}
    cmake --build --preset {{preset}}

# Rebuild changed only
rebuild preset="dev":
    cmake --build build/{{preset}}

# Test
test preset="dev":
    ctest --preset {{preset}} --output-on-failure

# Watch → rebuild on save
watch preset="dev":
    watchexec -e cpp,hpp,h,c,cmake,just -- just rebuild {{preset}}

# Watch → rebuild + run tests
watch-test preset="dev":
    watchexec -e cpp,hpp,h,c,cmake,just -- 'just rebuild {{preset}} && just test {{preset}}'

# Lint
lint:
    cppcheck --enable=all --suppress=missingIncludeSystem src/ --error-exitcode=1

# Format
format:
    clang-format -i src/**/*.cpp src/**/*.hpp

format-check:
    clang-format --dry-run -Werror src/**/*.cpp src/**/*.hpp

# TODO hunting
todos:
    rg "TODO|FIXME|HACK|XXX" --type cpp -C 2

# Count lines
loc:
    scc src/

# Clean
clean:
    rm -rf build/

# Benchmark
bench preset="dev" app="myapp":
    cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 --min-runs 20 "./build/{{preset}}/{{app}}"

# Docs
docs:
    doxygen Doxyfile
```

### Build acceleration

```bash
# ccache stats
ccache -s

# Clear cache
ccache -C

# Set cache size
ccache --max-size 10G

# Verify ccache works — build twice, check stats
cmake --build build/dev
ccache -s   # cache hit (direct) should be high

# sccache (alternative, same CMake launcher vars)
sccache --start-server
```

```cmake
# Precompiled headers — big win for MSVC
target_precompile_headers(myapp PRIVATE
    <vector>
    <string>
    <memory>
    <functional>
    "pch.hpp"
)

# Unity builds — CI only
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16)
```

| Technique | Full rebuild | Incremental (1 file) |
|-----------|-------------|----------------------|
| Ninja | ~80% baseline | ~70% baseline |
| + ccache | ~80% | **~15%** |
| + pch | ~60% | ~50% |

---

## 🐛 Debug & Profile

### Tracy (always-on profiling)

```cpp
#include <Tracy.hpp>

void physics_update(float dt) {
    ZoneScoped;                    // auto-named: "physics_update"
    TracyPlot("dt", dt * 1000.0);  // plot value over time

    for (auto& body : bodies) {
        ZoneScopedN("Integrate");  // named zone
        body.integrate(dt);
    }
}

void render() {
    FrameMark;  // mark frame boundaries
}
```

```cmake
# CMakeLists.txt
FetchContent_Declare(tracy GIT_REPOSITORY https://github.com/wolfpld/tracy.git GIT_TAG v0.11.1)
FetchContent_MakeAvailable(tracy)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
```

```bash
# Launch profiler, run app — it connects automatically
tracy-profiler &
./build/dev/myapp
```

### GDB essentials

```bash
gdb -tui ./myapp        # TUI mode (source visible)

# Inside GDB:
b main                  # breakpoint at main
b file.cpp:42           # breakpoint at line
r                       # run
n                       # next (step over)
s                       # step (step into)
c                       # continue
p var                   # print variable
bt                      # backtrace
bt full                 # backtrace with locals
info locals             # all local variables
info args               # function arguments
watch var               # break when var changes
layout split            # source + asm
tui disable             # back to plain mode
thread apply all bt     # all thread backtraces
```

### Sanitizers

```bash
# Clang/GCC — catches memory + UB at runtime
# In CMake: add to CXX_FLAGS and LINKER_FLAGS
-fsanitize=address,undefined  # ASan + UBSan
-fno-omit-frame-pointer       # better stack traces

# Run — sanitizer stops on first error with full report
./build/clang/myapp
```

---

## 🔍 Static Analysis

### cppcheck

```bash
# Full check
cppcheck --enable=all --suppress=missingIncludeSystem src/

# Specific checks only
cppcheck --enable=warning,performance,portability src/

# XML for CI
cppcheck --enable=all --xml src/ 2> cppcheck.xml
```

### clang-tidy

```bash
# Requires compile_commands.json
clang-tidy -p build/dev src/main.cpp

# Auto-fix
clang-tidy -p build/dev --fix src/main.cpp

# Run on all files
run-clang-tidy -p build/dev

# Best checks to start with:
clang-tidy -p build/dev src/main.cpp \
  -checks="-*,clang-analyzer-*,cppcoreguidelines-*,modernize-*,performance-*,readability-*"
```

### clang-format

```bash
# Generate config
clang-format -style=llvm -dump-config > .clang-format

# Check (CI)
clang-format --dry-run -Werror src/main.cpp

# Apply
clang-format -i src/main.cpp
```

### ast-grep

```bash
# Find patterns structurally (not text)
sg -p 'new $$$' --lang cpp       # all new expressions
sg -p 'delete $PTR' --lang cpp   # all raw deletes
sg -p '($TYPE)$EXPR' --lang cpp  # C-style casts
sg -p 'NULL' -r 'nullptr' --lang cpp -i  # fix NULLs
```

---

## 📂 Navigation

### fd — find files

```bash
fd -e cpp -e hpp          # all C++ files
fd "test.*\.cpp"          # test files
fd --changed-within 1day  # modified today
fd -e cpp -x wc -l {}     # line count per file
fd -e cpp --size -1b      # empty files
```

### fzf — fuzzy everything

```bash
# Open file (fzf preview)
fd -e cpp | fzf --preview 'bat --style=numbers {}' | xargs nvim

# Search git branches
git branch | fzf | xargs git checkout

# Search command history
history | fzf
```

### bat — `cat` but good

```bash
bat src/main.cpp              # syntax highlight + line numbers
bat --diff src/main.cpp       # show git changes in gutter
bat --list-themes             # pick a theme
```

### scc — code stats

```bash
scc src/                      # by language
scc --by-file src/            # per file
scc --complexity src/         # estimate complexity
```

---

## ⏱️ Benchmark

```bash
# Statistical benchmark
hyperfine --warmup 5 --min-runs 20 "./myapp --bench"

# Compare two versions
hyperfine "./old_app" "./new_app"

# Parameter sweep
hyperfine -P size 1024 65536 "./myapp --buf {size}"

# Export
hyperfine --export-markdown results.md "./myapp"
hyperfine --export-json results.json "./myapp"
```

```cpp
// Google Benchmark
#include <benchmark/benchmark.h>
static void BM_MyFunction(benchmark::State& state) {
    for (auto _ : state) {
        auto result = compute();
        benchmark::DoNotOptimize(result);
    }
}
BENCHMARK(BM_MyFunction);
BENCHMARK_MAIN();
```

---

## 📦 Package Managers

### vcpkg (manifest mode)

```json
// vcpkg.json — drop in project root
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": ["fmt", "spdlog", "nlohmann-json", "catch2"]
}
```

```bash
# CMake flag
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"

# Then use normally in CMakeLists.txt:
# find_package(fmt CONFIG REQUIRED)
# target_link_libraries(myapp PRIVATE fmt::fmt)
```

### Conan 2

```python
# conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMake, CMakeToolchain, CMakeDeps, cmake_layout

class MyApp(ConanFile):
    name = "myapp"
    version = "1.0"
    settings = "os", "compiler", "build_type", "arch"

    def requirements(self):
        self.requires("fmt/11.0.2")
        self.requires("spdlog/1.14.1")

    def layout(self):
        cmake_layout(self)

    def generate(self):
        CMakeToolchain(self).generate()
        CMakeDeps(self).generate()

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()
```

```bash
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
cmake --build build
```

---

## 🪟 Windows specifics

### Toolchain pick

| Compiler | Best for |
|----------|----------|
| **MSVC** (`cl`) | Windows-native, best PDB, production |
| **Clang-cl** | Cross-platform parity, sanitizers, clang-tidy |
| **GCC** (MSYS2) | Linux-like, POSIX APIs, autotools |

### CRT linking

```cmake
# Static CRT — no DLL dependency, larger binary
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# Dynamic CRT — smaller binary, needs vcredist
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
```

### PDB in release builds

```cmake
# Debug symbols even in release — helps with crash dumps
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
```

### Defender exclusion

```powershell
# Build folders get slowed by real-time scanning
Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"
```

### Long paths

```powershell
# Enable in registry (admin):
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" \
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

---

## 🧰 Binary tools

```bash
# Compress release binary
strip myapp.exe
upx --best myapp.exe

# Check DLL dependencies
depends myapp.exe

# Inspect PE sections + security flags
pe-bear myapp.exe

# Extract strings
strings myapp.exe | rg "password|secret|token|key"

# Hex editor with pattern language
imhex myapp.exe

# Reverse engineering
ghidra                    # full decompiler suite
cutter myapp.exe          # radare2 GUI
x64dbg myapp.exe          # runtime debugger
```

### Pre-ship checklist

```bash
# 1. No debug symbols leaking
strings myapp.exe | rg "\.pdb"

# 2. Security flags on
pe-bear myapp.exe                     # check ASLR (DYNAMIC_BASE), DEP (NX_COMPAT)

# 3. No suspicious imports
pe-bear myapp.exe                     # Imports tab

# 4. No hardcoded secrets
strings myapp.exe | rg -i "password|secret|token|key|api"

# 5. Size check
ls -lh myapp.exe
```

---

## 🔀 Cross-compilation (optional)

```bash
# Zig does cross-compilation trivially:
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux
zig c++ -target aarch64-linux-gnu -O2 main.cpp -o main_arm64

# Test with QEMU
qemu-x86_64 ./main_linux

# MSYS2 for POSIX environment on Windows:
# C:\Users\YOU\scoop\apps\msys2\current\msys2_shell.cmd
# Provides: bash, GCC, autotools, POSIX headers
```

---

## 📋 Quick-tip cards

```
BUILD:          cmake --preset dev && cmake --build --preset dev
WATCH:          watchexec -e cpp,hpp -- cmake --build build/dev
SEARCH:         rg "pattern" --type cpp -n -C 3
FIND FILES:     fd -e cpp -e hpp
TODO HUNT:      rg "TODO|FIXME|HACK" --type cpp -C 2
BENCHMARK:      hyperfine --warmup 5 --min-runs 20 "./myapp"
LINT:           cppcheck --enable=all --suppress=missingIncludeSystem src/
FORMAT:         clang-format -i src/**/*.cpp src/**/*.hpp
COUNT LINES:    scc src/
CODE STATS:     scc --complexity src/
DEPENDENCIES:   depends myapp.exe
COMPRESS:       strip myapp.exe && upx --best myapp.exe
COMPILE_DB:     cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
LINK DB:        ln -s build/dev/compile_commands.json .
CACHE:          ccache -s
PROFILE:        tracy-profiler & ./myapp
```

---

## 🗺️ Flow

```
WRITE CODE ──► watchexec rebuilds ──► cppcheck + clang-tidy lint
                                          │
     ┌────────────────────────────────────┘
     ▼
  gdb/raddebugger ←── crash? ──→ sanitizers catch it
     │                                │
     ▼                                ▼
  fix bug                         fix bug
     │                                │
     └────────────┬───────────────────┘
                  ▼
            tracy profile ──► hyperfine bench ──► ship
```

---

More detailed deep-dives in companion files: [cmake-presets.md](cmake-presets.md) · [justfile.md](justfile.md) · [build-acceleration.md](build-acceleration.md) · [debugging-profiling.md](debugging-profiling.md) · [static-analysis.md](static-analysis.md) · [search-navigation.md](search-navigation.md) · [benchmarking.md](benchmarking.md) · [binary-tools.md](binary-tools.md) · [windows-cpp.md](windows-cpp.md) · [cmake-package-managers.md](cmake-package-managers.md) · [cross-compilation.md](cross-compilation.md)
