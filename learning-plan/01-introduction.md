# 01 — Introduction: What DSA Is & Why in Go

> Part of **Part 1 — Foundations**, the entry point to the whole plan. Nothing here is a data
> structure yet — this lesson installs the *habits*: count the work instead of guessing, know what
> Big-O does and doesn't promise, and learn to measure in Go without fooling yourself. The one-line
> thesis: **a data structure is a decision about where things live, and that decision writes half
> your algorithm for you.**

## Goals
- Say what a **data structure** and an **algorithm** are, and why choosing the structure is usually the bigger decision.
- Read **Big-O** as a claim about *growth*, not about speed — and always name the case (best/average/worst).
- Reason about the **trade-off triangle**: time, space, and the complexity of the code you have to maintain.
- Know why **Go** is a good language to learn this in: explicit memory, few hidden allocations, and a benchmark runner in the toolchain.
- Measure honestly — count operations first, reach for the clock second, and never trust a benchmark without a control.
- Adopt the repo's **three-pass method**: build it → prove it → measure it.

## Concepts

- **The two halves.** An **algorithm** is a decision about *what steps to take*. A **data structure**
  is a decision about *where the data lives*. They aren't independent: pick the structure and the
  algorithm is half written. The same question — "is `x` in here?" — is a full scan over a slice and
  a single lookup in a map. Nothing about the question changed.
  ```go
  contains(data, 55)      // slice: O(n) per lookup, nothing to prepare
  _, ok := set[55]        // map:   O(1) per lookup, O(n) to build once
  ```
  That trade — pay once to make every later query cheap — is most of applied DSA.

- **Big-O is about growth, not seconds.** `O(f(n))` says: as `n` grows, the work grows *no faster
  than* a constant times `f(n)`. It deliberately throws away constants and lower-order terms, because
  those depend on your CPU and the compiler, and the growth rate doesn't. Two consequences people
  skip: an O(1) operation can be slower than an O(n) one at your actual `n`, and "optimizing" an
  O(n²) loop by a factor of 2 leaves it O(n²).

- **Always name the case.** The *same* linear search is O(1) best-case and O(n) worst-case. When
  someone says "search is O(n)" they mean worst case; when they say a hash map is O(1) they mean
  *average* — the worst case is O(n) when every key collides. Amortized is a third thing again
  (→ [03](03-complexity-analysis.md)): `append` is O(n) on the resize and O(1) on every other call,
  averaging to O(1) per operation.

- **Count operations before you time anything.** A comparison counter is machine-independent, noise-free,
  and reproducible; a stopwatch is none of those. Instrument the loop, print the count at n, 10n, 100n,
  and read the growth off the table. Only once the shape is confirmed does the clock add information.
  ```
  n        brute ops    map ops
  100           4950        100
  1000        499500       1000     <- 10x the input, 100x the work: that's O(n^2)
  ```

- **Time and space are a currency pair.** Naive Fibonacci makes ~30 million calls at n=35; the same
  recursion with a `map` memo makes 69. You bought a drop from O(2ⁿ) to O(n) with O(n) of space.
  Almost every "clever" algorithm in this plan is some version of that purchase — and the third
  currency, **how hard the code is to get right**, is the one people forget to price in.

- **Why Go for this.** Go makes the things that decide real performance *visible* instead of hiding
  them: a `[]T` is a contiguous block and a `[]*T` is not, and you can see which you wrote. There's no
  hidden boxing, no surprise copy-on-write, no interpreter between you and the machine. And the
  toolchain ships the measuring equipment: `testing.B`, `-benchmem`, `AllocsPerRun`, `-race`, `pprof`,
  `go test -fuzz`. You will use all of them in this plan.

- **Constants decide small n; Big-O decides large n.** Both halves of that sentence matter. On this
  machine, a map lookup only beats a linear scan of a slice from **n ≥ 16**; hand-written insertion
  sort still beats `slices.Sort` at **n = 16** and loses by **n = 32**. That isn't a paradox — Go's own `slices.Sort` (pdqsort)
  drops to insertion sort at `n <= 12` for exactly this reason. Asymptotics tell you which algorithm
  survives growth; measurement tells you which one to ship today.

- **In Go, memory layout is part of the algorithm.** Summing 1M `point` values takes ~0.30 ns each;
  summing 1M `*point` in shuffled order takes ~0.8 ns — same O(n), 2.7× the time, because the second
  one *chases* memory instead of walking it. And the pointers cost a second time: a GC cycle with 1M
  live pointers ran **~59× longer** than with 1M pointer-free values. Choosing `[]T` over `[]*T` is
  an algorithmic decision here, not a style one (→ [04](04-go-memory-model.md)).

- **Measurement has its own traps.** A benchmark that discards its result may measure an empty loop.
  The classic "accumulate into a sink" fix does not always save you. `b.Loop` (Go 1.24+) prevents the
  elimination but has its **own ~1.6 ns/op floor** — so a benchmark reporting 1.6 ns/op for a one-multiply
  function measured the harness, not the function. The defenses are simple: **always run an empty
  control**, and when the operation costs less than ~2 ns, **batch it and divide** (→ [42](42-benchmarking-profiling.md)).

- **Correctness first, and prove it cheaply.** The fastest way to trust a clever implementation is to
  write the obvious slow one next to it and feed both the same small random inputs. A deliberately
  broken two-sum is caught in **26 trials** by a brute-force oracle. This pattern verifies every
  implementation in the rest of the plan (→ [43](43-testing-algorithms.md)).

## Complexity Table

The growth rates, and the input size each one can still handle at ~1 billion operations/second:

| Class | Name | Typical source | Feasible n |
|-------|------|----------------|------------|
| O(1) | constant | map lookup, slice index, push/pop | any |
| O(log n) | logarithmic | binary search, balanced-tree descent | astronomically large |
| O(n) | linear | one pass over the input | ~10⁹ |
| O(n log n) | linearithmic | comparison sorts, divide & conquer | ~10⁸ (10⁹ takes ~30 s) |
| O(n²) | quadratic | nested loop over all pairs | ~10⁴ (10⁶ takes ~17 min) |
| O(n³) | cubic | triple loop, naive matrix multiply | ~10³ |
| O(2ⁿ) | exponential | subsets, naive recursion | ~25 (n=100 takes 10¹³ years) |
| O(n!) | factorial | permutations, brute-force TSP | ~11 |

Rule of thumb for a 1-second budget: **n ≤ 10⁸ → O(n)** · **n ≤ 10⁶ → O(n log n)** · **n ≤ 10⁴ → O(n²)** · **n ≤ 25 → O(2ⁿ)**.

## Exercises
1. Take a `[]string` of 10,000 words and answer "does this list contain `w`?" 1,000 times — first by scanning, then by building a `map[string]struct{}` once. Time both, including the build.
2. Instrument a linear search with a comparison counter. Print the count for n = 10, 100, 1000 in the best, average and worst case, and confirm which column is flat.
3. Write the O(n²) and O(n) two-sum. Count operations at n = 10 … 10,000 and compute the ratio at each step. Predict the next row before you print it.
4. Print the growth table (log n, n, n log n, n², 2ⁿ) for n up to 10,000. Then answer: at 10⁹ ops/sec, what's the largest n each class can finish in one second?
5. Write naive and memoized Fibonacci with a call counter. Find the n where the naive version first takes longer than one second, then measure how long the memoized one takes at that same n.
6. Use `testing.Benchmark` inside a normal `main` program to benchmark a `sum([]int)` at n = 100 … 1,000,000. Divide by n and confirm ns-per-element stays flat.
7. Use `testing.AllocsPerRun` to compare `append` to a `nil` slice against `make([]T, 0, n)`. Explain the allocation count you see.
8. Find the crossover: at what n does a `map` lookup start beating a linear slice scan on your machine? Now do the same for insertion sort vs `slices.Sort`.
9. Measure the real memory of 1,000,000 ints held as `[]int`, `[]*int`, and `map[int]struct{}` using `runtime.MemStats`. All three are "O(n) space" — report the bytes per element.
10. Stretch — write a benchmark that measures nothing (discard the result of a small pure function), prove it with an empty-loop control, then fix it by batching. Then take any function you've written before and verify it against a brute-force oracle on 100,000 random small inputs.

## Best Practices & Pitfalls
- **Name the case or the complexity is meaningless.** "Search is O(n)" is worst case; "map lookup is O(1)" is average. Write which one you mean, especially in a code comment.
- **Count first, time second.** Operation counts are reproducible and reviewable; wall-clock numbers depend on your machine, your thermal state, and what else is running.
- **Pitfall — a benchmark with no control.** If you can't state what an *empty* version of the benchmark costs, you don't know whether you measured your code or the harness. In this lesson's example, `b.Loop` around a one-multiply function reported 1.62 ns/op and an empty `b.Loop` reported 1.60.
- **Pitfall — the discarded result.** Small pure functions get inlined and deleted when their result is unused. Dead-code elimination is *selective* though — `fib(20)` discarded in a `b.N` loop still cost ~15 µs/op, because a recursive call can't be inlined or proven pure. Don't assume either way; check with a control.
- **Don't optimize the constant when the exponent is wrong.** Making an O(n²) loop 2× faster is a rounding error next to making it O(n log n). Find the exponent first.
- **But don't quote asymptotics at small n either.** Below the crossover, the "worse" algorithm wins. Measure your actual input size before you rewrite anything.
- **Prefer `[]T` to `[]*T` until you have a reason.** The pointer version costs you cache locality *and* GC scan time, and neither shows up in the Big-O.
- **Preallocate when you know the size.** `make([]T, 0, n)` turns ~25 allocations into 1 at n=100,000 — free, and it doesn't change the complexity you'd write down.
- **Get it right before you get it fast.** Write the slow obvious version, keep it as the test oracle, then make the fast one agree with it on random input.
- **The stdlib usually wins.** Hand-roll a structure to *learn* it; then use `slices.Sort`, `container/heap`, `map`, and friends in real code. Every lesson in this plan says which stdlib type replaces what you just built.

## Checklist
- [ ] I can explain the difference between a data structure and an algorithm, with an example where changing the structure changes the complexity.
- [ ] I can state what O(f(n)) claims, what it discards, and why the case (best/average/worst) must be named.
- [ ] I can recite the growth table and the feasible-n rule of thumb for a one-second budget.
- [ ] I can count operations to identify a complexity class without timing anything.
- [ ] I can write a `testing.Benchmark` with an empty-loop control and explain why the control is mandatory.
- [ ] I can name two Go-specific costs that Big-O doesn't capture (cache locality, allocations, GC scanning).
- [ ] I can describe the three-pass method and the brute-force-oracle pattern well enough to apply them in lesson 02.

## Resources
- The `testing` package (`B`, `Benchmark`, `AllocsPerRun`, `b.Loop`): https://pkg.go.dev/testing
- The `slices` package — what you'll be told to use instead of hand-rolling: https://pkg.go.dev/slices
- Go Slices — usage & internals (the memory model behind everything in Part 2): https://go.dev/blog/slices-intro
- Diagnostics — profiling, tracing, and the rest of the measurement toolbox: https://go.dev/doc/diagnostics
- Profiling Go programs: https://go.dev/blog/pprof
- Go 1.19 release notes — the switch of `sort` to **pdqsort**: https://go.dev/doc/go1.19
- Big-O cheat sheet (a reference, not a substitute for deriving it): https://www.bigocheatsheet.com/
- Examples: [examples/01-introduction](examples/01-introduction/) (14).
- Next: [02 — Environment & Toolkit](02-environment-setup.md) turns the measuring habits here into a reusable test + benchmark harness.
