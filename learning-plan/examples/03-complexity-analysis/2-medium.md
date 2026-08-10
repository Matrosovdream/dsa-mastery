# Step 03 — Complexity Analysis · 🟡 Medium

Examples **7–11**: amortized analysis, space (including the call stack), and recurrences.

Examples 7, 8, 10 and 11 are **deterministic** — they count, they don't time. Example 9 measures real
goroutine stack memory, so its numbers move a little between runs.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Why append is amortized O(1)

`🟡 medium` · *The aggregate method*

`append` copies the entire backing array when it resizes, which is Θ(n). So how is it "O(1)"? Count
the **total** work across all n appends and divide. The answer converges to a constant, and this
example also shows the capacity sequence Go really uses — which is not plain doubling.

**Steps:**

1. Detect each resize by watching `cap` change, and record how many elements were copied.
2. Tabulate copies-per-append as n grows by 10×.
3. Print the capacity sequence, then compare against a hypothetical `+1` growth policy.

```go
package main

import "fmt"

// growthOf appends n elements and records exactly what the runtime did:
// every capacity change, and how many elements had to be copied to the new array.
func growthOf(n int) (copies int, caps []int) {
	var xs []int
	for i := 0; i < n; i++ {
		before := cap(xs)
		oldLen := len(xs)
		xs = append(xs, i)
		if cap(xs) != before {
			// The runtime allocated a bigger array and copied the old contents.
			copies += oldLen
			caps = append(caps, cap(xs))
		}
	}
	return copies, caps
}

func main() {
	fmt.Println("append is O(n) when it resizes. So how is it 'O(1) amortized'?")
	fmt.Println("Answer: count the TOTAL work over all n appends, then divide.")
	fmt.Println()

	fmt.Printf("%10s %14s %14s %12s\n", "n", "total copies", "copies/append", "resizes")
	for _, n := range []int{10, 100, 1_000, 100_000, 1_000_000} {
		copies, caps := growthOf(n)
		fmt.Printf("%10d %14d %14.2f %12d\n", n, copies, float64(copies)/float64(n), len(caps))
	}

	fmt.Println()
	fmt.Println("copies/append converges to a constant instead of growing -> O(1) amortized.")
	fmt.Println("resizes grow like log(n), not n: each one is rarer than the last, in proportion")
	fmt.Println("to how much more it costs.")

	_, caps := growthOf(5000)
	fmt.Println()
	fmt.Println("the capacity sequence Go actually used:")
	fmt.Printf("  %v\n", caps)
	fmt.Println()
	fmt.Println("small slices double; above ~256 the factor tapers toward 1.25 to waste less memory.")
	fmt.Println("The trick is not the number 2 -- it is that the factor is a CONSTANT above 1.")
	fmt.Println("Growing by a factor means the copies you just paid for are spread over a")
	fmt.Println("proportional number of free appends. Grow by a fixed +1 instead and every")
	fmt.Println("single append copies everything: the total becomes n^2/2.")

	// Prove that last claim.
	fixedGrowth := 0
	for i := 0; i < 10000; i++ {
		fixedGrowth += i // a +1 growth policy copies i elements on every append
	}
	copies, _ := growthOf(10000)
	fmt.Println()
	fmt.Printf("at n=10000:  doubling policy = %d copies,  +1 policy = %d copies  (%.0fx worse)\n",
		copies, fixedGrowth, float64(fixedGrowth)/float64(copies))
}
```

**Output:**

```
append is O(n) when it resizes. So how is it 'O(1) amortized'?
Answer: count the TOTAL work over all n appends, then divide.

         n   total copies  copies/append      resizes
        10             12           1.20            3
       100            127           1.27            8
      1000           1871           1.87           12
    100000         402079           4.02           28
   1000000        4154015           4.15           38

copies/append converges to a constant instead of growing -> O(1) amortized.
resizes grow like log(n), not n: each one is rarer than the last, in proportion
to how much more it costs.

the capacity sequence Go actually used:
  [4 8 16 32 64 128 256 512 848 1280 1792 2560 3408 5120]

small slices double; above ~256 the factor tapers toward 1.25 to waste less memory.
The trick is not the number 2 -- it is that the factor is a CONSTANT above 1.
Growing by a factor means the copies you just paid for are spread over a
proportional number of free appends. Grow by a fixed +1 instead and every
single append copies everything: the total becomes n^2/2.

at n=10000:  doubling policy = 32412 copies,  +1 policy = 49995000 copies  (1542x worse)
```

Read the capacity sequence: `4 8 16 32 64 128 256` is exact doubling, then `512 848 1280 1792 2560
3408 5120` — a factor drifting toward **1.25**. That is why copies/append settles near 4 rather than
the textbook 1: with growth factor `g` the geometric series sums to `n/(g−1)`, and `1/(1.25−1) = 4`.

**Complexity:** one append Θ(n) worst case · n appends Θ(n) total → **Θ(1) amortized** · the `+1` policy is Θ(n²) total, 1542× more copies at n=10,000

---

## 8. Amortized is not average, and is not worst case

`🟡 medium` · *Three different claims about the same operation*

The same 100,000 appends, viewed as a distribution instead of a total. **99.975%** of them are free;
**25** of them copy; one copies **88,064 elements**. "Amortized O(1)" and "worst case O(n)" are both
true, simultaneously, about the same line of code.

**Steps:**

1. Record the copy cost of every individual append.
2. Report the amortized cost, the worst single cost, and how many appends were expensive.
3. State the aggregate-method bound, and work out why the constant is ~4 and not ~1.

```go
package main

import "fmt"

// perAppendCost returns the copy cost of every single append, in order.
func perAppendCost(n int) []int {
	costs := make([]int, 0, n)
	var xs []int
	for i := 0; i < n; i++ {
		before := cap(xs)
		oldLen := len(xs)
		xs = append(xs, i)
		if cap(xs) != before {
			costs = append(costs, oldLen) // resized: copied oldLen elements
		} else {
			costs = append(costs, 0) // free
		}
	}
	return costs
}

func main() {
	const n = 100_000
	costs := perAppendCost(n)

	total, max, expensive := 0, 0, 0
	maxAt := 0
	for i, c := range costs {
		total += c
		if c > 0 {
			expensive++
		}
		if c > max {
			max, maxAt = c, i
		}
	}

	fmt.Printf("%d appends:\n\n", n)
	fmt.Printf("  amortized cost (total/n)     %10.2f copies\n", float64(total)/float64(n))
	fmt.Printf("  worst single append          %10d copies  (at append #%d)\n", max, maxAt)
	fmt.Printf("  appends that copied anything %10d  (%.3f%% of them)\n",
		expensive, 100*float64(expensive)/float64(n))
	fmt.Printf("  appends that were free       %10d\n", n-expensive)

	fmt.Println()
	fmt.Println("AMORTIZED is not AVERAGE and it is definitely not WORST CASE:")
	fmt.Printf("  - amortized O(1)  : any sequence of n appends costs O(n) total. Guaranteed.\n")
	fmt.Printf("  - worst case O(n) : one unlucky append copied %d elements.\n", max)
	fmt.Println()
	fmt.Println("both statements are true at the same time. Amortized analysis bounds the")
	fmt.Println("SEQUENCE; it says nothing about any individual operation.")

	fmt.Println()
	fmt.Println("why it matters: if you care about throughput, amortized O(1) is the number.")
	fmt.Println("If you care about p99 LATENCY, that one expensive append is your tail --")
	fmt.Println("and example 15 measures it in nanoseconds.")

	fmt.Println()
	fmt.Println("the aggregate method, in one line:")
	fmt.Printf("  total copies %d <= 5n = %d, so cost per operation <= 5 = O(1).\n", total, 5*n)
	fmt.Println()
	fmt.Println("why 5 and not 2? With a growth factor g, the copies form a geometric series")
	fmt.Println("summing to n/(g-1). Textbook doubling (g=2) gives n, i.e. 1 copy per append.")
	fmt.Printf("Go tapers g toward 1.25 for large slices, so 1/(1.25-1) = 4 -- and we measured %.2f.\n",
		float64(total)/float64(n))
	fmt.Println("A smaller factor wastes less memory and copies more. Both are still O(1):")
	fmt.Println("what matters for amortization is only that g is a CONSTANT greater than 1.")
}
```

**Output:**

```
100000 appends:

  amortized cost (total/n)           4.02 copies
  worst single append               88064 copies  (at append #88064)
  appends that copied anything         25  (0.025% of them)
  appends that were free            99975

AMORTIZED is not AVERAGE and it is definitely not WORST CASE:
  - amortized O(1)  : any sequence of n appends costs O(n) total. Guaranteed.
  - worst case O(n) : one unlucky append copied 88064 elements.

both statements are true at the same time. Amortized analysis bounds the
SEQUENCE; it says nothing about any individual operation.

why it matters: if you care about throughput, amortized O(1) is the number.
If you care about p99 LATENCY, that one expensive append is your tail --
and example 15 measures it in nanoseconds.

the aggregate method, in one line:
  total copies 402076 <= 5n = 500000, so cost per operation <= 5 = O(1).

why 5 and not 2? With a growth factor g, the copies form a geometric series
summing to n/(g-1). Textbook doubling (g=2) gives n, i.e. 1 copy per append.
Go tapers g toward 1.25 for large slices, so 1/(1.25-1) = 4 -- and we measured 4.02.
A smaller factor wastes less memory and copies more. Both are still O(1):
what matters for amortization is only that g is a CONSTANT greater than 1.
```

Three numbers, three different truths about one operation: **4.02** (amortized), **88,064** (worst
case), **0.025%** (how often it matters). Which one you quote depends entirely on whether you are
optimizing throughput or latency.

**Complexity:** amortized Θ(1) · worst-case single op Θ(n) · number of expensive ops Θ(log n) · total Θ(n)

---

## 9. The call stack is space

`🟡 medium` · *Measuring recursion depth in bytes*

Recursion allocates memory without any line of code saying so. This measures how much: ~33–52 bytes
per frame, so a million-deep recursion is holding **33 MB** of goroutine stack.

The measurement detail matters as much as the result: each depth is measured in a **fresh goroutine**.
Go grows stacks but is slow to shrink them, so measuring on the main goroutine means the first deep
call sets a high-water mark that hides every later measurement behind it.

**Steps:**

1. Read `runtime.MemStats.StackInuse` as a baseline after a GC.
2. Recurse to a given depth inside a new goroutine, and read the counter at the bottom.
3. Divide by the depth to get bytes per frame.

```go
package main

import (
	"fmt"
	"runtime"
)

// sumRecursive uses one stack frame per element: O(n) space, invisible in the code.
func sumRecursive(xs []int) int {
	if len(xs) == 0 {
		return 0
	}
	return xs[0] + sumRecursive(xs[1:])
}

// sumIterative does the same arithmetic in O(1) space.
func sumIterative(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

// stackInUse reports how many bytes of goroutine stack the runtime holds.
func stackInUse() uint64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.StackInuse
}

func deepest(depth int, atBottom func()) {
	if depth == 0 {
		atBottom()
		return
	}
	deepest(depth-1, atBottom)
}

// measureAtDepth recurses in a FRESH goroutine, which starts with a small stack.
// Measuring on the main goroutine does not work: Go grows stacks but is slow to
// shrink them, so the first deep call sets a high-water mark that hides every
// later measurement behind it.
func measureAtDepth(depth int) uint64 {
	var peak uint64
	done := make(chan struct{})
	go func() {
		defer close(done)
		deepest(depth, func() { peak = stackInUse() })
	}()
	<-done
	return peak
}

func main() {
	runtime.GC()
	base := stackInUse()

	fmt.Println("a recursion's stack frames are memory, even though no line of code")
	fmt.Println("in it says 'allocate'. Each frame is measured here in a fresh goroutine:")
	fmt.Println()
	fmt.Printf("goroutine stacks in use, baseline: %8.1f KB\n\n", float64(base)/1024)
	fmt.Printf("%12s %14s %18s\n", "depth", "stacks (KB)", "bytes per frame")
	for _, depth := range []int{1_000, 10_000, 100_000, 1_000_000} {
		peak := measureAtDepth(depth)
		fmt.Printf("%12d %14.1f %18.0f\n",
			depth, float64(peak)/1024, float64(peak-base)/float64(depth))
		runtime.GC() // let the finished goroutine's stack go back
	}

	xs := make([]int, 100_000)
	for i := range xs {
		xs[i] = 1
	}
	fmt.Println()
	fmt.Println("both of these compute the same sum in the same O(n) time:")
	fmt.Printf("  sumRecursive = %d   (O(n) space -- 100,000 live frames)\n", sumRecursive(xs))
	fmt.Printf("  sumIterative = %d   (O(1) space)\n", sumIterative(xs))

	fmt.Println()
	fmt.Println("space complexity counts the CALL STACK. It is the cost people forget,")
	fmt.Println("because unlike a make() it never appears in the source.")
	fmt.Println()
	fmt.Println("Go grows goroutine stacks dynamically (8 KB at first, doubling, to a 1 GB")
	fmt.Println("default limit), so deep recursion does not fail fast the way it does in C --")
	fmt.Println("it quietly consumes memory instead. Past the limit the runtime does not")
	fmt.Println("panic, it dies: 'fatal error: stack overflow', which no recover() can catch.")
	fmt.Println()
	fmt.Println("rule: recursion depth IS space. Balanced-tree recursion is O(log n) and fine;")
	fmt.Println("recursion down a list or an array is O(n) -- write the loop instead.")
}
```

**Sample output** (stack accounting shifts a little between runs):

```
a recursion's stack frames are memory, even though no line of code
in it says 'allocate'. Each frame is measured here in a fresh goroutine:

goroutine stacks in use, baseline:    352.0 KB

       depth    stacks (KB)    bytes per frame
        1000          384.0                 33
       10000          864.0                 52
      100000         4448.0                 42
     1000000        33120.0                 34

both of these compute the same sum in the same O(n) time:
  sumRecursive = 100000   (O(n) space -- 100,000 live frames)
  sumIterative = 100000   (O(1) space)

space complexity counts the CALL STACK. It is the cost people forget,
because unlike a make() it never appears in the source.

Go grows goroutine stacks dynamically (8 KB at first, doubling, to a 1 GB
default limit), so deep recursion does not fail fast the way it does in C --
it quietly consumes memory instead. Past the limit the runtime does not
panic, it dies: 'fatal error: stack overflow', which no recover() can catch.

rule: recursion depth IS space. Balanced-tree recursion is O(log n) and fine;
recursion down a list or an array is O(n) -- write the loop instead.
```

Bytes-per-frame wobbles (33, 52, 42, 34) because stacks grow by **doubling**, so the measurement
catches a different amount of slack each time. The trend is what matters: **stack memory is linear in
depth**, and at depth 1,000,000 it is 33 MB.

**Complexity:** `sumRecursive` Θ(n) time, **Θ(n) space** · `sumIterative` Θ(n) time, **Θ(1) space** · at ~40 bytes/frame, a 1 GB stack limit is reached around depth 25,000,000

---

## 10. Auxiliary space vs total space

`🟡 medium` · *Θ(1), Θ(log n), Θ(n) side by side*

"Space complexity" nearly always means **auxiliary** space — what the algorithm needs *beyond* its
input. Three sorts, three different answers, and the difference is the entire argument for in-place
sorting when the data is large.

**Steps:**

1. Instrument allocation with a counter that tracks the **peak**, not the total (freed memory doesn't count twice).
2. Run merge sort (buffer), insertion sort (in place), and quicksort (recursion only).
3. Compare how each scales with n.

```go
package main

import "fmt"

// A counter for extra memory: every int the algorithm allocates beyond its input.
var extraInts int

// peakExtra tracks the high-water mark, because auxiliary space is a PEAK,
// not a total -- memory that is freed does not count twice.
var peakExtra int

func alloc(n int) []int {
	extraInts += n
	if extraInts > peakExtra {
		peakExtra = extraInts
	}
	return make([]int, n)
}

func free(n int) { extraInts -= n }

// mergeSort allocates a scratch buffer at every merge: O(n) auxiliary space.
func mergeSort(xs []int) []int {
	if len(xs) <= 1 {
		return xs
	}
	mid := len(xs) / 2
	left := mergeSort(xs[:mid])
	right := mergeSort(xs[mid:])

	out := alloc(len(xs))
	i, j := 0, 0
	for k := range out {
		switch {
		case i >= len(left):
			out[k] = right[j]
			j++
		case j >= len(right):
			out[k] = left[i]
			i++
		case left[i] <= right[j]:
			out[k] = left[i]
			i++
		default:
			out[k] = right[j]
			j++
		}
	}
	if len(left) > 1 {
		free(len(left))
	}
	if len(right) > 1 {
		free(len(right))
	}
	return out
}

// insertionSort sorts in place: O(1) auxiliary space, whatever n is.
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

// quickSortDepth reports the maximum recursion depth -- quicksort's auxiliary
// space is its stack, O(log n) when the pivots split evenly.
func quickSortDepth(xs []int, depth int) int {
	if len(xs) <= 1 {
		return depth
	}
	mid := len(xs) / 2 // pretend the pivot always splits evenly
	l := quickSortDepth(xs[:mid], depth+1)
	r := quickSortDepth(xs[mid:], depth+1)
	if l > r {
		return l
	}
	return r
}

func input(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = (i * 7919) % n
	}
	return xs
}

func main() {
	fmt.Println("'space complexity' almost always means AUXILIARY space:")
	fmt.Println("extra memory beyond the input itself. The input does not count.")
	fmt.Println()

	fmt.Printf("%10s %18s %18s %18s\n", "n", "merge sort peak", "insertion sort", "quicksort depth")
	for _, n := range []int{16, 256, 4096, 65536} {
		extraInts, peakExtra = 0, 0
		mergeSort(input(n))
		mergePeak := peakExtra

		xs := input(n)
		insertionSort(xs) // touches nothing but xs

		depth := quickSortDepth(input(n), 0)

		fmt.Printf("%10d %18d %18d %18d\n", n, mergePeak, 0, depth)
	}

	fmt.Println()
	fmt.Println("  merge sort      O(n) auxiliary  -- the scratch buffer scales with the input")
	fmt.Println("  insertion sort  O(1) auxiliary  -- swaps in place, allocates nothing")
	fmt.Println("  quicksort       O(log n)        -- no buffer, but the recursion IS space")
	fmt.Println()
	fmt.Println("this is the trade behind 'in-place': merge sort is the faster, stable sort,")
	fmt.Println("and it costs a whole extra copy of your data. At 4 GB of input that decides it.")
	fmt.Println()
	fmt.Println("(quicksort's O(log n) holds only while the pivots split evenly. Bad pivots")
	fmt.Println(" give depth n -- O(n) stack, which is the other half of its worst case.)")
}
```

**Output:**

```
'space complexity' almost always means AUXILIARY space:
extra memory beyond the input itself. The input does not count.

         n    merge sort peak     insertion sort    quicksort depth
        16                 32                  0                  4
       256                512                  0                  8
      4096               8192                  0                 12
     65536             131072                  0                 16

  merge sort      O(n) auxiliary  -- the scratch buffer scales with the input
  insertion sort  O(1) auxiliary  -- swaps in place, allocates nothing
  quicksort       O(log n)        -- no buffer, but the recursion IS space

this is the trade behind 'in-place': merge sort is the faster, stable sort,
and it costs a whole extra copy of your data. At 4 GB of input that decides it.

(quicksort's O(log n) holds only while the pivots split evenly. Bad pivots
 give depth n -- O(n) stack, which is the other half of its worst case.)
```

Merge sort's peak is exactly **2n** here (this naive version holds a level's buffers while building
the next); the quicksort depth column is exactly **log₂n**. Both are the classes you'd write down.

**Complexity:** merge sort Θ(n) auxiliary · insertion sort Θ(1) · quicksort Θ(log n) average stack, Θ(n) worst — full treatment in [13 — Efficient Sorts](../../13-efficient-sorts.md)

---

## 11. Recursion trees

`🟡 medium` · *Solving recurrences by drawing them*

A recurrence becomes arithmetic you can do by hand once you draw the tree: level k has `a^k` nodes of
size `n/b^k`. Three recurrences, three completely different shapes — and the shape tells you the
answer without any theorem.

**Steps:**

1. For each recurrence, print nodes / size / work at every level.
2. Ask which level dominates: all equal, or the top, or the bottom?
3. Then handle a recurrence that isn't divide-and-conquer at all.

```go
package main

import "fmt"

// A recursion tree turns a recurrence into arithmetic you can do by hand:
// draw the levels, cost each level, add them up.
//
// For T(n) = a*T(n/b) + f(n):
//   level k has a^k nodes, each of size n/b^k, so it costs a^k * f(n/b^k).
//   the tree is log_b(n) levels deep.

type recurrence struct {
	name  string
	fname string
	a     int
	b     int
	f     func(int) int
}

func (r recurrence) tree(n int) {
	fmt.Printf("\n%s\n  T(n) = %d*T(n/%d) + %s,  n = %d\n\n", r.name, r.a, r.b, r.fname, n)
	fmt.Printf("  %-7s %10s %12s %14s\n", "level", "nodes", "size each", "work at level")

	total := 0
	nodes := 1
	size := n
	level := 0
	for size >= 1 {
		work := nodes * r.f(size)
		total += work
		fmt.Printf("  %-7d %10d %12d %14d\n", level, nodes, size, work)
		nodes *= r.a
		size /= r.b
		level++
	}
	fmt.Printf("  %-7s %10s %12s %14d\n", "", "", "TOTAL", total)
}

func main() {
	n := 64

	recs := []recurrence{
		{"merge sort -- work is the same at every level", "n", 2, 2, func(k int) int { return k }},
		{"binary search -- one path down the tree", "1", 1, 2, func(k int) int { return 1 }},
		{"tree traversal -- work explodes at the leaves", "1", 2, 2, func(k int) int { return 1 }},
	}
	for _, r := range recs {
		r.tree(n)
	}

	fmt.Println("\nreading the three tables:")
	fmt.Printf("  merge sort:     %d levels x n work each  = n log n  (%d = %d*%d)\n", 7, 448, 64, 7)
	fmt.Println("  binary search:  1 unit per level, log n levels  = log n")
	fmt.Println("  tree traversal: the last level dominates -- 64 of the 127 nodes are leaves = n")
	fmt.Println()
	fmt.Println("that is the whole method: if every level costs the same, multiply by the depth;")
	fmt.Println("if one level dominates, that level IS the answer.")

	// A recurrence that is not divide-and-conquer at all.
	fmt.Println("\n----------------------------------------------------------------")
	fmt.Println("\nnot every recurrence is a/b-shaped. Selection sort peels one element:")
	fmt.Println("  T(n) = T(n-1) + n")
	fmt.Println()
	fmt.Printf("  %-10s %14s\n", "n", "expanding")
	fmt.Println("  T(n)       = n + T(n-1)")
	fmt.Println("  T(n-1)     = (n-1) + T(n-2)")
	fmt.Println("  ...")
	fmt.Println("  T(1)       = 1")
	fmt.Println()
	total := 0
	for i := 1; i <= n; i++ {
		total += i
	}
	fmt.Printf("  so T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = %d at n=%d  -> Theta(n^2)\n", total, n)
	fmt.Println()
	fmt.Println("shrinking by a FACTOR gives a log-deep tree; shrinking by a CONSTANT gives")
	fmt.Println("an n-deep one. That single difference is most of the gap between n log n and n^2.")
}
```

**Output:**

```
merge sort -- work is the same at every level
  T(n) = 2*T(n/2) + n,  n = 64

  level        nodes    size each  work at level
  0                1           64             64
  1                2           32             64
  2                4           16             64
  3                8            8             64
  4               16            4             64
  5               32            2             64
  6               64            1             64
                            TOTAL            448

binary search -- one path down the tree
  T(n) = 1*T(n/2) + 1,  n = 64

  level        nodes    size each  work at level
  0                1           64              1
  1                1           32              1
  2                1           16              1
  3                1            8              1
  4                1            4              1
  5                1            2              1
  6                1            1              1
                            TOTAL              7

tree traversal -- work explodes at the leaves
  T(n) = 2*T(n/2) + 1,  n = 64

  level        nodes    size each  work at level
  0                1           64              1
  1                2           32              2
  2                4           16              4
  3                8            8              8
  4               16            4             16
  5               32            2             32
  6               64            1             64
                            TOTAL            127

reading the three tables:
  merge sort:     7 levels x n work each  = n log n  (448 = 64*7)
  binary search:  1 unit per level, log n levels  = log n
  tree traversal: the last level dominates -- 64 of the 127 nodes are leaves = n

that is the whole method: if every level costs the same, multiply by the depth;
if one level dominates, that level IS the answer.

----------------------------------------------------------------

not every recurrence is a/b-shaped. Selection sort peels one element:
  T(n) = T(n-1) + n

  n              expanding
  T(n)       = n + T(n-1)
  T(n-1)     = (n-1) + T(n-2)
  ...
  T(1)       = 1

  so T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = 2080 at n=64  -> Theta(n^2)

shrinking by a FACTOR gives a log-deep tree; shrinking by a CONSTANT gives
an n-deep one. That single difference is most of the gap between n log n and n^2.
```

Three shapes, and they are the only three: **every level equal** (multiply by depth), **root
dominates** (the top level is the answer), **leaves dominate** (the bottom level is the answer).
Example 12 names them Cases 2, 3 and 1.

**Complexity:** `2T(n/2)+n` → Θ(n log n) · `T(n/2)+1` → Θ(log n) · `2T(n/2)+1` → Θ(n) · `T(n−1)+n` → Θ(n²)

---

> Next tier: [🔴 hard](3-hard.md) — the master theorem, hidden quadratics, and measuring complexity for real.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
