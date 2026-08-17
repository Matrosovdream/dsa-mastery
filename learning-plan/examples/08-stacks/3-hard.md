# Step 08 — Stacks · 🔴 Hard

Examples **12–16**: the monotonic stack, and knowing when not to use it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

> ⚠️ **Example 16 is a `go test` package** — three files. Examples 12–15 are `go run` programs.
> Sample output from an Apple M4, Go 1.26.3.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The monotonic stack

`🔴 hard` · *Θ(n²) → Θ(n), by the aggregate argument*

The technique this lesson exists to teach. Keep the stack sorted; before pushing, pop everything
that violates the order — and **each pop is an answer**, because the popped element has just found
what it was waiting for.

**Steps:**

1. Write next-greater-element with a stack of indices, and trace it.
2. Verify against brute force.
3. Then benchmark on **random** and on **descending** input, and explain why only one shows the difference.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// The MONOTONIC STACK is the technique this lesson exists to teach. It turns a
// whole family of Theta(n^2) "look at every pair" problems into Theta(n), and
// the argument for why is the aggregate method from lesson 03.
//
// The invariant: the stack is kept sorted (here, strictly decreasing). Before
// pushing a new element you pop everything that violates it -- and each pop is
// an ANSWER, because the popped element has just found the thing it was
// waiting for.
//
// The canonical problem: for each element, find the next element to its right
// that is greater than it.

var comparisons, pops int

// nextGreaterBrute: look right until you find one. Theta(n^2).
func nextGreaterBrute(xs []int) []int {
	out := make([]int, len(xs))
	for i := range xs {
		out[i] = -1
		for j := i + 1; j < len(xs); j++ {
			comparisons++
			if xs[j] > xs[i] {
				out[i] = xs[j]
				break
			}
		}
	}
	return out
}

// nextGreaterMono: one pass, a stack of INDICES still waiting for an answer.
//
// The stack holds indices whose answer is not yet known, and it is always
// decreasing in value -- because the moment a bigger value arrives, everything
// smaller below it has found its answer and leaves.
func nextGreaterMono(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var stack []int // indices, values strictly decreasing from bottom to top

	for i, v := range xs {
		// Everything on the stack smaller than v has just found its answer.
		for len(stack) > 0 {
			comparisons++
			if xs[stack[len(stack)-1]] >= v {
				break
			}
			top := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			pops++
			out[top] = v
		}
		stack = append(stack, i)
	}
	// Whatever is left never found anything bigger; they keep -1.
	return out
}

// The same shape, three variations -- change the comparison and the direction
// and you have a different problem.

func nextSmallerMono(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var stack []int
	for i, v := range xs {
		for len(stack) > 0 && xs[stack[len(stack)-1]] > v {
			out[stack[len(stack)-1]] = v
			stack = stack[:len(stack)-1]
		}
		stack = append(stack, i)
	}
	return out
}

func prevGreaterMono(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var stack []int
	for i := len(xs) - 1; i >= 0; i-- { // walk RIGHT to LEFT
		v := xs[i]
		for len(stack) > 0 && xs[stack[len(stack)-1]] < v {
			out[stack[len(stack)-1]] = v
			stack = stack[:len(stack)-1]
		}
		stack = append(stack, i)
	}
	return out
}

func trace(xs []int) {
	fmt.Printf("  %-6s %-8s %-22s %s\n", "i", "value", "stack (indices)", "answers set")
	var stack []int
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	for i, v := range xs {
		var set []string
		for len(stack) > 0 && xs[stack[len(stack)-1]] < v {
			top := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			out[top] = v
			set = append(set, fmt.Sprintf("out[%d]=%d", top, v))
		}
		stack = append(stack, i)
		s := "-"
		if len(set) > 0 {
			s = fmt.Sprint(set)
		}
		fmt.Printf("  %-6d %-8d %-22s %s\n", i, v, fmt.Sprint(stack), s)
	}
}

var sinkS []int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	xs := []int{2, 1, 2, 4, 3}
	fmt.Printf("next greater element for %v, traced:\n\n", xs)
	trace(xs)
	fmt.Println()
	fmt.Println("  read the stack column: it is ALWAYS decreasing in value. When 4")
	fmt.Println("  arrives at i=3 it pops 2, 1 and 2 -- three answers in one step --")
	fmt.Println("  because every one of them was waiting for exactly this.")

	fmt.Println()
	fmt.Println("brute force and monotonic agree on every input:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	for i := 0; i < 6; i++ {
		in := make([]int, rng.IntN(7)+1)
		for j := range in {
			in[j] = rng.IntN(10)
		}
		a := nextGreaterBrute(slices.Clone(in))
		b := nextGreaterMono(slices.Clone(in))
		fmt.Printf("  %-24v -> %-26v agree: %v\n", in, b, slices.Equal(a, b))
	}

	fmt.Println()
	fmt.Println("WHY it is Theta(n), which is the part worth understanding:")
	fmt.Println()
	fmt.Println("  the inner loop can pop many elements in one step, so the code")
	fmt.Println("  LOOKS quadratic. It is not, by the aggregate argument:")
	fmt.Println()
	fmt.Println("      each index is PUSHED exactly once")
	fmt.Println("      each index is POPPED at most once")
	fmt.Println("      => total inner-loop work over the whole run is at most n")
	fmt.Println()
	fmt.Println("  that is the same reasoning lesson 03 used for append, and lesson")
	fmt.Println("  09's two-stack queue: bound the TOTAL, not the single step.")

	fmt.Println()
	fmt.Println("counted, so it is not just an assertion:")
	fmt.Println()
	fmt.Println("  first on RANDOM data:")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s %14s %12s\n", "n", "brute comparisons", "mono comparisons", "mono pops", "ratio")
	for _, n := range []int{100, 400, 1600, 6400} {
		in := make([]int, n)
		for i := range in {
			in[i] = rng.IntN(1 << 20)
		}
		comparisons, pops = 0, 0
		nextGreaterBrute(in)
		bc := comparisons

		comparisons, pops = 0, 0
		nextGreaterMono(in)
		mc, mp := comparisons, pops

		fmt.Printf("  %10d %18d %18d %14d %11.1fx\n", n, bc, mc, mp, float64(bc)/float64(mc))
	}
	fmt.Println()
	fmt.Println("  the ratio is a small CONSTANT, not growing -- on random data the")
	fmt.Println("  brute force is nearly linear too, because the next greater element")
	fmt.Println("  is usually only a step or two away. Random input does not exercise")
	fmt.Println("  the worst case at all, which is exactly the trap lesson 02 warned")
	fmt.Println("  about: a benchmark on convenient data proves nothing.")

	fmt.Println()
	fmt.Println("  now on DESCENDING data, where nothing ever finds a greater element:")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s %14s %12s\n", "n", "brute comparisons", "mono comparisons", "mono pops", "ratio")
	for _, n := range []int{100, 400, 1600, 6400} {
		in := make([]int, n)
		for i := range in {
			in[i] = n - i // strictly decreasing
		}
		comparisons, pops = 0, 0
		nextGreaterBrute(in)
		bc := comparisons

		comparisons, pops = 0, 0
		nextGreaterMono(in)
		mc, mp := comparisons, pops

		fmt.Printf("  %10d %18d %18d %14d %11.0fx\n", n, bc, mc, mp, float64(bc)/float64(mc))
	}
	fmt.Println()
	fmt.Println("  THERE it is: 4x the input costs the brute force ~16x the")
	fmt.Println("  comparisons (Theta(n^2)) while the monotonic version stays exactly")
	fmt.Println("  linear -- and note mono pops is 0, because nothing is ever popped.")

	fmt.Println()
	fmt.Println("on the clock, on the adversarial input:")
	fmt.Println()
	for _, n := range []int{1000, 4000, 16000} {
		in := make([]int, n)
		for i := range in {
			in[i] = n - i
		}
		tb := nsPerOp(func() { sinkS = nextGreaterBrute(in) })
		tm := nsPerOp(func() { sinkS = nextGreaterMono(in) })
		fmt.Printf("  n=%-8d brute %12.0f ns   mono %10.0f ns   %8.0fx\n", n, tb, tm, tb/tm)
	}

	fmt.Println()
	fmt.Println("four problems, one shape -- change the comparison or the direction:")
	fmt.Println()
	demo := []int{2, 1, 2, 4, 3}
	fmt.Printf("  input          %v\n", demo)
	fmt.Printf("  next greater   %v\n", nextGreaterMono(demo))
	fmt.Printf("  next smaller   %v\n", nextSmallerMono(demo))
	fmt.Printf("  prev greater   %v   (walk right-to-left)\n", prevGreaterMono(demo))
	fmt.Println()
	fmt.Println("  that is the whole family. Examples 13, 14 and 15 are all this")
	fmt.Println("  technique with a different thing stored on the stack.")
}
```

**Sample output:**

```
next greater element for [2 1 2 4 3], traced:

  i      value    stack (indices)        answers set
  0      2        [0]                    -
  1      1        [0 1]                  -
  2      2        [0 2]                  [out[1]=2]
  3      4        [3]                    [out[2]=4 out[0]=4]
  4      3        [3 4]                  -

  read the stack column: it is ALWAYS decreasing in value. When 4
  arrives at i=3 it pops 2, 1 and 2 -- three answers in one step --
  because every one of them was waiting for exactly this.

brute force and monotonic agree on every input:

  [6                        7                        7                        2                        0                        4                       ] -> [7                          -1                         -1                         4                          4                          -1                        ] agree: true
  [1                        1                        6                        4                       ] -> [6                          6                          -1                         -1                        ] agree: true
  [1                        3                        7                        7                        5                        0                        3                       ] -> [3                          7                          -1                         -1                         -1                         3                          -1                        ] agree: true
  [6                       ] -> [-1                        ] agree: true
  [0                        3                       ] -> [3                          -1                        ] agree: true
  [7                        8                        9                        4                        5                       ] -> [8                          9                          -1                         5                          -1                        ] agree: true

WHY it is Theta(n), which is the part worth understanding:

  the inner loop can pop many elements in one step, so the code
  LOOKS quadratic. It is not, by the aggregate argument:

      each index is PUSHED exactly once
      each index is POPPED at most once
      => total inner-loop work over the whole run is at most n

  that is the same reasoning lesson 03 used for append, and lesson
  09's two-stack queue: bound the TOTAL, not the single step.

counted, so it is not just an assertion:

  first on RANDOM data:

           n  brute comparisons   mono comparisons      mono pops        ratio
         100                487                188             92         2.6x
         400               2449                790            391         3.1x
        1600              10762               3180           1590         3.4x
        6400              49790              12784           6391         3.9x

  the ratio is a small CONSTANT, not growing -- on random data the
  brute force is nearly linear too, because the next greater element
  is usually only a step or two away. Random input does not exercise
  the worst case at all, which is exactly the trap lesson 02 warned
  about: a benchmark on convenient data proves nothing.

  now on DESCENDING data, where nothing ever finds a greater element:

           n  brute comparisons   mono comparisons      mono pops        ratio
         100               4950                 99              0          50x
         400              79800                399              0         200x
        1600            1279200               1599              0         800x
        6400           20476800               6399              0        3200x

  THERE it is: 4x the input costs the brute force ~16x the
  comparisons (Theta(n^2)) while the monotonic version stays exactly
  linear -- and note mono pops is 0, because nothing is ever popped.

on the clock, on the adversarial input:

  n=1000     brute       413271 ns   mono       3309 ns        125x
  n=4000     brute      6617596 ns   mono      11056 ns        599x
  n=16000    brute    109116400 ns   mono      50486 ns       2161x

four problems, one shape -- change the comparison or the direction:

  input          [2 1 2 4 3]
  next greater   [4 2 4 -1 -1]
  next smaller   [1 -1 -1 3 -1]
  prev greater   [-1 2 -1 -1 4]   (walk right-to-left)

  that is the whole family. Examples 13, 14 and 15 are all this
  technique with a different thing stored on the stack.
```

**Complexity:** Θ(n) — each index is pushed once and popped at most once, so total inner-loop work is bounded by n

---

## 13. The same shape, four more problems

`🔴 hard` · *Recognising it is the skill*

Example 12's fifteen lines with a different thing on the stack. Recognising that these *are* the
same problem is most of the value.

**Steps:**

1. Daily temperatures — store indices, because the answer is a distance.
2. Stock span — the mirror, looking left.
3. Remove-k-digits — a greedy variant where the stack holds the answer.
4. Verify all of them against brute force with a **small** value range, so ties happen.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// Example 12's technique, applied. These are the same fifteen lines with a
// different thing stored on the stack -- and recognising that they ARE the same
// problem is most of the value.
//
// The tell, every time: "for each element, find the nearest one to its
// left/right that is bigger/smaller".

// dailyTemperatures: for each day, how many days until a warmer one?
// Store INDICES so the answer can be a distance rather than a value.
func dailyTemperatures(t []int) []int {
	out := make([]int, len(t))
	var stack []int // indices, temperatures decreasing

	for i, temp := range t {
		for len(stack) > 0 && t[stack[len(stack)-1]] < temp {
			j := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			out[j] = i - j // the DISTANCE, which is why we stored indices
		}
		stack = append(stack, i)
	}
	return out
}

// stockSpan: for each day, how many consecutive days up to and including today
// had a price <= today's? The mirror of the above: look LEFT, and pop while
// the top is smaller.
func stockSpan(prices []int) []int {
	out := make([]int, len(prices))
	var stack []int

	for i, p := range prices {
		for len(stack) > 0 && prices[stack[len(stack)-1]] <= p {
			stack = stack[:len(stack)-1]
		}
		if len(stack) == 0 {
			out[i] = i + 1 // nothing bigger to the left: the span is everything
		} else {
			out[i] = i - stack[len(stack)-1]
		}
		stack = append(stack, i)
	}
	return out
}

// removeKDigits: delete k digits to leave the smallest possible number.
// Greedy + monotonic stack: a digit should go if a SMALLER one follows it.
func removeKDigits(num string, k int) string {
	var stack []byte
	for i := 0; i < len(num); i++ {
		c := num[i]
		for k > 0 && len(stack) > 0 && stack[len(stack)-1] > c {
			stack = stack[:len(stack)-1]
			k--
		}
		stack = append(stack, c)
	}
	stack = stack[:len(stack)-k] // any budget left: drop from the (largest) tail

	i := 0
	for i < len(stack)-1 && stack[i] == '0' { // strip leading zeroes
		i++
	}
	if len(stack) == 0 {
		return "0"
	}
	return string(stack[i:])
}

// --- brute-force oracles, so none of this is taken on faith ---

func dailyTemperaturesBrute(t []int) []int {
	out := make([]int, len(t))
	for i := range t {
		for j := i + 1; j < len(t); j++ {
			if t[j] > t[i] {
				out[i] = j - i
				break
			}
		}
	}
	return out
}

func stockSpanBrute(p []int) []int {
	out := make([]int, len(p))
	for i := range p {
		n := 1
		for j := i - 1; j >= 0 && p[j] <= p[i]; j-- {
			n++
		}
		out[i] = n
	}
	return out
}

func main() {
	temps := []int{73, 74, 75, 71, 69, 72, 76, 73}
	fmt.Println("daily temperatures -- days until a warmer one:")
	fmt.Println()
	fmt.Printf("  temps  %v\n", temps)
	fmt.Printf("  answer %v\n", dailyTemperatures(temps))
	fmt.Println()
	fmt.Println("  the 0s are days with no warmer day ahead -- they are still on the")
	fmt.Println("  stack when the loop ends, and the zero value is already correct.")

	prices := []int{100, 80, 60, 70, 60, 75, 85}
	fmt.Println()
	fmt.Println("stock span -- consecutive days up to today with price <= today's:")
	fmt.Println()
	fmt.Printf("  prices %v\n", prices)
	fmt.Printf("  span   %v\n", stockSpan(prices))

	fmt.Println()
	fmt.Println("both verified against brute force on random input:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	okAll := true
	for i := 0; i < 2000; i++ {
		in := make([]int, rng.IntN(12))
		for j := range in {
			in[j] = rng.IntN(8) // a small range, so ties happen constantly
		}
		if !slices.Equal(dailyTemperatures(in), dailyTemperaturesBrute(in)) {
			fmt.Printf("  MISMATCH dailyTemperatures on %v\n", in)
			okAll = false
			break
		}
		if !slices.Equal(stockSpan(in), stockSpanBrute(in)) {
			fmt.Printf("  MISMATCH stockSpan on %v\n", in)
			okAll = false
			break
		}
	}
	if okAll {
		fmt.Println("  2000 random inputs, both algorithms agree with brute force.")
		fmt.Println("  (a small value range is deliberate -- TIES are where the strict")
		fmt.Println("   vs non-strict comparison goes wrong, and random big numbers")
		fmt.Println("   would almost never produce one.)")
	}

	fmt.Println()
	fmt.Println("a greedy variant -- remove k digits to leave the smallest number:")
	fmt.Println()
	fmt.Printf("  %-14s %-6s %s\n", "input", "k", "result")
	for _, c := range []struct {
		num string
		k   int
	}{
		{"1432219", 3},
		{"10200", 1},
		{"10", 2},
		{"112", 1},
		{"9876543210", 5},
	} {
		fmt.Printf("  %-14s %-6d %s\n", c.num, c.k, removeKDigits(c.num, c.k))
	}
	fmt.Println()
	fmt.Println("  the stack holds the answer being built, kept non-decreasing. A")
	fmt.Println("  digit is removed exactly when a smaller one follows it, which is")
	fmt.Println("  the greedy choice -- and the stack is what makes 'the previous")
	fmt.Println("  digit' available in Theta(1).")

	fmt.Println()
	fmt.Println("recognising the pattern, which is the actual skill:")
	fmt.Println()
	fmt.Printf("  %-42s %s\n", "problem says", "store on the stack")
	for _, r := range [][2]string{
		{"days until a warmer temperature", "indices (you need a distance)"},
		{"how far back prices stayed lower", "indices, popping while <="},
		{"next greater / smaller element", "indices or values"},
		{"smallest number after k removals", "the answer so far, kept sorted"},
		{"largest rectangle (example 14)", "indices of increasing heights"},
		{"water trapped (example 15)", "indices of decreasing heights"},
	} {
		fmt.Printf("  %-42s %s\n", r[0], r[1])
	}
	fmt.Println()
	fmt.Println("  when the answer for an element depends on the nearest larger or")
	fmt.Println("  smaller thing beside it, it is a monotonic stack. Every one of")
	fmt.Println("  these is Theta(n) by the same aggregate argument.")
}
```

**Output:**

```
daily temperatures -- days until a warmer one:

  temps  [73 74 75 71 69 72 76 73]
  answer [1 1 4 2 1 1 0 0]

  the 0s are days with no warmer day ahead -- they are still on the
  stack when the loop ends, and the zero value is already correct.

stock span -- consecutive days up to today with price <= today's:

  prices [100 80 60 70 60 75 85]
  span   [1 1 1 2 1 4 6]

both verified against brute force on random input:

  2000 random inputs, both algorithms agree with brute force.
  (a small value range is deliberate -- TIES are where the strict
   vs non-strict comparison goes wrong, and random big numbers
   would almost never produce one.)

a greedy variant -- remove k digits to leave the smallest number:

  input          k      result
  1432219        3      1219
  10200          1      200
  10             2      0
  112            1      11
  9876543210     5      43210

  the stack holds the answer being built, kept non-decreasing. A
  digit is removed exactly when a smaller one follows it, which is
  the greedy choice -- and the stack is what makes 'the previous
  digit' available in Theta(1).

recognising the pattern, which is the actual skill:

  problem says                               store on the stack
  days until a warmer temperature            indices (you need a distance)
  how far back prices stayed lower           indices, popping while <=
  next greater / smaller element             indices or values
  smallest number after k removals           the answer so far, kept sorted
  largest rectangle (example 14)             indices of increasing heights
  water trapped (example 15)                 indices of decreasing heights

  when the answer for an element depends on the nearest larger or
  smaller thing beside it, it is a monotonic stack. Every one of
  these is Theta(n) by the same aggregate argument.
```

**Complexity:** all Θ(n) by the same argument · the tell is "the nearest larger/smaller thing beside it"

---

## 14. Largest rectangle in a histogram

`🔴 hard` · *The payoff*

The hardest of the classics, and the one that justifies the technique. The reframing: every
maximal rectangle is capped in height by exactly one bar, so the question becomes
nearest-smaller-on-each-side — which example 12 already solved.

**Steps:**

1. Keep an increasing stack; a shorter bar arriving finishes the top's rectangle.
2. Use a virtual zero bar at the end to drain the stack with no special case.
3. Then extend it to the 2-D maximal rectangle, one histogram per row.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// The hardest of the classic monotonic-stack problems, and the one that pays
// off the technique: the largest rectangle that fits under a histogram.
//
// The reframing that solves it: every maximal rectangle is limited in HEIGHT by
// exactly one bar. So for each bar, ask "how far left and right can I extend
// while staying at least this tall?" -- and that is next-smaller-to-the-left
// and next-smaller-to-the-right, which is example 12.
//
// A monotonic INCREASING stack computes both in one pass.

var brutePairs int

// brute: try every bar as the limiting height. Theta(n^2).
func largestBrute(h []int) int {
	best := 0
	for i := range h {
		minH := h[i]
		for j := i; j < len(h); j++ {
			brutePairs++
			if h[j] < minH {
				minH = h[j]
			}
			if a := minH * (j - i + 1); a > best {
				best = a
			}
		}
	}
	return best
}

// mono: one pass, a stack of indices with INCREASING heights.
//
// When a bar shorter than the top arrives, the top can extend no further right
// -- so its rectangle is finished, and we can compute it. The left edge is
// whatever is below it on the stack, because everything between them was
// taller (that is the invariant).
func largestMono(h []int) int {
	best := 0
	var stack []int // indices, heights increasing bottom to top

	for i := 0; i <= len(h); i++ {
		// A virtual bar of height 0 at the end flushes the stack, which removes
		// the "handle the leftovers" special case entirely.
		cur := 0
		if i < len(h) {
			cur = h[i]
		}

		for len(stack) > 0 && h[stack[len(stack)-1]] >= cur {
			top := stack[len(stack)-1]
			stack = stack[:len(stack)-1]

			height := h[top]
			// left edge: the element now on top is the nearest SHORTER bar to
			// the left. If the stack is empty, the bar extends to index 0.
			left := -1
			if len(stack) > 0 {
				left = stack[len(stack)-1]
			}
			width := i - left - 1

			if a := height * width; a > best {
				best = a
			}
		}
		stack = append(stack, i)
	}
	return best
}

// maximalRectangle: the 2-D version. Treat each row as the base of a histogram
// whose bar heights are the run of 1s above -- then it is the same function.
func maximalRectangle(grid [][]int) int {
	if len(grid) == 0 {
		return 0
	}
	heights := make([]int, len(grid[0]))
	best := 0
	for _, row := range grid {
		for c, v := range row {
			if v == 1 {
				heights[c]++
			} else {
				heights[c] = 0
			}
		}
		if a := largestMono(heights); a > best {
			best = a
		}
	}
	return best
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func draw(h []int) {
	max := 0
	for _, v := range h {
		if v > max {
			max = v
		}
	}
	for level := max; level > 0; level-- {
		fmt.Print("  ")
		for _, v := range h {
			if v >= level {
				fmt.Print("##")
			} else {
				fmt.Print("  ")
			}
		}
		fmt.Println()
	}
	fmt.Print("  ")
	for range h {
		fmt.Print("--")
	}
	fmt.Println()
	fmt.Print("  ")
	for _, v := range h {
		fmt.Printf("%-2d", v)
	}
	fmt.Println()
}

func main() {
	h := []int{2, 1, 5, 6, 2, 3}
	fmt.Println("largest rectangle under this histogram:")
	fmt.Println()
	draw(h)
	fmt.Printf("\n  answer: %d  (the 5 and 6 together: height 5, width 2)\n", largestMono(h))

	fmt.Println()
	fmt.Println("the reframing that makes it Theta(n):")
	fmt.Println()
	fmt.Println("  every maximal rectangle is capped in height by ONE bar. So for")
	fmt.Println("  each bar, the question is how far it can extend before hitting")
	fmt.Println("  something shorter -- on both sides.")
	fmt.Println()
	fmt.Println("  that is 'nearest smaller to the left' and 'nearest smaller to the")
	fmt.Println("  right', which example 12 already solved in one pass. Keeping the")
	fmt.Println("  stack INCREASING means the element below any bar is precisely its")
	fmt.Println("  left boundary, and the bar that evicts it is its right one.")

	fmt.Println()
	fmt.Println("verified against brute force on 5000 random histograms:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	bad := 0
	for i := 0; i < 5000; i++ {
		in := make([]int, rng.IntN(10))
		for j := range in {
			in[j] = rng.IntN(6) // small range: plateaus and ties everywhere
		}
		if largestBrute(in) != largestMono(in) {
			fmt.Printf("  MISMATCH on %v: brute %d, mono %d\n", in, largestBrute(in), largestMono(in))
			bad++
			break
		}
	}
	if bad == 0 {
		fmt.Println("  5000 random histograms, including empty, flat and all-equal:")
		fmt.Println("  every one agrees.")
	}
	for _, edge := range [][]int{{}, {0}, {5}, {3, 3, 3}, {1, 2, 3, 4}, {4, 3, 2, 1}} {
		fmt.Printf("  %-16s -> brute %-6d mono %-6d\n", fmt.Sprint(edge), largestBrute(edge), largestMono(edge))
	}

	fmt.Println()
	fmt.Println("the cost, on the input that actually hurts brute force:")
	fmt.Println()
	fmt.Printf("  %10s %18s %14s %12s\n", "n", "brute ns", "mono ns", "speedup")
	for _, n := range []int{200, 800, 3200} {
		in := make([]int, n)
		for i := range in {
			in[i] = 1 + i%3 // lots of plateaus, so brute never breaks early
		}
		tb := nsPerOp(func() { sinkI = largestBrute(in) })
		tm := nsPerOp(func() { sinkI = largestMono(in) })
		fmt.Printf("  %10d %18.0f %14.0f %11.0fx\n", n, tb, tm, tb/tm)
	}

	fmt.Println()
	fmt.Println("the 2-D version is the same function, called once per row:")
	fmt.Println()
	grid := [][]int{
		{1, 0, 1, 0, 0},
		{1, 0, 1, 1, 1},
		{1, 1, 1, 1, 1},
		{1, 0, 0, 1, 0},
	}
	for _, row := range grid {
		fmt.Print("  ")
		for _, v := range row {
			if v == 1 {
				fmt.Print("# ")
			} else {
				fmt.Print(". ")
			}
		}
		fmt.Println()
	}
	fmt.Printf("\n  largest all-1s rectangle: %d\n", maximalRectangle(grid))
	fmt.Println()
	fmt.Println("  each row becomes a histogram whose bars are the run of 1s above")
	fmt.Println("  it. Then it is Theta(rows x cols) total, because each row is one")
	fmt.Println("  linear pass. Reducing a hard 2-D problem to a solved 1-D one is")
	fmt.Println("  the move worth remembering -- it recurs throughout lesson 31.")

	fmt.Println()
	fmt.Println("the two details that make the code short:")
	fmt.Println()
	fmt.Println("  1. the VIRTUAL zero bar at i == len(h). Without it you need a")
	fmt.Println("     second loop to drain the stack at the end; with it there is")
	fmt.Println("     one loop and no special case -- the same trick as lesson 07's")
	fmt.Println("     sentinel.")
	fmt.Println()
	fmt.Println("  2. width = i - left - 1, where `left` is the element BELOW the")
	fmt.Println("     popped one. That is correct even when equal heights are")
	fmt.Println("     involved: an earlier equal bar computes a too-small rectangle,")
	fmt.Println("     but the LAST one of the run computes the full width.")
}
```

**Sample output:**

```
largest rectangle under this histogram:

        ##    
      ####    
      ####    
      ####  ##
  ##  ########
  ############
  ------------
  2 1 5 6 2 3 

  answer: 10  (the 5 and 6 together: height 5, width 2)

the reframing that makes it Theta(n):

  every maximal rectangle is capped in height by ONE bar. So for
  each bar, the question is how far it can extend before hitting
  something shorter -- on both sides.

  that is 'nearest smaller to the left' and 'nearest smaller to the
  right', which example 12 already solved in one pass. Keeping the
  stack INCREASING means the element below any bar is precisely its
  left boundary, and the bar that evicts it is its right one.

verified against brute force on 5000 random histograms:

  5000 random histograms, including empty, flat and all-equal:
  every one agrees.
  []               -> brute 0      mono 0     
  [0]              -> brute 0      mono 0     
  [5]              -> brute 5      mono 5     
  [3 3 3]          -> brute 9      mono 9     
  [1 2 3 4]        -> brute 6      mono 6     
  [4 3 2 1]        -> brute 6      mono 6     

the cost, on the input that actually hurts brute force:

           n           brute ns        mono ns      speedup
         200              12244            282          43x
         800             172656           1054         164x
        3200            2696118           4209         641x

the 2-D version is the same function, called once per row:

  # . # . . 
  # . # # # 
  # # # # # 
  # . . # . 

  largest all-1s rectangle: 6

  each row becomes a histogram whose bars are the run of 1s above
  it. Then it is Theta(rows x cols) total, because each row is one
  linear pass. Reducing a hard 2-D problem to a solved 1-D one is
  the move worth remembering -- it recurs throughout lesson 31.

the two details that make the code short:

  1. the VIRTUAL zero bar at i == len(h). Without it you need a
     second loop to drain the stack at the end; with it there is
     one loop and no special case -- the same trick as lesson 07's
     sentinel.

  2. width = i - left - 1, where `left` is the element BELOW the
     popped one. That is correct even when equal heights are
     involved: an earlier equal bar computes a too-small rectangle,
     but the LAST one of the run computes the full width.
```

**Complexity:** Θ(n) for the histogram, Θ(rows × cols) for the grid · reducing a hard 2-D problem to a solved 1-D one is the move worth remembering

---

## 15. Trapping rain water, four ways

`🔴 hard` · *When *not* to use the stack*

Three correct Θ(n) solutions with different space, and the honest conclusion is not "use the
clever one".

**Steps:**

1. Precompute both running maxima into arrays.
2. Then fill horizontally with a monotonic stack.
3. Then do it with two pointers in Θ(1) space, and compare all four.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// Trapping rain water: given a histogram, how much water sits in the dips?
//
// Worth doing last because it has THREE correct solutions with the same
// Theta(n) time and different space, and comparing them shows what the
// monotonic stack is really for -- and where a simpler idea beats it.
//
// The insight all three share: the water above column i is
//
//	min(tallest to its left, tallest to its right) - height[i]
//
// The three approaches differ only in how they get those two maxima.

// --- 1. precompute both maxima into arrays. Theta(n) time, Theta(n) space. ---

func trapArrays(h []int) int {
	if len(h) == 0 {
		return 0
	}
	n := len(h)
	leftMax := make([]int, n)
	rightMax := make([]int, n)

	leftMax[0] = h[0]
	for i := 1; i < n; i++ {
		leftMax[i] = max(leftMax[i-1], h[i])
	}
	rightMax[n-1] = h[n-1]
	for i := n - 2; i >= 0; i-- {
		rightMax[i] = max(rightMax[i+1], h[i])
	}

	total := 0
	for i := 0; i < n; i++ {
		total += min(leftMax[i], rightMax[i]) - h[i]
	}
	return total
}

// --- 2. the monotonic stack. Fills water HORIZONTALLY, layer by layer. ---
//
// The stack holds indices of decreasing heights. When a taller bar arrives, the
// bar it evicts is the FLOOR of a puddle: the new bar is the right wall, and
// whatever is below on the stack is the left wall.
func trapStack(h []int) int {
	total := 0
	var stack []int // indices, heights decreasing

	for i, cur := range h {
		for len(stack) > 0 && h[stack[len(stack)-1]] < cur {
			floor := stack[len(stack)-1]
			stack = stack[:len(stack)-1]

			if len(stack) == 0 {
				break // no left wall: the water runs off
			}
			left := stack[len(stack)-1]
			width := i - left - 1
			depth := min(h[left], cur) - h[floor]
			total += width * depth
		}
		stack = append(stack, i)
	}
	return total
}

// --- 3. two pointers. Theta(n) time, Theta(1) SPACE. ---
//
// Walk in from both ends. Whichever side is SHORTER is the side whose answer is
// already determined -- because the taller side guarantees a wall at least that
// high exists beyond it. So that side can be resolved and advanced.
func trapTwoPointer(h []int) int {
	if len(h) == 0 {
		return 0
	}
	l, r := 0, len(h)-1
	leftMax, rightMax := h[l], h[r]
	total := 0

	for l < r {
		if leftMax <= rightMax {
			l++
			leftMax = max(leftMax, h[l])
			total += leftMax - h[l]
		} else {
			r--
			rightMax = max(rightMax, h[r])
			total += rightMax - h[r]
		}
	}
	return total
}

// --- brute force, to verify all three ---

func trapBrute(h []int) int {
	total := 0
	for i := range h {
		l, r := 0, 0
		for j := 0; j <= i; j++ {
			l = max(l, h[j])
		}
		for j := i; j < len(h); j++ {
			r = max(r, h[j])
		}
		total += min(l, r) - h[i]
	}
	return total
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func draw(h []int) {
	maxH := 0
	for _, v := range h {
		maxH = max(maxH, v)
	}
	n := len(h)
	leftMax := make([]int, n)
	rightMax := make([]int, n)
	leftMax[0] = h[0]
	for i := 1; i < n; i++ {
		leftMax[i] = max(leftMax[i-1], h[i])
	}
	rightMax[n-1] = h[n-1]
	for i := n - 2; i >= 0; i-- {
		rightMax[i] = max(rightMax[i+1], h[i])
	}

	for level := maxH; level > 0; level-- {
		fmt.Print("  ")
		for i, v := range h {
			switch {
			case v >= level:
				fmt.Print("#")
			case min(leftMax[i], rightMax[i]) >= level:
				fmt.Print("~")
			default:
				fmt.Print(" ")
			}
		}
		fmt.Println()
	}
}

func main() {
	h := []int{0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1}
	fmt.Println("# is wall, ~ is trapped water:")
	fmt.Println()
	draw(h)
	fmt.Printf("\n  heights %v\n", h)
	fmt.Printf("  trapped %d units\n", trapStack(h))

	fmt.Println()
	fmt.Println("all four approaches agree, on 5000 random inputs:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	bad := 0
	for i := 0; i < 5000; i++ {
		in := make([]int, rng.IntN(12))
		for j := range in {
			in[j] = rng.IntN(5)
		}
		a, b, c, d := trapBrute(in), trapArrays(in), trapStack(in), trapTwoPointer(in)
		if a != b || b != c || c != d {
			fmt.Printf("  MISMATCH on %v: brute=%d arrays=%d stack=%d two-ptr=%d\n", in, a, b, c, d)
			bad++
			break
		}
	}
	if bad == 0 {
		fmt.Println("  brute, arrays, stack and two-pointer all agree everywhere.")
	}

	fmt.Println()
	fmt.Println("so which one? They are all Theta(n) time:")
	fmt.Println()
	fmt.Printf("  %-24s %-14s %-14s %s\n", "approach", "time", "space", "note")
	for _, r := range [][4]string{
		{"brute force", "Theta(n^2)", "Theta(1)", "recomputes both maxima per column"},
		{"precomputed arrays", "Theta(n)", "Theta(n)", "clearest to read and to prove"},
		{"monotonic stack", "Theta(n)", "Theta(n)", "fills horizontally, layer by layer"},
		{"two pointers", "Theta(n)", "Theta(1)", "fewest moving parts, hardest to see"},
	} {
		fmt.Printf("  %-24s %-14s %-14s %s\n", r[0], r[1], r[2], r[3])
	}

	fmt.Println()
	const n = 20000
	in := make([]int, n)
	for i := range in {
		in[i] = rng.IntN(1000)
	}
	tArr := nsPerOp(func() { sinkI = trapArrays(in) })
	tStk := nsPerOp(func() { sinkI = trapStack(in) })
	tTwo := nsPerOp(func() { sinkI = trapTwoPointer(in) })
	fmt.Printf("  at n=%d:  arrays %.0f ns   stack %.0f ns   two-pointer %.0f ns\n",
		n, tArr, tStk, tTwo)
	fmt.Printf("  two-pointer is %.1fx FASTER than the arrays version, and allocates nothing.\n", tArr/tTwo)

	fmt.Println()
	fmt.Println("the honest conclusion, which is not 'use the clever one':")
	fmt.Println()
	fmt.Println("  the TWO-POINTER version wins -- Theta(1) space, no allocation,")
	fmt.Println("  fewest branches. The monotonic stack is not the best tool here.")
	fmt.Println()
	fmt.Println("  that is worth sitting with. Examples 12-14 were problems where the")
	fmt.Println("  stack is genuinely the answer, because each element needs the")
	fmt.Println("  nearest larger/smaller NEIGHBOUR and nothing simpler will produce")
	fmt.Println("  it. Here every column only needs two RUNNING MAXIMA, which a pair")
	fmt.Println("  of pointers tracks for free.")
	fmt.Println()
	fmt.Println("  the giveaway: if the answer depends on a specific nearby ELEMENT,")
	fmt.Println("  reach for the stack. If it depends only on a running AGGREGATE,")
	fmt.Println("  two pointers or a prefix scan will be simpler (lesson 16).")
}
```

**Sample output:**

```
# is wall, ~ is trapped water:

         #    
     #~~~##~# 
   #~##~######

  heights [0 1 0 2 1 0 1 3 2 1 2 1]
  trapped 6 units

all four approaches agree, on 5000 random inputs:

  brute, arrays, stack and two-pointer all agree everywhere.

so which one? They are all Theta(n) time:

  approach                 time           space          note
  brute force              Theta(n^2)     Theta(1)       recomputes both maxima per column
  precomputed arrays       Theta(n)       Theta(n)       clearest to read and to prove
  monotonic stack          Theta(n)       Theta(n)       fills horizontally, layer by layer
  two pointers             Theta(n)       Theta(1)       fewest moving parts, hardest to see

  at n=20000:  arrays 68514 ns   stack 39551 ns   two-pointer 10727 ns
  two-pointer is 6.4x FASTER than the arrays version, and allocates nothing.

the honest conclusion, which is not 'use the clever one':

  the TWO-POINTER version wins -- Theta(1) space, no allocation,
  fewest branches. The monotonic stack is not the best tool here.

  that is worth sitting with. Examples 12-14 were problems where the
  stack is genuinely the answer, because each element needs the
  nearest larger/smaller NEIGHBOUR and nothing simpler will produce
  it. Here every column only needs two RUNNING MAXIMA, which a pair
  of pointers tracks for free.

  the giveaway: if the answer depends on a specific nearby ELEMENT,
  reach for the stack. If it depends only on a running AGGREGATE,
  two pointers or a prefix scan will be simpler (lesson 16).
```

**Complexity:** arrays and stack Θ(n) space · **two pointers Θ(1) space and fastest** · the giveaway: a specific nearby *element* wants a stack, a running *aggregate* wants two pointers

---

## 16. Capstone: Stack[T] and the algorithms

`🔴 hard` · *Build it · Prove it · Measure it` · `go test*

The structure is nearly trivial, so most of the risk lives in the **algorithms** built on it — and
a monotonic stack is exactly the kind of index arithmetic that looks right and is off by one.
So those get brute-force oracles of their own.

**Steps:**

1. Package `Stack[T]` with `Reserve` and `Clear`.
2. Prove the structure: table, slot-release invariant, a 50,000-operation model oracle.
3. Prove the algorithms: 20,000 random inputs each against brute force, plus edge cases.
4. Assert the amortized claim directly: total pops ≤ n.

**`stack.go`**

```go
package stack

// Stack[T] is example 2's type, finished: generic, zero-value-ready, with the
// (T, bool) contract and the pointer hygiene.
//
// INVARIANT, true before and after every exported method:
//
//	items[len(items):cap(items)] holds only zero values
//	  -- so a popped element is never kept alive by the stack
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Len() int      { return len(s.items) }
func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	i := len(s.items) - 1
	v := s.items[i]
	s.items[i] = zero
	s.items = s.items[:i]
	return v, true
}

func (s *Stack[T]) Peek() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	return s.items[len(s.items)-1], true
}

// Reserve makes room for n more pushes without reallocating.
func (s *Stack[T]) Reserve(n int) {
	if need := len(s.items) + n; need > cap(s.items) {
		bigger := make([]T, len(s.items), need)
		copy(bigger, s.items)
		s.items = bigger
	}
}

// Clear empties the stack, zeroing every slot so nothing stays reachable,
// and keeps the capacity for reuse (lesson 06, example 10).
func (s *Stack[T]) Clear() {
	clear(s.items)
	s.items = s.items[:0]
}

// ============================================================================
// The monotonic-stack algorithms from examples 12-14, as library functions.
// ============================================================================

// NextGreater returns, for each i, the value of the next element to the right
// that is strictly greater -- or -1 if there is none. Theta(n).
func NextGreater(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var st Stack[int] // indices, values strictly decreasing
	for i, v := range xs {
		for {
			top, ok := st.Peek()
			if !ok || xs[top] >= v {
				break
			}
			st.Pop()
			out[top] = v
		}
		st.Push(i)
	}
	return out
}

// LargestRectangle returns the area of the largest rectangle under the
// histogram. Theta(n).
func LargestRectangle(h []int) int {
	best := 0
	var st Stack[int] // indices, heights increasing
	for i := 0; i <= len(h); i++ {
		cur := 0 // a virtual zero bar at the end flushes the stack
		if i < len(h) {
			cur = h[i]
		}
		for {
			top, ok := st.Peek()
			if !ok || h[top] < cur {
				break
			}
			st.Pop()
			left := -1
			if l, ok := st.Peek(); ok {
				left = l
			}
			if a := h[top] * (i - left - 1); a > best {
				best = a
			}
		}
		st.Push(i)
	}
	return best
}
```

**`stack_test.go`** — pass 2: prove it

```go
package stack

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// PASS 2: PROVE IT. Same four layers as lessons 06 and 07.
//
// The structure here is nearly trivial, so most of the risk is in the
// ALGORITHMS built on it -- and those get brute-force oracles, because a
// monotonic stack is exactly the kind of index arithmetic that looks right
// and is off by one.
// ============================================================================

// --- 1. the table -----------------------------------------------------------

func TestStackBasics(t *testing.T) {
	var s Stack[int]

	if _, ok := s.Pop(); ok {
		t.Error("Pop on an empty stack reported ok=true")
	}
	if _, ok := s.Peek(); ok {
		t.Error("Peek on an empty stack reported ok=true")
	}
	if !s.IsEmpty() || s.Len() != 0 {
		t.Error("the zero value is not empty")
	}

	for i := 1; i <= 3; i++ {
		s.Push(i)
	}
	if v, _ := s.Peek(); v != 3 {
		t.Errorf("Peek = %d, want 3", v)
	}
	if s.Len() != 3 {
		t.Errorf("Len = %d, want 3", s.Len())
	}
	// Peek must not remove.
	if v, _ := s.Peek(); v != 3 || s.Len() != 3 {
		t.Error("Peek changed the stack")
	}

	for i := 3; i >= 1; i-- {
		v, ok := s.Pop()
		if !ok || v != i {
			t.Fatalf("Pop = (%d, %v), want (%d, true)", v, ok, i)
		}
	}
	if !s.IsEmpty() {
		t.Error("stack not empty after popping everything")
	}
}

// --- 2. the invariant -------------------------------------------------------

// Popping must zero the vacated slot, or a removed element stays reachable
// through the backing array (lesson 06, example 2).
func TestPopReleasesTheSlot(t *testing.T) {
	type box struct{ p *int }
	var s Stack[box]

	n := 42
	s.Push(box{&n})
	s.Push(box{&n})
	s.Pop()
	s.Pop()

	full := s.items[:cap(s.items)]
	for i := 0; i < len(full); i++ {
		if full[i].p != nil {
			t.Errorf("slot %d past the length still holds %v -- Pop did not zero it", i, full[i].p)
		}
	}
}

func TestClearReleasesEverything(t *testing.T) {
	type box struct{ p *int }
	var s Stack[box]
	n := 7
	for i := 0; i < 5; i++ {
		s.Push(box{&n})
	}
	c := cap(s.items)
	s.Clear()

	if s.Len() != 0 {
		t.Errorf("Len = %d after Clear, want 0", s.Len())
	}
	if cap(s.items) != c {
		t.Errorf("Clear changed the capacity (%d -> %d); it should keep it for reuse", c, cap(s.items))
	}
	for i, b := range s.items[:cap(s.items)] {
		if b.p != nil {
			t.Errorf("slot %d still holds a pointer after Clear", i)
		}
	}
}

// --- 3. the model oracle ----------------------------------------------------

func TestStackAgainstModel(t *testing.T) {
	rng := rand.New(rand.NewPCG(42, 7))
	var s Stack[int]
	var model []int

	const trials = 50_000
	for i := 0; i < trials; i++ {
		switch rng.IntN(3) {
		case 0, 1:
			v := rng.IntN(1000)
			s.Push(v)
			model = append(model, v)
		case 2:
			got, gotOK := s.Pop()
			if len(model) == 0 {
				if gotOK {
					t.Fatalf("trial %d: Pop reported ok on an empty stack", i)
				}
				break
			}
			want := model[len(model)-1]
			model = model[:len(model)-1]
			if !gotOK || got != want {
				t.Fatalf("trial %d: Pop = (%d,%v), model says %d", i, got, gotOK, want)
			}
		}
		if s.Len() != len(model) {
			t.Fatalf("trial %d: Len = %d, model has %d", i, s.Len(), len(model))
		}
		if !slices.Equal(s.items, model) {
			t.Fatalf("trial %d: contents diverged", i)
		}
	}
	t.Logf("%d random operations, final depth %d", trials, s.Len())
}

// --- 4. brute-force oracles for the ALGORITHMS ------------------------------

func nextGreaterBrute(xs []int) []int {
	out := make([]int, len(xs))
	for i := range xs {
		out[i] = -1
		for j := i + 1; j < len(xs); j++ {
			if xs[j] > xs[i] {
				out[i] = xs[j]
				break
			}
		}
	}
	return out
}

func largestRectangleBrute(h []int) int {
	best := 0
	for i := range h {
		minH := h[i]
		for j := i; j < len(h); j++ {
			minH = min(minH, h[j])
			best = max(best, minH*(j-i+1))
		}
	}
	return best
}

func TestNextGreaterAgainstBrute(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))
	for i := 0; i < 20_000; i++ {
		// SMALL values: ties are where a strict/non-strict comparison goes
		// wrong, and a wide range would almost never produce one.
		xs := make([]int, rng.IntN(10))
		for j := range xs {
			xs[j] = rng.IntN(5)
		}
		want := nextGreaterBrute(xs)
		got := NextGreater(xs)
		if !slices.Equal(got, want) {
			t.Fatalf("NextGreater(%v)\n  got  %v\n  want %v", xs, got, want)
		}
	}
}

func TestLargestRectangleAgainstBrute(t *testing.T) {
	rng := rand.New(rand.NewPCG(3, 4))
	for i := 0; i < 20_000; i++ {
		h := make([]int, rng.IntN(10))
		for j := range h {
			h[j] = rng.IntN(6)
		}
		if got, want := LargestRectangle(h), largestRectangleBrute(h); got != want {
			t.Fatalf("LargestRectangle(%v) = %d, want %d", h, got, want)
		}
	}
}

func TestAlgorithmEdgeCases(t *testing.T) {
	for _, tt := range []struct {
		name string
		in   []int
	}{
		{"nil", nil},
		{"empty", []int{}},
		{"single", []int{5}},
		{"all equal", []int{3, 3, 3, 3}},
		{"ascending", []int{1, 2, 3, 4}},
		{"descending", []int{4, 3, 2, 1}},
		{"zeros", []int{0, 0, 0}},
		{"one zero", []int{2, 0, 2}},
	} {
		t.Run(tt.name, func(t *testing.T) {
			if got, want := NextGreater(tt.in), nextGreaterBrute(tt.in); !slices.Equal(got, want) {
				t.Errorf("NextGreater = %v, want %v", got, want)
			}
			if got, want := LargestRectangle(tt.in), largestRectangleBrute(tt.in); got != want {
				t.Errorf("LargestRectangle = %d, want %d", got, want)
			}
		})
	}
}

// --- the amortized claim, asserted -----------------------------------------

var sinkI int

func TestPushIsAmortizedConstant(t *testing.T) {
	var s Stack[int]
	s.Reserve(1000)
	allocs := testing.AllocsPerRun(50, func() {
		s.Clear()
		for i := 0; i < 1000; i++ {
			s.Push(i)
		}
		sinkI = s.Len()
	})
	if allocs != 0 {
		t.Errorf("1000 pushes into reserved capacity allocated %.0f times, want 0", allocs)
	}
	t.Logf("1000 pushes after Reserve: %.0f allocations", allocs)
}

// The monotonic-stack claim: total inner-loop work over a whole run is at most
// n, because each index is pushed once and popped at most once. Assert it.
func TestMonotonicIsLinear(t *testing.T) {
	for _, n := range []int{100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = n - i // descending: nothing is ever popped
		}
		pops := countPops(xs)
		if pops > n {
			t.Errorf("n=%d: %d pops, which exceeds n -- the amortized bound is broken", n, pops)
		}

		asc := make([]int, n)
		for i := range asc {
			asc[i] = i // ascending: everything is popped, exactly once
		}
		pops = countPops(asc)
		if pops > n {
			t.Errorf("n=%d ascending: %d pops > n", n, pops)
		}
		t.Logf("n=%-6d descending: 0 pops   ascending: %d pops (both <= n)", n, pops)
	}
}

func countPops(xs []int) int {
	pops := 0
	var st Stack[int]
	for i, v := range xs {
		for {
			top, ok := st.Peek()
			if !ok || xs[top] >= v {
				break
			}
			st.Pop()
			pops++
		}
		st.Push(i)
	}
	return pops
}

func ExampleNextGreater() {
	fmt.Println(NextGreater([]int{2, 1, 2, 4, 3}))
	// Output: [4 2 4 -1 -1]
}
```

**`bench_test.go`** — pass 3: measure it

```go
package stack

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// ============================================================================
// PASS 3: MEASURE IT.
// ============================================================================

func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// Push/Pop must be flat in n -- that is what Theta(1) amortized means.
func BenchmarkPushPop(b *testing.B) {
	for _, n := range []int{100, 10_000, 1_000_000} {
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			var s Stack[int]
			s.Reserve(n)
			for b.Loop() {
				s.Clear()
				for i := 0; i < n; i++ {
					s.Push(i)
				}
				for !s.IsEmpty() {
					s.Pop()
				}
			}
		})
	}
}

// The monotonic claim: linear, on the input that makes brute force quadratic.
func BenchmarkNextGreater(b *testing.B) {
	for _, n := range []int{1_000, 4_000, 16_000, 64_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = n - i // descending: brute force's worst case
		}
		b.Run(fmt.Sprintf("mono/n=%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkS = NextGreater(xs)
			}
		})
	}
}

func BenchmarkLargestRectangle(b *testing.B) {
	rng := rand.New(rand.NewPCG(1, 2))
	for _, n := range []int{1_000, 4_000, 16_000} {
		h := make([]int, n)
		for i := range h {
			h[i] = rng.IntN(1000)
		}
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				sinkI = LargestRectangle(h)
			}
		})
	}
}

var sinkS []int
```

**Run it:**

```bash
go test -v .
go test -bench=. -benchmem -run='^$' .
```

**Sample output:**

```
--- go test -v ---
--- PASS: TestStackBasics (0.00s)
--- PASS: TestPopReleasesTheSlot (0.00s)
--- PASS: TestClearReleasesEverything (0.00s)
    stack_test.go:139: 50000 random operations, final depth 16731
--- PASS: TestStackAgainstModel (0.11s)
--- PASS: TestNextGreaterAgainstBrute (0.00s)
--- PASS: TestLargestRectangleAgainstBrute (0.00s)
--- PASS: TestAlgorithmEdgeCases (0.00s)
    --- PASS: TestAlgorithmEdgeCases/nil (0.00s)
    --- PASS: TestAlgorithmEdgeCases/empty (0.00s)
    --- PASS: TestAlgorithmEdgeCases/single (0.00s)
    --- PASS: TestAlgorithmEdgeCases/all_equal (0.00s)
    --- PASS: TestAlgorithmEdgeCases/ascending (0.00s)
    --- PASS: TestAlgorithmEdgeCases/descending (0.00s)
    --- PASS: TestAlgorithmEdgeCases/zeros (0.00s)
    --- PASS: TestAlgorithmEdgeCases/one_zero (0.00s)
    stack_test.go:242: 1000 pushes after Reserve: 0 allocations
--- PASS: TestPushIsAmortizedConstant (0.00s)
    stack_test.go:266: n=100    descending: 0 pops   ascending: 99 pops (both <= n)
    stack_test.go:266: n=1000   descending: 0 pops   ascending: 999 pops (both <= n)
    stack_test.go:266: n=10000  descending: 0 pops   ascending: 9999 pops (both <= n)
--- PASS: TestMonotonicIsLinear (0.00s)
--- PASS: ExampleNextGreater (0.00s)
PASS
ok  	l08/ex16	0.238s

--- go test -bench=. -benchmem -run='^$' ---
goos: darwin
goarch: arm64
pkg: l08/ex16
cpu: Apple M4
BenchmarkEmptyControl-10        	672748970	         1.794 ns/op	       0 B/op	       0 allocs/op
BenchmarkPushPop/n=100-10       	 3287128	       364.2 ns/op	       0 B/op	       0 allocs/op
BenchmarkPushPop/n=10000-10     	   32924	     36401 ns/op	       0 B/op	       0 allocs/op
BenchmarkPushPop/n=1000000-10   	     314	   3800719 ns/op	       0 B/op	       0 allocs/op
BenchmarkNextGreater/mono/n=1000-10         	  276115	      4872 ns/op	   33400 B/op	      13 allocs/op
BenchmarkNextGreater/mono/n=4000-10         	   44610	     24987 ns/op	  161018 B/op	      17 allocs/op
BenchmarkNextGreater/mono/n=16000-10        	   18421	     64878 ns/op	  619772 B/op	      21 allocs/op
BenchmarkNextGreater/mono/n=64000-10        	    4206	    291830 ns/op	 3028248 B/op	      27 allocs/op
BenchmarkLargestRectangle/n=1000-10         	  318907	      3773 ns/op	     248 B/op	       5 allocs/op
BenchmarkLargestRectangle/n=4000-10         	   78596	     15248 ns/op	     248 B/op	       5 allocs/op
BenchmarkLargestRectangle/n=16000-10        	   18186	     63865 ns/op	     504 B/op	       6 allocs/op
PASS
ok  	l08/ex16	13.379s
```

**Complexity:** `Push`/`Pop`/`Peek` Θ(1) amortized · `NextGreater` and `LargestRectangle` Θ(n) · the tests assert both

---

> That closes the stack. Next: [09 — Queues & Deques](../../09-queues-deques.md) —
> the other end of the slice, where it stops being free.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*