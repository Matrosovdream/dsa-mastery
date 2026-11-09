# Step 18 — Binary Search Trees · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added. Several use Go 1.23's
range-over-func, so `go mod init scratch` should be run with a Go that is 1.23 or newer.

Examples marked **Output:** are deterministic — most of this lesson prints comparison, visit and
height counts, which your run should match character for character. **Sample output:** ones print
timings.

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

### 🟢 [Easy](1-easy.md) — the invariant, insert, in-order, neighbours, delete, and the failure mode

- [1. The invariant, and search as a descent](1-easy.md#1-the-invariant-and-search-as-a-descent)
- [2. Insert is a failed search](1-easy.md#2-insert-is-a-failed-search)
- [3. In-order is sorted](1-easy.md#3-in-order-is-sorted)
- [4. Floor, ceiling, successor and predecessor](1-easy.md#4-floor-ceiling-successor-and-predecessor)
- [5. Delete, case by case](1-easy.md#5-delete-case-by-case)
- [6. Sorted input makes a linked list](1-easy.md#6-sorted-input-makes-a-linked-list)

### 🟡 [Medium](2-medium.md) — validation, order statistics, ranges, balanced construction, two iterators

- [7. Validation: the bounds trick, not the naive check](2-medium.md#7-validation-the-bounds-trick-not-the-naive-check)
- [8. The k-th smallest, and what a size field buys](2-medium.md#8-the-k-th-smallest-and-what-a-size-field-buys)
- [9. Range queries: the path plus the answer](2-medium.md#9-range-queries-the-path-plus-the-answer)
- [10. Building a balanced tree from a sorted slice](2-medium.md#10-building-a-balanced-tree-from-a-sorted-slice)
- [11. Two-sum on a BST, with O(h) memory](2-medium.md#11-two-sum-on-a-bst-with-oh-memory)

### 🔴 [Hard](3-hard.md) — delete without recursion, the deletion drift, the workload, the problem set, the capstone

- [12. Delete without recursion: a cursor and a held pointer](3-hard.md#12-delete-without-recursion-a-cursor-and-a-held-pointer)
- [13. Hibbard's drift: random deletes unbalance the tree](3-hard.md#13-hibbards-drift-random-deletes-unbalance-the-tree)
- [14. Ordered and dynamic: a BST against a sorted slice](3-hard.md#14-ordered-and-dynamic-a-bst-against-a-sorted-slice)
- [15. The problems BSTs own: recover, closest, count smaller](3-hard.md#15-the-problems-bsts-own-recover-closest-count-smaller)
- [16. Capstone: `Tree[K,V]`, built, proved, measured](3-hard.md#16-capstone-treekv-built-proved-measured)

## The thread through the lesson

1 is the invariant and what a search costs; 2 is insert, and why the shape is the insertion order. 3
and 4 are what order buys — a sorted walk and every neighbour query. 5 is delete, and 6 is the way a
plain BST dies. 7–11 are the classic problems: validation, k-th smallest, ranges, building one
balanced, and two-sum with two iterators. 12 is delete done without recursion, and what a held pointer
sees. 13 is the slow lopsidedness that deletion itself causes. 14 puts the tree against a sorted slice
on the workload both earlier lessons deferred, 15 is the problem set, and 16 assembles all of it.

> Lesson: [18-binary-search-trees.md](../../18-binary-search-trees.md) · Previous: [17 — Binary Trees & Traversals](../17-binary-trees/README.md)
