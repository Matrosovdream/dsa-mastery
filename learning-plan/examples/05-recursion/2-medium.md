# Step 05 — Recursion & the Call Stack · 🟡 Medium

Examples **7–11**: explicit stacks, memoization, the stack limit, and the bug that ruins backtracking in Go.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 7, 8, 10 and 11 are **deterministic**. Example 9 measures stack bytes and spawns a child
process, so its numbers and traceback vary.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Recursion → loop + explicit stack

`🟡 medium` · *When one loop isn't enough*

Example 5 converted recursions with **one** recursive call. With two, a plain loop can't do it — but
a loop plus an explicit stack always can, because that's precisely what the call stack was doing.

**Steps:**

1. Write pre-order recursively and iteratively; note the reverse push order.
2. Do the same for in-order, which is harder.
3. Confirm both pairs produce identical sequences.

```go
package main

import "fmt"

// When a recursion makes MORE THAN ONE recursive call, you cannot turn it into
// a plain loop. But you can always turn it into a loop plus an explicit stack --
// because that is exactly what the call stack was doing for you.
//
// The transformation:
//   1. make a stack of the things the recursive parameter held
//   2. loop while the stack is non-empty: pop, do the visit, push the children
//   3. push children in REVERSE order, because a stack reverses them again

type node struct {
	val         int
	left, right *node
}

func leaf(v int) *node { return &node{val: v} }

// --- recursive: the call stack does the bookkeeping ---

func preorderRec(n *node, out *[]int) {
	if n == nil {
		return
	}
	*out = append(*out, n.val) // visit
	preorderRec(n.left, out)
	preorderRec(n.right, out)
}

// --- iterative: you do the bookkeeping ---

func preorderIter(root *node) []int {
	var out []int
	if root == nil {
		return out
	}
	stack := []*node{root}

	for len(stack) > 0 {
		n := stack[len(stack)-1]
		stack = stack[:len(stack)-1]

		out = append(out, n.val) // visit

		// Push RIGHT first so that LEFT is popped first -- a stack reverses.
		if n.right != nil {
			stack = append(stack, n.right)
		}
		if n.left != nil {
			stack = append(stack, n.left)
		}
	}
	return out
}

// --- in-order is the harder one: the visit happens BETWEEN the two calls ---

func inorderRec(n *node, out *[]int) {
	if n == nil {
		return
	}
	inorderRec(n.left, out)
	*out = append(*out, n.val) // visit is in the middle
	inorderRec(n.right, out)
}

// inorderIter needs a "walk left as far as possible, then visit, then go right"
// loop, because the visit happens after the left subtree is finished.
func inorderIter(root *node) []int {
	var out []int
	var stack []*node
	cur := root

	for cur != nil || len(stack) > 0 {
		for cur != nil { // descend left, remembering the way back
			stack = append(stack, cur)
			cur = cur.left
		}
		cur = stack[len(stack)-1] // the left subtree is done
		stack = stack[:len(stack)-1]

		out = append(out, cur.val) // visit
		cur = cur.right            // and now the right subtree
	}
	return out
}

// maxDepthIter also tracks the stack's own high-water mark, which is the
// recursion depth the recursive version would have used.
func maxStackDepth(root *node) int {
	type frame struct {
		n     *node
		depth int
	}
	stack := []frame{{root, 1}}
	best := 0
	for len(stack) > 0 {
		f := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if f.n == nil {
			continue
		}
		if f.depth > best {
			best = f.depth
		}
		stack = append(stack, frame{f.n.right, f.depth + 1}, frame{f.n.left, f.depth + 1})
	}
	return best
}

func build() *node {
	//         1
	//       /   \
	//      2     3
	//     / \   /
	//    4   5 6
	root := &node{val: 1,
		left:  &node{val: 2, left: leaf(4), right: leaf(5)},
		right: &node{val: 3, left: leaf(6)},
	}
	return root
}

func main() {
	root := build()

	var rec []int
	preorderRec(root, &rec)
	iter := preorderIter(root)

	fmt.Println("pre-order (visit, then left, then right):")
	fmt.Printf("  recursive  %v\n", rec)
	fmt.Printf("  iterative  %v\n", iter)

	var recIn []int
	inorderRec(root, &recIn)
	iterIn := inorderIter(root)

	fmt.Println()
	fmt.Println("in-order (left, then visit, then right):")
	fmt.Printf("  recursive  %v\n", recIn)
	fmt.Printf("  iterative  %v\n", iterIn)

	fmt.Println()
	fmt.Printf("max depth reached: %d (that is how many frames the recursion needed)\n",
		maxStackDepth(root))

	fmt.Println()
	fmt.Println("why pre-order is easy and in-order is not:")
	fmt.Println()
	fmt.Println("  pre-order visits BEFORE both calls, so a node is finished the moment")
	fmt.Println("  you pop it -- push children and move on.")
	fmt.Println()
	fmt.Println("  in-order visits BETWEEN the calls, so a popped node is only HALF done:")
	fmt.Println("  its left subtree must complete first. The explicit version encodes that")
	fmt.Println("  as 'descend left pushing everything, then pop-visit-go-right'.")
	fmt.Println()
	fmt.Println("that extra bookkeeping is precisely what the call stack was doing for free.")
	fmt.Println("Converting is worth it when depth can grow with n -- a degenerate tree is")
	fmt.Println("a linked list, and n frames deep (example 15).")
	fmt.Println()
	fmt.Println("note the explicit stack lives on the HEAP: it grows by reallocation")
	fmt.Println("(lesson 03) instead of by frame, so it can reach millions of entries")
	fmt.Println("without ever hitting the goroutine stack limit.")
}
```

**Output:**

```
pre-order (visit, then left, then right):
  recursive  [1 2 4 5 3 6]
  iterative  [1 2 4 5 3 6]

in-order (left, then visit, then right):
  recursive  [4 2 5 1 6 3]
  iterative  [4 2 5 1 6 3]

max depth reached: 3 (that is how many frames the recursion needed)

why pre-order is easy and in-order is not:

  pre-order visits BEFORE both calls, so a node is finished the moment
  you pop it -- push children and move on.

  in-order visits BETWEEN the calls, so a popped node is only HALF done:
  its left subtree must complete first. The explicit version encodes that
  as 'descend left pushing everything, then pop-visit-go-right'.

that extra bookkeeping is precisely what the call stack was doing for free.
Converting is worth it when depth can grow with n -- a degenerate tree is
a linked list, and n frames deep (example 15).

note the explicit stack lives on the HEAP: it grows by reallocation
(lesson 03) instead of by frame, so it can reach millions of entries
without ever hitting the goroutine stack limit.
```

Pre-order is easy because the visit happens *before* both calls — a popped node is finished. In-order
visits *between* the calls, so a popped node is only half-done and you need the "descend left pushing
everything, then pop-visit-go-right" dance. That's the bookkeeping the call stack was doing for free.

**Complexity:** both are Θ(n) time and Θ(h) space · the difference is that the explicit stack lives on the **heap**, so it can reach millions of entries without touching the goroutine stack limit

---

## 8. Memoization, and the bridge to DP

`🟡 medium` · *Collapsing a tree that repeats itself*

If the recursion tree contains the same subproblem twice, it contains it a catastrophic number of
times. Recording each answer the first time turns exponential into linear — and that single idea is
all of dynamic programming.

**Steps:**

1. Write the honest recursion and count calls.
2. Add three lines of memo and count again.
3. Measure the actual redundancy: how many times does `fib(30)` invoke `fib(5)`?

```go
package main

import "fmt"

// A recursion whose tree contains the SAME subproblem more than once is doing
// redundant work. Recording each answer the first time turns an exponential
// tree into a linear one -- and that single idea is the whole of dynamic
// programming (lessons 30-32).
//
// Three versions of the same function, in the order you should always write them:
//   1. plain recursion   -- correct, obvious, exponential
//   2. + memo            -- correct, obvious, linear. Change: 3 lines.
//   3. tabulation        -- same recurrence, bottom-up, no recursion at all

var calls int

// --- 1. the honest recursion ---

func fib(n int) int {
	calls++
	if n < 2 {
		return n
	}
	return fib(n-1) + fib(n-2)
}

// --- 2. the same function, plus a memo ---

func fibMemo(n int, memo map[int]int) int {
	calls++
	if n < 2 {
		return n
	}
	if v, ok := memo[n]; ok { // added
		return v // added
	} // added
	v := fibMemo(n-1, memo) + fibMemo(n-2, memo)
	memo[n] = v // added
	return v
}

// --- 3. bottom-up: fill the table in dependency order ---

func fibTable(n int) int {
	if n < 2 {
		return n
	}
	prev, cur := 0, 1
	for i := 2; i <= n; i++ {
		prev, cur = cur, prev+cur
	}
	return cur
}

// countTarget answers "how many times does computing fib(n) invoke fib(target)?"
// -- the redundancy, measured rather than asserted.
var targetHits int

func countTarget(n, target int) int {
	if n == target {
		targetHits++
	}
	if n < 2 {
		return n
	}
	return countTarget(n-1, target) + countTarget(n-2, target)
}

// --- the same three stages on a two-dimensional problem ---

// gridPaths counts routes from (0,0) to (r,c) moving only down or right.
func gridPaths(r, c int) int {
	calls++
	if r == 0 || c == 0 {
		return 1
	}
	return gridPaths(r-1, c) + gridPaths(r, c-1)
}

func gridPathsMemo(r, c int, memo map[[2]int]int) int {
	calls++
	if r == 0 || c == 0 {
		return 1
	}
	key := [2]int{r, c}
	if v, ok := memo[key]; ok {
		return v
	}
	v := gridPathsMemo(r-1, c, memo) + gridPathsMemo(r, c-1, memo)
	memo[key] = v
	return v
}

func main() {
	fmt.Println("fib(n): calls made by each version")
	fmt.Println()
	fmt.Printf("%6s %16s %14s %14s\n", "n", "plain", "memoized", "fib(n)")
	for _, n := range []int{10, 20, 30, 35} {
		calls = 0
		got := fib(n)
		plain := calls

		calls = 0
		fibMemo(n, make(map[int]int))
		memo := calls

		fmt.Printf("%6d %16d %14d %14d\n", n, plain, memo, got)
	}
	fmt.Printf("\n  tabulated fib(35) = %d, in 34 loop iterations and O(1) space\n", fibTable(35))

	fmt.Println()
	fmt.Println("grid paths on an r x c grid:")
	fmt.Println()
	fmt.Printf("%10s %16s %14s %14s\n", "grid", "plain", "memoized", "paths")
	for _, n := range []int{4, 8, 12, 14} {
		calls = 0
		got := gridPaths(n, n)
		plain := calls

		calls = 0
		gridPathsMemo(n, n, make(map[[2]int]int))
		memo := calls

		fmt.Printf("%8dx%-2d %16d %14d %14d\n", n, n, plain, memo, got)
	}

	fmt.Println()
	fmt.Println("the memo works because the recursion tree REVISITS subproblems.")
	fmt.Println("fib(5) asks for fib(3) twice. Deeper down it gets absurd:")
	fmt.Println()
	for _, target := range []int{20, 15, 10, 5} {
		targetHits = 0
		countTarget(30, target)
		fmt.Printf("  computing fib(30) invokes fib(%2d) %10d times\n", target, targetHits)
	}
	fmt.Println()
	fmt.Println("With a memo each distinct subproblem is computed exactly once, so the")
	fmt.Println("call count collapses to O(number of distinct subproblems).")
	fmt.Println()
	fmt.Println("  fib:        n distinct subproblems     -> Theta(n)")
	fmt.Println("  gridPaths:  r*c distinct subproblems   -> Theta(r*c)")
	fmt.Println()
	fmt.Println("that is the entire trick, and it has a name for each direction:")
	fmt.Println("  MEMOIZATION  top-down: recurse, and cache on the way back")
	fmt.Println("  TABULATION   bottom-up: order the subproblems and fill a table")
	fmt.Println()
	fmt.Println("memoization is easier to derive (it is your recursion plus 3 lines).")
	fmt.Println("Tabulation removes the recursion entirely, so it cannot overflow the")
	fmt.Println("stack and it often needs less space -- fibTable keeps two ints, not n.")
	fmt.Println()
	fmt.Println("when memoizing does NOT help: a recursion tree with no repeated")
	fmt.Println("subproblems. Merge sort's halves never overlap, so there is nothing")
	fmt.Println("to cache -- that is divide & conquer, not DP (example 13).")
}
```

**Output:**

```
fib(n): calls made by each version

     n            plain       memoized         fib(n)
    10              177             19             55
    20            21891             39           6765
    30          2692537             59         832040
    35         29860703             69        9227465

  tabulated fib(35) = 9227465, in 34 loop iterations and O(1) space

grid paths on an r x c grid:

      grid            plain       memoized          paths
       4x4               139             33             70
       8x8             25739            129          12870
      12x12          5408311            289        2704156
      14x14         80233199            393       40116600

the memo works because the recursion tree REVISITS subproblems.
fib(5) asks for fib(3) twice. Deeper down it gets absurd:

  computing fib(30) invokes fib(20)         89 times
  computing fib(30) invokes fib(15)        987 times
  computing fib(30) invokes fib(10)      10946 times
  computing fib(30) invokes fib( 5)     121393 times

With a memo each distinct subproblem is computed exactly once, so the
call count collapses to O(number of distinct subproblems).

  fib:        n distinct subproblems     -> Theta(n)
  gridPaths:  r*c distinct subproblems   -> Theta(r*c)

that is the entire trick, and it has a name for each direction:
  MEMOIZATION  top-down: recurse, and cache on the way back
  TABULATION   bottom-up: order the subproblems and fill a table

memoization is easier to derive (it is your recursion plus 3 lines).
Tabulation removes the recursion entirely, so it cannot overflow the
stack and it often needs less space -- fibTable keeps two ints, not n.

when memoizing does NOT help: a recursion tree with no repeated
subproblems. Merge sort's halves never overlap, so there is nothing
to cache -- that is divide & conquer, not DP (example 13).
```

**29,860,703 calls down to 69.** The redundancy table is the reason: computing `fib(30)` invokes
`fib(5)` **121,393 times**. With a memo each distinct subproblem is computed exactly once, so the call
count collapses to the number of distinct subproblems — Θ(n) for fib, Θ(r·c) for grid paths.

And note when this does **not** apply: merge sort's halves never overlap, so there's nothing to cache.
That's divide & conquer, not DP.

**Complexity:** plain `fib` Θ(φⁿ) · memoized Θ(n) time, Θ(n) space · tabulated Θ(n) time, **Θ(1)** space — full treatment in [30 — DP I](../../30-dp-one-dimensional.md)

---

## 9. How deep can you go, and what happens at the end

`🟡 medium` · *The stack limit is not a panic*

Go grows goroutine stacks on demand — 8 KB to a 1 GB default — so deep recursion works far longer
than in C. Then it doesn't, and the failure is **not** something you can catch.

**Steps:**

1. Measure bytes per frame at three depths, in a fresh goroutine.
2. Compute the practical frame limit.
3. Trigger a real overflow in a **child process** with a lowered limit, and check whether the deferred `recover()` runs.

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"runtime"
	"runtime/debug"
	"strings"
)

// How deep can a Go recursion actually go, and what happens at the end?
//
// Go does not give goroutines a fixed stack. They start at ~8 KB and GROW by
// copying to a bigger block, up to a limit you can read and set with
// runtime/debug.SetMaxStack (1 GB by default on 64-bit).
//
// The important part: running out is NOT a panic. It is a fatal error, and
// recover() cannot catch it. Your process dies.

func deep(n int) int {
	if n == 0 {
		return 0
	}
	return 1 + deep(n-1)
}

func stackInUse() uint64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.StackInuse
}

// probeDeep must be a PLAIN FUNCTION, not a recursive closure. A closure's
// frame here measured ~6.7 KB against ~40 bytes for this one -- enough that the
// measurement itself overflowed the 1 GB limit at depth 100,000.
func probeDeep(n int, atBottom func()) {
	if n == 0 {
		atBottom()
		return
	}
	probeDeep(n-1, atBottom)
}

// bytesPerFrame measures in a FRESH goroutine, which starts with a small stack.
// On the main goroutine Go's reluctance to shrink stacks would hide every
// measurement after the first (lesson 03, example 9).
func bytesPerFrame(depth int) float64 {
	runtime.GC()
	base := stackInUse()

	var peak uint64
	done := make(chan struct{})
	go func() {
		defer close(done)
		probeDeep(depth, func() { peak = stackInUse() })
	}()
	<-done
	runtime.GC()
	return float64(peak-base) / float64(depth)
}

// The child branch: run with a deliberately small stack limit and recurse
// forever, so we can show what the failure actually looks like.
func childOverflow() {
	debug.SetMaxStack(4 << 20) // 4 MB instead of the 1 GB default

	defer func() {
		// This never runs. A stack overflow is a fatal error, not a panic.
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()

	var forever func(int) int
	forever = func(n int) int { return 1 + forever(n+1) }
	fmt.Println(forever(0))
}

func main() {
	if os.Getenv("L05_OVERFLOW") == "1" {
		childOverflow()
		return
	}

	fmt.Println("Go grows goroutine stacks on demand, so deep recursion works --")
	fmt.Println("right up until it does not.")
	fmt.Println()

	fmt.Printf("%14s %18s\n", "depth", "bytes per frame")
	for _, d := range []int{10_000, 100_000, 1_000_000} {
		fmt.Printf("%14d %18.0f\n", d, bytesPerFrame(d))
	}

	fmt.Println()
	fmt.Printf("deep(1000000) returned %d, so a million frames is genuinely fine.\n", deep(1_000_000))

	perFrame := bytesPerFrame(1_000_000)
	limit := int64(1) << 30 // the 1 GB default on 64-bit
	fmt.Printf("\nat ~%.0f bytes/frame, the 1 GB default limit is about %.1f million frames.\n",
		perFrame, float64(limit)/perFrame/1e6)

	// Now show the failure, in a child process with a 4 MB limit.
	fmt.Println()
	fmt.Println("what happens past the limit -- run in a child process with SetMaxStack(4 MB)")
	fmt.Println("and a recursion that never terminates:")
	fmt.Println()

	cmd := exec.Command(os.Args[0])
	cmd.Env = append(os.Environ(), "L05_OVERFLOW=1")
	out, err := cmd.CombinedOutput()

	lines := strings.Split(strings.TrimSpace(string(out)), "\n")
	for i, ln := range lines {
		if i >= 4 {
			fmt.Printf("  ... (%d more lines of traceback)\n", len(lines)-4)
			break
		}
		fmt.Println("  " + ln)
	}
	fmt.Printf("\n  child exit status: %v\n", err)

	fmt.Println()
	fmt.Println("three things to take from that output:")
	fmt.Println()
	fmt.Println("  1. it says FATAL ERROR, not panic. The deferred recover() in the child")
	fmt.Println("     never ran -- you cannot catch this, and you cannot degrade gracefully.")
	fmt.Println("  2. the process exits non-zero. In a server, one deeply-nested request")
	fmt.Println("     takes down every other in-flight request with it.")
	fmt.Println("  3. the limit is per GOROUTINE, so this is not something a worker pool")
	fmt.Println("     or a timeout protects you from.")
	fmt.Println()
	fmt.Println("the defence is never a bigger limit. It is either bounding the depth")
	fmt.Println("(reject input deeper than N) or removing the recursion (examples 7, 15).")
}
```

**Sample output** (traceback trimmed; addresses vary):

```
Go grows goroutine stacks on demand, so deep recursion works --
right up until it does not.

         depth    bytes per frame
         10000                 52
        100000                 42
       1000000                 34

deep(1000000) returned 1000000, so a million frames is genuinely fine.

at ~34 bytes/frame, the 1 GB default limit is about 32.0 million frames.

what happens past the limit -- run in a child process with SetMaxStack(4 MB)
and a recursion that never terminates:

  runtime: goroutine stack exceeds 4194304-byte limit
  runtime: sp=0x531872760390 stack=[0x531872760000, 0x531872b60000]
  fatal error: stack overflow
  
  ... (274 more lines of traceback)

  child exit status: exit status 2

three things to take from that output:

  1. it says FATAL ERROR, not panic. The deferred recover() in the child
     never ran -- you cannot catch this, and you cannot degrade gracefully.
  2. the process exits non-zero. In a server, one deeply-nested request
     takes down every other in-flight request with it.
  3. the limit is per GOROUTINE, so this is not something a worker pool
     or a timeout protects you from.

the defence is never a bigger limit. It is either bounding the depth
(reject input deeper than N) or removing the recursion (examples 7, 15).
```

Note the measurement detail in the code: `probeDeep` must be a **plain function**. My first version
used a recursive closure, whose frames measured ~6.7 KB against ~40 bytes — enough that the
*measurement itself* overflowed 1 GB at depth 100,000.

**Complexity:** ~34–49 bytes per frame · ~32 million frames at the 1 GB default · past it: `fatal error`, exit 2, and `recover()` never runs

---

## 10. The shared-slice bug

`🟡 medium` · *Lesson 04's aliasing trap, where it hides best*

This is **the** bug of recursive algorithms in Go, and it isn't really a recursion bug at all — it's
`append` aliasing. Build a partial answer in a slice, record it at a leaf, and every recorded answer
turns out to be a view into one shared array.

**Steps:**

1. Generate all subsets, recording the working slice directly.
2. Count distinct results against the 8 there should be.
3. Fix it two ways, and note what makes the bug appear.

```go
package main

import (
	"fmt"
	"slices"
)

// THE bug of recursive algorithms in Go. It is not a recursion bug at all --
// it is lesson 04's append-aliasing trap, and recursion is where it hides best.
//
// The pattern: build up a partial answer in a slice, and record a copy when you
// reach a leaf. If you record the slice ITSELF, every recorded answer is a view
// into one shared backing array, and later branches overwrite earlier results.

// --- the broken version ---

func subsetsBad(nums []int, i int, cur []int, out *[][]int) {
	if i == len(nums) {
		*out = append(*out, cur) // BUG: cur is a view, not a copy
		return
	}
	subsetsBad(nums, i+1, cur, out)                  // exclude nums[i]
	subsetsBad(nums, i+1, append(cur, nums[i]), out) // include nums[i]
}

// --- fix 1: copy at the moment you record ---

func subsetsClone(nums []int, i int, cur []int, out *[][]int) {
	if i == len(nums) {
		*out = append(*out, slices.Clone(cur)) // detach before storing
		return
	}
	subsetsClone(nums, i+1, cur, out)
	subsetsClone(nums, i+1, append(cur, nums[i]), out)
}

// --- fix 2: the classic choose / explore / un-choose ---
//
// One slice, mutated in place and restored on the way out. This is the shape
// used by every backtracking algorithm in lesson 29.
func subsetsBacktrack(nums []int, i int, cur []int, out *[][]int) {
	if i == len(nums) {
		*out = append(*out, slices.Clone(cur)) // still must copy at the leaf
		return
	}
	// don't choose nums[i]
	subsetsBacktrack(nums, i+1, cur, out)

	// choose
	cur = append(cur, nums[i])
	subsetsBacktrack(nums, i+1, cur, out)
	cur = cur[:len(cur)-1] // un-choose
}

func run(name string, f func([]int, int, []int, *[][]int)) [][]int {
	nums := []int{1, 2, 3}
	var out [][]int

	// NOTE the preallocation. This is the "optimization" that exposes the bug:
	// with `[]int{}` (cap 0) every append reallocates, so nothing is ever shared
	// and the broken version accidentally works. Reserve capacity -- exactly what
	// lesson 03 told you to do -- and the aliasing becomes real.
	cur := make([]int, 0, len(nums))

	f(nums, 0, cur, &out)
	fmt.Printf("  %-18s %v\n", name, out)
	return out
}

func main() {
	fmt.Println("all subsets of [1 2 3] -- there should be 8, all distinct:")
	fmt.Println()
	bad := run("shared slice", subsetsBad)
	good := run("slices.Clone", subsetsClone)
	bt := run("backtracking", subsetsBacktrack)

	fmt.Println()
	fmt.Printf("distinct results: shared=%d  clone=%d  backtrack=%d  (want 8)\n",
		countDistinct(bad), countDistinct(good), countDistinct(bt))

	fmt.Println()
	fmt.Println("why the first one is wrong:")
	fmt.Println()
	fmt.Println("  cur starts as []int{} with some capacity. `append(cur, nums[i])` in the")
	fmt.Println("  INCLUDE branch may write into cur's spare capacity rather than allocating.")
	fmt.Println("  Both branches then hold slices over the SAME array. Whatever the last")
	fmt.Println("  branch writes is what every previously-recorded slice now shows.")
	fmt.Println()
	fmt.Println("  Worse, it is intermittent: whether append reallocates depends on cap,")
	fmt.Println("  which depends on the input size. Small tests pass; production does not.")

	fmt.Println()
	fmt.Println("three defences, in order of preference:")
	fmt.Println()
	fmt.Println("  1. slices.Clone(cur) at the moment you RECORD. Cheap, obvious, always right.")
	fmt.Println("  2. choose / explore / un-choose: one buffer, mutated and restored. Still")
	fmt.Println("     needs the clone at the leaf, but allocates once instead of per branch.")
	fmt.Println("  3. cur[:len(cur):len(cur)] when passing down, forcing append to copy.")
	fmt.Println("     Correct, but it allocates on every branch -- use 1 or 2 instead.")

	fmt.Println()
	fmt.Println("the rule: a slice you hand to a recursive call is SHARED STATE.")
	fmt.Println("Either copy it before storing, or restore it before returning.")
}

func countDistinct(xss [][]int) int {
	seen := map[string]struct{}{}
	for _, xs := range xss {
		seen[fmt.Sprint(xs)] = struct{}{}
	}
	return len(seen)
}
```

**Output:**

```
all subsets of [1 2 3] -- there should be 8, all distinct:

  shared slice       [[] [1] [1] [1 2] [1] [1 2] [1 2] [1 2 3]]
  slices.Clone       [[] [3] [2] [2 3] [1] [1 3] [1 2] [1 2 3]]
  backtracking       [[] [3] [2] [2 3] [1] [1 3] [1 2] [1 2 3]]

distinct results: shared=4  clone=8  backtrack=8  (want 8)

why the first one is wrong:

  cur starts as []int{} with some capacity. `append(cur, nums[i])` in the
  INCLUDE branch may write into cur's spare capacity rather than allocating.
  Both branches then hold slices over the SAME array. Whatever the last
  branch writes is what every previously-recorded slice now shows.

  Worse, it is intermittent: whether append reallocates depends on cap,
  which depends on the input size. Small tests pass; production does not.

three defences, in order of preference:

  1. slices.Clone(cur) at the moment you RECORD. Cheap, obvious, always right.
  2. choose / explore / un-choose: one buffer, mutated and restored. Still
     needs the clone at the leaf, but allocates once instead of per branch.
  3. cur[:len(cur):len(cur)] when passing down, forcing append to copy.
     Correct, but it allocates on every branch -- use 1 or 2 instead.

the rule: a slice you hand to a recursive call is SHARED STATE.
Either copy it before storing, or restore it before returning.
```

**Four distinct results instead of eight.** And look at what triggers it: `cur` is preallocated with
`make([]int, 0, len(nums))`. With `[]int{}` (cap 0) every append reallocates, nothing is shared, and
the broken version *accidentally works*. Reserve capacity — exactly what lesson 03 told you to do —
and the aliasing becomes real. That's why this bug reaches production: it passes the small test.

**Complexity:** Θ(2ⁿ) subsets either way · `slices.Clone` at the leaf adds Θ(n) per recorded answer, which you were paying anyway to store it

---

## 11. Mutual recursion

`🟡 medium` · *When the data refers to itself through two shapes*

Two functions that call each other. The termination rules are unchanged — you just check them across
the whole cycle. The textbook `isEven`/`isOdd` pair is a bad way to test parity; the point is the
**data** underneath it.

**Steps:**

1. Write the classic pair and note why it's a poor algorithm.
2. Define a value that is a scalar, a list of values, or a named group — and walk it.
3. Hit the Go-specific wrinkle with local closures.

```go
package main

import "fmt"

// MUTUAL RECURSION: two or more functions that call each other. The base
// case and shrinking argument rules are exactly the same -- you just have to
// check them across the whole cycle rather than in one function.

// --- 1. the textbook pair ---

func isEven(n int) bool {
	if n == 0 {
		return true
	}
	return isOdd(n - 1)
}

func isOdd(n int) bool {
	if n == 0 {
		return false
	}
	return isEven(n - 1)
}

// --- 2. where it is actually the right tool: mutually recursive DATA ---
//
// A value is either a scalar, a list of values, or a named group of values.
// The data definition refers to itself through two shapes, so the code does too.

type value struct {
	scalar int
	list   []value
	group  map[string]value
	kind   string // "scalar", "list", "group"
}

func countScalars(v value) int {
	switch v.kind {
	case "scalar":
		return 1
	case "list":
		return countList(v.list)
	case "group":
		return countGroup(v.group)
	}
	return 0
}

func countList(vs []value) int {
	total := 0
	for _, v := range vs {
		total += countScalars(v) // back into the other function
	}
	return total
}

func countGroup(g map[string]value) int {
	total := 0
	for _, v := range g {
		total += countScalars(v)
	}
	return total
}

func depthOf(v value) int {
	switch v.kind {
	case "scalar":
		return 1
	case "list":
		best := 0
		for _, c := range v.list {
			if d := depthOf(c); d > best {
				best = d
			}
		}
		return 1 + best
	case "group":
		best := 0
		for _, c := range v.group {
			if d := depthOf(c); d > best {
				best = d
			}
		}
		return 1 + best
	}
	return 0
}

func scalar(n int) value     { return value{kind: "scalar", scalar: n} }
func list(vs ...value) value { return value{kind: "list", list: vs} }
func group(m map[string]value) value {
	return value{kind: "group", group: m}
}

func main() {
	fmt.Println("1. the textbook pair -- isEven and isOdd call each other:")
	fmt.Println()
	for _, n := range []int{0, 1, 6, 7} {
		fmt.Printf("   %d: even=%-5v odd=%v\n", n, isEven(n), isOdd(n))
	}
	fmt.Println()
	fmt.Println("   correct, and a terrible way to test parity: it is Theta(n) time and")
	fmt.Println("   Theta(n) stack for something `n%2 == 0` answers in one instruction.")
	fmt.Println("   Mutual recursion is not the point here -- the DATA below is.")

	fmt.Println()
	fmt.Println("2. mutually recursive data needs mutually recursive code:")
	fmt.Println()

	doc := group(map[string]value{
		"id":   scalar(1),
		"tags": list(scalar(10), scalar(20), scalar(30)),
		"meta": group(map[string]value{
			"version": scalar(2),
			"nested":  list(scalar(40), list(scalar(50), scalar(60))),
		}),
	})

	fmt.Printf("   scalars in the document: %d\n", countScalars(doc))
	fmt.Printf("   maximum nesting depth:   %d\n", depthOf(doc))
	fmt.Println()
	fmt.Println("   a value contains lists and groups; lists and groups contain values.")
	fmt.Println("   The type definition is mutually recursive, so countScalars/countList/")
	fmt.Println("   countGroup are too. This is what JSON, XML, ASTs and file systems")
	fmt.Println("   all look like -- and why recursion is the natural tool for them.")

	fmt.Println()
	fmt.Println("3. a Go-specific wrinkle:")
	fmt.Println()
	fmt.Println("   package-level functions can call each other in any order -- isEven")
	fmt.Println("   references isOdd before it is declared, and that is fine.")
	fmt.Println()
	fmt.Println("   LOCAL closures cannot. This does not compile:")
	fmt.Println("       even := func(n int) bool { ... odd(n-1) ... }   // odd undefined")
	fmt.Println("   You must declare the variable first:")
	fmt.Println("       var odd func(int) bool")
	fmt.Println("       even := func(n int) bool { ... odd(n-1) ... }")
	fmt.Println("       odd = func(n int) bool { ... even(n-1) ... }")

	// Demonstrating exactly that.
	var odd func(int) bool
	even := func(n int) bool {
		if n == 0 {
			return true
		}
		return odd(n - 1)
	}
	odd = func(n int) bool {
		if n == 0 {
			return false
		}
		return even(n - 1)
	}
	fmt.Printf("\n   the closure version works once declared this way: even(6)=%v odd(6)=%v\n",
		even(6), odd(6))
	fmt.Println()
	fmt.Println("   (recursive closures are also much more expensive per frame -- example 9")
	fmt.Println("    measured ~6.7 KB against ~40 bytes for a plain function.)")
}
```

**Output:**

```
1. the textbook pair -- isEven and isOdd call each other:

   0: even=true  odd=false
   1: even=false odd=true
   6: even=true  odd=false
   7: even=false odd=true

   correct, and a terrible way to test parity: it is Theta(n) time and
   Theta(n) stack for something `n%2 == 0` answers in one instruction.
   Mutual recursion is not the point here -- the DATA below is.

2. mutually recursive data needs mutually recursive code:

   scalars in the document: 8
   maximum nesting depth:   5

   a value contains lists and groups; lists and groups contain values.
   The type definition is mutually recursive, so countScalars/countList/
   countGroup are too. This is what JSON, XML, ASTs and file systems
   all look like -- and why recursion is the natural tool for them.

3. a Go-specific wrinkle:

   package-level functions can call each other in any order -- isEven
   references isOdd before it is declared, and that is fine.

   LOCAL closures cannot. This does not compile:
       even := func(n int) bool { ... odd(n-1) ... }   // odd undefined
   You must declare the variable first:
       var odd func(int) bool
       even := func(n int) bool { ... odd(n-1) ... }
       odd = func(n int) bool { ... even(n-1) ... }

   the closure version works once declared this way: even(6)=true odd(6)=false

   (recursive closures are also much more expensive per frame -- example 9
    measured ~6.7 KB against ~40 bytes for a plain function.)
```

A value contains lists and groups; lists and groups contain values. The **type** is mutually
recursive, so the code is too. That's the shape of JSON, XML, ASTs and filesystems — and the reason
recursion is the natural tool for them, which example 16 builds on directly.

The Go wrinkle is worth remembering: package-level functions can reference each other in any order,
but a local closure cannot refer to a sibling that doesn't exist yet. Declare `var odd func(int) bool`
first.

**Complexity:** `isEven(n)` is Θ(n) time and Θ(n) stack for something `n%2` answers in one instruction · the document walk is Θ(nodes) time and Θ(nesting depth) stack

---

> Next tier: [🔴 hard](3-hard.md) — backtracking, depth as a design constraint, and a parser.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
