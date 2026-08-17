# Step 08 — Stacks · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. A slice is already a stack**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. A slice is already a stack
- [ ] 2. Stack[T], and what the wrapper costs
- [ ] 3. Balanced brackets
- [ ] 4. RPN evaluation
- [ ] 5. Undo/redo
- [ ] 6. Iterative DFS, and the one-line switch to BFS

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Shunting yard
- [ ] 8. Min-stack in Θ(1)
- [ ] 9. A queue from two stacks
- [ ] 10. Writing down the call stack
- [ ] 11. Which implementation?

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. The monotonic stack
- [ ] 13. The same shape, four more problems
- [ ] 14. Largest rectangle in a histogram
- [ ] 15. Trapping rain water, four ways
- [ ] 16. Capstone: Stack[T] and the algorithms

## The three-pass build

In `practice/08-stacks/`:

- [ ] **Build it** — `Stack[T]` with `(T, bool)` pop, slot zeroing, `Reserve`, `Clear`
- [ ] **Build it** — `NextGreater` and `LargestRectangle` on top of it
- [ ] **Prove it** — table, slot-release invariant, model oracle vs `[]T`
- [ ] **Prove it** — brute-force oracles for **each algorithm**, on small inputs with a narrow value range
- [ ] **Prove it** — assert the amortized claim directly: total pops ≤ n
- [ ] **Measure it** — empty control, push/pop sweep, monotonic sweep on adversarial input
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| Why is a slice a good stack? | every operation is at the tail, the cheap end |
| Why must pop return `(T, bool)`? | `xs[len-1]` on empty is a panic, not a nil |
| When must pop zero the slot? | when `T` contains a pointer |
| The giveaway phrase for a stack? | "the most recently …" |
| Why can't a counter match brackets? | `"([)]"` balances by count and is still wrong |
| RPN: which pop is the left operand? | the **second** |
| The undo/redo rule? | a new action **clears the redo stack** |
| DFS → BFS? | `xs[len-1]` → `xs[0]`; and mark `seen` on push, not pop |
| Where does precedence live in shunting yard? | a number you compare; associativity is strict vs non-strict |
| Why does Θ(1) min work on a stack but not a queue? | the contents are strictly nested |
| Min-stack: `<` or `<=`? | `<=`, or duplicates break it |
| Two-stack queue: the bound and the catch? | amortized Θ(1); one dequeue is Θ(n) |
| Why is a monotonic stack Θ(n)? | each index pushed once, popped at most once |
| Largest rectangle: what does the stack hold? | indices of **increasing** heights |
| When *not* to use a monotonic stack? | when the answer needs a running aggregate, not a neighbour |

## Numbers to find on your own machine

| What | Mine (M4, Go 1.26.3) | Yours |
|------|----------------------|-------|
| `Stack[T]` overhead per op (ex 2) | ~1.6 ns | |
| scan-for-min vs augmented (ex 8) | 89× | |
| aux min-stack size, 100k random vs descending (ex 8) | 15 vs 100,000 | |
| slice vs linked vs `container/list` stack (ex 11) | 1.0× / 3.2× / 5.9× | |
| monotonic vs brute, random vs descending (ex 12) | ~3× vs **2509×** | |
| largest rectangle vs brute at n=3200 (ex 14) | 616× | |
| rain water: two-pointer vs arrays (ex 15) | 6.4× faster | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
