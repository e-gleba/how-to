# Debugging & Profiling

## GDB quick reference

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

## Sanitizers (zero-effort bug catching)

```cmake
# In CMake preset cacheVariables:
"CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer",
"CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined"
```

Catches: use-after-free, buffer overflow, memory leaks, integer overflow, null deref.

> 💡 **Tip:** Run sanitizers in CI. They catch bugs that pass all unit tests.

## Tracy — runtime profiler

```cpp
#include <Tracy.hpp>

void heavy() {
    ZoneScoped;                     // auto-named zone
    ZoneScopedN("Physics");          // named zone
    TracyPlot("FrameTime", dt);      // plot values
    FrameMark;                       // mark frame
}
```

```cmake
# CMakeLists.txt
FetchContent_Declare(tracy GIT_REPOSITORY https://github.com/wolfpld/tracy.git GIT_TAG v0.11.1)
FetchContent_MakeAvailable(tracy)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
```

```bash
tracy-profiler &     # launch GUI
./build/dev/myapp    # app appears automatically
```

> 💡 Macros are nanoseconds. Keep Tracy on in dev builds always.

## Perfetto (system tracing)

```bash
perfetto --background    # start service
# Capture trace, then open https://ui.perfetto.dev
```

## RenderDoc (graphics)

```bash
renderdoc    # GUI — capture frames, inspect draw calls/shaders
```

## Printf debugging (when desperate)

```cpp
// Better than std::cout for debug prints
#include <fmt/core.h>
fmt::print(stderr, "[{}:{}] var = {}\n", __FILE__, __LINE__, var);
```
