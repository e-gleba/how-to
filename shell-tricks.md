# Shell Tricks & Power Moves

> Bash/zsh/fish tricks that save hours. Linux + macOS. Windows equivalents where possible.

## Process Substitution `<()` — the secret weapon

> **Did you know?** `<()` runs a command and gives you a file descriptor you can pass to anything that expects a file. No temp files needed.

```bash
# Compare two command outputs without temp files
diff <(cmake --build build --target help | sort) <(cmake --build build-rel --target help | sort)

# Compare compile_commands.json between two builds
diff <(jq '.[].file' build/compile_commands.json | sort) <(jq '.[].file' build-rel/compile_commands.json | sort)

# Feed grep with command output as if it were a file
grep "error" <(cmake --build build 2>&1)

# Compare git log of two branches side by side
diff <(git log main --oneline -20) <(git log feature --oneline -20)

# Sort + unique from multiple sources
cat <(rg "#include" --type cpp --no-filename) <(rg "#include" --type c --no-filename) | sort -u

# Use with tools that only accept file arguments
clang-tidy -p build <(cat src/a.cpp src/b.cpp)

# Feed a heredoc to a command expecting stdin
sqlite3 mydb.db <(echo "SELECT * FROM users WHERE id > 100;")

# Compare binary sizes across builds
paste <(ls -la build/*.o | awk '{print $5, $NF}') <(ls -la build-rel/*.o | awk '{print $5, $NF}')
```

### `>()` — reverse process substitution

```bash
# Pipe to multiple commands simultaneously
cmake --build build 2>&1 | tee >(grep -i error > errors.log) >(grep -i warning > warnings.log)

# Log everything AND filter in real time
./myapp 2>&1 | tee >(grep -i "crash\|fatal\|abort" >> crash.log) | tee app.log

# Split stdout and stderr to different files
./myapp > >(tee stdout.log) 2> >(tee stderr.log >&2)
```

## Pipe Tricks

```bash
# xargs with parallel execution — process files 8 at a time
fd -e cpp | xargs -P 8 -I {} clang-format -i {}

# Pipe to clipboard (Linux)
rg "pattern" --type cpp | xclip -selection clipboard
# macOS
rg "pattern" --type cpp | pbcopy
# Windows PowerShell
rg "pattern" --type cpp | clip

# Watch command output in real time
watch -n 1 'cmake --build build --target help | wc -l'

# Chain with error handling
cmake -B build && cmake --build build && ctest --test-dir build || echo "FAILED at step $?"

# Time each step in a pipeline
{ time cmake -B build; } 2>&1 | tee configure-time.log
{ time cmake --build build; } 2>&1 | tee build-time.log

# Filter + count + display in one pipe
rg "TODO" --type cpp -c | sort -t: -k2 -nr | head -10   # top 10 files with most TODOs
```

## Brace Expansion & Globbing

```bash
# Create directory structure fast
mkdir -p project/{src/{core,render,audio},include,test,build}

# Copy file with backup
cp main.cpp{,.bak}                        # same as: cp main.cpp main.cpp.bak

# Rename multiple files
for f in *.cpp; do mv "$f" "${f%.cpp}.cc"; done

# Sequence operations
echo file{001..100}.cpp                   # generate filenames
touch test_{a,b,c,d}.cpp                  # create multiple files

# zsh: recursive glob (**)
rm **/*.o                                 # all .o files recursively
vim **/*.cpp                              # open all cpp files
# bash: enable with: shopt -s globstar
```

## History & Navigation

```bash
# Search history (Ctrl+R is default, but fzf makes it amazing)
# Install fzf key bindings:
# bash:
$(brew --prefix)/opt/fzf/install          # macOS
# or:
~/.fzf/install                            # Linux

# Now Ctrl+R = fuzzy history search with preview
# Ctrl+T = fuzzy file search inline
# Alt+C = fuzzy cd

# History tricks
!!                                         # repeat last command
!$                                         # last argument of previous command
!*                                         # all arguments of previous command
^old^new                                   # replace in last command and re-run

# Navigate to previous directory
cd -                                       # toggle between two dirs
pushd /some/path && popd                   # directory stack

# fzf-powered file open (add to .bashrc/.zshrc)
vf() { vim $(fzf --preview 'bat --style=numbers {}'); }
```

## Parallel Execution

```bash
# GNU parallel — run commands in parallel
# Install: scoop install parallel / sudo apt install parallel / brew install parallel

# Format all files in parallel (8 jobs)
fd -e cpp -e hpp | parallel -j8 clang-format -i {}

# Run tests in parallel groups
parallel cmake --build build{1..8} ::: --target test_a --target test_b

# Build + test multiple configurations
parallel cmake --preset {} ::: dev release clang gcc

# Download files in parallel
cat urls.txt | parallel -j10 wget -q {}
```

## Fancy Output

```bash
# colordiff — colored diff output
# Install: sudo apt install colordiff / brew install colordiff
colordiff <(git show HEAD:file.cpp) file.cpp

# delta — beautiful git diffs
# Install: scoop install git-delta / brew install git-delta
git config --global core.pager delta
git config --global interactive.diffFilter 'delta --color-only'

# bat as cat replacement
alias cat=bat
# Now everything has syntax highlighting

# lsd — better ls with icons
# Install: scoop install lsd / brew install lsd
alias ls=lsd

# eza (exa successor) — modern ls
# Install: scoop install eza / brew install eza
alias ls='eza --icons --group-directories-first'
alias ll='eza -la --icons --time-style=relative'
alias lt='eza --tree --level=2 --icons'
```

## Redirects & FD Tricks

```bash
# Redirect stderr only
./myapp 2> errors.log

# Swap stdout and stderr (see errors inline, log output)
./myapp 3>&1 1>&2 2>&3

# Append to file without overwriting
./myapp >> log.txt 2>&1

# /dev/null tricks
./myapp > /dev/null 2>&1                  # silence everything
./myapp 2>/dev/null                        # see stdout only
./myapp >/dev/null                         # see stderr only

# Named pipe (FIFO) — connect two processes
mkfifo /tmp/mypipe
./producer > /tmp/mypipe &                 # write in background
./consumer < /tmp/mypipe                   # read
rm /tmp/mypipe

# Here-document for multi-line input
sqlite3 mydb.db <<EOF
CREATE TABLE test (id INTEGER, name TEXT);
INSERT INTO test VALUES (1, 'hello');
SELECT * FROM test;
EOF
```

## Useful Aliases (add to .bashrc/.zshrc)

```bash
# Git
alias gs='git status -sb'
alias gl='git log --oneline --graph --decorate -20'
alias gd='git diff'
alias gds='git diff --staged'
alias gc='git commit'
alias gp='git push'
alias gco='git checkout'
alias gb='git branch'

# Build
alias cb='cmake --build build'
alias ct='cmake --build build --target'
alias cr='cd build && ctest --output-on-failure && cd ..'

# Search
alias t='rg --type cpp'
alias tf='fd -e cpp -e hpp'

# Navigation
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'

# Quick edits
alias viz='vim ~/.zshrc && source ~/.zshrc'
alias vib='vim ~/.bashrc && source ~/.bashrc'
```

## Windows PowerShell Equivalents

```powershell
# Process substitution doesn't exist — use temp files or Compare-Object
$a = cmake --build build --target help | Sort-Object
$b = cmake --build build-rel --target help | Sort-Object
Compare-Object $a $b

# Pipe to clipboard
rg "pattern" --type cpp | clip

# Redirect stderr
./myapp.exe 2> errors.log

# Measure command time
Measure-Command { cmake --build build }

# Parallel execution
fd -e cpp | ForEach-Object -Parallel { clang-format -i $_ } -ThrottleLimit 8
```

---

**Related**: [Project navigation](project-navigation.md) — git blame, history, AI study tools
**Related**: [Git tricks & lazygit](git-tricks.md) — advanced git workflows
