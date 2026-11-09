# Step 18 — Binary Search Trees · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The invariant, and search as a descent**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The invariant, and search as a descent
- [ ] 2. Insert is a failed search
- [ ] 3. In-order is sorted
- [ ] 4. Floor, ceiling, successor and predecessor
- [ ] 5. Delete, case by case
- [ ] 6. Sorted input makes a linked list

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Validation: the bounds trick, not the naive check
- [ ] 8. The k-th smallest, and what a size field buys
- [ ] 9. Range queries: the path plus the answer
- [ ] 10. Building a balanced tree from a sorted slice
- [ ] 11. Two-sum on a BST, with O(h) memory

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Delete without recursion: a cursor and a held pointer
- [ ] 13. Hibbard's drift: random deletes unbalance the tree
- [ ] 14. Ordered and dynamic: a BST against a sorted slice
- [ ] 15. The problems BSTs own: recover, closest, count smaller
- [ ] 16. Capstone: `Tree[K,V]`, built, proved, measured

## The three-pass build

In `practice/18-binary-search-trees/`:

- [ ] **Build it** — `Tree[K, V]` with `Put`, `Get`, `Delete` (all three cases) and `Min` / `Max`
- [ ] **Build it** — `Floor`, `Ceil`, `Pred`, `Succ` as single descents, with no parent pointers
- [ ] **Build it** — a size field, `Rank` and `Select`, repaired on every insert and delete
- [ ] **Build it** — a pruned `Range`, an in-order iterator, and a balanced `Build` from a sorted slice
- [ ] **Prove it** — a model oracle: a sorted slice, on a **narrow key range** so duplicates and re-deletes occur
- [ ] **Prove it** — after *every* operation: strictly ascending in-order, the min/max bounds invariant, sizes recounted
- [ ] **Prove it** — a validator tested on every labelled tree of up to 7 nodes, and the naive validator shown wrong
- [ ] **Prove it** — every insert order × every delete order at n = 6
- [ ] **Measure it** — comparisons and height for random, sorted and zig-zag insertion order
- [ ] **Measure it** — the tree against a sorted slice on an update-heavy workload, at several sizes
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What is the BST invariant? | every key in the **left subtree** is smaller, every key in the **right subtree** larger |
| Why is "each child compares correctly with its parent" not enough? | a grandchild can sit on the wrong side of an ancestor: root 2, right child 3, whose left child is 1 |
| Comparisons for a hit at depth d? | d + 1 |
| Comparisons for a miss? | the depth of the nil link where the search fell off |
| What does an insert do? | a failed search, then hang the new node on that nil link |
| What decides a BST's shape? | the insertion order, nothing else |
| What does an in-order walk give? | the keys in strictly ascending order |
| How do you find a ceiling without parent pointers? | descend from the root, remembering the last node where you turned left |
| The three delete cases? | no children · one child · two children (replace with the in-order successor) |
| Copy the successor's key, or move its node? | moving keeps every surviving node's identity; copying leaves held pointers stale |
| How do you validate a BST? | recurse with the open interval (lo, hi); going left tightens hi, going right tightens lo |
| What breaks a sentinel-bounds validator? | a valid tree that contains `math.MinInt` or `math.MaxInt` |
| What does a size field give you? | k-th smallest and rank in Θ(h) instead of Θ(h + k) |
| What must you repair on every update when you keep sizes? | the size of every node on the search path |
| Range query cost? | Θ(h + m): two boundary paths plus the m results |
| The split node of a range query is…? | the lowest common ancestor of lo and hi |
| Height of a BST built from a sorted slice by the middle rule? | exactly ⌊log₂ n⌋ + 1 |
| Cost of inserting 1…n in order? | n(n−1)/2 comparisons; height n |
| Expected height of a random BST? | Θ(log n) |
| Why can the Rust cursor loop for delete fail to compile? | `while let Some(node) = cur` leaves a borrow of `*cur` live at the assignment `*cur = …` (E0506); test with a short-lived shared borrow, then reborrow |
| Does Go's standard library have an ordered map? | no — `container/` is `heap`, `list`, `ring` |
| What is Rust's `BTreeMap`? | a B-tree (nodes of 5–11 keys, linear search inside), not a binary search tree |

## Numbers to find on your own machine

- [ ] The smallest tree the naive validator accepts and a bounds validator rejects — and how many labelled trees of each size fool it
- [ ] The comparisons to insert 1…n in order, against n(n−1)/2, for several n
- [ ] The mean comparisons per successful search in a random tree, against `2(1 + 1/n)H_n − 3`
- [ ] The height of a balanced tree built from a sorted slice, against ⌊log₂ n⌋ + 1, for every n you try
- [ ] Nodes visited for a k-th smallest query, with and without a size field
- [ ] How far `visited − m` stays below 2h in a range query
- [ ] The mean depth and height of a tree after millions of delete-then-insert pairs, with each deletion policy
- [ ] The size at which a tree starts beating a sorted slice for updates — and a workload where the slice still wins
- [ ] In Rust, the comparisons per lookup in a `BTreeMap`, against a binary search and a random BST

> Lesson: [18-binary-search-trees.md](../../18-binary-search-trees.md) · Index: [README.md](README.md)
