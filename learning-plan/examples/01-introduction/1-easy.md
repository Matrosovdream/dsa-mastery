# Step 01 — Introduction · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Everything in this tier is **deterministic** — no clocks, no randomness. Your output should match
these blocks character for character.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. Same data, different structure

`🟢 easy` · *Structure vs algorithm*

The single most important idea in the plan, in fifteen lines: the **question doesn't change**, the
**structure does**, and the complexity follows the structure. A slice must look at every element to
answer "is `x` in here?"; a set hashes once and jumps straight to the answer.

**Steps:**

1. Answer the membership question by scanning a slice.
2. Load the same values into a `map[int]struct{}` — Go's idiomatic set (the empty struct costs 0 bytes).
3. Answer the same question with one lookup, and note what you paid to make that possible.

```go
package main

import "fmt"

// contains answers "is x in the data?" by looking at every element: O(n).
func contains(xs []int, x int) bool {
	for _, v := range xs {
		if v == x {
			return true
		}
	}
	return false
}

func main() {
	data := []int{41, 7, 93, 12, 55, 8, 77, 30}

	// Structure 1: a slice. Membership costs a scan.
	fmt.Println("slice contains 55:", contains(data, 55))
	fmt.Println("slice contains 56:", contains(data, 56))

	// Structure 2: the same values in a set. Membership is one hash lookup.
	set := make(map[int]struct{}, len(data))
	for _, v := range data {
		set[v] = struct{}{}
	}
	_, has55 := set[55]
	_, has56 := set[56]
	fmt.Println("set contains 55:  ", has55)
	fmt.Println("set contains 56:  ", has56)

	fmt.Println()
	fmt.Println("same data, same question, different structure:")
	fmt.Println("  slice -> O(n) per lookup, O(0) to prepare")
	fmt.Println("  set   -> O(1) per lookup, O(n) to prepare once")
}
```

**Output:**

```
slice contains 55: true
slice contains 56: false
set contains 55:   true
set contains 56:   false

same data, same question, different structure:
  slice -> O(n) per lookup, O(0) to prepare
  set   -> O(1) per lookup, O(n) to prepare once
```

**Complexity:** slice lookup O(n) · set lookup O(1) average, O(n) worst · set build O(n) time and O(n) space

---

## 2. Count the work, not the clock

`🟢 easy` · *Best vs worst case*

Before any stopwatch: **instrument the loop**. A comparison counter is reproducible on every machine,
immune to CPU noise, and shows the growth rate directly. It also makes the point that "linear search
is O(n)" is only half a sentence — the *same function* is O(1) in its best case.

**Steps:**

1. Return the comparison count alongside the result.
2. Search for the first element (best case) and for an absent element (worst case).
3. Read the two columns: one is flat, one tracks `n` exactly.

```go
package main

import "fmt"

// searchCounting returns the index of x plus the number of comparisons it took.
// Counting the work is machine-independent: the numbers are the same everywhere.
func searchCounting(xs []int, x int) (idx, comparisons int) {
	for i, v := range xs {
		comparisons++
		if v == x {
			return i, comparisons
		}
	}
	return -1, comparisons
}

func main() {
	fmt.Printf("%8s %14s %14s\n", "n", "best case", "worst case")
	for _, n := range []int{10, 100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		_, best := searchCounting(xs, 0)   // first element
		_, worst := searchCounting(xs, -1) // absent: every element checked
		fmt.Printf("%8d %14d %14d\n", n, best, worst)
	}

	fmt.Println()
	fmt.Println("best case is flat -> O(1); worst case tracks n exactly -> O(n)")
	fmt.Println("this is why Big-O always names a case: the same code is both")
}
```

**Output:**

```
       n      best case     worst case
      10              1             10
     100              1            100
    1000              1           1000
   10000              1          10000

best case is flat -> O(1); worst case tracks n exactly -> O(n)
this is why Big-O always names a case: the same code is both
```

**Complexity:** best O(1) · worst O(n) · average O(n/2) = O(n) · space O(1)

---

## 3. Quadratic vs linear, in operations

`🟢 easy` · *Growth rate — O(n²) vs O(n)*

Two-sum solved twice. Both loops are equally "optimized" — no tricks, no unrolling — and yet at
n=10,000 one does **5,000× more work** than the other. That factor is the algorithm choice, and no
amount of micro-optimization in the brute-force version would close it.

**Steps:**

1. Brute force: check every pair, counting each check.
2. Map version: for each value, ask whether its complement has already been seen.
3. Print the ratio and watch it grow linearly with `n` — that's the signature of O(n²) vs O(n).

```go
package main

import "fmt"

// twoSumBrute checks every pair: O(n^2).
func twoSumBrute(xs []int, target int) (i, j, ops int) {
	for a := 0; a < len(xs); a++ {
		for b := a + 1; b < len(xs); b++ {
			ops++
			if xs[a]+xs[b] == target {
				return a, b, ops
			}
		}
	}
	return -1, -1, ops
}

// twoSumMap remembers what it has already seen: O(n) time, O(n) space.
func twoSumMap(xs []int, target int) (i, j, ops int) {
	seen := make(map[int]int, len(xs))
	for b, v := range xs {
		ops++
		if a, ok := seen[target-v]; ok {
			return a, b, ops
		}
		seen[v] = b
	}
	return -1, -1, ops
}

func main() {
	fmt.Printf("%8s %16s %12s %12s\n", "n", "brute ops", "map ops", "ratio")
	for _, n := range []int{10, 100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		target := 2*n - 3 // the last two elements: the worst case for the brute force

		_, _, bruteOps := twoSumBrute(xs, target)
		_, _, mapOps := twoSumMap(xs, target)
		fmt.Printf("%8d %16d %12d %11.0fx\n", n, bruteOps, mapOps, float64(bruteOps)/float64(mapOps))
	}

	fmt.Println()
	fmt.Println("10x the input costs the brute force ~100x the work, the map ~10x")
	fmt.Println("that gap IS the algorithm choice -- both loops are equally 'optimized'")
}
```

**Output:**

```
       n        brute ops      map ops        ratio
      10               45           10           4x
     100             4950          100          50x
    1000           499500         1000         500x
   10000         49995000        10000        5000x

10x the input costs the brute force ~100x the work, the map ~10x
that gap IS the algorithm choice -- both loops are equally 'optimized'
```

**Complexity:** brute O(n²) time, O(1) space · map O(n) time, O(n) space — the classic time-for-space trade

---

## 4. What the growth curves actually cost

`🟢 easy` · *Feasible input size*

The growth table stops being abstract when you convert it to wall-clock time. At a billion operations
per second, an O(n log n) algorithm chews through a billion items in half a minute; an O(2ⁿ) one needs
**10¹³ years** at n=100. This is the table that tells you which complexity class you're allowed to ship.

**Steps:**

1. Print log n, n, n log n, n² and 2ⁿ side by side for n up to 10,000.
2. Notice 2ⁿ overflowing a `float64` before n reaches 10,000.
3. Convert three representative workloads into human time.

```go
package main

import (
	"fmt"
	"math"
)

// pow2 formats 2^n, which stops fitting in a float64 above n = 1024.
func pow2(n int) string {
	v := math.Pow(2, float64(n))
	if math.IsInf(v, 1) {
		return "overflows float64"
	}
	return fmt.Sprintf("%.2e", v)
}

func main() {
	fmt.Printf("%8s %10s %12s %14s %14s %20s\n", "n", "log2 n", "n", "n log2 n", "n^2", "2^n")
	for _, n := range []int{1, 10, 100, 1000, 10000} {
		f := float64(n)
		lg := math.Log2(f)
		fmt.Printf("%8d %10.1f %12d %14.0f %14.0f %20s\n", n, lg, n, f*lg, f*f, pow2(n))
	}

	fmt.Println()
	fmt.Println("at 1 billion ops/second:")
	for _, row := range []struct {
		name string
		ops  float64
	}{
		{"n log2 n at n=10^9", 1e9 * math.Log2(1e9)},
		{"n^2       at n=10^6", 1e12},
		{"2^n       at n=100", math.Pow(2, 100)},
	} {
		secs := row.ops / 1e9
		fmt.Printf("  %-20s %10.2e ops -> %s\n", row.name, row.ops, humanTime(secs))
	}
}

func humanTime(secs float64) string {
	switch {
	case secs < 60:
		return fmt.Sprintf("%.1f seconds", secs)
	case secs < 3600:
		return fmt.Sprintf("%.1f minutes", secs/60)
	case secs < 86400*365:
		return fmt.Sprintf("%.1f days", secs/86400)
	default:
		return fmt.Sprintf("%.2e years", secs/(86400*365))
	}
}
```

**Output:**

```
       n     log2 n            n       n log2 n            n^2                  2^n
       1        0.0            1              0              1             2.00e+00
      10        3.3           10             33            100             1.02e+03
     100        6.6          100            664          10000             1.27e+30
    1000       10.0         1000           9966        1000000            1.07e+301
   10000       13.3        10000         132877      100000000    overflows float64

at 1 billion ops/second:
  n log2 n at n=10^9     2.99e+10 ops -> 29.9 seconds
  n^2       at n=10^6    1.00e+12 ops -> 16.7 minutes
  2^n       at n=100     1.27e+30 ops -> 4.02e+13 years
```

**Complexity:** the program itself is O(1) — it's a lookup table for every complexity you'll meet later

---

## 5. Trading space for time

`🟢 easy` · *The trade-off triangle*

The trade at the heart of dynamic programming, visible in one counter. Naive Fibonacci recomputes the
same subproblems exponentially often — ~30 million calls at n=35. Handing it a `map` to remember
answers drops that to **69**. You spent O(n) space and bought O(2ⁿ) → O(n) time.

**Steps:**

1. Count calls in the naive recursion.
2. Count calls in the memoized one (the extra calls are the memo *hits*).
3. Watch the naive column explode while the memo column grows linearly.

```go
package main

import "fmt"

var naiveCalls int

// fibNaive recomputes the same subproblems over and over: O(2^n) calls.
func fibNaive(n int) int {
	naiveCalls++
	if n < 2 {
		return n
	}
	return fibNaive(n-1) + fibNaive(n-2)
}

var memoCalls int

// fibMemo spends O(n) space to remember each answer once: O(n) calls.
func fibMemo(n int, memo map[int]int) int {
	memoCalls++
	if n < 2 {
		return n
	}
	if v, ok := memo[n]; ok {
		return v
	}
	v := fibMemo(n-1, memo) + fibMemo(n-2, memo)
	memo[n] = v
	return v
}

func main() {
	fmt.Printf("%4s %16s %12s %12s\n", "n", "naive calls", "memo calls", "fib(n)")
	for _, n := range []int{5, 10, 20, 30, 35} {
		naiveCalls, memoCalls = 0, 0
		got := fibNaive(n)
		fibMemo(n, make(map[int]int, n))
		fmt.Printf("%4d %16d %12d %12d\n", n, naiveCalls, memoCalls, got)
	}

	fmt.Println()
	fmt.Println("the memo buys a drop from O(2^n) to O(n) calls by spending O(n) space")
	fmt.Println("time and space are a currency pair -- you trade one for the other")
}
```

**Output:**

```
   n      naive calls   memo calls       fib(n)
   5               15            9            5
  10              177           19           55
  20            21891           39         6765
  30          2692537           59       832040
  35         29860703           69      9227465

the memo buys a drop from O(2^n) to O(n) calls by spending O(n) space
time and space are a currency pair -- you trade one for the other
```

**Complexity:** naive O(φⁿ) ≈ O(1.618ⁿ) time, O(n) stack · memo O(n) time, O(n) space — revisited in full at [30 — DP I](../../30-dp-one-dimensional.md)

---

> Next tier: [🟡 medium](2-medium.md) — where the clock comes out.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
