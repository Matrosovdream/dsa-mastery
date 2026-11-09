# 15 — Binary Search & Its Variants

> Part of **Part 3 — Sorting & Searching**, and the payoff for the three sorting lessons before it:
> once the data is in order, one comparison throws away half of it. Binary search is the algorithm
> everyone can describe and almost nobody writes correctly the first time — Knuth notes that the
> first published version appeared in 1946 and the first one without a bug in 1962. The one-line
> thesis: **binary search is not "look in the middle"; it is a loop that keeps one invariant about
> a monotone predicate, and every bug in it is a place where the code stopped keeping it.**

## Goals
- State the invariant of a lower-bound search and prove both termination and correctness from it.
- Explain the `(lo + hi) / 2` overflow, and say exactly when it bites in Go and in Rust.
- Write **lower bound** and **upper bound** from memory, and get equal-range, count and insert position from them.
- Know what `sort.Search`, `sort.Find`, `slices.BinarySearch`, `binary_search` and `partition_point` return — **on duplicates** in particular.
- Turn any "smallest x such that…" question into a predicate and search it.
- Search a **rotated** array, and know the one input where it degrades to Θ(n).
- Solve the **binary-search-on-the-answer** family: Koko's bananas, ship packing, split-array-largest-sum.
- Search a 2-D matrix in both of its forms, and know which algorithm each form allows.
- Have one template you trust, and the argument for why it cannot loop forever.

## Concepts

- **Halving is the whole complexity.** Each probe removes half of what is left, so n candidates need
  about log₂ n probes: 7 questions find a number in 1..100, 20 find one in a million, 64 cover every
  `uint64`. The cost is the precondition — the data must be sorted, or at least the predicate must be
  monotone — and the price of establishing it was the previous three lessons.

- **Search for a boundary, not a value.** Reframe every binary search as: there is a predicate `p(i)`
  that is `false` for a prefix of `[0, n)` and `true` for the rest; find the first `true`. For "find
  x" the predicate is `a[i] >= x`. This one reframing gives you lower bound, upper bound, insert
  position, first-bad-version, integer square root and every "search on the answer" problem with the
  **same eight lines**.

- **The invariant.** Keep a half-open window `[lo, hi)` with two promises: everything left of `lo` is
  `false`, and `hi` is `true` (pretend `p(n)` is `true` — a sentinel you never read). The window
  starts as `[0, n)`. Probe `mid` in `[lo, hi)`: if `p(mid)` is false, `lo = mid + 1`; otherwise
  `hi = mid`. Both updates keep both promises and **strictly shrink** the window, so the loop
  terminates; when `lo == hi`, the promises say `lo` is the first `true`. That is the entire proof,
  and Go's `sort.Search` carries it in its comments: *"Invariant: f(i-1) == false, f(j) == true."*

- **Exact probe count.** That loop makes either ⌊log₂(n+1)⌋ or ⌈log₂(n+1)⌉ probes, whatever the
  answer — it never stops early, because a boundary search cannot know it is done until the window
  is empty ([example 1](examples/15-binary-search/1-easy.md) counts it for every target).

- **The overflow bug.** `mid := (lo + hi) / 2` is wrong when `lo + hi` exceeds the integer type: the
  sum wraps negative and `mid` becomes a negative index. It sat in the JDK's binary search for
  about nine years before Joshua Bloch wrote it up in 2006. With 64-bit `int` it takes an index range
  past 2⁶², so a slice will never trigger it — but **searching on the answer** happily searches value
  ranges up to `math.MaxInt64`, and 32-bit index types trigger it at a billion. The fixes:
  `lo + (hi-lo)/2`, or the unsigned trick Go's own library uses, `int(uint(lo+hi) >> 1)` — the sum of
  two non-negative `int`s always fits in a `uint`.

- **What the overflow does depends on the language.** Go's spec says signed overflow **wraps** and
  "does not cause a run-time panic" — the bug is silent until the negative index panics. In Rust,
  `lo + hi` **panics in a debug build** and **wraps in a release build** (`-O`), unless you pass
  `-C overflow-checks=on`. So the same Rust code fails loudly in tests and silently in production.
  The idiom is to make the intent explicit: `checked_add`, `wrapping_add`, or `usize::midpoint`
  ([example 2](examples/15-binary-search/1-easy.md) shows all of them without ever crashing).

- **Lower bound and upper bound.** `lower_bound(x)` = first index with `a[i] >= x`; `upper_bound(x)`
  = first index with `a[i] > x`. Together they give everything: `x` is present iff `lb < n &&
  a[lb] == x`; the run of `x` is `a[lb:ub]`; its count is `ub - lb`; inserting at `ub` keeps equal
  elements in arrival order (a stable insert). Both are the same loop with a different predicate.

- **`sort.Search(n, f)`** is exactly the boundary search: smallest `i` in `[0, n)` with `f(i)` true,
  or **`n`** if there is none — not `-1`. It calls `f` only for `i` in `[0, n)`. It works on anything
  indexable, or on nothing at all: `f` may compute instead of index.
- **`sort.Find(n, cmp)`** is the three-way form: first `i` with `cmp(i) <= 0`, and `found` is
  `i < n && cmp(i) == 0`.
- **`slices.BinarySearch(x, target)`** returns `(i, found)`: per its documentation, *"the earliest
  position where target is found, or the position where target would appear"*. It is a lower bound,
  so **on duplicates it returns the first one** — guaranteed. Floats use `cmp.Less`, which orders
  `NaN` before every number, and a `NaN` target is reported found if the slice holds a `NaN`.
  `slices.BinarySearchFunc` takes a comparator; `sort.SearchInts` & co. are thin wrappers.

- **Rotated arrays.** A sorted array rotated by k is two sorted runs. Compare `a[mid]` with
  `a[hi]`: if it is bigger, the minimum lies right of `mid`; otherwise at or left of it. That finds
  the rotation point in Θ(log n), and a one-pass variant searches for a target by asking which half
  is sorted. **With duplicates the trick fails**: when `a[mid] == a[hi]` you cannot tell which half
  holds the minimum, and the only safe step is `hi--` — Θ(n) on `[1,1,1,…,0,1,1]`.

- **Binary search on the answer.** When the question is "the smallest capacity / speed / time such
  that something is feasible", and feasibility is monotone (if capacity c works, c+1 works), you do
  not need a sorted array — you search the **answer range** with a Θ(n) feasibility check, for
  Θ(n log R) total. Koko eats piles at speed k: feasible iff `Σ ⌈p/k⌉ ≤ h`, search k in
  `[1, max p]`. Ship packages in D days: feasible iff a greedy left-to-right fill uses ≤ D ships,
  search capacity in `[max w, Σ w]` — the lower end is the heaviest package, not 1. The same code
  solves split-array-largest-sum and painter's partition: **"minimize the maximum"** is the tell.

- **Real-valued answers.** Bisection on floats needs a stopping rule. `for hi-lo > eps` can loop
  forever when `eps` is smaller than the spacing of floats near the answer; a fixed iteration count
  cannot. Because positive IEEE-754 doubles sort the same as their bit patterns, you can also binary
  search the **bits** and get the exact nearest float in at most 64 steps.

- **2-D matrices come in two kinds.** If each row starts after the previous one ends, the matrix
  *is* a sorted array of m·n elements: map `i → (i / cols, i % cols)` and do one search,
  Θ(log mn). If only rows and columns are sorted separately, that mapping is wrong; start at the
  top-right corner and step left or down — the **staircase**, Θ(m + n) — or search each row,
  Θ(m log n).

- **Binary search does not need sorted data.** It needs a rule that says which half still holds an
  answer. For a peak element: if `a[mid] < a[mid+1]`, some peak lies right of `mid`; otherwise one
  lies at or left of it. That keeps the window shrinking by half on an unsorted array, even though
  "is a peak" is not a monotone predicate.

- **Stopping boundary bugs for good.** There are only four decisions: the window shape (`[lo, hi)`
  or `[lo, hi]`), the loop test (`<` or `<=`), the two updates, and whether `mid` rounds down or up.
  Mix them and you get infinite loops (`lo = mid` with a mid that rounds down), skipped answers
  (`hi = mid - 1` when `mid` might be the answer), or out-of-range reads (`hi = n` with `<=`). The
  cure is to pick **one** template — half-open, `lo < hi`, `lo = mid + 1` / `hi = mid`, mid rounds
  down — and translate every problem into "first index where `p` is true". Then test it the way
  [lesson 02](02-environment-setup.md) taught: an exhaustive oracle over every small array.
  [Example 12](examples/15-binary-search/3-hard.md) enumerates the variants and runs each one.

- **Three-way search is not faster.** The textbook `if a[mid] == x { return mid }` exits early, but
  pays up to two comparisons per step and returns an *arbitrary* duplicate. The boundary search pays
  one comparison per step and returns a defined index ([example 6](examples/15-binary-search/1-easy.md)
  counts both).

### Go and Rust side by side

- **`binary_search` returns `Result<usize, usize>`**: `Ok(i)` found, `Err(i)` the insert position —
  Go's `(i, found)` as a type you have to match. `s.insert(i, x)` after `Err(i)` keeps the order.
- **On duplicates Rust promises nothing**: *"any one of the matches could be returned. The index is
  chosen deterministically, but is subject to change"* (`core/src/slice/mod.rs`). The current loop
  never exits on `Equal` and moves right on `Less` *or* `Equal`, so it lands on the **last** of a run —
  the opposite of Go. [Example 4](examples/15-binary-search/1-easy.md) shows it; don't rely on it.
- **`partition_point(pred)`** is Rust's `sort.Search` over a slice: the first index where `pred` is
  false (note: false, the mirror of Go's first-true). It is the guaranteed lower bound
  (`|x| x < &t`) or upper bound (`|x| x <= &t`). It is built on `binary_search_by` and calls the
  predicate exactly ⌈log₂ n⌉ + 1 times for n ≥ 1, whatever it returns.
- **`binary_search_by_key(&key, |e| e.field)`** searches records sorted by one field;
  `binary_search_by` takes a comparator returning `Ordering`.
- **`binary_search` needs `T: Ord`**, so `Vec<f64>` does not compile; use
  `binary_search_by(|p| p.total_cmp(&x))`. Go accepts floats and handles `NaN` in `cmp.Less`.
- **No range search in std.** Rust has no `sort.Search(n, f)` over an integer range; the twins write
  the eight-line loop. `usize::midpoint` (stable since 1.85) is the overflow-free average.
- **std's search is branchless**: the loop count depends only on the length and the next window is
  picked with `hint::select_unpredictable`. Go's `slices.BinarySearch` is written with an `if`, and
  it stays a branch: Go's compiler will not turn a conditional into a select when the result feeds a
  load address (example 14 shows both assemblies).

## Complexity Table

| Operation | Time | Space | Note |
|---|---|---|---|
| Lower / upper bound | **Θ(log n)** | Θ(1) | ⌊log₂(n+1)⌋ or ⌈log₂(n+1)⌉ probes, every query |
| Three-way search with early exit | Θ(log n) worst, Θ(1) best | Θ(1) | up to 2 comparisons per step; arbitrary duplicate |
| `slices.BinarySearch` | Θ(log n) | Θ(1) | earliest duplicate, guaranteed |
| Rust `binary_search` / `partition_point` | Θ(log n) | Θ(1) | ⌈log₂ n⌉ + 1 comparisons, always; duplicate unspecified |
| Equal range / count of `x` | Θ(log n) | Θ(1) | two searches |
| Insert into sorted slice | Θ(log n) find + **Θ(n)** shift | | the search is not the cost |
| Find minimum of rotated array | Θ(log n) | Θ(1) | **Θ(n)** worst with duplicates |
| Search on the answer, range R | Θ(log R · cost of `feasible`) | Θ(1) | Koko / ships: Θ(n log R) |
| Float bisection to precision ε | Θ(log((hi−lo)/ε)) | Θ(1) | ≤ 64 steps on the bit pattern |
| Row-major sorted matrix | Θ(log mn) | Θ(1) | one search over the flattened index |
| Row- and column-sorted matrix | **Θ(m + n)** staircase | Θ(1) | or Θ(m log n) row by row |
| Peak element | Θ(log n) | Θ(1) | data need not be sorted |
| Exponential (galloping) search for position i | Θ(log i) | Θ(1) | when the answer is near the front |

## Exercises
1. Write lower bound with the invariant as an assertion checked on every iteration. Count probes for every target in an array of n = 1000 and confirm the min and max match ⌊log₂(n+1)⌋ and ⌈log₂(n+1)⌉.
2. Reproduce the overflow: search a value range ending at `math.MaxInt64` with `(lo+hi)/2`, then fix it two ways. In Rust, show the same sum with `checked_add`, `wrapping_add` and `midpoint`, and build once without `-O`.
3. Write lower and upper bound. From them, write `Contains`, `Count`, `EqualRange` and a stable `Insert`.
4. Call `slices.BinarySearch` on an array of 9 equal elements; then Rust's `binary_search`. Explain the two answers from the source, and get Go's answer in Rust with `partition_point`.
5. Use `sort.Search` without an array: integer square root of 10¹⁸, first bad version (count the calls), and the guessing game.
6. Write the three-way search with early exit. Count comparisons over every successful and unsuccessful target and compare with lower bound.
7. Find the minimum of a rotated array, then search it for a target. Verify against a linear scan for every rotation of every n ≤ 12. Then add duplicates and find the input that forces Θ(n).
8. Solve Koko's bananas. Count feasibility calls against a linear scan over k, and check against brute force on random piles.
9. Solve ship-packages-in-D-days. Explain why the lower end of the range is `max(w)`. Reuse the code for split-array-largest-sum.
10. Search a row-major sorted matrix as a flat array. Then build a row/column-sorted matrix on which that search is wrong, and solve it with the staircase.
11. Find a peak in an unsorted array in Θ(log n). State the invariant that says which half still holds a peak, and verify against a linear scan.
12. Write all 32 combinations of (`<`/`<=`, `hi = n`/`n-1`, `hi = mid`/`mid-1`, `lo = mid`/`mid+1`, mid floor/ceil) and run each against an oracle on every sorted array of length ≤ 6 over 3 values. Classify each as correct, wrong, out of range, or non-terminating.
13. Compute √2 by bisection with `hi-lo > 1e-18` and an iteration cap. Then search the float's bit pattern and get the correctly rounded answer.
14. Write a branchless lower bound and an Eytzinger-layout search. Measure all three against the stdlib at sizes that fit in L1, L2 and RAM.
15. Write exponential search. Intersect a 100-element sorted list with a 1,000,000-element one three ways and count comparisons.
16. Stretch — build a sorted timestamp index (count in range, predecessor, successor) and prove it against brute force on a narrow value range, so duplicates actually occur.

## Best Practices & Pitfalls
- **Use the stdlib.** `slices.BinarySearch` / `sort.Search` in Go; `binary_search` / `partition_point` in Rust. Write your own only to search something that is not a slice.
- **Phrase every problem as "first index where p is true".** Then there is one loop to get right, and you already have it.
- **Pitfall — `(lo + hi) / 2`.** Write `lo + (hi-lo)/2`, or use `midpoint` in Rust.
- **Pitfall — treating `-1` as "not found".** `sort.Search` returns `n`; always check `i < n && a[i] == x`.
- **Pitfall — assuming which duplicate you get.** Go's `BinarySearch` gives the first; Rust's `binary_search` gives *some* one. Need the first or last? Use a bound.
- **Pitfall — `lo = mid` with a mid that rounds down.** When `hi = lo + 1` it never moves: an infinite loop.
- **Pitfall — a non-monotone predicate.** Binary search on it returns *an* answer, confidently. Check monotonicity with a brute-force sweep on small inputs.
- **Pitfall — the wrong answer range.** Ship packing searched from 1 returns capacities that cannot lift the heaviest package.
- **Pitfall — `for hi-lo > eps` on floats.** Cap the iterations or search the bit pattern.
- **Pitfall — an unsorted input.** Neither language checks; both return garbage. `slices.IsSorted` / `is_sorted` in a debug assertion.
- **Pitfall — rotated arrays with duplicates.** The Θ(log n) argument is gone; say so in the comment.
- **Test with an oracle.** Every array of length ≤ 6 over 3 values is a few thousand cases and covers every boundary.
- **Pitfall — sorting to search once.** Sorting is Θ(n log n); one linear scan is Θ(n). Binary search pays off over many queries.

## Checklist
- [ ] I can state the invariant of lower bound and prove termination from it.
- [ ] I can say how many probes a boundary search makes, exactly.
- [ ] I know when `(lo + hi) / 2` overflows, and what Go and Rust each do when it does.
- [ ] I can write lower and upper bound from memory and derive count and equal range.
- [ ] I know which duplicate `slices.BinarySearch` returns and that Rust's `binary_search` promises none.
- [ ] I can use `sort.Search` and `partition_point` with a predicate that is not an array lookup.
- [ ] I can search a rotated array and name the input that makes it Θ(n).
- [ ] I recognise "minimize the maximum" as binary search on the answer, and pick the range correctly.
- [ ] I can bisect on floats without risking an infinite loop.
- [ ] I can tell the two kinds of sorted matrix apart and search each.
- [ ] I have one template, and I test it against an exhaustive oracle.

## Resources
- Go's `sort.Search` — the invariant is in the comments: https://github.com/golang/go/blob/master/src/sort/search.go
- Go's `slices.BinarySearch`: https://github.com/golang/go/blob/master/src/slices/sort.go
- Rust's `binary_search_by` and `partition_point`: https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search_by
- Joshua Bloch, "Extra, Extra — Read All About It: Nearly All Binary Searches and Mergesorts are Broken" (2006): https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/
- Jon Bentley, *Programming Pearls*, column 4 "Writing Correct Programs" — the invariant method, applied to binary search.
- Knuth, TAOCP vol. 3, §6.2.1 — binary search, its history and its exact comparison counts.
- Khuong & Morin, "Array Layouts for Comparison-Based Searching" (2017) — branchless and Eytzinger layouts: https://arxiv.org/abs/1509.05053
- Examples: [examples/15-binary-search](examples/15-binary-search/) (16).
- Next: [16 — Two Pointers, Sliding Window & Prefix Sums](16-two-pointers-windows.md) — the other ways to avoid looking at everything twice.
