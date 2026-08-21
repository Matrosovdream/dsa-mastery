# Step 10 — Hash Tables & Sets · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. Θ(n) → Θ(1), and the three things you pay for it**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. Θ(n) → Θ(1), and the three things you pay for it
- [ ] 2. What makes a hash function good, and how to measure it
- [ ] 3. Separate chaining, and what it costs
- [ ] 4. Open addressing, and the one hard problem
- [ ] 5. Load factor and rehashing
- [ ] 6. Go's map is a Swiss table

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Sets, and why `map[T]struct{}` — for the right reason
- [ ] 8. Frequency counters and the idioms that surround them
- [ ] 9. What can be a map key
- [ ] 10. What preallocation actually buys
- [ ] 11. Four implementations, five workloads, and a column I forgot

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. The rearrangement: two-sum and group anagrams
- [ ] 13. Longest consecutive sequence, and the prefix-sum map
- [ ] 14. hashDoS: the attack that made everyone randomise
- [ ] 15. Seven times a map is the wrong answer
- [ ] 16. Capstone: `HashMap[K,V]`, built, proved, measured

## The three-pass build

In `practice/10-hash-tables/`:

- [ ] **Build it** — `HashMap[K,V]`: open addressing, power-of-two capacity, `(V, bool)` on `Get`
- [ ] **Build it** — tombstones on `Delete`, and a load factor that counts `used + tombstones`
- [ ] **Build it** — a **seeded** hash (`hash/maphash`), not `hash/fnv`
- [ ] **Build it** — `rehash` that may keep the same size when the live count is low
- [ ] **Build it** — `Set[T]` on top, with union/intersect/difference that iterate the *smaller* side
- [ ] **Prove it** — model oracle against Go's own `map[K]V` over ≥100,000 random operations
- [ ] **Prove it** — invariants after *every* operation: power-of-two capacity, counters match a full scan, load within limit
- [ ] **Prove it** — a **narrow key range** so collisions, updates and re-deletes actually occur
- [ ] **Prove it** — a churn loop: assert the load factor stays bounded over thousands of put/delete cycles
- [ ] **Prove it** — an adversarial key set; assert probes/lookup stays near 1
- [ ] **Measure it** — build, hit, miss, iterate and memory against `map[K]V`
- [ ] **Measure it** — the **worst single insert**, not just the mean
- [ ] **Measure it** — heap objects and GC time with the table live
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What does a hash table buy, in one word? | lookup **independent of n** |
| What does it give up? | order, contiguity, worst-case predictability |
| Two properties of a good hash? | uniform distribution **and** avalanche — they are different |
| Why does the finalizer matter? | Go's map uses the **top 57 bits** for H1; FNV's are its worst |
| Why are collisions unavoidable? | birthday: ~1.18·√b keys give a 50% chance |
| The two collision families? | chaining · open addressing |
| Mean chain length equals…? | the load factor, by definition |
| Empty buckets at load 1.0? | **37%** (e⁻¹) |
| Why can't `Delete` free a slot in open addressing? | it breaks every key whose probe ran through it |
| What does a tombstone cost? | misses cannot stop there — probes/miss climbs |
| What must the load factor count? | `used + tombstones` |
| When may a rehash keep the same size? | when the live count is low — otherwise a tombstoned table doubles forever |
| What is Go's map, since 1.24? | a **Swiss table**: groups of 8, one control word, H1/H2 |
| How many slots does one group check resolve? | **8**, in about four instructions |
| Why is the load factor 7/8? | a group check resolves the lookup whatever the density |
| Why are map values non-addressable? | a rehash moves every entry |
| Why `map[T]struct{}` over `map[T]bool`? | **not memory** — `bool` has two ways to be absent |
| What does `make(map, n)` buy? | time (1.2–3.5×), **not** memory |
| Does a map ever shrink? | **no**. Rebuild and copy |
| Why is `NaN` a broken key? | `NaN != NaN`: insertable, unreadable, undeletable |
| Why is `map[any]T` dangerous? | comparability is checked at **run time** |
| The Θ(n²) → Θ(n) move? | rewrite the inner loop as a membership question |
| The prefix-sum identity? | `prefix[i-1] == prefix[j] - k` |
| What is hashDoS? | attacker-chosen keys that all collide |
| Which stdlib hash is safe for untrusted keys? | `hash/maphash` (seeded). **Not** `hash/fnv` |

## Numbers to find on your own machine

Several of these contradicted what I expected — do not take them on trust.

- [ ] A linear scan **beats** a map at n=8; the crossover is near 16 for ints and near 3 for strings
- [ ] `make(map, n)` and growing into a map end at **identical** memory
- [ ] FNV-1a's **top** 6 bits are ~47× worse than its low 6; a finalizer fixes it
- [ ] At load factor 1.0, ~**37%** of chained buckets are empty
- [ ] Probes/miss under linear probing at 95% load: **~125**
- [ ] Tombstones raise probes/**miss** and leave probes/**hit** unchanged
- [ ] `map[int]struct{}` and `map[int]bool` cost **the same**
- [ ] Go's map miss/hit ratio is **not** a constant — find the dense and sparse points
- [ ] Chaining beats open addressing on speed and loses ~**30×** on GC time
- [ ] The Θ(n) longest-consecutive-sequence **loses** to `slices.Sort` at most sizes
- [ ] 8,000 crafted keys take a fixed-hash table **182×** slower; Go's map is unmoved
- [ ] A map holding 3 entries can still be holding **72 MB**

> Lesson: [10-hash-tables.md](../../10-hash-tables.md) · Index: [README.md](README.md)
