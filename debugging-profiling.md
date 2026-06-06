# Debugging & Profiling

> GDB, LLDB, raddebugger, sanitizers, Tracy, RenderDoc.

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

### GDB scripting

```bash
# Break on all functions matching pattern
rbreak .*MyClass.*

# Conditional breakpoint
b file.cpp:42 if counter > 100

# Reverse debugging (record mode)
target record-full
# ... run, then:
reverse-step
reverse-continue

# Print on every breakpoint
commands 1
  silent
  printf "hit bp1: x=%d\n", x
  continue
end
```

## LLDB Quick Reference

> macOS/iOS/Android NDK — [mobile.md](mobile.md) for remote debugging.

```bash
lldb ./myapp
```

| Command | Does |
|---------|------|
| `b main` | Breakpoint at main |
| `b file.cpp:42` | Breakpoint at line |
| `rb .` | Regex breakpoint (all matches) |
| `run` | Run |
| `next` | Step over |
| `step` | Step into |
| `continue` | Continue |
| `print var` | Print variable |
| `po obj` | Print object (calls description) |
| `bt` | Backtrace |
| `frame variable` | All locals |
| `thread list` | All threads |
| `thread backtrace all` | All thread backtraces |
| `expr myFunc(42)` | Call function |
| `memory read -count 64 0x7fff0000` | Read memory |
| `watchpoint set variable myvar` | Break on change |
| `image list` | Loaded libraries |
| `image lookup -n myFunc` | Find function address |

### LLDB tips

```bash
# Print view hierarchy (iOS)
(lldb) expr (void)[[[UIApplication sharedApplication] keyWindow] recursiveDescription]

# Force UI update
(lldb) expr (void)[CATransaction flush]

# Python scripting
(lldb) script import lldb
(lldb) script frame = lldb.debugger.GetSelectedTarget().GetProcess().GetSelectedThread().GetSelectedFrame()
```

## raddebugger (Windows native)

```bash
raddebugger ./myapp.exe
```

Fast startup, clean UI. Reads PDB/DWARF. No symbol server configuration.

## x64dbg (release builds)

```bash
x64dbg ./myapp.exe
```

Use when: stripped binaries, third-party DLLs, crash dumps without symbols.
→ **Advanced**: [reverse-engineering.md](reverse-engineering.md) — full x64dbg scripting

## Sanitizers (zero-effort bug catching)

```cmake
# In CMake configure:
-DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
-DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
```

| Sanitizer | Catches | Flag |
|-----------|---------|------|
| **AddressSanitizer** | use-after-free, buffer overflow, memory leaks | `-fsanitize=address` |
| **UndefinedBehavior** | integer overflow, null deref, alignment | `-fsanitize=undefined` |
| **ThreadSanitizer** | data races, deadlocks | `-fsanitize=thread` |
| **MemorySanitizer** | uninitialized reads | `-fsanitize=memory` |

> 💡 Run sanitizers in CI. They catch bugs that pass all unit tests.

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

```cmake
FetchContent_Declare(tracy GIT_REPOSITORY https://github.com/wolfpld/tracy.git GIT_TAG v0.11.1)
FetchContent_MakeAvailable(tracy)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
```

```bash
tracy-profiler &     # launch GUI
./build/dev/myapp    # app appears automatically
```

> 💡 Macros are nanoseconds. Keep Tracy on in dev builds always.

## Kernel Debugging

> Full reference: [reverse-engineering.md](reverse-engineering.md) — WinDbg section.

### Windows (WinDbg)

```powershell
# Enable on target
bcdedit /debug on
bcdedit /dbgsettings net hostip:192.168.1.100 port:50000 key:1.2.3.4

# Attach from host
windbg -k net:port=50000,key:1.2.3.4

# Key commands
!analyze -v         # auto-analyze crash
kb                  # call stack
!process 0 0        # all processes
lm                  # list modules
.sympath srv*https://msdl.microsoft.com/download/symbols
```

### Linux (GDB + QEMU)

```bash
# Target: boot with kgdb
qemu-system-x86_64 -kernel bzImage -append "kgdbwait kgdboc=ttyS0,1234" -serial stdio

# Host: connect
gdb vmlinux
(gdb) target remote :1234
(gdb) c
```

### macOS (LLDB)

```bash
# KDK required — download from developer.apple.com
sudo kdk-install   # install kernel debug kit

# Attach to local kernel
lldb -n kernel_task
```

## Perfetto (System Tracing)

```bash
perfetto --background    # start service
# Capture trace, then open https://ui.perfetto.dev
```

## RenderDoc (Graphics)

```bash
renderdoc    # GUI — capture frames, inspect draw calls/shaders
```

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
```

---

→ **Mobile debug**: [mobile.md](mobile.md) — LLDB remote, ADB NDK, Frida
→ **Kernel debug**: [reverse-engineering.md](reverse-engineering.md) — WinDbg, kgdb
→ **Install**: [tools-install.md](tools-install.md) — install all debuggers
→ **File tracking**: [README.md](README.md) section 10 — trace what files a command produces
