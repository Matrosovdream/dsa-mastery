# Step 17 — Binary Trees & Traversals · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

Examples marked **Output:** are deterministic — most of this lesson prints counts (nodes visited, stack
depth, queue size) from trees built by a fixed-seed generator, which your run should match character for
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

### 🟢 [Easy](1-easy.md) — the node, the three orders, the stack and the queue, and the answers that come back up

- [1. The node, and `nil` as the empty tree](1-easy.md#1-the-node-and-nil-as-the-empty-tree)
- [2. Pre, in and post: where the visit goes](1-easy.md#2-pre-in-and-post-where-the-visit-goes)
- [3. The call stack, written as a loop](1-easy.md#3-the-call-stack-written-as-a-loop)
- [4. Level order: the queue holds one layer](1-easy.md#4-level-order-the-queue-holds-one-layer)
- [5. Height, depth and diameter](1-easy.md#5-height-depth-and-diameter)
- [6. Invert and prune: assign what the child returns](1-easy.md#6-invert-and-prune-assign-what-the-child-returns)

### 🟡 [Medium](2-medium.md) — the classic problems: path sums, two trees at once, LCA, serialization, rebuilding

- [7. Path sums: down, up and anywhere](2-medium.md#7-path-sums-down-up-and-anywhere)
- [8. Two trees at once: same, mirror, subtree](2-medium.md#8-two-trees-at-once-same-mirror-subtree)
- [9. Lowest common ancestor, and the missing node](2-medium.md#9-lowest-common-ancestor-and-the-missing-node)
- [10. Serialize and deserialize: one marker per nil](2-medium.md#10-serialize-and-deserialize-one-marker-per-nil)
- [11. Rebuild a tree from two traversals](2-medium.md#11-rebuild-a-tree-from-two-traversals)

### 🔴 [Hard](3-hard.md) — no-stack traversal, deep trees and `Drop`, three owners in Rust, iterators, the capstone

- [12. Morris traversal: in-order with no stack](3-hard.md#12-morris-traversal-in-order-with-no-stack)
- [13. A tree you did not build: frames, depth and Drop](3-hard.md#13-a-tree-you-did-not-build-frames-depth-and-drop)
- [14. One tree, three owners: Box, Rc and an arena](3-hard.md#14-one-tree-three-owners-box-rc-and-an-arena)
- [15. A traversal that can stop: iterators](3-hard.md#15-a-traversal-that-can-stop-iterators)
- [16. Capstone: a `Tree` toolkit, proved and measured](3-hard.md#16-capstone-a-tree-toolkit-proved-and-measured)

> Lesson: [17-binary-trees.md](../../17-binary-trees.md) · Previous: [16 — Two Pointers, Sliding Window & Prefix Sums](../16-two-pointers-windows/README.md)
