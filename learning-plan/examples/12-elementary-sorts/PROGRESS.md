# Step 12 — Elementary Sorts · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Bubble sort, and the early exit**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Bubble sort, and the early exit
- [ ] 2. Selection sort never adapts
- [ ] 3. Insertion sort: shift, don't swap
- [ ] 4. Loop invariants, checked at every step
- [ ] 5. Stability, and the one-character bug
- [ ] 6. Inversions: what insertion sort really pays

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Three sorts, six inputs, every operation counted
- [ ] 8. Nearly sorted input: where insertion sort wins
- [ ] 9. Binary insertion sort: fewer comparisons, same moves
- [ ] 10. Multi-key sorting needs stability
- [ ] 11. The comparator contract

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Catching the stdlib's insertion-sort cutoff
- [ ] 13. What the stdlib does with presorted input
- [ ] 14. Shell sort: insertion sort with a gap
- [ ] 15. When writes cost more than comparisons
- [ ] 16. Capstone: `InsertionSortFunc`, built, proved, measured

## The three-pass build

In `practice/12-elementary-sorts/`:

- [ ] **Build it** — `BubbleSort`, `SelectionSort`, `InsertionSort`, each with an operation counter (comparisons, writes)
- [ ] **Build it** — generic `InsertionSortFunc[E any](s []E, cmp func(a, b E) int)`, shift-based, stable
- [ ] **Build it** — `BinaryInsertionSortFunc` with an **upper-bound** search, and `ShellSort` with Knuth's gaps
- [ ] **Prove it** — brute-force oracle against `slices.SortStableFunc` on a **narrow key range**, comparing whole records (so stability is checked too)
- [ ] **Prove it** — invariants after every outer iteration, for each of the three sorts
- [ ] **Prove it** — property: insertion sort's shifts == inversions, comparisons in `[I, I+n-1]`
- [ ] **Prove it** — edge cases: empty, one element, all equal, sorted, reversed
- [ ] **Measure it** — comparisons and writes on sorted, reversed, random, nearly sorted and few-distinct input
- [ ] **Measure it** — against `slices.SortFunc` and `slices.SortStableFunc` at small n, and on nearly sorted large n
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| Bubble sort's invariant after pass k? | the k largest are final, at the end |
| Selection sort's invariant? | `a[:i]` holds the i smallest, sorted, **final** |
| Insertion sort's invariant? | `a[:i]` is sorted — but **not** final |
| Selection sort's comparisons on sorted input? | **n(n−1)/2** — it never adapts |
| Selection sort's one virtue? | at most **n−1** swaps |
| What is an inversion? | a pair i < j with a[i] > a[j] |
| Insertion sort's shifts equal…? | the inversion count, **exactly** |
| …and its comparisons? | between I and I + n − 1 |
| Inversions in a random permutation, on average? | n(n−1)/4 |
| Why are adjacent-swap sorts Θ(n²) on average? | one swap removes one inversion, and there are Θ(n²) |
| Which of the three are stable? | bubble and insertion — **not** selection |
| The one-character stability bug? | `<` → `<=` |
| Three reasons stability is a correctness property? | multi-key passes · deterministic output · ties are normal |
| How to make any sort deterministic? | a **total** comparator (`cmp.Or`, `then_with`) |
| Where does Go's `slices.Sort` use insertion sort? | every sub-range of length **≤ 12** |
| …and Go's stable sort? | blocks of **20**, then an in-place merge |
| …and Rust's `sort` / `sort_unstable`? | the whole slice when length **≤ 20** |
| Why is `return a - b` a broken Go comparator? | it overflows |
| Binary insertion sort saves…? | comparisons — not moves |
| Which bound keeps binary insertion stable? | the **upper** bound |

> Lesson: [12-elementary-sorts.md](../../12-elementary-sorts.md) · Index: [README.md](README.md)
