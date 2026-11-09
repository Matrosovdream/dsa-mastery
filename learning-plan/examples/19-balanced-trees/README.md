# Step 19 — Balanced Trees & B-Trees · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

Examples marked **Output:** are deterministic — most of this lesson prints heights, rotation counts,
comparison counts and page reads from fixed-seed generators, which your run should match character for
character. **Sample output:** ones print timings.

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

### 🟢 [Easy](1-easy.md) — how much disorder protects a BST, the rotation, AVL's cases and bound, red-black's rules, a coin flip

- [1. How much disorder protects a BST](1-easy.md#1-how-much-disorder-protects-a-bst)
- [2. Rotations: reshape, never reorder](1-easy.md#2-rotations-reshape-never-reorder)
- [3. AVL: the four rebalance cases](1-easy.md#3-avl-the-four-rebalance-cases)
- [4. The sparsest AVL tree is a Fibonacci tree](1-easy.md#4-the-sparsest-avl-tree-is-a-fibonacci-tree)
- [5. Red-black: five rules, and a checker](1-easy.md#5-red-black-five-rules-and-a-checker)
- [6. A coin flip per node: skip-list levels](1-easy.md#6-a-coin-flip-per-node-skip-list-levels)

### 🟡 [Medium](2-medium.md) — AVL, red-black, a treap, a skip list and a B-tree, each built and proved

- [7. AVL tree, built and proved](2-medium.md#7-avl-tree-built-and-proved)
- [8. Red-black insertion with the CLRS fix-up](2-medium.md#8-red-black-insertion-with-the-clrs-fix-up)
- [9. Treap: priorities instead of balance factors](2-medium.md#9-treap-priorities-instead-of-balance-factors)
- [10. Skip list, built and measured in steps](2-medium.md#10-skip-list-built-and-measured-in-steps)
- [11. B-tree: fan-out against nodes touched](2-medium.md#11-b-tree-fan-out-against-nodes-touched)

### 🔴 [Hard](3-hard.md) — the worst delete, the B+ tree, the page model, comparisons vs node visits, the capstone

- [12. AVL deletion: the Fibonacci worst case](3-hard.md#12-avl-deletion-the-fibonacci-worst-case)
- [13. B+ tree: the leaf chain, and what it does not buy](3-hard.md#13-b-tree-the-leaf-chain-and-what-it-does-not-buy)
- [14. Pages read: a buffer pool in front of the tree](3-hard.md#14-pages-read-a-buffer-pool-in-front-of-the-tree)
- [15. Comparisons are not node visits](3-hard.md#15-comparisons-are-not-node-visits)
- [16. Capstone: `OrderedMap` as a B-tree, proved and measured](3-hard.md#16-capstone-orderedmap-as-a-b-tree-proved-and-measured)

## The thread through the lesson

1 asks how much disorder protects a plain BST (lesson 18 counted the sorted and random extremes). 2 is the rotation that every repair
is made of, 3–4 are AVL's four cases and the tallest tree it allows, and 5 is red-black's rules with a
checker that names the rule you broke. 6 is the coin flip that replaces all of it. 7–11 build the five
structures and prove each against a sorted-slice oracle and an invariant checker. 12 finds the delete
that makes AVL rotate at every level, 13–14 move to the page model — the B+ tree and a buffer pool in
front of it — 15 separates comparisons from node visits, and 16 ships the one that wins.

> Lesson: [19-balanced-trees.md](../../19-balanced-trees.md) · Previous: [18 — Binary Search Trees](../18-binary-search-trees/README.md)
