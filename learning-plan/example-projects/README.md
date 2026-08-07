# Example Projects — DSA applied

Eight runnable Go projects, easy → hard. Each one exists because a **data structure** makes it
possible — not as a CRUD app that happens to be written in Go. Every project is stdlib-only (no
external modules, no `go.sum`), ships a `README.md` (what it is, how to run, which lessons it uses)
and a `progress.md` build checklist, and comes with tests **and** benchmarks.

The rule: a project is finished when you can point at the benchmark that proves the structure was
worth it — the naive version is in the repo too, and it's slower.

---

## The shared shape

Small enough to stay flat, big enough to have a boundary:

```
<project>/
├── cmd/<name>/main.go     # CLI or HTTP entry point
├── internal/
│   ├── <structure>/       # the data structure, standalone & tested (trie, graph, lsm, ...)
│   └── <domain>/          # the thing built on top of it
├── testdata/              # fixtures + fuzz corpus
└── README.md + progress.md
```

`internal/<structure>/` must be importable and testable on its own — that's the whole point. If it
needs the rest of the app to be tested, it isn't a data structure yet.

---

## 🟢 Beginner

**1. `maze-solver-beginner`** — *lessons 09, 22, 24*
An ASCII maze loaded from a text file, solved four ways: DFS (finds *a* path), BFS (finds the
shortest), Dijkstra (weighted terrain), and A\* with a Manhattan heuristic. Renders the explored set
per algorithm so you can *see* why A\* visits fewer nodes. Benchmarks compare node-expansion counts.

**2. `autocomplete-service-beginner`** — *lessons 20, 21*
An HTTP endpoint that returns the top-K completions for a prefix, backed by a trie with per-node
frequency and a min-heap for the top-K pass. Includes the naive `strings.HasPrefix` scan over a
sorted slice as a benchmark baseline — find the n where the trie wins.

**3. `dep-graph-beginner`** — *lessons 22, 23, 25*
A build-order tool: reads a dependency file, topologically sorts it (Kahn's), and on failure reports
the **actual cycle**, not just "cycle detected". Adds parallel-batch output (which targets can build
concurrently) and a `--why A B` path query.

---

## 🟡 Intermediate

**4. `text-index-intermediate`** — *lessons 10, 11, 13, 16, 20, 35*
A miniature search engine over a folder of documents: tokenizer → **inverted index** (`map[term][]posting`)
→ boolean AND/OR queries via sorted-postings intersection → TF-IDF ranking with a top-K heap →
phrase queries using positional postings. Compare intersection strategies (galloping vs linear merge).

**5. `route-planner-intermediate`** — *lessons 20, 22, 24, 25*
Shortest routes over a real coordinate graph: Dijkstra with `container/heap`, A\* with a haversine
heuristic, bidirectional search, and k-shortest-paths (Yen's). Reports nodes expanded and wall time
per strategy — the benchmark *is* the deliverable.

**6. `rate-limiter-intermediate`** — *lessons 09, 38, 41*
A limiter library with four interchangeable implementations behind one interface: fixed window,
**sliding-window log** (ring buffer), **sliding-window counter**, and **token bucket**. Sharded for
concurrency, verified under `-race`, benchmarked at 1/8/64 goroutines, with an accuracy harness that
measures how badly each one over/under-admits at a burst boundary.

---

## 🔴 Hard

**7. `spellchecker-hard`** — *lessons 21, 31, 35, 37*
Fuzzy word lookup three ways: edit-distance DP (the baseline), a **BK-tree** over the dictionary, and
a trie walked with a DP row per node (the fast one). Adds a Bloom filter as an "is this word
definitely unknown?" front door. Benchmarks show the DP baseline losing by two orders of magnitude.

**8. `kvstore-lsm-hard`** — *lessons 19, 20, 37, 39, 40, 45*
The capstone as a standalone project: a persistent key-value store with a **skip-list memtable**,
**SSTable** flush with a sparse index, a **Bloom filter** per table, **k-way merge compaction** via a
heap, an **LRU** block cache, a write-ahead log for crash recovery, and range scans. Crash-recovery
tests, a fuzz-driven oracle against `map[string]string`, and read/write benchmarks at three data sizes.

---

## Order

Do 1–3 after Part 5 (graphs), 4–6 after Part 6 (paradigms), 7–8 after Part 8 (systems DSA). Nothing
here needs a database, a broker, or a network — a bare Go install runs all eight.

---
*Plan: [../README.md](../README.md) · Progress: [../PROGRESS.md](../PROGRESS.md).*
