# Step 15 — Binary Search & Its Variants · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The invariant, written down**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The invariant, written down
- [ ] 2. The overflow in `(lo + hi) / 2`
- [ ] 3. Lower bound and upper bound
- [ ] 4. Which duplicate do you get?
- [ ] 5. `sort.Search` is a predicate search
- [ ] 6. Three-way search is not faster

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Searching a rotated array
- [ ] 8. Binary search on the answer: Koko's bananas
- [ ] 9. Ship packing and "minimize the maximum"
- [ ] 10. Two kinds of sorted matrix
- [ ] 11. Finding a peak in unsorted data

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Every boundary bug, enumerated
- [ ] 13. Bisection on floats, and on their bits
- [ ] 14. Faster than the textbook: branchless and Eytzinger
- [ ] 15. Exponential search and lopsided intersections
- [ ] 16. Capstone: a timestamp index, built, proved, measured

## The three-pass build

In `practice/15-binary-search/`:

- [ ] **Build it** — `LowerBound` / `UpperBound` on the half-open template, overflow-safe `mid`
- [ ] **Build it** — `EqualRange`, `Count`, `Contains` and a stable `Insert` on top of the two bounds
- [ ] **Build it** — `FirstTrue(lo, hi int64, p)` over a value range that may reach `math.MaxInt64`
- [ ] **Build it** — rotated-array search, Koko, ship packing, matrix search and peak finding, each through `FirstTrue`
- [ ] **Prove it** — an exhaustive oracle: every sorted array of length ≤ 6 over 3 values, every target
- [ ] **Prove it** — random arrays over a **narrow value range**, so duplicates actually occur
- [ ] **Prove it** — the invariant asserted on every iteration, and the probe count bounded by ⌈log₂(n+1)⌉
- [ ] **Prove it** — search-on-the-answer results checked against a brute-force scan of the range
- [ ] **Measure it** — probes and comparisons, not just time
- [ ] **Measure it** — your bound against `slices.BinarySearch`, at sizes in cache and out of it
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What does binary search actually find? | the **first index where a monotone predicate becomes true** |
| The two promises of the half-open template? | everything left of `lo` is false; `hi` is true (with `p(n)` = true) |
| Why does it terminate? | both updates strictly shrink `[lo, hi)` |
| Probes for a lower bound over n elements? | ⌊log₂(n+1)⌋ or ⌈log₂(n+1)⌉ — never fewer |
| Why is `(lo + hi) / 2` a bug? | the sum overflows; `mid` goes negative |
| What does Go do on signed overflow? | wraps, silently — the spec forbids a panic |
| What does Rust do? | panics in debug, wraps in release (`-O`), unless `-C overflow-checks=on` |
| Two safe midpoints? | `lo + (hi-lo)/2` · `int(uint(lo+hi) >> 1)` (Go) · `usize::midpoint` (Rust) |
| Lower vs upper bound? | first `a[i] >= x` · first `a[i] > x` |
| Count of `x` in a sorted slice? | `upper - lower` |
| What does `sort.Search` return when nothing matches? | `n`, not `-1` |
| Which duplicate does `slices.BinarySearch` return? | the **first** — documented |
| Which does Rust's `binary_search` return? | **unspecified** — use `partition_point` for a defined one |
| `partition_point` returns the first index where the predicate is…? | **false** |
| Why does `Vec<f64>` not have `binary_search`? | it needs `T: Ord`; use `binary_search_by` + `total_cmp` |
| Rotated array: what do you compare `a[mid]` with? | `a[hi]` |
| When does rotated search become Θ(n)? | duplicates, when `a[mid] == a[hi]` |
| The tell for binary search on the answer? | "minimize the maximum" / "smallest X such that it is feasible" |
| Ship packing: lowest capacity worth trying? | `max(w)`, not 1 |
| Why can float bisection loop forever? | `eps` below the float spacing: `mid` stops moving |
| Two kinds of sorted matrix, and their algorithms? | row-major → one search Θ(log mn) · rows & columns → staircase Θ(m+n) |
| Why does `lo = mid` loop forever? | with mid rounding down and `hi = lo+1`, `mid == lo` |

## Numbers to find on your own machine

- [ ] The probe count of a lower bound over n = 1000, for every target: only two distinct values
- [ ] The Rust `binary_search` comparator call count — the same for every target
- [ ] Which index each language returns for 9 equal elements
- [ ] Comparisons per search, three-way vs two-way, successful and unsuccessful
- [ ] How many of the 32 boundary variants are correct, and how many never terminate
- [ ] The size at which a branchless or Eytzinger search starts beating the stdlib — if it does
- [ ] Comparisons for a 100 × 1,000,000 intersection: merge, binary search, galloping

> Lesson: [15-binary-search.md](../../15-binary-search.md) · Index: [README.md](README.md)
