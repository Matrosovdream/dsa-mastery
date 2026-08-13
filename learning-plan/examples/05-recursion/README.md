# Step 05 — Recursion & the Call Stack · Examples

A library of **16 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added — the output blocks are
real stdout.

- **Deterministic** (1–5, 7, 8, 10–12, 16): these count and print rather than timing anything, so your output should match character for character.
- **Sample output** (6, 9, 13, 14, 15): stack measurements and timings, from an Apple M4 with Go 1.26.3.
- Examples **9 and 15 spawn a child process** to demonstrate a real stack overflow, so their tracebacks differ every run.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — termination and shape

- [1. Base case and inductive step](1-easy.md#1-base-case-and-inductive-step)
- [2. Descend, then unwind](1-easy.md#2-descend-then-unwind)
- [3. Every frame keeps its own locals](1-easy.md#3-every-frame-keeps-its-own-locals)
- [4. Total calls vs maximum depth](1-easy.md#4-total-calls-vs-maximum-depth)
- [5. Linear recursion is a loop in disguise](1-easy.md#5-linear-recursion-is-a-loop-in-disguise)
- [6. Go has no tail-call optimisation](1-easy.md#6-go-has-no-tail-call-optimisation)

### 🟡 [Medium](2-medium.md) — converting, collapsing, and crashing

- [7. Recursion → loop + explicit stack](2-medium.md#7-recursion--loop--explicit-stack)
- [8. Memoization, and the bridge to DP](2-medium.md#8-memoization-and-the-bridge-to-dp)
- [9. How deep can you go, and what happens at the end](2-medium.md#9-how-deep-can-you-go-and-what-happens-at-the-end)
- [10. The shared-slice bug](2-medium.md#10-the-shared-slice-bug)
- [11. Mutual recursion](2-medium.md#11-mutual-recursion)

### 🔴 [Hard](3-hard.md) — search, depth, and a parser

- [12. Backtracking: choose, explore, un-choose](3-hard.md#12-backtracking-choose-explore-un-choose)
- [13. How fast the input shrinks decides everything](3-hard.md#13-how-fast-the-input-shrinks-decides-everything)
- [14. What a function call actually costs](3-hard.md#14-what-a-function-call-actually-costs)
- [15. Surviving input you didn't choose](3-hard.md#15-surviving-input-you-didnt-choose)
- [16. Capstone: a recursive-descent evaluator](3-hard.md#16-capstone-a-recursive-descent-evaluator)

## The shapes, in one table

| Recursive call | Total calls | Max depth | Verdict |
|---|---|---|---|
| `f(n-1)` | Θ(n) | **Θ(n)** | make it a loop (ex 5) |
| `f(n/2)` | Θ(log n) | Θ(log n) | leave it |
| `f(n/2)` twice | Θ(n) | Θ(log n) | leave it (ex 13) |
| `f(n-1), f(n-2)` | Θ(φⁿ) | Θ(n) | **memoize** (ex 8) |
| branch per candidate | Θ(bᵈ) − pruning | Θ(d) | prune early (ex 12) |

## The six results worth remembering

| # | Finding | Number |
|---|---------|--------|
| 4 | Total calls and max depth answer different questions | 31 calls at depth 4 vs 25 calls at depth 24 |
| 6 | Go has **no** tail-call optimisation | the "tail-recursive" stack grows exactly like the non-tail one |
| 8 | Memoization collapses a repeating tree | `fib(35)`: 29,860,703 calls → **69** |
| 9 | A stack overflow is fatal, not a panic | ~32M frame limit; `recover()` never runs; exit 2 |
| 10 | Recording a working slice aliases every answer | 8 subsets → **4 distinct** |
| 12 | Pruning *is* the algorithm | 8-queens: 2,057 vs **19,173,961** nodes |

## Two measurement traps this lesson hit

Both are worth knowing before you write your own version:

- **A recursive closure costs ~6.7 KB per frame** against ~40 bytes for a plain function. Example 9's first draft used one, and the *measurement itself* overflowed 1 GB at depth 100,000.
- **The shared-slice bug hides at capacity 0.** With `[]int{}` every `append` reallocates and the broken code accidentally works. Preallocate — as lesson 03 tells you to — and it breaks.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
