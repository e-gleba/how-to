# Benchmarking

## hyperfine

```bash
# Install
scoop install hyperfine

# Basic benchmark
hyperfine "./myapp"

# Warmup + min runs
hyperfine --warmup 5 --min-runs 20 "./myapp"

# Compare two versions
hyperfine "./build/old/myapp" "./build/new/myapp"

# Parameter sweep
hyperfine --parameter-scan buffer 1024 65536 --style exponential "./myapp --buffer-size {buffer}"

# Prepare (build) before each run
hyperfine --prepare "cmake --build build" "./build/myapp --bench"

# Export results
hyperfine --export-json results.json "./myapp"
hyperfine --export-markdown results.md "./myapp"
hyperfine --export-csv results.csv "./myapp"

# Compare with baseline (statistical)
hyperfine "./baseline" "./candidate"
# Output shows: which is faster, by how much, confidence
```

### Output interpretation

```
Benchmark 1: ./myapp
  Time (mean ± σ):      45.2 ms ±   2.1 ms    [User: 32.1 ms, System: 8.3 ms]
  Range (min … max):    42.1 ms …  52.3 ms    100 runs

  ⚠️ Warning: outliers detected (5)

# Key stats:
# mean ± σ  — main result, σ = standard deviation
# Range     — best vs worst case
# Outliers  — potential interference (close other apps)
```

## Google Benchmark

```cpp
#include <benchmark/benchmark.h>

static void BM_StringCopy(benchmark::State& state) {
    std::string x = "hello world this is a test string";
    for (auto _ : state) {
        std::string copy(x);
        benchmark::DoNotOptimize(copy);
    }
}
BENCHMARK(BM_StringCopy);

static void BM_VectorPush(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        v.reserve(state.range(0));
        for (int i = 0; i < state.range(0); ++i)
            v.push_back(i);
        benchmark::DoNotOptimize(v);
    }
}
BENCHMARK(BM_VectorPush)->Range(8, 8<<10);

BENCHMARK_MAIN();
```

### CMake integration

```cmake
FetchContent_Declare(
    benchmark
    GIT_REPOSITORY https://github.com/google/benchmark.git
    GIT_TAG v1.9.0
)
FetchContent_MakeAvailable(benchmark)

target_link_libraries(my_benchmarks PRIVATE benchmark::benchmark)
```

## Quick micro-benchmarks with chrono

```cpp
#include <chrono>
#include <iostream>

template<typename F>
auto measure(F&& f, int iterations = 1000000) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        f();
    }
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / iterations;
}

// Usage:
auto ns_per_call = measure([] { some_operation(); });
std::cout << ns_per_call << " ns/call\n";
```

## Benchmarking compilation

```bash
# Time a full rebuild
hyperfine --prepare "rm -rf build && cmake -B build -G Ninja" "cmake --build build" --warmup 1

# Time incremental rebuild after touching one file
hyperfine --prepare "cmake --build build && touch src/main.cpp" "cmake --build build" --warmup 2
```

## Tracy for production profiling

Not a benchmark per se, but gives you real-world performance data:
- Which functions take the most time
- Mutex contention
- Memory allocation patterns
- Frame timing

See [debugging-profiling.md](debugging-profiling.md) for Tracy setup.

## perf (Linux) / ETW (Windows)

```bash
# Linux
perf record ./myapp
perf report

# Windows — use Tracy or Perfetto instead
# Or: xperf / ETW (complex)
```

## Benchmarking checklist

1. **Warmup** — always warm up (cache, branch predictor)
2. **Multiple runs** — at least 10, ideally 30+
3. **No other apps** — close browsers, Discord, etc.
4. **Same conditions** — same CPU governor, no thermal throttling
5. **Compare statistically** — hyperfine does this automatically
6. **Prevent optimization** — use `benchmark::DoNotOptimize()` or I/O
7. **Measure the right thing** — real workload, not synthetic loops
