# Step 17 — Binary Trees & Traversals · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The node, and `nil` as the empty tree**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The node, and `nil` as the empty tree
- [ ] 2. Pre, in and post: where the visit goes
- [ ] 3. The call stack, written as a loop
- [ ] 4. Level order: the queue holds one layer
- [ ] 5. Height, depth and diameter
- [ ] 6. Invert and prune: assign what the child returns

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Path sums: down, up and anywhere
- [ ] 8. Two trees at once: same, mirror, subtree
- [ ] 9. Lowest common ancestor, and the missing node
- [ ] 10. Serialize and deserialize: one marker per nil
- [ ] 11. Rebuild a tree from two traversals

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Morris traversal: in-order with no stack
- [ ] 13. A tree you did not build: frames, depth and Drop
- [ ] 14. One tree, three owners: Box, Rc and an arena
- [ ] 15. A traversal that can stop: iterators
- [ ] 16. Capstone: a `Tree` toolkit, proved and measured

## The three-pass build

In `practice/17-binary-trees/`:

- [ ] **Build it** — `Node`, a seeded shape generator, and recursive `PreOrder` / `InOrder` / `PostOrder`
- [ ] **Build it** — the same three iteratively (post-order with a frame stack), and level-order with the snapshot idiom
- [ ] **Build it** — `Height`, `Diameter` (one pass), `MaxPathSum`, `Invert`, `LCA`
- [ ] **Build it** — `Serialize` / `Deserialize` with a marker per `nil`, returning an error on malformed input
- [ ] **Build it** — `BuildPreIn` with a value → index map, and Morris in-order
- [ ] **Prove it** — an oracle that stores every node as its `L`/`R` path string, over thousands of random small trees
- [ ] **Prove it** — every shape of n ≤ 8 nodes enumerated, to check which traversal pairs identify a tree
- [ ] **Prove it** — a round trip through the serializer on random trees, including the empty tree
- [ ] **Prove it** — the same functions on a million-node chain without a stack overflow
- [ ] **Measure it** — maximum stack and queue size per shape, and ns per node per traversal
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What is a binary tree, as a definition? | empty, or a node with a value and two binary trees |
| How many `nil` links does an n-node tree have? | **n + 1** (2n child slots, n − 1 used) |
| How many calls does a full recursive traversal make? | **2n + 1** — n with work, n + 1 base cases |
| Height range of an n-node tree, in nodes? | ⌈log₂(n+1)⌉ ≤ h ≤ n |
| What decides the space a depth-first walk needs? | the **height**, not n |
| The three depth-first orders differ in…? | where the visit goes: before, between, after the two calls |
| Why is iterative post-order the hard one? | a popped node is not done until both subtrees are |
| What does a level-order queue's size measure? | the **width** of the widest level |
| One pass for the diameter: what is returned and what recorded? | the height is returned; the best `left + right` is recorded |
| Why is `height` at every node quadratic on a chain? | it sums every subtree's size |
| The prefix-sum trick on a tree needs what, on the way back up? | **undo** the map entry |
| Starting a maximum path sum at 0 breaks on…? | an all-negative tree |
| When is the one-pass LCA wrong? | when one of the two nodes is not in the tree |
| Why does serialization need a marker per `nil`? | the markers say where each subtree ends |
| Which pairs of traversals determine a tree? | pre + in, post + in. **Not** pre + post (except full trees) |
| Why can Morris run in O(1) space? | it threads the in-order predecessor's `Right` back to the current node |
| What does Morris cost you? | the tree is modified while it walks |
| Why does a Rust `Box` tree need an iterative `Drop`? | the default drop recursion is one nested drop per level |
| `Box`, `Rc<RefCell<_>>`, arena: which for parent links? | `Rc` + `Weak`, or — better — an index arena |

> Lesson: [17-binary-trees.md](../../17-binary-trees.md) · Index: [README.md](README.md)
