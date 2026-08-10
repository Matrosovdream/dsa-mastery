# Step 03 — Complexity Analysis · 🔴 Hard

Examples **12–16**: the master theorem, quadratics hiding in single loops, and measuring complexity
for real with the lesson-02 harness.

> ⚠️ **Two shapes in this tier.** Examples 12 and 13 are `go run` programs. Examples **14, 15 and 16
> are `go test` packages** — folders with two files each, run with `go test -v`. Each says which.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The master theorem

`🔴 hard` · *Naming the three tree shapes* · `go run`

Example 11 drew three recursion trees and found three shapes. The master theorem is just those three
shapes given numbers: compare `d` (work at the top) against `log_b(a)` (how fast subproblems
multiply). It also shows *why* Karatsuba and Strassen matter — they win purely by making **fewer
recursive calls**, which moves `log_b(a)` down.

**Steps:**

1. Compute `log_b(a)` and compare it with `d`.
2. Apply it to seven classic recurrences.
3. Read the three recurrences it cannot solve, and why.

```go
package main

import (
	"fmt"
	"math"
)

// The master theorem solves T(n) = a*T(n/b) + Theta(n^d) by comparing the work
// created by the recursion (n^log_b(a)) against the work done at the top (n^d).
//
//   d <  log_b(a)  -> Case 1: the LEAVES dominate    -> Theta(n^log_b(a))
//   d == log_b(a)  -> Case 2: every level is equal   -> Theta(n^d * log n)
//   d >  log_b(a)  -> Case 3: the ROOT dominates     -> Theta(n^d)
//
// It is not a magic formula: it is the three shapes a recursion tree can have,
// which you just drew by hand in example 11.

// pow formats n^e the way a human writes it: n^0 is 1, n^1 is n, and an
// exponent that is not a whole number keeps two decimals.
func pow(e float64) string {
	const eps = 1e-9
	switch {
	case math.Abs(e) <= eps:
		return "1"
	case math.Abs(e-1) <= eps:
		return "n"
	case math.Abs(e-math.Round(e)) <= eps:
		return fmt.Sprintf("n^%d", int(math.Round(e)))
	default:
		return fmt.Sprintf("n^%.2f", e)
	}
}

func solve(a, b int, d float64) (which string, result string) {
	crit := math.Log(float64(a)) / math.Log(float64(b)) // log_b(a)

	const eps = 1e-9
	switch {
	case d < crit-eps:
		return "1 (leaves win)", fmt.Sprintf("Theta(%s)", pow(crit))
	case math.Abs(d-crit) <= eps:
		if math.Abs(d) <= eps {
			return "2 (all levels equal)", "Theta(log n)"
		}
		return "2 (all levels equal)", fmt.Sprintf("Theta(%s log n)", pow(d))
	default:
		return "3 (root wins)", fmt.Sprintf("Theta(%s)", pow(d))
	}
}

func main() {
	cases := []struct {
		name string
		a    int
		b    int
		d    float64
		note string
	}{
		{"binary search", 1, 2, 0, "T(n) = T(n/2) + O(1)"},
		{"tree traversal", 2, 2, 0, "T(n) = 2T(n/2) + O(1)"},
		{"merge sort", 2, 2, 1, "T(n) = 2T(n/2) + O(n)"},
		{"Karatsuba multiply", 3, 2, 1, "T(n) = 3T(n/2) + O(n)"},
		{"Strassen matrix", 7, 2, 2, "T(n) = 7T(n/2) + O(n^2)"},
		{"naive block matmul", 8, 2, 2, "T(n) = 8T(n/2) + O(n^2)"},
		{"one recursive call, linear work", 1, 2, 1, "T(n) = T(n/2) + O(n)"},
	}

	fmt.Printf("%-32s %-26s %8s %-22s %s\n", "algorithm", "recurrence", "log_b(a)", "case", "solution")
	for _, c := range cases {
		crit := math.Log(float64(c.a)) / math.Log(float64(c.b))
		which, result := solve(c.a, c.b, c.d)
		fmt.Printf("%-32s %-26s %8.2f %-22s %s\n", c.name, c.note, crit, which, result)
	}

	fmt.Println()
	fmt.Println("read the middle column as 'how fast the subproblems multiply'.")
	fmt.Println("Karatsuba beats naive multiplication (n^1.58 vs n^2) purely by making")
	fmt.Println("3 recursive calls instead of 4 -- log2(3)=1.58 against log2(4)=2.")
	fmt.Println("Strassen does the same to matrices: 7 calls instead of 8, n^2.81 vs n^3.")

	fmt.Println()
	fmt.Println("what the master theorem does NOT cover:")
	fmt.Println("  T(n) = 2T(n/2) + n/log n   -- f(n) sits in the gap between the cases")
	fmt.Println("  T(n) = T(n-1) + n          -- not a/b-shaped at all (see example 11)")
	fmt.Println("  T(n) = 2T(n/2) + 2^n       -- f grows too fast for case 3's regularity condition")
	fmt.Println("for those, draw the tree or expand the recurrence by hand.")
}
```

**Run it:**

```bash
go run .
```

**Output:**

```
algorithm                        recurrence                 log_b(a) case                   solution
binary search                    T(n) = T(n/2) + O(1)           0.00 2 (all levels equal)   Theta(log n)
tree traversal                   T(n) = 2T(n/2) + O(1)          1.00 1 (leaves win)         Theta(n)
merge sort                       T(n) = 2T(n/2) + O(n)          1.00 2 (all levels equal)   Theta(n log n)
Karatsuba multiply               T(n) = 3T(n/2) + O(n)          1.58 1 (leaves win)         Theta(n^1.58)
Strassen matrix                  T(n) = 7T(n/2) + O(n^2)        2.81 1 (leaves win)         Theta(n^2.81)
naive block matmul               T(n) = 8T(n/2) + O(n^2)        3.00 1 (leaves win)         Theta(n^3)
one recursive call, linear work  T(n) = T(n/2) + O(n)           0.00 3 (root wins)          Theta(n)

read the middle column as 'how fast the subproblems multiply'.
Karatsuba beats naive multiplication (n^1.58 vs n^2) purely by making
3 recursive calls instead of 4 -- log2(3)=1.58 against log2(4)=2.
Strassen does the same to matrices: 7 calls instead of 8, n^2.81 vs n^3.

what the master theorem does NOT cover:
  T(n) = 2T(n/2) + n/log n   -- f(n) sits in the gap between the cases
  T(n) = T(n-1) + n          -- not a/b-shaped at all (see example 11)
  T(n) = 2T(n/2) + 2^n       -- f grows too fast for case 3's regularity condition
for those, draw the tree or expand the recurrence by hand.
```

Compare rows 5 and 6: **7 recursive calls instead of 8** takes matrix multiplication from n³ to
n^2.81. That is the entire content of Strassen's algorithm, and the master theorem prices it in one line.

**Complexity:** the tool itself is Θ(1) · it prices any `a·T(n/b) + Θ(n^d)` recurrence, which covers most divide-and-conquer in [28 — Divide & Conquer](../../28-divide-conquer.md)

---

## 13. Quadratic hiding in a single loop

`🔴 hard` · *The O(n) operation inside the O(n) loop* · `go run`

Four functions, each a single loop — Θ(n) at a glance. Two of them are Θ(n²), because the operation
*inside* the loop is itself Θ(n). This is the most common real-world complexity bug there is, and it
never looks like one.

The diagnostic here is **work per element**: if that number grows with n, the loop is not linear.

**Steps:**

1. Count the bytes/elements copied by each loop body.
2. Divide by n and compare across three sizes — growing or bounded?
3. Confirm on the clock at n=200,000.

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

// Every function here looks like a single loop -- O(n) at a glance. Two of them
// are O(n^2), because the operation INSIDE the loop is itself O(n).
//
// bytesCopied / elemsCopied count the hidden work exactly, no clock involved.

// concatNaive: strings are immutable, so s += x allocates a new string and
// copies everything so far. Copying i bytes on iteration i totals n^2/2.
func concatNaive(n int) (result string, bytesCopied int) {
	s := ""
	for i := 0; i < n; i++ {
		bytesCopied += len(s) // the copy s += "x" performs
		s += "x"
	}
	return s, bytesCopied
}

// concatBuilder: amortized growth, exactly like append. O(n) total.
func concatBuilder(n int) (result string, bytesCopied int) {
	var b strings.Builder
	prev := 0
	for i := 0; i < n; i++ {
		before := b.Cap()
		b.WriteByte('x')
		if b.Cap() != before {
			bytesCopied += prev // only the resizes copy
		}
		prev = b.Len()
	}
	return b.String(), bytesCopied
}

// prependNaive: inserting at the front shifts everything. n^2/2 element moves.
func prependNaive(n int) (out []int, elemsCopied int) {
	var xs []int
	for i := 0; i < n; i++ {
		elemsCopied += len(xs)
		xs = append([]int{i}, xs...)
	}
	return xs, elemsCopied
}

// appendThenReverse: build at the cheap end, fix the order once. O(n).
func appendThenReverse(n int) (out []int, elemsCopied int) {
	xs := make([]int, 0, n)
	for i := 0; i < n; i++ {
		xs = append(xs, i) // preallocated: no copies at all
	}
	for i, j := 0, len(xs)-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
		elemsCopied += 2
	}
	return xs, elemsCopied
}

func timeIt(f func()) time.Duration {
	start := time.Now()
	f()
	return time.Since(start)
}

func main() {
	fmt.Println("four one-loop functions. Counting the work INSIDE the loop, as work PER")
	fmt.Println("ELEMENT -- if that column grows with n, the loop is not linear.")
	fmt.Println()
	fmt.Printf("%-22s %14s %14s %14s\n", "function", "n=2000", "n=8000", "n=32000")
	fmt.Printf("%-22s %14s %14s %14s\n", "", "copies/n", "copies/n", "copies/n")

	rows := []struct {
		name string
		f    func(int) int
	}{
		{"s += x", func(n int) int { _, c := concatNaive(n); return c }},
		{"strings.Builder", func(n int) int { _, c := concatBuilder(n); return c }},
		{"append([]T{v}, xs...)", func(n int) int { _, c := prependNaive(n); return c }},
		{"append then reverse", func(n int) int { _, c := appendThenReverse(n); return c }},
	}
	for _, r := range rows {
		perElem := func(n int) float64 { return float64(r.f(n)) / float64(n) }
		fmt.Printf("%-22s %14.2f %14.2f %14.2f\n", r.name, perElem(2000), perElem(8000), perElem(32000))
	}

	fmt.Println()
	fmt.Println("two of these loops are O(n^2), and nothing in their shape says so:")
	fmt.Println("  s += x and prepend    -> copies/n grows in step with n (it IS n/2) -> Theta(n^2)")
	fmt.Println("  Builder and append    -> copies/n stays bounded by a constant       -> Theta(n)")
	fmt.Println()
	fmt.Println("note the Builder's number wobbles (1.7 to 3.7) rather than sitting still.")
	fmt.Println("That is amortized growth: the count depends on where n falls between two")
	fmt.Println("resizes. BOUNDED is the property that matters, not CONSTANT.")

	fmt.Println()
	fmt.Println("the same thing on the clock, at n=200000:")
	const big = 200_000
	fmt.Printf("  s += x                 %12v\n", timeIt(func() { concatNaive(big) }).Round(time.Millisecond))
	fmt.Printf("  strings.Builder        %12v\n", timeIt(func() { concatBuilder(big) }).Round(time.Microsecond))
	fmt.Printf("  append([]T{v}, xs...)  %12v\n", timeIt(func() { prependNaive(big) }).Round(time.Millisecond))
	fmt.Printf("  append then reverse    %12v\n", timeIt(func() { appendThenReverse(big) }).Round(time.Microsecond))

	fmt.Println()
	fmt.Println("the rule from example 6: a call inside a loop is NESTED, so its cost")
	fmt.Println("MULTIPLIES. To analyse a loop you must know the complexity of every")
	fmt.Println("operation in its body -- including the ones that look like one line.")
	fmt.Println()
	fmt.Println("the two fixes are the same fix: work at the END of the structure,")
	fmt.Println("where amortized growth applies, and reorder once at the end if you must.")
}
```

**Run it:**

```bash
go run .
```

**Sample output** (the counts are exact; the four timings vary):

```
four one-loop functions. Counting the work INSIDE the loop, as work PER
ELEMENT -- if that column grows with n, the loop is not linear.

function                       n=2000         n=8000        n=32000
                             copies/n       copies/n       copies/n
s += x                         999.50        3999.50       15999.50
strings.Builder                  1.66           3.10           3.54
append([]T{v}, xs...)          999.50        3999.50       15999.50
append then reverse              1.00           1.00           1.00

two of these loops are O(n^2), and nothing in their shape says so:
  s += x and prepend    -> copies/n grows in step with n (it IS n/2) -> Theta(n^2)
  Builder and append    -> copies/n stays bounded by a constant       -> Theta(n)

note the Builder's number wobbles (1.7 to 3.7) rather than sitting still.
That is amortized growth: the count depends on where n falls between two
resizes. BOUNDED is the property that matters, not CONSTANT.

the same thing on the clock, at n=200000:
  s += x                        964ms
  strings.Builder               425µs
  append([]T{v}, xs...)        6.971s
  append then reverse           116µs
```

The counting table and the clock agree: **999 → 3,999 → 15,999** (work per element growing in step
with n) versus **1.66 → 3.10 → 3.54** (bounded). And the practical cost of getting it wrong is
**6.971 s versus 116 µs** — a factor of 60,000.

**Complexity:** `s += x` and prepend are Θ(n²) total · `strings.Builder` and preallocated append are Θ(n) · both fixes are the same fix: work at the cheap end of the structure

---

## 14. Measuring a complexity class

`🔴 hard` · *The ratio test on real timings* · `go test`

Example 4 ran the ratio test on exact operation counts. This runs it on **measured nanoseconds**,
using `testing.Benchmark` from lesson 02 — and finds the limit of the technique.

The honest result: timings separate classes that differ by a **factor of n** (log vs linear vs
quadratic) but **cannot separate O(n) from O(n log n)**. Doubling n multiplies an n log n cost by
`2·log(2n)/log(n)` — at n=65536 that's **2.125**, a 6% difference that does not survive contact with a
real CPU.

**Steps:**

1. Benchmark four functions of known complexity at four sizes each.
2. Take the **median** ratio (robust against one noisy run).
3. Classify — and note that two classes have to share a band.

**`algo.go`**

```go
package algo

import "slices"

// Four functions whose complexity we already know. The point of the test is to
// REDISCOVER it from measured timings alone.

// SumAll is Theta(n).
func SumAll(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

// Find is Theta(log n) on a sorted slice.
func Find(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo <= hi {
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

// SortInto is Theta(n log n). It sorts in place so that the benchmark measures
// the sort rather than an allocation.
func SortInto(xs []int) {
	slices.Sort(xs)
}

// CountPairs is Theta(n^2).
func CountPairs(xs []int, target int) int {
	n := 0
	for i := 0; i < len(xs); i++ {
		for j := i + 1; j < len(xs); j++ {
			if xs[i]+xs[j] == target {
				n++
			}
		}
	}
	return n
}
```

**`complexity_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"testing"
)

// nsPerOp runs one benchmark and returns its cost in nanoseconds, keeping the
// fractional part that BenchmarkResult.NsPerOp truncates away.
func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// classify turns a measured doubling-ratio into a complexity class.
//
// Note what is NOT in this list: separate bands for O(n) and O(n log n).
// Doubling n multiplies an O(n) cost by 2 and an O(n log n) cost by
// 2*(log(2n)/log(n)) -- which at n=65536 is 2.125. A 6% difference does not
// survive contact with a real CPU. Timings resolve classes that differ by a
// FACTOR of n; they cannot resolve a log factor. (Operation counts can --
// see example 4, where the same two classes came out 2.00 and 2.29.)
func classify(ratio float64) string {
	switch {
	case ratio < 1.5:
		return "O(log n)"
	case ratio < 3.0:
		return "O(n) or O(n log n)"
	case ratio < 5.0:
		return "O(n^2)"
	default:
		return "worse than quadratic"
	}
}

// TestIdentifyComplexity measures each function at four sizes and reads its
// complexity class off the timings alone -- no looking at the source.
func TestIdentifyComplexity(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))

	ascending := func(n int) []int {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		return xs
	}
	shuffled := func(n int) []int {
		xs := ascending(n)
		rng.Shuffle(n, func(i, j int) { xs[i], xs[j] = xs[j], xs[i] })
		return xs
	}

	cases := []struct {
		name  string
		base  int
		build func(n int) func()
		want  string
		truth string
	}{
		{"SumAll", 4096, func(n int) func() {
			xs := ascending(n)
			return func() { SumAll(xs) }
		}, "O(n) or O(n log n)", "O(n)"},

		{"Find", 4096, func(n int) func() {
			xs := ascending(n)
			return func() { Find(xs, n-1) }
		}, "O(log n)", "O(log n)"},

		// Starts at 32768 on purpose: below ~128 KB of data the whole slice fits
		// in a fast cache level, and crossing that boundary produces a ratio
		// spike that has nothing to do with the algorithm.
		{"SortCopy", 32768, func(n int) func() {
			src := shuffled(n)
			buf := make([]int, n)
			return func() { copy(buf, src); SortInto(buf) }
		}, "O(n) or O(n log n)", "O(n log n)"},

		{"CountPairs", 256, func(n int) func() {
			xs := ascending(n)
			return func() { CountPairs(xs, -1) }
		}, "O(n^2)", "O(n^2)"},
	}

	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			sizes := []int{c.base, c.base * 2, c.base * 4, c.base * 8}

			times := make([]float64, len(sizes))
			for i, n := range sizes {
				times[i] = nsPerOp(c.build(n))
			}

			t.Logf("%-11s %14s %10s", "n", "ns/op", "ratio")
			ratios := make([]float64, 0, len(sizes)-1)
			for i, n := range sizes {
				if i == 0 {
					t.Logf("%-11d %14.1f %10s", n, times[i], "-")
					continue
				}
				r := times[i] / times[i-1]
				ratios = append(ratios, r)
				t.Logf("%-11d %14.1f %10.2f", n, times[i], r)
			}

			// The median is more robust than the mean against one noisy run.
			median := ratios[len(ratios)/2]
			got := classify(median)
			t.Logf("median ratio %.2f -> %s   (actually %s)", median, got, c.truth)

			if got != c.want {
				t.Errorf("classified as %s from a ratio of %.2f, expected %s",
					got, median, c.want)
			}
		})
	}
}
```

**Run it:**

```bash
go test -v .
```

**Sample output** (ns/op varies; the ratios are the point):

```
=== RUN   TestIdentifyComplexity/SumAll
    complexity_test.go:99: n                    ns/op      ratio
    complexity_test.go:103: 4096                 969.9          -
    complexity_test.go:108: 8192                1874.4       1.93
    complexity_test.go:108: 16384               3742.5       2.00
    complexity_test.go:108: 32768               7467.6       2.00
    complexity_test.go:114: median ratio 2.00 -> O(n) or O(n log n)   (actually O(n))
=== RUN   TestIdentifyComplexity/Find
    complexity_test.go:103: 4096                   9.0          -
    complexity_test.go:108: 8192                   9.9       1.10
    complexity_test.go:108: 16384                 10.8       1.09
    complexity_test.go:108: 32768                 11.7       1.09
    complexity_test.go:114: median ratio 1.09 -> O(log n)   (actually O(log n))
=== RUN   TestIdentifyComplexity/SortCopy
    complexity_test.go:103: 32768            1103218.8          -
    complexity_test.go:108: 65536            2612202.1       2.37
    complexity_test.go:108: 131072           5723646.0       2.19
    complexity_test.go:108: 262144          12370146.7       2.16
    complexity_test.go:114: median ratio 2.19 -> O(n) or O(n log n)   (actually O(n log n))
=== RUN   TestIdentifyComplexity/CountPairs
    complexity_test.go:103: 256                 8773.5          -
    complexity_test.go:108: 512                32953.5       3.76
    complexity_test.go:108: 1024              125676.1       3.81
    complexity_test.go:108: 2048              488628.6       3.89
    complexity_test.go:114: median ratio 3.81 -> O(n^2)   (actually O(n^2))
--- PASS: TestIdentifyComplexity (18.89s)
ok  	l03/ex14	19.389s
```

Two things worth noticing:

- **The log signal is real but too small to bet on.** `SortCopy`'s ratios (2.37, 2.19, 2.16) do sit consistently above `SumAll`'s (1.93, 2.00, 2.00) — exactly as theory predicts. It just isn't a big enough margin to assert in a test.
- **Cache cliffs produce fake ratios.** An earlier version of this example started the sort sweep at n=4096 and measured a ratio of **4.07** at n=16384 — which classifies as quadratic. At 16384 ints the data crosses 128 KB and falls out of a fast cache level. Sweeping a wide range and taking the median is the defense.

**Complexity:** the measurement is Θ(sizes) benchmarks · the technique resolves Θ(log n) vs Θ(n) vs Θ(n²) reliably, and Θ(n) vs Θ(n log n) not at all

---

## 15. What amortized O(1) costs the unlucky caller

`🔴 hard` · *Tail latency* · `go test`

Example 8 counted the copies. This puts a clock on them. The amortized cost of an append is a couple
of nanoseconds; the batch containing the last resize takes **hundreds of microseconds**. The ratio of
max to median is **~8,600–14,000×**.

There is a measurement lesson here too: timing individual appends **does not work**. An append costs
~1–2 ns and the system clock resolves ~40 ns, so most samples read zero — and a median of zero makes
every ratio infinite. Batching lifts each sample clear of the clock while still isolating the resize.

**Steps:**

1. Time appends in batches of 100 rather than individually, and explain why.
2. Report p50 / p99 / p99.9 / max, with and without reserved capacity.
3. Assert that the growing slice has a tail and that reserving capacity removes it.

**`append.go`**

```go
package algo

import "time"

// BatchLatencies appends n ints and returns one timing sample per `batch`
// appends. If prealloc is set, the capacity is reserved up front and no resize
// ever happens -- same amortized cost, completely different tail.
//
// Why batches? A single append costs ~1-2 ns, and the system clock resolves
// about 40 ns. Timing them one at a time reports mostly zero, and a median of
// zero makes every ratio infinite. Batching lifts each sample well clear of the
// clock's resolution while still isolating the resize into one sample.
func BatchLatencies(n, batch int, prealloc bool) []time.Duration {
	var xs []int
	if prealloc {
		xs = make([]int, 0, n)
	}

	out := make([]time.Duration, 0, n/batch)
	for i := 0; i < n; i += batch {
		start := time.Now()
		for j := 0; j < batch; j++ {
			xs = append(xs, i+j)
		}
		out = append(out, time.Since(start))
	}
	return out
}
```

**`append_test.go`**

```go
package algo

import (
	"slices"
	"testing"
	"time"
)

func percentile(sorted []time.Duration, p float64) time.Duration {
	i := int(p / 100 * float64(len(sorted)-1))
	return sorted[i]
}

// TestAppendTailLatency measures what "amortized O(1)" costs the unlucky caller.
//
// Example 8 counted the copies. This one puts a clock on them: the amortized
// cost is a couple of nanoseconds per append, and the one batch that triggers
// the final resize takes hundreds of MICROseconds, because it moves the entire
// backing array.
func TestAppendTailLatency(t *testing.T) {
	const (
		n     = 1_000_000
		batch = 100
	)

	type result struct {
		name           string
		p50, p99, p999 time.Duration
		max            time.Duration
		total          time.Duration
		tailRatio      float64
		slowBatches    int
	}

	var results []result

	for _, c := range []struct {
		name     string
		prealloc bool
	}{
		{"growing slice", false},
		{"capacity reserved", true},
	} {
		lat := BatchLatencies(n, batch, c.prealloc)

		var total time.Duration
		for _, d := range lat {
			total += d
		}

		sorted := slices.Clone(lat)
		slices.Sort(sorted)
		p50 := percentile(sorted, 50)
		max := sorted[len(sorted)-1]

		slow := 0
		for _, d := range lat {
			if d > 10*p50 {
				slow++
			}
		}

		results = append(results, result{
			name: c.name, p50: p50,
			p99:         percentile(sorted, 99),
			p999:        percentile(sorted, 99.9),
			max:         max,
			total:       total,
			tailRatio:   float64(max) / float64(p50),
			slowBatches: slow,
		})
	}

	t.Logf("one sample = %d appends, %d samples per run\n", batch, n/batch)
	t.Logf("%-20s %10s %10s %10s %12s %10s %12s",
		"", "p50", "p99", "p99.9", "max", "max/p50", ">10x p50")
	for _, r := range results {
		t.Logf("%-20s %10v %10v %10v %12v %9.0fx %12d",
			r.name, r.p50, r.p99, r.p999, r.max, r.tailRatio, r.slowBatches)
	}
	t.Logf("")
	t.Logf("total wall time for %d appends: growing %v, reserved %v",
		n, results[0].total.Round(time.Millisecond), results[1].total.Round(time.Millisecond))

	growing, reserved := results[0], results[1]

	// The point: the growing slice has a tail, and reserving capacity removes it.
	if growing.tailRatio < 50 {
		t.Errorf("expected the growing slice to show a tail of at least 50x, got %.0fx",
			growing.tailRatio)
	}
	if reserved.tailRatio >= growing.tailRatio {
		t.Errorf("reserving capacity should shrink the tail: growing %.0fx, reserved %.0fx",
			growing.tailRatio, reserved.tailRatio)
	}
}
```

**Run it:**

```bash
go test -v .
```

**Sample output** (timings vary; the shape held across five runs):

```
    append_test.go:74: one sample = 100 appends, 10000 samples per run
    append_test.go:76:                     p50        p99      p99.9          max    max/p50     >10x p50
    append_test.go:78: growing slice              83ns      125ns   94.208µs    742.625µs      8947x           55
    append_test.go:78: capacity reserved          83ns    1.583µs    2.333µs     17.958µs       216x          488
    append_test.go:82: total wall time for 1000000 appends: growing 5ms, reserved 1ms
--- PASS: TestAppendTailLatency (0.12s)
ok  	l03/ex15	0.559s
```

Read the growing-slice row across: **p50 83 ns**, **p99 still only 125 ns**, then **p99.9 jumps to
94 µs** and the max is **742 µs**. That cliff between p99 and p99.9 is the exact shape of a tail
latency problem — invisible in an average, invisible at p99, and the thing your users complain about.

Reserving capacity cuts total time ~5× *and* collapses the tail from 8947× to 216×.

**Complexity:** append is Θ(1) amortized either way · the *distribution* is what changes — `make([]T, 0, n)` converts one Θ(n) spike into nothing

---

## 16. A complexity regression gate

`🔴 hard` · *Catching Θ(n) → Θ(n²) in CI* · `go test`

The capstone, and the piece worth keeping. A correctness test **cannot** catch a complexity
regression: both implementations below return byte-identical results on every input. Only a growth
measurement catches it.

The regression is realistic — someone "simplifies" a map away in favour of `slices.Contains`. The code
gets shorter, the tests stay green, and the function becomes quadratic.

**Steps:**

1. Measure doubling ratios with `testing.Benchmark` and take the median.
2. Assert the median falls inside a **wide** band — this is a tripwire, not an instrument.
3. Prove the gate would fire on the regression, and that a correctness test would not.

**`dedupe.go`**

```go
package algo

import "slices"

// Dedupe returns the distinct values of xs, preserving first-seen order.
// The map makes the "have I seen this?" check O(1), so the whole thing is O(n).
func Dedupe(xs []int) []int {
	seen := make(map[int]struct{}, len(xs))
	out := make([]int, 0, len(xs))
	for _, v := range xs {
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		out = append(out, v)
	}
	return out
}

// DedupeSlow is the same function after a well-meaning "simplification":
// the map is gone, and the membership check is a linear scan of the output.
//
// It is shorter, it has no auxiliary structure, it passes every correctness
// test Dedupe passes -- and it is O(n^2). This is the regression the gate below
// exists to catch, because no unit test ever will.
func DedupeSlow(xs []int) []int {
	out := make([]int, 0, len(xs))
	for _, v := range xs {
		if slices.Contains(out, v) { // O(len(out)) inside an O(n) loop
			continue
		}
		out = append(out, v)
	}
	return out
}
```

**`dedupe_test.go`**

```go
package algo

import (
	"slices"
	"testing"
)

// ============================================================================
// A complexity gate. Copy this alongside the lesson-02 harness.
// ============================================================================

// measureGrowth benchmarks build(n) at each size and returns the doubling
// ratios between consecutive sizes.
func measureGrowth(sizes []int, build func(int) func()) []float64 {
	times := make([]float64, len(sizes))
	for i, n := range sizes {
		run := build(n)
		res := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				run()
			}
		})
		times[i] = float64(res.T.Nanoseconds()) / float64(res.N)
	}

	ratios := make([]float64, 0, len(sizes)-1)
	for i := 1; i < len(times); i++ {
		ratios = append(ratios, times[i]/times[i-1])
	}
	return ratios
}

func median(xs []float64) float64 {
	s := slices.Clone(xs)
	slices.Sort(s)
	return s[len(s)/2]
}

// AssertGrowth fails the test if the measured doubling ratio falls outside
// [lo, hi]. Use a WIDE band: this is a tripwire for "somebody made it
// quadratic", not a precision instrument (see example 14 -- timings cannot
// resolve a log factor, so O(n) and O(n log n) share one band).
func AssertGrowth(t *testing.T, name string, sizes []int, build func(int) func(), lo, hi float64) {
	t.Helper()

	ratios := measureGrowth(sizes, build)
	m := median(ratios)
	t.Logf("%-12s sizes %v -> ratios %.2f, median %.2f (allowed %.1f-%.1f)",
		name, sizes, ratios, m, lo, hi)

	if m < lo || m > hi {
		t.Errorf("%s: measured doubling ratio %.2f is outside [%.1f, %.1f] -- "+
			"the complexity is not what this test claims", name, m, lo, hi)
	}
}

// Linear-ish: doubling n should roughly double the time.
const (
	linearLo = 1.5
	linearHi = 3.0
)

func distinctInput(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

// --- the gate, doing its job -------------------------------------------------

func TestDedupeStaysLinear(t *testing.T) {
	AssertGrowth(t, "Dedupe", []int{8192, 16384, 32768, 65536},
		func(n int) func() {
			xs := distinctInput(n)
			return func() { Dedupe(xs) }
		}, linearLo, linearHi)
}

// --- proof that the gate would catch the regression --------------------------

// If DedupeSlow were committed under the name Dedupe, the test above would
// fail. Here we assert that it WOULD -- so this demonstration is itself a
// passing test rather than a broken build.
func TestGateWouldCatchTheRegression(t *testing.T) {
	ratios := measureGrowth([]int{1024, 2048, 4096, 8192},
		func(n int) func() {
			xs := distinctInput(n)
			return func() { DedupeSlow(xs) }
		})
	m := median(ratios)
	t.Logf("DedupeSlow  ratios %.2f, median %.2f", ratios, m)

	if m <= linearHi {
		t.Errorf("expected DedupeSlow to exceed the linear band (%.1f), got %.2f -- "+
			"the gate would have missed the regression", linearHi, m)
	}
	t.Logf("median %.2f > %.1f, so the gate rejects it. A correctness test would not:",
		m, linearHi)

	// ...because the two functions agree on every input.
	for _, n := range []int{0, 1, 5, 50, 500} {
		xs := distinctInput(n)
		xs = append(xs, xs...) // add duplicates
		if !slices.Equal(Dedupe(xs), DedupeSlow(xs)) {
			t.Fatalf("the two implementations disagree at n=%d", n)
		}
	}
	t.Logf("both implementations return identical results on every input tested.")
}
```

**Run it:**

```bash
go test -v .
```

**Sample output** (stable across runs — the ratios moved by ≤0.02):

```
=== RUN   TestDedupeStaysLinear
    dedupe_test.go:74: Dedupe       sizes [8192 16384 32768 65536] -> ratios [2.07 2.10 1.97], median 2.07 (allowed 1.5-3.0)
--- PASS: TestDedupeStaysLinear (4.78s)
=== RUN   TestGateWouldCatchTheRegression
    dedupe_test.go:93: DedupeSlow  ratios [3.84 3.91 3.97], median 3.91
    dedupe_test.go:99: median 3.91 > 3.0, so the gate rejects it. A correctness test would not:
    dedupe_test.go:110: both implementations return identical results on every input tested.
--- PASS: TestGateWouldCatchTheRegression (4.78s)
ok  	l03/ex16	9.967s
```

**2.07 against 3.91.** The two functions are indistinguishable by output and separated by a factor of
almost two in growth rate. That gap is the only signal a test can use, and it is a big one.

Keep the band wide. This exists to catch "someone made it quadratic", not to certify an exact
complexity — example 14 already showed why the latter is not possible from timings.

**Complexity:** `Dedupe` Θ(n) time, Θ(n) space · `DedupeSlow` Θ(n²) time, Θ(n) space · the gate costs `len(sizes)` benchmarks ≈ 5 s, cheap enough for CI

---

> That's the lesson. Next: [04 — Go's Memory Model for DSA](../../04-go-memory-model.md), which
> explains the constants this lesson taught you to drop — and why they decided every crossover you
> measured here.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
