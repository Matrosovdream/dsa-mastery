# Step 03 — Complexity Analysis · Examples

A library of **16 runnable examples**, split into three files by difficulty.

**Two shapes in this lesson.** Examples 1–13 are single-file `package main` programs; examples
**14–16 are `go test` packages** (a folder of two files each). Every example says which, and gives
the exact command.

```bash
# examples 1-13
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .

# examples 14-16
mkdir -p /tmp/dsa-t/ex14 && cd /tmp/dsa-t   # once: go mod init scratch
go test -v ./ex14
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

**Most of this lesson counts operations rather than timing them**, which is why so much of it is
exactly reproducible:

- **Deterministic** (1–8, 10–12, and the count tables in 13): match character for character.
- **Sample output** (9, 13's timings, 14–16): machine-dependent. Produced on an Apple M4, Go 1.26.3.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the notation, and how to read it off numbers

- [1. Three bounds on one function](1-easy.md#1-three-bounds-on-one-function)
- [2. The case and the bound are different axes](1-easy.md#2-the-case-and-the-bound-are-different-axes)
- [3. Constants and lower-order terms drop](1-easy.md#3-constants-and-lower-order-terms-drop)
- [4. The ratio test](1-easy.md#4-the-ratio-test)
- [5. Nested loops that aren't quadratic](1-easy.md#5-nested-loops-that-arent-quadratic)
- [6. Sequential adds, nested multiplies](1-easy.md#6-sequential-adds-nested-multiplies)

### 🟡 [Medium](2-medium.md) — amortized, space, recurrences

- [7. Why append is amortized O(1)](2-medium.md#7-why-append-is-amortized-o1)
- [8. Amortized is not average, and is not worst case](2-medium.md#8-amortized-is-not-average-and-is-not-worst-case)
- [9. The call stack is space](2-medium.md#9-the-call-stack-is-space)
- [10. Auxiliary space vs total space](2-medium.md#10-auxiliary-space-vs-total-space)
- [11. Recursion trees](2-medium.md#11-recursion-trees)

### 🔴 [Hard](3-hard.md) — solving, spotting, and measuring

- [12. The master theorem](3-hard.md#12-the-master-theorem)
- [13. Quadratic hiding in a single loop](3-hard.md#13-quadratic-hiding-in-a-single-loop)
- [14. Measuring a complexity class](3-hard.md#14-measuring-a-complexity-class)
- [15. What amortized O(1) costs the unlucky caller](3-hard.md#15-what-amortized-o1-costs-the-unlucky-caller)
- [16. A complexity regression gate](3-hard.md#16-a-complexity-regression-gate)

## The ratio test, in one table

Double n, divide the costs. This is the tool the whole lesson is built on:

| ratio T(2n)/T(n) | class | seen in |
|---|---|---|
| ≈ 1 | O(log n) | ex 4, 14 |
| ≈ 2 | O(n) | ex 4, 14, 16 |
| ≈ 2.1–2.3 | O(n log n) | ex 4, 5, 14 |
| ≈ 4 | O(n²) | ex 4, 5, 6, 16 |
| ≈ 8 | O(n³) | ex 4 |

Exact on operation counts. On **timings**, the O(n) and O(n log n) bands merge — see example 14.

## The six results worth remembering

| # | Finding | Number |
|---|---------|--------|
| 1 | `O(n²)` is a *true* claim about a linear function — only Θ pins it down | — |
| 5 | Two double loops that look quadratic are Θ(n log n) | ratios 2.25 and 2.24, not 4 |
| 7 | Go's slice growth tapers to ~1.25, so amortized append is ~4 copies, not 1 | 4.02 copies/append |
| 8 | "Amortized O(1)" and "worst case O(n)" are both true at once | 4.02 vs 88,064 copies |
| 13 | A single loop can be Θ(n²) with nothing in its shape to say so | 6.971 s vs 116 µs |
| 15 | Amortized cost hides the tail entirely | max/p50 ≈ 8,900× |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
