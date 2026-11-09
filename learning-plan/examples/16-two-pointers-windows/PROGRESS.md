# Step 16 — Two Pointers, Sliding Window & Prefix Sums · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Pair sum: two pointers walking inward**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Pair sum: two pointers walking inward
- [ ] 2. Dedupe in place, keeping at most k
- [ ] 3. A fixed window: add one, drop one
- [ ] 4. Prefix sums: a range sum in Θ(1)
- [ ] 5. Prefix sums that overflow, and subtractions that underflow
- [ ] 6. Difference arrays: range add in Θ(1)

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Container with most water: why the shorter wall goes
- [ ] 8. The Dutch national flag, and the one-line bug
- [ ] 9. Minimum window substring
- [ ] 10. The longest window: at most k distinct
- [ ] 11. 2-D prefix sums and inclusion-exclusion

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Counting subarrays: exactly k as a difference
- [ ] 13. 3Sum: two pointers inside a loop
- [ ] 14. Which technique? A monotonicity audit
- [ ] 15. Beyond sums: XOR, remainders, and what has no inverse
- [ ] 16. Capstone: the toolkit, built, proved, measured

## The three-pass build

In `practice/16-two-pointers-windows/`:

- [ ] **Build it** — `PairSum` (converging) and `DedupeAtMost(k)` (read/write), each returning where it stopped, not just the answer
- [ ] **Build it** — `Prefix` on a half-open range with a zero in slot 0, `Prefix2D` with a zero border, and a `Diff` array with range add
- [ ] **Build it** — one `LongestValid` and one `ShortestValid` window loop driven by `add` / `remove` / `valid`, then min-window and at-most-k-distinct on top
- [ ] **Build it** — `Partition3` (Dutch flag) and `ThreeSum` on top of `PairSum`
- [ ] **Prove it** — every array of length ≤ 7 over a small alphabet (zeros, negatives and duplicates included) against a brute-force oracle
- [ ] **Prove it** — random inputs over a **narrow value range**, so ties, zeros and repeated values actually occur
- [ ] **Prove it** — step counts asserted, not just printed: pair sum ≤ n−1 steps, a window ≤ 2n pointer moves, the Dutch flag exactly n iterations
- [ ] **Prove it** — values near the integer limit: ranges whose true sum fits must come out exact, and reversed ranges must not wrap silently
- [ ] **Measure it** — operation counts first (comparisons, element reads, pointer moves), then time
- [ ] **Measure it** — each technique against its brute-force version at three sizes, so the ratio's growth is visible
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What do all three techniques refuse to do? | look at the same candidate twice |
| What does converging pair sum require of its input? | **sorted** order — or a Θ(n log n) sort first |
| The invariant of pair sum? | if any pair sums to the target, both indices lie inside `[lo, hi]` |
| Most steps pair sum can take? | n − 1 |
| The invariant of read/write dedupe? | `a[:write]` is the answer for `a[:read]` |
| To keep at most k copies, compare `a[read]` with…? | `a[write-k]` — k places back in the **output** |
| Container with most water: which wall moves? | the **shorter** one; any narrower pair that keeps it holds no more water |
| Dutch flag: what are the four regions? | `[0,lo)` zeros · `[lo,mid)` ones · `[mid,hi)` unknown · `[hi,n)` twos |
| Dutch flag: after swapping with `hi`, do you advance `mid`? | **no** — the element swapped in is unexamined |
| Why is the Dutch flag exactly n iterations? | every iteration shrinks the unknown region by exactly one |
| Cost of a fixed window by add-one-drop-one? | 2 updates per step, independent of k |
| Which aggregates can slide by subtraction? | those with an inverse: sum, count, XOR — **not** max |
| Longest valid window: when does `left` move? | while the window is **invalid** |
| Shortest valid window: when does `left` move? | while the window is **valid** |
| Why is the window loop not quadratic? | each index enters once and leaves at most once — at most 2n moves |
| Valid windows ending at `right`, for an at-most predicate? | `right − left + 1` |
| Exactly k, in terms of at-most? | `atMost(k) − atMost(k−1)` |
| When is a sliding window the wrong tool? | when validity is not monotone under containment — negatives with a sum, equality predicates |
| What is `P[0]`, and what is `sum(l, r)`? | `0` · `P[r] − P[l]` on a half-open range |
| What does changing one element cost a prefix array? | Θ(n) to repair |
| A difference array's range add? | `D[l] += v; D[r] -= v` — then one prefix sum rebuilds the array |
| The 2-D rectangle formula? | `P[r2][c2] − P[r1][c2] − P[r2][c1] + P[r1][c1]` — the corner was subtracted twice |
| What does Go do when a prefix sum overflows `int`? | wraps, silently — the spec forbids a panic |
| What does Rust do? | panics in debug, wraps with `-O` — and so does `Iterator::sum` |
| Why is a difference of wrapped prefixes still right? | wrapping is arithmetic mod 2⁶⁴, so it is exact whenever the true answer fits |
| What happens to `P[r] - P[l]` on `usize` when `l > r`? | underflow: panic in debug, wraps with `-O` — use `checked_sub` |
| Why do negative prefix sums break a mod-m count in Go? | `%` takes the dividend's sign, so residues `-1` and `m-1` land in different buckets |
| Rust's fix? | `rem_euclid` |
| What must an operation have for a prefix array to answer ranges? | an **inverse** — max, min and gcd have none |
| 3Sum in one sentence? | sort, fix `i`, run pair sum on the suffix, skip duplicates — Θ(n²) |

## Numbers to find on your own machine

- [ ] Pair sum when **no** pair exists, for several n: the step count is always n − 1
- [ ] The Dutch flag over every array of length n: the iteration count never varies
- [ ] How many of the 3ⁿ arrays the "advance `mid`" bug breaks — and the shortest input that does
- [ ] The most pointer moves any variable window makes, as a multiple of n
- [ ] The number of queries at which a prefix array starts to beat re-summing each range
- [ ] The first index at which a prefix sum of large values wraps negative
- [ ] The share of random rectangles whose left-to-right 2-D formula dips below zero in unsigned arithmetic
- [ ] How many arrays with negatives a sum-window gets wrong, against the prefix-sum map
- [ ] `Iterator::sum` on `[i64::MAX, 1]` with and without `-O`
- [ ] Which of the audited predicates are window-safe, and which counterexample breaks each of the others

> Lesson: [16-two-pointers-windows.md](../../16-two-pointers-windows.md) · Index: [README.md](README.md)
