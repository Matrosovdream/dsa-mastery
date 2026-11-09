# Step 20 — Heaps & Priority Queues · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The heap is an array**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The heap is an array
- [ ] 2. Sift up and sift down, counted
- [ ] 3. `container/heap`: five methods and a call order
- [ ] 4. Heapify is Θ(n), and n Pushes are not
- [ ] 5. Min, max, generic, and ties
- [ ] 6. Floats, `NaN` and the `Less` contract

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Top-K in Θ(n log k)
- [ ] 8. Merging k sorted lists
- [ ] 9. Running median with two heaps
- [ ] 10. Scheduling: meeting rooms and cooldowns
- [ ] 11. Cancel and reschedule: lazy deletion and what it leaks

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Huffman coding, against the entropy bound
- [ ] 13. Dijkstra's frontier, and the order it settles
- [ ] 14. What `container/heap` costs
- [ ] 15. Pop with fewer comparisons: bottom-up and d-ary
- [ ] 16. Capstone: `PQ[T]` with handles, built, proved, measured

## The three-pass build

In `practice/20-heaps-priority-queues/`:

- [ ] **Build it** — an array-backed min-heap: `parent`/`left`/`right`, `Push` (sift up), `Pop` (sift down), `Init` (Floyd heapify)
- [ ] **Build it** — `container/heap` over the same data: the five methods, pointer receivers, only `heap.Push` / `heap.Pop` called
- [ ] **Build it** — `PQ[T]` with a `less` function, a sequence-number tie-break, and handles for `Update` and `Remove`
- [ ] **Build it** — top-K, merge-k-sorted and the running median on top of it
- [ ] **Build it** — a Huffman code (tree, code table, encode, decode) and the Dijkstra frontier with lazy deletion
- [ ] **Prove it** — a model oracle (an unsorted slice and a linear scan for the minimum) over ≥100,000 random operations
- [ ] **Prove it** — the heap invariant, and the handle-to-index map, checked after *every* operation
- [ ] **Prove it** — a **narrow priority range** so ties, equal keys and repeated updates actually occur
- [ ] **Prove it** — exhaustive: every permutation of up to 8 distinct keys through `Init` yields a valid heap within the 2n comparison bound
- [ ] **Prove it** — Huffman against exhaustive merging on alphabets of up to 7 symbols, and top-K against the sorted prefix
- [ ] **Measure it** — comparisons and swaps, not just time: `Push`, `Pop` and heapify on random, ascending and descending input
- [ ] **Measure it** — `container/heap` against your generic heap: allocations per operation and ns per operation
- [ ] **Measure it** — `Fix` against lazy deletion: maximum heap size and stale pops
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| Parent and children of node `i`? | `⌊(i-1)/2⌋` · `2i+1`, `2i+2` |
| Where are the leaves of an `n`-element heap? | indices `⌊n/2⌋ … n-1` — half the nodes |
| Is a heap sorted? | no — only the minimum has a promised place |
| Is a sorted array a heap? | yes |
| Comparisons per level of a textbook sift-down? | two: smaller child, then against the sinking element |
| Why is heapify Θ(n)? | half the nodes are leaves; the work per node is its *height*, and heights sum to less than `n` |
| Upper bound on heapify comparisons? | `2n` |
| What does `heap.Pop` do before calling my `Pop`? | `Swap(0, n-1)`, then sift the new root down in the first `n-1` |
| What must my `Pop` do? | remove and return the **last** element |
| What happens if I call `h.Push(x)` directly? | the invariant is silently broken — only `heap.Push` sifts |
| Why do `Push` and `Pop` take pointer receivers? | they change the slice's length |
| Go min-heap vs max-heap? | `<` vs `>` in `Less` |
| Rust `BinaryHeap` min-heap? | wrap in `Reverse`, or reverse `Ord` |
| Are equal priorities popped first-in-first-out? | no — compare `(priority, sequence)` |
| Why does `BinaryHeap<f64>` not compile? | `f64` is not `Ord` (`NaN`) |
| What does `NaN` do to a Go heap under `<`? | nothing loud — it misorders |
| How is top-K held? | a size-k heap of the best so far; the root is the one to beat |
| Top-K time and space? | Θ(n log k) · Θ(k) |
| Running median structure? | max-heap (lower half) + min-heap (upper half), sizes within one |
| Merge k sorted lists: strength of the heap? | lazy — first `m` outputs cost Θ(k + m log k) |
| Best pairwise merge order for unequal lists? | always merge the two shortest (Huffman's) |
| Does either standard library have decrease-key? | **no** — Go has `Fix`/`Remove` by index; Rust has neither |
| The portable substitute? | lazy deletion: push a new entry, skip the stale one on pop |
| What does lazy deletion cost? | a pop per stale entry that surfaces; space for those that never do |
| Huffman: pops and pushes for `n` symbols? | `2(n-1)` · `n-1` |
| Huffman average length vs entropy? | `H ≤ L < H + 1` |
| What does Dijkstra's heap hold? | the frontier, keyed by tentative distance |
| Which `pop` does Rust use? | bottom-up: one comparison per level, then a short sift-up |
| Which sift does `into_sorted_vec` use? | the textbook two-comparison one |
| What does dropping a `PeekMut` do? | re-sifts the root, if you wrote through it |

## Numbers to find on your own machine

- [ ] The number of valid min-heaps of 10 distinct keys, out of 10!
- [ ] Comparisons per `Push` on random, ascending and descending input — which one is constant, which grows with `n`
- [ ] Comparisons per `Pop` while draining, against `2 lg n`
- [ ] Heapify against `n` pushes: comparisons per element on each input shape, and the worst case over all inputs of 10 keys
- [ ] How often a single `NaN` makes a drain come out unsorted
- [ ] Top-K replacements against `k ln(n/k)`, and the input shape that makes it Θ(n log k)
- [ ] Merge-k comparisons: heap, sequential, balanced and smallest-first, for equal and for skewed lists
- [ ] The heap's maximum size and stale pops under `Fix`, lazy deletion and lazy deletion with compaction
- [ ] Huffman average length against the entropy, and the depth Fibonacci weights force
- [ ] Dijkstra's stale pops and maximum heap size, lazy against `Fix`, on sparse and dense graphs
- [ ] Allocations per `heap.Push` for small ints, large ints, structs and pointers
- [ ] Comparisons per pop: textbook, bottom-up and 4-ary — and whether the order of the three holds for time too

> Lesson: [20-heaps-priority-queues.md](../../20-heaps-priority-queues.md) · Index: [README.md](README.md)
