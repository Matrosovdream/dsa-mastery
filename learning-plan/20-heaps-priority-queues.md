# 20 — Heaps & Priority Queues

> Part of **Part 4 — Trees**, and the tree that is never built out of nodes. A binary search tree
> ([18](18-binary-search-trees.md)) keeps *everything* in order and pays for it in pointers and
> rebalancing. A heap keeps only the **minimum** in a promised place, stores itself in a plain slice,
> and needs no balancing at all, because a complete tree's shape is fixed by its element count. The
> one-line thesis: **a heap is not sorted — it is ordered just enough to answer one question, "what is
> the smallest?", and almost every problem it solves is a problem where that is the only question asked.**

## Goals
- Derive the index arithmetic of an array-backed binary heap — parent, children, leaves, height — and state its invariant.
- Write **sift-up** and **sift-down**, and count their comparisons and swaps.
- Show that **heapify** is Θ(n), and that building by n separate `Push`es is a different algorithm.
- Implement `heap.Interface` correctly, call the *package* functions instead of your own methods, and say what `Pop`'s swap with the last element is for.
- Build min, max and generic heaps in Go and Rust, and break ties on purpose.
- State what `Less` / `Ord` must satisfy, and what `NaN` does to it.
- Solve top-K, merge-k-sorted, running median, scheduling, Huffman coding and the Dijkstra frontier with a heap, and say what each one's heap *holds*.
- Change or cancel an entry without a decrease-key — `Fix` / `Remove` with an index, or **lazy deletion** — and name what each standard library does and does not give you.

## Concepts

- **A heap is a partial order stored in an array.** Lay a complete binary tree out level by level:
  node `i` has children `2i+1` and `2i+2` and parent `⌊(i-1)/2⌋`. The min-heap invariant is that no
  child is less than its parent — `container/heap` states it as `!h.Less(j, i)` for
  `2*i+1 <= j <= 2*i+2`. There are no pointers and nothing to balance: the shape depends only on `n`,
  the height is `⌊lg n⌋`, and the leaves are indices `⌊n/2⌋ … n-1` — **half of all nodes are leaves**.

- **A heap is not sorted.** Siblings are unordered, and only the minimum has a promised position (the
  second smallest is at index 1 or 2; the k-th smallest has no promised place). A sorted array *is* a
  heap; a heap is almost never sorted. Of the `n!` orderings of `n` distinct keys, exactly
  `n! / ∏ (subtree sizes)` are heaps — [example 1](examples/20-heaps-priority-queues/1-easy.md) checks
  that exhaustively for small `n`.

- **The `-1` trap.** `(j-1)/2` at `j = 0`: Go's integer division truncates toward zero, so the parent
  of the root comes out as 0 (which is why `container/heap`'s `up` tests `i == j`). Rust's `usize`
  has no −1 — `0 - 1` panics in a debug build and wraps in a release build — so the loop condition
  is `pos > 0`.

- **Sift up is `Push`.** Append at the end (the only place the shape allows), then swap with the
  parent while smaller: at most `⌊lg n⌋` swaps. Worst case Θ(log n), but on random-order input the
  expected cost is Θ(1) — Rust's `push` documentation says so and warns that it degrades to Θ(log n)
  when elements arrive in the "wrong" direction ([example 2](examples/20-heaps-priority-queues/1-easy.md)
  counts all three shapes).

- **Sift down is `Pop`.** Remove the root, move the *last* element into its place, and sink it. Each
  level costs two comparisons — pick the smaller child, then compare it with the sinking element — so
  `Pop` is at most `2⌊lg n⌋` comparisons. [Lesson 13](13-efficient-sorts.md) already counted this loop
  inside heapsort; here it is the whole operation.

- **Heapify is Θ(n).** Floyd's construction sifts down every internal node, from `⌊n/2⌋-1` back to 0.
  Half the nodes are leaves and cost nothing; a quarter sift at most one level, an eighth at most two,
  so the total number of levels descended is `Σ h · n/2^(h+1) < n`, and the comparisons are at most
  `2n` (Rust's source says "about 2 * self.len() comparisons in the worst case"). `n` separate
  `Push`es are *not* the same algorithm: Θ(n log n) worst case, though Θ(n) on random input — so
  heapify's edge is the **guarantee**, plus that it works in place
  ([example 4](examples/20-heaps-priority-queues/1-easy.md)). `heap.Init` and `BinaryHeap::from(vec)`
  are both exactly this.

- **`container/heap` is five methods and five functions.** `heap.Interface` is `sort.Interface` (`Len`,
  `Less`, `Swap`) plus `Push(x any)` and `Pop() any`; the functions are `Init`, `Push`, `Pop`, `Fix`
  and `Remove`. The root is the minimum, at index 0. There is no generic version — the package
  predates generics and moves every element through `any`.

- **The contract everyone breaks once.** The `Push` and `Pop` *methods* on your type are hooks for the
  package: yours only appends, or removes the **last** element. Adding is `heap.Push(h, x)` (your
  `Push`, then sift up); removing is `heap.Pop(h)`, which calls `Swap(0, n-1)`, sifts the new root
  down within the first `n-1` elements, and **then** calls your `Pop` — which removes the old root,
  now sitting at the tail. A slice can only shrink at its tail; swapping first is what lets your
  `Pop` be a plain "remove last". Calling `h.Push(x)` directly silently breaks the invariant, and
  `h.Pop()` returns the last element, not the minimum ([example 3](examples/20-heaps-priority-queues/1-easy.md)
  prints the order of calls). `Push` and `Pop` need pointer receivers because they change the slice's
  length; for pointer elements the package's own example clears the vacated slot so the GC can reclaim it.

- **Min vs max is only `Less`.** `h[i] < h[j]` is a min-heap, `>` is a max-heap. **Ties are
  unspecified**: neither language's heap is stable, so equal priorities pop in whatever order the
  sifts left them. For first-in-first-out within a priority, compare `(priority, sequence number)`
  ([example 5](examples/20-heaps-priority-queues/1-easy.md)).

- **`Less` must be a strict weak order, and floats are not one.** `<` on a `NaN` is always false, so a
  `NaN` is "equal" to everything and equality stops being transitive: the heap does not crash, it
  misorders. `cmp.Less` and `cmp.Compare` define a total order (NaN smallest, `-0 == +0`). Rust
  refuses at compile time: `f64` is not `Ord`, so `BinaryHeap<f64>` does not build — wrap it with
  `total_cmp`, or map the bits to an ordered integer as [lesson 14](14-linear-sorts.md) did
  ([example 6](examples/20-heaps-priority-queues/1-easy.md)).

- **Top-K keeps a heap of the k best so far.** For the k largest, hold a *min*-heap of size k: each
  new `x` is compared with the root and replaces it only if larger. Θ(n log k) time, Θ(k) space, one
  pass. For random-order input the expected number of replacements is `Σ k/i ≈ k ln(n/k)`, so the
  typical cost is far below the bound; the worst case is input that keeps beating the root. Quickselect
  ([lesson 13](13-efficient-sorts.md)) is Θ(n) on average but needs the whole input in memory and
  rearranges it; the heap streams ([example 7](examples/20-heaps-priority-queues/2-medium.md)).

- **Merging k sorted lists: a heap of the k heads.** Θ(N log k) for N elements. It is not
  usually the cheapest in comparisons — a balanced tree of pairwise merges does at most `N⌈lg k⌉`, though
  at k = 2 every strategy ties and a plain left-to-right fold can win at small k; example 8 counts all
  four — but it is
  **lazy**: the first `m` outputs cost Θ(k + m log k), which pairwise merging cannot match. And when
  the lists have unequal sizes, the pairwise order that minimises the total size merged is Huffman's —
  always merge the two shortest — itself a heap problem. [Lesson 13](13-efficient-sorts.md)'s external
  sort used this merge on disk; example 8 compares the strategies.

- **Running median: two heaps.** A max-heap holds the lower half and a min-heap the upper half;
  keep their sizes within one and the top of the lower half ≤ the top of the upper half. Each insert
  is one push and at most one move across: Θ(log n); the median is read in Θ(1). The same split gives
  any fixed-rank statistic of a growing set.

- **Scheduling is "of what I can do now, which matters most?"** Meeting rooms: a min-heap of end
  times, and its final size is the number of rooms needed. Task scheduler with a cooldown: a max-heap
  by remaining count plus a queue of tasks that are cooling down; a closed form exists, which makes it
  a clean oracle ([example 10](examples/20-heaps-priority-queues/2-medium.md)).

- **There is no decrease-key — in either standard library.** Go offers `Fix(h, i)` and `Remove(h, i)`,
  but they take an *index*, so you must maintain each element's index inside your own `Swap`; Rust
  offers nothing equivalent (`peek_mut` reaches only the top). The portable answer is **lazy
  deletion**: push a new entry, and discard the stale one when it surfaces. The bill is the stale
  entries: each costs a pop *if it ever surfaces*, and if it never does (a cancelled timer due next
  year) it occupies space until you compact ([example 11](examples/20-heaps-priority-queues/2-medium.md)).

- **Huffman coding is a heap loop.** Repeatedly pop the two least-frequent trees and push their
  merge: `n-1` merges, `2(n-1)` pops and `n-1` pushes. The resulting prefix code is optimal, and its
  average length `L` satisfies `H ≤ L < H + 1` for the entropy `H` (for two or more symbols — one symbol
  has `H = 0` yet still needs a 1-bit code). *Why* greedy is optimal is an
  exchange argument ([lesson 27](27-greedy.md)); [example 12](examples/20-heaps-priority-queues/3-hard.md)
  checks it against exhaustive search instead. Different tie-breaks give different trees of the same cost.

- **Dijkstra's heap is the frontier.** Keyed by tentative distance, it pops vertices in non-decreasing
  distance order when weights are non-negative. With lazy deletion the heap can hold up to `E+1`
  entries; with an indexed `Fix` at most `V`; both are Θ((V+E) log V). Shortest paths proper —
  negative edges, Bellman–Ford, A\* — are [lesson 24](24-shortest-paths.md); when weights are only 0
  and 1 a deque replaces the heap (lesson 09, example 13).

- **Heapsort is the heap used as a sort.** Heapify, then `n-1` times swap the root to the tail and
  sift down. [Lesson 13](13-efficient-sorts.md) built and counted it in place; Rust's
  `into_sorted_vec` is that loop over the heap's own buffer.

- **The sinking element came from the bottom, so it usually goes back to the bottom.** Rust's `pop`
  exploits that: `sift_down_to_bottom` moves a hole down the larger-child path with **one**
  comparison per level, then sifts the element back up a short way — the source comment says it is
  "faster when the element is known to be large / should be closer to the bottom". Go's `down` and
  Rust's own `into_sorted_vec` use the two-comparisons-per-level form. A **d-ary heap** has depth
  `log_d n`: fewer levels to push through, more children to scan per level. Example 15 counts both.

- **A heap does not search.** Finding an arbitrary value is Θ(n); iteration is in array order, not
  priority order; there is no ordered traversal and merging two binary heaps is Θ(n). If you need
  those, you want a tree ([18](18-binary-search-trees.md)–[19](19-balanced-trees.md)) or a map
  *plus* a heap.

### Go and Rust side by side

- **`BinaryHeap<T: Ord>` is a max-heap**; Go's heap is whatever `Less` says. A min-heap in Rust is `Reverse(x)` (`core::cmp::Reverse`) or a reversed `Ord`.
- **Ordering lives in the element type** (`Ord`), not in the container. To heap by one field, wrap the item in a struct with its own `Ord`, or store a `(priority, item)` tuple; Go puts any rule in `Less`.
- **`BinaryHeap<f64>` does not compile** (`f64: Ord` is not implemented, error E0277); Go accepts floats and misorders on `NaN`. Use `total_cmp` or an ordered integer.
- **No interface and no `any`**: `push` and `pop` are methods, so safe code cannot skip the sift, and `pop` returns `Option<T>` where `heap.Pop` on an empty heap calls `Swap(0, -1)` (a slice-backed `Swap` panics).
- **`peek_mut()` returns a guard**: assign through it and the re-sift runs when it drops — Θ(log n) if you wrote, Θ(1) if you did not, per the source's own docs. `PeekMut::pop` removes the top. It reaches only the top: no `Fix`, no `Remove`, no handle.
- **Changing a key while it is inside is a logic error**: documented as unspecified but contained (no undefined behaviour), and in Rust it needs `Cell`/`RefCell` to even try. Go lets you do it with no warning ([example 11](examples/20-heaps-priority-queues/2-medium.md)).
- **`BinaryHeap::from(vec)` is `heap.Init`** — in place, Θ(n); `append` and `extend` choose between a rebuild and repeated sift-ups, from the sizes. `iter()` walks array order; `into_sorted_vec()` consumes the heap and returns ascending order; `into_vec()` is the array itself.
- **Different sifts**: Rust's `pop` is bottom-up, its `into_sorted_vec` and `heap.Pop` are the textbook two-comparison form; examples 14 and 15 count the difference.

## Complexity Table

| Operation / structure | Time | Space | Note |
|---|---|---|---|
| Peek min (or max) | **Θ(1)** | | index 0 |
| `Push` | Θ(log n) worst, Θ(1) on random order | | ≤ ⌊lg n⌋ swaps |
| `Pop` (remove the root) | **Θ(log n)** | | ≤ 2⌊lg n⌋ comparisons, textbook sift |
| Heapify (`Init`, `from(vec)`) | **Θ(n)** | O(1) in place | ≤ 2n comparisons |
| Build by n `Push`es | Θ(n log n) worst | | the guarantee heapify buys |
| `Fix` / `Remove` at a held index | Θ(log n) | | you maintain the index |
| Find or delete by value | **Θ(n)** | | a heap does not search |
| Merge two binary heaps | Θ(n) | | rebuild, or sift the smaller in |
| Heapsort | Θ(n log n) | O(1) | not stable; lesson 13 |
| d-ary heap push / pop | Θ(log_d n) levels | | pop scans d children per level |
| Sorted slice, as a priority queue | Θ(n) insert, Θ(1) pop of the end | | lesson 06 |
| Balanced BST, as a priority queue | Θ(log n) everything | Θ(n) pointers | also ordered iteration |
| Top-K of n | **Θ(n log k)** | Θ(k) | ≈ k ln(n/k) replacements on random order |
| Merge k sorted lists, heap | Θ(N log k) | Θ(k) | lazy: first m outputs Θ(k + m log k) |
| Merge k sorted lists, balanced pairwise | ≤ N⌈lg k⌉ comparisons | Θ(N) | eager |
| Running median | Θ(log n) per element, Θ(1) read | Θ(n) | two heaps |
| Huffman coding | Θ(n log n) | Θ(n) | n−1 merges |
| Dijkstra, lazy deletion | Θ((V+E) log V) | O(E) heap | stale entries surface as pops |
| Dijkstra, indexed with `Fix` | Θ((V+E) log V) | O(V) heap | needs the index bookkeeping |

This table states only what follows from the algorithms; the examples print the counts and timings.

## Exercises
Write these in `practice/20-heaps-priority-queues/`.
1. Write `parent`, `left`, `right` and a printer that draws a heap array as a tree. Verify the index laws for every `n` up to 1,000, then count the valid min-heaps of `n` distinct keys by brute force for `n ≤ 9` and match `n! / ∏ subtree sizes`.
2. Write sift-up and sift-down, counting comparisons and swaps. Measure `Push` on random, ascending and descending input, and `Pop` while draining; explain each from the code.
3. Implement `heap.Interface` for ints. Log every method call during one `heap.Push` and one `heap.Pop`; then call `h.Push` and `h.Pop` directly and show what breaks.
4. Heapify two ways — Floyd's sift-down loop and `n` pushes — on random, sorted and reversed input. Count comparisons, and find the input where the gap is Θ(log n) per element.
5. Build min and max heaps from one generic `PQ[T]` and a `less` function. Then make equal priorities pop first-in-first-out, and count how often they did not before.
6. Put one `NaN` into each of many random float heaps. Count the drains that come out unsorted with `<`, then fix it with `cmp.Less`. In Rust, build `BinaryHeap` over floats with `total_cmp`, and again with an ordered `u64`.
7. Return the k largest of a stream in Θ(n log k). Count comparisons on random and ascending input against sorting, and check against the sorted prefix on a narrow value range.
8. Merge k sorted slices three ways — heap of heads, sequential fold, balanced pairwise — and count comparisons for equal and for very unequal sizes. Then take only the first 10 outputs.
9. Implement the running median with two heaps. Assert both invariants after every insert, and compare with a sorted-slice oracle on streams with many duplicates.
10. Compute the minimum number of meeting rooms, and the length of the cooldown task schedule; check the second against a closed form and an exhaustive search on tiny inputs.
11. Reschedule and cancel timers three ways: `Fix`/`Remove` with an index, lazy deletion, and lazy deletion with compaction. Report the maximum heap size and the stale pops.
12. Build a Huffman code from a text. Check the Kraft sum, encode and decode round-trip, `H ≤ L < H + 1`, and optimality against exhaustive merging on alphabets of up to 7 symbols.
13. Run Dijkstra with lazy deletion and with `Fix` on a random graph. Check both against a relaxation-to-fixed-point oracle, and show the settle order is non-decreasing in distance.
14. Count the allocations of `heap.Push` and `heap.Pop` for small ints, large ints, structs and pointers, and the `Less` and `Swap` calls per operation. Compare with a hand-written generic heap.
15. Write a bottom-up `Pop` and a 4-ary heap. Count comparisons per pop, check all three produce the same sorted output, and rebuild heapsort on the cheaper sift.
16. Stretch — package `PQ[T]` with handles (`Push` returns one; `Update`, `Remove`, `Peek`, an O(n) `FromSlice`) and apply all three passes: a model oracle over 100,000 random operations on a narrow priority range, invariants after every operation including the index map, and a benchmark against `container/heap`.

## Best Practices & Pitfalls
- **Use `container/heap` or `BinaryHeap`.** Write a heap yourself to learn the index math, then delete it.
- **Pitfall — calling your own `Push` / `Pop`.** Only `heap.Push(h, x)` and `heap.Pop(h)` sift. Nothing complains when you forget.
- **Pitfall — passing a value where a pointer is needed.** `Push` and `Pop` have pointer receivers; pass `&h`, and expect a compile error if you do not.
- **Pitfall — changing a priority in place without `heap.Fix`.** And `Fix` needs the element's current index: keep it up to date in `Swap`, and set it to −1 in `Pop`.
- **Pitfall — a stale pointer in the vacated slot.** A slice-backed heap's `Pop` should nil the slot it gives up, or the garbage collector keeps the element alive.
- **Pitfall — expecting FIFO among equal priorities.** Add a sequence number to the comparison.
- **Pitfall — floats.** `<` with `NaN` misorders silently in Go; `f64` is not `Ord` in Rust. Use `cmp.Less` / `total_cmp`.
- **Pitfall — forgetting that `BinaryHeap` is a max-heap.** Dijkstra and top-K need `Reverse`, or the wrong end comes out.
- **Pitfall — iterating a heap and expecting priority order.** `range h` and `iter()` give array order; only repeated `Pop` gives sorted order.
- **Pitfall — lazy deletion without a stale check on pop,** or without compaction when stale entries may never surface.
- **Pitfall — building by `n` pushes when you hold all the data.** `heap.Init` / `BinaryHeap::from(vec)` is Θ(n) with a guarantee.
- **Pitfall — a heap for top-K when k is close to n,** or for a single k-th element. Sort, or use quickselect.
- **Pitfall — Dijkstra with a negative edge.** Pop order is no longer settle order; that is lesson 24's subject.
- **Pitfall — leaking a Rust `PeekMut` after writing through it.** The guard shortens the heap to one element while it is live and restores it on drop; forgetting it leaves the rest leaked.

## Checklist
- [ ] I can derive parent, children, leaves and height of an array heap, and state the invariant.
- [ ] I can write sift-up and sift-down and say how many comparisons each spends per level.
- [ ] I can prove heapify is Θ(n), and say when `n` pushes are as cheap anyway.
- [ ] I know what `heap.Pop` does before it calls my `Pop`, and why.
- [ ] I call `heap.Push` / `heap.Pop`, never my own methods.
- [ ] I can make a min-heap, a max-heap and a first-in-first-out priority queue in Go and in Rust.
- [ ] I know why `BinaryHeap<f64>` will not compile, and what `NaN` does to a Go heap.
- [ ] I can write top-K, merge-k and the running median, and say what each heap holds.
- [ ] I know neither standard library has a decrease-key, and I can use `Fix` or lazy deletion.
- [ ] I can say what lazy deletion costs when stale entries surface and when they do not.
- [ ] I can build a Huffman code and say how its length compares with the entropy.
- [ ] I can say why Rust's `pop` and `into_sorted_vec` sift differently, and what `peek_mut` does on drop.

## Resources
- Go's `container/heap` — short enough to read in full: https://github.com/golang/go/blob/master/src/container/heap/heap.go
- …and its priority-queue example, with `update` and `Fix`: https://github.com/golang/go/blob/master/src/container/heap/example_pq_test.go
- Rust's `BinaryHeap` — the module comments on `sift_down_to_bottom` and `PeekMut` are worth reading: https://github.com/rust-lang/rust/blob/master/library/alloc/src/collections/binary_heap/mod.rs
- `BinaryHeap` documentation, including its Dijkstra example: https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html
- Floyd, "Algorithm 245: Treesort" (CACM, 1964) — linear-time heap construction; Wegener, "Bottom-up heapsort" (Theoretical Computer Science, 1993) — the one-comparison-per-level sift.
- CLRS ch. 6 (heaps and priority queues), §16.3 (Huffman codes), §24.3 (Dijkstra); Sedgewick & Wayne, *Algorithms* 4th ed., §2.4: https://algs4.cs.princeton.edu/24pq/
- Huffman, "A Method for the Construction of Minimum-Redundancy Codes" (1952); Knuth, TAOCP vol. 1 §2.3.4.5 and vol. 3 §5.2.3.
- Examples: [examples/20-heaps-priority-queues](examples/20-heaps-priority-queues/) (16).
- Next: [21 — Tries & Prefix Structures](21-tries.md) — a tree keyed by prefixes, and a heap on top of it for autocomplete.
