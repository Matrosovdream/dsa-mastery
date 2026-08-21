# 09 — Queues & Deques

> Part of **Part 2 — Linear Structures**. [08](08-stacks.md) was a relief: a stack only touches the
> **end** of a slice, which is free. A queue touches **both ends**, and the front is the expensive one.
> The one-line thesis: **the naive slice queue is not slow — it is the fastest queue in this lesson —
> its problem is that it never gives memory back, and fixing that is a choice between copying on your
> own schedule and never copying at all.**

## Goals
- State precisely when `q = q[1:]` leaks and when it does not — the folklore is two-thirds right.
- Implement the two fixes: a **head index with compaction**, and a **ring buffer**.
- Explain why a ring's capacity must be a power of two, in two independent ways.
- Write `grow` for a ring correctly, and write the test that catches the wrong version.
- Build a generic `Deque[T]` and know which four algorithms in this lesson need the second end.
- Place `container/list`, `container/ring` and buffered channels precisely.
- Choose an **overflow policy** deliberately, and know that refusing to choose is choosing OOM.
- Write BFS, level-order BFS, multi-source BFS and 0-1 BFS, and say what the queue guarantees in each.
- Use a **monotonic deque** for sliding-window extrema, and know where the crossover is.

## Concepts

- **`q = q[1:]` in steady state does not leak.** This is worth measuring rather than believing. With
  1000 elements live over 2,000,000 push/pop pairs, the backing array's capacity peaked at **1535**
  and the heap grew by **0.0 MB**. `append` reuses the space the reslice vacated, and the array is
  recycled continuously.

- **It leaks after a drain, and after a burst.** Push 10,000,000 and dequeue 9,999,999 and you hold a
  slice reporting `len 1, cap 1` that is still pinning **76.3 MB** — because the slice header points
  into the middle of an array nothing has shrunk. Absorb a burst of 5,000,000 and drain to 1000 and
  **38.5 MB** stays held. The naive queue's failure mode is *shape of workload*, not time.

- **Fix one — a head index.** Keep the elements where they are; remember where the live region starts.
  The dead prefix is now yours to reclaim on a schedule you choose. Compacting when half the buffer is
  dead costs **0.49 copies per operation**; compacting on every dequeue costs **500** and makes the
  queue quadratic. It is the same hysteresis dial as [lesson 06](06-arrays-slices.md)'s shrink policy.

- **Fix two — a ring buffer.** Stop moving the data. The live region wraps instead of marching, so it
  is never copied. Three fields — `buf`, `head`, `count` — and every operation is **Θ(1) worst case**,
  not amortized, with **zero allocations** after the constructor.

- **`count` is stored, not derived.** With only `head` and `tail`, `head == tail` means both *empty*
  and *full* and nothing distinguishes them. The three fixes are: keep a count (clearest), waste one
  slot, or keep monotonic counters and index with `% cap`. Go's runtime uses the third for channels,
  because two ever-increasing integers are easier to update atomically.

- **The capacity must be a power of two, for two separate reasons.** First, speed: `% 2000` is an
  integer division, and replacing it with `& 2047` recovers **2.39 ns/op**. Second, and more
  important, *correctness of `PushFront`*: `head = (head-1) & mask` wraps backwards past zero with no
  branch, whereas in Go `-1 % 8` is `-1`, not `7`. A ring written with `%` must add `cap` first.

- **A wrapped live region is at most TWO contiguous pieces.** That single fact is the whole of `grow`,
  `Slice`, and any `io.Reader`/`io.Writer` you write for a ring:

  ```go
  n := copy(next, buf[head:])   // the piece before the wrap
  copy(next[n:], buf[:head])    // the piece after it
  ```

  The one-copy version is **correct when `head == 0`** — which it is in every example you write by
  hand. Rotate the queue first and it fails **6 of 8** starting offsets.

- **`Pop` must write the zero value.** The array is bounded; what the array *points at* is not. A deque
  with the zeroing line removed held **20.5 MB** while reporting `Len() == 0`, with nothing in a heap
  profile except a `[]*payload` that is "obviously" empty.

- **A deque is a ring that runs backwards too**, and it subsumes both of the last two lessons:
  `PushBack`+`PopBack` is a stack, `PushBack`+`PopFront` is a queue. This is why Go ships neither type:
  a slice is a stack, and everything else is thirty lines you write once.

- **BFS and DFS are the same algorithm with a different container.** Change `PopFront` to `PopBack` and
  breadth-first becomes depth-first. The shortest-path guarantee belongs to the **queue**: vertices
  come out in non-decreasing distance order, so the first arrival is by a shortest path.

- **Mark visited on enqueue, never on dequeue.** Marking on dequeue is still *correct* — the distances
  match exactly — but it queues a vertex once per incoming edge, so the queue holds Θ(E) entries
  instead of Θ(V) (**5×** more here, and nothing ever tells you). It stays correct only because the
  distance rides in the queue entry; move it to an array and **95,797** vertices get the wrong answer.

- **A buffered channel is a ring buffer with a mutex.** Go's `hchan` holds a circular array, a send
  index, a receive index, a count and a lock. The question is never "channel or ring buffer?" but "do
  I need the lock?" — measured **2.35 / 5.73 / 17.13 ns/op** for a ring, a ring behind a mutex, and a
  channel, all single-goroutine and uncontended. What the channel sells is blocking, `close` and
  `range`; what it costs is that you cannot look inside it.

- **Bounded means something must give, and there are exactly four candidates:** block (backpressure),
  drop newest, drop oldest, return an error. Every one of them loses something — that is arithmetic,
  not a design flaw. The policy is a property of the **data**, not of the queue: drop-oldest is correct
  for a video frame and a bug for an audit log.

- **Capacity is a latency dial, not a throughput dial.** Throughput is set by the consumer alone. Going
  from capacity 8 to 4096 cut producer blocking from 19.5 ms to 2.5 ms and raised **mean queueing
  delay from 7.0 µs to 881.4 µs**. That is bufferbloat, and it is why you size a queue for the burst
  you must absorb and then stop.

- **The monotonic deque** holds indices whose values strictly decrease. Two things happen to an index
  and only two: it **expires** off the front (it left the window) or it is **evicted** off the back (a
  larger, newer value arrived, so it can never win again). Each index is pushed once and popped at most
  once — **exactly 2.00 operations per element**, independent of `k`.

- **0-1 BFS is a priority queue with two priorities.** Weight-0 edges go on the **front**, weight-1 on
  the **back**; the deque then holds at most two distinct distances and stays sorted. Θ(V+E) against
  Dijkstra's Θ(E log V), measured **6.2–9.9× faster** with identical pop counts.

- **Work stealing is why the two ends being *different* matters.** The owner takes its own **back**
  (LIFO — the task it just created is still in L1); thieves take another worker's **front** (the oldest
  task, which in a divide-and-conquer workload is the **biggest** remaining subtree). The deque is
  sorted by size for free, purely as a side effect of push order.

- **Speed was the least interesting axis in this lesson.** Three workloads produced three different
  winners: head-index on steady-state speed, the unbounded queues on burst-then-drain (the bounded ones
  silently discarded **1,998,976** pushes), and `container/list` — the slowest, most allocation-hungry
  implementation — is the only one that returns its memory on drain.

## Complexity Table

| Operation / structure | Cost | Note |
|---|---|---|
| `append` + `q = q[1:]` | Θ(1) amortized | fastest here; memory is the problem |
| Head index + compaction (50%) | Θ(1) amortized | **0.49** copies/op measured |
| Head index, compact every dequeue | **Θ(n) per op** | 500 copies/op — quadratic queue |
| Ring buffer, all operations | **Θ(1) worst case** | not amortized; 0 allocations |
| Growable ring, `Push` | Θ(1) amortized | ≤ **2** copies/enqueue (doubling) |
| `Deque` `PushFront`/`PopBack` | **Θ(1)** | vs Θ(n) for a slice front-insert |
| `Deque.At(i)` | **Θ(1)** | `buf[(head+i) & mask]` |
| `container/list` as a queue | Θ(1) | **8.6×** slower, 1 Element + 1 box per value |
| Buffered channel send/recv | Θ(1) | **7.3×** a bare ring, uncontended |
| BFS / level order / multi-source | **Θ(V + E)** | multi-source is Θ(V+E) for *any* number of sources |
| BFS marking on dequeue | Θ(V + E) time, **Θ(E) space** | correct, and 5× the queue traffic |
| One BFS per source, take the min | Θ(S · V) | **S×** the work, measured |
| Sliding-window max, monotonic deque | **Θ(n)**, Θ(k) space | 2.00 ops/element, flat in k |
| Sliding-window max, brute force | Θ(n·k) | **wins** at k = 4 |
| 0-1 BFS | **Θ(V + E)** | vs Dijkstra Θ(E log V) — 6–9× |
| Two-stack queue | **Θ(1) amortized**, Θ(n) worst | 0.50 tips/op; worst single pop moved 577 |
| Min-queue (two min-stacks) | **Θ(1)** for `Min` | **3992.7×** a scan at 65,536 live |
| Work-stealing deque | Θ(1) both ends | owner and thief touch opposite ends |

Measured (Apple M4, Go 1.26.3): steady-state `q[1:]` leaks **nothing**; drained, it holds **76.3 MB** ·
non-power-of-two capacity **+2.39 ns/op** · one-copy `grow` fails **6 of 8** rotations · missing slot
zeroing holds **20.5 MB** at `Len() == 0` · deque front-insert **1580×** a slice · `container/ring`
costs the same **149,867** objects as `container/list` · capacity 8→4096 raises mean queueing delay
**126×** · plain BFS on a 0-1 graph is wrong on **2,939 of 3,000** grids · the same naive queue is
**1.82 ns/op** inline and **7.99** behind an interface, with identical allocations.

## Exercises
1. Measure `q = q[1:]` in three regimes — steady state, drained-and-held, burst-then-drained. Report the heap in each, and explain why only two of them leak.
2. Add a head index and compaction. Sweep the threshold from "every dequeue" to "75% dead" and plot copies/operation against peak waste.
3. Write a ring buffer with `head`, `count` and a derived `tail`. Benchmark it with capacity 2000 and 2048 and account for the whole difference.
4. Write `grow` with one `copy`. Then write the **rotation sweep** that catches it, and only then fix it.
5. Extend the ring to `Deque[T]`. Show that `(head-1) & mask` wraps and `(head-1) % cap` does not. Delete the slot-zeroing from `PopFront` and measure what it was doing.
6. Write BFS with your deque, then change one word to get DFS. Then write the mark-on-dequeue version, prove the distances still match, and count the pushes.
7. Benchmark `container/list` on the queue workload and count heap objects. Then find the one thing `container/ring` does that nothing else does in Θ(1).
8. Measure a ring, a ring behind `sync.Mutex`, and a buffered channel, all single-goroutine. Then write the worker pool that a ring buffer cannot express.
9. Implement all four overflow policies behind one ring. Run a producer 4× faster than the consumer and account for **every** item. Then sweep the capacity and measure per-item **queueing delay**, not producer block time.
10. Write sliding-window maximum with a monotonic deque. Verify against brute force on a **narrow value range** so ties occur, then find the crossover in `k`.
11. Put six queue implementations behind one interface and run three workloads that disagree. Then measure the same implementation inline and behind the interface, and explain the gap.
12. Implement level-order BFS (snapshot `len(q)`) and multi-source BFS. Prove the multi-source result equals the per-source minimum, then measure both.
13. Implement 0-1 BFS and check it against a real Dijkstra. Write the plain-queue version two ways — settle-on-arrival and re-relax — and find out which one is actually the bug.
14. Build a queue from two stacks; count total tips and report both the average and the **worst single pop**. Then swap in two min-stacks and get `Min` in Θ(1).
15. Build a work-stealing pool. Measure the steal rate, then steal from the back instead and measure both the steal count and the wall time. Explain why they disagree.
16. Stretch — package `Deque[T]` and apply all three passes: a model oracle, invariants after every operation, a **rotation sweep**, an allocation budget of 0, the **worst** single operation, and the memory still held after a burst and a full drain.

## Best Practices & Pitfalls
- **Default to a ring buffer** when the queue outlives a function. Default to `q = q[1:]` when it does not.
- **Round the capacity up to a power of two** in the constructor. It is one line and it is not optional.
- **Store `count`.** `head == tail` is ambiguous.
- **Pitfall — growing a ring with one `copy`.** A wrapped region is two pieces. This is *the* ring-buffer bug.
- **Pitfall — testing a ring with `head == 0`.** Add a rotation loop to every ring test you write; it costs three lines and catches the only real bug.
- **Pitfall — not zeroing the vacated slot** when `T` contains a pointer. Bounded array, unbounded retention.
- **Pitfall — marking visited on dequeue.** Correct, and Θ(E) queue traffic. Mark on push.
- **Pitfall — moving BFS's tentative distance out of the queue entry** into an array. That one *is* wrong.
- **Pitfall — reading `len(q)` inside the level-order inner loop.** Snapshot it first.
- **Pitfall — using a channel as a data structure.** You get `len` and `cap` and nothing else, and `len` is stale the moment you read it.
- **Pitfall — an unbounded queue at a system boundary.** A queue that cannot refuse work has no backpressure, and a queue with no backpressure is a countdown.
- **Pick the overflow policy where the data is defined**, and make the type say which one it is.
- **Size the queue for the burst you must absorb, then stop.** If you keep raising it, the consumer is too slow.
- **Pitfall — benchmarking a monotonic deque at small `k`.** Brute force wins at k = 4. Know your crossover.
- **Quote no ns/op figure without the harness it was measured in.** The same code was 1.82 and 7.99.

## Checklist
- [ ] I can say exactly when `q = q[1:]` leaks and when it does not.
- [ ] I can implement both fixes and say what each one costs.
- [ ] I can give **two** reasons a ring's capacity must be a power of two.
- [ ] I write `grow` with two `copy`s by reflex, and I rotate before I assert.
- [ ] I zero the vacated slot without being reminded.
- [ ] I can turn BFS into DFS by changing one word, and say what that costs me.
- [ ] I mark visited on enqueue, and I can explain the one algorithm where you must not.
- [ ] I can place `container/list`, `container/ring` and a buffered channel in one sentence each.
- [ ] I can name the four overflow policies and pick one for a given kind of data.
- [ ] I know capacity buys latency, not throughput.
- [ ] I can write the monotonic deque from the invariant, not from memory.
- [ ] I can explain why 0-1 BFS needs a deque and Dijkstra needs a heap.
- [ ] I know which end the owner takes and which end the thief takes, and why.

## Resources
- `container/ring` — circular *lists*, not buffers: https://pkg.go.dev/container/ring
- `container/list` — the doubly linked list: https://pkg.go.dev/container/list
- Go's `hchan`, the ring buffer inside every channel: https://github.com/golang/go/blob/master/src/runtime/chan.go
- Go's scheduler run queues and `runqsteal`: https://github.com/golang/go/blob/master/src/runtime/proc.go
- Chase–Lev work-stealing deque (the lock-free version of example 15): https://en.wikipedia.org/wiki/Work_stealing
- Bufferbloat — example 9's finding, at internet scale: https://www.bufferbloat.net/
- Dial's algorithm, of which 0-1 BFS is the C = 1 case: https://en.wikipedia.org/wiki/Shortest_path_problem
- Go slice tricks (queue idioms): https://go.dev/wiki/SliceTricks
- CLRS ch. 10.1 — stacks and queues; ch. 22.2 — breadth-first search.
- Examples: [examples/09-queues-deques](examples/09-queues-deques/) (16).
- Note: [08](08-stacks.md) says Θ(1) `Min` "fails on a queue". That is true of the *direct* trick — pairing each element with the min-so-far breaks when you remove from the other end. Example 14 shows the composition that works: two **min-stacks**.
- Next: [10 — Hash Tables & Sets](10-hash-tables.md) — from Θ(n) lookup to Θ(1), and what that costs.
