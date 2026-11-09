# 18 — Binary Search Trees

> Part of **Part 4 — Trees**. [17](17-binary-trees.md) gave you the node, the `nil` base case and the
> traversals; this lesson puts one rule on the keys and watches what it buys. A hash table answers "is
> `k` here?" and forgets order; a sorted slice answers everything but pays Θ(n) per insert. A binary
> search tree is the pointer-linked middle: **order and updates together**. The one-line thesis: **a BST
> is binary search made dynamic, but its Θ(log n) comes from the *shape* of the tree, and nothing in the
> structure protects the shape — the input does, or fails to.**

## Goals
- State the BST invariant precisely — about **subtrees**, not children — and say where equal keys go.
- Write search, insert and **delete with all three cases**, and count comparisons exactly.
- Explain why the shape of a BST is a pure function of insertion order, and name the order that ruins it.
- Use in-order = sorted as a feature, a tree sort and a test oracle.
- Find min, max, floor, ceiling, predecessor and successor with one descent and no parent pointers.
- Validate a BST with the **min/max bounds** and produce the three-node tree the naive check accepts.
- Answer **k-th smallest**, rank and range queries, and say what a per-node size field costs and buys.
- Build a **balanced** BST from a sorted slice, and from a preorder, in Θ(n).
- Write delete in Rust with `Option<Box<Node>>` and say which loop shapes the borrow checker refuses.
- Say what Go's standard library offers for ordered data (nothing), and what Rust's `BTreeMap` actually is.

## Concepts

- **What a BST is for.** It is a map or set that also knows **order**: first, last, next, range, rank. A
  hash map ([10](10-hash-tables.md)) has none of those; a sorted slice has all of them but every insert or delete
  moves Θ(n) elements. A BST makes every operation Θ(h), where `h` is its height.
  [Lesson 04, example 14](examples/04-go-memory-model/3-hard.md) and [lesson 06, example 12](examples/06-arrays-slices/3-hard.md)
  both ended on the same point: a sorted slice cannot be *updated* cheaply, so data that is ordered
  *and* changing wants a tree. This is that tree; [example 14](examples/18-binary-search-trees/3-hard.md) runs the workload.

- **The invariant is about subtrees, not children.** For every node, **every** key in the left subtree
  is smaller and **every** key in the right subtree is larger. If duplicates are allowed, pick one side
  for equal keys (or store a count in the node) and write the choice down — it is part of the invariant.
  The weaker rule "each child compares correctly with its parent" is *not enough*: a root `2` with right
  child `3` whose left child is `1` passes every parent/child check, and is not a BST, because `1` sits
  in `2`'s right subtree ([example 1](examples/18-binary-search-trees/1-easy.md), [7](examples/18-binary-search-trees/2-medium.md)).

- **Search is a descent: one comparison discards a whole subtree.** A hit costs `depth + 1`
  comparisons (the root has depth 0); a miss costs the depth of the `nil` link where you fell off. Let
  `I` be the sum of all node depths; the `n + 1` `nil` links have depths summing to `E`; for every binary
  tree `E = I + 2n`. So both average costs follow from one number, `I`.

- **Insert is a failed search.** Walk until you fall off, then hang the new node on that `nil` link. No
  existing node moves, so the **shape is a function of insertion order and nothing else**. Sorted order
  is one of exactly `2^(n-1)` orders that produce a path, one per path shape ([example 2](examples/18-binary-search-trees/1-easy.md)).

- **In-order is sorted.** `left, node, right` visits keys ascending — the invariant restated as an
  algorithm. Traversals are [17](17-binary-trees.md)'s business; this lesson uses one corollary of
  them: a free `Keys()`, a **tree sort** (insert `n` keys, walk once), and the cheapest correctness
  test there is — after every operation the in-order sequence must be strictly ascending. Go 1.23's
  range-over-func lets the caller `break` out of the walk early ([17, example 15](examples/17-binary-trees/3-hard.md) builds that iterator).

- **Min, max, floor, ceiling, predecessor, successor are one descent.** Walk from the root, and
  whenever you turn left remember that node (it is a candidate for ceiling/successor); whenever you turn
  right remember that node (floor/predecessor). Θ(h), no parent pointers, no recursion, and it works
  whether or not the query key is present. The textbook "successor of a *node*" — leftmost of the right
  subtree, else the lowest ancestor whose left subtree holds you — is the same answer found from below.
  Calling successor from the root `n` times to walk the tree costs Θ(h) each; an iterator with a stack
  does it in amortized Θ(1) per step ([example 11](examples/18-binary-search-trees/2-medium.md)).

- **Delete has three cases.** Find the node, then: **no children** — clear the link; **one child** —
  point the link at that child; **two children** — the node cannot simply vanish, so put in its place
  the **in-order successor** (leftmost node of its right subtree; the predecessor works equally well).
  That node has no left child, so detaching it is case 1 or 2. The in-order sequence loses exactly one key.

- **Copy the successor's key, or move the successor's node?** Copying is the textbook shortcut, but any
  pointer someone holds to the successor now points at a detached node while its key lives on in another
  one. Relinking the node keeps every surviving node's identity ([example 12](examples/18-binary-search-trees/3-hard.md)).
  In Go the loop is `link := &t.root` and a pointer-to-pointer, so root, middle and leaf are one case
  ([lesson 07, example 5](examples/07-linked-lists/1-easy.md) did the same for lists).

- **Deletion is not innocent.** Always replacing with the successor is lopsided: it only ever shortens
  right subtrees. The literature (Knuth; Culberson, 1985) reports that a long run of random
  delete-then-insert pairs on a tree built that way drifts away from the random-tree average; choosing
  predecessor or successor at random does not. [Example 13](examples/18-binary-search-trees/3-hard.md) measures it at a size you can run.

- **Validate with bounds, not neighbours.** Recurse with the open interval `(lo, hi)` every key in the
  subtree must lie in: going left tightens `hi` to the node's key, going right tightens `lo`. Θ(n) time,
  Θ(h) stack. Two traps: sentinel bounds (`math.MinInt`, `math.MaxInt`) wrongly reject a valid tree that
  contains that value — use a pointer or `Option` for "unbounded"; and `<=` vs `<` must match your
  duplicate policy. An in-order walk that remembers the previous key is an equivalent test.

- **k-th smallest and rank.** Without help, the k-th smallest is an in-order walk that stops after `k`
  keys: Θ(h + k). Store a **subtree size** in every node and it becomes a descent — compare `k` with
  `size(left)` and go left, stop, or go right with `k` reduced — Θ(h); `Rank(x)` (keys below `x`) is the
  same walk in reverse. The price is one more invariant, `size = 1 + size(left) + size(right)`, which
  every insert and delete must repair along its path. [Lesson 13](13-efficient-sorts.md)'s quickselect
  does the one-off version on an unsorted slice; the tree wins when queries and updates interleave.

- **Range queries cost the path plus the answer.** To report keys in `[lo, hi]`, recurse left only if
  `lo < key`, emit if in range, recurse right only if `key < hi`. The visited nodes are the `m` reported
  ones plus at most two root-to-boundary paths: Θ(h + m), against Θ(n) for a full scan. The two paths
  fork at the **split node** — the first node on the way down with `lo <= key <= hi` — which is the
  lowest common ancestor of `lo` and `hi`; in a BST that LCA needs no parent pointers. With sizes,
  *counting* the keys in a range is Θ(h) no matter how many there are (`Rank(hi+1) - Rank(lo)`).

- **Build a balanced tree when you already have the keys.** From a sorted slice, make the middle
  element the root and recurse on each half: Θ(n), **no comparisons at all**, and the height is exactly
  `⌊log₂ n⌋ + 1`, with every level but the last full. A BST is also determined by its **preorder alone**
  (rebuild with the same bounds idea in one pass). And "flatten with an in-order walk, rebuild from
  the slice" is the crudest rebalance there is — Θ(n), but it works on any tree ([19](19-balanced-trees.md) does it incrementally with rotations).

- **The failure mode: sorted input makes a linked list.** Insert `1, 2, 3, …, n` and every node has
  only a right child: the tree is a list of height `n`, the `i`-th insert costs `i` comparisons, so
  building it costs `n(n-1)/2` and finding every key once costs `n(n+1)/2`. The inputs that do this are
  not exotic — timestamps, auto-increment IDs, alphabetical imports, a log read in order. For *randomly
  ordered* keys the picture is the opposite: the expected height is O(log n) (CLRS, theorem 12.4) and a
  successful search takes `2(1 + 1/n)H_n − 3` comparisons on average (Knuth, §6.2.2) — about
  `2 ln n ≈ 1.39 log₂ n`. [Example 6](examples/18-binary-search-trees/1-easy.md) counts both.

- **Height is also stack depth.** A recursive operation on a degenerate tree recurses `n` deep. Go grows
  a goroutine's stack on demand up to a limit of **1,000,000,000 bytes on 64-bit** (`runtime/proc.go`) — 512 MiB in practice, since the stack doubles; a Rust
  thread's stack is fixed when the thread is created, and the main thread's is whatever the OS gave it
  (`ulimit -s`). Depth is [lesson 05](05-recursion.md)'s and [17, example 13](examples/17-binary-trees/3-hard.md)'s subject; here the rule is: iterate, or keep `n` small.

- **Pair problems with two iterators.** A stack iterator keeps the left spine; `Next` pops, then pushes
  the left spine of the popped node's right child. Every node is pushed once and popped once, so `n`
  calls cost Θ(n) total (amortized Θ(1)) with Θ(h) memory, and stopping early costs only what you took. Run
  one forward and one backward iterator toward each other and **two-sum on a tree** is Θ(n) time, Θ(h)
  space — no hash set, no copy of the keys ([example 11](examples/18-binary-search-trees/2-medium.md)).

- **Go ships no ordered map.** `container/` holds exactly three packages — `heap`, `list`, `ring` — and
  nothing in the public standard library is a tree. (The compiler has its own AVL tree,
  `cmd/compile/internal/abt`, which `internal` rules keep you from importing.) Your options: a sorted
  slice with `slices.BinarySearch` (read-mostly data), `slices.Sorted(maps.Keys(m))` for an occasional
  ordered dump of a map (Θ(n log n) per call), or write the 80 lines yourself — generic over
  `cmp.Ordered`, or taking a `cmp func(a, b K) int` for struct keys. Use `cmp.Compare` rather than `<`:
  it orders a float `NaN` before every number and equal to itself, so the tree stays coherent, whereas
  `NaN < x` and `NaN > x` are both false, so a descent written `<`, `>`, else-found calls it "equal".

- **A pointer per node is a cache miss per step.** Every node is its own allocation and every step of a
  descent follows a pointer to an address the hardware could not predict. [Lesson 04, example 14](examples/04-go-memory-model/3-hard.md)
  measured that on a *balanced* tree. It is why production ordered maps are B-trees, which pack many
  keys into one node — the next section's `BTreeMap`, and [lesson 19](19-balanced-trees.md).

### Go and Rust side by side

- **`BTreeMap` / `BTreeSet` are B-trees, not binary search trees.** Each node holds `B − 1` to `2B − 1`
  keys in a contiguous array with `B = 6` (`alloc/src/collections/btree/node.rs`), all leaves at the
  same depth ("trees must have uniform depth"), and the node is searched *linearly*. They cannot go
  degenerate, so sorted inserts are fine. The docs themselves say a BST needs fewer comparisons and a
  B-tree wins on allocations and cache misses; lesson 19 explains the structure.
- **Ordered queries are `range`.** Floor is `range(..=k).next_back()`, ceiling `range(k..).next()`,
  predecessor `range(..k).next_back()`, successor `range((Excluded(k), Unbounded)).next()`; also
  `first_key_value`, `last_key_value`, `pop_first`, `split_off`. The cursor API (`lower_bound`) is
  still unstable (`btree_cursors`).
- **`range` panics on an empty-looking range** — start > end, or equal and both excluded — but only when
  the map is non-empty. There is **no rank/select**: `iter().nth(k)` is the default `Iterator::nth`,
  `k + 1` calls to `next`. Iterators are documented as amortized constant time per item.
- **Keys must be `Ord`**, so `BTreeMap<f64, _>` does not compile (`E0277`); Go accepts floats and lets
  you build the `NaN` tree above. Wrap the key and use `f64::total_cmp`.
- **`Option<Box<Node>>` is the node**, and `take()` is the tool: `n.left = remove(n.left.take(), k)`
  hands the subtree to the recursive call and writes the result back. Insert with a `&mut
  Option<Box<Node>>` cursor compiles; the **same loop shape for delete does not** (`E0506`, and the
  obvious repairs give `E0499`) — the working form tests with a short-lived shared borrow, then
  reborrows ([example 12](examples/18-binary-search-trees/3-hard.md) shows each).
- **A `&Node` held across a delete is a compile error** (`E0502`) where Go lets the stale pointer live.
  There are no parent pointers without `Rc`/`Weak`, an arena of indices, or `unsafe` (`BTreeMap`
  itself keeps `NonNull` parent pointers), so the Rust twins navigate from the root.

## Complexity Table

| Operation | Time | Space | Note |
|---|---|---|---|
| Search / insert / min / max / floor / ceiling / pred / succ | **Θ(h)** | Θ(1) iterative | h = Θ(log n) balanced, Θ(n) degenerate |
| Delete (any of three cases) | Θ(h) | Θ(1) iterative | one descent, plus one more for the successor |
| Build by n inserts | Θ(n log n) expected, **Θ(n²)** sorted | Θ(n) | sorted: n(n−1)/2 comparisons |
| Build from a sorted slice | **Θ(n)**, 0 comparisons | Θ(n) | height exactly ⌊log₂ n⌋ + 1 |
| Build from a preorder | Θ(n) | Θ(h) | one pass with bounds |
| In-order walk | Θ(n) | Θ(h) stack | strictly ascending |
| Validate (bounds) | Θ(n) | Θ(h) | the naive parent/child check is wrong |
| k-th smallest, plain | Θ(h + k) | Θ(h) | in-order, stop after k |
| k-th smallest / rank, with sizes | **Θ(h)** | +1 word per node | sizes repaired on every update |
| Range query, m results | **Θ(h + m)** | Θ(h) | two boundary paths + the answer |
| Count in a range, with sizes | Θ(h) | Θ(1) | independent of m |
| Iterator: n calls to Next | Θ(n) total, amortized Θ(1) | Θ(h) | each node pushed and popped once |
| Two-sum on a BST | Θ(n) | Θ(h) | forward and backward iterator |
| Expected height, random insertion order | Θ(log n) | | ≈ 1.39 log₂ n comparisons per hit |
| Height, sorted insertion order | **n** | | a linked list |
| Flatten and rebuild balanced | Θ(n) | Θ(n) | the crude rebalance |

## Exercises
1. Write a node, `Contains` and an `Insert` that reports whether it added. Count comparisons for every key and every gap of a fixed 12-key tree, and check `E = I + 2n`.
2. Insert the same seven keys in three orders and print the three trees. Then enumerate all 5,040 orders and histogram the heights.
3. Write an in-order iterator with early exit. Compare its output to a sorted copy on 1,000 random trees, then use it as a tree sort.
4. Write floor, ceiling, predecessor and successor with one descent each. Verify every integer query in a range against a linear scan, present and absent keys alike.
5. Write delete. Delete a leaf, a one-child node, a two-child node whose successor is its right child, one whose successor is deeper, and the root; print the in-order sequence after each.
6. Insert sorted, reverse-sorted, zig-zag and shuffled keys. Count height and comparisons, and fit the sorted case to `n(n−1)/2`.
7. Write the naive validator and the bounds validator. Enumerate every labelled tree on up to 7 nodes and count the trees the naive one wrongly accepts. Then break the bounds version with `math.MinInt`.
8. Add subtree sizes. Write `Select` and `Rank`, repair sizes on insert and delete, and write an invariant check that catches a forgotten decrement.
9. Write a pruned range query and an unpruned one. Verify `visited − m ≤ 2h` on 10,000 queries, and count in a range with `Rank`.
10. Build a balanced BST from a sorted slice for every n ≤ 2,000 and check height `⌊log₂ n⌋ + 1`. Rebuild a degenerate tree by flatten-and-rebuild. Rebuild a tree from its preorder.
11. Write a stack-based iterator, forward and backward. Show total pushes equal n, then solve two-sum on a BST and check it against brute force.
12. Write delete with a pointer-to-pointer in Go and a `&mut Option<Box<Node>>` cursor in Rust. Compare it with the recursive one on 100,000 random operations. Then hold a pointer to a successor across a two-child delete — copying and relinking.
13. Run random delete-then-insert churn on 1,000 keys with always-successor deletion and with a coin flip. Record mean depth and height over many steps.
14. Race a sorted slice against a BST on an update-heavy workload, counting element moves and comparisons first and timing second; find the size at which the tree wins.
15. Recover a BST with two swapped keys, find the closest key to a target, and count smaller elements after each position — against brute force, on sorted and random input.
16. Stretch — build `Tree[K, V]` with sizes, floor/ceiling, rank/select and a range iterator; prove it against a sorted-slice model with an invariant check after every operation, and exhaustively for every insert order × delete order at n = 6.

## Best Practices & Pitfalls
- **Don't ship a hand-written BST.** The standard library has none, so write it to learn; for real ordered data in Go prefer a sorted slice, and in Rust use `BTreeMap`.
- **Pitfall — feeding it sorted input.** Shuffle first, build from a sorted slice, or use a balanced tree. The input you must worry about is the *natural* one.
- **Write the duplicate policy down.** Reject, overwrite, count in the node, or send equal keys one way — and make the validator agree.
- **Pitfall — validating parent against children.** Pass `(lo, hi)` down. Test the three-node counterexample.
- **Pitfall — sentinel bounds.** Use a pointer or `Option` for "unbounded", not `math.MinInt`.
- **Use `cmp.Compare`, not `<`,** when the key can be a float.
- **Pitfall — forgetting the one-child cases.** A delete that handles "two children" and "leaf" loses a subtree on the rest. Compare the in-order sequence before and after.
- **Pitfall — copying the successor when somebody holds the node.** Relink instead, or hold keys, not pointers.
- **Pitfall — always taking the successor in a long-lived tree.** Alternate or randomise.
- **Pitfall — a size field you forget to repair.** Recompute `size` from the children on the way back up, and assert the invariant in tests.
- **Prune range queries.** Without the two `if`s the query visits every node.
- **Prefer iteration to recursion on trees you do not control.** Recursion depth is the height.
- **Test with an oracle.** A sorted slice is the model; check strictly-ascending in-order, the bounds invariant and sizes after every operation, on a **narrow key range** so duplicates and re-deletes happen.
- **In Rust, don't hold a `&Node` across a mutation.** Hold the key and look it up again.
- **Pitfall — treating `BTreeMap` as a BST.** It is a B-tree; it has no rank, and `range` panics on a reversed range.

## Checklist
- [ ] I can state the BST invariant and show why parent-versus-child is not enough.
- [ ] I can count the comparisons a search or insert makes, and say what `I` and `E` are.
- [ ] I can explain why a BST's shape is a function of insertion order.
- [ ] I can write floor, ceiling, predecessor and successor without parent pointers.
- [ ] I can write delete with all three cases from memory, and say copy-versus-relink.
- [ ] I can validate with bounds and produce the counterexample the naive check accepts.
- [ ] I can add a size field, write `Select` and `Rank`, and say what must be repaired on update.
- [ ] I can answer a range query in Θ(h + m) and count a range in Θ(h).
- [ ] I can build a balanced BST from a sorted slice and give its exact height.
- [ ] I can say why sorted input gives `n(n−1)/2` comparisons and what inputs cause it in practice.
- [ ] I can write delete in Rust and name the loop shape the borrow checker rejects.
- [ ] I know Go has no ordered map, and that `BTreeMap` is a B-tree.

## Resources
- CLRS, ch. 12 — binary search trees; theorem 12.4 is the O(log n) expected height of a random tree. Knuth, TAOCP vol. 3, §6.2.2 — the exact average comparison counts, and deletion.
- Sedgewick & Wayne, *Algorithms* (4th ed.), §3.2 — a BST symbol table with `rank`, `select`, `floor` and `ceiling`.
- J. Culberson, "The effect of updates in binary search trees" (STOC 1985) — what long update sequences do to a BST.
- Rust's `BTreeMap` and its design note: https://doc.rust-lang.org/std/collections/struct.BTreeMap.html and `library/alloc/src/collections/btree/node.rs`.
- Go's `container` packages (heap, list, ring) and the compiler's internal AVL tree: https://github.com/golang/go/blob/master/src/cmd/compile/internal/abt/avlint32.go
- Go 1.23 range-over-func and `iter.Seq`: https://pkg.go.dev/iter
- Examples: [examples/18-binary-search-trees](examples/18-binary-search-trees/) (16).
- Next: [19 — Balanced Trees & B-Trees](19-balanced-trees.md) — rotations, AVL and red-black trees, and the B-tree behind `BTreeMap`.
