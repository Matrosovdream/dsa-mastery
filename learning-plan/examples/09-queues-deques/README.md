# Step 09 — Queues & Deques · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs. Examples 6, 10 and 13 additionally need the
`Deque[T]` type from example 5, dropped in the same folder as `deque.go`.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
go run -race .                            # example 15 is concurrent
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- **Deterministic** (2 partly, 4 partly, 6 partly, 12): your output should match character for character.
- **Sample output** (the rest): timings and memory, from an Apple M4 with Go 1.26.3.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the expensive end of a slice, and the two fixes

- [1. The naive slice queue, and what it actually costs](1-easy.md#1-the-naive-slice-queue-and-what-it-actually-costs)
- [2. Fix one: a head index and scheduled compaction](1-easy.md#2-fix-one-a-head-index-and-scheduled-compaction)
- [3. Fix two: the ring buffer](1-easy.md#3-fix-two-the-ring-buffer)
- [4. Growing a ring, and the bug that hides until it doesn't](1-easy.md#4-growing-a-ring-and-the-bug-that-hides-until-it-doesnt)
- [5. `Deque[T]` — opening the other end](1-easy.md#5-dequet--opening-the-other-end)
- [6. BFS, and why the container *is* the algorithm](1-easy.md#6-bfs-and-why-the-container-is-the-algorithm)

### 🟡 [Medium](2-medium.md) — the standard library, channels, backpressure

- [7. `container/list` and `container/ring` — what they are for](2-medium.md#7-containerlist-and-containerring--what-they-are-for)
- [8. A buffered channel is a ring buffer with a mutex](2-medium.md#8-a-buffered-channel-is-a-ring-buffer-with-a-mutex)
- [9. Bounded queues: the four things that can give](2-medium.md#9-bounded-queues-the-four-things-that-can-give)
- [10. The monotonic deque: sliding-window maximum in Θ(n)](2-medium.md#10-the-monotonic-deque-sliding-window-maximum-in-θn)
- [11. Six queues, three workloads, three different winners](2-medium.md#11-six-queues-three-workloads-three-different-winners)

### 🔴 [Hard](3-hard.md) — BFS variations, 0-1 BFS, work stealing, capstone

- [12. Level order and multi-source BFS](3-hard.md#12-level-order-and-multi-source-bfs)
- [13. 0-1 BFS: a deque instead of a heap](3-hard.md#13-0-1-bfs-a-deque-instead-of-a-heap)
- [14. A queue from two stacks, and the Θ(1) minimum that falls out](3-hard.md#14-a-queue-from-two-stacks-and-the-θ1-minimum-that-falls-out)
- [15. The work-stealing deque](3-hard.md#15-the-work-stealing-deque)
- [16. Capstone: `Deque[T]`, built, proved, measured](3-hard.md#16-capstone-dequet-built-proved-measured)

## What this lesson measured

Every number below came out of a program in this folder, and most of them contradicted the sentence
I had written before running it.

| Finding | Number |
|---|---|
| `q = q[1:]` in **steady state** | **no leak** — 1000 live, max cap 1535, 0.0 MB |
| `q = q[1:]` **drained and held** | **76.3 MB** held by a slice of `len 1, cap 1` |
| Head index + compaction at 50% | **0.49** copies/operation; compacting eagerly costs **500** |
| Ring buffer, steady state | **0 allocations**, Θ(1) **worst case** |
| Non-power-of-two capacity | **+2.39 ns/op** in integer division alone |
| Growing a ring with one `copy` | correct at `head == 0`, **fails 6 of 8** rotations |
| `Deque.Pop` without zeroing the slot | **20.5 MB** held at `Len() == 0` |
| Front insertion, deque vs `append([]T{v}, s...)` | **1580×** at n = 20,000 |
| BFS marking on dequeue | still correct, queues **5×** more |
| …with the distance moved out of the queue entry | **95,797** wrong distances |
| `container/list` vs a ring | **8.6×** slower, **149,869** heap objects to **1** |
| `container/ring` heap objects | **149,867** — same as `container/list` |
| Ring vs ring+mutex vs channel | **2.35** / **5.73** / **17.13** ns/op |
| Capacity 8 → 4096 under backpressure | mean queueing delay **7.0 µs → 881.4 µs** |
| Sliding-window max, deque vs brute | **flat in k**; brute **wins** at k = 4 (deque 0.52×) |
| Naive queue, inline vs behind an interface | **1.82** vs **7.99** ns/op, *same* allocations |
| 0-1 BFS vs Dijkstra | **6.2–9.9×**, identical pop counts |
| Plain BFS on a 0-1 graph | wrong on **2,939 of 3,000** random grids |
| Min-queue vs scanning | **3992.7×** at 65,536 live; scan wins under ~16 |
| Work stealing from the wrong end | **30–206×** more steals, a few percent on the clock |
| `Deque` worst single `PushBack` | **524,288** elements copied |

## The thread through the lesson

1–2–3 are one argument: the naive queue's problem is **memory, not speed**, and there are exactly two
fixes. 4–5 finish the structure. 6 shows that the container *is* the algorithm. 7–8 place the
standard library. 9 is the design decision the whole lesson builds to. 10, 13, 14 and 15 are what the
**second end** buys you. 16 assembles it.

> Lesson: [09-queues-deques.md](../../09-queues-deques.md) · Previous: [08 — Stacks](../08-stacks/README.md)
