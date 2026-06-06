# Tool Installation — Cross-Platform

> One command per platform. Scoop preferred on Windows.

## Legend

| Symbol | Meaning |
|--------|--------|
| ✅ | Available, recommended |
| ⚠️ | Available, caveats |
| ❌ | Not available via package manager |
| 🔧 | Manual install required |

## Core CLI Tools

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt (Ubuntu/Debian) | 🍎 Homebrew | What |
|------|----------|----------|------------------------|-------------|------|
| **ripgrep** | `scoop install ripgrep` | `choco install ripgrep` | `sudo apt install ripgrep` | `brew install ripgrep` | Search code fast |
| **fd** | `scoop install fd` | `choco install fd` | `sudo apt install fd-find` → alias `fd=fdfind` | `brew install fd` | Find files |
| **fzf** | `scoop install fzf` | `choco install fzf` | `sudo apt install fzf` | `brew install fzf` | Fuzzy finder |
| **bat** | `scoop install bat` | `choco install bat` | `sudo apt install bat` → alias `bat=batcat` | `brew install bat` | Cat + syntax |
| **just** | `scoop install just` | `choco install just` | `sudo apt install just` or `cargo install just` | `brew install just` | Task runner |
| **watchexec** | `scoop install watchexec` | ❌ | `sudo apt install watchexec` or `cargo install watchexec-cli` | `brew install watchexec` | Auto-rebuild on save |
| **hyperfine** | `scoop install hyperfine` | `choco install hyperfine` | `sudo apt install hyperfine` or `cargo install hyperfine` | `brew install hyperfine` | Benchmarking |
| **scc** | `scoop install scc` | ❌ | `go install github.com/boyter/scc/v3@latest` | `brew install scc` | Count code |

## Build System

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt | 🍎 Brew | What |
|------|----------|----------|--------|---------|------|
| **cmake** | `scoop install cmake` | `choco install cmake` | `sudo apt install cmake` | `brew install cmake` | Build system |
| **ninja** | `scoop install ninja` | `choco install ninja` | `sudo apt install ninja-build` | `brew install ninja` | Build tool |
| **ccache** | `scoop install ccache` | ❌ | `sudo apt install ccache` | `brew install ccache` | Compiler cache |
| **sccache** | `scoop install sccache` | ❌ | `cargo install sccache` | `brew install sccache` | Cloud compiler cache |
| **zig** | `scoop install zig` | ❌ | `snap install zig --classic --edge` | `brew install zig` | Cross-compiler |

## Compilers & LLVM

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt | 🍎 Brew | What |
|------|----------|----------|--------|---------|------|
| **llvm** (clang+lldb+tools) | `scoop install llvm` | `choco install llvm` | `sudo apt install llvm clang lld lldb` | `brew install llvm` | Full LLVM suite |
| **gcc/g++** | via MSYS2 | `choco install mingw` | `sudo apt install build-essential` | Xcode CLT | GNU toolchain |
| **MSVC** | 🔧 [VS Build Tools](https://visualstudio.microsoft.com/downloads/) | `choco install visualstudio2022buildtools` | ❌ | ❌ | Microsoft compiler |

## Static Analysis & Formatting

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt | 🍎 Brew | What |
|------|----------|----------|--------|---------|------|
| **cppcheck** | `scoop install cppcheck` | `choco install cppcheck` | `sudo apt install cppcheck` | `brew install cppcheck` | Static analysis |
| **clang-tidy** | via `scoop install llvm` | via `choco install llvm` | `sudo apt install clang-tidy` | via `brew install llvm` | Linter |
| **clang-format** | via `scoop install llvm` | via `choco install llvm` | `sudo apt install clang-format` | via `brew install llvm` | Formatter |
| **ast-grep** | `scoop install ast-grep` | ❌ | `npm install -g @ast-grep/cli` or `cargo install ast-grep` | `brew install ast-grep` | Structural search |

## Debuggers & Profilers

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt | 🍎 Brew | What |
|------|----------|----------|--------|---------|------|
| **gdb** | via MSYS2 | ❌ | `sudo apt install gdb` | `brew install gdb` | GNU debugger |
| **lldb** | via `scoop install llvm` | via `choco install llvm` | `sudo apt install lldb` | Xcode CLT | LLVM debugger |
| **raddebugger** | 🔧 [GitHub](https://github.com/EpicGamesExt/raddebugger) | ❌ | ❌ | ❌ | Fast native debugger |
| **x64dbg** | `scoop install x64dbg` | ❌ | ❌ | ❌ | Windows runtime debugger |
| **tracy** | 🔧 [GitHub](https://github.com/wolfpld/tracy/releases) | ❌ | `sudo apt install tracy` (some distros) | `brew install tracy` | Frame profiler |
| **renderdoc** | `scoop install renderdoc` | ❌ | `sudo apt install renderdoc` | ❌ (🍎 use Xcode GPU debugger) | GPU capture |
| **perfetto** | 🔧 [perfetto.dev](https://perfetto.dev) | ❌ | `sudo apt install perfetto` | ❌ | System tracing |

## Binary & Reverse Engineering

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt | 🍎 Brew | What |
|------|----------|----------|--------|---------|------|
| **upx** | `scoop install upx` | `choco install upx` | `sudo apt install upx` | `brew install upx` | Executable compressor |
| **ghidra** | `scoop install ghidra` | ❌ | 🔧 [ghidra-sre.org](https://ghidra-sre.org) | `brew install --cask ghidra` | Decompiler |
| **radare2** | `scoop install radare2` | ❌ | `sudo apt install radare2` | `brew install radare2` | RE framework |
| **cutter** | `scoop install cutter` | ❌ | 🔧 [cutter.re](https://cutter.re) | `brew install --cask cutter` | radare2 GUI |
| **imhex** | `scoop install imhex` | ❌ | 🔧 [GitHub](https://github.com/WerWolv/ImHex/releases) | `brew install --cask imhex` | Hex editor |
| **pe-bear** | `scoop install pe-bear` | ❌ | ❌ | ❌ | PE inspector |
| **depends** | 🔧 [depends.info](https://www.dependencywalker.com) | ❌ | `ldd` (built-in) | `otool -L` (built-in) | DLL/dylib dependencies |
| **strings** | via sysinternals | ❌ | `sudo apt install binutils` → `strings` | built-in | Extract strings |
| **cheat-engine** | `scoop install cheat-engine` | ❌ | 🔧 scanmem/gameconqueror | ❌ | Memory scanner |

## Mobile & Device

| Tool | 🪟 Scoop | 🪟 Choco | 🐧 apt | 🍎 Brew | What |
|------|----------|----------|--------|---------|------|
| **adb** | `scoop install adb` | `choco install adb` | `sudo apt install adb` | `brew install android-platform-tools` | Android debug bridge |
| **scrcpy** | `scoop install scrcpy` | `choco install scrcpy` | `sudo apt install scrcpy` | `brew install scrcpy` | Mirror Android screen |
| **android-studio** | `scoop install android-studio` | `choco install androidstudio` | 🔧 [developer.android.com](https://developer.android.com/studio) | `brew install --cask android-studio` | Full Android IDE |
| **android-clt** | `scoop install android-clt` | ❌ | via android-studio | `brew install --cask android-commandlinetools` | CLI tools only |
| **devicectl** | ❌ | ❌ | ❌ | via Xcode | iOS device management |
| **xcrun/xcodebuild** | ❌ | ❌ | ❌ | via Xcode | iOS/macOS build |
| **ios-deploy** | ❌ | ❌ | ❌ | `brew install ios-deploy` | Deploy to iOS device |
| **libimobiledevice** | ❌ | ❌ | `sudo apt install libimobiledevice-utils` | `brew install libimobiledevice` | iOS device tools |

## Sysinternals (Windows only)

```powershell
# All sysinternals
scoop install sysinternals

# Or individual
scoop install procmon procexp handle listdlls
```

| Tool | What |
|------|------|
| `procmon` | File/registry/network monitor |
| `procexp` | Process explorer |
| `handle` | Open handles viewer |
| `listdlls` | Loaded DLLs |
| `dbgview` | Debug output viewer |
| `autoruns` | Startup inspector |
| `tcpview` | Network connections |

## Tracing

| Tool | 🪟 Scoop | 🐧 apt | 🍎 Brew | What |
|------|----------|--------|---------|------|
| **drmemory** (drstrace) | `scoop install drmemory` | ❌ | ❌ | Syscall tracer (Windows) |
| **strace** | ❌ | `sudo apt install strace` | ❌ | Syscall tracer (Linux) |
| **ltrace** | ❌ | `sudo apt install ltrace` | ❌ | Library call tracer |
| **qemu** | `scoop install qemu` | `sudo apt install qemu-user` | `brew install qemu` | CPU emulator + strace |

---

## Platform-Specific Setup

### 🐧 Linux Essentials

```bash
# Ubuntu/Debian — one-shot dev environment
sudo apt update && sudo apt install -y \
  build-essential cmake ninja-build ccache \
  gdb lldb clang clang-tidy clang-format \
  ripgrep fd-find bat fzf \
  libx11-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev \
  libgl-dev libvulkan-dev \
  strace ltrace \
  pkg-config

# Fedora/RHEL
sudo dnf groupinstall "Development Tools"
sudo dnf install cmake ninja-build ccache gdb lldb clang ripgrep fd-find bat fzf

# Arch
sudo pacman -S base-devel cmake ninja ccache gdb lldb clang ripgrep fd bat fzf
```

### 🍎 macOS Essentials

```bash
# Xcode command line tools (gives clang, lldb, make, git)
xcode-select --install

# Homebrew for everything else
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install cmake ninja ccache ripgrep fd bat fzf just

# Vulkan SDK (optional)
brew install --cask vulkan-sdk
```

### 🪟 Windows Essentials

```powershell
# Scoop (preferred — fast, clean, no admin)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# Core dev tools in one line
scoop install git cmake ninja ccache ripgrep fd bat fzf just llvm zig

# Enable long paths (admin)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
    -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

# Defender exclusion for build dirs (admin)
Add-MpPreference -ExclusionPath "C:\Users\$env:USERNAME\projects"
```

### Scoop vs Chocolatey

| | Scoop | Chocolatey |
|---|-------|------------|
| **Admin rights** | ❌ Not needed | ✅ Usually needed |
| **Install location** | `~/scoop/` (clean) | `C:\ProgramData\chocolatey` |
| **Speed** | Fast | Slower |
| **PATH** | Auto-managed | Auto-managed |
| **Uninstall** | Clean | Sometimes leaves junk |
| **Verdict** | **Use this** | Backup only |

```powershell
# When scoop doesn't have it, try choco
choco install visualstudio2022buildtools --package-parameters "--add Microsoft.VisualStudio.Workload.VCTools"

# Or winget (Microsoft's official)
winget install Microsoft.VisualStudio.2022.BuildTools
```

---

## Verify Install

```bash
# After installing everything, verify:
cmake --version
ninja --version
ccache --version
rg --version
fd --version
clang --version
clang-tidy --version
clang-format --version
just --version

# All should print version numbers. If not, check PATH.
```

→ **Related**: [README.md](README.md) — daily usage one-liners for all these tools
