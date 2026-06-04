# CMake Package Managers

## vcpkg

Microsoft's package manager. Manifest mode is preferred.

### Setup

```bash
# Already installed via scoop: vcpkg
vcpkg --version

# Integrate with MSBuild (if using VS)
vcpkg integrate install
```

### Manifest mode

```json
// vcpkg.json — place in project root
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": [
    "fmt",
    "spdlog",
    "nlohmann-json",
    "catch2",
    {
      "name": "imgui",
      "features": ["docking-experimental"]
    }
  ]
}
```

```cmake
# CMakeLists.txt
# Point CMake to vcpkg toolchain
# Via CMakePresets.json:
# "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"

# Or command line:
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"

# Then use find_package normally:
find_package(fmt CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)

find_package(spdlog CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE spdlog::spdlog)
```

### Classic mode (global)

```bash
vcpkg install fmt:x64-windows
vcpkg install spdlog:x64-windows-static  # static linking

# List installed
vcpkg list

# Search available
vcpkg search sqlite
```

### Triplets

```bash
# Dynamic linking (DLL): x64-windows
# Static linking (lib): x64-windows-static
# Static CRT: x64-windows-static-md

# Set triplet in vcpkg.json:
{
  "default-features": [],
  "builtin-baseline": "...",
  "dependencies": ["fmt"]
}

# Or via CMake:
set(VCPKG_TARGET_TRIPLET "x64-windows-static")
```

## Conan 2

Decentralized package manager. Conan 2 has major improvements over 1.x.

### Setup

```bash
# Already installed via scoop: conan
conan --version

# Detect your profile
conan profile detect
# Creates ~/.conan2/profiles/default
```

### conanfile.py

```python
# conanfile.py — place in project root
from conan import ConanFile
from conan.tools.cmake import CMake, CMakeToolchain, CMakeDeps, cmake_layout

class MyApp(ConanFile):
    name = "myapp"
    version = "1.0"
    settings = "os", "compiler", "build_type", "arch"

    # Binary dependencies
    def requirements(self):
        self.requires("fmt/11.0.2")
        self.requires("spdlog/1.14.1")
        self.requires("nlohmann_json/3.11.3")

    # CMake integration
    def layout(self):
        cmake_layout(self)

    def generate(self):
        tc = CMakeToolchain(self)
        tc.generate()
        deps = CMakeDeps(self)
        deps.generate()

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()
```

### conanfile.txt (simpler)

```ini
[requires]
fmt/11.0.2
spdlog/1.14.1

[generators]
CMakeToolchain
CMakeDeps

[layout]
cmake_layout
```

### Commands

```bash
# Install dependencies
conan install . --output-folder=build --build=missing

# Configure CMake (uses generated toolchain)
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake

# Build
cmake --build build

# Or let Conan do both:
conan build .

# Create package
conan create .
```

### Conan 2 improvements over 1.x

- `CMakeToolchain` + `CMakeDeps` replace `cmake` generator
- `self.requires()` replaces `self.requires =`
- `layout()` method for build directory structure
- Better lockfile support for reproducible builds
- `conan list` replaces `conan search`

## vcpkg vs Conan

| | vcpkg | Conan |
|---|-------|-------|
| **Approach** | Centralized catalog | Decentralized (conancenter + custom) |
| **CMake integration** | Toolchain file | Generated toolchain + find modules |
| **Cross-compilation** | Triplet system | Profiles + settings |
| **Binary caching** | Built-in | Remotes + Artifactory |
| **Custom packages** | Overlay ports | conanfile.py export |
| **Versioning** | Baseline (snapshot) | Per-package version ranges |
| **Best for** | Simple projects, MSVC | Complex projects, cross-platform |

## Using both?

Generally pick one. Mixing leads to ABI issues. If you must:

```cmake
# Use vcpkg for most, Conan for specific packages not in vcpkg
# Use separate build directories, don't mix in same CMake invocation
```

## FetchContent (no package manager)

For header-only or small libraries:

```cmake
include(FetchContent)

FetchContent_Declare(
    nlohmann_json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG v3.11.3
)
FetchContent_MakeAvailable(nlohmann_json)

target_link_libraries(myapp PRIVATE nlohmann_json::nlohmann_json)
```

Pro: zero setup. Con: no version management, downloads on every clean build.

## CPM.cmake (lightweight wrapper)

```cmake
# CPM.cmake — single-file drop-in
# Download: https://github.com/cpm-cmake/CPM.cmake
include(cmake/CPM.cmake)

CPMAddPackage("gh:fmtlib/fmt#11.0.2")
CPMAddPackage("gh:gabime/spdlog#v1.14.1")

# Each provides CMake targets automatically
```

## Version pinning (important!)

```bash
# vcpkg: use baseline in vcpkg.json
{
  "builtin-baseline": "abc123def456..."
}

# Conan: use lockfiles
conan lock create conanfile.py
conan install . --lockfile=conan.lock

# FetchContent: pin to specific tags (GIT_TAG)
FetchContent_Declare(..., GIT_TAG v3.11.3)
```

Never use `master`/`main` branch — breaks reproducibility.
