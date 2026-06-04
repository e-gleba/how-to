# 📘 C++ Dev Handbook — All In One

> Grab-and-go reference. Windows-first, cross-platform aware.

---

## 1. 🏗️ Build

### CMake Presets (the only way)

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "dev",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_C_COMPILER_LAUNCHER": "ccache",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
      }
    },
    {
      "name": "clang",
      "inherits": "dev",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "clang-cl",
        "CMAKE_CXX_COMPILER": "clang-cl",
        "CMAKE_CXX_CLANG_TIDY": "clang-tidy"
      }
    },
    {
      "name": "linux-cross",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/linux",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "zig cc",
        "CMAKE_CXX_COMPILER": "zig c++",
        "CMAKE_C_COMPILER_TARGET": "x86_64-linux-gnu",
        "CMAKE_CXX_COMPILER_TARGET": "x86_64-linux-gnu"
      }
    }
  ],
  "buildPresets": [
    {"name": "dev", "configurePreset": "dev"},
    {"name": "clang", "configurePreset": "clang"},
    {"name": "linux-cross", "configurePreset": "linux-cross"}
  ],
  "testPresets": [
    {"name": "dev", "configurePreset": "dev", "output": {"outputOnFailure": true}}
  ]
}
```

```bash
cmake --list-presets          # what can I build?
cmake --preset dev            # configure
cmake --build --preset dev    # build
ctest --preset dev            # test
```

> 💡 **Tip:** Put `CMakeUserPresets.json` in `.gitignore`. Inherit from shared presets, add personal flags there.

### justfile (stop typing cmake commands)

```makefile
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

tidy preset="dev":
    clang-tidy -p build/{{preset}} src/**/*.cpp

bench app="myapp" preset="dev":
    cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 "./build/{{preset}}/{{app}}"

todos:
    rg "TODO|FIXME|HACK|XXX" --type cpp -n
```

```bash
just build preset=clang
just watch         # auto-rebuild on save
just todos         # what needs work?
```

### ccache — free rebuild speed

```bash
ccache -s           # stats
ccache -C           # clear
ccache --max-size 10G
```

In CMake: `CMAKE_CXX_COMPILER_LAUNCHER=ccache`

> 💡 **Tip:** Build once, touch a file, rebuild. `ccache -s` should show cache hits. If not, check launcher is set.

### Unity builds (CI speed)

```cmake
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16)
```

Full rebuild ~500 files: 3:20 → 0:45. Only for CI/release — hides ODR violations.

---

## 2. 🌐 Cross-Compilation

### Zig (one command)

```bash
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o app_linux
zig c++ -target aarch64-linux-gnu -O2 main.cpp -o app_arm

# List all targets
zig targets
```

### QEMU (test without VM)

```bash
qemu-x86_64 ./app_linux
qemu-x86_64 -strace ./app_linux   # trace syscalls
qemu-aarch64 ./app_arm
```

### CMake toolchain for Zig

```cmake
# zig-toolchain.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_C_COMPILER "zig cc")
set(CMAKE_CXX_COMPILER "zig c++")
set(CMAKE_C_COMPILER_TARGET x86_64-linux-gnu)
set(CMAKE_CXX_COMPILER_TARGET x86_64-linux-gnu)
```

```bash
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake -G Ninja
cmake --build build/linux
```

> 💡 **Tip:** Instead of `if(WIN32)` / `if(APPLE)`, use feature detection: `check_include_file("sys/epoll.h" HAS_EPOLL)`. Works across platforms automatically.

---

## 3. 🐛 Debugging

### GDB quick reference

```bash
gdb -tui ./myapp          # TUI mode — source code visible
```

```
b main          # breakpoint at main
b file.cpp:42   # breakpoint at line
r               # run
n               # next (step over)
s               # step (step into)
c               # continue
p var           # print variable
bt              # backtrace
bt full         # backtrace + all locals
info locals     # local variables
watch var       # break when var changes
layout split    # source + asm side by side
thread apply all bt   # all thread backtraces
```

### raddebugger (Windows native, fast)

```bash
raddebugger ./myapp.exe
```

Clean UI, fast startup. No symbol server drama.

### x64dbg (release builds, no PDB)

```bash
x64dbg ./myapp.exe
```

Use when debugging stripped binaries, third-party DLLs, or crash dumps without symbols.

### Sanitizers (catch 80% of bugs at zero effort)

```cmake
# CMakeLists.txt or preset
set(CMAKE_CXX_FLAGS "-fsanitize=address,undefined -fno-omit-frame-pointer")
set(CMAKE_EXE_LINKER_FLAGS "-fsanitize=address,undefined")
```

```bash
# Then just run — sanitizers catch:
# - Use-after-free, buffer overflow, memory leaks (ASan)
# - Integer overflow, null deref, misaligned access (UBSan)
./build/dev/myapp
```

> 💡 **Tip:** Run sanitizers in CI. They catch things that pass all unit tests.

---

## 4. 📊 Profiling

### Tracy — the profiler

```cpp
#include <Tracy.hpp>

void heavy() {
    ZoneScoped;                     // auto-named zone
    ZoneScopedN("PhysicsUpdate");    // named zone
    TracyPlot("FrameTime", dt);      // plot values over time
    FrameMark;                       // mark frame boundaries
}
```

```cmake
# CMakeLists.txt
FetchContent_Declare(tracy GIT_REPOSITORY https://github.com/wolfpld/tracy.git GIT_TAG v0.11.1)
FetchContent_MakeAvailable(tracy)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
```

```bash
# Run profiler GUI, then run app — appears automatically
tracy-profiler &
./build/dev/myapp
```

> 💡 **Tip:** Macro overhead is nanoseconds. Keep Tracy on in dev builds always. Use `TRACY_ENABLE` compile definition to conditionally compile it out for release.

### hyperfine — benchmark CLI

```bash
hyperfine --warmup 5 --min-runs 20 './myapp'
hyperfine --prepare 'cmake --build build' './myapp --bench'
hyperfine './old_app' './new_app'          # compare versions
hyperfine --export-markdown results.md './myapp'
```

Output shows: mean ± σ, min/max range, outlier detection, statistical comparison.

---

## 5. 🔍 Code Quality

### cppcheck

```bash
cppcheck --enable=all --suppress=missingIncludeSystem src/
cppcheck --enable=all --xml src/ 2> report.xml   # CI output
```

### clang-tidy

```bash
clang-tidy -p build src/main.cpp
clang-tidy -p build -checks='modernize-*,performance-*' src/**/*.cpp
clang-tidy -p build --fix src/main.cpp            # auto-fix
```

| Check group | What it catches |
|-------------|-----------------|
| `clang-analyzer-*` | Memory leaks, use-after-free, null deref |
| `modernize-*` | Use nullptr, override, range-for, make_unique |
| `performance-*` | Unnecessary copies, inefficient patterns |
| `bugprone-*` | API misuse, branch clones, suspension points |
| `cppcoreguidelines-*` | C++ Core Guidelines violations |

### clang-format

```bash
clang-format -style=llvm -dump-config > .clang-format
clang-format --dry-run -Werror src/**/*.cpp   # CI check
clang-format -i src/**/*.cpp                   # apply
```

### ast-grep (structural search)

```bash
sg -p 'new $$$' --lang cpp              # all raw new expressions
sg -p 'delete $PTR' --lang cpp          # all raw deletes
sg -p '($TYPE)$EXPR' --lang cpp         # C-style casts
sg -p 'NULL' -r 'nullptr' --lang cpp -i  # replace
```

---

## 6. 🔎 Search & Navigate

### ripgrep (use this, not grep)

```bash
rg "pattern" --type cpp                     # search C++ files
rg "pattern" --type cpp -n -C 3             # with context
rg "TODO" --type cpp -l                      # list files only
rg "pattern" --type cpp --files-without-match # files NOT matching
rg 'new\s+\w+' --type cpp                    # regex
rg -l "pattern" | xargs clang-format -i       # pipe to commands
```

### fd (find files, not find)

```bash
fd -e cpp -e hpp                    # all C++ files
fd test_                            # files with "test_" in name
fd --change-newer-than 1day         # modified today
fd -e cpp -x clang-format -i {}     # find + execute
```

### fzf power combos

```bash
# Fuzzy file open
fd -e cpp | fzf | xargs nvim

# Search code, preview, open
rg --line-number --no-heading . | fzf --delimiter : \
  --preview 'bat --style=numbers {1}' --preview-window +{2}

# Git branch checkout
git branch | fzf | xargs git checkout
```

---

## 7. 📦 Package Managers

### vcpkg (manifest mode)

```json
// vcpkg.json
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": ["fmt", "spdlog", "nlohmann-json", "catch2"]
}
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
```

```bash
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
cmake --build build
```

| | vcpkg | Conan 2 |
|---|-------|---------|
| Style | Central catalog | Decentralized |
| Best for | Simple, MSVC | Complex, cross-platform |
| Versioning | Baseline snapshot | Per-package ranges |

> 💡 **Tip:** Pick one. Mixing package managers = ABI hell. Use FetchContent for header-only libs: `FetchContent_Declare(nlohmann_json GIT_REPOSITORY https://github.com/nlohmann/json.git GIT_TAG v3.11.3)`

---

## 8. 🔧 Binary Tools

```bash
# Compress release binary
strip myapp.exe && upx --best myapp.exe

# Check DLL dependencies
depends myapp.exe

# Inspect PE sections, imports, security flags
pe-bear myapp.exe

# Hex editor with pattern language
imhex myapp.exe

# Quick binary analysis (CLI)
r2 -A myapp.exe
# Inside r2: aaaa (analyze), afl (functions), iz (strings), ii (imports)

# Extract strings
strings myapp.exe | rg "http"

# Pre-ship checklist
strings myapp.exe | rg -i "password|secret|token|key"   # leaked secrets?
strings myapp.exe | rg "\.pdb"                           # debug symbols leaked?
depends myapp.exe                                         # missing DLLs?
```

> 💡 **Tip:** UPX-packed binaries trigger AV false positives. Always test after packing.

---

## 9. 🪟 Windows-Specific

### Toolchains available

| Compiler | Path | Use for |
|----------|------|---------|
| MSVC (`cl`) | VS Build Tools | Windows-native, best PDB |
| Clang (`clang-cl`) | `scoop/apps/llvm` | Cross-platform parity, sanitizers |
| GCC (`g++`) | `scoop/apps/msys2` | Linux-like, POSIX APIs |
| Zig (`zig c++`) | `scoop/apps/zig` | Cross-compilation |

### Tips

- **Use forward slashes** in includes: `#include "src/utils/foo.h"` — works on Windows
- **RelWithDebInfo** for daily dev — fast + symbols for crash dumps
- **Exclude build dirs** from Windows Defender: `Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"`
- **Enable long paths**: `New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1`
- **Use `clang-cl` as MSVC drop-in**: same ABI, same flags, but adds sanitizers and better diagnostics

---

## 10. 📝 Pre-Commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit
STAGED=$(git diff --cached --name-only --diff-filter=ACM | rg '\.(cpp|hpp|h|c)$')
[ -z "$STAGED" ] && exit 0

echo "$STAGED" | xargs clang-format --dry-run -Werror || {
    echo "Run: just format"
    exit 1
}
echo "$STAGED" | xargs cppcheck --enable=warning --suppress=missingIncludeSystem || exit 1
```

---

## 11. ⚡ Quick Reference Card

```
# ── BUILD ──
cmake --preset dev && cmake --build --preset dev     # build
cmake --build build                                   # rebuild changed
cmake --graphviz=build/graph.dot . && dot -Tpng ...    # dependency graph

# ── WATCH ──
watchexec -e cpp,hpp -- cmake --build build           # auto-rebuild
watchexec -e cpp,hpp -- 'cmake --build build && ctest' # rebuild + test

# ── DEBUG ──
gdb -tui ./myapp                                      # TUI debugger
raddebugger ./myapp.exe                                # fast native debugger

# ── PROFILE ──
tracy-profiler & ./myapp                              # Tracy profiler
hyperfine --warmup 5 './myapp'                        # benchmark

# ── LINT ──
cppcheck --enable=all src/                            # static analysis
clang-tidy -p build src/main.cpp                      # clang-tidy
clang-format -i src/main.cpp                          # format

# ── SEARCH ──
rg "TODO" --type cpp                                  # todos
rg "new\s+\w+" --type cpp                             # raw pointers
fd -e cpp | fzf | xargs nvim                          # fuzzy open

# ── CROSS ──
zig c++ -target x86_64-linux-gnu main.cpp              # cross-compile
qemu-x86_64 ./app_linux                                # test

# ── BINARY ──
strip myapp.exe && upx --best myapp.exe                # compress
depends myapp.exe                                      # DLL check
strings myapp.exe | rg "secret"                        # leak check

# ── META ──
scc src/                                              # lines of code
git log -S "function_name" --oneline                   # when was this added?
```

---

## 🔗 See Also

- [cmake-presets.md](cmake-presets.md) — full preset examples
- [cross-compilation.md](cross-compilation.md) — deep dive on zig + qemu + android
- [debugging-profiling.md](debugging-profiling.md) — full Tracy instrumentation
- [static-analysis.md](static-analysis.md) — all clang-tidy check groups
- [windows-cpp.md](windows-cpp.md) — Windows-specific quirks

---

Built for C++ dev on Windows. Tools via [scoop](https://scoop.sh).
