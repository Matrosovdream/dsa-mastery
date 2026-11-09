# Step 14 — Linear-Time Sorts · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The decision-tree bound: log2(n!) comparisons**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The decision-tree bound: log2(n!) comparisons
- [ ] 2. Counting sort: count, then rewrite
- [ ] 3. Key-indexed counting: prefix sums and stability
- [ ] 4. The k in O(n + k)
- [ ] 5. LSD radix sort, one digit at a time
- [ ] 6. Why every LSD pass must be stable

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Choosing the radix: bits per pass
- [ ] 8. Signed keys and floats: flip the bits
- [ ] 9. Strings: LSD for fixed width, MSD for the rest
- [ ] 10. Bucket sort and the uniform assumption
- [ ] 11. The problems they own: maximum gap, top-K frequent, H-index

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. The crossover against `slices.Sort`
- [ ] 13. Real keys: timestamps, IDs and the passes you can skip
- [ ] 14. Shared prefixes: MSD radix vs comparison sort
- [ ] 15. American flag sort: MSD radix in place
- [ ] 16. Capstone: `radix`, built, proved, measured

## The three-pass build

In `practice/14-linear-sorts/`:

- [ ] **Build it** — stable counting sort for records, with a `min`/`max` pass so k is the range
- [ ] **Build it** — LSD radix sort for `uint64` with one reusable buffer and a chosen digit width
- [ ] **Build it** — `int64` and `float64` through an order-preserving bit transform
- [ ] **Build it** — MSD radix sort for strings, with an end-of-string bucket and an insertion-sort cutoff
- [ ] **Build it** — one-pass histograms, skipping every pass whose digit is constant
- [ ] **Prove it** — oracle against `slices.Sort` over thousands of random inputs
- [ ] **Prove it** — a **narrow key range**, so ties actually happen
- [ ] **Prove it** — stability: a stable `SortByKey` matches `slices.SortStableFunc` on records
- [ ] **Prove it** — edge cases: empty, one element, `MinInt64`/`MaxInt64`, ±0, ±Inf, NaN
- [ ] **Measure it** — against `slices.Sort` from n = 16 upward; record the crossover
- [ ] **Measure it** — allocations per call, against `slices.Sort`'s zero
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| Lower bound for any comparison sort? | ⌈log₂ n!⌉ ≈ n log₂ n − 1.44n comparisons, worst case |
| Why? | a binary decision tree needs n! leaves, so height ≥ log₂ n! |
| What does counting sort do instead of comparing? | uses the key as an **array index** — a k-way branch |
| Counting sort's cost? | Θ(n + k) time and space, zero comparisons |
| What is k? | the **range**, max − min + 1 — not the largest key |
| What makes counting sort stable? | placing in input order at start positions (or backwards at end positions) |
| The bug that keeps it sorted but unstable? | forwards with **end** positions — ties come out reversed |
| LSD radix sort's cost? | Θ(⌈w/b⌉·(n + 2ᵇ)) |
| LSD's invariant? | after pass i, sorted by the low i digits |
| Why must each pass be stable? | ties on digit i+1 must keep the order passes 1..i built |
| Why least-significant first? | the last pass dominates; it must be the most significant digit |
| `int64` in a `uint64` radix sort? | flip the sign bit |
| `float64` in a radix sort? | negatives: flip all bits; non-negatives: flip the sign bit |
| Which stdlib comparator uses that trick? | Rust's `f64::total_cmp` |
| Where does Go's `slices.Sort` put NaN? | **first** — and it treats −0 = +0 |
| MSD's end-of-string rule? | "ended" is a bucket below every byte |
| Why the MSD cutoff? | every call pays R + 1 counters, however small its bucket |
| Bucket sort's assumption? | keys **uniform** over the range — else Θ(n²) |
| Maximum gap in Θ(n)? | n − 1 pigeonhole buckets; the gap is never inside one |
| Does Go or Rust ship a radix sort? | **neither** |

## Numbers to find on your own machine

- [ ] The n at which LSD radix sort first beats `slices.Sort` on random `uint64`
- [ ] The same crossover for counting sort on keys below 1,024
- [ ] The digit width b that wins at n = 1,000,000 — and whether it is 8
- [ ] How many of eight passes a day of nanosecond timestamps lets you skip
- [ ] Bytes inspected by a comparison sort vs MSD on URLs with a shared prefix
- [ ] Bucket sort's in-bucket comparisons, uniform input vs skewed

> Lesson: [14-linear-sorts.md](../../14-linear-sorts.md) · Index: [README.md](README.md)
