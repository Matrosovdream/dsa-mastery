# 12 — Elementary Sorts

> Part of **Part 3 — Sorting & Searching**, and the start of it. Bubble, selection and insertion sort
> are Θ(n²), and every textbook says so on page one and moves on. Do not move on yet. One of the three
> runs inside `slices.Sort` and inside both of Rust's slice sorts, and the reason why is the whole
> lesson. The one-line thesis: **an elementary sort is judged by what it does on the input you
> actually have, and insertion sort is the only one of the three whose cost is the *disorder* of
> the input rather than its *size*.**

## Goals
- Write bubble, selection and insertion sort from their invariants, not from memory.
- Count **comparisons** and **moves** separately, and say which one each algorithm is careful with.
- Define an **inversion**, and state exactly what insertion sort pays per inversion.
- Prove why any sort that only swaps neighbours is Θ(n²) on average, and name the two ways out.
- Define **stability**, say which of the three sorts have it, and write the one-character bug that removes it.
- Explain why stability is a **correctness** property: multi-key sorts, deterministic output, ties.
- State the comparator contract (strict weak ordering) and three ways real comparators break it.
- Say precisely where Go's and Rust's standard sorts hand over to insertion sort, with the constants from the source.
- Know when insertion sort beats an O(n log n) sort on a *large* input, and why.

## Concepts

- **The cost model.** A comparison sort learns about the data only through `less(a, b)`. Count two
  things, and keep them apart: **comparisons** (expensive when keys are strings or a `cmp` function
  does real work) and **moves** (expensive when elements are large records). The three elementary
  sorts make different trades between the two, and that trade is the only reason to know more than one.

- **Bubble sort** walks the slice swapping adjacent out-of-order pairs. Invariant: after pass `k` the
  `k` largest elements are in their final places at the end. With an **early-exit flag** (stop after a
  pass with no swaps) it costs n−1 comparisons on sorted input; without it (but with the usual
  shrinking bound), always n(n−1)/2. Its weakness has a name: **turtles**. A large element near the front moves to the end in one pass (a
  rabbit); a small element near the end moves left only **one place per pass**, so `[2 3 … n 1]`
  needs n−1 passes although it is one rotation from sorted.

- **Selection sort** finds the minimum of the unsorted suffix and swaps it to the front. Invariant:
  the prefix `a[:i]` holds the `i` smallest elements, sorted, **in their final places**. It makes
  exactly n(n−1)/2 comparisons on *every* input — sorted, reversed, all equal — so it is **not
  adaptive** at all. Its one virtue: at most **n−1 swaps**, the fewest writes of any of the three.
  It is **not stable**: the long-distance swap can jump an element over its equal twin —
  `[2a 2b 1]` becomes `[1 2b 2a]` after the first swap.

- **Insertion sort** grows a sorted prefix: take `a[i]`, shift every larger element of `a[:i]` one
  place right, drop `a[i]` into the gap. Invariant: `a[:i]` is sorted and is a permutation of the
  original `a[:i]` (it is *not* final — later elements can still land inside it). **Shift, don't
  swap**: holding the element in a variable and shifting costs one write per step; a swap costs
  two writes plus a temporary. The comparison sequence is identical, so this changes moves, not comparisons.

- **Inversions.** A pair `i < j` with `a[i] > a[j]`. Sorted input has 0; reversed has n(n−1)/2; a
  random permutation has **n(n−1)/4 on average**. Insertion sort's shifts equal the inversion count
  **exactly**, and its comparisons lie between I and I + n − 1 (one extra per element that does not
  travel all the way to the front). So its running time is **Θ(n + I)**: the disorder, not the size.
  Bubble sort's swaps also equal I exactly; selection sort ignores I entirely.

- **The adjacent-swap barrier.** Swapping an out-of-order pair of *neighbours* removes exactly one
  inversion. A random
  input has Θ(n²) inversions, so **every algorithm that only moves elements one place at a time is
  Θ(n²) on average**, however cleverly it chooses which neighbours to swap. Two ways out: move
  elements **far** in one step (Shell sort, below), or stop comparing neighbours altogether (merge
  sort and quicksort, [lesson 13](13-efficient-sorts.md)).

- **Adaptive** means the cost falls as the input gets closer to sorted. If every element is at most
  `k` places from where it belongs, there are fewer than 2nk inversions, so insertion sort is
  **O(nk)** — linear for fixed `k`. That covers real workloads: a sorted log with a few late
  arrivals, a sorted slice with a handful of appends, a list re-sorted after one change. Example 8
  measures, at n = 1,000,000, up to which `k` it still beats `slices.Sort`.

- **Online.** Insertion sort never looks ahead, so it can sort a stream as it arrives — each new
  element costs O(its inversions).

- **Binary insertion sort** finds the slot by binary search: comparisons fall to Θ(n log n), moves
  stay Θ(n²). Worth it when a comparison costs far more than a move. It is stable only if the search
  returns the position **after** the last equal key (upper bound). `slices.BinarySearch` returns the
  *earliest* position, which is the wrong bound for this.

- **Shell sort** is insertion sort run on elements `h` apart, for a shrinking sequence of gaps
  ending in 1. The early, wide passes move elements far, which is how it breaks the adjacent-swap
  barrier. With Knuth's gaps (1, 4, 13, 40, …, `h = 3h+1`) the worst case is O(n^{3/2}). It is in
  place but **not stable** — the gapped passes reorder equal keys.

- **Stability** means equal keys come out in the order they went in. It is invisible when you sort
  plain integers and decisive when you sort records by *part* of the record. It is a **correctness**
  property, for three reasons:
  1. **Multi-key sorts by successive passes.** Sort by name, then *stably* by department, and you
     get departments with names alphabetical inside each. An unstable second pass destroys the
     first. This is what clicking two column headers in a spreadsheet does.
  2. **Determinism.** A stable sort's output is fully determined by the input. An unstable sort's
     tie order is implementation-defined: it can differ between Go and Rust, between library
     versions, and between n = 12 and n = 13 — example 12 shows exactly where.
  3. **Ties are the normal case** for real keys (status, date, score), not an edge case.

  The fix that needs no stability: make the comparator **total** by adding tie-breakers until no two
  distinct elements compare equal (`cmp.Or` in Go, `Ordering::then_with` in Rust). Then any sort is
  deterministic.

- **Which of the three are stable.** Bubble (if it swaps only on strict `>`) and insertion (if it
  shifts only on strict `<`) are stable; selection is not. In both stable ones, changing `<` to `<=`
  keeps the output sorted and silently makes it unstable — the one-character bug of this lesson.

- **The comparator contract.** `less` must be a **strict weak ordering**: irreflexive (`less(a, a)`
  is false), transitive, and "neither is less" must itself be transitive. Three common breaks:
  `<=` instead of `<`; `return a - b` in a Go `cmp` function, which **overflows** for large values
  and returns the wrong sign; and floating-point `NaN`, which is "equal" to everything under `<`,
  so incomparability stops being transitive and the output need not even be sorted.

- **Why insertion sort is the one that matters.** In place, stable, adaptive, online, and an inner
  loop of a compare, a move and a decrement — no recursion, no pivot, no buffer, almost no code. So
  fast general-purpose sorts are hybrids that **hand small inputs to it** (Rust's source gives its
  reason for the ≤ 20 rule as small code size and instruction-cache footprint). [Lesson 01, example 11](examples/01-introduction/3-hard.md#11-big-o-lies-at-small-n)
  timed that crossover on random input; this lesson counts it instead.

- **What Go's standard library ships** (Go 1.27, checked in `$(go env GOROOT)/src`):
  - `slices.Sort` and `slices.SortFunc` are **pdqsort**. Any sub-range of **length ≤ 12** is sorted
    by insertion sort (`const maxInsertion = 12` in `pdqsortOrdered`, `src/slices/zsortordered.go`),
    at **every** level of the recursion, not only at the top. `sort.Sort`, `sort.Slice` and
    `sort.Ints`/`Strings`/`Float64s` (which now just call `slices.Sort`) run the same code with the
    same constant (`src/sort/zsortinterface.go`, `zsortfunc.go`).
  - The stdlib's insertion sort is the **swap** version:
    `for j := i; j > a && cmp.Less(data[j], data[j-1]); j-- { swap }`.
  - When pivot sampling suggests the range is already ascending, pdqsort tries
    `partialInsertionSort`: it runs **5** rounds (`maxSteps = 5`), each round first scanning for the next
    out-of-order pair (returning "sorted" if it reaches the end) and then fixing it, shifting only on
    ranges of length ≥ 50. The check happens at the *start* of a round, so the 5th fix is never
    verified: it reports success only when **at most 4** pairs needed fixing (lesson 13 example 11
    counts the jump at 5).
  - `slices.SortStableFunc`, `sort.Stable` and `sort.SliceStable` insertion-sort **blocks of 20**
    (`blockSize := 20`) and then merge them **in place** with SymMerge — no scratch buffer;
    `sort.Stable`'s doc promises O(n log n) calls to `Less` and O(n log² n) to `Swap`. So in Go, a
    stable sort of ≤ 20 elements *is* insertion sort.
  - There is `slices.SortStableFunc` but **no `slices.SortStable`**.
  - Consequence: `slices.SortFunc` is stable for n ≤ 12 by accident. A test fixture of ten records
    cannot catch a missing-stability bug.
  - `slices.Sort` on floats orders **NaNs before** every other value (its doc says so);
    `slices.SortFunc` documents that `cmp` must be a strict weak ordering.
  - `slices.IsSorted` / `IsSortedFunc` check a result in one pass. `slices.MinFunc` returns the
    **first** minimum, which is what a stable selection step needs.

### Go and Rust side by side
- **Stability is in the name:** `sort`/`sort_by`/`sort_by_key` are stable (driftsort), `sort_unstable*` are not (ipnsort).
- **Both** insertion-sort any slice of **length ≤ 20** (`MAX_LEN_ALWAYS_INSERTION_SORT` in `core/src/slice/sort/{stable,unstable}/mod.rs`),
  **shifting through a hole** (`insert_tail`, `shared/smallsort.rs`) where Go's version swaps.
- **Inside the recursion** Rust's small-sort (threshold 16 or 32) is sorting networks for small `Copy` types in
  `sort_unstable`, otherwise sort4/sort8-plus-merge; plain insertion sort is only the fallback. Go's is plain insertion sort at every level.
- Above the cutoff both detect an ascending or **strictly descending** input in one pass (`find_existing_run`).
- `sort` **allocates** scratch (none for short slices, then up to `len`, clamped toward `len/2`, per its doc); Go's stable sort merges in place.
- `f64` is not `Ord`: `v.sort()` on floats **does not compile** — use `sort_by(f64::total_cmp)`. Rust's sorts
  **may panic** on a non-total comparator (documented); Go's docs state the requirement and nothing more.
- `binary_search` returns *any* match; `partition_point(|y| y <= x)` is the stable upper bound. `min_by_key` returns the first minimum, `max_by_key` the **last**.

## Complexity Table

| Algorithm | Best | Average | Worst | Comparisons | Moves | Stable | In place |
|---|---|---|---|---|---|---|---|
| Bubble, early exit | Θ(n) | Θ(n²) | Θ(n²) | n−1 … n(n−1)/2 | swaps = **I** | yes (strict `>`) | yes |
| Selection | Θ(n²) | Θ(n²) | Θ(n²) | **n(n−1)/2 always** | **≤ n−1 swaps** | **no** | yes |
| Insertion | **Θ(n)** | Θ(n²) | Θ(n²) | **I … I + n − 1** | shifts = **I** | yes (strict `<`) | yes |
| Insertion, each element ≤ k from home | O(nk) | | | | | | |
| Binary insertion | Θ(n log n) | Θ(n²) | Θ(n²) | **Θ(n log n)** | Θ(n²) | only with upper bound | yes |
| Shell sort, Knuth gaps | | | O(n^{3/2}) | | | **no** | yes |
| Any adjacent-swap sort | | **Ω(n²)** | | | ≥ I | | |
| Inversions, random permutation | | **n(n−1)/4** | n(n−1)/2 | | | | |

I = number of inversions. Reversed input: I = n(n−1)/2. Sorted input: I = 0.

Standard-library hand-off points (from source): Go `slices.Sort`/`SortFunc`/`sort.Sort` — every
sub-range **≤ 12** · Go `SortStableFunc`/`sort.Stable` — blocks of **20** · Rust `sort` and
`sort_unstable` — whole slice **≤ 20**.

## Exercises
In `practice/12-elementary-sorts/`, each with a test and a benchmark:
1. Implement bubble sort with an early-exit flag and a shrinking bound. Count passes on `[2 3 … n 1]` and on `[n 1 2 … n-1]`, and explain the difference.
2. Implement selection sort. Show that its comparison count is identical on sorted, reversed and all-equal input, and that its swaps never exceed n−1.
3. Implement insertion sort twice — swapping and shifting. Prove the comparison counts are equal and count the writes of each.
4. Write an invariant checker for each of the three sorts and call it after every outer iteration. Introduce an off-by-one and report the first step at which the invariant fails.
5. Sort `(key, tag)` pairs with each sort and report which ones keep equal keys in input order. Then change `<` to `<=` and find the smallest input that exposes it.
6. Count inversions by brute force. Assert, over 10,000 random inputs, that insertion sort's shifts equal the inversion count and its comparisons lie in `[I, I+n-1]`.
7. Run all three sorts on sorted, reversed, random, nearly sorted, three-distinct-values and organ-pipe input. Tabulate comparisons and writes, then predict the ranking before you look.
8. Generate k-sorted input and find, for several k, the largest n at which your insertion sort still beats `slices.Sort`.
9. Implement binary insertion sort with `sort.Search`. Use the lower bound first and find the input that breaks stability; then fix it. Compare comparison counts on strings with long common prefixes.
10. Sort records by name, then stably by department. Do the second pass with `slices.SortFunc` instead and find the smallest n where the result goes wrong. Then write the single comparator with `cmp.Or` that makes the stable pass unnecessary.
11. Break the comparator contract three ways (`<=`, `a - b` with large ints, `NaN`) and verify the result with `slices.IsSortedFunc` and a stability check.
12. Count the comparator calls `slices.SortFunc` and `slices.SortStableFunc` make, for n = 2…40, and find the n at which each stops matching your insertion sort. Then find the constants in the Go source.
13. Implement Shell sort with three gap sequences and count comparisons at n = 10³, 10⁴, 10⁵. Show it is not stable.
14. Sort large records (hundreds of bytes) with selection and insertion sort, then sort a slice of indices instead. Explain the ranking.
15. Stretch — package a generic `InsertionSortFunc` and apply all three passes: a brute-force oracle against `slices.SortStableFunc` on a **narrow key range** (so ties occur), the inversion identities as a property, and a benchmark against both stdlib sorts at small n and on nearly sorted large n.

## Best Practices & Pitfalls
- **Call the standard library.** `slices.SortFunc` when ties do not matter, `slices.SortStableFunc` when they do. The hand-written sorts here exist so you know what those calls do on small and nearly sorted input.
- **Decide stability on purpose.** If two elements can compare equal and you can tell them apart, you need either a stable sort or a tie-breaker.
- **Prefer a total comparator to a stable sort** when you control the comparator: `cmp.Or(cmp.Compare(a.Dept, b.Dept), strings.Compare(a.Name, b.Name))`.
- **Pitfall — testing stability on ten elements.** `slices.SortFunc` is insertion sort below 13 and Rust's `sort_unstable` below 21. Test with more elements than the cutoff, and with **many ties**.
- **Pitfall — `<=` in a less function.** Still sorted, no longer stable, and outside the documented contract.
- **Pitfall — `return a - b` as a Go `cmp` function.** It overflows. Use `cmp.Compare(a, b)`.
- **Pitfall — floats with NaN under `a < b`.** Use `slices.Sort` (NaNs first) or `cmp.Compare`; in Rust, `total_cmp`.
- **Pitfall — believing "selection sort is adaptive".** It does the same n(n−1)/2 comparisons on sorted input.
- **Shift, don't swap,** in insertion sort: one write per step instead of three.
- **Reach for insertion sort on purpose** when the data is known to be nearly sorted, or small and hot.
- **Pitfall — binary insertion with a lower-bound search.** It inserts before equal keys and breaks stability.
- **Sort indices, not records,** when the records are large and the algorithm moves a lot.
- **Write the invariant before the loop.** Every bug in this lesson is a loop that does not maintain one.

## Checklist
- [ ] I can write all three sorts from their invariants and state each invariant in one sentence.
- [ ] I can give the exact comparison count of selection sort, and of insertion sort on sorted and reversed input.
- [ ] I can define an inversion and say what insertion sort pays per inversion.
- [ ] I can prove that adjacent-swap sorts are Θ(n²) on average, and name the two ways out.
- [ ] I know which of the three are stable and which one-character change breaks it.
- [ ] I can give three reasons stability is a correctness property.
- [ ] I can make any sort deterministic with a total comparator.
- [ ] I can state the strict-weak-ordering contract and break it three ways.
- [ ] I know Go's cutoffs (12 for pdqsort, blocks of 20 for stable) and Rust's (20 for both), and that I read them in the source.
- [ ] I know when insertion sort beats `slices.Sort` on a large input.
- [ ] I can say when selection sort is the right choice, and when sorting indices is better than either.

## Resources
- Knuth, TAOCP vol. 3 — §5.1.1 (inversions), §5.2.1 (insertion sort and Shell sort), §5.2.2 (bubble sort), §5.2.3 (selection).
- CLRS ch. 2.1 — insertion sort and the three-part loop-invariant proof.
- Sedgewick & Wayne, *Algorithms* 4th ed., §2.1 — elementary sorts, with Shell sort.
- Go's pdqsort, with the `maxInsertion` constant: https://github.com/golang/go/blob/master/src/slices/zsortordered.go
- The pdqsort paper (cited in Go's source): https://arxiv.org/pdf/2106.05123.pdf
- Rust's driftsort and ipnsort design notes (cited in Rust's source): https://github.com/Voultapher/sort-research-rs/blob/main/writeup/driftsort_introduction/text.md and https://github.com/Voultapher/sort-research-rs/blob/main/writeup/ipnsort_introduction/text.md
- Ciura, "Best Increments for the Average Case of Shellsort" (2001) — the empirical gap sequence.
- Examples: [examples/12-elementary-sorts](examples/12-elementary-sorts/) (16).
- Next: [13 — Efficient Sorts: Divide & Conquer](13-efficient-sorts.md) — the two ways past the adjacent-swap barrier, and what `slices.Sort` does above n = 12.
