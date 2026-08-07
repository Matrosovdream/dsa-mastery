# Step 02 — Environment & Toolkit · 🟢 Easy

Examples **1–6**: the test file, the table, the comparison trap, the checklist, coverage, benchmarks.

> ⚠️ **These examples are `go test` packages, not `go run` programs.** Each one is a folder with two
> or three files. Create the folder, put the files in it, and run the command shown.

```bash
mkdir -p /tmp/dsa-t/ex01 && cd /tmp/dsa-t   # once: go mod init scratch
# put the files in ex01/, then:
go test -v ./ex01
```

Test output includes a wall-clock time on the `ok` line (`ok l02/ex01 0.546s`) — yours will differ.
Everything above that line is deterministic.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. Your first algorithm test

`🟢 easy` · *Test mechanics*

The whole contract: a file ending in `_test.go`, in the same package as the code, holding functions
named `TestXxx(t *testing.T)`. No framework, no assertions library, no annotations. Note the
signature under test — an algorithm that can fail (`Max` of nothing) says so with a `bool` rather
than panicking or inventing a zero.

**Steps:**

1. Put the code and the test in the same package, in two files.
2. Use `t.Fatal` when nothing after it would make sense, `t.Errorf` when you want the rest to run.
3. Run with `-v` to see each test named.

**`max.go`**

```go
package algo

// Max returns the largest element of xs. The bool reports whether xs had any
// elements at all -- an algorithm that can fail must say so in its signature.
func Max(xs []int) (int, bool) {
	if len(xs) == 0 {
		return 0, false
	}
	best := xs[0]
	for _, v := range xs[1:] {
		if v > best {
			best = v
		}
	}
	return best, true
}
```

**`max_test.go`**

```go
package algo

import "testing"

// A test file lives beside the code, ends in _test.go, and shares the package.
// Every test is func TestXxx(t *testing.T).

func TestMax(t *testing.T) {
	got, ok := Max([]int{3, 9, 4})
	if !ok {
		// Fatal stops this test now: the value below would be meaningless.
		t.Fatal("Max([3 9 4]) returned ok=false for a non-empty slice")
	}
	if got != 9 {
		// Errorf records the failure but keeps going.
		t.Errorf("Max([3 9 4]) = %d, want 9", got)
	}
}

func TestMaxEmpty(t *testing.T) {
	if _, ok := Max(nil); ok {
		t.Error("Max(nil) reported ok=true, want false")
	}
}
```

**Run it:**

```bash
go test -v ./ex01
```

**Output:**

```
=== RUN   TestMax
--- PASS: TestMax (0.00s)
=== RUN   TestMaxEmpty
--- PASS: TestMaxEmpty (0.00s)
PASS
ok  	l02/ex01	0.546s
```

**Takeaway:** tests are ordinary Go code in the same package — which is why they can reach the unexported fields your data structures will keep their invariants in.

---

## 2. The table-driven form

`🟢 easy` · *Table + subtests*

The default shape for every test in this plan. One slice of cases, one loop, one `t.Run` per case.
Three payoffs: adding a case is adding a line, each case reports its own name on failure, and you can
re-run exactly one case with `-run` while debugging.

**Steps:**

1. Declare an anonymous struct slice with a `name` field and the inputs/outputs.
2. Loop, calling `t.Run(tt.name, ...)`.
3. Re-run a single case with `-run 'TestMax/negatives'` — note that spaces in names become underscores.

**`max.go`**

```go
package algo

// Max returns the largest element of xs, and false if xs is empty.
func Max(xs []int) (int, bool) {
	if len(xs) == 0 {
		return 0, false
	}
	best := xs[0]
	for _, v := range xs[1:] {
		if v > best {
			best = v
		}
	}
	return best, true
}
```

**`max_test.go`**

```go
package algo

import "testing"

// The table-driven form: one slice of cases, one loop, one t.Run per case.
// Adding a case is adding a line -- which is the whole point.
func TestMax(t *testing.T) {
	tests := []struct {
		name   string
		xs     []int
		want   int
		wantOK bool
	}{
		{"empty", nil, 0, false},
		{"single", []int{7}, 7, true},
		{"ascending", []int{1, 2, 3}, 3, true},
		{"descending", []int{3, 2, 1}, 3, true},
		{"max in middle", []int{1, 9, 2}, 9, true},
		{"duplicates", []int{4, 4, 4}, 4, true},
		{"negatives", []int{-5, -2, -9}, -2, true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, ok := Max(tt.xs)
			if ok != tt.wantOK {
				t.Fatalf("Max(%v) ok = %v, want %v", tt.xs, ok, tt.wantOK)
			}
			if ok && got != tt.want {
				t.Errorf("Max(%v) = %d, want %d", tt.xs, got, tt.want)
			}
		})
	}
}
```

**Run it:**

```bash
go test -v ./ex02
```

**Output:**

```
=== RUN   TestMax
=== RUN   TestMax/empty
=== RUN   TestMax/single
=== RUN   TestMax/ascending
=== RUN   TestMax/descending
=== RUN   TestMax/max_in_middle
=== RUN   TestMax/duplicates
=== RUN   TestMax/negatives
--- PASS: TestMax (0.00s)
    --- PASS: TestMax/empty (0.00s)
    --- PASS: TestMax/single (0.00s)
    --- PASS: TestMax/ascending (0.00s)
    --- PASS: TestMax/descending (0.00s)
    --- PASS: TestMax/max_in_middle (0.00s)
    --- PASS: TestMax/duplicates (0.00s)
    --- PASS: TestMax/negatives (0.00s)
PASS
ok  	l02/ex02	0.289s
```

**Then run one case only:**

```bash
go test -v -run 'TestMax/negatives' ./ex02
```

```
=== RUN   TestMax
=== RUN   TestMax/negatives
--- PASS: TestMax (0.00s)
    --- PASS: TestMax/negatives (0.00s)
PASS
ok  	l02/ex02	0.135s
```

**Takeaway:** `-run 'TestName/case_name'` is the debugging loop — one case, one second, no noise.

---

## 3. nil vs empty, and how to compare slices

`🟢 easy` · *slices.Equal vs reflect.DeepEqual*

The trap that will bite you in half the lessons ahead. `var out []int` is **nil** until the first
`append`, so a filter that matches nothing returns nil, not `[]int{}`. Whether your test passes then
depends entirely on which comparison you reached for — and the popular one gets it wrong.

**Steps:**

1. Confirm the "no matches" return really is nil, and that its length is 0.
2. Check it against `[]int{}` with `slices.Equal` (contents) and `reflect.DeepEqual` (representation).
3. Note that the table below never has to decide between `nil` and `[]int{}` — `slices.Equal` doesn't care.

**`filter.go`**

```go
package algo

// Evens returns the even numbers in xs, in order.
//
// Note what it returns when nothing matches: a nil slice, because `var out []int`
// is nil until the first append. That is idiomatic Go -- and it is the single
// most common source of confusing test failures in this whole plan.
func Evens(xs []int) []int {
	var out []int
	for _, v := range xs {
		if v%2 == 0 {
			out = append(out, v)
		}
	}
	return out
}
```

**`filter_test.go`**

```go
package algo

import (
	"reflect"
	"slices"
	"testing"
)

// You cannot compare slices with == in Go. There are two real choices, and they
// disagree about exactly one thing: whether nil and empty are the same value.

func TestNilVersusEmpty(t *testing.T) {
	got := Evens([]int{1, 3, 5}) // nothing matches

	if got != nil {
		t.Errorf("expected a nil slice, got %#v", got)
	}
	if len(got) != 0 {
		t.Errorf("expected length 0, got %d", len(got))
	}

	// slices.Equal compares CONTENTS: nil and empty are both "no elements".
	if !slices.Equal(got, []int{}) {
		t.Error("slices.Equal(nil, []int{}) = false, want true")
	}

	// reflect.DeepEqual compares REPRESENTATION: nil != empty.
	if reflect.DeepEqual(got, []int{}) {
		t.Error("reflect.DeepEqual(nil, []int{}) = true, want false")
	}
}

// So: use slices.Equal for algorithm output, unless nil-ness is part of the
// contract you are actually testing.
func TestEvens(t *testing.T) {
	tests := []struct {
		name string
		xs   []int
		want []int
	}{
		{"nil input", nil, nil},
		{"no matches", []int{1, 3}, nil},
		{"all match", []int{2, 4}, []int{2, 4}},
		{"mixed", []int{1, 2, 3, 4}, []int{2, 4}},
		{"negatives", []int{-2, -3}, []int{-2}},
		{"zero is even", []int{0, 1}, []int{0}},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// slices.Equal treats want=nil and want=[]int{} identically, so the
			// table above does not have to care which one it wrote.
			if got := Evens(tt.xs); !slices.Equal(got, tt.want) {
				t.Errorf("Evens(%v) = %v, want %v", tt.xs, got, tt.want)
			}
		})
	}
}
```

**Run it:**

```bash
go test -v ./ex03
```

**Output:**

```
=== RUN   TestNilVersusEmpty
--- PASS: TestNilVersusEmpty (0.00s)
=== RUN   TestEvens
=== RUN   TestEvens/nil_input
=== RUN   TestEvens/no_matches
=== RUN   TestEvens/all_match
=== RUN   TestEvens/mixed
=== RUN   TestEvens/negatives
=== RUN   TestEvens/zero_is_even
--- PASS: TestEvens (0.00s)
    --- PASS: TestEvens/nil_input (0.00s)
    --- PASS: TestEvens/no_matches (0.00s)
    --- PASS: TestEvens/all_match (0.00s)
    --- PASS: TestEvens/mixed (0.00s)
    --- PASS: TestEvens/negatives (0.00s)
    --- PASS: TestEvens/zero_is_even (0.00s)
PASS
ok  	l02/ex03	0.285s
```

**Takeaway:** `slices.Equal` for contents. Reach for `reflect.DeepEqual` only when nil-vs-empty is the thing you mean to assert — and then say so in a comment.

---

## 4. The edge-case checklist

`🟢 easy` · *What to test before the interesting case*

Most algorithm bugs are not in the clever part. They're at the boundary: the empty input, the
single-element input, the two-element input, the all-duplicates input. This is the checklist every
sequence algorithm in this plan starts from, grouped so you can see what each group probes.

**Steps:**

1. Group the table by what it stresses: the empty/tiny boundary, degenerate content, ordering, value range.
2. Include `math.MaxInt` and `math.MinInt` — overflow bugs hide nowhere else.
3. `t.Fatalf` on the `ok` mismatch so a bad case doesn't cascade into confusing follow-up errors.

**`minmax.go`**

```go
package algo

// MinMax returns the smallest and largest elements of xs in a single pass.
func MinMax(xs []int) (min, max int, ok bool) {
	if len(xs) == 0 {
		return 0, 0, false
	}
	min, max = xs[0], xs[0]
	for _, v := range xs[1:] {
		if v < min {
			min = v
		}
		if v > max {
			max = v
		}
	}
	return min, max, true
}
```

**`minmax_test.go`**

```go
package algo

import (
	"math"
	"testing"
)

// The checklist. Every sequence algorithm in this plan gets these cases, in this
// order, before any "interesting" case is written. Most algorithm bugs live in
// the first four rows -- not in the logic you were concentrating on.
func TestMinMax(t *testing.T) {
	tests := []struct {
		name    string
		xs      []int
		wantMin int
		wantMax int
		wantOK  bool
	}{
		// --- the empty/tiny boundary ---
		{"nil", nil, 0, 0, false},
		{"empty", []int{}, 0, 0, false},
		{"single", []int{5}, 5, 5, true},
		{"two ascending", []int{1, 2}, 1, 2, true},
		{"two descending", []int{2, 1}, 1, 2, true},

		// --- degenerate content ---
		{"all equal", []int{3, 3, 3}, 3, 3, true},
		{"duplicates", []int{2, 5, 2, 5}, 2, 5, true},

		// --- ordering ---
		{"already sorted", []int{1, 2, 3, 4}, 1, 4, true},
		{"reverse sorted", []int{4, 3, 2, 1}, 1, 4, true},
		{"min at end", []int{5, 6, 0}, 0, 6, true},
		{"max at end", []int{5, 1, 9}, 1, 9, true},

		// --- value range ---
		{"negatives", []int{-3, -1, -7}, -7, -1, true},
		{"crosses zero", []int{-1, 0, 1}, -1, 1, true},
		{"extremes", []int{math.MaxInt, math.MinInt}, math.MinInt, math.MaxInt, true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			min, max, ok := MinMax(tt.xs)
			if ok != tt.wantOK {
				t.Fatalf("MinMax(%v) ok = %v, want %v", tt.xs, ok, tt.wantOK)
			}
			if !ok {
				return
			}
			if min != tt.wantMin || max != tt.wantMax {
				t.Errorf("MinMax(%v) = (%d, %d), want (%d, %d)", tt.xs, min, max, tt.wantMin, tt.wantMax)
			}
		})
	}
}
```

**Run it:**

```bash
go test ./ex04
```

**Output:**

```
ok  	l02/ex04	0.308s
```

**Takeaway:** the checklist is: **nil · empty · single · two · all-equal · duplicates · sorted · reverse-sorted · negatives · extremes**. Write it before you write the interesting cases.

---

## 5. 100% coverage, still wrong

`🟢 easy` · *go test -cover*

This is the example to remember. Below is a binary search with the most common binary-search bug
there is, and a table of four sensible-looking cases. Every case passes. `go test -cover` reports
**100.0% of statements**. The function is still broken — it cannot find the first or last element of
a slice, because the loop exits before checking a range that has narrowed to one.

An 8-line oracle over a tiny domain catches it instantly.

**Steps:**

1. Read `search.go` and try to spot the bug before reading on.
2. Run the table with `-cover` — note 100.0%, all green.
3. Run the oracle test and read the two counterexamples it prints.

**`search.go`**

```go
package algo

// Search returns the index of target in the sorted slice xs, or -1.
//
// This implementation has a bug. It is the single most common binary-search bug
// there is, and the tests in search_test.go do not catch it -- while reporting
// 100% statement coverage.
func Search(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo < hi { // <-- the bug lives here
		mid := (lo + hi) / 2
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

import "testing"

// A perfectly reasonable-looking table. Every branch of Search runs at least
// once, so `go test -cover` reports 100.0% -- and every case passes.
func TestSearchTable(t *testing.T) {
	xs := []int{1, 3, 5, 7}

	tests := []struct {
		name   string
		target int
		want   int
	}{
		{"hits mid immediately", 3, 1},
		{"found after moving lo", 5, 2},
		{"absent above range", 9, -1},
		{"absent below range", 0, -1},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Search(xs, tt.target); got != tt.want {
				t.Errorf("Search(%v, %d) = %d, want %d", xs, tt.target, got, tt.want)
			}
		})
	}
}
```

**`oracle_test.go`**

```go
package algo

import "testing"

// linearSearch is the oracle: too slow to ship, too simple to be wrong.
func linearSearch(xs []int, target int) int {
	for i, v := range xs {
		if v == target {
			return i
		}
	}
	return -1
}

// Exhaustive over a tiny domain -- no randomness needed. Every element of the
// slice, plus every value that isn't in it.
func TestSearchAgainstOracle(t *testing.T) {
	xs := []int{1, 3, 5, 7}

	for target := 0; target <= 8; target++ {
		want := linearSearch(xs, target)
		if got := Search(xs, target); got != want {
			t.Errorf("Search(%v, %d) = %d, oracle says %d", xs, target, got, want)
		}
	}
}
```

**Run the table with coverage:**

```bash
go test -cover -run 'TestSearchTable' ./ex05
```

```
ok  	l02/ex05	0.344s	coverage: 100.0% of statements
```

**Now run the oracle:**

```bash
go test -run 'TestSearchAgainstOracle' ./ex05
```

```
--- FAIL: TestSearchAgainstOracle (0.00s)
    oracle_test.go:23: Search([1 3 5 7], 1) = -1, oracle says 0
    oracle_test.go:23: Search([1 3 5 7], 7) = -1, oracle says 3
FAIL
FAIL	l02/ex05	0.283s
FAIL
```

The bug: `for lo < hi` exits when the range narrows to exactly one element, without ever comparing it.
So the first and last elements — the ones a binary search reaches last — are never found. The fix is
`for lo <= hi`. The four table cases all happened to terminate before that point.

**Takeaway:** coverage answers "which lines ran", never "were the assertions right". A 4-case table can be 100% covered and 0% correct.

---

## 6. Benchmarks in a test file

`🟢 easy` · *go test -bench -benchmem*

Benchmarks live in `_test.go` files too, as `func BenchmarkXxx(b *testing.B)`. Two flags matter from
day one: `-run='^$'` (match no test, so only benchmarks run) and `-benchmem` (add the allocation
columns). `b.Loop` is the Go 1.24+ loop form — it calibrates the iteration count and keeps the call
from being optimized away.

**Steps:**

1. Write two benchmarks: one that allocates nothing, one that allocates a lot.
2. Run with `-bench=. -benchmem -run='^$'`.
3. Read all five columns of the output.

**`sum.go`**

```go
package algo

// Sum adds every element of xs.
func Sum(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

// Collect returns a new slice holding the first n integers, without telling
// append how big the result will be.
func Collect(n int) []int {
	var out []int
	for i := 0; i < n; i++ {
		out = append(out, i)
	}
	return out
}
```

**`sum_test.go`**

```go
package algo

import "testing"

// Benchmarks live in _test.go files too, as func BenchmarkXxx(b *testing.B).

var benchInput = Collect(1000)

func BenchmarkSum(b *testing.B) {
	// b.Loop (Go 1.24+) calibrates the iteration count and keeps the call alive.
	for b.Loop() {
		Sum(benchInput)
	}
}

// b.ReportAllocs adds the B/op and allocs/op columns for this benchmark even
// without the -benchmem flag.
func BenchmarkCollect(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		Collect(1000)
	}
}
```

**Run it:**

```bash
go test -bench=. -benchmem -run='^$' ./ex06
```

**Sample output** (timings vary):

```
goos: darwin
goarch: arm64
pkg: l02/ex06
cpu: Apple M4
BenchmarkSum-10        	 4499440	       242.6 ns/op	       0 B/op	       0 allocs/op
BenchmarkCollect-10    	  658558	      1815 ns/op	   25208 B/op	      12 allocs/op
PASS
ok  	l02/ex06	2.640s
```

Reading a row: `BenchmarkSum-10` is the name plus `GOMAXPROCS`; `4499440` is how many iterations it
ran to get a stable measurement; then time per operation, bytes allocated per operation, and number of
allocations per operation. `Collect` needs **12 allocations** to build a 1000-element slice it could
have built in 1 — that's the growth story, measured.

**Takeaway:** `-run='^$'` skips tests, `-benchmem` adds the two columns that catch the mistakes Big-O can't see.

---

> Next tier: [🟡 medium](2-medium.md) — controls, sweeps, benchstat, `-race`, fuzzing.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
