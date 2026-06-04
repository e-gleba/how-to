# just — Command Runner

> Stop typing `cmake --build --preset ...`. Write this once.

## justfile (copy to project root)

```makefile
# Run `just` to list all recipes
default:
    @just --list

# ── Build ──
build preset="dev":
    cmake --preset {{preset}}
    cmake --build --preset {{preset}}

rebuild preset="dev":
    cmake --build build/{{preset}}

clean preset="dev":
    rm -rf build/{{preset}}

# ── Test ──
test preset="dev":
    ctest --preset {{preset}} --output-on-failure

test-filter filter preset="dev":
    ctest --preset {{preset}} -R "{{filter}}" --output-on-failure

# ── Watch ──
watch preset="dev":
    watchexec -e cpp,hpp,h,c,cmake,just -- just rebuild {{preset}}

watch-test preset="dev":
    watchexec -e cpp,hpp,h,c,cmake,just -- just test {{preset}}

# ── Code Quality ──
lint:
    cppcheck --enable=all --suppress=missingIncludeSystem src/ --error-exitcode=1

format-check:
    clang-format --dry-run -Werror src/**/*.cpp src/**/*.hpp

format:
    clang-format -i src/**/*.cpp src/**/*.hpp

tidy preset="dev":
    clang-tidy -p build/{{preset}} src/**/*.cpp

# ── Benchmark ──
bench app="myapp" preset="dev":
    cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 --min-runs 20 "./build/{{preset}}/{{app}}"

bench-compare app="myapp" preset="dev" old="main" new="HEAD":
    @git stash
    @git checkout {{old}}
    @cmake --build build/{{preset}} --target {{app}}
    @cp build/{{preset}}/{{app}} /tmp/app_old
    @git checkout {{new}}
    @git stash pop || true
    @cmake --build build/{{preset}} --target {{app}}
    hyperfine --warmup 5 "/tmp/app_old" "./build/{{preset}}/{{app}}"

# ── Cross ──
build-linux:
    cmake --preset linux-cross
    cmake --build --preset linux-cross

test-linux app="myapp":
    qemu-x86_64 ./build/linux/{{app}}

# ── Docs ──
docs:
    doxygen Doxyfile

# ── Utilities ──
todos:
    rg "TODO|FIXME|HACK|XXX" --type cpp -n

loc:
    scc src/

clean-all:
    rm -rf build/ .cache/
```

## Usage

```bash
just                    # list all recipes
just build preset=clang # build with specific preset
just watch              # auto-rebuild on file change
just lint               # run cppcheck
just todos              # what needs work?
just bench              # benchmark
just --dry-run build    # see what would run
```

## Tips

- **Default values:** `preset="dev"` — runs with dev if not specified
- **Variables:** `{{preset}}` — passed as `just build preset=clang`
- **No .PHONY:** every recipe is a real command, no Makefile weirdness
- **Shell:** uses `sh` by default. For PowerShell, add `set windows-shell := ["pwsh.exe", "-NoLogo", "-Command"]`
