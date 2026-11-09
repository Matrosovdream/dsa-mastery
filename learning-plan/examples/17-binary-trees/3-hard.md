# Step 17 — Binary Trees & Traversals · 🔴 Hard

Examples **12–16**: in-order with no stack, a tree you did not build (stack frames, depth and `Drop`), one
tree owned three ways, traversals that can stop, and the capstone.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match character for character; **Sample
output:** ones print timings.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Morris traversal: in-order with no stack

`🔴 hard` · *Threaded traversal*

In-order in Θ(n) time and **O(1) extra space**: for every node with a left child, point the empty right link of its in-order predecessor back at it (a *thread*), and cut the thread when the walk comes back. The way up is stored in the tree's own nil links instead of a stack, which is the right tool when the stack is the thing you cannot afford, and the price is that the tree is **modified while the walk runs**: stop early and it is left with a cycle. The program prints the walk event by event, counts the moves that make it Θ(n) (each node with a left child costs one extra move), checks it against the recursive orders on 2,000 random trees, stops a walk early to show the damage, and times it against the two stack versions. Rust cannot do this on a `Box` tree, because a thread is a second owner of a node, so its twin runs on an index arena.

**Steps:**

1. Run the walk on the hand-built tree with every thread, cut and visit printed, then count threads, cuts, moves and predecessor steps on random trees of 15, 1,000 and 100,000 nodes.
2. Compare the in-order walk and the pre-order variant with the recursive orders on 2,000 random trees, checking that each tree comes back unchanged; then stop a walk after 3 visits and run a recursive `size` on what is left.
3. Read the relations (threads = cuts = nodes with a left child, moves = n + threads), the live-threads-versus-stack columns, the cycle and the timing table.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// Morris traversal: in-order with no stack. For a node c with a left child,
// find its in-order predecessor p (the rightmost node of c's left subtree) and
// point p.Right back at c -- a THREAD, borrowed from p's empty right link. When
// the walk comes back to c through the thread, the left subtree is finished:
// cut the thread, visit c, go right. The way back to every ancestor is stored
// in the tree's own nil links, so no stack is needed -- and the tree is
// modified while the walk is in progress.

type Node struct {
	Val         int
	Left, Right *Node
}

func leaf(v int) *Node { return &Node{Val: v} }

// handBuilt is the 9-node tree used by the whole lesson.
func handBuilt() *Node {
	//         1
	//       /   \
	//      2     3
	//     / \     \
	//    4   5     6
	//       /     / \
	//      7     8   9
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// randomSplit is the shape only: the left subtree gets a uniformly random share
// of the other n-1 nodes. The draw happens BEFORE the recursion, so the stream
// is consumed in a fixed order (the Rust twin makes the same draws).
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{Left: left, Right: right}
}

// complete is heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
func complete(i, n int) *Node {
	if i >= n {
		return nil
	}
	return &Node{Val: i, Left: complete(2*i+1, n), Right: complete(2*i+2, n)}
}

// labelInOrder gives every node its rank, so an in-order walk prints 0, 1, 2...
func labelInOrder(n *Node, next *int) {
	if n == nil {
		return
	}
	labelInOrder(n.Left, next)
	n.Val = *next
	*next++
	labelInOrder(n.Right, next)
}

func height(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.Left), height(n.Right))
}

func clone(n *Node) *Node {
	if n == nil {
		return nil
	}
	return &Node{n.Val, clone(n.Left), clone(n.Right)}
}

func same(a, b *Node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Val == b.Val && same(a.Left, b.Left) && same(a.Right, b.Right)
}

// ---- the references Morris is checked against ----

func inRec(n *Node, out *[]int) {
	if n == nil {
		return
	}
	inRec(n.Left, out)
	*out = append(*out, n.Val)
	inRec(n.Right, out)
}

func preRec(n *Node, out *[]int) {
	if n == nil {
		return
	}
	*out = append(*out, n.Val)
	preRec(n.Left, out)
	preRec(n.Right, out)
}

// inStack is the explicit-stack in-order of example 3; it also reports the
// most entries its stack held at once.
func inStack(root *Node) (seq []int, hw int) {
	var st []*Node
	for cur := root; cur != nil || len(st) > 0; {
		for cur != nil {
			st = append(st, cur)
			hw = max(hw, len(st))
			cur = cur.Left
		}
		cur = st[len(st)-1]
		st = st[:len(st)-1]
		seq = append(seq, cur.Val)
		cur = cur.Right
	}
	return
}

// leftChildren and spineSum are COUNTED BY LOOKING at the tree, not by running
// Morris: how many nodes have a left child, and the sum over them of
// (length of the right spine of the left child) - 1.
func leftChildren(n *Node) int {
	if n == nil {
		return 0
	}
	c := leftChildren(n.Left) + leftChildren(n.Right)
	if n.Left != nil {
		c++
	}
	return c
}

func spineSum(n *Node) int {
	if n == nil {
		return 0
	}
	s := spineSum(n.Left) + spineSum(n.Right)
	if n.Left != nil {
		for p := n.Left; p.Right != nil; p = p.Right {
			s++
		}
	}
	return s
}

// ---- Morris, as a resumable walk ----

type stats struct {
	visits, threads, cuts, moves, predSteps int
	maxLive                                 int // most threads in the tree at once
}

// walker is Morris traversal as a value you can pause: next() runs until the
// next visit. In-order visits when the walk comes back to a node; pre-order
// (pre = true) visits when the thread is made, on the way down.
type walker struct {
	cur  *Node
	pre  bool
	live int // threads currently in the tree
	st   stats
	log  *[]string // when not nil, every event is appended here
}

func (w *walker) event(format string, args ...any) {
	if w.log != nil {
		*w.log = append(*w.log, fmt.Sprintf(format, args...))
	}
}

func (w *walker) next() (*Node, bool) {
	for w.cur != nil {
		w.st.moves++
		c := w.cur
		if c.Left == nil { // nothing to thread: visit, then go right
			// (when c.Right is a thread this is how the walk climbs back up)
			w.cur = c.Right
			w.st.visits++
			w.event("visit %d", c.Val)
			return c, true
		}
		p := c.Left                          // find the in-order predecessor: rightmost of the left subtree...
		for p.Right != nil && p.Right != c { // ...or the thread we left earlier
			p = p.Right
			w.st.predSteps++
		}
		if p.Right == nil { // first arrival: make the thread, descend
			p.Right = c
			w.st.threads++
			w.live++
			w.st.maxLive = max(w.st.maxLive, w.live)
			w.cur = c.Left
			w.event("thread %d -> %d", p.Val, c.Val)
			if w.pre {
				w.st.visits++
				w.event("visit %d", c.Val)
				return c, true
			}
		} else { // second arrival: the left subtree is done; cut the thread
			p.Right = nil
			w.st.cuts++
			w.live--
			w.cur = c.Right
			w.event("cut    %d -> %d", p.Val, c.Val)
			if !w.pre {
				w.st.visits++
				w.event("visit %d", c.Val)
				return c, true
			}
		}
	}
	return nil, false
}

// morris runs a walk to the end and returns the sequence and the counters.
func morris(root *Node, pre bool) ([]int, stats) {
	w := &walker{cur: root, pre: pre}
	var seq []int
	for n, ok := w.next(); ok; n, ok = w.next() {
		seq = append(seq, n.Val)
	}
	return seq, w.st
}

// ---- the hazard: what a half-finished walk leaves behind ----

// sizeCapped is the ordinary recursive size, except that it gives up after
// `cap` calls. A real tree of n nodes needs exactly 2n+1 calls.
func sizeCapped(n *Node, calls *int, cap int) (int, bool) {
	*calls++
	if *calls > cap {
		return 0, false
	}
	if n == nil {
		return 0, true
	}
	l, ok := sizeCapped(n.Left, calls, cap)
	if !ok {
		return 0, false
	}
	r, ok := sizeCapped(n.Right, calls, cap)
	if !ok {
		return 0, false
	}
	return 1 + l + r, true
}

// extraLinks scans with a seen-set (not by recursion, so it is safe on a
// threaded tree) and returns the links that lead to a node already reached.
// A tree has none; every live thread is one.
func extraLinks(root *Node) (extra []string) {
	if root == nil {
		return
	}
	seen := map[*Node]bool{root: true}
	work := []*Node{root}
	for len(work) > 0 {
		n := work[len(work)-1]
		work = work[:len(work)-1]
		for side, c := range []*Node{n.Left, n.Right} {
			if c == nil {
				continue
			}
			if seen[c] {
				extra = append(extra, fmt.Sprintf("%d.%s -> %d", n.Val, []string{"Left", "Right"}[side], c.Val))
				continue
			}
			seen[c] = true
			work = append(work, c)
		}
	}
	slices.Sort(extra)
	return
}

// ---- the timing contestants: the same checksum, three ways ----

func mix(acc uint64, v int) uint64 { return acc*31 + uint64(v) }

func inRecSum(n *Node, acc *uint64) {
	if n == nil {
		return
	}
	inRecSum(n.Left, acc)
	*acc = mix(*acc, n.Val)
	inRecSum(n.Right, acc)
}

func inStackSum(root *Node, st []*Node) uint64 {
	var acc uint64
	st = st[:0]
	for cur := root; cur != nil || len(st) > 0; {
		for cur != nil {
			st = append(st, cur)
			cur = cur.Left
		}
		cur = st[len(st)-1]
		st = st[:len(st)-1]
		acc = mix(acc, cur.Val)
		cur = cur.Right
	}
	return acc
}

// morrisSum is the walker's loop without the counters and the pause.
func morrisSum(root *Node) uint64 {
	var acc uint64
	for cur := root; cur != nil; {
		if cur.Left == nil {
			acc = mix(acc, cur.Val)
			cur = cur.Right
			continue
		}
		p := cur.Left
		for p.Right != nil && p.Right != cur {
			p = p.Right
		}
		if p.Right == nil {
			p.Right = cur
			cur = cur.Left
		} else {
			p.Right = nil
			acc = mix(acc, cur.Val)
			cur = cur.Right
		}
	}
	return acc
}

type bench struct {
	name string
	root *Node
	n    int
}

func sideways(n *Node, depth int) {
	if n == nil {
		return
	}
	sideways(n.Right, depth+1)
	fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.Val)
	sideways(n.Left, depth+1)
}

func joinInts(a []int) string {
	s := make([]string, len(a))
	for i, v := range a {
		s[i] = fmt.Sprint(v)
	}
	return strings.Join(s, " ")
}

func main() {
	// ---- (a) the walk, event by event, on the hand-built tree ----
	root := handBuilt()
	fmt.Println("the hand-built tree, sideways:")
	sideways(root, 0)

	var events []string
	w := &walker{cur: root, log: &events}
	var seq []int
	for n, ok := w.next(); ok; n, ok = w.next() {
		seq = append(seq, n.Val)
	}
	fmt.Println("\nMorris in-order, every event (thread p -> c points p's empty right link at c; cut removes it):")
	for i, e := range events {
		fmt.Printf("  %2d  %s\n", i+1, e)
	}
	fmt.Println("in-order:", joinInts(seq))
	var ref []int
	inRec(root, &ref)
	fmt.Printf("== recursive: %v, tree restored: %v, threads %d cuts %d visits %d moves %d pred steps %d, max live threads %d\n",
		slices.Equal(seq, ref), same(root, handBuilt()),
		w.st.threads, w.st.cuts, w.st.visits, w.st.moves, w.st.predSteps, w.st.maxLive)

	// ---- (b) the counts, on trees of 15, 1000 and 100000 nodes ----
	fmt.Println("\ncounts on randomSplit trees (one fresh PCG(1,2), drawn in this order):")
	fmt.Printf("%7s %4s %7s %7s %7s %8s %8s %8s %7s %9s %9s\n",
		"n", "h", "visits", "threads", "cuts", "left-ch", "moves", "pred", "hops/n", "max live", "stack hw")
	r := rand.New(rand.NewPCG(1, 2))
	type row struct {
		n, left, spine, hw int
		st                 stats
	}
	var rows []row
	for _, n := range []int{15, 1000, 100000} {
		t := randomSplit(n, r)
		next := 0
		labelInOrder(t, &next)
		_, st := morris(t, false)
		_, hw := inStack(t)
		lc := leftChildren(t)
		rows = append(rows, row{n, lc, spineSum(t), hw, st})
		fmt.Printf("%7d %4d %7d %7d %7d %8d %8d %8d %7.2f %9d %9d\n",
			n, height(t), st.visits, st.threads, st.cuts, lc, st.moves, st.predSteps,
			float64(st.moves+st.predSteps)/float64(n), st.maxLive, hw)
	}
	fmt.Println("(left-ch = nodes with a left child, counted by looking; hops/n = (moves + pred) / n, the pointer hops Morris makes per node;")
	fmt.Println(" max live = most threads in the tree at once; stack hw = high-water of the explicit in-order stack)")
	ok1, ok2, ok3, ok4, ok5 := true, true, true, true, true
	for _, x := range rows {
		ok1 = ok1 && x.st.threads == x.st.cuts && x.st.cuts == x.left
		ok2 = ok2 && x.st.visits == x.n
		ok3 = ok3 && x.st.moves == x.n+x.st.threads
		ok4 = ok4 && x.st.predSteps == 2*x.spine
		ok5 = ok5 && x.hw == x.st.maxLive+1
	}
	fmt.Printf("threads == cuts == nodes with a left child: %v\n", ok1)
	fmt.Printf("visits == n: %v\n", ok2)
	fmt.Printf("moves == n + threads: %v\n", ok3)
	fmt.Printf("pred steps == 2 * sum(right spine of the left child - 1): %v\n", ok4)
	fmt.Printf("explicit stack high-water == max live threads + 1: %v\n", ok5)

	// ---- (c) 2000 random trees: same sequences, and the tree comes back ----
	r = rand.New(rand.NewPCG(1, 2))
	inBad, preBad, notRestored, relBad, nodes := 0, 0, 0, 0, 0
	for range 2000 {
		n := r.IntN(80)
		t := randomSplit(n, r)
		next := 0
		labelInOrder(t, &next)
		nodes += n
		orig := clone(t)
		var wantIn, wantPre []int
		inRec(t, &wantIn)
		preRec(t, &wantPre)

		gotIn, st := morris(t, false)
		_, hw := inStack(t)
		if !slices.Equal(gotIn, wantIn) {
			inBad++
		}
		if !same(t, orig) {
			notRestored++
		}
		if st.threads != st.cuts || st.cuts != leftChildren(t) || st.moves != n+st.threads ||
			(n > 0 && hw != st.maxLive+1) {
			relBad++
		}
		gotPre, _ := morris(t, true)
		if !slices.Equal(gotPre, wantPre) {
			preBad++
		}
		if !same(t, orig) {
			notRestored++
		}
	}
	fmt.Printf("\n2000 random trees, sizes 0..79 (%d nodes in total):\n", nodes)
	fmt.Printf("  in-order mismatches vs recursive: %d, pre-order mismatches: %d\n", inBad, preBad)
	fmt.Printf("  trees not restored after a full walk (checked after both walks): %d\n", notRestored)
	fmt.Printf("  trees where a relation of (b) failed (threads, cuts, moves, stack high-water): %d\n", relBad)

	// ---- (d) the hazard: stop early, and the tree is a cycle ----
	fmt.Println("\nstop after 3 visits, then run an ordinary recursive size on the same tree:")
	root = handBuilt()
	w = &walker{cur: root}
	for range 3 {
		n, _ := w.next()
		fmt.Printf("  visited %d\n", n.Val)
	}
	extra := extraLinks(root)
	fmt.Printf("  threads made %d, cut %d: %d still in the tree\n", w.st.threads, w.st.cuts, w.live)
	fmt.Printf("  links to a node already reached (found with a seen-set): %s\n", strings.Join(extra, ", "))
	calls := 0
	n := 9
	_, done := sizeCapped(root, &calls, 10*n)
	fmt.Printf("  size with a cap of 10n = %d calls: finished = %v after %d calls (a real tree needs %d)\n", 10*n, done, calls, 2*n+1)
	if !done {
		fmt.Println("  did not finish: the tree contains a cycle")
	}
	fmt.Println("  now let the walk go on:")
	for nd, ok := w.next(); ok; nd, ok = w.next() {
		fmt.Printf("  visited %d\n", nd.Val)
	}
	extra = extraLinks(root)
	calls = 0
	size, done := sizeCapped(root, &calls, 10*n)
	fmt.Printf("  walk finished: threads %d, cuts %d; links to reached nodes: %d; size = %d (finished = %v, %d calls); restored: %v\n",
		w.st.threads, w.st.cuts, len(extra), size, done, calls, same(root, handBuilt()))

	// ---- (e) one timing table (the output differs from run to run) ----
	fmt.Println("\nns per node, best of 3 rounds (these numbers move from run to run):")
	r = rand.New(rand.NewPCG(1, 2))
	random := randomSplit(1_000_000, r)
	next := 0
	labelInOrder(random, &next)
	trees := []bench{
		{"randomSplit(1000000)", random, 1_000_000},
		{"complete(1048575)", complete(0, 1_048_575), 1_048_575},
	}
	best := make([][3]time.Duration, len(trees))
	for i := range best {
		best[i] = [3]time.Duration{1 << 62, 1 << 62, 1 << 62}
	}
	st := make([]*Node, 0, 128)
	for round := 0; round < 3; round++ {
		for i, tr := range trees {
			var sums [3]uint64
			t0 := time.Now()
			inRecSum(tr.root, &sums[0])
			best[i][0] = min(best[i][0], time.Since(t0))
			t0 = time.Now()
			sums[1] = inStackSum(tr.root, st)
			best[i][1] = min(best[i][1], time.Since(t0))
			t0 = time.Now()
			sums[2] = morrisSum(tr.root)
			best[i][2] = min(best[i][2], time.Since(t0))
			if sums[0] != sums[1] || sums[1] != sums[2] {
				fmt.Println("CHECKSUM MISMATCH")
			}
		}
	}
	fmt.Printf("%-22s %10s %12s %8s %15s %16s\n", "tree", "recursive", "explicit", "Morris", "Morris/explicit", "Morris/recursive")
	for i, tr := range trees {
		ns := func(d time.Duration) float64 { return float64(d.Nanoseconds()) / float64(tr.n) }
		fmt.Printf("%-22s %10.2f %12.2f %8.2f %14.2fx %15.2fx\n", tr.name, ns(best[i][0]), ns(best[i][1]), ns(best[i][2]),
			ns(best[i][2])/ns(best[i][1]), ns(best[i][2])/ns(best[i][0]))
	}
	fmt.Println("extra memory while walking: recursive = h stack frames, explicit = a slice that peaks near h, Morris = two local pointers (cur and p)")
}
```

**Sample output:**

```
the hand-built tree, sideways:
            9
        6
            8
    3
1
        5
            7
    2
        4

Morris in-order, every event (thread p -> c points p's empty right link at c; cut removes it):
   1  thread 5 -> 1
   2  thread 4 -> 2
   3  visit 4
   4  cut    4 -> 2
   5  visit 2
   6  thread 7 -> 5
   7  visit 7
   8  cut    7 -> 5
   9  visit 5
  10  cut    5 -> 1
  11  visit 1
  12  visit 3
  13  thread 8 -> 6
  14  visit 8
  15  cut    8 -> 6
  16  visit 6
  17  visit 9
in-order: 4 2 7 5 1 3 8 6 9
== recursive: true, tree restored: true, threads 4 cuts 4 visits 9 moves 13 pred steps 2, max live threads 2

counts on randomSplit trees (one fresh PCG(1,2), drawn in this order):
      n    h  visits threads    cuts  left-ch    moves     pred  hops/n  max live  stack hw
     15    6      15       8       8        8       23       10    2.20         4         5
   1000   20    1000     520     520      520     1520      952    2.47        16        17
 100000   41  100000   50029   50029    50029   150029    99932    2.50        26        27
(left-ch = nodes with a left child, counted by looking; hops/n = (moves + pred) / n, the pointer hops Morris makes per node;
 max live = most threads in the tree at once; stack hw = high-water of the explicit in-order stack)
threads == cuts == nodes with a left child: true
visits == n: true
moves == n + threads: true
pred steps == 2 * sum(right spine of the left child - 1): true
explicit stack high-water == max live threads + 1: true

2000 random trees, sizes 0..79 (79730 nodes in total):
  in-order mismatches vs recursive: 0, pre-order mismatches: 0
  trees not restored after a full walk (checked after both walks): 0
  trees where a relation of (b) failed (threads, cuts, moves, stack high-water): 0

stop after 3 visits, then run an ordinary recursive size on the same tree:
  visited 4
  visited 2
  visited 7
  threads made 3, cut 1: 2 still in the tree
  links to a node already reached (found with a seen-set): 5.Right -> 1, 7.Right -> 5
  size with a cap of 10n = 90 calls: finished = false after 91 calls (a real tree needs 19)
  did not finish: the tree contains a cycle
  now let the walk go on:
  visited 5
  visited 1
  visited 3
  visited 8
  visited 6
  visited 9
  walk finished: threads 4, cuts 4; links to reached nodes: 0; size = 9 (finished = true, 19 calls); restored: true

ns per node, best of 3 rounds (these numbers move from run to run):
tree                    recursive     explicit   Morris Morris/explicit Morris/recursive
randomSplit(1000000)         6.12         5.63     7.24           1.29x            1.18x
complete(1048575)            2.14         1.80     2.83           1.57x            1.32x
extra memory while walking: recursive = h stack frames, explicit = a slice that peaks near h, Morris = two local pointers (cur and p)
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Morris traversal: in-order with no stack. For a node c with a left child,
// find its in-order predecessor p (the rightmost node of c's left subtree) and
// point p.right back at c -- a THREAD, borrowed from p's empty right link. When
// the walk comes back to c through the thread, the left subtree is finished:
// cut the thread, visit c, go right. The way back to every ancestor is stored
// in the tree's own empty links, so no stack is needed -- and the tree is
// modified while the walk is in progress.
//
// Why this twin is an ARENA and not a tree of `Option<Box<Node>>`: a thread is a
// second link to a node that its parent already owns. A Box is the one owner,
// so the thread cannot be a Box -- and a reference cannot be stored in a Box's
// place either. Three attempts at it were compiled separately (they are not
// part of this program, which must build) and were refused on rustc 1.98.1:
//   `p.right = Some(c)` with c: Box<Node>       -> E0505 cannot move out of `c`
//                                                  because it is borrowed (by p)
//   `p.right = Some(c)` with c: &mut Box<Node>  -> E0308 mismatched types
//   a `thread: Option<&'a Node<'a>>` field set from `&*root` while the walk
//   holds `&mut` into root's subtree            -> E0502 cannot borrow `*root`
//                                                  as immutable (also borrowed as mutable)
// In an arena a link is a `u32` index: a thread is just another index, with no
// ownership attached. This is the arena of lesson 07 example 13.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed gives the same trees as the Go program.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

const NIL: u32 = u32::MAX; // the empty tree: an index no node has

#[derive(Clone, Copy, PartialEq)]
struct N {
    val: i64,
    left: u32,
    right: u32,
}

struct Arena {
    nodes: Vec<N>,
}

impl Arena {
    fn push(&mut self, val: i64, left: u32, right: u32) -> u32 {
        self.nodes.push(N { val, left, right });
        (self.nodes.len() - 1) as u32
    }

    fn leaf(&mut self, v: i64) -> u32 {
        self.push(v, NIL, NIL)
    }

    /// The hand-built 9-node tree used by the whole lesson.
    fn hand_built(&mut self) -> u32 {
        //         1
        //       /   \
        //      2     3
        //     / \     \
        //    4   5     6
        //       /     / \
        //      7     8   9
        let (n4, n7) = (self.leaf(4), self.leaf(7));
        let n5 = self.push(5, n7, NIL);
        let n2 = self.push(2, n4, n5);
        let (n8, n9) = (self.leaf(8), self.leaf(9));
        let n6 = self.push(6, n8, n9);
        let n3 = self.push(3, NIL, n6);
        self.push(1, n2, n3)
    }

    /// The shape only: the left subtree gets a uniformly random share of the
    /// other n-1 nodes. The draw happens BEFORE the recursion, so the stream is
    /// consumed in the order Go consumes it.
    fn random_split(&mut self, n: usize, r: &mut Pcg) -> u32 {
        if n == 0 {
            return NIL;
        }
        let k = r.int_n(n as u64) as usize;
        let id = self.leaf(0);
        let left = self.random_split(k, r);
        let right = self.random_split(n - 1 - k, r);
        self.nodes[id as usize].left = left;
        self.nodes[id as usize].right = right;
        id
    }

    /// Heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
    fn complete(&mut self, i: usize, n: usize) -> u32 {
        if i >= n {
            return NIL;
        }
        let id = self.leaf(i as i64);
        let left = self.complete(2 * i + 1, n);
        let right = self.complete(2 * i + 2, n);
        self.nodes[id as usize].left = left;
        self.nodes[id as usize].right = right;
        id
    }

    /// Gives every node its rank, so an in-order walk prints 0, 1, 2...
    fn label_in_order(&mut self, id: u32, next: &mut i64) {
        if id == NIL {
            return;
        }
        let N { left, right, .. } = self.nodes[id as usize];
        self.label_in_order(left, next);
        self.nodes[id as usize].val = *next;
        *next += 1;
        self.label_in_order(right, next);
    }

    fn height(&self, id: u32) -> usize {
        if id == NIL {
            return 0;
        }
        let n = self.nodes[id as usize];
        1 + self.height(n.left).max(self.height(n.right))
    }

    // ---- the references Morris is checked against ----

    fn in_rec(&self, id: u32, out: &mut Vec<i64>) {
        if id == NIL {
            return;
        }
        let n = self.nodes[id as usize];
        self.in_rec(n.left, out);
        out.push(n.val);
        self.in_rec(n.right, out);
    }

    fn pre_rec(&self, id: u32, out: &mut Vec<i64>) {
        if id == NIL {
            return;
        }
        let n = self.nodes[id as usize];
        out.push(n.val);
        self.pre_rec(n.left, out);
        self.pre_rec(n.right, out);
    }

    /// The explicit-stack in-order of example 3; it also reports the most
    /// entries its stack held at once.
    fn in_stack(&self, root: u32) -> (Vec<i64>, usize) {
        let (mut seq, mut hw, mut st) = (Vec::new(), 0, Vec::new());
        let mut cur = root;
        while cur != NIL || !st.is_empty() {
            while cur != NIL {
                st.push(cur);
                hw = hw.max(st.len());
                cur = self.nodes[cur as usize].left;
            }
            cur = st.pop().expect("stack is not empty here");
            seq.push(self.nodes[cur as usize].val);
            cur = self.nodes[cur as usize].right;
        }
        (seq, hw)
    }

    /// Counted BY LOOKING at the tree, not by running Morris: how many nodes
    /// have a left child ...
    fn left_children(&self, id: u32) -> usize {
        if id == NIL {
            return 0;
        }
        let n = self.nodes[id as usize];
        self.left_children(n.left) + self.left_children(n.right) + usize::from(n.left != NIL)
    }

    /// ... and the sum over them of (length of the right spine of the left child) - 1.
    fn spine_sum(&self, id: u32) -> usize {
        if id == NIL {
            return 0;
        }
        let n = self.nodes[id as usize];
        let mut s = self.spine_sum(n.left) + self.spine_sum(n.right);
        if n.left != NIL {
            let mut p = n.left;
            while self.nodes[p as usize].right != NIL {
                p = self.nodes[p as usize].right;
                s += 1;
            }
        }
        s
    }

    /// Ordinary recursive size, except that it gives up after `cap` calls. A
    /// real tree of n nodes needs exactly 2n+1 calls.
    fn size_capped(&self, id: u32, calls: &mut usize, cap: usize) -> Option<usize> {
        *calls += 1;
        if *calls > cap {
            return None;
        }
        if id == NIL {
            return Some(0);
        }
        let n = self.nodes[id as usize];
        let l = self.size_capped(n.left, calls, cap)?;
        let r = self.size_capped(n.right, calls, cap)?;
        Some(1 + l + r)
    }

    /// Scans with a seen-set (not by recursion, so it is safe on a threaded
    /// tree) and returns the links that lead to a node already reached. A tree
    /// has none; every live thread is one.
    fn extra_links(&self, root: u32) -> Vec<String> {
        let mut extra = Vec::new();
        if root == NIL {
            return extra;
        }
        let mut seen = vec![false; self.nodes.len()];
        seen[root as usize] = true;
        let mut work = vec![root];
        while let Some(id) = work.pop() {
            let n = self.nodes[id as usize];
            for (side, c) in [("Left", n.left), ("Right", n.right)] {
                if c == NIL {
                    continue;
                }
                if seen[c as usize] {
                    extra.push(format!(
                        "{}.{} -> {}",
                        n.val, side, self.nodes[c as usize].val
                    ));
                    continue;
                }
                seen[c as usize] = true;
                work.push(c);
            }
        }
        extra.sort();
        extra
    }

    /// Prints the right subtree ABOVE the node and the left BELOW it.
    fn sideways(&self, id: u32, depth: usize) {
        if id == NIL {
            return;
        }
        let n = self.nodes[id as usize];
        self.sideways(n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.val);
        self.sideways(n.left, depth + 1);
    }
}

// ---- Morris, as a resumable walk ----

#[derive(Default, Clone, Copy)]
struct Stats {
    visits: usize,
    threads: usize,
    cuts: usize,
    moves: usize,
    pred_steps: usize,
    max_live: usize, // most threads in the tree at once
}

/// Morris traversal as a value you can pause: `next` runs until the next
/// visit. In-order visits when the walk comes back to a node; pre-order
/// (`pre`) visits when the thread is made, on the way down.
struct Walker {
    cur: u32,
    pre: bool,
    live: usize, // threads currently in the tree
    st: Stats,
    log: Option<Vec<String>>, // when Some, every event is appended here
}

impl Walker {
    fn new(root: u32, pre: bool, logging: bool) -> Self {
        Walker {
            cur: root,
            pre,
            live: 0,
            st: Stats::default(),
            log: logging.then(Vec::new),
        }
    }

    fn event(&mut self, text: impl FnOnce() -> String) {
        if let Some(log) = &mut self.log {
            log.push(text());
        }
    }

    /// The arena is passed in on every call: the walker owns no tree, it only
    /// holds an index (the borrow checker is happy because nothing is stored).
    fn next(&mut self, a: &mut Arena) -> Option<u32> {
        while self.cur != NIL {
            self.st.moves += 1;
            let c = self.cur;
            let cn = a.nodes[c as usize];
            if cn.left == NIL {
                // nothing to thread: visit, then go right
                // (when c.right is a thread this is how the walk climbs back up)
                self.cur = cn.right;
                self.st.visits += 1;
                self.event(|| format!("visit {}", cn.val));
                return Some(c);
            }
            // find the in-order predecessor: rightmost of the left subtree...
            let mut p = cn.left;
            // ...or the thread we left earlier
            while a.nodes[p as usize].right != NIL && a.nodes[p as usize].right != c {
                p = a.nodes[p as usize].right;
                self.st.pred_steps += 1;
            }
            let pv = a.nodes[p as usize].val;
            if a.nodes[p as usize].right == NIL {
                // first arrival: make the thread, descend
                a.nodes[p as usize].right = c;
                self.st.threads += 1;
                self.live += 1;
                self.st.max_live = self.st.max_live.max(self.live);
                self.cur = cn.left;
                self.event(|| format!("thread {} -> {}", pv, cn.val));
                if self.pre {
                    self.st.visits += 1;
                    self.event(|| format!("visit {}", cn.val));
                    return Some(c);
                }
            } else {
                // second arrival: the left subtree is done; cut the thread
                a.nodes[p as usize].right = NIL;
                self.st.cuts += 1;
                self.live -= 1;
                self.cur = cn.right;
                self.event(|| format!("cut    {} -> {}", pv, cn.val));
                if !self.pre {
                    self.st.visits += 1;
                    self.event(|| format!("visit {}", cn.val));
                    return Some(c);
                }
            }
        }
        None
    }
}

/// Runs a walk to the end and returns the sequence and the counters.
fn morris(a: &mut Arena, root: u32, pre: bool) -> (Vec<i64>, Stats) {
    let mut w = Walker::new(root, pre, false);
    let mut seq = Vec::new();
    while let Some(id) = w.next(a) {
        seq.push(a.nodes[id as usize].val);
    }
    (seq, w.st)
}

// ---- the timing contestants: the same checksum, three ways ----

fn mix(acc: u64, v: i64) -> u64 {
    acc.wrapping_mul(31).wrapping_add(v as u64)
}

fn in_rec_sum(a: &Arena, id: u32, acc: &mut u64) {
    if id == NIL {
        return;
    }
    let n = a.nodes[id as usize];
    in_rec_sum(a, n.left, acc);
    *acc = mix(*acc, n.val);
    in_rec_sum(a, n.right, acc);
}

fn in_stack_sum(a: &Arena, root: u32, st: &mut Vec<u32>) -> u64 {
    let mut acc = 0;
    st.clear();
    let mut cur = root;
    while cur != NIL || !st.is_empty() {
        while cur != NIL {
            st.push(cur);
            cur = a.nodes[cur as usize].left;
        }
        cur = st.pop().expect("stack is not empty here");
        acc = mix(acc, a.nodes[cur as usize].val);
        cur = a.nodes[cur as usize].right;
    }
    acc
}

/// The walker's loop without the counters and the pause.
fn morris_sum(a: &mut Arena, root: u32) -> u64 {
    let mut acc = 0;
    let mut cur = root;
    while cur != NIL {
        let c = a.nodes[cur as usize];
        if c.left == NIL {
            acc = mix(acc, c.val);
            cur = c.right;
            continue;
        }
        let mut p = c.left;
        while a.nodes[p as usize].right != NIL && a.nodes[p as usize].right != cur {
            p = a.nodes[p as usize].right;
        }
        if a.nodes[p as usize].right == NIL {
            a.nodes[p as usize].right = cur;
            cur = c.left;
        } else {
            a.nodes[p as usize].right = NIL;
            acc = mix(acc, c.val);
            cur = c.right;
        }
    }
    acc
}

fn join_ints(a: &[i64]) -> String {
    let s: Vec<String> = a.iter().map(i64::to_string).collect();
    s.join(" ")
}

fn main() {
    // ---- (a) the walk, event by event, on the hand-built tree ----
    let mut a = Arena { nodes: Vec::new() };
    let root = a.hand_built();
    let pristine = a.nodes.clone();
    println!("the hand-built tree, sideways:");
    a.sideways(root, 0);

    let mut w = Walker::new(root, false, true);
    let mut seq = Vec::new();
    while let Some(id) = w.next(&mut a) {
        seq.push(a.nodes[id as usize].val);
    }
    println!(
        "\nMorris in-order, every event (thread p -> c points p's empty right link at c; cut removes it):"
    );
    for (i, e) in w.log.iter().flatten().enumerate() {
        println!("  {:>2}  {}", i + 1, e);
    }
    println!("in-order: {}", join_ints(&seq));
    let mut reference = Vec::new();
    a.in_rec(root, &mut reference);
    println!(
        "== recursive: {}, tree restored: {}, threads {} cuts {} visits {} moves {} pred steps {}, max live threads {}",
        seq == reference,
        a.nodes == pristine,
        w.st.threads,
        w.st.cuts,
        w.st.visits,
        w.st.moves,
        w.st.pred_steps,
        w.st.max_live
    );

    // ---- (b) the counts, on trees of 15, 1000 and 100000 nodes ----
    println!("\ncounts on randomSplit trees (one fresh PCG(1,2), drawn in this order):");
    println!(
        "{:>7} {:>4} {:>7} {:>7} {:>7} {:>8} {:>8} {:>8} {:>7} {:>9} {:>9}",
        "n",
        "h",
        "visits",
        "threads",
        "cuts",
        "left-ch",
        "moves",
        "pred",
        "hops/n",
        "max live",
        "stack hw"
    );
    let mut r = Pcg::new(1, 2);
    struct Row {
        n: usize,
        left: usize,
        spine: usize,
        hw: usize,
        st: Stats,
    }
    let mut rows = Vec::new();
    for n in [15usize, 1000, 100_000] {
        let mut t = Arena { nodes: Vec::new() };
        let root = t.random_split(n, &mut r);
        let mut next = 0;
        t.label_in_order(root, &mut next);
        let (_, st) = morris(&mut t, root, false);
        let (_, hw) = t.in_stack(root);
        let lc = t.left_children(root);
        rows.push(Row {
            n,
            left: lc,
            spine: t.spine_sum(root),
            hw,
            st,
        });
        println!(
            "{:>7} {:>4} {:>7} {:>7} {:>7} {:>8} {:>8} {:>8} {:>7.2} {:>9} {:>9}",
            n,
            t.height(root),
            st.visits,
            st.threads,
            st.cuts,
            lc,
            st.moves,
            st.pred_steps,
            (st.moves + st.pred_steps) as f64 / n as f64,
            st.max_live,
            hw
        );
    }
    println!(
        "(left-ch = nodes with a left child, counted by looking; hops/n = (moves + pred) / n, the pointer hops Morris makes per node;"
    );
    println!(
        " max live = most threads in the tree at once; stack hw = high-water of the explicit in-order stack)"
    );
    let all = |f: fn(&Row) -> bool| rows.iter().all(f);
    println!(
        "threads == cuts == nodes with a left child: {}",
        all(|x| x.st.threads == x.st.cuts && x.st.cuts == x.left)
    );
    println!("visits == n: {}", all(|x| x.st.visits == x.n));
    println!(
        "moves == n + threads: {}",
        all(|x| x.st.moves == x.n + x.st.threads)
    );
    println!(
        "pred steps == 2 * sum(right spine of the left child - 1): {}",
        all(|x| x.st.pred_steps == 2 * x.spine)
    );
    println!(
        "explicit stack high-water == max live threads + 1: {}",
        all(|x| x.hw == x.st.max_live + 1)
    );

    // ---- (c) 2000 random trees: same sequences, and the tree comes back ----
    let mut r = Pcg::new(1, 2);
    let (mut in_bad, mut pre_bad, mut not_restored, mut rel_bad) = (0, 0, 0, 0);
    let mut nodes = 0;
    for _ in 0..2000 {
        let n = r.int_n(80) as usize;
        let mut t = Arena { nodes: Vec::new() };
        let root = t.random_split(n, &mut r);
        let mut next = 0;
        t.label_in_order(root, &mut next);
        nodes += n;
        // the arena IS the tree (links are indices), so comparing the whole
        // vector with a copy made first checks structure and values at once
        let orig = t.nodes.clone();
        let (mut want_in, mut want_pre) = (Vec::new(), Vec::new());
        t.in_rec(root, &mut want_in);
        t.pre_rec(root, &mut want_pre);

        let (got_in, st) = morris(&mut t, root, false);
        let (_, hw) = t.in_stack(root);
        in_bad += usize::from(got_in != want_in);
        not_restored += usize::from(t.nodes != orig);
        if st.threads != st.cuts
            || st.cuts != t.left_children(root)
            || st.moves != n + st.threads
            || (n > 0 && hw != st.max_live + 1)
        {
            rel_bad += 1;
        }
        let (got_pre, _) = morris(&mut t, root, true);
        pre_bad += usize::from(got_pre != want_pre);
        not_restored += usize::from(t.nodes != orig);
    }
    println!("\n2000 random trees, sizes 0..79 ({nodes} nodes in total):");
    println!("  in-order mismatches vs recursive: {in_bad}, pre-order mismatches: {pre_bad}");
    println!("  trees not restored after a full walk (checked after both walks): {not_restored}");
    println!(
        "  trees where a relation of (b) failed (threads, cuts, moves, stack high-water): {rel_bad}"
    );

    // ---- (d) the hazard: stop early, and the tree is a cycle ----
    println!("\nstop after 3 visits, then run an ordinary recursive size on the same tree:");
    let mut a = Arena { nodes: Vec::new() };
    let root = a.hand_built();
    let mut w = Walker::new(root, false, false);
    for _ in 0..3 {
        let id = w.next(&mut a).expect("the tree has 9 nodes");
        println!("  visited {}", a.nodes[id as usize].val);
    }
    let extra = a.extra_links(root);
    println!(
        "  threads made {}, cut {}: {} still in the tree",
        w.st.threads, w.st.cuts, w.live
    );
    println!(
        "  links to a node already reached (found with a seen-set): {}",
        extra.join(", ")
    );
    let n = 9;
    let mut calls = 0;
    let done = a.size_capped(root, &mut calls, 10 * n);
    println!(
        "  size with a cap of 10n = {} calls: finished = {} after {} calls (a real tree needs {})",
        10 * n,
        done.is_some(),
        calls,
        2 * n + 1
    );
    if done.is_none() {
        println!("  did not finish: the tree contains a cycle");
    }
    println!("  now let the walk go on:");
    while let Some(id) = w.next(&mut a) {
        println!("  visited {}", a.nodes[id as usize].val);
    }
    let extra = a.extra_links(root);
    let mut calls = 0;
    let done = a.size_capped(root, &mut calls, 10 * n);
    println!(
        "  walk finished: threads {}, cuts {}; links to reached nodes: {}; size = {} (finished = {}, {} calls); restored: {}",
        w.st.threads,
        w.st.cuts,
        extra.len(),
        done.unwrap_or(0),
        done.is_some(),
        calls,
        a.nodes == pristine
    );

    // ---- (e) one timing table (the output differs from run to run) ----
    println!("\nns per node, best of 3 rounds (these numbers move from run to run):");
    let mut r = Pcg::new(1, 2);
    let mut random = Arena { nodes: Vec::new() };
    let random_root = random.random_split(1_000_000, &mut r);
    let mut next = 0;
    random.label_in_order(random_root, &mut next);
    let mut perfect = Arena { nodes: Vec::new() };
    let perfect_root = perfect.complete(0, 1_048_575);
    let mut trees = [
        ("randomSplit(1000000)", random, random_root, 1_000_000usize),
        ("complete(1048575)", perfect, perfect_root, 1_048_575usize),
    ];
    let mut best = [[Duration::MAX; 3]; 2];
    let mut st = Vec::with_capacity(128);
    for _round in 0..3 {
        for (i, (_, t, root, _)) in trees.iter_mut().enumerate() {
            let mut sums = [0u64; 3];
            let t0 = Instant::now();
            in_rec_sum(black_box(&*t), *root, &mut sums[0]);
            best[i][0] = best[i][0].min(t0.elapsed());
            let t0 = Instant::now();
            sums[1] = in_stack_sum(black_box(&*t), *root, &mut st);
            best[i][1] = best[i][1].min(t0.elapsed());
            let t0 = Instant::now();
            sums[2] = morris_sum(black_box(&mut *t), *root);
            best[i][2] = best[i][2].min(t0.elapsed());
            if sums[0] != sums[1] || sums[1] != sums[2] {
                println!("CHECKSUM MISMATCH");
            }
            black_box(sums);
        }
    }
    println!(
        "{:<22} {:>10} {:>12} {:>8} {:>15} {:>16}",
        "tree", "recursive", "explicit", "Morris", "Morris/explicit", "Morris/recursive"
    );
    for (i, (name, _, _, n)) in trees.iter().enumerate() {
        let ns = |d: Duration| d.as_nanos() as f64 / *n as f64;
        println!(
            "{:<22} {:>10.2} {:>12.2} {:>8.2} {:>14.2}x {:>15.2}x",
            name,
            ns(best[i][0]),
            ns(best[i][1]),
            ns(best[i][2]),
            ns(best[i][2]) / ns(best[i][1]),
            ns(best[i][2]) / ns(best[i][0])
        );
    }
    println!(
        "extra memory while walking: recursive = h stack frames, explicit = a Vec that peaks near h, Morris = two locals (cur and p)"
    );
}
```

**Rust sample output:**

```
the hand-built tree, sideways:
            9
        6
            8
    3
1
        5
            7
    2
        4

Morris in-order, every event (thread p -> c points p's empty right link at c; cut removes it):
   1  thread 5 -> 1
   2  thread 4 -> 2
   3  visit 4
   4  cut    4 -> 2
   5  visit 2
   6  thread 7 -> 5
   7  visit 7
   8  cut    7 -> 5
   9  visit 5
  10  cut    5 -> 1
  11  visit 1
  12  visit 3
  13  thread 8 -> 6
  14  visit 8
  15  cut    8 -> 6
  16  visit 6
  17  visit 9
in-order: 4 2 7 5 1 3 8 6 9
== recursive: true, tree restored: true, threads 4 cuts 4 visits 9 moves 13 pred steps 2, max live threads 2

counts on randomSplit trees (one fresh PCG(1,2), drawn in this order):
      n    h  visits threads    cuts  left-ch    moves     pred  hops/n  max live  stack hw
     15    6      15       8       8        8       23       10    2.20         4         5
   1000   20    1000     520     520      520     1520      952    2.47        16        17
 100000   41  100000   50029   50029    50029   150029    99932    2.50        26        27
(left-ch = nodes with a left child, counted by looking; hops/n = (moves + pred) / n, the pointer hops Morris makes per node;
 max live = most threads in the tree at once; stack hw = high-water of the explicit in-order stack)
threads == cuts == nodes with a left child: true
visits == n: true
moves == n + threads: true
pred steps == 2 * sum(right spine of the left child - 1): true
explicit stack high-water == max live threads + 1: true

2000 random trees, sizes 0..79 (79730 nodes in total):
  in-order mismatches vs recursive: 0, pre-order mismatches: 0
  trees not restored after a full walk (checked after both walks): 0
  trees where a relation of (b) failed (threads, cuts, moves, stack high-water): 0

stop after 3 visits, then run an ordinary recursive size on the same tree:
  visited 4
  visited 2
  visited 7
  threads made 3, cut 1: 2 still in the tree
  links to a node already reached (found with a seen-set): 5.Right -> 1, 7.Right -> 5
  size with a cap of 10n = 90 calls: finished = false after 91 calls (a real tree needs 19)
  did not finish: the tree contains a cycle
  now let the walk go on:
  visited 5
  visited 1
  visited 3
  visited 8
  visited 6
  visited 9
  walk finished: threads 4, cuts 4; links to reached nodes: 0; size = 9 (finished = true, 19 calls); restored: true

ns per node, best of 3 rounds (these numbers move from run to run):
tree                    recursive     explicit   Morris Morris/explicit Morris/recursive
randomSplit(1000000)         5.47         6.33     8.87           1.40x            1.62x
complete(1048575)            1.51         1.23     3.73           3.04x            2.47x
extra memory while walking: recursive = h stack frames, explicit = a Vec that peaks near h, Morris = two locals (cur and p)
```

> **Go vs Rust:** The output is identical to the Go run down to the timing table: the 17 events, every count, the cycle after 3 visits. A `Box` tree cannot be threaded: three attempts (compiled separately, not part of this file) were refused on rustc 1.98.1 — E0505 for moving the node into its predecessor's link while the walk still borrows it, E0308 for a `&mut Box<Node>` where a `Box<Node>` goes, E0502 for a thread field borrowed from the root — so the twin keeps its nodes in a `Vec<N>` whose `u32` links make a thread just another index (the arena of lesson 07 example 13). Morris was also the slowest of the three here, in all three runs: 1.4-1.5x the explicit stack on the random tree and 2.8-3.4x on the complete one (Go: 1.3x and 1.6-1.8x).

**Complexity:** Θ(n) time and **O(1) extra space**, but the tree is modified while the walk runs. The counts are exact: visits = n, threads = cuts = the nodes with a left child, moves = n + threads (150,029 moves for n = 100,000), and 2.5 pointer hops per node (moves plus predecessor steps) where the recursive and stack versions follow each link once. The live threads are the stack, stored in the tree: at most 26 at once for n = 100,000, against an explicit stack that peaked at 27. Stopped after 3 visits the walk leaves 2 threads and a cycle; finishing it restores the tree exactly. In the timing table (best of 3, run three times) Morris was the slowest of the three every time: about 1.3x the explicit stack on `randomSplit(1000000)` and 1.6-1.8x on `complete(1048575)`.

---

## 13. A tree you did not build: frames, depth and Drop

`🔴 hard` · *Stack and Drop depth*

Recursion depth is the height, so a tree handed to you can be n deep, and the questions are how many bytes one level costs and where the stack ends. The program measures **five recursive functions on a left-leaning and a right-leaning chain**, reading the bytes per level off the stack addresses themselves; **predicts** the depth at which a 16 MiB stack runs out, and checks the prediction from the safe side; and turns the deepest recursion (height and diameter) into a loop over explicit frames on the heap, whose limit is memory instead of the stack. Rust has a second recursion hiding in the destructor: dropping a `Box` tree nests once per level, so a tree you can build with a loop can still overflow when it goes out of scope. Go has no destructor to recurse, which the program shows beside the Rust count. Real overflows are never in a default run: they are behind a flag.

**Steps:**

1. Build left and right chains of 50,000 and 100,000 nodes; run `size`, the three walks and `diameter` through a measuring line, and read bytes per level from the slope between the two sizes; turn that into a predicted overflow depth.
2. Check the prediction from the safe side (a chain at 90% of the predicted depth, on a stack really limited to 16 MiB); count how deeply a drop nests on four shapes (Go: show that nothing does).
3. Run height and diameter over 1,000,000-node chains with an explicit stack of frames, compare with the recursion, and read the flags that trigger the real overflow.

```go
package main

import (
	"flag"
	"fmt"
	"math/bits"
	"math/rand/v2"
	"runtime"
	"runtime/debug"
	"unsafe"
)

// A tree you did not build can be n deep, and recursion depth is the height.
// Go grows a goroutine's stack on demand (to a limit), so the question is not
// "does it crash" but "how many bytes does one level cost, and where is the
// limit". This program measures the first, predicts the second, and checks the
// prediction from the safe side. The real overflow is opt-in:
//
//	go run . -overflow
//
// (a Go stack overflow is a fatal error, exit status 2 -- not recoverable).

type Node struct {
	Val         int
	Left, Right *Node
}

// ---- shapes, built without recursion where they can be deep ----

func leftChain(n int) *Node { // each node's only child is on the left; Val = depth
	var root *Node
	for i := n - 1; i >= 0; i-- {
		root = &Node{Val: i, Left: root}
	}
	return root
}

func rightChain(n int) *Node {
	var root *Node
	for i := n - 1; i >= 0; i-- {
		root = &Node{Val: i, Right: root}
	}
	return root
}

func zigzag(n int) *Node { // node i hangs off node i-1: Right if i is odd, Left if even
	var root *Node
	for i := n - 1; i >= 0; i-- {
		if (i+1)%2 == 1 {
			root = &Node{Val: i, Right: root}
		} else {
			root = &Node{Val: i, Left: root}
		}
	}
	return root
}

func complete(i, n int) *Node { // heap-shaped: node i has children 2i+1 and 2i+2 while < n
	if i >= n {
		return nil
	}
	return &Node{Val: i, Left: complete(2*i+1, n), Right: complete(2*i+2, n)}
}

// randomSplit: the left subtree gets a uniformly random share of the other n-1
// nodes; the draw happens BEFORE the recursion (the Rust twin draws in the same order).
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{Left: left, Right: right}
}

// ---- Section A: five recursive functions, each with the measuring line ----

// The stack is just memory: the address of a local says how far down it has
// grown. `low` remembers the deepest address any call reached, so the function
// that does the work also measures it. The measuring call is part of the frame.
//
//go:noinline
func stackAddress() uintptr {
	var marker byte
	return uintptr(unsafe.Pointer(&marker))
}

var low uintptr

func note() {
	if p := stackAddress(); p < low {
		low = p
	}
}

func size(n *Node) int {
	if n == nil {
		return 0
	}
	note()
	return 1 + size(n.Left) + size(n.Right)
}

func pre(n *Node, out *[]int) {
	if n != nil {
		note()
		*out = append(*out, n.Val)
		pre(n.Left, out)
		pre(n.Right, out)
	}
}

func inorder(n *Node, out *[]int) {
	if n != nil {
		note()
		inorder(n.Left, out)
		*out = append(*out, n.Val)
		inorder(n.Right, out)
	}
}

func post(n *Node, out *[]int) {
	if n != nil {
		note()
		post(n.Left, out)
		post(n.Right, out)
		*out = append(*out, n.Val)
	}
}

// diameter returns the height in nodes and records the best left+right in best (edges).
func diameter(n *Node, best *int) int {
	if n == nil {
		return 0
	}
	note()
	l := diameter(n.Left, best)
	r := diameter(n.Right, best)
	*best = max(*best, l+r)
	return 1 + max(l, r)
}

var scratch = make([]int, 0, 100_000)

type function struct {
	name string
	run  func(root *Node)
}

var functions = []function{
	{"size", func(t *Node) { size(t) }},
	{"pre-order walk", func(t *Node) { scratch = scratch[:0]; pre(t, &scratch) }},
	{"in-order walk", func(t *Node) { scratch = scratch[:0]; inorder(t, &scratch) }},
	{"post-order walk", func(t *Node) { scratch = scratch[:0]; post(t, &scratch) }},
	{"diameter", func(t *Node) { best := 0; diameter(t, &best) }},
}

// usedBy returns how many bytes of stack run() used. The first run lets the
// goroutine's stack grow to the size it needs (a stack that is copied to a bigger
// one moves, and addresses taken before the move mean nothing); the second run is
// the measurement. The garbage collector is off in main, because it may shrink
// an idle stack between the two runs.
func usedBy(root *Node, run func(*Node)) uintptr {
	run(root)
	base := stackAddress()
	low = base
	run(root)
	return base - low
}

// ---- Section C: the same recursion as a loop over explicit frames ----

// A literal copy of the call stack: a frame is (node, which step it is at, the
// left height it is holding).
type frame struct {
	n          *Node
	state      int // 0 = just entered, 1 = left done, 2 = right done
	leftHeight int
}

// heightDiameter returns the height in nodes, the diameter in edges, and the
// most frames held at once.
func heightDiameter(root *Node) (height, diam, highWater int) {
	if root == nil {
		return
	}
	stack := []frame{{n: root}}
	highWater = 1
	ret := 0
	for len(stack) > 0 {
		f := &stack[len(stack)-1]
		switch f.state {
		case 0:
			f.state = 1
			if f.n.Left != nil {
				stack = append(stack, frame{n: f.n.Left})
			} else {
				ret = 0
			}
		case 1:
			f.leftHeight = ret
			f.state = 2
			if f.n.Right != nil {
				stack = append(stack, frame{n: f.n.Right})
			} else {
				ret = 0
			}
		default:
			diam = max(diam, f.leftHeight+ret)
			ret = 1 + max(f.leftHeight, ret)
			stack = stack[:len(stack)-1]
		}
		highWater = max(highWater, len(stack))
	}
	return ret, diam, highWater
}

func heapMB() float64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

// A power of two: Go's stacks only come in these sizes, and the runtime checks the
// size of the stack it is about to grow INTO against the limit (runtime/stack.go,
// newstack: `newsize > maxstacksize`). So the biggest stack that fits under a limit
// is the largest power of two not above it.
func biggestStack(limit int) int { return 1 << (bits.Len(uint(limit)) - 1) }

const smallStack = 16 << 20

func main() {
	overflow := flag.Bool("overflow", false, "really overflow a 16 MiB stack (fatal error, exit status 2)")
	flag.Parse()
	if *overflow {
		debug.SetMaxStack(smallStack)
		fmt.Println("size over a 1000000-node left chain with the stack limited to 16 MiB...")
		fmt.Println("size =", size(leftChain(1_000_000)))
		return
	}

	// ---- Section A ----
	const n1, n2 = 50_000, 100_000
	fmt.Println("A. stack bytes per level of five recursive functions (the measuring call is part of the frame)")
	fmt.Printf("   slope between a %d- and a %d-node chain, on the main goroutine's stack:\n", n1, n2)
	oldGC := debug.SetGCPercent(-1)
	chains := [2][2]*Node{{leftChain(n1), leftChain(n2)}, {rightChain(n1), rightChain(n2)}}
	fmt.Printf("%-17s %11s %12s\n", "function", "left chain", "right chain")
	var perLevel [5][2]int
	for i, f := range functions {
		var cells [2]float64
		for lean, pair := range chains {
			u1, u2 := usedBy(pair[0], f.run), usedBy(pair[1], f.run)
			perLevel[i][lean] = int(u2-u1) / (n2 - n1)
			cells[lean] = float64(u2-u1) / float64(n2-n1)
		}
		fmt.Printf("%-17s %11.1f %12.1f\n", f.name, cells[0], cells[1])
	}
	chains = [2][2]*Node{}

	fmt.Printf("\npredicted depth at which a %d MiB stack runs out (stack bytes / bytes per level):\n", smallStack>>20)
	fmt.Printf("%-17s %11s %12s\n", "function", "left chain", "right chain")
	for i, f := range functions {
		fmt.Printf("%-17s %11s %12s\n", f.name, capacity(perLevel[i][0]), capacity(perLevel[i][1]))
	}
	// Check the prediction from the safe side: 90% of the predicted depth for the
	// hungriest function must survive with the limit really set to 16 MiB.
	worst := 0
	for i := range functions {
		if perLevel[i][0] > perLevel[worst][0] {
			worst = i
		}
	}
	safe := smallStack / perLevel[worst][0] * 9 / 10
	prev := debug.SetMaxStack(smallStack)
	got := usedBy(leftChain(safe), functions[worst].run)
	debug.SetMaxStack(prev)
	fmt.Printf("check: %s on a left chain of %d nodes (90%% of its prediction) with the limit set to %d MiB used %d of %d bytes -- no crash\n",
		functions[worst].name, safe, smallStack>>20, got, smallStack)
	fmt.Printf("by the rule in runtime/stack.go (a stack doubles; the new size may not pass the limit) the default limit of %d bytes allows a biggest stack of %d bytes, about %d levels of the hungriest function (read in the source, not run)\n",
		prev, biggestStack(prev), biggestStack(prev)/perLevel[worst][0])
	debug.SetGCPercent(oldGC)

	// ---- Section B ----
	fmt.Println("\nB. drop depth: Go has no destructor, so nothing nests when a tree goes out of scope")
	fmt.Printf("%-22s %7s %7s %12s\n", "shape", "n", "h", "destructor")
	shapes := []struct {
		name string
		root *Node
	}{
		{"chain (left)", leftChain(100_000)},
		{"zigzag", zigzag(100_000)},
		{"perfect(16)", complete(0, 65_535)},
		{"randomSplit(100000)", randomSplit(100_000, rand.New(rand.NewPCG(1, 2)))},
	}
	for _, s := range shapes {
		h, _, _ := heightDiameter(s.root)
		fmt.Printf("%-22s %7d %7d %12s\n", s.name, nodeCount(s.root), h, "none")
	}
	shapes = nil
	chain := leftChain(1_000_000)
	runtime.GC()
	before := heapMB()
	runtime.KeepAlive(chain)
	chain = nil
	runtime.GC()
	fmt.Printf("a 1000000-node chain: the collector frees it after `chain = nil` and runtime.GC(): HeapAlloc %.1f MB -> %.1f MB\n", before, heapMB())

	// ---- Section C ----
	fmt.Printf("\nC. the same recursion over explicit frames (%d bytes each, on the heap), 1,000,000-node chains\n", unsafe.Sizeof(frame{}))
	fmt.Printf("%-14s %9s %9s %9s %10s %12s %14s %10s\n",
		"shape", "n", "height", "diameter", "high-water", "heap bytes", "recursive", "stack used")
	for _, name := range []string{"left chain", "zigzag"} {
		n := 1_000_000
		root := leftChain(n)
		if name == "zigzag" {
			root = zigzag(n)
		}
		h, d, hw := heightDiameter(root)
		// the recursive version, run through the same measuring line
		var rh, rd int
		stackUsed := usedBy(root, func(t *Node) { rd = 0; rh = diameter(t, &rd) })
		verdict := "DIFFERENT"
		if rh == h && rd == d {
			verdict = "equal"
		}
		fmt.Printf("%-14s %9d %9d %9d %10d %12d %14s %10d\n",
			name, n, h, d, hw, hw*int(unsafe.Sizeof(frame{})), verdict, stackUsed)
	}

	// ---- Section D ----
	fmt.Println("\nD. the real overflow is behind a flag (a Go stack overflow is a fatal error, exit status 2):")
	fmt.Println("   not run: pass -overflow   (size over a 1000000-node left chain with the stack limited to 16 MiB)")
}

func capacity(bytesPerLevel int) string {
	if bytesPerLevel == 0 {
		return "no limit"
	}
	return fmt.Sprint(smallStack / bytesPerLevel)
}

func nodeCount(root *Node) int {
	n, work := 0, []*Node{root}
	for len(work) > 0 {
		x := work[len(work)-1]
		work = work[:len(work)-1]
		n++
		if x.Left != nil {
			work = append(work, x.Left)
		}
		if x.Right != nil {
			work = append(work, x.Right)
		}
	}
	return n
}
```

**Sample output:**

```
A. stack bytes per level of five recursive functions (the measuring call is part of the frame)
   slope between a 50000- and a 100000-node chain, on the main goroutine's stack:
function           left chain  right chain
size                     32.0         32.0
pre-order walk           96.0         96.0
in-order walk            96.0         96.0
post-order walk          96.0         96.0
diameter                 48.0         48.0

predicted depth at which a 16 MiB stack runs out (stack bytes / bytes per level):
function           left chain  right chain
size                   524288       524288
pre-order walk         174762       174762
in-order walk          174762       174762
post-order walk        174762       174762
diameter               349525       349525
check: pre-order walk on a left chain of 157285 nodes (90% of its prediction) with the limit set to 16 MiB used 15099392 of 16777216 bytes -- no crash
by the rule in runtime/stack.go (a stack doubles; the new size may not pass the limit) the default limit of 1000000000 bytes allows a biggest stack of 536870912 bytes, about 5592405 levels of the hungriest function (read in the source, not run)

B. drop depth: Go has no destructor, so nothing nests when a tree goes out of scope
shape                        n       h   destructor
chain (left)            100000  100000         none
zigzag                  100000  100000         none
perfect(16)              65535      16         none
randomSplit(100000)     100000      38         none
a 1000000-node chain: the collector frees it after `chain = nil` and runtime.GC(): HeapAlloc 24.3 MB -> 1.4 MB

C. the same recursion over explicit frames (24 bytes each, on the heap), 1,000,000-node chains
shape                  n    height  diameter high-water   heap bytes      recursive stack used
left chain       1000000   1000000    999999    1000000     24000000          equal   48000048
zigzag           1000000   1000000    999999    1000000     24000000          equal   48000048

D. the real overflow is behind a flag (a Go stack overflow is a fatal error, exit status 2):
   not run: pass -overflow   (size over a 1000000-node left chain with the stack limited to 16 MiB)
```

**Rust:**

```rust
use std::env;
use std::hint::black_box;
use std::mem::size_of;
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering::Relaxed};
use std::thread;

// A tree you did not build can be n deep, and in Rust depth costs you twice:
// once in every recursive function over the tree, and once more when the tree
// goes out of scope, because dropping a Box tree is itself a recursion.
//
// This program MEASURES both (bytes of stack per level, nesting of the drop),
// PREDICTS where a given stack runs out, and checks the prediction from the safe
// side. A real stack overflow aborts the process, and on macOS that writes a
// crash report on every run, so the real thing is opt-in:
//   ./ex13 --overflow N        recursion over an N-node chain on a 16 MiB thread
//   ./ex13 --drop-overflow N   dropping an N-node chain with the DEFAULT glue, same thread

// ---- the probe: a Node whose drop can be watched ----
//
// Fields drop in declaration order, so `_enter` (first) runs before the
// children are dropped and `_exit` (last) after. Both are zero-sized (the node
// stays 24 bytes). `_enter` also records how deep in the stack it runs. The
// counters are atomics, not thread_locals, because the drop runs on another thread.

static ITERATIVE: AtomicBool = AtomicBool::new(true); // which Drop the nodes use
static DEPTH: AtomicUsize = AtomicUsize::new(0); // drops currently nested
static MAX_DEPTH: AtomicUsize = AtomicUsize::new(0); // the deepest nesting seen
static DROPS: AtomicUsize = AtomicUsize::new(0); // nodes dropped
static LOWEST: AtomicUsize = AtomicUsize::new(usize::MAX); // deepest stack address seen

// The stack is just memory: the address of a local says how far down it has grown.
#[inline(never)]
fn stack_address() -> usize {
    let marker = 0u8;
    black_box(&marker) as *const u8 as usize
}

struct Enter;
impl Drop for Enter {
    fn drop(&mut self) {
        let depth = DEPTH.fetch_add(1, Relaxed) + 1;
        MAX_DEPTH.fetch_max(depth, Relaxed);
        DROPS.fetch_add(1, Relaxed);
        LOWEST.fetch_min(stack_address(), Relaxed);
    }
}

struct Exit;
impl Drop for Exit {
    fn drop(&mut self) {
        DEPTH.fetch_sub(1, Relaxed);
    }
}

struct Node {
    _enter: Enter,
    val: i64,
    left: Link,
    right: Link,
    _exit: Exit,
}
type Link = Option<Box<Node>>;

// One Node type, two ways to drop it. With ITERATIVE off this Drop does
// nothing, so what runs is the compiler's drop glue: drop the fields in order,
// each Box<Node> recursing into its own glue (what a plain struct gets).
// With it on, the children go onto a heap-allocated stack instead (lesson 05
// example 15, lesson 07). Go has no destructor to write.
impl Drop for Node {
    fn drop(&mut self) {
        if !ITERATIVE.load(Relaxed) {
            return;
        }
        let mut stack: Vec<Box<Node>> = Vec::new();
        stack.extend(self.left.take());
        stack.extend(self.right.take());
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
        }
    }
}

fn node(val: i64) -> Box<Node> {
    Box::new(Node {
        _enter: Enter,
        val,
        left: None,
        right: None,
        _exit: Exit,
    })
}

#[derive(Clone, Copy)]
enum Shape {
    Left,
    Right,
    Zigzag,
}

// Chains are built with a loop (a recursive builder would hit the limit it is
// about to measure): node i hangs off node i-1 on the left, on the right, or
// alternating (Right if i is odd, Left if even). Val = depth.
fn chain(n: usize, shape: Shape) -> Link {
    if n == 0 {
        return None;
    }
    let mut root = node(0);
    let mut cur = &mut root;
    for i in 1..n {
        let left = match shape {
            Shape::Left => true,
            Shape::Right => false,
            Shape::Zigzag => i % 2 == 0,
        };
        let child = node(i as i64);
        cur = if left {
            cur.left.insert(child)
        } else {
            cur.right.insert(child)
        };
    }
    Some(root)
}

// Heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
fn complete(i: usize, n: usize) -> Link {
    if i >= n {
        return None;
    }
    let mut b = node(i as i64);
    b.left = complete(2 * i + 1, n);
    b.right = complete(2 * i + 2, n);
    Some(b)
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

// The left subtree gets a uniformly random share of the other n-1 nodes; the
// draw happens BEFORE the recursion (the order the Go program uses).
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n as u64) as usize;
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    let mut b = node(0);
    b.left = left;
    b.right = right;
    Some(b)
}

// ---- Section A: five recursive functions, each with the measuring line ----
//
// `low` remembers the deepest stack address any call reached, so the function
// that does the work also measures it. The measuring call is part of the frame.

fn size(n: &Link, low: &mut usize) -> usize {
    match n {
        None => 0,
        Some(b) => {
            *low = (*low).min(stack_address());
            1 + size(&b.left, low) + size(&b.right, low)
        }
    }
}

fn pre(n: &Link, out: &mut Vec<i64>, low: &mut usize) {
    if let Some(b) = n {
        *low = (*low).min(stack_address());
        out.push(b.val);
        pre(&b.left, out, low);
        pre(&b.right, out, low);
    }
}

fn inorder(n: &Link, out: &mut Vec<i64>, low: &mut usize) {
    if let Some(b) = n {
        *low = (*low).min(stack_address());
        inorder(&b.left, out, low);
        out.push(b.val);
        inorder(&b.right, out, low);
    }
}

fn post(n: &Link, out: &mut Vec<i64>, low: &mut usize) {
    if let Some(b) = n {
        *low = (*low).min(stack_address());
        post(&b.left, out, low);
        post(&b.right, out, low);
        out.push(b.val);
    }
}

// Returns the height in nodes and records the best left+right in `best` (edges).
fn diameter(n: &Link, best: &mut usize, low: &mut usize) -> usize {
    match n {
        None => 0,
        Some(b) => {
            *low = (*low).min(stack_address());
            let l = diameter(&b.left, best, low);
            let r = diameter(&b.right, best, low);
            *best = (*best).max(l + r);
            1 + l.max(r)
        }
    }
}

type Walk = fn(&Link, &mut usize);
const FUNCTIONS: [(&str, Walk); 5] = [
    ("size", |t, low| {
        black_box(size(t, low));
    }),
    ("pre-order walk", |t, low| {
        let mut out = Vec::new();
        pre(t, &mut out, low);
        black_box(out);
    }),
    ("in-order walk", |t, low| {
        let mut out = Vec::new();
        inorder(t, &mut out, low);
        black_box(out);
    }),
    ("post-order walk", |t, low| {
        let mut out = Vec::new();
        post(t, &mut out, low);
        black_box(out);
    }),
    ("diameter", |t, low| {
        let mut best = 0;
        black_box(diameter(t, &mut best, low));
        black_box(best);
    }),
];

// Runs `walk` on a fresh thread with `stack` bytes of stack and returns how many
// of them it used. A Rust thread's stack is fixed when the thread is created
// (Go's grow on demand). Section A's slope is (used(100000) - used(50000)) / 50000.
fn used_by(stack: usize, tree: &Link, walk: Walk) -> usize {
    thread::scope(|s| {
        thread::Builder::new()
            .stack_size(stack)
            .spawn_scoped(s, || {
                let base = stack_address();
                let mut low = base;
                walk(tree, &mut low);
                base - low
            })
            .expect("spawn")
            .join()
            .expect("join")
    })
}

// ---- Section B: dropping on a big stack, counting the nesting ----

// Drops `tree` on a thread with `stack` bytes, with the default glue or the
// iterative Drop. Returns (deepest nesting of drops, nodes dropped, bytes of
// stack used).
fn drop_probe(tree: Link, iterative: bool, stack: usize) -> (usize, usize, usize) {
    ITERATIVE.store(iterative, Relaxed);
    DEPTH.store(0, Relaxed);
    MAX_DEPTH.store(0, Relaxed);
    DROPS.store(0, Relaxed);
    LOWEST.store(usize::MAX, Relaxed);
    let used = thread::Builder::new()
        .stack_size(stack)
        .spawn(move || {
            let base = stack_address();
            drop(black_box(tree));
            base.saturating_sub(LOWEST.load(Relaxed))
        })
        .expect("spawn")
        .join()
        .expect("join");
    ITERATIVE.store(true, Relaxed);
    (MAX_DEPTH.load(Relaxed), DROPS.load(Relaxed), used)
}

// ---- Section C: the same recursion as a loop over explicit frames ----

// A literal copy of the call stack: a frame is (node, which step it is at, the
// left height it is holding). Returns (height in nodes, diameter in edges,
// the most frames held at once).
struct Frame<'a> {
    node: &'a Node,
    state: u8, // 0 = just entered, 1 = left done, 2 = right done
    left_height: usize,
}

fn height_diameter(root: &Link) -> (usize, usize, usize) {
    let Some(root) = root.as_deref() else {
        return (0, 0, 0);
    };
    let first = Frame {
        node: root,
        state: 0,
        left_height: 0,
    };
    let (mut stack, mut ret, mut best, mut high_water) = (vec![first], 0usize, 0usize, 1usize);
    while let Some(f) = stack.last_mut() {
        let n = f.node;
        match f.state {
            0 => {
                f.state = 1;
                match n.left.as_deref() {
                    Some(l) => stack.push(Frame {
                        node: l,
                        state: 0,
                        left_height: 0,
                    }),
                    None => ret = 0,
                }
            }
            1 => {
                f.left_height = ret;
                f.state = 2;
                match n.right.as_deref() {
                    Some(r) => stack.push(Frame {
                        node: r,
                        state: 0,
                        left_height: 0,
                    }),
                    None => ret = 0,
                }
            }
            _ => {
                best = best.max(f.left_height + ret);
                ret = 1 + f.left_height.max(ret);
                stack.pop();
            }
        }
        high_water = high_water.max(stack.len());
    }
    (ret, best, high_water)
}

// ---- Section D: the real overflows, opt-in ----

const SMALL_STACK: usize = 16 << 20; // stands in for "a server under memory pressure"
const BIG_STACK: usize = 256 << 20; // the thread the deep recursions are given

fn overflow(n: usize) {
    println!("recursion over a {n}-node left chain on a 16 MiB thread...");
    let t = chain(n, Shape::Left);
    let used = used_by(SMALL_STACK, &t, FUNCTIONS[1].1);
    println!("survived, using {used} bytes of stack");
}

fn drop_overflow(n: usize) {
    println!("dropping a {n}-node left chain with the default glue on a 16 MiB thread...");
    let t = chain(n, Shape::Left);
    let (nesting, drops, used) = drop_probe(t, false, SMALL_STACK);
    println!("survived: nesting {nesting}, {drops} nodes, {used} bytes of stack");
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let count = |default| {
        args.get(2)
            .map_or(default, |s| s.parse().expect("a node count"))
    };
    match args.get(1).map(String::as_str) {
        Some("--overflow") => return overflow(count(1_000_000)),
        Some("--drop-overflow") => return drop_overflow(count(1_000_000)),
        _ => {}
    }

    // ---- Section A ----
    const N1: usize = 50_000;
    const N2: usize = 100_000;
    println!(
        "A. stack bytes per level of five recursive functions (the measuring call is part of the frame)"
    );
    println!(
        "   slope between a {N1}- and a {N2}-node chain, run on a thread with a 256 MiB stack:"
    );
    let chains = [
        (chain(N1, Shape::Left), chain(N2, Shape::Left)),
        (chain(N1, Shape::Right), chain(N2, Shape::Right)),
    ];
    println!(
        "{:<17} {:>11} {:>12}",
        "function", "left chain", "right chain"
    );
    let mut per_level = [[0usize; 2]; 5]; // [function][lean], whole bytes
    for (i, (name, walk)) in FUNCTIONS.iter().enumerate() {
        let mut cells = [0.0; 2];
        for (lean, (small, big)) in chains.iter().enumerate() {
            let (u1, u2) = (
                used_by(BIG_STACK, small, *walk),
                used_by(BIG_STACK, big, *walk),
            );
            per_level[i][lean] = (u2 - u1) / (N2 - N1);
            cells[lean] = (u2 - u1) as f64 / (N2 - N1) as f64;
        }
        println!("{:<17} {:>11.1} {:>12.1}", name, cells[0], cells[1]);
    }
    drop(chains);

    println!(
        "\npredicted depth at which a {} MiB stack runs out (stack bytes / bytes per level):",
        SMALL_STACK >> 20
    );
    println!(
        "{:<17} {:>11} {:>12}",
        "function", "left chain", "right chain"
    );
    for (i, (name, _)) in FUNCTIONS.iter().enumerate() {
        let cell = |b: usize| {
            SMALL_STACK
                .checked_div(b)
                .map_or("no limit".to_string(), |d| d.to_string())
        };
        println!(
            "{:<17} {:>11} {:>12}",
            name,
            cell(per_level[i][0]),
            cell(per_level[i][1])
        );
    }
    // Check the prediction from the safe side: 90% of the predicted depth for
    // the hungriest function must survive on a real 16 MiB thread.
    let worst = (0..5)
        .max_by_key(|&i| per_level[i][0])
        .expect("five functions");
    let safe = SMALL_STACK / per_level[worst][0] * 9 / 10;
    let t = chain(safe, Shape::Left);
    let used = used_by(SMALL_STACK, &t, FUNCTIONS[worst].1);
    drop(t);
    println!(
        "check: {} on a left chain of {safe} nodes (90% of its prediction) on a real {} MiB thread used {used} of {} bytes -- no crash",
        FUNCTIONS[worst].0,
        SMALL_STACK >> 20,
        SMALL_STACK
    );

    // ---- Section B ----
    println!(
        "\nB. drop depth: a probe node (size_of::<Node>() = {} bytes) counts how deeply its drop nests",
        size_of::<Node>()
    );
    println!(
        "{:<22} {:>7} {:>7} {:>16} {:>18} {:>11}",
        "shape", "n", "h", "default nesting", "iterative nesting", "node drops"
    );
    type Build = Box<dyn Fn() -> Link>;
    let shapes: [(&str, Build); 4] = [
        ("chain (left)", Box::new(|| chain(100_000, Shape::Left))),
        ("zigzag", Box::new(|| chain(100_000, Shape::Zigzag))),
        ("perfect(16)", Box::new(|| complete(0, 65_535))),
        (
            "randomSplit(100000)",
            Box::new(|| random_split(100_000, &mut Pcg::new(1, 2))),
        ),
    ];
    let (mut nesting_is_height, mut iterative_is_one, mut each_once) = (true, true, true);
    for (name, build) in &shapes {
        let t = build();
        let (h, _, _) = height_diameter(&t);
        let (nesting, drops, _) = drop_probe(t, false, BIG_STACK);
        let (nesting_it, drops_it, _) = drop_probe(build(), true, BIG_STACK);
        let n = drops;
        nesting_is_height &= nesting == h;
        iterative_is_one &= nesting_it == 1;
        each_once &= drops_it == n;
        println!("{name:<22} {n:>7} {h:>7} {nesting:>16} {nesting_it:>18} {drops:>11}");
    }
    println!("default nesting == height: {nesting_is_height}");
    println!("iterative nesting == 1: {iterative_is_one}");
    println!("both drops freed every node exactly once: {each_once}");
    let (_, _, d1) = drop_probe(chain(N1, Shape::Left), false, BIG_STACK);
    let (_, _, d2) = drop_probe(chain(N2, Shape::Left), false, BIG_STACK);
    let drop_per_level = (d2 - d1) / (N2 - N1);
    println!(
        "default drop glue: {:.1} bytes of stack per level (slope between {N1} and {N2} nodes)",
        (d2 - d1) as f64 / (N2 - N1) as f64
    );
    println!(
        "  -> predicted overflow at {} levels on a 16 MiB thread, {} on a 2 MiB thread (the default for a spawned thread per the std::thread docs; not tested here)",
        SMALL_STACK / drop_per_level,
        (2 << 20) / drop_per_level
    );
    let safe = SMALL_STACK / drop_per_level * 9 / 10;
    let (nesting, drops, used) = drop_probe(chain(safe, Shape::Left), false, SMALL_STACK);
    println!(
        "check: a left chain of {safe} nodes (90% of that) dropped by the default glue on a real 16 MiB thread: nesting {nesting}, {drops} nodes, {used} bytes of stack -- no crash"
    );

    // ---- Section C ----
    println!(
        "\nC. the same recursion over explicit frames ({} bytes each, on the heap), 1,000,000-node chains",
        size_of::<Frame>()
    );
    println!(
        "{:<14} {:>9} {:>9} {:>9} {:>10} {:>12} {:>14} {:>10}",
        "shape", "n", "height", "diameter", "high-water", "heap bytes", "recursive", "stack used"
    );
    for (name, shape) in [("left chain", Shape::Left), ("zigzag", Shape::Zigzag)] {
        let n = 1_000_000;
        let t = chain(n, shape);
        let (h, d, hw) = height_diameter(&t);
        // the recursive versions need the thread to be big
        let (rh, rd, used) = thread::scope(|s| {
            thread::Builder::new()
                .stack_size(BIG_STACK)
                .spawn_scoped(s, || {
                    let base = stack_address();
                    let (mut low, mut best) = (base, 0);
                    let h = diameter(&t, &mut best, &mut low);
                    (h, best, base - low)
                })
                .expect("spawn")
                .join()
                .expect("join")
        });
        println!(
            "{:<14} {:>9} {:>9} {:>9} {:>10} {:>12} {:>14} {:>10}",
            name,
            n,
            h,
            d,
            hw,
            hw * size_of::<Frame>(),
            if (rh, rd) == (h, d) {
                "equal"
            } else {
                "DIFFERENT"
            },
            used
        );
    }

    // ---- Section D ----
    println!(
        "\nD. the real overflows are behind flags (a Rust stack overflow aborts the process):"
    );
    println!(
        "   not run: pass --overflow N        (recursion over an N-node left chain on a 16 MiB thread)"
    );
    println!(
        "   not run: pass --drop-overflow N   (the default drop glue over an N-node left chain, same thread)"
    );
}
```

**Run it (Rust):**

```bash
rustc -O ex13.rs -o ex13 && ./ex13        # default: measures and predicts, never crashes
./ex13 --overflow 1000000        # opt-in: really overflows a 16 MiB thread and aborts the process
./ex13 --drop-overflow 1000000   # opt-in: the default drop glue overflows the same stack
```

**Rust sample output:**

```
A. stack bytes per level of five recursive functions (the measuring call is part of the frame)
   slope between a 50000- and a 100000-node chain, run on a thread with a 256 MiB stack:
function           left chain  right chain
size                     48.0          0.0
pre-order walk           64.0          0.0
in-order walk            64.0          0.0
post-order walk          48.0         48.0
diameter                 48.0         48.0

predicted depth at which a 16 MiB stack runs out (stack bytes / bytes per level):
function           left chain  right chain
size                   349525     no limit
pre-order walk         262144     no limit
in-order walk          262144     no limit
post-order walk        349525       349525
diameter               349525       349525
check: in-order walk on a left chain of 235929 nodes (90% of its prediction) on a real 16 MiB thread used 15099552 of 16777216 bytes -- no crash

B. drop depth: a probe node (size_of::<Node>() = 24 bytes) counts how deeply its drop nests
shape                        n       h  default nesting  iterative nesting  node drops
chain (left)            100000  100000           100000                  1      100000
zigzag                  100000  100000           100000                  1      100000
perfect(16)              65535      16               16                  1       65535
randomSplit(100000)     100000      38               38                  1      100000
default nesting == height: true
iterative nesting == 1: true
both drops freed every node exactly once: true
default drop glue: 48.0 bytes of stack per level (slope between 50000 and 100000 nodes)
  -> predicted overflow at 349525 levels on a 16 MiB thread, 43690 on a 2 MiB thread (the default for a spawned thread per the std::thread docs; not tested here)
check: a left chain of 314572 nodes (90% of that) dropped by the default glue on a real 16 MiB thread: nesting 314572, 314572 nodes, 15099456 bytes of stack -- no crash

C. the same recursion over explicit frames (24 bytes each, on the heap), 1,000,000-node chains
shape                  n    height  diameter high-water   heap bytes      recursive stack used
left chain       1000000   1000000    999999    1000000     24000000          equal   48000000
zigzag           1000000   1000000    999999    1000000     24000000          equal   48000000

D. the real overflows are behind flags (a Rust stack overflow aborts the process):
   not run: pass --overflow N        (recursion over an N-node left chain on a 16 MiB thread)
   not run: pass --drop-overflow N   (the default drop glue over an N-node left chain, same thread)
```

> **Go vs Rust:** The two languages disagree here. Go's frames are 32 / 96 / 96 / 96 / 48 B and identical on both leans; Rust's are 48 / 64 / 64 / 48 / 48 B on the left chain and **0** / 0 / 0 / 48 / 48 B on the right, because `-O` turned one of the two self-calls into a jump in `size`, `pre` and the in-order walk but not in `post` or `diameter` (counted in the assembly from `rustc -O --emit asm`, not by this program: one `bl` to itself in the first three, two in the others). The rule is wider than "the last thing a function does is call itself": `size` ends in an addition, and LLVM folded that into the loop too. Dropping is the recursion only Rust has: the default drop nests as deep as the tree is high (100,000 on a chain) at 48 B per level, so a 16 MiB thread should overflow near 349,525 levels; run once, `--overflow 1000000` and `--drop-overflow 1000000` both aborted (exit status 134, "has overflowed its stack", and macOS wrote a crash report), while `--overflow 260000` and `--drop-overflow 345000`, about 99% of the predictions, ran to the end.

**Complexity:** recursion depth = height, and the stack bytes per level depend on the function: 32 B for `size`, 96 B for the three walks, 48 B for `diameter`, the same on both leans (Go does not turn the last call into a loop; lesson 05 example 6). At 16 MiB that predicts 174,762 levels for the walks, and a chain at 90% of it ran fine. Go's default limit of 10^9 bytes is a ceiling on the size of the stack, and stacks only come in powers of two, so the biggest one is 512 MiB (read in `runtime/stack.go`, not run). Dropping a tree costs Go nothing nested: the collector took a 1,000,000-node chain from 24 MB of heap to 1.4 MB. The explicit-frame version reached the same height and diameter on two 1,000,000-node chains from a stack of 24-byte frames that peaked at n (24 MB, on the heap) where the recursion used 48 MB of thread stack. Run once with `-overflow`: fatal error, exit status 2, `goroutine stack exceeds 16777216-byte limit`, the stack range printed 16 MiB wide.

---

## 14. One tree, three owners: Box, Rc and an arena

`🔴 hard` · *Ownership*

The same 10,000-node tree held three ways, and what each costs in allocations, bytes and rules. Go uses plain pointers, pointers with a parent link, and an index arena; Rust uses `Box<Node>` (one owner, checked by the compiler, no parent links possible), `Rc<RefCell<Node>>` with `Weak` parents (shared owners, checked while the program runs) and the same index arena. All of them are built from one recorded shape, so sizes, heights and checksums must agree, and then the program counts: allocations and bytes to build, allocations to invert in place (none), and the things only one side has: Rust's reference counts, its leaked reference cycle and its runtime borrow check, against Go's collector, which reclaims the same cycle without being asked. Counts only; lesson 04 example 14 did the layout timings.

**Steps:**

1. Replay one recorded `randomSplit(10000)` shape into each representation and print allocations, bytes and bytes per node; then run size, height, the in-order checksum and an in-place invert on all of them and check they agree.
2. Walk up the parent links (or an arena parent table) and check the sum of depths against the top-down sum.
3. Read the ownership section: Go's weak pointer proving that a cycle is collected; in Rust the strong and weak counts, the tree freed under `Weak` parents, the 0-of-n freed under `Rc` parents, the `try_borrow_mut` error, and whether dropping an `Rc` chain nests.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"unsafe"
	"weak"
)

// One tree, three ways to hold it: plain pointers, pointers with a parent
// link, and an arena (a slice of nodes that point at each other by index).
// All three are built from the SAME shape: `splits` records, for every node in
// pre-order, how many of the other nodes go to its left subtree (the draws of
// randomSplit, replayed without the random generator so that every build is
// identical). Values are the in-order ranks 0..n-1.

type Node struct {
	Val         int
	Left, Right *Node
}

type PNode struct {
	Val                 int
	Left, Right, Parent *PNode
}

const nilIdx = -1

type ANode struct {
	Val         int32
	Left, Right int32 // indices into the arena; nilIdx is the empty tree
}

func drawSplits(n int, r *rand.Rand, out *[]int) {
	if n == 0 {
		return
	}
	k := r.IntN(n) // the draw comes BEFORE the recursion, as in randomSplit
	*out = append(*out, k)
	drawSplits(k, r, out)
	drawSplits(n-1-k, r, out)
}

// ---- builders: one per representation ----

func buildPtr(splits []int, pos *int, n int, next *int) *Node {
	if n == 0 {
		return nil
	}
	k := splits[*pos]
	*pos++
	node := &Node{}
	node.Left = buildPtr(splits, pos, k, next)
	node.Val = *next
	*next++
	node.Right = buildPtr(splits, pos, n-1-k, next)
	return node
}

func buildP(splits []int, pos *int, n int, next *int, parent *PNode) *PNode {
	if n == 0 {
		return nil
	}
	k := splits[*pos]
	*pos++
	node := &PNode{Parent: parent}
	node.Left = buildP(splits, pos, k, next, node)
	node.Val = *next
	*next++
	node.Right = buildP(splits, pos, n-1-k, next, node)
	return node
}

type Arena struct{ nodes []ANode }

func (a *Arena) build(splits []int, pos *int, n int, next *int) int32 {
	if n == 0 {
		return nilIdx
	}
	k := splits[*pos]
	*pos++
	id := int32(len(a.nodes))
	a.nodes = append(a.nodes, ANode{})
	left := a.build(splits, pos, k, next)
	a.nodes[id].Left = left
	a.nodes[id].Val = int32(*next)
	*next++
	a.nodes[id].Right = a.build(splits, pos, n-1-k, next)
	return id
}

// ---- the operations, once per representation ----

func mix(h uint64, v int) uint64 { return h*31 + uint64(v) }

func (t *Node) size() int {
	if t == nil {
		return 0
	}
	return 1 + t.Left.size() + t.Right.size()
}
func (t *Node) height() int {
	if t == nil {
		return 0
	}
	return 1 + max(t.Left.height(), t.Right.height())
}
func (t *Node) inorder(h *uint64) { // left, node, right
	if t != nil {
		t.Left.inorder(h)
		*h = mix(*h, t.Val)
		t.Right.inorder(h)
	}
}
func (t *Node) reverse(h *uint64) { // right, node, left
	if t != nil {
		t.Right.reverse(h)
		*h = mix(*h, t.Val)
		t.Left.reverse(h)
	}
}
func (t *Node) invert() {
	if t != nil {
		t.Left, t.Right = t.Right, t.Left
		t.Left.invert()
		t.Right.invert()
	}
}

func (t *PNode) size() int {
	if t == nil {
		return 0
	}
	return 1 + t.Left.size() + t.Right.size()
}
func (t *PNode) height() int {
	if t == nil {
		return 0
	}
	return 1 + max(t.Left.height(), t.Right.height())
}
func (t *PNode) inorder(h *uint64) {
	if t != nil {
		t.Left.inorder(h)
		*h = mix(*h, t.Val)
		t.Right.inorder(h)
	}
}
func (t *PNode) reverse(h *uint64) {
	if t != nil {
		t.Right.reverse(h)
		*h = mix(*h, t.Val)
		t.Left.reverse(h)
	}
}
func (t *PNode) invert() {
	if t != nil {
		t.Left, t.Right = t.Right, t.Left
		t.Left.invert()
		t.Right.invert()
	}
}

func (a *Arena) size(id int32) int {
	if id == nilIdx {
		return 0
	}
	return 1 + a.size(a.nodes[id].Left) + a.size(a.nodes[id].Right)
}
func (a *Arena) height(id int32) int {
	if id == nilIdx {
		return 0
	}
	return 1 + max(a.height(a.nodes[id].Left), a.height(a.nodes[id].Right))
}
func (a *Arena) inorder(id int32, h *uint64) {
	if id != nilIdx {
		a.inorder(a.nodes[id].Left, h)
		*h = mix(*h, int(a.nodes[id].Val))
		a.inorder(a.nodes[id].Right, h)
	}
}
func (a *Arena) reverse(id int32, h *uint64) {
	if id != nilIdx {
		a.reverse(a.nodes[id].Right, h)
		*h = mix(*h, int(a.nodes[id].Val))
		a.reverse(a.nodes[id].Left, h)
	}
}
func (a *Arena) invert(id int32) {
	if id != nilIdx {
		n := &a.nodes[id]
		n.Left, n.Right = n.Right, n.Left
		a.invert(n.Left)
		a.invert(n.Right)
	}
}

// ---- counting allocations: Mallocs and TotalAlloc only ever go up, so a
// before/after difference is exact (the garbage collector cannot change it) ----

func measure(f func()) (allocs, bytes uint64) {
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	f()
	runtime.ReadMemStats(&b)
	return b.Mallocs - a.Mallocs, b.TotalAlloc - a.TotalAlloc
}

// ops is one representation behind a common face, so the table is a loop.
type ops struct {
	name             string
	size, height     func() int
	inorder, reverse func() uint64
	invert           func()
}

// cycleDemo builds a parent-linked tree and returns weak pointers to its root
// and to a leaf; nothing else refers to the tree when it returns.
func cycleDemo(splits []int, n int) (weak.Pointer[PNode], weak.Pointer[PNode]) {
	pos, next := 0, 0
	root := buildP(splits, &pos, n, &next, nil)
	leaf := root
	for leaf.Left != nil || leaf.Right != nil {
		if leaf.Left != nil {
			leaf = leaf.Left
		} else {
			leaf = leaf.Right
		}
	}
	return weak.Make(root), weak.Make(leaf)
}

func main() {
	const n = 10000
	var splits []int
	drawSplits(n, rand.New(rand.NewPCG(1, 2)), &splits)

	fmt.Printf("the tree: randomSplit(%d), values = in-order ranks; sizeof: Node %d, PNode %d, ANode %d bytes\n",
		n, unsafe.Sizeof(Node{}), unsafe.Sizeof(PNode{}), unsafe.Sizeof(ANode{}))

	// ---- (a) building the same tree three ways ----
	var root *Node
	var proot *PNode
	var grown, reserved Arena
	type built struct {
		name          string
		nodeBytes     uintptr
		allocs, bytes uint64
	}
	var rows []built
	add := func(name string, nodeBytes uintptr, build func()) {
		allocs, bytes := measure(build)
		rows = append(rows, built{name, nodeBytes, allocs, bytes})
	}
	add("*Node (Left, Right)", unsafe.Sizeof(Node{}), func() { pos, next := 0, 0; root = buildPtr(splits, &pos, n, &next) })
	add("*PNode (+ Parent)", unsafe.Sizeof(PNode{}), func() { pos, next := 0, 0; proot = buildP(splits, &pos, n, &next, nil) })
	add("arena, grown by append", unsafe.Sizeof(ANode{}), func() { pos, next := 0, 0; grown.build(splits, &pos, n, &next) })
	add("arena, capacity reserved", unsafe.Sizeof(ANode{}), func() {
		reserved.nodes = make([]ANode, 0, n)
		pos, next := 0, 0
		reserved.build(splits, &pos, n, &next)
	})
	fmt.Println("\n(a) building the same tree: allocations and bytes the allocator handed out")
	fmt.Printf("%-26s %7s %8s %12s %9s\n", "representation", "node B", "allocs", "bytes", "B / node")
	for _, r := range rows {
		fmt.Printf("%-26s %7d %8d %12d %9.1f\n", r.name, r.nodeBytes, r.allocs, r.bytes, float64(r.bytes)/n)
	}

	// ---- (b) the same operations on each ----
	reps := []ops{
		{"*Node", root.size, root.height,
			func() uint64 { var h uint64; root.inorder(&h); return h },
			func() uint64 { var h uint64; root.reverse(&h); return h }, root.invert},
		{"*PNode", proot.size, proot.height,
			func() uint64 { var h uint64; proot.inorder(&h); return h },
			func() uint64 { var h uint64; proot.reverse(&h); return h }, proot.invert},
		{"arena (indices)", func() int { return reserved.size(0) }, func() int { return reserved.height(0) },
			func() uint64 { var h uint64; reserved.inorder(0, &h); return h },
			func() uint64 { var h uint64; reserved.reverse(0, &h); return h },
			func() { reserved.invert(0) }},
	}
	fmt.Println("\n(b) the same operations on each (checksum: h = h*31 + value, wrapping)")
	fmt.Printf("%-16s %6s %7s %18s %18s %14s %24s\n",
		"representation", "size", "height", "in-order checksum", "reverse checksum", "invert allocs", "after invert == reverse")
	var sums []uint64
	for _, r := range reps {
		in, rev := r.inorder(), r.reverse()
		sums = append(sums, in)
		allocs, _ := measure(r.invert)
		fmt.Printf("%-16s %6d %7d %18x %18x %14d %24v\n", r.name, r.size(), r.height(), in, rev, allocs, r.inorder() == rev)
		r.invert() // put it back
	}
	fmt.Printf("all three agree on the checksum: %v; the grown and the reserved arena hold equal nodes: %v\n",
		sums[0] == sums[1] && sums[1] == sums[2], slices.Equal(grown.nodes, reserved.nodes))

	// ---- (c) depth by walking up the parent links ----
	// Sum over all nodes of their depth, found three ways: top-down, by walking
	// up PNode.Parent, and by walking up an arena parent table.
	var topDown func(t *Node, d int) int
	topDown = func(t *Node, d int) int {
		if t == nil {
			return 0
		}
		return d + topDown(t.Left, d+1) + topDown(t.Right, d+1)
	}
	steps, work := 0, []*PNode{proot}
	for len(work) > 0 {
		t := work[len(work)-1]
		work = work[:len(work)-1]
		for p := t.Parent; p != nil; p = p.Parent {
			steps++
		}
		for _, c := range []*PNode{t.Left, t.Right} {
			if c != nil {
				work = append(work, c)
			}
		}
	}
	parent := make([]int32, len(reserved.nodes))
	parent[0] = nilIdx
	for i, nd := range reserved.nodes {
		if nd.Left != nilIdx {
			parent[nd.Left] = int32(i)
		}
		if nd.Right != nilIdx {
			parent[nd.Right] = int32(i)
		}
	}
	asteps := 0
	for i := range reserved.nodes {
		for p := parent[i]; p != nilIdx; p = parent[p] {
			asteps++
		}
	}
	fmt.Println("\n(c) depth of every node by walking up (steps = sum of depths)")
	fmt.Printf("top-down sum of depths: %d\n", topDown(root, 0))
	fmt.Printf("*PNode.Parent walk: %d steps (a field read each); arena parent table: %d steps (an index each, table = %d bytes)\n",
		steps, asteps, len(parent)*4)

	// ---- (d) Go has no counterpart to what Rust does at runtime here ----
	// A parent link makes every child point back at its parent: a cycle. Go's
	// collector does not care. A weak pointer becomes nil once the last strong
	// reference is gone, so it shows the whole cycle was collected.
	wRoot, wLeaf := cycleDemo(splits, n)
	runtime.GC()
	fmt.Println("\n(d) cycles and aliasing")
	fmt.Printf("a parent-linked tree after its last reference is gone and runtime.GC(): root collected = %v, a leaf collected = %v\n",
		wRoot.Value() == nil, wLeaf.Value() == nil)
	fmt.Println("no reference counts, no borrow flag: two *Node can point at one node and both may write it; Go's race detector, not the language, finds the unsynchronised cases")
}
```

**Sample output:**

```
the tree: randomSplit(10000), values = in-order ranks; sizeof: Node 24, PNode 32, ANode 12 bytes

(a) building the same tree: allocations and bytes the allocator handed out
representation              node B   allocs        bytes  B / node
*Node (Left, Right)             24    10000       240000      24.0
*PNode (+ Parent)               32    10000       320000      32.0
arena, grown by append          12       18       489464      48.9
arena, capacity reserved        12        1       122880      12.3

(b) the same operations on each (checksum: h = h*31 + value, wrapping)
representation     size  height  in-order checksum   reverse checksum  invert allocs  after invert == reverse
*Node             10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
*PNode            10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
arena (indices)   10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
all three agree on the checksum: true; the grown and the reserved arena hold equal nodes: true

(c) depth of every node by walking up (steps = sum of depths)
top-down sum of depths: 154829
*PNode.Parent walk: 154829 steps (a field read each); arena parent table: 154829 steps (an index each, table = 40000 bytes)

(d) cycles and aliasing
a parent-linked tree after its last reference is gone and runtime.GC(): root collected = true, a leaf collected = true
no reference counts, no borrow flag: two *Node can point at one node and both may write it; Go's race detector, not the language, finds the unsynchronised cases
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::RefCell;
use std::mem::size_of;
use std::rc::{Rc, Weak};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// One tree, three ways to own it. Go holds a tree with pointers and lets the
// garbage collector sort out who keeps what alive. Rust makes ownership part of
// the types:
//   Box<Node>             one owner per node: compile-time, free, but a tree only
//   Rc<RefCell<Node>>     shared owners, checked while the program RUNS (counts
//                         on every clone, a borrow flag on every access)
//   an arena of indices   no ownership graph at all: nodes are plain values in a
//                         Vec and a link is a number
// All three are built from the SAME shape: `splits` records, for every node in
// pre-order, how many of the other nodes go to its left subtree (the draws of
// randomSplit, replayed so every build is identical). Values are in-order ranks.
//
// Why a Box tree cannot have parent pointers (compiled separately, not part of
// this program, which must build; rustc 1.98.1):
//   parent: Option<Box<Node>>      `Box::new(root)` moves the parent into the
//                                  child; using `root` again -> E0382 (moved value)
//   parent: Option<&'a Node<'a>>   `child.parent = Some(&root)` borrows the parent
//                                  for as long as the child lives, so then giving
//                                  the parent its child, `root.left = ..` -> E0506
//                                  (cannot assign because it is borrowed)
// A parent link makes two things point at each other. Rc + Weak says so in the
// types; an arena says it with an index.

// ---- an allocation counter: Go's runtime.MemStats, deterministic ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes ever requested
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(new, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

// ---- the probe: a node whose drop can be watched (lesson 17 example 13) ----
// `_enter` runs when a node's drop starts, `_exit` when it ends; both are
// zero-sized. DROPS counts nodes, MAX_DEPTH the deepest nesting of drops.
static DEPTH: AtomicUsize = AtomicUsize::new(0);
static MAX_DEPTH: AtomicUsize = AtomicUsize::new(0);
static DROPS: AtomicUsize = AtomicUsize::new(0);

fn reset_probe() {
    for c in [&DEPTH, &MAX_DEPTH, &DROPS] {
        c.store(0, Relaxed);
    }
}

struct Enter;
impl Drop for Enter {
    fn drop(&mut self) {
        let depth = DEPTH.fetch_add(1, Relaxed) + 1;
        MAX_DEPTH.fetch_max(depth, Relaxed);
        DROPS.fetch_add(1, Relaxed);
    }
}
struct Exit;
impl Drop for Exit {
    fn drop(&mut self) {
        DEPTH.fetch_sub(1, Relaxed);
    }
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

fn draw_splits(n: usize, r: &mut Pcg, out: &mut Vec<usize>) {
    if n == 0 {
        return;
    }
    let k = r.int_n(n as u64) as usize; // the draw comes BEFORE the recursion
    out.push(k);
    draw_splits(k, r, out);
    draw_splits(n - 1 - k, r, out);
}

fn mix(h: u64, v: i64) -> u64 {
    h.wrapping_mul(31).wrapping_add(v as u64)
}

// What every representation can do. The three impls below are the example.
trait Tree {
    fn size(&self) -> usize;
    fn height(&self) -> usize;
    fn inorder(&self) -> u64; // left, node, right
    fn reverse(&self) -> u64; // right, node, left
    fn invert(&mut self);
}

// ---- 1. Box: one owner per node ----

struct BNode {
    _enter: Enter,
    val: i64,
    left: Option<Box<BNode>>,
    right: Option<Box<BNode>>,
    _exit: Exit,
}
type BLink = Option<Box<BNode>>;
struct BoxTree(BLink);

fn build_box(splits: &[usize], pos: &mut usize, n: usize, next: &mut i64) -> BLink {
    if n == 0 {
        return None;
    }
    let k = splits[*pos];
    *pos += 1;
    let mut node = Box::new(BNode {
        _enter: Enter,
        val: 0,
        left: None,
        right: None,
        _exit: Exit,
    });
    node.left = build_box(splits, pos, k, next);
    node.val = *next;
    *next += 1;
    node.right = build_box(splits, pos, n - 1 - k, next);
    Some(node)
}

impl Tree for BoxTree {
    fn size(&self) -> usize {
        fn go(n: &BLink) -> usize {
            n.as_ref().map_or(0, |b| 1 + go(&b.left) + go(&b.right))
        }
        go(&self.0)
    }
    fn height(&self) -> usize {
        fn go(n: &BLink) -> usize {
            n.as_ref().map_or(0, |b| 1 + go(&b.left).max(go(&b.right)))
        }
        go(&self.0)
    }
    fn inorder(&self) -> u64 {
        fn go(n: &BLink, h: &mut u64) {
            if let Some(b) = n {
                go(&b.left, h);
                *h = mix(*h, b.val);
                go(&b.right, h);
            }
        }
        let mut h = 0;
        go(&self.0, &mut h);
        h
    }
    fn reverse(&self) -> u64 {
        fn go(n: &BLink, h: &mut u64) {
            if let Some(b) = n {
                go(&b.right, h);
                *h = mix(*h, b.val);
                go(&b.left, h);
            }
        }
        let mut h = 0;
        go(&self.0, &mut h);
        h
    }
    fn invert(&mut self) {
        fn go(n: &mut BLink) {
            if let Some(b) = n {
                std::mem::swap(&mut b.left, &mut b.right);
                go(&mut b.left);
                go(&mut b.right);
            }
        }
        go(&mut self.0)
    }
}

// ---- 2. Rc<RefCell<Node>>: shared owners, checked at run time ----
//
// The node is generic over what its parent link is, so ONE definition gives the
// three trees below: `()` (no parent: the usual LeetCode TreeNode), `WeakUp`
// (a Weak parent) and `StrongUp` (an Rc parent: a reference cycle).

type Shared<P> = Rc<RefCell<RcNode<P>>>;
struct RcNode<P> {
    _enter: Enter,
    val: i64,
    left: Option<Shared<P>>,
    right: Option<Shared<P>>,
    parent: P,
    _exit: Exit,
}

trait Up: Sized {
    fn to(parent: Option<&Shared<Self>>) -> Self;
}
impl Up for () {
    fn to(_: Option<&Shared<()>>) {}
}
struct WeakUp(Weak<RefCell<RcNode<WeakUp>>>);
impl Up for WeakUp {
    fn to(parent: Option<&Shared<WeakUp>>) -> Self {
        WeakUp(parent.map_or_else(Weak::new, Rc::downgrade))
    }
}
// never read: it exists only to OWN the parent (hence the underscore)
struct StrongUp {
    _owns_parent: Option<Shared<StrongUp>>,
}
impl Up for StrongUp {
    fn to(parent: Option<&Shared<StrongUp>>) -> Self {
        StrongUp {
            _owns_parent: parent.cloned(),
        }
    }
}

struct RcTree<P>(Option<Shared<P>>);

fn build_rc<P: Up>(
    splits: &[usize],
    pos: &mut usize,
    n: usize,
    next: &mut i64,
    parent: Option<&Shared<P>>,
) -> Option<Shared<P>> {
    if n == 0 {
        return None;
    }
    let k = splits[*pos];
    *pos += 1;
    let node = Rc::new(RefCell::new(RcNode {
        _enter: Enter,
        val: 0,
        left: None,
        right: None,
        parent: P::to(parent),
        _exit: Exit,
    }));
    let left = build_rc(splits, pos, k, next, Some(&node));
    node.borrow_mut().left = left;
    node.borrow_mut().val = *next;
    *next += 1;
    let right = build_rc(splits, pos, n - 1 - k, next, Some(&node));
    node.borrow_mut().right = right;
    Some(node)
}

impl<P> Tree for RcTree<P> {
    fn size(&self) -> usize {
        fn go<P>(n: &Option<Shared<P>>) -> usize {
            n.as_ref().map_or(0, |r| {
                let b = r.borrow();
                1 + go(&b.left) + go(&b.right)
            })
        }
        go(&self.0)
    }
    fn height(&self) -> usize {
        fn go<P>(n: &Option<Shared<P>>) -> usize {
            n.as_ref().map_or(0, |r| {
                let b = r.borrow();
                1 + go(&b.left).max(go(&b.right))
            })
        }
        go(&self.0)
    }
    fn inorder(&self) -> u64 {
        fn go<P>(n: &Option<Shared<P>>, h: &mut u64) {
            if let Some(r) = n {
                let b = r.borrow();
                go(&b.left, h);
                *h = mix(*h, b.val);
                go(&b.right, h);
            }
        }
        let mut h = 0;
        go(&self.0, &mut h);
        h
    }
    fn reverse(&self) -> u64 {
        fn go<P>(n: &Option<Shared<P>>, h: &mut u64) {
            if let Some(r) = n {
                let b = r.borrow();
                go(&b.right, h);
                *h = mix(*h, b.val);
                go(&b.left, h);
            }
        }
        let mut h = 0;
        go(&self.0, &mut h);
        h
    }
    fn invert(&mut self) {
        fn go<P>(n: &Option<Shared<P>>) {
            if let Some(r) = n {
                let mut guard = r.borrow_mut(); // panics if someone holds a borrow
                let b = &mut *guard;
                std::mem::swap(&mut b.left, &mut b.right);
                go(&b.left);
                go(&b.right);
            }
        }
        go(&self.0)
    }
}

// ---- 3. an arena: nodes are values, links are indices ----

const NIL: u32 = u32::MAX;

#[derive(Clone, Copy, PartialEq)]
struct A {
    val: i32,
    left: u32,
    right: u32,
}
struct Arena(Vec<A>);

impl Arena {
    fn build(&mut self, splits: &[usize], pos: &mut usize, n: usize, next: &mut i32) -> u32 {
        if n == 0 {
            return NIL;
        }
        let k = splits[*pos];
        *pos += 1;
        let id = self.0.len() as u32;
        self.0.push(A {
            val: 0,
            left: NIL,
            right: NIL,
        });
        let left = self.build(splits, pos, k, next);
        self.0[id as usize].left = left;
        self.0[id as usize].val = *next;
        *next += 1;
        let right = self.build(splits, pos, n - 1 - k, next);
        self.0[id as usize].right = right;
        id
    }
}

impl Tree for Arena {
    fn size(&self) -> usize {
        fn go(a: &Arena, id: u32) -> usize {
            if id == NIL {
                return 0;
            }
            1 + go(a, a.0[id as usize].left) + go(a, a.0[id as usize].right)
        }
        go(self, 0)
    }
    fn height(&self) -> usize {
        fn go(a: &Arena, id: u32) -> usize {
            if id == NIL {
                return 0;
            }
            1 + go(a, a.0[id as usize].left).max(go(a, a.0[id as usize].right))
        }
        go(self, 0)
    }
    fn inorder(&self) -> u64 {
        fn go(a: &Arena, id: u32, h: &mut u64) {
            if id != NIL {
                go(a, a.0[id as usize].left, h);
                *h = mix(*h, i64::from(a.0[id as usize].val));
                go(a, a.0[id as usize].right, h);
            }
        }
        let mut h = 0;
        go(self, 0, &mut h);
        h
    }
    fn reverse(&self) -> u64 {
        fn go(a: &Arena, id: u32, h: &mut u64) {
            if id != NIL {
                go(a, a.0[id as usize].right, h);
                *h = mix(*h, i64::from(a.0[id as usize].val));
                go(a, a.0[id as usize].left, h);
            }
        }
        let mut h = 0;
        go(self, 0, &mut h);
        h
    }
    fn invert(&mut self) {
        fn go(a: &mut Arena, id: u32) {
            if id != NIL {
                let n = &mut a.0[id as usize];
                std::mem::swap(&mut n.left, &mut n.right);
                let (l, r) = (n.left, n.right);
                go(a, l);
                go(a, r);
            }
        }
        go(self, 0)
    }
}

/// One line of table (b): the operations, then the allocations an invert makes.
fn report(name: &str, t: &mut dyn Tree) -> u64 {
    let (inorder, reverse) = (t.inorder(), t.reverse());
    let (allocs, _, ()) = allocs_of(|| t.invert());
    println!(
        "{:<16} {:>6} {:>7} {:>18x} {:>18x} {:>14} {:>24}",
        name,
        t.size(),
        t.height(),
        inorder,
        reverse,
        allocs,
        t.inorder() == reverse
    );
    t.invert(); // put it back
    inorder
}

fn main() {
    const N: usize = 10_000;
    let mut splits = Vec::new();
    draw_splits(N, &mut Pcg::new(1, 2), &mut splits);

    println!(
        "the tree: randomSplit({N}), values = in-order ranks; size_of: BNode {}, RcNode<()> {}, RcNode<WeakUp> {}, A {} bytes",
        size_of::<BNode>(),
        size_of::<RcNode<()>>(),
        size_of::<RcNode<WeakUp>>(),
        size_of::<A>()
    );

    // ---- (a) building the same tree three ways ----
    println!("\n(a) building the same tree: allocations and bytes requested");
    println!(
        "{:<28} {:>7} {:>8} {:>12} {:>9} {:>11}",
        "representation", "node B", "allocs", "bytes", "B / node", "live after"
    );
    let row = |name: &str, node_bytes: usize, allocs: usize, bytes: usize, live: usize| {
        println!(
            "{name:<28} {node_bytes:>7} {allocs:>8} {bytes:>12} {:>9.1} {live:>11}",
            bytes as f64 / N as f64
        );
    };
    let live0 = LIVE.load(Relaxed);
    let (a, b, boxed) = allocs_of(|| build_box(&splits, &mut 0, N, &mut 0));
    row(
        "Box<BNode>",
        size_of::<BNode>(),
        a,
        b,
        LIVE.load(Relaxed) - live0,
    );
    let boxed = BoxTree(boxed);

    let live0 = LIVE.load(Relaxed);
    let (a, b, rc) = allocs_of(|| build_rc::<()>(&splits, &mut 0, N, &mut 0, None));
    row(
        "Rc<RefCell<RcNode<()>>>",
        size_of::<RcNode<()>>(),
        a,
        b,
        LIVE.load(Relaxed) - live0,
    );
    let rc = RcTree(rc);

    let live0 = LIVE.load(Relaxed);
    let (a, b, rcw) = allocs_of(|| build_rc::<WeakUp>(&splits, &mut 0, N, &mut 0, None));
    row(
        "... + Weak parent",
        size_of::<RcNode<WeakUp>>(),
        a,
        b,
        LIVE.load(Relaxed) - live0,
    );
    let rcw = RcTree(rcw);

    let live0 = LIVE.load(Relaxed);
    let (a, b, grown) = allocs_of(|| {
        let mut arena = Arena(Vec::new());
        arena.build(&splits, &mut 0, N, &mut 0);
        arena
    });
    row(
        "arena, grown by push",
        size_of::<A>(),
        a,
        b,
        LIVE.load(Relaxed) - live0,
    );

    let live0 = LIVE.load(Relaxed);
    let (a, b, mut reserved) = allocs_of(|| {
        let mut arena = Arena(Vec::with_capacity(N));
        arena.build(&splits, &mut 0, N, &mut 0);
        arena
    });
    row(
        "arena, capacity reserved",
        size_of::<A>(),
        a,
        b,
        LIVE.load(Relaxed) - live0,
    );

    // ---- (b) the same operations on each ----
    println!("\n(b) the same operations on each (checksum: h = h*31 + value, wrapping)");
    println!(
        "{:<16} {:>6} {:>7} {:>18} {:>18} {:>14} {:>24}",
        "representation",
        "size",
        "height",
        "in-order checksum",
        "reverse checksum",
        "invert allocs",
        "after invert == reverse"
    );
    let (mut boxed, mut rc, mut rcw) = (boxed, rc, rcw);
    let sums = [
        report("Box", &mut boxed),
        report("Rc<RefCell>", &mut rc),
        report("Rc + Weak parent", &mut rcw),
        report("arena (indices)", &mut reserved),
    ];
    println!(
        "all four agree on the checksum: {}; the grown and the reserved arena hold equal nodes: {}",
        sums.iter().all(|&s| s == sums[0]),
        grown.0 == reserved.0
    );
    drop((grown, rc));

    // ---- (c) depth by walking up the parent links ----
    // Sum over all nodes of their depth, found three ways: top-down, by walking
    // up Weak parents (an upgrade per step), and by walking up an arena parent table.
    fn top_down(n: &BLink, d: usize) -> usize {
        n.as_ref().map_or(0, |b| {
            d + top_down(&b.left, d + 1) + top_down(&b.right, d + 1)
        })
    }
    let mut steps = 0;
    let mut work: Vec<Shared<WeakUp>> = rcw.0.iter().cloned().collect();
    while let Some(node) = work.pop() {
        let mut up = node.borrow().parent.0.upgrade();
        while let Some(p) = up {
            steps += 1;
            up = p.borrow().parent.0.upgrade();
        }
        let b = node.borrow();
        work.extend(b.left.iter().chain(b.right.iter()).cloned());
    }
    let mut parent = vec![NIL; N];
    for (i, nd) in reserved.0.iter().enumerate() {
        for c in [nd.left, nd.right] {
            if c != NIL {
                parent[c as usize] = i as u32;
            }
        }
    }
    let mut asteps = 0;
    for i in 0..N {
        let mut p = parent[i];
        while p != NIL {
            asteps += 1;
            p = parent[p as usize];
        }
    }
    println!("\n(c) depth of every node by walking up (steps = sum of depths)");
    println!("top-down sum of depths: {}", top_down(&boxed.0, 0));
    println!(
        "Weak parent walk: {steps} steps (an upgrade() each); arena parent table: {asteps} steps (an index each, table = {} bytes)",
        parent.len() * 4
    );

    // ---- (d) what only Rust has: counts, cycles, a runtime borrow check ----
    println!("\n(d) Weak parents: the whole tree is freed");
    reset_probe();
    let live_before = LIVE.load(Relaxed);
    {
        let root = rcw.0.as_ref().expect("a root");
        // walk down the leftmost path to a leaf, keeping one handle to it
        let mut leaf = root.clone();
        loop {
            let next = {
                let b = leaf.borrow();
                b.left.clone().or_else(|| b.right.clone())
            };
            match next {
                Some(c) => leaf = c,
                None => break,
            }
        }
        println!(
            "root: strong {} weak {} (one Weak parent link per child); a leaf, held by one extra handle of ours: strong {} weak {}",
            Rc::strong_count(root),
            Rc::weak_count(root),
            Rc::strong_count(&leaf),
            Rc::weak_count(&leaf)
        );
        let parent_link = &leaf.borrow().parent.0;
        let before = parent_link.strong_count();
        let up = parent_link.upgrade().expect("a parent");
        println!(
            "the leaf's parent has strong count {before}; while an upgraded handle to it is alive: {}",
            Rc::strong_count(&up)
        );
    }
    drop(rcw);
    println!(
        "dropped the root: {} of {N} nodes freed, nested at most {} deep, {} bytes returned",
        DROPS.load(Relaxed),
        MAX_DEPTH.load(Relaxed),
        live_before.abs_diff(LIVE.load(Relaxed))
    );

    println!("\n(d) Rc parents instead of Weak: a reference cycle");
    reset_probe();
    let live_before = LIVE.load(Relaxed);
    let strong = RcTree(build_rc::<StrongUp>(&splits, &mut 0, N, &mut 0, None));
    let watcher = Rc::downgrade(strong.0.as_ref().expect("a root"));
    println!(
        "built: root strong count {} (our handle plus one parent link per child)",
        Rc::strong_count(strong.0.as_ref().expect("a root"))
    );
    drop(strong);
    println!(
        "dropped our handle: {} of {N} nodes freed; the root is still alive (upgrade() = {}); leaked bytes {}",
        DROPS.load(Relaxed),
        if watcher.upgrade().is_some() {
            "Some"
        } else {
            "None"
        },
        LIVE.load(Relaxed) - live_before
    );
    // The watcher can still reach the cycle: cut every parent link, then drop.
    let root = watcher.upgrade().expect("the cycle keeps the root alive");
    let mut work = vec![root.clone()];
    while let Some(node) = work.pop() {
        node.borrow_mut().parent = StrongUp { _owns_parent: None };
        let b = node.borrow();
        work.extend(b.left.iter().chain(b.right.iter()).cloned());
    }
    drop(root);
    println!(
        "cut the parent links through the watcher and dropped: {} of {N} nodes freed",
        DROPS.load(Relaxed)
    );
    drop((watcher, work));
    println!(
        "bytes still live from this tree: {}",
        LIVE.load(Relaxed) - live_before
    );

    println!("\n(d) the runtime borrow check, without panicking");
    let cell = RefCell::new(5);
    {
        let reader = cell.borrow();
        println!(
            "holding borrow() = {}: try_borrow_mut() = {:?}",
            *reader,
            cell.try_borrow_mut().map(|_| ())
        );
    }
    println!(
        "borrow released: try_borrow_mut() = {:?}",
        cell.try_borrow_mut().map(|_| ())
    );

    println!("\n(d) is dropping an Rc chain recursive? count the nesting on 1000-node chains");
    reset_probe();
    let mut b: BLink = None;
    for _ in 0..1000 {
        b = Some(Box::new(BNode {
            _enter: Enter,
            val: 0,
            left: b,
            right: None,
            _exit: Exit,
        }));
    }
    drop(b);
    let box_nesting = MAX_DEPTH.load(Relaxed);
    reset_probe();
    let mut r: Option<Shared<()>> = None;
    for _ in 0..1000 {
        r = Some(Rc::new(RefCell::new(RcNode {
            _enter: Enter,
            val: 0,
            left: r,
            right: None,
            parent: (),
            _exit: Exit,
        })));
    }
    drop(r);
    println!(
        "Box chain: drop nested {box_nesting} deep; Rc<RefCell> chain: drop nested {} deep",
        MAX_DEPTH.load(Relaxed)
    );
}
```

**Rust sample output:**

```
the tree: randomSplit(10000), values = in-order ranks; size_of: BNode 24, RcNode<()> 24, RcNode<WeakUp> 32, A 12 bytes

(a) building the same tree: allocations and bytes requested
representation                node B   allocs        bytes  B / node  live after
Box<BNode>                        24    10000       240000      24.0      240000
Rc<RefCell<RcNode<()>>>           24    10000       480000      48.0      480000
... + Weak parent                 32    10000       560000      56.0      560000
arena, grown by push              12       13       393168      39.3      196608
arena, capacity reserved          12        1       120000      12.0      120000

(b) the same operations on each (checksum: h = h*31 + value, wrapping)
representation     size  height  in-order checksum   reverse checksum  invert allocs  after invert == reverse
Box               10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
Rc<RefCell>       10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
Rc + Weak parent  10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
arena (indices)   10000      31   9d17b2b755118388   d55a565644d0fb78              0                     true
all four agree on the checksum: true; the grown and the reserved arena hold equal nodes: true

(c) depth of every node by walking up (steps = sum of depths)
top-down sum of depths: 154829
Weak parent walk: 154829 steps (an upgrade() each); arena parent table: 154829 steps (an index each, table = 40000 bytes)

(d) Weak parents: the whole tree is freed
root: strong 1 weak 2 (one Weak parent link per child); a leaf, held by one extra handle of ours: strong 2 weak 0
the leaf's parent has strong count 1; while an upgraded handle to it is alive: 2
dropped the root: 10000 of 10000 nodes freed, nested at most 31 deep, 560000 bytes returned

(d) Rc parents instead of Weak: a reference cycle
built: root strong count 3 (our handle plus one parent link per child)
dropped our handle: 0 of 10000 nodes freed; the root is still alive (upgrade() = Some); leaked bytes 560000
cut the parent links through the watcher and dropped: 10000 of 10000 nodes freed
bytes still live from this tree: 0

(d) the runtime borrow check, without panicking
holding borrow() = 5: try_borrow_mut() = Err(BorrowMutError)
borrow released: try_borrow_mut() = Ok(())

(d) is dropping an Rc chain recursive? count the nesting on 1000-node chains
Box chain: drop nested 1000 deep; Rc<RefCell> chain: drop nested 1000 deep
```

> **Go vs Rust:** The rows that exist in both languages agree: `Box` is the Go pointer tree allocation for allocation (10,000 of 24 B), and sizes, heights, checksums and the 154,829 steps are identical. `Rc<RefCell<_>>` costs 48 B a node (the two counts and the borrow flag on top of the 24 B node) and 56 B with a `Weak` parent, the arena 12 B a node in one allocation when its size is known and 13 allocations (196,608 B live, 393,168 B requested) when grown by `push`, where Go's `append` made 18. A `Box` tree cannot have parent pointers: compiled separately, an owning `Box` parent is E0382 (the parent is moved into the child) and a `&` parent is E0506 (the parent cannot be given its child while the child borrows it). Dropping an `Rc` chain nests as deep as the chain (1,000, same as `Box`; an overflow was not tried). Go has no counterpart to the 0-of-10,000 nodes freed under `Rc` parents (its collector reclaimed the same cycle) or to `try_borrow_mut`'s `Err(BorrowMutError)`.

**Complexity:** the pointer tree costs n allocations of 24 B (240,000 B for n = 10,000); a parent field makes it 32 B per node (320,000 B); the arena is one allocation of 12 B per node when its size is known (the allocator handed out 122,880 B for 120,000) and 18 allocations, 489,464 B, when it is grown by `append`. All three agree on size, height and checksum, inverting in place allocates nothing in any of them and reverses the in-order checksum, and walking up a parent link or a parent table costs one read per step (154,829 steps, the sum of the depths). A parent link makes the tree a cycle, and Go's collector reclaimed it without being asked: a weak pointer to its root was nil after `runtime.GC()`.

---

## 15. A traversal that can stop: iterators

`🔴 hard` · *Lazy traversal*

A recursive walk is naturally *push*: it calls your function once per node. An iterator is *pull*: you ask for the next
value when you want it. Go 1.23 wraps the push walk in an `iter.Seq`, so `for ... range` and `break` work on a tree (`break`
makes `yield` return `false`, and the recursion unwinds), and `iter.Pull` turns the same walk into `next`/`stop`. The payoff
is that stopping after k values costs the descent to the first one plus about two calls per value, `Θ(h + k)`, instead of the
`2n + 1` calls of building a slice. The classic use is the *same fringe* test: two trees with the same in-order sequence
and different shapes, decided without building either sequence, and stopped at the first difference. The program counts the
calls (nil hits included) for k = 1, 10, 100 and all on a 100,000-node tree, breaks the contract on purpose to read the
runtime's panic, pulls ten values and stops, runs same fringe on a rotated tree and on two big trees that differ at
position 5, and ends with one timing table for the price of each style.

**Steps:**

1. Write `walk(n, depth, yield) bool` that counts every call and returns `false` as soon as `yield` does, wrap it in
   `InOrder(root) iter.Seq[int]`, and print the calls and the deepest recursion for `break` after 1, 10, 100 values and for a
   full walk, next to eagerly collecting a slice.
2. Break the contract (a walk that ignores `yield`'s answer; a `yield` called after the loop ended) and catch the runtime
   panics, then drive the same walk with `iter.Pull`, and use two `Pull`s in lockstep for the same-fringe test on a
   rotated tree and on two big trees.
3. Read the counts: 11, 30 and 211 calls for the first 1, 10 and 100 values against 200,001 for all of them, Pull and
   `break` making identical calls, a difference at position 5 found in a few dozen calls instead of 400,002, and what `Pull`
   costs per value.

```go
package main

import (
	"fmt"
	"iter"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
	"time"
)

// Node is a binary tree node; nil is the empty tree.
type Node struct {
	Val         int
	Left, Right *Node
}

// randomSplit draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes. The draw happens before recursing, so the PCG stream is
// consumed in one fixed order (the Rust twin draws the same numbers).
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelInOrder gives the nodes the values 0, 1, 2, ... in in-order, so a
// tree's in-order sequence is just 0..n-1 whatever its shape.
func labelInOrder(t *Node, next *int) {
	if t == nil {
		return
	}
	labelInOrder(t.Left, next)
	t.Val = *next
	*next++
	labelInOrder(t.Right, next)
}

// setAt overwrites the value at in-order position target.
func setAt(t *Node, pos *int, target, v int) {
	if t == nil {
		return
	}
	setAt(t.Left, pos, target, v)
	if *pos == target {
		t.Val = v
	}
	*pos++
	setAt(t.Right, pos, target, v)
}

func leaf(v int) *Node { return &Node{Val: v} }

// handBuilt is the 9-node tree of examples 1-6.
func handBuilt() *Node {
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// rotateLeft lifts the right child over the root. The shape changes, the
// in-order sequence does not: a rotation is the classic same-fringe pair.
func rotateLeft(root *Node) *Node {
	pivot := root.Right
	root.Right = pivot.Left
	pivot.Left = root
	return pivot
}

func height(t *Node) int {
	if t == nil {
		return 0
	}
	return 1 + max(height(t.Left), height(t.Right))
}

func leftSpine(t *Node) int { // nodes on the path that only goes left
	n := 0
	for ; t != nil; t = t.Left {
		n++
	}
	return n
}

// draw prints a tree sideways: right subtree first, 4 spaces per depth.
func draw(t *Node, depth int) {
	if t == nil {
		return
	}
	draw(t.Right, depth+1)
	fmt.Printf("%s%d\n", strings.Repeat("    ", depth), t.Val)
	draw(t.Left, depth+1)
}

// ---- the walk, in push style ----

// walker counts what one walk does. calls is every walk() call, nil hits
// included (an n-node tree makes 2n+1 when the walk runs to the end);
// high is the deepest nesting of node frames: the stack a walk needs.
type walker struct{ calls, high int }

// walk is in-order with an early exit: it returns false as soon as yield
// does, and every caller up the chain returns false too.
func (w *walker) walk(n *Node, depth int, yield func(int) bool) bool {
	w.calls++
	if n == nil {
		return true
	}
	w.high = max(w.high, depth+1)
	return w.walk(n.Left, depth+1, yield) && yield(n.Val) && w.walk(n.Right, depth+1, yield)
}

// InOrder wraps the recursion in an iter.Seq, exactly as go/ast.Preorder wraps
// Inspect. Nothing runs until somebody ranges over it.
func InOrder(root *Node, w *walker) iter.Seq[int] {
	return func(yield func(int) bool) { w.walk(root, 0, yield) }
}

// stopAfter ranges over seq and breaks after k values (k <= 0: never).
func stopAfter(seq iter.Seq[int], k int) int {
	got := 0
	for range seq {
		got++
		if got == k {
			break // makes yield return false: the walk unwinds
		}
	}
	return got
}

// ---- (b) the bug: an iterator that ignores yield's answer ----

func (w *walker) badWalk(n *Node, yield func(int) bool) {
	w.calls++
	if n == nil {
		return
	}
	w.badWalk(n.Left, yield)
	yield(n.Val) // BUG: the result is thrown away, so "stop" is never heard
	w.badWalk(n.Right, yield)
}

func try(name string, f func()) {
	defer func() { fmt.Printf("  %-28s recovered: %v\n", name, recover()) }()
	f()
}

// ---- (d) same fringe ----

// sameFringe pulls both sequences in lockstep and stops at the first
// difference. It returns the position of that difference, or the common
// length when there is none.
func sameFringe(a, b iter.Seq[int]) (equal bool, pos int) {
	nextA, stopA := iter.Pull(a)
	defer stopA() // Pull's rule: stop what you do not finish
	nextB, stopB := iter.Pull(b)
	defer stopB()
	for i := 0; ; i++ {
		x, okX := nextA()
		y, okY := nextB()
		if okX != okY || x != y {
			return false, i
		}
		if !okX {
			return true, i
		}
	}
}

var sink int

// bestNsPerValue runs each strategy 3 rounds, interleaved, and keeps the best.
func bestNsPerValue(n, reps int, strategies []func() int) []float64 {
	best := make([]float64, len(strategies))
	for round := 0; round < 3; round++ {
		for i, f := range strategies {
			t := time.Now()
			for range reps {
				sink += f()
			}
			ns := float64(time.Since(t).Nanoseconds()) / float64(n*reps)
			if round == 0 || ns < best[i] {
				best[i] = ns
			}
		}
	}
	return best
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	const n = 100000
	big := randomSplit(n, r)
	rank := 0
	labelInOrder(big, &rank)
	fmt.Printf("tree A: randomSplit(%d), height %d, left spine %d nodes\n", n, height(big), leftSpine(big))

	// (a) stop after k: the walk costs the descent plus about two calls per value.
	fmt.Println("\n(a) in-order as an iter.Seq: break after k values")
	fmt.Printf("%12s %10s %10s %8s %10s\n", "stop after", "calls", "calls-2k", "frames", "values")
	spine := leftSpine(big)
	callsAt := map[int]int{} // k -> calls
	// k = 0 means never break: run to the end
	for _, k := range []int{1, 10, 100, 0} {
		var w walker
		got := stopAfter(InOrder(big, &w), k)
		label := fmt.Sprint(k)
		if k == 0 {
			label = "all"
		}
		fmt.Printf("%12s %10d %10d %8d %10d\n", label, w.calls, w.calls-2*got, w.high, got)
		callsAt[k] = w.calls
	}
	fmt.Printf("value 1 cost %d calls: the left spine (%d nodes) plus its nil\n", callsAt[1], spine)
	fmt.Printf("values 11..100 cost %.2f calls each\n", float64(callsAt[100]-callsAt[10])/90)

	var wEager walker
	all := slices.Collect(InOrder(big, &wEager))
	fmt.Printf("eager slices.Collect: %d calls (2n+1 = %d), %d values, %d bytes of []int\n",
		wEager.calls, 2*n+1, len(all), len(all)*8)

	// allocations: AllocsPerRun is a count, not a clock.
	lazy10 := testing.AllocsPerRun(20, func() { var w walker; stopAfter(InOrder(big, &w), 10) })
	eager := testing.AllocsPerRun(20, func() { var w walker; sink += len(slices.Collect(InOrder(big, &w))) })
	fmt.Printf("allocations: stop after 10 = %.0f, collect all = %.0f\n", lazy10, eager)

	// (b) the bug.
	fmt.Println("\n(b) a push iterator that ignores yield's result, then 'break' after 3")
	{
		var w walker
		bad := func(yield func(int) bool) { w.badWalk(big, yield) }
		got := 0
		try("break after 3 values", func() {
			for range bad {
				got++
				if got == 3 {
					break
				}
			}
		})
		fmt.Printf("  the body ran %d times; the walk made %d calls before the next yield caught it\n", got, w.calls)

		// The other misuse the runtime catches: yield kept and called after the loop is over.
		var saved func(int) bool
		keeper := func(yield func(int) bool) { saved = yield; yield(1) }
		for range keeper {
		}
		try("yield called after the loop", func() { saved(2) })
	}

	// (c) iter.Pull: the same walk, driven by next() instead of a loop body.
	fmt.Println("\n(c) iter.Pull: take 10 values, then stop")
	{
		var w walker
		next, stop := iter.Pull(InOrder(big, &w))
		sum := 0
		for range 10 {
			v, _ := next()
			sum += v
		}
		before := w.calls
		stop()
		_, ok := next()
		fmt.Printf("  10 values summed to %d; calls %d before stop(), %d after; next() after stop ok=%v\n",
			sum, before, w.calls, ok)
		var wb walker
		stopAfter(InOrder(big, &wb), 10)
		fmt.Printf("  break after 10 made %d calls: Pull and range walk identically (%v)\n", wb.calls, wb.calls == w.calls)
	}

	// (d) same fringe.
	fmt.Println("\n(d) same fringe: equal sequences from different shapes")
	t1, t2 := handBuilt(), rotateLeft(handBuilt())
	fmt.Println("tree 1:")
	draw(t1, 0)
	fmt.Println("tree 2 (tree 1 rotated left at the root: another shape, same in-order):")
	draw(t2, 0)
	var w1, w2 walker
	eq, pos := sameFringe(InOrder(t1, &w1), InOrder(t2, &w2))
	fmt.Printf("same fringe: equal=%v over %d values; calls %d and %d (2n+1 = %d)\n", eq, pos, w1.calls, w2.calls, 2*9+1)

	// Tree B has another shape; its in-order sequence is A's with the value at position 5 changed.
	other := randomSplit(n, r)
	rank = 0
	labelInOrder(other, &rank)
	pos = 0
	setAt(other, &pos, 5, -1)
	fmt.Printf("\ntree B: another randomSplit(%d), height %d, left spine %d, same values as A except position 5\n",
		n, height(other), leftSpine(other))

	var wa, wb walker
	eq, pos = sameFringe(InOrder(big, &wa), InOrder(other, &wb))
	fmt.Printf("lazy:  equal=%v, first difference at position %d; calls A %d, B %d (spines %d and %d)\n",
		eq, pos, wa.calls, wb.calls, spine, leftSpine(other))
	var ea, eb walker
	sa, sb := slices.Collect(InOrder(big, &ea)), slices.Collect(InOrder(other, &eb))
	fmt.Printf("eager: equal=%v; calls A %d, B %d, %d values built\n", slices.Equal(sa, sb), ea.calls, eb.calls, len(sa)+len(sb))
	fmt.Printf("the lazy test made %.0fx fewer calls than the eager one\n", float64(ea.calls+eb.calls)/float64(wa.calls+wb.calls))

	// (e) the price of the pull style, measured.
	fmt.Println("\n(e) ns per value, full walk of tree A (best of 3)")
	var w walker
	seq := InOrder(big, &w)
	best := bestNsPerValue(n, 20, []func() int{
		func() int { // push: plain callback
			s := 0
			w.walk(big, 0, func(v int) bool { s += v; return true })
			return s
		},
		func() int { // push: range over the Seq
			s := 0
			for v := range seq {
				s += v
			}
			return s
		},
		func() int { // pull: iter.Pull
			next, stop := iter.Pull(seq)
			defer stop()
			s := 0
			for {
				v, ok := next()
				if !ok {
					return s
				}
				s += v
			}
		},
		func() int { // eager: build the slice, then loop
			s := 0
			for _, v := range slices.Collect(seq) {
				s += v
			}
			return s
		},
	})
	names := []string{"callback", "range over Seq", "iter.Pull", "eager slice"}
	for i, name := range names {
		fmt.Printf("  %-16s %7.1f ns/value\n", name, best[i])
	}
	fmt.Printf("  Pull is about %.0fx the range loop\n", best[2]/best[1])
}
```

**Sample output:**

```
tree A: randomSplit(100000), height 38, left spine 10 nodes

(a) in-order as an iter.Seq: break after k values
  stop after      calls   calls-2k   frames     values
           1         11          9       10          1
          10         30         10       14         10
         100        211         11       21        100
         all     200001          1       38     100000
value 1 cost 11 calls: the left spine (10 nodes) plus its nil
values 11..100 cost 2.01 calls each
eager slices.Collect: 200001 calls (2n+1 = 200001), 100000 values, 800000 bytes of []int
allocations: stop after 10 = 0, collect all = 28

(b) a push iterator that ignores yield's result, then 'break' after 3
  break after 3 values         recovered: runtime error: range function continued iteration after function for loop body returned false
  the body ran 3 times; the walk made 18 calls before the next yield caught it
  yield called after the loop  recovered: runtime error: range function continued iteration after whole loop exit

(c) iter.Pull: take 10 values, then stop
  10 values summed to 45; calls 30 before stop(), 30 after; next() after stop ok=false
  break after 10 made 30 calls: Pull and range walk identically (true)

(d) same fringe: equal sequences from different shapes
tree 1:
            9
        6
            8
    3
1
        5
            7
    2
        4
tree 2 (tree 1 rotated left at the root: another shape, same in-order):
        9
    6
        8
3
    1
            5
                7
        2
            4
same fringe: equal=true over 9 values; calls 19 and 19 (2n+1 = 19)

tree B: another randomSplit(100000), height 40, left spine 11, same values as A except position 5
lazy:  equal=false, first difference at position 5; calls A 21, B 24 (spines 10 and 11)
eager: equal=false; calls A 200001, B 200001, 200000 values built
the lazy test made 8889x fewer calls than the eager one

(e) ns per value, full walk of tree A (best of 3)
  callback             6.7 ns/value
  range over Seq       7.0 ns/value
  iter.Pull           49.4 ns/value
  eager slice         11.4 ns/value
  Pull is about 7x the range loop
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::ops::ControlFlow;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::Instant;

// ---- allocation counter (the kit's): stands in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so both programs build the same trees from the seed (1, 2).
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
}

// A binary tree node; None is the empty tree. No custom Drop here: this
// program's trees are at most ~40 levels deep, so the default recursive drop
// glue is harmless (examples 13 and 16 deal with trees that can be deep).
struct Node {
    val: i64,
    left: Link,
    right: Link,
}

type Link = Option<Box<Node>>;

fn mk(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

// random_split draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes. The draw happens before recursing, so the PCG stream is
// consumed in the same order as in the Go program.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    mk(0, left, right)
}

// label_in_order gives the nodes the values 0, 1, 2, ... in in-order, so a
// tree's in-order sequence is just 0..n-1 whatever its shape.
fn label_in_order(t: &mut Link, next: &mut i64) {
    if let Some(n) = t {
        label_in_order(&mut n.left, next);
        n.val = *next;
        *next += 1;
        label_in_order(&mut n.right, next);
    }
}

// set_at overwrites the value at in-order position target.
fn set_at(t: &mut Link, pos: &mut usize, target: usize, v: i64) {
    if let Some(n) = t {
        set_at(&mut n.left, pos, target, v);
        if *pos == target {
            n.val = v;
        }
        *pos += 1;
        set_at(&mut n.right, pos, target, v);
    }
}

fn leaf(v: i64) -> Link {
    mk(v, None, None)
}

// hand_built is the 9-node tree of examples 1-6.
fn hand_built() -> Box<Node> {
    Box::new(Node {
        val: 1,
        left: mk(2, leaf(4), mk(5, leaf(7), None)),
        right: mk(3, None, mk(6, leaf(8), leaf(9))),
    })
}

// rotate_left lifts the right child over the root. The shape changes, the
// in-order sequence does not: a rotation is the classic same-fringe pair.
// Ownership makes the three pointer moves explicit: take() the pivot out,
// take() its left subtree out, then give the old root to the pivot.
fn rotate_left(mut root: Box<Node>) -> Box<Node> {
    let mut pivot = root.right.take().expect("rotate_left needs a right child");
    root.right = pivot.left.take();
    pivot.left = Some(root);
    pivot
}

fn height(t: &Link) -> usize {
    match t {
        None => 0,
        Some(n) => 1 + height(&n.left).max(height(&n.right)),
    }
}

fn left_spine(mut t: &Link) -> usize {
    // nodes on the path that only goes left
    let mut n = 0;
    while let Some(node) = t {
        n += 1;
        t = &node.left;
    }
    n
}

// draw prints a tree sideways: right subtree first, 4 spaces per depth.
fn draw(t: &Link, depth: usize) {
    if let Some(n) = t {
        draw(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.val);
        draw(&n.left, depth + 1);
    }
}

// ---- the walk, in push style: a recursive visitor that can say "stop" ----

// Walker counts what one walk does. calls is every walk() call, nil hits
// included (an n-node tree makes 2n+1 when the walk runs to the end); high is
// the deepest nesting of node frames: the stack a walk needs.
#[derive(Default)]
struct Walker {
    calls: usize,
    high: usize,
}

impl Walker {
    // walk is in-order with an early exit. ControlFlow::Break(()) is the
    // visitor's "stop", and `?` hands it to every caller up the chain: the
    // same job as `return false` in the Go version.
    fn walk(
        &mut self,
        n: &Link,
        depth: usize,
        f: &mut impl FnMut(i64) -> ControlFlow<()>,
    ) -> ControlFlow<()> {
        self.calls += 1;
        if let Some(b) = n {
            self.high = self.high.max(depth + 1);
            self.walk(&b.left, depth + 1, f)?;
            f(b.val)?;
            self.walk(&b.right, depth + 1, f)?;
        }
        ControlFlow::Continue(())
    }

    // bad_walk throws away what the visitor says, the way Go's badWalk throws
    // away yield's result. The compiler warns about the discarded
    // ControlFlow (it is #[must_use]); `let _ =` is how you tell it you mean it.
    fn bad_walk(&mut self, n: &Link, f: &mut impl FnMut(i64) -> ControlFlow<()>) {
        self.calls += 1;
        if let Some(b) = n {
            self.bad_walk(&b.left, f);
            let _ = f(b.val);
            self.bad_walk(&b.right, f);
        }
    }
}

// ---- the walk, in pull style: the call stack written as a Vec ----

// InOrder is the whole state of an in-order walk: the nodes whose left subtree
// is being walked (stack) and the link to look at next (cur). next() is the
// loop body of example 3, paused after every value. calls and high count
// exactly what Walker counts: one "call" per link looked at, nil or not.
struct InOrder<'a> {
    stack: Vec<&'a Node>,
    cur: Option<&'a Node>,
    calls: usize,
    high: usize,
}

impl<'a> InOrder<'a> {
    fn new(root: &'a Link) -> Self {
        InOrder {
            stack: Vec::new(),
            cur: root.as_deref(),
            calls: 0,
            high: 0,
        }
    }
}

impl Iterator for InOrder<'_> {
    type Item = i64;

    fn next(&mut self) -> Option<i64> {
        loop {
            self.calls += 1; // one look at a link
            match self.cur {
                Some(n) => {
                    self.stack.push(n);
                    self.high = self.high.max(self.stack.len());
                    self.cur = n.left.as_deref();
                }
                None => {
                    // an empty link: the top of the stack has finished its left side
                    let n = self.stack.pop()?; // empty stack: the walk is over
                    self.cur = n.right.as_deref();
                    return Some(n.val);
                }
            }
        }
    }
}

// stop_after takes k values and stops asking (k == 0: never stop).
fn stop_after(it: &mut InOrder, k: usize) -> usize {
    it.by_ref()
        .take(if k == 0 { usize::MAX } else { k })
        .count()
}

// same_fringe walks both sequences in lockstep and stops at the first
// difference. It returns the position of that difference, or the common
// length when there is none.
fn same_fringe(a: &mut InOrder, b: &mut InOrder) -> (bool, usize) {
    let mut pos = 0;
    loop {
        match (a.next(), b.next()) {
            (None, None) => return (true, pos),
            (x, y) if x != y => return (false, pos),
            _ => pos += 1,
        }
    }
}

// best_ns_per_value runs each strategy 3 rounds, interleaved, and keeps the best.
fn best_ns_per_value(
    n: usize,
    reps: usize,
    strategies: &mut [&mut dyn FnMut() -> i64],
) -> Vec<f64> {
    let mut best = vec![0.0f64; strategies.len()];
    for round in 0..3 {
        for (i, f) in strategies.iter_mut().enumerate() {
            let t = Instant::now();
            let mut sink = 0i64;
            for _ in 0..reps {
                sink = sink.wrapping_add(f());
            }
            black_box(sink);
            let ns = t.elapsed().as_nanos() as f64 / (n * reps) as f64;
            if round == 0 || ns < best[i] {
                best[i] = ns;
            }
        }
    }
    best
}

fn main() {
    let mut r = Pcg::new(1, 2);
    const N: usize = 100_000;
    let mut big = random_split(N, &mut r);
    let mut rank = 0;
    label_in_order(&mut big, &mut rank);
    println!(
        "tree A: randomSplit({N}), height {}, left spine {} nodes",
        height(&big),
        left_spine(&big)
    );

    // (a) stop after k: the walk costs the descent plus about two calls per value.
    // Two walks run side by side: the Iterator (explicit stack) and the recursive
    // visitor (call stack). frames = deepest recursion; stack = the Vec's high-water.
    println!("\n(a) in-order as an Iterator: take(k)");
    println!(
        "{:>12} {:>10} {:>10} {:>8} {:>8} {:>10}",
        "stop after", "calls", "calls-2k", "frames", "stack", "values"
    );
    let spine = left_spine(&big);
    let mut calls_at = std::collections::HashMap::new(); // k -> calls
    let mut calls_agree = true;
    let mut stack_never_deeper = true;
    for k in [1, 10, 100, 0] {
        let mut it = InOrder::new(&big);
        let got = stop_after(&mut it, k);
        // The recursive visitor with ControlFlow, stopped at the same value.
        let mut w = Walker::default();
        let mut seen = 0;
        let outcome = w.walk(&big, 0, &mut |_| {
            seen += 1;
            if seen == k {
                ControlFlow::Break(())
            } else {
                ControlFlow::Continue(())
            }
        });
        // (ControlFlow is #[must_use]: dropping it is an error under clippy -D warnings)
        calls_agree &= (w.calls, seen) == (it.calls, got) && outcome.is_break() == (k != 0);
        stack_never_deeper &= it.high <= w.high;
        let label = if k == 0 {
            "all".to_string()
        } else {
            k.to_string()
        };
        println!(
            "{:>12} {:>10} {:>10} {:>8} {:>8} {:>10}",
            label,
            it.calls,
            it.calls as i64 - 2 * got as i64,
            w.high,
            it.high,
            got
        );
        calls_at.insert(k, it.calls);
    }
    println!(
        "value 1 cost {} calls: the left spine ({spine} nodes) plus its nil",
        calls_at[&1]
    );
    println!(
        "values 11..100 cost {:.2} calls each",
        (calls_at[&100] - calls_at[&10]) as f64 / 90.0
    );
    println!(
        "the Iterator and the ControlFlow visitor made the same calls on every row: {calls_agree}"
    );
    println!(
        "the Iterator's stack was never deeper than the visitor's frames: {stack_never_deeper}"
    );

    let mut it = InOrder::new(&big);
    let all: Vec<i64> = it.by_ref().collect();
    println!(
        "eager collect: {} calls (2n+1 = {}), {} values, {} bytes of Vec<i64>",
        it.calls,
        2 * N + 1,
        all.len(),
        all.len() * 8
    );

    // allocations: the stack Vec is the one heap object a lazy walk owns.
    let mut lazy = InOrder::new(&big);
    let (lazy10, _) = allocs_of(|| stop_after(&mut lazy, 10));
    let (eager, _) = allocs_of(|| InOrder::new(&big).collect::<Vec<i64>>().len());
    println!(
        "allocations: stop after 10 = {lazy10} (its stack Vec ended at capacity {}), collect all = {eager}",
        lazy.stack.capacity()
    );

    // (b) Rust has no 'continued iteration' panic: a pull iterator is only
    // driven by whoever calls next(), and a visitor's Break is a value.
    println!("\n(b) a push visitor that ignores the stop signal, then Break after 3");
    {
        let mut w = Walker::default();
        let mut body = 0;
        w.bad_walk(&big, &mut |_| {
            body += 1;
            if body >= 3 {
                ControlFlow::Break(())
            } else {
                ControlFlow::Continue(())
            }
        });
        println!(
            "  no panic: the body ran {body} times (not 3) and the walk made {} calls (2n+1 = {})",
            w.calls,
            2 * N + 1
        );
    }

    // (c) Rust has one iterator style, the pull one: take 10, pause, carry on or drop it.
    // While `it` is alive it borrows the tree: `root.as_mut().unwrap().val = 99;`
    // between two next() calls is a compile error (E0502, cannot borrow as mutable
    // because it is also borrowed as immutable; compiled in a scratch program).
    println!("\n(c) the Iterator is already the pull style: take 10 values, pause, take 10 more");
    {
        let mut it = InOrder::new(&big);
        let sum: i64 = it.by_ref().take(10).sum();
        let before = it.calls;
        let sum2: i64 = it.by_ref().take(10).sum();
        println!(
            "  10 values summed to {sum}, {before} calls; paused, then 10 more summed to {sum2}, {} calls in all",
            it.calls
        );
        println!("  dropping the iterator is the whole of 'stop': there is no stop() to call");
    }

    // (d) same fringe.
    println!("\n(d) same fringe: equal sequences from different shapes");
    let t1 = Some(hand_built());
    let t2 = Some(rotate_left(hand_built()));
    println!("tree 1:");
    draw(&t1, 0);
    println!("tree 2 (tree 1 rotated left at the root: another shape, same in-order):");
    draw(&t2, 0);
    let (mut a1, mut a2) = (InOrder::new(&t1), InOrder::new(&t2));
    let (eq, len) = same_fringe(&mut a1, &mut a2);
    println!(
        "same fringe: equal={eq} over {len} values; calls {} and {} (2n+1 = {})",
        a1.calls,
        a2.calls,
        2 * 9 + 1
    );
    // The std adapter says the same. Iterator::eq pulls both sides in lockstep
    // and stops at the first mismatch (read in core/src/iter/traits/iterator.rs);
    // by_ref() lends the iterators so their counters survive the call.
    let (mut b1, mut b2) = (InOrder::new(&t1), InOrder::new(&t2));
    println!(
        "Iterator::eq agrees: {}, with calls {} and {}",
        b1.by_ref().eq(b2.by_ref()),
        b1.calls,
        b2.calls
    );

    // Tree B has another shape; its in-order sequence is A's with the value at position 5 changed.
    let mut other = random_split(N, &mut r);
    rank = 0;
    label_in_order(&mut other, &mut rank);
    let mut pos = 0;
    set_at(&mut other, &mut pos, 5, -1);
    println!(
        "\ntree B: another randomSplit({N}), height {}, left spine {}, same values as A except position 5",
        height(&other),
        left_spine(&other)
    );

    let (mut wa, mut wb) = (InOrder::new(&big), InOrder::new(&other));
    let (eq, pos) = same_fringe(&mut wa, &mut wb);
    println!(
        "lazy:  equal={eq}, first difference at position {pos}; calls A {}, B {} (spines {} and {})",
        wa.calls,
        wb.calls,
        spine,
        left_spine(&other)
    );
    // the same answer from the adapters: eq, and zip + position for the index
    let (mut qa, mut qb) = (InOrder::new(&big), InOrder::new(&other));
    println!(
        "       Iterator::eq: {}, with calls {} and {}",
        qa.by_ref().eq(qb.by_ref()),
        qa.calls,
        qb.calls
    );
    let (mut za, mut zb) = (InOrder::new(&big), InOrder::new(&other));
    let first_diff = za.by_ref().zip(zb.by_ref()).position(|(x, y)| x != y);
    println!(
        "       zip().position(): {:?}, with calls {} and {}",
        first_diff, za.calls, zb.calls
    );
    let (mut ea, mut eb) = (InOrder::new(&big), InOrder::new(&other));
    let (sa, sb): (Vec<i64>, Vec<i64>) = (ea.by_ref().collect(), eb.by_ref().collect());
    println!(
        "eager: equal={}; calls A {}, B {}, {} values built",
        sa == sb,
        ea.calls,
        eb.calls,
        sa.len() + sb.len()
    );
    println!(
        "the lazy test made {:.0}x fewer calls than the eager one",
        (ea.calls + eb.calls) as f64 / (wa.calls + wb.calls) as f64
    );

    // (e) what each style costs, measured.
    println!("\n(e) ns per value, full walk of tree A (best of 3)");
    let big = black_box(big);
    let best = best_ns_per_value(
        N,
        20,
        &mut [
            &mut || {
                // push: recursive visitor
                let mut s = 0i64;
                let done = Walker::default().walk(&big, 0, &mut |v| {
                    s += v;
                    ControlFlow::Continue(())
                });
                assert!(done.is_continue());
                s
            },
            &mut || InOrder::new(&big).sum(), // pull: the Iterator, no collect
            &mut || InOrder::new(&big).collect::<Vec<i64>>().iter().sum(), // eager: Vec, then loop
        ],
    );
    let names = ["visitor (ControlFlow)", "Iterator (sum)", "eager Vec"];
    for (name, ns) in names.iter().zip(&best) {
        println!("  {name:<22} {ns:>7.1} ns/value");
    }
}
```

**Rust sample output:**

```
tree A: randomSplit(100000), height 38, left spine 10 nodes

(a) in-order as an Iterator: take(k)
  stop after      calls   calls-2k   frames    stack     values
           1         11          9       10       10          1
          10         30         10       14       13         10
         100        211         11       21       16        100
         all     200001          1       38       25     100000
value 1 cost 11 calls: the left spine (10 nodes) plus its nil
values 11..100 cost 2.01 calls each
the Iterator and the ControlFlow visitor made the same calls on every row: true
the Iterator's stack was never deeper than the visitor's frames: true
eager collect: 200001 calls (2n+1 = 200001), 100000 values, 800000 bytes of Vec<i64>
allocations: stop after 10 = 3 (its stack Vec ended at capacity 16), collect all = 20

(b) a push visitor that ignores the stop signal, then Break after 3
  no panic: the body ran 100000 times (not 3) and the walk made 200001 calls (2n+1 = 200001)

(c) the Iterator is already the pull style: take 10 values, pause, take 10 more
  10 values summed to 45, 30 calls; paused, then 10 more summed to 145, 53 calls in all
  dropping the iterator is the whole of 'stop': there is no stop() to call

(d) same fringe: equal sequences from different shapes
tree 1:
            9
        6
            8
    3
1
        5
            7
    2
        4
tree 2 (tree 1 rotated left at the root: another shape, same in-order):
        9
    6
        8
3
    1
            5
                7
        2
            4
same fringe: equal=true over 9 values; calls 19 and 19 (2n+1 = 19)
Iterator::eq agrees: true, with calls 19 and 19

tree B: another randomSplit(100000), height 40, left spine 11, same values as A except position 5
lazy:  equal=false, first difference at position 5; calls A 21, B 24 (spines 10 and 11)
       Iterator::eq: false, with calls 21 and 24
       zip().position(): Some(5), with calls 21 and 24
eager: equal=false; calls A 200001, B 200001, 200000 values built
the lazy test made 8889x fewer calls than the eager one

(e) ns per value, full walk of tree A (best of 3)
  visitor (ControlFlow)      6.0 ns/value
  Iterator (sum)             5.7 ns/value
  eager Vec                  7.3 ns/value
```

> **Go vs Rust:** The counts are the same in both languages (11, 30, 211 and 200,001 calls; 21 and 24 for a difference at position 5), so push or pull, stopping early costs the same; Go's `iter.Pull` is a coroutine and measured about 7x a range loop in the Go run, while Rust's single `Iterator` style ran at the speed of the recursive `ControlFlow` visitor (5.7-6.0 ns per value over five runs). The pull iterator's explicit stack stayed shallower than the recursion it replaces (25 against 38 at the peak, because a visited node leaves the stack before its right subtree is walked) but it is a heap `Vec` (3 allocation events after 10 values, capacity 16 at the end) where the Go walk allocated none. Misuse is caught at different times: Go panics at run time when a push iterator ignores `yield`, Rust makes the same bug a compile-time `unused_must_use` warning on `ControlFlow` (an error under `clippy -D warnings`, which is how this program was checked), and changing the tree while an `InOrder` is alive is error E0502 (compiled in a scratch program).

**Complexity:** eager collect Θ(n) time and Θ(n) space; stopping after k values Θ(h + k) time and Θ(h) stack, about 2 calls per value after the descent (11, 30 and 211 calls for k = 1, 10, 100 on a 100,000-node tree whose left spine is 10 nodes; 200,001 = 2n + 1 for all); same fringe Θ(h + position of the first difference): 21 + 24 calls to find a difference at position 5, against 400,002 to build both slices. Pull drives the very same walk (30 calls for 10 values, as with `break`) but pays a coroutine switch per value: about 7x a range loop here (6.9-7.4x over eight runs, 48-53 ns against 7 ns per value; absolute times depend on the machine).

---

## 16. Capstone: a `Tree` toolkit, proved and measured

`🔴 hard` · *Applied mini-build*

The lesson in one program, proved by an oracle that has nothing in common with it. A tree *is* the set of its `(path, value)`
pairs, where a path is the string of `L` and `R` turns from the root, and every property has a slow, obviously correct
definition over those strings: pre-order is the paths sorted with a prefix before its extensions, post-order puts the
extensions first, in-order compares two paths by the next turn of the longer one, level order sorts by (length, path),
height is the longest path plus one, the diameter is the largest `len(a) + len(b) - 2·lcp(a, b)`, the LCA is the longest
common prefix, and a path sum is a sum over the prefixes between two paths. The toolkit (recursive and explicit-stack
pre/in/post, level order, Morris, height, depth, one-pass diameter, max path sum, downward paths with a prefix map, invert,
LCA with a found check, serialize and deserialize with errors, rebuild from pre + in with a value-to-index map) is run
against that oracle on 5,000 random trees and on every one of the 626 trees of up to seven nodes, then pointed at four
deliberately broken functions to show the oracle can fail, then run on a million-node chain, and finally timed. Every
operation is Θ(n); what differs between them is what they must remember. This is the longest program in the lesson, on
purpose: it is the lesson's code in one place.

**Steps:**

1. Write the generators (`randomSplit` from a seeded PCG, shuffled and small-valued labels, every shape of up to seven nodes)
   and the toolkit, and check the generator against its known heights (6, 20 and 41) before anything else.
2. Write the oracle as sorts and pair loops over path strings, and run every function against it on 5,626 trees, counting
   checks and mismatches per group and asserting 2n + 1 calls, an untouched tree after Morris, iterative equal to
   recursive, and stack high-water at most n. Then break four functions on purpose, and run the non-recursive ones on a
   chain of a million nodes.
3. Read the counts (zero mismatches, all four broken versions caught, high-water marks per shape) and then the timing
   table (best of 3), and keep apart what is a count (exact, the same every run) from what is a clock.

```go
package main

// Capstone: the whole lesson as one toolkit, proved against an oracle, then
// measured. It is much longer than the other examples (about a thousand lines,
// in three parts) because it is the lesson's code in one place.

import (
	"cmp"
	"errors"
	"fmt"
	"math"
	"math/rand/v2"
	"runtime"
	"slices"
	"strconv"
	"strings"
	"time"
)

// Node is a binary tree node; nil is the empty tree.
type Node struct {
	Val         int
	Left, Right *Node
}

// =====================================================================
// 1. BUILD: generators, then the toolkit. Every function is recursive or
// iterative on purpose; the oracle in part 2 shares no code with any of it.
// =====================================================================

// randomSplit draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes. The draw happens before recursing, so the PCG stream is
// consumed in one fixed order (the Rust twin draws the same numbers).
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelShuffled gives the nodes distinct labels 0..n-1 in a shuffled order
// (hand-written Fisher-Yates, assigned in pre-order).
func labelShuffled(t *Node, r *rand.Rand) {
	a := make([]int, size(t))
	for i := range a {
		a[i] = i
	}
	for i := len(a) - 1; i >= 1; i-- {
		j := r.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
	next := 0
	var assign func(*Node)
	assign = func(n *Node) {
		if n != nil {
			n.Val = a[next]
			next++
			assign(n.Left)
			assign(n.Right)
		}
	}
	assign(t)
}

// labelRange gives every node, in pre-order, a value from [lo, hi].
func labelRange(t *Node, lo, hi int, r *rand.Rand) {
	if t != nil {
		t.Val = lo + r.IntN(hi-lo+1)
		labelRange(t.Left, lo, hi, r)
		labelRange(t.Right, lo, hi, r)
	}
}

func complete(i, n int) *Node { // heap-shaped, Val = index
	if i >= n {
		return nil
	}
	return &Node{i, complete(2*i+1, n), complete(2*i+2, n)}
}

func leftChain(n int) *Node { // built in a loop, Val = depth
	var root *Node
	for i := n - 1; i >= 0; i-- {
		root = &Node{i, root, nil}
	}
	return root
}

func size(t *Node) int {
	if t == nil {
		return 0
	}
	return 1 + size(t.Left) + size(t.Right)
}

func clone(t *Node) *Node {
	if t == nil {
		return nil
	}
	return &Node{t.Val, clone(t.Left), clone(t.Right)}
}

func same(a, b *Node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Val == b.Val && same(a.Left, b.Left) && same(a.Right, b.Right)
}

// shapes lists every binary tree shape with n nodes, each a fresh tree.
func shapes(n int) []*Node {
	if n == 0 {
		return []*Node{nil}
	}
	var out []*Node
	for k := 0; k < n; k++ {
		for _, l := range shapes(k) {
			for _, r := range shapes(n - 1 - k) {
				out = append(out, &Node{0, clone(l), clone(r)})
			}
		}
	}
	return out
}

// ---- the three depth-first orders, recursive: each returns its call count ----

func PreRec(n *Node, visit func(int)) int {
	if n == nil {
		return 1
	}
	visit(n.Val)
	return 1 + PreRec(n.Left, visit) + PreRec(n.Right, visit)
}

func InRec(n *Node, visit func(int)) int {
	if n == nil {
		return 1
	}
	c := InRec(n.Left, visit)
	visit(n.Val)
	return 1 + c + InRec(n.Right, visit)
}

func PostRec(n *Node, visit func(int)) int {
	if n == nil {
		return 1
	}
	c := PostRec(n.Left, visit) + PostRec(n.Right, visit)
	visit(n.Val)
	return 1 + c
}

// ---- explicit stacks: each returns its high-water mark (most entries at once) ----

func PreStack(t *Node, visit func(int)) (high int) {
	if t == nil {
		return 0
	}
	st := []*Node{t}
	high = 1
	for len(st) > 0 {
		n := st[len(st)-1]
		st = st[:len(st)-1]
		visit(n.Val)
		if n.Right != nil {
			st = append(st, n.Right)
		}
		if n.Left != nil {
			st = append(st, n.Left)
		}
		high = max(high, len(st))
	}
	return high
}

func InStack(t *Node, visit func(int)) (high int) {
	var st []*Node
	for cur := t; cur != nil || len(st) > 0; {
		for ; cur != nil; cur = cur.Left {
			st = append(st, cur)
		}
		high = max(high, len(st))
		cur = st[len(st)-1]
		st = st[:len(st)-1]
		visit(cur.Val)
		cur = cur.Right
	}
	return high
}

// PostFrames is post-order as a literal simulation of the call stack: a frame
// is (node, how many of its two children have been started).
func PostFrames(t *Node, visit func(int)) (high int) {
	type frame struct {
		n     *Node
		state int
	}
	if t == nil {
		return 0
	}
	st := []frame{{t, 0}}
	high = 1
	for len(st) > 0 {
		f := &st[len(st)-1]
		switch f.state {
		case 0:
			f.state = 1
			if f.n.Left != nil {
				st = append(st, frame{f.n.Left, 0}) // f is stale after this append
			}
		case 1:
			f.state = 2
			if f.n.Right != nil {
				st = append(st, frame{f.n.Right, 0})
			}
		default:
			visit(f.n.Val)
			st = st[:len(st)-1]
		}
		high = max(high, len(st))
	}
	return high
}

// LevelOrder is BFS with the n := len(q) snapshot; it reports each value with
// its depth and returns the queue's high-water mark.
func LevelOrder(t *Node, visit func(depth, val int)) (high int) {
	if t == nil {
		return 0
	}
	q := []*Node{t}
	high = 1
	for depth := 0; len(q) > 0; depth++ {
		for i := len(q); i > 0; i-- { // the snapshot: this layer only
			n := q[0]
			q = q[1:]
			visit(depth, n.Val)
			if n.Left != nil {
				q = append(q, n.Left)
			}
			if n.Right != nil {
				q = append(q, n.Right)
			}
			high = max(high, len(q))
		}
	}
	return high
}

// Morris is in-order with no stack and no recursion: it threads each node's
// in-order predecessor back to it, and removes the thread on the second visit.
func Morris(t *Node, visit func(int)) {
	for cur := t; cur != nil; {
		if cur.Left == nil {
			visit(cur.Val)
			cur = cur.Right
			continue
		}
		p := cur.Left
		for p.Right != nil && p.Right != cur {
			p = p.Right
		}
		if p.Right == nil {
			p.Right = cur // thread
			cur = cur.Left
		} else {
			p.Right = nil // cut the thread
			visit(cur.Val)
			cur = cur.Right
		}
	}
}

// ---- measures: height is nodes (empty tree 0, leaf 1); diameter is edges ----

func Height(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + max(Height(n.Left), Height(n.Right))
}

// Depth is the number of edges from t down to target, or -1 if it is absent.
func Depth(t, target *Node) int {
	if t == nil {
		return -1
	}
	if t == target {
		return 0
	}
	if d := Depth(t.Left, target); d >= 0 {
		return d + 1
	}
	if d := Depth(t.Right, target); d >= 0 {
		return d + 1
	}
	return -1
}

// Diameter is one post-order pass: return the height, record the best bend.
func Diameter(t *Node) int {
	best := 0
	var h func(*Node) int
	h = func(n *Node) int {
		if n == nil {
			return 0
		}
		l, r := h(n.Left), h(n.Right)
		best = max(best, l+r)
		return 1 + max(l, r)
	}
	h(t)
	return best
}

// MaxPathSum returns the best sum over all downward-then-upward paths; the
// running best starts at -infinity, because an all-negative tree has no empty path.
func MaxPathSum(t *Node) (int, bool) {
	if t == nil {
		return 0, false
	}
	best := math.MinInt
	var gain func(*Node) int
	gain = func(n *Node) int {
		if n == nil {
			return 0
		}
		l, r := max(gain(n.Left), 0), max(gain(n.Right), 0)
		best = max(best, n.Val+l+r)
		return n.Val + max(l, r)
	}
	gain(t)
	return best, true
}

// PathsWithSum counts downward paths (ancestor to descendant, at least one
// node) with sum k: a prefix-sum map along the current root path, undone on return.
func PathsWithSum(t *Node, k int) int {
	prefix := map[int]int{0: 1}
	var walk func(*Node, int) int
	walk = func(n *Node, sum int) int {
		if n == nil {
			return 0
		}
		sum += n.Val
		count := prefix[sum-k]
		prefix[sum]++
		count += walk(n.Left, sum) + walk(n.Right, sum)
		prefix[sum]-- // the undo
		return count
	}
	return walk(t, 0)
}

// Invert swaps the children everywhere, in place: assign what the child returns.
func Invert(n *Node) *Node {
	if n == nil {
		return nil
	}
	n.Left, n.Right = Invert(n.Right), Invert(n.Left)
	return n
}

// LCA returns the lowest common ancestor of p and q, and false when either is
// not in the tree (the plain one-pass version would answer p in that case).
func LCA(root, p, q *Node) (*Node, bool) {
	var ans *Node
	var walk func(*Node) int // bit 1: p is below, bit 2: q is below
	walk = func(n *Node) int {
		if n == nil {
			return 0
		}
		mask := walk(n.Left) | walk(n.Right)
		if n == p {
			mask |= 1
		}
		if n == q {
			mask |= 2
		}
		if ans == nil && mask == 3 {
			ans = n // the deepest node that sees both
		}
		return mask
	}
	walk(root)
	return ans, ans != nil
}

// Serialize is pre-order with "#" for every nil: exactly 2n+1 tokens.
func Serialize(t *Node) string {
	var sb strings.Builder
	var walk func(*Node)
	walk = func(n *Node) {
		if sb.Len() > 0 {
			sb.WriteByte(',')
		}
		if n == nil {
			sb.WriteByte('#')
			return
		}
		sb.WriteString(strconv.Itoa(n.Val))
		walk(n.Left)
		walk(n.Right)
	}
	walk(t)
	return sb.String()
}

// Deserialize reads the tokens back, with an error for non-numbers, truncated
// or trailing input, and nesting deeper than maxDepth.
func Deserialize(s string, maxDepth int) (*Node, error) {
	toks := strings.Split(s, ",")
	pos := 0
	var read func(depth int) (*Node, error)
	read = func(depth int) (*Node, error) {
		if depth > maxDepth {
			return nil, fmt.Errorf("nested deeper than %d", maxDepth)
		}
		if pos >= len(toks) {
			return nil, errors.New("truncated: ran out of tokens")
		}
		tok := toks[pos]
		pos++
		if tok == "#" {
			return nil, nil
		}
		v, err := strconv.Atoi(tok)
		if err != nil {
			return nil, fmt.Errorf("token %d: not a number: %q", pos-1, tok)
		}
		left, err := read(depth + 1)
		if err != nil {
			return nil, err
		}
		right, err := read(depth + 1)
		if err != nil {
			return nil, err
		}
		return &Node{v, left, right}, nil
	}
	root, err := read(1)
	if err != nil {
		return nil, err
	}
	if pos != len(toks) {
		return nil, fmt.Errorf("trailing: %d unread tokens", len(toks)-pos)
	}
	return root, nil
}

// BuildPreIn rebuilds a tree from its pre-order and in-order (distinct values),
// finding each root in the in-order sequence through a value -> index map.
func BuildPreIn(pre, in []int) (*Node, error) {
	if len(pre) != len(in) {
		return nil, fmt.Errorf("lengths differ: %d and %d", len(pre), len(in))
	}
	pos := make(map[int]int, len(in))
	for i, v := range in {
		if _, dup := pos[v]; dup {
			return nil, fmt.Errorf("duplicate value %d", v)
		}
		pos[v] = i
	}
	next := 0
	var build func(lo, hi int) (*Node, error) // the tree for in[lo:hi]
	build = func(lo, hi int) (*Node, error) {
		if lo >= hi {
			return nil, nil
		}
		v := pre[next]
		next++
		i, ok := pos[v]
		if !ok || i < lo || i >= hi {
			return nil, fmt.Errorf("pre-order value %d does not belong to in-order range [%d,%d)", v, lo, hi)
		}
		left, err := build(lo, i)
		if err != nil {
			return nil, err
		}
		right, err := build(i+1, hi)
		if err != nil {
			return nil, err
		}
		return &Node{v, left, right}, nil
	}
	return build(0, len(in))
}

// =====================================================================
// 2. PROVE: the oracle. A tree IS the set of its (path, value) pairs, where
// a path is a string of L and R from the root. Every property below is
// defined over those strings alone, by the slowest obvious method.
// =====================================================================

type entry struct {
	node *Node
	path string
}

// entries lists every node with its path, in no particular order we rely on.
func entries(t *Node) []entry {
	var out []entry
	var walk func(*Node, string)
	walk = func(n *Node, path string) {
		if n != nil {
			out = append(out, entry{n, path})
			walk(n.Left, path+"L")
			walk(n.Right, path+"R")
		}
	}
	walk(t, "")
	return out
}

func isPrefix(a, b string) bool { return len(a) < len(b) && strings.HasPrefix(b, a) } // proper prefix

// The four orders, each as a comparison of two paths.
func cmpPre(a, b string) int { return strings.Compare(a, b) } // prefix first, then L < R
func cmpPost(a, b string) int { // extensions before their prefix
	switch {
	case isPrefix(a, b):
		return 1
	case isPrefix(b, a):
		return -1
	}
	return strings.Compare(a, b)
}
func cmpIn(a, b string) int { // a node sits between its left and its right subtree
	switch {
	case isPrefix(a, b): // b is below a: in the left subtree it comes first, in the right one after
		if b[len(a)] == 'L' {
			return 1
		}
		return -1
	case isPrefix(b, a):
		if a[len(b)] == 'L' {
			return -1
		}
		return 1
	}
	return strings.Compare(a, b)
}
func cmpLevel(a, b string) int { return cmp.Or(cmp.Compare(len(a), len(b)), strings.Compare(a, b)) }

func orderBy(es []entry, c func(a, b string) int) []int {
	sorted := slices.Clone(es)
	slices.SortFunc(sorted, func(x, y entry) int { return c(x.path, y.path) })
	vals := make([]int, len(sorted))
	for i, e := range sorted {
		vals[i] = e.node.Val
	}
	return vals
}

func lcp(a, b string) int { // length of the longest common prefix
	i := 0
	for i < len(a) && i < len(b) && a[i] == b[i] {
		i++
	}
	return i
}

func oHeight(es []entry) int {
	h := 0
	for _, e := range es {
		h = max(h, len(e.path)+1)
	}
	return h
}

func oDiameter(es []entry) int {
	d := 0
	for _, a := range es {
		for _, b := range es {
			d = max(d, len(a.path)+len(b.path)-2*lcp(a.path, b.path))
		}
	}
	return d
}

// oMaxPathSum: the best sum over every pair of nodes of the nodes on the path between them.
func oMaxPathSum(es []entry) int {
	val := map[string]int{}
	for _, e := range es {
		val[e.path] = e.node.Val
	}
	best := math.MinInt
	for _, a := range es {
		for _, b := range es {
			l, sum := lcp(a.path, b.path), 0
			for j := l; j <= len(a.path); j++ { // a's side, down from the meeting node
				sum += val[a.path[:j]]
			}
			for j := l + 1; j <= len(b.path); j++ { // b's side, below the meeting node
				sum += val[b.path[:j]]
			}
			best = max(best, sum)
		}
	}
	return best
}

// oPaths counts ancestor-descendant pairs (a may equal b) whose nodes sum to k.
func oPaths(es []entry, k int) int {
	val := map[string]int{}
	for _, e := range es {
		val[e.path] = e.node.Val
	}
	count := 0
	for _, b := range es {
		for i := 0; i <= len(b.path); i++ { // i = the ancestor's depth
			sum := 0
			for j := i; j <= len(b.path); j++ {
				sum += val[b.path[:j]]
			}
			if sum == k {
				count++
			}
		}
	}
	return count
}

func mirrorPath(p string) string {
	return strings.Map(func(c rune) rune { return map[rune]rune{'L': 'R', 'R': 'L'}[c] }, p)
}

// fingerprint is the whole tree as a sorted list of "path=value" strings,
// optionally mirrored (L and R swapped).
func fingerprint(es []entry, mirror bool) []string {
	out := make([]string, len(es))
	for i, e := range es {
		p := e.path
		if mirror {
			p = mirrorPath(p)
		}
		out[i] = p + "=" + strconv.Itoa(e.node.Val)
	}
	slices.Sort(out)
	return out
}

// ---- the checking harness ----

// kit holds the four functions that have a classic bug in this lesson, so the
// harness can be pointed at a broken one: an oracle that cannot fail proves nothing.
type kit struct {
	diameter     func(*Node) int
	maxPathSum   func(*Node) (int, bool)
	pathsWithSum func(*Node, int) int
	lca          func(root, p, q *Node) (*Node, bool)
}

var good = kit{Diameter, MaxPathSum, PathsWithSum, LCA}

// The four mutants, one per pitfall of the lesson.
var mutants = []struct {
	name string
	kit  kit
	hit  int // the group that should notice
}{
	{"diameter only through the root", kit{func(t *Node) int {
		if t == nil {
			return 0
		}
		return Height(t.Left) + Height(t.Right)
	}, MaxPathSum, PathsWithSum, LCA}, gShape},
	// starting the running best at 0 is the same as clamping the answer at 0
	{"max path sum starting at 0", kit{Diameter, func(t *Node) (int, bool) {
		best, ok := MaxPathSum(t)
		return max(best, 0), ok
	}, PathsWithSum, LCA}, gSums},
	{"paths with sum k, no undo", kit{Diameter, MaxPathSum, func(t *Node, k int) int {
		prefix := map[int]int{0: 1}
		var walk func(*Node, int) int
		walk = func(n *Node, sum int) int {
			if n == nil {
				return 0
			}
			sum += n.Val
			count := prefix[sum-k]
			prefix[sum]++ // wrong: never removed when the recursion returns
			return count + walk(n.Left, sum) + walk(n.Right, sum)
		}
		return walk(t, 0)
	}, LCA}, gSums},
	{"LCA without the found check", kit{Diameter, MaxPathSum, PathsWithSum, func(root, p, q *Node) (*Node, bool) {
		var walk func(*Node) *Node
		walk = func(n *Node) *Node {
			if n == nil || n == p || n == q {
				return n
			}
			l, r := walk(n.Left), walk(n.Right)
			if l != nil && r != nil {
				return n
			}
			return cmp.Or(l, r)
		}
		ans := walk(root)
		return ans, ans != nil
	}}, gStructure},
}

const (
	gTraversals = iota
	gShape
	gSums
	gStructure
	numGroups
)

var groupNames = [numGroups]string{"traversals", "shape", "sums", "structure"}

type prover struct {
	checks, bad [numGroups]int
	first       string // description of the first failure, if any
}

func (p *prover) check(g int, ok bool, what string) {
	p.checks[g]++
	if !ok {
		p.bad[g]++
		if p.first == "" {
			p.first = groupNames[g] + ": " + what
		}
	}
}

func appender(out *[]int) func(int) { return func(v int) { *out = append(*out, v) } }

// prove runs every function of the toolkit on one tree pair: T has distinct
// labels (for order, LCA, rebuild), S has the same shape and small values
// (for the sums). Both are fresh, so the checks may mutate clones.
func (p *prover) prove(T, S *Node, k kit) {
	es := entries(T)
	n := len(es)
	absent := &Node{Val: -1} // a node that is not in the tree

	// -- traversals: each against its sort of the path strings
	wantPre, wantIn, wantPost := orderBy(es, cmpPre), orderBy(es, cmpIn), orderBy(es, cmpPost)
	var rPre, rIn, rPost, sPre, sIn, sPost, mIn []int
	calls := [3]int{PreRec(T, appender(&rPre)), InRec(T, appender(&rIn)), PostRec(T, appender(&rPost))}
	hw := [3]int{PreStack(T, appender(&sPre)), InStack(T, appender(&sIn)), PostFrames(T, appender(&sPost))}
	recs, stks := [][]int{rPre, rIn, rPost}, [][]int{sPre, sIn, sPost}
	for i, want := range [][]int{wantPre, wantIn, wantPost} {
		p.check(gTraversals, slices.Equal(recs[i], want), "recursive order differs from the oracle")
		p.check(gTraversals, slices.Equal(stks[i], want), "explicit-stack order differs from the oracle")
		p.check(gTraversals, slices.Equal(stks[i], recs[i]), "iterative differs from recursive")
		p.check(gTraversals, calls[i] == 2*n+1, "calls != 2n+1")
		p.check(gTraversals, hw[i] <= n, "stack high-water above n")
	}
	var levels [][]int
	qHigh := LevelOrder(T, func(d, v int) {
		for len(levels) <= d {
			levels = append(levels, nil)
		}
		levels[d] = append(levels[d], v)
	})
	var wantLevels [][]int
	byLevel := slices.Clone(es)
	slices.SortFunc(byLevel, func(x, y entry) int { return cmpLevel(x.path, y.path) })
	for _, e := range byLevel {
		if len(e.path) == len(wantLevels) {
			wantLevels = append(wantLevels, nil)
		}
		wantLevels[len(e.path)] = append(wantLevels[len(e.path)], e.node.Val)
	}
	p.check(gTraversals, slices.EqualFunc(levels, wantLevels, slices.Equal[[]int]), "level order differs")
	p.check(gTraversals, qHigh <= n, "queue high-water above n")
	Morris(T, appender(&mIn))
	p.check(gTraversals, slices.Equal(mIn, wantIn), "Morris differs from the oracle")
	after := entries(T)
	p.check(gTraversals, slices.EqualFunc(es, after, func(a, b entry) bool { return a.node == b.node && a.path == b.path }),
		"Morris left the tree changed")

	// -- shape: height, depth, diameter
	p.check(gShape, Height(T) == oHeight(es), "height")
	p.check(gShape, k.diameter(T) == oDiameter(es), "diameter")
	p.check(gShape, Depth(T, absent) == -1, "depth of an absent node")
	for _, e := range es {
		p.check(gShape, Depth(T, e.node) == len(e.path), "depth")
	}

	// -- sums on S: max path sum, and downward paths with sum k for k in [-5, 5]
	ses := entries(S)
	best, ok := k.maxPathSum(S)
	if n == 0 {
		p.check(gSums, !ok, "max path sum of the empty tree")
	} else {
		p.check(gSums, ok && best == oMaxPathSum(ses), "max path sum")
	}
	for target := -5; target <= 5; target++ {
		p.check(gSums, k.pathsWithSum(S, target) == oPaths(ses, target), "paths with sum k")
	}

	// -- structure: invert, LCA, serialize, rebuild
	c := clone(T)
	Invert(c)
	p.check(gStructure, slices.Equal(fingerprint(entries(c), false), fingerprint(es, true)), "invert is not the mirror")
	Invert(c)
	p.check(gStructure, same(c, T), "invert twice is not the identity")
	for _, a := range es {
		for _, b := range es {
			got, found := k.lca(T, a.node, b.node)
			meet := a.path[:lcp(a.path, b.path)]
			p.check(gStructure, found && entryAt(es, meet) == got, "LCA")
		}
		_, f1 := k.lca(T, a.node, absent)
		_, f2 := k.lca(T, absent, a.node)
		p.check(gStructure, !f1 && !f2, "LCA with an absent node must report not found")
	}
	text := Serialize(T)
	p.check(gStructure, strings.Count(text, ",")+1 == 2*n+1, "serialize token count")
	back, err := Deserialize(text, 1000)
	p.check(gStructure, err == nil && same(back, T), "deserialize round trip")
	_, e1 := Deserialize(text+",5", 1000)
	_, e2 := Deserialize(text[:max(strings.LastIndex(text, ","), 0)], 1000)
	_, e3 := Deserialize("x,"+text, 1000)
	p.check(gStructure, e1 != nil && e2 != nil && e3 != nil, "deserialize accepted bad input")
	rebuilt, err := BuildPreIn(wantPre, wantIn)
	p.check(gStructure, err == nil && same(rebuilt, T), "rebuild from pre + in")
	_, e4 := BuildPreIn(wantPre, wantIn[:max(n-1, 0)])
	p.check(gStructure, n == 0 || e4 != nil, "rebuild accepted a length mismatch")
	if n > 0 {
		bad := slices.Clone(wantPre)
		bad[0] = -7
		_, e5 := BuildPreIn(bad, wantIn)
		p.check(gStructure, e5 != nil, "rebuild accepted a foreign root")
	}
}

func entryAt(es []entry, path string) *Node {
	for _, e := range es {
		if e.path == path {
			return e.node
		}
	}
	return nil
}

// =====================================================================
// 3. MEASURE
// =====================================================================

var sink int

// bestNsPerNode times each function 3 rounds (3 runs per round, interleaved)
// and keeps the best round.
func bestNsPerNode(n int, fs []func()) []float64 {
	best := make([]float64, len(fs))
	for round := 0; round < 3; round++ {
		for i, f := range fs {
			t := time.Now()
			for range 3 {
				f()
			}
			ns := float64(time.Since(t).Nanoseconds()) / float64(3*n)
			if round == 0 || ns < best[i] {
				best[i] = ns
			}
		}
	}
	return best
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	// ---- 1. build ----
	fmt.Println("1. BUILD: the generators")
	// The generator check: fresh PCG(1,2), randomSplit of 15, 1000 and 100000 nodes in that order.
	{
		a := rand.New(rand.NewPCG(1, 2))
		var hs []int
		for _, n := range []int{15, 1000, 100000} {
			hs = append(hs, Height(randomSplit(n, a)))
		}
		fmt.Printf("  randomSplit(15), (1000), (100000) from one fresh PCG(1,2): heights %v\n", hs)
		var counts []int
		total := 0
		for n := 0; n <= 7; n++ {
			counts = append(counts, len(shapes(n)))
			total += counts[n]
		}
		fmt.Printf("  shapes with n = 0..7 nodes: %v, %d in all (Catalan numbers)\n", counts, total)
	}

	// ---- 2. prove ----
	fmt.Println("\n2. PROVE: every function against the path-string oracle")
	var p prover
	const randomTrees = 5000
	nodes, empty, allNeg := 0, 0, 0
	shown := false
	for i := 0; i < randomTrees; i++ {
		n := r.IntN(41)
		T := randomSplit(n, r)
		labelShuffled(T, r)
		S := clone(T)
		labelRange(S, -3, 3, r)
		if n == 0 {
			empty++
		} else if slices.Max(orderBy(entries(S), cmpPre)) <= 0 {
			allNeg++
		}
		nodes += n
		if !shown && n == 7 { // show the oracle's view of one tree, with both labellings
			shown = true
			es, ses := entries(T), entries(S)
			slices.SortFunc(es, func(x, y entry) int { return cmpPre(x.path, y.path) })
			slices.SortFunc(ses, func(x, y entry) int { return cmpPre(x.path, y.path) })
			fmt.Printf("  tree #%d, n = %d, as path=value in pre-order ('.' is the root):\n", i, n)
			for _, name := range []string{"distinct labels (order, LCA, rebuild)", "small labels, same shape (sums)"} {
				fmt.Printf("    %-38s", name)
				for j, e := range es {
					v := e.node.Val
					if name[0] == 's' {
						v = ses[j].node.Val
					}
					fmt.Printf(" %s=%d", cmp.Or(e.path, "."), v)
				}
				fmt.Println()
			}
		}
		p.prove(T, S, good)
	}
	randomChecks := p.checks
	shapeTrees := 0
	for n := 0; n <= 7; n++ {
		for _, T := range shapes(n) {
			labelShuffled(T, r)
			S := clone(T)
			labelRange(S, -3, 3, r)
			p.prove(T, S, good)
			shapeTrees++
		}
	}
	fmt.Printf("  %d random trees (n uniform in [0,40]: %d nodes in all, %d empty, %d with every value <= 0) + %d shapes (every tree of n <= 7)\n",
		randomTrees, nodes, empty, allNeg, shapeTrees)
	allChecks := 0
	for g := 0; g < numGroups; g++ {
		fmt.Printf("  %-11s %d mismatches over %d trees, %d checks (%d on random trees, %d on shapes)\n",
			groupNames[g], p.bad[g], randomTrees+shapeTrees, p.checks[g], randomChecks[g], p.checks[g]-randomChecks[g])
		allChecks += p.checks[g]
	}
	fmt.Printf("  total: %d checks, first failure: %q\n", allChecks, p.first)

	// the error paths, with their messages
	fmt.Println("  Deserialize on text you did not write:")
	for _, s := range []string{"1,#,#", "1,#", "1,#,#,#", "1,x,#,#", "1,2,3,4,#,#"} {
		res := "ok"
		if _, err := Deserialize(s, 3); err != nil {
			res = err.Error()
		}
		fmt.Printf("    %-14q -> %s\n", s, res)
	}
	_, err := BuildPreIn([]int{1, 2, 3}, []int{2, 1})
	_, err2 := BuildPreIn([]int{9, 2}, []int{2, 1})
	_, err3 := BuildPreIn([]int{1, 1}, []int{1, 1})
	fmt.Printf("  BuildPreIn: %v | %v | %v\n", err, err2, err3)

	// the oracle must be able to fail: point it at each broken function
	fmt.Println("  the same checks on four broken functions (1000 fresh random trees + every shape):")
	for _, m := range mutants {
		var mp prover
		mr := rand.New(rand.NewPCG(3, 4))
		for i := 0; i < 1000; i++ {
			T := randomSplit(mr.IntN(41), mr)
			labelShuffled(T, mr)
			S := clone(T)
			labelRange(S, -3, 3, mr)
			mp.prove(T, S, m.kit)
		}
		for n := 0; n <= 7; n++ {
			for _, T := range shapes(n) {
				labelShuffled(T, mr)
				S := clone(T)
				labelRange(S, -3, 3, mr)
				mp.prove(T, S, m.kit)
			}
		}
		fmt.Printf("    %-32s %6d mismatches in %s; first: %s\n", m.name, mp.bad[m.hit], groupNames[m.hit], mp.first)
	}

	// ---- deep: a 1,000,000-node chain, only the functions that do not recurse ----
	const deepN = 1_000_000
	fmt.Printf("\n  deep: a left chain of %d nodes, built in a loop\n", deepN)
	chain := leftChain(deepN)
	// each function must produce the chain's values in the right order (no slice is built)
	deep := func(name string, want func(i int) int, run func(visit func(int)) string) {
		i, bad := 0, 0
		note := run(func(v int) {
			if v != want(i) {
				bad++
			}
			i++
		})
		fmt.Printf("    %-11s visited %d, wrong %d, %s\n", name, i, bad, note)
	}
	up := func(i int) int { return i }
	down := func(i int) int { return deepN - 1 - i }
	deep("PreStack", up, func(v func(int)) string { return fmt.Sprint("stack high-water ", PreStack(chain, v)) })
	deep("InStack", down, func(v func(int)) string { return fmt.Sprint("stack high-water ", InStack(chain, v)) })
	deep("PostFrames", down, func(v func(int)) string { return fmt.Sprint("stack high-water ", PostFrames(chain, v)) })
	deep("LevelOrder", up, func(v func(int)) string {
		return fmt.Sprint("queue high-water ", LevelOrder(chain, func(d, val int) {
			if d != val { // on this chain the depth of a node is its value
				val = -1
			}
			v(val)
		}))
	})
	deep("Morris", down, func(v func(int)) string { Morris(chain, v); return "no stack at all" })
	spine, stray := 0, 0
	for q := chain; q != nil; q = q.Left {
		spine++
		if q.Right != nil {
			stray++
		}
	}
	fmt.Printf("    after Morris the chain is intact: %d nodes on the left spine, %d stray right links\n", spine, stray)
	fmt.Printf("    Height (recursive, %d frames deep) = %d: Go grows the goroutine stack to fit\n", deepN, Height(chain))

	// ---- 3. measure ----
	fmt.Println("\n3. MEASURE: ns per node, best of 3, two shapes of about a million nodes")
	const nRandom, nComplete = 1_000_000, 1<<20 - 1
	rnd := randomSplit(nRandom, rand.New(rand.NewPCG(1, 2)))
	cpl := complete(0, nComplete)
	type shape struct {
		name string
		root *Node
		n    int
	}
	shs := []shape{{"randomSplit", rnd, nRandom}, {"complete", cpl, nComplete}}
	rows := []string{"recursive pre-order", "explicit pre-order", "recursive in-order", "explicit in-order",
		"Morris in-order", "level order", "post-order frame stack"}
	table := make([][]float64, len(shs))
	for si, s := range shs {
		sum := 0
		add := func(v int) { sum += v }
		t := s.root
		table[si] = bestNsPerNode(s.n, []func(){
			func() { sink += PreRec(t, add) },
			func() { sink += PreStack(t, add) },
			func() { sink += InRec(t, add) },
			func() { sink += InStack(t, add) },
			func() { Morris(t, add) },
			func() { sink += LevelOrder(t, func(d, v int) { sum += v }) },
			func() { sink += PostFrames(t, add) },
		})
		sink += sum
	}
	// the build: construct, time the construction only, let the garbage go
	builds := make([]float64, len(shs))
	for si, s := range shs {
		for round := 0; round < 3; round++ {
			runtime.GC()
			t0 := time.Now()
			var tree *Node
			if si == 0 {
				tree = randomSplit(s.n, rand.New(rand.NewPCG(1, 2)))
			} else {
				tree = complete(0, s.n)
			}
			ns := float64(time.Since(t0).Nanoseconds()) / float64(s.n)
			sink += tree.Val
			if round == 0 || ns < builds[si] {
				builds[si] = ns
			}
		}
	}
	fmt.Printf("  %-24s %12s(%d) %12s(%d)\n", "ns per node", shs[0].name, nRandom, shs[1].name, nComplete)
	for i, name := range rows {
		fmt.Printf("  %-24s %18.1f %18.1f\n", name, table[0][i], table[1][i])
	}
	fmt.Printf("  %-24s %18.1f %18.1f\n", "build the tree", builds[0], builds[1])
	cheaper := 0
	for i := range rows {
		if table[1][i] < table[0][i] {
			cheaper++
		}
	}
	fmt.Printf("  cheaper per node on complete than on randomSplit: %d of %d traversals; the build: %v\n",
		cheaper, len(rows), builds[1] < builds[0])
	fmt.Printf("  Morris / explicit-stack in-order, per node: %.1fx on randomSplit, %.1fx on complete\n",
		table[0][4]/table[0][3], table[1][4]/table[1][3])

	// the counts that explain who needs how much memory
	fmt.Println("\n  high-water marks (entries held at once; counts, identical in every run)")
	fmt.Printf("  %-24s %12s %12s %12s\n", "", "randomSplit", "complete", "left chain")
	all := append(shs, shape{"left chain", chain, deepN})
	row := func(name string, f func(*Node) int) {
		fmt.Printf("  %-24s", name)
		for _, s := range all {
			fmt.Printf(" %12d", f(s.root))
		}
		fmt.Println()
	}
	noop := func(int) {}
	row("height", Height)
	row("pre-order stack", func(t *Node) int { return PreStack(t, noop) })
	row("in-order stack", func(t *Node) int { return InStack(t, noop) })
	row("post-order frame stack", func(t *Node) int { return PostFrames(t, noop) })
	row("level-order queue", func(t *Node) int { return LevelOrder(t, func(int, int) {}) })
}
```

**Sample output:**

```
1. BUILD: the generators
  randomSplit(15), (1000), (100000) from one fresh PCG(1,2): heights [6 20 41]
  shapes with n = 0..7 nodes: [1 1 2 5 14 42 132 429], 626 in all (Catalan numbers)

2. PROVE: every function against the path-string oracle
  tree #75, n = 7, as path=value in pre-order ('.' is the root):
    distinct labels (order, LCA, rebuild)  .=2 R=3 RL=5 RLL=6 RLR=0 RLRL=1 RR=4
    small labels, same shape (sums)        .=3 R=3 RL=1 RLL=-1 RLR=-1 RLRL=3 RR=-3
  5000 random trees (n uniform in [0,40]: 100375 nodes in all, 109 empty, 157 with every value <= 0) + 626 shapes (every tree of n <= 7)
  traversals  0 mismatches over 5626 trees, 106894 checks (95000 on random trees, 11894 on shapes)
  shape       0 mismatches over 5626 trees, 121334 checks (115375 on random trees, 5959 on shapes)
  sums        0 mismatches over 5626 trees, 67512 checks (60000 on random trees, 7512 on shapes)
  structure   0 mismatches over 5626 trees, 2888544 checks (2852355 on random trees, 36189 on shapes)
  total: 3184284 checks, first failure: ""
  Deserialize on text you did not write:
    "1,#,#"        -> ok
    "1,#"          -> truncated: ran out of tokens
    "1,#,#,#"      -> trailing: 1 unread tokens
    "1,x,#,#"      -> token 1: not a number: "x"
    "1,2,3,4,#,#"  -> nested deeper than 3
  BuildPreIn: lengths differ: 3 and 2 | pre-order value 9 does not belong to in-order range [0,2) | duplicate value 1
  the same checks on four broken functions (1000 fresh random trees + every shape):
    diameter only through the root      243 mismatches in shape; first: shape: diameter
    max path sum starting at 0           28 mismatches in sums; first: sums: max path sum
    paths with sum k, no undo         10058 mismatches in sums; first: sums: paths with sum k
    LCA without the found check       24132 mismatches in structure; first: structure: LCA with an absent node must report not found

  deep: a left chain of 1000000 nodes, built in a loop
    PreStack    visited 1000000, wrong 0, stack high-water 1
    InStack     visited 1000000, wrong 0, stack high-water 1000000
    PostFrames  visited 1000000, wrong 0, stack high-water 1000000
    LevelOrder  visited 1000000, wrong 0, queue high-water 1
    Morris      visited 1000000, wrong 0, no stack at all
    after Morris the chain is intact: 1000000 nodes on the left spine, 0 stray right links
    Height (recursive, 1000000 frames deep) = 1000000: Go grows the goroutine stack to fit

3. MEASURE: ns per node, best of 3, two shapes of about a million nodes
  ns per node               randomSplit(1000000)     complete(1048575)
  recursive pre-order                     7.2                2.4
  explicit pre-order                      6.0                3.1
  recursive in-order                      6.6                2.3
  explicit in-order                       6.6                2.8
  Morris in-order                         8.2                2.6
  level order                            15.5               10.5
  post-order frame stack                  9.8                4.6
  build the tree                         14.4                6.8
  cheaper per node on complete than on randomSplit: 7 of 7 traversals; the build: true
  Morris / explicit-stack in-order, per node: 1.3x on randomSplit, 0.9x on complete

  high-water marks (entries held at once; counts, identical in every run)
                            randomSplit     complete   left chain
  height                             48           20      1000000
  pre-order stack                    31           20            1
  in-order stack                     32           20      1000000
  post-order frame stack             48           20      1000000
  level-order queue               84453       524288            1
```

**Rust:**

```rust
// Capstone: the whole lesson as one toolkit, proved against an oracle, then
// measured. It is much longer than the other examples (well over a thousand
// lines, in three parts) because it is the lesson's code in one place.
use std::cell::{Cell, RefCell};
use std::cmp::Ordering;
use std::collections::{HashMap, VecDeque};
use std::hint::black_box;
use std::ptr;
use std::thread;
use std::time::Instant;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so both programs build the same trees from the same seeds.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
}

// =====================================================================
// 1. BUILD: generators, then the toolkit. The oracle in part 2 shares no
// code with any of it.
// =====================================================================

// A binary tree node; None is the empty tree. Clone and PartialEq are derived
// (structural, recursive: as deep as the tree, so they are used on small trees only).
#[derive(Clone, PartialEq)]
struct Node {
    val: i64,
    left: Link,
    right: Link,
}

type Link = Option<Box<Node>>;

// Iterative Drop: the default drop glue recurses once per level, which the
// 1,000,000-node chain below would turn into a stack overflow.
impl Drop for Node {
    fn drop(&mut self) {
        let mut stack: Vec<Box<Node>> = Vec::new();
        stack.extend(self.left.take());
        stack.extend(self.right.take());
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
        }
    }
}

fn mk(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

// random_split draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes. The draw happens before recursing, so the PCG stream is
// consumed in the same order as in the Go program.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    mk(0, left, right)
}

// label_shuffled gives the nodes distinct labels 0..n-1 in a shuffled order
// (hand-written Fisher-Yates, assigned in pre-order).
fn label_shuffled(t: &mut Link, r: &mut Pcg) {
    let mut a: Vec<i64> = (0..size(t) as i64).collect();
    for i in (1..a.len()).rev() {
        let j = r.int_n(i + 1);
        a.swap(i, j);
    }
    fn assign(n: &mut Link, a: &[i64], next: &mut usize) {
        if let Some(b) = n {
            b.val = a[*next];
            *next += 1;
            assign(&mut b.left, a, next);
            assign(&mut b.right, a, next);
        }
    }
    assign(t, &a, &mut 0);
}

// label_range gives every node, in pre-order, a value from [lo, hi].
fn label_range(t: &mut Link, lo: i64, hi: i64, r: &mut Pcg) {
    if let Some(n) = t {
        n.val = lo + r.int_n((hi - lo + 1) as usize) as i64;
        label_range(&mut n.left, lo, hi, r);
        label_range(&mut n.right, lo, hi, r);
    }
}

fn complete(i: usize, n: usize) -> Link {
    // heap-shaped, val = index
    if i >= n {
        return None;
    }
    mk(i as i64, complete(2 * i + 1, n), complete(2 * i + 2, n))
}

fn left_chain(n: usize) -> Link {
    // built in a loop, val = depth
    let mut root: Link = None;
    for i in (0..n).rev() {
        root = mk(i as i64, root, None);
    }
    root
}

fn size(t: &Link) -> usize {
    t.as_ref().map_or(0, |n| 1 + size(&n.left) + size(&n.right))
}

// shapes lists every binary tree shape with n nodes, each a fresh tree.
fn shapes(n: usize) -> Vec<Link> {
    if n == 0 {
        return vec![None];
    }
    let mut out = Vec::new();
    for k in 0..n {
        for l in shapes(k) {
            for r in shapes(n - 1 - k) {
                out.push(mk(0, l.clone(), r.clone()));
            }
        }
    }
    out
}

// ---- the three depth-first orders, recursive: each returns its call count ----

fn pre_rec(n: &Link, visit: &mut impl FnMut(i64)) -> usize {
    match n {
        None => 1,
        Some(b) => {
            visit(b.val);
            1 + pre_rec(&b.left, visit) + pre_rec(&b.right, visit)
        }
    }
}

fn in_rec(n: &Link, visit: &mut impl FnMut(i64)) -> usize {
    match n {
        None => 1,
        Some(b) => {
            let c = in_rec(&b.left, visit);
            visit(b.val);
            1 + c + in_rec(&b.right, visit)
        }
    }
}

fn post_rec(n: &Link, visit: &mut impl FnMut(i64)) -> usize {
    match n {
        None => 1,
        Some(b) => {
            let c = post_rec(&b.left, visit) + post_rec(&b.right, visit);
            visit(b.val);
            1 + c
        }
    }
}

// ---- explicit stacks: each returns its high-water mark (most entries at once) ----

fn pre_stack(t: &Link, visit: &mut impl FnMut(i64)) -> usize {
    let Some(root) = t.as_deref() else { return 0 };
    let mut st: Vec<&Node> = vec![root];
    let mut high = 1;
    while let Some(n) = st.pop() {
        visit(n.val);
        st.extend(n.right.as_deref());
        st.extend(n.left.as_deref());
        high = high.max(st.len());
    }
    high
}

fn in_stack(t: &Link, visit: &mut impl FnMut(i64)) -> usize {
    let mut st: Vec<&Node> = Vec::new();
    let mut high = 0;
    let mut cur = t.as_deref();
    while cur.is_some() || !st.is_empty() {
        while let Some(n) = cur {
            st.push(n);
            cur = n.left.as_deref();
        }
        high = high.max(st.len());
        let n = st
            .pop()
            .expect("the loop condition keeps the stack non-empty here");
        visit(n.val);
        cur = n.right.as_deref();
    }
    high
}

// post_frames is post-order as a literal simulation of the call stack: a frame
// is (node, how many of its two children have been started).
fn post_frames(t: &Link, visit: &mut impl FnMut(i64)) -> usize {
    let Some(root) = t.as_deref() else { return 0 };
    let mut st: Vec<(&Node, u8)> = vec![(root, 0)];
    let mut high = 1;
    while let Some(top) = st.last_mut() {
        let n = top.0;
        match top.1 {
            0 => {
                top.1 = 1;
                if let Some(l) = n.left.as_deref() {
                    st.push((l, 0));
                }
            }
            1 => {
                top.1 = 2;
                if let Some(r) = n.right.as_deref() {
                    st.push((r, 0));
                }
            }
            _ => {
                visit(n.val);
                st.pop();
            }
        }
        high = high.max(st.len());
    }
    high
}

// level_order is BFS with the len() snapshot; it reports each value with its
// depth and returns the queue's high-water mark.
fn level_order(t: &Link, visit: &mut impl FnMut(usize, i64)) -> usize {
    let Some(root) = t.as_deref() else { return 0 };
    let mut q: VecDeque<&Node> = VecDeque::from([root]);
    let mut high = 1;
    let mut depth = 0;
    while !q.is_empty() {
        for _ in 0..q.len() {
            // the snapshot: this layer only
            let n = q.pop_front().expect("the snapshot counted this node");
            visit(depth, n.val);
            q.extend(n.left.as_deref());
            q.extend(n.right.as_deref());
            high = high.max(q.len());
        }
        depth += 1;
    }
    high
}

// ---- Morris needs a second link to a node its parent already owns. A Box is
// the one owner, so Morris cannot run on `Option<Box<Node>>` (example 12 compiled
// the attempts and listed the errors). It runs on an ARENA copy of the tree:
// a link is a u32 index, and a thread is just another index. ----

const NIL: u32 = u32::MAX; // the empty tree: an index no node has

#[derive(Clone, Copy, PartialEq)]
struct N {
    val: i64,
    left: u32,
    right: u32,
}

struct Arena {
    nodes: Vec<N>,
    root: u32,
}

impl Arena {
    // from_box copies a Box tree into an arena with an explicit stack (the
    // chain below is a million deep, so no recursion). Ids are in pre-order.
    fn from_box(t: &Link) -> Arena {
        let mut nodes: Vec<N> = Vec::new();
        let mut root = NIL;
        let mut work: Vec<(&Node, u32, bool)> = Vec::new(); // (node, parent id, is it the left child)
        if let Some(r) = t.as_deref() {
            work.push((r, NIL, true));
        }
        while let Some((n, parent, is_left)) = work.pop() {
            let id = nodes.len() as u32;
            nodes.push(N {
                val: n.val,
                left: NIL,
                right: NIL,
            });
            if parent == NIL {
                root = id;
            } else if is_left {
                nodes[parent as usize].left = id;
            } else {
                nodes[parent as usize].right = id;
            }
            work.extend(n.right.as_deref().map(|c| (c, id, false)));
            work.extend(n.left.as_deref().map(|c| (c, id, true)));
        }
        Arena { nodes, root }
    }

    // morris is in-order with no stack and no recursion.
    fn morris(&mut self, visit: &mut impl FnMut(i64)) {
        let mut cur = self.root;
        while cur != NIL {
            let N { val, left, right } = self.nodes[cur as usize];
            if left == NIL {
                visit(val);
                cur = right;
                continue;
            }
            let mut p = left;
            while self.nodes[p as usize].right != NIL && self.nodes[p as usize].right != cur {
                p = self.nodes[p as usize].right;
            }
            if self.nodes[p as usize].right == NIL {
                self.nodes[p as usize].right = cur; // thread
                cur = left;
            } else {
                self.nodes[p as usize].right = NIL; // cut the thread
                visit(val);
                cur = right;
            }
        }
    }

    // in_stack: the explicit-stack in-order on the arena, the like-for-like
    // comparison for Morris in the timing table.
    fn in_stack(&self, visit: &mut impl FnMut(i64)) -> usize {
        let mut st: Vec<u32> = Vec::new();
        let mut high = 0;
        let mut cur = self.root;
        while cur != NIL || !st.is_empty() {
            while cur != NIL {
                st.push(cur);
                cur = self.nodes[cur as usize].left;
            }
            high = high.max(st.len());
            let id = st
                .pop()
                .expect("the loop condition keeps the stack non-empty here");
            visit(self.nodes[id as usize].val);
            cur = self.nodes[id as usize].right;
        }
        high
    }
}

// ---- measures: height is nodes (empty tree 0, leaf 1); diameter is edges ----

fn height(n: &Link) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + height(&b.left).max(height(&b.right)),
    }
}

// depth is the number of edges from t down to target, or None if it is absent.
fn depth(t: &Link, target: &Node) -> Option<usize> {
    let n = t.as_deref()?;
    if ptr::eq(n, target) {
        return Some(0);
    }
    depth(&n.left, target)
        .or_else(|| depth(&n.right, target))
        .map(|d| d + 1)
}

// diameter is one post-order pass: return the height, record the best bend.
fn diameter(t: &Link) -> usize {
    fn h(n: &Link, best: &mut usize) -> usize {
        match n {
            None => 0,
            Some(b) => {
                let (l, r) = (h(&b.left, best), h(&b.right, best));
                *best = (*best).max(l + r);
                1 + l.max(r)
            }
        }
    }
    let mut best = 0;
    h(t, &mut best);
    best
}

// max_path_sum returns the best sum over all downward-then-upward paths; the
// running best starts at "nothing yet", because an all-negative tree has no
// empty path. None means the tree is empty.
fn max_path_sum(t: &Link) -> Option<i64> {
    fn gain(n: &Link, best: &mut Option<i64>) -> i64 {
        match n {
            None => 0,
            Some(b) => {
                let (l, r) = (gain(&b.left, best).max(0), gain(&b.right, best).max(0));
                let through = b.val + l + r;
                *best = Some(best.map_or(through, |x| x.max(through)));
                b.val + l.max(r)
            }
        }
    }
    let mut best = None;
    gain(t, &mut best);
    best
}

// paths_with_sum counts downward paths (ancestor to descendant, at least one
// node) with sum k: a prefix-sum map along the current root path, undone on return.
fn paths_with_sum(t: &Link, k: i64) -> usize {
    fn walk(n: &Link, sum: i64, k: i64, prefix: &mut HashMap<i64, usize>) -> usize {
        let Some(b) = n else { return 0 };
        let sum = sum + b.val;
        let mut count = prefix.get(&(sum - k)).copied().unwrap_or(0);
        *prefix.entry(sum).or_insert(0) += 1;
        count += walk(&b.left, sum, k, prefix) + walk(&b.right, sum, k, prefix);
        *prefix.entry(sum).or_insert(0) -= 1; // the undo
        count
    }
    walk(t, 0, k, &mut HashMap::from([(0, 1)]))
}

// invert swaps the children everywhere, in place: it needs &mut, and
// mem::swap exchanges the two links without moving either subtree.
fn invert(n: &mut Link) {
    if let Some(b) = n {
        std::mem::swap(&mut b.left, &mut b.right);
        invert(&mut b.left);
        invert(&mut b.right);
    }
}

// lca returns the lowest common ancestor of p and q as a reference INTO the
// tree (so its lifetime is tied to the borrow of the tree), and None when
// either is not in the tree (the plain one-pass version would answer p).
fn lca<'a>(root: &'a Link, p: &Node, q: &Node) -> Option<&'a Node> {
    // returns bit 1: p is below, bit 2: q is below
    fn walk<'a>(n: Option<&'a Node>, p: &Node, q: &Node, ans: &mut Option<&'a Node>) -> u8 {
        let Some(n) = n else { return 0 };
        let mut mask = walk(n.left.as_deref(), p, q, ans) | walk(n.right.as_deref(), p, q, ans);
        if ptr::eq(n, p) {
            mask |= 1;
        }
        if ptr::eq(n, q) {
            mask |= 2;
        }
        if ans.is_none() && mask == 3 {
            *ans = Some(n); // the deepest node that sees both
        }
        mask
    }
    let mut ans = None;
    walk(root.as_deref(), p, q, &mut ans);
    ans
}

// serialize is pre-order with "#" for every nil: exactly 2n+1 tokens.
fn serialize(t: &Link) -> String {
    fn walk(n: &Link, out: &mut String) {
        if !out.is_empty() {
            out.push(',');
        }
        match n {
            None => out.push('#'),
            Some(b) => {
                out.push_str(&b.val.to_string());
                walk(&b.left, out);
                walk(&b.right, out);
            }
        }
    }
    let mut out = String::new();
    walk(t, &mut out);
    out
}

// deserialize reads the tokens back, with an Err for non-numbers, truncated
// or trailing input, and nesting deeper than max_depth.
fn deserialize(s: &str, max_depth: usize) -> Result<Link, String> {
    fn read(
        toks: &[&str],
        pos: &mut usize,
        depth: usize,
        max_depth: usize,
    ) -> Result<Link, String> {
        if depth > max_depth {
            return Err(format!("nested deeper than {max_depth}"));
        }
        let Some(&tok) = toks.get(*pos) else {
            return Err("truncated: ran out of tokens".to_string());
        };
        *pos += 1;
        if tok == "#" {
            return Ok(None);
        }
        let Ok(v) = tok.parse::<i64>() else {
            return Err(format!("token {}: not a number: {tok:?}", *pos - 1));
        };
        let left = read(toks, pos, depth + 1, max_depth)?;
        let right = read(toks, pos, depth + 1, max_depth)?;
        Ok(mk(v, left, right))
    }
    let toks: Vec<&str> = s.split(',').collect();
    let mut pos = 0;
    let root = read(&toks, &mut pos, 1, max_depth)?;
    if pos != toks.len() {
        return Err(format!("trailing: {} unread tokens", toks.len() - pos));
    }
    Ok(root)
}

// build_pre_in rebuilds a tree from its pre-order and in-order (distinct
// values), finding each root in the in-order sequence through a value -> index map.
fn build_pre_in(pre: &[i64], inorder: &[i64]) -> Result<Link, String> {
    if pre.len() != inorder.len() {
        return Err(format!(
            "lengths differ: {} and {}",
            pre.len(),
            inorder.len()
        ));
    }
    let mut pos: HashMap<i64, usize> = HashMap::with_capacity(inorder.len());
    for (i, &v) in inorder.iter().enumerate() {
        if pos.insert(v, i).is_some() {
            return Err(format!("duplicate value {v}"));
        }
    }
    // the tree for inorder[lo..hi]
    fn build(
        pre: &[i64],
        next: &mut usize,
        pos: &HashMap<i64, usize>,
        lo: usize,
        hi: usize,
    ) -> Result<Link, String> {
        if lo >= hi {
            return Ok(None);
        }
        let v = pre[*next];
        *next += 1;
        match pos.get(&v) {
            Some(&i) if (lo..hi).contains(&i) => {
                let left = build(pre, next, pos, lo, i)?;
                let right = build(pre, next, pos, i + 1, hi)?;
                Ok(mk(v, left, right))
            }
            _ => Err(format!(
                "pre-order value {v} does not belong to in-order range [{lo},{hi})"
            )),
        }
    }
    build(pre, &mut 0, &pos, 0, inorder.len())
}

// =====================================================================
// 2. PROVE: the oracle. A tree IS the set of its (path, value) pairs, where
// a path is a string of L and R from the root. Every property below is
// defined over those strings alone, by the slowest obvious method.
// =====================================================================

struct Entry<'a> {
    node: &'a Node,
    path: String,
}

fn entries(t: &Link) -> Vec<Entry<'_>> {
    fn walk<'a>(n: &'a Link, path: &str, out: &mut Vec<Entry<'a>>) {
        if let Some(b) = n {
            out.push(Entry {
                node: b,
                path: path.to_string(),
            });
            walk(&b.left, &format!("{path}L"), out);
            walk(&b.right, &format!("{path}R"), out);
        }
    }
    let mut out = Vec::new();
    walk(t, "", &mut out);
    out
}

fn is_prefix(a: &str, b: &str) -> bool {
    a.len() < b.len() && b.starts_with(a) // a proper prefix
}

// The four orders, each as a comparison of two paths.
fn cmp_pre(a: &str, b: &str) -> Ordering {
    a.cmp(b) // prefix first, then L < R
}
fn cmp_post(a: &str, b: &str) -> Ordering {
    // extensions before their prefix
    if is_prefix(a, b) {
        Ordering::Greater
    } else if is_prefix(b, a) {
        Ordering::Less
    } else {
        a.cmp(b)
    }
}
fn cmp_in(a: &str, b: &str) -> Ordering {
    // a node sits between its left and its right subtree
    if is_prefix(a, b) {
        // b is below a: in the left subtree it comes first, in the right one after
        if b.as_bytes()[a.len()] == b'L' {
            Ordering::Greater
        } else {
            Ordering::Less
        }
    } else if is_prefix(b, a) {
        if a.as_bytes()[b.len()] == b'L' {
            Ordering::Less
        } else {
            Ordering::Greater
        }
    } else {
        a.cmp(b)
    }
}
fn cmp_level(a: &str, b: &str) -> Ordering {
    a.len().cmp(&b.len()).then_with(|| a.cmp(b))
}

fn order_by(es: &[Entry], c: fn(&str, &str) -> Ordering) -> Vec<i64> {
    let mut sorted: Vec<&Entry> = es.iter().collect();
    sorted.sort_by(|x, y| c(&x.path, &y.path));
    sorted.iter().map(|e| e.node.val).collect()
}

fn lcp(a: &str, b: &str) -> usize {
    // length of the longest common prefix
    a.bytes().zip(b.bytes()).take_while(|(x, y)| x == y).count()
}

fn o_height(es: &[Entry]) -> usize {
    es.iter().map(|e| e.path.len() + 1).max().unwrap_or(0)
}

fn o_diameter(es: &[Entry]) -> usize {
    let mut d = 0;
    for a in es {
        for b in es {
            d = d.max(a.path.len() + b.path.len() - 2 * lcp(&a.path, &b.path));
        }
    }
    d
}

fn values_by_path<'a>(es: &'a [Entry]) -> HashMap<&'a str, i64> {
    es.iter().map(|e| (e.path.as_str(), e.node.val)).collect()
}

// o_max_path_sum: the best sum over every pair of nodes of the nodes on the path between them.
fn o_max_path_sum(es: &[Entry]) -> i64 {
    let val = values_by_path(es);
    let mut best = i64::MIN;
    for a in es {
        for b in es {
            let l = lcp(&a.path, &b.path);
            let mut sum = 0;
            for j in l..=a.path.len() {
                // a's side, down from the meeting node
                sum += val[&a.path[..j]];
            }
            for j in l + 1..=b.path.len() {
                // b's side, below the meeting node
                sum += val[&b.path[..j]];
            }
            best = best.max(sum);
        }
    }
    best
}

// o_paths counts ancestor-descendant pairs (a may equal b) whose nodes sum to k.
fn o_paths(es: &[Entry], k: i64) -> usize {
    let val = values_by_path(es);
    let mut count = 0;
    for b in es {
        for i in 0..=b.path.len() {
            // i = the ancestor's depth
            let sum: i64 = (i..=b.path.len()).map(|j| val[&b.path[..j]]).sum();
            if sum == k {
                count += 1;
            }
        }
    }
    count
}

fn mirror_path(p: &str) -> String {
    p.chars()
        .map(|c| if c == 'L' { 'R' } else { 'L' })
        .collect()
}

// fingerprint is the whole tree as a sorted list of "path=value" strings,
// optionally mirrored (L and R swapped).
fn fingerprint(es: &[Entry], mirror: bool) -> Vec<String> {
    let mut out: Vec<String> = es
        .iter()
        .map(|e| {
            let p = if mirror {
                mirror_path(&e.path)
            } else {
                e.path.clone()
            };
            format!("{p}={}", e.node.val)
        })
        .collect();
    out.sort();
    out
}

fn entry_at<'a>(es: &[Entry<'a>], path: &str) -> Option<&'a Node> {
    es.iter().find(|e| e.path == path).map(|e| e.node)
}

// ---- the checking harness ----

// Kit holds the four functions that have a classic bug in this lesson, so the
// harness can be pointed at a broken one: an oracle that cannot fail proves nothing.
struct Kit {
    diameter: fn(&Link) -> usize,
    max_path_sum: fn(&Link) -> Option<i64>,
    paths_with_sum: fn(&Link, i64) -> usize,
    lca: for<'a> fn(&'a Link, &Node, &Node) -> Option<&'a Node>,
}

const GOOD: Kit = Kit {
    diameter,
    max_path_sum,
    paths_with_sum,
    lca,
};

// The four mutants, one per pitfall of the lesson.
fn bad_diameter(t: &Link) -> usize {
    // only through the root
    t.as_ref().map_or(0, |n| height(&n.left) + height(&n.right))
}

fn bad_max_path_sum(t: &Link) -> Option<i64> {
    // starting the running best at 0 is the same as clamping the answer at 0
    max_path_sum(t).map(|best| best.max(0))
}

fn bad_paths_with_sum(t: &Link, k: i64) -> usize {
    fn walk(n: &Link, sum: i64, k: i64, prefix: &mut HashMap<i64, usize>) -> usize {
        let Some(b) = n else { return 0 };
        let sum = sum + b.val;
        let count = prefix.get(&(sum - k)).copied().unwrap_or(0);
        *prefix.entry(sum).or_insert(0) += 1; // wrong: never removed when the recursion returns
        count + walk(&b.left, sum, k, prefix) + walk(&b.right, sum, k, prefix)
    }
    walk(t, 0, k, &mut HashMap::from([(0, 1)]))
}

fn bad_lca<'a>(root: &'a Link, p: &Node, q: &Node) -> Option<&'a Node> {
    // no found check: if q is absent this answers p
    fn walk<'a>(n: Option<&'a Node>, p: &Node, q: &Node) -> Option<&'a Node> {
        let n = n?;
        if ptr::eq(n, p) || ptr::eq(n, q) {
            return Some(n);
        }
        let (l, r) = (
            walk(n.left.as_deref(), p, q),
            walk(n.right.as_deref(), p, q),
        );
        if l.is_some() && r.is_some() {
            Some(n)
        } else {
            l.or(r)
        }
    }
    walk(root.as_deref(), p, q)
}

const G_TRAVERSALS: usize = 0;
const G_SHAPE: usize = 1;
const G_SUMS: usize = 2;
const G_STRUCTURE: usize = 3;
const NUM_GROUPS: usize = 4;
const GROUP_NAMES: [&str; NUM_GROUPS] = ["traversals", "shape", "sums", "structure"];

struct MutantSpec {
    name: &'static str,
    kit: Kit,
    hit: usize, // the group that should notice
}

#[derive(Default)]
struct Prover {
    checks: [usize; NUM_GROUPS],
    bad: [usize; NUM_GROUPS],
    first: String, // description of the first failure, if any
}

impl Prover {
    fn check(&mut self, g: usize, ok: bool, what: &str) {
        self.checks[g] += 1;
        if !ok {
            self.bad[g] += 1;
            if self.first.is_empty() {
                self.first = format!("{}: {what}", GROUP_NAMES[g]);
            }
        }
    }

    // prove runs every function of the toolkit on one tree pair: t has distinct
    // labels (for order, LCA, rebuild), s has the same shape and small values
    // (for the sums). Both are fresh, so the checks may mutate clones.
    fn prove(&mut self, t: &Link, s: &Link, k: &Kit) {
        let es = entries(t);
        let n = es.len();
        let absent = Node {
            val: -1,
            left: None,
            right: None,
        }; // a node that is not in the tree

        // -- traversals: each against its sort of the path strings
        let want_pre = order_by(&es, cmp_pre);
        let want_in = order_by(&es, cmp_in);
        let want_post = order_by(&es, cmp_post);
        let (mut r_pre, mut r_in, mut r_post) = (Vec::new(), Vec::new(), Vec::new());
        let (mut s_pre, mut s_in, mut s_post) = (Vec::new(), Vec::new(), Vec::new());
        let calls = [
            pre_rec(t, &mut |v| r_pre.push(v)),
            in_rec(t, &mut |v| r_in.push(v)),
            post_rec(t, &mut |v| r_post.push(v)),
        ];
        let hw = [
            pre_stack(t, &mut |v| s_pre.push(v)),
            in_stack(t, &mut |v| s_in.push(v)),
            post_frames(t, &mut |v| s_post.push(v)),
        ];
        let recs = [&r_pre, &r_in, &r_post];
        let stks = [&s_pre, &s_in, &s_post];
        for (i, want) in [&want_pre, &want_in, &want_post].into_iter().enumerate() {
            self.check(
                G_TRAVERSALS,
                recs[i] == want,
                "recursive order differs from the oracle",
            );
            self.check(
                G_TRAVERSALS,
                stks[i] == want,
                "explicit-stack order differs from the oracle",
            );
            self.check(
                G_TRAVERSALS,
                stks[i] == recs[i],
                "iterative differs from recursive",
            );
            self.check(G_TRAVERSALS, calls[i] == 2 * n + 1, "calls != 2n+1");
            self.check(G_TRAVERSALS, hw[i] <= n, "stack high-water above n");
        }
        let mut levels: Vec<Vec<i64>> = Vec::new();
        let q_high = level_order(t, &mut |d, v| {
            while levels.len() <= d {
                levels.push(Vec::new());
            }
            levels[d].push(v);
        });
        let mut want_levels: Vec<Vec<i64>> = Vec::new();
        let mut by_level: Vec<&Entry> = es.iter().collect();
        by_level.sort_by(|x, y| cmp_level(&x.path, &y.path));
        for e in by_level {
            if e.path.len() == want_levels.len() {
                want_levels.push(Vec::new());
            }
            want_levels[e.path.len()].push(e.node.val);
        }
        self.check(G_TRAVERSALS, levels == want_levels, "level order differs");
        self.check(G_TRAVERSALS, q_high <= n, "queue high-water above n");
        // Morris runs on an arena copy, and the copy must come back identical.
        let mut arena = Arena::from_box(t);
        let pristine = arena.nodes.clone();
        let mut m_in = Vec::new();
        arena.morris(&mut |v| m_in.push(v));
        self.check(
            G_TRAVERSALS,
            m_in == want_in,
            "Morris differs from the oracle",
        );
        self.check(
            G_TRAVERSALS,
            arena.nodes == pristine,
            "Morris left the tree changed",
        );

        // -- shape: height, depth, diameter
        self.check(G_SHAPE, height(t) == o_height(&es), "height");
        self.check(G_SHAPE, (k.diameter)(t) == o_diameter(&es), "diameter");
        self.check(
            G_SHAPE,
            depth(t, &absent).is_none(),
            "depth of an absent node",
        );
        for e in &es {
            self.check(G_SHAPE, depth(t, e.node) == Some(e.path.len()), "depth");
        }

        // -- sums on s: max path sum, and downward paths with sum k for k in [-5, 5]
        let ses = entries(s);
        let best = (k.max_path_sum)(s);
        if n == 0 {
            self.check(G_SUMS, best.is_none(), "max path sum of the empty tree");
        } else {
            self.check(G_SUMS, best == Some(o_max_path_sum(&ses)), "max path sum");
        }
        for target in -5..=5 {
            self.check(
                G_SUMS,
                (k.paths_with_sum)(s, target) == o_paths(&ses, target),
                "paths with sum k",
            );
        }

        // -- structure: invert, LCA, serialize, rebuild
        let mut c = t.clone();
        invert(&mut c);
        self.check(
            G_STRUCTURE,
            fingerprint(&entries(&c), false) == fingerprint(&es, true),
            "invert is not the mirror",
        );
        invert(&mut c);
        self.check(G_STRUCTURE, c == *t, "invert twice is not the identity");
        for a in &es {
            for b in &es {
                let got = (k.lca)(t, a.node, b.node);
                let meet = &a.path[..lcp(&a.path, &b.path)];
                let want = entry_at(&es, meet);
                self.check(
                    G_STRUCTURE,
                    got.is_some() && want.is_some_and(|w| ptr::eq(w, got.unwrap())),
                    "LCA",
                );
            }
            let (f1, f2) = ((k.lca)(t, a.node, &absent), (k.lca)(t, &absent, a.node));
            self.check(
                G_STRUCTURE,
                f1.is_none() && f2.is_none(),
                "LCA with an absent node must report not found",
            );
        }
        let text = serialize(t);
        self.check(
            G_STRUCTURE,
            text.matches(',').count() + 1 == 2 * n + 1,
            "serialize token count",
        );
        let back = deserialize(&text, 1000);
        self.check(
            G_STRUCTURE,
            back.as_ref().is_ok_and(|b| b == t),
            "deserialize round trip",
        );
        let e1 = deserialize(&format!("{text},5"), 1000);
        let e2 = deserialize(&text[..text.rfind(',').unwrap_or(0)], 1000);
        let e3 = deserialize(&format!("x,{text}"), 1000);
        self.check(
            G_STRUCTURE,
            e1.is_err() && e2.is_err() && e3.is_err(),
            "deserialize accepted bad input",
        );
        let rebuilt = build_pre_in(&want_pre, &want_in);
        self.check(
            G_STRUCTURE,
            rebuilt.as_ref().is_ok_and(|b| b == t),
            "rebuild from pre + in",
        );
        let e4 = build_pre_in(&want_pre, &want_in[..n.saturating_sub(1)]);
        self.check(
            G_STRUCTURE,
            n == 0 || e4.is_err(),
            "rebuild accepted a length mismatch",
        );
        if n > 0 {
            let mut bad = want_pre.clone();
            bad[0] = -7;
            self.check(
                G_STRUCTURE,
                build_pre_in(&bad, &want_in).is_err(),
                "rebuild accepted a foreign root",
            );
        }
    }
}

// =====================================================================
// 3. MEASURE
// =====================================================================

// best_ns_per_node times each function 3 rounds (3 runs per round, interleaved)
// and keeps the best round.
fn best_ns_per_node(n: usize, fs: &mut [&mut dyn FnMut()]) -> Vec<f64> {
    let mut best = vec![0.0f64; fs.len()];
    for round in 0..3 {
        for (i, f) in fs.iter_mut().enumerate() {
            let t = Instant::now();
            for _ in 0..3 {
                f();
            }
            let ns = t.elapsed().as_nanos() as f64 / (3 * n) as f64;
            if round == 0 || ns < best[i] {
                best[i] = ns;
            }
        }
    }
    best
}

fn main() {
    let mut r = Pcg::new(1, 2);

    // ---- 1. build ----
    println!("1. BUILD: the generators");
    {
        let mut a = Pcg::new(1, 2);
        let hs: Vec<usize> = [15, 1000, 100000]
            .iter()
            .map(|&n| height(&random_split(n, &mut a)))
            .collect();
        println!(
            "  randomSplit(15), (1000), (100000) from one fresh PCG(1,2): heights {}",
            go_slice(&hs)
        );
        let counts: Vec<usize> = (0..=7).map(|n| shapes(n).len()).collect();
        println!(
            "  shapes with n = 0..7 nodes: {}, {} in all (Catalan numbers)",
            go_slice(&counts),
            counts.iter().sum::<usize>()
        );
    }

    // ---- 2. prove ----
    println!("\n2. PROVE: every function against the path-string oracle");
    let mut p = Prover::default();
    const RANDOM_TREES: usize = 5000;
    let (mut nodes, mut empty, mut all_neg) = (0, 0, 0);
    let mut shown = false;
    for i in 0..RANDOM_TREES {
        let n = r.int_n(41);
        let mut t = random_split(n, &mut r);
        label_shuffled(&mut t, &mut r);
        let mut s = t.clone();
        label_range(&mut s, -3, 3, &mut r);
        if n == 0 {
            empty += 1;
        } else if order_by(&entries(&s), cmp_pre)
            .iter()
            .max()
            .is_some_and(|&m| m <= 0)
        {
            all_neg += 1;
        }
        nodes += n;
        if !shown && n == 7 {
            // show the oracle's view of one tree, with both labellings
            shown = true;
            let mut es = entries(&t);
            let mut ses = entries(&s);
            es.sort_by(|x, y| cmp_pre(&x.path, &y.path));
            ses.sort_by(|x, y| cmp_pre(&x.path, &y.path));
            println!("  tree #{i}, n = {n}, as path=value in pre-order ('.' is the root):");
            for (j, name) in [
                "distinct labels (order, LCA, rebuild)",
                "small labels, same shape (sums)",
            ]
            .iter()
            .enumerate()
            {
                print!("    {name:<38}");
                for (e, se) in es.iter().zip(&ses) {
                    let v = if j == 0 { e.node.val } else { se.node.val };
                    let path = if e.path.is_empty() { "." } else { &e.path };
                    print!(" {path}={v}");
                }
                println!();
            }
        }
        p.prove(&t, &s, &GOOD);
    }
    let random_checks = p.checks;
    let mut shape_trees = 0;
    for n in 0..=7 {
        for mut t in shapes(n) {
            label_shuffled(&mut t, &mut r);
            let mut s = t.clone();
            label_range(&mut s, -3, 3, &mut r);
            p.prove(&t, &s, &GOOD);
            shape_trees += 1;
        }
    }
    println!(
        "  {RANDOM_TREES} random trees (n uniform in [0,40]: {nodes} nodes in all, {empty} empty, {all_neg} with every value <= 0) + {shape_trees} shapes (every tree of n <= 7)"
    );
    let mut all_checks = 0;
    for g in 0..NUM_GROUPS {
        println!(
            "  {:<11} {} mismatches over {} trees, {} checks ({} on random trees, {} on shapes)",
            GROUP_NAMES[g],
            p.bad[g],
            RANDOM_TREES + shape_trees,
            p.checks[g],
            random_checks[g],
            p.checks[g] - random_checks[g]
        );
        all_checks += p.checks[g];
    }
    println!("  total: {all_checks} checks, first failure: {:?}", p.first);

    // the error paths, with their messages
    println!("  Deserialize on text you did not write:");
    for s in ["1,#,#", "1,#", "1,#,#,#", "1,x,#,#", "1,2,3,4,#,#"] {
        let res = deserialize(s, 3).err().unwrap_or_else(|| "ok".to_string());
        println!("    {:<14} -> {res}", format!("{s:?}"));
    }
    let err = build_pre_in(&[1, 2, 3], &[2, 1]).err().unwrap_or_default();
    let err2 = build_pre_in(&[9, 2], &[2, 1]).err().unwrap_or_default();
    let err3 = build_pre_in(&[1, 1], &[1, 1]).err().unwrap_or_default();
    println!("  BuildPreIn: {err} | {err2} | {err3}");

    // the oracle must be able to fail: point it at each broken function
    println!("  the same checks on four broken functions (1000 fresh random trees + every shape):");
    let mutants = [
        MutantSpec {
            name: "diameter only through the root",
            kit: Kit {
                diameter: bad_diameter,
                ..GOOD
            },
            hit: G_SHAPE,
        },
        MutantSpec {
            name: "max path sum starting at 0",
            kit: Kit {
                max_path_sum: bad_max_path_sum,
                ..GOOD
            },
            hit: G_SUMS,
        },
        MutantSpec {
            name: "paths with sum k, no undo",
            kit: Kit {
                paths_with_sum: bad_paths_with_sum,
                ..GOOD
            },
            hit: G_SUMS,
        },
        MutantSpec {
            name: "LCA without the found check",
            kit: Kit {
                lca: bad_lca,
                ..GOOD
            },
            hit: G_STRUCTURE,
        },
    ];
    for m in &mutants {
        let mut mp = Prover::default();
        let mut mr = Pcg::new(3, 4);
        for _ in 0..1000 {
            let n = mr.int_n(41);
            let mut t = random_split(n, &mut mr);
            label_shuffled(&mut t, &mut mr);
            let mut s = t.clone();
            label_range(&mut s, -3, 3, &mut mr);
            mp.prove(&t, &s, &m.kit);
        }
        for n in 0..=7 {
            for mut t in shapes(n) {
                label_shuffled(&mut t, &mut mr);
                let mut s = t.clone();
                label_range(&mut s, -3, 3, &mut mr);
                mp.prove(&t, &s, &m.kit);
            }
        }
        println!(
            "    {:<32} {:>6} mismatches in {}; first: {}",
            m.name, mp.bad[m.hit], GROUP_NAMES[m.hit], mp.first
        );
    }

    // ---- deep: a 1,000,000-node chain, only the functions that do not recurse ----
    const DEEP_N: usize = 1_000_000;
    println!("\n  deep: a left chain of {DEEP_N} nodes, built in a loop");
    let chain = left_chain(DEEP_N);
    // each function must produce the chain's values in the right order (no vector is built)
    fn deep(
        name: &str,
        want: impl Fn(usize) -> i64,
        run: impl FnOnce(&mut dyn FnMut(i64)) -> String,
    ) {
        let (mut i, mut bad) = (0usize, 0usize);
        let note = run(&mut |v| {
            if v != want(i) {
                bad += 1;
            }
            i += 1;
        });
        println!("    {name:<11} visited {i}, wrong {bad}, {note}");
    }
    let up = |i: usize| i as i64;
    let down = |i: usize| (DEEP_N - 1 - i) as i64;
    deep("PreStack", up, |v| {
        format!("stack high-water {}", pre_stack(&chain, &mut |x| v(x)))
    });
    deep("InStack", down, |v| {
        format!("stack high-water {}", in_stack(&chain, &mut |x| v(x)))
    });
    deep("PostFrames", down, |v| {
        format!("stack high-water {}", post_frames(&chain, &mut |x| v(x)))
    });
    deep("LevelOrder", up, |v| {
        format!(
            "queue high-water {}",
            level_order(&chain, &mut |d, val| {
                // on this chain the depth of a node is its value
                v(if d as i64 != val { -1 } else { val })
            })
        )
    });
    let mut chain_arena = Arena::from_box(&chain);
    deep("Morris", down, |v| {
        chain_arena.morris(&mut |x| v(x));
        "no stack at all (on an arena copy)".to_string()
    });
    let (mut spine, mut stray) = (0, 0);
    let mut q = chain_arena.root;
    while q != NIL {
        spine += 1;
        if chain_arena.nodes[q as usize].right != NIL {
            stray += 1;
        }
        q = chain_arena.nodes[q as usize].left;
    }
    println!(
        "    after Morris the chain is intact: {spine} nodes on the left spine, {stray} stray right links"
    );
    // The recursive height on the chain needs a stack sized for it: a spawned
    // thread gets 2 MiB by default (std::thread docs), so ask for 256 MiB.
    let chain_height = thread::scope(|sc| {
        thread::Builder::new()
            .stack_size(256 << 20)
            .spawn_scoped(sc, || height(&chain))
            .expect("the OS refused a 256 MiB stack")
            .join()
            .expect("the height thread panicked")
    });
    println!(
        "    Height (recursive, {DEEP_N} frames deep) = {chain_height}: run on a thread with a 256 MiB stack"
    );
    drop(chain_arena);

    // ---- 3. measure ----
    println!("\n3. MEASURE: ns per node, best of 3, two shapes of about a million nodes");
    const N_RANDOM: usize = 1_000_000;
    const N_COMPLETE: usize = (1 << 20) - 1;
    let rnd = random_split(N_RANDOM, &mut Pcg::new(1, 2));
    let cpl = complete(0, N_COMPLETE);
    let shs: [(&str, &Link, usize); 2] = [
        ("randomSplit", &rnd, N_RANDOM),
        ("complete", &cpl, N_COMPLETE),
    ];
    let rows = [
        "recursive pre-order",
        "explicit pre-order",
        "recursive in-order",
        "explicit in-order",
        "Morris in-order",
        "level order",
        "post-order frame stack",
    ];
    let mut table: Vec<Vec<f64>> = Vec::new();
    for &(_, t, n) in &shs {
        // Morris mutates its arena (and restores it) while the explicit-stack
        // partner only reads it: a RefCell lets the two closures share it.
        let arena = RefCell::new(Arena::from_box(t));
        let sum = Cell::new(0i64);
        let acc = |v: i64| sum.set(sum.get() ^ v);
        table.push(best_ns_per_node(
            n,
            &mut [
                &mut || {
                    black_box(pre_rec(black_box(t), &mut |v| acc(v)));
                },
                &mut || {
                    black_box(pre_stack(black_box(t), &mut |v| acc(v)));
                },
                &mut || {
                    black_box(in_rec(black_box(t), &mut |v| acc(v)));
                },
                &mut || {
                    black_box(in_stack(black_box(t), &mut |v| acc(v)));
                },
                &mut || arena.borrow_mut().morris(&mut |v| acc(v)),
                &mut || {
                    black_box(level_order(black_box(t), &mut |_, v| acc(v)));
                },
                &mut || {
                    black_box(post_frames(black_box(t), &mut |v| acc(v)));
                },
                // the like-for-like partner of Morris: the same arena, an explicit stack
                &mut || {
                    black_box(arena.borrow().in_stack(&mut |v| acc(v)));
                },
            ],
        ));
        black_box(sum.get());
    }
    // the build: construct, time the construction only, then drop outside the timer
    let mut builds = [0.0f64; 2];
    for (si, &(_, _, n)) in shs.iter().enumerate() {
        for round in 0..3 {
            let t0 = Instant::now();
            let tree = if si == 0 {
                random_split(n, &mut Pcg::new(1, 2))
            } else {
                complete(0, n)
            };
            let ns = t0.elapsed().as_nanos() as f64 / n as f64;
            black_box(&tree);
            drop(tree);
            if round == 0 || ns < builds[si] {
                builds[si] = ns;
            }
        }
    }
    println!(
        "  {:<24} {:>12}({}) {:>12}({})",
        "ns per node", shs[0].0, N_RANDOM, shs[1].0, N_COMPLETE
    );
    for (i, name) in rows.iter().enumerate() {
        println!("  {:<24} {:>18.1} {:>18.1}", name, table[0][i], table[1][i]);
        if i == 4 {
            println!(
                "  {:<24} {:>18.1} {:>18.1}",
                "  (explicit in-order, arena)", table[0][7], table[1][7]
            );
        }
    }
    println!(
        "  {:<24} {:>18.1} {:>18.1}",
        "build the tree", builds[0], builds[1]
    );
    let cheaper = (0..rows.len())
        .filter(|&i| table[1][i] < table[0][i])
        .count();
    println!(
        "  cheaper per node on complete than on randomSplit: {cheaper} of {} traversals; the build: {}",
        rows.len(),
        builds[1] < builds[0]
    );
    // like for like: Morris runs on the arena, so its partner is the arena's explicit stack
    println!(
        "  Morris / explicit-stack in-order on the same arena, per node: {:.1}x on randomSplit, {:.1}x on complete",
        table[0][4] / table[0][7],
        table[1][4] / table[1][7]
    );

    // the counts that explain who needs how much memory
    println!("\n  high-water marks (entries held at once; counts, identical in every run)");
    println!(
        "  {:<24} {:>12} {:>12} {:>12}",
        "", "randomSplit", "complete", "left chain"
    );
    let shapes3: [&Link; 3] = [&rnd, &cpl, &chain];
    let row = |name: &str, f: &dyn Fn(&Link) -> usize| {
        print!("  {name:<24}");
        for t in shapes3 {
            print!(" {:>12}", f(t));
        }
        println!();
    };
    // height on the chain is the recursive answer computed above on the big thread
    row("height", &|t| {
        if ptr::eq(t, &chain) {
            chain_height
        } else {
            height(t)
        }
    });
    row("pre-order stack", &|t| pre_stack(t, &mut |_| {}));
    row("in-order stack", &|t| in_stack(t, &mut |_| {}));
    row("post-order frame stack", &|t| post_frames(t, &mut |_| {}));
    row("level-order queue", &|t| level_order(t, &mut |_, _| {}));
}

// Go prints a slice as [1 2 3]; Rust's {:?} prints [1, 2, 3]. Format by hand.
fn go_slice(v: &[usize]) -> String {
    let parts: Vec<String> = v.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}
```

**Rust sample output:**

```
1. BUILD: the generators
  randomSplit(15), (1000), (100000) from one fresh PCG(1,2): heights [6 20 41]
  shapes with n = 0..7 nodes: [1 1 2 5 14 42 132 429], 626 in all (Catalan numbers)

2. PROVE: every function against the path-string oracle
  tree #75, n = 7, as path=value in pre-order ('.' is the root):
    distinct labels (order, LCA, rebuild)  .=2 R=3 RL=5 RLL=6 RLR=0 RLRL=1 RR=4
    small labels, same shape (sums)        .=3 R=3 RL=1 RLL=-1 RLR=-1 RLRL=3 RR=-3
  5000 random trees (n uniform in [0,40]: 100375 nodes in all, 109 empty, 157 with every value <= 0) + 626 shapes (every tree of n <= 7)
  traversals  0 mismatches over 5626 trees, 106894 checks (95000 on random trees, 11894 on shapes)
  shape       0 mismatches over 5626 trees, 121334 checks (115375 on random trees, 5959 on shapes)
  sums        0 mismatches over 5626 trees, 67512 checks (60000 on random trees, 7512 on shapes)
  structure   0 mismatches over 5626 trees, 2888544 checks (2852355 on random trees, 36189 on shapes)
  total: 3184284 checks, first failure: ""
  Deserialize on text you did not write:
    "1,#,#"        -> ok
    "1,#"          -> truncated: ran out of tokens
    "1,#,#,#"      -> trailing: 1 unread tokens
    "1,x,#,#"      -> token 1: not a number: "x"
    "1,2,3,4,#,#"  -> nested deeper than 3
  BuildPreIn: lengths differ: 3 and 2 | pre-order value 9 does not belong to in-order range [0,2) | duplicate value 1
  the same checks on four broken functions (1000 fresh random trees + every shape):
    diameter only through the root      243 mismatches in shape; first: shape: diameter
    max path sum starting at 0           28 mismatches in sums; first: sums: max path sum
    paths with sum k, no undo         10058 mismatches in sums; first: sums: paths with sum k
    LCA without the found check       24132 mismatches in structure; first: structure: LCA with an absent node must report not found

  deep: a left chain of 1000000 nodes, built in a loop
    PreStack    visited 1000000, wrong 0, stack high-water 1
    InStack     visited 1000000, wrong 0, stack high-water 1000000
    PostFrames  visited 1000000, wrong 0, stack high-water 1000000
    LevelOrder  visited 1000000, wrong 0, queue high-water 1
    Morris      visited 1000000, wrong 0, no stack at all (on an arena copy)
    after Morris the chain is intact: 1000000 nodes on the left spine, 0 stray right links
    Height (recursive, 1000000 frames deep) = 1000000: run on a thread with a 256 MiB stack

3. MEASURE: ns per node, best of 3, two shapes of about a million nodes
  ns per node               randomSplit(1000000)     complete(1048575)
  recursive pre-order                     6.0                1.5
  explicit pre-order                      5.2                1.5
  recursive in-order                      5.9                1.6
  explicit in-order                       5.7                1.7
  Morris in-order                         9.2                3.8
    (explicit in-order, arena)                6.1                1.5
  level order                             9.1                4.0
  post-order frame stack                  8.0                2.4
  build the tree                         21.0               13.9
  cheaper per node on complete than on randomSplit: 7 of 7 traversals; the build: true
  Morris / explicit-stack in-order on the same arena, per node: 1.5x on randomSplit, 2.6x on complete

  high-water marks (entries held at once; counts, identical in every run)
                            randomSplit     complete   left chain
  height                             48           20      1000000
  pre-order stack                    31           20            1
  in-order stack                     32           20      1000000
  post-order frame stack             48           20      1000000
  level-order queue               84453       524288            1
```

> **Go vs Rust:** Everything above the timing table is identical to the Go output (diffed), all 3,184,284 check counts included, except two lines: Morris runs on an arena copy because a `Box` tree cannot hold the thread (example 12), and the recursive `height` on the million-node chain needs a thread with a 256 MiB stack where Go just grows its own. Only `invert`, the two labelling generators and `Arena::morris` take `&mut`; `lca` returns an `Option<&Node>` tied to the borrow of the tree, `deserialize` and `build_pre_in` return `Result`, and the iterative `Drop` is what lets the million-node chain go out of scope. The clocks differ from Go's: Morris cost 1.5x the explicit-stack in-order on the same arena for randomSplit and 2.5-2.6x for complete in every Rust run (Go: 1.2-1.3x and 0.8-0.9x), level order only tied with Morris for slowest where Go's was clearly slowest, and building the tree took 20.5-21.2 ns per node for randomSplit against 13.4-15.5 in Go.

**Complexity:** every operation is Θ(n) time, with Θ(h) stack for the recursive and explicit-stack walks, Θ(w) queue for level order and Θ(1) for Morris; the oracle itself is O(n²) or O(n·h²) per tree, which is why it only runs on trees of at most 40 nodes. Result: 0 mismatches over 3,184,284 checks on 5,000 random trees and all 626 shapes of up to seven nodes (traversals 106,894 checks, shape 121,334, sums 67,512, structure 2,888,544), and each of the four broken functions is caught (243, 28, 10,058 and 24,132 mismatches). High-water marks are counts, identical in every run: on the million-node left chain PreStack and LevelOrder hold 1 entry while InStack and PostFrames hold 1,000,000; on randomSplit(1,000,000), height 48, the three stacks hold 31, 32 and 48 and the level-order queue 84,453; on complete(1,048,575) they hold 20, 20 and 20 and 524,288. Timing (best of 3, about a dozen runs on a loaded machine; the Go file is about 1,100 lines, over the size guide because it is the capstone): the level-order walk was the slowest of the seven traversals on both shapes (about 13-16 ns per node on randomSplit, 9-13 on complete) and the post-order frame stack the second slowest; every traversal was cheaper per node on the complete tree (7 of 7 in every run); Morris cost 1.2-1.3x the explicit-stack in-order on randomSplit and 0.8-0.9x on complete, so there it was no slower. Absolute nanoseconds depend on the machine.

---

> Next lesson: [18 — Binary Search Trees](../../18-binary-search-trees.md) — the invariant that makes
> in-order sorted, and the shape that can ruin it.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
