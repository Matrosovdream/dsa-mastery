# Step 13 — Efficient Sorts: Divide & Conquer · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The merge step, and the inversions it counts**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The merge step, and the inversions it counts
- [ ] 2. Top-down merge sort, one buffer
- [ ] 3. Stability is one character
- [ ] 4. Lomuto and Hoare partitions
- [ ] 5. Quicksort, and the input that makes it quadratic
- [ ] 6. Heapsort in place

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Bottom-up and natural merge sort
- [ ] 8. Six pivot rules, six input shapes
- [ ] 9. Three-way partition for duplicates
- [ ] 10. Quickselect: the k-th smallest without sorting
- [ ] 11. What `slices.Sort` actually runs

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. McIlroy's killer adversary
- [ ] 13. Introsort, then the pdqsort tricks
- [ ] 14. External sort: runs on disk and a k-way merge
- [ ] 15. Comparisons are not time: the full race
- [ ] 16. Capstone: `Sort`, `SortStable`, `Select`, built, proved, measured

## The three-pass build

In `practice/13-efficient-sorts/`:

- [ ] **Build it** — `MergeSort`: top-down, one scratch buffer allocated once, `<=` in the merge
- [ ] **Build it** — `QuickSort`: Hoare or 3-way partition, ninther pivot, recurse on the smaller side
- [ ] **Build it** — `HeapSort` in place, and an introsort that falls back to it at depth 2⌊lg n⌋
- [ ] **Build it** — `Select(xs, k)`: quickselect, iterative, no allocation
- [ ] **Prove it** — oracle against `slices.Sort` on random, sorted, reversed, organ-pipe and all-equal input
- [ ] **Prove it** — a **narrow value range** so duplicates actually occur
- [ ] **Prove it** — stability: tagged records with equal keys, compared against `slices.SortStableFunc`
- [ ] **Prove it** — the output is a permutation of the input, not just sorted
- [ ] **Prove it** — McIlroy's adversary: assert comparisons stay under c · n lg n
- [ ] **Measure it** — comparisons *and* time against `slices.Sort` and `slices.SortStableFunc`
- [ ] **Measure it** — allocations per sort, and maximum recursion depth
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| The comparison-sort lower bound? | lg(n!) ≈ n lg n − 1.44n |
| Comparisons to merge runs of m and n? | at most m + n − 1, at least min(m, n) |
| What makes the merge stable? | take from the **left** on ties: `<=`, not `<` |
| Merge sort's real cost? | Θ(n) extra space |
| What does a natural merge sort do with sorted input? | n − 1 comparisons, no merges |
| Lomuto vs Hoare on all-equal keys? | Lomuto Θ(n²); Hoare splits in the middle |
| Last-element pivot on sorted input? | exactly n(n−1)/2 comparisons, depth n − 1 |
| Random-pivot quicksort, expected comparisons? | ≈ 2n ln n ≈ 1.39 n lg n |
| How do you bound quicksort's stack? | recurse on the smaller side, loop on the larger |
| What does McIlroy's adversary need to know? | nothing about the pivot rule |
| Work of 3-way quicksort on k distinct keys? | about Θ(n log k) |
| Heapsort's two guarantees? | Θ(n log n) worst case **and** O(1) extra space |
| Why is heapsort not the default? | not adaptive, not stable, cache-hostile |
| What does `slices.Sort` run? | **pdqsort**: introsort + sorted/reversed detection + equal-partition + pattern breaking |
| Go's insertion-sort cutoff? Rust's? | ≤ 12 per range · ≤ 20 for a whole slice |
| When does Go's pdqsort switch to heapsort? | after `bits.Len(n)` unbalanced partitions |
| What does Go's stable sort do instead of allocating? | blocks of 20 + SymMerge: O(n log² n) swaps |
| Rust `sort` vs `sort_unstable`? | driftsort (stable, allocates) · ipnsort (unstable, in place) |
| Quickselect for the median, expected comparisons? | ≈ 3.39n |
| Θ(n) worst-case selection? | median of medians — Rust's `select_nth_unstable` fallback |
| Go's selection function? | there is none |
| External sort in one line? | sorted runs to disk, then a k-way heap merge |
| What does replacement selection buy? | runs of ≈ 2M instead of M |

## Numbers to find on your own machine

Comparison counts are the same everywhere; these are the ones that depend on your hardware.

- [ ] The n where your top-down merge sort stops losing to `slices.Sort` by a constant factor — or the fact that it never does
- [ ] How much slower heapsort is than merge sort at n = 10⁶, with almost the same comparison count
- [ ] The time a textbook quicksort takes on McIlroy's input at n = 10⁵, against `slices.SortFunc` on the same input
- [ ] Whether Lomuto or Hoare is faster on random `int`s — and whether that agrees with their swap counts
- [ ] Quickselect against sort-then-index for the median at n = 10⁷
- [ ] `slices.SortStableFunc` against `slices.SortFunc`: the price of stability without allocation

> Lesson: [13-efficient-sorts.md](../../13-efficient-sorts.md) · Index: [README.md](README.md)
