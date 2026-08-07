# Step 01 — Introduction · 🟡 Medium

Examples **6–10**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

> ⚠️ **This tier measures time and memory, so your numbers will differ from mine.** The outputs below
> were produced on an Apple M-series laptop with Go 1.26.3 — treat them as *sample output*. What must
> match is the **shape**: which column is flat, which grows, and who wins. Examples 8 and 9 are the
> exceptions — allocation counts and the crossover *shape* are stable everywhere.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 6. From counting to the clock

`🟡 medium` · *time.Since / three complexity classes*

Now that the counting has told you what to expect, the stopwatch confirms it. The same 200 lookups
over a million elements, answered three ways. Note the guard: all three strategies are checked to
agree before any timing is reported — **a fast wrong answer is not a result**.

**Steps:**

1. Build the same data as a sorted slice and as a set.
2. Time 200 lookups with a linear scan, `slices.BinarySearch`, and a map.
3. Confirm all three found the same number of hits, then print the ratios.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"time"
)

func main() {
	const n = 1_000_000
	const queries = 200

	// The even numbers 0, 2, 4, ... held two ways: a sorted slice and a set.
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i * 2
	}
	set := make(map[int]struct{}, n)
	for _, v := range xs {
		set[v] = struct{}{}
	}

	rng := rand.New(rand.NewPCG(1, 2))
	targets := make([]int, queries)
	for i := range targets {
		targets[i] = rng.IntN(n) * 2
	}

	start := time.Now()
	linHits := 0
	for _, t := range targets {
		for _, v := range xs {
			if v == t {
				linHits++
				break
			}
		}
	}
	linear := time.Since(start)

	start = time.Now()
	binHits := 0
	for _, t := range targets {
		if _, ok := slices.BinarySearch(xs, t); ok {
			binHits++
		}
	}
	binary := time.Since(start)

	start = time.Now()
	mapHits := 0
	for _, t := range targets {
		if _, ok := set[t]; ok {
			mapHits++
		}
	}
	hashed := time.Since(start)

	fmt.Printf("%d lookups over %d elements (all three found %d)\n\n", queries, n, linHits)
	if linHits != binHits || binHits != mapHits {
		panic("the three strategies disagree")
	}

	fmt.Printf("linear scan    O(n)      %10v\n", linear.Round(time.Microsecond))
	fmt.Printf("binary search  O(log n)  %10v   %8.0fx faster\n", binary.Round(time.Microsecond), float64(linear)/float64(binary))
	fmt.Printf("map lookup     O(1)      %10v   %8.0fx faster\n", hashed.Round(time.Microsecond), float64(linear)/float64(hashed))

	fmt.Println()
	fmt.Println("the clock agrees with the counting -- but only because n is big enough")
}
```

**Sample output** (timings vary):

```
200 lookups over 1000000 elements (all three found 200)

linear scan    O(n)        26.713ms
binary search  O(log n)        22µs       1198x faster
map lookup     O(1)            10µs       2596x faster

the clock agrees with the counting -- but only because n is big enough
```

**Complexity:** scan O(n) per query · binary search O(log n), needs sorted input · map O(1) average, needs O(n) build + O(n) space

---

## 7. Benchmarking from a plain program

`🟡 medium` · *testing.Benchmark / b.Loop*

You don't need a `_test.go` file to use Go's benchmark machinery. `testing.Benchmark` runs the same
loop-calibration logic from an ordinary `main`, which is what makes single-file examples like these
possible. Dividing ns/op by n gives the payoff: **ns-per-element stays flat, and that flatness *is*
the definition of O(n)**.

**Steps:**

1. Wrap the work in `func(b *testing.B)` and pass it to `testing.Benchmark`.
2. Use `for b.Loop()` (Go 1.24+) — it calibrates the iteration count and keeps the call alive.
3. Divide `res.NsPerOp()` by n and read the third column.

```go
package main

import (
	"fmt"
	"testing"
)

func sum(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

func main() {
	fmt.Printf("%9s %14s %18s %14s\n", "n", "ns/op", "ns per element", "B/op")
	for _, n := range []int{100, 1_000, 10_000, 100_000, 1_000_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}

		// testing.Benchmark runs the same machinery as `go test -bench`,
		// but from an ordinary program: it picks b.N for you and reports ns/op.
		res := testing.Benchmark(func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				sum(xs)
			}
		})

		fmt.Printf("%9d %14d %18.3f %14d\n", n, res.NsPerOp(), float64(res.NsPerOp())/float64(n), res.AllocedBytesPerOp())
	}

	fmt.Println()
	fmt.Println("ns/op grows 10x with n, ns-per-element stays flat -> that is what O(n) looks like")
}
```

**Sample output** (timings vary):

```
        n          ns/op     ns per element           B/op
      100             28              0.280              0
     1000            234              0.234              0
    10000           2288              0.229              0
   100000          24228              0.242              0
  1000000         245884              0.246              0

ns/op grows 10x with n, ns-per-element stays flat -> that is what O(n) looks like
```

**Complexity:** O(n) time, O(1) space, **0 allocations** — `sum` never touches the heap

---

## 8. Counting allocations

`🟡 medium` · *testing.AllocsPerRun*

Allocation counts are the best of both worlds: as meaningful as timing, as **reproducible as
counting**. Two functions here are both O(n), both return an identical slice — but one knows the
final size and the other discovers it 25 reallocations later. Preallocation is the cheapest
performance win in Go and it never changes the complexity you'd write down.

**Steps:**

1. Write the same loop twice: `var xs []int` vs `make([]int, 0, n)`.
2. Measure both with `testing.AllocsPerRun` (50 warm runs, returns the average).
3. Assign to a package-level `sink` so the result can't be optimized away.

```go
package main

import (
	"fmt"
	"testing"
)

// Both functions are O(n) and produce an identical slice.
// Only one of them knows how big the answer will be.

func growFromNil(n int) []int {
	var xs []int
	for i := 0; i < n; i++ {
		xs = append(xs, i)
	}
	return xs
}

func growPrealloc(n int) []int {
	xs := make([]int, 0, n)
	for i := 0; i < n; i++ {
		xs = append(xs, i)
	}
	return xs
}

var sink []int

func main() {
	fmt.Printf("%9s %18s %18s\n", "n", "nil-slice allocs", "prealloc allocs")
	for _, n := range []int{10, 100, 1_000, 10_000, 100_000} {
		a := testing.AllocsPerRun(50, func() { sink = growFromNil(n) })
		b := testing.AllocsPerRun(50, func() { sink = growPrealloc(n) })
		fmt.Printf("%9d %18.0f %18.0f\n", n, a, b)
	}

	fmt.Println()
	fmt.Println("same O(n), same output -- one reallocates and copies every time it outgrows cap")
	fmt.Println("allocation counts follow the growth policy, not the clock: this table is stable")
}
```

**Output** (stable — allocation counts don't depend on your CPU):

```
        n   nil-slice allocs    prealloc allocs
       10                  2                  1
      100                  5                  1
     1000                  9                  1
    10000                 16                  1
   100000                 25                  1

same O(n), same output -- one reallocates and copies every time it outgrows cap
allocation counts follow the growth policy, not the clock: this table is stable
```

**Complexity:** both O(n) time and O(n) space · allocations O(log n) vs **O(1)** — the amortized-append story of [06](../../06-arrays-slices.md)

---

## 9. Finding the crossover point

`🟡 medium` · *Constants vs asymptotics*

The most useful experiment in this lesson. O(1) beats O(n) — *eventually*. Below n=16 on this
machine, hashing the key costs more than scanning the entire slice. Every "which structure should I
use?" question has a crossover, and you can always find it with fifteen lines of benchmark.

**Steps:**

1. For each n, build both a slice and a map holding the same values.
2. Search for the **last** element — the scan's worst case, so the comparison is fair.
3. Print fractional ns/op (`NsPerOp()` truncates to whole nanoseconds, and at n=1 the whole operation costs less than one).

```go
package main

import (
	"fmt"
	"testing"
)

func sliceContains(xs []int, x int) bool {
	for _, v := range xs {
		if v == x {
			return true
		}
	}
	return false
}

func mapContains(m map[int]struct{}, x int) bool {
	_, ok := m[x]
	return ok
}

// nsPerOp keeps the fractional part that BenchmarkResult.NsPerOp truncates away
// -- at n=1 the whole operation costs less than a nanosecond.
func nsPerOp(r testing.BenchmarkResult) float64 {
	return float64(r.T.Nanoseconds()) / float64(r.N)
}

func main() {
	fmt.Printf("%7s %14s %14s   %s\n", "n", "slice ns/op", "map ns/op", "winner")
	crossover := 0

	for n := 1; n <= 1024; n *= 2 {
		xs := make([]int, n)
		m := make(map[int]struct{}, n)
		for i := range xs {
			xs[i] = i
			m[i] = struct{}{}
		}
		target := n - 1 // the last element: the scan's worst case

		sliceRes := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				sliceContains(xs, target)
			}
		})
		mapRes := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				mapContains(m, target)
			}
		})

		sliceNs, mapNs := nsPerOp(sliceRes), nsPerOp(mapRes)
		winner := "slice"
		if mapNs < sliceNs {
			winner = "map"
			if crossover == 0 {
				crossover = n
			}
		}
		fmt.Printf("%7d %14.2f %14.2f   %s\n", n, sliceNs, mapNs, winner)
	}

	fmt.Println()
	fmt.Printf("O(1) only starts beating O(n) at n >= %d on this machine\n", crossover)
	fmt.Println("below that, the hash costs more than the whole scan -- constants decide small n")
}
```

**Sample output** (the exact crossover moves, the shape doesn't):

```
      n    slice ns/op      map ns/op   winner
      1           1.61           1.69   slice
      2           1.59           1.71   slice
      4           1.61           1.96   slice
      8           2.23           2.95   slice
     16           4.53           3.27   map
     32           8.43           3.27   map
     64          15.84           3.27   map
    128          35.05           3.27   map
    256          66.73           3.28   map
    512         125.83           3.27   map
   1024         242.76           3.34   map

O(1) only starts beating O(n) at n >= 16 on this machine
below that, the hash costs more than the whole scan -- constants decide small n
```

Read the two columns: the slice column **doubles** every row (O(n)); the map column is **flat at ~3.3 ns** (O(1)).

> Note: the ~1.6 ns floor in the first rows is the benchmark harness itself, not the lookup — example 12 proves it.

**Complexity:** slice O(n) with a tiny constant · map O(1) with a larger one — which is exactly why the crossover exists

---

## 10. What "O(n) space" actually costs

`🟡 medium` · *runtime.MemStats*

Three structures, the same million integers, "O(n) space" for all three — and a **4.7× spread** in
actual bytes. Complexity notation deliberately discards the constant; when you're deciding whether an
index fits in RAM, the constant is the entire question.

**Steps:**

1. Force a GC and read `HeapAlloc` to get a stable "live bytes" baseline.
2. Build each structure in turn, re-measuring after each.
3. Keep everything alive with `runtime.KeepAlive` until after the last measurement, or the GC will collect it mid-experiment.

```go
package main

import (
	"fmt"
	"runtime"
)

// heapAlloc forces a GC and reports live heap bytes -- the closest thing to
// "how much space is this structure actually using".
func heapAlloc() uint64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.HeapAlloc
}

func main() {
	const n = 1_000_000
	base := heapAlloc()

	values := make([]int, n)
	for i := range values {
		values[i] = i
	}
	afterValues := heapAlloc()

	pointers := make([]*int, n)
	for i := range pointers {
		v := i
		pointers[i] = &v
	}
	afterPointers := heapAlloc()

	set := make(map[int]struct{}, n)
	for i := 0; i < n; i++ {
		set[i] = struct{}{}
	}
	afterSet := heapAlloc()

	report := func(label string, delta uint64) {
		fmt.Printf("%-22s %8.1f MB %8.1f bytes/element\n",
			label, float64(delta)/(1<<20), float64(delta)/n)
	}
	report("[]int", afterValues-base)
	report("[]*int", afterPointers-afterValues)
	report("map[int]struct{}", afterSet-afterPointers)

	// Keep everything reachable until after the last measurement.
	runtime.KeepAlive(values)
	runtime.KeepAlive(pointers)
	runtime.KeepAlive(set)

	fmt.Println()
	fmt.Println("all three hold the same 1,000,000 integers -- 'O(n) space' hides a 5x spread")
	fmt.Println("[]*int pays twice: the pointer AND the pointed-to int, each separately allocated")
}
```

**Sample output** (map overhead depends on the Go version's map implementation):

```
[]int                       7.6 MB      8.0 bytes/element
[]*int                     15.3 MB     16.0 bytes/element
map[int]struct{}           36.1 MB     37.8 bytes/element

all three hold the same 1,000,000 integers -- 'O(n) space' hides a 5x spread
[]*int pays twice: the pointer AND the pointed-to int, each separately allocated
```

`[]int` is exactly 8 bytes/element — the int itself, nothing more. `[]*int` is 16: an 8-byte pointer
plus an 8-byte heap object. The map's ~38 buys you the O(1) lookup from example 1: **that's the price
of the hash table**, and now you know what it is.

**Complexity:** all three O(n) space · the constants are 1×, 2×, and ~4.7× — and the constant is what you have to budget for

---

> Next tier: [🔴 hard](3-hard.md) — where the measurements start lying to you.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
