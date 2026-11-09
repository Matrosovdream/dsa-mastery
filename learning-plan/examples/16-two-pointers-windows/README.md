# Step 16 — Two Pointers, Sliding Window & Prefix Sums · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

Examples marked **Output:** are deterministic and should match character for character; **Sample
output:** ones print timings.

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

### 🟢 [Easy](1-easy.md) — two pointers in their two shapes, a fixed window, and the prefix-sum family

- [1. Pair sum: two pointers walking inward](1-easy.md#1-pair-sum-two-pointers-walking-inward)
- [2. Dedupe in place, keeping at most k](1-easy.md#2-dedupe-in-place-keeping-at-most-k)
- [3. A fixed window: add one, drop one](1-easy.md#3-a-fixed-window-add-one-drop-one)
- [4. Prefix sums: a range sum in Θ(1)](1-easy.md#4-prefix-sums-a-range-sum-in-θ1)
- [5. Prefix sums that overflow, and subtractions that underflow](1-easy.md#5-prefix-sums-that-overflow-and-subtractions-that-underflow)
- [6. Difference arrays: range add in Θ(1)](1-easy.md#6-difference-arrays-range-add-in-θ1)

### 🟡 [Medium](2-medium.md) — the classic problems: container, Dutch flag, both variable windows, 2-D prefix sums

- [7. Container with most water: why the shorter wall goes](2-medium.md#7-container-with-most-water-why-the-shorter-wall-goes)
- [8. The Dutch national flag, and the one-line bug](2-medium.md#8-the-dutch-national-flag-and-the-one-line-bug)
- [9. Minimum window substring](2-medium.md#9-minimum-window-substring)
- [10. The longest window: at most k distinct](2-medium.md#10-the-longest-window-at-most-k-distinct)
- [11. 2-D prefix sums and inclusion-exclusion](2-medium.md#11-2-d-prefix-sums-and-inclusion-exclusion)

### 🔴 [Hard](3-hard.md) — counting windows, 3Sum, when a window is legal, what has no inverse, the capstone

- [12. Counting subarrays: exactly k as a difference](3-hard.md#12-counting-subarrays-exactly-k-as-a-difference)
- [13. 3Sum: two pointers inside a loop](3-hard.md#13-3sum-two-pointers-inside-a-loop)
- [14. Which technique? A monotonicity audit](3-hard.md#14-which-technique-a-monotonicity-audit)
- [15. Beyond sums: XOR, remainders, and what has no inverse](3-hard.md#15-beyond-sums-xor-remainders-and-what-has-no-inverse)
- [16. Capstone: the toolkit, built, proved, measured](3-hard.md#16-capstone-the-toolkit-built-proved-measured)

## The thread through the lesson

1 and 2 are the two shapes of two pointers — converging on a sorted array, and a read pointer chasing a
write pointer — and 3 is the fixed window that adds one element and drops one. 4–6 are the prefix-sum
family: a range sum in one subtraction, the integer limits of that subtraction in both languages, and
its inverse, the difference array. 7 and 8 are two pointers with a proof obligation: which wall to
discard, and the invariant that makes the Dutch flag finish in exactly n steps. 9 and 10 are the two
shapes of variable window, 11 is prefix sums in two dimensions. 12 counts windows, 13 puts the pair-sum
loop inside another loop, 14 tests by brute force *when* a window is legal at all, 15 asks which
operations a prefix array can answer, and 16 assembles the toolkit.

> Lesson: [16-two-pointers-windows.md](../../16-two-pointers-windows.md) · Previous: [15 — Binary Search & Its Variants](../15-binary-search/README.md)
