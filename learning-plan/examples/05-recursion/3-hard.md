# Step 05 — Recursion & the Call Stack · 🔴 Hard

Examples **12–16**: backtracking, depth as a design constraint, and the program that justifies recursion.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 12 and 16 are **deterministic**. Examples 13–15 measure stack or time, and 15 spawns a child
process. Sample output is from an Apple M4, Go 1.26.3.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Backtracking: choose, explore, un-choose

`🔴 hard` · *Pruning is the whole algorithm*

Backtracking is recursion with an un-do, and the skeleton never changes. What changes — by four orders
of magnitude — is how early you reject a candidate.

**Steps:**

1. Write permutations with the skeleton and a `used` array.
2. Write N-queens with a safety check inside the loop.
3. Write it again with the check moved to the leaves, and count tree nodes for both.

```go
package main

import (
	"fmt"
	"slices"
)

// BACKTRACKING is recursion with an un-do. The skeleton never changes:
//
//   if the partial answer is complete -> record it, return
//   for each candidate at this position:
//       if the candidate is not allowed -> skip it        <- PRUNING
//       CHOOSE     apply it to the partial answer
//       EXPLORE    recurse
//       UN-CHOOSE  undo it
//
// The whole game is pruning: a check that kills a branch before you explore it
// removes an entire subtree of work.

var nodesVisited int

// --- permutations: no pruning beyond "already used" ---

func permute(nums []int, cur []int, used []bool, out *[][]int) {
	nodesVisited++
	if len(cur) == len(nums) {
		*out = append(*out, slices.Clone(cur)) // clone -- example 10
		return
	}
	for i, v := range nums {
		if used[i] {
			continue
		}
		used[i] = true                // choose
		cur = append(cur, v)          //
		permute(nums, cur, used, out) // explore
		cur = cur[:len(cur)-1]        // un-choose
		used[i] = false               //
	}
}

// --- N-queens: pruning is the entire algorithm ---

// safe reports whether a queen at (row, col) conflicts with the queens already
// placed in rows 0..row-1. cols[r] is the column of the queen in row r.
func safe(cols []int, row, col int) bool {
	for r := 0; r < row; r++ {
		c := cols[r]
		if c == col || row-r == col-c || row-r == c-col {
			return false // same column, or same diagonal
		}
	}
	return true
}

func queens(n int, row int, cols []int, count *int) {
	nodesVisited++
	if row == n {
		*count++
		return
	}
	for col := 0; col < n; col++ {
		if !safe(cols, row, col) {
			continue // PRUNE: this subtree cannot contain a solution
		}
		cols[row] = col
		queens(n, row+1, cols, count)
		cols[row] = -1
	}
}

// queensNoPrune explores every arrangement and only checks at the leaves --
// what backtracking looks like with the pruning removed.
func queensNoPrune(n int, row int, cols []int, count *int) {
	nodesVisited++
	if row == n {
		for r := 0; r < n; r++ {
			if !safe(cols[:r+1], r, cols[r]) {
				return
			}
		}
		*count++
		return
	}
	for col := 0; col < n; col++ {
		cols[row] = col
		queensNoPrune(n, row+1, cols, count)
	}
}

func board(cols []int) string {
	s := ""
	for _, c := range cols {
		for j := range cols {
			if j == c {
				s += "Q "
			} else {
				s += ". "
			}
		}
		s += "\n"
	}
	return s
}

func main() {
	fmt.Println("permutations of [1 2 3 4] -- choose / explore / un-choose:")
	nums := []int{1, 2, 3, 4}
	var out [][]int
	nodesVisited = 0
	permute(nums, make([]int, 0, len(nums)), make([]bool, len(nums)), &out)
	fmt.Printf("  %d permutations from %d tree nodes\n", len(out), nodesVisited)
	fmt.Printf("  first six: %v\n", out[:6])

	fmt.Println()
	fmt.Println("N-queens -- the same skeleton, with one pruning check added:")
	fmt.Println()
	fmt.Printf("%5s %14s %18s %14s %12s\n", "n", "solutions", "nodes (pruned)", "nodes (not)", "saved")
	for _, n := range []int{4, 5, 6, 7, 8} {
		cols := make([]int, n)

		nodesVisited = 0
		count := 0
		queens(n, 0, cols, &count)
		pruned := nodesVisited

		nodesVisited = 0
		count2 := 0
		queensNoPrune(n, 0, cols, &count2)
		full := nodesVisited

		if count != count2 {
			panic("the two versions disagree")
		}
		fmt.Printf("%5d %14d %18d %14d %11.1fx\n",
			n, count, pruned, full, float64(full)/float64(pruned))
	}

	fmt.Println()
	fmt.Println("one 8-queens solution:")
	fmt.Println()
	cols := make([]int, 8)
	found := 0
	var first []int
	var collect func(row int)
	collect = func(row int) {
		if found > 0 {
			return
		}
		if row == 8 {
			first = slices.Clone(cols)
			found++
			return
		}
		for col := 0; col < 8; col++ {
			if safe(cols, row, col) {
				cols[row] = col
				collect(row + 1)
			}
		}
	}
	collect(0)
	fmt.Print(board(first))

	fmt.Println()
	fmt.Println("read the N-queens table: at n=8 the pruned search visits 2057 nodes and")
	fmt.Println("the unpruned one visits over 19 million -- for the SAME 92 solutions.")
	fmt.Println()
	fmt.Println("that is the lesson. Backtracking's complexity is not the size of the")
	fmt.Println("search space, it is the size of the space MINUS everything your pruning")
	fmt.Println("check removes. Write the check as early in the loop as you can:")
	fmt.Println("rejecting at depth 2 skips every node below it.")
	fmt.Println()
	fmt.Println("the un-choose step is what makes one buffer safe to reuse across")
	fmt.Println("branches -- and the clone at the leaf is what keeps the recorded")
	fmt.Println("answers from aliasing it (example 10).")
}
```

**Output:**

```
permutations of [1 2 3 4] -- choose / explore / un-choose:
  24 permutations from 65 tree nodes
  first six: [[1 2 3 4] [1 2 4 3] [1 3 2 4] [1 3 4 2] [1 4 2 3] [1 4 3 2]]

N-queens -- the same skeleton, with one pruning check added:

    n      solutions     nodes (pruned)    nodes (not)        saved
    4              2                 17            341        20.1x
    5             10                 54           3906        72.3x
    6              4                153          55987       365.9x
    7             40                552         960800      1740.6x
    8             92               2057       19173961      9321.3x

one 8-queens solution:

Q . . . . . . . 
. . . . Q . . . 
. . . . . . . Q 
. . . . . Q . . 
. . Q . . . . . 
. . . . . . Q . 
. Q . . . . . . 
. . . Q . . . . 

read the N-queens table: at n=8 the pruned search visits 2057 nodes and
the unpruned one visits over 19 million -- for the SAME 92 solutions.

that is the lesson. Backtracking's complexity is not the size of the
search space, it is the size of the space MINUS everything your pruning
check removes. Write the check as early in the loop as you can:
rejecting at depth 2 skips every node below it.

the un-choose step is what makes one buffer safe to reuse across
branches -- and the clone at the leaf is what keeps the recorded
answers from aliasing it (example 10).
```

**2,057 nodes against 19,173,961**, for the same 92 solutions. That is what a single well-placed
`continue` is worth. Backtracking's complexity is not the size of the search space — it's the size of
the space *minus everything your pruning removes*, so put the check as early in the loop as it can go.

Note the two habits from example 10 in the skeleton: `un-choose` makes one buffer safe to reuse across
branches, and `slices.Clone` at the leaf keeps the recorded answers from aliasing it.

**Complexity:** permutations Θ(n!) · N-queens Θ(n!) worst case but vastly less in practice — full treatment in [29 — Backtracking](../../29-backtracking.md)

---

## 13. How fast the input shrinks decides everything

`🔴 hard` · *Depth as a design constraint*

Two recursive sorts. One shrinks by half, one by one. They have the same *kind* of definition and
completely different fates on real input.

**Steps:**

1. Instrument merge sort and a recursive insertion sort with a depth watermark.
2. Measure the peak stack of each in a fresh goroutine.
3. Compare the depth columns at n = 1,000…100,000.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
)

// Lesson 03 said recursion depth is space. That makes "how fast does the input
// shrink?" the single most important question about any recursion:
//
//   n -> n-1   depth n       -- overflows on real data
//   n -> n/2   depth log n   -- effectively unbounded
//
// Divide & conquer is not just about doing less work. It is about the stack.

var maxDepth int

func note(d int) {
	if d > maxDepth {
		maxDepth = d
	}
}

// mergeSort: two recursive calls, each on HALF. Depth is log2(n).
func mergeSort(xs []int, depth int) []int {
	note(depth)
	if len(xs) <= 1 {
		return xs
	}
	mid := len(xs) / 2
	left := mergeSort(xs[:mid], depth+1)
	right := mergeSort(xs[mid:], depth+1)

	out := make([]int, 0, len(xs))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			out = append(out, left[i])
			i++
		} else {
			out = append(out, right[j])
			j++
		}
	}
	return append(append(out, left[i:]...), right[j:]...)
}

// insertionSortRec: one recursive call, on n-1. Depth is n.
func insertionSortRec(xs []int, n, depth int) {
	note(depth)
	if n <= 1 {
		return
	}
	insertionSortRec(xs, n-1, depth+1)
	v := xs[n-1]
	j := n - 2
	for j >= 0 && xs[j] > v {
		xs[j+1] = xs[j]
		j--
	}
	xs[j+1] = v
}

func stackInUse() uint64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.StackInuse
}

// peak runs fn in a fresh goroutine and reports stack bytes at its deepest point.
func peak(fn func(probe func())) uint64 {
	runtime.GC()
	base := stackInUse()
	var top uint64
	done := make(chan struct{})
	go func() {
		defer close(done)
		fn(func() { top = stackInUse() })
	}()
	<-done
	runtime.GC()
	if top < base {
		return 0
	}
	return top - base
}

// Probe variants that sample the stack at maximum depth.

func mergeProbe(xs []int, probe func()) []int {
	if len(xs) <= 1 {
		probe()
		return xs
	}
	mid := len(xs) / 2
	left := mergeProbe(xs[:mid], probe)
	right := mergeProbe(xs[mid:], probe)
	out := make([]int, 0, len(xs))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			out = append(out, left[i])
			i++
		} else {
			out = append(out, right[j])
			j++
		}
	}
	return append(append(out, left[i:]...), right[j:]...)
}

func insertionProbe(xs []int, n int, probe func()) {
	if n <= 1 {
		probe()
		return
	}
	insertionProbe(xs, n-1, probe)
	v := xs[n-1]
	j := n - 2
	for j >= 0 && xs[j] > v {
		xs[j+1] = xs[j]
		j--
	}
	xs[j+1] = v
}

func input(n int) []int {
	rng := rand.New(rand.NewPCG(1, 2))
	xs := make([]int, n)
	for i := range xs {
		xs[i] = rng.IntN(1 << 20)
	}
	return xs
}

func main() {
	fmt.Println("two recursive sorts, two shrink rates:")
	fmt.Println()
	fmt.Printf("%10s %16s %16s %16s %16s\n",
		"n", "merge depth", "insertion depth", "merge stack", "insertion stack")

	for _, n := range []int{1_000, 10_000, 100_000} {
		src := input(n)

		maxDepth = 0
		mergeSort(slices.Clone(src), 0)
		md := maxDepth

		maxDepth = 0
		insertionSortRec(slices.Clone(src), n, 0)
		id := maxDepth

		ms := peak(func(p func()) { mergeProbe(slices.Clone(src), p) })
		is := peak(func(p func()) { insertionProbe(slices.Clone(src), n, p) })

		fmt.Printf("%10d %16d %16d %13.0f KB %13.0f KB\n",
			n, md, id, float64(ms)/1024, float64(is)/1024)
	}

	fmt.Println()
	fmt.Println("the depth columns are the whole story:")
	fmt.Println()
	fmt.Println("  merge sort     depth ~ log2(n). At n=100,000 that is 17.")
	fmt.Println("                 At n = 1 BILLION it would be about 30.")
	fmt.Println("  insertion sort depth = n. At n=100,000 that is 99,999 frames,")
	fmt.Println("                 and at a billion it is a dead process.")
	fmt.Println()
	fmt.Println("(merge sort's stack reads 0 KB because 17 frames do not move the")
	fmt.Println(" runtime's span-level accounting at all -- it is below the noise floor.)")
	fmt.Println()
	fmt.Println("both are 'recursive sorts'. Only one of them can be handed real input.")

	fmt.Println()
	fmt.Println("this is why the divide & conquer shape matters beyond its running time:")
	fmt.Println("halving makes the stack a NON-ISSUE, so you never have to convert it to")
	fmt.Println("an explicit stack (example 7) or worry about the limit (example 9).")
	fmt.Println()
	fmt.Println("the rule of thumb for the rest of this plan:")
	fmt.Println("  recursion that HALVES  -> leave it recursive, it is O(log n) deep")
	fmt.Println("  recursion that DECREMENTS -> make it a loop before it reaches production")

	// Confirm both sorts actually work.
	src := input(1000)
	a := mergeSort(slices.Clone(src), 0)
	b := slices.Clone(src)
	insertionSortRec(b, len(b), 0)
	fmt.Printf("\nboth sorts agree with slices.Sort: %v\n",
		slices.IsSorted(a) && slices.Equal(a, b))
}
```

**Sample output:**

```
two recursive sorts, two shrink rates:

         n      merge depth  insertion depth      merge stack  insertion stack
      1000               10              999             0 KB           128 KB
     10000               14             9999             0 KB          1024 KB
    100000               17            99999             0 KB          8192 KB

the depth columns are the whole story:

  merge sort     depth ~ log2(n). At n=100,000 that is 17.
                 At n = 1 BILLION it would be about 30.
  insertion sort depth = n. At n=100,000 that is 99,999 frames,
                 and at a billion it is a dead process.

(merge sort's stack reads 0 KB because 17 frames do not move the
 runtime's span-level accounting at all -- it is below the noise floor.)

both are 'recursive sorts'. Only one of them can be handed real input.

this is why the divide & conquer shape matters beyond its running time:
halving makes the stack a NON-ISSUE, so you never have to convert it to
an explicit stack (example 7) or worry about the limit (example 9).

the rule of thumb for the rest of this plan:
  recursion that HALVES  -> leave it recursive, it is O(log n) deep
  recursion that DECREMENTS -> make it a loop before it reaches production

both sorts agree with slices.Sort: true
```

Depth **17 versus 99,999** at n=100,000. At a billion elements merge sort would be ~30 deep and the
other would be a dead process. Both are "recursive sorts"; only one can be handed real input.

The rule for the rest of this plan: **recursion that halves — leave it alone. Recursion that
decrements — make it a loop before it reaches production.**

**Complexity:** merge sort Θ(n log n) time, **Θ(log n)** stack · recursive insertion sort Θ(n²) time, **Θ(n)** stack

---

## 14. What a function call actually costs

`🔴 hard` · *Three rows, three different answers*

"Recursion is slow" is too blunt to be useful. Here are three recursive/iterative pairs whose ratios
span four orders of magnitude — for three completely different reasons.

**Steps:**

1. Benchmark recursive and iterative sum, fib and binary search.
2. Compare the ratios.
3. Work out which row is about call overhead, which is about the algorithm, and which is about nothing.

```go
package main

import (
	"fmt"
	"testing"
)

var sink int

// What does a function call actually cost? Enough to matter when it is the
// innermost operation, and not enough to matter otherwise.
//
// Each pair below computes the same answer, recursively and iteratively.

func sumRec(xs []int) int {
	if len(xs) == 0 {
		return 0
	}
	return xs[0] + sumRec(xs[1:])
}

func sumIter(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

func fibRec(n int) int {
	if n < 2 {
		return n
	}
	return fibRec(n-1) + fibRec(n-2)
}

func fibIter(n int) int {
	if n < 2 {
		return n
	}
	prev, cur := 0, 1
	for i := 2; i <= n; i++ {
		prev, cur = cur, prev+cur
	}
	return cur
}

func searchRec(xs []int, target, lo, hi int) int {
	if lo > hi {
		return -1
	}
	mid := lo + (hi-lo)/2
	switch {
	case xs[mid] == target:
		return mid
	case xs[mid] < target:
		return searchRec(xs, target, mid+1, hi)
	default:
		return searchRec(xs, target, lo, mid-1)
	}
}

func searchIter(xs []int, target int) int {
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

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	xs := make([]int, 10_000)
	for i := range xs {
		xs[i] = i
	}
	sorted := make([]int, 1<<20)
	for i := range sorted {
		sorted[i] = i * 2
	}

	fmt.Printf("%-34s %14s %14s %10s\n", "", "recursive", "iterative", "ratio")

	rows := []struct {
		name     string
		rec, itr func()
	}{
		{"sum of 10,000 ints",
			func() { sink += sumRec(xs) },
			func() { sink += sumIter(xs) }},
		{"fib(25) -- exponential either way",
			func() { sink += fibRec(25) },
			func() { sink += fibIter(25) }},
		{"binary search in 2^20",
			func() { sink += searchRec(sorted, 999999, 0, len(sorted)-1) },
			func() { sink += searchIter(sorted, 999999) }},
	}

	for _, r := range rows {
		a, b := nsPerOp(r.rec), nsPerOp(r.itr)
		fmt.Printf("%-34s %11.1f ns %11.1f ns %9.1fx\n", r.name, a, b, a/b)
	}

	fmt.Println()
	fmt.Println("read each row separately -- they are three different stories:")
	fmt.Println()
	fmt.Println("  sum       the call IS the loop body, so you pay call overhead 10,000")
	fmt.Println("            times for one addition each. This is the case where the")
	fmt.Println("            rewrite pays -- and it also removes 10,000 stack frames.")
	fmt.Println()
	fmt.Println("  fib(25)   recursion is not the problem here; the exponential tree is.")
	fmt.Println("            The iterative version wins by a colossal margin because it")
	fmt.Println("            is a different ALGORITHM (example 8), not because it is a loop.")
	fmt.Println()
	fmt.Println("  search    ~20 calls total, so the overhead is invisible. Rewriting this")
	fmt.Println("            as a loop buys nothing measurable -- do it for taste, not speed.")
	fmt.Println()
	fmt.Println("what a call actually costs: push the arguments and return address, adjust")
	fmt.Println("the stack pointer, maybe check whether the stack needs to grow, jump, and")
	fmt.Println("unwind on the way back. A handful of nanoseconds -- irrelevant next to")
	fmt.Println("real work, and dominant when the work is one addition.")
	fmt.Println()
	fmt.Println("so: convert a recursion to a loop when the DEPTH is the problem")
	fmt.Println("(examples 9, 13, 15), or when the call is the innermost operation.")
	fmt.Println("Not because 'recursion is slow' -- that is far too blunt to be useful.")
}
```

**Sample output:**

```
                                        recursive      iterative      ratio
sum of 10,000 ints                     28297.2 ns      2290.4 ns      12.4x
fib(25) -- exponential either way     157044.2 ns         7.1 ns   22184.9x
binary search in 2^20                     23.2 ns        16.6 ns       1.4x

read each row separately -- they are three different stories:

  sum       the call IS the loop body, so you pay call overhead 10,000
            times for one addition each. This is the case where the
            rewrite pays -- and it also removes 10,000 stack frames.

  fib(25)   recursion is not the problem here; the exponential tree is.
            The iterative version wins by a colossal margin because it
            is a different ALGORITHM (example 8), not because it is a loop.

  search    ~20 calls total, so the overhead is invisible. Rewriting this
            as a loop buys nothing measurable -- do it for taste, not speed.

what a call actually costs: push the arguments and return address, adjust
the stack pointer, maybe check whether the stack needs to grow, jump, and
unwind on the way back. A handful of nanoseconds -- irrelevant next to
real work, and dominant when the work is one addition.

so: convert a recursion to a loop when the DEPTH is the problem
(examples 9, 13, 15), or when the call is the innermost operation.
Not because 'recursion is slow' -- that is far too blunt to be useful.
```

- **sum, 12.4×** — the call *is* the loop body, so you pay call overhead 10,000 times for one addition each. Converting pays, and removes 10,000 frames.
- **fib, 21,838×** — recursion isn't the problem; the exponential tree is. The iterative version wins because it's a different *algorithm* (example 8).
- **search, 1.4×** — ~20 calls total. Rewriting buys nothing measurable.

So convert when the **depth** is the problem, or when the call is the innermost operation. Not because
"recursion is slow".

**Complexity:** a call is a few nanoseconds — irrelevant beside real work, dominant when the work is one addition

---

## 15. Surviving input you didn't choose

`🔴 hard` · *The production version of this lesson*

A tree traversal is the textbook case for recursion, right up until someone hands you a tree that's
really a 5,000,000-long chain. Deeply-nested JSON, a comment thread, a linked list, an attacker's
document — all the same shape.

**Steps:**

1. Build a degenerate tree that's a straight chain.
2. Sum it recursively, iteratively, and with a depth-bounded recursion.
3. Crash the recursive one in a child process with a lowered stack limit.

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"runtime/debug"
	"strings"
)

// The production version of this lesson. A tree traversal is the textbook case
// for recursion -- right up until someone hands you a tree that is really a
// 500,000-long chain, and your service dies.
//
// This is not hypothetical: deeply-nested JSON, a linked list, a comment thread,
// a filesystem with a symlink loop, and an attacker-supplied document all
// produce exactly this shape.

type node struct {
	val         int
	left, right *node
}

// degenerate builds a tree that is a straight line to the right -- perfectly
// legal, and n levels deep.
func degenerate(n int) *node {
	root := &node{val: 0}
	cur := root
	for i := 1; i < n; i++ {
		cur.right = &node{val: i}
		cur = cur.right
	}
	return root
}

// sumRec is the natural implementation. Its depth is the tree's depth.
func sumRec(n *node) int {
	if n == nil {
		return 0
	}
	return n.val + sumRec(n.left) + sumRec(n.right)
}

// sumIter does the same walk with a heap-allocated stack. Its depth is bounded
// by available memory, not by the goroutine stack limit.
func sumIter(root *node) int {
	if root == nil {
		return 0
	}
	total := 0
	stack := []*node{root}
	for len(stack) > 0 {
		n := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		total += n.val
		if n.right != nil {
			stack = append(stack, n.right)
		}
		if n.left != nil {
			stack = append(stack, n.left)
		}
	}
	return total
}

// sumRecBounded refuses input it cannot handle instead of dying on it.
const maxAllowedDepth = 10_000

var errTooDeep = fmt.Errorf("input nested deeper than %d", maxAllowedDepth)

func sumRecBounded(n *node, depth int) (int, error) {
	if n == nil {
		return 0, nil
	}
	if depth > maxAllowedDepth {
		return 0, errTooDeep
	}
	l, err := sumRecBounded(n.left, depth+1)
	if err != nil {
		return 0, err
	}
	r, err := sumRecBounded(n.right, depth+1)
	if err != nil {
		return 0, err
	}
	return n.val + l + r, nil
}

func childCrash() {
	// A 16 MB limit stands in for "a real server under memory pressure, or a
	// tree deeper than 1 GB / 48 bytes". The failure mode is identical.
	debug.SetMaxStack(16 << 20)

	tree := degenerate(5_000_000)
	fmt.Println("recursive sum:", sumRec(tree))
}

func main() {
	if os.Getenv("L05_DEEP") == "1" {
		childCrash()
		return
	}

	const n = 1_000_000
	tree := degenerate(n)

	fmt.Printf("a 'tree' of %d nodes that is really a chain %d levels deep.\n\n", n, n)

	fmt.Printf("  iterative (explicit stack): %d\n", sumIter(tree))
	fmt.Printf("  recursive (unbounded):      %d\n", sumRec(tree))

	got, err := sumRecBounded(tree, 0)
	fmt.Printf("  recursive (depth-limited):  %d, err = %v\n", got, err)

	fmt.Println()
	fmt.Println("at a million deep the recursive version still works -- Go's 1 GB stack")
	fmt.Println("limit is generous. Now the same code in a child process with a 16 MB")
	fmt.Println("limit and a 5-million-node chain:")
	fmt.Println()

	cmd := exec.Command(os.Args[0])
	cmd.Env = append(os.Environ(), "L05_DEEP=1")
	out, err := cmd.CombinedOutput()
	lines := strings.Split(strings.TrimSpace(string(out)), "\n")
	for i, ln := range lines {
		if i >= 3 {
			fmt.Printf("  ... (%d more lines)\n", len(lines)-3)
			break
		}
		fmt.Println("  " + ln)
	}
	fmt.Printf("\n  child exit status: %v\n", err)

	fmt.Println()
	fmt.Println("the iterative version would have returned an answer, because its stack")
	fmt.Println("is a []*node on the HEAP: it grows by reallocation (lesson 03) and is")
	fmt.Println("bounded by memory, not by a per-goroutine limit.")

	fmt.Println()
	fmt.Println("three ways to handle recursion over untrusted input, in order:")
	fmt.Println()
	fmt.Println("  1. BOUND THE DEPTH. Return an error past a limit you chose. Cheapest")
	fmt.Println("     to write, keeps the readable recursion, and turns a process kill")
	fmt.Println("     into a 400 Bad Request. encoding/json does exactly this.")
	fmt.Println()
	fmt.Println("  2. CONVERT TO AN EXPLICIT STACK (example 7). No limit at all, at the")
	fmt.Println("     cost of less obvious code.")
	fmt.Println()
	fmt.Println("  3. RESTRUCTURE so the depth is logarithmic (example 13) -- only")
	fmt.Println("     possible if you control the shape of the data.")
	fmt.Println()
	fmt.Println("what is NOT on the list: raising SetMaxStack, or recovering from the")
	fmt.Println("crash. The first postpones the problem; the second is impossible.")
}
```

**Sample output** (traceback trimmed):

```
a 'tree' of 1000000 nodes that is really a chain 1000000 levels deep.

  iterative (explicit stack): 499999500000
  recursive (unbounded):      499999500000
  recursive (depth-limited):  0, err = input nested deeper than 10000

at a million deep the recursive version still works -- Go's 1 GB stack
limit is generous. Now the same code in a child process with a 16 MB
limit and a 5-million-node chain:

  runtime: goroutine stack exceeds 16777216-byte limit
  runtime: sp=0x42ff7e8003a0 stack=[0x42ff7e800000, 0x42ff7f800000]
  fatal error: stack overflow
  ... (395 more lines)

  child exit status: exit status 2

the iterative version would have returned an answer, because its stack
is a []*node on the HEAP: it grows by reallocation (lesson 03) and is
bounded by memory, not by a per-goroutine limit.

three ways to handle recursion over untrusted input, in order:

  1. BOUND THE DEPTH. Return an error past a limit you chose. Cheapest
     to write, keeps the readable recursion, and turns a process kill
     into a 400 Bad Request. encoding/json does exactly this.

  2. CONVERT TO AN EXPLICIT STACK (example 7). No limit at all, at the
     cost of less obvious code.

  3. RESTRUCTURE so the depth is logarithmic (example 13) -- only
     possible if you control the shape of the data.

what is NOT on the list: raising SetMaxStack, or recovering from the
crash. The first postpones the problem; the second is impossible.
```

The iterative version would have returned an answer, because its stack is a `[]*node` on the **heap**:
it grows by reallocation and is bounded by memory, not by a per-goroutine limit.

Three defences, in order: **bound the depth** (four lines, turns a process kill into a `400`; this is
what `encoding/json` does), **convert to an explicit stack** (example 7), or **restructure so depth is
logarithmic** (example 13). What is not on the list: raising `SetMaxStack`, or recovering from the
crash — the first postpones it, the second is impossible.

**Complexity:** all three are Θ(n) time · recursive Θ(n) **goroutine stack**, iterative Θ(n) **heap** — and only one of those has a hard 1 GB ceiling

---

## 16. Capstone: a recursive-descent evaluator

`🔴 hard` · *The program that justifies recursion*

Everything so far has been about when *not* to recurse. This is the case where recursion isn't a
clever trick but a direct transcription of the problem: the input is defined recursively, so the
reader is too.

```
expr   := term   (('+' | '-') term)*
term   := factor (('*' | '/') factor)*
factor := NUMBER | '-' factor | '(' expr ')'
```

One function per rule. Where a rule names another rule, the function calls it.

**Steps:**

1. Write one function per grammar rule.
2. Use a **loop** for each `(...)*` repetition, and recursion only where a rule nests.
3. Add the depth guard from example 15.

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

// The capstone: a recursive-descent evaluator for arithmetic expressions.
//
// This is the program that justifies recursion. The INPUT is recursive -- an
// expression contains expressions -- so a recursive reader is not a clever
// trick, it is a direct transcription of the grammar:
//
//   expr   := term   (('+' | '-') term)*
//   term   := factor (('*' | '/') factor)*
//   factor := NUMBER | '-' factor | '(' expr ')'
//
// One function per rule. Where a rule mentions another rule, the function calls
// it. Precedence falls out of the NESTING: expr calls term, so term binds
// tighter; factor calls back into expr inside parentheses, and that single
// mutually-recursive edge is what makes nesting work to any depth.

type parser struct {
	tokens []string
	pos    int
	depth  int
	maxDep int
	err    error
}

const maxDepth = 200 // example 15: bound the depth of untrusted input

func (p *parser) peek() string {
	if p.pos < len(p.tokens) {
		return p.tokens[p.pos]
	}
	return ""
}

func (p *parser) next() string {
	t := p.peek()
	p.pos++
	return t
}

func (p *parser) enter() bool {
	p.depth++
	if p.depth > p.maxDep {
		p.maxDep = p.depth
	}
	if p.depth > maxDepth {
		p.fail("expression nested deeper than %d", maxDepth)
		return false
	}
	return true
}

func (p *parser) leave() { p.depth-- }

func (p *parser) fail(format string, args ...any) {
	if p.err == nil {
		p.err = fmt.Errorf(format, args...)
	}
}

// expr := term (('+' | '-') term)*
//
// The repetition is a LOOP, not recursion: `a - b - c` must group left, and a
// recursive call here would group it right and compute the wrong answer.
func (p *parser) expr() float64 {
	if !p.enter() {
		return 0
	}
	defer p.leave()

	left := p.term()
	for p.err == nil {
		switch p.peek() {
		case "+":
			p.next()
			left += p.term()
		case "-":
			p.next()
			left -= p.term()
		default:
			return left
		}
	}
	return left
}

// term := factor (('*' | '/') factor)*
func (p *parser) term() float64 {
	if !p.enter() {
		return 0
	}
	defer p.leave()

	left := p.factor()
	for p.err == nil {
		switch p.peek() {
		case "*":
			p.next()
			left *= p.factor()
		case "/":
			p.next()
			right := p.factor()
			if right == 0 {
				p.fail("division by zero")
				return 0
			}
			left /= right
		default:
			return left
		}
	}
	return left
}

// factor := NUMBER | '-' factor | '(' expr ')'
//
// This is where the recursion closes the loop: a parenthesised expression sends
// us back to expr(), one level deeper.
func (p *parser) factor() float64 {
	if !p.enter() {
		return 0
	}
	defer p.leave()

	switch t := p.next(); {
	case t == "":
		p.fail("unexpected end of expression")
		return 0

	case t == "-": // unary minus: -x, and -(-x) too
		return -p.factor()

	case t == "(":
		v := p.expr()
		if p.next() != ")" {
			p.fail("missing closing parenthesis")
		}
		return v

	default:
		v, err := strconv.ParseFloat(t, 64)
		if err != nil {
			p.fail("not a number: %q", t)
		}
		return v
	}
}

func tokenize(s string) []string {
	for _, op := range []string{"+", "-", "*", "/", "(", ")"} {
		s = strings.ReplaceAll(s, op, " "+op+" ")
	}
	return strings.Fields(s)
}

func eval(s string) (float64, int, error) {
	p := &parser{tokens: tokenize(s)}
	v := p.expr()
	if p.err == nil && p.pos < len(p.tokens) {
		p.fail("unexpected %q", p.tokens[p.pos])
	}
	return v, p.maxDep, p.err
}

func main() {
	fmt.Println("recursive-descent evaluation -- one function per grammar rule:")
	fmt.Println()
	fmt.Println("  expr   := term   (('+' | '-') term)*")
	fmt.Println("  term   := factor (('*' | '/') factor)*")
	fmt.Println("  factor := NUMBER | '-' factor | '(' expr ')'")
	fmt.Println()

	inputs := []string{
		"1 + 2",
		"2 + 3 * 4",
		"(2 + 3) * 4",
		"10 - 2 - 3",
		"2 * (3 + (4 - 1) * 2)",
		"-5 + 3",
		"-(2 + 3) * -2",
		"100 / 4 / 5",
		"1 + 2 * 3 - 4 / 2",
	}

	fmt.Printf("%-26s %14s %10s\n", "expression", "result", "max depth")
	for _, in := range inputs {
		v, depth, err := eval(in)
		if err != nil {
			fmt.Printf("%-26s %14s %10d\n", in, "ERR: "+err.Error(), depth)
			continue
		}
		fmt.Printf("%-26s %14g %10d\n", in, v, depth)
	}

	fmt.Println()
	fmt.Println("errors are refused, not crashed on:")
	fmt.Println()
	for _, in := range []string{"1 +", "(1 + 2", "1 / 0", "2 $ 3"} {
		_, _, err := eval(in)
		fmt.Printf("  %-12s -> %v\n", in, err)
	}

	// Depth guard: 300 nested parens exceeds maxDepth.
	deep := strings.Repeat("(", 300) + "1" + strings.Repeat(")", 300)
	_, d, err := eval(deep)
	fmt.Printf("  %-12s -> %v (reached depth %d)\n", "300 nested (", err, d)

	fmt.Println()
	fmt.Println("what to take from this:")
	fmt.Println()
	fmt.Println("  1. the code IS the grammar. Three rules, three functions. When the")
	fmt.Println("     input's definition is recursive, so is the natural reader --")
	fmt.Println("     this is the shape from example 11's mutually recursive data.")
	fmt.Println()
	fmt.Println("  2. PRECEDENCE comes from the call graph, not from any precedence")
	fmt.Println("     table: expr calls term, so * binds tighter than +. Look at")
	fmt.Println("     '2 + 3 * 4' = 14 and '(2 + 3) * 4' = 20.")
	fmt.Println()
	fmt.Println("  3. ASSOCIATIVITY comes from using a LOOP for the repetition. If expr")
	fmt.Println("     recursed on itself for the tail, '10 - 2 - 3' would group as")
	fmt.Println("     10 - (2 - 3) = 11 instead of the correct 5.")
	fmt.Println()
	fmt.Println("  4. the depth guard from example 15 costs four lines and turns a")
	fmt.Println("     hostile input from a process kill into an error value.")
	fmt.Println()
	fmt.Println("that is Part 1. From here on the structures do the work -- and every")
	fmt.Println("one of them is built out of what these five lessons measured.")
}
```

**Output:**

```
recursive-descent evaluation -- one function per grammar rule:

  expr   := term   (('+' | '-') term)*
  term   := factor (('*' | '/') factor)*
  factor := NUMBER | '-' factor | '(' expr ')'

expression                         result  max depth
1 + 2                                   3          3
2 + 3 * 4                              14          3
(2 + 3) * 4                            20          6
10 - 2 - 3                              5          3
2 * (3 + (4 - 1) * 2)                  18          9
-5 + 3                                 -2          4
-(2 + 3) * -2                          10          7
100 / 4 / 5                             5          3
1 + 2 * 3 - 4 / 2                       5          3

errors are refused, not crashed on:

  1 +          -> unexpected end of expression
  (1 + 2       -> missing closing parenthesis
  1 / 0        -> division by zero
  2 $ 3        -> unexpected "$"
  300 nested ( -> expression nested deeper than 200 (reached depth 201)

what to take from this:

  1. the code IS the grammar. Three rules, three functions. When the
     input's definition is recursive, so is the natural reader --
     this is the shape from example 11's mutually recursive data.

  2. PRECEDENCE comes from the call graph, not from any precedence
     table: expr calls term, so * binds tighter than +. Look at
     '2 + 3 * 4' = 14 and '(2 + 3) * 4' = 20.

  3. ASSOCIATIVITY comes from using a LOOP for the repetition. If expr
     recursed on itself for the tail, '10 - 2 - 3' would group as
     10 - (2 - 3) = 11 instead of the correct 5.

  4. the depth guard from example 15 costs four lines and turns a
     hostile input from a process kill into an error value.

that is Part 1. From here on the structures do the work -- and every
one of them is built out of what these five lessons measured.
```

Four things worth extracting:

1. **The code is the grammar.** Three rules, three functions — the shape from example 11's mutually recursive data.
2. **Precedence comes from the call graph**, not a table. `expr` calls `term`, so `*` binds tighter: `2 + 3 * 4` = 14 while `(2 + 3) * 4` = 20.
3. **Associativity comes from using a loop** for the repetition. Had `expr` recursed on its own tail, `10 - 2 - 3` would group right and give 11 instead of 5.
4. **The depth guard costs four lines** and turns 300 nested parens from a process kill into an error value.

**Complexity:** Θ(tokens) time, Θ(nesting depth) stack · the guard caps the stack at a number you chose rather than one the input chose

---

> That closes **Part 1**. Everything from here builds structures with what these five lessons
> measured — starting at [06 — Dynamic Arrays & Slices](../../06-arrays-slices.md).

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
