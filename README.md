# how-to

> C++ dev handbook — grab-and-go reference for Windows

**[→ HANDBOOK.md](HANDBOOK.md)** — all-in-one. Commands, presets, tips, daily patterns.

---

## Quick start

```bash
git clone https://github.com/e-gleba/how-to.git
bat HANDBOOK.md
```

## What's inside

| Section | Covers |
|---------|--------|
| ripgrep | File types, custom types, daily patterns, replace, git-aware search |
| Build | CMake presets, justfile, ccache, Ninja, pch, unity builds |
| Debug | GDB, raddebugger, sanitizers (ASan + UBSan) |
| Profile | Tracy instrumentation, hyperfine benchmarks |
| Static analysis | cppcheck, clang-tidy, clang-format, ast-grep |
| Navigation | fd, fzf, bat, scc |
| Packages | vcpkg manifest mode, Conan 2 conanfile.py |
| Windows | Toolchain picks, CRT linking, PDB, Defender exclusions |
| Binary tools | UPX, pe-bear, depends, ImHex, pre-ship checklist |
| Cross-compile | Zig + QEMU (optional) |

## Tools assumed

```bash
scoop install cmake ninja ccache just watchexec ripgrep fd bat fzf hyperfine cppcheck llvm
```

---

Companion deep-dives: [cmake-presets.md](cmake-presets.md) · [justfile.md](justfile.md) · [build-acceleration.md](build-acceleration.md) · [debugging-profiling.md](debugging-profiling.md) · [static-analysis.md](static-analysis.md) · [search-navigation.md](search-navigation.md) · [benchmarking.md](benchmarking.md) · [binary-tools.md](binary-tools.md) · [windows-cpp.md](windows-cpp.md) · [cmake-package-managers.md](cmake-package-managers.md) · [cross-compilation.md](cross-compilation.md)
