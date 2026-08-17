# 08 — Stacks

> Part of **Part 2 — Linear Structures**. After [06](06-arrays-slices.md)'s trade-off and
> [07](07-linked-lists.md)'s honest verdict, the stack is a relief: it only ever touches the **end** of
> the sequence, which is the cheap end of a slice. The one-line thesis: **in Go a slice already is a
> stack, so the lesson is not the structure — it's recognising the problems that are secretly stacks,
> and the monotonic-stack technique that turns a family of Θ(n²) problems into Θ(n).**

## Goals
- Use a slice as a stack, and know why LIFO falls out of `append`/reslice for free.
- Write a generic `Stack[T]` whose zero value works, with the `(T, bool)` contract.
- Recognise "the most recently opened X" as the giveaway for a stack.
- Implement the classics: brackets, RPN, undo/redo, iterative DFS, shunting yard.
- Get Θ(1) `Min` on a stack, and know why the same trick fails on a queue.
- Prove the two-stack queue's amortized bound with the aggregate method.
- Write the **monotonic stack** and state why it's Θ(n).
- Apply it to next-greater, daily temperatures, largest rectangle — and know when *not* to.

## Concepts

- **A slice already is a stack.** `append` / `xs[len-1]` / `xs[:len-1]` — every operation is at the
  tail, nothing shifts, nothing is chased. A stack has none of lesson 07's problems because it never
  asks for anything a slice is bad at.

- **The one thing to get right is the empty case.** `xs[len(xs)-1]` on an empty slice **panics** — it's
  not a nil you can test. So `Pop() (T, bool)`, the same shape as a map lookup or a channel receive.

- **The generic wrapper is cheap, not free.** Measured: ~**1.6 ns per operation** over a bare slice,
  because the generic methods don't inline and `Pop` writes a zero to the vacated slot. Irrelevant
  unless the stack operation is your innermost loop — and worth it everywhere else for the contract.

- **Zero the popped slot** when `T` holds a pointer, or the removed element stays reachable through
  the backing array (→ [06](06-arrays-slices.md)).

- **The giveaway phrase.** "The most recently opened/seen/pushed X" → **stack**. "The oldest
  outstanding X" → queue. "The smallest outstanding X" → heap. That one line resolves most "which
  structure?" questions on sight.

- **Bracket matching needs a stack, not a counter.** `"([)]"` is perfectly balanced by *count* and
  still wrong — only a stack knows which bracket is currently innermost. And the stack's high-water
  mark is exactly the recursion depth a recursive parser would use, because an explicit stack and the
  call stack are the same structure.

- **RPN needs no parentheses and no precedence**, because the token order already encodes the tree.
  Evaluation is one pass with a stack. The trap: **the second pop is the left operand** — get it
  backwards and `+`/`*` still look fine while `-`/`/` silently flip sign.

- **Undo/redo is two stacks and one rule**: a new action must **clear the redo stack**. Skip it and
  you can redo your way into a document state that never existed. Note both stacks store *commands*,
  not snapshots — Θ(edit) instead of Θ(document).

- **DFS and BFS are the same code with a different container.** `xs[len-1]` → depth-first;
  `xs[0]` → breadth-first. One line. Two details make the iterative DFS match the recursive one: push
  neighbours **in reverse**, and mark `seen` on **pop**, not push (BFS is the opposite).

- **Shunting yard puts the grammar in data.** Precedence is a number you compare; associativity is
  whether the test is strict. `10-2-3 → 5` needs the non-strict test; `2^3^2 → 512` needs the strict
  one. Recursive descent puts the same grammar in the call graph — same trade as the two DFS versions.

- **Θ(1) min on a stack works because the contents are strictly nested.** What's below an element
  cannot change while that element is present, so an answer computed at push time stays valid exactly
  as long as the element does. Measured: scanning is **89× slower**; the auxiliary-stack version holds
  just **15 entries for 100,000 random pushes** — but **100,000** on descending input. Watch the
  `<=` versus `<`: with `<`, duplicates break it.

- **The two-stack queue is amortized Θ(1) by the aggregate argument**: each element transfers from
  `in` to `out` exactly once, so n transfers cover 2n operations. And one unlucky dequeue moves **n**
  elements — amortized Θ(1) and worst-case Θ(n), simultaneously true. The mirror (a stack from two
  queues) has no such trick and is Θ(n) per push, forever.

- **The monotonic stack is the technique this lesson exists for.** Keep the stack sorted; before
  pushing, pop everything that violates the order — and *each pop is an answer*. It's Θ(n) because
  each index is pushed once and popped at most once.

- **Benchmark it on adversarial input or you'll prove nothing.** On *random* data the brute force is
  nearly linear too (ratio a flat ~3×), because the next greater element is usually a step away. On
  *descending* data: **20,476,800 comparisons vs 6,399**, and **2509×** on the clock.

- **Largest rectangle is the payoff.** Every maximal rectangle is capped by one bar, so the question
  becomes nearest-smaller-on-each-side — one increasing stack. A virtual zero bar at the end removes
  the drain-the-stack special case. Measured **616×** faster than brute force at n=3200. The 2-D
  version is the same function called once per row.

- **Know when *not* to reach for it.** Trapping rain water has three Θ(n) solutions, and the
  **two-pointer** one wins: Θ(1) space, no allocation, **6.4× faster** than the array version and
  faster than the stack. The giveaway: if the answer depends on a specific nearby **element**, use a
  stack; if it depends only on a running **aggregate**, two pointers will be simpler.

## Complexity Table

| Operation / problem | Cost | Note |
|---|---|---|
| Push / Pop / Peek | **Θ(1)** amortized | on a slice |
| `Min()` on a stack | **Θ(1)** | pair each element with the min-so-far |
| Two-stack queue enqueue/dequeue | **Θ(1) amortized**, Θ(n) worst | n transfers per n elements |
| Stack from two queues | **Θ(n) per push** | no amortization saves it |
| Balanced brackets | Θ(n) time, Θ(depth) space | |
| RPN evaluation | Θ(n) time, Θ(depth) space | |
| Infix → postfix | Θ(n) time, Θ(depth) space | |
| Next greater / smaller | **Θ(n)** | vs Θ(n²) brute force |
| Daily temperatures / stock span | **Θ(n)** | same shape |
| Largest rectangle in histogram | **Θ(n)** | vs Θ(n²) |
| Maximal rectangle (2-D) | Θ(rows × cols) | one histogram per row |
| Trapping rain water | Θ(n) time, **Θ(1) space** | two pointers beat the stack here |

Measured (Apple M4, Go 1.26.3): generic wrapper **+1.6 ns/op** · scan-for-min **89×** slower ·
list-based stack **3.2×** slower, `container/list` **5.9×** · monotonic vs brute on adversarial input
**2509×** · largest rectangle **616×** · two-pointer rain water **6.4×** faster than the array version.

## Exercises
1. Use a bare `[]int` as a stack. Then write a pop that reports emptiness instead of panicking.
2. Write `Stack[T]` with a zero value that works and a `(T, bool)` pop. Prove that `Pop` releases the vacated slot, and measure the wrapper against a bare slice.
3. Write a bracket matcher that reports *where* and *why* it failed. Find an input a counter accepts and a stack rejects.
4. Evaluate RPN with a stack. Include a test that would catch a swapped operand order — `+` and `*` alone will not.
5. Build undo/redo with two stacks. Demonstrate what goes wrong if a new action doesn't clear the redo stack.
6. Write DFS recursively and with an explicit stack, then change one line to make it BFS. Explain the reverse-push and mark-on-pop details.
7. Implement shunting yard. Show that `10-2-3` and `2^3^2` both come out right, and say which comparison controls each.
8. Implement a min-stack three ways. Find the duplicate-value input that breaks the strict `<` version.
9. Build a queue from two stacks, count total transfers, and show it's exactly n for 2n operations. Then build a stack from two queues and explain why it has no such bound.
10. Convert a post-order traversal to an explicit stack, and say what the `stage` field corresponds to in the CPU.
11. Benchmark a slice stack against a linked stack, a pooled linked stack, and `container/list`.
12. Write next-greater-element with a monotonic stack. Then benchmark it against brute force on **random** and on **descending** input, and explain why only one of them shows the difference.
13. Implement daily temperatures and stock span, and verify both against brute force with a **small value range** so ties actually occur.
14. Implement largest-rectangle-in-histogram. Explain the virtual zero bar and the `i - left - 1` width. Then extend it to the 2-D maximal rectangle.
15. Solve trapping rain water four ways and compare time *and* space. Say which you'd ship.
16. Stretch — package `Stack[T]` plus the monotonic algorithms, and apply all three passes: table, invariants, a model oracle, brute-force oracles for each algorithm, an allocation budget, and an assertion that total pops ≤ n.

## Best Practices & Pitfalls
- **Use a slice.** There is no workload in this lesson where a linked stack wins.
- **Preallocate** with `Reserve`/`make(…, 0, n)` when the depth is known.
- **Pitfall — popping empty.** `xs[len(xs)-1]` panics. Return `(T, bool)`.
- **Pitfall — the second pop is the left operand** in RPN. Test with `-` and `/`, never just `+` and `*`.
- **Pitfall — forgetting to clear redo** on a new action.
- **Pitfall — marking `seen` on push in a DFS** (or on pop in a BFS). Each traversal has its own rule.
- **Pitfall — `<` instead of `<=`** in a min-stack. Duplicates break it, and only a duplicate-heavy test finds it.
- **Pitfall — benchmarking a monotonic stack on random data.** Brute force is nearly linear there. Use adversarial input.
- **Use a virtual sentinel element** (the zero bar) to drain a monotonic stack instead of a second loop.
- **Store indices, not values**, when the answer is a distance or a width.
- **Verify index-arithmetic algorithms against brute force** on small inputs with a narrow value range — that's where ties live.
- **Don't reach for a monotonic stack reflexively.** If the answer depends on a running aggregate rather than a specific neighbour, two pointers are simpler and faster.

## Checklist
- [ ] I can use a slice as a stack and explain why the tail is the cheap end.
- [ ] I write `Pop() (T, bool)` by reflex and zero the vacated slot for pointer types.
- [ ] I recognise "most recently …" as the giveaway for a stack.
- [ ] I can write bracket matching, RPN and shunting yard from scratch.
- [ ] I can turn a DFS into a BFS by changing one line, and explain the two accompanying rules.
- [ ] I can implement Θ(1) `Min` and explain why the same trick fails on a queue.
- [ ] I can state the aggregate argument for the two-stack queue and for the monotonic stack.
- [ ] I can write next-greater-element and derive daily-temperatures from it.
- [ ] I can solve largest-rectangle-in-histogram and explain the width formula.
- [ ] I know when a two-pointer scan beats a monotonic stack.

## Resources
- Shunting-yard algorithm: https://en.wikipedia.org/wiki/Shunting_yard_algorithm
- Reverse Polish notation: https://en.wikipedia.org/wiki/Reverse_Polish_notation
- Monotonic stack, with the standard problem set: https://en.wikipedia.org/wiki/Monotonic_stack
- `container/list` — if you insist on a linked stack: https://pkg.go.dev/container/list
- Go slice tricks (stack idioms): https://go.dev/wiki/SliceTricks#stack
- CLRS ch. 10.1 — stacks and queues.
- Examples: [examples/08-stacks](examples/08-stacks/) (16).
- Next: [09 — Queues & Deques](09-queues-deques.md) — the other end of the slice, where it stops being free.
