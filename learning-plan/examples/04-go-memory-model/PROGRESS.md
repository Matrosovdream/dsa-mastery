# Step 04 — Go's Memory Model · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. What a variable costs**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. What a variable costs
- [ ] 2. Aliasing, and when append betrays you
- [ ] 3. Ten bytes that pin eight megabytes
- [ ] 4. Struct padding, and the field order that fixes it
- [ ] 5. What escapes to the heap
- [ ] 6. A million integers, eight ways

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Measuring your cache line from Go
- [ ] 8. The same reads, in a different order
- [ ] 9. The three taxes on `[]*T`
- [ ] 10. Size classes: what you asked for vs what you got
- [ ] 11. Interface boxing vs generics

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. The GC does not look at pointer-free memory
- [ ] 13. Array of structs vs struct of arrays
- [ ] 14. The same tree, in two places in memory
- [ ] 15. Sorted slice vs map — the honest comparison
- [ ] 16. One graph, three layouts, one BFS

## Recall drill

Cover the right column and answer from memory.

| Question | Answer |
|---|---|
| Size of a slice header? | 24 bytes (ptr, len, cap) |
| Size of a `string`? An `any`? A `map` variable? | 16, 16, 8 |
| Does `append` to a subslice touch the parent? | Only if there is spare capacity. `xs[a:b:c]` prevents it. |
| Why does a 10-byte slice hold 8 MB alive? | It points into the original array; the memory is genuinely reachable |
| `{bool, int64, bool}` — size? Reordered? | 24 → 16 |
| Does `p := &point{}` allocate? | No, unless it outlives the frame |
| Two things that force boxing to allocate | a runtime value (not a constant), outside 0..255 |
| Ask the allocator for 33 bytes — what do you get? | 48 |
| How do you find your cache-line size in Go? | stride sweep; plateau at stride × 8 bytes |
| Sequential vs shuffled reads over 32 MB | ~7× |
| Three taxes on `[]*T` | memory, locality, GC tracing |
| How do you make a structure free for the GC? | no pointer fields — link by index |
| When does SoA beat AoS? | scanning one field across many records |
| Why choose a sorted slice over a map? | 4.5× less memory, ordered/range queries, disk-friendly |
| What does CSR give up? | cheap edge insertion — it is build-once, query-many |

## Numbers to find on your own machine

| What | Mine (M4, Go 1.26.3) | Yours |
|------|----------------------|-------|
| cache line (ex 7 plateau × 8 bytes) | **128 bytes** | |
| sequential vs shuffled, 32 MB (ex 8) | 7.1× | |
| `[]T` vs `[]*T`: time / GC (ex 9) | 2.4× / 59× | |
| pointer field vs int32 index, GC (ex 12) | 22× | |
| AoS vs SoA, one-field scan (ex 13) | 1.5× and 3.6× | |
| compact vs scattered tree at n=2²² (ex 14) | 1.9× | |
| map vs sorted slice memory (ex 15) | 4.5× | |
| CSR vs map BFS, and heap objects (ex 16) | 2.9×, 1,004,118 → 4 | |

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
