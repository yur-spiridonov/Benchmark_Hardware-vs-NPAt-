# Hardware IEEE 754 Baseline Benchmark

Reference benchmark for measuring **hardware FPU performance** on sequential floating-point accumulation.  
Published as part of the [NPAt Pathway 1](https://github.com/yur-spiridonov/Benchmark_Hardware-vs-NPAt-) verification suite.

---

## Purpose

This benchmark establishes the **Hardware IEEE 754 baseline** — the performance of a native FPU (`ADDSD` instruction) on a sequential accumulation loop with 10⁹ iterations.

It is the reference point against which **NPAt Pathway 1** (a software ALU-based accumulator) is compared. The benchmark is intentionally minimal and transparent so that results can be independently reproduced.

---

## What It Measures

- **Total CPU cycles** via `__rdtsc()` for Z = 1,000,000,000 iterations
- **Cycles per iteration** (mean and median across 9 runs)
- **Spread** (%) — variance across runs
- **Final sum** — printed to 50 decimal places for bit-exact verification

---

## Test Configuration

| Parameter | Value |
|---|---|
| Compiler | MSVC — Release x64 |
| Optimization | `/O2` (maximum) |
| OS | Windows 11 |
| Thread affinity | Core 0 only |
| Thread priority | `REALTIME_PRIORITY_CLASS` |
| Warm-up | 1 full run before measurement |
| Runs | 9 independent runs |
| Iterations (Z) | 1,000,000,000 |
| Output precision | 50 decimal places |

---

## Input Data

The benchmark accepts any `double` values for X1 and X2 — set them directly in the source before building. Results have been verified across a wide range of input types:

| Input type | Example |
|---|---|
| Subnormal (denormalized) | X1 = −9.999e−311, X2 = +9.999e−312 |
| Small normal, subtraction | X1 = −1.789e−31, X2 = +1.765e−26 |
| Normal fractional | X1 = 0.125, X2 = 0.625 |
| Large integers | X1 = 3.456e+06, X2 = 8.765e+09 |
| Extreme magnitude | X1 = 3.456e+200, X2 = 9.876e+200 |

The reference run uses **subnormal (denormalized) numbers** — a particularly interesting case because subnormal inputs trigger microcode assist on x86-64, increasing FPU pipeline latency. NPAt operates entirely in integer registers and is unaffected by this overhead, which is reflected in its best measured speedup of **×2.12** for this input type.

> The **worst case for NPAt** is when the binary structure of the input mantissas causes
> the guard bits to require a rounding correction on almost every iteration (e.g. X1 = −1.789e−31, X2 = +1.765e−26),
> reducing the speedup to **×1.46**. Even so, NPAt remains faster than hardware FPU across all tested inputs.

```
X1 (initial):  -9.999999999999996945e-311   // reference run — subnormal
X2 (addend):    9.999999999999947538e-312   // reference run — subnormal
Z  (iters):     1000000000
Mode:           VOLATILE (L1 Latency)
```

---

## Reference Results

Hardware IEEE 754 baseline — subnormal inputs:

![Hardware IEEE 754 baseline — subnormal](Screenshot%202026-04-18%20131638.png)

NPAt Pathway 1 — same inputs, ×2.12 speedup, bit-exact result:

![NPAt Pathway 1 — subnormal](Screenshot%202026-04-18%20131920.png)

---

## Methodology

**Sequential dependency chain.** The accumulator variable is declared `volatile` to prevent the compiler from reordering or vectorizing the loop. Each addition depends on the result of the previous one — this is the target pattern for NPAt Pathway 1.

**Cycle counting.** `__rdtsc()` is read immediately before and after the hot loop. The thread is pinned to Core 0 with `REALTIME_PRIORITY_CLASS` to minimize OS scheduling noise.

**Multiple runs.** 9 independent runs are executed. Min, median, and max cycles/iter are reported. Spread is computed as `(max − min) / median × 100%`. A spread below 2% is considered a clean measurement.

**Warm-up.** One full Z-iteration run is executed before measurement to bring code and data into L1/L2 cache.

**Output verification.** The final sum is printed with 50 significant decimal digits. This value is compared bit-for-bit against the NPAt Pathway 1 result to confirm IEEE 754 correctness.

---

## Results Across Input Types

Speedup varies by input type — from **×1.46** (heavy subtraction) to **×2.26** (normal fractional, same sign).  
Full results with raw output for all 7 tested input ranges: **[RESULTS.md](RESULTS.md)**

---

## Building

```bash
# MSVC (recommended — matches reference results)
cl /O2 /fp:precise /EHsc benchmark_hw.cpp /Fe:benchmark_hw.exe

# Run
benchmark_hw.exe
```

> **Note:** Results are hardware- and compiler-specific. Reference numbers above were obtained on a specific x86-64 machine. Your cycle counts will vary by CPU microarchitecture, but the methodology and spread should be consistent.

---

## Repository Structure

```
benchmark/
├── benchmark_hw.cpp                    # Hardware IEEE 754 baseline source
├── README.md                           # This file
├── RESULTS.md                          # Full results across all input types
└── *.png                               # Benchmark output screenshots
```

---

*Part of the NPAt Pathway 1 verification suite · Author: Yuri Spiridonov · 2026*
