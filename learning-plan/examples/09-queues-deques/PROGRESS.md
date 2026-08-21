# Step 09 — Queues & Deques · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. The naive slice queue, and what it actually costs**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. The naive slice queue, and what it actually costs
- [ ] 2. Fix one: a head index and scheduled compaction
- [ ] 3. Fix two: the ring buffer
- [ ] 4. Growing a ring, and the bug that hides until it doesn't
- [ ] 5. `Deque[T]` — opening the other end
- [ ] 6. BFS, and why the container *is* the algorithm

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. `container/list` and `container/ring` — what they are for
- [ ] 8. A buffered channel is a ring buffer with a mutex
- [ ] 9. Bounded queues: the four things that can give
- [ ] 10. The monotonic deque: sliding-window maximum in Θ(n)
- [ ] 11. Six queues, three workloads, three different winners

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. Level order and multi-source BFS
- [ ] 13. 0-1 BFS: a deque instead of a heap
- [ ] 14. A queue from two stacks, and the Θ(1) minimum that falls out
- [ ] 15. The work-stealing deque
- [ ] 16. Capstone: `Deque[T]`, built, proved, measured

## The three-pass build

In `practice/09-queues-deques/`:

- [ ] **Build it** — `RingBuffer[T]`: power-of-two capacity, stored `count`, `(T, bool)` on both pops
- [ ] **Build it** — `Deque[T]`: `PushFront`/`PushBack`/`PopFront`/`PopBack`/`At`/`Slice`, two-copy grow
- [ ] **Build it** — one explicit overflow policy, named in the type
- [ ] **Prove it** — table-driven, plus a model oracle against a plain `[]T` over ≥100,000 random ops
- [ ] **Prove it** — invariants after *every* operation: capacity is a power of two, `mask == cap-1`, `head` in range, `count` in range
- [ ] **Prove it** — a **rotation sweep**: drive `head` to every offset before asserting, then force a grow
- [ ] **Prove it** — a brute-force oracle for sliding-window maximum, on a **narrow value range** so ties actually occur
- [ ] **Measure it** — allocation budget: the steady state must be **0**
- [ ] **Measure it** — empty control, then push/pop sweep; report the **worst** single operation, not just the mean
- [ ] **Measure it** — memory still held after a burst and a full drain
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| Does `q = q[1:]` leak in steady state? | **no** — it leaks after a drain or a burst |
| The two fixes? | head index + compaction; a circular array |
| Why must a ring's capacity be a power of two? | `& mask` replaces `%`, and `(head-1)&mask` wraps — `-1 % 8` is `-1` in Go |
| Why store `count` rather than derive it? | `head == tail` means both empty and full |
| Why does growing a ring need **two** `copy`s? | a wrapped live region is two contiguous pieces |
| What does a ring test need that a stack test doesn't? | a **rotation sweep** — `head == 0` hides every real bug |
| Why zero the slot on `Pop`? | otherwise `T`'s pointers stay reachable through the array |
| Mark visited on enqueue or dequeue? | **enqueue** — on dequeue is correct but queues Θ(E) |
| What does a buffered channel add to a ring buffer? | a mutex, blocking, `close`, `range` — and no way to inspect it |
| The four overflow policies? | block · drop newest · drop oldest · error |
| Which one for a video frame? | **drop oldest** — the stale value is worthless |
| What does raising the capacity buy? | a longer absorbable **burst**, paid for in latency |
| Deque invariant for sliding-window max? | indices whose values strictly decrease |
| 0-1 BFS rule? | weight 0 → `PushFront`, weight 1 → `PushBack` |
| Work stealing: which end for whom? | owner takes the **back** (LIFO, warm), thief the **front** (oldest = biggest) |

## Numbers to find on your own machine

Do not take these on trust — the whole point is that several of them contradicted what I expected.

- [ ] Steady-state `q = q[1:]` holds **0.0 MB**; the same queue drained from 10,000,000 holds **76.3 MB**
- [ ] Compaction at 50% settles at **0.49** copies/operation; compacting every dequeue costs **500**
- [ ] Your cache line is 128 bytes (lesson 04) — check what ring capacity that implies for `[]int`
- [ ] A ring with capacity 2000 costs **2.39 ns/op** more than one with 2048
- [ ] The one-copy grow passes rotation 0 and **fails 6 of 8**
- [ ] `container/ring` costs the same heap objects as `container/list` (**~149,867**)
- [ ] Buffered channel is **7.3×** a bare ring, single-goroutine and uncontended
- [ ] Capacity 8 → 4096 raises mean queueing delay **126×**
- [ ] Brute-force sliding-window max **beats** the deque at k = 4
- [ ] The same naive queue measures **1.82** ns/op inline and **7.99** behind an interface

> Lesson: [09-queues-deques.md](../../09-queues-deques.md) · Index: [README.md](README.md)
