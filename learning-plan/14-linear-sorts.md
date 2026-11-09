# 14 — Linear-Time Sorts

> Part of **Part 3 — Sorting & Searching**. [12](12-elementary-sorts.md) and [13](13-efficient-sorts.md)
> sorted by asking one question over and over: *is `a` less than `b`?* This lesson first proves that
> the question has a price — at least **log₂(n!) ≈ n log₂ n − 1.44n** answers, whatever the algorithm —
> and then stops asking it. The one-line thesis: **you beat n log n only by knowing something about
> the keys — their range, their width, their distribution — and every linear-time sort is exactly as
> fast as that knowledge is true.**

## Goals
- Derive the Ω(n log n) lower bound from the decision tree, and say precisely which algorithms it covers.
- Explain the loophole: reading a key's *value* is a k-way branch, not a two-way one.
- Implement **counting sort** for bare keys and for records, and make it **stable** on purpose.
- Say when the `k` in Θ(n + k) ruins it, and shift by the minimum so that `k` is the range, not the maximum.
- Implement **LSD radix sort**, choose the digit width, and prove why every pass must be stable.
- Sort **signed integers and floats** with a radix sort by transforming the bits.
- Implement **MSD radix sort** for strings, and LSD for fixed-width strings.
- Implement **bucket sort**, state its assumption, and break it on purpose.
- Recognise the real keys that fit — IDs, timestamps, fixed-width codes — and the problems linear sorts own.
- Find, by measurement, the n below which `slices.Sort` beats all of them.

## Concepts

- **A comparison sort is a decision tree.** Every run of a comparison sort on n distinct keys is a path
  from the root of a binary tree — each internal node is one `a < b?`, each leaf is one of the **n!**
  possible orderings. Every ordering must have its own leaf, and a binary tree of height h has at most
  2ʰ leaves, so h ≥ ⌈log₂ n!⌉. That is the *worst-case* comparison count of **any** comparison sort.
  Example 1 builds an adversary that forces it out of real algorithms.

- **The numbers.** ⌈log₂ n!⌉ is **22** at n = 10, **29** at n = 12, and about **18.5 million** at
  n = 10⁶. Stirling gives log₂ n! = n log₂ n − n log₂ e + O(log n), so the bound is n log₂ n minus
  about 1.44n — merge sort is within a few percent of optimal. The bound is not always achievable:
  the true minimum for n = 12 is **30**, not 29 (Knuth vol. 3, §5.3.1).

- **What the bound does not cover.** It is a theorem about algorithms whose *only* access to a key is
  a pairwise comparison. Counting sort never compares two keys: it uses a key as an **array index**,
  which is a k-way branch in one step. That is the whole loophole, and its price is that keys must be
  small integers — or mappable to integers *without changing their order*.

- **Counting sort.** Three passes: count each key, turn the counts into positions with a **prefix
  sum**, place each element at its position. Θ(n + k) time, Θ(n + k) space, **zero key comparisons**.
  For bare integers you can skip the placement and rewrite the array from the counts (example 2). For
  records — a key with data attached — you cannot, and the prefix sum is what makes it work (example 3).

- **Stability is chosen, not inherited.** Walk the input **forwards** and place at each key's *start*
  position, incrementing — or walk it **backwards** and place at the *end* position, decrementing
  (CLRS). Mix the two — forwards with end positions — and every group of equal keys comes out
  reversed. The output is still sorted by key, so a test that only checks sortedness passes.

- **The `k` is the whole catch.** Counting sort is linear only when k = O(n). For 32-bit keys k = 2³²:
  over four billion counters, 32 GiB at 8 bytes each, to sort perhaps a thousand numbers. Two
  rules: scan for `min` and `max` first and use **k = max − min + 1** (keys near 10⁹ with a spread of
  1,000 have k = 1,000); and negative keys are just an offset by `min`. Example 4 finds where n + k
  overtakes n log₂ n.

- **LSD radix sort: many small counting sorts.** Split a w-bit key into d = ⌈w/b⌉ digits of b bits and
  counting-sort on each digit, **least significant first**. Θ(d·(n + 2ᵇ)) time, one extra buffer of n.
  For 64-bit keys, 8-bit digits mean **8 passes over 256 counters**; 16-bit digits mean **4 passes over
  65,536**. Bigger digits mean fewer passes but a histogram that stops fitting in cache — the right `b`
  is a measurement, which example 7 makes.

- **Why every LSD pass must be stable.** The invariant: *after pass i, the array is sorted by its low
  i digits.* Pass i + 1 sorts by digit i + 1; for two keys with the same digit i + 1, only stability
  keeps them in the order the previous passes established. Use an unstable pass and the invariant
  breaks at the first tie — example 6 prints the pass at which it fails.

- **The order of the passes matters too.** Most-significant digit first, over the whole array, is
  wrong: the last pass wins and scrambles everything before it. MSD works only by **recursing into
  each bucket separately**, which is a different algorithm.

- **Signed keys.** In two's complement a negative number has its top bit set, so an unsigned radix sort
  puts **every negative after every positive**. Fix: flip the sign bit, `uint64(x) ^ 1<<63`. That maps
  `MinInt64 → 0`, `-1 → 0x7FFF…F`, `0 → 0x8000…0` — an order-preserving bijection onto `uint64`.

- **Floats.** The bit patterns of non-negative IEEE-754 floats already sort like unsigned integers;
  negative ones sort **backwards**. Fix: if the sign bit is set, flip **all** the bits; otherwise flip
  only the sign bit. The result is the IEEE 754 *totalOrder*: −NaN < −∞ < … < **−0 < +0** < … < +∞ <
  +NaN. That is not the order `slices.Sort` uses (see below), and example 8 prints both.

- **Strings, fixed width.** Dates as `YYYY-MM-DD`, airport codes, plates, hex IDs: W bytes each, so LSD
  makes **W passes with R = 256** buckets, right to left. Θ(W·(n + R)).

- **Strings, variable length: MSD.** Counting-sort on the first byte, then recurse into each bucket on
  the next byte. "End of string" must be its own bucket, *below* every byte, so that `"she"` sorts
  before `"shells"`. MSD reads only the **distinguishing prefix** of each string — it never touches a
  byte after the point where a string became unique. Two costs: every recursive call clears and scans
  R + 1 counters, so thousands of tiny buckets cost far more than their contents (cut off to insertion
  sort below a few dozen elements); and an out-of-place MSD needs a buffer of n.

- **Why that matters for strings in particular.** A comparison sort compares *whole keys*: two URLs
  sharing a 30-byte prefix cost 30 byte comparisons every time they meet, and they meet O(log n)
  times. MSD pays for each shared byte about once. Example 14 counts bytes inspected by both.

- **Bucket sort.** Spread n keys from [0, 1) over n equal-width buckets, insertion-sort each bucket,
  concatenate. If the keys are **uniform**, the expected Σ nᵢ² is 2n − 1, so the expected time is Θ(n).
  If they are not — all in one bucket — insertion sort makes it **Θ(n²)**. The assumption *is* the
  algorithm. Bucket occupancy is the same Poisson picture as chaining in [10](10-hash-tables.md): at
  one key per bucket on average, about 1/e of the buckets are empty. Example 10 breaks it with skew.

- **American flag sort: MSD in place.** Count the bucket sizes, then permute elements into their
  buckets by **cycles of swaps** instead of copying out to a buffer. No n-element buffer — only the
  histogram and the recursion — and the price is **stability**. Example 15.

- **Where they show up for real.** Dense **IDs** (a slice indexed by ID *is* a counting sort);
  **timestamps** from one day share their high bytes, so several 8-bit passes see a single bucket and
  can be skipped (example 13); **fixed-width keys** — IPv4 addresses, `YYYYMMDD` dates, ZIP codes — are
  radix sorts waiting to happen. And a family of interview problems where the counting or bucketing
  *is* the answer: maximum gap, top-K by frequency, H-index (example 11).

- **Linear is not the same as fast.** Each LSD pass reads n keys and writes them to 2ᵇ scattered
  places, d times, plus a second n-sized buffer. A pdqsort on contiguous ints is very good at small n.
  There is a crossover, it depends on key width and distribution, and example 12 measures it.

- **What Go ships.** Nothing in this lesson: there is **no counting, radix or bucket sort in Go's
  standard library**. What there is:
  - `slices.Sort` runs **pdqsort** (`pdqsortOrdered` in `slices/zsortordered.go`), falling back to
    insertion sort for ranges of **12** or fewer. It is **not stable**.
  - `sort.Ints`, `sort.Strings` and `sort.Float64s` simply call `slices.Sort` (since Go 1.22).
  - `slices.SortStableFunc` is insertion sort on blocks of **20**, then repeated in-place **SymMerge** —
    no allocation at all; `sort.Stable` documents O(n log n) calls to `Less` and O(n log² n) to `Swap`.
  - For floats, `slices.Sort` orders **NaNs before every other value**, and `cmp.Compare` treats
    **−0.0 and 0.0 as equal**.
  - Strings compare **lexically byte-wise** (the spec), which is exactly the order MSD radix produces.
  - The bit tools: `math.Float64bits` / `Float64frombits`, `math/bits`.

### Go and Rust side by side
- Rust's std has **no radix or counting sort either**; the twins build them, as the Go programs do.
- `slice::sort_unstable` is **ipnsort**: in place, never allocates, O(n log n) worst case, plain
  insertion sort for slices of **20** or fewer. It is the Rust counterpart of `slices.Sort` here.
- `slice::sort` is **driftsort**, stable — and unlike Go's stable sort it **allocates**: a 4 KiB stack
  buffer for short slices, otherwise max(n − n/2, min(n, 8 MB / `size_of::<T>()`)) elements of scratch.
  In Rust an LSD radix sort's n-element buffer costs about what `sort()` already pays; in Go it is extra.
- `f64` is not `Ord`, so `v.sort()` does not compile; you write `v.sort_unstable_by(f64::total_cmp)`.
  And `total_cmp` is **implemented with the radix trick**: it flips every bit but the sign of negative
  values and compares the result as `i64`. So Rust's float order agrees with a radix float sort
  (−0 before +0, NaN placed by its sign); Go's `slices.Sort` does not.
- `str`'s `Ord` is `as_bytes().cmp(...)` — byte-wise, like Go.
- `x as u64` on an `i64` reinterprets the bits, like Go's `uint64(x)`; `u64::to_be_bytes` gives a key
  as big-endian bytes, the form an MSD byte radix sort wants.

## Complexity Table

| Algorithm | Time | Extra space | Stable | Needs |
|---|---|---|---|---|
| Any comparison sort (lower bound) | **Ω(n log n)** worst | — | — | only `<` |
| `slices.Sort` / `sort_unstable` | Θ(n log n) | O(log n) stack, no heap | no | `cmp.Ordered` / `Ord` |
| `slices.SortStableFunc` | O(n log n) compares, O(n log² n) swaps | **none** | yes | a `cmp` func |
| Rust `slice::sort` | O(n log n) | ≈ n/2 to n elements | yes | `Ord` |
| Counting sort | **Θ(n + k)** | Θ(n + k) | yes, if placed in order | integer keys, small range k |
| LSD radix sort, w-bit keys, b-bit digits | **Θ(⌈w/b⌉·(n + 2ᵇ))** | Θ(n + 2ᵇ) | yes | fixed-width keys |
| LSD on fixed-width strings, width W | Θ(W·(n + R)) | Θ(n + R) | yes | equal lengths |
| MSD radix sort, strings | Θ(D) byte reads + Θ(R) per call; D = total distinguishing prefix | Θ(n + depth·R) | yes (out of place) | byte-wise order |
| American flag sort | Θ(D) byte reads + Θ(R) per call | Θ(depth·R), **no n buffer** | **no** | byte-wise order |
| Bucket sort, uniform keys | **Θ(n)** expected | Θ(n) | yes, with insertion sort | uniform distribution |
| Bucket sort, adversarial keys | **Θ(n²)** | Θ(n) | | — the assumption failed |
| Maximum gap (pigeonhole buckets) | Θ(n) | Θ(n) | | integer or float keys |

Exact counts that follow from the algorithms: ⌈log₂ 10!⌉ = **22** · 64-bit keys, 8-bit digits =
**8 passes** · 16-bit digits = **4 passes**, 65,536 counters · counting sort on 32-bit keys = **2³²**
counters · bucket sort on uniform keys: E[Σ nᵢ²] = **2n − 1**.

## Exercises
In `practice/14-linear-sorts/`:
1. Write an adversary that answers each comparison so as to keep the most orderings alive. Run insertion sort and merge sort against it for n = 4 … 8 and confirm both are forced to at least ⌈log₂ n!⌉.
2. Write counting sort for `[]uint8`. Count every array read and write, and confirm it makes zero key comparisons.
3. Write key-indexed counting for records. Then write the forwards-with-end-positions version, and write the test that catches it — a sortedness check will not.
4. Generalise counting sort to `[]int` with a `min`/`max` pass. Find the range k at which it loses to `slices.Sort` for n = 10,000.
5. Write LSD radix sort for `[]uint32` with 8-bit digits. Print the array after every pass on a small input.
6. Replace one pass of your LSD sort with an unstable one. Check the invariant "sorted by the low i digits" after every pass and report where it first fails.
7. Parameterise the digit width b. Count operations for b = 1 … 16, then time b = 4, 8, 11, 16, and explain the winner.
8. Sort `[]int64` containing `MinInt64`, `-1`, `0`, `MaxInt64`. Then sort `[]float64` with ±0, ±Inf, subnormals and NaNs, and compare your order with `slices.Sort`'s.
9. Write LSD for fixed-width strings and MSD for variable-length strings. Make `"she"` sort before `"shells"`, then add the insertion-sort cutoff and count how many counter slots it saves.
10. Write bucket sort for `[0, 1)`. Measure in-bucket comparisons on uniform input, then on `x⁸`, and explain the gap.
11. Solve maximum gap in Θ(n) with pigeonhole buckets, top-K frequent with frequency buckets, and H-index with a capped count. Verify each against a sort-based brute force.
12. Race your radix sort against `slices.Sort` from n = 16 to 4,000,000. Find the crossover, and find it again for keys below 1,024 with counting sort.
13. Sort one day of nanosecond timestamps. Compute all eight histograms in one read pass, skip every pass whose digit is constant, and count the passes you skipped.
14. Sort URLs with a long shared prefix. Count the bytes a comparison sort inspects against the bytes MSD inspects, then time both.
15. Write American flag sort. Show that it makes no n-sized allocation, and show the input where it is not stable.
16. Stretch — package `radix` with `SortUint64`, `SortInt64`, `SortFloat64`, `SortStrings` and a stable `SortByKey`, and apply all three passes: an oracle against the stdlib (and a stable oracle for `SortByKey`), edge cases (empty, one element, extremes, NaN, ±0), a narrow-range fuzz loop, and a benchmark that decides when to fall back to `slices.Sort`.

## Best Practices & Pitfalls
- **Default to `slices.Sort`.** Reach for a linear sort only when you know the keys and have measured.
- **Pitfall — believing "linear beats n log n".** Below the crossover it loses, and the crossover is not small.
- **Pitfall — counting sort on a wide range.** k is the range. Scan for `min` and `max` first; refuse when k ≫ n.
- **Pitfall — the reversed-ties bug.** Forwards with end positions sorts correctly and destroys stability. Test stability with records, not bare ints.
- **Never use an unstable pass inside LSD.** Stability is not a nicety there; it is the proof.
- **Pitfall — MSD-first passes over the whole array.** Only LSD can make whole-array passes. MSD must recurse.
- **Pitfall — radix-sorting `int64` as `uint64`.** Every negative lands after every positive. Flip the sign bit.
- **Pitfall — radix-sorting floats by their raw bits.** Negatives come out backwards. Flip all bits of negatives.
- **Know which float order you want.** Radix (and Rust's `total_cmp`) gives −0 < +0 and sign-placed NaNs; Go's `slices.Sort` gives NaNs first and −0 = +0.
- **Pitfall — MSD without a cutoff.** Every tiny bucket pays for R + 1 counters. Hand small buckets to insertion sort.
- **Pitfall — bucket sort on data you have not looked at.** Skew turns it quadratic, silently.
- **Skip the passes you do not need.** Compute every digit's histogram in one read; a digit with one non-empty bucket is a pass you can skip.
- **Allocate the buffer once** and ping-pong between it and the input; an even number of passes ends back in the input.
- **Prove a sort against the stdlib**, on narrow value ranges so that ties happen, before you benchmark it.

## Checklist
- [ ] I can draw the decision tree and derive ⌈log₂ n!⌉ from it.
- [ ] I can say which algorithms the Ω(n log n) bound covers, and why counting sort escapes it.
- [ ] I can write counting sort for records, stable, and name the bug that reverses ties.
- [ ] I know that k is the range, and I can say when k ruins counting sort.
- [ ] I can write LSD radix sort and state its invariant.
- [ ] I can explain why each pass must be stable, and why the passes go least-significant first.
- [ ] I can choose a digit width and say what is traded.
- [ ] I can radix-sort `int64` and `float64`, and state where NaN and −0 end up.
- [ ] I can write MSD radix sort for strings, with the end-of-string bucket and a cutoff.
- [ ] I can state bucket sort's assumption and construct the input that breaks it.
- [ ] I can solve maximum gap in Θ(n) and say why the answer is never inside a bucket.
- [ ] I know that neither Go nor Rust ships a radix sort, and what each stdlib sort actually runs.
- [ ] I can name the n at which radix sort starts to beat `slices.Sort` on my machine.

## Resources
- CLRS ch. 8, *Sorting in Linear Time* — the decision-tree bound, counting, radix and bucket sort.
- Sedgewick & Wayne, *Algorithms* (4th ed.), §5.1 *String Sorts* — key-indexed counting, LSD, MSD, with the end-of-string convention used here: https://algs4.cs.princeton.edu/51radix/
- McIlroy, Bostic & McIlroy, "Engineering Radix Sort" (1993) — American flag sort.
- Knuth, TAOCP vol. 3, §5.3.1 — minimum-comparison sorting, and why n = 12 needs 30, not 29.
- Go's sorts: https://github.com/golang/go/blob/master/src/slices/zsortordered.go
- Rust's sorts (ipnsort, driftsort): https://github.com/rust-lang/rust/tree/master/library/core/src/slice/sort
- `f64::total_cmp`, whose source is the float radix transform: https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp
- Examples: [examples/14-linear-sorts](examples/14-linear-sorts/) (16).
- Previous: [13 — Efficient Sorts: Divide & Conquer](13-efficient-sorts.md) · Next: [15 — Binary Search & Its Variants](15-binary-search.md).
