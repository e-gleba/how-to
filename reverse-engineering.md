# Reverse Engineering Cookbook

> Ghidra, radare2, x64dbg, WinDbg, IDA — one-liners for binary analysis and RE.

## Tool Overview

| Tool | Type | Platform | Install | Best for |
|------|------|----------|---------|----------|
| **Ghidra** | Decompiler + RE suite | Cross-platform | `scoop install ghidra` / `brew install --cask ghidra` | Decompilation, analysis |
| **IDA Pro/Free** | Disassembler + debugger | Cross-platform | 🔧 [hex-rays.com](https://hex-rays.com/ida-free/) | Professional disassembly |
| **radare2** | RE framework | Cross-platform | `scoop install radare2` / `apt install radare2` | CLI-based RE |
| **cutter** | radare2 GUI | Cross-platform | `scoop install cutter` | Visual RE |
| **x64dbg** | Runtime debugger | Windows | `scoop install x64dbg` | Windows binary analysis |
| **WinDbg** | Kernel + user debugger | Windows | 🔧 [Microsoft Store](https://apps.microsoft.com/detail/9pgjgd53tn86) | Kernel debugging, crash dumps |
| **GDB** | Debugger | Cross-platform | `apt install gdb` / `brew install gdb` | Linux/native debugging |
| **ImHex** | Hex editor | Cross-platform | `scoop install imhex` | Binary editing, patterns |
| **Binary Ninja** | Decompiler | Cross-platform | 🔧 [binary.ninja](https://binary.ninja/) | Modern decompiler |
| **pe-bear** | PE inspector | Windows | `scoop install pe-bear` | PE header analysis |
| **depends** | DLL dependency walker | Windows | 🔧 [dependencywalker.com](https://www.dependencywalker.com) | Missing DLLs |
| **checksec** | Security checker | Cross-platform | `scoop install checksec` / script | Binary security flags |

---

## Ghidra

### Setup

```bash
# Launch
ghidra                     # or from Start Menu / Applications

# Headless analysis (CI/automation)
# Linux/macOS:
/opt/ghidra/support/analyzeHeadless /tmp/project MyProgram -import ./myapp -postScript DecompileAll.java

# Windows:
C:\ghidra\support\analyzeHeadless.bat C:\tmp project -import myapp.exe
```

### Key shortcuts (Ghidra code browser)

| Key | Action |
|-----|--------|
| `G` | Go to address/symbol |
| `F5` | Decompile current function |
| `L` | Rename label/function |
| `T` | Set data type |
| `D` | Disassemble at cursor |
| `X` | Cross-references to |
| `Ctrl+Shift+F` | Search all |
| `/` | Add comment |
| `;` | Add plate comment |

### Ghidra scripting (Python)

```python
# Find all calls to a function
func = getFunction("malloc")
refs = getReferencesTo(func.getEntryPoint())
for ref in refs:
    print(f"Called from {ref.getFromAddress()}")

# List all strings
strings = findStrings(None, 4, 1, False, False)
for s in strings:
    print(f"{s.getString()} at {s.getAddress()}")

# Rename function at address
func = getFunctionAt(toAddr("0x401000"))
func.setName("my_decrypt_func")

# Get cross-references
def get_xrefs(addr):
    refs = getReferencesTo(addr)
    return [(r.getFromAddress(), r.getReferenceType()) for r in refs]

# Find XOR patterns (common in crypto/obfuscation)
findBytes(None, "\\x31\\xc0")  # XOR EAX, EAX (zero register)
```

### Ghidra analysis workflow

```
1. Import binary → File → Import File
2. Auto-analyze → accept defaults + check "Decompile"
3. Navigate → Symbol Tree → functions
4. Decompile → select function → F5
5. Rename → L key → give meaningful names
6. Cross-refs → X key → find callers
7. Data types → T key → fix types for readability
8. Export → File → Export Program → C/C++ source
```

---

## radare2 / r2

### Install

```bash
scoop install radare2      # Windows
sudo apt install radare2   # Linux
brew install radare2       # macOS
```

### Essential commands

```bash
r2 myapp.exe               # open binary
r2 -A myapp.exe             # open + auto-analyze (aaaa)
r2 -d myapp.exe             # debug mode

# Inside r2:
aaaa                          # full analysis
afl                           # list all functions
afl~main                      # find main function
iz                            # list all strings
iz~password                   # find password strings
ii                            # list imports
ie                            # list exports
iI                            # binary info (arch, bits, endian)
il                            # loaded libraries
is                            # symbols

# Navigation
s main                        # seek to main
s 0x401000                    # seek to address
pdf                           # print disassembly of function
pdf@main                      # disassemble main
pdc                           # print C-like pseudocode
px 64                         # hexdump 64 bytes
pxw 64                        # hexdump words

# Visual mode
V                             # visual mode
VV                            # visual graph mode (CFG)
VV @ main                     # graph of main
:                             # enter command mode from visual
q                             # quit visual mode

# Debug mode
r2 -d myapp.exe
db main                       # breakpoint at main
db 0x401000                   # breakpoint at address
dc                            # continue
dr                            # show registers
dr rip                        # show specific register
ds                            # step
dso                           # step over
dm                            # memory map

# Search
/x 9090                       # search hex pattern (NOP NOP)
/j "password"                 # search JSON strings
/p "pattern"                  # search pattern

# Write/patch
oo+                           # reopen in write mode
wa nop @ 0x401000             # write NOP
wx 90909090 @ 0x401000       # write hex bytes

# Quit
q                             # quit
```

### Cutter (radare2 GUI)

```bash
cutter myapp.exe              # launch GUI
```

Features: disassembly graph, decompiler view, hex editor, strings browser, imports/exports, debugger.

---

## x64dbg (Windows)

```bash
x64dbg myapp.exe              # open in debugger
```

### Key commands

| Key | Action |
|-----|--------|
| `F2` | Toggle breakpoint |
| `F7` | Step into |
| `F8` | Step over |
| `F9` | Run/continue |
| `Ctrl+F9` | Run to cursor |
| `Ctrl+G` | Go to address |
| `Ctrl+F` | Search in current module |
| `Alt+E` | View modules |
| `Alt+S` | View symbols |
| `Ctrl+L` | Log window |

### x64dbg scripting

```javascript
// x64dbg command log (run in command bar)
bp SetBPX "kernel32.CreateFileW"
bp SetBPX "ntdll.NtCreateFile"
bp log "eax"                  // log EAX on breakpoint hit
bp logcondition "eax != 0"    // conditional log

// Find patterns
findall 0, "E8 ?? ?? ?? ??"   // find all CALL instructions
findall 0, "password"         // find string

// Patch
// Right-click → Assemble → type new instruction
// Or use Edit → Patch

// Plugins
// Scylla — import reconstruction
// SharpOD — anti-anti-debug
// x64dbgpy — Python scripting
```

---

## WinDbg (Windows Kernel Debugging)

### Install

```powershell
# WinDbg Preview (modern, from Microsoft Store)
winget install Microsoft.WinDbg

# Or classic WinDbg via SDK
# Part of Windows SDK / WDK
```

### User-mode debugging

```
# Open crash dump
windbg -z crash.dmp

# Common commands
!analyze -v                   # auto-analyze crash
kb                            # call stack with params
!heap -p -a <addr>            # analyze heap allocation
dt ntdll!_PEB                 # dump PEB structure
lm                            # list modules
bp myapp!main                 # breakpoint
g                             # go (continue)
!sym noisy                    # verbose symbol loading
.reload /f                    # force reload symbols
.sympath srv*https://msdl.microsoft.com/download/symbols  # Microsoft symbol server
```

### Kernel debugging setup

```
# Target machine: enable kernel debugging
bcdedit /debug on
bcdedit /dbgsettings net hostip:192.168.1.100 port:50000 key:1.2.3.4

# Host machine: attach
windbg -k net:port=50000,key:1.2.3.4

# Useful kernel debug commands
!process 0 0                  # list all processes
!thread                        # current thread info
!devobj <addr>                # device object info
!drvobj \Driver\mydriver 7    # driver object + IRP dispatch
dt nt!_EPROCESS               # EPROCESS structure
r @cr3                        # page table base
!pool <addr>                  # pool tag analysis
```

### WinDbg scripting

```
# Break on module load
sxe ld myapp.dll

# Break on exception
sxe -c "" av                   # break on access violation

# Log all calls to a function
bp myapp!myFunc ".printf \"myFunc(%p, %p)\n", @rcx, @rdx; g"

# Stack trace on every breakpoint
bp myapp!myFunc "kb; g"
```

---

## Binary Analysis Patterns

### Quick recon (first 5 minutes)

```bash
# 1. File type and info
file myapp                     # ELF/PE/Mach-O
checksec --file=myapp          # security features

# 2. Strings
strings myapp | head -50       # first 50 strings
strings myapp | rg -i "password|secret|key|token|http|flag"

# 3. Imports (what APIs does it use?)
# Linux:
readelf -d myapp | grep NEEDED  # shared libs
nm -D myapp | grep " U "        # undefined symbols (imports)

# macOS:
otool -L myapp                 # linked libraries
nm -u myapp                    # undefined symbols

# Windows:
objdump -p myapp.exe | rg "DLL Name"  # imports

# 4. Sections
readelf -S myapp               # ELF sections
objdump -h myapp.exe           # PE sections

# 5. Entry point
readelf -h myapp | grep Entry
objdump -f myapp.exe | grep start
```

### checksec — security audit

```bash
checksec --file=myapp

# Output:
# RELRO:     Full RELRO          ← GOT is read-only (good)
# STACK:     Canary found        ← stack protector (good)
# NX:        NX enabled          ← non-executable stack (good)
# PIE:       PIE enabled         ← ASLR (good)
# FORTIFY:   Fortified           ← fortified libc funcs (good)
```

### Patching a binary

```bash
# NOP out a check (replace conditional jump with NOPs)
# In radare2:
oo+                            # write mode
s 0x401234                     # seek to jump
wx 909090909090               # write 6 NOP bytes

# In Ghidra:
# Right-click instruction → Patch Instruction → type "nop"

# In x64dbg:
# Select instruction → right-click → Assemble → "nop"
```

### Extracting embedded resources

```bash
# Extract all strings with context
strings -n 8 myapp > strings.txt

# Find embedded URLs
strings myapp | rg "https?://"

# Find embedded files (PE resources)
# Ghidra: Window → Defined Strings → search for file extensions
# radare2: iz~.png or iz~.dll

# Extract PE resources
# Use ResourceHacker (Windows GUI) or wrestool (Linux):
wrestool -x myapp.exe -o resources/
```

---

## Pre-Ship Security Checklist

```bash
# 1. Security features
checksec --file=myapp

# 2. Leaked symbols
strings myapp | rg -i "\.pdb|debug|test_password"
nm myapp | rg -i "debug|test|password"

# 3. Hardcoded secrets
strings myapp | rg -i "password|secret|token|api_key|aws_"

# 4. Debug artifacts
strings myapp | rg "assert|ASSERT|__FILE__|__LINE__"

# 5. Dependencies
depends myapp.exe              # Windows: DLL deps
ldd myapp                      # Linux: shared lib deps
otool -L myapp                 # macOS: dylib deps
```

---

→ **Binary tools**: [binary-tools.md](binary-tools.md) — UPX, pe-bear, hex editing
→ **Debugging**: [debugging-profiling.md](debugging-profiling.md) — GDB, LLDB, sanitizers
→ **Install**: [tools-install.md](tools-install.md) — install all RE tools
