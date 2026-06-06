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

## Windows Crash Minidumps + LLDB

> **Did you know?** LLDB on Windows can read minidumps. Generate them programmatically or catch crashes automatically.

### Generate minidumps

```cpp
// In your app — write minidump on crash
#include <windows.h>
#include <dbghelp.h>
#pragma comment(lib, "dbghelp.lib")

LONG WINAPI CrashHandler(EXCEPTION_POINTERS* ep) {
    HANDLE file = CreateFileA("crash.dmp", GENERIC_WRITE, 0, NULL, CREATE_ALWAYS, 0, NULL);
    MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(), file,
                      MiniDumpWithFullMemory, ep, NULL, NULL);
    CloseHandle(file);
    return EXCEPTION_EXECUTE_HANDLER;
}

// In main():
SetUnhandledExceptionFilter(CrashHandler);
```

### Analyze minidumps with LLDB (Windows)

```bash
# LLDB can open minidumps directly
lldb -c crash.dmp

# Inside LLDB:
(lldb) bt                                   # backtrace at crash point
(lldb) thread list                          # all threads at crash
(lldb) thread select 1                      # switch thread
(lldb) bt                                   # that thread's backtrace
(lldb) frame variable                       # locals at crash
(lldb) image list                           # loaded modules
(lldb) memory read -count 64 0x7fff0000     # read memory from dump
```

### Analyze with WinDbg (more powerful for Windows dumps)

```powershell
# Install: winget install Microsoft.WinDbg
windbg -z crash.dmp

# Key commands:
!analyze -v                                 # auto-analysis (best first step)
kb                                          # call stack with params
!peb                                        # process environment block
!teb                                        # thread environment block
!heap -p -a <address>                       # analyze heap allocation
lm                                          # loaded modules
dv                                          # local variables
dt ntdll!_PEB                               # dump structure
```

### Windows automatic crash dumps (registry)

```powershell
# Enable LocalDumps (no code changes needed!)
$regPath = "HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps"
New-Item -Path $regPath -Force
Set-ItemProperty -Path $regPath -Name "DumpFolder" -Value "C:\CrashDumps" -Type ExpandString
Set-ItemProperty -Path $regPath -Name "DumpType" -Value 2  # Full dump
Set-ItemProperty -Path $regPath -Name "DumpCount" -Value 5

# Now ANY crash auto-generates a .dmp in C:\CrashDumps
# Disable:
Remove-Item $regPath -Recurse
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
    ZoneScopedN("Physics");         // named zone
    TracyPlot("FrameTime", dt);     // plot values
    FrameMark;                      // mark frame
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
