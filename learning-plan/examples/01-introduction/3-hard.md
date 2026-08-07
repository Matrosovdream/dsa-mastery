# Step 01 — Introduction · 🔴 Hard

Examples **11–14**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

This tier is about the ways a measurement can be **wrong**, and the one cheap way to be sure your
code is **right**. Examples 11–13 print timings that vary by machine; example 14 is deterministic.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 11. Big-O lies at small n

`🔴 hard` · *Crossover / why hybrid sorts exist*

Insertion sort is O(n²) and `slices.Sort` is O(n log n), so the textbook says this is no contest. The
textbook is talking about the limit. At n=16 the "inferior" algorithm is **20% faster**, because it
has no pivot selection, no recursion, no bookkeeping — just a tight loop over data that fits in L1.
This is not a curiosity: it's why Go's own `slices.Sort` (pdqsort) drops to insertion sort at
`length <= 12`.

**Steps:**

1. Write insertion sort; sort a *copy* so each iteration starts from the same unsorted data.
2. Put the `copy` **inside** the timed loop for both contenders, so they pay the same overhead.
3. Sweep n from 4 to 8192 and find where the winner flips.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// insertionSort is O(n^2) -- textbook-inferior to any O(n log n) sort.
func insertionSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		xs[j+1] = v
	}
}

func main() {
	rng := rand.New(rand.NewPCG(7, 11))

	fmt.Printf("%7s %18s %18s   %s\n", "n", "insertion ns/op", "slices.Sort ns/op", "winner")
	crossover := 0

	for _, n := range []int{4, 8, 16, 32, 64, 128, 1024, 8192} {
		src := make([]int, n)
		for i := range src {
			src[i] = rng.IntN(1 << 20)
		}

		// Copy fresh data inside the timed loop -- both contenders pay the same copy.
		insRes := testing.Benchmark(func(b *testing.B) {
			buf := make([]int, n)
			for b.Loop() {
				copy(buf, src)
				insertionSort(buf)
			}
		})
		stdRes := testing.Benchmark(func(b *testing.B) {
			buf := make([]int, n)
			for b.Loop() {
				copy(buf, src)
				slices.Sort(buf)
			}
		})

		winner := "insertion"
		if stdRes.NsPerOp() < insRes.NsPerOp() {
			winner = "slices.Sort"
			if crossover == 0 {
				crossover = n
			}
		}
		fmt.Printf("%7d %18d %18d   %s\n", n, insRes.NsPerOp(), stdRes.NsPerOp(), winner)
	}

	fmt.Println()
	fmt.Printf("the O(n^2) sort wins until n = %d, then loses by orders of magnitude\n", crossover)
	fmt.Println("this is not a paradox: Big-O describes the limit, not your actual input size")
	fmt.Println("Go's own slices.Sort (pdqsort) drops to insertion sort at n <= 12 for exactly this reason")
}
```

**Sample output** (timings vary; the crossover is typically 16–64):

```
      n    insertion ns/op  slices.Sort ns/op   winner
      4                  3                  4   insertion
      8                 11                 17   insertion
     16                 33                 40   insertion
     32                106                 97   slices.Sort
     64                418                256   slices.Sort
    128               1697                627   slices.Sort
   1024              97376               8418   slices.Sort
   8192            5953672              92974   slices.Sort

the O(n^2) sort wins until n = 32, then loses by orders of magnitude
this is not a paradox: Big-O describes the limit, not your actual input size
Go's own slices.Sort (pdqsort) drops to insertion sort at n <= 12 for exactly this reason
```

Watch the last two rows: 8× the input costs insertion sort **61×** the time (≈8²) and `slices.Sort`
**11×** (≈8·log). The asymptotics were right all along — they just weren't relevant at n=16.

**Complexity:** insertion O(n²) worst, O(n) on nearly-sorted input, in-place · pdqsort O(n log n) worst, O(log n) stack — both in full at [12](../../12-elementary-sorts.md) and [13](../../13-efficient-sorts.md)

---

## 12. The benchmark that measured nothing

`🔴 hard` · *Dead-code elimination / harness overhead*

The most important example in this lesson, and the least comfortable. Three standard ways to
benchmark a one-multiply function, and **all three report a number that isn't the function's cost**.
The fix isn't a trick — it's a control.

Two independent failures are on display:

1. **Dead-code elimination.** `square` is small and pure, so if its result is unused the compiler
   deletes the call. The famous "accumulate into a package-level sink" defense **does not save you
   here**: the compiler can still see through the whole loop.
2. **Harness overhead.** `b.Loop` prevents the elimination — but costs ~1.6 ns per iteration itself.
   A benchmark reporting 1.6 ns/op for a single multiply measured the loop, not the multiply. You
   only discover this by running an **empty** `b.Loop` as a control.

And a warning against over-correcting: DCE is *selective*. `fib(20)` discarded in the very same kind
of loop is **not** removed — it's recursive, can't be inlined, and can't be proven pure.

**Steps:**

1. Benchmark `square` four ways: discarded, sunk, via `b.Loop`, and with an empty `b.Loop` control.
2. Compare C against D. If they're equal, C measured D.
3. Batch 1000 calls into one op and divide — the only number here that means anything.
4. Repeat the discard trap on `fib(20)` to see where DCE stops.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// square is small and pure, so the compiler can inline it -- and then delete it
// entirely if nobody uses the result.
func square(x int) int { return x * x }

// fib is recursive, so it cannot be inlined and the compiler will not remove it.
func fib(n int) int {
	if n < 2 {
		return n
	}
	return fib(n-1) + fib(n-2)
}

var sink int

// nsPerOp keeps the fractional part that BenchmarkResult.NsPerOp truncates to 0
// -- which is exactly the range this example lives in.
func nsPerOp(r testing.BenchmarkResult) float64 {
	return float64(r.T.Nanoseconds()) / float64(r.N)
}

func main() {
	rng := rand.New(rand.NewPCG(9, 4))
	const batch = 1000
	xs := make([]int, batch)
	for i := range xs {
		xs[i] = rng.IntN(1 << 20)
	}

	// A. The trap: the result is discarded, so the call is deleted.
	discarded := testing.Benchmark(func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			square(i)
		}
	})

	// B. The "classic fix" -- accumulate into a package-level sink.
	sunk := testing.Benchmark(func(b *testing.B) {
		s := 0
		for i := 0; i < b.N; i++ {
			s += square(i)
		}
		sink = s
	})

	// C. b.Loop (Go 1.24+) keeps the call alive without a sink.
	looped := testing.Benchmark(func(b *testing.B) {
		i := 0
		for b.Loop() {
			square(i)
			i++
		}
	})

	// D. THE CONTROL. An empty b.Loop. Whatever this costs, C cannot cost less.
	control := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
		}
	})

	// E. Batch the work so one "op" is 1000 calls, then divide.
	batched := testing.Benchmark(func(b *testing.B) {
		s := 0
		for b.Loop() {
			for _, v := range xs {
				s += square(v)
			}
		}
		sink = s
	})

	// F. The same discard trap on a function the compiler cannot see through.
	fibDiscarded := testing.Benchmark(func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			fib(20)
		}
	})

	fmt.Println("measuring square(x) -- one multiply:")
	fmt.Printf("  A. discarded, b.N loop      %9.4f ns/op  (%d iters)\n", nsPerOp(discarded), discarded.N)
	fmt.Printf("  B. summed into a sink       %9.4f ns/op  (%d iters)\n", nsPerOp(sunk), sunk.N)
	fmt.Printf("  C. b.Loop                   %9.4f ns/op  (%d iters)\n", nsPerOp(looped), looped.N)
	fmt.Printf("  D. b.Loop, EMPTY BODY       %9.4f ns/op  <- the control\n", nsPerOp(control))
	fmt.Printf("  E. %d per op, divided     %9.4f ns/call\n", batch, nsPerOp(batched)/batch)

	fmt.Println()
	fmt.Println("what actually happened:")
	fmt.Println("  A and B were both optimized away -- the sink did NOT save B.")
	fmt.Printf("  C (%.2f) is indistinguishable from D (%.2f): it measured the harness.\n", nsPerOp(looped), nsPerOp(control))
	fmt.Println("  only E measured square, by making one op big enough to see.")

	fmt.Println()
	fmt.Printf("but DCE is selective -- fib(20) discarded in a b.N loop: %.0f ns/op\n", nsPerOp(fibDiscarded))
	fmt.Println("recursive, non-inlinable, not provably pure -> the compiler leaves it alone.")

	fmt.Println()
	fmt.Println("rules: always run an empty control; below ~2 ns/op, batch and divide.")
}
```

**Sample output** (values vary; the A≈B≪C≈D relationship is the point):

```
measuring square(x) -- one multiply:
  A. discarded, b.N loop         0.2325 ns/op  (1000000000 iters)
  B. summed into a sink          0.2604 ns/op  (1000000000 iters)
  C. b.Loop                      1.6206 ns/op  (742929212 iters)
  D. b.Loop, EMPTY BODY          1.6042 ns/op  <- the control
  E. 1000 per op, divided        1.0504 ns/call

what actually happened:
  A and B were both optimized away -- the sink did NOT save B.
  C (1.62) is indistinguishable from D (1.60): it measured the harness.
  only E measured square, by making one op big enough to see.

but DCE is selective -- fib(20) discarded in a b.N loop: 14676 ns/op
recursive, non-inlinable, not provably pure -> the compiler leaves it alone.

rules: always run an empty control; below ~2 ns/op, batch and divide.
```

The tell in A and B is the iteration count: **1,000,000,000** is `testing`'s hard cap. A benchmark
that hits the cap and still reports a fraction of a nanosecond ran an empty loop.

**Complexity:** not about complexity at all — this is about whether your *measurement* of one is real. Revisited in depth at [42](../../42-benchmarking-profiling.md).

---

## 13. `[]T` vs `[]*T`: the pointer tax

`🔴 hard` · *Cache locality / GC scanning*

Two loops, both O(n), both summing a million points. One walks contiguous memory; the other follows a
million shuffled pointers. Same complexity, **2.7× the time** — and then a second bill arrives that
Big-O never mentions: a GC cycle with a million live pointers took **~59× longer** than with a
million pointer-free values, because the collector must trace every one of them, on every cycle.

Note the structure of the experiment: each phase builds its data **inside its own function**, so the
other structure is unreachable when the GC is timed. Measuring both while both are alive would time
the same heap twice and show no difference at all.

**Steps:**

1. Build `[]point` in one scope, benchmark the sum, time 10 GC cycles, return.
2. `runtime.GC()` to drop phase 1, then repeat with `[]*point` — shuffled, because a real structure isn't laid out in allocation order.
3. Compare both the sum time and the GC time.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
	"time"
)

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

// avgGC times a full GC cycle, averaged over n runs. Whatever is reachable when
// this runs is what the collector has to walk.
func avgGC(n int) time.Duration {
	runtime.GC() // warm up: don't let the first, larger cycle skew the average
	start := time.Now()
	for i := 0; i < n; i++ {
		runtime.GC()
	}
	return time.Since(start) / time.Duration(n)
}

// Each phase builds its structure in its own scope, so that by the time the
// other phase runs, this one is unreachable and the GC numbers are comparable.

func measureValues(n int) (testing.BenchmarkResult, time.Duration) {
	values := make([]point, n)
	for i := range values {
		values[i] = point{x: i, y: i}
	}
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			sumValues(values)
		}
	})
	gc := avgGC(10)
	runtime.KeepAlive(values)
	return res, gc
}

func measurePointers(n int) (testing.BenchmarkResult, time.Duration) {
	pointers := make([]*point, n)
	for i := range pointers {
		pointers[i] = &point{x: i, y: i}
	}
	// Shuffle: a structure built up over time is not laid out in allocation
	// order, and the CPU prefetcher can no longer guess what comes next.
	rng := rand.New(rand.NewPCG(3, 5))
	rng.Shuffle(n, func(i, j int) { pointers[i], pointers[j] = pointers[j], pointers[i] })

	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			sumPointers(pointers)
		}
	})
	gc := avgGC(10)
	runtime.KeepAlive(pointers)
	return res, gc
}

func main() {
	const n = 1_000_000

	valRes, valGC := measureValues(n)
	runtime.GC() // drop phase 1 before phase 2 allocates
	ptrRes, ptrGC := measurePointers(n)

	fmt.Printf("sum of %d points\n\n", n)
	fmt.Printf("[]point   (contiguous)  %10d ns/op  %6.2f ns/element\n", valRes.NsPerOp(), float64(valRes.NsPerOp())/n)
	fmt.Printf("[]*point  (chased)      %10d ns/op  %6.2f ns/element  %.1fx slower\n",
		ptrRes.NsPerOp(), float64(ptrRes.NsPerOp())/n, float64(ptrRes.NsPerOp())/float64(valRes.NsPerOp()))

	fmt.Printf("\nGC cycle, 1M values live   %12v   (0 pointers to scan)\n", valGC.Round(time.Microsecond))
	fmt.Printf("GC cycle, 1M pointers live %12v   (%.1fx more work)\n",
		ptrGC.Round(time.Microsecond), float64(ptrGC)/float64(valGC))

	fmt.Println()
	fmt.Println("both loops are O(n). one walks memory, the other chases it.")
	fmt.Println("and every pointer is a second tax: the GC must trace it on every cycle.")
	fmt.Println("in Go, choosing []T over []*T is an algorithmic decision, not a style one.")
}
```

**Sample output** (timings vary; the ratios are stable):

```
sum of 1000000 points

[]point   (contiguous)      307238 ns/op    0.31 ns/element
[]*point  (chased)          817159 ns/op    0.82 ns/element  2.7x slower

GC cycle, 1M values live           64µs   (0 pointers to scan)
GC cycle, 1M pointers live      3.753ms   (58.8x more work)

both loops are O(n). one walks memory, the other chases it.
and every pointer is a second tax: the GC must trace it on every cycle.
in Go, choosing []T over []*T is an algorithmic decision, not a style one.
```

This is the reason Part 2 keeps saying "default to a slice", and the reason a linked list — which is
*nothing but* pointer chasing — loses to a slice far more often than the complexity table suggests.

**Complexity:** both O(n) time, O(n) space · the difference is entirely in the constant — and in GC work the notation doesn't model at all. Full treatment at [04](../../04-go-memory-model.md).

---

## 14. The brute-force oracle

`🔴 hard` · *Differential testing*

The pattern that verifies every clever implementation in the rest of this plan. Write the obviously
correct, obviously slow version. Write the fast one. Feed both the **same small random inputs** and
assert they agree. A deliberately broken two-sum — one that lets an element pair with itself — is
caught here in **26 trials**, complete with a minimal counterexample you can paste into a unit test.

Note the design choice: the oracle compares *existence*, not indices. Two correct implementations may
legitimately return different valid pairs, so compare a property both must agree on.

**Steps:**

1. Write the O(n²) reference — it only has to be right.
2. Write the O(n) candidate, and a variant with a realistic off-by-one bug.
3. Run both against the oracle on thousands of tiny random inputs (small n and a small value range make collisions — and therefore bugs — frequent).

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

// The oracle: obviously correct, obviously slow. O(n^2).
func hasPairBrute(xs []int, target int) bool {
	for i := 0; i < len(xs); i++ {
		for j := i + 1; j < len(xs); j++ {
			if xs[i]+xs[j] == target {
				return true
			}
		}
	}
	return false
}

// The candidate: fast, and correct. O(n).
func hasPairFast(xs []int, target int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		if _, ok := seen[target-v]; ok {
			return true
		}
		seen[v] = struct{}{}
	}
	return false
}

// The same idea with one bug: it records the value BEFORE looking for its
// complement, so an element can pair with itself.
func hasPairBuggy(xs []int, target int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		seen[v] = struct{}{}
		if _, ok := seen[target-v]; ok {
			return true
		}
	}
	return false
}

func check(name string, candidate func([]int, int) bool) {
	rng := rand.New(rand.NewPCG(42, 7))
	const trials = 200_000

	for t := 0; t < trials; t++ {
		xs := make([]int, rng.IntN(7))
		for i := range xs {
			xs[i] = rng.IntN(11) - 5
		}
		target := rng.IntN(11) - 5

		want := hasPairBrute(xs, target)
		got := candidate(xs, target)
		if want != got {
			fmt.Printf("%-12s FAIL after %d trials\n", name, t+1)
			fmt.Printf("             xs=%v target=%d -> oracle=%v candidate=%v\n", xs, target, want, got)
			return
		}
	}
	fmt.Printf("%-12s ok  (%d random trials agree with the oracle)\n", name, trials)
}

func main() {
	check("fast", hasPairFast)
	check("buggy", hasPairBuggy)

	fmt.Println()
	fmt.Println("small random inputs + a slow reference = the cheapest correctness proof there is")
	fmt.Println("this pattern verifies every clever implementation in the rest of the plan")
}
```

**Output** (deterministic — the PCG seed is fixed):

```
fast         ok  (200000 random trials agree with the oracle)
buggy        FAIL after 26 trials
             xs=[5 3 2] target=4 -> oracle=false candidate=true

small random inputs + a slow reference = the cheapest correctness proof there is
this pattern verifies every clever implementation in the rest of the plan
```

Read the counterexample: `[5 3 2]` with target 4 has no valid pair, but the buggy version inserts `2`,
then looks for `4-2 = 2`, finds the copy it just inserted, and reports success. **Small inputs are a
feature** — a failure on `n=3` is a failure you can debug by hand.

**Complexity:** oracle O(n²) · candidate O(n) · the harness is O(trials · n²), and still finishes in under a second because n ≤ 6. Full treatment, including `go test -fuzz`, at [43](../../43-testing-algorithms.md).

---

> That's the tier. Back to the [index](README.md), or on to [02 — Environment & Toolkit](../../02-environment-setup.md),
> which turns examples 7, 8, 12 and 14 into a reusable harness.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
