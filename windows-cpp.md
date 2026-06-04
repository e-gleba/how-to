# Windows C++ Development

## Toolchains

You have three toolchains installed:

| Toolchain | Compiler | Path | Best for |
|-----------|----------|------|----------|
| **MSVC** (via vcredist) | `cl.exe` | Visual Studio Build Tools | Windows-native, best PDB support |
| **LLVM/Clang** | `clang-cl.exe` | `scoop/apps/llvm/current/bin` | Cross-platform parity, sanitizers |
| **GCC** (MinGW via MSYS2) | `gcc.exe` / `g++.exe` | `scoop/apps/msys2/current` | Linux-like behavior, POSIX APIs |
| **Zig** (cross) | `zig c++` | `scoop/apps/zig/current` | Cross-compilation to Linux/macOS |

## MSVC-specific tips

```bash
# Clang-cl is compatible with MSVC ABI — use it as drop-in replacement
clang-cl /EHsc /std:c++20 main.cpp

# MSVC flags to Clang-cl mapping:
# /O2    → -O2
# /Zi    → -g  (debug info in PDB)
# /EHsc  → same (exception handling)
# /MD    → same (dynamic CRT)
# /MT    → same (static CRT)

# Use clang-cl in CMake:
cmake -B build -G Ninja -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl
```

## CRT linking

```cmake
# Choose CRT at CMake level
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")     # /MT or /MTd
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")  # /MD or /MDd

# Static CRT = no DLL dependency, larger binary
# Dynamic CRT = smaller binary, requires vcredist
```

## PDB / Debug symbols

```cmake
# Generate PDB even for release builds (useful for crash dumps)
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
```

```bash
# List PDB contents
# Use Visual Studio's dumpbin or:
llvm-pdbutil dump --all myapp.pdb
```

## Windows-specific APIs

```cpp
// High-resolution timer (better than std::chrono on Windows)
#include <profileapi.h>

LARGE_INTEGER freq, start, end;
QueryPerformanceFrequency(&freq);
QueryPerformanceCounter(&start);
// ... work ...
QueryPerformanceCounter(&end);
double elapsed_ms = (end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
```

```cpp
// Memory-mapped files (fast I/O)
#include <windows.h>

HANDLE hFile = CreateFileA(path, GENERIC_READ, ...);
HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
void* data = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
// ... use data ...
UnmapViewOfFile(data);
CloseHandle(hMapping);
CloseHandle(hFile);
```

```cpp
// Windows sockets (Winsock2)
#include <winsock2.h>
#pragma comment(lib, "ws2_32.lib")

WSADATA wsa;
WSAStartup(MAKEWORD(2, 2), &wsa);
SOCKET s = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
// ...
closesocket(s);
WSACleanup();
```

## MSYS2 for POSIX compatibility

```bash
# Start MSYS2 shell
# C:\Users\YOU\scoop\apps\msys2\current\msys2_shell.cmd

# It provides:
# - bash shell
# - POSIX paths (/c/Users/YOU/project)
# - Linux-like headers
# - autotools/build-essential
#
# Use when source expects Linux environment
# (configure scripts, Makefiles, etc.)
```

## Case-insensitive filesystem

Windows filesystem is case-insensitive (usually). This causes issues:

```bash
# Git can help
# .gitconfig
[core]
    ignorecase = false  # make Git case-sensitive
```

```cmake
# CMakeLists.txt — checks to catch case mismatches
# #include "MyHeader.h" when actual file is "myheader.h"
# Won't error on Windows, will fail on Linux
```

## Path handling

```cpp
// Use forward slashes everywhere — Windows accepts them
#include "src/utils/helpers.h"  // ✓ works on Windows
#include "src\\utils\\helpers.h"  // ✗ don't do this

// For filesystem operations, use std::filesystem
#include <filesystem>
namespace fs = std::filesystem;
fs::path p = "src/utils";
p.make_preferred();  // converts / to \ on Windows
```

## Console / Terminal

```bash
# Windows Terminal (install from Microsoft Store) + pwsh
# Better than cmd.exe:
# - TrueColor support
# - Tabs, panes
# - GPU-accelerated rendering
# - UTF-8 by default

# Set UTF-8 globally (recommended):
# Control Panel → Region → Administrative → Change system locale → UTF-8
```

## File watching (FS notifications)

```bash
# watchexec uses OS-level file watching (ReadDirectoryChangesW on Windows)
watchexec -e cpp,hpp -- cmake --build build

# Alternative: cargo watch (Rust-based, similar)
```

## Environment variables for compilers

```powershell
# PowerShell profile ($PROFILE)
# Set up compiler paths

$env:CCACHE_DIR = "D:\ccache"  # move cache off SSD if needed
$env:CCACHE_MAXSIZE = "10G"
```

## Windows Defender exclusions

Build directories can be slow due to real-time scanning:

```powershell
# Add exclusion for build folders (PowerShell as Admin)
Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"
# Or via: Windows Security → Virus & threat protection → Exclusions
```

## Long path support

```powershell
# Enable long paths (PowerShell as Admin)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

Then in CMake:
```cmake
# Manifest setting for long path awareness
# In your app manifest:
# <application xmlns="urn:schemas-microsoft-com:asm.v3">
#   <windowsSettings>
#     <longPathAware xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">true</longPathAware>
#   </windowsSettings>
# </application>
```

## Debug vs Release

```cmake
# Debug: full symbols, no optimizations, asserts enabled
# RelWithDebInfo: optimizations + symbols (for profiling)
# Release: full optimizations, no symbols
# MinSizeRel: optimize for size

# Recommendation for daily dev: RelWithDebInfo
# - Fast enough to use
# - Debug symbols for crash investigation
# - With Tracy: profiling works on optimized code (more realistic)
```
