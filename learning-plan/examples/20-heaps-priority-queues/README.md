# Step 20 — Heaps & Priority Queues · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

Examples marked **Output:** are deterministic — most of this lesson prints comparison, swap and pop
counts, which your run should match character for character. **Sample output:** ones print timings.

**Every example also has a Rust twin**, in the same tier file: Go → Go output → Rust → Rust output →
a short *Go vs Rust* note. The twins are std-only single files, built with `rustc --edition 2024 -O`, and
each one passed `rustfmt`, `clippy -D warnings` and a real run before being added; the Rust output
blocks are the binary's real stdout, captured — never typed. Where the two languages disagree, the note
says so and the Rust program prints what *its* numbers show, not Go's.

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste a Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the array, the two sifts, `container/heap`, heapify, ties and floats

- [1. The heap is an array](1-easy.md#1-the-heap-is-an-array)
- [2. Sift up and sift down, counted](1-easy.md#2-sift-up-and-sift-down-counted)
- [3. `container/heap`: five methods and a call order](1-easy.md#3-containerheap-five-methods-and-a-call-order)
- [4. Heapify is Θ(n), and n Pushes are not](1-easy.md#4-heapify-is-θn-and-n-pushes-are-not)
- [5. Min, max, generic, and ties](1-easy.md#5-min-max-generic-and-ties)
- [6. Floats, `NaN` and the `Less` contract](1-easy.md#6-floats-nan-and-the-less-contract)

### 🟡 [Medium](2-medium.md) — the problems heaps own: top-K, merging, the median, scheduling, cancellation

- [7. Top-K in Θ(n log k)](2-medium.md#7-top-k-in-θn-log-k)
- [8. Merging k sorted lists](2-medium.md#8-merging-k-sorted-lists)
- [9. Running median with two heaps](2-medium.md#9-running-median-with-two-heaps)
- [10. Scheduling: meeting rooms and cooldowns](2-medium.md#10-scheduling-meeting-rooms-and-cooldowns)
- [11. Cancel and reschedule: lazy deletion and what it leaks](2-medium.md#11-cancel-and-reschedule-lazy-deletion-and-what-it-leaks)

### 🔴 [Hard](3-hard.md) — Huffman, the Dijkstra frontier, what the stdlib costs, a cheaper pop, the capstone

- [12. Huffman coding, against the entropy bound](3-hard.md#12-huffman-coding-against-the-entropy-bound)
- [13. Dijkstra's frontier, and the order it settles](3-hard.md#13-dijkstras-frontier-and-the-order-it-settles)
- [14. What `container/heap` costs](3-hard.md#14-what-containerheap-costs)
- [15. Pop with fewer comparisons: bottom-up and d-ary](3-hard.md#15-pop-with-fewer-comparisons-bottom-up-and-d-ary)
- [16. Capstone: `PQ[T]` with handles, built, proved, measured](3-hard.md#16-capstone-pqt-with-handles-built-proved-measured)

## The thread through the lesson

1 lays the heap out as an array and counts how few arrays are heaps. 2 is the two sifts, counted, and 3
is `container/heap` — five methods and the order the package calls them in. 4 is heapify, and why it is
not just `n` pushes; 5 and 6 are what `Less` must promise: ties, and the float that breaks it. 7–10 are
the problems that ask only "what is the smallest?" — the best k, the next list head, the middle of a
stream, the next task. 11 is what a heap cannot do (change a priority) and the two ways around it. 12
and 13 put the heap inside Huffman's algorithm and Dijkstra's frontier, 14 prices `container/heap`
itself, 15 sifts with fewer comparisons, and 16 assembles all of it.

> Lesson: [20-heaps-priority-queues.md](../../20-heaps-priority-queues.md) · Previous: [19 — Balanced Trees & B-Trees](../19-balanced-trees/README.md)
