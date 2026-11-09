# Step 19 — Balanced Trees & B-Trees · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. How much disorder protects a BST**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. How much disorder protects a BST
- [ ] 2. Rotations: reshape, never reorder
- [ ] 3. AVL: the four rebalance cases
- [ ] 4. The sparsest AVL tree is a Fibonacci tree
- [ ] 5. Red-black: five rules, and a checker
- [ ] 6. A coin flip per node: skip-list levels

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. AVL tree, built and proved
- [ ] 8. Red-black insertion with the CLRS fix-up
- [ ] 9. Treap: priorities instead of balance factors
- [ ] 10. Skip list, built and measured in steps
- [ ] 11. B-tree: fan-out against nodes touched

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. AVL deletion: the Fibonacci worst case
- [ ] 13. B+ tree: the leaf chain, and what it does not buy
- [ ] 14. Pages read: a buffer pool in front of the tree
- [ ] 15. Comparisons are not node visits
- [ ] 16. Capstone: `OrderedMap` as a B-tree, proved and measured

## The three-pass build

In `practice/19-balanced-trees/`:

- [ ] **Build it** — `rotateLeft` / `rotateRight`, and a plain-BST `Insert` as the control
- [ ] **Build it** — AVL: stored heights, all four rebalance cases, then `Delete`
- [ ] **Build it** — red-black insertion (CLRS fix-up) with a parent link or an index arena
- [ ] **Build it** — a treap with seeded priorities, and a skip list for p = 1/2 and p = 1/4
- [ ] **Build it** — a B-tree of minimum degree `t`: split on insert, borrow and merge on delete
- [ ] **Build it** — `OrderedMap[K, V]` on the B-tree: `Get`, `Put`, `Delete`, `Floor`, `Ceil`, `Range`
- [ ] **Prove it** — a sorted-slice oracle over ≥ 100,000 random operations on a **narrow key range**
- [ ] **Prove it** — a checker run after *every* operation: order, stored heights or colours, balance, counts
- [ ] **Prove it** — the checker catches one planted bug per structure, and names the broken rule
- [ ] **Prove it** — the B-tree at `t = 2`, where every split, borrow and merge occurs
- [ ] **Prove it** — every permutation of 1…6 through AVL insertion: at most one fix-up per insert
- [ ] **Measure it** — heights and rotation counts for sorted and random input, next to the theorem's bound
- [ ] **Measure it** — comparisons *and* node visits per search, for every structure on one workload
- [ ] **Measure it** — page reads per lookup behind an LRU buffer pool of several sizes
- [ ] **Measure it** — against a sorted slice and a `map` in Go, and against `BTreeMap` in Rust
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What does inserting 1…n into a plain BST cost? | n(n−1)/2 comparisons — the height is n |
| The fewest levels any binary tree of n nodes can have? | ⌈lg(n+1)⌉ |
| What does a rotation change, and what never changes? | three subtrees' depths · the in-order sequence |
| The four AVL cases? | left-left, right-right (one rotation) · left-right, right-left (two) |
| How do you pick single or double? | the sign of the heavy child's balance factor |
| Fix-ups per AVL insert? Per delete? | at most one · up to one per level |
| Fewest nodes in an AVL tree of height h? | N(h) = N(h−1) + N(h−2) + 1 = F(h+2) − 1 |
| The AVL height bound? | h < 1.4405·lg(n+2) − 0.3277 |
| The five red-black rules? | red or black · root black · nil black · red has black children · equal black height |
| The red-black height bound? | ≤ 2·lg(n+1) |
| Rotations per red-black insert / delete? | at most 2 / at most 3 |
| A red-black tree is which B-tree in disguise? | a 2-3-4 tree (t = 2) |
| Why is a treap's shape independent of insertion order? | keys plus priorities determine it uniquely |
| A skip list's expected search cost? | (1/p)·log base 1/p of n comparisons |
| Its pointers per node? | 1/(1−p) on average |
| Keys per B-tree node, minimum degree t? | t−1 … 2t−1 (root excepted) |
| How does a B-tree grow? | a root split adds a level, at the top |
| What does a B-tree save? | node visits — not comparisons |
| What does a B+ tree change? | values only in leaves, higher fan-out, a leaf chain for range scans |
| What does Go's standard library ship for ordered data? | nothing — `container/` is `heap`, `list`, `ring` |
| Rust's `BTreeMap` — `B`, `CAPACITY`, in-node search? | 6 · 11 · linear scan |

## Numbers to find on your own machine

Treat each as a prediction to test — several of mine may be wrong.

- [ ] How much disorder a nearly-sorted input can have before a plain BST stops being a path
- [ ] The height of an AVL tree after sorted insertion, against random insertion, against the bound
- [ ] The rotations per insert under sorted and under random input, for AVL, red-black and a treap
- [ ] The largest number of rotations one AVL delete causes on the sparsest tree of height 16
- [ ] That a treap's shape is the same under three insertion orders — and its rotation counts are not
- [ ] Steps per search for p = 1/2 and p = 1/4, and the pointers per node each costs
- [ ] The fill factor of a B-tree under sorted insertion and under random insertion
- [ ] How many allocations `BTreeMap` makes for a million keys, and what that implies about its node fill
- [ ] The buffer-pool size at which a B-tree lookup falls to one page read
- [ ] Whether a B+ tree's leaf chain is adjacent in memory after incremental inserts and after a bulk load
- [ ] Comparisons per search against node visits per search, for a B-tree and for an AVL tree
- [ ] The size at which a sorted slice stops beating your tree for inserts

> Lesson: [19-balanced-trees.md](../../19-balanced-trees.md) · Index: [README.md](README.md)
