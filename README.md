# how-to

Personal C++ development cheatsheet. Windows-first, cross-platform aware.

Focus: **CMake, build acceleration, cross-compilation, debugging, profiling, static analysis, benchmarking.**

---

## Index

| File | Topic |
|------|-------|
| [cmake-presets.md](cmake-presets.md) | CMakePresets.json — multi-config, multi-platform presets |
| [justfile.md](justfile.md) | Command runner — build, test, watch, bench tasks |
| [cross-compilation.md](cross-compilation.md) | Zig as cross-compiler, QEMU testing, Linux/Android targets from Windows |
| [build-acceleration.md](build-acceleration.md) | ccache, sccache, Ninja — faster rebuilds |
| [debugging-profiling.md](debugging-profiling.md) | GDB, raddebugger, x64dbg, Tracy, Perfetto, RenderDoc |
| [static-analysis.md](static-analysis.md) | cppcheck, clang-tidy, ast-grep — catch bugs before runtime |
| [search-navigation.md](search-navigation.md) | ripgrep, fd, fzf, bat, ugrep — codebase exploration combos |
| [benchmarking.md](benchmarking.md) | hyperfine + Tracy — measure everything |
| [binary-tools.md](binary-tools.md) | UPX, pe-bear, ImHex, depends, Sysinternals |
| [windows-cpp.md](windows-cpp.md) | Windows-specific C++ tips, toolchain quirks, MSYS2 vs native |
| [cmake-package-managers.md](cmake-package-managers.md) | vcpkg + conan integration patterns |

---

## Quick start

```bash
# Clone and browse
git clone https://github.com/e-gleba/how-to.git
cd how-to

# Read any file
bat cmake-presets.md
```

## Tools assumed installed

```
scoop install cmake ninja ccache sccache zig just watchexec ripgrep fd bat fzf hyperfine cppcheck doxygen gdb
```

LLVM (clang-tidy, clang-format, clangd) via scoop: `scoop install llvm`

---

Generated from real C++ dev workflows on Windows. PRs welcome.
