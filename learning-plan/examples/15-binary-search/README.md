# Step 15 — Binary Search & Its Variants · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

Examples marked **Output:** are deterministic — most of this lesson prints probe and comparison
counts, which your run should match character for character. **Sample output:** ones print timings.

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

### 🟢 [Easy](1-easy.md) — the invariant, the overflow, the bounds, and what the stdlib returns

- [1. The invariant, written down](1-easy.md#1-the-invariant-written-down)
- [2. The overflow in `(lo + hi) / 2`](1-easy.md#2-the-overflow-in-lo--hi--2)
- [3. Lower bound and upper bound](1-easy.md#3-lower-bound-and-upper-bound)
- [4. Which duplicate do you get?](1-easy.md#4-which-duplicate-do-you-get)
- [5. `sort.Search` is a predicate search](1-easy.md#5-sortsearch-is-a-predicate-search)
- [6. Three-way search is not faster](1-easy.md#6-three-way-search-is-not-faster)

### 🟡 [Medium](2-medium.md) — the classic problems: rotated arrays, searching the answer, matrices, peaks

- [7. Searching a rotated array](2-medium.md#7-searching-a-rotated-array)
- [8. Binary search on the answer: Koko's bananas](2-medium.md#8-binary-search-on-the-answer-kokos-bananas)
- [9. Ship packing and "minimize the maximum"](2-medium.md#9-ship-packing-and-minimize-the-maximum)
- [10. Two kinds of sorted matrix](2-medium.md#10-two-kinds-of-sorted-matrix)
- [11. Finding a peak in unsorted data](2-medium.md#11-finding-a-peak-in-unsorted-data)

### 🔴 [Hard](3-hard.md) — every boundary bug, floats, the faster layouts, galloping, the capstone

- [12. Every boundary bug, enumerated](3-hard.md#12-every-boundary-bug-enumerated)
- [13. Bisection on floats, and on their bits](3-hard.md#13-bisection-on-floats-and-on-their-bits)
- [14. Faster than the textbook: branchless and Eytzinger](3-hard.md#14-faster-than-the-textbook-branchless-and-eytzinger)
- [15. Exponential search and lopsided intersections](3-hard.md#15-exponential-search-and-lopsided-intersections)
- [16. Capstone: a timestamp index, built, proved, measured](3-hard.md#16-capstone-a-timestamp-index-built-proved-measured)

## The thread through the lesson

1 writes the invariant down and counts the probes it implies. 2 is the overflow, in both languages.
3–5 are the boundary search in its three forms — bounds, the stdlib's duplicates, and a predicate
with no array at all — and 6 is why the textbook early exit does not pay. 7–11 are the classic
problems, each one a boundary search in disguise. 12 runs every way to get the boundaries wrong.
13 searches real numbers, 14 makes the search faster without changing its probe count, 15 searches
when the answer is near the front, and 16 assembles all of it.

> Lesson: [15-binary-search.md](../../15-binary-search.md) · Previous: [14 — Linear-Time Sorts](../14-linear-sorts/README.md)
