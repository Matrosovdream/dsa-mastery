# 13 — Efficient Sorts: Divide & Conquer

> Part of **Part 3 — Sorting & Searching**. [12](12-elementary-sorts.md) sorted by moving one element
> at a time, and paid Θ(n²) for it. This lesson splits the problem in half instead, and reaches the
> Θ(n log n) that no comparison sort can beat. There are three classic ways to do that, and each one
> fails somewhere: **merge sort** needs a second array, **quicksort** has an input that makes it
> quadratic, and **heapsort** is slow on real caches. The one-line thesis: **no standard library ships
> any of the three on its own — `slices.Sort` and Rust's `sort_unstable` are hybrids that run each
> algorithm only where the other two fail, and once you can read them you know all three.**

## Goals
- Write the **merge step** correctly, and say what one character in it does to stability.
- Write merge sort **top-down** and **bottom-up**, with one scratch buffer and no allocation per call.
- Write **Lomuto** and **Hoare** partitions, and say which one breaks on duplicates and why.
- Explain why the pivot choice decides quicksort's complexity, and build the input that defeats a fixed rule.
- Use a **3-way partition** when the input has few distinct values.
- Write **heapsort** in place, and explain why it is the fallback rather than the default.
- Read `slices.Sort`'s source and name what each part of it is for (**pdqsort**).
- Find the k-th smallest element in expected Θ(n) with **quickselect**, without sorting.
- Sketch an **external sort**: sorted runs on disk, then a k-way merge.

## Concepts

- **The lower bound comes first.** A comparison sort must tell apart all n! orderings of its input,
  and each comparison has two outcomes, so it needs at least **lg(n!) ≈ n lg n − 1.44n** comparisons
  in the worst case. Every algorithm here is measured against that line. [14](14-linear-sorts.md)
  beats it by not comparing at all.

- **Divide & conquer, two ways round.** Merge sort does the trivial split (the middle) and the real
  work on the way *up*, in the merge. Quicksort does the real work on the way *down*, in the
  partition, and its combine step is nothing at all. The recurrence T(n) = 2T(n/2) + Θ(n) gives
  Θ(n log n) for both — but only quicksort depends on the split being even.

- **The merge step.** Two sorted runs of lengths m and n merge in **at most m + n − 1 comparisons**,
  and in as few as min(m, n) when one run lies entirely below the other. Take from the left run when
  the two heads are **equal** (`<=`, not `<`), and the sort is stable. Write `<` and it still sorts,
  passes every test on distinct keys, and is unstable. The same loop also counts **inversions** for
  free: each time an element comes from the right, it jumps every element still waiting on the left.

- **Top-down merge sort.** Recurse on each half, merge into a scratch buffer, copy back. Allocate
  that buffer **once** and pass it down; allocating inside every call turns n log n comparisons into
  n allocations as well. For n a power of two it makes at most **n lg n − n + 1** comparisons. It is
  Θ(n log n) on every input, stable, and needs Θ(n) extra space. That last point is the whole price.

- **Bottom-up merge sort** drops the recursion. Merge runs of width 1, then 2, then 4, swapping the
  roles of the two buffers each pass, so nothing is copied back. It also lets you start from the
  runs the input already contains: a **natural** merge sort scans for ascending runs first, so sorted
  input costs n − 1 comparisons and no merges. Real stable sorts are built this way.

- **External sort is the merge step at scale.** If the data is bigger than memory, sort chunks of M
  elements that fit, write each chunk to disk as a run, then merge all k runs at once with a min-heap
  over their heads: Θ(n log k) comparisons, and every element is read and written a fixed number of
  times. **Replacement selection** — a heap that keeps emitting while new elements arrive — makes runs
  of about **2M** on random input, so it halves the number of runs. Example 14 builds both.

- **Quicksort's partition** picks a pivot and moves smaller elements left and larger ones right. The
  two classic schemes:

  | | **Lomuto** | **Hoare** |
  |---|---|---|
  | Scans | one index, left to right | two indices, from both ends toward each other |
  | Swaps on random input | about n/2 per partition | about n/6 per partition |
  | Pivot ends up | at its final index | *not* necessarily — returns a split point |
  | All keys equal | every key goes to one side: **Θ(n²)** | indices meet in the middle: Θ(n log n) |
  | Easy to get right | yes | no — off-by-one in the bounds, and `j` can step below `lo` |

  Lomuto is the one textbooks teach, and it is quadratic on an array of identical keys. Hoare's
  version stops on equal keys from both sides, which is exactly what splits a run of duplicates evenly.

- **The pivot decides everything.** With the last element as pivot, sorted input splits every range
  into n − 1 and 0, and quicksort makes exactly **n(n−1)/2 comparisons** with **recursion depth
  n − 1**. Median-of-three fixes sorted and reversed input, and a random pivot makes the expected
  count **≈ 2n ln n ≈ 1.39 n lg n** on *every* input. But no deterministic rule is safe: for any fixed
  rule there is an input that defeats it. Recurse on the smaller side and loop on the larger, and the
  stack depth is at most lg n even when the time is not.

- **The adversary.** McIlroy's *Killer Adversary for Quicksort* (1999) does not need to know your
  pivot rule. It runs your sort with a comparator that decides element values *lazily*, freezing
  whichever element looks like the pivot as late as possible, and the values it ends up with form a
  concrete input. Replay that input and the quicksort goes quadratic. Example 12 runs it against a
  textbook quicksort and against the standard libraries — and the standard libraries survive, though
  not for the reason you might guess: the depth-limit fallback never fires. Their up-front checks for
  existing runs and presorted ranges steer the lazily-decided values into an almost-sorted input, so
  the attack backfires. Example 12 counts it.

- **3-way partition for duplicates.** Split into `< p`, `== p`, `> p` (Dijkstra's Dutch-flag scheme)
  and never recurse into the middle. On input with k distinct values the work drops to about
  **Θ(n log k)**, and an all-equal array is one Θ(n) pass. This is a correctness-of-complexity fix,
  not a micro-optimisation: real data (status codes, booleans, ages) is full of duplicates.

- **Heapsort.** Build a max-heap in place in Θ(n), bottom-up, then swap the root to the end and sift
  down, n − 1 times. It is Θ(n log n) **worst case**, in place, with O(1) extra space — the only one of
  the three with both guarantees. It is not stable and not adaptive (sorted input costs about the
  same), and its sift-down jumps from i to 2i + 1, so on a large array almost every step is a cache
  miss. The heap itself is lesson [20](20-heaps-priority-queues.md); here it is a sorting algorithm.

- **Introsort** (Musser, 1997) is the hybrid that ends the argument: quicksort, but count the
  recursion depth and switch to **heapsort** when it passes c·lg n, and finish small ranges with
  **insertion sort**. Quicksort's speed, heapsort's guarantee, insertion sort's constant factor.

- **What `slices.Sort` actually runs.** `slices.Sort`, `slices.SortFunc`, `sort.Sort` and `sort.Slice`
  all call **pdqsort** — pattern-defeating quicksort, an introsort with three extra tricks.
  `sort.Ints`, `sort.Strings` and `sort.Float64s` simply call `slices.Sort` since Go 1.22. Read
  `src/slices/zsortordered.go`; it is about 130 lines:

  | Piece | What the source does |
  |---|---|
  | Small ranges | insertion sort at **length ≤ 12** (`maxInsertion`) |
  | Pivot | static under 8 elements; **median of 3** for 8–49; **Tukey's ninther** (median of three medians) from 50 |
  | Partition | Hoare-style, two indices; also reports whether the range was already partitioned |
  | Sorted input | if the pivot sample needed no swaps, try `partialInsertionSort`: up to 5 rounds of "find the next out-of-order pair, fix it" (only on ranges ≥ 50); it stops early only if **at most 4** fixes sorted the range, because the 5th fix is never re-checked (example 11 counts it) — **Θ(n)** on sorted input |
  | Reversed input | if the ninther needed every one of its 12 swaps, **reverse the range** and carry on as sorted |
  | Duplicates | if the element just before the range is not less than the pivot, the pivot is the range minimum: partition into `== p` and `> p`, and drop the equal block |
  | Bad split | a split whose smaller side is under length/8 decrements `limit` and calls `breakPatterns`, which swaps three elements near the middle using an xorshift seeded by the **length** — so the sort is deterministic |
  | Guarantee | `limit` starts at `bits.Len(n)`; when it reaches 0, **heapsort** the range — Θ(n log n) worst case |

  It recurses on the smaller side and loops on the larger. It is **not stable**: equal elements may be
  reordered. `slices.Sort` orders NaNs before every other value.

- **Go's stable sorts are a different algorithm.** `slices.SortStableFunc`, `sort.Stable` and
  `sort.SliceStable` insertion-sort blocks of 20 and then merge them with **SymMerge**, an in-place
  merge built from rotations. They allocate nothing; the price is **O(n log n) comparisons and
  O(n log² n) swaps**, as the `sort.Stable` doc says. There is no `slices.SortStable` for ordered
  types — use `slices.SortStableFunc(x, cmp.Compare)`.

- **Quickselect.** To find the k-th smallest, partition once and recurse into **only the side that
  contains k**. With a random pivot the expected work is Θ(n) — about **3.39n comparisons** for the
  median — against Θ(n log n) to sort and index. The worst case is still Θ(n²), and the textbook fix
  is **median of medians** (Blum, Floyd, Pratt, Rivest, Tarjan, 1973): a pivot guaranteed to sit
  between the 30th and 70th percentile, for a Θ(n) worst case and a large constant. Go's standard
  library has **no selection function**: you sort, or you write quickselect yourself.

- **Divide & conquer has a stack cost.** Merge sort's depth is ⌈lg n⌉ on every input. Naive
  quicksort's depth is n − 1 on sorted input. Go survives that because a goroutine stack grows (up to
  1 GB on 64-bit); a Rust thread has a fixed stack, and example 5 shows what that means.

### Go and Rust side by side
- **Two sorts, explicitly named.** `v.sort()` / `sort_by` / `sort_by_key` are **stable** and run
  **driftsort** — a natural-run merge sort that hands unsorted stretches to a stable quicksort. It
  **allocates**: nothing for short slices (a 4 KiB stack buffer), up to `len` for medium ones,
  clamping at `len / 2` for large ones. Go's stable sort allocates nothing and swaps more.
- `v.sort_unstable()` / `sort_unstable_by` run **ipnsort**: if the whole slice is one ascending or
  strictly descending run it finishes in Θ(n) (reversing if needed); otherwise quicksort with a budget
  of **2⌊lg n⌋** partition levels — spent on *every* level, where Go spends its `bits.Len(n)` only on
  bad splits — then **heapsort**. Both Rust sorts insertion-sort a whole slice of **≤ 20** elements, and use
  their own small-sorts on sub-ranges; Go insertion-sorts every range of **≤ 12**.
- ipnsort's partition is **Lomuto, branchless**, for types up to 96 bytes, and Hoare for larger ones —
  the textbook loser wins once the branch is gone. Its pivot is a recursive median-of-3 from n ≥ 64.
- `v.select_nth_unstable(k)` is quickselect in the standard library: introselect, with linear min/max
  scans for k = 0 and k = len−1, insertion sort at ≤ 16, and after 16 rounds a fallback to **median of
  medians** — Θ(n) worst case, no allocation. It returns `(left, &mut kth, right)`.
- `BinaryHeap::into_sorted_vec()` is an in-place heapsort of the heap's own buffer.
- `f64` is not `Ord`, so `v.sort()` on floats does not compile; use `sort_by(f64::total_cmp)`. A
  comparator that is not a total order **may panic** in Rust; Go does not check, and produces some order.

## Complexity Table

| Algorithm | Best | Average | Worst | Extra space | Stable |
|---|---|---|---|---|---|
| Merge step (m + n) | min(m, n) cmp | | m + n − 1 cmp | Θ(m + n) | yes, with `<=` |
| Merge sort, top-down / bottom-up | Θ(n log n) | Θ(n log n) | Θ(n log n) | Θ(n) + O(log n) stack | yes |
| Natural merge sort | **Θ(n)** sorted | Θ(n log n) | Θ(n log n) | Θ(n) | yes |
| Quicksort, fixed pivot | Θ(n log n) | Θ(n log n) | **Θ(n²)** | O(log n) if smaller side first | no |
| Quicksort, random pivot | Θ(n log n) | ≈ 1.39 n lg n cmp | Θ(n²), improbably | O(log n) | no |
| Lomuto on all-equal keys | | | **Θ(n²)** | | |
| 3-way quicksort, k distinct keys | **Θ(n)** all equal | Θ(n log k) | Θ(n²) fixed pivot | O(log n) | no |
| Heapsort | Θ(n log n) | Θ(n log n) | Θ(n log n) | **O(1)** | no |
| Introsort / pdqsort (`slices.Sort`) | **Θ(n)** sorted/reversed | Θ(n log n) | **Θ(n log n)** | O(log n) | no |
| `slices.SortStableFunc` | | Θ(n log n) cmp | Θ(n log² n) swaps | O(log n) stack, no heap | yes |
| Rust `sort` (driftsort) | Θ(n) sorted/reversed | Θ(n log n) | Θ(n log n) | up to n | yes |
| Quickselect | Θ(n) | Θ(n), ≈ 3.39n cmp for the median | Θ(n²) | O(1) iterative | — |
| Median of medians / `select_nth_unstable` | | Θ(n) | **Θ(n)** | O(1) | — |
| External sort, k runs | | Θ(n log n) cmp | | Θ(M) memory + disk | with care |
| Any comparison sort | | | ≥ lg(n!) ≈ n lg n − 1.44n | | |

## Exercises
1. Write `merge(a, b []int) []int` and count comparisons. Find the inputs that hit m + n − 1 and min(m, n). Then make it count inversions, and check it against an O(n²) brute force.
2. Write top-down merge sort with **one** scratch buffer. Count comparisons for n = 2¹⁰ … 2¹⁶ and print them next to n lg n − n + 1 and lg(n!).
3. Change `<=` to `<` in your merge. Sort records by key and show the equal keys coming out in the wrong order. Then show `slices.SortFunc` doing the same on a large enough input.
4. Write bottom-up merge sort that swaps buffers instead of copying back. Then make it natural, and show sorted input costs n − 1 comparisons.
5. Write Lomuto and Hoare partitions. Count swaps on random input, and run both on 10,000 equal keys.
6. Write quicksort with a last-element pivot. Confirm n(n−1)/2 comparisons and depth n − 1 on sorted input. Then make it recurse on the smaller side and measure the depth again.
7. Race six pivot rules (first, last, middle, median-of-3, ninther, random) on six input shapes and tabulate comparisons and maximum depth.
8. Write a 3-way partition. Count comparisons for k = 2, 10, 100 and n distinct values against your 2-way quicksort.
9. Write in-place heapsort. Count comparisons on random, sorted and reversed input, and explain why sorted does not help.
10. Write quickselect with a random pivot. Average its comparisons for the median over many seeds and compare with 3.39n. Then find the k smallest without sorting them.
11. Instrument `slices.SortFunc` with a counting comparator on sorted, reversed, all-equal and random input, for n = 1 … 30 and n = 10⁵. Find the insertion-sort cutoff from the counts alone.
12. Implement McIlroy's adversary. Run it against your quicksort from exercise 6 *with median-of-3*, then against `slices.SortFunc`, and explain the difference from the source.
13. Build introsort: your quicksort, a depth limit of 2⌊lg n⌋, a heapsort fallback, an insertion-sort cutoff. Show the fallback firing on the adversary's input and not on random input.
14. Sort 10⁶ integers with only 10⁴ in memory: write sorted runs to temporary files, then k-way merge them with `container/heap`. Then add replacement selection and count the runs.
15. Race every sort in this lesson against `slices.Sort` at n = 10⁶ on five input shapes. Print comparisons **and** time, and find the algorithm whose two columns disagree.
16. Stretch — package `Sort`, `SortStable` and `Select` with a `cmp` function, prove them against `slices.Sort` on random, adversarial and duplicate-heavy input, check stability with tagged records, and benchmark against the standard library.

## Best Practices & Pitfalls
- **Call `slices.Sort` / `slices.SortFunc`.** The hand-written sorts in this lesson exist so you can read the one you call.
- **Choose stable or unstable on purpose.** `SortFunc` may reorder equal elements; `SortStableFunc` will not. In Rust the stable one is the short name: `sort`.
- **Pitfall — `<` in the merge.** It still sorts, and it is no longer stable. Tests with distinct keys will not catch it.
- **Pitfall — Lomuto on duplicates.** An array of equal keys is Θ(n²). Use Hoare or 3-way.
- **Pitfall — a fixed pivot on input you do not control.** Sorted, reversed and crafted inputs all exist. Who chooses the input is a security question, just as with hashDoS in [10](10-hash-tables.md).
- **Recurse on the smaller side.** It bounds the stack at lg n whatever the pivots do.
- **Allocate the merge buffer once.** Not inside the recursion.
- **Pitfall — Hoare with unsigned indices.** Common variants step `j` to `lo − 1`; with `lo = 0` that is −1, a `usize` underflow panic in Rust.
- **Pitfall — midpoints.** `(lo + hi) / 2` can overflow; use `lo + (hi-lo)/2`, as Go's source does with `uint`.
- **Need one order statistic? Don't sort.** Quickselect in Go, `select_nth_unstable` in Rust.
- **Pitfall — a comparator that is not a strict weak order.** Go does not detect it and returns some order; Rust may panic. Floats need `total_cmp` in Rust and a NaN policy in Go.
- **Don't trust comparison counts as a clock.** Heapsort's sift-down spends two comparisons per level, so it makes roughly twice merge sort's comparisons — yet the time gap is far smaller than the count gap, and how far depends on language and machine. Example 15 measures both.
- **Sort by a key that is expensive to compute?** Precompute it — Rust has `sort_by_cached_key` for exactly this.

## Checklist
- [ ] I can derive the n lg n lower bound for comparison sorting.
- [ ] I can write the merge step, and say what `<=` versus `<` does to it.
- [ ] I can write top-down and bottom-up merge sort with one buffer, and say what a natural merge sort adds.
- [ ] I can write Lomuto and Hoare partitions and say which one is quadratic on equal keys.
- [ ] I can state quicksort's comparison count on sorted input with a last-element pivot, exactly.
- [ ] I can explain why no deterministic pivot rule is safe, and what McIlroy's adversary does.
- [ ] I can write a 3-way partition and say when it is needed.
- [ ] I can write in-place heapsort and explain why it is the fallback, not the default.
- [ ] I can name what each piece of Go's pdqsort is for, and where Rust's sorts differ.
- [ ] I know Go's stable sort allocates nothing, and what it pays instead.
- [ ] I can find the k-th smallest in expected Θ(n), and name the Θ(n) worst-case method.
- [ ] I can sketch an external sort and say what replacement selection buys.

## Resources
- Go's pdqsort, worth reading top to bottom: https://github.com/golang/go/blob/master/src/slices/zsortordered.go
- Orson Peters, "Pattern-defeating Quicksort" (2021): https://arxiv.org/abs/2106.05123
- Rust's driftsort and ipnsort design notes: https://github.com/Voultapher/sort-research-rs/tree/main/writeup
- M. D. McIlroy, "A Killer Adversary for Quicksort", *Software — Practice and Experience* 29(4), 1999.
- J. L. Bentley & M. D. McIlroy, "Engineering a Sort Function", *Software — Practice and Experience* 23(11), 1993 — the 3-way partition and ninther.
- D. R. Musser, "Introspective Sorting and Selection Algorithms", *Software — Practice and Experience* 27(8), 1997.
- A. Alexandrescu, "Fast Deterministic Selection" (SEA 2017) — the median-of-ninthers fallback Rust's `select_nth_unstable` uses.
- Sedgewick & Wayne, *Algorithms* 4th ed., §2.2–2.4 — mergesort, quicksort, heapsort: https://algs4.cs.princeton.edu/20sorting/
- CLRS ch. 2, 6, 7 and 9 — merge sort, heapsort, quicksort, selection in linear time.
- Knuth, TAOCP vol. 3, §5.2.2, §5.2.3, §5.4 — quicksort, heapsort, external sorting and replacement selection.
- Examples: [examples/13-efficient-sorts](examples/13-efficient-sorts/) (16).
- Next: [14 — Linear-Time Sorts](14-linear-sorts.md) — beating lg(n!) by not comparing at all.
