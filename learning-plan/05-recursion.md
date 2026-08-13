# 05 — Recursion & the Call Stack

> Closes **Part 1 — Foundations**. Lesson [03](03-complexity-analysis.md) showed that recursion depth
> is space; lesson [04](04-go-memory-model.md) showed what a frame and a pointer really cost. This
> lesson is about the technique itself: how to write a recursion that terminates, how to convert one
> that shouldn't be recursive, and how to keep an untrusted input from killing your process. The
> one-line thesis: **recursion is the right tool when the DATA is recursive — and a liability when
> the depth grows with n.**

## Goals
- Write a recursion by stating its **base case** and a **strictly smaller** inductive step.
- Read the descend/unwind shape of a call stack, and know what's alive at the deepest point.
- Separate **total calls** (time) from **max depth** (space) — they're different questions.
- Convert linear recursion to a loop, and multi-call recursion to a loop plus an **explicit stack**.
- Know that Go has **no tail-call optimisation**, and prove it by measuring.
- Add a **memo** to collapse an exponential recursion tree — the bridge to DP.
- Avoid the aliasing bug that ruins backtracking in Go.
- Bound recursion depth so hostile input returns an error instead of killing the process.

## Concepts

- **Two parts, and three ways they fail.** A recursion is a base case plus a step that makes the
  problem *strictly smaller*. It fails if there's no base case, if the argument never shrinks, or —
  the subtle one — if shrinking never *arrives* at the base case. `if n == 1` looks fine and works
  for every input except 0, where `n-1` walks past it forever. Write the base case first, and test
  the boundary, not the middle.

- **Descend, then unwind.** Calls go all the way down doing nothing, and the work happens on the way
  back up. At the deepest point every frame is still alive, each holding its own copy of the
  parameters — which is the Θ(depth) space from lesson 03, and why the frame at depth 0 still has its
  `n` waiting after four nested calls came and went.

- **Total calls and max depth are different questions.** Time is the *size of the recursion tree*;
  space is its *height*. At n=24: `f(n/2)` twice makes 31 calls at depth 4 (divide & conquer);
  `f(n-1), f(n-2)` makes 150,049 calls at depth 23 (memoize it); `f(n-1)` makes 25 calls at depth 24
  — trivial time, and the only one of the three that overflows on real data.

- **Linear recursion is a loop in disguise.** One recursive call on an input one step smaller
  converts mechanically: the base case becomes the initial value, the inductive step becomes the body.
  Do it whenever depth grows with n.

- **Go has no tail-call optimisation.** Rewriting with an accumulator so nothing is pending after the
  call *does not* make the stack flat — measured, the "tail-recursive" version grows exactly like the
  non-tail one, while the loop stays at 64 KB. What the accumulator rewrite is actually good for in Go
  is making the loop obvious enough to write by hand.

- **More than one recursive call needs an explicit stack.** Push what the recursive parameter held,
  loop while the stack is non-empty, push children in *reverse* (a stack reverses them again).
  Pre-order is easy because the visit happens before both calls; in-order is harder because a popped
  node is only half-done. That extra bookkeeping is exactly what the call stack was doing for free —
  and the explicit stack lives on the **heap**, so it can reach millions of entries.

- **Memoization collapses a tree with repeated subproblems.** Computing `fib(30)` invokes `fib(10)`
  **10,946 times** and `fib(5)` **121,393 times**. Three added lines cut 2,692,537 calls to 59. The
  call count drops to the number of *distinct* subproblems — Θ(n) for fib, Θ(r·c) for grid paths.
  Tabulation is the same recurrence bottom-up, with no stack at all. When there are no repeated
  subproblems (merge sort's halves never overlap), there is nothing to memoize — that's divide &
  conquer, not DP.

- **The Go-specific backtracking bug.** Record a partial answer by storing the slice itself and every
  recorded answer is a view into one shared array. Subsets of `[1 2 3]` came back as
  `[[] [1] [1] [1 2] [1] [1 2] [1 2] [1 2 3]]` — **4 distinct instead of 8**. The trap: it only fires
  when `cur` has spare capacity, so `[]int{}` accidentally works and a `make([]int, 0, n)`
  "optimisation" breaks it. Fix with `slices.Clone` at the moment you record.

- **Backtracking is recursion with an un-do, and pruning is the whole game.** choose / explore /
  un-choose, with a validity check as early in the loop as possible. On 8-queens, pruning visits
  **2,057 nodes** against **19,173,961** unpruned — for the same 92 solutions.

- **How fast the input shrinks decides whether you can ship it.** `n → n/2` gives depth log n
  (17 at n=100,000, ~30 at a billion). `n → n-1` gives depth n. Both are "recursive sorts"; only one
  can be handed real input.

- **A stack overflow is fatal, not a panic.** Measured: ~34–49 bytes per frame, so the 1 GB default
  limit is roughly **32 million frames** — and past it the runtime prints `fatal error: stack overflow`
  and exits. `recover()` does not run. The limit is per *goroutine*, so no worker pool or timeout
  protects you. The defence is bounding the depth or removing the recursion, never a bigger limit.

- **Recursion is the right tool when the data is recursive.** JSON, ASTs, filesystems and expression
  grammars are defined recursively, so the natural reader is too. In a recursive-descent parser the
  code *is* the grammar: precedence comes from the call graph, associativity from using a loop for
  repetition.

- **"Recursion is slow" is too blunt to be useful.** Measured: summing 10,000 ints recursively is
  **12.4×** slower (the call *is* the loop body); binary search recursively is **1.4×** (≈20 calls
  total — invisible). And `fib(25)` is 21,838× slower recursively because of the *algorithm*, not
  the calls. Convert when the depth is the problem, or when the call is the innermost operation.

## Complexity Table

| Shape | Recursive call | Total calls | Max depth | Fix |
|-------|---------------|-------------|-----------|-----|
| Linear | `f(n-1)` | Θ(n) | **Θ(n)** | make it a loop |
| Halving | `f(n/2)` | Θ(log n) | Θ(log n) | fine as is |
| Divide & conquer | `f(n/2)` twice | Θ(n) | Θ(log n) | fine as is |
| Tree recursion | `f(n-1), f(n-2)` | Θ(φⁿ) | Θ(n) | **memoize** |
| Backtracking | branch per candidate | Θ(bᵈ) minus pruning | Θ(d) | prune early |

Measured on this machine (Apple M4, Go 1.26.3):

| Fact | Number |
|------|--------|
| Bytes per stack frame (plain function) | **~34–49** |
| Bytes per frame (recursive **closure**) | **~6,700** |
| Practical depth limit at the 1 GB default | **~32 million frames** |
| Stack overflow | `fatal error`, exit 2, **`recover()` never runs** |
| `fib(30)` invocations of `fib(5)` | **121,393** |
| Memo on `fib(35)` | 29,860,703 calls → **69** |
| 8-queens, pruned vs not | 2,057 vs **19,173,961** nodes |
| Recursive vs iterative sum of 10,000 | **12.4×** |
| Recursive vs iterative binary search | 1.4× |

## Exercises
1. Write `factorial` correctly, then write three broken versions — no base case, a base case that's never reached, an argument that doesn't shrink. Add a depth seatbelt so you can observe each one.
2. Instrument a recursive `sum` to print on entry and on exit with indentation. Explain from the output why nothing is added until the deepest call returns.
3. Show that each frame keeps its own locals: record a parameter on entry and again after the recursive call returns, at every depth.
4. Count total calls *and* max depth for `f(n-1)`, `f(n/2)`, `f(n/2)` twice, and `f(n-1),f(n-2)`. Say which column predicts time and which predicts space.
5. Convert four linear recursions to loops: sum, factorial, reverse-in-place, binary search. Explain which one didn't need converting and why.
6. Write a tail-recursive sum with an accumulator, and measure its stack against the non-tail version and a loop. Conclude what Go does with tail calls.
7. Write recursive and explicit-stack versions of pre-order and in-order traversal. Explain why in-order needs more bookkeeping than pre-order.
8. Take naive `fib` and `gridPaths`, add memos, and tabulate the call counts. Then measure how many times `fib(30)` invokes `fib(10)`.
9. Measure bytes per stack frame at three depths, compute your machine's practical frame limit, then trigger a real stack overflow in a child process with `debug.SetMaxStack` and confirm `recover()` does not run.
10. Generate all subsets by recording the working slice directly — with `cur` preallocated. Count the distinct results, then fix it two ways.
11. Write `isEven`/`isOdd` mutually, and then a mutually recursive walker over a nested list/group document. Show why a local recursive closure needs `var f func(...)` first.
12. Implement permutations and N-queens with choose/explore/un-choose. Count tree nodes with and without the safety check at n = 4…8.
13. Compare recursion depth and stack bytes for merge sort against a recursive insertion sort at n = 1,000…100,000.
14. Benchmark recursive vs iterative for sum, fib and binary search. Explain why the three ratios are so different.
15. Build a degenerate 1,000,000-deep tree. Sum it recursively, iteratively, and with a depth-bounded recursion. Then crash the recursive one in a child process and say which of the three you'd ship.
16. Stretch — write a recursive-descent evaluator for `+ - * /`, parentheses and unary minus. Show that precedence comes from the call graph and associativity from using a loop, then add a depth guard.

## Best Practices & Pitfalls
- **Write the base case first.** It's the answer you already know, and it tells you what "smaller" has to mean.
- **Check that shrinking arrives at the base case**, not just that it shrinks. `n-1` from 0 never reaches 1.
- **Ask "how deep?" before "how many calls?"** Depth decides whether the code is safe on real input.
- **Pitfall — the shared slice in backtracking.** `slices.Clone` whatever you record. The bug hides when capacity is 0, so it appears the day someone preallocates.
- **Pitfall — assuming an accumulator makes it constant-stack.** Go has no TCO. Write the loop.
- **Pitfall — recursive closures.** They measured ~6.7 KB per frame against ~40 bytes for a plain function; a closure-based probe overflowed 1 GB at depth 100,000. Use package-level functions for deep recursion.
- **Pitfall — trying to `recover()` from a stack overflow.** It is a fatal error. The deferred function never runs.
- **Bound the depth on untrusted input.** Four lines turns a process kill into a `400 Bad Request`. `encoding/json` does exactly this.
- **Don't convert halving recursions to loops for "performance".** ~20 calls is invisible; do it for taste if at all.
- **Memoize before optimising anything else** when the recursion tree repeats subproblems. Three lines beat every micro-optimisation.
- **Prune as early in the loop as possible.** Rejecting at depth 2 skips the entire subtree below it.
- **Leave it recursive when the data is recursive.** A parser or a tree walker written iteratively is worse code for no benefit — as long as you've bounded the depth.

## Checklist
- [ ] I can state the base case and the shrinking step for any recursion I write, and name the three ways termination fails.
- [ ] I can trace descend/unwind and say what is alive at maximum depth.
- [ ] I can compute total calls and max depth separately, and say which is time and which is space.
- [ ] I can convert a linear recursion to a loop, and a two-call recursion to a loop with an explicit stack.
- [ ] I know Go has no TCO and can demonstrate it by measurement.
- [ ] I can add a memo to a recursion and predict the resulting call count from the number of distinct subproblems.
- [ ] I always clone what I record in a backtracking search, and I can explain why the bug hides at cap 0.
- [ ] I can write the choose/explore/un-choose skeleton from memory and say where the pruning check goes.
- [ ] I know a stack overflow is fatal and uncatchable, and I bound depth on untrusted input.
- [ ] I can write a recursive-descent parser and explain where precedence and associativity come from.

## Resources
- Effective Go — functions, defer, and closures: https://go.dev/doc/effective_go
- `runtime/debug.SetMaxStack` — the goroutine stack limit: https://pkg.go.dev/runtime/debug#SetMaxStack
- Go FAQ — why there is no tail-call optimisation: https://go.dev/doc/faq
- `runtime.MemStats.StackInuse`: https://pkg.go.dev/runtime#MemStats
- `slices.Clone` — the fix for the backtracking bug: https://pkg.go.dev/slices#Clone
- CLRS, *Introduction to Algorithms* — ch. 4 (divide & conquer), ch. 15 (dynamic programming).
- Recursive descent parsing: https://en.wikipedia.org/wiki/Recursive_descent_parser
- Examples: [examples/05-recursion](examples/05-recursion/) (16).
- Next: **Part 2 — Linear Structures** starts at [06 — Dynamic Arrays & Slices](06-arrays-slices.md), where the foundations stop being theory and start being data structures.
