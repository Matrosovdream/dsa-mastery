# 17 — Binary Trees & Traversals

> Part of **Part 4 — Trees**, and where recursion finally pays for itself. A linked list ([07](07-linked-lists.md))
> has one successor per node; a binary tree has two, and that one extra pointer is the difference
> between a loop and a recursion. The one-line thesis: **a tree algorithm is a decision about *when* to
> look at a node relative to its two subtrees — before, between, after, or one layer at a time — and
> the number that decides what it costs in memory is not `n` but the height `h`**, which can be as
> small as log₂ n or as large as `n`.

## Goals
- Define a binary tree recursively, write the node in Go and in Rust, and say why the empty tree is the base case.
- State how many `nil` links an n-node tree has, and so how many calls any full traversal makes.
- Write **pre-, in- and post-order** recursively and say which line moves when you change the order.
- Convert all three to an **explicit stack**, and say what the stack holds in each — and why post-order is the awkward one.
- Write **level-order** with a queue, and say what the queue's size measures.
- Compute **height, depth and diameter**, and recognise "return one thing, record another".
- Solve the **path-sum** family: root-to-leaf, any downward path, and the maximum bent path with negative values.
- **Invert** a tree, and compare two trees in lockstep (same, mirror, symmetric, subtree).
- Find a **lowest common ancestor**, and know the input on which the one-pass version is wrong.
- **Serialize and deserialize**, and say why the marker-free version is ambiguous.
- **Rebuild** a tree from two traversals, and say which pairs determine it and which do not.
- Traverse in **O(1) extra space** with Morris threading, and name what it costs you.
- Own a tree in Rust three ways — `Box`, `Rc<RefCell<_>>`, an arena — and know how a deep tree breaks each.

## Concepts

- **A binary tree is a recursive definition.** It is either empty, or a node holding a value and two
  binary trees. In Go that is `type Node struct { Val int; Left, Right *Node }` with `nil` as the empty
  tree; in Rust it is `Option<Box<Node>>`. Every function in this lesson is a case analysis on that
  definition — the empty case, then the node case that calls itself on both children — and a function
  that skips the empty case is wrong on every leaf.

- **An n-node tree has exactly n + 1 `nil` links.** There are 2n child slots and only n − 1 are used
  (every node but the root is someone's child), so n + 1 are empty. A recursive traversal therefore
  makes **2n + 1 calls**: n that do work and n + 1 that hit the base case. This is the counting fact
  behind the traversal costs, behind the serialized length in example 10, and behind why a tree is
  mostly empty pointers.

- **The shape is the cost.** Time for a traversal is Θ(n) whatever the shape; *space* is the height.
  Counting height in nodes (empty tree = 0, a leaf = 1; some books count edges and start at −1 —
  pick one), an n-node tree has ⌈log₂(n+1)⌉ ≤ h ≤ n. A perfect tree sits at the low end, a chain at the
  high end, and nothing in the definition says which one you were given. The number of distinct
  binary-tree *shapes* with n nodes is the Catalan number `C(2n, n) / (n + 1)`.

- **Three depth-first orders are one recursion with the visit in three places.** Visit before the two
  calls and you have **pre-order**; between them, **in-order**; after both, **post-order**. Each node is
  touched three times — on the way down, between the children, on the way up — and the order is only
  which touch you keep. Use pre-order to copy or serialize (parents before children), post-order when a
  node needs its children's answers (size, height, evaluate, free), and in-order when the tree is
  ordered, which is lesson [18](18-binary-search-trees.md).

- **Iterative means the call stack becomes a slice.** Pre-order: pop, visit, push right then left (a
  stack reverses). In-order: walk left pushing every node, pop and visit, then move to the right child
  ([lesson 05](05-recursion.md) example 7 wrote both). **Post-order is the hard one**, because a popped
  node is not finished until *both* subtrees are, and you must know which child you are returning from.
  There are three fixes: two stacks (the second one receives every node, so it holds all n), one stack
  plus a `last` pointer, or one stack of `(node, state)` frames, which is a literal simulation of the
  call stack and handles all three orders with one loop. The stack does *not* always hold the height:
  what it holds depends on the method and on which way the tree leans (example 3).

- **Level-order is BFS with a queue.** [Lesson 09](09-queues-deques.md) built the queue and showed the
  `n := len(q)` snapshot that separates layers; on a tree there is no `seen` set, because a tree has no
  cycles. The queue holds at most one layer, so its size is the tree's **width** (at most (n + 1)/2,
  reached by a perfect tree), while a depth-first stack holds about the **height**. The two visit the
  same nodes and differ only in what they must remember: a chain is the best case for BFS and the worst
  for DFS, a perfect tree the reverse (example 4).

- **Return one thing, record another.** `height(n) = 1 + max(height(left), height(right))` is a
  post-order recursion. The **diameter** through a node is `height(left) + height(right)` (heights in nodes, so the sum
  counts edges), so a single pass can return the height upward while updating a running best on the side. Calling `height`
  separately at every node instead costs the sum of all subtree sizes — Θ(n²) on a chain. Lesson
  [32](32-dp-advanced.md) returns to this shape as DP on trees.

- **Mutation returns the new subtree.** The idiom for changing a tree is `n.Left = fix(n.Left)`: the
  recursive call hands back what the subtree should now be, and the parent stores it. **Invert** swaps
  the two children at every node, in place, with no allocation. **Prune** and every BST insert and
  delete in [18](18-binary-search-trees.md) are the same move. Rust needs `&mut` or ownership to do it
  (below).

- **Path sums are three different problems.** *Root-to-leaf*: carry the remaining sum down, test at a
  leaf. *Any downward path with sum k*: keep a map of prefix sums along the current root path, the same
  `prefix[j] − k` identity as lesson [10](10-hash-tables.md), and **undo the entry on the way back up**
  — the choose/explore/un-choose of [lesson 05](05-recursion.md). *Maximum path sum* with negative
  values: return the best **downward** path from each node (never below 0), record the best **bent**
  path through it, and do not start the running best at 0 — an all-negative tree has no empty path.

- **Two trees at once.** *Same*, *mirror* and *symmetric* walk two trees in lockstep with one
  recursion on a pair. *Is T a subtree of S* is Θ(|S|·|T|) if you compare at every node; hashing each
  subtree (a Merkle hash) or serializing it makes the comparison cheaper at the cost of collisions or
  of an extra string search ([lesson 35](35-string-algorithms.md)).

- **Lowest common ancestor.** One post-order pass: return the node if it is `p` or `q`; if both
  children return something, this node is the answer; otherwise pass the one non-empty result up. It
  **assumes both nodes are in the tree**. If `q` is absent it returns `p` as though it were the answer —
  silently. Alternatives: parent pointers plus depths, or root paths compared from the top. Answering
  many queries fast is binary lifting ([28](28-divide-conquer.md)) or an Euler tour with a sparse table
  ([36](36-range-queries.md)).

- **Serialization needs markers.** Pre-order *with a marker for every `nil`* determines the tree,
  because the markers say where each subtree ends; it is exactly 2n + 1 tokens, the 2n + 1 calls above.
  Without markers, pre-order alone is not enough (two different shapes share a sequence). Level-order
  with `null`s is the format most judges use; it can drop its trailing `null`s. Deserializing is itself
  a recursive descent over a token stream ([lesson 05](05-recursion.md) example 16), so on input you did
  not write it must return an error for truncated or trailing tokens and bound its depth.

- **Which pairs of traversals determine a tree?** *Pre-order + in-order* and *post-order + in-order*
  do, for distinct values: the first element of pre-order (the last of post-order) is the root, and its
  position in the in-order sequence says how many nodes belong to the left subtree. Finding that
  position by scanning costs up to O(h) per node and Θ(n²) on a chain; a value → index map makes the
  build Θ(n) (lesson 10 again). **Pre-order + post-order do not**: a node with one child cannot say
  whether it is the left or the right. (A *full* tree — every node has 0 or 2 children — is the
  exception: there they do identify it.) None of the three alone suffices.

- **Morris traversal: in-order in O(1) extra space.** For each node with a left child, find its
  in-order predecessor (the rightmost node of the left subtree) and temporarily point that node's
  `Right` at the current node — a *thread* that serves as the way back. On the second arrival the
  thread is found, removed, and the node is visited. Time stays Θ(n); no stack, no recursion. The price
  is that **the tree is modified during the walk**: stop early, or panic, and it is left threaded (a
  cycle); concurrent readers see it too. In Rust it needs two links to one node, which `Box` cannot
  express — an index arena can (example 12).

- **Depth is the hazard when you did not choose the tree.** Recursion depth equals the height, so a
  chain of n nodes needs n frames. Go grows a goroutine stack on demand up to a limit of
  1,000,000,000 bytes on 64-bit (`runtime/proc.go`) — and since each growth doubles the stack, the largest
  it can actually reach is 512 MiB (example 13 measures it) — adjustable with `debug.SetMaxStack`; `encoding/json` refuses nesting past 10,000
  (`maxNestingDepth` in `scanner.go`). Rust's spawned threads get **2 MiB** by default on Tier-1 platforms
  (`std::thread` documentation, which also says it is subject to change), and the main thread's stack is
  *not determined by Rust* (it comes with the process; see `ulimit -s`).
  Rust adds a second hazard: **dropping a `Box` tree is itself a recursion**, one nested drop per
  level, so a tree you can build iteratively can still overflow when it goes out of scope ([lesson
  07](07-linked-lists.md) wrote the iterative `Drop`). Lesson [05](05-recursion.md) example 15 first
  measured the traversal side; example 13 here counts the drop side and the cost per level of each function.

- **A traversal can be an iterator, and an iterator can stop.** A recursive walk is naturally *push*:
  it calls your function for each node. Go's `iter.Seq[T]` is that shape — `go/ast.Preorder` is a
  recursive `Inspect` wrapped in one — and `break` stops the walk by making `yield` return `false`;
  `iter.Pull` turns it into `next()`/`stop()`. Rust's `Iterator` is *pull*, and an explicit-stack
  in-order walk is exactly the iterator's state: `next` is the loop body. Stopping after k nodes costs
  the descent to the first plus k, not n (example 15).

- **Neither standard library ships a binary tree.** Go's `container` holds `heap`, `list` and `ring`;
  trees appear inside packages (`go/ast` offers `Walk`, `Inspect` and `Preorder` over its own nodes).
  Rust's `BTreeMap` is a B-tree, not a binary one ([19](19-balanced-trees.md)). You write the node.

### Go and Rust side by side

- **The empty tree.** Go: `nil`, and a method may have a nil receiver (`func (n *Node) Size() int { if n == nil { return 0 } … }`), which is idiomatic. Rust: `Option<Box<Node>>`, and you `match` — the compiler will not let you forget the empty case. `Option<Box<T>>` is guaranteed the size of a `Box<T>` (the null-pointer optimisation, listed in `core/src/option.rs`), so `None` costs nothing.
- **Recursion takes a reference to the link:** `fn f(n: &Option<Box<Node>>)`, or `Option<&Node>` via `as_deref()`. Returning a reference into the tree (LCA) ties its lifetime to the borrow of the tree.
- **Mutation needs `&mut`.** Go changes a link with `*Node`, or `**Node` to change where it points; Rust uses `&mut Option<Box<Node>>` as a cursor, `take()` to move a subtree out, and `std::mem::swap` to swap two links. Assigning through the cursor **drops** the old subtree.
- **One owner means no sharing.** `Box` gives a tree, not a graph: no parent pointers, no shared subtrees. The two escapes are `Rc<RefCell<Node>>` (the usual LeetCode style, with `Weak` for parents; `borrow_mut` **panics** if a borrow is live, `try_borrow_mut` returns an error) and an arena — a `Vec<Node>` with integer indices, which needs neither and is what [lesson 07](07-linked-lists.md)'s arena did (example 14 compares all three).
- **No destructor in Go, a recursive one in Rust.** The GC needs no `Drop`; Rust's default drop glue recurses once per level, so any tree that can be deep wants an iterative `Drop`.
- **Stacks differ.** Go's grow on demand (limit 1,000,000,000 bytes, so 512 MiB in practice); a Rust thread's is fixed when it is created (2 MiB by default; `thread::Builder::stack_size` changes it).
- **Neither promises tail calls.** Whether a recursive call becomes a loop is the optimiser's decision; example 13 checks which of this lesson's functions get it in a `-O` build.
- **Visitors.** Go: a function argument or `iter.Seq`. Rust: `impl FnMut(&Node)`, or a struct implementing `Iterator`.

## Complexity Table

| Operation | Time | Extra space | Note |
|---|---|---|---|
| Any full traversal (pre / in / post / level) | **Θ(n)** | Θ(h) DFS, Θ(w) BFS | 2n + 1 calls; w = widest level ≤ (n + 1)/2 |
| Iterative DFS, explicit stack | Θ(n) | ≤ n (method-dependent) | the two-stack post-order holds all n |
| Morris traversal | Θ(n) | **Θ(1)** | mutates the tree while it walks |
| Height of an n-node tree | — | — | ⌈log₂(n+1)⌉ ≤ h ≤ n |
| Height / size / leaves | Θ(n) | Θ(h) | one post-order pass |
| Diameter, one pass | Θ(n) | Θ(h) | return height, record the best |
| Diameter, `height` at every node | O(n·h) | Θ(h) | the sum of subtree sizes: Θ(n²) on a chain |
| Root-to-leaf path sum | Θ(n) | Θ(h) | |
| Downward paths summing to k, prefix map | Θ(n) expected | Θ(h) map | vs O(n·h) starting a walk at every node |
| Maximum path sum | Θ(n) | Θ(h) | return the downward gain, record the bend |
| Invert in place | Θ(n) | Θ(h) | zero allocations; copy-and-invert allocates n |
| Same / mirror / symmetric | Θ(min(n, m)) | Θ(h) | lockstep |
| Is T a subtree of S | Θ(n·m) naive | Θ(h) | hashed subtrees: Θ(n + m) expected |
| LCA, one pass | Θ(n) per query | Θ(h) | wrong when a node is absent |
| LCA, parent pointers + depth | Θ(h) per query | Θ(n) setup | |
| Serialize / deserialize | Θ(n) | Θ(h) | pre-order with markers: 2n + 1 tokens |
| Build from pre + in, scan for the root | O(n·h) | Θ(h) | Θ(n²) on a chain |
| Build from pre + in, value → index map | **Θ(n)** | Θ(n) | |
| Shapes of n nodes | — | — | `C(2n, n)/(n + 1)` (Catalan) |

## Exercises
1. Build a tree by hand and by a seeded generator. Count calls, work calls and `nil` hits for five shapes and confirm n, n + 1 and 2n + 1. Then call `Size` on a nil `*Node` in Go and write the Rust version that cannot forget.
2. Write pre-, in- and post-order recursively from one skeleton. Print the enter / visit / leave events for a 9-node tree and say which touch each order keeps.
3. Write all three iteratively, including post-order **three ways** (two stacks, a `last` pointer, a frame stack). Record each method's maximum stack size on a random tree, a perfect tree, and left-, right- and zigzag chains.
4. Write level-order with the snapshot idiom, then with two slices instead of a queue. Compare the queue's maximum size with a DFS stack's on the same five shapes.
5. Write `Height`, `Depth` and `Diameter`. Write the O(n²) diameter on purpose, count its `height` calls on a chain, and check the one-pass version against all-pairs distances on thousands of small trees.
6. Write `Invert` in place and as a copy, and `Prune` (remove subtrees whose values are all ≤ 0). Show in Rust what `*cursor = …` drops.
7. Solve has-path-sum, count-paths-with-sum-k (prefix map, with the undo) and maximum-path-sum. Check each against a brute force, and use an all-negative tree for the last.
8. Write `Same`, `Mirror`, `Symmetric` and `IsSubtree`. Build an input on which the naive subtree check does Θ(n·m) comparisons, then fix it with a subtree hash.
9. Write the one-pass LCA and a parent-pointer LCA. Feed the first a query with one absent node and count how many of 20,000 random queries it gets wrong.
10. Serialize with markers and without; show two shapes with the same marker-free pre-order. Round-trip thousands of random trees (including the empty one), then feed the deserializer truncated and padded input.
11. Rebuild from pre + in and post + in, with and without a value → index map. Enumerate every shape of n ≤ 8 nodes and check which pairs of traversals identify all of them.
12. Write Morris in-order. Count threads, removals and moves. Stop it early and show the damage, then finish the walk and show the tree restored.
13. Measure the stack bytes per level of five recursive functions on left- and right-leaning chains. Count the nesting depth of a default `Drop` against an iterative one. Convert diameter to an explicit stack and run it on a million-node chain.
14. Write one tree as `Box`, as `Rc<RefCell<_>>` and as an arena (and the Go pointer, index and implicit-array equivalents). Count allocations and bytes; write `invert` on each.
15. Write in-order as an `iter.Seq` and as a pull iterator, and as a Rust `Iterator` over an explicit stack. Count nodes visited when you stop after k.
16. Stretch — build a `Tree` toolkit (traversals, height, diameter, LCA, serialize, rebuild) and prove it against an oracle that stores every node as its `L`/`R` path string. Measure it.

## Best Practices & Pitfalls
- **Write the empty case first, in every function.** The rest of the function then only ever sees a real node.
- **Pitfall — mixing height conventions.** Nodes or edges, empty tree 0 or −1: choose once and write it in the comment.
- **Pitfall — `height` inside a loop over the nodes.** That is Θ(n·h). Return the height upward from one post-order pass.
- **Pitfall — assuming the tree is shallow.** Recursion depth is the height; for a tree from outside, bound the depth, use an explicit stack, or run on a thread you sized.
- **Pitfall — iterative in-order that forgets to step right,** and iterative post-order that visits a node on the way down. A frame stack with a state makes both impossible.
- **Pitfall — `len(q)` read inside the level loop.** Snapshot it before ([lesson 09](09-queues-deques.md)).
- **Pitfall — pre-order without markers as a serialization.** Different trees share it. Emit a marker for every `nil`.
- **Pitfall — believing pre + post identifies a tree.** It identifies a *full* binary tree and nothing else.
- **Pitfall — LCA on a node that may be absent.** Check membership first, or return a found flag with the answer.
- **Pitfall — pruning a path sum when `sum > k`.** Negative values make that wrong; only a leaf (or the end of a path) decides.
- **Pitfall — starting a maximum at 0.** An all-negative tree has a best path that is negative.
- **Pitfall — forgetting the undo.** The prefix-sum map must lose an entry when the recursion returns, or siblings count each other's paths.
- **Rust — reach for `Rc<RefCell<_>>` only when you need sharing or parent links.** Prefer `Box` for a pure tree and an index arena when nodes point at each other.
- **Rust — give any tree that may be deep an iterative `Drop`,** or build the deep case in a thread whose stack you sized. Assigning to a link also drops the old subtree.
- **Pitfall — Morris on a tree someone else reads.** It threads the tree while it walks. Never share it, and never leave a walk half finished.
- **Test with an oracle.** A tree *is* the set of its `L`/`R` path strings; every property in this lesson has a slow, obviously-correct definition over those strings.

## Checklist
- [ ] I can write the node, the empty case, and say how many `nil` links an n-node tree has.
- [ ] I can write the three depth-first orders from one skeleton and say which touch each keeps.
- [ ] I can convert each order to an explicit stack, and say what the stack holds.
- [ ] I can write level-order and say why the queue's size is the width.
- [ ] I can write height, depth and diameter, and say why one pass beats `height` at every node.
- [ ] I can solve the three path-sum problems and name the pitfall in each.
- [ ] I can invert a tree and compare two trees in lockstep.
- [ ] I can write an LCA and name the input that breaks the one-pass version.
- [ ] I can serialize with markers, deserialize with errors, and say why the markers are needed.
- [ ] I know which pairs of traversals determine a tree and which do not, and can say why.
- [ ] I can explain Morris threading and what it costs.
- [ ] I can say how a deep tree breaks recursion, `Drop`, and the stack on each language.
- [ ] I can write a tree in Rust with `Box`, with `Rc<RefCell<_>>` and with an arena, and choose between them.

## Resources
- Go's `go/ast` walkers — `Walk`, `Inspect`, `Preorder`: https://github.com/golang/go/blob/master/src/go/ast/walk.go
- Go's `iter` package — `Seq` and `Pull`: https://pkg.go.dev/iter
- Rust's `Option` representation (the null-pointer optimisation): https://doc.rust-lang.org/std/option/index.html#representation
- *The Rust Programming Language*, ch. 15 — `Box`, `Rc`, `RefCell`, reference cycles and `Weak`: https://doc.rust-lang.org/book/ch15-00-smart-pointers.html
- *Learn Rust With Entirely Too Many Linked Lists* — the `Box` list, and why `Drop` needs to be iterative: https://rust-unofficial.github.io/too-many-lists/
- J. M. Morris, "Traversing binary trees simply and cheaply", *Information Processing Letters* 9(5), 1979.
- Knuth, TAOCP vol. 1, §2.3.1 — traversing binary trees, and threaded trees.
- CLRS ch. 10.4 (representing rooted trees) and §12.1 (the in-order walk).
- Examples: [examples/17-binary-trees](examples/17-binary-trees/) (16).
- Next: [18 — Binary Search Trees](18-binary-search-trees.md) — the invariant that makes in-order sorted, and the shape that can ruin it.
