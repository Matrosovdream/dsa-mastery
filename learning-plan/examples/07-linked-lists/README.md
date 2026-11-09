# Step 07 — Linked Lists · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- **Deterministic** — examples marked **Output:** should match your run character for character.
- **Sample output** — examples marked **Sample output:** print timings or memory, which vary by machine and run.

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

### 🟢 [Easy](1-easy.md) — the node, the surgery, and the three ways to kill the head special case

- [1. The node, and `nil` as the empty list](1-easy.md#1-the-node-and-nil-as-the-empty-list)
- [2. Wire first, splice second](1-easy.md#2-wire-first-splice-second)
- [3. The sentinel, and the special case it deletes](1-easy.md#3-the-sentinel-and-the-special-case-it-deletes)
- [4. A tail pointer, and why `PopBack` stays Θ(n)](1-easy.md#4-a-tail-pointer-and-why-popback-stays-θn)
- [5. Pointer-to-pointer: one loop for head and middle](1-easy.md#5-pointer-to-pointer-one-loop-for-head-and-middle)
- [6. The doubly linked ring](1-easy.md#6-the-doubly-linked-ring)

### 🟡 [Medium](2-medium.md) — the classic problems, and the standard library's list

- [7. Reverse in three lines, and the recursion that dies](2-medium.md#7-reverse-in-three-lines-and-the-recursion-that-dies)
- [8. Middle and Nth from end, in one pass](2-medium.md#8-middle-and-nth-from-end-in-one-pass)
- [9. Floyd's cycle detection](2-medium.md#9-floyds-cycle-detection)
- [10. Merging with zero allocations](2-medium.md#10-merging-with-zero-allocations)
- [11. `container/list`, read in the source](2-medium.md#11-containerlist-read-in-the-source)

### 🔴 [Hard](3-hard.md) — the honest verdict, the faster layouts, and the capstone

- [12. When a list beats a slice (rarely), measured](3-hard.md#12-when-a-list-beats-a-slice-rarely-measured)
- [13. The arena: links without pointers](3-hard.md#13-the-arena-links-without-pointers)
- [14. Palindrome in Θ(1) space, and putting it back](3-hard.md#14-palindrome-in-θ1-space-and-putting-it-back)
- [15. The unrolled list](3-hard.md#15-the-unrolled-list)
- [16. Capstone: an LRU cache, built, proved, measured](3-hard.md#16-capstone-an-lru-cache-built-proved-measured)

## The thread through the lesson

1 is the node and `nil`. 2 is the order of two assignments, which is the whole game. 3–5 remove the head
special case three ways — a sentinel, a tail pointer, and pointer-to-pointer — and 4 finds the one thing a tail
cannot fix, which is why 6 adds `prev` and closes the ring. 7–10 are the classics: reverse, two pointers, Floyd,
and the merge that allocates nothing. 11 reads Go's own list in the source. 12 is the honest verdict against a
slice; 13 and 15 are the two layouts that win back what 12 measured; 14 composes the classics; 16 is the LRU
cache, the structure a doubly linked list exists for.

> Lesson: [07-linked-lists.md](../../07-linked-lists.md) · Previous: [06 — Dynamic Arrays & Slices](../06-arrays-slices/README.md)
