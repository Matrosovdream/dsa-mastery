# Step 04 — Go's Memory Model · Examples

A library of **16 runnable examples**, split into three files by difficulty. Each is a complete
`package main` program.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added — the output blocks are
real stdout.

- **Deterministic** (1, 2, 4, 10): `Sizeof`, aliasing and allocation sizes are the same on any 64-bit machine.
- **Sample output** (3, 5–9, 11–16): measured on an Apple M4, Go 1.26.3, with a **128-byte cache line**. On a typical x86 machine the line is 64 bytes and example 7's plateau arrives at stride 8. Match the shapes, not the digits.

Examples 14 and 16 take roughly 30 seconds each.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — what things cost and where they live

- [1. What a variable costs](1-easy.md#1-what-a-variable-costs)
- [2. Aliasing, and when append betrays you](1-easy.md#2-aliasing-and-when-append-betrays-you)
- [3. Ten bytes that pin eight megabytes](1-easy.md#3-ten-bytes-that-pin-eight-megabytes)
- [4. Struct padding, and the field order that fixes it](1-easy.md#4-struct-padding-and-the-field-order-that-fixes-it)
- [5. What escapes to the heap](1-easy.md#5-what-escapes-to-the-heap)
- [6. A million integers, eight ways](1-easy.md#6-a-million-integers-eight-ways)

### 🟡 [Medium](2-medium.md) — when layout starts costing time

- [7. Measuring your cache line from Go](2-medium.md#7-measuring-your-cache-line-from-go)
- [8. The same reads, in a different order](2-medium.md#8-the-same-reads-in-a-different-order)
- [9. The three taxes on `[]*T`](2-medium.md#9-the-three-taxes-on-t)
- [10. Size classes: what you asked for vs what you got](2-medium.md#10-size-classes-what-you-asked-for-vs-what-you-got)
- [11. Interface boxing vs generics](2-medium.md#11-interface-boxing-vs-generics)

### 🔴 [Hard](3-hard.md) — when layout beats asymptotics

- [12. The GC does not look at pointer-free memory](3-hard.md#12-the-gc-does-not-look-at-pointer-free-memory)
- [13. Array of structs vs struct of arrays](3-hard.md#13-array-of-structs-vs-struct-of-arrays)
- [14. The same tree, in two places in memory](3-hard.md#14-the-same-tree-in-two-places-in-memory)
- [15. Sorted slice vs map — the honest comparison](3-hard.md#15-sorted-slice-vs-map--the-honest-comparison)
- [16. One graph, three layouts, one BFS](3-hard.md#16-one-graph-three-layouts-one-bfs)

## The constants, in one table

Everything lesson 03 taught you to drop:

| Fact | Number | Example |
|------|--------|---------|
| Slice header | 24 bytes (string 16, `any` 16, map 8) | 1 |
| A 10-byte slice can pin | **8 MB** | 3 |
| Field order, `{bool,int64,bool}` | 24 → **16** bytes | 4 |
| Taking an address | **0 allocations** | 5 |
| Bytes/element, `[]int` → `map[int]struct{}` | 8 → **37.8** | 6 |
| Cache line, measured from Go | **128 bytes** | 7 |
| Sequential vs shuffled reads, 32 MB | **7.1×** | 8 |
| `[]T` vs `[]*T` | 2.4× time, **59× GC**, 5748× objects | 9 |
| Size-class rounding, 33 bytes | → **48** | 10 |
| `[]int` vs `[]any` | 3× memory, up to 2.5× time | 11 |
| Pointer link vs int32 index | **22× GC** | 12 |
| Same tree, compact vs scattered nodes | **1.9×** at n=2²² | 14 |
| Graph as map vs as CSR | 2.9× time, 4.1× memory, **1,004,118 → 4 objects** | 16 |

## The five results worth remembering

| # | Finding |
|---|---------|
| 5 | Taking an address is free; *outliving the frame* is what costs. And `-gcflags=-m` saying "does not escape" does **not** mean "does not allocate". |
| 7 | You can measure your CPU's cache-line size in pure Go — but only with a sink, or the compiler deletes the loads and every stride reads identical. |
| 12 | The GC skips pointer-free memory entirely. Swapping a `*T` link for an `int32` index made an identical structure **22× cheaper** to collect. |
| 14 | The *same tree* doing the *same comparisons* differs by 1.9× purely on where `malloc` put the nodes — and is identical below cache size. |
| 16 | Same Θ(V+E) BFS, three layouts: 2.9× time and a million-to-four difference in heap objects. |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
