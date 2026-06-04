# Search & Navigation

## ripgrep (rg)

```bash
# Install
scoop install ripgrep

# Search by file type
rg "function_name" --type cpp

# Show line numbers + context
rg "function_name" --type cpp -n -C 3

# Only list files with matches
rg "TODO" --type cpp -l

# Count matches per file
rg "TODO" --type cpp -c

# Invert match (files NOT containing)
rg "#include" --type cpp --files-without-match

# Search specific paths
rg "Widget" src/widgets/ src/core/

# JSON output (for scripts)
rg "error" --json

# Replace (dry-run first!)
rg "old_name" --type cpp -l | xargs sed -i 's/old_name/new_name/g'

# Ignore case
rg -i "socket"

# Whole word only
rg -w "class"

# Regex
rg "Widget\w+" --type cpp
```

### Common patterns

```bash
# Find raw owning pointers
rg "new\s+\w+" --type cpp

# Find C-style casts
rg "\(\w+\*\)" --type cpp

# Find potential memory leaks (new without delete nearby)
rg "new\s" --type cpp -A 5 | rg -v "delete|unique_ptr|shared_ptr|make_unique|make_shared"

# Find functions with many parameters
rg "^\s*\w+\s+\w+\([^)]{80,}" --type cpp

# Find long functions (>200 lines)
# (use scc for this: scc --by-function)

# Find all #include directives
rg "^#include" --type cpp

# Find unused includes (subjective)
rg "^#include <(vector|string|map)>" --type cpp -l | while read f; do
    rg -q "std::(vector|string|map)" "$f" || echo "Maybe unused: $f"
done
```

## fd — file finder

```bash
# Install
scoop install fd

# Find all C++ files
fd -e cpp -e hpp -e h

# Find files matching pattern
fd "test_.*\.cpp"

# Find files modified today
fd --change-newer-than 1day

# Find + execute command
fd -e cpp -x clang-format -i {}

# Find empty files
fd -e cpp --size -1b

# Find files larger than 1MB
fd --size +1m
```

## fzf — fuzzy finder

```bash
# Install
scoop install fzf

# Interactive file search
fd -e cpp | fzf

# Search file contents
rg --line-number --no-heading . | fzf --delimiter : --preview 'bat --style=numbers --color=always {1}' --preview-window +{2}

# Git branch checkout
git branch | fzf | xargs git checkout

# Kill process
ps aux | fzf -m | awk '{print $2}' | xargs kill -9
```

### Power combos

```bash
# Find TODOs with context, fuzzy filter
alias todos='rg "TODO|FIXME|HACK|XXX" --type cpp -n --no-heading | fzf --ansi'

# Search + open in editor
alias vrg='rg -l "$@" | fzf --preview "bat --style=numbers {}" | xargs nvim'

# Find most-churned files
alias churn="git log --format=format: --name-only | rg '.*' | sort | uniq -c | sort -nr | head -20"

# Find files by type, pipe to fzf, open
fd -e cpp | fzf --preview 'bat --style=numbers --color=always {}' | xargs nvim
```

## bat — cat with syntax

```bash
# Install
scoop install bat

# View file with line numbers
bat src/main.cpp

# Show git changes in gutter
bat --diff src/main.cpp

# Show all themes
bat --list-themes

# Use as less replacement
alias less='bat'

# Use as fzf previewer (as above)
```

## ugrep — grep on steroids

```bash
# Install
scoop install ugrep

# Fuzzy search (typo-tolerant)
ugrep -Z "Widgte"

# Boolean search
ugrep --bool "socket AND (connect OR bind)"

# Interactive TUI
ugrep -Q "pattern"

# Search with context, formatted
ugrep -n -C 3 --color "function"

# Search compressed files too
ugrep -z "pattern"
```

## ag / ack — simpler searchers

```bash
# ack — perl-based, older
ack "pattern"

# ag (silver searcher) — faster than ack, slower than rg
ag "pattern"

# Honestly, just use rg.
```

## scc — code counter

```bash
# Install
scoop install scc

# Count by language
scc src/

# Count by file
scc --by-file src/

# Estimate complexity
scc --complexity src/

# Wide output for more details
scc -w src/

# Output JSON for CI
scc --format json src/
```

## tree-sitter

```bash
# Install
scoop install tree-sitter

# Parse a file and show AST
tree-sitter parse src/main.cpp

# Query AST
# Create query.scm:
# (function_definition
#   declarator: (function_declarator
#     declarator: (identifier) @name)) @func

tree-sitter query query.scm src/main.cpp
```

## Git history search

```bash
# Find when a function was added/removed
git log -S "function_name" --oneline

# Find commits that touched a line range
git log -L 100,150:src/main.cpp

# Search commit messages
git log --grep="fix" --oneline

# Find who wrote each line
git blame src/main.cpp | rg "TODO"
```
