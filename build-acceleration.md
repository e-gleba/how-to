# Build Acceleration

> Faster rebuilds = faster iteration.

## ccache

```bash
# Install:
scoop install ccache            # Windows
sudo apt install ccache         # Linux
brew install ccache             # macOS

ccache -s                       # stats
ccache --max-size 10G           # increase cache
ccache -C                       # clear cache
ccache -z                       # zero stats (before benchmark)
```

**CMake:** add to configure:
```bash
cmake -B build -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
```

**Verify:** build twice. `ccache -s` should show cache hits on second build.

## sccache

Like ccache but with cloud backends (S3, GCS). Good for CI.

```bash
scoop install sccache           # Windows
cargo install sccache           # Linux
brew install sccache            # macOS
# Same CMake variables as ccache
```

## Ninja

```bash
scoop install ninja             # Windows
sudo apt install ninja-build    # Linux
brew install ninja              # macOS

cmake -B build -G Ninja
```

Faster than Make because:
- No recursive-make problems
- Better parallel job scheduling
- Native `compile_commands.json` support

> 💡 Never use `-j` with Ninja — it auto-detects core count.

## Precompiled Headers

```cmake
target_precompile_headers(myapp PRIVATE
    <vector>
    <string>
    <memory>
    "pch.hpp"
)
```

Especially effective with MSVC.

## Unity Builds (CI only)

```cmake
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16)
```

Full rebuild of ~500 .cpp: 3:20 → 0:45.

⚠️ Don't use for incremental dev — hides ODR violations, macros leak across files.

## Expected Speedups

| Technique | Cold build | Hot rebuild (1 file changed) |
|-----------|-----------|------------------------------|
| Baseline | 100% | 100% |
| + Ninja | ~80% | ~70% |
| + ccache | ~80% | **~15%** |
| + PCH | ~60% | ~50% |
| + Unity | ~25% | n/a (full rebuild) |

> 💡 **Daily dev:** Ninja + ccache + PCH. **CI:** + Unity.

→ **CMake one-liners**: [cmake.md](cmake.md) — configure with ccache in one line
→ **Install**: [tools-install.md](tools-install.md) — install ccache, ninja, sccache
