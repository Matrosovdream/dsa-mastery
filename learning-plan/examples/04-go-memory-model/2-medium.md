# Step 04 — Go's Memory Model · 🟡 Medium

Examples **7–11**: cache lines, access order, and the three taxes on a pointer.

> ⚠️ **Sample output.** This whole tier measures time on real hardware — an Apple M4 with Go 1.26.3
> and a **128-byte** cache line. On a typical x86 machine the line is 64 bytes and example 7's
> plateau arrives at stride 8 instead of 16. The *shapes* are what must match, not the digits.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Measuring your cache line from Go

`🟡 medium` · *The stride cliff*

The CPU never loads one `int`; it loads a **cache line** containing that int and its neighbours. Walk
an array with a growing stride and the cost *per touch* climbs until each touch needs its own line —
then it flattens. **Where it flattens tells you the line size.**

Note the `sink`. Without it the compiler deletes the loads and every stride reports an identical
0.23 ns per touch — which would be faster than memory bandwidth allows. (Lesson 01, example 12.)

**Steps:**

1. Sum every k-th element of a 32 MB array for k = 1…128.
2. Divide by the number of touches, not by n.
3. Find where the per-touch cost stops climbing, and multiply by 8 bytes.

```go
package main

import (
	"fmt"
	"testing"
)

// sink consumes the result. Without it the compiler deletes the loads and every
// stride measures the same -- an impossible 0.23 ns per touch of a 32 MB array.
// (Lesson 01, example 12: a benchmark that discards its result measures nothing.)
var sink int

// The CPU never loads one int. It loads a CACHE LINE -- a fixed block of bytes
// containing that int and its neighbours (64 on most x86, 128 on Apple silicon).
//
// sumStride touches every k-th element, so it does n/k additions. If memory were
// flat, the cost PER TOUCH would be constant. Instead it climbs until each touch
// needs its own cache line, then flattens at the price of one line fetch.
func sumStride(xs []int, stride int) int {
	total := 0
	for i := 0; i < len(xs); i += stride {
		total += xs[i]
	}
	return total
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
	const n = 1 << 22 // 4M ints = 32 MB, far larger than any cache
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}

	fmt.Printf("summing every k-th element of %d ints (%d MB)\n\n", n, n*8/(1<<20))
	fmt.Printf("%8s %10s %12s %14s %16s\n",
		"stride", "bytes", "touches", "total ns", "ns per touch")

	var first float64
	for _, stride := range []int{1, 2, 4, 8, 16, 32, 64, 128} {
		touches := n / stride
		total := nsPerOp(func() { sink += sumStride(xs, stride) })
		perTouch := total / float64(touches)
		if first == 0 {
			first = perTouch
		}

		note := ""
		switch {
		case perTouch > 4*first:
			note = "  <- every touch is its own line"
		case perTouch > 2*first:
			note = "  <- neighbours mostly wasted"
		}
		fmt.Printf("%8d %10d %12d %14.0f %16.2f%s\n",
			stride, stride*8, touches, total, perTouch, note)
	}

	fmt.Println()
	fmt.Println("at stride 1 you pay for a line once and get its other 15 ints nearly free.")
	fmt.Println("As the stride grows, fewer of those neighbours are used -- until each touch")
	fmt.Println("needs a line of its own and the per-touch cost stops climbing.")
	fmt.Println()
	fmt.Println("WHERE it stops tells you the line size: the jump lands at stride 16,")
	fmt.Println("and 16 * 8 bytes = 128 bytes. That is this machine's cache line, measured")
	fmt.Println("from Go with no system calls. (On a 64-byte machine it lands at stride 8.)")
	fmt.Println()
	fmt.Println("the last row usually dips a little. At stride 128 only 32768 lines are")
	fmt.Println("touched -- 4 MB of distinct lines, small enough to start fitting in cache")
	fmt.Println("again. The plateau is the signal; the tail is a second-order effect.")
	fmt.Println()
	fmt.Println("consequence: an algorithm's cost is not its operation count, it is its")
	fmt.Println("LINE count. Touching 8 neighbours costs about the same as touching 1.")
}
```

**Sample output:**

```
summing every k-th element of 4194304 ints (32 MB)

  stride      bytes      touches       total ns     ns per touch
       1          8      4194304        1137326             0.27
       2         16      2097152         580937             0.28
       4         32      1048576         455940             0.43
       8         64       524288         444651             0.85  <- neighbours mostly wasted
      16        128       262144         467095             1.78  <- every touch is its own line
      32        256       131072         238921             1.82  <- every touch is its own line
      64        512        65536         118824             1.81  <- every touch is its own line
     128       1024        32768          41582             1.27  <- every touch is its own line

at stride 1 you pay for a line once and get its other 15 ints nearly free.
As the stride grows, fewer of those neighbours are used -- until each touch
needs a line of its own and the per-touch cost stops climbing.

WHERE it stops tells you the line size: the jump lands at stride 16,
and 16 * 8 bytes = 128 bytes. That is this machine's cache line, measured
from Go with no system calls. (On a 64-byte machine it lands at stride 8.)

the last row usually dips a little. At stride 128 only 32768 lines are
touched -- 4 MB of distinct lines, small enough to start fitting in cache
again. The plateau is the signal; the tail is a second-order effect.

consequence: an algorithm's cost is not its operation count, it is its
LINE count. Touching 8 neighbours costs about the same as touching 1.
```

The plateau lands at **stride 16 = 128 bytes**, which is this machine's line size — derived from Go
alone, no system calls. On a 64-byte machine it lands at stride 8.

**Complexity:** every row does Θ(n/stride) additions · the cost is governed by **lines touched**, not elements touched, and those stop being the same thing at stride 16

---

## 8. The same reads, in a different order

`🟡 medium` · *Locality*

`sumIndexed` reads exactly `len(idx)` elements. Hand it `0,1,2,…` and it walks memory; hand it a
shuffle and it hops. Identical instruction count, identical Θ(n), identical data — and up to **7.1×**
apart.

**Steps:**

1. Build one index slice in order and one shuffled.
2. Run both at four working-set sizes, from L1-resident to far larger than any cache.
3. Watch the penalty appear only once the data stops fitting.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// Without the sink the compiler removes the loads and both orders measure
// identically -- see lesson 01, example 12.
var sink int

// sumIndexed reads exactly len(idx) elements of xs, in whatever order idx says.
// Feed it 0,1,2,... and it walks memory. Feed it a shuffle and it hops.
// Same instruction count, same Big-O, same data. Only the ORDER changes.
func sumIndexed(xs, idx []int) int {
	total := 0
	for _, i := range idx {
		total += xs[i]
	}
	return total
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

	fmt.Println("identical reads, identical count -- only the ORDER differs:")
	fmt.Println()
	fmt.Printf("%10s %12s %14s %14s %14s %10s\n",
		"n", "size", "in order ns", "shuffled ns", "ns/read shuf", "penalty")

	for _, n := range []int{1 << 10, 1 << 14, 1 << 18, 1 << 22} {
		xs := make([]int, n)
		inOrder := make([]int, n)
		for i := range xs {
			xs[i] = i
			inOrder[i] = i
		}
		shuffled := make([]int, n)
		copy(shuffled, inOrder)
		rng.Shuffle(n, func(i, j int) { shuffled[i], shuffled[j] = shuffled[j], shuffled[i] })

		seq := nsPerOp(func() { sink += sumIndexed(xs, inOrder) })
		rnd := nsPerOp(func() { sink += sumIndexed(xs, shuffled) })

		size := fmt.Sprintf("%d KB", n*8/1024)
		fmt.Printf("%10d %12s %14.0f %14.0f %14.2f %9.1fx\n",
			n, size, seq, rnd, rnd/float64(n), rnd/seq)
	}

	fmt.Println()
	fmt.Println("the penalty grows with the working set:")
	fmt.Println("  8 KB    -- fits in L1, order is irrelevant")
	fmt.Println("  2 MB    -- spills out of L2, misses start to bite")
	fmt.Println("  32 MB   -- every shuffled read is a miss")
	fmt.Println()
	fmt.Println("prefetching is why in-order is so cheap: the CPU spots the pattern and")
	fmt.Println("fetches ahead. A random order defeats it completely, and you pay full")
	fmt.Println("latency on every single read.")
	fmt.Println()
	fmt.Println("this one effect underlies most of the plan's 'the slice wins' results:")
	fmt.Println("  - a linked list is a shuffled walk BY CONSTRUCTION (example 9)")
	fmt.Println("  - a tree descent jumps to an unrelated line at every level (example 14)")
	fmt.Println("  - it is why an O(n) scan beats an O(log n) search at small n")
}
```

**Sample output:**

```
identical reads, identical count -- only the ORDER differs:

         n         size    in order ns    shuffled ns   ns/read shuf    penalty
      1024         8 KB            286            289           0.28       1.0x
     16384       128 KB           4585           5566           0.34       1.2x
    262144      2048 KB          92611         146144           0.56       1.6x
   4194304     32768 KB        1233507        8743736           2.08       7.1x

the penalty grows with the working set:
  8 KB    -- fits in L1, order is irrelevant
  2 MB    -- spills out of L2, misses start to bite
  32 MB   -- every shuffled read is a miss

prefetching is why in-order is so cheap: the CPU spots the pattern and
fetches ahead. A random order defeats it completely, and you pay full
latency on every single read.

this one effect underlies most of the plan's 'the slice wins' results:
  - a linked list is a shuffled walk BY CONSTRUCTION (example 9)
  - a tree descent jumps to an unrelated line at every level (example 14)
  - it is why an O(n) scan beats an O(log n) search at small n
```

At 8 KB the order is irrelevant — everything is in L1. The penalty grows with the working set until,
at 32 MB, every shuffled read is a miss. **Prefetching** is the mechanism: the CPU spots a sequential
pattern and fetches ahead; a random order defeats it entirely.

**Complexity:** both are Θ(n) with identical operation counts · the 7.1× is entirely locality, and it is invisible in every complexity table

---

## 9. The three taxes on `[]*T`

`🟡 medium` · *Pointers cost more than a dereference*

Lesson 01 showed this comparison; here is the full accounting. A pointer slice pays on **memory**,
**locality** and **GC tracing** — and the third column below makes the mechanism concrete.

**Steps:**

1. Build 1M points as values, and as separately-allocated pointers in shuffled order.
2. Measure each in its own scope, so the GC timing reflects only that structure.
3. Compare sum time, GC cycle time, and live heap objects.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
	"time"
)

var sink int

type point struct{ x, y int }

func sumValues(ps []point) int {
	total := 0
	for i := range ps {
		total += ps[i].x + ps[i].y
	}
	return total
}

func sumPointers(ps []*point) int {
	total := 0
	for _, p := range ps {
		total += p.x + p.y
	}
	return total
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// avgGC times a full collection. Whatever is REACHABLE when this runs is what
// the collector has to walk -- so each phase must measure in its own scope.
func avgGC(rounds int) time.Duration {
	runtime.GC() // warm up; do not let the first, larger cycle skew the average
	start := time.Now()
	for i := 0; i < rounds; i++ {
		runtime.GC()
	}
	return time.Since(start) / time.Duration(rounds)
}

func measureValues(n int) (float64, time.Duration, uint64) {
	values := make([]point, n)
	for i := range values {
		values[i] = point{x: i, y: i}
	}
	t := nsPerOp(func() { sink += sumValues(values) })
	gc := avgGC(10)
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	runtime.KeepAlive(values)
	return t, gc, ms.HeapObjects
}

func measurePointers(n int) (float64, time.Duration, uint64) {
	pointers := make([]*point, n)
	for i := range pointers {
		pointers[i] = &point{x: i, y: i}
	}
	// Shuffle: a structure built up over its lifetime is not laid out in
	// allocation order, and the prefetcher can no longer guess what is next.
	rng := rand.New(rand.NewPCG(3, 5))
	rng.Shuffle(n, func(i, j int) { pointers[i], pointers[j] = pointers[j], pointers[i] })

	t := nsPerOp(func() { sink += sumPointers(pointers) })
	gc := avgGC(10)
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	runtime.KeepAlive(pointers)
	return t, gc, ms.HeapObjects
}

func main() {
	const n = 1_000_000

	valT, valGC, valObjs := measureValues(n)
	runtime.GC() // drop phase 1 before phase 2 allocates
	ptrT, ptrGC, ptrObjs := measurePointers(n)

	fmt.Printf("summing %d points, held two ways\n\n", n)

	fmt.Printf("%-24s %14s %16s %16s\n", "", "sum (ns/op)", "GC cycle", "live heap objects")
	fmt.Printf("%-24s %14.0f %16v %16d\n", "[]point (contiguous)", valT, valGC.Round(time.Microsecond), valObjs)
	fmt.Printf("%-24s %14.0f %16v %16d\n", "[]*point (chased)", ptrT, ptrGC.Round(time.Microsecond), ptrObjs)
	fmt.Printf("%-24s %13.1fx %15.0fx %15.0fx\n", "ratio",
		ptrT/valT, float64(ptrGC)/float64(valGC), float64(ptrObjs)/float64(valObjs))

	fmt.Println()
	fmt.Println("the pointer version pays THREE separate taxes, none of them in the Big-O:")
	fmt.Println()
	fmt.Println("  1. memory     2x the bytes (example 6): the pointer AND the pointee")
	fmt.Println("  2. locality   each element is a separate object, so walking the slice")
	fmt.Println("                is a shuffled walk -- the example 8 penalty, by construction")
	fmt.Println("  3. GC         the collector must trace every one of those pointers on")
	fmt.Println("                every cycle. []point contains no pointers at all, so the")
	fmt.Println("                GC can skip the whole block (example 12)")
	fmt.Println()
	fmt.Println("both loops are Theta(n). One walks memory, the other chases it.")
	fmt.Println("In Go, choosing []T over []*T is an algorithmic decision, not a style one.")
	fmt.Println()
	fmt.Println("when you DO need []*T: elements are huge and copied often, elements are")
	fmt.Println("shared/mutated through several owners, or the type is polymorphic.")
}
```

**Sample output:**

```
summing 1000000 points, held two ways

                            sum (ns/op)         GC cycle live heap objects
[]point (contiguous)             311141             62µs              174
[]*point (chased)                758038          3.666ms          1000206
ratio                              2.4x              59x            5748x

the pointer version pays THREE separate taxes, none of them in the Big-O:

  1. memory     2x the bytes (example 6): the pointer AND the pointee
  2. locality   each element is a separate object, so walking the slice
                is a shuffled walk -- the example 8 penalty, by construction
  3. GC         the collector must trace every one of those pointers on
                every cycle. []point contains no pointers at all, so the
                GC can skip the whole block (example 12)

both loops are Theta(n). One walks memory, the other chases it.
In Go, choosing []T over []*T is an algorithmic decision, not a style one.

when you DO need []*T: elements are huge and copied often, elements are
shared/mutated through several owners, or the type is polymorphic.
```

**174 heap objects versus 1,000,206.** That ratio *is* the GC column: the collector has a million
pointers to trace in one case and effectively none in the other.

**Complexity:** both sums are Θ(n) · the pointer version costs 2× memory, 2.4× time, 59× GC — and the shuffle is not artificial, it is what any structure built up over its lifetime looks like

---

## 10. Size classes: what you asked for vs what you got

`🟡 medium` · *Allocator rounding*

Go's allocator keeps free lists for a fixed set of **size classes** and rounds every allocation up to
the next one. Ask for 33 bytes, get billed 48. On a tree with millions of nodes this is real memory.

**Steps:**

1. Measure `AllocedBytesPerOp` for `make([]byte, k)` across k.
2. Read the size-class boundaries straight off the jumps.
3. Apply it to two node structs and scale to 10M nodes.

```go
package main

import (
	"fmt"
	"testing"
	"unsafe"
)

var sinkB []byte

// Go's allocator does not hand out arbitrary sizes. It keeps free lists for a
// fixed set of SIZE CLASSES (8, 16, 24, 32, 48, 64, 80, 96, 112, 128, ...) and
// rounds every allocation UP to the next one. The rounding is invisible from
// the language and very visible in your RSS.
func allocatedFor(size int) uint64 {
	res := testing.Benchmark(func(b *testing.B) {
		b.ReportAllocs()
		for b.Loop() {
			sinkB = make([]byte, size)
		}
	})
	return uint64(res.AllocedBytesPerOp())
}

type node24 struct { // 8 + 8 + 8
	key, val int
	next     *node24
}

type nodeFat struct { // 8 + 8 + 8 + 8 + 1 -> 40 declared, 48 allocated
	key, val int
	next     *nodeFat
	weight   float64
	deleted  bool
}

func main() {
	fmt.Println("what you ask for vs what you get:")
	fmt.Println()
	fmt.Printf("%10s %14s %10s %10s\n", "requested", "allocated", "wasted", "waste %")

	for _, size := range []int{1, 8, 9, 16, 17, 24, 25, 32, 33, 48, 49, 64, 65, 96, 128, 129} {
		got := allocatedFor(size)
		wasted := got - uint64(size)
		fmt.Printf("%10d %14d %10d %9.0f%%\n",
			size, got, wasted, 100*float64(wasted)/float64(got))
	}

	fmt.Println()
	fmt.Println("the jumps are the size classes. Ask for 33 bytes and you are billed 48.")
	fmt.Println()
	fmt.Println("(the first rows look free because objects under 16 bytes with no pointers")
	fmt.Println(" go through the TINY ALLOCATOR, which packs several of them into one block.")
	fmt.Println(" Add a pointer field and they rejoin the size-class table.)")

	fmt.Println()
	fmt.Println("why a data-structures course cares -- node sizes:")
	var a node24
	var b nodeFat
	fmt.Printf("  node24: Sizeof = %2d, allocated = %2d, waste %d bytes/node\n",
		unsafe.Sizeof(a), allocatedFor(int(unsafe.Sizeof(a))), allocatedFor(int(unsafe.Sizeof(a)))-uint64(unsafe.Sizeof(a)))
	fmt.Printf("  nodeFat: Sizeof = %2d, allocated = %2d, waste %d bytes/node\n",
		unsafe.Sizeof(b), allocatedFor(int(unsafe.Sizeof(b))), allocatedFor(int(unsafe.Sizeof(b)))-uint64(unsafe.Sizeof(b)))

	const n = 10_000_000
	waste := allocatedFor(int(unsafe.Sizeof(b))) - uint64(unsafe.Sizeof(b))
	fmt.Printf("\n  a %d-node tree of nodeFat wastes %.0f MB on rounding alone.\n",
		n, float64(waste)*n/(1<<20))

	fmt.Println()
	fmt.Println("two ways out, both of which this plan uses repeatedly:")
	fmt.Println("  1. shrink the struct to land on a class boundary (drop a field, use")
	fmt.Println("     int32 indices instead of 8-byte pointers -- example 16)")
	fmt.Println("  2. stop allocating per node: put the nodes in ONE []T and link them by")
	fmt.Println("     index. One allocation, no rounding, no GC pointers (examples 9, 13, 16)")
	fmt.Println()
	fmt.Println("note this is rounding on top of the struct padding from example 4:")
	fmt.Println("a badly ordered struct pads to 40, then the allocator rounds that to 48.")
}
```

**Output** (allocation sizes are deterministic — this table is stable):

```
what you ask for vs what you get:

 requested      allocated     wasted    waste %
         1              1          0         0%
         8              8          0         0%
         9             16          7        44%
        16             16          0         0%
        17             24          7        29%
        24             24          0         0%
        25             32          7        22%
        32             32          0         0%
        33             48         15        31%
        48             48          0         0%
        49             64         15        23%
        64             64          0         0%
        65             80         15        19%
        96             96          0         0%
       128            128          0         0%
       129            144         15        10%

the jumps are the size classes. Ask for 33 bytes and you are billed 48.

(the first rows look free because objects under 16 bytes with no pointers
 go through the TINY ALLOCATOR, which packs several of them into one block.
 Add a pointer field and they rejoin the size-class table.)

why a data-structures course cares -- node sizes:
  node24: Sizeof = 24, allocated = 24, waste 0 bytes/node
  nodeFat: Sizeof = 40, allocated = 48, waste 8 bytes/node

  a 10000000-node tree of nodeFat wastes 76 MB on rounding alone.

two ways out, both of which this plan uses repeatedly:
  1. shrink the struct to land on a class boundary (drop a field, use
     int32 indices instead of 8-byte pointers -- example 16)
  2. stop allocating per node: put the nodes in ONE []T and link them by
     index. One allocation, no rounding, no GC pointers (examples 9, 13, 16)

note this is rounding on top of the struct padding from example 4:
a badly ordered struct pads to 40, then the allocator rounds that to 48.
```

Note this **stacks on top of example 4**: a badly ordered struct pads to 40 bytes, and then the
allocator rounds that 40 up to 48.

**Complexity:** Θ(1) per allocation · but it multiplies every per-node structure by up to 1.5×, and it is the second half of the argument for storing nodes in one slice

---

## 11. Interface boxing vs generics

`🟡 medium` · *Why containers take type parameters*

Before generics, a reusable container held `any`. That has a memory model: an interface value is two
words, and storing a non-pointer in one usually means heap-allocating a copy for it to point at.
Generics compile per concrete type and cost **nothing**.

**Steps:**

1. Hold a million ints as `[]int` and as `[]any`; compare memory.
2. Sum them three ways: concrete, generic, and through the interface.
3. Then shuffle the `[]any` — because a container filled over its lifetime is not contiguous.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
)

var sink int

// Before generics, a reusable container held `any`. That has a memory model:
// an interface value is two words (type pointer + data pointer), and storing a
// non-pointer value in one usually means allocating a copy on the heap for the
// data word to point at.

func sumInts(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

func sumAny(xs []any) int {
	total := 0
	for _, v := range xs {
		total += v.(int) // a type assertion on every element
	}
	return total
}

type Number interface{ ~int | ~int64 | ~float64 }

// Generic code is COMPILED for the concrete type. No boxing, no assertions --
// this compiles to the same machine code as sumInts.
func sumGeneric[T Number](xs []T) T {
	var total T
	for _, v := range xs {
		total += v
	}
	return total
}

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
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

	base := heapMB()
	ints := make([]int, n)
	for i := range ints {
		ints[i] = i + 1000 // above 255, so nothing comes from the small-int cache
	}
	afterInts := heapMB()

	boxed := make([]any, n)
	for i := range boxed {
		boxed[i] = i + 1000
	}
	afterBoxed := heapMB()

	// The same values, in a shuffled order. Allocating them back-to-back (above)
	// leaves them contiguous, which is the BEST case a []any ever gets. A real
	// container filled over its lifetime looks like this one instead.
	scattered := make([]any, n)
	copy(scattered, boxed)
	rng := rand.New(rand.NewPCG(3, 5))
	rng.Shuffle(n, func(i, j int) { scattered[i], scattered[j] = scattered[j], scattered[i] })

	fmt.Printf("holding %d ints:\n\n", n)
	fmt.Printf("  []int    %7.1f MB   (%.0f bytes/element)\n",
		afterInts-base, (afterInts-base)*(1<<20)/n)
	fmt.Printf("  []any    %7.1f MB   (%.0f bytes/element: 16-byte header + a boxed int)\n",
		afterBoxed-afterInts, (afterBoxed-afterInts)*(1<<20)/n)
	fmt.Printf("  ratio    %7.1fx\n", (afterBoxed-afterInts)/(afterInts-base))

	tInts := nsPerOp(func() { sink += sumInts(ints) })
	tGen := nsPerOp(func() { sink += sumGeneric(ints) })
	tAny := nsPerOp(func() { sink += sumAny(boxed) })
	tScattered := nsPerOp(func() { sink += sumAny(scattered) })

	fmt.Println()
	fmt.Printf("%-30s %14s %14s\n", "summing them", "ns/op", "vs []int")
	fmt.Printf("%-30s %14.0f %14s\n", "sumInts([]int)", tInts, "1.0x")
	fmt.Printf("%-30s %14.0f %13.1fx\n", "sumGeneric([]int)", tGen, tGen/tInts)
	fmt.Printf("%-30s %14.0f %13.1fx\n", "sumAny([]any) contiguous", tAny, tAny/tInts)
	fmt.Printf("%-30s %14.0f %13.1fx\n", "sumAny([]any) scattered", tScattered, tScattered/tInts)

	runtime.KeepAlive(ints)
	runtime.KeepAlive(boxed)
	runtime.KeepAlive(scattered)

	fmt.Println()
	fmt.Println("generics cost nothing here: Go compiles sumGeneric for int and produces")
	fmt.Println("the same loop as the hand-written version.")
	fmt.Println()
	fmt.Println("note the two []any rows. Filled back-to-back the boxes land contiguously")
	fmt.Println("and it costs only ~1.2x -- that is the BEST case, and it is what a naive")
	fmt.Println("benchmark measures. Shuffle them, as any real container ends up, and the")
	fmt.Println("pointer chase from example 8 doubles it again.")
	fmt.Println()
	fmt.Println("[]any costs on every axis at once:")
	fmt.Println("  memory   16 bytes of interface header per element, PLUS a separate")
	fmt.Println("           heap object per value (example 5's boxing)")
	fmt.Println("  locality the values are scattered, so the sum is a pointer chase (example 8)")
	fmt.Println("  CPU      a type assertion per element")
	fmt.Println("  GC       a million more pointers to trace (example 9)")
	fmt.Println()
	fmt.Println("the rule for this plan: containers take TYPE PARAMETERS, not `any`.")
	fmt.Println("Stack[T], Queue[T], Tree[T] -- never Stack of interface{}.")
	fmt.Println()
	fmt.Println("(this is also why container/list and container/heap, which predate")
	fmt.Println(" generics and store `any`, lose to a hand-rolled generic equivalent.)")
}
```

**Sample output:**

```
holding 1000000 ints:

  []int        7.6 MB   (8 bytes/element)
  []any       22.9 MB   (24 bytes/element: 16-byte header + a boxed int)
  ratio        3.0x

summing them                            ns/op       vs []int
sumInts([]int)                         264887           1.0x
sumGeneric([]int)                      253573           1.0x
sumAny([]any) contiguous               316801           1.2x
sumAny([]any) scattered                670359           2.5x

generics cost nothing here: Go compiles sumGeneric for int and produces
the same loop as the hand-written version.

note the two []any rows. Filled back-to-back the boxes land contiguously
and it costs only ~1.2x -- that is the BEST case, and it is what a naive
benchmark measures. Shuffle them, as any real container ends up, and the
pointer chase from example 8 doubles it again.

[]any costs on every axis at once:
  memory   16 bytes of interface header per element, PLUS a separate
           heap object per value (example 5's boxing)
  locality the values are scattered, so the sum is a pointer chase (example 8)
  CPU      a type assertion per element
  GC       a million more pointers to trace (example 9)

the rule for this plan: containers take TYPE PARAMETERS, not `any`.
Stack[T], Queue[T], Tree[T] -- never Stack of interface{}.

(this is also why container/list and container/heap, which predate
 generics and store `any`, lose to a hand-rolled generic equivalent.)
```

Two readings matter here. **Generics are free** — 1.0× against the hand-written loop, because Go
compiles `sumGeneric` for `int`. And the naive `[]any` benchmark is *too kind*: filled back-to-back
the boxes land contiguously and it costs only 1.2×. Shuffled, as any real container ends up, example
8's penalty doubles it again.

**Complexity:** all three sums are Θ(n) · `[]any` costs 3× the memory, a type assertion per element, and a million extra pointers for the GC — which is why every container in this plan is `Stack[T]`, never `Stack` of `any`

---

> Next tier: [🔴 hard](3-hard.md) — where layout starts beating asymptotics.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
