# DSA Learning Plan — Data Structures & Algorithms in Go

A step-by-step path from complexity analysis to graphs, dynamic programming and production-grade
structures — every idea implemented, tested and benchmarked in Go. Each step is a self-contained
lesson with goals, concepts, exercises, best-practice notes, pitfalls, a checklist and resources.
Claude is your tutor — ask questions as you go.

## How to use this plan

1. Work through steps in order (01 → 32); the rest (33 → 45) are extensions you can take in any order
   once the core is done.
2. For each step:
   - Read the lesson file (`NN-title.md`).
   - Work the examples in `examples/NN-title/` (easy → medium → hard), retyping each one and running it.
   - Write the exercises in `practice/NN-title/` — implementation **+ test + benchmark**, every time.
   - Run the gates: `gofmt -l .`, `go vet ./...`, `go test -race ./...`, `go test -bench=. -benchmem ./...`.
3. When you finish a step, update [PROGRESS.md](PROGRESS.md) — that's where we track where you are.

Every lesson has a **Complexity Table** (the numbers you should be able to recite), a **Best Practices
& Pitfalls** section, and a **Go-specific** note: what the standard library already gives you, and what
Go's memory model does to the textbook answer.

## The three-pass method

Each structure is learned in three passes, and the lesson is only "done" when all three are:

| Pass | What you do | Why |
|------|-------------|-----|
| **1. Build it** | Hand-roll the structure from scratch, no stdlib shortcuts | You can't reason about what you can't build |
| **2. Prove it** | Table-driven tests + a brute-force oracle or `go test -fuzz` property | Off-by-one errors hide in every index-math structure |
| **3. Measure it** | `testing.B` + `-benchmem`, compare against the stdlib equivalent | Big-O is the ranking; constants decide the winner |

## Stack we're targeting

- **Go 1.22+** (generics are used freely; `slices`/`maps`/`cmp` assume 1.21+, iterators assume 1.23+)
- **Standard library only** — `slices`, `sort`, `container/heap`, `container/list`, `math/bits`, `math/big`
- **`testing`** — table-driven tests, `testing.B` benchmarks, `go test -fuzz` for property checks
- **`testing/quick`** and brute-force oracles for randomized verification
- **`pprof`** + `benchstat` for the measurement pass
- No external dependencies anywhere in the plan. Everything here is buildable with a bare Go install.

> **Note:** Practice code lives in one module at `practice/` (`go mod init github.com/Matrosovdream/dsa-mastery/practice`),
> one package per lesson. Examples are standalone `package main` programs meant for a scratch folder.

## Steps

### Part 1 — Foundations
The vocabulary. Nothing here is a data structure yet — it's how to *talk about* and *measure* one, plus
the Go-specific memory facts that make the textbook complexity numbers lie.
- [01 — Introduction: What DSA Is & Why in Go](01-introduction.md) — structures vs algorithms, the trade-off triangle (time/space/complexity-of-code), why Go is a good DSA language (explicit memory, no hidden allocations, benchmarks in the toolchain), how to read this plan
- [02 — Environment & Toolkit](02-environment-setup.md) — the practice module layout, `go test` table-driven style, `testing.B` + `-benchmem`, `-race`, `go test -fuzz`, `pprof`, `benchstat`, and the verification harness you'll reuse for 44 lessons
- [03 — Complexity Analysis](03-complexity-analysis.md) — Big-O / Θ / Ω, best/average/worst, **amortized** analysis (why `append` is O(1)), space complexity & the recursion stack, solving recurrences (recursion tree + **master theorem**), and the complexity table you must know cold
- [04 — Go's Memory Model for DSA](04-go-memory-model.md) — arrays vs slices (header, len/cap, growth), pointers & the pointer-chasing tax, **escape analysis** & the heap/stack split, **cache lines** and why an O(n) slice scan beats an O(log n) tree at small n, struct layout/padding, and why `[]T` almost always beats `[]*T`
- [05 — Recursion & the Call Stack](05-recursion.md) — base case + inductive step, tracing the stack, recursion → iteration (explicit stack), tail calls (and Go's lack of TCO), **memoization** as the bridge to DP, stack-depth limits, and mutual/tree recursion

### Part 2 — Linear Structures
The workhorses. In Go the slice already *is* your array, stack, queue and deque — so the theme here is
knowing when it isn't enough.
- [06 — Dynamic Arrays & Slices](06-arrays-slices.md) — the growth strategy & amortized O(1) `append`, `copy`, insert/delete at index, aliasing & the backing-array leak, in-place algorithms (reverse, rotate, dedupe, partition), `slices` toolbox, and building your own `Vector[T]` to see the resize math
- [07 — Linked Lists](07-linked-lists.md) — singly/doubly/circular, the node-as-pointer-struct shape, insert/delete O(1) at a held position, the classics (reverse, middle via slow/fast, **Floyd's cycle detection**, merge two sorted, remove Nth from end, palindrome), `container/list`, and the honest verdict on when a list beats a slice (rarely)
- [08 — Stacks](08-stacks.md) — LIFO from a slice, generic `Stack[T]` with a `(T, bool)` pop, the applications that *are* stacks (balanced brackets, RPN eval, infix→postfix, iterative DFS, undo), **min-stack** in O(1), queue-from-two-stacks, and the **monotonic stack** (next greater element, largest rectangle in histogram, daily temperatures)
- [09 — Queues & Deques](09-queues-deques.md) — FIFO, the slice-queue memory leak and its two fixes (head index + compaction, **ring buffer**), the deque, **monotonic deque** for sliding-window maximum, BFS as the canonical consumer, circular buffers in the wild, and the **buffered channel** as Go's built-in concurrent queue
- [10 — Hash Tables & Sets](10-hash-tables.md) — the hash function, collision resolution (**chaining** vs **open addressing**/linear probing), load factor & rehashing, Go's `map` internals (buckets, tophash, iteration randomization, non-addressable values), writing your own `HashMap[K,V]`, sets via `map[T]struct{}`, frequency counters, and the problems maps own (two-sum, group anagrams, longest consecutive sequence, subarray-sum-equals-k)
- [11 — Strings, Runes & Bytes](11-strings-bytes.md) — immutability & the `string`/`[]byte`/`[]rune` triangle, UTF-8 indexing traps, `strings.Builder` vs `+=` (O(n) vs O(n²)), the string problem set (anagram, palindrome, reverse words, compress, longest substring without repeats), and the cost model behind every string operation

### Part 3 — Sorting & Searching
The most-studied algorithms in CS, and the ones with the biggest gap between "what's in the textbook"
and "what you should actually call".
- [12 — Elementary Sorts](12-elementary-sorts.md) — bubble/selection/**insertion** sort, why insertion sort is the one that matters (adaptive, stable, in-place, and what `pdqsort` falls back to under n≈12), **stability** and why it's a correctness property, invariants, and comparison counting
- [13 — Efficient Sorts: Divide & Conquer](13-efficient-sorts.md) — **merge sort** (top-down/bottom-up, the merge step, stability, external-sort seed), **quicksort** (Lomuto vs Hoare partition, pivot choice, the O(n²) adversary, 3-way partition for duplicates), **heapsort**, **introsort/pdqsort** (what `slices.Sort` actually runs), and **quickselect** for k-th order statistics
- [14 — Linear-Time Sorts](14-linear-sorts.md) — beating the Ω(n log n) comparison bound by not comparing: **counting sort**, **radix sort** (LSD/MSD, sorting ints and strings), **bucket sort**, the distribution assumptions each one needs, and where they show up for real (IDs, timestamps, fixed-width keys)
- [15 — Binary Search & Its Variants](15-binary-search.md) — the invariant that makes it correct, the overflow bug, **lower/upper bound**, `sort.Search` & `slices.BinarySearch`, searching a rotated array, **binary search on the answer** (min capacity, Koko/ship-packing shape), search in a 2-D matrix, and how to stop getting the boundaries wrong forever
- [16 — Two Pointers, Sliding Window & Prefix Sums](16-two-pointers-windows.md) — the three techniques that turn O(n²) into O(n): **two pointers** (pair sum, dedupe in place, container-with-most-water, **Dutch national flag**), **sliding window** fixed & variable (longest substring, min window, max sum), **prefix sums** & difference arrays (range sum, subarray-sum-k with a map, 2-D prefix), and how to recognize which one a problem wants

### Part 4 — Trees
Where recursion pays off. Every structure here is a pointer struct with `nil` as its base case.
- [17 — Binary Trees & Traversals](17-binary-trees.md) — the node struct, `nil`-base-case recursion, **pre/in/post-order** (recursive *and* iterative with an explicit stack), **level-order BFS** with a queue, height/depth/diameter, path sums, invert, lowest common ancestor, serialize/deserialize, and building a tree from traversals
- [18 — Binary Search Trees](18-binary-search-trees.md) — the BST invariant, search/insert/**delete** (all three cases), in-order = sorted, successor/predecessor, validation (the `min/max` bounds trick, not the naive one), k-th smallest, range queries, building a balanced BST from a sorted slice, and the degenerate-to-a-linked-list failure mode
- [19 — Balanced Trees & B-Trees](19-balanced-trees.md) — why balance matters (O(n) → O(log n) guaranteed): rotations, **AVL** (height-balance + the 4 rebalance cases), **red-black** (the rules, and why Go's stdlib skips them), **B-trees / B+ trees** and the disk/page model that makes databases and filesystems use them, plus treaps & skip lists as the randomized alternative
- [20 — Heaps & Priority Queues](20-heaps-priority-queues.md) — heap-as-array index math, sift up/down, O(n) `heapify`, **`container/heap`** done right (implement the interface, call the package functions), min/max & generic heaps, **heapsort**, and the problem set heaps own: **top-K** in O(n log k), merge-k-sorted, **running median** (two heaps), task scheduling, **Huffman coding**, and the Dijkstra frontier
- [21 — Tries & Prefix Structures](21-tries.md) — the prefix tree (insert/search/starts-with), array vs map children and the memory trade-off, **autocomplete** (trie + heap for top-k), word search & wildcard matching, **compressed/radix tries** (and how `http.ServeMux` and IP routing tables use them), suffix tries as the bridge to Part 7, and the **BK-tree** for fuzzy lookup

### Part 5 — Graphs
The general case that everything else is a special case of. Trees are graphs; grids are graphs;
build systems, routes and dependencies are graphs.
- [22 — Graph Representations & Traversal](22-graph-basics.md) — adjacency **list vs matrix** (and the density trade-off), directed/undirected/weighted, a generic `Graph[T]`, **BFS** (shortest path in an unweighted graph + path reconstruction), **DFS** (recursive and with an explicit stack), connected components, and **grids as implicit graphs** (islands, flood fill, multi-source BFS)
- [23 — Topological Sort & Cycle Detection](23-topological-sort.md) — DAGs and what they model (build systems, task schedulers, migrations, spreadsheet recalc), **Kahn's algorithm** (in-degrees + queue) vs **DFS post-order**, **cycle detection** (3-color for directed, parent-tracking for undirected), reporting the cycle not just its existence, and longest path in a DAG
- [24 — Shortest Paths](24-shortest-paths.md) — **Dijkstra** with `container/heap` (and why it breaks on negative edges), **Bellman-Ford** + negative-cycle detection, **0-1 BFS** with a deque, **Floyd-Warshall** for all pairs, **A\*** with an admissible heuristic, and path reconstruction via the predecessor map
- [25 — Union-Find & Minimum Spanning Trees](25-union-find-mst.md) — **disjoint-set union** with path compression + union by rank/size (the near-O(1) proof sketch), its uses (connectivity, cycle detection, Kruskal, percolation, account merging), **Kruskal's** (sort + DSU) vs **Prim's** (heap), and the cut/cycle properties that make both correct
- [26 — Advanced Graph Algorithms](26-advanced-graphs.md) — **strongly connected components** (Kosaraju & Tarjan), **bridges & articulation points**, bipartite checking / 2-coloring, **maximum bipartite matching** (Hopcroft–Karp intro), **max flow / min cut** (Ford–Fulkerson, Edmonds–Karp), Euler & Hamilton paths, and the modelling skill: turning a word problem into a graph

### Part 6 — Algorithmic Paradigms
Not structures — *strategies*. This is where interview problems stop being lookups and start being
derivations.
- [27 — Greedy Algorithms](27-greedy.md) — the greedy-choice property + optimal substructure, proving correctness with an **exchange argument**, when greedy fails (and how to notice fast), the canonical set — interval scheduling, **merge intervals**, minimum platforms, jump game, gas station, fractional knapsack, Huffman — and the greedy-vs-DP decision rule
- [28 — Divide & Conquer](28-divide-conquer.md) — split/solve/combine, the **master theorem** applied, merge sort & quickselect revisited, counting inversions, closest pair of points, maximum subarray (D&C vs **Kadane**), matrix exponentiation & fast power, and **binary lifting**
- [29 — Backtracking](29-backtracking.md) — the choose/explore/un-choose skeleton, the state-space tree, **pruning** as the whole game, the canonical set — permutations & combinations, subsets, **N-queens**, **Sudoku**, word search, palindrome partitioning, combination sum — plus dedupe strategies and the complexity you should quote
- [30 — Dynamic Programming I: One Dimension](30-dp-one-dimensional.md) — the two properties (overlapping subproblems + optimal substructure), **memoization → tabulation → rolling array**, defining the state (the only hard part), Fibonacci, climbing stairs, house robber, coin change, word break, decode ways, **LIS** in O(n²) *and* O(n log n) with binary search, and Kadane as DP
- [31 — Dynamic Programming II: Two Dimensions & Knapsack](31-dp-two-dimensional.md) — grid paths & min path sum, **edit distance**, **LCS**/**LPS**, regex & wildcard matching, **0/1 knapsack** and **unbounded knapsack**, subset-sum & partition, the space-optimization pass (2-D → 1-D, iterate backwards and know why), and reconstructing the solution not just the cost
- [32 — Dynamic Programming III: Advanced](32-dp-advanced.md) — **bitmask DP** (travelling salesman, assignment), **DP on trees** (rerooting, tree diameter, house robber III), interval DP (matrix chain, burst balloons), digit DP, **DP on DAGs**, state-machine DP (stock problems), and how to spot the state you're missing

### Part 7 — Specialized Structures & Math (beyond the core plan)
Sharper tools for narrower jobs. Take these once Parts 1–6 are solid — each one is independent.
- [33 — Bit Manipulation](33-bit-manipulation.md) — the operator set, two's complement, the trick catalogue (`x & (x-1)`, `x & -x`, swap, toggle, mask, set/clear/test), **`math/bits`** (`OnesCount`, `LeadingZeros`, `TrailingZeros`, `Len`), subset enumeration, **bitsets** for memory-cheap membership, XOR problems (single number, missing number), and Gray code
- [34 — Math & Number Theory](34-math-number-theory.md) — GCD/LCM (Euclid + extended), **sieve of Eratosthenes** & linear sieve, primality (trial division → Miller–Rabin), factorization, **modular arithmetic** (fast power, modular inverse, Fermat), combinatorics (nCr with precomputed factorials, Catalan), `math/big` for overflow-free work, and the integer-overflow traps in Go
- [35 — String Algorithms](35-string-algorithms.md) — beyond brute force: **KMP** (the failure function, and *why* it works), **Rabin–Karp** rolling hash (+ the collision story), **Z-algorithm**, Boyer–Moore, **suffix arrays** & LCP, Manacher's for palindromes, Aho–Corasick multi-pattern, **edit distance** revisited, and the RE2 guarantee behind Go's `regexp` (no catastrophic backtracking)
- [36 — Range Query Structures](36-range-queries.md) — the "answer queries over a subrange, with updates" family: prefix sums (static), **Fenwick tree / BIT** (point update + prefix query, the `i & -i` magic), **segment trees** (build/query/update, lazy propagation for range updates), **sparse tables** for idempotent static queries, and the decision table for which one a problem needs
- [37 — Probabilistic & Randomized Structures](37-probabilistic-structures.md) — trading exactness for space: **Bloom filters** (false-positive math, sizing), **counting Bloom**, **Count–Min sketch** for heavy hitters, **HyperLogLog** for cardinality, **skip lists** as a randomized balanced tree, **reservoir sampling** for streams, Fisher–Yates shuffle, and randomized quickselect

### Part 8 — Systems & Concurrency DSA (beyond the core plan)
The structures a backend service actually runs. This is the bridge from interview DSA to the code in
[golang-learning](https://github.com/Matrosovdream/golang-learning).
- [38 — Concurrent Data Structures](38-concurrent-structures.md) — what breaks under goroutines: the race-free counter, mutex-guarded vs **sharded** maps, **`sync.Map`** (and its two actual use cases), `sync/atomic` & `atomic.Pointer`, **copy-on-write** structures, lock-free stack/queue with CAS, the **ABA** problem, `sync.Pool`, and verifying all of it with `-race`
- [39 — Caching & Eviction Structures](39-caching-eviction.md) — **LRU** (`container/list` + map, O(1)), **LFU** (freq buckets), **ARC**/2Q/**W-TinyLFU** sketches, TTL expiry (lazy vs a heap of deadlines), sharding for concurrency, cache stampede & **singleflight**, and measuring hit rate as the only metric that matters
- [40 — External & Streaming Algorithms](40-external-streaming.md) — when the data doesn't fit in RAM: **external merge sort** (run generation + **k-way merge** with a heap), the I/O model and why B-trees win on disk, **LSM trees** (memtable + SSTables + compaction + Bloom filters), streaming one-pass algorithms (running stats, top-k with Count–Min, quantiles/t-digest), and chunked processing in Go with `bufio`
- [41 — Scheduling, Timers & Rate Limiting](41-scheduling-rate-limiting.md) — the structures behind time: a **heap-based timer queue** vs a **hashed timing wheel** (what Go's runtime uses), **token bucket** & leaky bucket, sliding-window log vs sliding-window counter, priority scheduling & aging, interval trees for overlap queries, and job queues with retry/backoff

### Part 9 — Practice & Mastery (beyond the core plan)
Turning knowledge into recall. Do these alongside the later parts, not after.
- [42 — Benchmarking & Profiling Your Implementations](42-benchmarking-profiling.md) — `testing.B` done right (`b.ResetTimer`, `b.ReportAllocs`, dead-code-elimination traps, `b.Loop`), **`benchstat`** for real comparisons, the crossover experiment (at what n does your tree beat a linear scan?), `pprof` CPU/heap profiles on an algorithm, allocation counting with `testing.AllocsPerRun`, and reading the numbers honestly
- [43 — Testing Algorithms](43-testing-algorithms.md) — table-driven tests for algorithms, the **brute-force oracle** pattern (verify the clever version against the obvious one on random input), **property-based testing** with `go test -fuzz` (sorted output is a permutation of the input; a BST's in-order is ascending), invariant checks inside the structure, edge-case checklists (empty/one/duplicate/max), and `testing/quick`
- [44 — Interview Patterns Playbook](44-interview-patterns.md) — the pattern → template map (two pointers, sliding window, fast & slow, merge intervals, cyclic sort, in-place reversal, BFS/DFS, two heaps, subsets, modified binary search, top-K, k-way merge, DP, topological sort), how to recognize each from the problem statement in under a minute, the complexity talk track, and a spaced-repetition schedule
- [45 — Capstone: An In-Memory Database Index](45-capstone.md) — one build that uses the whole plan: a small key-value store with a **skip-list memtable**, **B+ tree** disk index, **Bloom filter** per SSTable, **LSM compaction** with a k-way merge heap, an **LRU** page cache, a **trie**-backed prefix scan, and a **query planner** picking between index scan and full scan — fully tested, fully benchmarked

## Example projects

Bigger builds that apply several lessons at once — see [example-projects/](example-projects/).

## Progress

See [PROGRESS.md](PROGRESS.md) for the current step and notes from past lessons.
