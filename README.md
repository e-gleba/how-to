# how-to

> **C++ Cross-Platform Dev Cookbook** — one-liners, tools, and patterns for [game dev](https://www.gameenginebook.com), research, and [reverse engineering](https://ghidra-sre.org).
>
> Install once. Use everywhere. Copy-paste friendly.

---

## 📋 Index

| # | Topic | What | Deep Dive |
|---|-------|------|----------|
| 0 | [Install](#0-install-tools) | One-command install per platform | [tools-install.md](tools-install.md) |
| 1 | [CMake](#1-cmake) | CLI one-liners: configure, build, test, pro tricks | [cmake.md](cmake.md) |
| 2 | [Project Navigation](#2-project-navigation) | Study any repo fast: blame, history, diff, AI | [project-navigation.md](project-navigation.md) |
| 3 | [Git & lazygit](#3-git--lazygit) | Rebase, bisect, stash, cherry-pick, lazygit TUI | [git-tricks.md](git-tricks.md) |
| 4 | [Shell Tricks](#4-shell-tricks) | `<()` process substitution, pipes, parallel, aliases | [shell-tricks.md](shell-tricks.md) |
| 5 | [Build Speed](#5-build-speed) | ccache, Ninja, PCH, unity | [build-acceleration.md](build-acceleration.md) |
| 6 | [Search](#6-search) | ripgrep, fd, fzf, bat | [search-navigation.md](search-navigation.md) |
| 7 | [Static Analysis](#7-static-analysis) | cppcheck, clang-tidy, ast-grep | [static-analysis.md](static-analysis.md) |
| 8 | [Debug & Profile](#8-debug--profile) | GDB/LLDB pro tricks, threads, reverse debug, minidumps | [debugging-profiling.md](debugging-profiling.md) |
| 9 | [Network Debug](#9-network-debug) | tcpdump, Wireshark, DNS, TLS, why can't it connect | [network-debugging.md](network-debugging.md) |
| 10 | [Cross-Compile](#10-cross-compile) | Zig, Android, iOS | [cross-compilation.md](cross-compilation.md) |
| 11 | [Packages](#11-packages) | vcpkg, Conan, FetchContent | [cmake-package-managers.md](cmake-package-managers.md) |
| 12 | [Task Runner](#12-task-runner) | just — never type cmake again | [justfile.md](justfile.md) |
| 13 | [Benchmarking](#13-benchmarking) | hyperfine, Google Benchmark | [benchmarking.md](benchmarking.md) |
| 14 | [File Tracking](#14-file-tracking) | What did that command produce? | ↓ |
| 15 | [Mobile Dev](#15-mobile-dev) | ADB, root, Xcode, devicectl, LLDB | [mobile.md](mobile.md) |
| 16 | [Reverse Engineering](#16-reverse-engineering) | Ghidra, r2, x64dbg, WinDbg | [reverse-engineering.md](reverse-engineering.md) |
| 17 | [Binary Tools](#17-binary-tools) | UPX, hex, PE, shipping | [binary-tools.md](binary-tools.md) |
| 18 | [Controls & Input](#18-controls--input) | [Steam Golden Rules](https://partner.steamgames.com/doc/features/steam_controller/getting_started_for_devs), gamepad | [controls.md](controls.md) |
| 19 | [Windows](#19-windows-cpp) | MSVC, CRT, PDB, Defender | [windows-cpp.md](windows-cpp.md) |
| 20 | [Resources](#20-resources) | Books, talks, channels | [resources.md](resources.md) |

---

## 0. Install Tools

> **Scoop preferred on Windows.** Full table with every tool: [tools-install.md](tools-install.md)

```bash
# 🪟 Windows (scoop — no admin needed, clean install to ~/scoop/)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
scoop install git cmake ninja ccache ripgrep fd bat fzf just llvm zig lazygit

# 🐧 Linux (Ubuntu/Debian)
sudo apt update && sudo apt install -y \
  build-essential cmake ninja-build ccache gdb lldb clang clang-tidy \
  ripgrep fd-find bat fzf strace tcpdump

# 🍎 macOS (get Xcode CLT first for clang + lldb + git)
xcode-select --install
brew install cmake ninja ccache ripgrep fd bat fzf just lazygit
```

---

## 1. CMake

> Full CLI cookbook with pro tricks: [cmake.md](cmake.md)
> See also: [Modern CMake guide](https://cliutils.gitlab.io/modern-cmake/), [Professional CMake book](https://crascit.com/professional-cmake/)

```bash
# THE daily command — debug + ninja + ccache + compile_commands
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
cmake --build build
ln -sf build/compile_commands.json .         # clangd/clang-tidy/IDE need this at root

# Build one target
cmake --build build --target myapp

# Test
ctest --test-dir build --output-on-failure

# Sanitizers
cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"

# Presets (if project has CMakePresets.json)
cmake --list-presets
cmake --preset dev && cmake --build --preset dev && ctest --preset dev

# Study the project — what targets exist?
cmake --build build --target help | sort
# What features/options does it expose?
cmake -B build -LAH | rg -i "feature|enable|option|with"
# Generate dependency graph (needs graphviz)
cmake -B build --graphviz=deps.dot && dot -Tpng deps.dot -o deps.png
# Profile configure speed (CMake 3.28+)
cmake -B build --profiling-output=prof.json --profiling-format=google-trace
# → open prof.json at https://ui.perfetto.dev

# Cross-compile
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake
cmake -B build/android -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-24
cmake -B build/ios -GXcode -DCMAKE_SYSTEM_NAME=iOS -DCMAKE_OSX_ARCHITECTURES=arm64
```

---

## 2. Project Navigation

> Just cloned a big repo? Full guide: [project-navigation.md](project-navigation.md)
> Git official docs: [git-log](https://git-scm.com/docs/git-log), [git-blame](https://git-scm.com/docs/git-blame), [git-bisect](https://git-scm.com/docs/git-bisect)

### Orient yourself in 60 seconds

```bash
# What is this project?
scc                                         # language mix + line counts
tree -L 2 --dirsfirst -I 'build|.git|node_modules'
bat README.md

# What does it build?
cmake -B build -G Ninja && cmake --build build --target help | sort

# Who works on it?
git shortlog -sn --all | head -10           # top contributors
git log --oneline -20                       # recent activity

# Hottest files (most changed = most complex, most bugs)
git log --format=format: --name-only --since="6 months ago" | sort | uniq -c | sort -rn | head -15
```

### Understand any file

```bash
# Who wrote this and when?
git blame src/engine/renderer.cpp | bat -l gitblame
git log --oneline -10 -- src/engine/renderer.cpp

# When was this function added?
git log -S "initVulkan" --oneline

# What changed in this file recently?
git diff HEAD~10 -- src/engine/renderer.cpp
```

### AI-assisted study

```bash
# aider — AI pair programmer in terminal (https://aider.chat)
pip install aider-chat
aider --read-only src/engine/               # study mode: ask questions about code
# > "explain how the rendering pipeline works"
# > "what are the main abstractions and data flow?"

# GitHub Copilot Chat in terminal (https://cli.github.com)
gh copilot suggest "find all functions that allocate without RAII"
gh copilot explain "git rebase -i HEAD~5"
```

### bisect — find the commit that broke it

```bash
git bisect start
git bisect bad HEAD                          # current is broken
git bisect good v1.0                         # v1.0 was good
# Test each checkout → git bisect good/bad → repeat until found
git bisect reset

# Automated (if you have a test):
git bisect start HEAD v1.0
git bisect run cmake --build build && ctest --test-dir build
```

---

## 3. Git & lazygit

> Full reference with lazygit keybindings: [git-tricks.md](git-tricks.md)
> Tool: [lazygit](https://github.com/jesseduffield/lazygit) — TUI git client, makes rebase/bisect/cherry-pick visual

```bash
# Install: scoop install lazygit / brew install lazygit / sudo apt install lazygit
lazygit                                      # launch TUI — everything is visual

# Interactive rebase (lazygit: go to Commits [3], press 'e' to edit, 's' to squash)
git rebase -i HEAD~5                         # CLI equivalent

# Cherry-pick
git cherry-pick abc1234                      # one commit
git cherry-pick abc1234..def5678             # range

# Stash tricks
git stash push -m "WIP: feature X"          # named stash
git stash push -- src/a.cpp src/b.cpp       # specific files only
git stash push -u -m "with untracked"       # include untracked
git stash list                              # see all stashes
git stash apply stash@{2}                   # apply without dropping

# Reflog — recover anything
git reflog                                  # every HEAD movement ever
git checkout -b recovered-branch abc1234    # recover deleted branch

# Worktrees — work on multiple branches simultaneously
git worktree add ../project-hotfix hotfix-branch
cd ../project-hotfix                        # separate dir, same repo

# Pretty log
git log --oneline --graph --decorate --all -30
```

> 💡 **Pro tip**: lazygit makes interactive rebase 10× faster. Select commit → press `e` to edit, `s` to squash, `d` to drop. Resolve conflicts inline. No CLI editor needed.

---

## 4. Shell Tricks

> Full reference with process substitution, pipes, aliases: [shell-tricks.md](shell-tricks.md)

### Process substitution `<()` — the secret weapon

> **Did you know?** `<()` runs a command and gives you a temp file descriptor. No temp files needed.

```bash
# Compare two command outputs
diff <(cmake --build build --target help | sort) <(cmake --build build-rel --target help | sort)

# Compare compile_commands between builds
diff <(jq '.[].file' build/compile_commands.json | sort) <(jq '.[].file' build-rel/compile_commands.json | sort)

# Compare git log of two branches
diff <(git log main --oneline -20) <(git log feature --oneline -20)

# Feed multiple sources to one command
cat <(rg "#include" --type cpp --no-filename) <(rg "#include" --type c --no-filename) | sort -u

# Pipe to multiple commands simultaneously
cmake --build build 2>&1 | tee >(grep -i error > errors.log) >(grep -i warning > warnings.log)
```

### Pipe tricks

```bash
# xargs with parallel execution
fd -e cpp | xargs -P 8 -I {} clang-format -i {}

# Pipe to clipboard
rg "pattern" --type cpp | xclip -selection clipboard    # Linux
rg "pattern" --type cpp | pbcopy                         # macOS
rg "pattern" --type cpp | clip                           # Windows

# GNU parallel — process files 8 at a time
fd -e cpp -e hpp | parallel -j8 clang-format -i {}

# Brace expansion — create project structure fast
mkdir -p project/{src/{core,render,audio},include,test,build}
cp main.cpp{,.bak}                           # same as: cp main.cpp main.cpp.bak
```

### fzf-powered workflow

```bash
# Install fzf key bindings: ~/.fzf/install
# Now you get:
# Ctrl+R = fuzzy history search with preview
# Ctrl+T = fuzzy file search inline
# Alt+C  = fuzzy cd

# Fuzzy open file with preview
fd -e cpp | fzf --preview 'bat --style=numbers {}' | xargs vim

# Fuzzy git branch checkout
git branch | fzf | xargs git checkout
```

---

## 5. Build Speed

> Full reference: [build-acceleration.md](build-acceleration.md)
> Compiler cache comparison: [ccache](https://ccache.dev) vs [sccache](https://github.com/nickel-org/sccache)

```bash
# ccache — 5-10× faster rebuilds
ccache -s                                    # stats
ccache -C                                    # clear

# Enable in cmake:
cmake -B build -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

# Unity builds (CI only!)
cmake -B build -DCMAKE_UNITY_BUILD=ON
```

| Technique | Cold build | Hot rebuild |
|-----------|-----------|-------------|
| + [Ninja](https://ninja-build.org) | ~80% | ~70% |
| + ccache | ~80% | **~15%** |
| + PCH | ~60% | ~50% |
| + Unity | ~25% | n/a |

---

## 6. Search

> Full reference: [search-navigation.md](search-navigation.md)
> Tools: [ripgrep](https://github.com/BurntSushi/ripgrep), [fd](https://github.com/sharkdp/fd), [fzf](https://github.com/junegunn/fzf), [bat](https://github.com/sharkdp/bat)

```bash
# ripgrep — search code
rg "pattern" --type cpp -n -C 3             # with context
rg "TODO|FIXME|HACK" --type cpp             # todos
rg "new\s+\w+" --type cpp                   # raw owning pointers (smart ptr candidates)
rg "^#include" --type cpp --no-filename | sort -u  # all unique includes

# fd — find files
fd -e cpp -e hpp                            # all C++ files
fd --changed-within 1day                    # modified today

# fzf — fuzzy find
fd -e cpp | fzf --preview 'bat {}'          # fuzzy open with preview
git branch | fzf | xargs git checkout       # fuzzy branch switch

# bat — cat with syntax
bat src/main.cpp --diff                     # show git changes in gutter
```

---

## 7. Static Analysis

> Full reference: [static-analysis.md](static-analysis.md)
> Tools: [cppcheck](https://cppcheck.sourceforge.io), [clang-tidy](https://clang.llvm.org/extra/clang-tidy/), [ast-grep](https://ast-grep.github.io)

```bash
# cppcheck — find bugs without compiling
cppcheck --enable=all --suppress=missingIncludeSystem src/

# clang-tidy (needs compile_commands.json from cmake configure)
clang-tidy -p build src/main.cpp
clang-tidy -p build --fix src/main.cpp      # auto-fix what it can

# clang-format (https://clang.llvm.org/docs/ClangFormat.html)
clang-format -style=llvm -dump-config > .clang-format
clang-format -i src/**/*.cpp src/**/*.hpp   # format all files

# ast-grep — structural search (AST-aware, not text)
sg -p 'new $$$' --lang cpp                  # all raw new
sg -p 'NULL' -r 'nullptr' --lang cpp -i     # replace all NULLs with nullptr
```

---

## 8. Debug & Profile

> Full pro reference with thread control & reverse debugging: [debugging-profiling.md](debugging-profiling.md)
> Debuggers: [GDB](https://www.gnu.org/software/gdb/), [LLDB](https://lldb.llvm.org), [raddebugger](https://github.com/EpicGamesExt/raddebugger), [x64dbg](https://x64dbg.com)
> Profiler: [Tracy](https://github.com/wolfpld/tracy), [Perfetto](https://perfetto.dev), [Valgrind](https://valgrind.org)

### GDB
```bash
gdb -tui ./myapp
# b main | r | n | s | c | p var | bt | bt full | watch var | layout split | thread apply all bt
```

### LLDB (preferred on macOS/iOS — ships with [Xcode](https://developer.apple.com/xcode/))
```bash
lldb ./myapp
# b main | run | next | step | continue | print var | po obj | bt | frame variable | expr myFunc(42)
```

### 🧵 Thread isolation — debug one thread, freeze all others

> **Did you know?** You can freeze all threads except one. This is HUGE for multithreaded apps.

```bash
# GDB — freeze all threads except current
info threads                                 # list threads
thread 3                                     # switch to thread 3
set scheduler-locking on                     # NOW only YOUR thread steps
n                                            # next — only thread 3 moves
s                                            # step — only thread 3 moves
set scheduler-locking off                    # resume all

# LLDB — same concept
thread list                                  # list threads
thread select 3                              # switch to thread 3
settings set target.process.stop-others true # freeze others during step
next                                         # only thread 3 moves
settings set target.process.stop-others false
```

### ⏪ Reverse debugging — step BACKWARDS in time

> **Did you know?** GDB can record execution and replay it backwards. Find "how did we get here?"

```bash
# GDB — record & reverse
(gdb) target record-full                     # start recording EVERY instruction
(gdb) continue                                # run forward normally
# ... hit breakpoint or crash ...
(gdb) reverse-step                            # step BACKWARDS
(gdb) reverse-continue                        # run backwards to previous breakpoint
(gdb) reverse-next                            # next line, backwards
```

### Windows crash minidumps + LLDB

> **Did you know?** LLDB on Windows can open .dmp files directly. No WinDbg needed for basic analysis.

```bash
# Open minidump in LLDB
lldb -c crash.dmp
(lldb) bt                                    # backtrace at crash point
(lldb) thread list                           # all threads at crash
(lldb) frame variable                        # locals at crash
(lldb) image list                            # loaded modules

# Or WinDbg for deeper analysis:
windbg -z crash.dmp
!analyze -v                                  # auto-analysis
kb                                           # call stack with params
!heap -p -a <address>                        # analyze heap allocation
```

### Sanitizers ([ASan/UBSan/TSan docs](https://github.com/google/sanitizers/wiki))
```bash
cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
```

### Tracy (always-on profiler)
```cpp
#include <Tracy.hpp>
void heavy() { ZoneScoped; TracyPlot("dt", dt); FrameMark; }
```
```bash
tracy-profiler & ./build/dev/myapp          # GUI connects automatically
```

### Advanced debugging
- **LLDB remote** for iOS/[Android NDK](https://developer.android.com/ndk) → [mobile.md](mobile.md)
- **[WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/)** kernel debug, crash dumps → [reverse-engineering.md](reverse-engineering.md)
- **[raddebugger](https://github.com/EpicGamesExt/raddebugger)** — fast native Windows debugger by Epic
- **[x64dbg](https://x64dbg.com)** — runtime debugger for stripped binaries
- **[Valgrind](https://valgrind.org)** — `valgrind --leak-check=full ./myapp` for memory leaks (Linux)

---

## 9. Network Debug

> Full reference with tcpdump, Wireshark, DNS, TLS decryption: [network-debugging.md](network-debugging.md)
> Tools: [tcpdump](https://www.tcpdump.org), [Wireshark](https://www.wireshark.org), [curl](https://curl.se), [netcat](https://netcat.sourceforge.net)

### Why can't my app connect?

```bash
# Step 1: Can I reach the host?
ping -c 4 host.com                           # by name
ping -c 4 1.2.3.4                            # by IP (bypasses DNS)

# Step 2: Is DNS working?
dig host.com                                 # DNS lookup (Linux/macOS)
nslookup host.com                            # all platforms

# Step 3: Is the port open?
nc -zv host.com 443 -w 3                     # TCP connect test
# PowerShell:
Test-NetConnection host.com -Port 443

# Step 4: What's the route?
traceroute host.com                          # Linux/macOS
tracert host.com                             # Windows
```

### Packet capture

```bash
# tcpdump — capture everything on a port
sudo tcpdump -i any port 8080 -w capture.pcap
# Open capture.pcap in Wireshark

# Filter by host + save
sudo tcpdump -i any -w capture.pcap host api.example.com

# Show only connection attempts (SYN) and refusals (RST)
sudo tcpdump -i any 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'

# DNS queries only
sudo tcpdump -i any port 53
```

### TLS decryption in Wireshark

```bash
# Set SSLKEYLOGFILE before running your app
export SSLKEYLOGFILE=~/sslkeys.log
curl https://api.example.com                 # or run your app
# In Wireshark: Preferences → Protocols → TLS → set key log file
# Now you see decrypted HTTP content!
```

### curl timing — where is it slow?

```bash
curl -o /dev/null -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nFirst byte: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://api.example.com
# If DNS is slow → check /etc/resolv.conf
# If Connect is slow → firewall or routing issue
# If TLS is slow → certificate chain too long
# If First byte is slow → server-side processing
```

### Host spoofing (test against different server)

```bash
# /etc/hosts — redirect domain to localhost
sudo sh -c 'echo "127.0.0.1 api.example.com" >> /etc/hosts'
# Or without modifying hosts:
curl --resolve api.example.com:443:127.0.0.1 https://api.example.com
```

---

## 10. Cross-Compile

> Full reference: [cross-compilation.md](cross-compilation.md)
> Cross-compiler: [Zig](https://ziglang.org) — one tool, every target
> Emulator: [QEMU](https://www.qemu.org) — test Linux ARM/ARM64 binaries on any host

```bash
# Zig — one-command cross-compile
zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux
zig c++ -target aarch64-linux-gnu -O2 main.cpp -o main_arm
zig c++ -target x86_64-macos-none -O2 main.cpp -o main_mac

# Test with QEMU
qemu-x86_64 ./main_linux
qemu-x86_64 -strace ./main_linux            # trace syscalls

# CMake + Zig toolchain
cmake -B build/linux -DCMAKE_TOOLCHAIN_FILE=zig-toolchain.cmake -G Ninja
```

---

## 11. Packages

> Full reference: [cmake-package-managers.md](cmake-package-managers.md)
> Managers: [vcpkg](https://vcpkg.io), [Conan](https://conan.io), [CPM.cmake](https://github.com/cpm-cmake/CPM.cmake)

```bash
# vcpkg (install: scoop install vcpkg / brew install vcpkg)
cmake -B build -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
vcpkg install fmt spdlog nlohmann-json       # classic mode

# Conan 2 (install: pip install conan)
conan install . --output-folder=build --build=missing
cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
```

---

## 12. Task Runner

> Full reference: [justfile.md](justfile.md)
> Tool: [just](https://github.com/casey/just) — like Make but without the pain

```makefile
# justfile
default:
    @just --list

build preset="dev":
    cmake --preset {{preset}}
    cmake --build --preset {{preset}}

test preset="dev":
    ctest --preset {{preset}} --output-on-failure

watch preset="dev":
    watchexec -e cpp,hpp,h,c,cmake,just -- just build {{preset}}

lint:
    cppcheck --enable=all --suppress=missingIncludeSystem src/

format:
    clang-format -i src/**/*.cpp src/**/*.hpp

todos:
    rg "TODO|FIXME|HACK|XXX" --type cpp -n
```

```bash
just build preset=clang
just watch
just todos
```

---

## 13. Benchmarking

> Full reference: [benchmarking.md](benchmarking.md)
> Tool: [hyperfine](https://github.com/sharkdp/hyperfine) — statistical CLI benchmarking

```bash
# Basic
hyperfine --warmup 5 --min-runs 20 "./myapp"

# Compare two versions
hyperfine "./old" "./new"

# Parameter sweep
hyperfine -P size 1024 65536 "./myapp --buf {size}"

# Compilation benchmark
hyperfine --prepare 'rm -rf build && cmake -B build -G Ninja' 'cmake --build build'
```

---

## 14. File Tracking

> What files did that command/exe actually produce?

### Linux/macOS

```bash
# strace — trace all file opens/writes (https://strace.io)
strace -f -e trace=open,openat,write -o trace.log ./myapp
strace -f -e trace=openat ./myapp 2>&1 | grep "O_CREAT"   # only new files

# macOS — fs_usage (built-in, comprehensive)
fs_usage -f filesys ./myapp

# Quick before/after snapshot
touch marker && ./myapp && find . -newer marker -type f
```

### Windows

```powershell
# procmon — GUI file/registry/network tracker ([Sysinternals](https://learn.microsoft.com/en-us/sysinternals/))
procmon /Quiet /Minimized /BackingFile trace.pml /AcceptEula
./myapp.exe
procmon /Terminate
procmon /OpenLog trace.pml /SaveAs trace.csv

# drstrace — CLI syscall tracing
scoop install drmemory
drstrace -- myapp.exe                        # → drstrace.myapp.exe.PID.log
```

### Output tracking table

| Command | Produces | Where |
|---------|----------|-------|
| `cmake -B build` | `CMakeCache.txt`, `build.ninja` | `build/` |
| `cmake --build build` | `.o`, `.a`, `.exe`, `.so`, `.dylib` | `build/` |
| `clang-tidy --fix` | Modified `.cpp` (in-place) | source tree |
| `clang-format -i` | Modified `.cpp` (in-place) | source tree |
| `upx --best app` | Compressed binary (in-place) | same path |
| `strip app` | Stripped binary (in-place) | same path |
| `rgp capture` | `*.rgp` GPU trace | specified path |
| `tracy-profiler` | `*.tracy` capture | current dir |
| `doxygen` | `html/`, `latex/` | `docs/` or configured |

---

## 15. Mobile Dev

> Full reference: [mobile.md](mobile.md)
> Android: [ADB docs](https://developer.android.com/tools/adb), [Android NDK](https://developer.android.com/ndk)
> iOS: [devicectl](https://developer.apple.com/documentation/xcode/using-xcode-devicectl), [xcrun](https://developer.apple.com/documentation/xcode/running-your-app-in-the-simulator-or-on-a-device)

### Android

```bash
# Install: scoop install adb / brew install android-platform-tools
adb devices -l                              # list devices
adb install -r app.apk                      # install/reinstall
adb logcat -s MyTag *:E                     # filtered errors

# Root (eng/userdebug builds or [Magisk](https://topjohnwu.github.io/Magisk/))
adb root                                    # restart adbd as root
adb shell setenforce 0                      # disable SELinux
adb shell getenforce                        # check: Enforcing/Permissive

# Google APIs image (no Play Services — for clean testing)
sdkmanager "system-images;android-34;google_apis;arm64-v8a"
emulator -avd pixel_test -writable-system -no-snapshot
```

### iOS (macOS only — requires [Xcode](https://developer.apple.com/xcode/))

```bash
xcrun simctl list devices                   # list simulators
xcrun simctl boot "iPhone 15"               # boot one
xcrun simctl install booted MyApp.app       # install
xcrun simctl io booted screenshot screen.png

# Physical device (Xcode 15+)
xcrun devicectl list devices
xcrun devicectl device install app --device <id> MyApp.app

# Build
xcodebuild -project MyApp.xcodeproj -scheme MyApp -sdk iphonesimulator build

# LLDB remote debug
lldb -n MyApp
```

---

## 16. Reverse Engineering

> Full reference: [reverse-engineering.md](reverse-engineering.md)
> Tools: [Ghidra](https://ghidra-sre.org) (NSA decompiler), [radare2](https://rada.re) (RE framework), [x64dbg](https://x64dbg.com) (Windows debugger), [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/) (kernel debug), [IDA](https://hex-rays.com/ida-free/)

```bash
# Quick recon
file myapp                                   # ELF/PE/Mach-O?
checksec --file=myapp                        # ASLR, DEP, PIE, canary?
strings myapp | rg -i "password|secret|key|token"

# Ghidra — decompile to C
ghidra                                       # launch GUI
# F5 = decompile | L = rename | X = cross-refs | G = go to address

# radare2 — CLI RE
r2 -A myapp                                  # open + full analyze
# afl = functions | iz = strings | pdf = disasm | VV = graph | s main = seek to main

# x64dbg — Windows runtime debug
x64dbg myapp.exe
# F2 = breakpoint | F7 = step in | F8 = step over | F9 = run

# WinDbg — kernel debug + crash dumps
windbg -z crash.dmp
# !analyze -v | kb | lm | .sympath srv*https://msdl.microsoft.com/download/symbols

# Binary patching (r2)
oo+                                        # reopen in write mode
wa nop @ 0x401000                            # NOP out instruction
```

---

## 17. Binary Tools

> Full reference: [binary-tools.md](binary-tools.md)
> Tools: [UPX](https://upx.github.io), [pe-bear](https://github.com/hasherezade/pe-bear), [ImHex](https://github.com/WerWolv/ImHex)

```bash
# Compress
strip myapp.exe && upx --best myapp.exe

# DLL/dylib/so dependencies
depends myapp.exe                            # Windows ([dependencywalker.com](https://www.dependencywalker.com))
ldd myapp                                    # Linux (built-in)
otool -L myapp                               # macOS (built-in)

# PE/ELF inspection
pe-bear myapp.exe                            # Windows PE viewer
readelf -h myapp                             # ELF header
objdump -h myapp.exe                         # PE sections

# Strings audit
strings myapp.exe | rg -i "password|secret|pdb"

# Pre-ship checklist
checksec --file=myapp                        # security flags
strings myapp | rg "\.pdb"                   # debug symbols leaked?
strings myapp | rg -i "password"             # secrets?
depends myapp.exe                            # missing DLLs?
```

---

## 18. Controls & Input

> Full reference: [controls.md](controls.md)
> Source: [Valve Steam Input docs](https://partner.steamgames.com/doc/features/steam_controller/getting_started_for_devs), [ISteamInput API](https://partner.steamgames.com/doc/api/ISteamInput), [Steamworks](https://partner.steamgames.com/doc)

### [Steam's 5 Golden Rules](https://partner.steamgames.com/doc/features/steam_controller/getting_started_for_devs)

1. **On-screen icons match device** — detect controller type, swap glyphs at runtime
2. **Cursor matches device** — visible for mouse, hidden for gamepad
3. **All devices work 100%** — test disconnect/reconnect, [Steam Remote Play](https://store.steampowered.com/remoteplay)
4. **All inputs navigate menus** — d-pad, stick, mouse all traverse UI
5. **Disconnect → pause** — single player pause, multiplayer mark disconnected

**Bonus**: Gamepad + mouse must work **simultaneously** — #1 cause of [Steam Input](https://partner.steamgames.com/doc/features/steam_controller) compatibility issues.

```cpp
// Steam Input API — get real controller type for correct glyphs
SteamInput()->Init();
SteamInput()->RunFrame();
InputHandle_t ctrl = SteamInput()->GetControllerForGamepadIndex(0);
EInputActionOrigin origin = SteamInput()->GetActionOriginFromXboxOrigin(ctrl, k_EXboxOrigin_A);
const char* glyph = SteamInput()->GetGlyphForActionOrigin(origin);
// Load glyph as texture → future-proof for any new controller
```

> **Critical**: Camera/aim action type MUST be `absolute_mouse` in [IGA file](https://partner.steamgames.com/doc/features/steam_controller/getting_started_for_devs#igas). Steam Input converts gyro/trackpad/stick → `absolute_mouse`. Cannot convert back from `joystick_move`.

---

## 19. Windows C++

> Full reference: [windows-cpp.md](windows-cpp.md)

```bash
# Static CRT (no vcruntime.dll dependency)
cmake -B build -DCMAKE_MSVC_RUNTIME_LIBRARY="MultiThreaded$<$<CONFIG:Debug>:Debug>"

# PDB in release (crash dumps work)
cmake -B build -DCMAKE_CXX_FLAGS_RELEASE="/Zi" -DCMAKE_EXE_LINKER_FLAGS_RELEASE="/DEBUG /OPT:REF /OPT:ICF"
```

```powershell
# Defender exclusion for build dirs (admin — massive speedup)
Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"

# Long paths (admin — fixes deep build tree issues)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

| Compiler | Best for |
|----------|----------|
| **MSVC** (`cl`) | Production, best PDB, [Visual Studio](https://visualstudio.microsoft.com) integration |
| **Clang-cl** | [Sanitizers](https://clang.llvm.org/docs/AddressSanitizer.html), clang-tidy, cross-platform parity |
| **GCC** (MSYS2) | POSIX APIs, autotools compat |

---

## 20. Resources

> Full reference with all links: [resources.md](resources.md)

### Must-Read Books

| Book | Author | Focus |
|------|--------|-------|
| [Professional CMake](https://crascit.com/professional-cmake/) | Craig Scott | CMake definitive guide |
| [C++ Best Practices](https://leanpub.com/cppbestpractices) | [Jason Turner](https://www.youtube.com/@lefticus) | 45+ actionable rules |
| [Game Engine Architecture](https://www.gameenginebook.com) | Jason Gregory | Engine design (Naughty Dog) |
| [Practical Reverse Engineering](https://www.wiley.com/en-us/Practical+Reverse+Engineering-p-9781118787311) | Dang et al. | RE for x86/ARM/Windows |
| [Optimized C++](https://www.oreilly.com/library/view/optimized-c/9781491922057/) | Kurt Guntheroth | Performance patterns |

### Must-Watch

| Channel | Focus |
|---------|-------|
| [C++ Weekly](https://www.youtube.com/@lefticus) (Jason Turner) | 600+ weekly C++ episodes |
| [The Cherno](https://www.youtube.com/@TheCherno) | Game engine dev, C++ deep dives |
| [CppCon](https://www.youtube.com/@CppCon) | Annual conference, 300+ talks |
| [LiveOverflow](https://www.youtube.com/@LiveOverflow) | Binary exploitation, RE |
| [GDC](https://www.youtube.com/@GDC) | Game dev conference talks |
| [GPUOpen](https://www.youtube.com/@GPUOpen) | AMD GPU tools, Radeon profiling |

### Essential Online

| Resource | Link |
|----------|------|
| C++ reference | [en.cppreference.com](https://en.cppreference.com) |
| Compiler Explorer | [godbolt.org](https://godbolt.org) |
| C++ Core Guidelines | [isocpp.github.io/CppCoreGuidelines](https://isocpp.github.io/CppCoreGuidelines/) |
| Modern CMake | [cliutils.gitlab.io/modern-cmake](https://cliutils.gitlab.io/modern-cmake/) |
| Steamworks | [partner.steamgames.com/doc](https://partner.steamgames.com/doc) |
| Ghidra | [ghidra-sre.org](https://ghidra-sre.org) |

---

## ⚡ Quick Reference Card

```
INSTALL:   scoop install ripgrep fd bat fzf just cmake ninja ccache llvm zig lazygit
BUILD:     cmake -B build -G Ninja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON && cmake --build build
TARGETS:   cmake --build build --target help | sort
FEATURES:  cmake -B build -LAH | rg -i "feature|enable|option"
GRAPH:     cmake -B build --graphviz=deps.dot && dot -Tpng deps.dot -o deps.png
PROFILE:   cmake -B build --profiling-output=prof.json --profiling-format=google-trace
TRACE:     cmake -B build --trace-expand 2>&1 | tee trace.log
LAZYGIT:   lazygit                          (visual rebase, cherry-pick, bisect)
REBASE:    git rebase -i HEAD~5             (or lazygit: [3] → 'e'/'s'/'d')
BISECT:    git bisect start HEAD v1.0 && git bisect run cmake --build build && ctest
WORKTREE:  git worktree add ../hotfix hotfix-branch
PROC_SUB:  diff <(cmd1 | sort) <(cmd2 | sort)  (compare outputs, no temp files)
PARALLEL:  fd -e cpp | xargs -P 8 -I {} clang-format -i {}
WATCH:     watchexec -e cpp,hpp -- cmake --build build
SEARCH:    rg "pattern" --type cpp -n -C 3
FIND:      fd -e cpp -e hpp
TODOS:     rg "TODO|FIXME|HACK" --type cpp -C 2
BLAME:     git blame src/file.cpp | bat -l gitblame
HISTORY:   git log -S "functionName" --oneline
HOT FILES: git log --format=format: --name-only --since="6 months ago" | sort | uniq -c | sort -rn | head -15
DIFF:      git diff main...HEAD --stat
BENCH:     hyperfine --warmup 5 --min-runs 20 "./myapp"
LINT:      cppcheck --enable=all --suppress=missingIncludeSystem src/
FORMAT:    clang-format -i src/**/*.cpp src/**/*.hpp
PROFILE:   tracy-profiler & ./myapp
DEBUG:     gdb -tui ./myapp  |  lldb ./myapp
THREADS:   set scheduler-locking on         (GDB: freeze all threads except current)
REVERSE:   target record-full → reverse-step (GDB: step BACKWARDS in time)
MINIDUMP:  lldb -c crash.dmp → bt           (LLDB: open Windows crash dump)
NETWORK:   nc -zv host 443 -w 3             (is port open?)
CAPTURE:   sudo tcpdump -i any port 8080 -w cap.pcap
TLS DEBUG: export SSLKEYLOGFILE=~/sslkeys.log → Wireshark decrypts HTTPS
CURL TIME: curl -o /dev/null -w "%{time_total}s\n" https://host
CACHE:     ccache -s
COMPILEDB: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON && ln -sf build/compile_commands.json .
STRACE:    strace -f -e trace=openat ./myapp  |  drstrace -- myapp.exe
TRACK:     touch marker && ./myapp && find . -newer marker -type f
GPU:       rgp capture --output frames.rgp --app myapp --frame 42
ANDROID:   adb devices -l && adb install -r app.apk && adb logcat -s Tag
IOS:       xcrun simctl boot "iPhone 15" && xcrun simctl install booted App.app
RE:        r2 -A myapp  |  ghidra  |  x64dbg myapp.exe
CHECKSEC:  checksec --file=myapp
SHIP:      strip app && upx --best app && depends app && strings app | rg pdb
STEAM:     SteamInput()->Init() + GetGlyphForActionOrigin() → future-proof glyphs
CROSS:     zig c++ -target x86_64-linux-gnu -O2 main.cpp -o main_linux
LOC:       scc src/
AI STUDY:  aider --read-only src/engine/     (ask questions about code)
```

---

## 🗺️ Dev Flow

```
CLONE REPO ──► scc + tree + bat README ──► git shortlog + log (orient)
                                              │
WRITE CODE ──► lazygit commit ─────────► cppcheck + clang-tidy (analyze)
                    │                            │
                    ▼                            ▼
              gdb / lldb                  sanitizers (ASAN/UBSAN)
              set scheduler-locking on    thread isolation
              target record-full          reverse debugging
                    │                            │
                    └──────────┬─────────────────┘
                               ▼
                          tracy profile (CPU) | RGP profile (GPU)
                               ▼
                          hyperfine bench
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
          cross-compile   strace/tcpdump    ship
          Zig/QEMU        (trace files,     checksec + strip + UPX
          ADB deploy      packets,          depends/ldd check
          xcrun deploy    network, TLS)     strings secret scan
                               │            Steam Input (IGA)
                               ▼            Big Picture mode
                          git bisect (find the regression)
                               ▼
                          Ghidra/r2/x64dbg (when you need to go deeper)
                               ▼
                          lldb -c crash.dmp (analyze minidumps)
```

---

Companion deep-dives linked inline throughout. Install via [scoop](https://scoop.sh), [apt](https://ubuntu.com), or [brew](https://brew.sh).
