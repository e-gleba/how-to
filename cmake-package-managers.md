# Package Managers

## vcpkg (manifest mode)

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

```bash
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
```

```cmake
# Then find_package normally:
find_package(fmt CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

### Classic mode

```bash
vcpkg install fmt:x64-windows
vcpkg install spdlog:x64-windows-static   # static
vcpkg list                                # installed
vcpkg search sqlite                       # available
```

### Triplets

- `x64-windows` — dynamic CRT, DLLs
- `x64-windows-static` — static CRT, static libs
- `x64-windows-static-md` — dynamic CRT, static libs

## Conan 2

```python
# conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMake, CMakeToolchain, CMakeDeps, cmake_layout

class MyApp(ConanFile):
    name = "myapp"
    version = "1.0"
    settings = "os", "compiler", "build_type", "arch"

    def requirements(self):
        self.requires("fmt/11.0.2")
        self.requires("spdlog/1.14.1")

    def layout(self):
        cmake_layout(self)

    def generate(self):
        CMakeToolchain(self).generate()
        CMakeDeps(self).generate()

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()
```

```bash
conan profile detect
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
cmake --build build
```

## FetchContent (no package manager)

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

Good for header-only libs. Avoid for large compiled libs.

## CPM.cmake (lightweight)

```cmake
# Drop CPM.cmake into cmake/CPM.cmake
include(cmake/CPM.cmake)
CPMAddPackage("gh:fmtlib/fmt#11.0.2")
CPMAddPackage("gh:gabime/spdlog#v1.14.1")
```

## Comparison

| | vcpkg | Conan 2 |
|---|-------|---------|
| Approach | Central catalog | Decentralized |
| CMake | Toolchain file | Generated toolchain |
| Cross-compile | Triplets | Profiles + settings |
| Custom packages | Overlay ports | conanfile.py export |
| Best for | Simple, MSVC | Complex, cross-platform |

## Version pinning

```bash
# vcpkg: baseline in vcpkg.json
"builtin-baseline": "abc123def456..."

# Conan: lockfiles
conan lock create conanfile.py
conan install . --lockfile=conan.lock
```

> 💡 **Tip:** Pick one package manager. Mixing = ABI hell. Pin versions. Never use `master` branch.
