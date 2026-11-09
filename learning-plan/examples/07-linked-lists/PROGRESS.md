# Step 07 — Linked Lists · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The node, and `nil` as the empty list**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The node, and `nil` as the empty list
- [ ] 2. Wire first, splice second
- [ ] 3. The sentinel, and the special case it deletes
- [ ] 4. A tail pointer, and why `PopBack` stays Θ(n)
- [ ] 5. Pointer-to-pointer: one loop for head and middle
- [ ] 6. The doubly linked ring

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Reverse in three lines, and the recursion that dies
- [ ] 8. Middle and Nth from end, in one pass
- [ ] 9. Floyd's cycle detection
- [ ] 10. Merging with zero allocations
- [ ] 11. `container/list`, read in the source

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. When a list beats a slice (rarely), measured
- [ ] 13. The arena: links without pointers
- [ ] 14. Palindrome in Θ(1) space, and putting it back
- [ ] 15. The unrolled list
- [ ] 16. Capstone: an LRU cache, built, proved, measured

## The three-pass build

In `practice/07-linked-lists/`:

- [ ] **Build it** — `List[T]` (singly, sentinel, tail pointer, size) and `DList[T]` (sentinel ring)
- [ ] **Build it** — reverse, middle, Nth-from-end, Floyd (start + length), merge, merge sort
- [ ] **Build it** — `LRU[K, V]`: `map[K]*node` + `DList`, `Get` moves to front, `Put` evicts the back
- [ ] **Prove it** — model oracle against `[]T` for every list operation, over random op sequences
- [ ] **Prove it** — invariants after *every* operation: `size` matches a walk, forward walk == reversed backward walk, `tail.next == nil`
- [ ] **Prove it** — exhaustive small cases (every list of length 0..10) for reverse-between, remove-Nth, palindrome, reorder
- [ ] **Prove it** — functions that mutate their input to save space leave it **identical** afterwards
- [ ] **Prove it** — LRU against a brute-force recency-ordered slice, narrow key range, every result and every eviction compared
- [ ] **Measure it** — traversal fresh *and* aged, building, delete-by-handle, against `[]T`
- [ ] **Measure it** — heap objects per element, and allocations per merge (should be 0)
- [ ] **Measure it** — LRU eviction against a map-plus-timestamp design, at three capacities
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What is the empty list? | `nil` — one base case, no special code |
| Which assignment comes first in an insert? | wire the new node (`n.next = p.next`), **then** splice (`p.next = n`) |
| What does the wrong order produce? | a node pointing at itself, and the rest unreachable — no panic |
| Three things a sentinel buys? | no head branch, no returned head, a usable zero value |
| Why is `PopBack` Θ(n) with a tail pointer? | you need the node *before* the tail; a singly list can't go back |
| What does `pp` point at in pointer-to-pointer? | the **link** that reaches the current node |
| Its one trap? | advancing after unlinking — `*pp` is already the next node |
| What does `prev` buy? | Θ(1) removal of a node you hold |
| Why a ring? | no nil checks: insert/remove are four unconditional writes |
| The line people forget in reverse? | saving `next` before flipping the link |
| Why not reverse recursively? | Θ(n) stack |
| Which middle does `fast != nil && fast.next != nil` give on even n? | the **second** |
| Floyd phase 2 — why does it work? | `2(L+k) = L+k+mC` ⇒ `L = mC − k` |
| What is special about merging lists? | Θ(1) space and **zero allocations** — every node already exists |
| Why can't you merge the same pair twice? | merge rewires its inputs: the second run builds a cycle |
| What does `container/list` cost for value types? | an object per element, a type assertion per read, no compile-time type check |
| The one workload a list wins? | removing / splicing at a handle you already hold |
| What does an arena give up? | type safety on the link, and per-node freeing |
| Why does an unrolled list traverse faster? | one hop delivers k elements |
| Why does an LRU need both a map and a list? | the map finds the entry; the list orders by recency and moves in Θ(1) |

## Numbers to find on your own machine

Measure these yourself — the examples print them, and none of them should be taken on trust.

- [ ] How much slower is traversal on an **aged** (shuffled) list than on a freshly built one?
- [ ] How many heap objects does a 100,000-element list cost, against a slice?
- [ ] Is building a list by prepends faster or slower than `append` — despite Θ(1) per prepend?
- [ ] By how much does delete-by-handle beat `slices.Delete`?
- [ ] How much cheaper is a GC cycle with an arena live than with a pointer list?
- [ ] How close does an unrolled list (k = 32) get to a slice on traversal?
- [ ] How much faster is LRU eviction than a timestamp scan at capacity 4,096?

> Lesson: [07-linked-lists.md](../../07-linked-lists.md) · Index: [README.md](README.md)
