# Step 12 — Elementary Sorts · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added. Examples marked
**Output:** are deterministic — your output should match character for character; **Sample output:**
ones print timings, which vary by machine. Most of this lesson is operation counts — comparisons,
swaps, writes, inversions — so most of it is deterministic.

**Every example also has a Rust twin**, in the same tier file: Go → Go output → Rust → Rust output →
a short *Go vs Rust* note.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the three sorts, their invariants, stability and inversions

- [1. Bubble sort, and the early exit](1-easy.md#1-bubble-sort-and-the-early-exit)
- [2. Selection sort never adapts](1-easy.md#2-selection-sort-never-adapts)
- [3. Insertion sort: shift, don't swap](1-easy.md#3-insertion-sort-shift-dont-swap)
- [4. Loop invariants, checked at every step](1-easy.md#4-loop-invariants-checked-at-every-step)
- [5. Stability, and the one-character bug](1-easy.md#5-stability-and-the-one-character-bug)
- [6. Inversions: what insertion sort really pays](1-easy.md#6-inversions-what-insertion-sort-really-pays)

### 🟡 [Medium](2-medium.md) — counted head to head, nearly sorted data, binary insertion, multi-key sorts, the comparator contract

- [7. Three sorts, six inputs, every operation counted](2-medium.md#7-three-sorts-six-inputs-every-operation-counted)
- [8. Nearly sorted input: where insertion sort wins](2-medium.md#8-nearly-sorted-input-where-insertion-sort-wins)
- [9. Binary insertion sort: fewer comparisons, same moves](2-medium.md#9-binary-insertion-sort-fewer-comparisons-same-moves)
- [10. Multi-key sorting needs stability](2-medium.md#10-multi-key-sorting-needs-stability)
- [11. The comparator contract](2-medium.md#11-the-comparator-contract)

### 🔴 [Hard](3-hard.md) — inside the standard library, past the adjacent-swap barrier, and the capstone

- [12. Catching the stdlib's insertion-sort cutoff](3-hard.md#12-catching-the-stdlibs-insertion-sort-cutoff)
- [13. What the stdlib does with presorted input](3-hard.md#13-what-the-stdlib-does-with-presorted-input)
- [14. Shell sort: insertion sort with a gap](3-hard.md#14-shell-sort-insertion-sort-with-a-gap)
- [15. When writes cost more than comparisons](3-hard.md#15-when-writes-cost-more-than-comparisons)
- [16. Capstone: `InsertionSortFunc`, built, proved, measured](3-hard.md#16-capstone-insertionsortfunc-built-proved-measured)

> Lesson: [12-elementary-sorts.md](../../12-elementary-sorts.md) · Previous: [11 — Strings, Runes & Bytes](../11-strings-bytes/README.md)
