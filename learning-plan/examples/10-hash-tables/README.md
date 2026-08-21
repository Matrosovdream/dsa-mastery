# Step 10 — Hash Tables & Sets · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- **Deterministic** (parts of 2, 3, 4, 8, 9, 12): your output should match character for character.
- **Sample output** (the rest): timings and memory, from an Apple M4 with Go 1.26.3.

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — the mechanism, from the hash function to Go's implementation

- [1. Θ(n) → Θ(1), and the three things you pay for it](1-easy.md#1-θn--θ1-and-the-three-things-you-pay-for-it)
- [2. What makes a hash function good, and how to measure it](1-easy.md#2-what-makes-a-hash-function-good-and-how-to-measure-it)
- [3. Separate chaining, and what it costs](1-easy.md#3-separate-chaining-and-what-it-costs)
- [4. Open addressing, and the one hard problem](1-easy.md#4-open-addressing-and-the-one-hard-problem)
- [5. Load factor and rehashing](1-easy.md#5-load-factor-and-rehashing)
- [6. Go's map is a Swiss table](1-easy.md#6-gos-map-is-a-swiss-table)

### 🟡 [Medium](2-medium.md) — sets, idioms, keys, tuning, and a head-to-head

- [7. Sets, and why `map[T]struct{}` — for the right reason](2-medium.md#7-sets-and-why-maptstruct--for-the-right-reason)
- [8. Frequency counters and the idioms that surround them](2-medium.md#8-frequency-counters-and-the-idioms-that-surround-them)
- [9. What can be a map key](2-medium.md#9-what-can-be-a-map-key)
- [10. What preallocation actually buys](2-medium.md#10-what-preallocation-actually-buys)
- [11. Four implementations, five workloads, and a column I forgot](2-medium.md#11-four-implementations-five-workloads-and-a-column-i-forgot)

### 🔴 [Hard](3-hard.md) — the problems maps own, the attack, the limits, the capstone

- [12. The rearrangement: two-sum and group anagrams](3-hard.md#12-the-rearrangement-two-sum-and-group-anagrams)
- [13. Longest consecutive sequence, and the prefix-sum map](3-hard.md#13-longest-consecutive-sequence-and-the-prefix-sum-map)
- [14. hashDoS: the attack that made everyone randomise](3-hard.md#14-hashdos-the-attack-that-made-everyone-randomise)
- [15. Seven times a map is the wrong answer](3-hard.md#15-seven-times-a-map-is-the-wrong-answer)
- [16. Capstone: `HashMap[K,V]`, built, proved, measured](3-hard.md#16-capstone-hashmapkv-built-proved-measured)

## What this lesson measured

Every number below came out of a program in this folder, and a striking number of them contradicted
the sentence I had written before running it.

| Finding | Number |
|---|---|
| Map lookup vs linear scan at n=1,000,000 | **4.9 ns vs 190 µs** |
| …and at n=8 | the **scan wins** (0.6×) |
| Map memory vs raw keys | **2.4–3.7×**, swinging with the power-of-two boundary |
| `make(map, n)` effect on final memory | **none** — identical every run |
| FNV-1a distribution / avalanche | **77** (good) / **25.4** (mediocre) |
| FNV-1a top 6 bits vs low 6 bits | **47× worse** |
| …with a 3-multiply finalizer | **54 / 32.0**, top/low **1×** |
| Empty buckets at load factor 1.0 | **37%** (= e⁻¹) |
| Chained lookup with every key colliding | **1000.5 links**, **139×** slower |
| Linear probing, probes/miss at load 0.25 → 0.95 | **1.38 → 125.00** |
| Tombstones: probes/miss with live keys constant | **4.56 → 14.28** |
| Amortized moves per insert while growing | **1.97** |
| Worst single `Put` during a 200,000-insert build | **196,608 entries moved** |
| Go map miss/hit ratio | **not a constant**: 2.58× / 1.29× / 0.83× |
| `map[int]struct{}` vs `map[int]bool` | **identical** — 36.1 bytes/key both |
| `unsafe.Sizeof(struct{int; struct{}})` | **16** — same as `{int, int}` |
| Intersecting the larger set instead of the smaller | **79,796×** slower |
| Size hint speedup at 10,000 vs 5,000,000 keys | **3.5×** vs **1.2×** |
| A map with 2,000,000 entries deleted down to 3 | still holds **73,888 KB** |
| Chaining vs linear probing, heap objects | **200,404 vs 409** |
| …and GC time per cycle | **~30×** |
| Two-sum: map vs brute force at n=50,000 | **343×** |
| Two-sum with the insert in the wrong place | wrong on **1,460 of 20,000** |
| Prime-product anagram key, 64 a's | **0** — every long word collides |
| Longest-consecutive: Θ(n) set vs Θ(n log n) sort | set **loses** at 3 of 4 sizes |
| `counts[0] = 1` omitted | **6,769 of 20,000** wrong |
| hashDoS: longest bucket, normal vs attack | **8 vs 8,000**, **182×** slower |
| …the same keys through Go's map | **1.08×** |
| Slice/map crossover, string keys vs int keys | **2–4** vs **8–16** |
| `map[int]*Config` vs `map[int]Config` | **fewer bytes**, **301,357 vs 1,355** objects |
| `[]int` vs `map[int]int` on dense keys | **100× faster, 4.7× smaller** |
| `sync.Map` read, uncontended | **3.2×** a plain map read |
| Capstone vs Go's map | wins build & miss, loses hit **1.5×**, memory **2.85×** |

## The thread through the lesson

1 sets the price. 2 is the hash function, and 3–4 are the only two ways to handle collisions. 5 is why
the table must grow, and 6 is what Go actually built — a **Swiss table**, not the buckets-and-tophash
design most articles still describe. 7–10 are how to use it well. 11 is the head-to-head, and the
column I forgot. 12–13 are the problems maps own, including one where the map **loses**. 14 is the
security property. 15 is the seven times not to. 16 assembles all of it.

> Lesson: [10-hash-tables.md](../../10-hash-tables.md) · Previous: [09 — Queues & Deques](../09-queues-deques/README.md)
