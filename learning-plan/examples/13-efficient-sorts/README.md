# Step 13 — Efficient Sorts: Divide & Conquer · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- Examples marked **Output:** are deterministic: your output should match character for character.
  Most of this lesson is **comparison counts**, which belong to the algorithm, not the machine.
- Examples marked **Sample output:** print timings, which vary by machine and run.

**Every example also has a Rust twin**, in the same tier file: Go → Go output → Rust → Rust output →
a short *Go vs Rust* note. The twins are std-only single files, built with `rustc --edition 2024 -O`, and
each one passed `rustfmt`, `clippy -D warnings` and a real run before being added; the Rust output
blocks are the binary's real stdout, captured — never typed. Where the two languages disagree, the note
says so and the Rust program prints what *its* numbers show, not Go's.

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste a Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the three algorithms, one idea at a time

- [1. The merge step, and the inversions it counts](1-easy.md#1-the-merge-step-and-the-inversions-it-counts)
- [2. Top-down merge sort, one buffer](1-easy.md#2-top-down-merge-sort-one-buffer)
- [3. Stability is one character](1-easy.md#3-stability-is-one-character)
- [4. Lomuto and Hoare partitions](1-easy.md#4-lomuto-and-hoare-partitions)
- [5. Quicksort, and the input that makes it quadratic](1-easy.md#5-quicksort-and-the-input-that-makes-it-quadratic)
- [6. Heapsort in place](1-easy.md#6-heapsort-in-place)

### 🟡 [Medium](2-medium.md) — the versions you'd write, the fixes, and what the stdlib runs

- [7. Bottom-up and natural merge sort](2-medium.md#7-bottom-up-and-natural-merge-sort)
- [8. Six pivot rules, six input shapes](2-medium.md#8-six-pivot-rules-six-input-shapes)
- [9. Three-way partition for duplicates](2-medium.md#9-three-way-partition-for-duplicates)
- [10. Quickselect: the k-th smallest without sorting](2-medium.md#10-quickselect-the-k-th-smallest-without-sorting)
- [11. What `slices.Sort` actually runs](2-medium.md#11-what-slicessort-actually-runs)

### 🔴 [Hard](3-hard.md) — the adversary, the hybrid, the disk, the race, the capstone

- [12. McIlroy's killer adversary](3-hard.md#12-mcilroys-killer-adversary)
- [13. Introsort, then the pdqsort tricks](3-hard.md#13-introsort-then-the-pdqsort-tricks)
- [14. External sort: runs on disk and a k-way merge](3-hard.md#14-external-sort-runs-on-disk-and-a-k-way-merge)
- [15. Comparisons are not time: the full race](3-hard.md#15-comparisons-are-not-time-the-full-race)
- [16. Capstone: `Sort`, `SortStable`, `Select`, built, proved, measured](3-hard.md#16-capstone-sort-sortstable-select-built-proved-measured)

## The thread through the lesson

1–3 are merge sort: the merge step, the recursion around it, and the one character that decides
stability. 4–5 are quicksort: the partition, and the input that breaks a fixed pivot. 6 is heapsort,
the one with the guarantee. 7–9 are the fixes — natural runs, better pivots, a third partition for
duplicates — and 10 is the same partition used to *select* instead of sort. 11 opens up what
`slices.Sort` and Rust's sorts actually run. 12 attacks them, 13 rebuilds the hybrid that survives the
attack, 14 takes the merge to disk, 15 races everything, and 16 assembles it.

> Lesson: [13-efficient-sorts.md](../../13-efficient-sorts.md) · Previous: [12 — Elementary Sorts](../12-elementary-sorts/README.md)
