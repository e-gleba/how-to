# Debugging & Profiling

## GDB

```bash
# Install
scoop install gdb

# Start with TUI
# '-tui' shows source code alongside commands
gdb -tui ./myapp

# Useful commands inside GDB:
# b main          — breakpoint at main
# b file.cpp:42   — breakpoint at line
# r               — run
# n               — next (step over)
# s               — step (step into)
# c               — continue
# p var           — print variable
# bt              — backtrace
# info locals     — all local variables
# watch var       — break on variable change
# layout asm      — switch to assembly view
# layout split    — source + assembly side by side
```

### GDB on Windows with MSYS2

```bash
# GDB from MSYS2 understands POSIX paths
# If using native Windows GCC, use gdb from MSYS2
# Or use raddebugger for native Windows debugging
```

### GDB with core dumps

```bash
# Enable core dumps (Windows)
# GDB can load minidumps

gdb ./myapp core.dump
bt full     # full backtrace with locals
thread apply all bt  # all threads
```

## raddebugger

Ryan Fleury's debugger. Native Windows, extremely fast, clean UI.

```bash
# Installed via scoop: raddebugger-alpha
raddebugger ./myapp.exe
```

Features:
- Source-level debugging
- Breakpoints, watchpoints
- Register view
- Memory view
- Disassembly
- No symbol server needed (reads PDB/DWARF)

## x64dbg

Native Windows debugger. Best for PE inspection, reverse engineering, Windows-specific crashes.

```bash
# Installed via scoop (x64dbg)
# Launch from Start Menu or:
x96dbg  # 32-bit
x64dbg  # 64-bit
```

Use cases:
- Debugging release builds without PDBs
- Understanding third-party DLLs
- Finding crash addresses in stripped binaries
- Patch analysis

## Tracy — Profiler

### Setup (CMake)

```cmake
# Clone Tracy into your project, or use FetchContent
FetchContent_Declare(
    tracy
    GIT_REPOSITORY https://github.com/wolfpld/tracy.git
    GIT_TAG v0.11.1
)
FetchContent_MakeAvailable(tracy)

target_link_libraries(myapp PRIVATE Tracy::TracyClient)
```

### Instrumentation

```cpp
#include <Tracy.hpp>

void heavy_loop() {
    ZoneScoped;  // appears in Tracy profiler with function name
    for (int i = 0; i < 1000000; ++i) {
        ZoneScopedN("InnerLoop");
        // ...
    }
}

void named_zone() {
    ZoneScopedN("PhysicsUpdate");
    // ...
}

// Plot values over time
void frame(float dt) {
    TracyPlot("FrameTime", dt * 1000.0f);
}

// Memory tracking
void* ptr = TracyAlloc(ptr, size);
TracyFree(ptr);

// Mutex profiling
std::mutex mtx;
{
    ZoneScopedN("LockContention");
    std::lock_guard lock(mtx);   // Tracy tracks wait time
}

// Mark frames (for frame profiler)
void render() {
    FrameMark;
}
```

### Connect

```bash
# Launch Tracy profiler GUI (installed via scoop: tracy)
tracy-profiler

# Run your app — it will appear in the GUI automatically
./build/dev/myapp
```

### Conditional profiling

```cmake
# CMakeLists.txt
option(TRACY_ENABLE "Enable Tracy profiling" ON)

if(TRACY_ENABLE)
    target_compile_definitions(myapp PRIVATE TRACY_ENABLE)
    target_link_libraries(myapp PRIVATE Tracy::TracyClient)
endif()
```

```cpp
// Macros become no-ops when TRACY_ENABLE is not defined
ZoneScoped;  // zero overhead when disabled
```

## Perfetto

Google's tracing framework. Web-based UI, supports Android tracing natively.

```bash
# Installed via scoop: perfetto
# Start tracing service
perfetto --background

# Record a trace
perfetto -c - --txt <<EOF
buffers: { size_kb: 65536 }
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
    }
  }
}
duration_ms: 10000
EOF

# Open https://ui.perfetto.dev in browser to view trace
```

## RenderDoc — Graphics Debugging

```bash
# Installed via scoop: renderdoc
# Launch from Start Menu or:
renderdoc
```

Use for:
- Capturing Vulkan/OpenGL/D3D frames
- Inspecting draw calls, shaders, textures
- Performance counters per draw call
- Shader debugging (step through shader code)

### Programmatic capture

```cpp
#include <renderdoc_app.h>

// Inject RenderDoc into your app
// Then trigger captures from code or the in-app overlay
```

## Other useful tools

```bash
# strace for Windows (see system calls)
# Use Sysinternals Process Monitor (procmon) — installed via scoop
procmon

# Filter by process name to see file/registry/network activity

# Dependency Walker
depends ./myapp.exe

# PE Bear — modern PE viewer
pe-bear ./myapp.exe
```

## Quick debugging workflow

1. **Assert early**: `assert()` + custom `ASSERT` macros with messages
2. **Sanitizers first**: ASan + UBSan catch 80% of bugs at zero effort
3. **GDB for logic**: breakpoints, watches, step through
4. **raddebugger for Windows-native**: faster startup, clean UI
5. **x64dbg for release builds**: when you don't have debug symbols
6. **Tracy for performance**: always-on instrumentation in dev builds
