# Step 14 — Linear-Time Sorts · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- Examples marked **Output:** are deterministic: your output should match character for character.
- **Sample output:** ones print timings, which will differ on your machine.

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

### 🟢 [Easy](1-easy.md) — the lower bound, the loophole, and the two building blocks

- [1. The decision-tree bound: log2(n!) comparisons](1-easy.md#1-the-decision-tree-bound-log2n-comparisons)
- [2. Counting sort: count, then rewrite](1-easy.md#2-counting-sort-count-then-rewrite)
- [3. Key-indexed counting: prefix sums and stability](1-easy.md#3-key-indexed-counting-prefix-sums-and-stability)
- [4. The k in O(n + k)](1-easy.md#4-the-k-in-on--k)
- [5. LSD radix sort, one digit at a time](1-easy.md#5-lsd-radix-sort-one-digit-at-a-time)
- [6. Why every LSD pass must be stable](1-easy.md#6-why-every-lsd-pass-must-be-stable)

### 🟡 [Medium](2-medium.md) — real key types, strings, buckets, and the problems they own

- [7. Choosing the radix: bits per pass](2-medium.md#7-choosing-the-radix-bits-per-pass)
- [8. Signed keys and floats: flip the bits](2-medium.md#8-signed-keys-and-floats-flip-the-bits)
- [9. Strings: LSD for fixed width, MSD for the rest](2-medium.md#9-strings-lsd-for-fixed-width-msd-for-the-rest)
- [10. Bucket sort and the uniform assumption](2-medium.md#10-bucket-sort-and-the-uniform-assumption)
- [11. The problems they own: maximum gap, top-K frequent, H-index](2-medium.md#11-the-problems-they-own-maximum-gap-top-k-frequent-h-index)

### 🔴 [Hard](3-hard.md) — the crossover, real keys, strings measured, in place, the capstone

- [12. The crossover against `slices.Sort`](3-hard.md#12-the-crossover-against-slicessort)
- [13. Real keys: timestamps, IDs and the passes you can skip](3-hard.md#13-real-keys-timestamps-ids-and-the-passes-you-can-skip)
- [14. Shared prefixes: MSD radix vs comparison sort](3-hard.md#14-shared-prefixes-msd-radix-vs-comparison-sort)
- [15. American flag sort: MSD radix in place](3-hard.md#15-american-flag-sort-msd-radix-in-place)
- [16. Capstone: `radix`, built, proved, measured](3-hard.md#16-capstone-radix-built-proved-measured)

## The thread through the lesson

1 proves the wall and names the loophole. 2–4 are counting sort — bare keys, records, and the `k` that
can ruin it. 5–6 build LSD radix sort from it and show that stability is its proof. 7–9 extend it to
real key types: wide integers, signed integers, floats and strings. 10 is the sort that needs a
distribution, and 11 the problems where counting *is* the answer. 12 finds where all of it starts to
pay, 13–15 are the real keys, the strings and the in-place variant, and 16 assembles it.

> Lesson: [14-linear-sorts.md](../../14-linear-sorts.md) · Previous: [13 — Efficient Sorts](../13-efficient-sorts/README.md)
