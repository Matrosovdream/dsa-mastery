# Step 06 — Dynamic Arrays & Slices · Examples

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

- **Deterministic** (1–9, 15): these count and rearrange rather than timing, so your output should match character for character.
- **Sample output** (10–14, 16): timings and heap measurements, from an Apple M4 with Go 1.26.3.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the operations and the idiom

- [1. The four operations](1-easy.md#1-the-four-operations)
- [2. Two ways to delete](1-easy.md#2-two-ways-to-delete)
- [3. The write-index idiom](1-easy.md#3-the-write-index-idiom)
- [4. Reverse and rotate in place](1-easy.md#4-reverse-and-rotate-in-place)
- [5. A sorted slice is a data structure](1-easy.md#5-a-sorted-slice-is-a-data-structure)
- [6. What the stdlib already has](1-easy.md#6-what-the-stdlib-already-has)

### 🟡 [Medium](2-medium.md) — building it yourself

- [7. Building `Vector[T]`, and deriving amortized Θ(1)](2-medium.md#7-building-vectort-and-deriving-amortized-θ1)
- [8. Shrinking, and how it thrashes](2-medium.md#8-shrinking-and-how-it-thrashes)
- [9. Choosing the growth factor](2-medium.md#9-choosing-the-growth-factor)
- [10. `buf[:0]` — the best optimisation in a hot loop](2-medium.md#10-buf0--the-best-optimisation-in-a-hot-loop)
- [11. Three ways to lay out a grid](2-medium.md#11-three-ways-to-lay-out-a-grid)

### 🔴 [Hard](3-hard.md) — choosing, dodging, and proving

- [12. Sorted slice vs map, with the workload as a variable](3-hard.md#12-sorted-slice-vs-map-with-the-workload-as-a-variable)
- [13. The gap buffer](3-hard.md#13-the-gap-buffer)
- [14. The accidental quadratic](3-hard.md#14-the-accidental-quadratic)
- [15. Partition, and the invariant that makes it work](3-hard.md#15-partition-and-the-invariant-that-makes-it-work)
- [16. Capstone: `Vector[T]`, all three passes](3-hard.md#16-capstone-vectort-all-three-passes)

## The costs, in one table

| Operation | Cost | Example |
|---|---|---|
| `xs[i]` | Θ(1) — 1.7 ns at any n | 1, 16 |
| `append` | Θ(1) amortized | 7 |
| Insert / Delete at i | **Θ(n)** | 1 |
| Swap-delete | **Θ(1)**, destroys order | 2 |
| Delete k, one pass | Θ(n) | 3, 14 |
| Delete k, in a loop | **Θ(n·k)** | 14 |
| Sorted insert | Θ(log n) find + Θ(n) shift | 5 |
| Reverse / rotate | Θ(n) time, **Θ(1)** space | 4 |
| Partition (2- or 3-way) | Θ(n) time, Θ(1) space | 15 |
| Gap-buffer insert at cursor | **Θ(1)** amortized | 13 |

## The six results worth remembering

| # | Finding | Number |
|---|---------|--------|
| 3 | Deleting inside a forward loop is not just slow, it's **wrong** | skips elements |
| 7 | Amortized Θ(1), derived by counting | copies/push → **1.00**; +k growth → 781 and climbing |
| 8 | "Shrink to fit when half empty" destroys the amortized bound | **2048 copies** per push/pop pair |
| 10 | `buf[:0]` reuse | **3.2×** faster, 13 allocations → **0** |
| 14 | `slices.Delete` in a loop | **355×** slower at n=64,000, and growing |
| 16 | A model oracle over 20,000 random ops is what actually proves a structure | — |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
