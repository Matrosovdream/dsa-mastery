# 03 — Complexity Analysis

> Part of **Part 1 — Foundations**. Lesson [01](01-introduction.md) said Big-O is about growth;
> lesson [02](02-environment-setup.md) built the equipment to measure things. This lesson makes the
> notation precise — O vs Θ vs Ω, best/average/worst, **amortized**, space, and recurrences — and
> then *verifies* it with that equipment. The one-line thesis: **complexity is a claim about how the
> cost grows, and you can check that claim by counting operations or by doubling the input.**

## Goals
- Use **O**, **Ω** and **Θ** correctly, and know why `O(n²)` can be a true statement about a linear function.
- Keep the **case** (best/average/worst) and the **bound** (O/Ω/Θ) as separate axes, because they are.
- Apply the **ratio test**: double n, divide the costs, read off the complexity class.
- Analyse loops by counting how often the innermost body runs — including loops that look quadratic and aren't.
- Explain **amortized O(1)** with the aggregate method, and why it is neither average nor worst case.
- Count **space**, including the call stack and the difference between auxiliary and total.
- Solve recurrences with a **recursion tree** and the **master theorem**, and know what they don't cover.
- Build a **complexity regression gate** that fails CI when someone makes a linear function quadratic.

## Concepts

- **The three symbols are three different claims.** For a function with worst-case cost exactly `n`:
  `O(n)` (upper bound, tight), `O(n²)` (upper bound, true but weak), `Ω(n)` (lower bound), `Θ(n)`
  (both at once). Saying "this is O(n²)" about a linear function is **not wrong** — it's just useless.
  Say **Θ** when you mean tight, which is nearly always what you mean.

- **Case and bound are independent axes.** Linear search is Θ(1) in its best case, Θ(n) in its average
  case *(exactly (n+1)/2 comparisons)*, and Θ(n) in its worst. "An O(n) algorithm" is shorthand for
  "worst case is Θ(n)". When someone says a hash map is O(1) they mean **average** — the worst case is
  O(n) with adversarial collisions. Always ask which case a complexity refers to.

- **Constants and lower-order terms drop, and that is a deliberate loss of information.** `T(n) = 3n+7`
  is Θ(n) because `T(n)/(3n) → 1`. But `3n+7` beats `n²/100` until n ≈ 300. Asymptotics tell you what
  happens *eventually*; constants tell you what happens *today* (→ [01](01-introduction.md)).

- **The ratio test is the whole toolkit in one line.** Double the input and divide the costs:
  | ratio T(2n)/T(n) | class |
  |---|---|
  | ≈ 1 (grows by an additive constant) | O(log n) |
  | ≈ 2 | O(n) |
  | ≈ 2.1–2.3 | O(n log n) |
  | ≈ 4 | O(n²) |
  | ≈ 8 | O(n³) |
  
  It works on **operation counts** (exact, machine-independent) and on **measured times** (noisy — see
  the caveat below).

- **"Nested loop" is not a complexity.** Count how many times the *innermost body* runs, in total.
  `for j := i; j < n` is n(n+1)/2 — half the work, still Θ(n²). But `for i := 1; i <= n; i *= 2` runs
  only log n times, and `for j := 0; j < n; j += i` is the harmonic series `n·H(n) ≈ n ln n`. Both are
  double loops; both are Θ(n log n).

- **Sequential steps add; nested steps multiply.** `O(n)` then `O(n)` is `Θ(n)`. `O(n)` inside `O(n)`
  is `Θ(n²)`. `O(n)` then `O(n²)` is `Θ(n²)` — at n=400 the linear pass is **0.25%** of the work,
  which is precisely why lower-order terms are dropped. The trap: a cheap-*looking* call inside a loop
  is nested, so its cost multiplies.

- **Amortized O(1): bound the sequence, not the operation.** `append` is O(n) when it resizes, yet any
  sequence of n appends costs O(n) total. The **aggregate method**: with growth factor `g`, the copies
  form a geometric series summing to `n/(g−1)`. Textbook doubling (g=2) gives 1 copy per append; Go
  tapers g toward **1.25** for large slices, so the measured cost is **~4 copies per append** — still a
  constant, which is all that matters. Grow by a fixed `+1` instead and the total becomes n²/2.

- **Amortized is not average and is definitely not worst case.** Over 100,000 appends: amortized cost
  **4.02 copies**, worst single append **88,064 copies**, and only **25 appends (0.025%)** copied
  anything at all. Both "amortized O(1)" and "worst case O(n)" are true simultaneously. If you care
  about throughput, use the amortized number; if you care about **p99 latency**, that one expensive
  append *is* your tail.

- **Space includes the call stack.** A recursion whose depth grows with n uses Θ(depth) space, even
  though no line in it says `make`. Measured here: ~33–52 bytes per frame, so a 1,000,000-deep
  recursion holds **~33 MB** of goroutine stack. Go grows stacks dynamically (8 KB → 1 GB default),
  so deep recursion doesn't fail fast — it quietly consumes memory, and past the limit dies with
  `fatal error: stack overflow`, which **`recover()` cannot catch**.

- **Auxiliary vs total space.** "Space complexity" almost always means **auxiliary**: memory beyond the
  input. Merge sort is Θ(n) auxiliary; insertion sort is Θ(1); quicksort allocates no buffer but its
  recursion is Θ(log n) space (Θ(n) with bad pivots). That trade is the entire argument for in-place
  sorting when the data is large.

- **Recurrences: draw the tree.** For `T(n) = a·T(n/b) + f(n)`, level k has `a^k` nodes of size `n/b^k`.
  Merge sort's tree costs n at *every* level over log n levels → n log n. Binary search's costs 1 per
  level → log n. A tree traversal's **last level dominates** → n. Then the **master theorem** just names
  which of those three shapes you have, by comparing `d` against `log_b(a)`. And note that shrinking by
  a **factor** gives a log-deep tree while shrinking by a **constant** (`T(n) = T(n−1) + n`) gives an
  n-deep one — that single difference is most of the gap between n log n and n².

- **Quadratic hides inside single loops.** `s += x` on a string and `append([]T{v}, xs...)` are both
  Θ(n) *per iteration*, so a single loop around them is Θ(n²). Measured work per element: **999 → 3,999
  → 15,999** as n goes 2000 → 8000 → 32000 (growing = quadratic), versus `strings.Builder` at **1.7–3.5**
  (bounded = linear). On the clock at n=200,000: 964 ms and 6.97 s versus 425 µs and 116 µs.

- **The ratio test on timings cannot resolve a log factor.** Doubling n multiplies an O(n log n) cost by
  `2·log(2n)/log(n)` — at n=65536 that is **2.125**, a 6% difference from linear that does not survive
  contact with a real CPU. Measured: `SumAll` 2.00 vs `SortCopy` 2.19 — the signal is there, but it is
  too small to bet an assertion on. Timings separate classes that differ by a **factor of n**; operation
  counts separate everything. Watch for **cache cliffs** too: sorting showed a spurious ratio of 4.07 at
  n=16384, where the data crossed a cache boundary.

## Complexity Table

The costs to know by heart, with the case each one refers to:

| Operation | Typical | Worst | Note |
|-----------|---------|-------|------|
| slice index `xs[i]` | Θ(1) | Θ(1) | |
| `append` | Θ(1) **amortized** | Θ(n) | one op can copy everything |
| slice `copy` / full scan | Θ(n) | Θ(n) | |
| prepend `append([]T{v}, xs…)` | Θ(n) | Θ(n) | Θ(n²) inside a loop |
| map get / set / delete | Θ(1) **average** | Θ(n) | all keys colliding |
| map iteration | Θ(n) | Θ(n) | random order |
| `slices.BinarySearch` | Θ(log n) | Θ(log n) | needs sorted input |
| `slices.Sort` (pdqsort) | Θ(n log n) | Θ(n log n) | Θ(log n) stack |
| `slices.Contains` | Θ(n) | Θ(n) | the classic accidental O(n²) |
| string `s += x` | Θ(len(s)) | Θ(n) | Θ(n²) inside a loop |
| `strings.Builder.WriteX` | Θ(1) amortized | Θ(n) | |
| balanced-tree search / heap push-pop | Θ(log n) | Θ(log n) | |
| BFS / DFS over a graph | Θ(V+E) | Θ(V+E) | |

Recurrence shortcuts: `T(n)=T(n/2)+O(1)` → **log n** · `T(n)=2T(n/2)+O(n)` → **n log n** ·
`T(n)=2T(n/2)+O(1)` → **n** · `T(n)=T(n−1)+O(n)` → **n²** · `T(n)=2T(n−1)+O(1)` → **2ⁿ**.

## Exercises
1. Write a function whose worst-case cost is exactly `n`, and list which of `O(n)`, `O(n²)`, `Ω(n)`, `Ω(1)`, `Θ(n)`, `Θ(n²)` are true statements about it. Justify each with the definition.
2. Instrument linear search to report comparisons. Compute the exact average over all successful targets and confirm it equals `(n+1)/2`; then confirm it empirically with random targets.
3. Write a function that performs exactly `3n + 7` operations. Tabulate `T(n)/(3n)` and find the n beyond which it is within 1% of 1. Then find the n where `3n+7` overtakes `n²/100`.
4. Build the ratio-test table: six functions with known complexity, each returning an operation count, and the measured ratio `T(2n)/T(n)`. Confirm each lands in its predicted band.
5. Write four double loops — `j := 0`, `j := i`, outer `i *= 2`, inner `j += i` — and count the innermost body. Classify each, then verify against the closed forms `n(n+1)/2` and `n(log₂n+1)`.
6. Demonstrate that sequential steps add and nested steps multiply. For `O(n) + O(n²)` at n=400, compute what percentage of the work the linear pass contributes.
7. Instrument `append` to count total elements copied over n appends. Tabulate copies-per-append up to n=1,000,000 and explain why it converges to ~4 rather than ~1.
8. From the same run, report the **worst single** append and how many appends copied anything. Write one sentence that makes "amortized O(1)" and "worst case O(n)" both true.
9. Measure bytes of goroutine stack at recursion depths 1,000 → 1,000,000 (each in a fresh goroutine — explain why that matters). Derive bytes-per-frame, and compute the depth at which you'd hit a 1 GB limit.
10. Compare auxiliary space for merge sort, insertion sort, and quicksort by instrumenting allocations and recursion depth. Which is Θ(n), which Θ(1), which Θ(log n)?
11. Draw recursion trees for `T(n)=2T(n/2)+n`, `T(n)=T(n/2)+1` and `T(n)=2T(n/2)+1` at n=64: nodes, size, and work per level. Say which level dominates in each.
12. Implement the master theorem as a function of `(a, b, d)` and apply it to binary search, merge sort, Karatsuba, and Strassen. Then name one recurrence it cannot solve, and say why.
13. Count the work-per-element for `s += x`, `strings.Builder`, `append([]T{v}, xs...)`, and append-then-reverse. Show which columns grow with n. Then time all four at n=200,000.
14. Use `testing.Benchmark` to measure four functions at four sizes each and classify them from the ratios alone. Explain why O(n) and O(n log n) cannot be told apart this way.
15. Time appends in batches of 100 and report p50/p99/p99.9/max. Explain why timing individual appends does not work. Then show that reserving capacity removes the tail.
16. Stretch — write `AssertGrowth(t, sizes, build, lo, hi)` that fails when a measured doubling ratio leaves its band. Use it to catch a `Dedupe` that was "simplified" from a map to `slices.Contains` — and confirm that no correctness test would have caught it.

## Best Practices & Pitfalls
- **Say Θ when you mean Θ.** `O` is an upper bound. If you mean "this is exactly linear", `O(n)` technically allows the function to be far cheaper; `Θ(n)` says what you mean.
- **Always name the case.** "Map lookup is O(1)" is an average-case claim. "Quicksort is O(n log n)" is average-case; its worst case is O(n²). Writing the case down prevents the most common analysis error there is.
- **Count before you time.** Operation counts are exact, reproducible on every machine, and immune to cache effects. Get the shape from counts, then use timings to confirm the constants.
- **Pitfall — trusting a timing ratio too precisely.** Cache boundaries produce spurious ratios (4.07 for an n log n sort at n=16384). Sweep a wide range of n, use the median ratio, and never claim a log factor from timings.
- **Pitfall — the O(n) operation inside the O(n) loop.** `slices.Contains`, `append([]T{v}, xs...)`, `s += x`, and a database call in a loop are all this bug. To analyse a loop you must know the complexity of *every* operation in its body.
- **Pitfall — assuming amortized means "always fast".** It bounds the sequence. For a latency SLO, the resize spike is the number that matters: measured here at **8,600–14,000× the median**.
- **Preallocate when you know the size.** `make([]T, 0, n)` removes every resize, which cuts total time ~4× *and* eliminates the tail.
- **Count the stack.** Recursion depth is space. Tree recursion at Θ(log n) is fine; recursion down a list at Θ(n) should be a loop.
- **Pitfall — `recover()` will not save you from stack overflow.** It is a fatal runtime error, not a panic. Bound your recursion depth instead.
- **Drop the lower-order term only after checking it is lower-order.** `n + n²` is Θ(n²), but `n·m` with independent n and m is not Θ(n²) — keep both variables.
- **A complexity gate belongs in CI.** Correctness tests cannot see a Θ(n) → Θ(n²) regression; both implementations return identical results. A wide-banded ratio assertion catches it.

## Checklist
- [ ] I can state the definitions of O, Ω and Θ and explain why `O(n²)` is a true claim about a linear function.
- [ ] I never write a complexity without knowing which case it describes.
- [ ] I can apply the ratio test to operation counts and to timings, and I know why the second is less trustworthy.
- [ ] I can count the innermost body of a double loop and correctly classify the triangular, doubling and harmonic cases.
- [ ] I can prove `append` is amortized O(1) with the aggregate method, and explain why Go's number is ~4 and not ~1.
- [ ] I can state the difference between amortized, average and worst case, with an example where all three differ.
- [ ] I count the call stack as space, and I can measure it.
- [ ] I can distinguish auxiliary from total space and give an algorithm at each of Θ(1), Θ(log n), Θ(n).
- [ ] I can solve a recurrence with a recursion tree and with the master theorem, and name a case the master theorem misses.
- [ ] I can spot the O(n) operation hiding inside a loop, and I have a complexity gate in my test suite.

## Resources
- CLRS, *Introduction to Algorithms* — ch. 3 (growth of functions), ch. 4 (recurrences, master theorem), ch. 17 (amortized analysis). The reference text for this lesson.
- Go slice growth — the source of the amortized story: https://go.dev/blog/slices-intro
- `runtime.MemStats` (`StackInuse`, `HeapAlloc`): https://pkg.go.dev/runtime#MemStats
- `runtime/debug.SetMaxStack` — the 1 GB goroutine stack limit: https://pkg.go.dev/runtime/debug#SetMaxStack
- Master theorem: https://en.wikipedia.org/wiki/Master_theorem_(analysis_of_algorithms)
- Amortized analysis: https://en.wikipedia.org/wiki/Amortized_analysis
- Big-O cheat sheet — a reference, not a substitute for deriving it: https://www.bigocheatsheet.com/
- Examples: [examples/03-complexity-analysis](examples/03-complexity-analysis/) (16).
- Next: [04 — Go's Memory Model for DSA](04-go-memory-model.md) explains the constants this lesson taught you to drop.
