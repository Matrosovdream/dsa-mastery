# 16 — Two Pointers, Sliding Window & Prefix Sums

> Part of **Part 3 — Sorting & Searching**, and its last lesson. Lessons 12–15 bought you order and a way
> to exploit it in Θ(log n); this one is about the problems where the data is not searched at all and
> the cost is a nested loop over pairs or subarrays. The three techniques here have one thing in common:
> they stop **looking at the same candidate twice**. The one-line thesis: **two pointers, a sliding
> window and a prefix array each throw away or reuse an answer you already have, and each is legal only
> when a specific property holds — so the skill is knowing the property, not writing the loop.**

## Goals
- State the invariant of converging two pointers and prove from it that pair sum takes at most n − 1 steps.
- Write the read/write pointer loop, generalised from "no duplicates" to "at most k copies".
- Explain why container-with-most-water moves the **shorter** wall, and why moving the taller is wrong.
- Write the Dutch national flag with its four regions, and say why `mid` does not advance after a swap with `hi`.
- Write fixed and variable sliding windows, and say which variable shape shrinks while the window is *valid* and which while it is *invalid*.
- Argue that a window loop is Θ(n) although it has a loop inside a loop.
- Build a prefix-sum array, a difference array and a 2-D prefix table without an off-by-one.
- State what each technique requires (sorted order, an invertible aggregate, monotone validity) and spot the cue in a problem statement.
- Say what a prefix sum does when it overflows, in Go and in Rust, and why a difference of wrapped prefixes can still be exact.
- Count subarrays with exactly k of something, with a window and with a map, and know which tolerates negative numbers.

## Concepts

- **The shared idea: never evaluate the same candidate twice.** Pairs of indices number n(n−1)/2 and
  subarrays n(n+1)/2; the nested loops that visit them are the Θ(n²) baseline. Each technique finds a
  reason that one step of work retires a whole set of candidates: two pointers retire a row or column of
  the pair matrix per comparison, a window reuses the previous window's aggregate, and a prefix array
  pays Θ(n) once so every later range is a subtraction. The examples count the work (comparisons,
  element reads, pointer moves) instead of timing it, because the count *is* the argument.

- **Two pointers, converging.** On a sorted array, `a[lo] + a[hi]` below the target means `a[lo]` cannot
  pair with anything inside the window (everything it could pair with is already ≤ `a[hi]`), so `lo++`
  retires `hi − lo` pairs at once. The invariant: *if a pair summing to the target exists, both indices
  lie in `[lo, hi]`*. Each step shrinks the gap by one, so the loop takes at most n − 1 steps — exactly
  n − 1 when no pair exists. The test is `lo < hi`, not `<=`: with `<=` an element pairs with itself
  ([example 1](examples/16-two-pointers-windows/1-easy.md)). The price is sorted input; on unsorted
  data the hash set of [lesson 10](10-hash-tables.md) is Θ(n) time and space, and sorting first is
  Θ(n log n).

- **Two pointers, same direction (read/write).** `read` visits every element, `write` marks where the
  next kept element goes, and the invariant is *`a[:write]` is the answer for `a[:read]`*.
  [Lesson 06](06-arrays-slices.md) introduced it as a filter. On a sorted slice, "keep at most k copies"
  is the same loop comparing `a[read]` with `a[write-k]` — k places back in the **output** — and k = 1 is
  dedupe. Equal values must be adjacent: on unsorted input only adjacent repeats go
  ([example 2](examples/16-two-pointers-windows/1-easy.md)). Whatever sits beyond `write` afterwards is
  stale unless you clear it.

- **Container with most water is a discard argument, not a trick.** Walls `i < j` hold
  `(j − i) · min(h[i], h[j])`. From the widest pair, every narrower pair that keeps the **shorter** wall
  is capped by that wall, so the shorter wall can never do better and its pointer moves. Moving the
  taller wall is wrong, and small arrays show it. n − 1 evaluations replace n(n−1)/2
  ([example 7](examples/16-two-pointers-windows/2-medium.md)). Trapping rain water in
  [lesson 08](08-stacks.md) is a different question — it sums water over every position.

- **The Dutch national flag** sorts three classes in one pass with four regions: `[0,lo)` first class,
  `[lo,mid)` second, `[mid,hi)` unknown, `[hi,n)` third. Each iteration moves exactly one of `mid` or
  `hi`, so the unknown region shrinks by exactly one and the loop runs **exactly n times**. The
  one-line bug is advancing `mid` after swapping with `a[hi-1]`: the element that arrived has not been
  examined. [Lesson 06](06-arrays-slices.md) and [lesson 13](13-efficient-sorts.md) used this
  partition; [example 8](examples/16-two-pointers-windows/2-medium.md) proves it against every small input.

- **3Sum is pair sum inside a loop.** Sort, fix `i`, run converging pointers on the suffix, and skip
  equal values at `i` and after each hit so each triple is reported once. Θ(n²) after the sort, against
  n(n−1)(n−2)/6 triples for brute force, with no structure beyond the sort and the output
  ([example 13](examples/16-two-pointers-windows/3-hard.md)).

- **Sliding window, fixed width.** Moving the window right adds one element and drops one: two updates
  per step instead of k, so Θ(n) whatever k is. That needs an aggregate that can be **undone** — sum,
  count and XOR can; **max cannot** (the monotonic deque of [lesson 09](09-queues-deques.md), example 10,
  is the answer). In floating point, add-then-subtract is not exact, so a running sum can drift from a
  recomputed one ([example 3](examples/16-two-pointers-windows/1-easy.md)).

- **Sliding window, variable width — two shapes.** A window `[left, right]` plus state updated on add
  and remove; the validity test must be O(1), so keep a counter and never recount. If a valid window
  stays valid when it **shrinks** (at most k distinct, sum ≤ k over non-negatives, no repeats), you want
  the **longest**: grow `right` every step, shrink `left` while *invalid*, record afterwards. If a valid
  window stays valid when it **grows** (contains every character of T, sum ≥ k over non-negatives), you
  want the **shortest**: grow until valid, shrink while *valid*, record inside the loop.
  [Example 9](examples/16-two-pointers-windows/2-medium.md) is the shortest shape (minimum window
  substring), [example 10](examples/16-two-pointers-windows/2-medium.md) the longest (at most k
  distinct); longest-substring-without-repeats ([lesson 11](11-strings-bytes.md), example 11) is the
  longest shape with the predicate "all distinct".

- **Why a loop inside a loop is still Θ(n).** `left` and `right` only move forward and each stops at n,
  so the whole run is at most 2n moves whatever the input — the aggregate argument of the monotonic
  stack ([lesson 08](08-stacks.md)) and deque ([lesson 09](09-queues-deques.md)) again. A `left` that can
  move backwards loses it (lesson 11 example 11 shows the missing `max`).

- **Counting windows: exactly k = at most k − at most (k−1).** "Exactly k distinct" is not monotone — a
  valid window can become invalid by growing *or* shrinking — but "at most k" is. For a fixed `right`
  every start in `[left, right]` is valid, so one pass adds `right − left + 1` per step; two passes and a
  subtraction give exactly k ([example 12](examples/16-two-pointers-windows/3-hard.md)).

- **When a window is the wrong tool.** The argument needs validity to be monotone under containment.
  A sum with **negative numbers** breaks it (growing the window can lower the sum), and so does every
  "equals k" or "divisible by k" predicate.
  [Example 14](examples/16-two-pointers-windows/3-hard.md) enumerates every small array and tests the
  property directly for ten predicates, then runs the window loop and compares it with brute force.

- **Prefix sums.** `P[0] = 0`, `P[i+1] = P[i] + a[i]`; the sum of the half-open range `a[l:r]` is
  `P[r] − P[l]`. The zero in slot 0 deletes the `l == 0` special case and the half-open range makes the
  empty range free. Θ(n) build, Θ(1) query, Θ(n) space — for **immutable** data, since changing one
  element invalidates every later entry ([example 4](examples/16-two-pointers-windows/1-easy.md)).

- **Prefix sums plus a map.** `a[l:r]` has sum k iff `P[r] − P[l] = k`. Scanning `r` and looking up
  `P[r] − k` among earlier prefixes counts them in Θ(n) for **any** integers; [lesson 10](10-hash-tables.md),
  example 13, built it, including the `counts[0] = 1` seed.
  [Example 12](examples/16-two-pointers-windows/3-hard.md) sets it beside the window, which needs only
  Θ(1) space but is valid only for non-negatives.

- **Difference arrays are the inverse.** `D[l] += v; D[r] -= v` records "add v to `a[l:r]`" in two
  writes, and one prefix sum over `D` rebuilds the array: m updates and one read cost Θ(n + m) instead of
  Θ(Σ lengths). Prefix sum and difference undo each other
  ([example 6](examples/16-two-pointers-windows/1-easy.md)).

- **2-D prefix sums are inclusion-exclusion.** `P[i+1][j+1] = a[i][j] + P[i][j+1] + P[i+1][j] − P[i][j]`:
  the two strips overlap in the corner, which was added twice. The rectangle `[r1,r2) × [c1,c2)` is
  `P[r2][c2] − P[r1][c2] − P[r2][c1] + P[r1][c1]`: the corner was subtracted twice, so it comes back.
  Θ(mn) build, 4 reads per query ([example 11](examples/16-two-pointers-windows/2-medium.md)).

- **Integer limits.** A prefix sum can pass the integer type although every element and every range
  answer fits. Go's signed overflow **wraps** — the spec says it "does not cause a run-time panic" — and
  unsigned arithmetic is modulo 2ⁿ. Wrapping is exact arithmetic mod 2⁶⁴, so `P[r] − P[l]` is still
  **correct whenever the true range sum fits**: the overflow cancels
  ([example 5](examples/16-two-pointers-windows/1-easy.md)). With unsigned prefixes (`uint`, `usize`) the
  subtraction itself underflows when `l > r`, and in 2-D the left-to-right order of the four terms can dip
  below zero midway even when the answer cannot. Choose the behaviour on purpose.

- **A prefix array answers a range only if the operation can be undone.** `+`, XOR and counts have an
  inverse, so `P[r] ^ P[l]` is the XOR of the range; max, min and gcd do not, so a prefix max cannot give
  a range max. Go's `%` takes the sign of the dividend, so "sum divisible by m" over data with negatives
  needs the residue normalised, or equal residues land in different buckets
  ([example 15](examples/16-two-pointers-windows/3-hard.md)).

- **Recognising the technique.** *Sorted + pair or triple with a target* → converging pointers.
  *Contiguous subarray + longest / shortest / at most / at least, validity monotone* → window.
  *Contiguous + "equals k" or "divisible by k", negatives allowed, or many range queries* → prefix sums,
  with a map to count. *Many range updates, one read* → difference array. *Sort items into 2–3 classes in
  place* → flag. *Unsorted pair with a target* → a map ([lesson 10](10-hash-tables.md)); sorting would
  cost more than the pass. The oracle in [example 16](examples/16-two-pointers-windows/3-hard.md) keeps all of them honest.

### Go and Rust side by side

- Go's `slices` ships `Reverse` (converging pointers), `Compact` / `CompactFunc` (read/write — and they
  **zero the tail**), `Delete` and `Chunk` (non-overlapping pieces; panics if n < 1). Nothing there does
  windows or prefix sums: you write the loop.
- Rust's `slice::windows(k)` yields overlapping `&[T]` views without allocating, **panics if `k == 0`**
  and yields nothing if `k` exceeds the length. `windows(k).map(|w| w.iter().sum())` recomputes every
  window — Θ(nk); the slide is a loop. `chunks` and `chunks_exact` do not overlap.
- `Iterator::scan` carries state and yields each step: a prefix sum in one adapter. Neither language
  ships a prefix-sum function.
- `Vec::dedup` / `dedup_by_key` are the read/write idiom in std (its source states the invariant
  `vec.len() > read > write > write-1 >= 0`) and they *drop* the removed elements.
  `slice::partition_dedup` is unstable.
- Overflow: Go wraps in every build. Rust panics in a debug build and wraps under `-O` for `+`, `-`, `*`
  and `Iterator::sum` alike; `checked_*`, `wrapping_*`, `saturating_*` and `abs_diff` say which you meant.
- `%` truncates in both (`-7 % 3 == -1`). Rust adds `rem_euclid`; Go's `int` has none (only `math/big`'s
  `Int.Mod` is Euclidean), so write `((x % m) + m) % m`.
- Two pointers in Rust are `usize` indices, not references: swap with `slice.swap(i, j)`, and use
  `split_at_mut` for two live `&mut` halves. `usize` also makes `hi - 1` and `P[r] - P[l]` underflow
  hazards (panic in debug, wrap under `-O`); Go's `int` just goes negative, and a negative index panics.

## Complexity Table

| Operation | Time | Space | Note |
|---|---|---|---|
| Pair sum on a sorted slice | **Θ(n)** | Θ(1) | at most n − 1 steps; n(n−1)/2 pairs by brute force |
| Pair sum, unsorted | Θ(n log n) or Θ(n) | Θ(1) or Θ(n) | sort + pointers, or the map of lesson 10 |
| Dedupe / keep at most k, in place | Θ(n) | Θ(1) | n reads, at most n writes; needs equal values adjacent |
| Container with most water | Θ(n) | Θ(1) | n − 1 area evaluations |
| Dutch national flag | **Θ(n)** | Θ(1) | exactly n iterations, at most n swaps |
| 3Sum | Θ(n²) | the sort + output | after an Θ(n log n) sort; brute force is Θ(n³) |
| Fixed window, sum | Θ(n) | Θ(1) | 2 updates per step; recomputing is Θ(nk) |
| Fixed window, max | Θ(n) | Θ(k) | needs the monotonic deque (lesson 09) |
| Variable window | Θ(n) | Θ(σ) or Θ(1) | at most 2n pointer moves |
| Minimum window substring | Θ(n + m) | Θ(σ) | O(1) validity check via a `missing` counter |
| Count subarrays, exactly k | Θ(n) | Θ(1)–Θ(σ) | two window passes; non-monotone otherwise |
| Prefix sums: build / query / update | Θ(n) / **Θ(1)** / Θ(n) | Θ(n) | for immutable data |
| Subarray sum = k with a map | Θ(n) | Θ(n) | any integers; the window needs non-negatives |
| Difference array: range add / read all | **Θ(1)** / Θ(n) | Θ(n) | m updates cost Θ(n + m) in total |
| 2-D prefix: build / rectangle | Θ(mn) / **Θ(1)** | Θ(mn) | 4 reads per query |

## Exercises
1. Write pair sum on a sorted slice and count steps over inputs with no pair; check against brute force on a narrow value range, and reproduce the `lo <= hi` bug.
2. Write `DedupeAtMost(k)`. Assert the invariant after every step, compare with `slices.Compact` at k = 1, and show what stays beyond `write`. In Rust, compare with `Vec::dedup`.
3. Write the fixed-window maximum sum. Count additions against recomputing each window, then break the float version with one huge value.
4. Build a prefix array on a half-open range with `P[0] = 0`. Find the crossover between re-summing and prefix queries, and write the off-by-one version your oracle must catch.
5. Fill a prefix array with values near 2⁵⁹ and find where it wraps. Show ranges whose true sum fits come out exact. In Rust, do it with `wrapping_add`, `checked_sub` and `i128`, and build once without `-O`.
6. Implement a difference array with range add; prove `prefix(diff(a)) == a` and `diff(prefix(a)) == a`, and use it to find the maximum overlap of a set of intervals.
7. Solve container-with-most-water. For each step brute-force check that the discarded wall cannot do better, then run the "move the taller wall" rule against the oracle.
8. Write the Dutch flag with the invariant asserted at every step. Enumerate every array of length ≤ 10 over three values; then add the one-line bug and find the shortest failing input.
9. Solve minimum window substring with a `missing` counter, then with a set instead of counts, and find where the set is wrong.
10. Solve longest-substring-with-at-most-k-distinct in both loop shapes (`for` and `if` inside); count pointer moves for each.
11. Build 2-D prefix sums on a zero-bordered table and check every rectangle of a small grid. Remove one term, then run the unsigned version and count the queries whose middle step goes below zero.
12. Count subarrays with exactly k distinct values and with sum exactly k, as a difference of two passes. Then add negatives and compare with the prefix-sum map.
13. Solve 3Sum with sorting and pointers; write both dedupe bugs and the hash-set version, and count operations for all three.
14. For ten predicates, enumerate every array of length ≤ 7 over a small alphabet and test whether valid windows stay valid when shrunk or grown; run the window loop and compare.
15. Count subarrays whose sum is divisible by m with negative values, three ways: `%`, a normalised remainder, and brute force. Then show prefix max failing as a range max.
16. Stretch — build the toolkit (pair sum, flag, windows, 1-D, 2-D and difference prefixes), prove each against brute force on narrow ranges with step-count assertions, and benchmark each against its brute-force version.

## Best Practices & Pitfalls
- **Write down the invariant before the loop.** Every bug in these techniques is a line that stopped keeping it.
- **Pitfall — pair sum with `lo <= hi`.** An element pairs with itself. Test with a target that is exactly twice some element.
- **Pitfall — dedupe on unsorted input.** Only adjacent repeats go. Sort first, or use a set.
- **Pitfall — moving the taller wall.** The argument only licenses discarding the shorter.
- **Pitfall — advancing `mid` after the swap with `hi`.** The arriving element is unexamined.
- **Pitfall — recomputing the window's validity.** Keep a counter; a recount makes the loop Θ(nσ) or worse.
- **Pitfall — a window over data with negatives.** Growing no longer makes the sum monotone. Use prefix sums and a map.
- **Pitfall — forgetting `P[0] = 0`** (and `counts[0] = 1`). Every range starting at index 0 is then wrong or out of range.
- **Pitfall — mixing inclusive and half-open ranges.** Pick `[l, r)` once. **And `P[r] - P[l]` on unsigned types:** a reversed range wraps to a huge number; use `checked_sub` or assert.
- **Pitfall — assuming a prefix array can answer max, min or gcd.** It needs an inverse. Use a deque, a sparse table or a tree.
- **Pitfall — `x % m` on negative `x`.** The remainder takes the dividend's sign; normalise or use `rem_euclid`.
- **Pitfall — a sliding float sum.** Recompute periodically, or sum with a prefix array of exact integers.
- **Test with an oracle on a narrow value range** (zeros and negatives included), and assert step counts as well as answers.
- **Prefer the simplest tool that is legal.** If the data is sorted or the validity monotone, pointers or a window beat a map; if neither, the prefix array does.

## Checklist
- [ ] I can state the invariant of converging pair sum and say why it takes at most n − 1 steps.
- [ ] I can write read/write dedupe for at most k copies and explain why it compares with `a[write-k]`.
- [ ] I can explain which wall container-with-most-water moves and prove it.
- [ ] I can write the Dutch national flag with four regions and explain the "do not advance `mid`" case.
- [ ] I can write a fixed window and say which aggregates can slide by subtraction.
- [ ] I can write both variable-window shapes, say when each shrinks, and argue the loop makes at most 2n moves.
- [ ] I can count subarrays with exactly k using two at-most passes.
- [ ] I can say exactly when a sliding window is the wrong tool, and what to use instead.
- [ ] I can build 1-D, 2-D and difference prefix arrays on half-open ranges with a zero border.
- [ ] I know what happens when a prefix sum overflows or a prefix subtraction underflows, in Go and in Rust.
- [ ] I can say why a prefix array needs an inverse, and name three operations that have none.
- [ ] I can read a problem statement and name the technique from its cues.

## Resources
- Go's `slices` package — `Reverse`, `Compact`, `Delete`, `Chunk`: https://pkg.go.dev/slices
- …and the source of `Compact`, a read/write loop that clears its tail: https://github.com/golang/go/blob/master/src/slices/slices.go
- Go spec on integer overflow and on `%`: https://go.dev/ref/spec#Integer_overflow · https://go.dev/ref/spec#Integer_operators
- Rust `slice::windows`: https://doc.rust-lang.org/std/primitive.slice.html#method.windows · `Vec::dedup_by`: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.dedup_by · `Iterator::scan`: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.scan
- The Dutch national flag problem: https://en.wikipedia.org/wiki/Dutch_national_flag_problem
- Prefix sums and summed-area tables (the 2-D case): https://en.wikipedia.org/wiki/Prefix_sum · https://en.wikipedia.org/wiki/Summed-area_table
- Dijkstra, *A Discipline of Programming* (1976) — where the flag problem is solved by invariant.
- Examples: [examples/16-two-pointers-windows](examples/16-two-pointers-windows/) (16).
- Next: [17 — Binary Trees & Traversals](17-binary-trees.md) — the first structure where recursion is the natural shape of the data.
