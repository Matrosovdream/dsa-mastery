# 19 — Balanced Trees & B-Trees

> Part of **Part 4 — Trees**, and the lesson where the BST of [18](18-binary-search-trees.md) gets the
> guarantee it was missing. A plain BST is Θ(log n) *on average over insertion orders* and Θ(n) on the
> order real data tends to arrive in: sorted. Every structure here removes that "if" a different way —
> rotations that repair the shape after each change, random numbers that make the shape independent of
> the input, or nodes so wide the tree stays a few levels tall. The one-line thesis: **balance is a bill,
> and the structures differ in what they pay it with — rotations (AVL, red-black), random bits (treap,
> skip list) or fan-out (B-tree) — and which currency is cheap depends on whether a node visit costs a
> pointer chase, a cache line or a disk read.**

## Goals
- State why a plain BST's height depends on insertion order, and exactly what sorted input costs.
- Write a rotation from memory; say what it changes (three subtrees' depths) and what it never changes (in-order).
- Implement **AVL** insertion with all four rebalance cases, and say which need one rotation and which two.
- State the AVL height bound, the tree that meets it, and why insertion needs one fix-up but deletion may need many.
- State the five red-black rules, derive the 2·lg(n+1) bound from them, and relate them to a 2-3-4 tree.
- Explain **treaps** and **skip lists**, and what "balanced with high probability" does and does not promise.
- Describe a **B-tree**: node capacity, split, borrow/merge, growth at the root, height log_t n.
- Explain the disk/page model — why the unit of cost is a node visit — and what a **B+ tree** changes.
- Say what Go's standard library ships for ordered data (nothing) and what Rust's `BTreeMap` actually is.
- Prove a tree with a sorted-slice oracle **and** an invariant checker run after every operation.

## Concepts

- **Why balance matters.** A search costs the depth of the node it ends at, and in a plain BST that depth
  is a function of the *insertion order*, not of n. [Lesson 18](18-binary-search-trees.md)'s failure mode:
  inserting 1, 2, …, n builds a path, and building it costs exactly 0 + 1 + … + (n−1) = n(n−1)/2
  comparisons, while a random order gives expected depth Θ(log n). The trouble is that real keys —
  timestamps, auto-increment IDs, sorted imports — are not random, they are *nearly* sorted. The target is
  the floor: a binary tree of height h holds at most 2ʰ − 1 nodes, so no tree has fewer than ⌈lg(n+1)⌉
  levels. Example 1 measures how much disorder it takes to protect a plain BST.

- **A rotation is the one repair operation.** Rotate right at `y` with `x = y.left`: `x` becomes the
  subtree root, `y` its right child, and `x`'s old right subtree becomes `y`'s left subtree — two child
  writes and one write to the parent's link. The three subtrees A (under `x`, left), B (between) and C
  (under `y`, right) keep their left-to-right order, so **BST order is preserved by construction**; what
  changes is depth: A rises a level, C sinks one, B stays. Left rotation is the mirror and the inverse.
  Θ(1) each. AVL, red-black and treap are all "insert like a BST, then rotate on the way back up".

- **AVL: balance by height.** Invariant: at every node, |height(left) − height(right)| ≤ 1. Store the
  height (or a balance factor in {−1, 0, +1}); on the way back up from an insert, recompute heights and
  stop at the first node whose factor reaches ±2. Four cases, named for where the extra height is:
  **left-left** and **right-right** take one rotation; **left-right** and **right-left** (the heavy
  child leans the *other* way) take two — rotate the child to straighten the zig-zag, then rotate the
  node. The test is the sign of the heavy child's factor. Insertion needs **at most one** fix-up per
  operation: afterwards the subtree is as tall as before the insert, so nothing above changes. Deletion is
  different — shrinking a subtree can shrink its parent, so a fix-up may be needed at every level of the
  path: **Θ(log n) rotations for one delete** (example 12 builds the tree where it happens).

- **How tall can an AVL tree be?** Let N(h) be the fewest nodes in an AVL tree of height h. The sparsest
  tree hangs a sparsest subtree of height h−1 and one of h−2 off the root, so N(h) = N(h−1) + N(h−2) + 1
  — Fibonacci: N(h) = F(h+2) − 1. Inverting gives **h < 1.4405·lg(n+2) − 0.3277** (height counted in
  nodes, a single node has height 1). The worst AVL tree is therefore about 44% taller than the ideal
  ⌈lg(n+1)⌉ and no more. It is a theorem, so the examples print the tallest tree they ever saw next to it.

- **Red-black: balance by colour.** Five rules: every node is red or black; the root is black; every nil
  leaf is black; a red node has only black children; every path from a node down to a nil leaf crosses the
  same number of black nodes (its **black height**). No two reds are adjacent and every path has equal
  black count, so the longest path is at most twice the shortest, and black height b forces at least
  2ᵇ − 1 nodes: **height ≤ 2·lg(n+1)**. That is a looser height than AVL's, bought for cheaper repair:
  insertion colours the new node red, recolours upward while its uncle is red, and finishes with at most
  **two rotations**; deletion needs at most **three** (CLRS ch. 13). The invariant is easiest to read as a
  **2-3-4 tree**: glue each black node to its red children and you get nodes of 1–3 keys, all at the same
  depth — a B-tree with t = 2 drawn as a binary tree. Java's `TreeMap` and the Linux kernel's `rbtree`
  are red-black trees.

- **Why Go does not ship one.** `container/` holds `heap`, `list` and `ring` — no tree — and `maps` has
  no ordered variant; sorted iteration is `slices.Sorted(maps.Keys(m))`, Θ(n log n) per pass. The Go
  distribution *does* contain balanced trees, just not importable ones: the runtime keeps goroutines
  blocked on a semaphore address in a **treap** whose priority is a random ticket (`runtime/sema.go`),
  and the compiler has a persistent ("applicative") height-balanced tree over `int32` keys
  (`cmd/compile/internal/abt/avlint32.go`). The source does not say *why* there is no `container/tree`,
  and this lesson does not guess. It gives the comparison that matters: in Go today you reach for a
  **sorted slice** (`slices.BinarySearch` + `slices.Insert`: Θ(n) per update, one contiguous move), a
  **map** if you never need order, or a third-party B-tree. Example 16 sets them side by side.

- **Treaps: let a random number do the balancing.** Give every key a random priority and keep the tree a
  BST by key *and* a heap by priority. Given the keys and priorities that tree is **unique**, so its
  shape cannot depend on insertion order: it is the BST you would get by inserting in priority order —
  a random BST, with expected depth Θ(log n) for *every* key set, sorted included. Insert = BST insert,
  then rotate the new node up while it outranks its parent; delete = rotate down to a leaf and cut.
  Expected rotations per update are constant (Aragon & Seidel). The price: the guarantee is
  probabilistic, and an adversary who can see the priorities wins.

- **Skip lists: balance without a tree.** A sorted linked list in which node `i` also sits on a geometric
  number of higher "express lanes": a node reaches level k with probability pᵏ⁻¹. Search starts on the top
  lane and drops a level whenever the next node overshoots. Expected cost is (1/p)·log_{1/p} n comparisons
  and 1/(1−p) forward pointers per node, so p = 1/2 and p = 1/4 share the same leading term, 2·lg n,
  for different memory (example 10 counts the real steps — the lower-order terms and the tail differ). Nothing is ever rebalanced — insert and delete splice the node in or out of each of
  its lanes — and ordered iteration is a walk along the bottom lane. The price is a worst case that is
  only probabilistic and nodes of variable size.

- **B-trees: pay with fan-out.** Everything above assumes one key per node and counts comparisons. If a
  node visit is the expensive thing — a disk page, a cache miss — change the unit: put many keys in a node
  and spend comparisons freely inside it. A **B-tree of minimum degree t** keeps every non-root node
  between t−1 and 2t−1 keys (t to 2t children), all leaves at the **same depth**. Insert descends to a
  leaf; a node that would overflow **splits** around its median and the median moves up. Only a root split
  adds a level, **at the top** — so a B-tree cannot become unbalanced. Delete mirrors it: an underflowing
  node **borrows** a key through the parent from a sibling, or **merges** with one; the root shrinks when
  it empties. Height ≤ log_t((n+1)/2) edges. Searching a node by binary search costs lg t comparisons per
  level over log_t n levels — lg n in total, the same as a BST. **A B-tree does not save comparisons; it
  saves node visits.**

- **The page model.** Storage is read in fixed-size pages, and one page read dwarfs any work on the page
  once it is in memory. Size a node to a page and a search costs (height) page reads — and the few top
  levels are small enough to stay in a cache, so most lookups reach the disk once. The same model holds
  one level up: an in-memory node of a few cache lines replaces a pointer chase per key. A **B+ tree**
  keeps values only in the leaves, only separator keys in internal nodes (more fit per page: higher
  fan-out), and links the leaves so a range scan is a walk along the bottom. PostgreSQL's default index is
  a B-tree variant and SQLite stores tables and indexes in B-trees.

- **Choosing.** AVL when lookups dominate and rotations are affordable; red-black when updates matter
  more; a treap or skip list when simplicity or lock-free variants matter; a B-tree when n is large or the
  data lives outside memory — and in Go, first ask whether a sorted slice or a map is enough.

### Go and Rust side by side

- **Rust ships an ordered map; Go does not.** `BTreeMap` is a B-tree with `B = 6`: a node holds at most
  `CAPACITY = 2*B - 1 = 11` keys, a non-root node at least `B - 1 = 5` (`alloc/src/collections/btree/node.rs`).
  The in-node search is a **linear scan** (`find_key_index`, `search.rs`); the docs call that "generally
  worse than a BST" in comparisons and add "In practice, however, performance is excellent."
- **API.** `range(a..b)`; floor is `range(..=k).next_back()`, ceiling `range(k..).next()`; `pop_first`. No
  rank/select (the k-th key means walking k entries); cursors sit behind the unstable `btree_cursors`. Keys
  need `Ord`, so `BTreeMap<f64, _>` does not compile; Go's `cmp.Ordered` admits `float64` — use `cmp.Compare`.
- **Your own tree in safe Rust.** `Option<Box<Node>>` + `take()` for one-owner, root-to-leaf recursions (AVL,
  treap); an **arena** of indices when nodes have parent links or several predecessors (red-black fix-up, skip
  list, B-tree pages). `BTreeMap` itself is neither: "we make do with lots of unsafety".
- **Dropping a deep `Box` tree recurses**: a degenerate tree can overflow the stack on `drop`, which Go's
  collector never does ([lesson 17](17-binary-trees.md), example 13).

## Complexity Table

| Operation / structure | Average | Worst | Note |
|---|---|---|---|
| Plain BST search / insert | Θ(log n) over random orders | **Θ(n)** | sorted input; building costs n(n−1)/2 comparisons |
| AVL search / insert / delete | Θ(log n) | **Θ(log n)** | height < 1.4405·lg(n+2) − 0.3277 |
| AVL rebalancing per insert | ≤ 1 fix-up | ≤ 1 fix-up | one single or one double rotation |
| AVL rebalancing per delete | — | **Θ(log n) rotations** | the Fibonacci tree, example 12 |
| Red-black search / insert / delete | Θ(log n) | **Θ(log n)** | height ≤ 2·lg(n+1) |
| Red-black rotations per insert / delete | ≤ 2 / ≤ 3 | | recolouring does the rest |
| Treap search / insert / delete | Θ(log n) expected | Θ(n) | priorities independent of keys |
| Skip list search / insert / delete | Θ(log n) expected | Θ(n) | (1/p)·log_{1/p} n comparisons, 1/(1−p) pointers/node |
| B-tree search | Θ(log_t n) node visits | same | Θ(log n) comparisons in total |
| B-tree insert / delete | Θ(log_t n) node visits | same | Θ(t) work per split, borrow or merge |
| B+ tree range scan of k keys | Θ(log_t n + k/t) node visits | | leaf chain |
| Sorted slice: search / insert | Θ(log n) / Θ(n) | | one contiguous move |
| Ordered iteration of a `map` | Θ(n log n) | | collect keys, then sort |
| k-th smallest, no augmentation | Θ(k) | | needs subtree sizes for Θ(log n) |

## Exercises
1. Write `rotateLeft` and `rotateRight`. Check that in-order is unchanged, that a rotation and its inverse restore the exact shape, and apply 100,000 random rotations to a random BST with a checker after each.
2. Insert 1…n, then n…1, then a random order, then a sorted order with k random swaps, into a plain BST. Print height against k.
3. Write AVL insertion with stored heights and all four cases. Insert every permutation of 1…6 and check the invariant after each insert; report the largest number of fix-ups in one insert.
4. Compute N(h) and the exact tallest AVL height for n = 10³ and 10⁶. Compare with the 1.4405·lg(n+2) − 0.3277 bound and with the tallest tree your inserts produce.
5. Add AVL delete. Check it against a sorted-slice oracle on a **narrow key range**, then build the sparsest tree of height 12 and find the delete that rotates most.
6. Write a red-black *checker* first: five rules plus BST order, each failing separately on a hand-built tree. Then implement the CLRS insert and run the checker after every insert.
7. Collapse a red-black tree into its 2-3-4 form and assert every leaf is at the same depth.
8. Build a treap. Insert the same keys in three orders with fixed priorities and show the shapes are identical; count rotations per order.
9. Build a skip list for p = 1/2 and p = 1/4. Count steps per search and pointers per node, and compare with (1/p)·log_{1/p} n and 1/(1−p).
10. Build an insert-only B-tree of minimum degree t. Check key counts, equal leaf depth and separator order. For t = 2, 6 and 64 report height and fill factor on sorted and random input.
11. Add B-tree delete (borrow, merge, root shrink). Run it at t = 2 against the oracle — the smallest degree exercises every case.
12. Build a B+ tree with a leaf chain. Compare pages read by range scans with your B-tree, then compare an incrementally built leaf chain with a bulk-loaded one.
13. Put an LRU buffer pool in front of the B-tree and print page misses per lookup against pool size.
14. Compare your best tree with a sorted slice (and, in Rust, `BTreeMap`) for lookup, insert, delete and range scan at three sizes. Say which operations the sorted slice wins.
15. Stretch — package `OrderedMap[K, V]` with `Get`, `Put`, `Delete`, `Floor`, `Ceil` and `Range`, and apply all three passes: a sorted-slice oracle, invariants after every operation, and a benchmark.
16. Stretch — rebuild the treap's insert and delete from `split` and `merge`, with no rotations.

## Best Practices & Pitfalls
- **In Go, start with a sorted slice or a map.** Reach for a tree when updates *and* ordered queries are both frequent and n is large; measure before you build (example 16).
- **Write the checker before the tree**, and run it after *every* operation in tests: order, stored heights or colours, balance, counts.
- **Pitfall — stale heights.** After a rotation, recompute the node that moved *down* first, then the one that moved up.
- **Pitfall — a single rotation on a zig-zag.** It leaves the tree exactly as unbalanced, mirrored. Check the heavy child's factor before choosing.
- **Pitfall — assuming delete costs what insert costs.** Insert needs one AVL fix-up; delete can walk to the root.
- **Pitfall — testing with distinct random keys only.** Duplicates, re-deletes and the empty tree are where the bugs live; use a narrow key range.
- **Pitfall — recursing on a degenerate tree.** Go grows the stack; in Rust, dropping a deep `Box` chain can overflow it.
- **Treap priorities must be independent of the keys**, and fixed per key if you want the shape to be order-independent.
- **Test a B-tree at t = 2.** The smallest degree reaches every split, borrow and merge.
- **Size a B-tree node to the thing that costs** — a page, or a few cache lines. The right t is a property of the hardware, not of the algorithm.
- **Pitfall — believing a B+ leaf chain is sequential.** It is only adjacent in memory if the tree was bulk-loaded; incremental inserts scatter leaves.
- **Count what the claim is about**: comparisons for algorithm claims, node visits for locality claims, a clock only for "who wins".
- **Pitfall — `<` on `float64` keys.** Use `cmp.Compare`; a `NaN` key has no consistent position under `<`.

## Checklist
- [ ] I can say what sorted input costs a plain BST, with the exact comparison count.
- [ ] I can write a rotation, and say which subtrees move and which stay.
- [ ] I can implement AVL insertion and name all four cases and their fixes.
- [ ] I can state the AVL bound, build the tree that meets it, and say why delete can rotate Θ(log n) times.
- [ ] I can list the five red-black rules and derive the height bound from them.
- [ ] I can draw a red-black tree as a 2-3-4 tree.
- [ ] I can explain why a treap's shape does not depend on insertion order.
- [ ] I can state a skip list's expected cost and pointer count, and what it cannot promise.
- [ ] I can split, borrow and merge in a B-tree, and say why it grows at the root.
- [ ] I can explain why a B-tree saves node visits rather than comparisons.
- [ ] I can say what a B+ tree changes and what a leaf chain does and does not guarantee.
- [ ] I know Go ships no ordered map, and where the tree-shaped code in its own source is.
- [ ] I know `BTreeMap`'s `B` and `CAPACITY`, and how it searches inside a node.

## Resources
- CLRS ch. 13 (red-black trees) and ch. 18 (B-trees); Knuth, TAOCP vol. 3, §6.2.3 (balanced trees, the AVL bound) and §6.2.4 (multiway trees).
- Adelson-Velsky & Landis, "An algorithm for the organization of information" (1962) — the AVL paper.
- Aragon & Seidel, "Randomized Search Trees" — the treap paper, linked from Go's own source: https://faculty.washington.edu/aragon/pubs/rst89.pdf
- Pugh, "Skip Lists: A Probabilistic Alternative to Balanced Trees", CACM 33(6), 1990.
- Rust's `BTreeMap` — the header comment of `node.rs` and the "Background" section of `map.rs` are worth reading: https://github.com/rust-lang/rust/tree/master/library/alloc/src/collections/btree
- Go's treap, in the runtime: https://github.com/golang/go/blob/master/src/runtime/sema.go — and the compiler's AVL tree: https://github.com/golang/go/blob/master/src/cmd/compile/internal/abt/avlint32.go
- `maps` and `slices` (`slices.Sorted`, `slices.BinarySearch`): https://pkg.go.dev/slices
- SQLite's file format (B-trees on disk): https://www.sqlite.org/fileformat2.html · PostgreSQL's B-tree indexes: https://www.postgresql.org/docs/current/btree.html
- Third-party, not stdlib: `github.com/google/btree`, a B-tree for Go.
- Examples: [examples/19-balanced-trees](examples/19-balanced-trees/) (16).
- Next: [20 — Heaps & Priority Queues](20-heaps-priority-queues.md) — the tree that lives in a slice.
