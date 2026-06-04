# just — Command Runner

`just` is a command runner — like Make but without the build-system baggage. No `.PHONY`, no weird syntax, just commands.

## Basic justfile

```makefile
# justfile — place in project root

# Default recipe (runs when you type `just`)
default:
    @just --list

# === Build ===

# Configure + build with preset
build preset="win-clang":
    cmake --preset {{preset}}
    cmake --build --preset {{preset}}

# Rebuild only changed files (Ninja does this automatically)
rebuild preset="win-clang":
    cmake --build build/{{preset}}

# Clean build directory
clean preset="win-clang":
    rm -rf build/{{preset}}

# === Test ===

test preset="win-clang":
    ctest --preset {{preset}} --output-on-failure

# Run specific test by regex
test-filter filter preset="win-clang":
    ctest --preset {{preset}} -R "{{filter}}" --output-on-failure

# === Watch ===

# Watch source files, rebuild on change
watch preset="win-clang":
    watchexec -e cpp,hpp,h,c,cmake,just -- just rebuild {{preset}}

# Watch + rebuild + run tests
watch-test preset="win-clang":
    watchexec -e cpp,hpp,h,c,cmake,just -- just rebuild {{preset}} && just test {{preset}}

# === Code Quality ===

lint:
    cppcheck --enable=all --suppress=missingIncludeSystem src/ --error-exitcode=1

lint-file file:
    cppcheck --enable=all --suppress=missingIncludeSystem {{file}}

tidy preset="win-clang":
    clang-tidy -p build/{{preset}} src/**/*.cpp

format-check:
    clang-format --dry-run -Werror src/**/*.cpp src/**/*.hpp

format:
    clang-format -i src/**/*.cpp src/**/*.hpp

# === Profiling ===

profile preset="win-clang" app="myapp":
    cmake --build build/{{preset}} --target {{app}}
    tracy-profiler &
    ./build/{{preset}}/{{app}}

# === Benchmarking ===

bench preset="win-clang" app="myapp":
    cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 --min-runs 20 "./build/{{preset}}/{{app}}"

bench-compare preset="win-clang" app="myapp" old="main" new="HEAD":
    git stash
    git checkout {{old}}
    cmake --build build/{{preset}} --target {{app}}
    cp build/{{preset}}/{{app}} /tmp/app_old
    git checkout {{new}}
    git stash pop || true
    cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 "/tmp/app_old" "./build/{{preset}}/{{app}}"

# === Cross-compilation ===

build-linux:
    cmake --preset linux-cross
    cmake --build --preset linux-cross

test-linux app="myapp":
    qemu-x86_64 ./build/linux-cross/{{app}}

# === Docs ===

docs:
    doxygen Doxyfile

docs-open:
    doxygen Doxyfile && start html/index.html

# === Utilities ===

# Find TODOs in source
todos:
    rg "TODO|FIXME|HACK|XXX" --type cpp -n

# Count lines of code
loc:
    scc src/

# Show dependency graph
deps preset="win-clang":
    cmake --graphviz=build/{{preset}}/graph.dot build/{{preset}}
    dot -Tpng build/{{preset}}/graph.dot -o deps.png
    start deps.png

# Clean everything
clean-all:
    rm -rf build/
    rm -rf .cache/
```

## Tips

- Variables: `{{preset}}` — passed as `just build preset=win-clang`
- Default values: `preset="win-clang"` — fallback if not passed
- List recipes: `just -l` or just `just` (if you add the default recipe)
- Dry run: `just --dry-run build` — prints commands without running
- Shell: uses `sh` on Unix, can set `set windows-shell := ["pwsh.exe", "-NoLogo", "-Command"]` for PowerShell

## Install

```bash
scoop install just
```
