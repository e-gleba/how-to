# CMake Presets

> Reference card. Copy, paste, adapt.

## Minimal preset

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
    }
  ],
  "buildPresets": [{"name": "dev", "configurePreset": "dev"}],
  "testPresets": [{"name": "dev", "configurePreset": "dev", "output": {"outputOnFailure": true}}]
}
```

## Multi-compiler

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "win-msvc",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/msvc",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "cl",
        "CMAKE_CXX_COMPILER": "cl",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      },
      "condition": {"type": "equals", "lhs": "${hostSystemName}", "rhs": "Windows"}
    },
    {
      "name": "win-clang",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/clang",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "clang-cl",
        "CMAKE_CXX_COMPILER": "clang-cl",
        "CMAKE_CXX_CLANG_TIDY": "clang-tidy;--header-filter=${sourceDir}/*",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      },
      "condition": {"type": "equals", "lhs": "${hostSystemName}", "rhs": "Windows"}
    },
    {
      "name": "gcc",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/gcc",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "gcc",
        "CMAKE_CXX_COMPILER": "g++",
        "CMAKE_CXX_COMPILER_LAUNCHER": "ccache",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
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
    {"name": "win-msvc", "configurePreset": "win-msvc"},
    {"name": "win-clang", "configurePreset": "win-clang"},
    {"name": "gcc", "configurePreset": "gcc"},
    {"name": "linux-cross", "configurePreset": "linux-cross"}
  ]
}
```

## Commands

```bash
cmake --list-presets           # what's available?
cmake --preset win-clang       # configure
cmake --build --preset win-clang  # build
ctest --preset win-clang       # test
cmake --graphviz=build/graph.dot . && dot -Tpng graph.dot -o deps.png  # dependency graph
```

## User presets (gitignored)

```json
// CMakeUserPresets.json — don't commit
{
  "version": 6,
  "configurePresets": [{
    "name": "my",
    "inherits": "win-clang",
    "cacheVariables": {
      "CMAKE_BUILD_TYPE": "RelWithDebInfo",
      "MY_FLAG": "ON"
    }
  }],
  "buildPresets": [{"name": "my", "configurePreset": "my"}]
}
```

## Sanitizers

```json
// Inside cacheVariables:
"CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer",
"CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined"
```

## Tips

- **Always** export `compile_commands.json` — needed by clangd, clang-tidy
- **Symlink** it to project root: `ln -s build/dev/compile_commands.json .`
- **Ninja** over Make — better parallel scheduling, no recursive-make issues
- **Use presets** even for single-compiler projects — self-documenting, CI-ready
- **Unity builds** for CI speed: `CMAKE_UNITY_BUILD=ON`, `CMAKE_UNITY_BUILD_BATCH_SIZE=16`
