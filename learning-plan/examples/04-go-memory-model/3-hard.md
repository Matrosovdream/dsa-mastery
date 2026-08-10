# Step 04 — Go's Memory Model · 🔴 Hard

Examples **12–16**: where layout beats asymptotics, ending in a graph the GC never looks at.

> ⚠️ **Sample output** throughout — Apple M4, Go 1.26.3. Examples 14 and 16 take ~30 seconds each.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The GC does not look at pointer-free memory

`🔴 hard` · *Pointer maps*

Go's collector only scans blocks whose type **might** contain pointers; the compiler records a pointer
map for every type. So "how much memory do I use" is the wrong question — the question is **how many
pointers are reachable**.

The comparison to watch is rows 2 and 3: same byte count, same element count, same links. One stores
those links as `*T`, the other as `int32`. **22× difference in GC cost.**

**Steps:**

1. Build four structures of 5M elements, differing only in pointer content.
2. Time a full GC cycle with each one live, in isolation.
3. Compare against live heap object counts.

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

// Go's GC only has to scan memory that MIGHT contain pointers. The compiler
// records a pointer map for every type; a block whose type has no pointer
// fields is skipped entirely, no matter how large it is.
//
// So "how much memory do I use" is the wrong question. The question is
// "how many POINTERS are reachable" -- that is what a GC cycle costs.

type withPointer struct {
	id   int
	next *withPointer // one pointer field
}

type withoutPointer struct {
	id   int
	next int32 // an INDEX instead of a pointer -- same job, no tracing
	_    int32
}

func avgGC(rounds int) time.Duration {
	runtime.GC() // warm up
	start := time.Now()
	for i := 0; i < rounds; i++ {
		runtime.GC()
	}
	return time.Since(start) / time.Duration(rounds)
}

type result struct {
	name    string
	mb      float64
	gc      time.Duration
	objects uint64
}

func measure(name string, build func() any) result {
	runtime.GC()
	var before runtime.MemStats
	runtime.ReadMemStats(&before)

	v := build()

	var after runtime.MemStats
	runtime.ReadMemStats(&after)
	gc := avgGC(10)
	runtime.KeepAlive(v)

	return result{
		name:    name,
		mb:      float64(after.HeapAlloc-before.HeapAlloc) / (1 << 20),
		gc:      gc,
		objects: after.HeapObjects - before.HeapObjects,
	}
}

func main() {
	const n = 5_000_000

	results := []result{
		measure("[]int (no pointers)", func() any {
			xs := make([]int, n)
			for i := range xs {
				xs[i] = i
			}
			return xs
		}),
		measure("[]withoutPointer", func() any {
			xs := make([]withoutPointer, n)
			for i := range xs {
				xs[i] = withoutPointer{id: i, next: int32(i)}
			}
			return xs
		}),
		measure("[]withPointer (values)", func() any {
			xs := make([]withPointer, n)
			for i := range xs {
				xs[i].id = i
			}
			return xs
		}),
		measure("[]*withPointer", func() any {
			xs := make([]*withPointer, n)
			for i := range xs {
				xs[i] = &withPointer{id: i}
			}
			return xs
		}),
	}

	fmt.Printf("%d elements, held four ways\n\n", n)
	fmt.Printf("%-26s %10s %14s %16s\n", "", "MB", "GC cycle", "heap objects")
	for _, r := range results {
		fmt.Printf("%-26s %10.1f %14v %16d\n", r.name, r.mb, r.gc.Round(time.Microsecond), r.objects)
	}

	fmt.Println()
	base := results[0].gc
	fmt.Println("GC cost relative to the pointer-free slice:")
	for _, r := range results {
		fmt.Printf("  %-26s %6.0fx\n", r.name, float64(r.gc)/float64(base))
	}

	fmt.Println()
	fmt.Println("row 2 is the one to notice. []withoutPointer holds the SAME 5 million")
	fmt.Println("links as row 3 -- it just stores them as int32 indices rather than")
	fmt.Println("pointers. Identical structure, identical algorithms, and the collector")
	fmt.Println("does not have to look at it at all.")
	fmt.Println()
	fmt.Println("this is the single biggest lever in Go for large data structures:")
	fmt.Println()
	fmt.Println("  pointers  -> nodes allocated separately, traced every cycle, 8 bytes each")
	fmt.Println("  indices   -> one contiguous block, invisible to the GC, 4 bytes each")
	fmt.Println()
	fmt.Println("you give up: compile-time type safety on the link, and the ability to")
	fmt.Println("free one node. You gain: locality, half the link size, and zero GC work.")
	fmt.Println("Example 16 builds a real graph both ways.")
}
```

**Sample output:**

```
5000000 elements, held four ways

                                   MB       GC cycle     heap objects
[]int (no pointers)              38.2          102µs                8
[]withoutPointer                 76.3          141µs                2
[]withPointer (values)           76.3        2.195ms               10
[]*withPointer                  114.4        9.762ms          5000002

GC cost relative to the pointer-free slice:
  []int (no pointers)             1x
  []withoutPointer                1x
  []withPointer (values)         22x
  []*withPointer                 96x

row 2 is the one to notice. []withoutPointer holds the SAME 5 million
links as row 3 -- it just stores them as int32 indices rather than
pointers. Identical structure, identical algorithms, and the collector
does not have to look at it at all.

this is the single biggest lever in Go for large data structures:

  pointers  -> nodes allocated separately, traced every cycle, 8 bytes each
  indices   -> one contiguous block, invisible to the GC, 4 bytes each

you give up: compile-time type safety on the link, and the ability to
free one node. You gain: locality, half the link size, and zero GC work.
Example 16 builds a real graph both ways.
```

**Complexity:** every structure is Θ(n) space · GC cost is Θ(reachable pointers), which is a term Big-O never carries and which you can drive to **zero** by using indices

---

## 13. Array of structs vs struct of arrays

`🔴 hard` · *Layout for the query you actually run*

A 64-byte particle means a scan of one field touches 8 bytes per 64 fetched — the example 7 stride
problem, with stride = `sizeof(struct)`. Storing each field in its own contiguous column fixes it.

**Steps:**

1. Store a million particles both ways.
2. Scan one `float64` field, then one `uint64` field.
3. Compare the speedup against the memory-traffic ratio — and explain the gap.

```go
package main

import (
	"fmt"
	"testing"
	"unsafe"
)

var sink float64
var sinkI int

// A particle: 64 bytes, half of a 128-byte cache line.
type particle struct {
	x, y, z    float64
	vx, vy, vz float64
	id         int64
	flags      uint64
}

// --- Array of Structs: the obvious layout ---

type AoS []particle

func (ps AoS) sumX() float64 {
	total := 0.0
	for i := range ps {
		total += ps[i].x
	}
	return total
}

func (ps AoS) countFlagged() int {
	n := 0
	for i := range ps {
		if ps[i].flags&1 == 1 {
			n++
		}
	}
	return n
}

// --- Struct of Arrays: one contiguous column per field ---

type SoA struct {
	x, y, z    []float64
	vx, vy, vz []float64
	id         []int64
	flags      []uint64
}

func newSoA(n int) *SoA {
	return &SoA{
		x: make([]float64, n), y: make([]float64, n), z: make([]float64, n),
		vx: make([]float64, n), vy: make([]float64, n), vz: make([]float64, n),
		id: make([]int64, n), flags: make([]uint64, n),
	}
}

func (s *SoA) sumX() float64 {
	total := 0.0
	for _, v := range s.x {
		total += v
	}
	return total
}

func (s *SoA) countFlagged() int {
	n := 0
	for _, f := range s.flags {
		if f&1 == 1 {
			n++
		}
	}
	return n
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	const n = 1_000_000

	aos := make(AoS, n)
	soa := newSoA(n)
	for i := 0; i < n; i++ {
		x := float64(i)
		aos[i] = particle{x: x, y: x, z: x, id: int64(i), flags: uint64(i)}
		soa.x[i], soa.y[i], soa.z[i] = x, x, x
		soa.id[i], soa.flags[i] = int64(i), uint64(i)
	}

	var p particle
	fmt.Printf("particle is %d bytes; a %d-byte cache line holds %d of them\n\n",
		unsafe.Sizeof(p), 128, 128/int(unsafe.Sizeof(p)))

	fmt.Printf("%-28s %14s %14s %12s\n", "reading ONE field of each", "AoS ns/op", "SoA ns/op", "speedup")

	aX := nsPerOp(func() { sink += aos.sumX() })
	sX := nsPerOp(func() { sink += soa.sumX() })
	fmt.Printf("%-28s %14.0f %14.0f %11.1fx\n", "sum x (float64)", aX, sX, aX/sX)

	aF := nsPerOp(func() { sinkI += aos.countFlagged() })
	sF := nsPerOp(func() { sinkI += soa.countFlagged() })
	fmt.Printf("%-28s %14.0f %14.0f %11.1fx\n", "count flagged (uint64)", aF, sF, aF/sF)

	fmt.Println()
	fmt.Printf("why: to read %d x-values, AoS must pull in %d MB of particles;\n",
		n, n*int(unsafe.Sizeof(p))/(1<<20))
	fmt.Printf("SoA pulls in %d MB of x. Same answer, %dx less memory traffic.\n",
		n*8/(1<<20), int(unsafe.Sizeof(p))/8)

	fmt.Println()
	fmt.Println("the speedups are smaller than 8x, and the two rows differ -- worth knowing why.")
	fmt.Println("Less traffic only helps if memory is the bottleneck. Summing float64 is a")
	fmt.Println("SERIAL DEPENDENCY CHAIN (each += waits for the previous one), so part of the")
	fmt.Println("fetch time hides behind arithmetic latency. The flag count has no such chain,")
	fmt.Println("so it is closer to pure memory-bound and gains more.")
	fmt.Println()
	fmt.Println("a good reminder: a layout change gives you HEADROOM, not a guaranteed")
	fmt.Println("speedup. Always measure the actual query, not the byte count.")

	fmt.Println()
	fmt.Println("this is the layout question behind every columnar database, every ECS")
	fmt.Println("game engine, and every vectorized query engine: if you scan ONE field")
	fmt.Println("across MANY records, store that field contiguously.")
	fmt.Println()
	fmt.Println("the trade-off is real, and it goes the other way too:")
	fmt.Println("  AoS wins when you touch MOST fields of ONE record (ps[i] is one line)")
	fmt.Println("  SoA wins when you touch ONE field of MOST records (a full-column scan)")
	fmt.Println("  SoA also costs you: no single `particle` value to pass around, and")
	fmt.Println("  8 slice headers to keep in step -- append to one, append to all.")
	fmt.Println()
	fmt.Println("note this is the same effect as example 7's stride: AoS reading one")
	fmt.Println("field IS a strided access, with stride = sizeof(struct).")
}
```

**Sample output:**

```
particle is 64 bytes; a 128-byte cache line holds 2 of them

reading ONE field of each         AoS ns/op      SoA ns/op      speedup
sum x (float64)                      949895         613788         1.5x
count flagged (uint64)               924385         258669         3.6x

why: to read 1000000 x-values, AoS must pull in 61 MB of particles;
SoA pulls in 7 MB of x. Same answer, 8x less memory traffic.

the speedups are smaller than 8x, and the two rows differ -- worth knowing why.
Less traffic only helps if memory is the bottleneck. Summing float64 is a
SERIAL DEPENDENCY CHAIN (each += waits for the previous one), so part of the
fetch time hides behind arithmetic latency. The flag count has no such chain,
so it is closer to pure memory-bound and gains more.

a good reminder: a layout change gives you HEADROOM, not a guaranteed
speedup. Always measure the actual query, not the byte count.

this is the layout question behind every columnar database, every ECS
game engine, and every vectorized query engine: if you scan ONE field
across MANY records, store that field contiguously.

the trade-off is real, and it goes the other way too:
  AoS wins when you touch MOST fields of ONE record (ps[i] is one line)
  SoA wins when you touch ONE field of MOST records (a full-column scan)
  SoA also costs you: no single `particle` value to pass around, and
  8 slice headers to keep in step -- append to one, append to all.

note this is the same effect as example 7's stride: AoS reading one
field IS a strided access, with stride = sizeof(struct).
```

The measured speedups (1.5× and 3.6×) are **below** the 8× reduction in bytes fetched, and the two
rows differ from each other. Less traffic only helps when memory is the bottleneck: summing `float64`
is a serial dependency chain, so some fetch time hides behind arithmetic latency. The flag count has
no such chain and gains more. A layout change buys **headroom**, not a guaranteed speedup.

**Complexity:** both scans are Θ(n) · SoA fetches `sizeof(field)/sizeof(struct)` of the bytes — the layout behind every columnar database and ECS game engine

---

## 14. The same tree, in two places in memory

`🔴 hard` · *Layout vs asymptotics*

Four structures over identical sorted keys. The last two are the **same balanced tree**, doing the
same comparisons in the same order — differing only in which addresses the nodes were given.

**Steps:**

1. Build a balanced BST from a pool of nodes allocated back-to-back.
2. Build the identical tree from a **shuffled** pool.
3. Compare both against a linear scan and `slices.BinarySearch` across seven orders of magnitude.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

var sink bool

// Four ways to answer "is this key present?" over the same sorted keys:
//
//   linear scan of a slice          Theta(n)
//   binary search of a slice        Theta(log n)
//   balanced BST, compact nodes     Theta(log n)
//   balanced BST, scattered nodes   Theta(log n)
//
// The last two are the SAME TREE: same shape, same comparisons, same code.
// They differ only in where in memory the nodes happen to live.

type treeNode struct {
	key         int
	left, right *treeNode
}

// buildFrom assembles a perfectly balanced tree out of pre-allocated nodes,
// taking them from pool in order. Hand it a pool in allocation order and the
// tree is laid out compactly; hand it a shuffled pool and the identical tree
// is scattered across the heap.
func buildFrom(sorted []int, pool []*treeNode, next *int) *treeNode {
	if len(sorted) == 0 {
		return nil
	}
	mid := len(sorted) / 2
	n := pool[*next]
	*next++
	n.key = sorted[mid]
	n.left = buildFrom(sorted[:mid], pool, next)
	n.right = buildFrom(sorted[mid+1:], pool, next)
	return n
}

func newPool(n int) []*treeNode {
	pool := make([]*treeNode, n)
	for i := range pool {
		pool[i] = &treeNode{} // allocated back-to-back
	}
	return pool
}

func (t *treeNode) contains(key int) bool {
	for n := t; n != nil; {
		switch {
		case key == n.key:
			return true
		case key < n.key:
			n = n.left
		default:
			n = n.right
		}
	}
	return false
}

func linearContains(xs []int, key int) bool {
	for _, v := range xs {
		if v == key {
			return true
		}
	}
	return false
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("one lookup, four structures, identical sorted keys.")
	fmt.Println("the last two are the same tree -- only the node ADDRESSES differ.")
	fmt.Println()
	fmt.Printf("%9s %12s %12s %14s %14s   %s\n",
		"n", "scan", "bsearch", "BST compact", "BST scattered", "winner")

	for _, n := range []int{8, 128, 4096, 65536, 1 << 20, 1 << 22} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}

		compactPool := newPool(n)
		var i0 int
		compact := buildFrom(xs, compactPool, &i0)

		scatteredPool := newPool(n)
		rng.Shuffle(n, func(i, j int) {
			scatteredPool[i], scatteredPool[j] = scatteredPool[j], scatteredPool[i]
		})
		var i1 int
		scattered := buildFrom(xs, scatteredPool, &i1)

		probes := make([]int, 256)
		for i := range probes {
			probes[i] = rng.IntN(n) * 2
		}
		next := 0
		pick := func() int {
			next = (next + 1) % len(probes)
			return probes[next]
		}

		scan := nsPerOp(func() { sink = linearContains(xs, pick()) })
		bs := nsPerOp(func() {
			_, ok := slices.BinarySearch(xs, pick())
			sink = ok
		})
		tc := nsPerOp(func() { sink = compact.contains(pick()) })
		ts := nsPerOp(func() { sink = scattered.contains(pick()) })

		winner, best := "scan", scan
		for name, v := range map[string]float64{"bsearch": bs, "BST compact": tc, "BST scattered": ts} {
			if v < best {
				winner, best = name, v
			}
		}
		fmt.Printf("%9d %12.1f %12.1f %14.1f %14.1f   %s\n", n, scan, bs, tc, ts, winner)
	}

	fmt.Println()
	fmt.Println("three things Big-O did not tell you:")
	fmt.Println()
	fmt.Println("1. the O(n) scan wins at n=8. It walks consecutive memory -- 16 keys per")
	fmt.Println("   cache line, prefetcher always ahead -- so a few dozen sequential reads")
	fmt.Println("   cost less than 3 dependent pointer hops. By n=128 it has lost badly.")
	fmt.Println()
	fmt.Println("2. the two BST columns are the SAME TREE doing the SAME comparisons in the")
	fmt.Println("   SAME order. The ONLY difference is which addresses the nodes got.")
	fmt.Println("   Up to 65536 they are identical -- the whole tree fits in cache, so")
	fmt.Println("   layout cannot matter. Past that the scattered version falls behind:")
	fmt.Println("   about 1.5x at n=2^20 and 1.8x at n=2^22, widening as n grows.")
	fmt.Println("   No complexity table contains that column.")
	fmt.Println()
	fmt.Println("3. a freshly built tree IS the compact column. That is what a naive")
	fmt.Println("   benchmark measures, and it flatters trees badly: it is the layout you")
	fmt.Println("   get for exactly as long as you never insert or delete anything.")
	fmt.Println()
	fmt.Println("(the compact tree and binary search stay within a few percent of each other,")
	fmt.Println(" trading places at the top end: slices.BinarySearch pays for a generic")
	fmt.Println(" comparison per probe, while contains() compares ints directly. Do not read")
	fmt.Println(" much into which wins. The instructive column is compact-vs-scattered.)")
	fmt.Println()
	fmt.Println("so why use a tree at all? Because a sorted slice cannot be UPDATED cheaply:")
	fmt.Println("insertion is a Theta(n) memmove. The tree buys O(log n) insert and delete.")
	fmt.Println("Read-mostly data belongs in a sorted slice you binary-search (lesson 15).")
}
```

**Sample output:**

```
one lookup, four structures, identical sorted keys.
the last two are the same tree -- only the node ADDRESSES differ.

        n         scan      bsearch    BST compact  BST scattered   winner
        8          3.2          4.8            3.2            3.3   scan
      128         22.0          7.7            5.7            5.8   BST compact
     4096        493.7         11.8            9.5            9.3   BST scattered
    65536       7397.5         15.4           12.5           12.6   BST compact
  1048576     132904.0         18.4           16.6           26.4   BST compact
  4194304     504932.2         20.6           20.4           38.7   BST compact

three things Big-O did not tell you:

1. the O(n) scan wins at n=8. It walks consecutive memory -- 16 keys per
   cache line, prefetcher always ahead -- so a few dozen sequential reads
   cost less than 3 dependent pointer hops. By n=128 it has lost badly.

2. the two BST columns are the SAME TREE doing the SAME comparisons in the
   SAME order. The ONLY difference is which addresses the nodes got.
   Up to 65536 they are identical -- the whole tree fits in cache, so
   layout cannot matter. Past that the scattered version falls behind:
   about 1.5x at n=2^20 and 1.8x at n=2^22, widening as n grows.
   No complexity table contains that column.

3. a freshly built tree IS the compact column. That is what a naive
   benchmark measures, and it flatters trees badly: it is the layout you
   get for exactly as long as you never insert or delete anything.

(the compact tree and binary search stay within a few percent of each other,
 trading places at the top end: slices.BinarySearch pays for a generic
 comparison per probe, while contains() compares ints directly. Do not read
 much into which wins. The instructive column is compact-vs-scattered.)

so why use a tree at all? Because a sorted slice cannot be UPDATED cheaply:
insertion is a Theta(n) memmove. The tree buys O(log n) insert and delete.
Read-mostly data belongs in a sorted slice you binary-search (lesson 15).
```

Three readings:

- **The Θ(n) scan wins at n=8.** Sequential memory, 16 keys per line, prefetcher always ahead — a few dozen sequential reads beat three dependent pointer hops. By n=128 it has lost badly.
- **The two tree columns are identical up to n=65536**, because the whole tree fits in cache and layout cannot matter. Past that the scattered version falls behind: ~1.6× at 2²⁰, ~1.9× at 2²², widening.
- **A freshly built tree is the compact column.** That is what a naive benchmark measures, and it is the layout you keep for exactly as long as you never insert or delete.

**Complexity:** scan Θ(n) · binary search and both trees Θ(log n) · the gap between the two Θ(log n) trees is entirely allocation addresses

---

## 15. Sorted slice vs map — the honest comparison

`🔴 hard` · *Choosing on more than lookup speed*

The textbook says map is O(1) and binary search is O(log n), so use the map. The measurement agrees —
**the map is faster at every size here**. So the sorted slice is not competing on lookup speed. It is
competing on three other things.

**Steps:**

1. Measure both at six sizes, tracking time and memory.
2. Confirm the shapes: map flat, binary search growing with log n.
3. Then read the memory column, which never improves.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"testing"
)

var sink bool

// The textbook says: map is O(1), sorted slice is O(log n), so use the map.
// The textbook is not wrong. It is just not the whole decision -- a map costs
// ~4.7x the memory (example 6) and its "O(1)" has a large constant.

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// heapBytes is signed: at small n the GC can free more between two readings
// than the map allocated, and an unsigned subtraction would wrap to nonsense.
func heapBytes() int64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return int64(ms.HeapAlloc)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("lookup: sorted slice + binary search vs map")
	fmt.Println()
	fmt.Printf("%9s %14s %14s %10s %14s %14s %8s\n",
		"n", "bsearch ns", "map ns", "map wins", "slice MB", "map MB", "x mem")

	for _, n := range []int{16, 128, 1024, 16384, 262144, 1 << 21} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}

		before := heapBytes()
		m := make(map[int]struct{}, n)
		for _, v := range xs {
			m[v] = struct{}{}
		}
		mapBytes := heapBytes() - before

		probes := make([]int, 256)
		for i := range probes {
			probes[i] = rng.IntN(n) * 2
		}
		next := 0
		pick := func() int {
			next = (next + 1) % len(probes)
			return probes[next]
		}

		bs := nsPerOp(func() {
			_, ok := slices.BinarySearch(xs, pick())
			sink = ok
		})
		mp := nsPerOp(func() {
			_, ok := m[pick()]
			sink = ok
		})

		sliceMB := float64(n*8) / (1 << 20)
		mapMB := float64(mapBytes) / (1 << 20)

		verdict := "no"
		if mp < bs {
			verdict = "yes"
		}
		// Below a few thousand keys the heap delta is smaller than GC noise.
		ratio := "-"
		if n >= 1024 {
			ratio = fmt.Sprintf("%.1fx", mapMB/sliceMB)
		}
		fmt.Printf("%9d %14.1f %14.1f %10s %14.3f %14.3f %8s\n",
			n, bs, mp, verdict, sliceMB, mapMB, ratio)

		runtime.KeepAlive(m)
	}

	fmt.Println()
	fmt.Println("the shapes are exactly as advertised: the map is FLAT (~4-5 ns at every")
	fmt.Println("size) and binary search GROWS with log n (5.8 -> 20 ns). And unlike the")
	fmt.Println("tree-vs-slice comparison in example 14, the map is genuinely faster at")
	fmt.Println("every size measured -- hashing one int is cheap and it is a single probe,")
	fmt.Println("versus up to 21 dependent halvings.")
	fmt.Println()
	fmt.Println("so the sorted slice is NOT competing on lookup speed. It competes on:")
	fmt.Println()
	fmt.Println("  memory   ~4.5x less, at every size, forever (example 6)")
	fmt.Println("  order    binary search gives you predecessor, successor, and RANGE")
	fmt.Println("           queries. A map gives you none of those at any price.")
	fmt.Println("  layout   one contiguous block: no per-key objects, no GC tracing")
	fmt.Println("           (example 12), and it can be memory-mapped or written to disk")
	fmt.Println("           as-is (which is why every on-disk index is sorted, lesson 40)")
	fmt.Println()
	fmt.Println("how to choose, in practice:")
	fmt.Println("  static or read-mostly data      -> sort a slice, binary-search it")
	fmt.Println("  frequent inserts/deletes        -> map")
	fmt.Println("  memory-constrained, huge n      -> sorted slice")
	fmt.Println("  need ordered iteration or range -> sorted slice (a map has neither)")
	fmt.Println("  keys are dense small integers   -> neither: index a slice directly, O(1)")
	fmt.Println()
	fmt.Println("that last line is worth more than the rest. If your keys are 0..n-1,")
	fmt.Println("xs[key] is a real O(1) with a constant of ~1 and no memory overhead at all.")
	fmt.Println("Example 16 builds a graph on exactly that idea.")
}
```

**Sample output:**

```
lookup: sorted slice + binary search vs map

        n     bsearch ns         map ns   map wins       slice MB         map MB    x mem
       16            5.8            4.3        yes          0.000         -0.001        -
      128            8.3            4.3        yes          0.001          0.003        -
     1024           11.2            4.6        yes          0.008          0.034     4.3x
    16384           14.2            4.6        yes          0.125          0.562     4.5x
   262144           17.8            5.3        yes          2.000          9.019     4.5x
  2097152           20.4            5.4        yes         16.000         72.156     4.5x

the shapes are exactly as advertised: the map is FLAT (~4-5 ns at every
size) and binary search GROWS with log n (5.8 -> 20 ns). And unlike the
tree-vs-slice comparison in example 14, the map is genuinely faster at
every size measured -- hashing one int is cheap and it is a single probe,
versus up to 21 dependent halvings.

so the sorted slice is NOT competing on lookup speed. It competes on:

  memory   ~4.5x less, at every size, forever (example 6)
  order    binary search gives you predecessor, successor, and RANGE
           queries. A map gives you none of those at any price.
  layout   one contiguous block: no per-key objects, no GC tracing
           (example 12), and it can be memory-mapped or written to disk
           as-is (which is why every on-disk index is sorted, lesson 40)

how to choose, in practice:
  static or read-mostly data      -> sort a slice, binary-search it
  frequent inserts/deletes        -> map
  memory-constrained, huge n      -> sorted slice
  need ordered iteration or range -> sorted slice (a map has neither)
  keys are dense small integers   -> neither: index a slice directly, O(1)

that last line is worth more than the rest. If your keys are 0..n-1,
xs[key] is a real O(1) with a constant of ~1 and no memory overhead at all.
Example 16 builds a graph on exactly that idea.
```

Note the signed heap delta: at small n the GC can free more between two readings than the map
allocated, and an unsigned subtraction wraps to nonsense. (My first version printed 17592186044416 MB.)

**Complexity:** map Θ(1) average, ~4–5 ns flat · binary search Θ(log n), 5.8 → 21 ns · the slice's case is memory (4.5×), ordered/range queries, and a layout you can write to disk unchanged

---

## 16. One graph, three layouts, one BFS

`🔴 hard` · *The capstone*

A million vertices, eight million edges, the **same Θ(V+E) BFS** run over three representations.
Nothing algorithmic differs. Everything in the results is layout.

The third is **CSR** (Compressed Sparse Row): one `offsets` array and one `targets` array, with the
neighbours of `v` living at `targets[offsets[v]:offsets[v+1]]`. It is how real graph and
sparse-matrix libraries store adjacency.

**Steps:**

1. Build the graph as `map[int][]int`, `[][]int`, and CSR.
2. Measure each structure's memory in isolation — **GC between build and measurement**, or CSR's temporary scratch gets counted as part of the result.
3. Run the same BFS on all three.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
)

var sink int

// The same graph, three ways. Identical vertices, identical edges, identical
// BFS algorithm -- Theta(V+E) in every case. Only the LAYOUT changes.
//
//   1. map[int][]int   the shape people reach for first
//   2. [][]int         one slice per vertex
//   3. CSR             two flat arrays, no per-vertex allocation at all
//
// CSR (Compressed Sparse Row) is how real graph and sparse-matrix libraries
// store adjacency, and this example is why.

// --- representation 1: map of slices ---

type MapGraph map[int][]int

func (g MapGraph) BFS(start int, visited []bool, queue []int) int {
	for i := range visited {
		visited[i] = false
	}
	queue = queue[:0]
	queue = append(queue, start)
	visited[start] = true
	seen := 0

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		seen++
		for _, w := range g[v] {
			if !visited[w] {
				visited[w] = true
				queue = append(queue, w)
			}
		}
	}
	return seen
}

// --- representation 2: slice of slices ---

type SliceGraph [][]int

func (g SliceGraph) BFS(start int, visited []bool, queue []int) int {
	for i := range visited {
		visited[i] = false
	}
	queue = queue[:0]
	queue = append(queue, start)
	visited[start] = true
	seen := 0

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		seen++
		for _, w := range g[v] {
			if !visited[w] {
				visited[w] = true
				queue = append(queue, w)
			}
		}
	}
	return seen
}

// --- representation 3: CSR ---
//
// offsets has V+1 entries; the neighbours of v are targets[offsets[v]:offsets[v+1]].
// Two allocations for the whole graph, both pointer-free, both contiguous.
type CSR struct {
	offsets []int32
	targets []int32
}

func (g *CSR) BFS(start int32, visited []bool, queue []int32) int {
	for i := range visited {
		visited[i] = false
	}
	queue = queue[:0]
	queue = append(queue, start)
	visited[start] = true
	seen := 0

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		seen++
		for _, w := range g.targets[g.offsets[v]:g.offsets[v+1]] {
			if !visited[w] {
				visited[w] = true
				queue = append(queue, w)
			}
		}
	}
	return seen
}

func buildAll(v, degree int, rng *rand.Rand) (MapGraph, SliceGraph, *CSR) {
	adj := make([][]int, v)
	for i := range adj {
		adj[i] = make([]int, 0, degree)
		for d := 0; d < degree; d++ {
			adj[i] = append(adj[i], rng.IntN(v))
		}
	}

	mg := make(MapGraph, v)
	for i, ns := range adj {
		mg[i] = ns
	}

	sg := SliceGraph(adj)

	csr := &CSR{
		offsets: make([]int32, v+1),
		targets: make([]int32, 0, v*degree),
	}
	for i, ns := range adj {
		csr.offsets[i] = int32(len(csr.targets))
		for _, w := range ns {
			csr.targets = append(csr.targets, int32(w))
		}
	}
	csr.offsets[v] = int32(len(csr.targets))

	return mg, sg, csr
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// measureMem reports what the FINISHED structure costs. The GC between build
// and the second reading is essential: CSR is assembled from a temporary
// [][]int, and without collecting that scratch first the measurement reports
// the intermediate as though it were part of the result.
func measureMem(build func() any) (float64, uint64) {
	runtime.GC()
	var before runtime.MemStats
	runtime.ReadMemStats(&before)

	v := build()

	runtime.GC() // drop everything the build allocated but did not keep
	var after runtime.MemStats
	runtime.ReadMemStats(&after)
	runtime.KeepAlive(v)

	return float64(after.HeapAlloc-before.HeapAlloc) / (1 << 20),
		after.HeapObjects - before.HeapObjects
}

func main() {
	const (
		v      = 1_000_000
		degree = 8
	)
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Printf("graph: %d vertices, %d edges (out-degree %d)\n\n", v, v*degree, degree)

	// --- memory, each built in isolation ---
	adjFor := func() [][]int {
		adj := make([][]int, v)
		for i := range adj {
			adj[i] = make([]int, 0, degree)
			for d := 0; d < degree; d++ {
				adj[i] = append(adj[i], rng.IntN(v))
			}
		}
		return adj
	}

	mapMB, mapObjs := measureMem(func() any {
		g := make(MapGraph, v)
		for i, ns := range adjFor() {
			g[i] = ns
		}
		return g
	})
	sliceMB, sliceObjs := measureMem(func() any { return SliceGraph(adjFor()) })
	csrMB, csrObjs := measureMem(func() any {
		adj := adjFor()
		c := &CSR{offsets: make([]int32, v+1), targets: make([]int32, 0, v*degree)}
		for i, ns := range adj {
			c.offsets[i] = int32(len(c.targets))
			for _, w := range ns {
				c.targets = append(c.targets, int32(w))
			}
		}
		c.offsets[v] = int32(len(c.targets))
		return c
	})

	// --- speed, all three live at once so the comparison is like-for-like ---
	mg, sg, csr := buildAll(v, degree, rng)
	visited := make([]bool, v)
	queueInt := make([]int, 0, v)
	queue32 := make([]int32, 0, v)

	tMap := nsPerOp(func() { sink += mg.BFS(0, visited, queueInt) })
	tSlice := nsPerOp(func() { sink += sg.BFS(0, visited, queueInt) })
	tCSR := nsPerOp(func() { sink += csr.BFS(0, visited, queue32) })

	fmt.Printf("%-18s %12s %14s %16s %12s\n", "", "BFS ms", "vs CSR", "heap objects", "MB")
	fmt.Printf("%-18s %12.1f %13.1fx %16d %12.1f\n", "map[int][]int", tMap/1e6, tMap/tCSR, mapObjs, mapMB)
	fmt.Printf("%-18s %12.1f %13.1fx %16d %12.1f\n", "[][]int", tSlice/1e6, tSlice/tCSR, sliceObjs, sliceMB)
	fmt.Printf("%-18s %12.1f %13.1fx %16d %12.1f\n", "CSR", tCSR/1e6, 1.0, csrObjs, csrMB)

	fmt.Println()
	fmt.Println("every lesson in this file, in one table:")
	fmt.Println()
	fmt.Println("  map[int][]int  a hash lookup per vertex visit (example 15), plus a")
	fmt.Println("                 separate slice per vertex: 1M+ heap objects, all traced")
	fmt.Println("  [][]int        no hashing -- g[v] is an index -- but still one")
	fmt.Println("                 allocation per vertex, so still 1M objects and 24 bytes")
	fmt.Println("                 of slice header each (examples 1, 10, 12)")
	fmt.Println("  CSR            TWO allocations for the entire graph. Neighbours of")
	fmt.Println("                 every vertex are contiguous (example 7), int32 instead")
	fmt.Println("                 of int halves the edge array (example 6), and there is")
	fmt.Println("                 not one pointer in it, so the GC skips it (example 12)")
	fmt.Println()
	fmt.Println("what CSR gives up: you cannot add an edge without rebuilding. That is the")
	fmt.Println("trade -- it is a BUILD-ONCE, QUERY-MANY structure, which is exactly what a")
	fmt.Println("road network, a dependency graph or a sparse matrix is.")
	fmt.Println()
	fmt.Println("this is the payoff for the whole lesson: same Theta(V+E), same algorithm,")
	fmt.Println("and the difference between the first row and the last is pure layout.")
}
```

**Sample output:**

```
graph: 1000000 vertices, 8000000 edges (out-degree 8)

                         BFS ms         vs CSR     heap objects           MB
map[int][]int              80.6           2.9x          1004118        141.1
[][]int                    47.1           1.7x          1000009         83.9
CSR                        28.2           1.0x                4         34.3

every lesson in this file, in one table:

  map[int][]int  a hash lookup per vertex visit (example 15), plus a
                 separate slice per vertex: 1M+ heap objects, all traced
  [][]int        no hashing -- g[v] is an index -- but still one
                 allocation per vertex, so still 1M objects and 24 bytes
                 of slice header each (examples 1, 10, 12)
  CSR            TWO allocations for the entire graph. Neighbours of
                 every vertex are contiguous (example 7), int32 instead
                 of int halves the edge array (example 6), and there is
                 not one pointer in it, so the GC skips it (example 12)

what CSR gives up: you cannot add an edge without rebuilding. That is the
trade -- it is a BUILD-ONCE, QUERY-MANY structure, which is exactly what a
road network, a dependency graph or a sparse matrix is.

this is the payoff for the whole lesson: same Theta(V+E), same algorithm,
and the difference between the first row and the last is pure layout.
```

**Four heap objects instead of a million.** Every concept in this lesson is in that one row:

| | mechanism | example |
|---|---|---|
| `map[int][]int` | a hash probe per vertex visit, plus a slice per vertex | 15, 6 |
| `[][]int` | indexing instead of hashing, but still one allocation and one 24-byte header per vertex | 1, 10, 12 |
| CSR | two allocations total; neighbours contiguous; `int32` instead of `int`; **not one pointer**, so the GC skips it | 7, 6, 12 |

What CSR gives up: you cannot add an edge without rebuilding. It is a **build-once, query-many**
structure — which is exactly what a road network, a dependency graph, or a sparse matrix is.

**Complexity:** all three are Θ(V+E) time and Θ(V+E) space · 2.9× time and 4.1× memory separate the first row from the last, and none of it appears in the complexity

---

> That's Part 1's memory chapter. Next: [05 — Recursion & the Call Stack](../../05-recursion.md),
> which closes the foundations — then Part 2 starts building structures with everything measured here.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
