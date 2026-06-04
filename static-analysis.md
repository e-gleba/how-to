# Static Analysis

> Catch bugs before runtime.

## cppcheck

```bash
cppcheck --enable=all --suppress=missingIncludeSystem src/
cppcheck --enable=all --xml src/ 2> report.xml    # CI output
cppcheck --enable=all --inconclusive src/          # more checks, more noise
```

### CMake target

```cmake
add_custom_target(cppcheck
    COMMAND cppcheck --enable=all --suppress=missingIncludeSystem --error-exitcode=1 ${CMAKE_SOURCE_DIR}/src
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    COMMENT "Running cppcheck..."
)
```

```bash
cmake --build build --target cppcheck
```

## clang-tidy

```bash
clang-tidy -p build src/main.cpp
clang-tidy -p build --fix src/main.cpp             # auto-fix
clang-tidy -p build -checks='modernize-*,performance-*' src/**/*.cpp
```

### In CMake (runs on every compile)

```cmake
set(CMAKE_CXX_CLANG_TIDY
    clang-tidy;
    -header-filter=${CMAKE_SOURCE_DIR}/*;
    -checks=-*,clang-analyzer-*,cppcoreguidelines-*,modernize-*,performance-*,readability-*;
)
```

### Check groups

| Group | Catches |
|-------|---------|
| `clang-analyzer-*` | Memory leaks, use-after-free, null deref |
| `modernize-*` | nullptr, override, range-for, make_unique |
| `performance-*` | Unnecessary copies, inefficient patterns |
| `bugprone-*` | API misuse, branch clones |
| `cppcoreguidelines-*` | Core Guidelines violations |
| `readability-*` | Naming, redundant code, implicit casts |

## clang-format

```bash
clang-format -style=llvm -dump-config > .clang-format   # generate config
clang-format --dry-run -Werror src/**/*.cpp              # CI check
clang-format -i src/**/*.cpp                              # apply
```

## ast-grep — structural search

```bash
sg -p 'new $$$' --lang cpp              # all raw new
sg -p 'delete $PTR' --lang cpp          # all raw delete
sg -p '($TYPE)$EXPR' --lang cpp         # C-style casts
sg -p 'NULL' -r 'nullptr' --lang cpp -i  # replace
```

## Pre-commit hook

```bash
#!/bin/bash
# .git/hooks/pre-commit
STAGED=$(git diff --cached --name-only --diff-filter=ACM | rg '\.(cpp|hpp|h|c)$')
[ -z "$STAGED" ] && exit 0
echo "$STAGED" | xargs clang-format --dry-run -Werror || { echo "Run: just format"; exit 1; }
echo "$STAGED" | xargs cppcheck --enable=warning --suppress=missingIncludeSystem || exit 1
```

> 💡 **Tip:** Run cppcheck + clang-format in CI on every push. Zero tolerance for warnings.
