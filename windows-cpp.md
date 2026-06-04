# Windows C++ Development

## Toolchains

| Compiler | Path | Best for |
|----------|------|----------|
| MSVC (`cl`) | VS Build Tools | Windows-native, PDB support |
| Clang (`clang-cl`) | `scoop/apps/llvm/current/bin` | Cross-platform parity, sanitizers |
| GCC (`g++`) | `scoop/apps/msys2/current` | Linux-like, POSIX APIs |
| Zig (`zig c++`) | `scoop/apps/zig/current` | Cross-compilation |

## CRT linking

```cmake
# Static CRT (/MT, /MTd) — larger binary, no DLL dependency
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# Dynamic CRT (/MD, /MDd) — smaller binary, needs vcredist
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
```

## PDB in release builds

```cmake
# Symbols for crash dumps even in release
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
```

## Path handling

```cpp
// Use forward slashes — Windows accepts them everywhere
#include "src/utils/helpers.h"      // ✓
#include "src\\utils\\helpers.h"    // ✗

// std::filesystem handles conversion
#include <filesystem>
std::filesystem::path p = "src/utils";
p.make_preferred();  // / → \ on Windows
```

## Windows performance APIs

```cpp
// High-res timer (better than chrono on Windows)
#include <profileapi.h>
LARGE_INTEGER freq, start, end;
QueryPerformanceFrequency(&freq);
QueryPerformanceCounter(&start);
// ... work ...
QueryPerformanceCounter(&end);
double ms = (end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
```

```cpp
// Memory-mapped files
#include <windows.h>
HANDLE h = CreateFileA(path, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
HANDLE m = CreateFileMappingA(h, NULL, PAGE_READONLY, 0, 0, NULL);
void* data = MapViewOfFile(m, FILE_MAP_READ, 0, 0, 0);
// ... use data ...
UnmapViewOfFile(data); CloseHandle(m); CloseHandle(h);
```

## Environment tips

- **Windows Terminal** (from Store) + pwsh — TrueColor, tabs, GPU rendering
- **UTF-8 globally:** Control Panel → Region → Administrative → Change system locale → UTF-8
- **Defender exclusion:** `Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"`
- **Long paths:** `New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1`
- **Case sensitivity:** `git config core.ignorecase false`
- **Forward slashes** everywhere — Windows accepts them, Linux requires them

## Recommended build type for daily dev

```cmake
# RelWithDebInfo — fast + symbols + profiling works on real code
set(CMAKE_BUILD_TYPE "RelWithDebInfo")
```

Not Debug (slow), not Release (no symbols). RelWithDebInfo is the sweet spot.

> 💡 **Tip:** Use `clang-cl` as a drop-in MSVC replacement. Same ABI, same flags, but adds sanitizers and better error messages.
