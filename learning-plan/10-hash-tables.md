# 10 — Hash Tables & Sets

> Part of **Part 2 — Linear Structures**, and the lesson where the subject changes. Everything from
> [06](06-arrays-slices.md) to [09](09-queues-deques.md) was about **order** — what is next, what is
> at index `i`, what came first. A hash table throws order away and buys the one thing none of them
> could: **find an arbitrary key in Θ(1), independent of n**. The one-line thesis: **that Θ(1) is an
> average over a randomised hash, not a guarantee — and almost everything worth knowing about hash
> tables is about what happens when the average does not hold.**

## Goals
- State what a hash table buys and the three things it costs, with numbers.
- Measure a hash function two ways — **distribution** and **avalanche** — and know why both are needed.
- Implement **separate chaining** and **open addressing**, and say why the industry moved.
- Explain tombstones, and why deleting a key in an open-addressed table can delete its neighbours.
- State the load-factor rule that keeps a churned table terminating.
- Describe Go's map as it actually is today: a **Swiss table**, not buckets and `tophash`.
- Use `map[T]struct{}` for the right reason, and know that the usual reason is false.
- Know exactly what can be a key, and the two types that compile and then betray you.
- Turn a Θ(n²) loop into Θ(n) by rewriting it as a membership question — and know when not to.
- Explain hashDoS and why Go's map is immune.

## Concepts

- **What it buys.** At n = 1,000,000 a map lookup is **4.9 ns** against **190 µs** for a linear scan.
  Read the map column *down* a table of growing n and it barely moves; that is what Θ(1) looks like on
  a clock. At n = 8 the **linear scan wins** — eight ints are one cache line, and a map still has to
  hash. The crossover is around 16 for int keys and around 3 for strings, because it depends on how
  expensive `==` is for your key type.

- **What it costs.** Three things, and the third is the one people forget: **order** (no first, no
  last, no range query), **contiguity** (every lookup is a cache miss), and **predictability** (Θ(1)
  is amortized *and* average-case). Plus **2.4–3.7× the memory** of the raw keys — you are buying
  empty space, and a hash table is only fast while it stays sparse.

- **A good hash is one whose output looks random when the input does not.** Real keys are `user_1`,
  `/api/v1/orders`, timestamps a second apart. Measure it two ways: chi-square against uniform (should
  land near the bucket count) and **avalanche** — flip one input bit, count output bits changed
  (should be ~32 of 64). They catch different failures: FNV-1a scores **77** on the first and only
  **25.4** on the second.

- **Which bits are good is a third question.** FNV-1a's **top 6 bits are 47× worse than its low 6**,
  which matters because Go's map takes its group index from the **upper 57 bits**. A three-multiply
  finalizer (the splitmix64 mixer) fixes both ends at once: **54 and 32.0**, top/low ratio **1×**. If
  you write a hash, finalize it.

- **Collisions are arithmetic, not a defect.** By the birthday bound you need only ~1.18·√b keys
  before a collision is more likely than not — **77,163 keys in a four-billion-slot table**. Every
  hash table therefore needs a collision strategy, and there are exactly two families.

- **Chaining**: each bucket holds a list. It cannot fail, load factors above 1 are legal, and deletion
  is genuinely Θ(1). The **mean chain length *is* the load factor** — that is the definition, not a
  coincidence — so an unsuccessful lookup costs exactly that. At load factor 1.0, **37% of buckets are
  still empty** (e⁻¹), which is why chaining wastes space exactly where it stops being fast.

- **Open addressing**: no nodes, no pointers, one contiguous array. When a slot is taken, take the
  next. Probes/miss follows Knuth's `(1 + 1/(1-a)²)/2` — measured **1.38 at load 0.25** and **125.00
  at 0.95** — so a full table does not merely slow down, it stops terminating.

- **A tombstone is not an optimisation, it is the only correct delete.** Marking a slot free breaks
  every key whose probe sequence ran through it: deleting one key in a three-key cluster made the
  other two **unreachable while still sitting in the array**. The tombstone says "nothing lives here,
  keep walking", which is why an open-addressed table needs three slot states, not two.

- **The load factor must count `used + tombstones`.** Counting only live entries means a churned table
  never grows, never clears its tombstones, and eventually never terminates. Measured: with the live
  count **constant**, probes/miss went from **4.56 to 14.28** while probes/*hit* never moved — the
  damage is invisible if you only measure successful lookups.

- **A rehash must be allowed to keep the same size.** A grow routine that always doubles will double a
  table that is 90% tombstones — and then it is 45% tombstones and twice as big, forever.

- **Amortized Θ(1), for the fourth time in this repo.** Growing costs **1.97 moves per insert**
  averaged, while the **worst single `Put` moved 196,608 entries**. Same shape as
  [06](06-arrays-slices.md)'s `append` and [09](09-queues-deques.md)'s ring: the average says nothing
  about the tail.

- **Go's map is a Swiss table, and has been since Go 1.24.** If you have read about `bmap`, buckets of
  8 with a `tophash` array, and overflow buckets — that implementation no longer exists. What replaced
  it:

  | Term | Meaning |
  |---|---|
  | slot | one key/elem pair |
  | group | **8 slots + one 8-byte control word** |
  | control byte | empty (`0b10000000`), deleted (`0b11111110`), or the low 7 bits of a hash |
  | H1 | upper 57 bits — chooses the group |
  | H2 | lower 7 bits — lives in the control word |
  | table | array of groups; directory of tables above that (extendible hashing) |

  Eight 7-bit hash fragments sit in one `uint64`, so a single SWAR expression compares a candidate
  against **all eight slots in about four instructions**. That is the whole answer to "why 7/8 load
  factor": a probe is a *group* check, so density barely matters.

- **Miss/hit is not a constant.** I expected a miss to be as fast as a hit — it resolves in the control
  word without touching a key. Measured: **2.58× on a dense table** (a miss cannot stop until it finds
  an *empty* slot), **1.29× on a sparse one**, and **0.85× at 2,000,000 keys**, where both are
  memory-bound and the *hit* pays an extra cache line to compare the key.

- **Map values are not addressable, and now you know why.** A rehash moves every entry, so `&m[k]`
  would dangle. `m[k]++` is fine; `m[k].Field = 1` is not. Store a `*T` if you need it — at one
  allocation and one traced pointer per entry.

- **`map[T]struct{}` does not save memory.** `map[int]struct{}`, `map[int]bool`, `map[int]int8` and
  `map[int]int` all measured **36.1 bytes per key** — identical. A struct ending in a zero-sized field
  is padded, so `unsafe.Sizeof(struct{k int; v struct{}})` is **16**, exactly like `{int, int}`. Use
  `struct{}` for the real reason: `map[T]bool` has **two ways to be absent**, and `if m[k]` is wrong
  for every key stored as `false`.

- **`comparable` has sharp edges.** Slices, maps and funcs are compile errors — a kindness. The two
  that compile are the problem: `NaN != NaN`, so three inserts make three entries, **none of them
  retrievable or deletable**; and `map[any]T` defers the comparability check to **run time**, where it
  panics with `hash of unhashable type []int`. Pointer keys mean identity, not equality, and
  `time.Time` compares unequal for the same instant.

- **The size hint buys time, not memory.** Hinted and grown maps end at **identical** capacity, every
  run. What it saves is the rehashing on the way up: **3.5× at 10,000 keys and only 1.2× at
  5,000,000**, because at scale the build is memory-bound and the rehash work hides in the stalls.

- **A map never shrinks.** 2,000,000 entries deleted down to 3 still held **73,888 KB**. `clear(m)`
  keeps the capacity by design; the only way to return the memory is to build a new map and copy.
  Third structure with this policy after [06](06-arrays-slices.md) and [09](09-queues-deques.md).

- **The rearrangement.** A family of Θ(n²) problems collapses the moment you notice the inner loop is
  asking a membership question: `nums[i] + nums[j] == target` becomes `nums[j] == target - nums[i]`;
  `sum(i..j) == k` becomes `prefix[i-1] == prefix[j] - k`. Measured **343×** at n = 50,000. But a map
  beats a **nested loop** reliably and a **sort** only sometimes: the Θ(n) longest-consecutive-sequence
  *lost* to `slices.Sort` at three of four sizes.

- **hashDoS.** If the hash is deterministic and public, an attacker can compute keys that all collide
  and hand them to any endpoint that builds a map from input. Measured: 8,000 crafted keys took the
  longest bucket from **8 to 8,000** and insertion **182× slower**. Go seeds every hash per process,
  so the same keys cost **1.08×**. `hash/fnv` takes no seed and is not safe for this; **`hash/maphash`
  exists precisely for it**.

- **The general principle behind that:** a structure whose worst case is much worse than its average
  case has a **security** property, not just a performance one. Ask who chooses the input.

## Complexity Table

| Operation / structure | Average | Worst | Note |
|---|---|---|---|
| `map[K]V` get / put / delete | **Θ(1)** | Θ(n) | worst needs a broken or attacked hash |
| Chained get, load factor α | Θ(1 + α) | Θ(n) | unsuccessful lookup walks the whole chain |
| Open-addressed get, load α | Θ(1) | Θ(n) | probes/miss ≈ (1 + 1/(1−α)²)/2 |
| Rehash | Θ(n) | Θ(n) | amortizes to **1.97 moves/insert** |
| Worst single insert during growth | — | **Θ(n)** | measured 196,608 entries moved |
| Iterate a map | Θ(capacity) | | scans empty slots too; **random order** |
| Iterate in sorted order | Θ(n log n) | | collect keys, then sort |
| Smallest key / next key above x | **Θ(n)** | | a map is the wrong structure |
| Set union / difference | Θ(\|a\| + \|b\|) | | |
| Set intersection | **Θ(min(\|a\|,\|b\|))** | | *only* if you iterate the smaller side |
| Two-sum, group anagrams | **Θ(n)** | | vs Θ(n²) — measured 343× |
| Subarray-sum-equals-k | **Θ(n)** | | vs Θ(n²) — measured 749× |
| Longest consecutive sequence | **Θ(n)** | | and it still loses to an Θ(n log n) sort |
| Dense integer keys in `[]T` | **Θ(1)** | Θ(1) | 113× faster and 4.7× smaller than a map |

Measured (Apple M4, Go 1.26.3): map lookup **4.9 ns** at n = 10⁶ · linear scan **wins at n = 8** ·
map memory **2.4–3.7×** the keys · size hint changes memory by **0%** · FNV-1a avalanche **25.4 → 32.0**
with a finalizer · **37%** of buckets empty at load 1.0 · probes/miss **125** at 95% load · tombstones
took probes/miss **4.56 → 14.28** with live keys constant · `map[T]struct{}` and `map[T]bool`
**identical** · intersecting the larger set **79,796×** slower · a map with 3 entries holding
**73,888 KB** · chaining **200,404 heap objects vs 409** and ~30× the GC time · hashDoS **182×**, and
**1.08×** through Go's map.

## Exercises
1. Race a linear scan, a binary search and a map across n = 8 … 10⁶. Find the crossover for **int** keys and for **string** keys, and explain why they differ.
2. Measure a map's bytes-per-key at 2¹⁹, 700,000, 10⁶ and 2²⁰. Explain the swing. Then check whether a size hint changes it — three times, and say why three.
3. Write four hash functions and score each on chi-square *and* avalanche. Then score their **top** bits separately from their low bits, and add a finalizer.
4. Implement separate chaining. Write `Put` without the duplicate check first, then show a table that reports 3 entries for 1 key and returns a stale value after `Delete`.
5. Measure chain length against load factor. Confirm the mean equals the load factor and that ~37% of buckets are empty at 1.0.
6. Implement open addressing. Write `Delete` as "mark it free", then find the input where it silently deletes two other keys.
7. Add tombstones. Then churn the table with the live count constant, and measure probes/**hit** and probes/**miss** separately.
8. Implement rehashing. Report moves/insert *and* the worst single insert. Then make your `grow` keep the same size when the table is mostly tombstones, and prove it fires.
9. Reimplement `matchH2` from Go's runtime and demonstrate it resolving eight slots at once. Then explain the 7/8 load factor from that.
10. Measure Go's map hit against miss at several sizes. Explain the non-monotonic ratio using bytes-per-key.
11. Measure `map[int]struct{}` against `map[int]bool`, `map[int]int8` and `map[int]int`. Then explain the result with `unsafe.Sizeof`.
12. Write `Set[T]` with union, intersection and difference. Make intersection iterate the smaller side and measure what that is worth.
13. Put `NaN` in a map three times. Then read it, delete it, and range over it. Then do the same with `map[any]T` and a slice.
14. Solve two-sum with a map. Find the input where inserting before the lookup gives the wrong answer, and verify against brute force on a **narrow value range**.
15. Solve subarray-sum-equals-k. Omit `counts[0] = 1` and measure how many of 20,000 random cases break.
16. Generate a colliding key set for a fixed hash and measure the slowdown. Then add a seed and measure again. Then run the same keys through `map[string]struct{}`.
17. Stretch — package `HashMap[K,V]` and apply all three passes: a model oracle against Go's own map, invariants after every operation, a churn loop that bounds the load factor, an adversarial key set, and a benchmark against the map you are imitating.

## Best Practices & Pitfalls
- **Write `map[K]V`.** The hand-written tables in this lesson exist so you can say *why* it is shaped the way it is, not so you ship one.
- **Pass the size hint** whenever you know it. One extra argument, 1.2–3.5×, no memory cost.
- **Pitfall — assuming the hint saves memory.** It does not. It saves rehashing.
- **Pitfall — expecting a map to shrink.** It never does. Rebuild and copy, on a schedule you choose.
- **Pitfall — `clear(m)` when you meant to release the memory.** `clear` keeps the capacity.
- **Use `map[T]struct{}` for sets** — because `map[T]bool` has two ways to be absent, not to save space.
- **Use comma-ok whenever the zero value is meaningful.** That is the whole rule.
- **Pitfall — a nested `map[K]map[K2]V`.** Prefer a composite key: structs and arrays are comparable.
- **Pitfall — iterating the larger set in an intersection.** Θ(min) is one `if` away.
- **Sort before you emit.** Anything that produces a file, a hash, a golden test or a diff must sort the keys, and the sort must be a **total** order.
- **Pitfall — adding to a map while ranging it.** Deleting is guaranteed safe; adding is *unspecified*, which is worse, because it will work in testing.
- **Pitfall — float keys.** `NaN` makes entries that cannot be read or deleted.
- **Pitfall — `map[any]T`.** The comparability check happens at run time, in production, on a caller's value.
- **Pitfall — `time.Time` as a key.** Use `UnixNano()`.
- **Store small values by value, not by pointer** — for the collector, not for the byte count.
- **Never use an unseeded hash (`hash/fnv`, `crc32`) as a table hash for untrusted keys.** Use `hash/maphash`.
- **Reach for an array when the keys are dense integers.** A slice index *is* a perfect hash.
- **Don't assume Θ(n) beats Θ(n log n).** Measure. Sorting contiguous memory is very fast.

## Checklist
- [ ] I can state what a hash table buys and the three things it costs.
- [ ] I can measure a hash function's distribution *and* its avalanche, and say why both matter.
- [ ] I know Go's map takes its group index from the **top** bits, and why that constrains the hash.
- [ ] I can implement chaining and open addressing, and argue for open addressing on **GC**, not speed.
- [ ] I can explain tombstones, and why the load factor must count them.
- [ ] I know a rehash may keep the same size, and what breaks if it cannot.
- [ ] I can describe a Swiss table: group, control word, H1, H2 — and derive 7/8 from it.
- [ ] I know why map values are not addressable.
- [ ] I use `map[T]struct{}` and can give the correct reason.
- [ ] I can name the key types that compile and then fail at run time.
- [ ] I recognise "for each x, is there a y…" as a map problem, and I test it on a narrow value range.
- [ ] I can explain hashDoS and name the safe stdlib hash.
- [ ] I can name three cases where a map is the wrong answer, with numbers.

## Resources
- Abseil's Swiss Table design note — the origin of Go's current map: https://abseil.io/about/design/swisstables
- Go's implementation, worth reading for the header comment alone: https://github.com/golang/go/blob/master/src/internal/runtime/maps/map.go
- …and the SWAR trick itself, `ctrlGroup.matchH2`: https://github.com/golang/go/blob/master/src/internal/runtime/maps/group.go
- `hash/maphash` — the seeded hash to use in your own tables: https://pkg.go.dev/hash/maphash
- Go spec on map iteration order and comparability: https://go.dev/ref/spec#For_range
- "Efficient Denial of Service Attacks on Web Application Platforms" (28C3, 2011) — the hashDoS disclosure: https://en.wikipedia.org/wiki/Collision_attack#Hash_flooding
- Knuth, TAOCP vol. 3, §6.4 — linear probing, and the (1 + 1/(1−α)²)/2 estimate.
- CLRS ch. 11 — hash tables, universal hashing, open addressing.
- Examples: [examples/10-hash-tables](examples/10-hash-tables/) (16).
- Note: this lesson supersedes the common description of Go's map as buckets with a `tophash` array and overflow chains. That was accurate before Go 1.24 and is not accurate now.
- Next: [11 — Strings, Runes & Bytes](11-strings-bytes.md) — the type whose length is three different numbers.
