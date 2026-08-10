# 04 — Go's Memory Model for DSA

> Part of **Part 1 — Foundations**, and the counterweight to [03](03-complexity-analysis.md). That
> lesson taught you to drop the constants; this one shows you what they were. Every crossover you
> measured in lessons 01 and 03 — the map that loses below n=16, the O(n²) sort that wins to n=32 —
> is explained here. The one-line thesis: **an algorithm's cost is not its operation count, it is its
> cache-line count and its pointer count.**

## Goals
- Read the **slice header** (3 words) and explain every "slices are weird" surprise from it.
- Predict when a subslice **aliases** and when `append` silently reallocates, and use `xs[a:b:c]`.
- Recognize the **reslice leak**: 10 bytes pinning 8 MB.
- Order struct fields to avoid **padding**, and know when padding is unavoidable.
- Say what **escape analysis** does and doesn't do — and why "takes a pointer" ≠ "heap".
- Explain the **cache line** and measure yours from Go.
- Quantify the three taxes on `[]*T` versus `[]T`: memory, locality, and **GC tracing**.
- Choose between **AoS and SoA**, and between a map, a sorted slice, and a plain index.
- Replace pointers with **indices** to make a structure invisible to the GC.

## Concepts

- **An array is a value; a slice is a view.** `[10]int` *is* 80 bytes of data — assigning it copies
  all of it. A slice is a 3-word header (pointer, len, cap) pointing at someone else's array, so
  passing one is 24 bytes regardless of length. `string` is 2 words, `any` is 2 words, `map` and
  `chan` are 1 word (a pointer to a runtime struct). Every slice surprise follows from that sentence.

- **Aliasing is the point, and `append` is where it bites.** A subslice shares the backing array, so
  writes go straight through. Whether `append` also writes through depends on **spare capacity**:
  ```go
  view := base[:2]     // len 2, cap 5 -> append OVERWRITES base[2]
  view := base[:2:2]   // len 2, cap 2 -> append reallocates, base untouched
  ```
  The three-index form is how you hand a view to code you don't control.

- **A slice pins its whole backing array.** `data[:10]` keeps all 8 MB alive because the memory is
  genuinely reachable — the GC is not at fault. Measured: a 10-byte slice held **8.0 MB**. The fix is
  `slices.Clone` whenever a small result must outlive a large input. This is the same bug as the
  `q = q[1:]` queue leak in [09](09-queues-deques.md).

- **Go never reorders struct fields; you do.** Every field sits at an offset divisible by its
  alignment, and the compiler inserts padding. `{bool, int64, bool}` is **24 bytes**; the same three
  fields as `{int64, bool, bool}` are **16** — a 33% saving from field order alone, 76 MB at 10M
  elements. Rule: largest alignment first. Some padding is unavoidable (13 bytes of payload cannot
  round to a multiple of 4), and the only escape from that is a different layout entirely.

- **Escape analysis asks "does this outlive the frame?", not "is there a pointer?"** Taking an address
  is free if the pointer stays local. What forces the heap: returning a pointer, storing into an
  interface, a size too large for a frame, or a `make` whose length the compiler cannot bound.
  Measured: `&point{…}` used locally = **0 allocations**; the same struct returned = **1**.
  `make([]int, 16)` = 0; `make([]int, size)` with `size` a variable = **1**.

- **Read `-gcflags=-m` carefully.** For `make([]int, size)` the compiler prints *"does not escape"* and
  the allocation still happens — "escapes" means "outlives the frame", and a variable-length `make`
  can fail to escape yet still be heap-allocated because the frame size isn't known. Trust
  `testing.AllocsPerRun`.

- **The allocator rounds up to size classes** (8, 16, 24, 32, 48, 64, 80, 96, …). Ask for 33 bytes,
  get 48. A 40-byte node is billed 48 — **76 MB of pure rounding** in a 10M-node tree. (Objects under
  16 bytes with no pointers go through the tiny allocator and dodge this.) This stacks on top of
  padding: a badly ordered struct pads to 40, then the allocator rounds that to 48.

- **The CPU loads cache lines, not values.** Summing every k-th element of a 32 MB array costs
  **0.27 ns per touch at stride 1** and plateaus at **~1.8 ns from stride 16** — and 16 × 8 bytes =
  **128 bytes**, this machine's line size, measured from pure Go. Touching 8 neighbours costs about
  the same as touching 1, so an algorithm's real currency is **lines touched**.

- **Access order is worth up to 7× on identical work.** The same n reads of the same array, in order
  versus shuffled: **1.0× at 8 KB** (all in L1), **1.6× at 2 MB**, **7.1× at 32 MB**. Prefetching is
  why sequential is nearly free; a random order defeats it completely. This single effect is why the
  slice keeps winning throughout the plan.

- **`[]*T` pays three taxes, none of them in the Big-O.** Measured over 1M points: **2.4× slower** to
  sum, **59× more GC time**, **5,748× more heap objects** (174 vs 1,000,206), and 2× the bytes. Use
  `[]T` unless elements are huge, shared and mutated, or polymorphic.

- **The GC skips pointer-free memory entirely.** It only scans blocks whose type *might* contain
  pointers. Two slices of the same size and element count, one linking by `*T` and one by `int32`
  index: **22× difference in GC cycle time**, and the index version is indistinguishable from a plain
  `[]int`. Replacing pointers with indices is the single biggest lever for large structures in Go.

- **Containers take type parameters, never `any`.** An interface value is 2 words plus, usually, a
  heap-allocated box: `[]any` of ints costs **24 bytes/element vs 8** (3.0×) and sums **1.2× slower
  when contiguous, 2.5× when scattered**. Generics compile per concrete type — `sumGeneric[int]`
  measured **1.0×** against the hand-written loop. (This is why `container/list` and `container/heap`,
  which predate generics, lose to a hand-rolled generic equivalent.)

- **Layout beats asymptotics more often than you'd think.** A balanced BST and `slices.BinarySearch`
  are both Θ(log n) — but the *same tree* with nodes allocated compactly versus scattered differs by
  **1.6× at n=2²⁰ and 1.9× at n=2²²**, growing with n. A freshly built tree is the compact case, which
  is what naive benchmarks measure and why they flatter trees.

- **Know the three lookup structures.** A map's O(1) is genuinely flat (~4–5 ns at every size) and
  beats binary search on speed everywhere — but costs **~4.5× the memory**, gives no ordered
  iteration, no range queries, and cannot be written to disk as-is. And if your keys are dense small
  integers, neither is right: index a slice directly.

## Complexity Table

The constants Big-O drops, measured on this machine (Apple M4, Go 1.26.3):

| Fact | Number |
|------|--------|
| Slice header | 24 bytes · string 16 · `any` 16 · map/chan/func 8 |
| Cache line | **128 bytes** (64 on most x86) |
| Sequential vs random access, 32 MB | **7.1×** |
| Cache hit vs miss, per touch | 0.27 ns vs ~1.8 ns |
| `[]T` vs `[]*T` scan | **2.4×** |
| `[]T` vs `[]*T` GC cycle | **59×** |
| Pointer field vs int32 index, GC cycle | **22×** |
| `[]int` vs `[]any` memory | 8 vs 24 bytes/element |
| `[]int` vs `map[int]struct{}` memory | 8 vs **37.8** bytes/element |
| Struct field order, `{bool,int64,bool}` | 24 → **16** bytes |
| Size-class rounding, 33 bytes | → **48** bytes |
| Node allocation | 1 heap object **per node**, always |

Bytes per element for a million ints: `[]int32` 4 · `[]int` 8 · `[]*int` 16 · linked list 16 ·
`map[int]struct{}` 37.8.

## Exercises
1. Print `unsafe.Sizeof` for an array, a slice, a string, a map, a chan, a func and an `any`. Explain why passing a slice is O(1) and passing an array is O(n).
2. Demonstrate all four aliasing cases: array copy, slice assignment, subslice write-through, and `append` with and without spare capacity. Then fix the last one with `xs[a:b:c]`.
3. Write a function returning `data[:10]` from an 8 MB buffer. Measure the heap with and without `slices.Clone`, and explain why the GC cannot help.
4. Take `{bool, int64, bool}` and reorder it. Report `Sizeof`, `Alignof` and every `Offsetof`. Then find a struct where reordering cannot remove the padding, and say why.
5. Write seven functions — local value, local pointer, returned pointer, boxed constant, boxed runtime value, oversized `make`, variable-length `make` — and measure allocations for each. Then read `go build -gcflags='-m'` and find one place where its wording disagrees with the measurement.
6. Measure bytes-per-element for a million ints held seven ways. Explain the `[]*int` number from `Sizeof` alone, and check whether `map[K]struct{}` actually beats `map[K]int` for your key type.
7. Sum every k-th element of a 32 MB array for k = 1…128 and plot ns-per-touch. Find the plateau and derive your machine's cache-line size. **Add a sink first** — without it the loads are deleted and every stride reports the same number.
8. Read the same n elements in order and shuffled, at 8 KB, 128 KB, 2 MB and 32 MB. Report the penalty at each size and explain why it grows.
9. Sum 1M points as `[]point` and as shuffled `[]*point`. Report time, GC cycle time, and live heap objects. Explain each of the three taxes.
10. Measure `AllocedBytesPerOp` for `make([]byte, k)` at k = 1…129 and reconstruct Go's size-class table. Compute the rounding waste for a 40-byte tree node at 10M nodes.
11. Compare `[]int`, `[]any` and a generic `sumGeneric[T]`. Report memory and speed, and measure `[]any` both contiguous and shuffled. Explain the gap between the two.
12. Build the same structure with a `*T` link field and with an `int32` index field. Measure GC cycle time for both and explain why one is free.
13. Store a 64-byte particle as AoS and as SoA. Time a one-field scan of each. Explain why the measured speedup is below the memory-traffic ratio.
14. Build one balanced BST twice — nodes allocated compactly, and the identical tree with nodes scattered. Find the n at which they diverge, and explain why they're identical below it.
15. Compare `slices.BinarySearch` against a map at six sizes, measuring both time and memory. List three things the sorted slice gives you that the map cannot.
16. Stretch — build a 1M-vertex graph as `map[int][]int`, `[][]int`, and **CSR** (two flat arrays). Run the same BFS on all three and report time, heap objects and memory. Then say what CSR gives up.

## Best Practices & Pitfalls
- **Default to `[]T`.** Reach for `[]*T` only when elements are large and copied often, shared and mutated through several owners, or genuinely polymorphic.
- **Prefer indices to pointers in big structures.** `int32` indices are half the size, contiguous, and invisible to the GC. You give up per-node freeing and type safety on the link.
- **Preallocate with `make([]T, 0, n)`** whenever n is known — it removes every resize *and* the latency tail ([03](03-complexity-analysis.md)).
- **Order struct fields largest-alignment-first**, and check with `unsafe.Sizeof` or the `fieldalignment` analyzer.
- **Pitfall — the retained backing array.** Returning a small subslice of a large buffer pins the whole thing. `slices.Clone` at the boundary.
- **Pitfall — `append` aliasing.** A subslice with spare capacity will overwrite its parent. Use the three-index form when handing out views.
- **Pitfall — benchmarking memory without a sink.** Without one the compiler deletes the loads and every access pattern measures identically. I hit this writing examples 7 and 8: a 32 MB random walk "measured" 0.23 ns per read, which is beyond memory bandwidth.
- **Pitfall — measuring a freshly built pointer structure.** It has the best layout it will ever have. Shuffle it, or accept that your benchmark is the optimistic case.
- **Pitfall — trusting `-gcflags=-m` over a measurement.** "Does not escape" does not mean "does not allocate".
- **Pitfall — assuming `map[K]struct{}` is a cheap set.** At `int` keys it measured identical to `map[K]int`; the table's own overhead dominates. Measure your key/value pair.
- **Never build a container on `any`.** Use type parameters. Boxing costs memory, locality, a type assertion, and GC work all at once.
- **If your keys are dense integers, index a slice.** It beats both the map and the sorted slice on every axis.

## Checklist
- [ ] I can state the size of a slice header, a string, an interface and a map variable, and explain what each word holds.
- [ ] I can predict whether `append` to a subslice will overwrite its parent, and force it not to.
- [ ] I can explain the reslice leak and name three places it occurs in real code.
- [ ] I can reorder a struct to remove padding, and recognize when padding is unavoidable.
- [ ] I can name four things that force a heap allocation, and explain why taking an address is not one of them.
- [ ] I can measure my machine's cache line size from Go, and explain the sink that makes the measurement valid.
- [ ] I can quantify the three taxes on `[]*T` and choose correctly between `[]T` and `[]*T`.
- [ ] I can explain why converting pointer links to index links makes a structure free for the GC.
- [ ] I can choose between AoS and SoA given an access pattern, and say what SoA costs in ergonomics.
- [ ] I can justify a sorted slice over a map on grounds other than lookup speed.

## Resources
- Go Slices: usage and internals — the header, aliasing, growth: https://go.dev/blog/slices-intro
- Arrays, slices (and strings): the mechanics of `append` and `copy`: https://go.dev/blog/slices
- `unsafe` — `Sizeof`, `Alignof`, `Offsetof`: https://pkg.go.dev/unsafe
- `runtime.MemStats` — `HeapAlloc`, `HeapObjects`, `StackInuse`: https://pkg.go.dev/runtime#MemStats
- The Go GC guide — pointer maps, why pointer-free memory is skipped: https://go.dev/doc/gc-guide
- `fieldalignment` analyzer: https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/fieldalignment
- Compressed Sparse Row, the layout behind example 16: https://en.wikipedia.org/wiki/Sparse_matrix#Compressed_sparse_row_(CSR,_CRS_or_Yale_format)
- Examples: [examples/04-go-memory-model](examples/04-go-memory-model/) (16).
- Next: [05 — Recursion & the Call Stack](05-recursion.md) closes Part 1, then Part 2 starts building structures with everything measured here.
