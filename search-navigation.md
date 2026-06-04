# Search & Navigation

## ripgrep (rg)

```bash
rg "pattern" --type cpp                # search C++ files only
rg "pattern" --type cpp -n -C 3        # with line numbers + context
rg "TODO" --type cpp -l                 # list files with matches
rg "pattern" --type cpp -c              # count matches per file
rg -w "class"                           # whole word only
rg -i "socket"                          # case insensitive
rg "new\s+\w+" --type cpp               # regex: raw owning pointers
rg -l "pattern" | xargs clang-format -i  # pipe to commands
```

### Daily patterns

```bash
# Find TODOs
rg "TODO|FIXME|HACK|XXX" --type cpp -n

# Raw owning pointers (candidates for smart ptr)
rg 'new\s+\w+' --type cpp

# C-style casts
rg '\(\w+\*\)' --type cpp

# Long parameter lists
rg '^\s*\w+\s+\w+\([^)]{80,}' --type cpp

# Files NOT containing a header
rg "#include" --type cpp --files-without-match
```

## fd — file finder

```bash
fd -e cpp -e hpp              # all C++ files
fd test_                      # files with "test_" in name
fd --change-newer-than 1day   # modified today
fd -e cpp -x clang-format -i {}  # find + execute
```

## fzf — fuzzy finder

```bash
# Fuzzy file open
fd -e cpp | fzf | xargs nvim

# Search + preview + open
rg --line-number --no-heading . | fzf --delimiter : \
  --preview 'bat --style=numbers {1}' --preview-window +{2}

# Git branch checkout
git branch | fzf | xargs git checkout
```

### Power aliases

```bash
alias todos='rg "TODO|FIXME|HACK|XXX" --type cpp -n --no-heading | fzf --ansi'
alias vrg='rg -l "$@" | fzf --preview "bat --style=numbers {}" | xargs nvim'
alias churn='git log --format=format: --name-only | rg "." | sort | uniq -c | sort -nr | head -20'
```

## bat

```bash
bat src/main.cpp              # syntax highlighting
bat --diff src/main.cpp        # show git changes in gutter
```

## scc (count code)

```bash
scc src/                       # by language
scc --by-file src/             # per file
scc --complexity src/          # estimate complexity
```

## Git history search

```bash
git log -S "function_name" --oneline     # when was this added/removed?
git log -L 100,150:src/main.cpp          # line range history
git log --grep="fix" --oneline           # search commit messages
git blame src/main.cpp | rg "TODO"        # who left this?
```

> 💡 **Tip:** `rg` is 10-50x faster than `grep`, respects `.gitignore`, and has better output formatting.
