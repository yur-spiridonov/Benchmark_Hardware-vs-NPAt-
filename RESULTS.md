# Benchmark Results — NPAt Pathway 1 vs Hardware IEEE 754

All tests: Z = 1,000,000,000 iterations · 9 runs · MSVC /O2 · Windows 11 · Core 0 · REALTIME_PRIORITY_CLASS  
Hardware mode: `VOLATILE (L1 Latency)` · NPAt mode: `VOLATILE (Full Normalization)`

---

## Summary Table

| # | Input Range | Type | HW cycles/iter | NPAt cycles/iter | Speedup |
|---|---|---|:---:|:---:|:---:|
| 1 | X1: −9.999e−311 / X2: +9.999e−312 | Subnormal (denormalized) | 5.2147 | 2.4553 | **×2.12** |
| 2 | X1: −1.789e−31 / X2: +1.765e−26 | Small normal, heavy subtraction | 5.4773 | 3.7429 | **×1.46** |
| 3 | X1: 1.25e−01 / X2: 6.25e−01 | Normal fractional, same sign | 5.2580 | 2.3242 | **×2.26** |
| 4 | X1: 3.456e+06 / X2: 8.765e+09 | Large integers, same sign | 5.4411 | 2.5621 | **×2.12** |
| 5 | X1: 3.456e+15 / X2: 9.876e+15 | Very large, same sign | 5.2571 | 2.7267 | **×1.93** |
| 6 | X1: 3.456e+200 / X2: 9.876e+200 | Extreme magnitude, same sign | 5.2394 | 2.6708 | **×1.96** |
| 7 | X1: −3.456e+200 / X2: +9.876e+200 | Extreme magnitude, subtraction | 5.2793 | 2.7235 | **×1.94** |

> **Key observation:** NPAt Pathway 1 outperforms hardware FPU across all tested input ranges.  
> The minimum speedup of **×1.46** occurs in the heavy subtraction case (Test 2) — explained below.

---

## Detailed Results

---

### Test 1 — Subnormal (Denormalized) Numbers

> The most demanding case for hardware FPU: subnormal inputs trigger microcode assist on x86-64,
> significantly increasing FPU pipeline latency. NPAt operates entirely in integer registers
> and is unaffected by subnormal handling overhead.

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — subnormal](Screenshot%202026-04-18%20131638.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — subnormal](Screenshot%202026-04-18%20131920.png)

**Result: ×2.12 speedup · 100% bit-exact match**

---

### Test 2 — Small Normal Numbers, Heavy Subtraction

> The worst case for NPAt performance in this test suite. The frequency of rounding corrections
> in NPAt is determined by the binary representation of the input mantissas — not by the
> exponent gap between X1 and X2. For this particular pair (X1 = −1.789e−31, X2 = +1.765e−26),
> the binary structure of the mantissas causes the guard bits to require a rounding correction
> on almost every iteration — the mantissa is incremented or decremented nearly every cycle.
> This persistent rounding overhead reduces the speedup to ×1.46. Other input pairs in this
> suite trigger rounding much less frequently, resulting in higher throughput.

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — small normal subtraction](Screenshot%202026-04-21%20184530.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — small normal subtraction](Screenshot%202026-04-21%20181129.png)

**Result: ×1.46 speedup · NPAt still faster despite heavy subtraction workload**

---

### Test 3 — Normal Fractional Numbers, Same Sign

> Clean addition case: both operands positive, moderate magnitude, exponents close.
> NPAt achieves its best result here — ×2.26. The hot loop runs without triggering
> the rounding branch (`K1 < X_max`), so the ALU integer pipeline executes at full throughput.
> Note: "normalization" in NPAt refers exclusively to the one-time decomposition of X1 and X2
> into 53-bit integer mantissas at initialization — there is no normalization step inside the loop.

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — normal fractional](Screenshot%202026-04-20%20120513.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — normal fractional](Screenshot%202026-04-20%20120003.png)

**Result: ×2.26 speedup · 100% bit-exact match**

---

### Test 4 — Large Integers, Same Sign

> Large positive operands with significant exponent difference (e+06 vs e+09).
> NPAt handles the shift in integer registers cleanly.

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — large integers](Screenshot%202026-04-21%20112550.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — large integers](Screenshot%202026-04-20%20121846.png)

**Result: ×2.12 speedup · 100% bit-exact match**

---

### Test 5 — Very Large Numbers, Same Sign

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — very large](Screenshot%202026-04-20%20122555.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — very large](Screenshot%202026-04-20%20122136.png)

**Result: ×1.93 speedup · 100% bit-exact match**

---

### Test 6 — Extreme Magnitude, Same Sign

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — extreme magnitude](Screenshot%202026-04-20%20122726.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — extreme magnitude](Screenshot%202026-04-20%20123030.png)

**Result: ×1.96 speedup · 100% bit-exact match**

---

### Test 7 — Extreme Magnitude, Subtraction (negative X1)

> Opposite signs at extreme magnitude. Performance is comparable to Tests 5 and 6 (×1.94)
> because NPAt performance does not depend on the exponent values themselves — only the
> maximum exponent is tracked, incremented by 1 on overflow. The dominant cost in both
> execution paths is the rounding step: correcting the mantissa based on the state of the
> guard bits. This cost is similar regardless of sign or magnitude, which is why the
> speedup remains consistent here.

**Hardware IEEE 754 Baseline**

![Hardware IEEE 754 — extreme magnitude subtraction](Screenshot%202026-04-20%20123322.png)

**NPAt Pathway 1**

![NPAt Pathway 1 — extreme magnitude subtraction](Screenshot%202026-04-21%20111721.png)

**Result: ×1.94 speedup · 100% bit-exact match**

---

## Notes on Spread

Hardware FPU spread is consistently **< 2%** — the FPU pipeline runs at a highly predictable latency.

NPAt spread is **7–12%** — higher variance is expected because the mantissa rounding step fires conditionally based on the state of the guard bits, which varies with the accumulated value. This creates variable per-iteration cost depending on input data. This is a measurement artifact, not an instability: the minimum and median values are consistent across sessions.

---

## Reproducibility

All results obtained on the same machine under identical conditions. To reproduce:

1. Build `benchmark_hw.cpp` with `cl /O2 /fp:precise benchmark_hw.cpp`
2. Set X1, X2 to the values listed above
3. Run on a quiet system (no background load)
4. Results will vary by CPU microarchitecture — cycles/iter will differ, but the **speedup ratio** should be consistent on any modern x86-64

Full source and build instructions: [README.md](https://github.com/yur-spiridonov/Benchmark_Hardware-vs-NPAt-/blob/main/README.md)
