# Step 01 — Introduction · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Same data, different structure**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Same data, different structure
- [ ] 2. Count the work, not the clock
- [ ] 3. Quadratic vs linear, in operations
- [ ] 4. What the growth curves actually cost
- [ ] 5. Trading space for time

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 6. From counting to the clock
- [ ] 7. Benchmarking from a plain program
- [ ] 8. Counting allocations
- [ ] 9. Finding the crossover point
- [ ] 10. What "O(n) space" actually costs

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 11. Big-O lies at small n
- [ ] 12. The benchmark that measured nothing
- [ ] 13. `[]T` vs `[]*T`: the pointer tax
- [ ] 14. The brute-force oracle

## Numbers to find on your own machine

Examples 9, 11 and 12 measure *your* hardware, not mine. Fill these in — they're the first entries
in your own performance intuition:

| What | Mine (M-series, Go 1.26.3) | Yours |
|------|----------------------------|-------|
| map beats linear scan at n ≥ | 16 | |
| `slices.Sort` beats insertion sort at n ≥ | 32 | |
| empty `b.Loop` floor | ~1.6 ns/op | |
| `[]*T` vs `[]T` sum, 1M elements | 2.7× slower | |
| GC cycle, 1M pointers vs 1M values | ~59× | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
