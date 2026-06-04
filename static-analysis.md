# Static Analysis

## cppcheck

```bash
# Install
scoop install cppcheck

# Full check
cppcheck --enable=all --suppress=missingIncludeSystem src/

# With XML output for CI
cppcheck --enable=all --xml src/ 2> cppcheck.xml

# Check specific file
cppcheck --enable=all src/main.cpp

# Enable performance checks
cppcheck --enable=performance src/

# Inconclusive checks (more false positives but catches more)
cppcheck --enable=all --inconclusive src/
```

### Common suppressions

```bash
cppcheck \
    --suppress=missingIncludeSystem \
    --suppress=unmatchedSuppression \
    --suppress=knownConditionTrueFalse \
    --enable=all \
    src/
```

### In CMake

```cmake
# CMakeLists.txt
add_custom_target(cppcheck
    COMMAND cppcheck
        --enable=all
        --suppress=missingIncludeSystem
        --error-exitcode=1
        ${CMAKE_SOURCE_DIR}/src
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    COMMENT "Running cppcheck..."
)
```

```bash
cmake --build build --target cppcheck
```

Or via justfile:

```bash
just lint
```

## clang-tidy

### In CMake

```cmake
# CMakeLists.txt
set(CMAKE_CXX_CLANG_TIDY
    clang-tidy;
    -header-filter=${CMAKE_SOURCE_DIR}/*;
    -checks=-*,clang-analyzer-*,cppcoreguidelines-*,modernize-*,performance-*,readability-*;
    -warnings-as-errors=*;
)
```

Or per-target:

```cmake
set_target_properties(myapp PROPERTIES
    CXX_CLANG_TIDY "clang-tidy;-checks=-*,modernize-*,performance-*"
)
```

This makes every compilation also run clang-tidy. For large projects, use a separate target:

```cmake
add_custom_target(tidy
    COMMAND run-clang-tidy
        -p ${CMAKE_BINARY_DIR}
        -header-filter=${CMAKE_SOURCE_DIR}/src/.*
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
)
```

```bash
cmake --build build --target tidy
```

### Auto-fix

```bash
clang-tidy -p build --fix src/main.cpp
clang-tidy -p build --fix-errors src/main.cpp  # also fix compiler errors
```

### Standalone

```bash
# Requires compile_commands.json in project root
clang-tidy src/main.cpp -p .

# Check specific checks
clang-tidy src/main.cpp -p . -checks=modernize-*,readability-*
```

### Useful check groups

| Group | Catches |
|-------|---------|
| `clang-analyzer-*` | Memory leaks, use-after-free, null deref |
| `cppcoreguidelines-*` | C++ Core Guidelines violations |
| `modernize-*` | Use `nullptr`, `override`, range-for, `make_unique` |
| `performance-*` | Unnecessary copies, inefficient algorithms |
| `readability-*` | Naming, redundant code, implicit casts |
| `bugprone-*` | Misuses of APIs, suspension points, branch clones |
| `misc-*` | Miscellaneous checks |

## ast-grep — Structural Code Search

```bash
# Install
scoop install ast-grep

# Find all raw `new` expressions (no smart pointers)
sg -p 'new $$$' --lang cpp

# Find all raw `delete`
sg -p 'delete $PTR' --lang cpp

# Find C-style casts
sg -p '($TYPE)$EXPR' --lang cpp

# Find functions without const
sg -p '$$$ $FUNC($$$) { $$$ }' --lang cpp

# Replace patterns
sg -p 'NULL' -r 'nullptr' --lang cpp -i
```

### Custom rules

```yaml
# sgconfig.yml
ruleDirs:
  - rules
```

```yaml
# rules/no-raw-new.yml
id: no-raw-new
message: Use std::make_unique or std::make_shared instead
severity: warning
language: cpp
rule:
  pattern: new $_
```

```bash
sg scan
```

## clang-format

```bash
# Generate .clang-format file (pick a style as base)
clang-format -style=llvm -dump-config > .clang-format

# Check formatting (CI)
clang-format --dry-run -Werror src/**/*.cpp src/**/*.hpp

# Apply formatting
clang-format -i src/**/*.cpp src/**/*.hpp
```

### .clang-format essentials

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 120
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: None
AllowShortIfStatementsOnASingleLine: Never
AlwaysBreakTemplateDeclarations: Yes
BreakBeforeBraces: Allman
PointerAlignment: Left
```

## CI integration

```yaml
# .github/workflows/lint.yml
name: Lint
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt install cppcheck clang-tidy clang-format
      - run: cppcheck --enable=all --error-exitcode=1 src/
      - run: clang-format --dry-run -Werror src/**/*.cpp
      - run: clang-tidy src/**/*.cpp -- -std=c++20
```

## Git pre-commit hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Format staged files
STAGED=$(git diff --cached --name-only --diff-filter=ACM | rg '\.(cpp|hpp|h|c)$')
if [ -n "$STAGED" ]; then
    echo "$STAGED" | xargs clang-format --dry-run -Werror || {
        echo "Run: just format"
        exit 1
    }
    echo "$STAGED" | xargs cppcheck --enable=warning --suppress=missingIncludeSystem || exit 1
fi
```
