# 06 — Dynamic Arrays & Slices

> Opens **Part 2 — Linear Structures**, and the first lesson where you *build* rather than measure.
> Part 1 established the tools; from here the **three-pass method** applies to every structure:
> build it, prove it, measure it. The one-line thesis: **a dynamic array gives you free random access
> and charges Θ(n) for structural change in the middle — and every other linear structure in Part 2
> is a different answer to that one sentence.**

## Goals
- Implement the four dynamic-array operations and know which two are Θ(1) and which two are Θ(n).
- Choose between order-preserving and swap deletion, and nil the vacated slot when `T` holds a pointer.
- Write the **write-index idiom** by reflex — the single most useful slice pattern there is.
- Reverse and rotate in place, in Θ(1) space.
- Build `Vector[T]`, derive the amortized bound by counting, and pick a growth factor deliberately.
- Avoid **shrink thrashing** with hysteresis.
- Lay out 2-D data three ways and know when each wins.
- Recognize the accidental Θ(n²) that `slices.Delete` in a loop produces.
- Partition in place, including the three-way Dutch national flag.

## Concepts

- **Four operations, two costs.** `Get` is one address calculation; `Append` is amortized Θ(1);
  `Insert` and `Delete` are Θ(n) because the tail must shift. Both are one `copy` call — and `copy`
  compiles to a `memmove` that moves whole cache lines, so Θ(n) here has a very small constant.

- **Two deletions, and they're a design decision.** Order-preserving delete shifts (Θ(n));
  **swap-delete** moves the last element into the hole (Θ(1)) and scrambles the order. Emptying a
  slice from the front is Θ(n²) one way and Θ(n) the other. If order is meaningless, say so.

- **Nil the vacated slot when `T` contains a pointer.** Both deletions leave a live copy in
  `[len:cap]` where the GC can still see it. For `[]int` nobody cares; for `[]*Session` in a
  long-lived server it's a leak (→ [04](04-go-memory-model.md)).

- **The write-index idiom.** Two cursors over one array: `read` visits everything, `write` marks where
  the next kept element goes. Because `write` never overtakes `read`, you can overwrite in place —
  Θ(n) time, **Θ(1) space, zero allocations**:
  ```go
  write := 0
  for read := range xs {
      if keep(xs[read]) { xs[write] = xs[read]; write++ }
  }
  return xs[:write]
  ```
  Filter, dedupe, move-zeroes and remove-all are the same five lines with a different condition. The
  alternative — deleting inside the loop — is both Θ(n²) *and* wrong, because the tail slides under
  the cursor and `i++` steps over an element.

- **Reverse and rotate.** Reverse is two cursors meeting in the middle (the odd-length case needs no
  special handling). Rotate is **three reversals**: reverse the first k, reverse the rest, reverse
  everything. Θ(1) space and three sequential passes, versus a copy version that needs a second array
  and writes in a scattered order. Normalize k with `((k % n) + n) % n` — Go's `%` keeps the dividend's sign.

- **A sorted slice is a real structure.** It answers predecessor, successor, rank and range — none of
  which a map can do at any price. Maintaining it splits in two: **Θ(log n)** to find the position,
  **Θ(n)** to make room. Building one by repeated insertion is Θ(n²); if you have the data up front,
  append everything and sort once.

- **Deriving amortized Θ(1).** Counted, not asserted: doubling from 1 to n copies `1+2+4+…+n/2 = n−1`
  elements for n pushes, so **copies/push converges to 1.00**. With growth factor `g` the series sums
  to `n/(g−1)` — a constant multiple of n for any fixed `g > 1`. Growing by a fixed **+k** instead
  makes the total Θ(n²) and the per-push cost grow without bound (measured: 7.7 → 78.4 → 781.2).

- **The growth factor is a real trade.** Measured over 1M appends: ×1.25 costs **4.49 copies/push**
  and never over-allocates past 1.25×; ×4 costs **0.35** and over-allocates up to 4×. Every row's
  copies/push matches `1/(g−1)`. There's a third consideration too: with `g = 2` each new block
  exceeds the sum of all freed blocks, so the allocator can never reuse them — which is why several
  implementations chose 1.5.

- **Shrinking is where hand-rolled vectors break.** "Never waste memory — shrink to fit when half
  empty" is a natural policy and it **thrashes**: at `len == cap` a push grows and copies n, the pop
  leaves it exactly half empty, shrink-to-fit copies n again and lands back on `len == cap`. Measured
  at **2048 elements copied per push/pop pair**, forever. The fix is **hysteresis** — grow at 100%
  full, shrink at 25% full, and only halve. This is why Go's slices never shrink at all: the runtime
  declines to guess.

- **`buf[:0]` is the best optimisation available to a hot loop.** Reusing a buffer measured
  **3.2× faster and 0 allocations** versus 13. Know the three ways to empty a slice: `xs[:0]` reuses
  the memory, `clear(xs)` zeroes the values and keeps the length, `xs = nil` releases it. If `T`
  holds a pointer, `clear` before reslicing.

- **`[][]int` is a slice of slices, not a 2-D array.** One allocation *per row* — measured 997 heap
  objects for a 1000×1000 grid, versus a handful for a single backing array. The four-line
  **backing-array** version keeps `g[r][c]` syntax with one allocation, and is usually the right
  answer. A flat `[]T` with `r*cols + c` is fastest to scan, but only when rows are short enough for
  the indirection to matter (measured **1.2×** at 4-wide rows, and *nothing* at 1000-wide). Choose on
  allocations, not speed.

- **`slices.Delete` in a loop is Θ(n²).** Removing half of 64,000 elements one at a time measured
  **355× slower** than one pass — and the ratio grows with n, which is a complexity-class difference,
  not a constant. `slices.DeleteFunc` *is* the write-index idiom, written once in the stdlib.

- **Partition maintains an invariant.** Two-way partition is the write-index idiom with a swap; Hoare's
  two-pointer version does the same job with **half the swaps** (2524 vs 4976) because it only touches
  genuinely misplaced pairs. The **Dutch national flag** three-way partition handles duplicates in one
  pass, which is why pdqsort uses it.

- **You can't remove the Θ(n) shift, but you can choose where it happens.** A **gap buffer** keeps the
  free space at the cursor, making a keystroke Θ(1) and cursor movement Θ(distance). Measured on
  front-insertion: **46× faster at n=16,000**, and the gap widens with n. It's a bet that edits
  cluster — which for a human typing, they do.

## Complexity Table

| Operation | Cost | Note |
|-----------|------|------|
| `xs[i]` | **Θ(1)** | one address calculation, ~1.7 ns at any n |
| `append` | **Θ(1) amortized** | Θ(n) on the resize |
| Insert at end | Θ(1) amortized | this is `append` |
| Insert at i | **Θ(n)** | shifts `n−i` elements |
| Insert at front | Θ(n) | worst case: shifts everything |
| Delete at i, ordered | **Θ(n)** | shifts `n−i−1` |
| Delete at i, swap | **Θ(1)** | destroys order |
| Delete k elements, one pass | **Θ(n)** | write-index |
| Delete k elements, in a loop | **Θ(n·k)** | the accidental quadratic |
| Search unsorted | Θ(n) | |
| Search sorted | Θ(log n) | plus Θ(n) to insert |
| Reverse / rotate | Θ(n) time, **Θ(1) space** | |
| Partition (2- or 3-way) | Θ(n) time, Θ(1) space | |

Measured (Apple M4, Go 1.26.3): copies/push → **1.00** at ×2 · thrashing costs **2048 copies per
push/pop pair** · buffer reuse **3.2×** · `slices.Delete` in a loop **355×** slower at n=64,000 ·
gap buffer **46×** faster at n=16,000.

## Exercises
1. Implement `Insert(xs, i, v)` and `Delete(xs, i)` with `copy`. Tabulate how many elements shift for i at the front, middle and end.
2. Write both deletions. Count the total shifts to empty a slice from the front each way. Then show that a `[]*T` delete leaves a live pointer past the length, and fix it.
3. Write `FilterInPlace` with the write-index idiom. Then write the delete-inside-the-loop version and find an input where it produces the wrong answer — explain why.
4. Implement `Reverse` and `RotateLeft` via triple reversal. Test k = 0, 1, n−1, n, n+2 and −1 against a copy-based reference.
5. Build a sorted slice by repeated insertion, counting comparisons and shifts separately. Show one grows like n log n and the other like n². Then answer predecessor, successor, rank and range queries with it.
6. Replace your hand-written versions with the `slices` equivalents. Then demonstrate that `slices.Compact` does *not* dedupe unsorted input.
7. Build `Vector[T]` with `Push`/`Pop`. Count total elements copied for n = 16…1,048,576 and show copies/push converges. Then implement grow-by-+64 and show it doesn't.
8. Add `Insert`/`Delete`/shrink. Implement shrink-to-fit-at-half and demonstrate thrashing at the capacity boundary; then fix it with hysteresis and show the cost goes to zero.
9. Measure ×1.25, ×1.5, ×2 and ×4: copies per push and worst-case capacity/length. Check each against `1/(g−1)`.
10. Benchmark a fresh slice per iteration against `buf[:0]` reuse. Report ns/op and allocs/op. Then show the pointer-retention trap and fix it with `clear`.
11. Build a 1000×1000 grid three ways: row-per-allocation, one backing array, and flat with index math. Compare heap objects and scan time — then repeat at 250,000×4 and explain why the ranking changes.
12. Compare a sorted slice against a map for lookup-only, and then for mixed insert/lookup workloads at several ratios. Report the ratio and say what actually decides the choice.
13. Implement a gap buffer with `Insert`, `Delete` and `MoveTo`. Benchmark front-insertion against a plain `[]byte` at three sizes and confirm the Θ(n²)-vs-Θ(n) signature.
14. Remove half a slice's elements with `slices.Delete` in a loop, and with one pass. Show the ratio growing with n.
15. Implement two-way partition (write-index and Hoare) and count swaps for each. Then implement the Dutch national flag and explain the "do not advance mid" case.
16. Stretch — finish `Vector[T]` and apply all three passes: a table, invariant checks, a **model oracle** against a plain `[]T` over 20,000 random operations, an allocation budget, a geometric-growth gate, and a no-thrash gate. Then benchmark against native `append`.

## Best Practices & Pitfalls
- **Reach for the write-index idiom** whenever you're removing more than one element. It's Θ(n), Θ(1) space, and allocation-free.
- **Pitfall — `slices.Delete` inside a loop.** Θ(n²). Calling the stdlib makes an operation correct, not cheap.
- **Pitfall — deleting while iterating forward.** The tail slides under your cursor and you skip elements. Iterate backwards, or filter in one pass.
- **Pitfall — a stale pointer past `len`.** Zero the vacated slot when `T` contains a pointer.
- **Preallocate with `make([]T, 0, n)`** when n is known — it eliminates every copy and the latency tail.
- **Reuse with `buf[:0]`** in hot loops, but never return a view of a reused buffer to a caller who outlives the next iteration.
- **Pitfall — `[][]T` built with one shared row.** `[][]int{row, row, row}` is one row three times. Allocate inside the loop, or use a backing array.
- **Prefer one backing array for 2-D data.** Four lines, keeps `g[r][c]`, and turns n+1 allocations into 2.
- **Iterate row-major.** Rows are contiguous; swapping the loop order turns a sequential walk into a strided one.
- **If you write a shrink policy, give it hysteresis.** Grow at full, shrink at a quarter, halve — never shrink to fit.
- **Pitfall — building a sorted slice by repeated insertion.** Θ(n²). Append everything and sort once.
- **Don't choose a sorted slice for lookup speed** — a map wins on speed at every size measured. Choose it for memory, ordering and range queries.

## Checklist
- [ ] I can implement insert and delete with `copy` and state the cost of each by position.
- [ ] I know both deletion idioms, when each applies, and why the vacated slot may need zeroing.
- [ ] I write the write-index idiom without thinking, and I can name four problems it solves.
- [ ] I can rotate a slice in Θ(1) space and normalize a negative k.
- [ ] I can derive amortized Θ(1) from the geometric series and explain why +k growth fails.
- [ ] I can pick a growth factor and defend it on both copies and waste.
- [ ] I can explain shrink thrashing and write a policy with hysteresis.
- [ ] I know the three ways to empty a slice and what each means for memory.
- [ ] I can lay out a 2-D grid three ways and say which to use given an access pattern.
- [ ] I can spot the `Delete`-in-a-loop quadratic in a code review.
- [ ] I can write a three-way partition and state its invariant.
- [ ] I have applied all three passes to a structure I built: table, invariants, model oracle, allocation budget, growth gate.

## Resources
- Go Slices: usage and internals: https://go.dev/blog/slices-intro
- Arrays, slices: the mechanics of `append` and `copy`: https://go.dev/blog/slices
- `slices` package: https://pkg.go.dev/slices
- `slices.DeleteFunc` — the write-index idiom, in the stdlib: https://pkg.go.dev/slices#DeleteFunc
- Go slice tricks (the community cheat sheet): https://go.dev/wiki/SliceTricks
- Dutch national flag problem: https://en.wikipedia.org/wiki/Dutch_national_flag_problem
- Gap buffer: https://en.wikipedia.org/wiki/Gap_buffer
- CLRS ch. 17.4 — dynamic tables and the amortized analysis of table doubling.
- Examples: [examples/06-arrays-slices](examples/06-arrays-slices/) (16).
- Next: [07 — Linked Lists](07-linked-lists.md) — the mirror image of this lesson's trade-off, and the honest verdict on when it wins.
