# Step 02 — Environment & Toolkit · 🟡 Medium

Examples **7–11**: the empty control, the size sweep, `benchstat`, `-race`, and fuzzing.

These are the four tools that find what a table-driven test cannot: a benchmark that measured nothing,
a complexity you assumed, a race that hides behind a passing test, and an input you never imagined.

> ⚠️ Examples 7–9 print timings — **sample output**, produced on an Apple M4 with Go 1.26.3. Example
> 10's race report contains addresses and goroutine IDs that change every run. Example 11's fuzzing
> is nondeterministic: you'll get a different counterexample and a different corpus filename.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. The empty control

`🟡 medium` · *Benchmark hygiene*

Lesson 01 proved that `b.Loop` has a per-iteration cost of its own. This is that lesson turned into a
habit: **ship an empty benchmark in every package**. It measures nothing on purpose, and it is the
floor below which no other benchmark in that package can be believed.

Watch what happens: `BenchmarkSquare` reports **1.607 ns/op** and the empty control reports
**1.654 ns/op**. The "measurement" of `Square` is *below* the cost of the loop that measures it —
which means it is noise, not a result.

**Steps:**

1. Write `BenchmarkEmptyControl` with an empty `b.Loop` body.
2. Write the benchmark you actually care about.
3. Compare. If they're within noise, batch the work and divide.

**`square.go`**

```go
package algo

// Square is one multiply -- far cheaper than the benchmark harness that measures it.
func Square(x int) int { return x * x }
```

**`square_test.go`**

```go
package algo

import "testing"

// Ship this benchmark in every package. It measures nothing, on purpose: it is
// the floor below which no other benchmark in this file can be believed.
func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// Looks like it measures Square. Compare it to the control before believing it.
func BenchmarkSquare(b *testing.B) {
	i := 0
	for b.Loop() {
		Square(i)
		i++
	}
}

// The fix: make one "op" big enough that the harness cost is noise. Divide the
// reported ns/op by 1000 yourself to get the per-call number.
func BenchmarkSquareBatch1000(b *testing.B) {
	xs := make([]int, 1000)
	for i := range xs {
		xs[i] = i
	}
	s := 0
	for b.Loop() {
		for _, v := range xs {
			s += Square(v)
		}
	}
	// Keep the accumulated value observable so the loop cannot be deleted.
	if s == 0 {
		b.Fatal("unreachable, but the compiler does not know that")
	}
}
```

**Run it:**

```bash
go test -bench=. -run='^$' ./ex07
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex07
cpu: Apple M4
BenchmarkEmptyControl-10       	635776324	         1.654 ns/op
BenchmarkSquare-10             	747192772	         1.607 ns/op
BenchmarkSquareBatch1000-10    	 1548980	       774.7 ns/op
PASS
ok  	l02/ex07	3.973s
```

`774.7 / 1000 = 0.77 ns` per call — the only real number in the table, and less than half of what the
naive benchmark "measured".

**Takeaway:** any benchmark within noise of the empty control is a benchmark of the empty control. Batch until the work dominates.

---

## 8. The size sweep

`🟡 medium` · *Sub-benchmarks / reading complexity off a benchmark*

A single ns/op number tells you nothing about an algorithm — complexity is a statement about *growth*,
so you have to vary n to see it. `b.Run` makes sub-benchmarks; the naming convention
`name/n=1000` keeps the output readable and greppable.

Read the two columns below and you can classify both algorithms without looking at their source.

**Steps:**

1. Loop over sizes on the outside, `b.Run` on the inside.
2. Search for the **last** element, so the linear scan is measured at its worst case.
3. Compare each row to the one above: what does 8× the input cost?

**`search.go`**

```go
package algo

// LinearSearch scans every element: O(n).
func LinearSearch(xs []int, target int) int {
	for i, v := range xs {
		if v == target {
			return i
		}
	}
	return -1
}

// BinarySearch halves the range each step: O(log n). xs must be sorted.
func BinarySearch(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2 // not (lo+hi)/2 -- that can overflow
		switch {
		case xs[mid] == target:
			return mid
		case xs[mid] < target:
			lo = mid + 1
		default:
			hi = mid - 1
		}
	}
	return -1
}
```

**`search_test.go`**

```go
package algo

import (
	"fmt"
	"testing"
)

// One benchmark function, one sub-benchmark per (algorithm, size) pair. The
// sweep is the point: a single ns/op number tells you nothing about growth.
func BenchmarkSearch(b *testing.B) {
	for _, n := range []int{64, 512, 4096, 32768, 262144} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}
		target := (n - 1) * 2 // the last element: worst case for the linear scan

		b.Run(fmt.Sprintf("linear/n=%d", n), func(b *testing.B) {
			for b.Loop() {
				LinearSearch(xs, target)
			}
		})
		b.Run(fmt.Sprintf("binary/n=%d", n), func(b *testing.B) {
			for b.Loop() {
				BinarySearch(xs, target)
			}
		})
	}
}
```

**Run it:**

```bash
go test -bench=. -run='^$' ./ex08
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex08
cpu: Apple M4
BenchmarkSearch/linear/n=64-10         	57249464	        20.73 ns/op
BenchmarkSearch/binary/n=64-10         	320650563	         3.750 ns/op
BenchmarkSearch/linear/n=512-10        	 9574267	       125.6 ns/op
BenchmarkSearch/binary/n=512-10        	192973693	         6.236 ns/op
BenchmarkSearch/linear/n=4096-10       	 1266687	       946.3 ns/op
BenchmarkSearch/binary/n=4096-10       	133159178	         9.061 ns/op
BenchmarkSearch/linear/n=32768-10      	  162292	      7581 ns/op
BenchmarkSearch/binary/n=32768-10      	100000000	        11.36 ns/op
BenchmarkSearch/linear/n=262144-10     	   19742	     60921 ns/op
BenchmarkSearch/binary/n=262144-10     	80904103	        14.66 ns/op
```

Every row is 8× the n of the row two above it:

- **linear:** 20.73 → 125.6 → 946.3 → 7581 → 60921. Each step is ~8× the time. That is O(n).
- **binary:** 3.75 → 6.24 → 9.06 → 11.36 → 14.66. Each step adds a flat **~3 ns**, because 8× the input is 3 more halvings. That is O(log n).

At n=262144 the gap is **4155×**, and it keeps widening forever.

**Takeaway:** you can measure a complexity class. Sweep n by a constant factor and look at what each step costs — multiplicative for O(n), additive for O(log n).

---

## 9. Before and after, with statistics

`🟡 medium` · *benchstat + build tags*

Two ns/op numbers are not a comparison — benchmarks are noisy, and "it looks faster" is how people
ship regressions. `benchstat` runs the statistics: median, variation, percentage delta, and a p-value
saying whether the difference is real.

The mechanism worth stealing here is the **build tag**. Both versions of `Grow` have the same name in
the same package, selected at build time, so there is one benchmark name for benchstat to match on —
and the test suite proves both versions are still correct.

**Install it once:**

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

**Steps:**

1. Put the two implementations in tag-guarded files (`!prealloc` and `prealloc`).
2. Run each `-count=10` into its own file.
3. Compare with `benchstat old.txt new.txt`.

**`grow_nil.go`**

```go
//go:build !prealloc

package algo

// Grow builds the first n integers without telling append how many are coming.
// This is the "before" version -- build it with no tags.
func Grow(n int) []int {
	var out []int
	for i := 0; i < n; i++ {
		out = append(out, i)
	}
	return out
}
```

**`grow_prealloc.go`**

```go
//go:build prealloc

package algo

// Grow builds the first n integers, reserving the capacity up front.
// This is the "after" version -- build it with -tags=prealloc.
func Grow(n int) []int {
	out := make([]int, 0, n)
	for i := 0; i < n; i++ {
		out = append(out, i)
	}
	return out
}
```

**`grow_test.go`**

```go
package algo

import "testing"

// Both builds must pass this: the optimization is only allowed to change speed.
func TestGrow(t *testing.T) {
	got := Grow(5)
	if len(got) != 5 {
		t.Fatalf("len = %d, want 5", len(got))
	}
	for i, v := range got {
		if v != i {
			t.Errorf("got[%d] = %d, want %d", i, v, i)
		}
	}
}

// One benchmark, one name -- benchstat matches the two runs by this name.
func BenchmarkGrow(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		Grow(10000)
	}
}
```

**Run it:**

```bash
go test -bench=Grow -benchmem -count=10 -run='^$' ./ex09 > old.txt
go test -tags=prealloc -bench=Grow -benchmem -count=10 -run='^$' ./ex09 > new.txt
benchstat old.txt new.txt
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex09
cpu: Apple M4
        │   old.txt    │               new.txt               │
        │    sec/op    │   sec/op     vs base                │
Grow-10   25.471µ ± 2%   5.647µ ± 1%  -77.83% (p=0.000 n=10)

        │    old.txt    │               new.txt                │
        │     B/op      │     B/op      vs base                │
Grow-10   349.24Ki ± 0%   80.00Ki ± 0%  -77.09% (p=0.000 n=10)

        │   old.txt   │              new.txt               │
        │  allocs/op  │ allocs/op   vs base                │
Grow-10   19.000 ± 0%   1.000 ± 0%  -94.74% (p=0.000 n=10)
```

Reading it: `± 2%` is the variation across the 10 runs, `-77.83%` is the change, and `p=0.000` says
the difference is statistically significant. If benchstat prints `~` instead of a percentage, the two
sides are indistinguishable and your "optimization" did nothing.

**Takeaway:** `-count=10` and benchstat, or it didn't happen. One knowing `make([]T, 0, n)` cut 78% of the time and 95% of the allocations.

---

## 10. The race a passing test cannot see

`🟡 medium` · *go test -race*

The function below reduces into a shared variable from four goroutines. It is unambiguously wrong.
Its test **passed 5 times out of 5** — the race window is small, and the answer usually comes out
right anyway. That is what makes concurrency bugs so expensive: they pass CI for months, then corrupt
data under production load.

`go test -race` finds it on the first run, and names the two lines involved.

**Steps:**

1. Run the test normally, several times. Watch it pass.
2. Run it with `-race`.
3. Read the fix: no mutex — each worker gets a private accumulator, combined once at the end.

**`parallel.go`**

```go
package algo

import "sync"

// MaxParallelRacy splits xs across workers and reduces into a shared variable.
// The reduction is unsynchronized: two goroutines can read `best`, both decide
// they are larger, and one write is lost.
//
// It usually returns the right answer anyway -- which is exactly the problem.
func MaxParallelRacy(xs []int, workers int) int {
	best := xs[0]
	chunk := (len(xs) + workers - 1) / workers

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		lo := w * chunk
		hi := min(lo+chunk, len(xs))
		if lo >= hi {
			continue
		}
		wg.Add(1)
		go func(part []int) {
			defer wg.Done()
			for _, v := range part {
				if v > best { // <-- DATA RACE: read and write of a shared int
					best = v
				}
			}
		}(xs[lo:hi])
	}
	wg.Wait()
	return best
}

// MaxParallel is the fix: each worker reduces its own chunk into a private
// variable, and the results are combined once, in one goroutine. No shared
// mutable state means no lock and no race.
func MaxParallel(xs []int, workers int) int {
	chunk := (len(xs) + workers - 1) / workers
	partials := make([]int, workers)

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		lo := w * chunk
		hi := min(lo+chunk, len(xs))
		if lo >= hi {
			partials[w] = xs[0]
			continue
		}
		wg.Add(1)
		go func(w int, part []int) {
			defer wg.Done()
			local := part[0] // only this goroutine touches partials[w]
			for _, v := range part[1:] {
				if v > local {
					local = v
				}
			}
			partials[w] = local
		}(w, xs[lo:hi])
	}
	wg.Wait()

	best := partials[0]
	for _, v := range partials[1:] {
		if v > best {
			best = v
		}
	}
	return best
}
```

**`parallel_test.go`**

```go
package algo

import "testing"

func input() []int {
	xs := make([]int, 10000)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

// This test passes. Run it with -race and it fails anyway.
func TestMaxParallelRacy(t *testing.T) {
	if got := MaxParallelRacy(input(), 4); got != 9999 {
		t.Errorf("MaxParallelRacy = %d, want 9999", got)
	}
}

func TestMaxParallel(t *testing.T) {
	if got := MaxParallel(input(), 4); got != 9999 {
		t.Errorf("MaxParallel = %d, want 9999", got)
	}
}
```

**Run it normally, five times:**

```bash
for i in 1 2 3 4 5; do go test -count=1 ./ex10 | tail -1; done
```

```
ok  	l02/ex10	0.341s
ok  	l02/ex10	0.136s
ok  	l02/ex10	0.108s
ok  	l02/ex10	0.106s
ok  	l02/ex10	0.106s
```

**Now with the race detector:**

```bash
go test -race -count=1 ./ex10
```

**Sample output** (trimmed — the real report also prints full goroutine creation stacks; addresses and
goroutine IDs differ every run):

```
==================
WARNING: DATA RACE
Read at 0x00c0000b6178 by goroutine 10:
  l02/ex10.MaxParallelRacy.func1()
      /tmp/dsa-t/ex10/parallel.go:25 +0xc4

Previous write at 0x00c0000b6178 by goroutine 11:
  l02/ex10.MaxParallelRacy.func1()
      /tmp/dsa-t/ex10/parallel.go:26 +0xdc
...
==================
--- FAIL: TestMaxParallelRacy (0.00s)
    testing.go:1712: race detected during execution of test
FAIL
FAIL	l02/ex10	0.301s
FAIL
```

Line 25 is the `if v > best` read; line 26 is the `best = v` write. The detector names both.

**Takeaway:** for concurrent code a passing test is not evidence. `-race` is the evidence. It costs ~10× time and memory, so it's a gate — never a benchmark mode.

---

## 11. Fuzzing writes your regression test

`🟡 medium` · *go test -fuzz*

Back to the buggy binary search from example 5 — but this time nobody tells the test where to look.
A fuzz target takes random bytes, shapes them into a valid input, and compares against an oracle. The
fuzzer explores; coverage guidance steers it toward new code paths.

It found the bug in **0.02 seconds**, automatically **minimized** a 53-byte input down to a
3-element slice, and wrote the failing case to `testdata/` — where it becomes a permanent regression
test that runs on every plain `go test` from now on.

**Steps:**

1. Add seeds with `f.Add` — valid, representative, and *passing*, so the fuzzer has to do the work.
2. In `f.Fuzz`, shape the raw bytes into something that satisfies your function's precondition (here: sorted and deduplicated).
3. Compare against the oracle and fail loudly.
4. Run with `-fuzz` and a time limit, then look at what it wrote.

**`search.go`**

```go
package algo

// Search returns the index of target in the sorted slice xs, or -1.
// It carries the same bug as example 5 -- this time nobody tells the fuzzer
// where to look.
func Search(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		switch {
		case xs[mid] == target:
			return mid
		case xs[mid] < target:
			lo = mid + 1
		default:
			hi = mid - 1
		}
	}
	return -1
}
```

**`search_test.go`**

```go
package algo

import (
	"slices"
	"testing"
)

func linearSearch(xs []int, target int) int {
	for i, v := range xs {
		if v == target {
			return i
		}
	}
	return -1
}

// sortedSet turns arbitrary fuzzer bytes into a valid input for Search: sorted,
// and deduplicated so that "the" correct index is unambiguous.
//
// Shaping the input like this is most of the work in writing a fuzz target --
// the fuzzer supplies chaos, and you narrow it to your function's precondition.
func sortedSet(data []byte) []int {
	seen := make(map[int]struct{}, len(data))
	xs := make([]int, 0, len(data))
	for _, b := range data {
		v := int(b)
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		xs = append(xs, v)
	}
	slices.Sort(xs)
	return xs
}

func FuzzSearch(f *testing.F) {
	// Seeds: valid, representative, and all passing. The fuzzer mutates from here.
	f.Add([]byte{2, 4, 6, 8}, byte(4)) // hit on the first probe
	f.Add([]byte{1, 2, 3}, byte(9))    // absent, above the range
	f.Add([]byte{5, 5, 5}, byte(0))    // absent, below the range

	f.Fuzz(func(t *testing.T, data []byte, target byte) {
		xs := sortedSet(data)

		want := linearSearch(xs, int(target))
		got := Search(xs, int(target))
		if got != want {
			t.Fatalf("Search(%v, %d) = %d, oracle says %d", xs, target, got, want)
		}
	})
}
```

**First, confirm the seeds pass** (a plain `go test` runs seeds only, no fuzzing):

```bash
go test ./ex11
```

```
ok  	l02/ex11	0.438s
```

**Now let it hunt:**

```bash
go test -fuzz=FuzzSearch -fuzztime=30s ./ex11
```

**Sample output** (your counterexample and filename will differ):

```
fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 10 workers
fuzz: minimizing 53-byte failing input file
fuzz: elapsed: 0s, minimizing
--- FAIL: FuzzSearch (0.02s)
    --- FAIL: FuzzSearch (0.00s)
        search_test.go:49: Search([4 48 49], 4) = -1, oracle says 0

    Failing input written to testdata/fuzz/FuzzSearch/d8dadd70ed1a164a
    To re-run:
    go test -run=FuzzSearch/d8dadd70ed1a164a
FAIL
```

**Look at what it wrote:**

```bash
cat ex11/testdata/fuzz/FuzzSearch/d8dadd70ed1a164a
```

```
go test fuzz v1
[]byte("0\x041")
byte('\x04')
```

Those bytes are 48, 4, 49 — which `sortedSet` turns into `[4 48 49]`, with target 4. The first element
again: exactly the case example 5's table missed.

**And now the plain test suite reproduces it forever:**

```bash
go test ./ex11
```

```
--- FAIL: FuzzSearch (0.00s)
    --- FAIL: FuzzSearch/d8dadd70ed1a164a (0.00s)
        search_test.go:49: Search([4 48 49], 4) = -1, oracle says 0
FAIL
FAIL	l02/ex11	0.139s
FAIL
```

**Takeaway:** fuzzing is an oracle test where you don't have to invent the inputs. **Commit `testdata/fuzz/`** — those files are the regression suite the fuzzer earned for you.

---

> Next tier: [🔴 hard](3-hard.md) — the reusable harness.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
