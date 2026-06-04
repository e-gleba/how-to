# CMake Presets

## Minimal preset

```json
// CMakePresets.json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "dev",
      "displayName": "Development (Ninja + ccache)",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "CMAKE_C_COMPILER_LAUNCHER": "ccache",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "dev",
      "configurePreset": "dev"
    }
  ],
  "testPresets": [
    {
      "name": "dev",
      "configurePreset": "dev",
      "output": {"outputOnFailure": true}
    }
  ]
}
```

## Multi-compiler presets

```json
// CMakePresets.json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "win-msvc",
      "displayName": "Windows MSVC",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/msvc",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "cl",
        "CMAKE_CXX_COMPILER": "cl",
        "CMAKE_C_COMPILER_LAUNCHER": "ccache",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      },
      "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Windows"
      }
    },
    {
      "name": "win-clang",
      "displayName": "Windows Clang",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/clang",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "clang-cl",
        "CMAKE_CXX_COMPILER": "clang-cl",
        "CMAKE_CXX_CLANG_TIDY": "clang-tidy;--header-filter=${sourceDir}/*",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      },
      "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Windows"
      }
    },
    {
      "name": "gcc",
      "displayName": "GCC (MSYS2)",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/gcc",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "gcc",
        "CMAKE_CXX_COMPILER": "g++",
        "CMAKE_C_COMPILER_LAUNCHER": "ccache",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      }
    },
    {
      "name": "linux-cross",
      "displayName": "Cross-compile to Linux (zig)",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/linux-cross",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "zig cc",
        "CMAKE_CXX_COMPILER": "zig c++",
        "CMAKE_C_COMPILER_TARGET": "x86_64-linux-gnu",
        "CMAKE_CXX_COMPILER_TARGET": "x86_64-linux-gnu"
      }
    }
  ],
  "buildPresets": [
    {"name": "win-msvc", "configurePreset": "win-msvc"},
    {"name": "win-clang", "configurePreset": "win-clang"},
    {"name": "gcc", "configurePreset": "gcc"},
    {"name": "linux-cross", "configurePreset": "linux-cross"}
  ]
}
```

## Usage

```bash
# List all presets
cmake --list-presets

# Configure
cmake --preset win-clang

# Build
cmake --build --preset win-clang

# Or use shortcut
cmake --build build/clang

# Test
ctest --preset win-clang
```

## compile_commands.json

Always export this — needed by clangd (LSP), clang-tidy, and other tools:

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

Then symlink from build dir to project root so clangd finds it:

```powershell
# PowerShell — run from project root
New-Item -ItemType SymbolicLink -Path compile_commands.json -Target build/clang/compile_commands.json
```

## User presets (gitignored)

```json
// CMakeUserPresets.json (don't commit)
{
  "version": 6,
  "configurePresets": [
    {
      "name": "my",
      "inherits": "win-clang",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "RelWithDebInfo",
        "MY_SPECIAL_FLAG": "ON"
      }
    }
  ],
  "buildPresets": [
    {"name": "my", "configurePreset": "my"}
  ]
}
```

```gitignore
# .gitignore
CMakeUserPresets.json
```

## Dependency graph

```bash
cmake --graphviz=graph.dot build/clang
dot -Tpng graph.dot -o deps.png
```

## Sanitizers (Clang/GCC)

```json
// Inside configure preset cacheVariables:
"CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer",
"CMAKE_C_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer",
"CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined",
"CMAKE_SHARED_LINKER_FLAGS": "-fsanitize=address,undefined"
```

## Unity builds (fast full rebuilds)

```cmake
# CMakeLists.txt
set(CMAKE_UNITY_BUILD ON)
set(CMAKE_UNITY_BUILD_BATCH_SIZE 16)
```

Only use for CI or when iterating on logic (not headers). Disable for incremental dev.
