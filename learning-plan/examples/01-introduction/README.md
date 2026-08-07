# Step 01 — Introduction · Examples

A library of **14 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program: read the concept and steps, then **retype the code block** into a scratch
folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Every example was compiled, `gofmt`-checked, `go vet`-ed and run before being added — the **Output**
under each one is real stdout. Examples 1–5, 8 and 14 are fully **deterministic** and should match
character for character; 6, 7, 9–13 print timings and are labelled **sample output** (they were
produced on an Apple M-series laptop with Go 1.26.3 — what must match is the shape, not the digits).

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–5 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 6–10 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 11–14 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — count, don't guess

- [1. Same data, different structure](1-easy.md#1-same-data-different-structure)
- [2. Count the work, not the clock](1-easy.md#2-count-the-work-not-the-clock)
- [3. Quadratic vs linear, in operations](1-easy.md#3-quadratic-vs-linear-in-operations)
- [4. What the growth curves actually cost](1-easy.md#4-what-the-growth-curves-actually-cost)
- [5. Trading space for time](1-easy.md#5-trading-space-for-time)

### 🟡 [Medium](2-medium.md) — measuring in Go

- [6. From counting to the clock](2-medium.md#6-from-counting-to-the-clock)
- [7. Benchmarking from a plain program](2-medium.md#7-benchmarking-from-a-plain-program)
- [8. Counting allocations](2-medium.md#8-counting-allocations)
- [9. Finding the crossover point](2-medium.md#9-finding-the-crossover-point)
- [10. What "O(n) space" actually costs](2-medium.md#10-what-on-space-actually-costs)

### 🔴 [Hard](3-hard.md) — when measurements lie

- [11. Big-O lies at small n](3-hard.md#11-big-o-lies-at-small-n)
- [12. The benchmark that measured nothing](3-hard.md#12-the-benchmark-that-measured-nothing)
- [13. `[]T` vs `[]*T`: the pointer tax](3-hard.md#13-t-vs-t-the-pointer-tax)
- [14. The brute-force oracle](3-hard.md#14-the-brute-force-oracle)

## The five results worth remembering

| # | Finding | Number |
|---|---------|--------|
| 3 | 10× the input costs an O(n²) algorithm ~100× the work | 5000× more ops than O(n) at n=10,000 |
| 9 | A map only beats a linear scan above a crossover | n ≥ 16 |
| 11 | An O(n²) sort beats `slices.Sort` below a crossover | n ≤ 16–32 (Go's own cutoff: 12) |
| 12 | `b.Loop` has its own per-iteration cost — always run an empty control | ~1.6 ns/op floor |
| 13 | Pointers cost cache locality *and* GC scan time | 2.7× slower, ~59× more GC work |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
