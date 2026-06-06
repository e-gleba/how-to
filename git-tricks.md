# Git Tricks & lazygit

> Advanced git workflows + [lazygit](https://github.com/jesseduffield/lazygit) TUI for speed.
> Install: `scoop install lazygit` (Windows) / `sudo apt install lazygit` (Linux) / `brew install lazygit` (macOS)

## lazygit — TUI git client

> **Did you know?** lazygit makes interactive rebase, cherry-pick, stash, and conflict resolution 10× faster than CLI.

```bash
lazygit                                    # launch TUI
lg                                         # if you alias it
```

### Key bindings (inside lazygit)

| Key | Action |
|-----|--------|
| `[1-5]` | Switch panels: Status, Branches, Commits, Stash, Remotes |
| `Space` | Stage/unstage file |
| `a` | Stage/unstage all |
| `c` | Commit (opens editor) |
| `C` | Commit with message prompt |
| `p` | Push |
| `P` | Pull |
| `n` | New branch |
| `M` | Merge selected branch |
| `r` | Rebase selected branch |
| `e` | Edit commit (interactive rebase) |
| `s` | Squash commit into one above |
| `S` | Stash |
| `Enter` | View commit details / diff |
| `/` | Search |
| `?` | Show all keybindings |
| `:` | Cancel / close menu |
| `q` | Quit |

### lazygit power moves

```
INTERACTIVE REBASE:
  1. Go to Commits panel [3]
  2. Navigate to commit you want to edit
  3. Press 'e' to edit, 's' to squash, 'd' to drop
  4. Press 'Enter' to confirm → resolves conflicts inline

CHERRY-PICK:
  1. Go to Commits panel [3]
  2. Navigate to commit
  3. Press 'c' to copy
  4. Go to your branch
  5. Press 'V' to paste (cherry-pick)

BISECT (visual):
  1. Go to Commits panel [3]
  2. Mark commit as good: 'b' → good
  3. Mark HEAD as bad: 'b' → bad
  4. lazygit runs bisect, shows result

AMEND LAST COMMIT:
  1. Stage your changes
  2. Press 'A' (capital A) → amends last commit without new message

STASH POP SPECIFIC:
  1. Go to Stash panel [4]
  2. Navigate to stash
  3. Press 'g' to pop (or 'a' to apply without dropping)
```

## Interactive Rebase — the CLI way

```bash
# Rebase last 5 commits interactively
git rebase -i HEAD~5

# In the editor, you see:
# pick abc1234 Add feature X
# pick def5678 Fix bug in Y
# pick ghi9012 Refactor Z
# pick jkl3456 Update docs
# pick mno7890 Final cleanup

# Commands:
# pick   = use commit as-is
# reword = use commit but edit message
# edit   = stop at commit for amending
# squash = combine with previous commit
# fixup  = squash but discard this commit's message
# drop   = remove commit entirely

# Common patterns:
# Squash last 3 into one:
# pick abc1234
# squash def5678
# squash ghi9012

# Edit a commit message:
# reword abc1234

# Reorder commits (just move lines around):
# pick ghi9012    ← moved up
# pick abc1234
# pick def5678
```

## Git Bisect — find bugs with binary search

```bash
# Manual bisect
git bisect start
git bisect bad HEAD                        # current is broken
git bisect good v1.0                       # v1.0 was good
# Git checks out middle commit
# Test it → mark good or bad:
git bisect good                            # or: git bisect bad
git bisect reset                           # done, back to original branch

# Automated bisect (if you have a test)
git bisect start HEAD v1.0
git bisect run bash -c "cmake --build build && ctest --test-dir build"
# Git runs your test on each commit, finds the breaker automatically

# Bisect with a script
cat > /tmp/bisect-test.sh << 'EOF'
#!/bin/bash
cmake -B build -G Ninja && cmake --build build && ctest --test-dir build
EOF
chmod +x /tmp/bisect-test.sh
git bisect start HEAD v1.0
git bisect run /tmp/bisect-test.sh
```

## Stash Tricks

```bash
# Stash with message
git stash push -m "WIP: feature X"

# Stash specific files only
git stash push -m "only these files" -- src/a.cpp src/b.cpp

# Stash untracked files too
git stash push -u -m "including untracked"

# List stashes
git stash list

# Apply specific stash (don't drop)
git stash apply stash@{2}

# Pop (apply + drop)
git stash pop stash@{1}

# Create branch from stash
git stash branch feature-from-stash stash@{0}

# Stash only staged changes
git stash push --staged -m "staged only"

# View stash diff
git stash show -p stash@{0}
```

## Cherry-pick & Patch

```bash
# Cherry-pick one commit
git cherry-pick abc1234

# Cherry-pick range (exclusive start)
git cherry-pick abc1234..def5678

# Cherry-pick without committing (stage only)
git cherry-pick abc1234 --no-commit

# Create patch from commit
git format-patch -1 abc1234                # → 0001-commit-message.patch

# Apply patch
git am 0001-commit-message.patch

# Create patch from diff
git diff main..feature > feature.patch
git apply feature.patch

# Check if patch applies cleanly
git apply --check feature.patch
```

## Reflog — recover anything

```bash
# View reflog (every HEAD movement)
git reflog
# abc1234 HEAD@{0}: commit: Add feature
def5678 HEAD@{1}: checkout: moving from feature to main
# ...

# Recover deleted branch
git reflog | grep "moving from deleted-branch"
git checkout -b deleted-branch abc1234     # recreate at that commit

# Recover after bad rebase
git reflog                                 # find the commit before rebase
git reset --hard abc1234                   # back to pre-rebase state

# Recover dropped stash
git fsck --unreachable | grep commit       # find dangling commits
git show <dangling-commit>                 # check if it's your stash
git stash apply <dangling-commit>
```

## Worktrees — multiple branches at once

```bash
# Create worktree for a branch (separate directory, same repo)
git worktree add ../project-hotfix hotfix-branch
cd ../project-hotfix
# Edit, commit, push — all in the hotfix worktree
# Main project directory still on your feature branch

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../project-hotfix

# Prune stale worktrees
git worktree prune
```

## Advanced Log & Blame

```bash
# Pretty log with graph
git log --oneline --graph --decorate --all -30

# Show commits that changed a function
git log -L :functionName:src/file.cpp

# Shortlog — who did what
git shortlog -sn --all --since="1 year ago"

# Log with diffstat
git log --stat --oneline -10

# Blame with previous revision
git blame -C -C -M src/file.cpp            # detect moved/copied code

# Blame specific line range
git blame -L 10,20 src/file.cpp

# Show who last modified each line + commit message
git blame --line-porcelain src/file.cpp | grep -E "^author |^summary "

# Log for specific author
git log --author="John" --oneline -20

# Log with date ranges
git log --since="2 weeks ago" --until="1 week ago" --oneline
git log --after="2024-01-01" --before="2024-06-01" --oneline

# What changed between two tags
git log --oneline v1.0..v2.0
git diff v1.0..v2.0 --stat
```

## Git Config Essentials

```bash
# Better defaults
git config --global pull.rebase true       # rebase on pull (no merge commits)
git config --global init.defaultBranch main
git config --global fetch.prune true       # auto-delete remote branches that no longer exist
git config --global push.autoSetupRemote true  # auto-create remote branch on first push
git config --global rerere.enabled true    # remember conflict resolutions

# Delta as default pager (beautiful diffs)
git config --global core.pager delta
git config --global interactive.diffFilter 'delta --color-only'
git config --global delta.navigate true
git config --global delta.side-by-side true

# Useful aliases
git config --global alias.st 'status -sb'
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg 'log --oneline --graph --decorate -20'
git config --global alias.last 'log -1 HEAD'
git config --global alias.undo 'reset HEAD~1 --mixed'
git config --global alias.unstage 'reset HEAD --'
git config --global alias.wip '!git add -A && git commit -m "WIP"'
```

## Hooks — automate on git events

```bash
# Pre-commit: run clang-format on staged files
# .git/hooks/pre-commit
#!/bin/bash
STAGED=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(cpp|hpp|h)$')
[ -z "$STAGED" ] && exit 0
echo "$STAGED" | xargs clang-format --dry-run -Werror || { echo "Format check failed. Run: clang-format -i"; exit 1; }

# Commit-msg: enforce conventional commits
# .git/hooks/commit-msg
#!/bin/bash
if ! head -1 "$1" | grep -qE '^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?!?: .{1,72}$'; then
    echo "Commit message must follow Conventional Commits: type(scope): description"
    exit 1
fi

# Make executable
chmod +x .git/hooks/pre-commit .git/hooks/commit-msg
```

---

**Related**: [Project navigation](project-navigation.md) — orient yourself in any repo
**Related**: [Shell tricks](shell-tricks.md) — process substitution, pipes, aliases
**Related**: [lazygit docs](https://github.com/jesseduffield/lazygit) — full keybinding reference
