# 🛠️ C++ Dev Handbook

> Windows · CMake · Cross-platform · Grab-and-go reference

Quick reference for C++ development on Windows. Everything here uses tools already installed via `scoop`.

---

## 📋 Quick Tips

| # | Tip | Command |
|---|-----|---------|
| 1 | Use Ninja, not Make | `cmake -B build -G Ninja` |
| 2 | Cache compiles | `-DCMAKE_CXX_COMPILER_LAUNCHER=ccache` |
| 3 | Never type build commands | Write a `justfile`, then `just build` |
| 4 | Export compile_commands.json | `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` |
| 5 | Symlink it to project root | `ln -s build/compile_commands.json .` |
| 6 | Watch files, auto-rebuild | `watchexec -e cpp,hpp -- cmake --build build` |
| 7 | Cross-compile to Linux | `zig c++ -target x86_64-linux-gnu main.cpp` |
| 8 | Test Linux binary on Windows | `qemu-x86_64 ./myapp_linux` |
| 9 | Benchmark properly | `hyperfine --warmup 5 './myapp'` |
| 10 | Search code, not grep | `rg "pattern" --type cpp` |
| 11 | Instrument with Tracy | `#include <Tracy.hpp>` + `ZoneScoped;` |
| 12 | Sanitizers on Clang/GCC | `-fsanitize=address,undefined` |

---

## 📖 Read the Handbook

**[→ HANDBOOK.md](HANDBOOK.md)** — all-in-one, everything in a single file.

Or jump to a specific topic:

| File | Topic |
|------|-------|
| [cmake-presets.md](cmake-presets.md) | Presets for MSVC, Clang, GCC, Zig cross |
| [justfile.md](justfile.md) | Task runner — build, test, watch, bench, lint |
| [build-acceleration.md](build-acceleration.md) | ccache, sccache, Ninja, unity builds, pch |
| [cross-compilation.md](cross-compilation.md) | Zig, QEMU, Android NDK, MSYS2 |
| [debugging-profiling.md](debugging-profiling.md) | GDB, raddebugger, x64dbg, Tracy, Perfetto |
| [static-analysis.md](static-analysis.md) | cppcheck, clang-tidy, ast-grep, clang-format |
| [search-navigation.md](search-navigation.md) | ripgrep, fd, fzf, bat, ugrep |
| [benchmarking.md](benchmarking.md) | hyperfine, Google Benchmark, Tracy |
| [binary-tools.md](binary-tools.md) | UPX, pe-bear, ImHex, Ghidra, radare2 |
| [windows-cpp.md](windows-cpp.md) | Toolchains, CRT, PDB, Windows APIs |
| [cmake-package-managers.md](cmake-package-managers.md) | vcpkg, Conan 2, FetchContent, CPM |

---

## ⚡ One-liners I Use Daily

```bash
# Rebuild only what changed
cmake --build build

# Find all TODOs with context
rg "TODO|FIXME|HACK" --type cpp -C 2

# Find files with most git churn
git log --format=format: --name-only | rg "." | sort | uniq -c | sort -nr | head -20

# Time a benchmark properly
hyperfine --warmup 5 --min-runs 20 './build/myapp --bench'

# Watch source, rebuild on save
watchexec -e cpp,hpp,h,c,cmake -- 'cmake --build build && ctest'

# List CMake presets
cmake --list-presets

# Cross-compile single file to Linux
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux

# Check what DLLs a binary needs
depends ./build/myapp.exe

# Compress release binary
strip myapp.exe && upx --best myapp.exe

# Find raw owning pointers (candidates for smart ptr)
rg 'new\s+\w+' --type cpp

# Count lines of code
scc src/
```

---

## 🧰 Tool Map

```
┌─ BUILD ──────────────────────────────────────────┐
│  cmake  →  ninja  →  ccache / sccache            │
│  just (task runner)  +  watchexec (file watch)   │
│  vcpkg / conan (packages)                        │
├─ CODE ───────────────────────────────────────────┤
│  cppcheck  +  clang-tidy  +  ast-grep            │
│  clang-format                                    │
│  ripgrep / fd / fzf (search & navigate)          │
├─ DEBUG ──────────────────────────────────────────┤
│  gdb  │  raddebugger  │  x64dbg                  │
│  tracy (profiler)  │  perfetto (tracing)         │
├─ BINARY ─────────────────────────────────────────┤
│  pe-bear  │  depends  │  ImHex  │  Ghidra         │
│  upx  │  radare2 / cutter                        │
├─ CROSS ──────────────────────────────────────────┤
│  zig (cross-compiler)  +  qemu (test)            │
│  msys2 (POSIX env)  │  android-clt (NDK)         │
└──────────────────────────────────────────────────┘
```

---

## 🚀 Setup In 5 Minutes

```powershell
# 1. Install essentials (if missing)
scoop install cmake ninja ccache zig just watchexec ripgrep fd bat fzf hyperfine

# 2. Drop this justfile in your project root
#    (copy from justfile.md)

# 3. Drop CMakePresets.json in your project root
#    (copy from cmake-presets.md)

# 4. Symlink compile_commands.json
cmake --preset dev
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build/dev/compile_commands.json

# 5. Go
just watch
```

---

Built from real C++ dev on Windows. Tools via [scoop](https://scoop.sh).
