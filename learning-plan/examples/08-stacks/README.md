# Step 08 — Stacks · Examples

A library of **16 runnable examples**, split into three files by difficulty.

Examples 1–15 are single-file `package main` programs; **example 16 is a `go test` package** (three
files).

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .                                  # examples 1-15

mkdir -p /tmp/dsa-t/ex16 && cd /tmp/dsa-t # example 16
go test -v ./ex16
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- **Deterministic** (1, 3–7, 9, 10, 13): your output should match character for character.
- **Sample output** (2, 8, 11, 12, 14, 15, 16): timings, from an Apple M4 with Go 1.26.3.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the structure, and the problems that *are* stacks

- [1. A slice is already a stack](1-easy.md#1-a-slice-is-already-a-stack)
- [2. Stack[T], and what the wrapper costs](1-easy.md#2-stackt-and-what-the-wrapper-costs)
- [3. Balanced brackets](1-easy.md#3-balanced-brackets)
- [4. RPN evaluation](1-easy.md#4-rpn-evaluation)
- [5. Undo/redo](1-easy.md#5-undoredo)
- [6. Iterative DFS, and the one-line switch to BFS](1-easy.md#6-iterative-dfs-and-the-one-line-switch-to-bfs)

### 🟡 [Medium](2-medium.md) — parsing, augmentation, amortization

- [7. Shunting yard](2-medium.md#7-shunting-yard)
- [8. Min-stack in Θ(1)](2-medium.md#8-min-stack-in-θ1)
- [9. A queue from two stacks](2-medium.md#9-a-queue-from-two-stacks)
- [10. Writing down the call stack](2-medium.md#10-writing-down-the-call-stack)
- [11. Which implementation?](2-medium.md#11-which-implementation)

### 🔴 [Hard](3-hard.md) — the monotonic stack

- [12. The monotonic stack](3-hard.md#12-the-monotonic-stack)
- [13. The same shape, four more problems](3-hard.md#13-the-same-shape-four-more-problems)
- [14. Largest rectangle in a histogram](3-hard.md#14-largest-rectangle-in-a-histogram)
- [15. Trapping rain water, four ways](3-hard.md#15-trapping-rain-water-four-ways)
- [16. Capstone: Stack[T] and the algorithms](3-hard.md#16-capstone-stackt-and-the-algorithms)

## The giveaway phrases

| The problem says | Reach for |
|---|---|
| "the most recently opened / seen / pushed X" | **a stack** |
| "the oldest outstanding X" | a queue (lesson 09) |
| "the smallest outstanding X" | a heap (lesson 20) |
| "the nearest larger / smaller element beside it" | **a monotonic stack** |
| "a running maximum from each side" | two pointers (lesson 16) |

## The six results worth remembering

| # | Finding | Number |
|---|---------|--------|
| 2 | The generic wrapper is cheap, **not free** | ~1.6 ns/op over a bare slice |
| 6 | DFS and BFS differ by **one line** | `xs[len-1]` vs `xs[0]` |
| 8 | Θ(1) min works because a stack's contents are strictly nested | aux stack: **15** entries for 100k random pushes, **100k** for descending |
| 9 | Two-stack queue: amortized Θ(1) *and* worst-case Θ(n) | n transfers per 2n ops; one dequeue moves n |
| 12 | Random data does **not** exercise the worst case | brute is ~3× on random, **2509×** on descending |
| 14 | Largest rectangle | **616×** faster than brute at n=3200 |

## Two traps this lesson hit

- **Benchmarking a monotonic stack on random input proves nothing.** The brute force is nearly linear there, because the next greater element is usually one step away. The gap only appears on adversarial input — where it is 20,476,800 comparisons against 6,399.
- **A `uint64` heap-object delta underflows** when the GC frees more between two readings than the build allocated. It printed 18446744073709551613 before being made signed — the same trap as lesson 06, example 15.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
