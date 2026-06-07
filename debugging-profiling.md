# Debugging & Profiling — Pro Edition

> GDB, LLDB, raddebugger, sanitizers, Tracy, RenderDoc. Thread control, reverse debugging, crash dumps.

## GDB Quick Reference

```bash
gdb -tui ./myapp    # TUI mode — source visible
```

| Command | Does |
|---------|------|
| `b main` | Breakpoint at main |
| `b file.cpp:42` | Breakpoint at line |
| `r` | Run |
| `n` | Next (step over) |
| `s` | Step (step into) |
| `c` | Continue |
| `p var` | Print variable |
| `bt` | Backtrace |
| `bt full` | Backtrace + all locals |
| `info locals` | Local variables |
| `watch var` | Break when var changes |
| `layout split` | Source + assembly |
| `thread apply all bt` | All thread backtraces |

---

## 🧵 Thread Control — the game changer

> **Did you know?** You can freeze all threads except one. Debug a single thread while everything else is paused. This is HUGE for multithreaded apps.

### GDB thread isolation

```bash
# List all threads
info threads

# Switch to thread 3
thread 3

# Freeze ALL threads except current
set scheduler-locking on
# Now only YOUR thread steps/runs. Others are frozen in place.

# Step in the frozen-threads world
n                                          # next — only your thread moves
s                                          # step — only your thread moves

# Resume all threads
set scheduler-locking off

# Step mode — lock only during step/next, unlock on continue
set scheduler-locking step
```

### LLDB thread isolation

```bash
# List threads
thread list

# Switch to thread 3
thread select 3

# Freeze all threads except current during stepping
settings set target.process.stop-others true
# Now 'step'/'next' only moves your thread

# Or: freeze specific thread
thread select 2
process continue --stop-others             # continue only this thread

# Back to normal
settings set target.process.stop-others false
```

### Pulse debugging — step through time

```bash
# GDB: reverse debugging (record & replay!)
(gdb) target record-full                   # start recording execution
(gdb) continue                              # run forward normally
# ... hit breakpoint or crash ...
(gdb) reverse-step                          # step BACKWARDS
(gdb) reverse-continue                      # run backwards to previous breakpoint
(gdb) reverse-next                          # next line, backwards
(gdb) info record                           # see recording state

# LLDB: reverse stepping (with process record-replay)
(lldb) process record                       # start recording (lldb 17+)
(lldb) continue
# ... hit breakpoint ...
(lldb) process record step-backward         # go back one instruction
(lldb) process record step-instruction-backward
```

> 💡 **Pro tip**: Record mode captures EVERY instruction. You can replay forward/backward infinitely. Perfect for "how did we get here?" debugging.

---

## Breakpoint Mastery

### Conditional breakpoints

```bash
# GDB
b file.cpp:42 if counter > 100             # break only when condition met
b file.cpp:42 if strcmp(name, "bad") == 0  # break on specific string
b malloc if size > 1000000                  # break on large allocations

# LLDB
b file.cpp:42 -c "counter > 100"
b file.cpp:42 --condition 'strcmp(name, "bad") == 0'
```

### Breakpoint with side effects (log + continue)

```bash
# GDB: print something every time breakpoint hits, then continue
commands 1
  silent
  printf "Hit #%d: x=%d, y=%d\n", ++hit_count, x, y
  continue
end

# GDB: break every Nth hit
b file.cpp:42
ignore 1 99                                # skip first 99 hits, break on 100th

# LLDB: breakpoint command
(lldb) breakpoint command add 1
Enter your debugger command(s).  Type 'DONE' to end.
> expr (void)printf("Hit: x=%d\n", x)
> continue
> DONE
```

### Break on all functions matching pattern

```bash
# GDB: break on every method of a class
rbreak MyClass::.*                          # regex breakpoint

# GDB: break on every virtual function
rbreak .*::vtable.*

# LLDB: regex breakpoint
rb .MyClass                                 # all methods matching MyClass
rb '\[UIViewController'                     # all ObjC methods starting with UIViewController
```

### Hardware watchpoints (no slowdown!)

```bash
# GDB
awatch var                                 # break on read OR write
rwatch var                                 # break on read only
watch var                                  # break on write only

# GDB: watch a memory range
watch *(int*)0x7fff1234                    # watch specific address
watch -l myArray[0] -l myArray[99]         # watch array range

# LLDB
watchpoint set expression -- var
watchpoint set variable myvar
watchpoint set expression -- *(int*)0x7fff1234
```

---

## 💀 Crash Dump Debugging — Ultimate Cross-Platform Reference

> The art of post-mortem debugging: your app crashed, you have a dump file, now find out why.
> Every platform has its own dump format, tools, and symbol requirements. This section covers them all.

### ⚠️ The Golden Rule — You Need THREE Things

> **Without all three, you get useless hex addresses.** Every crash dump debugging session requires:
> 1. **The dump file** — what the crash produced
> 2. **The exact binary** — same build, same commit, same compiler flags
> 3. **The debug symbols** — PDB, dSYM, .debug, or unstripped binary
>
> Miss any one → no function names, no line numbers, no locals. Just `0x7FF61234abcd`.

| Platform | Dump file | Symbols file | Binary | Tool |
|----------|-----------|-------------|--------|------|
| **Windows** | `.dmp` (minidump) | `.pdb` (**MANDATORY**) | `.exe` / `.dll` | LLDB, WinDbg, cdb |
| **Linux** | `core` / `core.<pid>` | `.debug` or unstripped binary | ELF binary | GDB, LLDB |
| **macOS** | `.crash` / `.ips` | `.dSYM/` bundle (**MANDATORY**) | Mach-O binary | LLDB, Console.app |
| **iOS** | `.ips` / `.crash` | `.dSYM/` bundle (**MANDATORY**) | Mach-O binary | LLDB, Xcode Organizer |
| **Android** | `tombstone_<NN>` | unstripped `.so` (**MANDATORY**) | `.so` / `.apk` | `ndk-stack`, `addr2line`, LLDB |

> **Rule**: always ship/save symbols separately. Strip binary for users, keep unstripped + symbols for debugging.
> **Rule**: symbols must match the EXACT build. One recompile = old symbols are garbage.

---

### 🪟 Windows — Minidumps (.dmp) + PDB

#### Files needed

| File | Purpose | Required? |
|------|---------|-----------|
| `crash.dmp` | The minidump itself | ✅ Yes |
| **`myapp.exe`** | **The exact binary that crashed — LLDB needs it as target** | ✅ **MANDATORY** |
| **`myapp.pdb`** | **Debug symbols — function names, line numbers, locals** | ✅ **MANDATORY** |
| `ntdll.pdb`, `kernel32.pdb` | OS symbols (auto-fetched by WinDbg) | ⚠️ For OS frames |

> ⚠️ **Without the `.exe`, LLDB has no target → no PDB loaded → backtrace is raw hex.**
> ⚠️ **Without the `.pdb`, LLDB has target but no symbols → backtrace is module+offset, not function names.**
> ⚠️ **PDB must match the exact build.** Even one recompile invalidates the PDB. Store PDBs in a symbol server or alongside each release.

#### Generate minidumps from code

```cpp
// In your app — write minidump on crash
#include <windows.h>
#include <dbghelp.h>
#pragma comment(lib, "dbghelp.lib")

LONG WINAPI CrashHandler(EXCEPTION_POINTERS* ep) {
    // Include timestamp in filename
    char path[MAX_PATH];
    SYSTEMTIME st;
    GetLocalTime(&st);
    snprintf(path, sizeof(path), "crash_%04d%02d%02d_%02d%02d%02d.dmp",
             st.wYear, st.wMonth, st.wDay, st.wHour, st.wMinute, st.wSecond);

    HANDLE file = CreateFileA(path, GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, 0, NULL);
    MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(), file,
                      MiniDumpWithFullMemory, ep, NULL, NULL);
    CloseHandle(file);
    return EXCEPTION_EXECUTE_HANDLER;
}

// In main():
SetUnhandledExceptionFilter(CrashHandler);
```

Minidump size options:

| Flag | Size | Contains |
|------|------|----------|
| `MiniDumpNormal` | ~100 KB | Threads, stacks, loaded modules |
| `MiniDumpWithFullMemory` | ~RAM size | Everything — full process memory |
| `MiniDumpWithHandleData` | +10-50 MB | All handles (files, mutexes, events) |
| `MiniDumpWithThreadInfo` | +few MB | Thread times, priorities |

> 💡 Use `MiniDumpNormal` for telemetry uploads, `MiniDumpWithFullMemory` for local debugging.

#### Windows auto crash dumps (registry — no code changes!)

```powershell
# Enable LocalDumps — ANY app crash auto-generates .dmp
$regPath = "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps"
New-Item -Path $regPath -Force
Set-ItemProperty -Path $regPath -Name "DumpFolder" -Value "C:\CrashDumps" -Type ExpandString
Set-ItemProperty -Path $regPath -Name "DumpType" -Value 2       # 2 = Full dump, 1 = Mini, 0 = Custom
Set-ItemProperty -Path $regPath -Name "DumpCount" -Value 5       # max dumps to keep

# Per-app override (optional):
New-Item -Path "$regPath\myapp.exe" -Force
Set-ItemProperty -Path "$regPath\myapp.exe" -Name "DumpFolder" -Value "C:\CrashDumps"
Set-ItemProperty -Path "$regPath\myapp.exe" -Name "DumpType" -Value 2

# Now ANY crash → C:\CrashDumps\myapp.exe.<pid>.dmp
# Disable:
Remove-Item $regPath -Recurse
```

#### Analyze with LLDB — the correct way

```bash
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# ⚠️ YOU MUST PROVIDE THE EXECUTABLE (-e flag)!
# Without -e, LLDB has no target → no PDB → useless output
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# THE COMMAND — exe as target, dump as core:
lldb myapp.exe -c crash.dmp

# Or equivalently with long flags:
lldb --file myapp.exe --core crash.dmp

# If PDB not next to exe, tell LLDB where to look:
(lldb) settings set symbols.exec-search-paths "C:\path\to\pdbs"

# Or load PDB explicitly:
(lldb) target symbols add C:\builds\release\myapp.pdb

# Verify symbols loaded:
(lldb) image list myapp.exe                  # should show PDB path
# If it says "No debug info" → PDB missing or mismatched. Fix it.

# Essential commands inside dump session:
(lldb) bt                                    # backtrace at crash point
(lldb) bt all                                # ALL threads at once
(lldb) thread list                           # all threads at crash
(lldb) thread select 1                       # switch thread
(lldb) bt                                    # that thread's backtrace
(lldb) frame variable                        # locals at crash frame
(lldb) frame variable -a                     # locals with addresses
(lldb) frame select 3                        # jump to frame 3
(lldb) frame variable                        # locals at frame 3
(lldb) image list                            # loaded modules + base addresses
(lldb) image lookup -a 0x7FF61234            # address → symbol
(lldb) memory read -count 128 0x7fff0000     # read memory from dump
(lldb) memory read -format x 0x7fff0000      # hex dump
(lldb) disassemble -n crashed_function       # disasm from symbols
(lldb) register read                         # CPU registers at crash
(lldb) expr (int)some_global                 # evaluate expression (if memory present)
```

> 💡 **Common mistake**: running `lldb -c crash.dmp` alone. This opens the dump but with NO target binary. You'll see `ntdll!NtWaitForSingleObject+0x1a` instead of `MyClass::processData() line 42`. **Always use `lldb myapp.exe -c crash.dmp`.**

#### Analyze with WinDbg (deeper Windows analysis)

```powershell
# Install: winget install Microsoft.WinDbg
# Or scoop: scoop install windbg (if available)
windbg -z crash.dmp

# WinDbg auto-finds the exe from dump metadata.
# But you still need PDBs:

# First things to run:
.sympath srv*https://msdl.microsoft.com/download/symbols   # Microsoft symbol server
.sympath+ C:\path\to\your\pdbs                              # add YOUR symbols
.reload /f                                                   # force reload symbols
!analyze -v                                                  # auto-analysis (BEST first step)

# Verify symbols loaded:
lm vm myapp                                                   # check "pdb" column shows your PDB path
# If "exported symbols" → no PDB. Fix path.

# Key WinDbg commands:
kb                                          # call stack with params
!peb                                        # process environment block
!teb                                        # thread environment block
!heap -p -a <address>                       # analyze heap allocation at address
lm                                          # loaded modules
dv                                          # local variables
dt ntdll!_PEB                               # dump PEB structure
dt ntdll!_TEB                               # dump TEB structure
!locks                                      # held locks (deadlock detection)
!handle                                     # open handles
~*kb                                        # all thread call stacks

# Switch threads:
~0s                                         # switch to thread 0
~3s                                         # switch to thread 3
~*kb                                        # all threads backtrace

# Script: dump all thread stacks to file:
.logopen C:\debug_stacks.txt
~*kb
.logclose
```

#### Analyze with cdb (CLI WinDbg — scriptable)

```powershell
# cdb = CLI version of WinDbg, ships with Debugging Tools for Windows
# Install: winget install Microsoft.WindowsSDK (includes cdb)

# One-shot analysis (great for CI/scripts):
cdb -z crash.dmp -y "C:\path\to\pdbs" -c "!analyze -v; q"
#     -z = dump file
#     -y = symbol path (where to find PDBs!)

# Batch: analyze all dumps in a folder
Get-ChildItem C:\CrashDumps\*.dmp | ForEach-Object {
    cdb -z $_.FullName -y "C:\path\to\pdbs" -c "!analyze -v; q" | Out-File "$($_.BaseName).analysis.txt"
}

# Dump all call stacks:
cdb -z crash.dmp -y "C:\path\to\pdbs" -c "~*kb; q"

# Run debugger extension commands:
cdb -z crash.dmp -y "C:\path\to\pdbs" -c "!analyze -v; !heap -p -a <addr>; lm; q"
```

#### PDB management

```cmake
# ALWAYS generate PDB in release builds — without PDB, crash dumps are useless
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")

# Separate PDB from exe (for distribution):
# Build produces: myapp.exe + myapp.pdb
# Ship: myapp.exe (to users)
# Store: myapp.pdb (on symbol server for crash analysis)
```

```powershell
# Symbol server (local) — organize PDBs by hash
# symstore from Windows SDK:
symstore add /r /f C:\build\*.pdb /s C:\SymbolStore /t "MyApp"

# Point WinDbg/LLDB at it:
.sympath+ C:\SymbolStore
```

---

### 🐧 Linux — Core Dumps + GDB/LLDB

#### Files needed

| File | Purpose | Required? |
|------|---------|-----------|
| `core` or `core.<pid>` | The core dump | ✅ Yes |
| **Unstripped ELF binary** | **The exact binary with debug info** | ✅ **MANDATORY** |
| `.debug` files (separate debug) | Split debug symbols | ⚠️ If using split debug |
| Shared `.so` with debug info | Library symbols for stack frames | ⚠️ For lib crashes |

> ⚠️ **Binary must have debug info (`-g` flag at compile).** Stripped binary = no function names, no line numbers.

#### Enable core dumps

```bash
# Check current limit
ulimit -c                                   # 0 = disabled

# Enable for current session
ulimit -c unlimited

# Permanent (systemd — modern distros):
# /etc/systemd/system.conf:
# DefaultLimitCORE=infinity

# Where do cores go? Check:
cat /proc/sys/kernel/core_pattern
# Common: core, core.%p, |/usr/lib/systemd/systemd-coredump %p ...

# If using systemd-coredump:
coredumpctl list                            # all captured cores
coredumpctl info <pid>                      # details
coredumpctl dump <pid> -o core.<pid>        # extract to file
```

#### Debug with GDB

```bash
# THE COMMAND — binary + core:
gdb ./myapp core.12345

# Essential commands:
(gdb) bt                                    # backtrace at crash
(gdb) bt full                               # backtrace + all locals at every frame
(gdb) thread apply all bt                   # ALL threads
(gdb) thread apply all bt full              # ALL threads with locals
(gdb) info threads                          # thread list with state
(gdb) frame 5                               # jump to frame 5
(gdb) info locals                           # locals in current frame
(gdb) info args                             # function arguments
(gdb) list                                  # source code around crash
(gdb) disassemble                           # assembly at crash point
(gdb) info registers                        # CPU registers
(gdb) print my_var                          # print variable
(gdb) print *my_ptr                          # dereference pointer
(gdb) x/64xg 0x7fff0000                    # hex dump memory (64 giant words)

# Verify debug info loaded:
(gdb) info files                             # should show "Reading symbols from..."
(gdb) info sharedlibrary                    # check "Syms Read" column = Yes
# If "No debugging symbols found" → binary is stripped or compiled without -g. Fix it.

# With systemd-coredump (auto-finds binary):
coredumpctl gdb <pid>                       # opens GDB with correct binary + core
```

#### Debug with LLDB

```bash
# THE COMMAND — binary as target, core as dump:
lldb ./myapp -c core.12345

# Or equivalently:
lldb --file ./myapp --core core.12345

# Verify symbols:
(lldb) image list                            # check "myapp" shows debug info loaded
# If no debug info → recompile with -g

# Commands:
(lldb) bt                                    # backtrace
(lldb) bt all                                # all threads
(lldb) frame variable                        # locals
(lldb) frame select 3                        # jump to frame 3
(lldb) frame variable                        # locals at frame 3
(lldb) thread list                           # all threads
(lldb) image list                            # loaded shared libs
(lldb) disassemble -n main                   # disasm function
(lldb) memory read -count 128 0x7fff0000    # read memory
(lldb) register read                         # registers at crash
```

#### Split debug symbols (keep small binaries, debug with full symbols)

```bash
# Extract debug info to separate file
objcopy --only-keep-debug myapp myapp.debug
strip myapp                                  # remove debug info from binary
objcopy --add-gnu-debuglink=myapp.debug myapp  # link debug file to stripped binary

# Now GDB auto-loads myapp.debug when debugging myapp
# Ship myapp to users, keep myapp.debug for crash analysis
# Put myapp.debug next to myapp or in /usr/lib/debug/
```

#### Core dump with GDB (generate from running process)

```bash
# Attach and generate core without killing process
gdb -p <pid>
(gdb) generate-core-file
(gdb) detach
(gdb) quit

# Or: gcore (ships with GDB)
gcore -o myapp.core <pid>                   # generate core, process keeps running
```

---

### 🍎 macOS — Crash Reports + LLDB

#### Files needed

| File | Purpose | Required? |
|------|---------|-----------|
| `.crash` or `.ips` | Crash report (text or binary plist) | ✅ Yes |
| **`.dSYM/` bundle** | **Debug symbols (DWARF)** | ✅ **MANDATORY** |
| **Mach-O binary** | **The exact binary that crashed** | ✅ **MANDATORY** |

> **dSYM must match exact build UUID.** Check with: `dwarfdump --uuid MyApp.app/MyApp` and `dwarfdump --uuid MyApp.app.dSYM`
> ⚠️ **No dSYM = no function names.** The .ips file only has hex addresses. dSYM is what translates them.

#### Generate dSYM (Xcode / CLI)

```bash
# Xcode project — enable in Build Settings:
# Debug Information Format = "DWARF with dSYM File" (DEBUG_INFORMATION_FORMAT=dwarf-with-dsym)

# CLI build:
xcodebuild -project MyApp.xcodeproj -scheme MyApp \
  -configuration Release \
  DEBUG_INFORMATION_FORMAT=dwarf-with-dsym \
  build

# dSYM appears in build products dir
# Archive it! You'll need it for crash analysis later
```

```bash
# Verify dSYM matches binary:
dwarfdump --uuid MyApp.app/MyApp             # binary UUID
dwarfdump --uuid MyApp.app.dSYM             # dSYM UUID — must match!
# If UUIDs differ → dSYM is from wrong build. Useless.
```

#### Find crash reports

```bash
# User crash reports:
ls ~/Library/Logs/DiagnosticReports/

# System crash reports:
ls /Library/Logs/DiagnosticReports/

# Console.app → Crash Reports (GUI)

# Copy .crash/.ips file for analysis
cp ~/Library/Logs/DiagnosticReports/MyApp*.ips ~/Desktop/
```

#### Analyze with LLDB

```bash
# Open crash report with LLDB:
lldb ./MyApp -c MyApp-2024-01-15.ips

# If LLDB can't find dSYM, point it explicitly:
(lldb) target symbols add /path/to/MyApp.app.dSYM/Contents/Resources/DWARF/MyApp

# Verify:
(lldb) image list MyApp                      # should show dSYM path
# If "No debug info" → dSYM missing or UUID mismatch

# Commands:
(lldb) bt                                    # backtrace
(lldb) bt all                                # all threads
(lldb) frame variable                        # locals
(lldb) frame select 5                        # jump to frame
(lldb) frame variable                        # locals at that frame
(lldb) image list                            # loaded frameworks
(lldb) image lookup -a 0x1234               # address → symbol

# Manual symbolication from .ips crash report (no LLDB needed):
# .ips files have raw addresses. Use atos to symbolicate:
atos -arch arm64 -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l 0x100000000 0x100004321
# -l = load address from crash report, 0x100004321 = crash address

# Batch symbolicate entire crash report:
xcrun symbolicatecrash MyApp.ips MyApp.app.dSYM > symbolicated.crash
```

#### Generate crash dump programmatically

```cpp
// macOS — generate Mach-O core dump
#include <sys/resource.h>
#include <signal.h>

// Enable core dumps:
struct rlimit rl;
rl.rlim_cur = RLIM_INFINITY;
rl.rlim_max = RLIM_INFINITY;
setrlimit(RLIMIT_CORE, &rl);

// Or use task_dump:
// #include <mach/mach.h>
// task_t task = mach_task_self();
// Use vm_read_overwrite to dump memory regions
```

---

### 📱 iOS — Crash Logs + LLDB

#### Files needed

| File | Purpose | Required? |
|------|---------|-----------|
| `.ips` / `.crash` | Device crash log | ✅ Yes |
| **`.dSYM/` bundle** | **Debug symbols from archived build** | ✅ **MANDATORY** |
| **`.app` binary** | **The Mach-O binary** | ✅ **MANDATORY** |

> ⚠️ **No dSYM = raw hex addresses.** iOS crash logs are useless without the exact dSYM from the build that produced the crash.

#### Get crash logs from device

```bash
# Xcode 15+ (devicectl):
xcrun devicectl device info logs --device <device-id>

# Older method — Xcode Organizer:
# Xcode → Window → Devices and Simulators → select device → View Device Logs

# iTunes sync stores logs here:
# macOS: ~/Library/Logs/CrashReporter/MobileDevice/<device-name>/
# Windows: %APPDATA%\Apple Computer\Logs\CrashReporter\MobileDevice\

# Copy .ips files for offline analysis
```

#### Symbolicate iOS crash logs

```bash
# Need: .ips crash log + .dSYM + .app binary — ALL THREE

# Method 1: xcrun symbolicatecrash
export DEVELOPER_DIR="/Applications/Xcode.app/Contents/Developer"
xcrun symbolicatecrash crash.ips MyApp.app.dSYM > symbolicated.crash

# Method 2: atos (manual, one address at a time)
# From crash report: Binary Images section → find load address
# Then:
atos -arch arm64 -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp \
  -l 0x100000000 0x100004321

# Method 3: Xcode Organizer (automatic)
# Xcode → Window → Organizer → Crashes tab
# Upload dSYM → Xcode symbolicates automatically
```

#### LLDB remote debug iOS crash

```bash
# Reproduce crash with debugger attached:
xcrun devicectl device process launch --device <device-id> \
  --start-stopped com.example.myapp

# Connect LLDB:
lldb
(lldb) platform select remote-ios
(lldb) platform connect connect://<device-ip>:1234
(lldb) process connect connect://<device-ip>:1234
(lldb) continue                              # run until crash
# When crash happens:
(lldb) bt                                    # backtrace at crash
(lldb) bt all                                # all threads
(lldb) frame variable                        # locals
```

---

### 🤖 Android — Tombstones + NDK Stack

#### Files needed

| File | Purpose | Required? |
|------|---------|-----------|
| `tombstone_<NN>` | Native crash log (text) | ✅ Yes |
| **Unstripped `.so` files** | **Debug symbols for native libs** | ✅ **MANDATORY** |
| `app/build/intermediates/` | Intermediate build artifacts | ⚠️ Sometimes needed |

> ⚠️ **Always keep unstripped .so files from release builds.** Gradle strips them for the APK but keeps originals in `obj/local/`.
> ⚠️ **No unstripped .so = ndk-stack shows `#00 pc 0x12345 libmyapp.so` with no function name.**

#### Get tombstones from device

```bash
# Tombstones are in /data/tombstones/ (need root or adb root)
adb root
adb shell ls /data/tombstones/
adb pull /data/tombstones/tombstone_00 .

# Or from adb logcat (crash summary):
adb logcat -d | grep -A 50 "tombstone"

# Android 11+ — bugreport includes tombstones:
adb bugreport > bugreport.zip
# Extract: bugreport/tombstones/
```

#### Symbolicate with ndk-stack

```bash
# ndk-stack reads logcat and replaces addresses with symbols
# Needs path to unstripped .so files — THIS IS THE KEY:
adb logcat | ndk-stack -sym app/build/intermediates/cmake/release/obj/arm64-v8a

# Or from saved tombstone:
ndk-stack -sym path/to/unstripped/so/ -dump tombstone_00

# With NDK installed:
$ANDROID_NDK/ndk-stack -sym obj/local/arm64-v8a < tombstone_00

# If ndk-stack shows raw addresses → wrong symbol path or stripped .so
# Fix: point -sym to directory containing UNSTRIPPED .so files
```

#### Symbolicate with addr2line

```bash
# For individual addresses:
$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-addr2line \
  -e libmyapp.so -f -C 0x12345

# -e = the UNSTRIPPED .so (not the one in the APK!)
# -f = function names
# -C = demangle C++ names
# If output is "??" → .so is stripped or wrong file
```

#### Debug with LLDB (Android NDK)

```bash
# Start lldb-server on device:
adb push $ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/lib/clang/*/lib/linux/aarch64/lldb-server \
  /data/local/tmp/
adb shell chmod 755 /data/local/tmp/lldb-server
adb shell "/data/local/tmp/lldb-server platform --listen '*:1234' --server" &

# Forward port:
adb forward tcp:1234 tcp:1234

# Connect from host — with UNSTRIPPED .so as target:
lldb
(lldb) platform select remote-android
(lldb) platform connect connect://localhost:1234
(lldb) target create /path/to/UNSTRIPPED/libmyapp.so    # ← unstripped is critical!
(lldb) process connect connect://localhost:1234
(lldb) bt                                                # backtrace
(lldb) frame variable                                    # locals
```

#### ProGuard / R8 mapping (Java/Kotlin crashes)

```bash
# If using R8/ProGuard, need mapping.txt to deobfuscate Java stack traces
# Found in: app/build/outputs/mapping/release/mapping.txt

# Deobfuscate:
$ANDROID_HOME/tools/proguard/bin/retrace.sh mapping.txt obfuscated_trace.txt
```

---

### 🔧 Cross-Platform: Breakpad & Crashpad

> Google's open-source crash reporting. Used by Chrome, Electron, VS Code, Discord, and many games.

#### Crashpad (modern, recommended)

```cpp
// Integrate Crashpad (successor to Breakpad)
#include "client/crashpad_client.h"
#include "client/settings.h"
#include "client/crash_report_database.h"

bool InitializeCrashpad() {
    base::FilePath db_path("crashpad_db");
    auto database = crashpad::CrashReportDatabase::Initialize(db_path);

    crashpad::CrashpadClient client;
    std::map<std::string, std::string> annotations;
    annotations["version"] = "1.0.0";
    annotations["build"] = "release";

    std::vector<std::string> arguments;
    return client.StartHandler(
        base::FilePath("crashpad_handler"),  // handler binary
        db_path,
        base::FilePath(),                     // metrics dir
        "https://your-crash-server.com/upload",
        annotations,
        arguments,
        true,                                 // restart on crash
        true                                  // asynchronous start
    );
}

// In main():
InitializeCrashpad();
// Now all crashes auto-upload to your server
```

#### Breakpad (legacy but still used)

```cpp
#include "client/windows/handler/exception_handler.h"

// Windows:
google_breakpad::ExceptionHandler eh(
    L"crashdumps",                            // dump directory
    NULL,                                     // filter callback
    NULL,                                     // minidump callback
    NULL,                                     // callback context
    google_breakpad::ExceptionHandler::HANDLER_ALL
);

// On crash: crashdumps/<guid>.dmp is generated
// Upload with symupload tool:
// symupload myapp.pdb https://symbols.example.com/
```

#### Minidump analysis (Breakpad/Crashpad format)

```bash
# dump_syms — extract symbols from binary into Breakpad format
dump_syms myapp > myapp.sym

# minidump_stackwalk — analyze dump with symbols
minidump_stackwalk crash.dmp myapp.sym > analysis.txt

# Or with LLDB (if minidump is standard Windows format):
lldb myapp.exe -c crash.dmp                  # ← still need exe as target!
```

---

### 📊 Symbol Server Setup

> Centralized symbol storage — upload once, debug anywhere.

#### Microsoft Symbol Server format (Symsrv)

```
srv*C:\LocalCache*https://msdl.microsoft.com/download/symbols
```

WinDbg and LLDB both understand this format. Local cache avoids re-downloading.

#### Local symbol server

```bash
# Organize symbols by GUID + filename (Symsrv format):
# C:\SymbolStore\myapp.pdb\ABCDEF1234567890\myapp.pdb

# Add to WinDbg search path:
.sympath srv*C:\SymbolStore
.sympath+ srv*https://msdl.microsoft.com/download/symbols
.reload /f

# Add to LLDB:
(lldb) settings set symbols.exec-search-paths /path/to/symbols
```

#### CI/CD: archive symbols on every release

```bash
# In your release pipeline:
# 1. Build with symbols
cmake --build build --config RelWithDebInfo

# 2. Archive PDB/dSYM/debug files — WITHOUT THESE, CRASH DUMPS ARE USELESS
# Windows:
mkdir symbols/release-1.2.3/
cp build/Release/*.pdb symbols/release-1.2.3/
cp build/Release/*.exe symbols/release-1.2.3/   # need exe too for LLDB!

# macOS:
mkdir symbols/release-1.2.3/
cp -R build/Release/*.dSYM symbols/release-1.2.3/
cp build/Release/MyApp symbols/release-1.2.3/   # need binary too!

# Linux:
mkdir symbols/release-1.2.3/
cp build/myapp symbols/release-1.2.3/            # unstripped binary = both binary + symbols
# Or if using split debug:
cp build/myapp.debug symbols/release-1.2.3/
cp build/myapp.stripped symbols/release-1.2.3/

# 3. Upload to artifact storage (S3, Azure Artifacts, GitHub Releases)
# 4. Point debuggers at it
```

---

### 🔑 Debug Symbols Quick Reference

| Build flag | Platform | Produces | Notes |
|-----------|----------|----------|-------|
| `/Zi` | MSVC | `.pdb` file | **Add to Release builds!** No PDB = no crash analysis |
| `/DEBUG` (linker) | MSVC | Embeds PDB path in exe | Required for crash dumps |
| `-g` | GCC/Clang | DWARF in binary | Linux/macOS — **mandatory for core dump analysis** |
| `-g -gsplit-dwarf` | GCC | `.dwo` split files | Faster linking, separate debug |
| `DEBUG_INFORMATION_FORMAT=dwarf-with-dsym` | Xcode | `.dSYM/` bundle | **Must archive!** No dSYM = no symbolication |

```cmake
# Cross-platform: always generate symbols in Release
if(MSVC)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
else()
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -g")
endif()
```

---

## Sanitizers (zero-effort bug catching)

```bash
# AddressSanitizer + UndefinedBehavior
cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer" -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"

# ThreadSanitizer (separate build — conflicts with ASAN)
cmake -B build-tsan -DCMAKE_CXX_FLAGS="-fsanitize=thread" -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"
```

| Sanitizer | Catches | Flag |
|-----------|---------|------|
| **AddressSanitizer** | use-after-free, buffer overflow, memory leaks | `-fsanitize=address` |
| **UndefinedBehavior** | integer overflow, null deref, alignment | `-fsanitize=undefined` |
| **ThreadSanitizer** | data races, deadlocks | `-fsanitize=thread` |
| **MemorySanitizer** | uninitialized reads (clang only) | `-fsanitize=memory` |
| **LeakSanitizer** | memory leaks (standalone) | `-fsanitize=leak` |

> 💡 Run sanitizers in CI. They catch bugs that pass all unit tests.

---

## Valgrind — memory debugging (Linux)

```bash
# Install: sudo apt install valgrind

# Memory leak check
valgrind --leak-check=full --show-leak-kinds=all ./myapp

# Detailed leak report with stack traces
valgrind --leak-check=full --track-origins=yes --read-var-info=yes ./myapp

# Check for uninitialized memory reads
valgrind --track-origins=yes ./myapp

# Generate suppressions for known issues
valgrind --gen-suppressions=all --log-file=valgrind.log ./myapp

# Callgrind — profiling
valgrind --tool=callgrind ./myapp
kcachegrind callgrind.out.*                  # GUI visualization

# Helgrind — thread error detector
valgrind --tool=helgrind ./myapp
```

---

## Tracy — Runtime Profiler

```cpp
#include <Tracy.hpp>

void heavy() {
    ZoneScoped;                     // auto-named zone
    ZoneScopedN("Physics");         # named zone
    TracyPlot("FrameTime", dt);     # plot values
    FrameMark;                      # mark frame
}
```

```bash
# FetchContent in CMake:
# FetchContent_Declare(tracy GIT_REPOSITORY https://github.com/wolfpld/tracy.git GIT_TAG v0.11.1)
# FetchContent_MakeAvailable(tracy)
# target_link_libraries(myapp PRIVATE Tracy::TracyClient)

tracy-profiler &                   # launch GUI
./build/dev/myapp                  # app connects automatically
```

> 💡 Macros are nanoseconds overhead. Keep Tracy on in dev builds always.

---

## Kernel Debugging

> Full reference: [reverse-engineering.md](reverse-engineering.md) — WinDbg section.

### Windows (WinDbg network debug)

```powershell
# Enable on target machine (admin)
bcdedit /debug on
bcdedit /dbgsettings net hostip:192.168.1.100 port:50000 key:1.2.3.4

# Attach from host
windbg -k net:port=50000,key:1.2.3.4

# Essential kernel debug commands:
!analyze -v                          # auto-analyze crash
kb                                   # call stack with params
!process 0 0                         # list all processes
!thread                               # current thread info
lm                                   # list loaded modules (kernel drivers)
dt nt!_EPROCESS                       # dump EPROCESS structure
!devobj <addr>                       # device object info
!drvobj \Driver\mydriver 7           # driver object + IRP dispatch table
bp nt!NtCreateFile                   # break on kernel function
.reload /f                            # force reload symbols
.sympath srv*https://msdl.microsoft.com/download/symbols  # Microsoft symbols
```

### Linux (GDB + QEMU)

```bash
# Target: boot kernel with kgdb
qemu-system-x86_64 -kernel bzImage -append "kgdbwait kgdboc=ttyS0,1234" -serial stdio

# Host: connect
gdb vmlinux
(gdb) target remote :1234
(gdb) c                                # continue kernel execution

# Break on kernel function
(gdb) b sys_open
(gdb) b schedule                       # break on context switch

# Kernel debugging tips:
(gdb) p init_task                       # first process
(gdb) p current                         # current process
(gdb) info threads                      # all kernel threads
```

### macOS (LLDB kernel debug)

```bash
# Install KDK (Kernel Debug Kit) from developer.apple.com
# Then:
sudo kdk-install                         # install kernel debug kit
lldb -n kernel_task                      # attach to local kernel

# Or remote kernel debug:
# Target: nvram boot-args="debug=0x14e"
# Reboot, then from host:
lldb
(lldb) kdp-remote <target-ip>
```

---

## Remote Debugging

### GDB remote (Linux → Linux)

```bash
# On target machine:
gdbserver :1234 ./myapp

# On host machine:
gdb ./myapp
(gdb) target remote 192.168.1.100:1234
(gdb) b main
(gdb) c
```

### LLDB remote (macOS → iOS device)

```bash
# On device (via Xcode or devicectl):
xcrun devicectl device process launch --device <id> --start-stopped com.example.app

# On host:
lldb
(lldb) platform select remote-ios
(lldb) platform connect connect://<device-ip>:1234
(lldb) process connect connect://<device-ip>:1234
(lldb) b main
(lldb) c
```

---

## Perfetto (System Tracing)

```bash
# Install: sudo apt install perfetto / download from perfetto.dev
perfetto --background                      # start service

# Capture system-wide trace (Linux)
perfetto -c config.textproto -o trace.perfetto-trace

# Open in browser: https://ui.perfetto.dev → drag trace file
```

## RenderDoc (Graphics)

```bash
renderdoc                                  # GUI — capture frames, inspect draw calls/shaders
# Or from CLI:
qrenderdoc --capture-frame ./myapp
```

---

## Printf Debugging (when desperate)

```cpp
// Better than std::cout
#include <fmt/core.h>
fmt::print(stderr, "[{}:{}] var = {}\n", __FILE__, __LINE__, var);

// With source location (C++20)
#include <source_location>
void debug_log(std::string_view msg, std::source_location loc = std::source_location::current()) {
    fmt::print(stderr, "[{}:{}] {}\n", loc.file_name(), loc.line(), msg);
}

// Compile-time conditional debug (zero cost in release)
#ifndef NDEBUG
  #define DBG(...) fmt::print(stderr, "[{}:{}] " __VA_ARGS__ "\n", __FILE__, __LINE__)
#else
  #define DBG(...) ((void)0)
#endif
```

---

**Related**: [Mobile debug](mobile.md) — LLDB remote, ADB NDK, Frida
**Related**: [Kernel debug](reverse-engineering.md) — WinDbg deep dive, kgdb
**Related**: [Network debug](network-debugging.md) — tcpdump, Wireshark, DNS, TLS
**Related**: [Install](tools-install.md) — install all debuggers
**Related**: [File tracking](README.md#11-file-tracking) — trace what files a command produces
