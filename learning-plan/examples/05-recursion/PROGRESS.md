# Step 05 — Recursion & the Call Stack · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Base case and inductive step**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Base case and inductive step
- [ ] 2. Descend, then unwind
- [ ] 3. Every frame keeps its own locals
- [ ] 4. Total calls vs maximum depth
- [ ] 5. Linear recursion is a loop in disguise
- [ ] 6. Go has no tail-call optimisation

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Recursion → loop + explicit stack
- [ ] 8. Memoization, and the bridge to DP
- [ ] 9. How deep can you go, and what happens at the end
- [ ] 10. The shared-slice bug
- [ ] 11. Mutual recursion

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Backtracking: choose, explore, un-choose
- [ ] 13. How fast the input shrinks decides everything
- [ ] 14. What a function call actually costs
- [ ] 15. Surviving input you didn't choose
- [ ] 16. Capstone: a recursive-descent evaluator

## Recall drill

Cover the right column and answer from memory.

| Question | Answer |
|---|---|
| The two parts of any recursion? | base case + strictly smaller inductive step |
| Three ways termination fails? | no base case · argument never shrinks · shrinking never *arrives* |
| Why is `if n == 1` a bad base case for factorial? | `factorial(0)` walks past it forever |
| Which measures time — calls or depth? | calls. Depth measures space |
| `f(n/2)` twice: time and space? | Θ(n) time, Θ(log n) space |
| `f(n-1), f(n-2)`: what's the fix? | memoize — the tree repeats subproblems |
| Does Go optimise tail calls? | **No.** Write the loop yourself |
| Converting a 2-call recursion needs what? | a loop plus an explicit stack (on the heap) |
| Why is in-order harder than pre-order iteratively? | the visit is *between* the calls, so a popped node is half-done |
| Can you `recover()` from a stack overflow? | No — it is a fatal error, not a panic |
| Bytes per frame: plain function vs closure? | ~40 vs ~6,700 |
| Why does the shared-slice bug hide? | at cap 0 every append reallocates; preallocating exposes it |
| The backtracking skeleton? | choose · explore · un-choose, with pruning first |
| Where does precedence come from in recursive descent? | the call graph — `expr` calls `term` |
| Where does associativity come from? | using a **loop** for repetition, not recursion |
| Defence against deep untrusted input? | bound the depth · explicit stack · restructure to log n |

## Numbers to find on your own machine

| What | Mine (M4, Go 1.26.3) | Yours |
|------|----------------------|-------|
| bytes per stack frame (ex 9) | ~34–49 | |
| frames before the 1 GB limit (ex 9) | ~32 million | |
| `fib(35)`: plain vs memoized calls (ex 8) | 29,860,703 → 69 | |
| `fib(30)` invocations of `fib(5)` (ex 8) | 121,393 | |
| distinct subsets, broken version (ex 10) | 4 of 8 | |
| 8-queens nodes, pruned vs not (ex 12) | 2,057 vs 19,173,961 | |
| merge vs insertion recursion depth at n=100k (ex 13) | 17 vs 99,999 | |
| recursive vs iterative sum of 10,000 (ex 14) | 12.4× | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
