# Step 06 — Dynamic Arrays & Slices · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The four operations**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The four operations
- [ ] 2. Two ways to delete
- [ ] 3. The write-index idiom
- [ ] 4. Reverse and rotate in place
- [ ] 5. A sorted slice is a data structure
- [ ] 6. What the stdlib already has

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Building `Vector[T]`, and deriving amortized Θ(1)
- [ ] 8. Shrinking, and how it thrashes
- [ ] 9. Choosing the growth factor
- [ ] 10. `buf[:0]` — the best optimisation in a hot loop
- [ ] 11. Three ways to lay out a grid

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Sorted slice vs map, with the workload as a variable
- [ ] 13. The gap buffer
- [ ] 14. The accidental quadratic
- [ ] 15. Partition, and the invariant that makes it work
- [ ] 16. Capstone: `Vector[T]`, all three passes

## The first three-pass build

Lesson 06 is where the plan's method starts applying. Tick these for your own `Vector[T]` in
`practice/06-arrays-slices/`:

- [ ] **Build it** — `At`, `Set`, `Push`, `Pop`, `Insert`, `Delete`, `Reserve`, with the invariant written down
- [ ] **Prove it** — a table, invariant checks, a **model oracle** vs plain `[]T` over 20,000 random ops
- [ ] **Prove it** — allocation budget, geometric-growth gate, no-thrash gate
- [ ] **Measure it** — empty control, `Push` sweep vs native `append`, `InsertFront` sweep, `At` at two sizes
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| Cost of `Insert(i, v)`? | Θ(n) — shifts `n−i` |
| Two deletions, and their costs? | ordered Θ(n) · swap Θ(1), destroys order |
| When must you nil the vacated slot? | when `T` contains a pointer |
| The write-index idiom, in one line? | `if keep { xs[write] = xs[read]; write++ }` |
| Why is deleting in a forward loop wrong? | the tail slides under the cursor; `i++` skips |
| Rotate in Θ(1) space? | reverse first k, reverse rest, reverse all |
| Why does doubling give amortized Θ(1)? | copies form a geometric series → `n/(g−1)` |
| Copies per push at factor g? | `1/(g−1)` |
| Why does grow-by-+k fail? | total becomes Θ(n²); per-push cost is unbounded |
| What is shrink thrashing, and the fix? | touching grow/shrink thresholds; hysteresis (grow full, shrink at ¼) |
| Three ways to empty a slice? | `xs[:0]` reuse · `clear(xs)` zero values · `xs = nil` release |
| Best 2-D layout, usually? | `[][]T` over ONE backing array |
| `slices.Delete` in a loop? | Θ(n·k) — use `slices.DeleteFunc` |
| Why three-way partition? | duplicates: finishes the equal block in one pass |
| What does a gap buffer buy? | Θ(1) insert at the cursor; Θ(distance) to move it |

## Numbers to find on your own machine

| What | Mine (M4, Go 1.26.3) | Yours |
|------|----------------------|-------|
| copies/push at ×2 (ex 7) | 1.00 | |
| copies/push, grow by +64, n=100k (ex 7) | 781.2 | |
| shrink-to-fit thrash cost (ex 8) | 2048 copies per push/pop | |
| `buf[:0]` speedup and allocs (ex 10) | 3.2×, 13 → 0 | |
| jagged vs backed heap objects, 1000² (ex 11) | 997 vs a handful | |
| `Delete`-in-loop vs one pass, n=64k (ex 14) | 355× | |
| gap buffer vs plain, n=16k (ex 13) | 46× | |
| `Vector.At` at n=1k and n=1M (ex 16) | 1.70 / 1.72 ns | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
