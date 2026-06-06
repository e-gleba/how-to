# Project Navigation — Study Any Repo Fast

> You just cloned a 500K LOC project. Now what? These commands get you oriented in minutes.

## First 5 minutes — understand the project

```bash
# What language mix?
scc                                        # install: scoop install scc / brew install scc
scc --no-cocomo                            # without cost estimate noise

# Project structure at a glance
tree -L 2 --dirsfirst -I 'build|.git|node_modules|.cache'   # directory tree
fd -e cpp -e hpp -e h --type f | head -30  # first 30 source files
fd CMakeLists.txt                           # all CMake files (project structure)

# What does it build? (targets)
cmake -B build -G Ninja && cmake --build build --target help
# or if already built:
cmake --build build --target help | sort

# README, docs, contributing
cat README.md | bat                         # or just: bat README.md
fd -e md                                    # all markdown docs
bat CONTRIBUTING.md ARCHITECTURE.md DESIGN.md  # if they exist

# What dependencies?
cat CMakeLists.txt | rg 'find_package|FetchContent|CPMAddPackage'
cat vcpkg.json 2>/dev/null                  # vcpkg manifest
cat conanfile.py 2>/dev/null                # conan deps
cat conanfile.txt 2>/dev/null
```

## Git history — who changed what and when

```bash
# Recent activity (last 20 commits, one line each)
git log --oneline -20

# Overall project activity (commits per week)
git log --oneline --since="6 months ago" | wc -l
git log --format='%ad' --date=short | sort | uniq -c | tail -30

# Who are the main contributors?
git shortlog -sn --all | head -10           # top 10 by commit count
git shortlog -sn --all --since="1 year ago" | head -10  # recent contributors

# When was this file last touched?
git log --oneline -5 -- src/engine/renderer.cpp

# Full history of a file
git log --follow -p -- src/engine/renderer.cpp | bat -l diff

# What changed in the last release?
git log --oneline $(git describe --tags --abbrev=0)..HEAD

# Search commit messages
git log --oneline --grep="fix crash"
git log --oneline --grep="refactor" --author="John"

# When was a function/variable added or removed?
git log -S "functionName" --oneline         # -S searches diffs for string
git log -G "regex.*pattern" --oneline       # -G uses regex on diffs

# What changed between two tags/branches?
git log --oneline v1.0..v2.0
git diff v1.0..v2.0 --stat                  # file-level summary
git diff v1.0..v2.0 -- src/                 # only src/ changes
```

## git blame — who wrote this line and why

```bash
# Blame a file (annotated with commit + author per line)
git blame src/main.cpp | bat -l gitblame
git blame -L 10,50 src/main.cpp             # lines 10-50 only

# Blame with original commit messages
git blame -s src/main.cpp                   # show commit hash
git blame -e src/main.cpp                   # show email

# Blame with line movement tracking (renames/copies)
git blame -C -C -M src/main.cpp            # expensive but thorough

# Who last touched each line in a directory?
for f in src/*.cpp; do echo "=== $f ==="; git blame --line-porcelain "$f" | rg "^author " | sort | uniq -c | sort -rn | head -3; done

# Interactive blame (web-like, browseable)
git gui blame src/main.cpp                  # Tk GUI blame

# Blame + the commit that changed it
git blame src/main.cpp | rg "TODO"          # who left this TODO?
git blame src/main.cpp | rg "HACK|FIXME"
```

## git diff — understand changes

```bash
# What changed since I cloned?
git diff HEAD~1 --stat                      # file summary of last commit
git diff HEAD~5 --stat                      # last 5 commits combined
git diff origin/main --stat                 # vs remote main

# What's in this branch vs main?
git diff main...HEAD --stat                 # ... shows changes in branch only
git diff main...HEAD                        # full diff

# Diff a specific file across branches
git diff main..feature -- src/renderer.cpp

# Word-level diff (great for docs/comments)
git diff --word-diff HEAD~1

# Statistical diff — how much changed?
git diff --stat --numstat HEAD~10
git diff --shortstat main...HEAD            # total lines changed

# Side-by-side diff (needs delta: scoop install git-delta / brew install git-delta)
git diff HEAD~1 | delta                     # beautiful diffs
git diff HEAD~1 | delta --side-by-side      # side-by-side view

# Diff two commits
git diff abc1234 def5678 --stat
git show abc1234 --stat                     # what one commit changed
git show abc1234                            # full commit diff
```

## bisect — find the commit that broke something

```bash
# Binary search for a bad commit
git bisect start
git bisect bad HEAD                          # current is broken
git bisect good v1.0                         # v1.0 was good
# Git checks out middle commit → test → mark good/bad
git bisect good                              # this one works
git bisect bad                               # this one is broken
# ... repeat until found
git bisect reset                             # back to original branch

# Automated bisect (if you have a test)
git bisect start HEAD v1.0
git bisect run cmake --build build && ctest --test-dir build
# Git runs your test on each commit, finds the breaker automatically
```

## Find things fast in unfamiliar repos

```bash
# Where is main()?
rg "int main\(" --type cpp -n
rg "int main\(" --type cpp -l               # files only

# Where are the entry points?
rg "WinMain|wWinMain|SDL_main|glfwCreateWindow" --type cpp -n

# Find the build system entry point
cat CMakeLists.txt | rg "add_executable|add_library" -n

# What external libraries does it use?
rg "find_package|FetchContent_Declare|target_link_libraries" -n -g "CMakeLists.txt"
rg "#include.*<" --type cpp --no-filename | sort -u | head -40  # external includes

# Find TODOs, HACKs, FIXMEs
rg "TODO|FIXME|HACK|XXX|BUG|WORKAROUND" --type cpp -n -C 1

# Find test files
fd -e cpp "test|Test|spec" | head -20
fd -e cpp -e hpp | xargs rg "TEST_CASE|TEST_F|TEST\(" -l  # test framework files

# Find the hottest files (most changed = most complex/buggy)
git log --format=format: --name-only --since="6 months ago" | sort | uniq -c | sort -rn | head -20

# Find orphaned headers (included nowhere)
for h in $(fd -e hpp); do rg -q "$(basename $h)" --type cpp || echo "UNUSED: $h"; done
```

## AI-assisted study

```bash
# Use AI to summarize a file
# Claude: paste file contents, ask "what does this do?"
# GitHub Copilot Chat: /explain in VS Code on any file

# AI-powered repo understanding tools:
# greptile — semantic search across repos (https://greptile.com)
# Sourcegraph Cody — AI code search (https://sourcegraph.com/cody)
# Bloop — local AI code search (https://bloop.ai)
# Aider — AI pair programming in terminal (https://aider.chat)

# aider — terminal AI coding assistant
pip install aider-chat
aider                                    # opens interactive session
aider --model gpt-4o                     # specify model
aider --read-only src/engine/            # study mode: read-only, ask questions
# In aider: "> explain how the renderer works"
# In aider: "> what are the main abstractions?"
# In aider: "> find all places where memory is allocated without RAII"

# Cursor IDE — AI-native editor (https://cursor.sh)
# Cmd+K = inline edit, Cmd+L = chat, @file = reference file
# Best for: "explain this function", "find all usages", "refactor"

# GitHub Copilot in terminal (if you have gh CLI + copilot extension)
gh copilot suggest "find all TODO comments"
gh copilot explain "git rebase -i HEAD~5"

# Use AI to generate compile_commands.json symlink explanation:
# "Why does clangd need compile_commands.json and how does it work?"
```

## Churn analysis — find problem areas

```bash
# Most-changed files in last 6 months (= likely complex/buggy)
git log --format=format: --name-only --since="6 months ago" | \
  sort | uniq -c | sort -rn | head -20

# Files with most unique authors (= knowledge spread, good or bad)
git log --format='%aN' --since="1 year ago" -- src/ | sort -u | wc -l

# Commit frequency by hour (when does the team work?)
git log --format='%ad' --date=format:'%H' | sort | uniq -c | sort -k2n

# Largest files (potential refactoring candidates)
fd -e cpp -e hpp --exec wc -l | sort -rn | head -20
# or with scc:
scc --by-file src/ | sort -k2 -rn | head -20

# Dead code candidates — functions defined but never called
rg "void \w+\(" --type cpp -n | while read line; do
  func=$(echo "$line" | rg -o "\w+\(" | head -1 | tr -d '(')
  count=$(rg -c "$func" --type cpp | wc -l)
  [ "$count" -le 1 ] && echo "DEAD? $line"
done
```

## Quick orientation checklist

```
STEP 1:  scc                                    → language mix, line counts
STEP 2:  tree -L 2 --dirsfirst -I '.git|build'  → directory structure
STEP 3:  bat README.md                          → what is this project?
STEP 4:  cat CMakeLists.txt | rg "add_executable|add_library|find_package"
                                                → build targets + dependencies
STEP 5:  git log --oneline -20                  → recent activity
STEP 6:  git shortlog -sn --all | head -10      → who works on this?
STEP 7:  cmake -B build && cmake --build build --target help
                                                → all build targets
STEP 8:  rg "int main" --type cpp               → entry points
STEP 9:  rg "TODO|FIXME|HACK" --type cpp -c     → known issues count
STEP 10: git log --format=format: --name-only --since="6 months ago" |
         sort | uniq -c | sort -rn | head -10   → hottest files
```

---

**Related**: [CMake CLI](cmake.md) — configure, build, inspect one-liners
**Related**: [Search & navigation](search-navigation.md) — ripgrep, fd, fzf deep dive
**Related**: [Project navigation](https://git-scm.com/docs/git-log) — official git-log docs
