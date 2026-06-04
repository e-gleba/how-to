# Benchmarking

## hyperfine

```bash
scoop install hyperfine

hyperfine './myapp'                                        # basic
hyperfine --warmup 5 --min-runs 20 './myapp'                # proper
hyperfine './old' './new'                                   # compare
hyperfine --prepare 'cmake --build build' './myapp --bench' # build first
hyperfine --export-markdown results.md './myapp'            # export
```

### Output

```
Benchmark 1: ./myapp
  Time (mean ± σ):      45.2 ms ±   2.1 ms    [User: 32.1 ms, System: 8.3 ms]
  Range (min … max):    42.1 ms …  52.3 ms    100 runs
```

Mean ± σ is the main result. Range shows best/worst. Outliers flagged.

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
        for (int i = 0; i < state.range(0); ++i) v.push_back(i);
        benchmark::DoNotOptimize(v);
    }
}
BENCHMARK(BM_VectorPush)->Range(8, 8<<10);

BENCHMARK_MAIN();
```

```cmake
FetchContent_Declare(benchmark GIT_REPOSITORY https://github.com/google/benchmark.git GIT_TAG v1.9.0)
FetchContent_MakeAvailable(benchmark)
target_link_libraries(mybench PRIVATE benchmark::benchmark)
```

## Quick micro-benchmark

```cpp
#include <chrono>
template<typename F>
auto measure(F&& f, int n = 1'000'000) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < n; ++i) f();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / n;
}
// Usage: auto ns = measure([]{ do_work(); });
```

## Compilation benchmarking

```bash
# Full rebuild time
hyperfine --prepare 'rm -rf build && cmake -B build -G Ninja' 'cmake --build build' --warmup 1

# Incremental rebuild (touch one file)
hyperfine --prepare 'cmake --build build && touch src/main.cpp' 'cmake --build build' --warmup 2
```

## Checklist

1. **Warmup** — always warm up (caches, branch predictor)
2. **Multiple runs** — at least 10, ideally 30+
3. **Quiet system** — close browsers, Discord, etc.
4. **Same conditions** — no thermal throttling, same CPU governor
5. **Prevent optimization** — use `benchmark::DoNotOptimize()`
6. **Measure the real thing** — not synthetic loops

> 💡 **Tip:** hyperfine is for CLI apps, Google Benchmark for micro-benchmarks, Tracy for whole-system profiling.
