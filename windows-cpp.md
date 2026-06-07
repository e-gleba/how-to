# Windows C++ Development — Ultimate Setup

> From fresh Win 11 install to full C++ dev environment. Dev Mode, WSL, terminal, PowerToys, coreutils, compilers, CRT, PDB.

## 🚀 First Things First — Win 11 Setup

### Enable Developer Mode

> Unlocks sideloading, symlink creation, OpenSSH server, and dev tools without admin prompts.

```powershell
# Settings → System → For Developers → Developer Mode → ON
# Or via registry (admin):
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" /t REG_DWORD /f /v "AllowDevelopmentWithoutDevLicense" /d "1"

# Verify:
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" -Name "AllowDevelopmentWithoutDevLicense"
```

What you get:
- **Symlinks without admin** — `mklink` works in normal terminal
- **OpenSSH Server** — `Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0`
- **Windows Subsystem for Linux** — `wsl --install`
- **WinGet dev packages** — `winget install` without UAC

### ⌨️ Essential Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| **Win + X, I** | Open Windows Terminal (Power User menu → Terminal) |
| **Win + X, A** | Open Terminal as Admin |
| **Win + X** | Power User menu (Device Manager, Disk Mgmt, Event Viewer, Task Manager) |
| **Win + R** | Run dialog → `wt` for terminal, `explorer .` for current dir |
| **Ctrl + Shift + Esc** | Task Manager directly |
| **Win + V** | Clipboard history (enable it!) |
| **Win + . ** | Emoji/symbol picker |

> **Win + X + I** is the fastest way to open a terminal. Power User menu → Terminal. No searching, no clicking.

### Windows Terminal — vanilla install (NOT from Store)

> Store version auto-updates silently, can break profiles, slower. Scoop/GitHub = full control.

```powershell
# Option 1: Scoop (preferred — clean, updatable)
scoop install windows-terminal

# Option 2: GitHub release (latest, manual)
# https://github.com/microsoft/terminal/releases → download .msixbundle
# Install: Add-AppxPackage .\Microsoft.WindowsTerminal_<version>.msixbundle

# Option 3: winget (Microsoft official, but Store-adjacent)
winget install Microsoft.WindowsTerminal
```

After install, set as default terminal:
```powershell
# Settings → System → For Developers → Terminal → set "Let Windows choose" to "Windows Terminal"
# Or: Settings → Privacy & Security → For Developers → Terminal → Windows Terminal
```

#### Windows Terminal settings.json essentials

```jsonc
{
    "defaultProfile": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",  // PowerShell 7
    "profiles": {
        "defaults": {
            "font": { "face": "CaskaydiaCove Nerd Font", "size": 11 },
            "colorScheme": "Catppuccin Mocha",
            "opacity": 92,
            "useAcrylic": true,
            "cursorShape": "bar",
            "startingDirectory": "~"
        },
        "list": [
            {
                "name": "PowerShell 7",
                "commandline": "pwsh.exe -NoLogo",
                "icon": "ms-appx:///ProfileIcons/{574e775e-4f2a-5b96-ac1e-a2962a402336}.png"
            },
            {
                "name": "WSL Ubuntu",
                "commandline": "wsl.exe ~",
                "startingDirectory": "\\\\wsl$\\Ubuntu\\home\\user"
            },
            {
                "name": "x64 Native Tools",
                "commandline": "cmd.exe /k \"C:\\Program Files\\Microsoft Visual Studio\\2022\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat\""
            }
        ]
    },
    "keybindings": [
        { "keys": "ctrl+shift+f", "command": "find" },
        { "keys": "ctrl+shift+t", "command": "newTab" },
        { "keys": "ctrl+shift+w", "command": "closePane" },
        { "keys": "alt+shift+plus", "command": "splitPane", "args": { "split": "auto", "splitMode": "duplicate" } },
        { "keys": "alt+left", "command": "moveFocus", "args": { "direction": "left" } },
        { "keys": "alt+right", "command": "moveFocus", "args": { "direction": "right" } }
    ]
}
```

> 💡 Install Nerd Font first: `scoop install nerd-fonts` or download [Cascadia Code](https://github.com/microsoft/cascadia-code/releases).

---

### 🛠️ PowerToys — must-have utilities

```powershell
# Install via scoop (preferred)
scoop install powertoys

# Or winget:
winget install Microsoft.PowerToys
```

| PowerToy | What it does | Shortcut |
|----------|-------------|----------|
| **PowerToys Run** | Spotlight-like launcher (apps, files, calc, unit convert) | `Alt + Space` |
| **FancyZones** | Custom window layouts — snap to zones | `Win + Shift + ``` |
| **Color Picker** | Pick any color from screen | `Win + Shift + C` |
| **Text Extractor** | OCR any text from screen | `Win + Shift + T` |
| **Always on Top** | Pin window above others | `Win + Ctrl + T` |
| **Keyboard Manager** | Remap keys and shortcuts | GUI only |
| **PowerRename** | Batch rename in Explorer (regex) | Right-click → PowerRename |
| **Image Resizer** | Batch resize images | Right-click → Resize |
| **Mouse Jump** | Teleport mouse across large screens | `Win + Shift + D` |
| **Peek** | Preview files without opening (like macOS Space) | `Ctrl + Space` in Explorer |
| **Hosts File Editor** | GUI for /etc/hosts | GUI only |
| **Registry Preview** | Visual .reg file editor | GUI only |

> **Top 3 for devs**: PowerToys Run (Alt+Space), FancyZones (window management), Color Picker (UI debugging).

---

### 🐧 WSL — Windows Subsystem for Linux

```powershell
# One command (admin PowerShell):
wsl --install

# Installs WSL2 + Ubuntu by default. Reboot required.

# After reboot — set default version:
wsl --set-default-version 2

# List available distros:
wsl --list --online

# Install specific distro:
wsl --install -d Ubuntu-24.04
wsl --install -d Debian

# Manage:
wsl --list --verbose                         # installed distros + state
wsl --set-default Ubuntu-24.04               # default distro
wsl --shutdown                                # stop all WSL
wsl --update                                  # update WSL kernel
```

#### WSL tips for C++ dev

```bash
# Inside WSL — install dev tools:
sudo apt update && sudo apt install -y \
  build-essential cmake ninja-build ccache \
  gdb lldb clang clang-tidy clang-format \
  ripgrep fd-find bat fzf \
  libx11-dev libxrandr-dev libgl-dev libvulkan-dev

# Access Windows files from WSL:
ls /mnt/c/Users/you/projects/                # Windows C: drive

# Access WSL files from Windows Explorer:
# Type in Explorer address bar: \\wsl$\Ubuntu\home\you\

# Use Windows tools from WSL:
cmake.exe -B build -G Ninja                  # call Windows cmake
code .                                        # open VS Code

# Performance tip: keep projects in WSL filesystem, not /mnt/c/
# /mnt/c/ (9P filesystem) is 5-10x slower for file operations
# WSL native ext4 filesystem is full Linux speed
mkdir -p ~/projects && cd ~/projects         # ← store projects here

# Networking — WSL2 uses NAT:
# Access WSL server from Windows: localhost:<port> (auto-forwarded)
# Access Windows server from WSL: $(hostname).local or $(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
```

#### WSL ↔ VS Code integration

```bash
# Install VS Code "WSL" extension
# Then from WSL:
code .                                       # opens VS Code connected to WSL filesystem
# Full Linux dev experience with Windows GUI editor
```

---

### 📦 Coreutils & Unix Tools on Windows

> Windows PowerShell is missing many Unix staples. Fix that.

```powershell
# uutils-coreutils — Rust rewrite of GNU coreutils (fast, complete)
scoop install uutils-coreutils

# Now you get: ls, cat, cp, mv, mkdir, touch, head, tail, sort, uniq, wc, etc.
# Works in both PowerShell and cmd

# Or GNU coreutils via MSYS2 (full POSIX):
scoop install msys2

# Essential Unix-style tools via scoop:
scoop install wget curl jq yq tree which less vim nano
scoop install sed grep gawk                      # GNU text processing
scoop install openssh                             # ssh, scp, sftp
scoop install zip unzip gzip tar                  # archive tools

# Make PowerShell aliases for GNU tools:
# Add to $PROFILE:
# Set-Alias -Name grep -Value findstr            # or use ripgrep
# Set-Alias -Name ls -Value Get-ChildItem         # PS default
# Set-Alias -Name cat -Value Get-Content           # PS default
```

#### PowerShell vs CMD vs Bash

| Shell | Best for | Launch |
|-------|---------|--------|
| **PowerShell 7** (`pwsh`) | Windows admin, .NET, Azure, scripting | Win+X+I → default |
| **CMD** | Legacy scripts, `vcvars64.bat` | `cmd.exe` |
| **WSL Bash** | Linux tools, cross-platform scripts, autotools | `wsl` |
| **Git Bash** | Quick git operations, light Unix tools | `scoop install git` includes it |

> 💡 Use **PowerShell 7** (not 5.1). Install: `scoop install pwsh` or `winget install Microsoft.PowerShell`. v7 has `&&` operator, ternary, null-coalescing, SSH remoting.

#### PowerShell profile (add to `$PROFILE`)

```powershell
# Open profile: notepad $PROFILE

# Useful aliases and functions:
Set-Alias -Name ll -Value Get-ChildItem
function la { Get-ChildItem -Force }
function .. { Set-Location .. }
function ... { Set-Location ..\.. }
function mkcd($dir) { New-Item -ItemType Directory -Force -Path $dir; Set-Location $dir }

# Quick dev shortcuts:
function vs { & "C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat" }
function proj { Set-Location ~/projects }

# FZF integration:
Set-PSReadLineKeyHandler -Chord "Ctrl+r" -ScriptBlock {
    $history = Get-Content (Get-PSReadlineOption).HistorySavePath | Select-Object -Unique | fzf
    [Microsoft.PowerShell.PSConsoleReadLine]::Insert($history)
}
```

---

## Toolchains

| Compiler | Path | Best for |
|----------|------|----------|
| MSVC (`cl`) | VS Build Tools | Windows-native, PDB support |
| Clang (`clang-cl`) | `scoop/apps/llvm/current/bin` | Cross-platform parity, sanitizers |
| GCC (`g++`) | `scoop/apps/msys2/current` | Linux-like, POSIX APIs |
| Zig (`zig c++`) | `scoop/apps/zig/current` | Cross-compilation |

### Install compilers

```powershell
# MSVC — Build Tools (no IDE, ~8 GB)
winget install Microsoft.VisualStudio.2022.BuildTools --override `
  "--add Microsoft.VisualStudio.Workload.VCTools --includeRecommended --passive"

# LLVM (clang + clang-cl + lld + lldb + clang-tidy + clang-format)
scoop install llvm

# MSYS2 (GCC + Unix tools on Windows)
scoop install msys2

# Zig (cross-compiler + C/C++ compiler)
scoop install zig
```

### Select compiler for CMake

```powershell
# MSVC (default on Windows):
cmake -B build -G "Visual Studio 17 2022"
# Or Ninja with vcvars:
# Open "x64 Native Tools Command Prompt" first, then:
cmake -B build -G Ninja

# Clang-cl (drop-in MSVC replacement):
cmake -B build -G Ninja -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl

# GCC via MSYS2:
cmake -B build -G Ninja -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++

# Zig as C/C++ compiler:
cmake -B build -G Ninja -DCMAKE_C_COMPILER="zig cc" -DCMAKE_CXX_COMPILER="zig c++"
```

---

## CRT linking

```cmake
# Static CRT (/MT, /MTd) — larger binary, no DLL dependency
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# Dynamic CRT (/MD, /MDd) — smaller binary, needs vcredist
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
```

| CRT | Flag | Ships | When to use |
|-----|------|-------|-------------|
| Static | `/MT` | Nothing extra | Standalone tools, single exe |
| Dynamic | `/MD` | `vcruntime140.dll` | Most apps, shared libs |
| Static Debug | `/MTd` | Nothing | Debug builds, local dev |
| Dynamic Debug | `/MDd` | Debug CRT DLL | Debug builds, DLL compat |

> ⚠️ **All linked DLLs must use same CRT.** Mixing `/MT` and `/MD` in same process = heap corruption. Use dynamic CRT (`/MD`) when linking third-party DLLs.

---

## PDB in release builds

```cmake
# Symbols for crash dumps even in release
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
```

> **Always ship PDBs to symbol server.** Users get stripped exe, you keep PDB for crash analysis. See [debugging-profiling.md](debugging-profiling.md) → Crash Dump section.

---

## Path handling

```cpp
// Use forward slashes — Windows accepts them everywhere
#include "src/utils/helpers.h"      // ✓
#include "src\\utils\\helpers.h"    // ✗

// std::filesystem handles conversion
#include <filesystem>
std::filesystem::path p = "src/utils";
p.make_preferred();  // / → \ on Windows
```

---

## Windows performance APIs

```cpp
// High-res timer (better than chrono on Windows)
#include <profileapi.h>
LARGE_INTEGER freq, start, end;
QueryPerformanceFrequency(&freq);
QueryPerformanceCounter(&start);
// ... work ...
QueryPerformanceCounter(&end);
double ms = (end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
```

```cpp
// Memory-mapped files
#include <windows.h>
HANDLE h = CreateFileA(path, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, 0, NULL);
HANDLE m = CreateFileMappingA(h, NULL, PAGE_READONLY, 0, 0, NULL);
void* data = MapViewOfFile(m, FILE_MAP_READ, 0, 0, 0);
// ... use data ...
UnmapViewOfFile(data); CloseHandle(m); CloseHandle(h);
```

---

## Environment tips

- **UTF-8 globally:** Control Panel → Region → Administrative → Change system locale → Beta: UTF-8
- **Defender exclusion:** `Add-MpPreference -ExclusionPath "C:\Users\YOU\projects"` (massive build speedup)
- **Long paths:** `New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1` (admin)
- **Case sensitivity:** `git config core.ignorecase false`
- **Forward slashes** everywhere — Windows accepts them, Linux requires them
- **Disable Cortana/search indexing on project dirs:** Properties → Advanced → uncheck "Allow files to have contents indexed"

---

## Recommended build type for daily dev

```cmake
# RelWithDebInfo — fast + symbols + profiling works on real code
set(CMAKE_BUILD_TYPE "RelWithDebInfo")
```

Not Debug (slow), not Release (no symbols). RelWithDebInfo is the sweet spot.

---

## Complete Dev Environment Checklist

```powershell
# 1. Enable Dev Mode
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" /t REG_DWORD /f /v "AllowDevelopmentWithoutDevLicense" /d "1"

# 2. Scoop
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# 3. Terminal
scoop install windows-terminal nerd-fonts

# 4. Shell
scoop install pwsh

# 5. Core tools
scoop install git cmake ninja ccache ripgrep fd bat fzf just lazygit

# 6. Compilers
scoop install llvm zig
winget install Microsoft.VisualStudio.2022.BuildTools --override "--add Microsoft.VisualStudio.Workload.VCTools --includeRecommended --passive"

# 7. Unix tools
scoop install uutils-coreutils wget curl jq tree sed grep openssh

# 8. PowerToys
scoop install powertoys

# 9. WSL
wsl --install

# 10. Performance
Add-MpPreference -ExclusionPath "C:\Users\$env:USERNAME\projects"
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

> 💡 **Tip:** Use `clang-cl` as a drop-in MSVC replacement. Same ABI, same flags, but adds sanitizers and better error messages.

---

**Related**: [Debugging & Profiling](debugging-profiling.md) — crash dumps, GDB/LLDB, sanitizers
**Related**: [Tools Install](tools-install.md) — full tool-by-tool install table
**Related**: [CMake](cmake.md) — build system deep dive
