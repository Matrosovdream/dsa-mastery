# Step 02 — Environment & Toolkit · 🔴 Hard

Examples **12–16**: the pieces of the reusable harness, then the harness itself.

Everything here is written once and copied into all 43 remaining lessons. Example 16 is the
deliverable of this whole lesson — the folder you'll start every future practice session from.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. A generic differential oracle

`🔴 hard` · *Generics / reusable test helper*

Lesson 01 wrote the oracle pattern by hand. Written once as a generic helper, it applies to every
algorithm in the plan without modification — the type parameters absorb the differences between
problems.

The design decisions worth copying: **`Out comparable`** so `==` is a valid verdict; **`t.Helper()`**
so failures point at the caller; a **fixed seed** so a failure reproduces exactly; and **fail fast** —
one readable counterexample beats a thousand identical ones.

**Steps:**

1. Write `Differential` once, parameterized over the input and output types.
2. Supply a generator that makes *small* inputs from a narrow value range.
3. Point it at two different problems to prove the helper is actually reusable.

**`algo.go`**

```go
package algo

// HasPair reports whether two distinct elements of xs sum to target. O(n).
func HasPair(xs []int, target int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		if _, ok := seen[target-v]; ok {
			return true
		}
		seen[v] = struct{}{}
	}
	return false
}

// CountDistinct returns how many distinct values xs holds. O(n).
func CountDistinct(xs []int) int {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		seen[v] = struct{}{}
	}
	return len(seen)
}
```

**`differential_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"testing"
)

// Differential is the oracle pattern, written once and reused forever.
//
//   - In  is whatever your function takes (often a struct of several arguments).
//   - Out must be comparable so that == is a valid verdict.
//   - gen builds one random input; keep it SMALL, so failures are readable.
//   - seed is fixed, so a failure reproduces exactly on the next run.
//
// It stops at the first disagreement: one minimal counterexample beats a
// thousand identical ones.
func Differential[In any, Out comparable](
	t *testing.T,
	trials int,
	seed uint64,
	gen func(*rand.Rand) In,
	oracle func(In) Out,
	candidate func(In) Out,
) {
	t.Helper()
	rng := rand.New(rand.NewPCG(seed, seed+1))

	for i := 0; i < trials; i++ {
		in := gen(rng)
		want := oracle(in)
		got := candidate(in)
		if got != want {
			t.Fatalf("disagreement on trial %d\n  input:     %+v\n  oracle:    %v\n  candidate: %v",
				i+1, in, want, got)
		}
	}
}

// --- use 1: two-sum ---------------------------------------------------------

type pairInput struct {
	Xs     []int
	Target int
}

func hasPairBrute(in pairInput) bool {
	for i := 0; i < len(in.Xs); i++ {
		for j := i + 1; j < len(in.Xs); j++ {
			if in.Xs[i]+in.Xs[j] == in.Target {
				return true
			}
		}
	}
	return false
}

func TestHasPairAgainstOracle(t *testing.T) {
	gen := func(rng *rand.Rand) pairInput {
		xs := make([]int, rng.IntN(7))
		for i := range xs {
			xs[i] = rng.IntN(11) - 5 // small range -> frequent collisions -> real coverage
		}
		return pairInput{Xs: xs, Target: rng.IntN(11) - 5}
	}

	Differential(t, 50_000, 42, gen,
		hasPairBrute,
		func(in pairInput) bool { return HasPair(in.Xs, in.Target) },
	)
}

// --- use 2: distinct count, same helper, different shapes --------------------

func countDistinctBrute(xs []int) int {
	n := 0
	for i, v := range xs {
		first := true
		for j := 0; j < i; j++ {
			if xs[j] == v {
				first = false
				break
			}
		}
		if first {
			n++
		}
	}
	return n
}

func TestCountDistinctAgainstOracle(t *testing.T) {
	gen := func(rng *rand.Rand) []int {
		xs := make([]int, rng.IntN(10))
		for i := range xs {
			xs[i] = rng.IntN(5)
		}
		return xs
	}

	Differential(t, 50_000, 7, gen, countDistinctBrute, CountDistinct)
}
```

**Run it:**

```bash
go test -v ./ex12
```

**Output:**

```
=== RUN   TestHasPairAgainstOracle
--- PASS: TestHasPairAgainstOracle (0.01s)
=== RUN   TestCountDistinctAgainstOracle
--- PASS: TestCountDistinctAgainstOracle (0.01s)
PASS
ok  	l02/ex12	0.623s
```

100,000 random trials across two problems, in 20 milliseconds. That's the whole argument for keeping
generated inputs tiny — you get exhaustive-feeling coverage of the interesting region for free.

**Takeaway:** one generic helper, every algorithm. Note the second call passes a plain `[]int` as `In` while the first passes a struct — the type parameters absorb the difference.

---

## 13. One property is never enough

`🔴 hard` · *Property-based invariants*

Everyone's first instinct for testing a sort is "check the output is sorted". Here is a function that
satisfies that property **on every input** and is catastrophically wrong: it's insertion sort with the
final write deleted — a one-line omission that survives code review.

It still shifts larger elements rightward, so the result is still non-decreasing. But the element it
was supposed to insert is never written back; it gets overwritten by a copy of its neighbour. Data
disappears, silently, behind a perfectly sorted-looking result.

The **permutation** property — same values, same counts — catches it immediately. Every sort needs both.

**Steps:**

1. Write both properties: `isSorted` and `isPermutation`.
2. Verify the real sort satisfies both across 20,000 random inputs.
3. Verify `BadSort` satisfies the weak one across 20,000 inputs — and *assert* that it violates the strong one, so the demonstration is itself a passing test.

**`sort.go`**

```go
package algo

// InsertionSort sorts xs in place.
func InsertionSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		xs[j+1] = v // drop this one line and you get BadSort
	}
}

// BadSort is InsertionSort with the final write missing -- a one-line omission,
// the kind that survives code review.
//
// It still shifts the larger elements right, so the result is still
// non-decreasing on every input. But the element it was supposed to insert is
// never written back: it is overwritten by a copy of its neighbour. The sort
// silently destroys data while looking perfectly sorted.
func BadSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		// missing: xs[j+1] = v
	}
}
```

**`sort_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"slices"
	"testing"
)

// --- the two properties every sort must satisfy -----------------------------

// Property 1: the output is non-decreasing. The obvious check -- and on its own,
// worthless.
func isSorted(xs []int) bool {
	for i := 1; i < len(xs); i++ {
		if xs[i] < xs[i-1] {
			return false
		}
	}
	return true
}

// Property 2: the output is a permutation of the input -- same values, same
// counts. This is the property that says "you didn't invent or lose data".
func isPermutation(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}
	count := make(map[int]int, len(a))
	for _, v := range a {
		count[v]++
	}
	for _, v := range b {
		count[v]--
		if count[v] < 0 {
			return false
		}
	}
	return true
}

func randomSlice(rng *rand.Rand) []int {
	xs := make([]int, rng.IntN(12))
	for i := range xs {
		xs[i] = rng.IntN(6) // small range: duplicates are the interesting case
	}
	return xs
}

// --- the real sort passes both ----------------------------------------------

func TestInsertionSortProperties(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))
	for i := 0; i < 20_000; i++ {
		in := randomSlice(rng)
		out := slices.Clone(in)
		InsertionSort(out)

		if !isSorted(out) {
			t.Fatalf("not sorted: %v -> %v", in, out)
		}
		if !isPermutation(in, out) {
			t.Fatalf("not a permutation: %v -> %v", in, out)
		}
	}
}

// --- and here is why one property is never enough ---------------------------

// BadSort satisfies "is sorted" on every input. A test suite that checks only
// this property is green.
func TestBadSortSatisfiesTheWeakProperty(t *testing.T) {
	rng := rand.New(rand.NewPCG(3, 4))
	for i := 0; i < 20_000; i++ {
		out := randomSlice(rng)
		BadSort(out)
		if !isSorted(out) {
			t.Fatalf("BadSort produced unsorted output: %v", out)
		}
	}
}

// The permutation property rejects it -- and here we assert that it does, so
// the demonstration itself is a passing test.
func TestBadSortViolatesThePermutationProperty(t *testing.T) {
	in := []int{3, 1, 3} // any input with a duplicate
	out := slices.Clone(in)
	BadSort(out)

	if !isSorted(out) {
		t.Fatalf("expected BadSort output to look sorted, got %v", out)
	}
	if isPermutation(in, out) {
		t.Fatalf("expected BadSort to lose data, but %v is a permutation of %v", out, in)
	}
	t.Logf("BadSort(%v) = %v -- perfectly sorted, and the 1 is simply gone", in, out)
}
```

**Run it:**

```bash
go test -v ./ex13
```

**Output:**

```
=== RUN   TestInsertionSortProperties
--- PASS: TestInsertionSortProperties (0.01s)
=== RUN   TestBadSortSatisfiesTheWeakProperty
--- PASS: TestBadSortSatisfiesTheWeakProperty (0.00s)
=== RUN   TestBadSortViolatesThePermutationProperty
    sort_test.go:95: BadSort([3 1 3]) = [3 3 3] -- perfectly sorted, and the 1 is simply gone
--- PASS: TestBadSortViolatesThePermutationProperty (0.00s)
PASS
ok  	l02/ex13	0.369s
```

`BadSort([3 1 3])` returns `[3 3 3]`. Sorted? Yes. A sort? No — the 1 is gone. **20,000 random inputs
could not tell the difference using `isSorted` alone.**

**Takeaway:** a property test is only as good as the property. For any rearrangement, always assert the multiset is preserved — that's the property that says "you didn't invent or lose data".

---

## 14. Where did the time go

`🔴 hard` · *pprof*

Profiling takes thirty seconds and routinely contradicts your guess. Here a function does one O(n)
pass followed by one O(n²) pass; the profile settles which one matters without any argument.

**Steps:**

1. Run the benchmark with `-cpuprofile`, keeping the test binary with `-o`.
2. Open the profile with `go tool pprof -top`.
3. Read the `flat` column: time spent *in* that function; `cum` includes everything it called.

**`pairs.go`**

```go
package algo

// CountPairs counts index pairs (i<j) whose values sum to target.
// Deliberately quadratic -- this is the function the profiler should point at.
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

// normalize is a cheap helper called once per element. It is here to show what
// a *small* contributor looks like in a profile, next to the real hot spot.
func normalize(xs []int, mod int) []int {
	out := make([]int, len(xs))
	for i, v := range xs {
		out[i] = ((v % mod) + mod) % mod
	}
	return out
}

// Analyze is the entry point: one O(n) pass, then one O(n^2) pass.
func Analyze(xs []int, target, mod int) int {
	return CountPairs(normalize(xs, mod), target)
}
```

**`pairs_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"testing"
)

func benchInput(n int) []int {
	rng := rand.New(rand.NewPCG(5, 6))
	xs := make([]int, n)
	for i := range xs {
		xs[i] = rng.IntN(1000) - 500
	}
	return xs
}

func BenchmarkAnalyze(b *testing.B) {
	xs := benchInput(4000)
	b.ReportAllocs()
	for b.Loop() {
		Analyze(xs, 100, 97)
	}
}
```

**Run it:**

```bash
go test -bench=Analyze -run='^$' -cpuprofile=cpu.out -o ex14.test ./ex14
go tool pprof -top -nodecount=8 ex14.test cpu.out
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex14
cpu: Apple M4
BenchmarkAnalyze-10    	     537	   1992599 ns/op	   32798 B/op	       1 allocs/op
PASS
ok  	l02/ex14	1.587s
```

```
File: ex14.test
Type: cpu
Duration: 1.21s, Total samples = 940ms (77.63%)
Showing nodes accounting for 940ms, 100% of 940ms total
Showing top 8 nodes out of 28
      flat  flat%   sum%        cum   cum%
     760ms 80.85% 80.85%      820ms 87.23%  l02/ex14.CountPairs (inline)
      80ms  8.51% 89.36%       80ms  8.51%  runtime.madvise
      60ms  6.38% 95.74%       60ms  6.38%  runtime.asyncPreempt
      10ms  1.06% 96.81%       30ms  3.19%  l02/ex14.normalize (inline)
      10ms  1.06% 97.87%       10ms  1.06%  runtime.(*mspan).init
      10ms  1.06% 98.94%       10ms  1.06%  runtime.kevent
      10ms  1.06%   100%       10ms  1.06%  runtime.memclrNoHeapPointers
         0     0%   100%      850ms 90.43%  l02/ex14.Analyze
```

`CountPairs` is **80.85%** flat; `normalize` is **1.06%**. Optimizing `normalize` to infinite speed
would buy 1%. Other views worth knowing: `-list=CountPairs` annotates the source line by line,
`-http=:8080` opens the flame graph, and `-memprofile` does the same for allocations.

**Takeaway:** `flat` is where the CPU actually is. Profile first — the answer is often not where you'd bet, and it is never worth optimizing a 1% line.

---

## 15. An allocation budget that fails the build

`🔴 hard` · *testing.AllocsPerRun as a unit test*

A benchmark reports allocations; nobody notices when the number creeps up. A **test** with a budget
fails CI the moment someone regresses it — no human required to read a number and judge.

Allocation counts are deterministic (they follow the growth policy, not the clock), which is what makes
this viable as a hard assertion rather than a flaky one.

**Steps:**

1. Assign the result to a package-level sink so nothing is optimized away.
2. Measure with `testing.AllocsPerRun`, compare against a budget, and `t.Logf` the actual number either way.
3. Set each budget to the value you **measured**, not one you hoped for.

**`index.go`**

```go
package algo

// Dedupe returns the distinct values of xs in first-seen order.
// The capacity is reserved up front: one allocation for the result, whatever n is.
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

// SumEven adds the even elements of xs. It allocates nothing at all, and a test
// should hold it to that.
func SumEven(xs []int) int {
	total := 0
	for _, v := range xs {
		if v%2 == 0 {
			total += v
		}
	}
	return total
}
```

**`index_test.go`**

```go
package algo

import "testing"

var (
	sinkSlice []int
	sinkInt   int
)

func input(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i % 500 // guarantees duplicates
	}
	return xs
}

// An allocation budget is a unit test, not a benchmark: it fails the build when
// someone regresses it, and it needs no human to read a number and judge.
//
// Set each budget to the value you MEASURED, not to a round number you hoped for.
func TestAllocationBudget(t *testing.T) {
	xs := input(1000)

	tests := []struct {
		name   string
		budget float64
		fn     func()
	}{
		{"SumEven allocates nothing", 0, func() { sinkInt = SumEven(xs) }},
		{"Dedupe: 1 slice + the map's buckets", 6, func() { sinkSlice = Dedupe(xs) }},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := testing.AllocsPerRun(100, tt.fn)
			if got > tt.budget {
				t.Errorf("allocs/op = %.0f, over the budget of %.0f", got, tt.budget)
			}
			t.Logf("allocs/op = %.0f (budget %.0f)", got, tt.budget)
		})
	}
}
```

**Run it:**

```bash
go test -v ./ex15
```

**Output:**

```
=== RUN   TestAllocationBudget
=== RUN   TestAllocationBudget/SumEven_allocates_nothing
    index_test.go:40: allocs/op = 0 (budget 0)
=== RUN   TestAllocationBudget/Dedupe:_1_slice_+_the_map's_buckets
    index_test.go:40: allocs/op = 6 (budget 6)
--- PASS: TestAllocationBudget (0.00s)
    --- PASS: TestAllocationBudget/SumEven_allocates_nothing (0.00s)
    --- PASS: TestAllocationBudget/Dedupe:_1_slice_+_the_map's_buckets (0.00s)
PASS
ok  	l02/ex15	0.373s
```

**What a regression looks like.** Writing this example I guessed a budget of 5 for `Dedupe` before
measuring. Here is what the same test printed:

```
=== RUN   TestAllocationBudget/Dedupe:_1_slice_+_the_map
    index_test.go:38: allocs/op = 6, over the budget of 5
    index_test.go:40: allocs/op = 6 (budget 5)
--- FAIL: TestAllocationBudget (0.00s)
    --- PASS: TestAllocationBudget/SumEven_allocates_nothing (0.00s)
    --- FAIL: TestAllocationBudget/Dedupe:_1_slice_+_the_map (0.00s)
FAIL
```

Six, not five: one for the result slice and five for the map's internal growth. **Measure, then write
the number down** — the same rule as everywhere else in this plan.

**Takeaway:** turn any number you care about into an assertion. Allocation counts are deterministic enough to be a hard gate, unlike timings.

---

## 16. The harness

`🔴 hard` · *The deliverable*

This is what lesson 02 exists to produce: a practice folder you can copy into every remaining lesson.
Four files, three of which never change.

| File | Changes per lesson? | What it is |
|------|--------------------|------------|
| `reverse.go` | ✅ replaced | whatever the lesson asked you to implement |
| `reverse_test.go` | ✅ rewritten | the lesson's own tests — table, invariants, oracle |
| `harness_test.go` | ❌ copied as-is | `Differential`, generators, `IsPermutation`, `EdgeCases`, `CheckInPlace` |
| `bench_test.go` | 🔸 lightly edited | empty control + the size sweep |
| `check.sh` | ❌ copied as-is | the five gates, one command |

The test file shows the **four layers** every lesson should have: table → invariants → oracle → a
named property. Each catches things the others don't.

**Steps:**

1. Copy the four harness files into `practice/NN-topic/`.
2. Replace the implementation file and rewrite the test file for the lesson's structure.
3. Run `./check.sh` until it prints "all gates passed", then tick the lesson in PROGRESS.md.

**`reverse.go`** — the part that changes

```go
package algo

// This file stands in for "whatever the lesson asked you to implement".
// Everything else in this folder is the harness, and does not change.

// Reverse reverses xs in place. O(n) time, O(1) space.
func Reverse(xs []int) {
	for i, j := 0, len(xs)-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
	}
}

// Rotate shifts xs left by k positions, in place, using the triple-reversal
// trick. O(n) time, O(1) space.
func Rotate(xs []int, k int) {
	n := len(xs)
	if n == 0 {
		return
	}
	k = ((k % n) + n) % n // normalize, including negative k
	Reverse(xs[:k])
	Reverse(xs[k:])
	Reverse(xs)
}
```

**`harness_test.go`** — copy this into every lesson, unchanged

```go
package algo

import (
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// The harness. Copy this file into every practice/NN-topic/ folder unchanged.
// ============================================================================

// Differential runs candidate against a slow-but-obviously-correct oracle on
// `trials` generated inputs, and stops at the first disagreement.
func Differential[In any, Out comparable](
	t *testing.T,
	trials int,
	seed uint64,
	gen func(*rand.Rand) In,
	oracle func(In) Out,
	candidate func(In) Out,
) {
	t.Helper()
	rng := rand.New(rand.NewPCG(seed, seed+1))

	for i := 0; i < trials; i++ {
		in := gen(rng)
		want := oracle(in)
		got := candidate(in)
		if got != want {
			t.Fatalf("disagreement on trial %d\n  input:     %+v\n  oracle:    %v\n  candidate: %v",
				i+1, in, want, got)
		}
	}
}

// RandomSlice generates a small slice with a small value range -- small enough
// that a failure is readable, narrow enough that duplicates actually occur.
func RandomSlice(rng *rand.Rand, maxLen, maxVal int) []int {
	xs := make([]int, rng.IntN(maxLen+1))
	for i := range xs {
		xs[i] = rng.IntN(maxVal)
	}
	return xs
}

// IsPermutation reports whether a and b hold the same values with the same
// counts. The property that catches a "sort" which loses data.
func IsPermutation(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}
	count := make(map[int]int, len(a))
	for _, v := range a {
		count[v]++
	}
	for _, v := range b {
		count[v]--
		if count[v] < 0 {
			return false
		}
	}
	return true
}

// EdgeCases returns the inputs that break sequence algorithms, in the order
// they should be tried. Start every table with these.
func EdgeCases() []struct {
	Name string
	Xs   []int
} {
	return []struct {
		Name string
		Xs   []int
	}{
		{"nil", nil},
		{"empty", []int{}},
		{"single", []int{1}},
		{"two", []int{1, 2}},
		{"all equal", []int{7, 7, 7}},
		{"duplicates", []int{1, 2, 1, 2}},
		{"sorted", []int{1, 2, 3, 4, 5}},
		{"reverse sorted", []int{5, 4, 3, 2, 1}},
		{"negatives", []int{-2, -1, 0, 1}},
	}
}

// CheckInPlace runs fn over every edge case and asserts the invariant that
// holds for any in-place rearrangement: the multiset is unchanged.
func CheckInPlace(t *testing.T, name string, fn func([]int)) {
	t.Helper()
	for _, tc := range EdgeCases() {
		t.Run(name+"/"+tc.Name, func(t *testing.T) {
			original := slices.Clone(tc.Xs)
			got := slices.Clone(tc.Xs)
			fn(got)

			if len(got) != len(original) {
				t.Fatalf("length changed: %d -> %d", len(original), len(got))
			}
			if !IsPermutation(original, got) {
				t.Errorf("not a permutation: %v -> %v", original, got)
			}
		})
	}
}
```

**`reverse_test.go`** — the four layers, rewritten each lesson

```go
package algo

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// The lesson's own tests. This is the only file you write per lesson.
// ============================================================================

// 1. The table: specific inputs, specific expected outputs.
func TestReverse(t *testing.T) {
	tests := []struct {
		name string
		xs   []int
		want []int
	}{
		{"nil", nil, nil},
		{"empty", []int{}, nil},
		{"single", []int{1}, []int{1}},
		{"even length", []int{1, 2, 3, 4}, []int{4, 3, 2, 1}},
		{"odd length", []int{1, 2, 3}, []int{3, 2, 1}},
		{"duplicates", []int{1, 1, 2}, []int{2, 1, 1}},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := slices.Clone(tt.xs)
			Reverse(got)
			if !slices.Equal(got, tt.want) {
				t.Errorf("Reverse(%v) = %v, want %v", tt.xs, got, tt.want)
			}
		})
	}
}

func TestRotate(t *testing.T) {
	tests := []struct {
		name string
		xs   []int
		k    int
		want []int
	}{
		{"k=0", []int{1, 2, 3}, 0, []int{1, 2, 3}},
		{"k=1", []int{1, 2, 3}, 1, []int{2, 3, 1}},
		{"k=len", []int{1, 2, 3}, 3, []int{1, 2, 3}},
		{"k>len", []int{1, 2, 3}, 4, []int{2, 3, 1}},
		{"k negative", []int{1, 2, 3}, -1, []int{3, 1, 2}},
		{"empty", nil, 5, nil},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := slices.Clone(tt.xs)
			Rotate(got, tt.k)
			if !slices.Equal(got, tt.want) {
				t.Errorf("Rotate(%v, %d) = %v, want %v", tt.xs, tt.k, got, tt.want)
			}
		})
	}
}

// 2. The invariants: what must hold for EVERY input, checked over the edge cases.
func TestInPlaceInvariants(t *testing.T) {
	CheckInPlace(t, "Reverse", Reverse)
	CheckInPlace(t, "Rotate", func(xs []int) { Rotate(xs, 3) })
}

// 3. The oracle: agreement with a slow reference, over random input.
func TestReverseAgainstOracle(t *testing.T) {
	// The oracle: allocate a new slice and fill it backwards. Obviously correct.
	oracle := func(xs []int) string {
		out := make([]int, len(xs))
		for i, v := range xs {
			out[len(xs)-1-i] = v
		}
		return fmt.Sprint(out)
	}
	candidate := func(xs []int) string {
		out := slices.Clone(xs)
		Reverse(out)
		return fmt.Sprint(out)
	}

	Differential(t, 50_000, 1,
		func(rng *rand.Rand) []int { return RandomSlice(rng, 10, 6) },
		oracle, candidate)
}

// 4. Reverse is its own inverse -- a property worth stating outright.
func TestReverseTwiceIsIdentity(t *testing.T) {
	rng := rand.New(rand.NewPCG(2, 3))
	for i := 0; i < 20_000; i++ {
		original := RandomSlice(rng, 12, 8)
		got := slices.Clone(original)
		Reverse(got)
		Reverse(got)
		if !slices.Equal(got, original) {
			t.Fatalf("reversing twice changed %v into %v", original, got)
		}
	}
}
```

> Note the oracle returns a `string`: `Differential` needs a `comparable` output, and slices aren't
> comparable. Formatting with `fmt.Sprint` is the cheap, general way to compare non-comparable
> results — fine at 50,000 tiny inputs.

**`bench_test.go`** — control + sweep

```go
package algo

import (
	"fmt"
	"testing"
)

// ============================================================================
// The benchmark skeleton. Copy this file too; change only the sweep body.
// ============================================================================

// Always present, always first. No benchmark below this floor means anything.
func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// The sweep: the same operation across growing n, so the GROWTH is visible.
// Read the ns/op column, not any single row.
func BenchmarkReverse(b *testing.B) {
	for _, n := range []int{16, 256, 4096, 65536} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				Reverse(xs)
			}
		})
	}
}
```

**`check.sh`** — the five gates in one command

```bash
#!/usr/bin/env bash
# The gates. Run this before ticking a lesson done in PROGRESS.md.
# Usage: ./check.sh        (from inside a practice/NN-topic/ folder)

set -uo pipefail
fail=0

step() { printf '\n\033[1m== %s ==\033[0m\n' "$1"; }
ok()   { printf 'ok\n'; }

step "gofmt"
unformatted=$(gofmt -l .)
if [ -n "$unformatted" ]; then
	echo "needs formatting:"
	echo "$unformatted"
	fail=1
else
	ok
fi

step "go vet"
go vet ./... && ok || fail=1

step "go test -race"
go test -race ./... || fail=1

step "coverage"
go test -cover ./... || fail=1

step "benchmarks"
go test -bench=. -benchmem -run='^$' ./... || fail=1

printf '\n'
if [ "$fail" -ne 0 ]; then
	printf '\033[31mFAILED\033[0m -- fix the above before ticking this lesson done\n'
	exit 1
fi
printf '\033[32mall gates passed\033[0m\n'
```

**Run it:**

```bash
chmod +x check.sh
./check.sh
```

**Sample output** (colour codes stripped; timings vary):

```
== gofmt ==
ok

== go vet ==
ok

== go test -race ==
ok  	l02/ex16	1.614s

== coverage ==
ok  	l02/ex16	0.355s	coverage: 100.0% of statements

== benchmarks ==
goos: darwin
goarch: arm64
pkg: l02/ex16
cpu: Apple M4
BenchmarkEmptyControl-10    	638505613	         1.651 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=16-10    	385413799	         3.135 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=256-10   	24295677	        50.04 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=4096-10  	 1638211	       733.4 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=65536-10 	   99662	     12152 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	l02/ex16	6.196s

all gates passed
```

Read the sweep: 16× the input costs 16× the time, all the way up, with **zero allocations** — exactly
what an in-place O(n) algorithm should look like. And the control at 1.651 ns confirms the n=16 row
(3.135 ns) is real work rather than harness noise.

**Takeaway:** this folder is the starting point for every lesson from 03 to 45. Copy it, replace the implementation, keep the gates green.

---

> That's the toolkit. Next: [03 — Complexity Analysis](../../03-complexity-analysis.md), where you'll
> use the sweep from example 8 to confirm the notation's claims empirically.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
