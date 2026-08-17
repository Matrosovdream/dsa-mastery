# 07 — Linked Lists

> Part of **Part 2 — Linear Structures**, and the mirror image of [06](06-arrays-slices.md). A slice
> gives you free random access and charges Θ(n) for structural change; a list does exactly the
> opposite. The one-line thesis: **a linked list buys Θ(1) splice-at-a-held-handle, and pays for it
> with an allocation and a cache miss per element — which in Go means it loses almost every
> comparison except the one it was designed for.**

## Goals
- Write the node struct and use `nil` as the empty list, with no special cases.
- Get the pointer-surgery order right, and recognize what happens when you don't.
- Remove the head special case three ways: a sentinel, a tail pointer, and pointer-to-pointer.
- Implement the classics: reverse, middle, Nth-from-end, cycle detection, merge.
- Know precisely when a list beats a slice — and be able to defend the answer with numbers.
- Replace pointers with an arena of indices when you need list semantics at scale.
- Build an LRU cache, the structure a doubly linked list exists for.

## Concepts

- **Two lines define the structure.** `type node struct { val T; next *node }`, and **`nil` is the
  empty list**. That's what lets every recursive function have exactly one base case — and it's a
  binary tree with one more field ([17](17-binary-trees.md)).

- **Order of assignment is the whole game.** Wire the new node up *first*, splice it in *second*.
  Reversed, you don't get a wrong answer — you get a node pointing at itself and the rest of the list
  gone, with no panic and no error. Every traversal then runs forever.

- **The sentinel deletes the head special case.** One extra node before everything means every real
  node has a predecessor. Three benefits: no head branch, no return value for callers to forget, and
  the **zero value works** (`var l List`) because the sentinel is an embedded value. This is what
  `container/list` does.

- **A tail pointer makes append Θ(1).** Without one, n appends is Θ(n²) — measured **116× slower** at
  n=3200. But `PopBack` stays Θ(n) even *with* a tail, because you need the node *before* it and a
  singly linked list cannot go backwards. That single sentence is why doubly linked lists exist.

- **The pointer-to-pointer idiom.** `pp := &head; pp = &(*pp).next` — `pp` always points at *the link
  that reaches the current node*, so assigning through it rewires the head and the middle identically.
  No `prev`, no sentinel, no special case, no allocation. The one trap: after unlinking, don't advance
  — the same shape as [06](06-arrays-slices.md)'s delete-in-a-forward-loop bug.

- **Doubly linked: prev buys Θ(1) removal of a node you hold.** Wire the sentinel into a **ring**
  (`sentinel.next` = first, `sentinel.prev` = last) and there are no nil checks anywhere: insert and
  remove are four unconditional assignments. A `*node` is then a **stable handle** — valid across
  every other insertion and deletion, which a slice index is not.

- **Reverse is three lines**, and the first one is the one people forget:
  ```go
  next := cur.next   // SAVE before you destroy the link
  cur.next = prev    // flip
  prev, cur = cur, next
  ```
  The recursive version is Θ(n) *stack* — [05](05-recursion.md)'s example 13, the shape that dies on real input.

- **Two pointers answer "middle" and "Nth from end" in one pass.** Different *speeds* find the middle;
  a fixed *offset* finds the Nth from the end. On an even-length list the loop condition decides which
  of the two middles you get — write it in the doc comment, it's a classic off-by-one.

- **Floyd's cycle detection is Θ(1) space.** Phase 1: slow +1, fast +2 must meet inside a cycle,
  because fast closes the gap by exactly one per step. Phase 2: reset one pointer to the head, move
  both by 1, and they meet at the cycle's start — because `2(L+k) = L+k+mC` gives `L = mC − k`. The
  `visited` map alternative is also Θ(n) time but Θ(n) space at ~38 bytes per node.

- **Merging is the one thing lists do better on their own terms**: Θ(1) space and **zero allocations**,
  because every output node already exists. The slice merge must allocate the whole output. This is
  why merge sort on a list needs no auxiliary array.

- **`container/list` is the ring-with-sentinel design, and it predates generics.** Every element is an
  `any`, so: **149,997 heap objects** for 100,000 ints (vs 10 for a slice), a type assertion on every
  read, **4.9× slower** traversal, and `PushBack("oops")` compiles.

- **The honest verdict, measured.** Traversal: slice wins **3.8×** against a freshly-built list and
  **20.8×** against an aged one. Building: slice wins **13.6×** even though the list is Θ(1) per
  prepend, because each prepend is a heap allocation. Memory: **99,995 heap objects vs 2**. Delete by
  handle: **list wins 25.5×**, and nothing else comes close. *"Insertion is O(1)" is true and almost
  never the reason.*

- **Arena lists: keep the nodes in one slice, link by `int32`.** You keep Θ(1) splice-by-handle and
  stable handles, and gain one allocation instead of n, contiguous memory, 4-byte links, and **no
  pointers at all** — so the GC skips the whole structure (**12–16× cheaper collection**, 217 heap
  objects vs 200,201). What you give up: type safety on the link, and per-node freeing. Note it is
  *slower* to traverse than a freshly-built pointer list (bounds checks, no locality gain) and **4.1×
  faster** than an aged one — because an arena cannot age.

- **Unrolled lists put an array in each node.** One cache miss buys k elements. At chunk=32:
  **1.3× of a slice** versus **4.1×** for a plain list, and **31× fewer heap objects**. Same idea as a
  B-tree ([19](19-balanced-trees.md)) — when a hop is expensive, make each hop deliver more.

## Complexity Table

| Operation | Slice | Singly linked | Doubly linked |
|-----------|-------|---------------|---------------|
| Index `i` | **Θ(1)** | Θ(i) | Θ(i) |
| Insert/delete at front | Θ(n) | **Θ(1)** | **Θ(1)** |
| Insert at back | Θ(1) amortized | Θ(1) *with a tail* | **Θ(1)** |
| Delete at back | Θ(1) | **Θ(n)** | **Θ(1)** |
| Insert/delete at a **held** position | Θ(n) | Θ(1) *after* | **Θ(1)** |
| Search | Θ(n) | Θ(n) | Θ(n) |
| Merge two sorted | Θ(n) time, **Θ(n) space** | Θ(n) time, **Θ(1) space** | — |
| Memory per element | 8 bytes | 16 + one object | 24 + one object |

Measured (Apple M4, Go 1.26.3): traversal **3.8×** (fresh) / **20.8×** (aged) slower than a slice ·
building **13.6×** slower · delete-by-handle **25.5×** faster · `container/list` **4.9×** slower ·
arena GC **12–16×** cheaper · unrolled (k=32) **1.3×** of a slice · LRU eviction vs a map scan
**394×** at n=4096.

## Exercises
1. Write the node struct, `prepend`, `appendTail`, `length` and `at`. Explain why `nil` needs no special case in the recursive versions.
2. Write `insertAfter` correctly, then write it with the two assignments swapped. Print the result with a depth limit and explain what happened.
3. Implement delete-by-value with and without a sentinel. Count the branches each needs.
4. Add a tail pointer and a size counter. Benchmark n appends with and without the tail, and show the Θ(n²)-vs-Θ(n) signature. Then explain why `PopBack` is still Θ(n).
5. Build a doubly linked list as a sentinel **ring**. Implement `Remove(node)` and `MoveToFront(node)` with no branches.
6. Write "remove all matching" three ways: with `prev`, with a sentinel, and with pointer-to-pointer. Test the all-removed and consecutive-removal cases.
7. Reverse a list iteratively and recursively, and trace the three pointers. Then implement `reverseBetween(m, n)`.
8. Find the middle and the Nth-from-end in one pass. Show which middle each loop condition picks on an even-length list.
9. Implement Floyd's detection, cycle start, and cycle length. Write out the algebra that makes phase 2 work.
10. Merge two sorted lists with zero allocations. Then merge k lists pairwise, and write merge sort on a list.
11. Use `container/list`. Measure its heap objects and traversal against a slice, and demonstrate that it accepts a value of the wrong type.
12. Benchmark list vs slice on three workloads: traversal (fresh *and* aged), building, and delete-by-handle. Write the verdict yourself.
13. Reimplement the doubly linked list as an arena of `int32` indices with a free list. Measure traversal and GC against pointer lists both fresh and aged.
14. Implement `isPalindrome` in Θ(1) space by composing the middle and reverse — and restore the list afterwards. Then implement `reorder`.
15. Implement an unrolled list with chunk=32. Compare traversal and heap objects against a plain list and a slice.
16. Stretch — build an **LRU cache** (map + doubly linked list), apply all three passes, and benchmark eviction against a map-plus-timestamp design that scans to find the LRU.

## Best Practices & Pitfalls
- **Default to a slice.** Reach for a list only when you hold stable handles and must splice in Θ(1).
- **Use a sentinel** in every list you write. One node, and every head special case disappears.
- **Wire the new node first, splice second.** Never overwrite a pointer while it is the only way to reach something.
- **Pitfall — advancing after unlinking.** In the pointer-to-pointer idiom, `*pp` is already the next node.
- **Pitfall — assuming a fresh list benchmarks like a real one.** A freshly built list has the best layout it will ever have; measure a shuffled one too.
- **Pitfall — `container/list` for value types.** Boxing costs an object per element, a type assertion per read, and all compile-time safety.
- **Nil the links of a removed node** (`n.prev, n.next = nil, nil`) — it helps the GC and makes use-after-remove fail loudly.
- **A function that mutates its input to save memory must restore it**, or document loudly that it consumes the list.
- **Pitfall — benchmarking a consuming operation.** `merge` rewires its inputs; running it twice on the same pair builds a cycle and hangs. (This bit me writing example 10.)
- **Prefer an arena of indices at scale.** One allocation, no GC tracing, contiguous memory — and the same Θ(1) splice.
- **Consider an unrolled list** before a plain one: k values per node recovers most of the locality for none of the algorithmic cost.

## Checklist
- [ ] I can write the node struct and explain why `nil` is the empty list.
- [ ] I get the pointer-surgery order right, and can describe the failure mode when it's wrong.
- [ ] I use a sentinel by default and can name the three things it buys.
- [ ] I can explain why `PopBack` is Θ(n) on a singly linked list even with a tail pointer.
- [ ] I can write the pointer-to-pointer removal loop and its one trap.
- [ ] I can reverse a list from memory, and say why the recursive version is the wrong choice.
- [ ] I can find the middle and the Nth-from-end in a single pass.
- [ ] I can state Floyd's two phases and the algebra behind the second.
- [ ] I can defend "use a slice" with measured numbers, and name the one workload where a list wins.
- [ ] I can convert a pointer list to an index arena and say what I gave up.
- [ ] I can build an LRU cache and explain why neither a map nor a slice can do it alone.

## Resources
- `container/list`: https://pkg.go.dev/container/list
- `container/ring` — a circular list: https://pkg.go.dev/container/ring
- Floyd's cycle-finding algorithm: https://en.wikipedia.org/wiki/Cycle_detection#Floyd's_tortoise_and_hare
- Unrolled linked list: https://en.wikipedia.org/wiki/Unrolled_linked_list
- LRU cache: https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_Recently_Used_(LRU)
- `golang.org/x/exp` and `hashicorp/golang-lru` — production LRUs built on exactly this pair.
- CLRS ch. 10.2 — linked lists, sentinels, and the ring representation.
- Examples: [examples/07-linked-lists](examples/07-linked-lists/) (16).
- Next: [08 — Stacks](08-stacks.md) — the structure that only ever touches the cheap end, and therefore never has this lesson's problem.
