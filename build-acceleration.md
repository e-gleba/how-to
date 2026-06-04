# Build Acceleration

> Faster rebuilds = faster iteration.

## ccache

```bash
scoop install ccache

ccache -s              # stats
ccache --max-size 10G   # increase cache
ccache -C              # clear cache
ccache -z              # zero stats (before benchmark)
```

**CMake:** add to presets cacheVariables:

```json
"CMAKE_C_COMPILER_LAUNCHER": "ccache",
"CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
```

**Verify:** build twice. `ccache -s` should show cache hits on second build.

## sccache

Like ccache but with cloud backends (S3, GCS). Good for CI.

```bash
scoop install sccache
# Same CMake variables as ccache
```

## Ninja

```bash
scoop install ninja
cmake -B build -G Ninja
```

Faster than Make because:
- No recursive-make problems
- Better parallel job scheduling
- Native `compile_commands.json` support

> 💡 Never use `-j` with Ninja — it auto-detects core count.

## Precompiled headers

```cmake
target_precompile_headers(myapp PRIVATE
    <vector>
    <string>
    <memory>
    "pch.hpp"
)
```

Especially effective with MSVC.

## Unity builds (CI only)

```cmake
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16)
```

Full rebuild of ~500 .cpp: 3:20 → 0:45.

⚠️ Don't use for incremental dev — hides ODR violations, macros leak across files.

## Expected speedups

| Technique | Cold build | Hot rebuild (1 file changed) |
|-----------|-----------|------------------------------|
| Baseline | 100% | 100% |
| + Ninja | ~80% | ~70% |
| + ccache | ~80% | **~15%** |
| + PCH | ~60% | ~50% |
| + Unity | ~25% | n/a (full rebuild) |

> 💡 **Daily dev:** Ninja + ccache + PCH. **CI:** + Unity.
