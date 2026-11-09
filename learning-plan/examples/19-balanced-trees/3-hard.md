# Step 19 — Balanced Trees & B-Trees · 🔴 Hard

Examples **12–16**: the delete that makes AVL rotate at every level, the B+ tree, page reads behind a buffer pool,
comparisons against node visits, and the capstone ordered map.

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

## 12. AVL deletion: the Fibonacci worst case

`🔴 hard` · *Rebalancing cost asymmetry*

An AVL insert needs at most one fix-up, because the first repair restores the subtree to the height it had before, so nothing above it changes. A delete has no such luck: a repair can leave its subtree one level *shorter* than before, which tips the parent over, and the walk back up can fire at every level. The sparsest AVL tree of height h (example 4's Fibonacci tree) is the tree built to make that happen, so the experiment is to delete every key of it from a fresh copy and count rotations: the worst delete grows by one rotation for every two levels, Θ(log n) per delete, while the same trees take every insert with at most one fix-up. Counting rotations instead of time makes the answer exact and identical on every run, and a typical tree is then measured to see what the worst case costs on average.

**Steps:**

1. Build the sparsest AVL tree of height h (left subtree h−1, right subtree h−2) with the even keys 2, 4, 6, …, using the recursive AVL insert and delete of example 7 plus a checker run after every operation.
2. The key move: for each key of the tree, rebuild a fresh copy, delete that key and record rotations and fix-ups; then insert every odd key (the gaps) into fresh copies for contrast, and finally delete all 100,000 keys of random-built and ascending-built trees.
3. Read the worst delete per height against h/2−1, the trace of one worst delete (each repair leaves its subtree shorter), the insert table (never more than one fix-up) and the typical-case rows.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
)

// AVL with stored heights (a leaf has height 1, nil has 0). The tree is the
// recursive one from example 7: insert and delete return the NEW root of the
// subtree they were given, and every node on the way back up is passed through
// rebalance. That is the whole difference this example is about:
//
//	insert: the first rebalance that fires restores the subtree to its old
//	        height, so nothing above it changes -> at most ONE fix-up.
//	delete: a fix-up can leave the subtree one level SHORTER than before,
//	        which unbalances its parent, and so on up to the root.
type node struct {
	key         int
	h           int
	left, right *node
}

func ht(n *node) int {
	if n == nil {
		return 0
	}
	return n.h
}

func upd(n *node)         { n.h = 1 + max(ht(n.left), ht(n.right)) }
func balance(n *node) int { return ht(n.left) - ht(n.right) }
func rotateRight(y *node) *node {
	x := y.left
	y.left = x.right
	x.right = y
	upd(y) // the node that went DOWN first
	upd(x)
	return x
}

func rotateLeft(x *node) *node {
	y := x.right
	x.right = y.left
	y.left = x
	upd(x)
	upd(y)
	return y
}

// event records one fix-up for the trace in part (b).
type event struct {
	key, bal, childBal, before, after int
	double                            bool
}

// stats counts the repair work. A double rotation counts as 2 rotations and 1 fix-up.
type stats struct {
	rots, fixups int
	tracing      bool
	events       []event
}

func rebalance(n *node, st *stats) *node {
	before := n.h // stale on purpose: the height this subtree had BEFORE the update
	upd(n)
	b := balance(n)
	if b > -2 && b < 2 {
		return n
	}
	key := n.key
	heavy := n.left
	if b < 0 {
		heavy = n.right
	}
	cb := balance(heavy)
	double := (b > 0 && cb < 0) || (b < 0 && cb > 0)
	if b > 0 {
		if cb < 0 {
			n.left = rotateLeft(n.left)
		}
		n = rotateRight(n)
	} else {
		if cb > 0 {
			n.right = rotateRight(n.right)
		}
		n = rotateLeft(n)
	}
	st.fixups++
	st.rots++
	if double {
		st.rots++
	}
	if st.tracing {
		st.events = append(st.events, event{key, b, cb, before, n.h, double})
	}
	return n
}

func insert(n *node, key int, st *stats) *node {
	if n == nil {
		return &node{key: key, h: 1}
	}
	if key < n.key {
		n.left = insert(n.left, key, st)
	} else if key > n.key {
		n.right = insert(n.right, key, st)
	} else {
		return n
	}
	return rebalance(n, st)
}

// takeMin removes the smallest node of the subtree and returns the new root and its key.
func takeMin(n *node, st *stats) (*node, int) {
	if n.left == nil {
		return n.right, n.key
	}
	var m int
	n.left, m = takeMin(n.left, st)
	return rebalance(n, st), m
}

func del(n *node, key int, st *stats) *node {
	if n == nil {
		return nil
	}
	switch {
	case key < n.key:
		n.left = del(n.left, key, st)
	case key > n.key:
		n.right = del(n.right, key, st)
	default:
		if n.left == nil {
			return n.right
		}
		if n.right == nil {
			return n.left
		}
		// two children: the in-order successor takes this node's place
		n.right, n.key = takeMin(n.right, st)
	}
	return rebalance(n, st)
}

// fibTree builds the sparsest AVL tree of height h: a sparsest tree of height h-1
// on the left and one of height h-2 on the right. Keys are 2, 4, 6, ... in order,
// so every odd number is a fresh key that fits in a gap.
func fibTree(h int, next *int) *node {
	if h <= 0 {
		return nil
	}
	n := &node{h: h}
	n.left = fibTree(h-1, next)
	*next += 2
	n.key = *next
	n.right = fibTree(h-2, next)
	return n
}

func build(h int) (*node, int) {
	next := 0
	root := fibTree(h, &next)
	return root, next / 2
}

// check is the invariant checker: BST order, stored heights, |balance| <= 1, size.
// It returns a message instead of panicking.
func check(n *node, lo, hi int) (h, size int, err string) {
	if n == nil {
		return 0, 0, ""
	}
	if n.key <= lo || n.key >= hi {
		return 0, 0, fmt.Sprintf("order violated at key %d", n.key)
	}
	lh, ls, e := check(n.left, lo, n.key)
	if e != "" {
		return 0, 0, e
	}
	rh, rs, e := check(n.right, n.key, hi)
	if e != "" {
		return 0, 0, e
	}
	h = 1 + max(lh, rh)
	if n.h != h {
		return 0, 0, fmt.Sprintf("stale height at key %d", n.key)
	}
	if lh-rh > 1 || rh-lh > 1 {
		return 0, 0, fmt.Sprintf("unbalanced at key %d", n.key)
	}
	return h, ls + rs + 1, ""
}

// verify returns "" when the tree is a valid AVL tree with exactly want nodes.
func verify(root *node, want int) string {
	_, size, err := check(root, 0, math.MaxInt)
	if err == "" && size != want {
		err = fmt.Sprintf("size %d, want %d", size, want)
	}
	return err
}

func depthOf(n *node, key int) int {
	d := 0
	for n != nil {
		d++
		if key == n.key {
			return d
		}
		if key < n.key {
			n = n.left
		} else {
			n = n.right
		}
	}
	return -1
}

func shape(n *node) string {
	switch {
	case n == nil:
		return ""
	case n.left == nil && n.right == nil:
		return fmt.Sprint(n.key)
	}
	return fmt.Sprintf("%d(%s,%s)", n.key, shape(n.left), shape(n.right))
}

func shuffle(rng *rand.Rand, a []int) {
	for i := len(a) - 1; i >= 1; i-- {
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
}

// sweep is the result of applying ONE operation to a fresh copy of the sparsest
// tree, once for every candidate key, checking the invariant after each.
type sweep struct {
	n, ops                                   int // tree size, operations performed
	maxRots, maxKey, maxDepth, maxFix, total int
	bad                                      int // invariant failures
	hist                                     [16]int
}

func (s *sweep) mean() float64 { return float64(s.total) / float64(s.ops) }

// run deletes each of the n keys 2, 4, ..., 2n (inserting=false) or inserts each of
// the n+1 odd keys 1, 3, ..., 2n+1 (inserting=true), each time into a freshly built tree.
func run(h int, inserting bool) sweep {
	_, n := build(h)
	s := sweep{n: n, ops: n, maxRots: -1}
	first, want := 2, n-1
	if inserting {
		s.ops, first, want = n+1, 1, n+1
	}
	for i := 0; i < s.ops; i++ {
		key := first + 2*i
		root, _ := build(h)
		depth := depthOf(root, key)
		st := &stats{}
		if inserting {
			root = insert(root, key, st)
		} else {
			root = del(root, key, st)
		}
		if verify(root, want) != "" {
			s.bad++
		}
		s.total += st.rots
		s.maxFix = max(s.maxFix, st.fixups)
		s.hist[st.rots]++
		if st.rots > s.maxRots { // the first key to reach a new maximum
			s.maxRots, s.maxKey, s.maxDepth = st.rots, key, depth
		}
	}
	return s
}

func main() {
	hs := []int{4, 6, 8, 10, 12, 14, 16}
	const hBig = 16
	bad := 0 // invariant failures over the whole program

	fmt.Println("== (a) delete EVERY key of the sparsest AVL tree of height h, each from a fresh copy ==")
	fmt.Printf("%3s %6s %9s %8s %10s %10s %11s %6s\n", "h", "n", "max rots", "at key", "key depth", "mean rots", "max fixups", "h/2-1")
	exact := 0
	var big sweep
	for _, h := range hs {
		r := run(h, false)
		bad += r.bad
		if r.maxRots == h/2-1 {
			exact++
		}
		if h == hBig {
			big = r
		}
		fmt.Printf("%3d %6d %9d %8d %10d %10.2f %11d %6d\n", h, r.n, r.maxRots, r.maxKey, r.maxDepth, r.mean(), r.maxFix, h/2-1)
	}
	fmt.Printf("max rotations in one delete equals h/2-1 in %d of %d rows; invariant failures so far: %d\n", exact, len(hs), bad)

	fmt.Println("\n== (b) anatomy: the worst delete in the h = 6 tree, as a trace ==")
	fmt.Println("shape: key(left,right), a leaf is just its key. A fix-up is reported under the key its node holds")
	fmt.Println("after the delete: a node with two children takes its in-order successor's key.")
	root, _ := build(6)
	fmt.Printf("before: %s\n", shape(root))
	worstKey := run(6, false).maxKey
	st := &stats{tracing: true}
	root = del(root, worstKey, st)
	fmt.Printf("delete %d: %d rotations in %d fix-ups, bottom-up:\n", worstKey, st.rots, st.fixups)
	shorter := 0
	for i, e := range st.events {
		kind := "single"
		if e.double {
			kind = "double"
		}
		if e.after == e.before-1 {
			shorter++
		}
		fmt.Printf("  fix-up %d at key %-3d balance %+d, heavy child %+d -> %s rotation; subtree height %d -> %d\n",
			i+1, e.key, e.bal, e.childBal, kind, e.before, e.after)
	}
	fmt.Printf("after:  %s\n", shape(root))
	fmt.Printf("in %d of %d fix-ups the repaired subtree ended one level shorter than it was before the delete, so the node above sees a taller sibling\n",
		shorter, len(st.events))

	fmt.Printf("\n== (c) h = %d: how many rotations does one delete cost? (all %d deletes) ==\n", hBig, big.n)
	fmt.Printf("%9s %8s %7s\n", "rotations", "deletes", "share")
	for r, c := range big.hist {
		if c > 0 {
			fmt.Printf("%9d %8d %6.1f%%\n", r, c, 100*float64(c)/float64(big.n))
		}
	}

	fmt.Println("\n== (d) the same trees, INSERT every fresh odd key into a fresh copy ==")
	fmt.Printf("%3s %6s %8s %11s %9s %10s\n", "h", "n", "inserts", "max fixups", "max rots", "mean rots")
	maxInsFix, totalInserts := 0, 0
	for _, h := range hs {
		r := run(h, true)
		bad += r.bad
		maxInsFix = max(maxInsFix, r.maxFix)
		totalInserts += r.ops
		fmt.Printf("%3d %6d %8d %11d %9d %10.2f\n", h, r.n, r.ops, r.maxFix, r.maxRots, r.mean())
	}
	fmt.Printf("%d inserts into the worst-shaped trees, at most %d fix-up each; invariant failures so far: %d\n", totalInserts, maxInsFix, bad)

	fmt.Println("\n== (e) typical trees: n = 100,000, delete EVERY key (checker every 10,000 deletes) ==")
	const n = 100000
	rng := rand.New(rand.NewPCG(1, 2))
	keys := func() []int {
		a := make([]int, n)
		for i := range a {
			a[i] = i + 1
		}
		return a
	}
	insertOrder, deleteOrder, deleteOrder2 := keys(), keys(), keys()
	shuffle(rng, insertOrder)
	shuffle(rng, deleteOrder)
	shuffle(rng, deleteOrder2)
	fmt.Printf("%-24s %7s %10s %10s %9s %10s %8s\n", "inserts / deletes", "height", "total rots", "mean rots", "max rots", "max fixups", "no rots")
	var typMean float64
	var typMax, typHeavy int
	var typHist [16]int
	for i, sc := range []struct {
		name     string
		ins, del []int
	}{
		{"random / random", insertOrder, deleteOrder},
		{"ascending / random", keys(), deleteOrder2},
		{"ascending / ascending", keys(), keys()},
	} {
		var root *node
		ib := &stats{}
		for _, k := range sc.ins {
			root = insert(root, k, ib)
		}
		if verify(root, n) != "" {
			bad++
		}
		height := ht(root)
		ds := &stats{}
		maxRots, maxFix, none := 0, 0, 0
		for j, k := range sc.del {
			rots, fix := ds.rots, ds.fixups
			root = del(root, k, ds)
			maxRots = max(maxRots, ds.rots-rots)
			maxFix = max(maxFix, ds.fixups-fix)
			if ds.rots == rots {
				none++
			}
			if i == 0 {
				typHist[ds.rots-rots]++
			}
			if (j+1)%10000 == 0 && verify(root, n-j-1) != "" {
				bad++
			}
		}
		if i == 0 {
			typMean, typMax = float64(ds.rots)/n, maxRots
			for r, c := range typHist {
				if r >= 4 {
					typHeavy += c
				}
			}
		}
		fmt.Printf("%-24s %7d %10d %10.3f %9d %10d %7.1f%%\n", sc.name, height, ds.rots, float64(ds.rots)/n, maxRots, maxFix, 100*float64(none)/n)
	}
	fmt.Print("random / random, deletes by rotations:")
	for r, c := range typHist {
		if c > 0 {
			fmt.Printf("  %d: %d", r, c)
		}
	}
	fmt.Println()
	fmt.Printf("invariant failures over every delete and insert above: %d\n", bad)

	fmt.Println("\n== (f) worst case against typical case ==")
	fmt.Printf("Fibonacci tree, h = %d (n = %d): one delete costs up to %d rotations (mean %.2f); one insert never more than %d fix-up\n",
		hBig, big.n, big.maxRots, big.mean(), maxInsFix)
	fmt.Printf("random tree, n = %d: mean %.3f rotations per delete, worst single delete %d; %d deletes (%.2f%%) cost 4 or more\n",
		n, typMean, typMax, typHeavy, 100*float64(typHeavy)/n)
	if typMax >= big.maxRots {
		fmt.Printf("so the random tree contains a delete as costly as the Fibonacci tree's worst (%d vs %d), but it is rare\n", typMax, big.maxRots)
	} else {
		fmt.Printf("so no delete in the random tree reached the Fibonacci tree's worst (%d vs %d)\n", typMax, big.maxRots)
	}
	if typMean < 1 {
		fmt.Printf("worst case Theta(log n) rotations per delete (h/2-1 in %d of %d rows of (a)); typical case below one rotation (%.3f)\n", exact, len(hs), typMean)
	} else {
		fmt.Printf("worst case Theta(log n) rotations per delete; typical case %.3f\n", typMean)
	}
}
```

**Output:**

```
== (a) delete EVERY key of the sparsest AVL tree of height h, each from a fresh copy ==
  h      n  max rots   at key  key depth  mean rots  max fixups  h/2-1
  4      7         1        6          2       0.71           1      1
  6     20         2       22          3       1.25           2      2
  8     54         3       64          4       1.78           3      3
 10    143         4      174          5       2.31           4      4
 12    376         5      462          6       2.86           5      5
 14    986         6     1216          7       3.41           6      6
 16   2583         7     3190          8       3.96           7      7
max rotations in one delete equals h/2-1 in 7 of 7 rows; invariant failures so far: 0

== (b) anatomy: the worst delete in the h = 6 tree, as a trace ==
shape: key(left,right), a leaf is just its key. A fix-up is reported under the key its node holds
after the delete: a node with two children takes its in-order successor's key.
before: 26(16(10(6(4(2,),8),14(12,)),22(20(18,),24)),36(32(30(28,),34),40(38,)))
delete 22: 2 rotations in 2 fix-ups, bottom-up:
  fix-up 1 at key 24  balance +2, heavy child +1 -> single rotation; subtree height 3 -> 2
  fix-up 2 at key 16  balance +2, heavy child +1 -> single rotation; subtree height 5 -> 4
after:  26(10(6(4(2,),8),16(14(12,),20(18,24))),36(32(30(28,),34),40(38,)))
in 2 of 2 fix-ups the repaired subtree ended one level shorter than it was before the delete, so the node above sees a taller sibling

== (c) h = 16: how many rotations does one delete cost? (all 2583 deletes) ==
rotations  deletes   share
        0        2    0.1%
        1       41    1.6%
        2      222    8.6%
        3      605   23.4%
        4      870   33.7%
        5      630   24.4%
        6      196    7.6%
        7       17    0.7%

== (d) the same trees, INSERT every fresh odd key into a fresh copy ==
  h      n  inserts  max fixups  max rots  mean rots
  4      7        8           1         2       0.75
  6     20       21           1         2       0.71
  8     54       55           1         2       0.71
 10    143      144           1         2       0.71
 12    376      377           1         2       0.71
 14    986      987           1         2       0.71
 16   2583     2584           1         2       0.71
4176 inserts into the worst-shaped trees, at most 1 fix-up each; invariant failures so far: 0

== (e) typical trees: n = 100,000, delete EVERY key (checker every 10,000 deletes) ==
inserts / deletes         height total rots  mean rots  max rots max fixups  no rots
random / random               20      36902      0.369         7          4    76.3%
ascending / random            17      26329      0.263         5          3    82.3%
ascending / ascending         17      49984      0.500         1          1    50.0%
random / random, deletes by rotations:  0: 76281  1: 13088  2: 8783  3: 1220  4: 566  5: 50  6: 10  7: 2
invariant failures over every delete and insert above: 0

== (f) worst case against typical case ==
Fibonacci tree, h = 16 (n = 2583): one delete costs up to 7 rotations (mean 3.96); one insert never more than 1 fix-up
random tree, n = 100000: mean 0.369 rotations per delete, worst single delete 7; 628 deletes (0.63%) cost 4 or more
so the random tree contains a delete as costly as the Fibonacci tree's worst (7 vs 7), but it is rare
worst case Theta(log n) rotations per delete (h/2-1 in 7 of 7 rows of (a)); typical case below one rotation (0.369)
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the numbers the Go program draws (seed 1, 2).
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

// AVL with stored heights (a leaf has height 1, nil has 0). The tree is the
// recursive one from example 7: insert and delete take the subtree by value
// and return the NEW root of it, and every node on the way back up is passed
// through rebalance. That is the whole difference this example is about:
//
//   insert: the first rebalance that fires restores the subtree to its old
//           height, so nothing above it changes -> at most ONE fix-up.
//   delete: a fix-up can leave the subtree one level SHORTER than before,
//           which unbalances its parent, and so on up to the root.
struct Node {
    key: u64,
    h: i32,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

fn ht(l: &Link) -> i32 {
    l.as_ref().map_or(0, |n| n.h)
}

fn upd(n: &mut Node) {
    n.h = 1 + ht(&n.left).max(ht(&n.right));
}

fn balance(l: &Link) -> i32 {
    l.as_ref().map_or(0, |n| ht(&n.left) - ht(&n.right))
}

// A rotation is three moves through take(): Go's `y.left = x.right` has no
// direct spelling here because you cannot move a field out from behind &mut.
// let-else makes both rotations total: with no child to lift they return the node.
fn rotate_right(mut y: Box<Node>) -> Box<Node> {
    let Some(mut x) = y.left.take() else { return y };
    y.left = x.right.take();
    upd(&mut y); // the node that went DOWN first
    x.right = Some(y);
    upd(&mut x);
    x
}

fn rotate_left(mut x: Box<Node>) -> Box<Node> {
    let Some(mut y) = x.right.take() else {
        return x;
    };
    x.right = y.left.take();
    upd(&mut x);
    y.left = Some(x);
    upd(&mut y);
    y
}

// Event records one fix-up for the trace in part (b).
struct Event {
    key: u64,
    bal: i32,
    child_bal: i32,
    before: i32,
    after: i32,
    double: bool,
}

// Stats counts the repair work. A double rotation counts as 2 rotations and 1 fix-up.
#[derive(Default)]
struct Stats {
    rots: usize,
    fixups: usize,
    tracing: bool,
    events: Vec<Event>,
}

fn rebalance(mut n: Box<Node>, st: &mut Stats) -> Box<Node> {
    let before = n.h; // stale on purpose: the height this subtree had BEFORE the update
    upd(&mut n);
    let b = ht(&n.left) - ht(&n.right);
    if (-1..=1).contains(&b) {
        return n;
    }
    let key = n.key;
    let cb = balance(if b > 0 { &n.left } else { &n.right });
    let double = (b > 0 && cb < 0) || (b < 0 && cb > 0);
    if b > 0 {
        if cb < 0
            && let Some(l) = n.left.take()
        {
            n.left = Some(rotate_left(l));
        }
        n = rotate_right(n);
    } else {
        if cb > 0
            && let Some(r) = n.right.take()
        {
            n.right = Some(rotate_right(r));
        }
        n = rotate_left(n);
    }
    st.fixups += 1;
    st.rots += if double { 2 } else { 1 };
    if st.tracing {
        st.events.push(Event {
            key,
            bal: b,
            child_bal: cb,
            before,
            after: n.h,
            double,
        });
    }
    n
}

fn insert(link: Link, key: u64, st: &mut Stats) -> Box<Node> {
    let Some(mut n) = link else {
        return Box::new(Node {
            key,
            h: 1,
            left: None,
            right: None,
        });
    };
    if key < n.key {
        n.left = Some(insert(n.left.take(), key, st));
    } else if key > n.key {
        n.right = Some(insert(n.right.take(), key, st));
    } else {
        return n;
    }
    rebalance(n, st)
}

// take_min removes the smallest node of the subtree and returns the new root and its key.
fn take_min(mut n: Box<Node>, st: &mut Stats) -> (Link, u64) {
    match n.left.take() {
        None => (n.right.take(), n.key),
        Some(l) => {
            let (rest, m) = take_min(l, st);
            n.left = rest;
            (Some(rebalance(n, st)), m)
        }
    }
}

fn del(link: Link, key: u64, st: &mut Stats) -> Link {
    let mut n = link?; // deleting from an empty subtree changes nothing
    if key < n.key {
        n.left = del(n.left.take(), key, st);
    } else if key > n.key {
        n.right = del(n.right.take(), key, st);
    } else {
        match (n.left.take(), n.right.take()) {
            (None, right) => return right,
            (left, None) => return left,
            // two children: the in-order successor takes this node's place
            (left, Some(right)) => {
                let (rest, m) = take_min(right, st);
                n.left = left;
                n.right = rest;
                n.key = m;
            }
        }
    }
    Some(rebalance(n, st))
}

// fib_tree builds the sparsest AVL tree of height h: a sparsest tree of height h-1
// on the left and one of height h-2 on the right. Keys are 2, 4, 6, ... in order,
// so every odd number is a fresh key that fits in a gap.
fn fib_tree(h: i32, next: &mut u64) -> Link {
    if h <= 0 {
        return None;
    }
    let left = fib_tree(h - 1, next);
    *next += 2;
    let key = *next;
    let right = fib_tree(h - 2, next);
    Some(Box::new(Node {
        key,
        h,
        left,
        right,
    }))
}

fn build(h: i32) -> (Link, usize) {
    let mut next = 0;
    let root = fib_tree(h, &mut next);
    (root, (next / 2) as usize)
}

// check is the invariant checker: BST order, stored heights, |balance| <= 1, size.
// It returns a message instead of panicking.
fn check(l: &Link, lo: u64, hi: u64) -> Result<(i32, usize), String> {
    let Some(n) = l else { return Ok((0, 0)) };
    if n.key <= lo || n.key >= hi {
        return Err(format!("order violated at key {}", n.key));
    }
    let (lh, ls) = check(&n.left, lo, n.key)?;
    let (rh, rs) = check(&n.right, n.key, hi)?;
    let h = 1 + lh.max(rh);
    if n.h != h {
        return Err(format!("stale height at key {}", n.key));
    }
    if (lh - rh).abs() > 1 {
        return Err(format!("unbalanced at key {}", n.key));
    }
    Ok((h, ls + rs + 1))
}

// verify is true when the tree is a valid AVL tree with exactly want nodes.
fn verify(root: &Link, want: usize) -> bool {
    matches!(check(root, 0, u64::MAX), Ok((_, size)) if size == want)
}

fn depth_of(root: &Link, key: u64) -> usize {
    let (mut d, mut cur) = (0, root.as_deref());
    while let Some(n) = cur {
        d += 1;
        if key == n.key {
            return d;
        }
        cur = if key < n.key {
            n.left.as_deref()
        } else {
            n.right.as_deref()
        };
    }
    0
}

fn shape(l: &Link) -> String {
    match l {
        None => String::new(),
        Some(n) if n.left.is_none() && n.right.is_none() => n.key.to_string(),
        Some(n) => format!("{}({},{})", n.key, shape(&n.left), shape(&n.right)),
    }
}

fn shuffle(rng: &mut Pcg, a: &mut [u64]) {
    for i in (1..a.len()).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
}

// Sweep is the result of applying ONE operation to a fresh copy of the sparsest
// tree, once for every candidate key, checking the invariant after each.
#[derive(Default)]
struct Sweep {
    n: usize,
    ops: usize, // tree size, operations performed
    max_rots: usize,
    max_key: u64,
    max_depth: usize,
    max_fix: usize,
    total: usize,
    bad: usize, // invariant failures
    hist: [usize; 16],
}

impl Sweep {
    fn mean(&self) -> f64 {
        self.total as f64 / self.ops as f64
    }
}

// run deletes each of the n keys 2, 4, ..., 2n (inserting = false) or inserts each of
// the n+1 odd keys 1, 3, ..., 2n+1 (inserting = true), each time into a freshly built tree.
fn run(h: i32, inserting: bool) -> Sweep {
    let (_, n) = build(h);
    let (ops, first, want) = if inserting {
        (n + 1, 1, n + 1)
    } else {
        (n, 2, n - 1)
    };
    let mut s = Sweep {
        n,
        ops,
        ..Sweep::default()
    };
    for i in 0..ops {
        let key = first + 2 * i as u64;
        let (root, _) = build(h);
        let depth = depth_of(&root, key);
        let mut st = Stats::default();
        let root = if inserting {
            Some(insert(root, key, &mut st))
        } else {
            del(root, key, &mut st)
        };
        if !verify(&root, want) {
            s.bad += 1;
        }
        s.total += st.rots;
        s.max_fix = s.max_fix.max(st.fixups);
        s.hist[st.rots] += 1;
        if i == 0 || st.rots > s.max_rots {
            // the first key to reach a new maximum
            (s.max_rots, s.max_key, s.max_depth) = (st.rots, key, depth);
        }
    }
    s
}

fn main() {
    let hs = [4, 6, 8, 10, 12, 14, 16];
    const H_BIG: i32 = 16;
    let mut bad = 0; // invariant failures over the whole program

    println!(
        "== (a) delete EVERY key of the sparsest AVL tree of height h, each from a fresh copy =="
    );
    println!(
        "{:>3} {:>6} {:>9} {:>8} {:>10} {:>10} {:>11} {:>6}",
        "h", "n", "max rots", "at key", "key depth", "mean rots", "max fixups", "h/2-1"
    );
    let mut exact = 0;
    let mut big = Sweep::default();
    for h in hs {
        let r = run(h, false);
        bad += r.bad;
        if r.max_rots == h as usize / 2 - 1 {
            exact += 1;
        }
        println!(
            "{:>3} {:>6} {:>9} {:>8} {:>10} {:>10.2} {:>11} {:>6}",
            h,
            r.n,
            r.max_rots,
            r.max_key,
            r.max_depth,
            r.mean(),
            r.max_fix,
            h / 2 - 1
        );
        if h == H_BIG {
            big = r;
        }
    }
    println!(
        "max rotations in one delete equals h/2-1 in {exact} of {} rows; invariant failures so far: {bad}",
        hs.len()
    );

    println!("\n== (b) anatomy: the worst delete in the h = 6 tree, as a trace ==");
    println!(
        "shape: key(left,right), a leaf is just its key. A fix-up is reported under the key its node holds"
    );
    println!("after the delete: a node with two children takes its in-order successor's key.");
    let (root, _) = build(6);
    println!("before: {}", shape(&root));
    let worst_key = run(6, false).max_key;
    let mut st = Stats {
        tracing: true,
        ..Stats::default()
    };
    let root = del(root, worst_key, &mut st);
    println!(
        "delete {worst_key}: {} rotations in {} fix-ups, bottom-up:",
        st.rots, st.fixups
    );
    let mut shorter = 0;
    for (i, e) in st.events.iter().enumerate() {
        let kind = if e.double { "double" } else { "single" };
        if e.after == e.before - 1 {
            shorter += 1;
        }
        println!(
            "  fix-up {} at key {:<3} balance {:+}, heavy child {:+} -> {kind} rotation; subtree height {} -> {}",
            i + 1,
            e.key,
            e.bal,
            e.child_bal,
            e.before,
            e.after
        );
    }
    println!("after:  {}", shape(&root));
    println!(
        "in {shorter} of {} fix-ups the repaired subtree ended one level shorter than it was before the delete, so the node above sees a taller sibling",
        st.events.len()
    );

    println!(
        "\n== (c) h = {H_BIG}: how many rotations does one delete cost? (all {} deletes) ==",
        big.n
    );
    println!("{:>9} {:>8} {:>7}", "rotations", "deletes", "share");
    for (r, &c) in big.hist.iter().enumerate() {
        if c > 0 {
            println!(
                "{:>9} {:>8} {:>6.1}%",
                r,
                c,
                100.0 * c as f64 / big.n as f64
            );
        }
    }

    println!("\n== (d) the same trees, INSERT every fresh odd key into a fresh copy ==");
    println!(
        "{:>3} {:>6} {:>8} {:>11} {:>9} {:>10}",
        "h", "n", "inserts", "max fixups", "max rots", "mean rots"
    );
    let (mut max_ins_fix, mut total_inserts) = (0, 0);
    for h in hs {
        let r = run(h, true);
        bad += r.bad;
        max_ins_fix = max_ins_fix.max(r.max_fix);
        total_inserts += r.ops;
        println!(
            "{:>3} {:>6} {:>8} {:>11} {:>9} {:>10.2}",
            h,
            r.n,
            r.ops,
            r.max_fix,
            r.max_rots,
            r.mean()
        );
    }
    println!(
        "{total_inserts} inserts into the worst-shaped trees, at most {max_ins_fix} fix-up each; invariant failures so far: {bad}"
    );

    println!(
        "\n== (e) typical trees: n = 100,000, delete EVERY key (checker every 10,000 deletes) =="
    );
    const N: usize = 100000;
    let mut rng = Pcg::new(1, 2);
    let keys = || -> Vec<u64> { (1..=N as u64).collect() };
    let (mut insert_order, mut delete_order, mut delete_order2) = (keys(), keys(), keys());
    shuffle(&mut rng, &mut insert_order);
    shuffle(&mut rng, &mut delete_order);
    shuffle(&mut rng, &mut delete_order2);
    println!(
        "{:<24} {:>7} {:>10} {:>10} {:>9} {:>10} {:>8}",
        "inserts / deletes",
        "height",
        "total rots",
        "mean rots",
        "max rots",
        "max fixups",
        "no rots"
    );
    let (mut typ_mean, mut typ_max, mut typ_heavy) = (0.0, 0, 0);
    let mut typ_hist = [0usize; 16];
    let scenarios = [
        ("random / random", insert_order, delete_order),
        ("ascending / random", keys(), delete_order2),
        ("ascending / ascending", keys(), keys()),
    ];
    for (i, (name, ins, dels)) in scenarios.iter().enumerate() {
        let mut root: Link = None;
        let mut ib = Stats::default();
        for &k in ins {
            root = Some(insert(root, k, &mut ib));
        }
        if !verify(&root, N) {
            bad += 1;
        }
        let height = ht(&root);
        let mut ds = Stats::default();
        let (mut max_rots, mut max_fix, mut none) = (0, 0, 0);
        for (j, &k) in dels.iter().enumerate() {
            let (rots, fix) = (ds.rots, ds.fixups);
            root = del(root, k, &mut ds);
            max_rots = max_rots.max(ds.rots - rots);
            max_fix = max_fix.max(ds.fixups - fix);
            if ds.rots == rots {
                none += 1;
            }
            if i == 0 {
                typ_hist[ds.rots - rots] += 1;
            }
            if (j + 1) % 10000 == 0 && !verify(&root, N - j - 1) {
                bad += 1;
            }
        }
        if i == 0 {
            typ_mean = ds.rots as f64 / N as f64;
            typ_max = max_rots;
            typ_heavy = typ_hist[4..].iter().sum::<usize>();
        }
        println!(
            "{:<24} {:>7} {:>10} {:>10.3} {:>9} {:>10} {:>7.1}%",
            name,
            height,
            ds.rots,
            ds.rots as f64 / N as f64,
            max_rots,
            max_fix,
            100.0 * none as f64 / N as f64
        );
    }
    print!("random / random, deletes by rotations:");
    for (r, &c) in typ_hist.iter().enumerate() {
        if c > 0 {
            print!("  {r}: {c}");
        }
    }
    println!();
    println!("invariant failures over every delete and insert above: {bad}");

    println!("\n== (f) worst case against typical case ==");
    println!(
        "Fibonacci tree, h = {H_BIG} (n = {}): one delete costs up to {} rotations (mean {:.2}); one insert never more than {max_ins_fix} fix-up",
        big.n,
        big.max_rots,
        big.mean()
    );
    println!(
        "random tree, n = {N}: mean {typ_mean:.3} rotations per delete, worst single delete {typ_max}; {typ_heavy} deletes ({:.2}%) cost 4 or more",
        100.0 * typ_heavy as f64 / N as f64
    );
    if typ_max >= big.max_rots {
        println!(
            "so the random tree contains a delete as costly as the Fibonacci tree's worst ({typ_max} vs {}), but it is rare",
            big.max_rots
        );
    } else {
        println!(
            "so no delete in the random tree reached the Fibonacci tree's worst ({typ_max} vs {})",
            big.max_rots
        );
    }
    if typ_mean < 1.0 {
        println!(
            "worst case Theta(log n) rotations per delete (h/2-1 in {exact} of {} rows of (a)); typical case below one rotation ({typ_mean:.3})",
            hs.len()
        );
    } else {
        println!("worst case Theta(log n) rotations per delete; typical case {typ_mean:.3}");
    }
}
```

**Rust output:**

```
== (a) delete EVERY key of the sparsest AVL tree of height h, each from a fresh copy ==
  h      n  max rots   at key  key depth  mean rots  max fixups  h/2-1
  4      7         1        6          2       0.71           1      1
  6     20         2       22          3       1.25           2      2
  8     54         3       64          4       1.78           3      3
 10    143         4      174          5       2.31           4      4
 12    376         5      462          6       2.86           5      5
 14    986         6     1216          7       3.41           6      6
 16   2583         7     3190          8       3.96           7      7
max rotations in one delete equals h/2-1 in 7 of 7 rows; invariant failures so far: 0

== (b) anatomy: the worst delete in the h = 6 tree, as a trace ==
shape: key(left,right), a leaf is just its key. A fix-up is reported under the key its node holds
after the delete: a node with two children takes its in-order successor's key.
before: 26(16(10(6(4(2,),8),14(12,)),22(20(18,),24)),36(32(30(28,),34),40(38,)))
delete 22: 2 rotations in 2 fix-ups, bottom-up:
  fix-up 1 at key 24  balance +2, heavy child +1 -> single rotation; subtree height 3 -> 2
  fix-up 2 at key 16  balance +2, heavy child +1 -> single rotation; subtree height 5 -> 4
after:  26(10(6(4(2,),8),16(14(12,),20(18,24))),36(32(30(28,),34),40(38,)))
in 2 of 2 fix-ups the repaired subtree ended one level shorter than it was before the delete, so the node above sees a taller sibling

== (c) h = 16: how many rotations does one delete cost? (all 2583 deletes) ==
rotations  deletes   share
        0        2    0.1%
        1       41    1.6%
        2      222    8.6%
        3      605   23.4%
        4      870   33.7%
        5      630   24.4%
        6      196    7.6%
        7       17    0.7%

== (d) the same trees, INSERT every fresh odd key into a fresh copy ==
  h      n  inserts  max fixups  max rots  mean rots
  4      7        8           1         2       0.75
  6     20       21           1         2       0.71
  8     54       55           1         2       0.71
 10    143      144           1         2       0.71
 12    376      377           1         2       0.71
 14    986      987           1         2       0.71
 16   2583     2584           1         2       0.71
4176 inserts into the worst-shaped trees, at most 1 fix-up each; invariant failures so far: 0

== (e) typical trees: n = 100,000, delete EVERY key (checker every 10,000 deletes) ==
inserts / deletes         height total rots  mean rots  max rots max fixups  no rots
random / random               20      36902      0.369         7          4    76.3%
ascending / random            17      26329      0.263         5          3    82.3%
ascending / ascending         17      49984      0.500         1          1    50.0%
random / random, deletes by rotations:  0: 76281  1: 13088  2: 8783  3: 1220  4: 566  5: 50  6: 10  7: 2
invariant failures over every delete and insert above: 0

== (f) worst case against typical case ==
Fibonacci tree, h = 16 (n = 2583): one delete costs up to 7 rotations (mean 3.96); one insert never more than 1 fix-up
random tree, n = 100000: mean 0.369 rotations per delete, worst single delete 7; 628 deletes (0.63%) cost 4 or more
so the random tree contains a delete as costly as the Fibonacci tree's worst (7 vs 7), but it is rare
worst case Theta(log n) rotations per delete (h/2-1 in 7 of 7 rows of (a)); typical case below one rotation (0.369)
```

> **Go vs Rust:** The output is byte-for-byte the Go output: same PCG stream, same algorithm, same counts. Go's `y.left = x.right` becomes a `take()` of each child out of its box and a put back, `rotate_right` is total (`let ... else { return y }`), and the nil case of `del` is `link?`; each call returns the new subtree root, so the node being repaired can be replaced without a `&mut` to its parent. The recursive `Drop` of these trees is safe (height at most 20 here), unlike the degenerate chains of lesson 17 example 13.

**Complexity:** one AVL delete costs Θ(log n) rotations in the worst case: on the sparsest tree of height h the worst delete costs exactly h/2 − 1 rotations (1 at h = 4 up to 7 at h = 16 with 2,583 nodes, mean 0.71 up to 3.96), each repair leaving its subtree one level shorter, while all 4,176 inserts into those same trees needed at most 1 fix-up (at most 2 rotations). Typical trees are much cheaper: a random 100,000-node tree pays 0.369 rotations per delete (76.3% of deletes need none, 0.63% need four or more), yet its single worst delete, 7 rotations, equals the Fibonacci tree's worst, so the bad case exists in ordinary trees and is merely rare. Deleting an ascending-built tree in ascending order costs exactly 0.5 per delete, never more than 1 rotation.

---

## 13. B+ tree: the leaf chain, and what it does not buy

`🔴 hard` · *B+ tree*

A B+ tree is the database variant of the B-tree: entries live only in the leaves, inner pages hold nothing but separator keys and child ids (so more of them fit in a page), and the leaves are linked so a range scan is a walk along the bottom. This example measures honestly what that buys and what it does not, because a leaf chain is only *physically sequential* if the tree was bulk-loaded. The page sizes are stated assumptions (4,096-byte pages, 8-byte keys and values, 4-byte page ids), a model and not any real engine, and the cost is counted in pages read and in seeks (a read of a page whose id is not the previous id + 1), so every number is exact and identical on every run. A lookup costs Θ(log_f n) pages for fan-out f, and a scan of k keys costs the descent plus about k divided by the leaf fill.

**Steps:**

1. Derive the page capacities from the stated sizes (203 keys per B-tree node, 339 per B+ inner page, 254 per B+ leaf) and build, from the same 1,000,000 keys, a B-tree, two incrementally built B+ trees (random and ascending insertion order) and a bulk-loaded B+ tree, each with its own checker.
2. The key move: run 100,000 point lookups and 1,000 range scans for each of k = 1, 100 and 10,000 on every structure, counting distinct pages and seeks and checking every scan against the sorted slice; then walk each leaf chain and count the steps where `next` is `id + 1`.
3. Read the lookup table (a B+ lookup is exactly `levels` pages), the scan table (page counts move, but seeks only collapse where the chain is adjacent) and the chain table (which build produces an adjacent chain).

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// ---- the page model: STATED ASSUMPTIONS, not any real engine ----
// Real pages have slot arrays, variable-length keys and checksums. Here a page is
// 4,096 bytes, its header 16, a key 8, a value 8, a page id 4. One extra id is
// always present (a node with n keys has n+1 children; a leaf has a next link),
// so the room for entries is pageSize - header - idB. Everything below is
// integer arithmetic on those numbers.
const (
	pageSize = 4096
	header   = 16
	keyB     = 8
	valB     = 8
	idB      = 4
	usable   = pageSize - header - idB

	bMax     = usable / (keyB + valB + idB) // B-tree node: key + value + child id per key
	innerMax = usable / (keyB + idB)        // B+ inner page: separator key + child id (no values)
	leafMax  = usable / (keyB + valB)       // B+ leaf: key + value (no child ids)
	bT       = (bMax + 1) / 2               // CLRS minimum degree: bMax = 2t-1
	bulkLeaf = leafMax * 9 / 10             // a bulk load fills leaves to 90%
	bulkKids = (innerMax + 1) * 9 / 10      // ... and inner pages to 90% of their children
	minLeaf  = leafMax / 2                  // smallest leaf a split leaves behind
	minInner = innerMax / 2                 // smallest inner page (keys) a split leaves behind
)

// Entries are keys only: the 8 value bytes are paid for in the capacity
// arithmetic above, not stored, so a "page" here is a slice of keys.

// ---- one page type for both trees; a leaf is a page with no children ----
// The arena index IS the page id, which is exactly the disk model.
type page struct {
	keys []int   // B-tree: keys; B+ leaf: the entries; B+ inner: separators
	kids []int32 // nil for a leaf, otherwise len(keys)+1 children
	next int32   // B+ leaf only: the next leaf in key order, -1 at the end
}

type tree struct {
	pages []page
	root  int32
}

func (t *tree) add(p page) int32 {
	t.pages = append(t.pages, p)
	return int32(len(t.pages) - 1)
}

// shape returns the number of levels, the number of leaf pages and the total
// number of keys stored in pages (separators included).
func (t *tree) shape() (levels, leaves, keys int) {
	for _, p := range t.pages {
		if p.kids == nil {
			leaves++
		}
		keys += len(p.keys)
	}
	for id := t.root; ; id = t.pages[id].kids[0] {
		levels++
		if t.pages[id].kids == nil {
			return levels, leaves, keys
		}
	}
}

// check: order inside pages and against the separators, page sizes, one leaf
// depth, the entry total == want; for a B+ tree also that the leaf chain visits
// exactly the leaves in tree order and ends. A message, never a panic.
// A B-tree separator is a key of the tree (children lie strictly between
// separators); a B+ separator is a COPY of the first key to its right (>=).
func (t *tree) check(want int, plus bool) string {
	leafDepth, total := -1, 0
	var leaves []int32
	var walk func(id int32, lo, hi, depth int) string
	walk = func(id int32, lo, hi, depth int) string {
		p := &t.pages[id]
		leaf := p.kids == nil
		minK, maxK := bT-1, bMax
		if plus && leaf {
			minK, maxK = minLeaf, leafMax
		} else if plus {
			minK, maxK = minInner, innerMax
		}
		if len(p.keys) > maxK || (id != t.root && len(p.keys) < minK) {
			return fmt.Sprintf("page %d holds %d keys", id, len(p.keys))
		}
		for i, k := range p.keys {
			if k < lo || k >= hi || (i > 0 && k <= p.keys[i-1]) {
				return fmt.Sprintf("page %d: key %d out of order", id, k)
			}
		}
		if !plus || leaf {
			total += len(p.keys)
		}
		if leaf {
			if leafDepth >= 0 && leafDepth != depth {
				return fmt.Sprintf("leaf %d at depth %d, others at %d", id, depth, leafDepth)
			}
			leafDepth = depth
			leaves = append(leaves, id)
			return ""
		}
		if len(p.kids) != len(p.keys)+1 {
			return fmt.Sprintf("page %d: %d keys, %d children", id, len(p.keys), len(p.kids))
		}
		for i, c := range p.kids {
			l, h := lo, hi
			if i > 0 {
				l = p.keys[i-1]
				if !plus {
					l++
				}
			}
			if i < len(p.keys) {
				h = p.keys[i]
			}
			if e := walk(c, l, h, depth+1); e != "" {
				return e
			}
		}
		return ""
	}
	if e := walk(t.root, 0, 1<<62, 1); e != "" {
		return e
	}
	for i, id := range leaves {
		next := int32(-1)
		if i+1 < len(leaves) {
			next = leaves[i+1]
		}
		if plus && t.pages[id].next != next {
			return fmt.Sprintf("leaf %d: next is %d, tree order says %d", id, t.pages[id].next, next)
		}
	}
	if total != want {
		return fmt.Sprintf("%d entries, want %d", total, want)
	}
	return ""
}

// ---- B-tree (example 11's, proactive split, here with capacity 203) ----
type btree struct{ tree }

func newBTree() *btree { return &btree{tree{pages: []page{{next: -1}}}} }

func (t *btree) splitChild(x int32, i int) {
	y := t.pages[x].kids[i]
	mid := bT - 1 // index of the median of a full node (2t-1 keys)
	yk := t.pages[y].keys
	median := yk[mid]
	right := page{keys: slices.Clone(yk[mid+1:]), next: -1}
	if t.pages[y].kids != nil {
		right.kids = slices.Clone(t.pages[y].kids[mid+1:])
		t.pages[y].kids = t.pages[y].kids[:mid+1]
	}
	t.pages[y].keys = yk[:mid]
	z := t.add(right)
	px := &t.pages[x]
	px.keys = slices.Insert(px.keys, i, median)
	px.kids = slices.Insert(px.kids, i+1, z)
}

func (t *btree) insert(key int) {
	if len(t.pages[t.root].keys) == bMax {
		old := t.root
		t.root = t.add(page{kids: []int32{old}, next: -1})
		t.splitChild(t.root, 0)
	}
	x := t.root
	for {
		n := &t.pages[x]
		i, found := slices.BinarySearch(n.keys, key)
		if found {
			return
		}
		if n.kids == nil {
			n.keys = slices.Insert(n.keys, i, key)
			return
		}
		if len(t.pages[n.kids[i]].keys) == bMax {
			t.splitChild(x, i)
			n = &t.pages[x] // the arena may have moved
			if key > n.keys[i] {
				i++
			}
		}
		x = n.kids[i]
	}
}

// ---- B+ tree: entries only in leaves, separators only in inner pages ----
type bplus struct{ tree }

func newBPlus() *bplus { return &bplus{tree{pages: []page{{next: -1}}}} }

// childIndex: a separator is the FIRST key of the right subtree, so a key equal
// to a separator goes right.
func childIndex(keys []int, key int) int {
	i, found := slices.BinarySearch(keys, key)
	if found {
		i++
	}
	return i
}

// insert descends and returns (separator, new page, true) when the page split.
// A leaf split COPIES the separator up (it stays in the leaf); an inner split
// PUSHES it up (it leaves the page). Pages may move when the arena grows, so
// the pointer is taken again after every recursive call.
func (t *bplus) insert(id int32, key int) (int, int32, bool) {
	p := &t.pages[id]
	if p.kids == nil {
		i, found := slices.BinarySearch(p.keys, key)
		if found {
			return 0, 0, false
		}
		p.keys = slices.Insert(p.keys, i, key)
		if len(p.keys) <= leafMax {
			return 0, 0, false
		}
		mid := len(p.keys) / 2
		right := page{keys: slices.Clone(p.keys[mid:]), next: p.next}
		p.keys = p.keys[:mid]
		rid := t.add(right)
		t.pages[id].next = rid
		return right.keys[0], rid, true
	}
	i := childIndex(p.keys, key)
	sep, rid, split := t.insert(p.kids[i], key)
	if !split {
		return 0, 0, false
	}
	p = &t.pages[id]
	p.keys = slices.Insert(p.keys, i, sep)
	p.kids = slices.Insert(p.kids, i+1, rid)
	if len(p.keys) <= innerMax {
		return 0, 0, false
	}
	mid := len(p.keys) / 2
	up := p.keys[mid]
	right := page{keys: slices.Clone(p.keys[mid+1:]), kids: slices.Clone(p.kids[mid+1:]), next: -1}
	p.keys, p.kids = p.keys[:mid], p.kids[:mid+1]
	return up, t.add(right), true
}

func (t *bplus) Insert(key int) {
	if sep, rid, split := t.insert(t.root, key); split {
		t.root = t.add(page{keys: []int{sep}, kids: []int32{t.root, rid}, next: -1})
	}
}

// bulkLoad builds the tree bottom-up from SORTED keys: leaves are written left
// to right (so their ids ascend and each next is id+1), then each inner level
// is built over the one below. Entries are spread evenly over the fewest pages
// that respect the 90% fill, so no page is a stub.
func bulkLoad(sorted []int) *bplus {
	type ref struct {
		id  int32
		min int
	}
	// spread splits total items into the fewest groups of at most per items, as evenly as possible
	spread := func(total, per int) (groups int, size func(g int) int) {
		groups = (total + per - 1) / per
		return groups, func(g int) int {
			if g < total%groups {
				return total/groups + 1
			}
			return total / groups
		}
	}
	t := &bplus{}
	var level []ref
	groups, size := spread(len(sorted), bulkLeaf)
	pos := 0
	for g := 0; g < groups; g++ {
		keys := slices.Clone(sorted[pos : pos+size(g)])
		pos += len(keys)
		next := int32(len(t.pages) + 1)
		if g == groups-1 {
			next = -1
		}
		level = append(level, ref{t.add(page{keys: keys, next: next}), keys[0]})
	}
	for len(level) > 1 {
		groups, size := spread(len(level), bulkKids)
		var up []ref
		pos := 0
		for g := 0; g < groups; g++ {
			kids := level[pos : pos+size(g)]
			pos += len(kids)
			p := page{next: -1}
			for i, c := range kids {
				p.kids = append(p.kids, c.id)
				if i > 0 {
					p.keys = append(p.keys, c.min)
				}
			}
			up = append(up, ref{t.add(p), kids[0].min})
		}
		level = up
	}
	t.root = level[0].id
	return t
}

// ---- measuring ----
// cost counts the pages a lookup or a scan reads. A SEEK is a read of a page
// whose id is not the previous page's id + 1: the model says ids are disk
// addresses, so id+1 is the next sector and anything else moves the head.
type cost struct {
	pages, seeks int
	prev         int32
}

func (c *cost) read(id int32) {
	c.pages++
	if id != c.prev+1 {
		c.seeks++
	}
	c.prev = id
}

func (t *btree) find(key int) (found bool, pages int) {
	id := t.root
	for {
		pages++
		n := &t.pages[id]
		i, ok := slices.BinarySearch(n.keys, key)
		if ok {
			return true, pages
		}
		if n.kids == nil {
			return false, pages
		}
		id = n.kids[i]
	}
}

func (t *bplus) find(key int) (found bool, pages int) {
	id := t.root
	for {
		pages++
		p := &t.pages[id]
		if p.kids == nil {
			_, ok := slices.BinarySearch(p.keys, key)
			return ok, pages
		}
		id = p.kids[childIndex(p.keys, key)]
	}
}

// scan returns the k keys >= start in order. B-tree: an in-order traversal with
// an explicit stack of (page, next key index); a page is counted when it is
// pushed, and a page is pushed once per scan, so pages = distinct pages. The
// seeks of a B-tree scan ignore re-reads of a parent between its children,
// which flatters it.
func (t *btree) scan(start, k int, out []int) ([]int, cost) {
	out = out[:0]
	c := cost{prev: -2}
	type frame struct {
		id int32
		i  int
	}
	var stack []frame
	id := t.root
	for {
		c.read(id)
		n := &t.pages[id]
		i, found := slices.BinarySearch(n.keys, start)
		stack = append(stack, frame{id, i})
		if n.kids == nil || found {
			break
		}
		id = n.kids[i]
	}
	for len(out) < k && len(stack) > 0 {
		f := &stack[len(stack)-1]
		n := &t.pages[f.id]
		if f.i >= len(n.keys) {
			stack = stack[:len(stack)-1]
			continue
		}
		out = append(out, n.keys[f.i])
		f.i++
		if n.kids != nil {
			id := n.kids[f.i]
			for {
				c.read(id)
				stack = append(stack, frame{id, 0})
				if t.pages[id].kids == nil {
					break
				}
				id = t.pages[id].kids[0]
			}
		}
	}
	return out, c
}

// scan on a B+ tree: one descent to the leaf, then a walk along the leaf chain.
func (t *bplus) scan(start, k int, out []int) ([]int, cost) {
	out = out[:0]
	c := cost{prev: -2}
	id := t.root
	for {
		c.read(id)
		p := &t.pages[id]
		if p.kids == nil {
			break
		}
		id = p.kids[childIndex(p.keys, start)]
	}
	i, _ := slices.BinarySearch(t.pages[id].keys, start)
	for {
		keys := t.pages[id].keys
		for ; i < len(keys) && len(out) < k; i++ {
			out = append(out, keys[i])
		}
		next := t.pages[id].next
		if len(out) == k || next < 0 {
			return out, c
		}
		id, i = next, 0
		c.read(id)
	}
}

type index interface {
	find(key int) (bool, int)
	scan(start, k int, out []int) ([]int, cost)
}

func shuffle(rng *rand.Rand, a []int) {
	for i := len(a) - 1; i >= 1; i-- {
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
}

// chain walks the leaf chain from the leftmost leaf and counts the steps whose
// next id is exactly id+1.
func (t *tree) chain() (steps, adjacent int) {
	id := t.root
	for t.pages[id].kids != nil {
		id = t.pages[id].kids[0]
	}
	for t.pages[id].next >= 0 {
		steps++
		if t.pages[id].next == id+1 {
			adjacent++
		}
		id = t.pages[id].next
	}
	return steps, adjacent
}

func main() {
	fmt.Println("== the page model (assumptions, not a real engine: real pages have slot arrays and variable-length keys) ==")
	fmt.Printf("page %d B, header %d B, key %d B, value %d B, page id %d B; room for entries = %d - %d - %d = %d B\n",
		pageSize, header, keyB, valB, idB, pageSize, header, idB, usable)
	fmt.Printf("B-tree node       : %d / (%d+%d+%d) = %d keys   (key, value and child id in every node)\n", usable, keyB, valB, idB, bMax)
	fmt.Printf("B+ inner page     : %d / (%d+%d)    = %d keys   (separator and child id, no values)\n", usable, keyB, idB, innerMax)
	fmt.Printf("B+ leaf page      : %d / (%d+%d)    = %d keys   (key and value, no child ids, plus a next link)\n", usable, keyB, valB, leafMax)
	fmt.Printf("bulk load fills leaves to %d of %d and inner pages to %d of %d children (90%%)\n", bulkLeaf, leafMax, bulkKids, innerMax+1)

	fmt.Println("\n== on paper: the most entries a tree of L levels can hold, every page full ==")
	fmt.Printf("%6s %16s %16s %7s\n", "levels", "B-tree", "B+ tree", "ratio")
	for l := 1; l <= 4; l++ {
		b, bp := 1, leafMax
		for i := 0; i < l; i++ {
			b *= bMax + 1
		}
		b--
		for i := 1; i < l; i++ {
			bp *= innerMax + 1
		}
		fmt.Printf("%6d %16d %16d %7.2f\n", l, b, bp, float64(bp)/float64(b))
	}

	const n = 1000000
	rng := rand.New(rand.NewPCG(1, 2))
	sorted := make([]int, n)
	for i := range sorted {
		sorted[i] = i + 1
	}
	order := slices.Clone(sorted)
	shuffle(rng, order)

	bt := newBTree()
	incr, asc := newBPlus(), newBPlus()
	for i, k := range order {
		bt.insert(k)
		incr.Insert(k)
		asc.Insert(sorted[i])
	}
	bulk := bulkLoad(sorted)

	fmt.Printf("\n== built from the same %d keys (PCG(1,2) shuffle; 'ascending' inserts them in sorted order) ==\n", n)
	fmt.Printf("%-24s %6s %7s %7s %7s %8s %11s %9s  %s\n", "structure", "levels", "pages", "leaves", "inner", "fill", "entries/pg", "index MB", "checker")
	type entry struct {
		name string
		idx  index
		t    *tree
		plus bool
	}
	all := []entry{
		{"B-tree (random)", bt, &bt.tree, false},
		{"B+ incremental (random)", incr, &incr.tree, true},
		{"B+ incremental (asc)", asc, &asc.tree, true},
		{"B+ bulk-loaded", bulk, &bulk.tree, true},
	}
	for _, e := range all {
		levels, leaves, keys := e.t.shape()
		pages := len(e.t.pages)
		fill := float64(keys) / float64(pages*bMax)
		if e.plus {
			fill = float64(n) / float64(leaves*leafMax)
		}
		verdict := e.t.check(n, e.plus)
		if verdict == "" {
			verdict = "OK"
		}
		fmt.Printf("%-24s %6d %7d %7d %7d %7.1f%% %11.1f %9.2f  %s\n", e.name, levels, pages, leaves, pages-leaves,
			100*fill, float64(n)/float64(pages), float64(pages*pageSize)/(1<<20), verdict)
	}
	fmt.Printf("fill: B-tree = keys / (pages x %d); B+ = entries / (leaves x %d)\n", bMax, leafMax)

	// (a) point lookups
	fmt.Println("\n== (a) 100,000 point lookups of present keys: pages read per lookup ==")
	probes := make([]int, 100000)
	for i := range probes {
		probes[i] = sorted[rng.IntN(n)]
	}
	fmt.Printf("%-24s %5s %7s %5s %9s\n", "structure", "min", "mean", "max", "not found")
	for _, e := range all {
		lo, hi, sum, missing := 1<<30, 0, 0, 0
		for _, k := range probes {
			ok, pg := e.idx.find(k)
			if !ok {
				missing++
			}
			lo, hi, sum = min(lo, pg), max(hi, pg), sum+pg
		}
		fmt.Printf("%-24s %5d %7.3f %5d %9d\n", e.name, lo, float64(sum)/float64(len(probes)), hi, missing)
	}

	// (b) range scans
	fmt.Println("\n== (b) range scans: 1,000 random starts, k keys each, every scan checked against the sorted slice ==")
	fmt.Println("pages = distinct pages read; seeks = reads whose id is not previous id + 1 (first read counts)")
	fmt.Printf("%7s %-24s %10s %10s %9s %9s\n", "k", "structure", "pages/scan", "seeks/scan", "pages vs", "mismatch")
	var out []int
	for _, k := range []int{1, 100, 10000} {
		starts := make([]int, 1000)
		for i := range starts {
			starts[i] = rng.IntN(n - k + 1)
		}
		var base float64
		for j, e := range all {
			pages, seeks, bad := 0, 0, 0
			for _, r := range starts {
				var c cost
				out, c = e.idx.scan(sorted[r], k, out)
				pages += c.pages
				seeks += c.seeks
				if !slices.Equal(out, sorted[r:r+k]) {
					bad++
				}
			}
			mean := float64(pages) / float64(len(starts))
			if j == 0 {
				base = mean
			}
			fmt.Printf("%7d %-24s %10.2f %10.2f %8.2fx %9d\n", k, e.name, mean, float64(seeks)/float64(len(starts)), mean/base, bad)
		}
	}
	fmt.Println("'pages vs' is pages/scan divided by the B-tree's")

	// (c) the leaf chain
	fmt.Println("\n== (c) leaf chain: is next == id + 1? (walk the whole chain, leftmost leaf to the end) ==")
	fmt.Printf("%-24s %9s %9s %9s %17s\n", "structure", "steps", "adjacent", "fraction", "seeks, full scan")
	var fullSeeks []int
	for _, e := range all[1:] {
		steps, adj := e.t.chain()
		fullSeeks = append(fullSeeks, steps-adj+1)
		fmt.Printf("%-24s %9d %9d %9.3f %17d\n", e.name, steps, adj, float64(adj)/float64(steps), steps-adj+1)
	}
	fmt.Printf("a full scan of the leaf level costs %d seeks on the randomly built tree, %d on the ascending one and %d on the bulk-loaded one\n",
		fullSeeks[0], fullSeeks[1], fullSeeks[2])
}
```

**Output:**

```
== the page model (assumptions, not a real engine: real pages have slot arrays and variable-length keys) ==
page 4096 B, header 16 B, key 8 B, value 8 B, page id 4 B; room for entries = 4096 - 16 - 4 = 4076 B
B-tree node       : 4076 / (8+8+4) = 203 keys   (key, value and child id in every node)
B+ inner page     : 4076 / (8+4)    = 339 keys   (separator and child id, no values)
B+ leaf page      : 4076 / (8+8)    = 254 keys   (key and value, no child ids, plus a next link)
bulk load fills leaves to 228 of 254 and inner pages to 306 of 340 children (90%)

== on paper: the most entries a tree of L levels can hold, every page full ==
levels           B-tree          B+ tree   ratio
     1              203              254    1.25
     2            41615            86360    2.08
     3          8489663         29362400    3.46
     4       1731891455       9983216000    5.76

== built from the same 1000000 keys (PCG(1,2) shuffle; 'ascending' inserts them in sorted order) ==
structure                levels   pages  leaves   inner     fill  entries/pg  index MB  checker
B-tree (random)               3    7285    7220      65    67.6%       137.3     28.46  OK
B+ incremental (random)       3    5664    5636      28    69.9%       176.6     22.12  OK
B+ incremental (asc)          3    7921    7874      47    50.0%       126.2     30.94  OK
B+ bulk-loaded                3    4402    4386      16    89.8%       227.2     17.20  OK
fill: B-tree = keys / (pages x 203); B+ = entries / (leaves x 254)

== (a) 100,000 point lookups of present keys: pages read per lookup ==
structure                  min    mean   max not found
B-tree (random)              1   2.993     3         0
B+ incremental (random)      3   3.000     3         0
B+ incremental (asc)         3   3.000     3         0
B+ bulk-loaded               3   3.000     3         0

== (b) range scans: 1,000 random starts, k keys each, every scan checked against the sorted slice ==
pages = distinct pages read; seeks = reads whose id is not previous id + 1 (first read counts)
      k structure                pages/scan seeks/scan  pages vs  mismatch
      1 B-tree (random)                3.00       2.99     1.00x         0
      1 B+ incremental (random)        3.00       3.00     1.00x         0
      1 B+ incremental (asc)           3.00       2.99     1.00x         0
      1 B+ bulk-loaded                 3.00       3.00     1.00x         0
    100 B-tree (random)                3.72       3.70     1.00x         0
    100 B+ incremental (random)        3.55       3.55     0.96x         0
    100 B+ incremental (asc)           3.79       3.00     1.02x         0
    100 B+ bulk-loaded                 3.44       3.00     0.93x         0
  10000 B-tree (random)               75.79      75.77     1.00x         0
  10000 B+ incremental (random)       59.17      59.17     0.78x         0
  10000 B+ incremental (asc)          81.74       3.46     1.08x         0
  10000 B+ bulk-loaded                46.84       3.00     0.62x         0
'pages vs' is pages/scan divided by the B-tree's

== (c) leaf chain: is next == id + 1? (walk the whole chain, leftmost leaf to the end) ==
structure                    steps  adjacent  fraction  seeks, full scan
B+ incremental (random)       5635         0     0.000              5636
B+ incremental (asc)          7873      7827     0.994                47
B+ bulk-loaded                4385      4385     1.000                 1
a full scan of the leaf level costs 5636 seeks on the randomly built tree, 47 on the ascending one and 1 on the bulk-loaded one
```

**Rust:**

```rust
use std::collections::BTreeMap;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the numbers the Go program draws (seed 1, 2).
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

// ---- the page model: STATED ASSUMPTIONS, not any real engine ----
// Real pages have slot arrays, variable-length keys and checksums. Here a page is
// 4,096 bytes, its header 16, a key 8, a value 8, a page id 4. One extra id is
// always present (a node with n keys has n+1 children; a leaf has a next link),
// so the room for entries is PAGE_SIZE - HEADER - ID_B. Everything below is
// integer arithmetic on those numbers.
const PAGE_SIZE: usize = 4096;
const HEADER: usize = 16;
const KEY_B: usize = 8;
const VAL_B: usize = 8;
const ID_B: usize = 4;
const USABLE: usize = PAGE_SIZE - HEADER - ID_B;

const B_MAX: usize = USABLE / (KEY_B + VAL_B + ID_B); // B-tree node: key + value + child id per key
const INNER_MAX: usize = USABLE / (KEY_B + ID_B); // B+ inner page: separator key + child id (no values)
const LEAF_MAX: usize = USABLE / (KEY_B + VAL_B); // B+ leaf: key + value (no child ids)
const B_T: usize = B_MAX.div_ceil(2); // CLRS minimum degree: B_MAX = 2t-1
const BULK_LEAF: usize = LEAF_MAX * 9 / 10; // a bulk load fills leaves to 90%
const BULK_KIDS: usize = (INNER_MAX + 1) * 9 / 10; // ... and inner pages to 90% of their children
const MIN_LEAF: usize = LEAF_MAX / 2; // smallest leaf a split leaves behind
const MIN_INNER: usize = INNER_MAX / 2; // smallest inner page (keys) a split leaves behind

// Entries are keys only: the 8 value bytes are paid for in the capacity
// arithmetic above, not stored, so a "page" here is a vector of keys.

// cost counts the pages a lookup or a scan reads. A SEEK is a read of a page
// whose id is not the previous page's id + 1: the model says ids are disk
// addresses, so id+1 is the next sector and anything else moves the head.
#[derive(Default)]
struct Cost {
    pages: usize,
    seeks: usize,
    prev: Option<u32>,
}

impl Cost {
    fn read(&mut self, id: u32) {
        self.pages += 1;
        if self.prev.is_none_or(|p| id != p + 1) {
            self.seeks += 1;
        }
        self.prev = Some(id);
    }
}

// What a structure reports about itself in the build table.
struct Shape {
    levels: usize,
    pages: usize,
    leaves: usize,
    fill: f64,
}

// Both trees and the measuring code meet in this trait, so one loop can drive them all.
trait Index {
    fn find(&self, key: u64) -> (bool, usize);
    fn scan(&self, start: u64, k: usize, out: &mut Vec<u64>) -> Cost;
    fn shape(&self) -> Shape;
    fn check(&self, want: usize) -> Result<(), String>;
}

// The checkers walk with this state: the depth of the first leaf, the entries
// counted so far and the leaves in tree order.
#[derive(Default)]
struct Walk {
    leaf_depth: Option<usize>,
    total: usize,
    leaves: Vec<u32>,
}

impl Walk {
    fn leaf(&mut self, id: u32, depth: usize) -> Result<(), String> {
        match self.leaf_depth {
            Some(d) if d != depth => Err(format!("leaf {id} at depth {depth}, others at {d}")),
            _ => {
                self.leaf_depth = Some(depth);
                self.leaves.push(id);
                Ok(())
            }
        }
    }
}

fn ordered(id: u32, keys: &[u64], lo: u64, hi: u64) -> Result<(), String> {
    for (i, &k) in keys.iter().enumerate() {
        if k < lo || k >= hi || (i > 0 && k <= keys[i - 1]) {
            return Err(format!("page {id}: key {k} out of order"));
        }
    }
    Ok(())
}

// ---- B-tree (example 11's, proactive split, here with capacity 203) ----
// The arena index IS the page id, which is exactly the disk model.
struct BNode {
    keys: Vec<u64>,
    kids: Vec<u32>, // empty for a leaf
}

struct BTree {
    pages: Vec<BNode>,
    root: u32,
}

impl BTree {
    fn new() -> Self {
        BTree {
            pages: vec![BNode {
                keys: vec![],
                kids: vec![],
            }],
            root: 0,
        }
    }

    fn add(&mut self, n: BNode) -> u32 {
        self.pages.push(n);
        (self.pages.len() - 1) as u32
    }

    fn split_child(&mut self, x: u32, i: usize) {
        let y = self.pages[x as usize].kids[i] as usize;
        // keys[t-1] is the median of a full node (2t-1 keys): it moves up,
        // the keys after it move to a new right page.
        let mut right_keys = self.pages[y].keys.split_off(B_T - 1);
        let median = right_keys.remove(0);
        let right_kids = if self.pages[y].kids.is_empty() {
            vec![]
        } else {
            self.pages[y].kids.split_off(B_T)
        };
        let z = self.add(BNode {
            keys: right_keys,
            kids: right_kids,
        });
        let px = &mut self.pages[x as usize];
        px.keys.insert(i, median);
        px.kids.insert(i + 1, z);
    }

    fn insert(&mut self, key: u64) {
        if self.pages[self.root as usize].keys.len() == B_MAX {
            let old = self.root;
            self.root = self.add(BNode {
                keys: vec![],
                kids: vec![old],
            });
            self.split_child(self.root, 0);
        }
        let mut x = self.root as usize;
        loop {
            let Err(mut i) = self.pages[x].keys.binary_search(&key) else {
                return; // already present
            };
            if self.pages[x].kids.is_empty() {
                self.pages[x].keys.insert(i, key);
                return;
            }
            let child = self.pages[x].kids[i] as usize;
            if self.pages[child].keys.len() == B_MAX {
                self.split_child(x as u32, i);
                if key > self.pages[x].keys[i] {
                    i += 1;
                }
            }
            x = self.pages[x].kids[i] as usize;
        }
    }

    // B-tree separators are keys of the tree: children lie STRICTLY between them.
    fn walk(&self, w: &mut Walk, id: u32, lo: u64, hi: u64, depth: usize) -> Result<(), String> {
        let n = &self.pages[id as usize];
        if n.keys.len() > B_MAX || (id != self.root && n.keys.len() < B_T - 1) {
            return Err(format!("page {id} holds {} keys", n.keys.len()));
        }
        ordered(id, &n.keys, lo, hi)?;
        w.total += n.keys.len();
        if n.kids.is_empty() {
            return w.leaf(id, depth);
        }
        if n.kids.len() != n.keys.len() + 1 {
            return Err(format!(
                "page {id}: {} keys, {} children",
                n.keys.len(),
                n.kids.len()
            ));
        }
        for (i, &c) in n.kids.iter().enumerate() {
            let l = if i > 0 { n.keys[i - 1] + 1 } else { lo };
            let h = n.keys.get(i).copied().unwrap_or(hi);
            self.walk(w, c, l, h, depth + 1)?;
        }
        Ok(())
    }
}

impl Index for BTree {
    fn find(&self, key: u64) -> (bool, usize) {
        let (mut id, mut pages) = (self.root as usize, 0);
        loop {
            pages += 1;
            let n = &self.pages[id];
            match n.keys.binary_search(&key) {
                Ok(_) => return (true, pages),
                Err(_) if n.kids.is_empty() => return (false, pages),
                Err(i) => id = n.kids[i] as usize,
            }
        }
    }

    // scan returns the k keys >= start in order: an in-order traversal with an
    // explicit stack of (page, next key index). A page is counted when it is
    // pushed, and a page is pushed once per scan, so pages = distinct pages. The
    // seeks of a B-tree scan ignore re-reads of a parent between its children,
    // which flatters it.
    fn scan(&self, start: u64, k: usize, out: &mut Vec<u64>) -> Cost {
        out.clear();
        let mut c = Cost::default();
        let mut stack: Vec<(u32, usize)> = Vec::new();
        let mut id = self.root;
        loop {
            c.read(id);
            let n = &self.pages[id as usize];
            let (i, found) = match n.keys.binary_search(&start) {
                Ok(i) => (i, true),
                Err(i) => (i, false),
            };
            stack.push((id, i));
            if n.kids.is_empty() || found {
                break;
            }
            id = n.kids[i];
        }
        while out.len() < k {
            let Some(top) = stack.last_mut() else { break };
            let n = &self.pages[top.0 as usize];
            if top.1 >= n.keys.len() {
                stack.pop();
                continue;
            }
            out.push(n.keys[top.1]);
            top.1 += 1;
            if !n.kids.is_empty() {
                let mut id = n.kids[top.1];
                loop {
                    c.read(id);
                    stack.push((id, 0));
                    let child = &self.pages[id as usize];
                    if child.kids.is_empty() {
                        break;
                    }
                    id = child.kids[0];
                }
            }
        }
        c
    }

    fn shape(&self) -> Shape {
        let leaves = self.pages.iter().filter(|p| p.kids.is_empty()).count();
        let keys: usize = self.pages.iter().map(|p| p.keys.len()).sum();
        let (mut levels, mut id) = (1, self.root as usize);
        while let Some(&c) = self.pages[id].kids.first() {
            levels += 1;
            id = c as usize;
        }
        Shape {
            levels,
            pages: self.pages.len(),
            leaves,
            fill: keys as f64 / (self.pages.len() * B_MAX) as f64,
        }
    }

    // check: keys sorted and inside the separators, 101..203 keys in every
    // non-root page, all leaves at one depth, key total == want.
    fn check(&self, want: usize) -> Result<(), String> {
        let mut w = Walk::default();
        self.walk(&mut w, self.root, 0, 1 << 62, 1)?;
        if w.total != want {
            return Err(format!("{} keys, want {want}", w.total));
        }
        Ok(())
    }
}

// ---- B+ tree: entries only in leaves, separators only in inner pages ----
// Rust's enum puts `next` only on the leaves and `kids` only on inner pages;
// Go's one struct carries both fields on every page.
enum Page {
    Leaf { keys: Vec<u64>, next: Option<u32> },
    Inner { keys: Vec<u64>, kids: Vec<u32> },
}

struct BPlus {
    pages: Vec<Page>,
    root: u32,
}

// child_index: a separator is the FIRST key of the right subtree, so a key equal
// to a separator goes right: the child index is the number of separators <= key.
fn child_index(keys: &[u64], key: u64) -> usize {
    keys.partition_point(|&s| s <= key)
}

impl BPlus {
    fn new() -> Self {
        BPlus {
            pages: vec![Page::Leaf {
                keys: vec![],
                next: None,
            }],
            root: 0,
        }
    }

    fn add(&mut self, p: Page) -> u32 {
        self.pages.push(p);
        (self.pages.len() - 1) as u32
    }

    // insert_rec descends and returns Some((separator, new page)) when the page
    // split. A leaf split COPIES the separator up (it stays in the leaf); an
    // inner split PUSHES it up (it leaves the page). The recursive call needs
    // &mut self, so no borrow of a page may live across it: the page is looked
    // up again afterwards (Go re-takes its pointer for the same reason).
    fn insert_rec(&mut self, id: u32, key: u64) -> Option<(u64, u32)> {
        let descend = match &self.pages[id as usize] {
            Page::Inner { keys, kids } => {
                let i = child_index(keys, key);
                Some((i, kids[i]))
            }
            Page::Leaf { .. } => None,
        };
        let new_id;
        let (sep, right) = match descend {
            None => {
                new_id = self.pages.len() as u32;
                let Page::Leaf { keys, next } = &mut self.pages[id as usize] else {
                    return None;
                };
                let Err(i) = keys.binary_search(&key) else {
                    return None;
                };
                keys.insert(i, key);
                if keys.len() <= LEAF_MAX {
                    return None;
                }
                let right_keys = keys.split_off(keys.len() / 2);
                // the new right leaf inherits the old next; the left one now points at it
                let right_next = next.replace(new_id);
                (
                    right_keys[0],
                    Page::Leaf {
                        keys: right_keys,
                        next: right_next,
                    },
                )
            }
            Some((i, child)) => {
                let (child_sep, child_right) = self.insert_rec(child, key)?;
                new_id = self.pages.len() as u32;
                let Page::Inner { keys, kids } = &mut self.pages[id as usize] else {
                    return None;
                };
                keys.insert(i, child_sep);
                kids.insert(i + 1, child_right);
                if keys.len() <= INNER_MAX {
                    return None;
                }
                let mid = keys.len() / 2;
                let mut right_keys = keys.split_off(mid);
                let up = right_keys.remove(0);
                let right_kids = kids.split_off(mid + 1);
                (
                    up,
                    Page::Inner {
                        keys: right_keys,
                        kids: right_kids,
                    },
                )
            }
        };
        self.add(right);
        Some((sep, new_id))
    }

    fn insert(&mut self, key: u64) {
        if let Some((sep, right)) = self.insert_rec(self.root, key) {
            let old = self.root;
            self.root = self.add(Page::Inner {
                keys: vec![sep],
                kids: vec![old, right],
            });
        }
    }

    // bulk_load builds the tree bottom-up from SORTED keys: leaves are written left
    // to right (so their ids ascend and each next is id+1), then each inner level
    // is built over the one below. Entries are spread evenly over the fewest pages
    // that respect the 90% fill, so no page is a stub.
    fn bulk_load(sorted: &[u64]) -> BPlus {
        // spread splits total items into the fewest groups of at most `per` items, as evenly as possible
        fn spread(total: usize, per: usize) -> (usize, impl Fn(usize) -> usize) {
            let groups = total.div_ceil(per);
            (groups, move |g| {
                total / groups + usize::from(g < total % groups)
            })
        }
        let mut t = BPlus {
            pages: vec![],
            root: 0,
        };
        let mut level: Vec<(u32, u64)> = vec![]; // (page id, smallest key under it)
        let (groups, size) = spread(sorted.len(), BULK_LEAF);
        let mut pos = 0;
        for g in 0..groups {
            let keys = sorted[pos..pos + size(g)].to_vec();
            pos += keys.len();
            let id = t.pages.len() as u32;
            level.push((id, keys[0]));
            t.add(Page::Leaf {
                keys,
                next: (g + 1 < groups).then_some(id + 1),
            });
        }
        while level.len() > 1 {
            let (groups, size) = spread(level.len(), BULK_KIDS);
            let mut up = vec![];
            let mut pos = 0;
            for g in 0..groups {
                let kids = &level[pos..pos + size(g)];
                pos += kids.len();
                let keys = kids[1..].iter().map(|&(_, min)| min).collect();
                let ids = kids.iter().map(|&(id, _)| id).collect();
                up.push((t.pages.len() as u32, kids[0].1));
                t.add(Page::Inner { keys, kids: ids });
            }
            level = up;
        }
        t.root = level[0].0;
        t
    }

    // chain walks the leaf chain from the leftmost leaf and counts the steps whose
    // next id is exactly id+1.
    fn chain(&self) -> (usize, usize) {
        let mut id = self.root;
        while let Page::Inner { kids, .. } = &self.pages[id as usize] {
            id = kids[0];
        }
        let (mut steps, mut adjacent) = (0, 0);
        while let Page::Leaf {
            next: Some(next), ..
        } = &self.pages[id as usize]
        {
            steps += 1;
            adjacent += usize::from(*next == id + 1);
            id = *next;
        }
        (steps, adjacent)
    }

    // A B+ separator is a COPY of the first key to its right: children are >= it.
    fn walk(&self, w: &mut Walk, id: u32, lo: u64, hi: u64, depth: usize) -> Result<(), String> {
        match &self.pages[id as usize] {
            Page::Leaf { keys, .. } => {
                if keys.len() > LEAF_MAX || (id != self.root && keys.len() < MIN_LEAF) {
                    return Err(format!("page {id} holds {} keys", keys.len()));
                }
                ordered(id, keys, lo, hi)?;
                w.total += keys.len();
                w.leaf(id, depth)
            }
            Page::Inner { keys, kids } => {
                if keys.len() > INNER_MAX || (id != self.root && keys.len() < MIN_INNER) {
                    return Err(format!("page {id} holds {} keys", keys.len()));
                }
                ordered(id, keys, lo, hi)?;
                if kids.len() != keys.len() + 1 {
                    return Err(format!(
                        "page {id}: {} keys, {} children",
                        keys.len(),
                        kids.len()
                    ));
                }
                for (i, &c) in kids.iter().enumerate() {
                    let l = if i > 0 { keys[i - 1] } else { lo };
                    let h = keys.get(i).copied().unwrap_or(hi);
                    self.walk(w, c, l, h, depth + 1)?;
                }
                Ok(())
            }
        }
    }
}

impl Index for BPlus {
    fn find(&self, key: u64) -> (bool, usize) {
        let (mut id, mut pages) = (self.root as usize, 0);
        loop {
            pages += 1;
            match &self.pages[id] {
                Page::Leaf { keys, .. } => return (keys.binary_search(&key).is_ok(), pages),
                Page::Inner { keys, kids } => id = kids[child_index(keys, key)] as usize,
            }
        }
    }

    // scan: one descent to the leaf, then a walk along the leaf chain.
    fn scan(&self, start: u64, k: usize, out: &mut Vec<u64>) -> Cost {
        out.clear();
        let mut c = Cost::default();
        let mut id = self.root;
        loop {
            c.read(id);
            match &self.pages[id as usize] {
                Page::Inner { keys, kids } => id = kids[child_index(keys, start)],
                Page::Leaf { .. } => break,
            }
        }
        let mut i = match &self.pages[id as usize] {
            Page::Leaf { keys, .. } => keys.partition_point(|&x| x < start),
            Page::Inner { .. } => return c,
        };
        loop {
            let Page::Leaf { keys, next } = &self.pages[id as usize] else {
                return c;
            };
            while i < keys.len() && out.len() < k {
                out.push(keys[i]);
                i += 1;
            }
            match next {
                Some(next) if out.len() < k => {
                    id = *next;
                    i = 0;
                    c.read(id);
                }
                _ => return c,
            }
        }
    }

    fn shape(&self) -> Shape {
        let leaf_entries: Vec<usize> = self
            .pages
            .iter()
            .filter_map(|p| match p {
                Page::Leaf { keys, .. } => Some(keys.len()),
                Page::Inner { .. } => None,
            })
            .collect();
        let (mut levels, mut id) = (1, self.root as usize);
        while let Page::Inner { kids, .. } = &self.pages[id] {
            levels += 1;
            id = kids[0] as usize;
        }
        Shape {
            levels,
            pages: self.pages.len(),
            leaves: leaf_entries.len(),
            fill: leaf_entries.iter().sum::<usize>() as f64
                / (leaf_entries.len() * LEAF_MAX) as f64,
        }
    }

    // check: order inside pages and against the separators, page sizes, one leaf
    // depth, the leaf chain visits exactly the leaves in tree order and ends, and
    // the entry total == want.
    fn check(&self, want: usize) -> Result<(), String> {
        let mut w = Walk::default();
        self.walk(&mut w, self.root, 0, 1 << 62, 1)?;
        for (i, &id) in w.leaves.iter().enumerate() {
            let expect = w.leaves.get(i + 1).copied();
            if let Page::Leaf { next, .. } = &self.pages[id as usize]
                && *next != expect
            {
                return Err(format!(
                    "leaf {id}: next is {next:?}, tree order says {expect:?}"
                ));
            }
        }
        if w.total != want {
            return Err(format!("{} entries, want {want}", w.total));
        }
        Ok(())
    }
}

fn shuffle(rng: &mut Pcg, a: &mut [u64]) {
    for i in (1..a.len()).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
}

fn main() {
    println!(
        "== the page model (assumptions, not a real engine: real pages have slot arrays and variable-length keys) =="
    );
    println!(
        "page {PAGE_SIZE} B, header {HEADER} B, key {KEY_B} B, value {VAL_B} B, page id {ID_B} B; room for entries = {PAGE_SIZE} - {HEADER} - {ID_B} = {USABLE} B"
    );
    println!(
        "B-tree node       : {USABLE} / ({KEY_B}+{VAL_B}+{ID_B}) = {B_MAX} keys   (key, value and child id in every node)"
    );
    println!(
        "B+ inner page     : {USABLE} / ({KEY_B}+{ID_B})    = {INNER_MAX} keys   (separator and child id, no values)"
    );
    println!(
        "B+ leaf page      : {USABLE} / ({KEY_B}+{VAL_B})    = {LEAF_MAX} keys   (key and value, no child ids, plus a next link)"
    );
    println!(
        "bulk load fills leaves to {BULK_LEAF} of {LEAF_MAX} and inner pages to {BULK_KIDS} of {} children (90%)",
        INNER_MAX + 1
    );

    println!("\n== on paper: the most entries a tree of L levels can hold, every page full ==");
    println!(
        "{:>6} {:>16} {:>16} {:>7}",
        "levels", "B-tree", "B+ tree", "ratio"
    );
    for l in 1..=4u32 {
        let b = (B_MAX as u64 + 1).pow(l) - 1;
        let bp = LEAF_MAX as u64 * (INNER_MAX as u64 + 1).pow(l - 1);
        println!(
            "{:>6} {:>16} {:>16} {:>7.2}",
            l,
            b,
            bp,
            bp as f64 / b as f64
        );
    }

    const N: usize = 1000000;
    let mut rng = Pcg::new(1, 2);
    let sorted: Vec<u64> = (1..=N as u64).collect();
    let mut order = sorted.clone();
    shuffle(&mut rng, &mut order);

    let mut bt = BTree::new();
    let (mut incr, mut asc) = (BPlus::new(), BPlus::new());
    for (i, &k) in order.iter().enumerate() {
        bt.insert(k);
        incr.insert(k);
        asc.insert(sorted[i]);
    }
    let bulk = BPlus::bulk_load(&sorted);

    println!(
        "\n== built from the same {N} keys (PCG(1,2) shuffle; 'ascending' inserts them in sorted order) =="
    );
    println!(
        "{:<24} {:>6} {:>7} {:>7} {:>7} {:>8} {:>11} {:>9}  checker",
        "structure", "levels", "pages", "leaves", "inner", "fill", "entries/pg", "index MB"
    );
    let all: [(&str, &dyn Index); 4] = [
        ("B-tree (random)", &bt),
        ("B+ incremental (random)", &incr),
        ("B+ incremental (asc)", &asc),
        ("B+ bulk-loaded", &bulk),
    ];
    for (name, t) in all {
        let s = t.shape();
        let verdict = t.check(N).err().unwrap_or_else(|| "OK".to_string());
        println!(
            "{:<24} {:>6} {:>7} {:>7} {:>7} {:>7.1}% {:>11.1} {:>9.2}  {}",
            name,
            s.levels,
            s.pages,
            s.leaves,
            s.pages - s.leaves,
            100.0 * s.fill,
            N as f64 / s.pages as f64,
            (s.pages * PAGE_SIZE) as f64 / (1 << 20) as f64,
            verdict
        );
    }
    println!("fill: B-tree = keys / (pages x {B_MAX}); B+ = entries / (leaves x {LEAF_MAX})");

    // (a) point lookups
    println!("\n== (a) 100,000 point lookups of present keys: pages read per lookup ==");
    let probes: Vec<u64> = (0..100000)
        .map(|_| sorted[rng.int_n(N as u64) as usize])
        .collect();
    println!(
        "{:<24} {:>5} {:>7} {:>5} {:>9}",
        "structure", "min", "mean", "max", "not found"
    );
    for (name, t) in all {
        let (mut lo, mut hi, mut sum, mut missing) = (usize::MAX, 0, 0, 0);
        for &k in &probes {
            let (ok, pg) = t.find(k);
            missing += usize::from(!ok);
            (lo, hi, sum) = (lo.min(pg), hi.max(pg), sum + pg);
        }
        println!(
            "{:<24} {:>5} {:>7.3} {:>5} {:>9}",
            name,
            lo,
            sum as f64 / probes.len() as f64,
            hi,
            missing
        );
    }

    // (b) range scans
    println!(
        "\n== (b) range scans: 1,000 random starts, k keys each, every scan checked against the sorted slice =="
    );
    println!(
        "pages = distinct pages read; seeks = reads whose id is not previous id + 1 (first read counts)"
    );
    println!(
        "{:>7} {:<24} {:>10} {:>10} {:>9} {:>9}",
        "k", "structure", "pages/scan", "seeks/scan", "pages vs", "mismatch"
    );
    let mut out: Vec<u64> = Vec::new();
    // Rust only: std's BTreeMap answers the same scans (see the last section).
    // (inserted one by one: `collect()` would sort first and bulk-build, a different tree)
    let mut std_map: BTreeMap<u64, ()> = BTreeMap::new();
    for &k in &order {
        std_map.insert(k, ());
    }
    let (mut std_ok, mut std_total) = (0, 0);
    for k in [1, 100, 10000] {
        let starts: Vec<usize> = (0..1000)
            .map(|_| rng.int_n((N - k + 1) as u64) as usize)
            .collect();
        let mut base = 0.0;
        for (j, (name, t)) in all.iter().enumerate() {
            let (mut pages, mut seeks, mut bad) = (0, 0, 0);
            for &r in &starts {
                let c = t.scan(sorted[r], k, &mut out);
                pages += c.pages;
                seeks += c.seeks;
                bad += usize::from(out != sorted[r..r + k]);
            }
            let mean = pages as f64 / starts.len() as f64;
            if j == 0 {
                base = mean;
            }
            println!(
                "{:>7} {:<24} {:>10.2} {:>10.2} {:>8.2}x {:>9}",
                k,
                name,
                mean,
                seeks as f64 / starts.len() as f64,
                mean / base,
                bad
            );
        }
        for &r in &starts {
            let from_map = std_map.range(sorted[r]..).map(|(&key, _)| key).take(k);
            std_ok += usize::from(from_map.eq(sorted[r..r + k].iter().copied()));
            std_total += 1;
        }
    }
    println!("'pages vs' is pages/scan divided by the B-tree's");

    // (c) the leaf chain
    println!(
        "\n== (c) leaf chain: is next == id + 1? (walk the whole chain, leftmost leaf to the end) =="
    );
    println!(
        "{:<24} {:>9} {:>9} {:>9} {:>17}",
        "structure", "steps", "adjacent", "fraction", "seeks, full scan"
    );
    let mut full_seeks = Vec::new();
    for (name, t) in [
        ("B+ incremental (random)", &incr),
        ("B+ incremental (asc)", &asc),
        ("B+ bulk-loaded", &bulk),
    ] {
        let (steps, adj) = t.chain();
        full_seeks.push(steps - adj + 1);
        println!(
            "{:<24} {:>9} {:>9} {:>9.3} {:>17}",
            name,
            steps,
            adj,
            adj as f64 / steps as f64,
            steps - adj + 1
        );
    }
    println!(
        "a full scan of the leaf level costs {} seeks on the randomly built tree, {} on the ascending one and {} on the bulk-loaded one",
        full_seeks[0], full_seeks[1], full_seeks[2]
    );

    println!(
        "\n== Rust only: std's BTreeMap<u64, ()> over the same keys (inserted in the same shuffled order) =="
    );
    println!(
        "range(start..).take(k) agrees with the sorted slice on {std_ok} of {std_total} scans; its nodes hold at most 11 keys (B = 6), so its page counts are not comparable to the 4 KiB pages above"
    );
}
```

**Rust output:**

```
== the page model (assumptions, not a real engine: real pages have slot arrays and variable-length keys) ==
page 4096 B, header 16 B, key 8 B, value 8 B, page id 4 B; room for entries = 4096 - 16 - 4 = 4076 B
B-tree node       : 4076 / (8+8+4) = 203 keys   (key, value and child id in every node)
B+ inner page     : 4076 / (8+4)    = 339 keys   (separator and child id, no values)
B+ leaf page      : 4076 / (8+8)    = 254 keys   (key and value, no child ids, plus a next link)
bulk load fills leaves to 228 of 254 and inner pages to 306 of 340 children (90%)

== on paper: the most entries a tree of L levels can hold, every page full ==
levels           B-tree          B+ tree   ratio
     1              203              254    1.25
     2            41615            86360    2.08
     3          8489663         29362400    3.46
     4       1731891455       9983216000    5.76

== built from the same 1000000 keys (PCG(1,2) shuffle; 'ascending' inserts them in sorted order) ==
structure                levels   pages  leaves   inner     fill  entries/pg  index MB  checker
B-tree (random)               3    7285    7220      65    67.6%       137.3     28.46  OK
B+ incremental (random)       3    5664    5636      28    69.9%       176.6     22.12  OK
B+ incremental (asc)          3    7921    7874      47    50.0%       126.2     30.94  OK
B+ bulk-loaded                3    4402    4386      16    89.8%       227.2     17.20  OK
fill: B-tree = keys / (pages x 203); B+ = entries / (leaves x 254)

== (a) 100,000 point lookups of present keys: pages read per lookup ==
structure                  min    mean   max not found
B-tree (random)              1   2.993     3         0
B+ incremental (random)      3   3.000     3         0
B+ incremental (asc)         3   3.000     3         0
B+ bulk-loaded               3   3.000     3         0

== (b) range scans: 1,000 random starts, k keys each, every scan checked against the sorted slice ==
pages = distinct pages read; seeks = reads whose id is not previous id + 1 (first read counts)
      k structure                pages/scan seeks/scan  pages vs  mismatch
      1 B-tree (random)                3.00       2.99     1.00x         0
      1 B+ incremental (random)        3.00       3.00     1.00x         0
      1 B+ incremental (asc)           3.00       2.99     1.00x         0
      1 B+ bulk-loaded                 3.00       3.00     1.00x         0
    100 B-tree (random)                3.72       3.70     1.00x         0
    100 B+ incremental (random)        3.55       3.55     0.96x         0
    100 B+ incremental (asc)           3.79       3.00     1.02x         0
    100 B+ bulk-loaded                 3.44       3.00     0.93x         0
  10000 B-tree (random)               75.79      75.77     1.00x         0
  10000 B+ incremental (random)       59.17      59.17     0.78x         0
  10000 B+ incremental (asc)          81.74       3.46     1.08x         0
  10000 B+ bulk-loaded                46.84       3.00     0.62x         0
'pages vs' is pages/scan divided by the B-tree's

== (c) leaf chain: is next == id + 1? (walk the whole chain, leftmost leaf to the end) ==
structure                    steps  adjacent  fraction  seeks, full scan
B+ incremental (random)       5635         0     0.000              5636
B+ incremental (asc)          7873      7827     0.994                47
B+ bulk-loaded                4385      4385     1.000                 1
a full scan of the leaf level costs 5636 seeks on the randomly built tree, 47 on the ascending one and 1 on the bulk-loaded one

== Rust only: std's BTreeMap<u64, ()> over the same keys (inserted in the same shuffled order) ==
range(start..).take(k) agrees with the sorted slice on 3000 of 3000 scans; its nodes hold at most 11 keys (B = 6), so its page counts are not comparable to the 4 KiB pages above
```

> **Go vs Rust:** Everything above the last section is identical to the Go output (same PCG stream, same splits, same page ids); the last section is Rust-only because Go has no ordered map to put beside it. Rust's `enum Page { Leaf { keys, next }, Inner { keys, kids } }` gives `next` only to leaves and `kids` only to inner pages, where Go's one struct carries both and tests `kids == nil`; the recursive insert looks a page up again after the call that may grow the arena, as Go re-takes its pointer. `BTreeMap` (B = 6, at most 11 keys per node, read in `node.rs`, not run) answers the same scans correctly but is a different shape from these 203-key pages, so no page counts are printed for it.

**Complexity:** a B+ lookup reads exactly `levels` pages (3 for 1,000,000 keys; the B-tree's mean is 2.993 because a few keys sit in inner pages, so at this size the two trees are the same height), a scan of k keys reads the descent plus about k / leaf fill pages, and fan-out is page bytes over (key + pointer): on paper a full 4-level B+ tree holds 5.76× the entries of a full B-tree, and for these keys the index takes 17.2 MB bulk-loaded and 22.1 MB built randomly against the B-tree's 28.5 MB. Page counts per 10,000-key scan are not equal: 46.8 (bulk-loaded) and 59.2 (random) against the B-tree's 75.8, but 81.7 for the ascending-built tree whose leaves are half full. The chain buys sequential reads only when it is adjacent: 0 of 5,635 steps on the randomly built tree (a full leaf scan costs 5,636 seeks), 7,827 of 7,873 on the ascending-built one (47 seeks, at 50% fill) and all 4,385 on the bulk-loaded one (1 seek), so a 10,000-key scan costs 59.2, 3.5 and 3.0 seeks, against 75.8 for the B-tree. Whether a real disk makes that difference visible depends on its seek cost, which this page-id model only assumes.

---

## 14. Pages read: a buffer pool in front of the tree

`🔴 hard` · *The disk/page model*

This makes the case for the second half of the lesson countable: a simulated disk (page id = address) behind an exact-LRU buffer pool of M pages, where a lookup is a sequence of page ids and touching a page that is not in the pool is a miss, one disk read. It compares the implicit perfectly balanced binary tree over a sorted array (one key per page, about lg n pages per lookup) with a B-tree of t = 64, both over 1,000,000 keys, as the pool grows, under uniform random lookups and under consecutive keys. A lookup then costs about (levels − the levels the pool keeps) misses. It is a model with no readahead, no OS cache and no write path, so it counts how many reads a pool of a given size removes and says nothing about any real disk's speed.

**Steps:**

1. Build both structures over 1,000,000 keys (the implicit binary tree, whose page id is the mid index of each binary-search step, and a B-tree of t = 64 from a shuffle, with its checker) and prove the LRU pool against a slice-based reference before trusting it.
2. The key move: send the same 100,000 uniform-random lookups and the same 100,000 consecutive-key lookups through a fresh pool of M = 0, 1, 7, 100, 1,000, 10,000 and 100,000 pages for each structure, counting misses and hits.
3. Read the miss table column by column (where each structure's misses start to fall and where they level off) and the closing lines, whose numbers are computed from it.

```go
package main

import (
	"container/list"
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// The model: a simulated disk of pages (page id = address) behind a buffer pool
// that holds M pages with exact LRU eviction. A lookup is a sequence of page
// ids; touching a page that is not in the pool is a MISS (one disk read), and a
// hit is free. No readahead, no OS cache, no write path: a model, not a database.

// ---- the pool: a map plus a recency list, the design of lesson 07 example 16 ----
// Here the list is the standard container/list. Go has no ordered map, so
// recency lives in a linked list (front = newest).
type pool struct {
	cap          int
	order        *list.List
	at           map[int32]*list.Element
	hits, misses int
}

func newPool(m int) *pool {
	return &pool{cap: m, order: list.New(), at: map[int32]*list.Element{}}
}

// access touches a page and reports whether it was already in the pool.
func (p *pool) access(id int32) bool {
	if p.cap == 0 { // no pool at all: every touch is a disk read
		p.misses++
		return false
	}
	if e, ok := p.at[id]; ok {
		p.order.MoveToFront(e)
		p.hits++
		return true
	}
	p.misses++
	if p.order.Len() == p.cap {
		oldest := p.order.Back()
		delete(p.at, oldest.Value.(int32))
		p.order.Remove(oldest)
	}
	p.at[id] = p.order.PushFront(id)
	return false
}

// refLRU is the obviously-correct LRU (a slice, oldest first) used to check the pool.
type refLRU struct {
	cap int
	q   []int32
}

func (r *refLRU) access(id int32) bool {
	if i := slices.Index(r.q, id); i >= 0 {
		r.q = append(slices.Delete(r.q, i, i+1), id)
		return true
	}
	if len(r.q) == r.cap {
		r.q = r.q[1:]
	}
	r.q = append(r.q, id)
	return false
}

// ---- structure A: the implicit perfectly balanced binary tree ----
// One key per page; the page id of a step of binary search is its mid index.
func lookupA(sorted []int, key int, access func(int32)) bool {
	lo, hi := 0, len(sorted)-1
	for lo <= hi {
		mid := (lo + hi) / 2
		access(int32(mid))
		switch {
		case sorted[mid] == key:
			return true
		case sorted[mid] < key:
			lo = mid + 1
		default:
			hi = mid - 1
		}
	}
	return false
}

// ---- structure B: a B-tree of minimum degree t = 64 (example 11's, up to 127 keys per page) ----
// The arena index is the page id.
const (
	bT   = 64
	bMax = 2*bT - 1
)

type bnode struct {
	keys []int
	kids []int32 // nil for a leaf
}

type btree struct {
	pages []bnode
	root  int32
}

func (t *btree) add(n bnode) int32 {
	t.pages = append(t.pages, n)
	return int32(len(t.pages) - 1)
}

func (t *btree) splitChild(x int32, i int) {
	y := t.pages[x].kids[i]
	yk := t.pages[y].keys
	median := yk[bT-1]
	right := bnode{keys: slices.Clone(yk[bT:])}
	if t.pages[y].kids != nil {
		right.kids = slices.Clone(t.pages[y].kids[bT:])
		t.pages[y].kids = t.pages[y].kids[:bT]
	}
	t.pages[y].keys = yk[:bT-1]
	z := t.add(right)
	px := &t.pages[x]
	px.keys = slices.Insert(px.keys, i, median)
	px.kids = slices.Insert(px.kids, i+1, z)
}

func (t *btree) insert(key int) {
	if len(t.pages[t.root].keys) == bMax {
		old := t.root
		t.root = t.add(bnode{kids: []int32{old}})
		t.splitChild(t.root, 0)
	}
	x := t.root
	for {
		n := &t.pages[x]
		i, found := slices.BinarySearch(n.keys, key)
		if found {
			return
		}
		if n.kids == nil {
			n.keys = slices.Insert(n.keys, i, key)
			return
		}
		if len(t.pages[n.kids[i]].keys) == bMax {
			t.splitChild(x, i)
			n = &t.pages[x]
			if key > n.keys[i] {
				i++
			}
		}
		x = n.kids[i]
	}
}

func (t *btree) lookup(key int, access func(int32)) bool {
	id := t.root
	for {
		access(id)
		n := &t.pages[id]
		i, found := slices.BinarySearch(n.keys, key)
		if found {
			return true
		}
		if n.kids == nil {
			return false
		}
		id = n.kids[i]
	}
}

// check: keys sorted and strictly inside the separators, 63..127 keys in every
// non-root page, all leaves at one depth, key total == want. Returns a message.
func (t *btree) check(want int) string {
	leafDepth, total := -1, 0
	var walk func(id int32, lo, hi, depth int) string
	walk = func(id int32, lo, hi, depth int) string {
		n := &t.pages[id]
		if len(n.keys) > bMax || (id != t.root && len(n.keys) < bT-1) {
			return fmt.Sprintf("page %d holds %d keys", id, len(n.keys))
		}
		for i, k := range n.keys {
			if k < lo || k >= hi || (i > 0 && k <= n.keys[i-1]) {
				return fmt.Sprintf("page %d: key %d out of order", id, k)
			}
		}
		total += len(n.keys)
		if n.kids == nil {
			if leafDepth >= 0 && leafDepth != depth {
				return fmt.Sprintf("leaf %d at depth %d, others at %d", id, depth, leafDepth)
			}
			leafDepth = depth
			return ""
		}
		if len(n.kids) != len(n.keys)+1 {
			return fmt.Sprintf("page %d: %d keys, %d children", id, len(n.keys), len(n.kids))
		}
		for i, c := range n.kids {
			l, h := lo, hi
			if i > 0 {
				l = n.keys[i-1] + 1
			}
			if i < len(n.keys) {
				h = n.keys[i]
			}
			if e := walk(c, l, h, depth+1); e != "" {
				return e
			}
		}
		return ""
	}
	if e := walk(t.root, 0, 1<<62, 1); e != "" {
		return e
	}
	if total != want {
		return fmt.Sprintf("%d keys, want %d", total, want)
	}
	return ""
}

// levelSizes counts the pages on each level, root first.
func (t *btree) levelSizes() []int {
	var sizes []int
	level := []int32{t.root}
	for len(level) > 0 {
		sizes = append(sizes, len(level))
		var next []int32
		for _, id := range level {
			next = append(next, t.pages[id].kids...)
		}
		level = next
	}
	return sizes
}

func shuffle(rng *rand.Rand, a []int) {
	for i := len(a) - 1; i >= 1; i-- {
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
}

// run sends every key through the lookup with a fresh pool of m pages (cold start).
func run(m int, lookup func(key int, access func(int32)) bool, keys []int) (misses, hits, notFound int) {
	p := newPool(m)
	for _, k := range keys {
		if !lookup(k, func(id int32) { p.access(id) }) {
			notFound++
		}
	}
	return p.misses, p.hits, notFound
}

func main() {
	fmt.Println("== the pool, checked before it is trusted ==")
	{
		p, hits := newPool(3), []bool{}
		for _, id := range []int32{1, 2, 3, 1, 4, 2} {
			hits = append(hits, p.access(id))
		}
		fmt.Printf("capacity 3, accesses 1 2 3 1 4 2: hit? %v -> %d hits, %d misses (by hand: 4 evicts 2, then 2 evicts 3: 1 hit, 5 misses)\n",
			hits, p.hits, p.misses)
	}
	check := rand.New(rand.NewPCG(3, 4))
	agree, total := 0, 0
	for _, m := range []int{1, 2, 7, 20, 59, 60, 100} {
		p, ref := newPool(m), &refLRU{cap: m}
		for i := 0; i < 20000; i++ {
			id := int32(check.IntN(60))
			total++
			if p.access(id) == ref.access(id) {
				agree++
			}
		}
	}
	fmt.Printf("against a slice-based LRU: %d of %d hit/miss decisions agree (7 capacities, 20,000 accesses over 60 pages each)\n", agree, total)

	const n, lookups = 1000000, 100000
	rng := rand.New(rand.NewPCG(1, 2))
	sorted := make([]int, n)
	for i := range sorted {
		sorted[i] = i + 1
	}
	order := slices.Clone(sorted)
	shuffle(rng, order)
	bt := &btree{pages: []bnode{{}}}
	for _, k := range order {
		bt.insert(k)
	}

	sizes := bt.levelSizes()
	internal := 0
	for _, s := range sizes[:len(sizes)-1] {
		internal += s
	}
	verdict := bt.check(n)
	if verdict == "" {
		verdict = "OK"
	}
	fmt.Printf("\n== the structures, n = %d keys ==\n", n)
	fmt.Printf("A: implicit balanced binary tree, one key per page, %d levels (page id = the mid index of each binary-search step)\n", bits.Len(uint(n)))
	fmt.Printf("B: B-tree, t = %d (up to %d keys per page), PCG(1,2) shuffle insertion: %d levels, %d pages, checker %s\n",
		bT, bMax, len(sizes), len(bt.pages), verdict)
	fmt.Printf("   pages per level, root first: %v  -> %d internal pages, %d leaves\n", sizes, internal, sizes[len(sizes)-1])

	// Two workloads of the same length. Every pool size sees exactly the same keys.
	uniform := make([]int, lookups)
	for i := range uniform {
		uniform[i] = sorted[rng.IntN(n)]
	}
	start := rng.IntN(n - lookups + 1)
	consecutive := sorted[start : start+lookups]

	lookupB := bt.lookup
	lookupAs := func(key int, access func(int32)) bool { return lookupA(sorted, key, access) }
	ms := []int{0, 1, 7, 100, 1000, 10000, 100000}
	type cell struct{ misses, hits, notFound int }
	grid := make([][4]cell, len(ms)) // A random, B random, A sequential, B sequential
	for r, m := range ms {
		for c, w := range []struct {
			lookup func(int, func(int32)) bool
			keys   []int
		}{{lookupAs, uniform}, {lookupB, uniform}, {lookupAs, consecutive}, {lookupB, consecutive}} {
			mi, hi, nf := run(m, w.lookup, w.keys)
			grid[r][c] = cell{mi, hi, nf}
		}
	}
	notFound := 0
	for _, row := range grid {
		for _, c := range row {
			notFound += c.notFound
		}
	}

	fmt.Printf("\n== page misses per lookup (%d lookups per cell; a cold pool, so the first touches count) ==\n", lookups)
	fmt.Printf("workloads: random = uniform random present keys; sequential = %d consecutive keys starting at %d\n", lookups, sorted[start])
	fmt.Printf("%8s %8s %9s %9s %9s %9s %9s\n", "M pages", "A pins", "A naive", "A random", "B random", "A seq", "B seq")
	levelsA := bits.Len(uint(n))
	for r, m := range ms {
		pins := bits.Len(uint(m+1)) - 1
		row := grid[r]
		per := func(c cell) float64 { return float64(c.misses) / lookups }
		fmt.Printf("%8d %8d %9d %9.3f %9.3f %9.3f %9.3f\n", m, pins, levelsA-pins, per(row[0]), per(row[1]), per(row[2]), per(row[3]))
	}
	fmt.Println("A pins = floor(lg(M+1)) whole levels of A that M pages can hold; A naive = levels - pins")

	fmt.Println("\n== hit rate (hits / page touches) ==")
	fmt.Printf("%8s %9s %9s %9s %9s\n", "M pages", "A random", "B random", "A seq", "B seq")
	for r, m := range ms {
		fmt.Printf("%8d", m)
		for _, c := range grid[r] {
			fmt.Printf(" %8.1f%%", 100*float64(c.hits)/float64(c.hits+c.misses))
		}
		fmt.Println()
	}
	fmt.Printf("lookups that did not find their key, over all %d runs: %d\n", len(ms)*4, notFound)

	fmt.Println("\n== what the table says ==")
	per := func(r, c int) float64 { return float64(grid[r][c].misses) / lookups }
	hitRate := func(r, c int) float64 {
		return 100 * float64(grid[r][c].hits) / float64(grid[r][c].hits+grid[r][c].misses)
	}
	last := len(ms) - 1
	fmt.Printf("no pool (M = 0): A reads %.2f pages per random lookup, B reads %.2f (levels: %d and %d)\n", per(0, 0), per(0, 1), levelsA, len(sizes))
	for r, m := range ms {
		if float64(m) >= per(0, 0) {
			fmt.Printf("A's lookup path is %.2f pages long; with M = %d, the largest pool below that, A's hit rate is %.1f%% (random) and %.1f%% (sequential)\n",
				per(0, 0), ms[r-1], hitRate(r-1, 0), hitRate(r-1, 2))
			break
		}
	}
	for r, m := range ms {
		if m >= internal {
			fmt.Printf("smallest M tried that holds all %d internal pages of B: M = %d: B random %.3f misses per lookup (A random %.3f)\n",
				internal, m, per(r, 1), per(r, 0))
			break
		}
	}
	fmt.Printf("M = %d holds %.1fx all %d pages of B and never has to evict: B random %.3f misses per lookup = %d misses, so no page is read twice\n",
		ms[last], float64(ms[last])/float64(len(bt.pages)), len(bt.pages), per(last, 1), grid[last][1].misses)
	fmt.Printf("the same pool is %.1f%% of A's %d pages and leaves A random at %.3f misses per lookup\n",
		100*float64(ms[last])/n, n, per(last, 0))
	fmt.Printf("sequential lookups: A pays %.3f misses each (every key is a page of its own), B one miss per %.0f lookups (B holds %.1f keys per page)\n",
		per(last, 2), lookups/float64(grid[last][3].misses), float64(n)/float64(len(bt.pages)))
}
```

**Output:**

```
== the pool, checked before it is trusted ==
capacity 3, accesses 1 2 3 1 4 2: hit? [false false false true false false] -> 1 hits, 5 misses (by hand: 4 evicts 2, then 2 evicts 3: 1 hit, 5 misses)
against a slice-based LRU: 140000 of 140000 hit/miss decisions agree (7 capacities, 20,000 accesses over 60 pages each)

== the structures, n = 1000000 keys ==
A: implicit balanced binary tree, one key per page, 20 levels (page id = the mid index of each binary-search step)
B: B-tree, t = 64 (up to 127 keys per page), PCG(1,2) shuffle insertion: 4 levels, 11365 pages, checker OK
   pages per level, root first: [1 2 128 11234]  -> 131 internal pages, 11234 leaves

== page misses per lookup (100000 lookups per cell; a cold pool, so the first touches count) ==
workloads: random = uniform random present keys; sequential = 100000 consecutive keys starting at 160539
 M pages   A pins   A naive  A random  B random     A seq     B seq
       0        0        20    18.944     3.989    18.951     3.988
       1        1        19    18.944     3.989    18.951     3.988
       7        3        17    18.944     2.223    18.951     0.012
     100        6        14    15.013     1.640     1.000     0.012
    1000        9        11    11.323     0.911     1.000     0.012
   10000       13         7     7.517     0.183     1.000     0.012
  100000       16         4     3.784     0.114     1.000     0.012
A pins = floor(lg(M+1)) whole levels of A that M pages can hold; A naive = levels - pins

== hit rate (hits / page touches) ==
 M pages  A random  B random     A seq     B seq
       0      0.0%      0.0%      0.0%      0.0%
       1      0.0%      0.0%      0.0%      0.0%
       7      0.0%     44.3%      0.0%     99.7%
     100     20.8%     58.9%     94.7%     99.7%
    1000     40.2%     77.2%     94.7%     99.7%
   10000     60.3%     95.4%     94.7%     99.7%
  100000     80.0%     97.2%     94.7%     99.7%
lookups that did not find their key, over all 28 runs: 0

== what the table says ==
no pool (M = 0): A reads 18.94 pages per random lookup, B reads 3.99 (levels: 20 and 4)
A's lookup path is 18.94 pages long; with M = 7, the largest pool below that, A's hit rate is 0.0% (random) and 0.0% (sequential)
smallest M tried that holds all 131 internal pages of B: M = 1000: B random 0.911 misses per lookup (A random 11.323)
M = 100000 holds 8.8x all 11365 pages of B and never has to evict: B random 0.114 misses per lookup = 11357 misses, so no page is read twice
the same pool is 10.0% of A's 1000000 pages and leaves A random at 3.784 misses per lookup
sequential lookups: A pays 1.000 misses each (every key is a page of its own), B one miss per 86 lookups (B holds 88.0 keys per page)
```

**Rust:**

```rust
use std::collections::{BTreeMap, HashMap};

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the numbers the Go program draws.
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

// The model: a simulated disk of pages (page id = address) behind a buffer pool
// that holds M pages with exact LRU eviction. A lookup is a sequence of page
// ids; touching a page that is not in the pool is a MISS (one disk read), and a
// hit is free. No readahead, no OS cache, no write path: a model, not a database.

// ---- the pool: HashMap + BTreeMap ----
// Rust has an ordered map, so recency is a BTreeMap from a logical clock to the
// page: the least recently used page is its first entry (`pop_first`). Go has
// no ordered map and keeps a linked list instead (lesson 07 example 16).
struct Pool {
    cap: usize,
    tick: u64,
    at: HashMap<u32, u64>,      // page -> tick of its last touch
    by_age: BTreeMap<u64, u32>, // tick -> page, oldest first
    hits: usize,
    misses: usize,
}

impl Pool {
    fn new(cap: usize) -> Self {
        Pool {
            cap,
            tick: 0,
            at: HashMap::new(),
            by_age: BTreeMap::new(),
            hits: 0,
            misses: 0,
        }
    }

    // access touches a page and reports whether it was already in the pool.
    fn access(&mut self, id: u32) -> bool {
        if self.cap == 0 {
            // no pool at all: every touch is a disk read
            self.misses += 1;
            return false;
        }
        self.tick += 1;
        if let Some(old) = self.at.insert(id, self.tick) {
            self.by_age.remove(&old);
            self.by_age.insert(self.tick, id);
            self.hits += 1;
            return true;
        }
        self.misses += 1;
        self.by_age.insert(self.tick, id);
        if self.at.len() > self.cap
            && let Some((_, oldest)) = self.by_age.pop_first()
        {
            self.at.remove(&oldest);
        }
        false
    }
}

// RefLru is the obviously-correct LRU (a vector, oldest first) used to check the pool.
struct RefLru {
    cap: usize,
    q: Vec<u32>,
}

impl RefLru {
    fn access(&mut self, id: u32) -> bool {
        if let Some(i) = self.q.iter().position(|&x| x == id) {
            self.q.remove(i);
            self.q.push(id);
            return true;
        }
        if self.q.len() == self.cap {
            self.q.remove(0);
        }
        self.q.push(id);
        false
    }
}

// ---- structure A: the implicit perfectly balanced binary tree ----
// One key per page; the page id of a step of binary search is its mid index.
fn lookup_a(sorted: &[u64], key: u64, access: &mut dyn FnMut(u32)) -> bool {
    let (mut lo, mut hi) = (0, sorted.len() as isize - 1);
    while lo <= hi {
        let mid = (lo + hi) / 2;
        access(mid as u32);
        match sorted[mid as usize].cmp(&key) {
            std::cmp::Ordering::Equal => return true,
            std::cmp::Ordering::Less => lo = mid + 1,
            std::cmp::Ordering::Greater => hi = mid - 1,
        }
    }
    false
}

// ---- structure B: a B-tree of minimum degree t = 64 (example 11's, up to 127 keys per page) ----
// The arena index is the page id.
const B_T: usize = 64;
const B_MAX: usize = 2 * B_T - 1;

struct BNode {
    keys: Vec<u64>,
    kids: Vec<u32>, // empty for a leaf
}

struct BTree {
    pages: Vec<BNode>,
    root: u32,
}

impl BTree {
    fn new() -> Self {
        BTree {
            pages: vec![BNode {
                keys: vec![],
                kids: vec![],
            }],
            root: 0,
        }
    }

    fn add(&mut self, n: BNode) -> u32 {
        self.pages.push(n);
        (self.pages.len() - 1) as u32
    }

    fn split_child(&mut self, x: u32, i: usize) {
        let y = self.pages[x as usize].kids[i] as usize;
        let mut right_keys = self.pages[y].keys.split_off(B_T - 1);
        let median = right_keys.remove(0);
        let right_kids = if self.pages[y].kids.is_empty() {
            vec![]
        } else {
            self.pages[y].kids.split_off(B_T)
        };
        let z = self.add(BNode {
            keys: right_keys,
            kids: right_kids,
        });
        let px = &mut self.pages[x as usize];
        px.keys.insert(i, median);
        px.kids.insert(i + 1, z);
    }

    fn insert(&mut self, key: u64) {
        if self.pages[self.root as usize].keys.len() == B_MAX {
            let old = self.root;
            self.root = self.add(BNode {
                keys: vec![],
                kids: vec![old],
            });
            self.split_child(self.root, 0);
        }
        let mut x = self.root as usize;
        loop {
            let Err(mut i) = self.pages[x].keys.binary_search(&key) else {
                return; // already present
            };
            if self.pages[x].kids.is_empty() {
                self.pages[x].keys.insert(i, key);
                return;
            }
            let child = self.pages[x].kids[i] as usize;
            if self.pages[child].keys.len() == B_MAX {
                self.split_child(x as u32, i);
                if key > self.pages[x].keys[i] {
                    i += 1;
                }
            }
            x = self.pages[x].kids[i] as usize;
        }
    }

    fn lookup(&self, key: u64, access: &mut dyn FnMut(u32)) -> bool {
        let mut id = self.root;
        loop {
            access(id);
            let n = &self.pages[id as usize];
            match n.keys.binary_search(&key) {
                Ok(_) => return true,
                Err(_) if n.kids.is_empty() => return false,
                Err(i) => id = n.kids[i],
            }
        }
    }

    // check: keys sorted and strictly inside the separators, 63..127 keys in every
    // non-root page, all leaves at one depth, key total == want. Returns a message.
    fn walk(
        &self,
        id: u32,
        (lo, hi): (u64, u64),
        depth: usize,
        leaf_depth: &mut Option<usize>,
        total: &mut usize,
    ) -> Result<(), String> {
        let n = &self.pages[id as usize];
        if n.keys.len() > B_MAX || (id != self.root && n.keys.len() < B_T - 1) {
            return Err(format!("page {id} holds {} keys", n.keys.len()));
        }
        for (i, &k) in n.keys.iter().enumerate() {
            if k < lo || k >= hi || (i > 0 && k <= n.keys[i - 1]) {
                return Err(format!("page {id}: key {k} out of order"));
            }
        }
        *total += n.keys.len();
        if n.kids.is_empty() {
            return match *leaf_depth {
                Some(d) if d != depth => Err(format!("leaf {id} at depth {depth}, others at {d}")),
                _ => {
                    *leaf_depth = Some(depth);
                    Ok(())
                }
            };
        }
        if n.kids.len() != n.keys.len() + 1 {
            return Err(format!(
                "page {id}: {} keys, {} children",
                n.keys.len(),
                n.kids.len()
            ));
        }
        for (i, &c) in n.kids.iter().enumerate() {
            let l = if i > 0 { n.keys[i - 1] + 1 } else { lo };
            let h = n.keys.get(i).copied().unwrap_or(hi);
            self.walk(c, (l, h), depth + 1, leaf_depth, total)?;
        }
        Ok(())
    }

    fn check(&self, want: usize) -> Result<(), String> {
        let (mut leaf_depth, mut total) = (None, 0);
        self.walk(self.root, (0, 1 << 62), 1, &mut leaf_depth, &mut total)?;
        if total != want {
            return Err(format!("{total} keys, want {want}"));
        }
        Ok(())
    }

    // level_sizes counts the pages on each level, root first.
    fn level_sizes(&self) -> Vec<usize> {
        let mut sizes = vec![];
        let mut level = vec![self.root];
        while !level.is_empty() {
            sizes.push(level.len());
            level = level
                .iter()
                .flat_map(|&id| self.pages[id as usize].kids.iter().copied())
                .collect();
        }
        sizes
    }
}

fn shuffle(rng: &mut Pcg, a: &mut [u64]) {
    for i in (1..a.len()).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
}

type Lookup<'a> = &'a dyn Fn(u64, &mut dyn FnMut(u32)) -> bool;

#[derive(Clone, Copy, Default)]
struct Cell {
    misses: usize,
    hits: usize,
    not_found: usize,
}

// run sends every key through the lookup with a fresh pool of m pages (cold start).
fn run(m: usize, lookup: Lookup, keys: &[u64]) -> Cell {
    let mut p = Pool::new(m);
    let mut not_found = 0;
    for &k in keys {
        if !lookup(k, &mut |id| {
            p.access(id);
        }) {
            not_found += 1;
        }
    }
    Cell {
        misses: p.misses,
        hits: p.hits,
        not_found,
    }
}

fn join<T: ToString>(v: &[T]) -> String {
    v.iter()
        .map(|x| x.to_string())
        .collect::<Vec<_>>()
        .join(" ")
}

fn main() {
    println!("== the pool, checked before it is trusted ==");
    {
        let mut p = Pool::new(3);
        let hits: Vec<bool> = [1, 2, 3, 1, 4, 2].iter().map(|&id| p.access(id)).collect();
        println!(
            "capacity 3, accesses 1 2 3 1 4 2: hit? [{}] -> {} hits, {} misses (by hand: 4 evicts 2, then 2 evicts 3: 1 hit, 5 misses)",
            join(&hits),
            p.hits,
            p.misses
        );
    }
    let mut check = Pcg::new(3, 4);
    let (mut agree, mut total) = (0, 0);
    for m in [1, 2, 7, 20, 59, 60, 100] {
        let mut p = Pool::new(m);
        let mut reference = RefLru { cap: m, q: vec![] };
        for _ in 0..20000 {
            let id = check.int_n(60) as u32;
            total += 1;
            if p.access(id) == reference.access(id) {
                agree += 1;
            }
        }
    }
    println!(
        "against a slice-based LRU: {agree} of {total} hit/miss decisions agree (7 capacities, 20,000 accesses over 60 pages each)"
    );

    const N: usize = 1000000;
    const LOOKUPS: usize = 100000;
    let mut rng = Pcg::new(1, 2);
    let sorted: Vec<u64> = (1..=N as u64).collect();
    let mut order = sorted.clone();
    shuffle(&mut rng, &mut order);
    let mut bt = BTree::new();
    for &k in &order {
        bt.insert(k);
    }

    let sizes = bt.level_sizes();
    let internal: usize = sizes[..sizes.len() - 1].iter().sum();
    let verdict = bt.check(N).err().unwrap_or_else(|| "OK".to_string());
    let levels_a = (usize::BITS - N.leading_zeros()) as usize;
    println!("\n== the structures, n = {N} keys ==");
    println!(
        "A: implicit balanced binary tree, one key per page, {levels_a} levels (page id = the mid index of each binary-search step)"
    );
    println!(
        "B: B-tree, t = {B_T} (up to {B_MAX} keys per page), PCG(1,2) shuffle insertion: {} levels, {} pages, checker {verdict}",
        sizes.len(),
        bt.pages.len()
    );
    println!(
        "   pages per level, root first: [{}]  -> {internal} internal pages, {} leaves",
        join(&sizes),
        sizes[sizes.len() - 1]
    );

    // Two workloads of the same length. Every pool size sees exactly the same keys.
    let uniform: Vec<u64> = (0..LOOKUPS)
        .map(|_| sorted[rng.int_n(N as u64) as usize])
        .collect();
    let start = rng.int_n((N - LOOKUPS + 1) as u64) as usize;
    let consecutive = &sorted[start..start + LOOKUPS];

    let lookup_b = |key: u64, access: &mut dyn FnMut(u32)| bt.lookup(key, access);
    let lookup_as = |key: u64, access: &mut dyn FnMut(u32)| lookup_a(&sorted, key, access);
    let ms = [0, 1, 7, 100, 1000, 10000, 100000];
    let workloads: [(Lookup, &[u64]); 4] = [
        (&lookup_as, &uniform),
        (&lookup_b, &uniform),
        (&lookup_as, consecutive),
        (&lookup_b, consecutive),
    ]; // A random, B random, A sequential, B sequential
    let grid: Vec<[Cell; 4]> = ms
        .iter()
        .map(|&m| {
            let mut row = [Cell::default(); 4];
            for (cell, (lookup, keys)) in row.iter_mut().zip(workloads) {
                *cell = run(m, lookup, keys);
            }
            row
        })
        .collect();
    let not_found: usize = grid.iter().flatten().map(|c| c.not_found).sum();

    println!(
        "\n== page misses per lookup ({LOOKUPS} lookups per cell; a cold pool, so the first touches count) =="
    );
    println!(
        "workloads: random = uniform random present keys; sequential = {LOOKUPS} consecutive keys starting at {}",
        sorted[start]
    );
    println!(
        "{:>8} {:>8} {:>9} {:>9} {:>9} {:>9} {:>9}",
        "M pages", "A pins", "A naive", "A random", "B random", "A seq", "B seq"
    );
    let per = |r: usize, c: usize| grid[r][c].misses as f64 / LOOKUPS as f64;
    for (r, &m) in ms.iter().enumerate() {
        let pins = (usize::BITS - (m + 1).leading_zeros()) as usize - 1;
        println!(
            "{:>8} {:>8} {:>9} {:>9.3} {:>9.3} {:>9.3} {:>9.3}",
            m,
            pins,
            levels_a - pins,
            per(r, 0),
            per(r, 1),
            per(r, 2),
            per(r, 3)
        );
    }
    println!(
        "A pins = floor(lg(M+1)) whole levels of A that M pages can hold; A naive = levels - pins"
    );

    println!("\n== hit rate (hits / page touches) ==");
    println!(
        "{:>8} {:>9} {:>9} {:>9} {:>9}",
        "M pages", "A random", "B random", "A seq", "B seq"
    );
    let hit_rate = |r: usize, c: usize| {
        100.0 * grid[r][c].hits as f64 / (grid[r][c].hits + grid[r][c].misses) as f64
    };
    for (r, &m) in ms.iter().enumerate() {
        print!("{m:>8}");
        for c in 0..4 {
            print!(" {:>8.1}%", hit_rate(r, c));
        }
        println!();
    }
    println!(
        "lookups that did not find their key, over all {} runs: {not_found}",
        ms.len() * 4
    );

    println!("\n== what the table says ==");
    let last = ms.len() - 1;
    println!(
        "no pool (M = 0): A reads {:.2} pages per random lookup, B reads {:.2} (levels: {levels_a} and {})",
        per(0, 0),
        per(0, 1),
        sizes.len()
    );
    if let Some(r) = ms.iter().position(|&m| m as f64 >= per(0, 0)) {
        println!(
            "A's lookup path is {:.2} pages long; with M = {}, the largest pool below that, A's hit rate is {:.1}% (random) and {:.1}% (sequential)",
            per(0, 0),
            ms[r - 1],
            hit_rate(r - 1, 0),
            hit_rate(r - 1, 2)
        );
    }
    if let Some(r) = ms.iter().position(|&m| m >= internal) {
        println!(
            "smallest M tried that holds all {internal} internal pages of B: M = {}: B random {:.3} misses per lookup (A random {:.3})",
            ms[r],
            per(r, 1),
            per(r, 0)
        );
    }
    println!(
        "M = {} holds {:.1}x all {} pages of B and never has to evict: B random {:.3} misses per lookup = {} misses, so no page is read twice",
        ms[last],
        ms[last] as f64 / bt.pages.len() as f64,
        bt.pages.len(),
        per(last, 1),
        grid[last][1].misses
    );
    println!(
        "the same pool is {:.1}% of A's {N} pages and leaves A random at {:.3} misses per lookup",
        100.0 * ms[last] as f64 / N as f64,
        per(last, 0)
    );
    println!(
        "sequential lookups: A pays {:.3} misses each (every key is a page of its own), B one miss per {:.0} lookups (B holds {:.1} keys per page)",
        per(last, 2),
        LOOKUPS as f64 / grid[last][3].misses as f64,
        N as f64 / bt.pages.len() as f64
    );
}
```

**Rust output:**

```
== the pool, checked before it is trusted ==
capacity 3, accesses 1 2 3 1 4 2: hit? [false false false true false false] -> 1 hits, 5 misses (by hand: 4 evicts 2, then 2 evicts 3: 1 hit, 5 misses)
against a slice-based LRU: 140000 of 140000 hit/miss decisions agree (7 capacities, 20,000 accesses over 60 pages each)

== the structures, n = 1000000 keys ==
A: implicit balanced binary tree, one key per page, 20 levels (page id = the mid index of each binary-search step)
B: B-tree, t = 64 (up to 127 keys per page), PCG(1,2) shuffle insertion: 4 levels, 11365 pages, checker OK
   pages per level, root first: [1 2 128 11234]  -> 131 internal pages, 11234 leaves

== page misses per lookup (100000 lookups per cell; a cold pool, so the first touches count) ==
workloads: random = uniform random present keys; sequential = 100000 consecutive keys starting at 160539
 M pages   A pins   A naive  A random  B random     A seq     B seq
       0        0        20    18.944     3.989    18.951     3.988
       1        1        19    18.944     3.989    18.951     3.988
       7        3        17    18.944     2.223    18.951     0.012
     100        6        14    15.013     1.640     1.000     0.012
    1000        9        11    11.323     0.911     1.000     0.012
   10000       13         7     7.517     0.183     1.000     0.012
  100000       16         4     3.784     0.114     1.000     0.012
A pins = floor(lg(M+1)) whole levels of A that M pages can hold; A naive = levels - pins

== hit rate (hits / page touches) ==
 M pages  A random  B random     A seq     B seq
       0      0.0%      0.0%      0.0%      0.0%
       1      0.0%      0.0%      0.0%      0.0%
       7      0.0%     44.3%      0.0%     99.7%
     100     20.8%     58.9%     94.7%     99.7%
    1000     40.2%     77.2%     94.7%     99.7%
   10000     60.3%     95.4%     94.7%     99.7%
  100000     80.0%     97.2%     94.7%     99.7%
lookups that did not find their key, over all 28 runs: 0

== what the table says ==
no pool (M = 0): A reads 18.94 pages per random lookup, B reads 3.99 (levels: 20 and 4)
A's lookup path is 18.94 pages long; with M = 7, the largest pool below that, A's hit rate is 0.0% (random) and 0.0% (sequential)
smallest M tried that holds all 131 internal pages of B: M = 1000: B random 0.911 misses per lookup (A random 11.323)
M = 100000 holds 8.8x all 11365 pages of B and never has to evict: B random 0.114 misses per lookup = 11357 misses, so no page is read twice
the same pool is 10.0% of A's 1000000 pages and leaves A random at 3.784 misses per lookup
sequential lookups: A pays 1.000 misses each (every key is a page of its own), B one miss per 86 lookups (B holds 88.0 keys per page)
```

> **Go vs Rust:** The output is byte-for-byte the Go output, including every miss count and the 140,000 pool-versus-reference decisions, because the LRU policy and the PCG stream are deterministic. The recency index is where the languages differ: Rust keeps a `BTreeMap` from a logical clock to the page and evicts with `pop_first`, while Go has no ordered map and keeps a `container/list` beside its `map`. Both lookups take the access hook as a closure (`&mut dyn FnMut(u32)` here, `func(int32)` in Go) so one pool serves both structures.

**Complexity:** a lookup costs about (levels − the levels the pool keeps) page misses. With no pool the binary tree reads 18.94 pages per random lookup and the B-tree (4 levels of 1, 2, 128 and 11,234 pages, 131 of them internal) reads 3.99. A pool smaller than the lookup path caches nothing in the binary tree (0.0% hits at M = 7 against an 18.94-page path), while the B-tree already hits 44.3% of its touches at M = 7 and falls to 0.911 misses per lookup once the pool (M = 1,000) holds its 131 internal pages, against 11.323 for the binary tree. At M = 100,000 the B-tree's 0.114 misses per lookup are just its 11,357 first reads while the binary tree still pays 3.784, and consecutive keys cost 1.000 miss each in the binary tree against one miss per 86 lookups in the B-tree once the pool holds a path (M ≥ 7). These counts depend on the LRU policy and this key distribution, and say nothing about real disk latency.

---

## 15. Comparisons are not node visits

`🔴 hard` · *Count the right thing*

An algorithm analysis counts **comparisons**; a cache or a disk charges for **hops**, the number of links followed into a node. For a binary tree the two are the same number, which is why they are so easy to confuse, and a B-tree is exactly where they come apart: a whole page is one hop however many keys are compared inside it, so it can spend comparisons to save hops. So this example puts one workload through every structure of the lesson (a sorted slice, a plain BST, AVL, red-black, a treap, skip lists with p = 1/2 and p = 1/4, and B-trees with t = 6 and t = 64) and counts both numbers per successful search, with a checker run on every structure it builds. Counting is the right tool here because both numbers are exact and identical on every run, where a clock would mix them with allocator and cache effects (example 16 is the one that reads the clock). Searching is Θ(log n) comparisons for all of them; hops are Θ(log n) for the binary structures and Θ(log_t n) for the B-trees. It is the longest program of the lesson (about 900 lines of Go) because six structures are rebuilt here, insert and search only, each with its own invariant checker.

**Steps:**

1. Define the two counters in the output (a comparison is one three-way compare of the key; a hop is following a link into a node, so a B-tree page is one hop however many keys it scans), then build each of the ten structures from the same 100,000 keys inserted in random order, behind one small interface (`find`, `levels`, `nodes`, `check`).
2. The key move: search every key once and report the mean comparisons and mean hops, each also divided by lg n; then repeat the build with the keys inserted in sorted order (the plain BST only with 5,000 keys, since sorted insertion makes it quadratic).
3. Read the two tables and the closing lines, which are computed from them: which structure has the fewest comparisons, which the fewest hops, what the linear scan inside a t = 6 page costs against AVL, and what sorted input does to each structure.

```go
package main

import (
	"cmp"
	"errors"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

// One workload, ten structures, two different counts.
//
//	comparison = one three-way compare of the search key with a stored key
//	             (a skip list's failed look at the next key counts too)
//	hop        = following a link (pointer or page id) into a node: the root
//	             counts, dropping a skip-list level does not, a B-tree page is
//	             ONE hop however many keys it scans, an array element is none
//
// Comparisons are what an algorithm analysis charges. Hops are what a cache or
// a disk charges. The structures are re-assembled from examples 7-11 (insert
// and search only), each in a few lines, behind one tiny interface.

type counts struct{ cmps, hops int }

type lookup interface {
	find(key int, c *counts) bool
	levels() int // height in nodes / pages / lanes (an array is 1)
	nodes() int  // allocated nodes or pages
	check(n int) error
}

// ---- sorted slice: binary search over one array ------------------------------

type sliceIdx struct{ a []int }

func buildSlice(order []int) lookup {
	a := slices.Clone(order) // build = copy the keys, then sort once; inserting one by one is O(n^2)
	slices.Sort(a)
	return &sliceIdx{a}
}

func (s *sliceIdx) find(key int, c *counts) bool {
	lo, hi := 0, len(s.a)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		c.cmps++
		switch r := cmp.Compare(key, s.a[mid]); {
		case r == 0:
			return true
		case r > 0:
			lo = mid + 1
		default:
			hi = mid
		}
	}
	return false
}
func (s *sliceIdx) levels() int { return 1 }
func (s *sliceIdx) nodes() int  { return 1 }
func (s *sliceIdx) check(n int) error {
	if len(s.a) != n || !slices.IsSorted(s.a) {
		return errors.New("slice is not sorted or has the wrong length")
	}
	return nil
}

// ---- plain BST, AVL and treap share one node and one search ------------------

type tnode struct {
	key         int
	h           int    // AVL height (leaf = 1); only AVL reads it
	pri         uint64 // treap priority (min-heap); only the treap reads it
	left, right *tnode
}

type tree struct {
	kind string // "bst", "avl", "treap"
	root *tnode
}

func (t *tree) find(key int, c *counts) bool {
	for x := t.root; x != nil; {
		c.hops++
		c.cmps++
		switch r := cmp.Compare(key, x.key); {
		case r < 0:
			x = x.left
		case r > 0:
			x = x.right
		default:
			return true
		}
	}
	return false
}

// levels: a level-order walk, so a degenerate tree cannot overflow any stack.
func (t *tree) levels() int {
	if t.root == nil {
		return 0
	}
	level, h := []*tnode{t.root}, 0
	for len(level) > 0 {
		h++
		var next []*tnode
		for _, x := range level {
			if x.left != nil {
				next = append(next, x.left)
			}
			if x.right != nil {
				next = append(next, x.right)
			}
		}
		level = next
	}
	return h
}

func (t *tree) nodes() int {
	n, stack := 0, []*tnode{t.root}
	for len(stack) > 0 {
		x := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if x != nil {
			n++
			stack = append(stack, x.left, x.right)
		}
	}
	return n
}

func (t *tree) check(n int) error {
	// BST order and size, with an explicit stack (the plain tree can be 5,000 deep).
	var stack []*tnode
	count, prev := 0, math.MinInt
	for x := t.root; x != nil || len(stack) > 0; {
		for ; x != nil; x = x.left {
			stack = append(stack, x)
		}
		x = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if x.key <= prev {
			return fmt.Errorf("order broken at key %d", x.key)
		}
		prev, count, x = x.key, count+1, x.right
	}
	if count != n {
		return fmt.Errorf("%d nodes, want %d", count, n)
	}
	var walk func(x *tnode) error // AVL and treap are O(log n) deep, so recursion is safe
	walk = func(x *tnode) error {
		if x == nil {
			return nil
		}
		switch t.kind {
		case "avl":
			if x.h != 1+max(ht(x.left), ht(x.right)) {
				return fmt.Errorf("stale height at key %d", x.key)
			}
			if d := ht(x.left) - ht(x.right); d < -1 || d > 1 {
				return fmt.Errorf("balance factor %d at key %d", d, x.key)
			}
		case "treap":
			if (x.left != nil && x.left.pri < x.pri) || (x.right != nil && x.right.pri < x.pri) {
				return fmt.Errorf("heap order broken at key %d", x.key)
			}
		}
		if err := walk(x.left); err != nil {
			return err
		}
		return walk(x.right)
	}
	if t.kind == "bst" {
		return nil
	}
	return walk(t.root)
}

func rotR(y *tnode) *tnode { x := y.left; y.left = x.right; x.right = y; return x }
func rotL(x *tnode) *tnode { y := x.right; x.right = y.left; y.left = x; return y }

func buildBST(order []int) lookup {
	t := &tree{kind: "bst"}
	for _, k := range order { // iterative: the cursor is a pointer to a link
		p := &t.root
		for *p != nil {
			if k < (*p).key {
				p = &(*p).left
			} else {
				p = &(*p).right
			}
		}
		*p = &tnode{key: k}
	}
	return t
}

func ht(x *tnode) int {
	if x == nil {
		return 0
	}
	return x.h
}
func upd(x *tnode) { x.h = 1 + max(ht(x.left), ht(x.right)) }

func avlRotR(y *tnode) *tnode { x := rotR(y); upd(x.right); upd(x); return x }
func avlRotL(x *tnode) *tnode { y := rotL(x); upd(y.left); upd(y); return y }

func avlInsert(x *tnode, key int) *tnode {
	if x == nil {
		return &tnode{key: key, h: 1}
	}
	if key < x.key {
		x.left = avlInsert(x.left, key)
	} else {
		x.right = avlInsert(x.right, key)
	}
	upd(x)
	switch bf := ht(x.left) - ht(x.right); {
	case bf > 1:
		if ht(x.left.left) < ht(x.left.right) {
			x.left = avlRotL(x.left)
		}
		return avlRotR(x)
	case bf < -1:
		if ht(x.right.right) < ht(x.right.left) {
			x.right = avlRotR(x.right)
		}
		return avlRotL(x)
	}
	return x
}

func buildAVL(order []int) lookup {
	t := &tree{kind: "avl"}
	for _, k := range order {
		t.root = avlInsert(t.root, k)
	}
	return t
}

// Treap: priorities are drawn ONCE per key, in key order, before any insert, so
// the shape is the same for every insertion order (the point of example 9).
func treapInsert(x *tnode, key int, pri uint64) *tnode {
	if x == nil {
		return &tnode{key: key, pri: pri}
	}
	if key < x.key {
		x.left = treapInsert(x.left, key, pri)
		if x.left.pri < x.pri {
			x = rotR(x)
		}
	} else {
		x.right = treapInsert(x.right, key, pri)
		if x.right.pri < x.pri {
			x = rotL(x)
		}
	}
	return x
}

func buildTreap(order []int) lookup {
	rng := rand.New(rand.NewPCG(3, 4))
	pri := make([]uint64, len(order)+1)
	for k := 1; k <= len(order); k++ {
		pri[k] = rng.Uint64()
	}
	t := &tree{kind: "treap"}
	for _, k := range order {
		t.root = treapInsert(t.root, k, pri[k])
	}
	return t
}

// ---- red-black tree, CLRS 13.3: parent links and one shared black nil node ----

type rbnode struct {
	key                 int
	red                 bool
	left, right, parent *rbnode
}

type rbtree struct{ root, nilN *rbnode }

func (t *rbtree) rotL(x *rbnode) {
	y := x.right
	x.right = y.left
	if y.left != t.nilN {
		y.left.parent = x
	}
	y.parent = x.parent
	switch {
	case x.parent == t.nilN:
		t.root = y
	case x == x.parent.left:
		x.parent.left = y
	default:
		x.parent.right = y
	}
	y.left, x.parent = x, y
}

func (t *rbtree) rotR(y *rbnode) {
	x := y.left
	y.left = x.right
	if x.right != t.nilN {
		x.right.parent = y
	}
	x.parent = y.parent
	switch {
	case y.parent == t.nilN:
		t.root = x
	case y == y.parent.right:
		y.parent.right = x
	default:
		y.parent.left = x
	}
	x.right, y.parent = y, x
}

func (t *rbtree) insert(key int) {
	z := &rbnode{key: key, red: true, left: t.nilN, right: t.nilN}
	y, x := t.nilN, t.root
	for x != t.nilN {
		y = x
		if key < x.key {
			x = x.left
		} else {
			x = x.right
		}
	}
	z.parent = y
	switch {
	case y == t.nilN:
		t.root = z
	case key < y.key:
		y.left = z
	default:
		y.right = z
	}
	for z.parent.red { // the parent is red, so it is not the root and has a parent
		g := z.parent.parent
		if z.parent == g.left {
			if u := g.right; u.red {
				z.parent.red, u.red, g.red = false, false, true
				z = g
			} else {
				if z == z.parent.right {
					z = z.parent
					t.rotL(z)
				}
				z.parent.red, z.parent.parent.red = false, true
				t.rotR(z.parent.parent)
			}
		} else {
			if u := g.left; u.red {
				z.parent.red, u.red, g.red = false, false, true
				z = g
			} else {
				if z == z.parent.left {
					z = z.parent
					t.rotR(z)
				}
				z.parent.red, z.parent.parent.red = false, true
				t.rotL(z.parent.parent)
			}
		}
	}
	t.root.red = false
}

func buildRB(order []int) lookup {
	nilN := &rbnode{} // black
	t := &rbtree{root: nilN, nilN: nilN}
	for _, k := range order {
		t.insert(k)
	}
	return t
}

func (t *rbtree) find(key int, c *counts) bool {
	for x := t.root; x != t.nilN; {
		c.hops++
		c.cmps++
		switch r := cmp.Compare(key, x.key); {
		case r < 0:
			x = x.left
		case r > 0:
			x = x.right
		default:
			return true
		}
	}
	return false
}

func (t *rbtree) levels() int {
	if t.root == t.nilN {
		return 0
	}
	level, h := []*rbnode{t.root}, 0
	for len(level) > 0 {
		h++
		var next []*rbnode
		for _, x := range level {
			if x.left != t.nilN {
				next = append(next, x.left)
			}
			if x.right != t.nilN {
				next = append(next, x.right)
			}
		}
		level = next
	}
	return h
}

func (t *rbtree) nodes() int { // real nodes, not the shared nil sentinel
	n, stack := 0, []*rbnode{t.root}
	for len(stack) > 0 {
		x := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if x != t.nilN {
			n++
			stack = append(stack, x.left, x.right)
		}
	}
	return n
}

func (t *rbtree) check(n int) error {
	if t.root.red {
		return errors.New("root is red")
	}
	count := 0
	var walk func(x *rbnode, lo, hi int) (int, error) // returns the black height
	walk = func(x *rbnode, lo, hi int) (int, error) {
		if x == t.nilN {
			return 1, nil
		}
		count++
		if x.key <= lo || x.key >= hi {
			return 0, fmt.Errorf("order broken at key %d", x.key)
		}
		if x.red && (x.left.red || x.right.red) {
			return 0, fmt.Errorf("red node %d has a red child", x.key)
		}
		if x.left != t.nilN && x.left.parent != x || x.right != t.nilN && x.right.parent != x {
			return 0, fmt.Errorf("parent link broken under key %d", x.key)
		}
		lb, err := walk(x.left, lo, x.key)
		if err != nil {
			return 0, err
		}
		rb, err := walk(x.right, x.key, hi)
		if err != nil {
			return 0, err
		}
		if lb != rb {
			return 0, fmt.Errorf("black heights %d and %d under key %d", lb, rb, x.key)
		}
		if !x.red {
			lb++
		}
		return lb, nil
	}
	if _, err := walk(t.root, math.MinInt, math.MaxInt); err != nil {
		return err
	}
	if count != n {
		return fmt.Errorf("%d nodes, want %d", count, n)
	}
	return nil
}

// ---- skip list: a node reaches lane k with probability p^(k-1) ---------------

const maxLane = 32

type snode struct {
	key  int
	next []*snode // len(next) = the lanes this node sits on
}

type skipList struct {
	head  *snode
	level int // lanes in use
	n     int
}

func buildSkip(pBits int) func(order []int) lookup {
	return func(order []int) lookup {
		rng := rand.New(rand.NewPCG(5, 6))
		s := &skipList{head: &snode{next: make([]*snode, maxLane)}}
		for _, k := range order {
			var update [maxLane]*snode
			x := s.head
			for lane := s.level - 1; lane >= 0; lane-- {
				for x.next[lane] != nil && x.next[lane].key < k {
					x = x.next[lane]
				}
				update[lane] = x
			}
			lvl := 1
			for lvl < maxLane && rng.IntN(1<<pBits) == 0 {
				lvl++
			}
			for i := s.level; i < lvl; i++ {
				update[i] = s.head
			}
			s.level = max(s.level, lvl)
			nn := &snode{key: k, next: make([]*snode, lvl)}
			for i := range lvl {
				nn.next[i], update[i].next[i] = update[i].next[i], nn
			}
			s.n++
		}
		return s
	}
}

// find: on each lane move right while the next key is smaller, then drop a lane.
// Every evaluation of "next key < key" is a comparison; every move right is a
// hop; the last hop enters the node holding the key, and one more comparison
// confirms it.
func (s *skipList) find(key int, c *counts) bool {
	x := s.head
	for lane := s.level - 1; lane >= 0; lane-- {
		for x.next[lane] != nil {
			c.cmps++
			if x.next[lane].key >= key {
				break
			}
			x = x.next[lane]
			c.hops++
		}
	}
	nx := x.next[0]
	if nx == nil {
		return false
	}
	c.hops++
	c.cmps++
	return nx.key == key
}
func (s *skipList) levels() int { return s.level }
func (s *skipList) nodes() int  { return s.n + 1 } // + the head
func (s *skipList) check(n int) error {
	var below []int
	for lane := 0; lane < s.level; lane++ {
		var keys []int
		for x := s.head.next[lane]; x != nil; x = x.next[lane] {
			if len(x.next) <= lane {
				return fmt.Errorf("node %d is linked on lane %d but is only %d lanes tall", x.key, lane, len(x.next))
			}
			keys = append(keys, x.key)
		}
		if !slices.IsSorted(keys) {
			return fmt.Errorf("lane %d is not sorted", lane)
		}
		if lane == 0 && len(keys) != n {
			return fmt.Errorf("lane 0 has %d keys, want %d", len(keys), n)
		}
		for _, k := range keys {
			if lane > 0 {
				if _, ok := slices.BinarySearch(below, k); !ok {
					return fmt.Errorf("key %d is on lane %d but not below it", k, lane)
				}
			}
		}
		below = keys
	}
	if s.level > 0 && s.head.next[s.level-1] == nil {
		return errors.New("top lane is empty")
	}
	return nil
}

// ---- B-tree: CLRS minimum degree t, proactive split, arena index = page id ----

type bpage struct {
	keys []int
	kids []int32 // empty for a leaf
}

type btree struct {
	t      int
	binary bool // in-node search: binary (true) or a left-to-right scan
	pages  []bpage
	root   int32
}

func buildBTree(t int, binary bool) func(order []int) lookup {
	return func(order []int) lookup {
		b := &btree{t: t, binary: binary, pages: []bpage{{keys: make([]int, 0, 2*t-1)}}}
		for _, k := range order {
			b.insert(k)
		}
		return b
	}
}

// splitChild splits the full child kids[i] of x around its median.
func (b *btree) splitChild(x int32, i int) {
	t, y := b.t, b.pages[b.pages[x].kids[i]].keys
	yid := b.pages[x].kids[i]
	z := bpage{keys: append(make([]int, 0, 2*t-1), y[t:]...)}
	if len(b.pages[yid].kids) > 0 {
		z.kids = append(make([]int32, 0, 2*t), b.pages[yid].kids[t:]...)
		b.pages[yid].kids = b.pages[yid].kids[:t]
	}
	median := y[t-1]
	b.pages[yid].keys = y[:t-1]
	b.pages = append(b.pages, z) // may move the slice: take no pointers across it
	zid := int32(len(b.pages) - 1)
	px := &b.pages[x]
	px.keys = slices.Insert(px.keys, i, median)
	px.kids = slices.Insert(px.kids, i+1, zid)
}

func (b *btree) insert(key int) {
	if len(b.pages[b.root].keys) == 2*b.t-1 { // full root: grow the tree at the top
		b.pages = append(b.pages, bpage{kids: []int32{b.root}})
		b.root = int32(len(b.pages) - 1)
		b.splitChild(b.root, 0)
	}
	x := b.root
	for {
		i, _ := slices.BinarySearch(b.pages[x].keys, key)
		if len(b.pages[x].kids) == 0 {
			b.pages[x].keys = slices.Insert(b.pages[x].keys, i, key)
			return
		}
		if len(b.pages[b.pages[x].kids[i]].keys) == 2*b.t-1 {
			b.splitChild(x, i)
			if key > b.pages[x].keys[i] {
				i++
			}
		}
		x = b.pages[x].kids[i]
	}
}

func (b *btree) find(key int, c *counts) bool {
	x := b.root
	for {
		c.hops++ // one page, however many keys are scanned inside it
		pg := &b.pages[x]
		i := 0
		if b.binary {
			lo, hi := 0, len(pg.keys)
			for lo < hi {
				mid := int(uint(lo+hi) >> 1)
				c.cmps++
				switch r := cmp.Compare(key, pg.keys[mid]); {
				case r == 0:
					return true
				case r > 0:
					lo = mid + 1
				default:
					hi = mid
				}
			}
			i = lo
		} else {
			for i < len(pg.keys) {
				c.cmps++
				r := cmp.Compare(key, pg.keys[i])
				if r == 0 {
					return true
				}
				if r < 0 {
					break
				}
				i++
			}
		}
		if len(pg.kids) == 0 {
			return false
		}
		x = pg.kids[i]
	}
}

func (b *btree) levels() int {
	h := 1
	for x := b.root; len(b.pages[x].kids) > 0; x = b.pages[x].kids[0] {
		h++
	}
	return h
}
func (b *btree) nodes() int { return len(b.pages) }

func (b *btree) check(n int) error {
	total, leafDepth := 0, -1
	var walk func(x int32, depth int, lo, hi int) error
	walk = func(x int32, depth int, lo, hi int) error {
		pg := b.pages[x]
		if x != b.root && len(pg.keys) < b.t-1 || len(pg.keys) > 2*b.t-1 {
			return fmt.Errorf("page %d holds %d keys (allowed %d..%d)", x, len(pg.keys), b.t-1, 2*b.t-1)
		}
		for i, k := range pg.keys {
			if k <= lo || k >= hi || i > 0 && k <= pg.keys[i-1] {
				return fmt.Errorf("order broken in page %d at key %d", x, k)
			}
		}
		total += len(pg.keys)
		if len(pg.kids) == 0 {
			if leafDepth >= 0 && depth != leafDepth {
				return fmt.Errorf("leaves at depths %d and %d", leafDepth, depth)
			}
			leafDepth = depth
			return nil
		}
		if len(pg.kids) != len(pg.keys)+1 {
			return fmt.Errorf("page %d has %d keys and %d children", x, len(pg.keys), len(pg.kids))
		}
		for i, kid := range pg.kids {
			l, h := lo, hi
			if i > 0 {
				l = pg.keys[i-1]
			}
			if i < len(pg.keys) {
				h = pg.keys[i]
			}
			if err := walk(kid, depth+1, l, h); err != nil {
				return err
			}
		}
		return nil
	}
	if err := walk(b.root, 0, math.MinInt, math.MaxInt); err != nil {
		return err
	}
	if total != n {
		return fmt.Errorf("%d keys, want %d", total, n)
	}
	return nil
}

// ---- the experiment ---------------------------------------------------------

type row struct {
	name                 string
	n, levels, nodes     int
	cmps, hops, lgn      float64
	cmpsPerLg, hopsPerLg float64
	check                string
}

type contender struct {
	name  string
	build func(order []int) lookup
}

func contenders() []contender {
	return []contender{
		{"sorted slice, binary search", buildSlice},
		{"plain BST", buildBST},
		{"AVL", buildAVL},
		{"red-black", buildRB},
		{"treap", buildTreap},
		{"skip list p=1/2", buildSkip(1)},
		{"skip list p=1/4", buildSkip(2)},
		{"B-tree t=6, linear scan", buildBTree(6, false)},
		{"B-tree t=6, binary search", buildBTree(6, true)},
		{"B-tree t=64, binary search", buildBTree(64, true)},
	}
}

// measure builds the structure from `order`, searches every key 1..n once and
// reports the MEAN comparisons and hops per successful search.
func measure(c contender, order []int) row {
	n := len(order)
	ix := c.build(order)
	var ct counts
	for k := 1; k <= n; k++ {
		if !ix.find(k, &ct) {
			panic(fmt.Sprintf("%s: key %d not found", c.name, k))
		}
	}
	r := row{name: c.name, n: n, levels: ix.levels(), nodes: ix.nodes(), lgn: math.Log2(float64(n))}
	r.cmps, r.hops = float64(ct.cmps)/float64(n), float64(ct.hops)/float64(n)
	r.cmpsPerLg, r.hopsPerLg = r.cmps/r.lgn, r.hops/r.lgn
	r.check = "OK"
	if err := ix.check(n); err != nil {
		r.check = "FAILED: " + err.Error()
	}
	return r
}

func table(rows []row) {
	fmt.Printf("%-29s %7s %6s %11s %10s %9s %9s %8s  %s\n",
		"structure", "n", "levels", "cmps/search", "hops/search", "cmps/lg n", "hops/lg n", "nodes", "check")
	for _, r := range rows {
		fmt.Printf("%-29s %7d %6d %11.2f %10.2f %9.2f %9.2f %8d  %s\n",
			r.name, r.n, r.levels, r.cmps, r.hops, r.cmpsPerLg, r.hopsPerLg, r.nodes, r.check)
	}
}

func find(rows []row, name string) row {
	for _, r := range rows {
		if r.name == name {
			return r
		}
	}
	panic("no row " + name)
}

func main() {
	const n = 100_000
	const nSmall = 5_000 // the plain BST on sorted input is quadratic to build, so it gets 5,000

	rng := rand.New(rand.NewPCG(1, 2))
	random := make([]int, n)
	for i := range random {
		random[i] = i + 1
	}
	for i := n - 1; i >= 1; i-- { // shuffle by hand
		j := rng.IntN(i + 1)
		random[i], random[j] = random[j], random[i]
	}
	sorted := make([]int, n)
	for i := range sorted {
		sorted[i] = i + 1
	}

	fmt.Printf("keys 1..%d, every key searched once; lg n = %.2f\n", n, math.Log2(n))
	fmt.Println("comparison = one three-way compare of the search key with a stored key (a skip list's failed look at the next key counts)")
	fmt.Println("hop        = following a link into a node: the root counts, a skip-list level drop does not,")
	fmt.Println("             a B-tree page is ONE hop however many keys it scans, an array element is no hop")
	fmt.Println("B-tree t=6 has the node shape of Rust's BTreeMap (at most 11 keys, at least 5: read in its source, node.rs, not run)")

	var randRows, sortedRows []row
	for _, c := range contenders() {
		randRows = append(randRows, measure(c, random))
	}
	fmt.Printf("\n== (a) %d keys inserted in RANDOM order ==\n", n)
	table(randRows)

	for _, c := range contenders() {
		if c.name == "plain BST" {
			small := make([]int, nSmall)
			for i := range small {
				small[i] = i + 1
			}
			sortedRows = append(sortedRows, measure(c, small))
			continue
		}
		sortedRows = append(sortedRows, measure(c, sorted))
	}
	fmt.Printf("\n== (b) the same keys inserted in SORTED order (the plain BST only with n = %d) ==\n", nSmall)
	table(sortedRows)

	// ---- the closing lines are computed from the two tables ----
	fmt.Println("\n== (c) reading the tables ==")
	minBy := func(rows []row, f func(row) float64, skipArray bool) row {
		best := rows[0]
		first := true
		for _, r := range rows {
			if skipArray && r.name == "sorted slice, binary search" {
				continue
			}
			if first || f(r) < f(best) {
				best, first = r, false
			}
		}
		return best
	}
	cmpOf := func(r row) float64 { return r.cmps }
	hopOf := func(r row) float64 { return r.hops }
	fc, fh := minBy(randRows, cmpOf, false), minBy(randRows, hopOf, false)
	lc, lh := minBy(randRows, cmpOf, true), minBy(randRows, hopOf, true)
	fmt.Printf("random order, all ten: fewest comparisons: %s (%.2f); fewest hops: %s (%.2f)\n", fc.name, fc.cmps, fh.name, fh.hops)
	fmt.Printf("random order, linked structures only (an array element is not a hop): fewest comparisons: %s (%.2f); fewest hops: %s (%.2f); the same structure: %v\n",
		lc.name, lc.cmps, lh.name, lh.hops, lc.name == lh.name)
	avl, lin, bin, wide := find(randRows, "AVL"), find(randRows, "B-tree t=6, linear scan"),
		find(randRows, "B-tree t=6, binary search"), find(randRows, "B-tree t=64, binary search")
	fmt.Printf("B-tree t=6, linear scan vs AVL: %.2f vs %.2f comparisons (%+.0f%%), %.2f vs %.2f hops (%.1fx fewer)\n",
		lin.cmps, avl.cmps, 100*(lin.cmps/avl.cmps-1), lin.hops, avl.hops, avl.hops/lin.hops)
	fmt.Printf("B-tree t=6, binary search in the page: %.2f comparisons, same %.2f hops; linear scan costs %.2f more comparisons per search\n",
		bin.cmps, bin.hops, lin.cmps-bin.cmps)
	fmt.Printf("B-tree t=64: %.2f hops per search (%d levels), at %.2f comparisons\n", wide.hops, wide.levels, wide.cmps)

	s2, s4 := find(randRows, "skip list p=1/2"), find(randRows, "skip list p=1/4")
	fmt.Printf("skip lists: p=1/2 costs %.2f comparisons for %.2f hops, p=1/4 %.2f for %.2f; theory (1/p)*log_{1/p}(n) is %.2f for both (a tree does one comparison per hop, a skip list pays one failed look per lane it leaves)\n",
		s2.cmps, s2.hops, s4.cmps, s4.hops, 2*math.Log2(n))

	plain := find(sortedRows, "plain BST")
	fmt.Printf("sorted order, plain BST (n = %d): %d levels, %.2f comparisons per search = (n+1)/2 = %.1f\n",
		plain.n, plain.levels, plain.cmps, float64(plain.n+1)/2)
	lo, hi := math.Inf(1), math.Inf(-1)
	var loName, hiName string
	for _, r := range sortedRows {
		if r.name == "plain BST" {
			continue
		}
		q := r.cmps / find(randRows, r.name).cmps
		if q < lo {
			lo, loName = q, r.name
		}
		if q > hi {
			hi, hiName = q, r.name
		}
	}
	fmt.Printf("every other structure, sorted vs random insertion: comparisons per search change by %.2fx (%s) to %.2fx (%s)\n", lo, loName, hi, hiName)
	rb, avlS := find(sortedRows, "red-black"), find(sortedRows, "AVL")
	fmt.Printf("sorted input: red-black has %d levels, AVL %d (ideal %d): red-black is %d taller\n",
		rb.levels, avlS.levels, bitsLen(n), rb.levels-avlS.levels)
	fmt.Printf("treap rows identical in both tables (same priorities, so the same shape): %v\n",
		find(randRows, "treap") == find(sortedRows, "treap"))
	bad := 0
	for _, r := range append(randRows, sortedRows...) {
		if r.check != "OK" {
			bad++
		}
	}
	fmt.Printf("invariant checker failures over the %d structures built: %d\n", len(randRows)+len(sortedRows), bad)
}

func bitsLen(n int) int { // ceil(lg(n+1)): nodes on the shortest possible binary tree of n keys
	h := 0
	for ; n > 0; n >>= 1 {
		h++
	}
	return h
}
```

**Output:**

```
keys 1..100000, every key searched once; lg n = 16.61
comparison = one three-way compare of the search key with a stored key (a skip list's failed look at the next key counts)
hop        = following a link into a node: the root counts, a skip-list level drop does not,
             a B-tree page is ONE hop however many keys it scans, an array element is no hop
B-tree t=6 has the node shape of Rust's BTreeMap (at most 11 keys, at least 5: read in its source, node.rs, not run)

== (a) 100000 keys inserted in RANDOM order ==
structure                           n levels cmps/search hops/search cmps/lg n hops/lg n    nodes  check
sorted slice, binary search    100000      1       15.69       0.00      0.94      0.00        1  OK
plain BST                      100000     41       21.42      21.42      1.29      1.29   100000  OK
AVL                            100000     20       15.94      15.94      0.96      0.96   100000  OK
red-black                      100000     20       16.00      16.00      0.96      0.96   100000  OK
treap                          100000     40       20.99      20.99      1.26      1.26   100000  OK
skip list p=1/2                100000     17       32.29      16.01      1.94      0.96   100001  OK
skip list p=1/4                100000      8       32.97      25.73      1.99      1.55   100001  OK
B-tree t=6, linear scan        100000      6       22.77       5.87      1.37      0.35    13461  OK
B-tree t=6, binary search      100000      6       16.01       5.87      0.96      0.35    13461  OK
B-tree t=64, binary search     100000      3       15.73       2.99      0.95      0.18     1138  OK

== (b) the same keys inserted in SORTED order (the plain BST only with n = 5000) ==
structure                           n levels cmps/search hops/search cmps/lg n hops/lg n    nodes  check
sorted slice, binary search    100000      1       15.69       0.00      0.94      0.00        1  OK
plain BST                        5000   5000     2500.50    2500.50    203.50    203.50     5000  OK
AVL                            100000     17       15.69      15.69      0.94      0.94   100000  OK
red-black                      100000     31       16.10      16.10      0.97      0.97   100000  OK
treap                          100000     40       20.99      20.99      1.26      1.26   100000  OK
skip list p=1/2                100000     17       32.04      15.37      1.93      0.93   100001  OK
skip list p=1/4                100000      8       32.92      25.27      1.98      1.52   100001  OK
B-tree t=6, linear scan        100000      7       20.51       6.80      1.23      0.41    19997  OK
B-tree t=6, binary search      100000      7       16.08       6.80      0.97      0.41    19997  OK
B-tree t=64, binary search     100000      3       15.69       2.98      0.94      0.18     1587  OK

== (c) reading the tables ==
random order, all ten: fewest comparisons: sorted slice, binary search (15.69); fewest hops: sorted slice, binary search (0.00)
random order, linked structures only (an array element is not a hop): fewest comparisons: B-tree t=64, binary search (15.73); fewest hops: B-tree t=64, binary search (2.99); the same structure: true
B-tree t=6, linear scan vs AVL: 22.77 vs 15.94 comparisons (+43%), 5.87 vs 15.94 hops (2.7x fewer)
B-tree t=6, binary search in the page: 16.01 comparisons, same 5.87 hops; linear scan costs 6.76 more comparisons per search
B-tree t=64: 2.99 hops per search (3 levels), at 15.73 comparisons
skip lists: p=1/2 costs 32.29 comparisons for 16.01 hops, p=1/4 32.97 for 25.73; theory (1/p)*log_{1/p}(n) is 33.22 for both (a tree does one comparison per hop, a skip list pays one failed look per lane it leaves)
sorted order, plain BST (n = 5000): 5000 levels, 2500.50 comparisons per search = (n+1)/2 = 2500.5
every other structure, sorted vs random insertion: comparisons per search change by 0.90x (B-tree t=6, linear scan) to 1.01x (red-black)
sorted input: red-black has 31 levels, AVL 17 (ideal 17): red-black is 14 taller
treap rows identical in both tables (same priorities, so the same shape): true
invariant checker failures over the 20 structures built: 0
```

**Rust:**

```rust
// One workload, ten structures, two different counts.
//
//   comparison = one three-way compare of the search key with a stored key
//                (a skip list's failed look at the next key counts too)
//   hop        = following a link (pointer or page id) into a node: the root
//                counts, dropping a skip-list level does not, a B-tree page is
//                ONE hop however many keys it scans, an array element is none
//
// Comparisons are what an algorithm analysis charges. Hops are what a cache or
// a disk charges. The structures are re-assembled from examples 7-11 (insert
// and search only), each in a few lines, behind one tiny trait.
//
// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the shuffle, priorities and coin flips the Go program draws.
use std::ops::{Index, IndexMut};

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

#[derive(Default)]
struct Counts {
    cmps: usize,
    hops: usize,
}

trait Lookup {
    fn find(&self, key: u64, c: &mut Counts) -> bool;
    fn levels(&self) -> usize; // height in nodes / pages / lanes (an array is 1)
    fn nodes(&self) -> usize; // allocated nodes or pages
    fn check(&self, n: usize) -> Result<(), String>;
}

// ---- sorted slice: binary search over one array ------------------------------

struct SliceIdx {
    a: Vec<u64>,
}

fn build_slice(order: &[u64]) -> Box<dyn Lookup> {
    let mut a = order.to_vec(); // build = copy the keys, then sort once; inserting one by one is O(n^2)
    a.sort_unstable();
    Box::new(SliceIdx { a })
}

impl Lookup for SliceIdx {
    fn find(&self, key: u64, c: &mut Counts) -> bool {
        let (mut lo, mut hi) = (0, self.a.len());
        while lo < hi {
            let mid = lo + (hi - lo) / 2;
            c.cmps += 1;
            match key.cmp(&self.a[mid]) {
                std::cmp::Ordering::Equal => return true,
                std::cmp::Ordering::Greater => lo = mid + 1,
                std::cmp::Ordering::Less => hi = mid,
            }
        }
        false
    }
    fn levels(&self) -> usize {
        1
    }
    fn nodes(&self) -> usize {
        1
    }
    fn check(&self, n: usize) -> Result<(), String> {
        if self.a.len() != n || !self.a.is_sorted() {
            return Err("slice is not sorted or has the wrong length".to_string());
        }
        Ok(())
    }
}

// ---- plain BST, AVL and treap share one node and one search ------------------

struct Node {
    key: u64,
    h: i32,   // AVL height (leaf = 1); only AVL reads it
    pri: u64, // treap priority (min-heap); only the treap reads it
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

impl Node {
    fn new(key: u64, h: i32, pri: u64) -> Box<Node> {
        Box::new(Node {
            key,
            h,
            pri,
            left: None,
            right: None,
        })
    }
}

#[derive(PartialEq)]
enum Kind {
    Bst,
    Avl,
    Treap,
}

struct Tree {
    kind: Kind,
    root: Link,
}

// An iterative Drop (lesson 05 example 15, lesson 17 example 13): the plain
// BST built from sorted keys is a 5,000-deep chain, and the compiler-generated
// drop recurses once per level.
impl Drop for Tree {
    fn drop(&mut self) {
        let mut stack: Vec<Box<Node>> = self.root.take().into_iter().collect();
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
        }
    }
}

fn ht(l: &Link) -> i32 {
    l.as_ref().map_or(0, |n| n.h)
}

fn upd(n: &mut Node) {
    n.h = 1 + ht(&n.left).max(ht(&n.right));
}

// Pure pointer moves; a rotation with no child to lift returns the node.
fn rot_r(mut y: Box<Node>) -> Box<Node> {
    let Some(mut x) = y.left.take() else {
        return y;
    };
    y.left = x.right.take();
    x.right = Some(y);
    x
}

fn rot_l(mut x: Box<Node>) -> Box<Node> {
    let Some(mut y) = x.right.take() else {
        return x;
    };
    x.right = y.left.take();
    y.left = Some(x);
    y
}

fn avl_rot_r(y: Box<Node>) -> Box<Node> {
    let mut x = rot_r(y);
    if let Some(r) = x.right.as_mut() {
        upd(r); // the node that went DOWN first
    }
    upd(&mut x);
    x
}

fn avl_rot_l(x: Box<Node>) -> Box<Node> {
    let mut y = rot_l(x);
    if let Some(l) = y.left.as_mut() {
        upd(l);
    }
    upd(&mut y);
    y
}

fn avl_insert(link: Link, key: u64) -> Box<Node> {
    let Some(mut x) = link else {
        return Node::new(key, 1, 0);
    };
    if key < x.key {
        x.left = Some(avl_insert(x.left.take(), key));
    } else {
        x.right = Some(avl_insert(x.right.take(), key));
    }
    upd(&mut x);
    let bf = ht(&x.left) - ht(&x.right);
    if bf > 1 {
        let l = x.left.as_ref().unwrap();
        if ht(&l.left) < ht(&l.right) {
            x.left = Some(avl_rot_l(x.left.take().unwrap()));
        }
        return avl_rot_r(x);
    }
    if bf < -1 {
        let r = x.right.as_ref().unwrap();
        if ht(&r.right) < ht(&r.left) {
            x.right = Some(avl_rot_r(x.right.take().unwrap()));
        }
        return avl_rot_l(x);
    }
    x
}

// Treap: priorities are drawn ONCE per key, in key order, before any insert, so
// the shape is the same for every insertion order (the point of example 9).
fn treap_insert(link: Link, key: u64, pri: u64) -> Box<Node> {
    let Some(mut x) = link else {
        return Node::new(key, 0, pri);
    };
    if key < x.key {
        x.left = Some(treap_insert(x.left.take(), key, pri));
        if x.left.as_ref().unwrap().pri < x.pri {
            x = rot_r(x);
        }
    } else {
        x.right = Some(treap_insert(x.right.take(), key, pri));
        if x.right.as_ref().unwrap().pri < x.pri {
            x = rot_l(x);
        }
    }
    x
}

fn build_bst(order: &[u64]) -> Box<dyn Lookup> {
    let mut t = Tree {
        kind: Kind::Bst,
        root: None,
    };
    for &k in order {
        // iterative: the cursor is a &mut to a link
        let mut cur = &mut t.root;
        while let Some(n) = cur {
            cur = if k < n.key { &mut n.left } else { &mut n.right };
        }
        *cur = Some(Node::new(k, 0, 0));
    }
    Box::new(t)
}

fn build_avl(order: &[u64]) -> Box<dyn Lookup> {
    let mut t = Tree {
        kind: Kind::Avl,
        root: None,
    };
    for &k in order {
        t.root = Some(avl_insert(t.root.take(), k));
    }
    Box::new(t)
}

fn build_treap(order: &[u64]) -> Box<dyn Lookup> {
    let mut rng = Pcg::new(3, 4);
    let mut pri = vec![0u64; order.len() + 1];
    for p in pri.iter_mut().skip(1) {
        *p = rng.next_u64();
    }
    let mut t = Tree {
        kind: Kind::Treap,
        root: None,
    };
    for &k in order {
        t.root = Some(treap_insert(t.root.take(), k, pri[k as usize]));
    }
    Box::new(t)
}

fn walk_check(kind: &Kind, l: &Link) -> Result<(), String> {
    let Some(x) = l else { return Ok(()) };
    match kind {
        Kind::Avl => {
            if x.h != 1 + ht(&x.left).max(ht(&x.right)) {
                return Err(format!("stale height at key {}", x.key));
            }
            let d = ht(&x.left) - ht(&x.right);
            if !(-1..=1).contains(&d) {
                return Err(format!("balance factor {d} at key {}", x.key));
            }
        }
        Kind::Treap => {
            let bad = |c: &Link| c.as_ref().is_some_and(|c| c.pri < x.pri);
            if bad(&x.left) || bad(&x.right) {
                return Err(format!("heap order broken at key {}", x.key));
            }
        }
        Kind::Bst => {}
    }
    walk_check(kind, &x.left)?;
    walk_check(kind, &x.right)
}

impl Lookup for Tree {
    fn find(&self, key: u64, c: &mut Counts) -> bool {
        let mut x = self.root.as_deref();
        while let Some(n) = x {
            c.hops += 1;
            c.cmps += 1;
            x = match key.cmp(&n.key) {
                std::cmp::Ordering::Less => n.left.as_deref(),
                std::cmp::Ordering::Greater => n.right.as_deref(),
                std::cmp::Ordering::Equal => return true,
            };
        }
        false
    }

    // levels: a level-order walk, so a degenerate tree cannot overflow any stack.
    fn levels(&self) -> usize {
        let mut level: Vec<&Node> = self.root.as_deref().into_iter().collect();
        let mut h = 0;
        while !level.is_empty() {
            h += 1;
            level = level
                .iter()
                .flat_map(|n| [n.left.as_deref(), n.right.as_deref()])
                .flatten()
                .collect();
        }
        h
    }

    fn nodes(&self) -> usize {
        let mut count = 0;
        let mut stack: Vec<&Node> = self.root.as_deref().into_iter().collect();
        while let Some(n) = stack.pop() {
            count += 1;
            stack.extend(n.left.as_deref());
            stack.extend(n.right.as_deref());
        }
        count
    }

    fn check(&self, n: usize) -> Result<(), String> {
        // BST order and size, with an explicit stack (the plain tree can be 5,000 deep).
        let mut stack: Vec<&Node> = Vec::new();
        let (mut count, mut prev) = (0, 0u64); // keys are 1.., so 0 is below every key
        let mut x = self.root.as_deref();
        loop {
            while let Some(node) = x {
                stack.push(node);
                x = node.left.as_deref();
            }
            let Some(node) = stack.pop() else { break };
            if node.key <= prev {
                return Err(format!("order broken at key {}", node.key));
            }
            prev = node.key;
            count += 1;
            x = node.right.as_deref();
        }
        if count != n {
            return Err(format!("{count} nodes, want {n}"));
        }
        // AVL and treap are O(log n) deep, so recursion is safe.
        if self.kind == Kind::Bst {
            return Ok(());
        }
        walk_check(&self.kind, &self.root)
    }
}

// ---- red-black tree, CLRS 13.3: parent links and one shared black nil node ----
// Parent links are why this one lives in an arena: Box cannot express them.
// Index 0 is the shared black sentinel that stands for every nil leaf.

const NIL: u32 = 0;

struct RbNode {
    key: u64,
    red: bool,
    left: u32,
    right: u32,
    parent: u32,
}

struct RbTree {
    nodes: Vec<RbNode>,
    root: u32,
}

// `t[x]` reads like Go's `x.` field access: index the arena with a u32 id.
impl Index<u32> for RbTree {
    type Output = RbNode;
    fn index(&self, i: u32) -> &RbNode {
        &self.nodes[i as usize]
    }
}

impl IndexMut<u32> for RbTree {
    fn index_mut(&mut self, i: u32) -> &mut RbNode {
        &mut self.nodes[i as usize]
    }
}

impl RbTree {
    fn rot_l(&mut self, x: u32) {
        let y = self[x].right;
        let yl = self[y].left;
        self[x].right = yl;
        if yl != NIL {
            self[yl].parent = x;
        }
        let p = self[x].parent;
        self[y].parent = p;
        if p == NIL {
            self.root = y;
        } else if x == self[p].left {
            self[p].left = y;
        } else {
            self[p].right = y;
        }
        self[y].left = x;
        self[x].parent = y;
    }

    fn rot_r(&mut self, y: u32) {
        let x = self[y].left;
        let xr = self[x].right;
        self[y].left = xr;
        if xr != NIL {
            self[xr].parent = y;
        }
        let p = self[y].parent;
        self[x].parent = p;
        if p == NIL {
            self.root = x;
        } else if y == self[p].right {
            self[p].right = x;
        } else {
            self[p].left = x;
        }
        self[x].right = y;
        self[y].parent = x;
    }

    fn insert(&mut self, key: u64) {
        let (mut y, mut x) = (NIL, self.root);
        while x != NIL {
            y = x;
            x = if key < self[x].key {
                self[x].left
            } else {
                self[x].right
            };
        }
        let mut z = self.nodes.len() as u32;
        self.nodes.push(RbNode {
            key,
            red: true,
            left: NIL,
            right: NIL,
            parent: y,
        });
        if y == NIL {
            self.root = z;
        } else if key < self[y].key {
            self[y].left = z;
        } else {
            self[y].right = z;
        }
        // the parent is red, so it is not the root and has a parent
        while self[self[z].parent].red {
            let p = self[z].parent;
            let g = self[p].parent;
            let left_side = p == self[g].left;
            let u = if left_side {
                self[g].right
            } else {
                self[g].left
            };
            if self[u].red {
                self[p].red = false;
                self[u].red = false;
                self[g].red = true;
                z = g;
            } else {
                if left_side && z == self[p].right {
                    z = p;
                    self.rot_l(z);
                } else if !left_side && z == self[p].left {
                    z = p;
                    self.rot_r(z);
                }
                let p = self[z].parent;
                let g = self[p].parent;
                self[p].red = false;
                self[g].red = true;
                if left_side {
                    self.rot_r(g);
                } else {
                    self.rot_l(g);
                }
            }
        }
        let r = self.root;
        self[r].red = false;
    }

    // Returns the black height (counting the nil leaf), verifying order, colours and parents.
    fn walk(&self, x: u32, lo: u64, hi: u64, count: &mut usize) -> Result<usize, String> {
        if x == NIL {
            return Ok(1);
        }
        let n = &self[x];
        *count += 1;
        if n.key <= lo || n.key >= hi {
            return Err(format!("order broken at key {}", n.key));
        }
        let (l, r) = (&self[n.left], &self[n.right]);
        if n.red && (l.red || r.red) {
            return Err(format!("red node {} has a red child", n.key));
        }
        if (n.left != NIL && l.parent != x) || (n.right != NIL && r.parent != x) {
            return Err(format!("parent link broken under key {}", n.key));
        }
        let lb = self.walk(n.left, lo, n.key, count)?;
        let rb = self.walk(n.right, n.key, hi, count)?;
        if lb != rb {
            return Err(format!("black heights {lb} and {rb} under key {}", n.key));
        }
        Ok(lb + usize::from(!n.red))
    }
}

fn build_rb(order: &[u64]) -> Box<dyn Lookup> {
    let sentinel = RbNode {
        key: 0,
        red: false,
        left: NIL,
        right: NIL,
        parent: NIL,
    };
    let mut t = RbTree {
        nodes: vec![sentinel],
        root: NIL,
    };
    for &k in order {
        t.insert(k);
    }
    Box::new(t)
}

impl Lookup for RbTree {
    fn find(&self, key: u64, c: &mut Counts) -> bool {
        let mut x = self.root;
        while x != NIL {
            c.hops += 1;
            c.cmps += 1;
            let n = &self[x];
            x = match key.cmp(&n.key) {
                std::cmp::Ordering::Less => n.left,
                std::cmp::Ordering::Greater => n.right,
                std::cmp::Ordering::Equal => return true,
            };
        }
        false
    }

    fn levels(&self) -> usize {
        let mut level: Vec<u32> = if self.root == NIL {
            vec![]
        } else {
            vec![self.root]
        };
        let mut h = 0;
        while !level.is_empty() {
            h += 1;
            level = level
                .iter()
                .flat_map(|&x| [self[x].left, self[x].right])
                .filter(|&c| c != NIL)
                .collect();
        }
        h
    }

    fn nodes(&self) -> usize {
        self.nodes.len() - 1 // real nodes, not the shared nil sentinel
    }

    fn check(&self, n: usize) -> Result<(), String> {
        if self[self.root].red {
            return Err("root is red".to_string());
        }
        let mut count = 0;
        self.walk(self.root, 0, u64::MAX, &mut count)?;
        if count != n {
            return Err(format!("{count} nodes, want {n}"));
        }
        Ok(())
    }
}

// ---- skip list: a node reaches lane k with probability p^(k-1) ---------------
// Arena of nodes; index 0 is the head, so 0 doubles as "no next node".

const MAX_LANE: usize = 32;

struct SNode {
    key: u64,
    next: Vec<u32>, // next.len() = the lanes this node sits on
}

struct SkipList {
    nodes: Vec<SNode>,
    level: usize, // lanes in use
}

fn build_skip(order: &[u64], p_bits: u32) -> Box<dyn Lookup> {
    let mut rng = Pcg::new(5, 6);
    let head = SNode {
        key: 0,
        next: vec![0; MAX_LANE],
    };
    let mut s = SkipList {
        nodes: vec![head],
        level: 0,
    };
    for &k in order {
        let mut update = [0u32; MAX_LANE];
        let mut x = 0usize;
        for lane in (0..s.level).rev() {
            loop {
                let nx = s.nodes[x].next[lane] as usize;
                if nx != 0 && s.nodes[nx].key < k {
                    x = nx;
                } else {
                    break;
                }
            }
            update[lane] = x as u32;
        }
        let mut lvl = 1;
        while lvl < MAX_LANE && rng.int_n(1 << p_bits) == 0 {
            lvl += 1;
        }
        // a taller node than any so far: the new lanes start at the head (update is 0 there)
        s.level = s.level.max(lvl);
        let id = s.nodes.len() as u32;
        let mut next = vec![0; lvl];
        for (i, slot) in next.iter_mut().enumerate() {
            let pred = update[i] as usize;
            *slot = s.nodes[pred].next[i];
            s.nodes[pred].next[i] = id;
        }
        s.nodes.push(SNode { key: k, next });
    }
    Box::new(s)
}

impl Lookup for SkipList {
    // find: on each lane move right while the next key is smaller, then drop a lane.
    // Every evaluation of "next key < key" is a comparison; every move right is a
    // hop; the last hop enters the node holding the key, and one more comparison
    // confirms it.
    fn find(&self, key: u64, c: &mut Counts) -> bool {
        let mut x = 0usize;
        for lane in (0..self.level).rev() {
            loop {
                let nx = self.nodes[x].next[lane] as usize;
                if nx == 0 {
                    break;
                }
                c.cmps += 1;
                if self.nodes[nx].key >= key {
                    break;
                }
                x = nx;
                c.hops += 1;
            }
        }
        let nx = self.nodes[x].next[0] as usize;
        if nx == 0 {
            return false;
        }
        c.hops += 1;
        c.cmps += 1;
        self.nodes[nx].key == key
    }

    fn levels(&self) -> usize {
        self.level
    }

    fn nodes(&self) -> usize {
        self.nodes.len() // n keys + the head
    }

    fn check(&self, n: usize) -> Result<(), String> {
        let mut below: Vec<u64> = Vec::new();
        for lane in 0..self.level {
            let mut keys = Vec::new();
            let mut x = self.nodes[0].next[lane] as usize;
            while x != 0 {
                let node = &self.nodes[x];
                if node.next.len() <= lane {
                    return Err(format!(
                        "node {} is linked on lane {lane} but is only {} lanes tall",
                        node.key,
                        node.next.len()
                    ));
                }
                keys.push(node.key);
                x = node.next[lane] as usize;
            }
            if !keys.is_sorted() {
                return Err(format!("lane {lane} is not sorted"));
            }
            if lane == 0 && keys.len() != n {
                return Err(format!("lane 0 has {} keys, want {n}", keys.len()));
            }
            if lane > 0 {
                for k in &keys {
                    if below.binary_search(k).is_err() {
                        return Err(format!("key {k} is on lane {lane} but not below it"));
                    }
                }
            }
            below = keys;
        }
        if self.level > 0 && self.nodes[0].next[self.level - 1] == 0 {
            return Err("top lane is empty".to_string());
        }
        Ok(())
    }
}

// ---- B-tree: CLRS minimum degree t, proactive split, arena index = page id ----
// This is the model of BTreeMap (B = 6, CAPACITY = 11), NOT BTreeMap itself: that
// one cannot be instrumented, so its comparisons are not counted here.

struct Page {
    keys: Vec<u64>,
    kids: Vec<u32>, // empty for a leaf
}

struct BTree {
    t: usize,
    binary: bool, // in-node search: binary (true) or a left-to-right scan
    pages: Vec<Page>,
    root: u32,
}

impl BTree {
    // split_child splits the full child kids[i] of x around its median.
    fn split_child(&mut self, x: usize, i: usize) {
        let t = self.t;
        let yid = self.pages[x].kids[i] as usize;
        let zkeys = self.pages[yid].keys.split_off(t); // keys[t..]
        let median = self.pages[yid].keys.pop().expect("a full page has keys"); // keys[t-1]
        let zkids = if self.pages[yid].kids.is_empty() {
            Vec::new()
        } else {
            self.pages[yid].kids.split_off(t)
        };
        let zid = self.pages.len() as u32;
        self.pages.push(Page {
            keys: zkeys,
            kids: zkids,
        });
        self.pages[x].keys.insert(i, median);
        self.pages[x].kids.insert(i + 1, zid);
    }

    fn insert(&mut self, key: u64) {
        let full = 2 * self.t - 1;
        if self.pages[self.root as usize].keys.len() == full {
            // full root: grow the tree at the top
            self.pages.push(Page {
                keys: Vec::new(),
                kids: vec![self.root],
            });
            self.root = (self.pages.len() - 1) as u32;
            self.split_child(self.root as usize, 0);
        }
        let mut x = self.root as usize;
        loop {
            let mut i = self.pages[x].keys.partition_point(|&k| k < key);
            if self.pages[x].kids.is_empty() {
                self.pages[x].keys.insert(i, key);
                return;
            }
            let kid = self.pages[x].kids[i] as usize;
            if self.pages[kid].keys.len() == full {
                self.split_child(x, i);
                if key > self.pages[x].keys[i] {
                    i += 1;
                }
            }
            x = self.pages[x].kids[i] as usize;
        }
    }

    fn walk(
        &self,
        x: u32,
        depth: usize,
        (lo, hi): (u64, u64),
        st: &mut (usize, Option<usize>),
    ) -> Result<(), String> {
        let pg = &self.pages[x as usize];
        let (min, max) = (self.t - 1, 2 * self.t - 1);
        if (x != self.root && pg.keys.len() < min) || pg.keys.len() > max {
            return Err(format!(
                "page {x} holds {} keys (allowed {min}..{max})",
                pg.keys.len()
            ));
        }
        for (i, &k) in pg.keys.iter().enumerate() {
            if k <= lo || k >= hi || (i > 0 && k <= pg.keys[i - 1]) {
                return Err(format!("order broken in page {x} at key {k}"));
            }
        }
        st.0 += pg.keys.len();
        if pg.kids.is_empty() {
            match st.1 {
                Some(d) if d != depth => return Err(format!("leaves at depths {d} and {depth}")),
                _ => st.1 = Some(depth),
            }
            return Ok(());
        }
        if pg.kids.len() != pg.keys.len() + 1 {
            return Err(format!(
                "page {x} has {} keys and {} children",
                pg.keys.len(),
                pg.kids.len()
            ));
        }
        for (i, &kid) in pg.kids.iter().enumerate() {
            let l = if i > 0 { pg.keys[i - 1] } else { lo };
            let h = if i < pg.keys.len() { pg.keys[i] } else { hi };
            self.walk(kid, depth + 1, (l, h), st)?;
        }
        Ok(())
    }
}

fn build_btree(order: &[u64], t: usize, binary: bool) -> Box<dyn Lookup> {
    let mut b = BTree {
        t,
        binary,
        pages: vec![Page {
            keys: Vec::new(),
            kids: Vec::new(),
        }],
        root: 0,
    };
    for &k in order {
        b.insert(k);
    }
    Box::new(b)
}

impl Lookup for BTree {
    fn find(&self, key: u64, c: &mut Counts) -> bool {
        let mut x = self.root as usize;
        loop {
            c.hops += 1; // one page, however many keys are scanned inside it
            let pg = &self.pages[x];
            let mut i = 0;
            if self.binary {
                let (mut lo, mut hi) = (0, pg.keys.len());
                while lo < hi {
                    let mid = lo + (hi - lo) / 2;
                    c.cmps += 1;
                    match key.cmp(&pg.keys[mid]) {
                        std::cmp::Ordering::Equal => return true,
                        std::cmp::Ordering::Greater => lo = mid + 1,
                        std::cmp::Ordering::Less => hi = mid,
                    }
                }
                i = lo;
            } else {
                while i < pg.keys.len() {
                    c.cmps += 1;
                    match key.cmp(&pg.keys[i]) {
                        std::cmp::Ordering::Equal => return true,
                        std::cmp::Ordering::Less => break,
                        std::cmp::Ordering::Greater => i += 1,
                    }
                }
            }
            if pg.kids.is_empty() {
                return false;
            }
            x = pg.kids[i] as usize;
        }
    }

    fn levels(&self) -> usize {
        let mut h = 1;
        let mut x = self.root as usize;
        while let Some(&k) = self.pages[x].kids.first() {
            h += 1;
            x = k as usize;
        }
        h
    }

    fn nodes(&self) -> usize {
        self.pages.len()
    }

    fn check(&self, n: usize) -> Result<(), String> {
        let mut st = (0, None);
        self.walk(self.root, 0, (0, u64::MAX), &mut st)?;
        if st.0 != n {
            return Err(format!("{} keys, want {n}", st.0));
        }
        Ok(())
    }
}

// ---- the experiment ---------------------------------------------------------

#[derive(PartialEq)]
struct Row {
    name: &'static str,
    n: usize,
    levels: usize,
    nodes: usize,
    cmps: f64,
    hops: f64,
    cmps_per_lg: f64,
    hops_per_lg: f64,
    check: String,
}

const ARRAY: &str = "sorted slice, binary search";

type Build = fn(&[u64]) -> Box<dyn Lookup>;

fn contenders() -> Vec<(&'static str, Build)> {
    vec![
        (ARRAY, build_slice),
        ("plain BST", build_bst),
        ("AVL", build_avl),
        ("red-black", build_rb),
        ("treap", build_treap),
        ("skip list p=1/2", |o| build_skip(o, 1)),
        ("skip list p=1/4", |o| build_skip(o, 2)),
        ("B-tree t=6, linear scan", |o| build_btree(o, 6, false)),
        ("B-tree t=6, binary search", |o| build_btree(o, 6, true)),
        ("B-tree t=64, binary search", |o| build_btree(o, 64, true)),
    ]
}

// measure builds the structure from `order`, searches every key 1..n once and
// reports the MEAN comparisons and hops per successful search.
fn measure(name: &'static str, build: Build, order: &[u64]) -> Row {
    let n = order.len();
    let ix = build(order);
    let mut ct = Counts::default();
    for k in 1..=n as u64 {
        assert!(ix.find(k, &mut ct), "{name}: key {k} not found");
    }
    let lgn = (n as f64).log2();
    let (cmps, hops) = (ct.cmps as f64 / n as f64, ct.hops as f64 / n as f64);
    Row {
        name,
        n,
        levels: ix.levels(),
        nodes: ix.nodes(),
        cmps,
        hops,
        cmps_per_lg: cmps / lgn,
        hops_per_lg: hops / lgn,
        check: match ix.check(n) {
            Ok(()) => "OK".to_string(),
            Err(e) => format!("FAILED: {e}"),
        },
    }
}

fn table(rows: &[Row]) {
    let h = [
        "structure",
        "n",
        "levels",
        "cmps/search",
        "hops/search",
        "cmps/lg n",
        "hops/lg n",
        "nodes",
        "check",
    ];
    println!(
        "{:<29} {:>7} {:>6} {:>11} {:>10} {:>9} {:>9} {:>8}  {}",
        h[0], h[1], h[2], h[3], h[4], h[5], h[6], h[7], h[8]
    );
    for r in rows {
        println!(
            "{:<29} {:>7} {:>6} {:>11.2} {:>10.2} {:>9.2} {:>9.2} {:>8}  {}",
            r.name, r.n, r.levels, r.cmps, r.hops, r.cmps_per_lg, r.hops_per_lg, r.nodes, r.check
        );
    }
}

fn find<'a>(rows: &'a [Row], name: &str) -> &'a Row {
    rows.iter()
        .find(|r| r.name == name)
        .unwrap_or_else(|| panic!("no row {name}"))
}

fn min_by(rows: &[Row], f: fn(&Row) -> f64, skip_array: bool) -> &Row {
    let mut best: Option<&Row> = None;
    for r in rows {
        if skip_array && r.name == ARRAY {
            continue;
        }
        if best.is_none_or(|b| f(r) < f(b)) {
            best = Some(r);
        }
    }
    best.expect("rows is not empty")
}

fn main() {
    const N: usize = 100_000;
    const N_SMALL: usize = 5_000; // the plain BST on sorted input is quadratic to build, so it gets 5,000

    let mut rng = Pcg::new(1, 2);
    let mut random: Vec<u64> = (1..=N as u64).collect();
    for i in (1..N).rev() {
        // shuffle by hand
        let j = rng.int_n(i as u64 + 1) as usize;
        random.swap(i, j);
    }
    let sorted: Vec<u64> = (1..=N as u64).collect();

    println!(
        "keys 1..{N}, every key searched once; lg n = {:.2}",
        (N as f64).log2()
    );
    println!(
        "comparison = one three-way compare of the search key with a stored key (a skip list's failed look at the next key counts)"
    );
    println!(
        "hop        = following a link into a node: the root counts, a skip-list level drop does not,"
    );
    println!(
        "             a B-tree page is ONE hop however many keys it scans, an array element is no hop"
    );
    println!(
        "B-tree t=6 has the node shape of Rust's BTreeMap (at most 11 keys, at least 5: read in its source, node.rs, not run)"
    );

    let cs = contenders();
    let rand_rows: Vec<Row> = cs
        .iter()
        .map(|&(name, b)| measure(name, b, &random))
        .collect();
    println!("\n== (a) {N} keys inserted in RANDOM order ==");
    table(&rand_rows);

    let small: Vec<u64> = (1..=N_SMALL as u64).collect();
    let sorted_rows: Vec<Row> = cs
        .iter()
        .map(|&(name, b)| measure(name, b, if name == "plain BST" { &small } else { &sorted }))
        .collect();
    println!(
        "\n== (b) the same keys inserted in SORTED order (the plain BST only with n = {N_SMALL}) =="
    );
    table(&sorted_rows);

    // ---- the closing lines are computed from the two tables ----
    println!("\n== (c) reading the tables ==");
    let cmp_of: fn(&Row) -> f64 = |r| r.cmps;
    let hop_of: fn(&Row) -> f64 = |r| r.hops;
    let fc = min_by(&rand_rows, cmp_of, false);
    let fh = min_by(&rand_rows, hop_of, false);
    let lc = min_by(&rand_rows, cmp_of, true);
    let lh = min_by(&rand_rows, hop_of, true);
    println!(
        "random order, all ten: fewest comparisons: {} ({:.2}); fewest hops: {} ({:.2})",
        fc.name, fc.cmps, fh.name, fh.hops
    );
    println!(
        "random order, linked structures only (an array element is not a hop): fewest comparisons: {} ({:.2}); fewest hops: {} ({:.2}); the same structure: {}",
        lc.name,
        lc.cmps,
        lh.name,
        lh.hops,
        lc.name == lh.name
    );
    let avl = find(&rand_rows, "AVL");
    let lin = find(&rand_rows, "B-tree t=6, linear scan");
    let bin = find(&rand_rows, "B-tree t=6, binary search");
    let wide = find(&rand_rows, "B-tree t=64, binary search");
    println!(
        "B-tree t=6, linear scan vs AVL: {:.2} vs {:.2} comparisons ({:+.0}%), {:.2} vs {:.2} hops ({:.1}x fewer)",
        lin.cmps,
        avl.cmps,
        100.0 * (lin.cmps / avl.cmps - 1.0),
        lin.hops,
        avl.hops,
        avl.hops / lin.hops
    );
    println!(
        "B-tree t=6, binary search in the page: {:.2} comparisons, same {:.2} hops; linear scan costs {:.2} more comparisons per search",
        bin.cmps,
        bin.hops,
        lin.cmps - bin.cmps
    );
    println!(
        "B-tree t=64: {:.2} hops per search ({} levels), at {:.2} comparisons",
        wide.hops, wide.levels, wide.cmps
    );
    let s2 = find(&rand_rows, "skip list p=1/2");
    let s4 = find(&rand_rows, "skip list p=1/4");
    println!(
        "skip lists: p=1/2 costs {:.2} comparisons for {:.2} hops, p=1/4 {:.2} for {:.2}; theory (1/p)*log_{{1/p}}(n) is {:.2} for both (a tree does one comparison per hop, a skip list pays one failed look per lane it leaves)",
        s2.cmps,
        s2.hops,
        s4.cmps,
        s4.hops,
        2.0 * (N as f64).log2()
    );

    let plain = find(&sorted_rows, "plain BST");
    println!(
        "sorted order, plain BST (n = {}): {} levels, {:.2} comparisons per search = (n+1)/2 = {:.1}",
        plain.n,
        plain.levels,
        plain.cmps,
        (plain.n + 1) as f64 / 2.0
    );
    let (mut lo, mut hi) = (f64::INFINITY, f64::NEG_INFINITY);
    let (mut lo_name, mut hi_name) = ("", "");
    for r in sorted_rows.iter().filter(|r| r.name != "plain BST") {
        let q = r.cmps / find(&rand_rows, r.name).cmps;
        if q < lo {
            (lo, lo_name) = (q, r.name);
        }
        if q > hi {
            (hi, hi_name) = (q, r.name);
        }
    }
    println!(
        "every other structure, sorted vs random insertion: comparisons per search change by {lo:.2}x ({lo_name}) to {hi:.2}x ({hi_name})"
    );
    let rb = find(&sorted_rows, "red-black");
    let avl_s = find(&sorted_rows, "AVL");
    let ideal = (usize::BITS - N.leading_zeros()) as usize; // ceil(lg(n+1))
    println!(
        "sorted input: red-black has {} levels, AVL {} (ideal {}): red-black is {} taller",
        rb.levels,
        avl_s.levels,
        ideal,
        rb.levels - avl_s.levels
    );
    println!(
        "treap rows identical in both tables (same priorities, so the same shape): {}",
        find(&rand_rows, "treap") == find(&sorted_rows, "treap")
    );
    let bad = rand_rows
        .iter()
        .chain(&sorted_rows)
        .filter(|r| r.check != "OK")
        .count();
    println!(
        "invariant checker failures over the {} structures built: {bad}",
        rand_rows.len() + sorted_rows.len()
    );
}
```

**Rust output:**

```
keys 1..100000, every key searched once; lg n = 16.61
comparison = one three-way compare of the search key with a stored key (a skip list's failed look at the next key counts)
hop        = following a link into a node: the root counts, a skip-list level drop does not,
             a B-tree page is ONE hop however many keys it scans, an array element is no hop
B-tree t=6 has the node shape of Rust's BTreeMap (at most 11 keys, at least 5: read in its source, node.rs, not run)

== (a) 100000 keys inserted in RANDOM order ==
structure                           n levels cmps/search hops/search cmps/lg n hops/lg n    nodes  check
sorted slice, binary search    100000      1       15.69       0.00      0.94      0.00        1  OK
plain BST                      100000     41       21.42      21.42      1.29      1.29   100000  OK
AVL                            100000     20       15.94      15.94      0.96      0.96   100000  OK
red-black                      100000     20       16.00      16.00      0.96      0.96   100000  OK
treap                          100000     40       20.99      20.99      1.26      1.26   100000  OK
skip list p=1/2                100000     17       32.29      16.01      1.94      0.96   100001  OK
skip list p=1/4                100000      8       32.97      25.73      1.99      1.55   100001  OK
B-tree t=6, linear scan        100000      6       22.77       5.87      1.37      0.35    13461  OK
B-tree t=6, binary search      100000      6       16.01       5.87      0.96      0.35    13461  OK
B-tree t=64, binary search     100000      3       15.73       2.99      0.95      0.18     1138  OK

== (b) the same keys inserted in SORTED order (the plain BST only with n = 5000) ==
structure                           n levels cmps/search hops/search cmps/lg n hops/lg n    nodes  check
sorted slice, binary search    100000      1       15.69       0.00      0.94      0.00        1  OK
plain BST                        5000   5000     2500.50    2500.50    203.50    203.50     5000  OK
AVL                            100000     17       15.69      15.69      0.94      0.94   100000  OK
red-black                      100000     31       16.10      16.10      0.97      0.97   100000  OK
treap                          100000     40       20.99      20.99      1.26      1.26   100000  OK
skip list p=1/2                100000     17       32.04      15.37      1.93      0.93   100001  OK
skip list p=1/4                100000      8       32.92      25.27      1.98      1.52   100001  OK
B-tree t=6, linear scan        100000      7       20.51       6.80      1.23      0.41    19997  OK
B-tree t=6, binary search      100000      7       16.08       6.80      0.97      0.41    19997  OK
B-tree t=64, binary search     100000      3       15.69       2.98      0.94      0.18     1587  OK

== (c) reading the tables ==
random order, all ten: fewest comparisons: sorted slice, binary search (15.69); fewest hops: sorted slice, binary search (0.00)
random order, linked structures only (an array element is not a hop): fewest comparisons: B-tree t=64, binary search (15.73); fewest hops: B-tree t=64, binary search (2.99); the same structure: true
B-tree t=6, linear scan vs AVL: 22.77 vs 15.94 comparisons (+43%), 5.87 vs 15.94 hops (2.7x fewer)
B-tree t=6, binary search in the page: 16.01 comparisons, same 5.87 hops; linear scan costs 6.76 more comparisons per search
B-tree t=64: 2.99 hops per search (3 levels), at 15.73 comparisons
skip lists: p=1/2 costs 32.29 comparisons for 16.01 hops, p=1/4 32.97 for 25.73; theory (1/p)*log_{1/p}(n) is 33.22 for both (a tree does one comparison per hop, a skip list pays one failed look per lane it leaves)
sorted order, plain BST (n = 5000): 5000 levels, 2500.50 comparisons per search = (n+1)/2 = 2500.5
every other structure, sorted vs random insertion: comparisons per search change by 0.90x (B-tree t=6, linear scan) to 1.01x (red-black)
sorted input: red-black has 31 levels, AVL 17 (ideal 17): red-black is 14 taller
treap rows identical in both tables (same priorities, so the same shape): true
invariant checker failures over the 20 structures built: 0
```

> **Go vs Rust:** The output is byte-for-byte the Go output: same PCG stream for the shuffle, the treap priorities and the skip-list coin flips, same counts. AVL and the treap are `Option<Box<Node>>` with recursion that returns the new subtree root, the plain BST is built through a `&mut Link` cursor and freed by an iterative `Drop` so the 5,000-deep chain cannot overflow the stack, and red-black, the skip list and the B-tree are `Vec` arenas indexed by `u32` (a parent link or several predecessors have no single `Box` owner). The B-tree rows are a model of `BTreeMap` (same shape, B = 6 and CAPACITY = 11, and its `find_key_index` in `search.rs` is the same left-to-right three-way scan as the "linear scan" row: read in the source, not run), not `BTreeMap` itself, whose comparisons cannot be counted from outside.

**Complexity:** a successful search is Θ(log n) comparisons in every structure here: 15.69 to 32.97 at n = 100,000 (lg n = 16.61), with the sorted slice, AVL, red-black and the binary-search B-trees all at 0.94-0.97 lg n, a plain BST on random input at 1.29 lg n (21.42) and the skip lists at about 2 lg n (32.29 and 32.97, theory 33.22 for both p). Hops are what separates them: Θ(log n) for the binary structures (AVL 15.94, plain BST 21.42), Θ(log_t n) for B-trees (5.87 for t = 6 in 6 levels, 2.99 for t = 64 in 3 levels), none for the array. The expected trade shows only for the linear scan: a t = 6 page scanned left to right costs 22.77 comparisons against AVL's 15.94 (+43%) for 2.7x fewer hops, but binary search inside the page gives 16.01 at the same 5.87 hops, and the t = 64 B-tree has both the fewest comparisons (15.73) and the fewest hops (2.99) of any linked structure, so a B-tree does not have to buy hops with comparisons. Sorted insertion breaks only the plain BST (5,000 levels, 2,500.5 comparisons per search at n = 5,000); the others change by 0.90x-1.01x, though red-black ends 31 levels tall against AVL's 17, and the treap comes out identical. All 20 builds pass their checkers.

---

## 16. Capstone: `OrderedMap` as a B-tree, proved and measured

`🔴 hard` · *The ordered map Go does not ship*

This is the structure the lesson argues for, built complete: an `OrderedMap[K, V]` with `Get`, `Put` (upsert), `Delete`, `Min`, `Max`, `Floor`, `Ceil`, `Range` (a range-over-func iterator) and `Len`, as a B-tree of minimum degree t. Go's standard library has no ordered map, so there is no stdlib row to beat; the proof is against a sorted slice of pairs, and the measurements are against what you would actually reach for instead, a sorted slice and (for the point operations only, since it has no order) a built-in `map`. The tree is proved before it is measured: 200,000 random operations per degree t ∈ {2, 3, 6, 16} on a narrow key range, so duplicates, absent deletes and re-puts are constant, with the invariant checker and a full comparison with the oracle after every single operation, then a grow-to-5,000 / shrink-to-0 / regrow phase that forces the root to split and collapse, then two deliberately planted bugs to show that the checker would have caught them. Every operation costs Θ(log_t n) page visits, `Range` Θ(log_t n + k), and a split or merge is amortised O(1) per update. It is the longest program of the lesson (about 1,080 lines of Go, far over the size guide) because it is the whole data structure, its checker, its oracle harness and its benchmark.

**Steps:**

1. Build the tree on an arena of pages (a page id is its slice index; merged-away pages go on a free list that the next split reuses): `Put` splits a full page on the way down, `Delete` makes sure the child it is about to enter has at least t keys by borrowing from a sibling or merging with one, and both grow and shrink the tree only at the root.
2. The key move is the proof: run the same 200,000 operations from `PCG(1, 2)` through the map and a sorted-slice oracle at each t, with values `op*1000+key` so a stale value shows, and run the checker (key counts, equal leaf depth, order and separators, children = keys + 1, every page reachable or on the free list, levels within the theorem's bound) after every operation; then plant two bugs and read where the first one is caught.
3. Read the counters (splits, merges, borrows, root growths and shrinks, pages per `Get`), the grow-shrink table, the two bug reports, and finally the one timing table of the lesson, against a sorted slice and a `map` at seven sizes.

```go
package main

import (
	"cmp"
	"fmt"
	"iter"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// OrderedMap[K, V]: the ordered map Go's standard library does not ship, as a
// B-tree of minimum degree t (every page but the root holds t-1 .. 2t-1 keys;
// a page has one more child than keys; all leaves are at one depth).
//
// The pages live in an arena (a slice), a page id is its index, and a page
// merged away goes on a free list that the next split reuses. Insert is CLRS:
// split a full page on the way DOWN, so one pass suffices. Delete is CLRS too:
// make sure the child you are about to enter has at least t keys (borrow from a
// sibling or merge with one), so the key can always be removed from a page that
// can spare it. Both grow and shrink the tree only at the root.

type page[K cmp.Ordered, V any] struct {
	keys []K
	vals []V
	kids []int32 // page ids; empty for a leaf
}

// Stats counts structural work and node visits (pages entered).
type Stats struct {
	Splits, Merges, BorrowsL, BorrowsR, RootGrows, RootShrinks int
	Visits                                                     int
}

type OrderedMap[K cmp.Ordered, V any] struct {
	t     int
	pages []page[K, V]
	free  []int32 // ids of pages merged away
	root  int32
	n     int
	Stats
	bug int // planted bugs for part (c): 0 none, 1 a merge forgets to pull the separator down, 2 a left borrow leaves a stale value behind
}

func NewOrderedMap[K cmp.Ordered, V any](t int) *OrderedMap[K, V] {
	if t < 2 {
		panic("OrderedMap: minimum degree must be at least 2")
	}
	m := &OrderedMap[K, V]{t: t}
	m.root = m.alloc()
	return m
}

func (m *OrderedMap[K, V]) Len() int { return m.n }

func (m *OrderedMap[K, V]) alloc() int32 {
	if n := len(m.free); n > 0 {
		id := m.free[n-1]
		m.free = m.free[:n-1]
		return id
	}
	m.pages = append(m.pages, page[K, V]{})
	return int32(len(m.pages) - 1)
}

func (m *OrderedMap[K, V]) release(id int32) {
	m.pages[id] = page[K, V]{}
	m.free = append(m.free, id)
}

func (m *OrderedMap[K, V]) Get(k K) (v V, ok bool) {
	x := m.root
	for {
		m.Visits++
		pg := &m.pages[x]
		i, found := slices.BinarySearch(pg.keys, k)
		if found {
			return pg.vals[i], true
		}
		if len(pg.kids) == 0 {
			return v, false
		}
		x = pg.kids[i]
	}
}

// splitChild splits the full child kids[i] of x around its median key, which
// moves up into x. Taking the pointers only after alloc() matters: alloc may
// grow (and so move) the arena.
func (m *OrderedMap[K, V]) splitChild(x int32, i int) {
	t := m.t
	yid := m.pages[x].kids[i]
	zid := m.alloc()
	y, z := &m.pages[yid], &m.pages[zid]
	z.keys = append(z.keys, y.keys[t:]...)
	z.vals = append(z.vals, y.vals[t:]...)
	medK, medV := y.keys[t-1], y.vals[t-1]
	if len(y.kids) > 0 {
		z.kids = append(z.kids, y.kids[t:]...)
		y.kids = y.kids[:t]
	}
	y.keys, y.vals = y.keys[:t-1], y.vals[:t-1]
	px := &m.pages[x]
	px.keys = slices.Insert(px.keys, i, medK)
	px.vals = slices.Insert(px.vals, i, medV)
	px.kids = slices.Insert(px.kids, i+1, zid)
	m.Splits++
}

// Put is an upsert: it returns the old value if the key was already there.
// A full page is split on the way down, even when the key turns out to be an
// update; that costs one harmless split now and then and keeps the pass single.
func (m *OrderedMap[K, V]) Put(k K, v V) (old V, replaced bool) {
	full := 2*m.t - 1
	if len(m.pages[m.root].keys) == full { // a full root: the tree grows at the top
		nr := m.alloc()
		m.pages[nr].kids = []int32{m.root}
		m.root = nr
		m.splitChild(nr, 0)
		m.RootGrows++
	}
	x := m.root
	for {
		m.Visits++
		pg := &m.pages[x]
		i, found := slices.BinarySearch(pg.keys, k)
		if found {
			old, pg.vals[i] = pg.vals[i], v
			return old, true
		}
		if len(pg.kids) == 0 {
			pg.keys = slices.Insert(pg.keys, i, k)
			pg.vals = slices.Insert(pg.vals, i, v)
			m.n++
			return old, false
		}
		if len(m.pages[pg.kids[i]].keys) == full {
			m.splitChild(x, i)
			pg = &m.pages[x] // the arena may have moved
			switch c := cmp.Compare(k, pg.keys[i]); {
			case c == 0: // the median that moved up is the key
				old, pg.vals[i] = pg.vals[i], v
				return old, true
			case c > 0:
				i++
			}
		}
		x = pg.kids[i]
	}
}

// merge folds kids[j], the separator keys[j] and kids[j+1] into kids[j]:
// 2(t-1) + 1 = 2t-1 keys, exactly a full page.
func (m *OrderedMap[K, V]) merge(x int32, j int) {
	px := &m.pages[x]
	lid, rid := px.kids[j], px.kids[j+1]
	l, r := &m.pages[lid], &m.pages[rid]
	if m.bug != 1 {
		l.keys = append(l.keys, px.keys[j])
		l.vals = append(l.vals, px.vals[j])
	}
	l.keys = append(l.keys, r.keys...)
	l.vals = append(l.vals, r.vals...)
	l.kids = append(l.kids, r.kids...)
	px.keys = slices.Delete(px.keys, j, j+1)
	px.vals = slices.Delete(px.vals, j, j+1)
	px.kids = slices.Delete(px.kids, j+1, j+2)
	m.release(rid)
	m.Merges++
}

// borrowFromLeft: the separator comes down into the front of kids[i], the left
// sibling's last key goes up to replace it, and (internal pages) the sibling's
// last child crosses over.
func (m *OrderedMap[K, V]) borrowFromLeft(x int32, i int) {
	px := &m.pages[x]
	c, s := &m.pages[px.kids[i]], &m.pages[px.kids[i-1]]
	c.keys = slices.Insert(c.keys, 0, px.keys[i-1])
	c.vals = slices.Insert(c.vals, 0, px.vals[i-1])
	last := len(s.keys) - 1
	px.keys[i-1], px.vals[i-1] = s.keys[last], s.vals[last]
	if m.bug == 2 { // planted: the key moves up, its value stays where the parent's old one was
		px.vals[i-1] = c.vals[0]
	}
	s.keys, s.vals = s.keys[:last], s.vals[:last]
	if len(s.kids) > 0 {
		c.kids = slices.Insert(c.kids, 0, s.kids[len(s.kids)-1])
		s.kids = s.kids[:len(s.kids)-1]
	}
	m.BorrowsL++
}

func (m *OrderedMap[K, V]) borrowFromRight(x int32, i int) {
	px := &m.pages[x]
	c, s := &m.pages[px.kids[i]], &m.pages[px.kids[i+1]]
	c.keys = append(c.keys, px.keys[i])
	c.vals = append(c.vals, px.vals[i])
	px.keys[i], px.vals[i] = s.keys[0], s.vals[0]
	s.keys, s.vals = slices.Delete(s.keys, 0, 1), slices.Delete(s.vals, 0, 1)
	if len(s.kids) > 0 {
		c.kids = append(c.kids, s.kids[0])
		s.kids = slices.Delete(s.kids, 0, 1)
	}
	m.BorrowsR++
}

// ensure makes kids[i] of x hold at least t keys, by borrowing from a sibling
// that can spare one or, failing that, merging with a sibling. It returns the
// index of the child that now covers the key (a merge with the LEFT sibling,
// used only for the last child, shifts it down by one).
func (m *OrderedMap[K, V]) ensure(x int32, i int) int {
	px := &m.pages[x]
	if len(m.pages[px.kids[i]].keys) >= m.t {
		return i
	}
	switch {
	case i > 0 && len(m.pages[px.kids[i-1]].keys) >= m.t:
		m.borrowFromLeft(x, i)
		return i
	case i < len(px.kids)-1 && len(m.pages[px.kids[i+1]].keys) >= m.t:
		m.borrowFromRight(x, i)
		return i
	case i < len(px.kids)-1:
		m.merge(x, i)
		return i
	default:
		m.merge(x, i-1)
		return i - 1
	}
}

// extreme returns the largest (wantMax) or smallest key of the subtree at x.
func (m *OrderedMap[K, V]) extreme(x int32, wantMax bool) (K, V) {
	for {
		m.Visits++
		pg := &m.pages[x]
		if len(pg.kids) == 0 {
			if wantMax {
				return pg.keys[len(pg.keys)-1], pg.vals[len(pg.vals)-1]
			}
			return pg.keys[0], pg.vals[0]
		}
		if wantMax {
			x = pg.kids[len(pg.kids)-1]
		} else {
			x = pg.kids[0]
		}
	}
}

// del removes k from the subtree at x. On entry x has at least t keys, or is the root.
func (m *OrderedMap[K, V]) del(x int32, k K) (v V, ok bool) {
	m.Visits++
	pg := &m.pages[x]
	i, found := slices.BinarySearch(pg.keys, k)
	if len(pg.kids) == 0 { // a leaf: remove it, or it was never here
		if !found {
			return v, false
		}
		v = pg.vals[i]
		pg.keys = slices.Delete(pg.keys, i, i+1)
		pg.vals = slices.Delete(pg.vals, i, i+1)
		return v, true
	}
	if found { // the key sits in an internal page
		v = pg.vals[i]
		left, right := pg.kids[i], pg.kids[i+1]
		switch {
		case len(m.pages[left].keys) >= m.t: // a rich left child: its maximum takes the key's place
			pk, pv := m.extreme(left, true)
			pg.keys[i], pg.vals[i] = pk, pv
			m.del(left, pk)
		case len(m.pages[right].keys) >= m.t: // else a rich right child: its minimum
			sk, sv := m.extreme(right, false)
			pg.keys[i], pg.vals[i] = sk, sv
			m.del(right, sk)
		default: // both children are minimal: merge them around the key, then delete it from the merged page
			m.merge(x, i)
			m.del(left, k)
		}
		return v, true
	}
	i = m.ensure(x, i) // the key is not here: enter kids[i], after making sure it can spare a key
	return m.del(m.pages[x].kids[i], k)
}

func (m *OrderedMap[K, V]) Delete(k K) (v V, ok bool) {
	v, ok = m.del(m.root, k)
	if ok {
		m.n--
	}
	if r := &m.pages[m.root]; len(r.keys) == 0 && len(r.kids) > 0 { // a merge emptied the root: the tree shrinks at the top
		old := m.root
		m.root = r.kids[0]
		m.release(old)
		m.RootShrinks++
	}
	return v, ok
}

func (m *OrderedMap[K, V]) Min() (k K, v V, ok bool) {
	x := m.root
	for {
		m.Visits++
		pg := &m.pages[x]
		if len(pg.kids) == 0 {
			if len(pg.keys) == 0 {
				return k, v, false
			}
			return pg.keys[0], pg.vals[0], true
		}
		x = pg.kids[0]
	}
}

func (m *OrderedMap[K, V]) Max() (k K, v V, ok bool) {
	x := m.root
	for {
		m.Visits++
		pg := &m.pages[x]
		if len(pg.kids) == 0 {
			if len(pg.keys) == 0 {
				return k, v, false
			}
			return pg.keys[len(pg.keys)-1], pg.vals[len(pg.vals)-1], true
		}
		x = pg.kids[len(pg.kids)-1]
	}
}

// Floor is the greatest key <= k; Ceil the least key >= k. One descent each:
// remember the last candidate seen on the way down, the deepest one is nearest.
func (m *OrderedMap[K, V]) Floor(k K) (fk K, fv V, ok bool) {
	x := m.root
	for {
		m.Visits++
		pg := &m.pages[x]
		i, found := slices.BinarySearch(pg.keys, k)
		if found {
			return pg.keys[i], pg.vals[i], true
		}
		if i > 0 {
			fk, fv, ok = pg.keys[i-1], pg.vals[i-1], true
		}
		if len(pg.kids) == 0 {
			return fk, fv, ok
		}
		x = pg.kids[i]
	}
}

func (m *OrderedMap[K, V]) Ceil(k K) (ck K, cv V, ok bool) {
	x := m.root
	for {
		m.Visits++
		pg := &m.pages[x]
		i, found := slices.BinarySearch(pg.keys, k)
		if found {
			return pg.keys[i], pg.vals[i], true
		}
		if i < len(pg.keys) {
			ck, cv, ok = pg.keys[i], pg.vals[i], true
		}
		if len(pg.kids) == 0 {
			return ck, cv, ok
		}
		x = pg.kids[i]
	}
}

// Range yields the pairs with lo <= key < hi in order: one descent to lo, then
// an in-order walk with an explicit stack of (page, next key index). A page
// below a key is entered only when the consumer asks for the next pair, so
// breaking out of the loop early costs nothing more.
func (m *OrderedMap[K, V]) Range(lo, hi K) iter.Seq2[K, V] {
	type frame struct {
		x int32
		i int
	}
	return func(yield func(K, V) bool) {
		var stack []frame
		x := m.root
		for { // descend along the path to lo
			m.Visits++
			pg := &m.pages[x]
			i, _ := slices.BinarySearch(pg.keys, lo)
			stack = append(stack, frame{x, i})
			if len(pg.kids) == 0 {
				break
			}
			x = pg.kids[i]
		}
		for len(stack) > 0 {
			top := &stack[len(stack)-1]
			pg := &m.pages[top.x]
			if top.i >= len(pg.keys) {
				stack = stack[:len(stack)-1]
				continue
			}
			k, v := pg.keys[top.i], pg.vals[top.i]
			if cmp.Compare(k, hi) >= 0 {
				return
			}
			top.i++
			if !yield(k, v) {
				return
			}
			if len(pg.kids) > 0 { // the subtree to the right of the key comes next
				x = pg.kids[top.i]
				for {
					m.Visits++
					stack = append(stack, frame{x, 0})
					if len(m.pages[x].kids) == 0 {
						break
					}
					x = m.pages[x].kids[0]
				}
			}
		}
	}
}

func (m *OrderedMap[K, V]) height() int {
	h := 1
	for x := m.root; len(m.pages[x].kids) > 0; x = m.pages[x].kids[0] {
		h++
	}
	return h
}

// maxLevels is the theorem: n keys fit in at most floor(log_t((n+1)/2)) + 1 levels.
func maxLevels(n, t int) int {
	e := 0
	for p := 1; n > 0 && p*t*2 <= n+1; p *= t {
		e++
	}
	return e + 1
}

// check verifies every invariant and returns the contents in key order. It
// returns an error, never panics, so a planted bug can be reported.
func (m *OrderedMap[K, V]) check() (keys []K, vals []V, err error) {
	reached := make([]bool, len(m.pages))
	leafDepth := -1
	var walk func(x int32, depth int, lo, hi *K) error
	walk = func(x int32, depth int, lo, hi *K) error {
		if x < 0 || int(x) >= len(m.pages) {
			return fmt.Errorf("page id %d is outside the arena", x)
		}
		if reached[x] {
			return fmt.Errorf("page %d is reachable twice", x)
		}
		reached[x] = true
		pg := &m.pages[x]
		if len(pg.keys) != len(pg.vals) {
			return fmt.Errorf("page %d has %d keys but %d values", x, len(pg.keys), len(pg.vals))
		}
		lowest, highest := m.t-1, 2*m.t-1
		if len(pg.keys) > highest || (x != m.root && len(pg.keys) < lowest) {
			return fmt.Errorf("page %d holds %d keys (allowed %d..%d)", x, len(pg.keys), lowest, highest)
		}
		for i, k := range pg.keys {
			if (lo != nil && k <= *lo) || (hi != nil && k >= *hi) || (i > 0 && k <= pg.keys[i-1]) {
				return fmt.Errorf("page %d: key %v is out of order or outside its separators", x, k)
			}
		}
		if len(pg.kids) == 0 {
			if leafDepth >= 0 && depth != leafDepth {
				return fmt.Errorf("leaves at depths %d and %d", leafDepth, depth)
			}
			leafDepth = depth
			keys, vals = append(keys, pg.keys...), append(vals, pg.vals...)
			return nil
		}
		if len(pg.kids) != len(pg.keys)+1 {
			return fmt.Errorf("page %d has %d keys and %d children", x, len(pg.keys), len(pg.kids))
		}
		if len(pg.keys) == 0 {
			return fmt.Errorf("internal page %d has no keys", x)
		}
		for i, kid := range pg.kids {
			l, h := lo, hi
			if i > 0 {
				l = &pg.keys[i-1]
			}
			if i < len(pg.keys) {
				h = &pg.keys[i]
			}
			if err := walk(kid, depth+1, l, h); err != nil {
				return err
			}
			if i < len(pg.keys) {
				keys, vals = append(keys, pg.keys[i]), append(vals, pg.vals[i])
			}
		}
		return nil
	}
	if err := walk(m.root, 0, nil, nil); err != nil {
		return nil, nil, err
	}
	if len(keys) != m.n {
		return nil, nil, fmt.Errorf("%d keys are reachable but Len() is %d", len(keys), m.n)
	}
	onFree := make([]bool, len(m.pages))
	for _, id := range m.free {
		if reached[id] || onFree[id] {
			return nil, nil, fmt.Errorf("free page %d is reachable or listed twice", id)
		}
		onFree[id] = true
	}
	for id := range m.pages {
		if !reached[id] && !onFree[id] {
			return nil, nil, fmt.Errorf("page %d is neither reachable nor free (a leak)", id)
		}
	}
	if h := m.height(); h > maxLevels(m.n, m.t) {
		return nil, nil, fmt.Errorf("%d levels for %d keys exceeds the bound %d", h, m.n, maxLevels(m.n, m.t))
	}
	return keys, vals, nil
}

// ---------------------------------------------------------------------------
// The proof: a sorted slice of pairs is the oracle.

type entry struct{ k, v int }

type oracle []entry

func (o oracle) find(k int) (int, bool) {
	return slices.BinarySearchFunc(o, k, func(e entry, k int) int { return cmp.Compare(e.k, k) })
}

// res is the answer of one operation, so any two answers compare with ==.
type res struct {
	k, v int
	ok   bool
}

type report struct {
	ops, putNew, putOld, delHit, delMiss, ranges, rangeKeys int
	maxKeys, maxLevels, getVisits, gets                     int
	mismatches, invFailures                                 int
	firstOp                                                 int
	firstDesc, firstResult, firstCheck                      string
	st                                                      Stats
}

type harness struct {
	m   *OrderedMap[int, int]
	o   oracle
	rng *rand.Rand
	r   report
	// the problems of the operation being run
	desc, resultMsg string
}

func (h *harness) mismatch(format string, args ...any) {
	h.r.mismatches++
	if h.resultMsg == "" {
		h.resultMsg = fmt.Sprintf(format, args...)
	}
}

func (h *harness) expect(got, want res) {
	if got != want {
		h.mismatch("%s returned %v, oracle says %v", h.desc, got, want)
	}
}

// step runs ONE random operation on the map and on the oracle and compares
// their answers. op is the 1-based operation number; values are op*1000+key, so
// a stale value (an update that did not take, a value left behind by a borrow)
// is told apart from a fresh one.
func (h *harness) step(op int) {
	kind, k := h.rng.IntN(100), h.rng.IntN(600)
	h.resultMsg = ""
	i, found := h.o.find(k) // where k is, or would go
	var at entry
	if found {
		at = h.o[i]
	}
	switch {
	case kind < 40:
		h.desc = fmt.Sprintf("Put(%d)", k)
		v := op*1000 + k
		if found {
			h.o[i].v = v
			h.r.putOld++
		} else {
			h.o = slices.Insert(h.o, i, entry{k, v})
			h.r.putNew++
		}
		old, replaced := h.m.Put(k, v)
		h.expect(res{v: old, ok: replaced}, res{v: at.v, ok: found})
	case kind < 75:
		h.desc = fmt.Sprintf("Delete(%d)", k)
		if found {
			h.o = slices.Delete(h.o, i, i+1)
			h.r.delHit++
		} else {
			h.r.delMiss++
		}
		v, ok := h.m.Delete(k)
		h.expect(res{v: v, ok: ok}, res{v: at.v, ok: found})
	case kind < 85:
		h.desc = fmt.Sprintf("Get(%d)", k)
		before := h.m.Visits
		v, ok := h.m.Get(k)
		h.r.getVisits += h.m.Visits - before
		h.r.gets++
		h.expect(res{v: v, ok: ok}, res{v: at.v, ok: found})
	case kind < 90:
		h.desc = fmt.Sprintf("Floor(%d)", k)
		if found { // an exact hit is its own floor
			i++
		}
		want := res{}
		if i > 0 {
			want = res{h.o[i-1].k, h.o[i-1].v, true}
		}
		fk, fv, ok := h.m.Floor(k)
		h.expect(res{fk, fv, ok}, want)
	case kind < 95:
		h.desc = fmt.Sprintf("Ceil(%d)", k)
		want := res{}
		if i < len(h.o) {
			want = res{h.o[i].k, h.o[i].v, true}
		}
		ck, cv, ok := h.m.Ceil(k)
		h.expect(res{ck, cv, ok}, want)
	default:
		hi := k + h.rng.IntN(120)
		h.desc = fmt.Sprintf("Range(%d, %d)", k, hi)
		j, _ := h.o.find(hi)
		want := h.o[i:j]
		h.r.ranges++
		h.r.rangeKeys += len(want)
		var got []entry
		for rk, rv := range h.m.Range(k, hi) {
			got = append(got, entry{rk, rv})
		}
		if !slices.Equal(got, want) {
			h.mismatch("%s yielded %d pairs, oracle has %d (or they differ)", h.desc, len(got), len(want))
		}
	}
}

// verify runs the invariant checker and compares the whole contents, Len, Min
// and Max with the oracle. It returns the checker's message ("" when it passes).
func (h *harness) verify() string {
	keys, vals, err := h.m.check()
	msg := ""
	switch {
	case err != nil:
		msg = err.Error()
	case len(keys) != len(h.o):
		msg = fmt.Sprintf("the map holds %d keys, the oracle %d", len(keys), len(h.o))
	default:
		for i, e := range h.o {
			if keys[i] != e.k || vals[i] != e.v {
				msg = fmt.Sprintf("contents differ at pair %d: map (%d, %d), oracle (%d, %d)", i, keys[i], vals[i], e.k, e.v)
				break
			}
		}
	}
	if msg != "" {
		h.r.invFailures++
	}
	if h.m.Len() != len(h.o) {
		h.mismatch("Len() is %d, oracle has %d", h.m.Len(), len(h.o))
	}
	var first, last res
	if len(h.o) > 0 {
		first, last = res{h.o[0].k, h.o[0].v, true}, res{h.o[len(h.o)-1].k, h.o[len(h.o)-1].v, true}
	}
	mk, mv, ok := h.m.Min()
	h.desc = "Min()"
	h.expect(res{mk, mv, ok}, first)
	xk, xv, ok := h.m.Max()
	h.desc = "Max()"
	h.expect(res{xk, xv, ok}, last)
	h.r.maxKeys = max(h.r.maxKeys, len(h.o))
	h.r.maxLevels = max(h.r.maxLevels, h.m.height())
	return msg
}

// runOps drives `ops` random operations from PCG(1, 2) (the same stream for
// every t) and checks after EVERY one. With stopAtFirst it returns at the first
// problem and remembers which operation it was.
func runOps(t, ops int, stopAtFirst bool, bug int) report {
	h := &harness{m: NewOrderedMap[int, int](t), rng: rand.New(rand.NewPCG(1, 2))}
	h.m.bug = bug
	for op := 1; op <= ops; op++ {
		h.step(op)
		h.r.ops = op
		desc := h.desc
		msg := h.verify()
		if (msg != "" || h.resultMsg != "") && h.r.firstOp == 0 {
			h.r.firstOp, h.r.firstDesc, h.r.firstResult, h.r.firstCheck = op, desc, h.resultMsg, msg
		}
		if stopAtFirst && h.r.firstOp != 0 {
			break
		}
	}
	h.r.st = h.m.Stats
	return h.r
}

// growShrink: up to 5,000 keys in random order, down to 0 in another random
// order, up again in sorted order; the checker runs after every operation, and
// the empty tree gets every query. It returns the levels at 5,000 keys, at 0
// keys and after the regrowth, and the harness for its counters.
func growShrink(t int) (levels [3]int, h *harness) {
	const n = 5000
	rng := rand.New(rand.NewPCG(7, 8))
	shuffled := func() []int {
		a := make([]int, n)
		for i := range a {
			a[i] = i
		}
		for i := n - 1; i >= 1; i-- {
			j := rng.IntN(i + 1)
			a[i], a[j] = a[j], a[i]
		}
		return a
	}
	h = &harness{m: NewOrderedMap[int, int](t)}
	put := func(k int) {
		i, _ := h.o.find(k)
		h.o = slices.Insert(h.o, i, entry{k, k*7 + 1})
		h.m.Put(k, k*7+1)
		h.verify()
	}
	for _, k := range shuffled() {
		put(k)
	}
	levels[0] = h.m.height()
	for _, k := range shuffled() {
		i, _ := h.o.find(k)
		want := h.o[i].v
		h.o = slices.Delete(h.o, i, i+1)
		v, ok := h.m.Delete(k)
		h.desc = fmt.Sprintf("Delete(%d)", k)
		h.expect(res{v: v, ok: ok}, res{v: want, ok: true})
		h.verify()
	}
	levels[1] = h.m.height()
	// the empty tree answers every query with "nothing", and deleting from it changes nothing
	_, ok1 := h.m.Get(3)
	_, _, ok2 := h.m.Floor(3)
	_, _, ok3 := h.m.Ceil(3)
	_, ok4 := h.m.Delete(3)
	empty := true
	for range h.m.Range(0, n) {
		empty = false
	}
	if ok1 || ok2 || ok3 || ok4 || !empty {
		h.mismatch("the empty map answered a query with a key")
	}
	h.verify()
	for k := 0; k < n; k++ {
		put(k)
	}
	levels[2] = h.m.height()
	return levels, h
}

// ---------------------------------------------------------------------------
// The timing: the alternatives Go actually has.

// inputs are the probe keys, generated once per size.
type inputs struct {
	hits, fresh, starts []int
}

const span = 1_000 // pairs in one Range

// bencher is one structure under test. run performs `op` once over the inputs
// (0: Get every hit, 1: Put+Delete every fresh key, 2: Range from every start)
// and reports false when the structure cannot do that operation.
type bencher interface {
	run(op int, in *inputs) bool
}

var sink int // keeps the results alive

type treeBench struct{ m *OrderedMap[int, int] }

func (b treeBench) run(op int, in *inputs) bool {
	switch op {
	case 0:
		for _, k := range in.hits {
			v, _ := b.m.Get(k)
			sink += v
		}
	case 1:
		for _, k := range in.fresh {
			b.m.Put(k, k)
			b.m.Delete(k)
		}
	default:
		for _, lo := range in.starts {
			c := 0
			for _, v := range b.m.Range(lo, lo+2*span) {
				sink += v
				c++
			}
			if c != span {
				panic("range returned the wrong count")
			}
		}
	}
	return op < 2 || len(in.starts) > 0
}

type sliceBench struct{ s *[]entry }

func (b sliceBench) run(op int, in *inputs) bool {
	s := *b.s
	find := func(k int) int {
		i, _ := slices.BinarySearchFunc(s, k, func(e entry, k int) int { return cmp.Compare(e.k, k) })
		return i
	}
	switch op {
	case 0:
		for _, k := range in.hits {
			sink += s[find(k)].v
		}
	case 1:
		for _, k := range in.fresh {
			s = slices.Insert(s, find(k), entry{k, k})
			i := find(k)
			s = slices.Delete(s, i, i+1)
		}
		*b.s = s
	default:
		for _, lo := range in.starts {
			c := 0
			for i := find(lo); i < len(s) && s[i].k < lo+2*span; i++ {
				sink += s[i].v
				c++
			}
			if c != span {
				panic("range returned the wrong count")
			}
		}
	}
	return op < 2 || len(in.starts) > 0
}

type mapBench struct{ m map[int]int }

func (b mapBench) run(op int, in *inputs) bool {
	switch op {
	case 0:
		for _, k := range in.hits {
			sink += b.m[k]
		}
	case 1:
		for _, k := range in.fresh {
			b.m[k] = k
			delete(b.m, k)
		}
	}
	return op < 2 // a map has no order, so no Range
}

// bench returns ns per op for [structure][operation] at size n, each the best
// of 3 interleaved rounds (-1: not measured), and the levels of the B-tree. Keys
// are the even numbers 0..2n-2, so every odd number is a fresh key. The
// structures are built once, the tree from shuffled input (so the sorted slice
// never pays for its own inserts).
func bench(n int) (best [3][3]float64, levels int) {
	const gets, pairs, scans = 200_000, 2_000, 200
	rng := rand.New(rand.NewPCG(9, 10))
	order := make([]int, n)
	for i := range order {
		order[i] = i
	}
	for i := n - 1; i >= 1; i-- {
		j := rng.IntN(i + 1)
		order[i], order[j] = order[j], order[i]
	}
	tree, hash, sorted := NewOrderedMap[int, int](16), make(map[int]int, n), make([]entry, n)
	for _, i := range order {
		tree.Put(2*i, 2*i)
		hash[2*i] = 2 * i
	}
	for i := range sorted {
		sorted[i] = entry{2 * i, 2 * i}
	}
	in := &inputs{make([]int, gets), make([]int, pairs), nil}
	for i := range in.hits {
		in.hits[i] = 2 * rng.IntN(n)
	}
	for i := range in.fresh {
		in.fresh[i] = 2*rng.IntN(n) + 1
	}
	if n >= span { // a Range of 1,000 pairs needs 1,000 keys
		in.starts = make([]int, scans)
		for i := range in.starts {
			in.starts[i] = 2 * rng.IntN(n-span+1) // [start, start+2*span) holds exactly span keys
		}
	}
	structs := []bencher{treeBench{tree}, sliceBench{&sorted}, mapBench{hash}}
	per := [3]int{gets, pairs, scans}
	for s := range best {
		for op := range best[s] {
			best[s][op] = -1
		}
	}
	for round := 0; round < 3; round++ {
		for s, b := range structs {
			for op := range per {
				start := time.Now()
				ok := b.run(op, in)
				ns := float64(time.Since(start).Nanoseconds()) / float64(per[op])
				if ok && (best[s][op] < 0 || ns < best[s][op]) {
					best[s][op] = ns
				}
			}
		}
	}
	return best, tree.height()
}

func fmtNs(x float64) string {
	switch {
	case x >= 1e6:
		return fmt.Sprintf("%.2f ms", x/1e6)
	case x >= 1e3:
		return fmt.Sprintf("%.2f us", x/1e3)
	}
	return fmt.Sprintf("%.1f ns", x)
}

func main() {
	fmt.Println("OrderedMap[K, V]: a B-tree with Get, Put, Delete, Min, Max, Floor, Ceil, Range, Len")
	fmt.Println("Go's standard library has no ordered map, so there is no stdlib row to beat here:")
	fmt.Println("the proof is against a sorted slice of pairs, the measurements against a sorted slice and a built-in map.")

	// ---- (a) the proof ----
	const ops = 200_000
	degrees := []int{2, 3, 6, 16}
	fmt.Printf("\n== (a) proof: %d random operations per t, checker and oracle comparison after EVERY operation ==\n", ops)
	fmt.Println("operations drawn from PCG(1, 2), the same stream for every t: Put 40%, Delete 35%, Get 10%, Floor 5%, Ceil 5%, Range 5%, keys 0..599")
	fmt.Println("values are op*1000+key; the checker: keys per page t-1..2t-1 (root exempt), equal leaf depth, keys sorted and inside their separators,")
	fmt.Println("children = keys+1, Len() = reachable keys, every page reachable or on the free list, levels <= floor(log_t((n+1)/2))+1")
	reports := make([]report, len(degrees))
	for i, t := range degrees {
		reports[i] = runOps(t, ops, false, 0)
	}
	r := reports[0]
	fmt.Printf("workload: Put %d new keys + %d replacing; Delete %d hits + %d absent; %d Range scans returning %d pairs in all; the map held at most %d keys\n",
		r.putNew, r.putOld, r.delHit, r.delMiss, r.ranges, r.rangeKeys, r.maxKeys)
	fmt.Printf("%3s %8s %8s %8s %8s %9s %10s %10s %11s %11s\n",
		"t", "splits", "merges", "borrowsL", "borrowsR", "root +/-", "max levels", "pages/Get", "mismatches", "inv. fails")
	total := 0
	for i, t := range degrees {
		r, s := reports[i], reports[i].st
		fmt.Printf("%3d %8d %8d %8d %8d %4d/%-4d %10d %10.2f %11d %11d\n",
			t, s.Splits, s.Merges, s.BorrowsL, s.BorrowsR, s.RootGrows, s.RootShrinks, r.maxLevels,
			float64(r.getVisits)/float64(r.gets), r.mismatches, r.invFailures)
		total += r.ops
	}
	fmt.Printf("%d operations checked in all\n", total)

	// ---- (b) grow, shrink, grow ----
	fmt.Println("\n== (b) grow to 5,000 keys, shrink to 0, grow again (checker after every operation; the empty map gets every query) ==")
	fmt.Printf("%3s %9s %9s %10s %10s %7s %7s %7s %8s %11s\n",
		"t", "levels@5k", "levels@0", "levels@5k'", "max levels", "root +", "root -", "splits", "merges", "borrows")
	for _, t := range degrees {
		lv, h := growShrink(t)
		s := h.m.Stats
		fmt.Printf("%3d %9d %9d %10d %10d %7d %7d %7d %8d %11d\n",
			t, lv[0], lv[1], lv[2], h.r.maxLevels, s.RootGrows, s.RootShrinks, s.Splits, s.Merges, s.BorrowsL+s.BorrowsR)
		fmt.Printf("    levels = 1 + root growths - root shrinks: %v; empty map has %d level; mismatches %d, invariant failures %d\n",
			1+s.RootGrows-s.RootShrinks == lv[2], lv[1], h.r.mismatches, h.r.invFailures)
	}

	// ---- (c) planted bugs ----
	fmt.Println("\n== (c) planted bugs at t = 2: the same operations, stopped at the first problem ==")
	bugs := []string{"", "merge forgets to pull the separator down", "a left borrow moves the key up but leaves a stale value"}
	for bug := 1; bug < len(bugs); bug++ {
		fmt.Printf("bug %d, %s:\n", bug, bugs[bug])
		b := runOps(2, ops, true, bug)
		if b.firstOp == 0 {
			fmt.Println("  the checker did NOT notice the bug in", ops, "operations")
			continue
		}
		fmt.Printf("  first problem at operation #%d, %s\n", b.firstOp, b.firstDesc)
		if b.firstCheck != "" {
			fmt.Printf("  invariant checker: %s\n", b.firstCheck)
		} else {
			fmt.Println("  invariant checker: passed")
		}
		if b.firstResult != "" {
			fmt.Printf("  the operation's own answer: %s\n", b.firstResult)
		} else {
			fmt.Println("  the operation's own answer: agreed with the oracle")
		}
	}
	fmt.Printf("without a bug the same %d operations at t = 2 (part a): %d mismatches, %d invariant failures\n",
		ops, reports[0].mismatches, reports[0].invFailures)

	// ---- (d) the clock ----
	fmt.Println("\n== (d) time per operation, best of 3 interleaved rounds ==")
	fmt.Println("Go's standard library has no ordered map; these are the alternatives: an OrderedMap B-tree (t = 16),")
	fmt.Println("a sorted slice (slices.BinarySearchFunc, slices.Insert, slices.Delete) and a built-in map, which has no order, so no Range.")
	fmt.Println("keys are 0, 2, ..., 2n-2; Put+Delete is a pair on a key that is not in the map (the size stays n); a Range returns 1,000 pairs")
	sizes := []int{16, 64, 256, 1_000, 10_000, 100_000, 1_000_000}
	results, levels := make([][3][3]float64, len(sizes)), make([]int, len(sizes))
	for i, n := range sizes {
		results[i], levels[i] = bench(n)
	}
	names := [3]string{"B-tree t=16", "sorted slice", "built-in map"}
	ops3 := [3]string{"Get (random hit)", "Put+Delete pair", "Range of 1,000 pairs"}
	fmt.Printf("%-22s %-13s", "operation", "structure")
	for _, n := range sizes {
		fmt.Printf(" %12s", fmt.Sprintf("n=%d", n))
	}
	fmt.Println()
	for op := range ops3 {
		for s := range names {
			label := ""
			if s == 0 {
				label = ops3[op]
			}
			fmt.Printf("%-22s %-13s", label, names[s])
			for i := range sizes {
				switch {
				case op == 2 && s == 2:
					fmt.Printf(" %12s", "n/a: no order")
				case results[i][s][op] < 0:
					fmt.Printf(" %12s", "n/a: n < 1000")
				default:
					fmt.Printf(" %12s", fmtNs(results[i][s][op]))
				}
			}
			fmt.Println()
		}
	}
	fmt.Printf("%-36s", "B-tree levels")
	for i := range sizes {
		fmt.Printf(" %12d", levels[i])
	}
	fmt.Println()
	fmt.Println()
	for op := range ops3 {
		var at []string // the fastest structure at each size where the operation was measured
		first, same := -1, true
		for i, n := range sizes {
			w := -1
			for s := range names {
				if results[i][s][op] >= 0 && (w < 0 || results[i][s][op] < results[i][w][op]) {
					w = s
				}
			}
			if w < 0 {
				continue
			}
			if first < 0 {
				first = w
			}
			same = same && w == first
			at = append(at, fmt.Sprintf("n=%d %s", n, names[w]))
		}
		if same {
			fmt.Printf("fastest, %s: %s at every size measured\n", ops3[op], names[first])
		} else {
			fmt.Printf("fastest, %s: %s\n", ops3[op], strings.Join(at, "; "))
		}
	}
	for i, n := range sizes {
		q := results[i][1][1] / results[i][0][1] // sorted slice time over B-tree time
		verdict := "within 25%: no clear winner"
		if q > 1.25 {
			verdict = fmt.Sprintf("the B-tree is faster, %.1fx", q)
		} else if q < 0.8 {
			verdict = fmt.Sprintf("the sorted slice is faster, %.1fx", 1/q)
		}
		fmt.Printf("Put+Delete at n=%-9d: sorted slice %s, B-tree %s: %s\n",
			n, fmtNs(results[i][1][1]), fmtNs(results[i][0][1]), verdict)
	}
}
```

**Sample output:**

```
OrderedMap[K, V]: a B-tree with Get, Put, Delete, Min, Max, Floor, Ceil, Range, Len
Go's standard library has no ordered map, so there is no stdlib row to beat here:
the proof is against a sorted slice of pairs, the measurements against a sorted slice and a built-in map.

== (a) proof: 200000 random operations per t, checker and oracle comparison after EVERY operation ==
operations drawn from PCG(1, 2), the same stream for every t: Put 40%, Delete 35%, Get 10%, Floor 5%, Ceil 5%, Range 5%, keys 0..599
values are op*1000+key; the checker: keys per page t-1..2t-1 (root exempt), equal leaf depth, keys sorted and inside their separators,
children = keys+1, Len() = reachable keys, every page reachable or on the free list, levels <= floor(log_t((n+1)/2))+1
workload: Put 37377 new keys + 42769 replacing; Delete 37051 hits + 32874 absent; 9786 Range scans returning 285022 pairs in all; the map held at most 366 keys
  t   splits   merges borrowsL borrowsR  root +/- max levels  pages/Get  mismatches  inv. fails
  2    48760    48559    39840    33915  998/993           7       5.69           0           0
  3    17365    17255    20100    13528 6358/6355          5       3.91           0           0
  6     2578     2533     9622     4549    5/3             3       2.92           0           0
 16      367      352     4136     1000    2/1             2       1.98           0           0
800000 operations checked in all

== (b) grow to 5,000 keys, shrink to 0, grow again (checker after every operation; the empty map gets every query) ==
  t levels@5k  levels@0 levels@5k' max levels  root +  root -  splits   merges     borrows
  2        10         1         12         12      20       9    7827     2846        4663
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0
  3         6         1          8          8      12       5    4048     1560        3185
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0
  6         4         1          5          5       7       3    1667      674        2161
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0
 16         3         1          3          3       4       2     562      233        1538
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0

== (c) planted bugs at t = 2: the same operations, stopped at the first problem ==
bug 1, merge forgets to pull the separator down:
  first problem at operation #23, Delete(566)
  invariant checker: page 1 has 2 keys and 4 children
  the operation's own answer: agreed with the oracle
bug 2, a left borrow moves the key up but leaves a stale value:
  first problem at operation #16, Delete(301)
  invariant checker: contents differ at pair 1: map (24, 5110), oracle (24, 3024)
  the operation's own answer: agreed with the oracle
without a bug the same 200000 operations at t = 2 (part a): 0 mismatches, 0 invariant failures

== (d) time per operation, best of 3 interleaved rounds ==
Go's standard library has no ordered map; these are the alternatives: an OrderedMap B-tree (t = 16),
a sorted slice (slices.BinarySearchFunc, slices.Insert, slices.Delete) and a built-in map, which has no order, so no Range.
keys are 0, 2, ..., 2n-2; Put+Delete is a pair on a key that is not in the map (the size stays n); a Range returns 1,000 pairs
operation              structure             n=16         n=64        n=256       n=1000      n=10000     n=100000    n=1000000
Get (random hit)       B-tree t=16        17.5 ns      26.8 ns      34.0 ns      45.5 ns      63.0 ns      92.7 ns     159.4 ns
                       sorted slice       21.7 ns      31.4 ns      41.0 ns      50.9 ns      72.2 ns      98.5 ns     159.3 ns
                       built-in map        3.6 ns       3.6 ns       3.8 ns       3.9 ns       5.2 ns       6.7 ns      21.7 ns
Put+Delete pair        B-tree t=16        38.3 ns      63.0 ns      76.4 ns     116.2 ns     167.4 ns     221.5 ns     326.1 ns
                       sorted slice       43.8 ns      66.0 ns     114.9 ns     241.5 ns      1.52 us     21.88 us    229.57 us
                       built-in map       21.7 ns      14.1 ns      14.9 ns      16.7 ns      25.7 ns      36.0 ns      76.6 ns
Range of 1,000 pairs   B-tree t=16   n/a: n < 1000 n/a: n < 1000 n/a: n < 1000      1.45 us      1.77 us      1.98 us      4.39 us
                       sorted slice  n/a: n < 1000 n/a: n < 1000 n/a: n < 1000     464.6 ns     537.1 ns     581.2 ns     675.2 ns
                       built-in map  n/a: no order n/a: no order n/a: no order n/a: no order n/a: no order n/a: no order n/a: no order
B-tree levels                                   1            2            2            3            3            4            5

fastest, Get (random hit): built-in map at every size measured
fastest, Put+Delete pair: built-in map at every size measured
fastest, Range of 1,000 pairs: sorted slice at every size measured
Put+Delete at n=16       : sorted slice 43.8 ns, B-tree 38.3 ns: within 25%: no clear winner
Put+Delete at n=64       : sorted slice 66.0 ns, B-tree 63.0 ns: within 25%: no clear winner
Put+Delete at n=256      : sorted slice 114.9 ns, B-tree 76.4 ns: the B-tree is faster, 1.5x
Put+Delete at n=1000     : sorted slice 241.5 ns, B-tree 116.2 ns: the B-tree is faster, 2.1x
Put+Delete at n=10000    : sorted slice 1.52 us, B-tree 167.4 ns: the B-tree is faster, 9.1x
Put+Delete at n=100000   : sorted slice 21.88 us, B-tree 221.5 ns: the B-tree is faster, 98.8x
Put+Delete at n=1000000  : sorted slice 229.57 us, B-tree 326.1 ns: the B-tree is faster, 704.1x
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::Ordering;
use std::collections::{BTreeMap, HashMap};
use std::fmt::Debug;
use std::time::Instant;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the operations, shuffles and keys the Go program draws.
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

// OrderedMap<K, V>: the ordered map Go's standard library does not ship, as a
// B-tree of minimum degree t (every page but the root holds t-1 .. 2t-1 keys;
// a page has one more child than keys; all leaves are at one depth).
//
// The pages live in an arena (a Vec), a page id is its index, and a page merged
// away goes on a free list that the next split reuses. Insert is CLRS: split a
// full page on the way DOWN, so one pass suffices. Delete is CLRS too: make
// sure the child you are about to enter has at least t keys (borrow from a
// sibling or merge with one), so the key can always be removed from a page that
// can spare it. Both grow and shrink the tree only at the root.
//
// Safe Rust, no Rc, no unsafe: children are u32 ids into the arena, which is
// also what makes the free list possible (a Box tree has no ids to reuse).

struct Page<K, V> {
    keys: Vec<K>,
    vals: Vec<V>,
    kids: Vec<u32>, // page ids; empty for a leaf
}

impl<K, V> Page<K, V> {
    fn empty() -> Self {
        Page {
            keys: Vec::new(),
            vals: Vec::new(),
            kids: Vec::new(),
        }
    }
}

/// Structural work counters; node visits are counted separately in a Cell
/// because the read-only methods (`get`, `floor`, ...) take `&self`.
#[derive(Clone, Copy, Default)]
struct Stats {
    splits: usize,
    merges: usize,
    borrows_l: usize,
    borrows_r: usize,
    root_grows: usize,
    root_shrinks: usize,
}

struct OrderedMap<K, V> {
    t: usize,
    pages: Vec<Page<K, V>>,
    free: Vec<u32>, // ids of pages merged away
    root: u32,
    n: usize,
    st: Stats,
    visits: Cell<usize>,
    bug: u8, // planted bugs for part (c): 0 none, 1 a merge forgets to pull the separator down, 2 a left borrow leaves a stale value behind
}

impl<K: Ord + Clone + Debug, V: Clone> OrderedMap<K, V> {
    fn new(t: usize) -> Self {
        assert!(t >= 2, "OrderedMap: minimum degree must be at least 2");
        OrderedMap {
            t,
            pages: vec![Page::empty()],
            free: Vec::new(),
            root: 0,
            n: 0,
            st: Stats::default(),
            visits: Cell::new(0),
            bug: 0,
        }
    }

    fn len(&self) -> usize {
        self.n
    }

    fn visit(&self) {
        self.visits.set(self.visits.get() + 1);
    }

    fn alloc(&mut self) -> u32 {
        if let Some(id) = self.free.pop() {
            return id;
        }
        self.pages.push(Page::empty());
        (self.pages.len() - 1) as u32
    }

    fn release(&mut self, id: u32) {
        self.pages[id as usize] = Page::empty();
        self.free.push(id);
    }

    fn get(&self, k: &K) -> Option<&V> {
        let mut x = self.root as usize;
        loop {
            self.visit();
            let pg = &self.pages[x];
            match pg.keys.binary_search(k) {
                Ok(i) => return Some(&pg.vals[i]),
                Err(i) => {
                    if pg.kids.is_empty() {
                        return None;
                    }
                    x = pg.kids[i] as usize;
                }
            }
        }
    }

    // split_child splits the full child kids[i] of x around its median key,
    // which moves up into x. `split_off` and `pop` do what Go's reslicing does.
    fn split_child(&mut self, x: usize, i: usize) {
        let t = self.t;
        let yid = self.pages[x].kids[i] as usize;
        let zid = self.alloc() as usize;
        let y = &mut self.pages[yid];
        let zkeys = y.keys.split_off(t);
        let zvals = y.vals.split_off(t);
        let med_k = y.keys.pop().expect("a full page has keys");
        let med_v = y.vals.pop().expect("a full page has values");
        let zkids = if y.kids.is_empty() {
            Vec::new()
        } else {
            y.kids.split_off(t)
        };
        self.pages[zid] = Page {
            keys: zkeys,
            vals: zvals,
            kids: zkids,
        };
        let px = &mut self.pages[x];
        px.keys.insert(i, med_k);
        px.vals.insert(i, med_v);
        px.kids.insert(i + 1, zid as u32);
        self.st.splits += 1;
    }

    // put is an upsert: it returns the old value if the key was already there.
    // A full page is split on the way down, even when the key turns out to be an
    // update; that costs one harmless split now and then and keeps the pass single.
    fn put(&mut self, k: K, v: V) -> Option<V> {
        let full = 2 * self.t - 1;
        if self.pages[self.root as usize].keys.len() == full {
            // a full root: the tree grows at the top
            let nr = self.alloc();
            self.pages[nr as usize].kids.push(self.root);
            self.root = nr;
            self.split_child(nr as usize, 0);
            self.st.root_grows += 1;
        }
        let mut x = self.root as usize;
        loop {
            self.visit();
            match self.pages[x].keys.binary_search(&k) {
                Ok(i) => return Some(std::mem::replace(&mut self.pages[x].vals[i], v)),
                Err(mut i) => {
                    if self.pages[x].kids.is_empty() {
                        let pg = &mut self.pages[x];
                        pg.keys.insert(i, k);
                        pg.vals.insert(i, v);
                        self.n += 1;
                        return None;
                    }
                    let kid = self.pages[x].kids[i] as usize;
                    if self.pages[kid].keys.len() == full {
                        self.split_child(x, i);
                        match k.cmp(&self.pages[x].keys[i]) {
                            // the median that moved up is the key
                            Ordering::Equal => {
                                return Some(std::mem::replace(&mut self.pages[x].vals[i], v));
                            }
                            Ordering::Greater => i += 1,
                            Ordering::Less => {}
                        }
                    }
                    x = self.pages[x].kids[i] as usize;
                }
            }
        }
    }

    // merge folds kids[j], the separator keys[j] and kids[j+1] into kids[j]:
    // 2(t-1) + 1 = 2t-1 keys, exactly a full page.
    fn merge(&mut self, x: usize, j: usize) {
        let lid = self.pages[x].kids[j] as usize;
        let rid = self.pages[x].kids[j + 1];
        let sep_k = self.pages[x].keys.remove(j);
        let sep_v = self.pages[x].vals.remove(j);
        self.pages[x].kids.remove(j + 1);
        let mut r = std::mem::replace(&mut self.pages[rid as usize], Page::empty());
        let l = &mut self.pages[lid];
        if self.bug != 1 {
            l.keys.push(sep_k);
            l.vals.push(sep_v);
        }
        l.keys.append(&mut r.keys);
        l.vals.append(&mut r.vals);
        l.kids.append(&mut r.kids);
        self.free.push(rid);
        self.st.merges += 1;
    }

    // borrow_from_left: the separator comes down into the front of kids[i], the
    // left sibling's last key goes up to replace it, and (internal pages) the
    // sibling's last child crosses over.
    fn borrow_from_left(&mut self, x: usize, i: usize) {
        let cid = self.pages[x].kids[i] as usize;
        let sid = self.pages[x].kids[i - 1] as usize;
        let up_k = self.pages[sid]
            .keys
            .pop()
            .expect("the sibling can spare a key");
        let up_v = self.pages[sid]
            .vals
            .pop()
            .expect("the sibling can spare a value");
        let down_k = std::mem::replace(&mut self.pages[x].keys[i - 1], up_k);
        let down_v = std::mem::replace(&mut self.pages[x].vals[i - 1], up_v);
        if self.bug == 2 {
            // planted: the key moves up, its value stays where the parent's old one was
            self.pages[x].vals[i - 1] = down_v.clone();
        }
        let c = &mut self.pages[cid];
        c.keys.insert(0, down_k);
        c.vals.insert(0, down_v);
        if let Some(kid) = self.pages[sid].kids.pop() {
            self.pages[cid].kids.insert(0, kid);
        }
        self.st.borrows_l += 1;
    }

    fn borrow_from_right(&mut self, x: usize, i: usize) {
        let cid = self.pages[x].kids[i] as usize;
        let sid = self.pages[x].kids[i + 1] as usize;
        let up_k = self.pages[sid].keys.remove(0);
        let up_v = self.pages[sid].vals.remove(0);
        let down_k = std::mem::replace(&mut self.pages[x].keys[i], up_k);
        let down_v = std::mem::replace(&mut self.pages[x].vals[i], up_v);
        let c = &mut self.pages[cid];
        c.keys.push(down_k);
        c.vals.push(down_v);
        if !self.pages[sid].kids.is_empty() {
            let kid = self.pages[sid].kids.remove(0);
            self.pages[cid].kids.push(kid);
        }
        self.st.borrows_r += 1;
    }

    fn kid_keys(&self, x: usize, j: usize) -> usize {
        self.pages[self.pages[x].kids[j] as usize].keys.len()
    }

    // ensure makes kids[i] of x hold at least t keys, by borrowing from a sibling
    // that can spare one or, failing that, merging with a sibling. It returns the
    // index of the child that now covers the key (a merge with the LEFT sibling,
    // used only for the last child, shifts it down by one).
    fn ensure(&mut self, x: usize, i: usize) -> usize {
        let t = self.t;
        let last = self.pages[x].kids.len() - 1;
        if self.kid_keys(x, i) >= t {
            i
        } else if i > 0 && self.kid_keys(x, i - 1) >= t {
            self.borrow_from_left(x, i);
            i
        } else if i < last && self.kid_keys(x, i + 1) >= t {
            self.borrow_from_right(x, i);
            i
        } else if i < last {
            self.merge(x, i);
            i
        } else {
            self.merge(x, i - 1);
            i - 1
        }
    }

    // extreme returns the largest (want_max) or smallest key of the subtree at x.
    fn extreme(&self, mut x: u32, want_max: bool) -> (K, V) {
        loop {
            self.visit();
            let pg = &self.pages[x as usize];
            if pg.kids.is_empty() {
                let i = if want_max { pg.keys.len() - 1 } else { 0 };
                return (pg.keys[i].clone(), pg.vals[i].clone());
            }
            x = if want_max {
                pg.kids[pg.kids.len() - 1]
            } else {
                pg.kids[0]
            };
        }
    }

    // del removes k from the subtree at x. On entry x has at least t keys, or is the root.
    fn del(&mut self, x: u32, k: &K) -> Option<V> {
        self.visit();
        let xi = x as usize;
        let (i, found) = match self.pages[xi].keys.binary_search(k) {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        if self.pages[xi].kids.is_empty() {
            // a leaf: remove it, or it was never here
            if !found {
                return None;
            }
            self.pages[xi].keys.remove(i);
            return Some(self.pages[xi].vals.remove(i));
        }
        if found {
            // the key sits in an internal page
            let (left, right) = (self.pages[xi].kids[i], self.pages[xi].kids[i + 1]);
            let v;
            if self.pages[left as usize].keys.len() >= self.t {
                // a rich left child: its maximum takes the key's place
                let (pk, pv) = self.extreme(left, true);
                v = std::mem::replace(&mut self.pages[xi].vals[i], pv);
                self.pages[xi].keys[i] = pk.clone();
                self.del(left, &pk);
            } else if self.pages[right as usize].keys.len() >= self.t {
                // else a rich right child: its minimum
                let (sk, sv) = self.extreme(right, false);
                v = std::mem::replace(&mut self.pages[xi].vals[i], sv);
                self.pages[xi].keys[i] = sk.clone();
                self.del(right, &sk);
            } else {
                // both children are minimal: merge them around the key, then
                // delete it from the merged page
                v = self.pages[xi].vals[i].clone();
                self.merge(xi, i);
                self.del(left, k);
            }
            return Some(v);
        }
        // the key is not here: enter kids[i], after making sure it can spare a key
        let i = self.ensure(xi, i);
        let kid = self.pages[xi].kids[i];
        self.del(kid, k)
    }

    fn delete(&mut self, k: &K) -> Option<V> {
        let v = self.del(self.root, k);
        if v.is_some() {
            self.n -= 1;
        }
        let r = &self.pages[self.root as usize];
        if r.keys.is_empty() && !r.kids.is_empty() {
            // a merge emptied the root: the tree shrinks at the top
            let old = self.root;
            self.root = r.kids[0];
            self.release(old);
            self.st.root_shrinks += 1;
        }
        v
    }

    fn min(&self) -> Option<(&K, &V)> {
        let mut x = self.root as usize;
        loop {
            self.visit();
            let pg = &self.pages[x];
            if pg.kids.is_empty() {
                return pg.keys.first().map(|k| (k, &pg.vals[0]));
            }
            x = pg.kids[0] as usize;
        }
    }

    fn max(&self) -> Option<(&K, &V)> {
        let mut x = self.root as usize;
        loop {
            self.visit();
            let pg = &self.pages[x];
            if pg.kids.is_empty() {
                return pg.keys.last().map(|k| (k, &pg.vals[pg.vals.len() - 1]));
            }
            x = pg.kids[pg.kids.len() - 1] as usize;
        }
    }

    // floor is the greatest key <= k; ceil the least key >= k. One descent each:
    // remember the last candidate seen on the way down, the deepest one is nearest.
    fn floor(&self, k: &K) -> Option<(&K, &V)> {
        let mut x = self.root as usize;
        let mut best = None;
        loop {
            self.visit();
            let pg = &self.pages[x];
            match pg.keys.binary_search(k) {
                Ok(i) => return Some((&pg.keys[i], &pg.vals[i])),
                Err(i) => {
                    if i > 0 {
                        best = Some((&pg.keys[i - 1], &pg.vals[i - 1]));
                    }
                    if pg.kids.is_empty() {
                        return best;
                    }
                    x = pg.kids[i] as usize;
                }
            }
        }
    }

    fn ceil(&self, k: &K) -> Option<(&K, &V)> {
        let mut x = self.root as usize;
        let mut best = None;
        loop {
            self.visit();
            let pg = &self.pages[x];
            match pg.keys.binary_search(k) {
                Ok(i) => return Some((&pg.keys[i], &pg.vals[i])),
                Err(i) => {
                    if i < pg.keys.len() {
                        best = Some((&pg.keys[i], &pg.vals[i]));
                    }
                    if pg.kids.is_empty() {
                        return best;
                    }
                    x = pg.kids[i] as usize;
                }
            }
        }
    }

    // range yields the pairs with lo <= key < hi in order: one descent to lo, then
    // an in-order walk with an explicit stack of (page, next key index), the same
    // state machine as the Go iterator. Go pushes pairs into a `yield` callback
    // (iter.Seq2); Rust pulls them with `next`, so "the page to the right of the
    // key is entered only when the consumer asks for the next pair" becomes a
    // `pending` field that `next` acts on first.
    fn range<'a>(&'a self, lo: &K, hi: &'a K) -> RangeIter<'a, K, V> {
        let mut stack = Vec::new();
        let mut x = self.root;
        loop {
            self.visit();
            let pg = &self.pages[x as usize];
            let i = pg.keys.partition_point(|k| k < lo); // the first key >= lo
            stack.push((x, i));
            if pg.kids.is_empty() {
                break;
            }
            x = pg.kids[i];
        }
        RangeIter {
            m: self,
            hi,
            stack,
            pending: None,
        }
    }

    fn height(&self) -> usize {
        let mut h = 1;
        let mut x = self.root as usize;
        while let Some(&kid) = self.pages[x].kids.first() {
            h += 1;
            x = kid as usize;
        }
        h
    }
}

struct RangeIter<'a, K, V> {
    m: &'a OrderedMap<K, V>,
    hi: &'a K,
    stack: Vec<(u32, usize)>, // (page, index of the next key to yield)
    pending: Option<u32>,     // a page to enter, leftmost path first, on the next call
}

impl<'a, K: Ord, V> Iterator for RangeIter<'a, K, V> {
    type Item = (&'a K, &'a V);

    fn next(&mut self) -> Option<Self::Item> {
        if let Some(mut x) = self.pending.take() {
            loop {
                self.m.visits.set(self.m.visits.get() + 1);
                self.stack.push((x, 0));
                match self.m.pages[x as usize].kids.first() {
                    Some(&kid) => x = kid,
                    None => break,
                }
            }
        }
        loop {
            let &(x, i) = self.stack.last()?;
            let pg = &self.m.pages[x as usize];
            if i >= pg.keys.len() {
                self.stack.pop();
                continue;
            }
            let k = &pg.keys[i];
            if k >= self.hi {
                self.stack.clear(); // done: the iterator stays finished
                return None;
            }
            if let Some(top) = self.stack.last_mut() {
                top.1 = i + 1;
            }
            if !pg.kids.is_empty() {
                self.pending = Some(pg.kids[i + 1]); // the subtree to the right of the key comes next
            }
            return Some((k, &pg.vals[i]));
        }
    }
}

// maxLevels is the theorem: n keys fit in at most floor(log_t((n+1)/2)) + 1 levels.
fn max_levels(n: usize, t: usize) -> usize {
    let mut e = 0;
    let mut p = 1;
    while n > 0 && p * t * 2 <= n + 1 {
        e += 1;
        p *= t;
    }
    e + 1
}

// What `check` walks with: which pages were reached, the leaf depth seen so
// far, and the contents in key order.
struct Walk<K, V> {
    reached: Vec<bool>,
    leaf_depth: Option<usize>,
    keys: Vec<K>,
    vals: Vec<V>,
}

impl<K: Ord + Clone + Debug, V: Clone> OrderedMap<K, V> {
    fn walk(
        &self,
        x: u32,
        depth: usize,
        bounds: (Option<&K>, Option<&K>),
        w: &mut Walk<K, V>,
    ) -> Result<(), String> {
        let (lo, hi) = bounds;
        let Some(pg) = self.pages.get(x as usize) else {
            return Err(format!("page id {x} is outside the arena"));
        };
        if w.reached[x as usize] {
            return Err(format!("page {x} is reachable twice"));
        }
        w.reached[x as usize] = true;
        if pg.keys.len() != pg.vals.len() {
            return Err(format!(
                "page {x} has {} keys but {} values",
                pg.keys.len(),
                pg.vals.len()
            ));
        }
        let (lowest, highest) = (self.t - 1, 2 * self.t - 1);
        if pg.keys.len() > highest || (x != self.root && pg.keys.len() < lowest) {
            return Err(format!(
                "page {x} holds {} keys (allowed {lowest}..{highest})",
                pg.keys.len()
            ));
        }
        for (i, k) in pg.keys.iter().enumerate() {
            if lo.is_some_and(|l| k <= l)
                || hi.is_some_and(|h| k >= h)
                || (i > 0 && k <= &pg.keys[i - 1])
            {
                return Err(format!(
                    "page {x}: key {k:?} is out of order or outside its separators"
                ));
            }
        }
        if pg.kids.is_empty() {
            if let Some(d) = w.leaf_depth.filter(|&d| d != depth) {
                return Err(format!("leaves at depths {d} and {depth}"));
            }
            w.leaf_depth = Some(depth);
            w.keys.extend(pg.keys.iter().cloned());
            w.vals.extend(pg.vals.iter().cloned());
            return Ok(());
        }
        if pg.kids.len() != pg.keys.len() + 1 {
            return Err(format!(
                "page {x} has {} keys and {} children",
                pg.keys.len(),
                pg.kids.len()
            ));
        }
        if pg.keys.is_empty() {
            return Err(format!("internal page {x} has no keys"));
        }
        for (i, &kid) in pg.kids.iter().enumerate() {
            let l = if i > 0 { Some(&pg.keys[i - 1]) } else { lo };
            let h = pg.keys.get(i).or(hi);
            self.walk(kid, depth + 1, (l, h), w)?;
            if i < pg.keys.len() {
                w.keys.push(pg.keys[i].clone());
                w.vals.push(pg.vals[i].clone());
            }
        }
        Ok(())
    }

    // check verifies every invariant and returns the contents in key order. It
    // returns an error, never panics, so a planted bug can be reported.
    fn check(&self) -> Result<(Vec<K>, Vec<V>), String> {
        let mut w = Walk {
            reached: vec![false; self.pages.len()],
            leaf_depth: None,
            keys: Vec::new(),
            vals: Vec::new(),
        };
        self.walk(self.root, 0, (None, None), &mut w)?;
        if w.keys.len() != self.n {
            return Err(format!(
                "{} keys are reachable but Len() is {}",
                w.keys.len(),
                self.n
            ));
        }
        let mut on_free = vec![false; self.pages.len()];
        for &id in &self.free {
            if w.reached[id as usize] || on_free[id as usize] {
                return Err(format!("free page {id} is reachable or listed twice"));
            }
            on_free[id as usize] = true;
        }
        if let Some(id) = (0..self.pages.len()).find(|&id| !w.reached[id] && !on_free[id]) {
            return Err(format!("page {id} is neither reachable nor free (a leak)"));
        }
        let h = self.height();
        if h > max_levels(self.n, self.t) {
            return Err(format!(
                "{h} levels for {} keys exceeds the bound {}",
                self.n,
                max_levels(self.n, self.t)
            ));
        }
        Ok((w.keys, w.vals))
    }
}

// ---------------------------------------------------------------------------
// The proof: a sorted Vec of pairs is the oracle (and, in Rust, BTreeMap is a
// third opinion).

type Entry = (u64, u64);

fn find(o: &[Entry], k: u64) -> (usize, bool) {
    let i = o.partition_point(|e| e.0 < k);
    (i, i < o.len() && o[i].0 == k)
}

// Res is the answer of one operation, so any two answers compare with ==.
#[derive(Clone, Copy, Default, PartialEq, Debug)]
struct Res {
    k: u64,
    v: u64,
    ok: bool,
}

impl Res {
    fn val(v: Option<u64>) -> Res {
        Res {
            k: 0,
            v: v.unwrap_or(0),
            ok: v.is_some(),
        }
    }

    fn pair(p: Option<Entry>) -> Res {
        p.map_or(Res::default(), |(k, v)| Res { k, v, ok: true })
    }
}

#[derive(Default)]
struct Report {
    ops: usize,
    put_new: usize,
    put_old: usize,
    del_hit: usize,
    del_miss: usize,
    ranges: usize,
    range_keys: usize,
    max_keys: usize,
    max_levels: usize,
    get_visits: usize,
    gets: usize,
    mismatches: usize,
    inv_failures: usize,
    first_op: usize,
    first_desc: String,
    first_result: String,
    first_check: String,
    st: Stats,
}

struct Harness {
    m: OrderedMap<u64, u64>,
    o: Vec<Entry>,
    bt: BTreeMap<u64, u64>, // Rust only: the standard library's B-tree, as a third opinion
    bt_mismatches: usize,
    rng: Pcg,
    r: Report,
    // the problems of the operation being run
    desc: String,
    result_msg: String,
}

impl Harness {
    fn new(t: usize, rng: Pcg) -> Self {
        Harness {
            m: OrderedMap::new(t),
            o: Vec::new(),
            bt: BTreeMap::new(),
            bt_mismatches: 0,
            rng,
            r: Report::default(),
            desc: String::new(),
            result_msg: String::new(),
        }
    }

    fn mismatch(&mut self, msg: String) {
        self.r.mismatches += 1;
        if self.result_msg.is_empty() {
            self.result_msg = msg;
        }
    }

    fn expect(&mut self, got: Res, want: Res) {
        if got != want {
            self.mismatch(format!(
                "{} returned {got:?}, oracle says {want:?}",
                self.desc
            ));
        }
    }

    // `theirs` is BTreeMap's answer to the same operation.
    fn expect3(&mut self, got: Res, theirs: Res, want: Res) {
        self.expect(got, want);
        if theirs != want {
            self.bt_mismatches += 1;
        }
    }

    // step runs ONE random operation on the map and on the oracle and compares
    // their answers. op is the 1-based operation number; values are op*1000+key,
    // so a stale value (an update that did not take, a value left behind by a
    // borrow) is told apart from a fresh one.
    fn step(&mut self, op: usize) {
        let kind = self.rng.int_n(100);
        let k = self.rng.int_n(600);
        self.result_msg.clear();
        let (mut i, found) = find(&self.o, k); // where k is, or would go
        let at = found.then(|| self.o[i].1);
        match kind {
            0..40 => {
                self.desc = format!("Put({k})");
                let v = op as u64 * 1000 + k;
                if found {
                    self.o[i].1 = v;
                    self.r.put_old += 1;
                } else {
                    self.o.insert(i, (k, v));
                    self.r.put_new += 1;
                }
                let (got, theirs) = (self.m.put(k, v), self.bt.insert(k, v));
                self.expect3(Res::val(got), Res::val(theirs), Res::val(at));
            }
            40..75 => {
                self.desc = format!("Delete({k})");
                if found {
                    self.o.remove(i);
                    self.r.del_hit += 1;
                } else {
                    self.r.del_miss += 1;
                }
                let (got, theirs) = (self.m.delete(&k), self.bt.remove(&k));
                self.expect3(Res::val(got), Res::val(theirs), Res::val(at));
            }
            75..85 => {
                self.desc = format!("Get({k})");
                let before = self.m.visits.get();
                let got = self.m.get(&k).copied();
                self.r.get_visits += self.m.visits.get() - before;
                self.r.gets += 1;
                let theirs = self.bt.get(&k).copied();
                self.expect3(Res::val(got), Res::val(theirs), Res::val(at));
            }
            85..90 => {
                self.desc = format!("Floor({k})");
                if found {
                    i += 1; // an exact hit is its own floor
                }
                let want = Res::pair(i.checked_sub(1).map(|j| self.o[j]));
                let got = self.m.floor(&k).map(|(a, b)| (*a, *b));
                // floor = range(..=k).next_back(): the idiom the lesson names
                let theirs = self.bt.range(..=k).next_back().map(|(a, b)| (*a, *b));
                self.expect3(Res::pair(got), Res::pair(theirs), want);
            }
            90..95 => {
                self.desc = format!("Ceil({k})");
                let want = Res::pair(self.o.get(i).copied());
                let got = self.m.ceil(&k).map(|(a, b)| (*a, *b));
                // ceil = range(k..).next()
                let theirs = self.bt.range(k..).next().map(|(a, b)| (*a, *b));
                self.expect3(Res::pair(got), Res::pair(theirs), want);
            }
            _ => {
                let hi = k + self.rng.int_n(120);
                self.desc = format!("Range({k}, {hi})");
                let (j, _) = find(&self.o, hi);
                let want: Vec<Entry> = self.o[i..j].to_vec(); // a copy, so `self` can be borrowed mutably below
                self.r.ranges += 1;
                self.r.range_keys += want.len();
                let got: Vec<Entry> = self.m.range(&k, &hi).map(|(a, b)| (*a, *b)).collect();
                if got != want {
                    let msg = format!(
                        "{} yielded {} pairs, oracle has {} (or they differ)",
                        self.desc,
                        got.len(),
                        want.len()
                    );
                    self.mismatch(msg);
                }
                let theirs: Vec<Entry> = self.bt.range(k..hi).map(|(a, b)| (*a, *b)).collect();
                if theirs != want {
                    self.bt_mismatches += 1;
                }
            }
        }
    }

    // verify runs the invariant checker and compares the whole contents, Len, Min
    // and Max with the oracle. It returns the checker's message ("" when it passes).
    fn verify(&mut self) -> String {
        let mut msg = String::new();
        match self.m.check() {
            Err(e) => msg = e,
            Ok((keys, _)) if keys.len() != self.o.len() => {
                msg = format!(
                    "the map holds {} keys, the oracle {}",
                    keys.len(),
                    self.o.len()
                );
            }
            Ok((keys, vals)) => {
                for (i, e) in self.o.iter().enumerate() {
                    if (keys[i], vals[i]) != *e {
                        msg = format!(
                            "contents differ at pair {i}: map ({}, {}), oracle ({}, {})",
                            keys[i], vals[i], e.0, e.1
                        );
                        break;
                    }
                }
            }
        }
        if !msg.is_empty() {
            self.r.inv_failures += 1;
        }
        if self.m.len() != self.o.len() {
            self.mismatch(format!(
                "Len() is {}, oracle has {}",
                self.m.len(),
                self.o.len()
            ));
        }
        self.desc = "Min()".to_string();
        let (first, last) = (
            Res::pair(self.o.first().copied()),
            Res::pair(self.o.last().copied()),
        );
        self.expect(Res::pair(self.m.min().map(|(a, b)| (*a, *b))), first);
        self.desc = "Max()".to_string();
        self.expect(Res::pair(self.m.max().map(|(a, b)| (*a, *b))), last);
        // Rust only: BTreeMap must hold exactly the oracle's contents too
        if !self
            .bt
            .iter()
            .map(|(a, b)| (*a, *b))
            .eq(self.o.iter().copied())
        {
            self.bt_mismatches += 1;
        }
        self.r.max_keys = self.r.max_keys.max(self.o.len());
        self.r.max_levels = self.r.max_levels.max(self.m.height());
        msg
    }
}

// run_ops drives `ops` random operations from PCG(1, 2) (the same stream for
// every t) and checks after EVERY one. With stop_at_first it returns at the
// first problem and remembers which operation it was.
fn run_ops(t: usize, ops: usize, stop_at_first: bool, bug: u8) -> (Report, usize) {
    let mut h = Harness::new(t, Pcg::new(1, 2));
    h.m.bug = bug;
    for op in 1..=ops {
        h.step(op);
        h.r.ops = op;
        let desc = h.desc.clone();
        let msg = h.verify();
        if (!msg.is_empty() || !h.result_msg.is_empty()) && h.r.first_op == 0 {
            h.r.first_op = op;
            h.r.first_desc = desc;
            h.r.first_result = h.result_msg.clone();
            h.r.first_check = msg;
        }
        if stop_at_first && h.r.first_op != 0 {
            break;
        }
    }
    h.r.st = h.m.st;
    (h.r, h.bt_mismatches)
}

fn shuffled(rng: &mut Pcg, n: u64) -> Vec<u64> {
    let mut a: Vec<u64> = (0..n).collect();
    for i in (1..n as usize).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
    a
}

// grow_shrink: up to 5,000 keys in random order, down to 0 in another random
// order, up again in sorted order; the checker runs after every operation, and
// the empty tree gets every query. It returns the levels at 5,000 keys, at 0
// keys and after the regrowth, and the harness for its counters.
fn grow_shrink(t: usize) -> ([usize; 3], Harness) {
    const N: u64 = 5000;
    let mut rng = Pcg::new(7, 8);
    let mut h = Harness::new(t, Pcg::new(1, 2));
    let mut levels = [0; 3];
    let put = |h: &mut Harness, k: u64| {
        let (i, _) = find(&h.o, k);
        h.o.insert(i, (k, k * 7 + 1));
        h.m.put(k, k * 7 + 1);
        h.verify();
    };
    for k in shuffled(&mut rng, N) {
        put(&mut h, k);
    }
    levels[0] = h.m.height();
    for k in shuffled(&mut rng, N) {
        let (i, _) = find(&h.o, k);
        let want = h.o.remove(i).1;
        let got = h.m.delete(&k);
        h.desc = format!("Delete({k})");
        h.expect(Res::val(got), Res::val(Some(want)));
        h.verify();
    }
    levels[1] = h.m.height();
    // the empty tree answers every query with "nothing", and deleting from it changes nothing
    let answered = h.m.get(&3).is_some()
        || h.m.floor(&3).is_some()
        || h.m.ceil(&3).is_some()
        || h.m.delete(&3).is_some()
        || h.m.range(&0, &N).next().is_some();
    if answered {
        h.mismatch("the empty map answered a query with a key".to_string());
    }
    h.verify();
    for k in 0..N {
        put(&mut h, k);
    }
    levels[2] = h.m.height();
    (levels, h)
}

// ---------------------------------------------------------------------------
// The timing: the alternatives Rust has (BTreeMap is the stdlib row Go lacks).

// Inputs are the probe keys, generated once per size.
struct Inputs {
    hits: Vec<u64>,
    fresh: Vec<u64>,
    starts: Vec<u64>,
}

const SPAN: usize = 1_000; // pairs in one Range

// Bencher is one structure under test. `run` performs `op` once over the
// inputs (0: get every hit, 1: put+delete every fresh key, 2: range from every
// start) and returns a checksum, or None when the structure cannot do that
// operation.
trait Bencher {
    fn run(&mut self, op: usize, inp: &Inputs) -> Option<u64>;
}

fn supported(op: usize, inp: &Inputs) -> bool {
    op < 2 || !inp.starts.is_empty()
}

struct TreeBench(OrderedMap<u64, u64>);

impl Bencher for TreeBench {
    fn run(&mut self, op: usize, inp: &Inputs) -> Option<u64> {
        let mut sum = 0;
        match op {
            0 => {
                for k in &inp.hits {
                    sum += self.0.get(k).copied().unwrap_or(0);
                }
            }
            1 => {
                for &k in &inp.fresh {
                    self.0.put(k, k);
                    self.0.delete(&k);
                }
            }
            _ => {
                for &lo in &inp.starts {
                    let hi = lo + 2 * SPAN as u64;
                    let mut c = 0;
                    for (_, v) in self.0.range(&lo, &hi) {
                        sum += v;
                        c += 1;
                    }
                    assert_eq!(c, SPAN, "range returned the wrong count");
                }
            }
        }
        supported(op, inp).then_some(sum)
    }
}

struct StdBench(BTreeMap<u64, u64>);

impl Bencher for StdBench {
    fn run(&mut self, op: usize, inp: &Inputs) -> Option<u64> {
        let mut sum = 0;
        match op {
            0 => {
                for k in &inp.hits {
                    sum += self.0.get(k).copied().unwrap_or(0);
                }
            }
            1 => {
                for &k in &inp.fresh {
                    self.0.insert(k, k);
                    self.0.remove(&k);
                }
            }
            _ => {
                for &lo in &inp.starts {
                    let mut c = 0;
                    for (_, v) in self.0.range(lo..lo + 2 * SPAN as u64) {
                        sum += v;
                        c += 1;
                    }
                    assert_eq!(c, SPAN, "range returned the wrong count");
                }
            }
        }
        supported(op, inp).then_some(sum)
    }
}

struct VecBench(Vec<Entry>);

impl Bencher for VecBench {
    fn run(&mut self, op: usize, inp: &Inputs) -> Option<u64> {
        let s = &mut self.0;
        let mut sum = 0;
        match op {
            0 => {
                for &k in &inp.hits {
                    sum += s[s.partition_point(|e| e.0 < k)].1;
                }
            }
            1 => {
                for &k in &inp.fresh {
                    let i = s.partition_point(|e| e.0 < k);
                    s.insert(i, (k, k));
                    let i = s.partition_point(|e| e.0 < k);
                    s.remove(i);
                }
            }
            _ => {
                for &lo in &inp.starts {
                    let mut c = 0;
                    let mut i = s.partition_point(|e| e.0 < lo);
                    while i < s.len() && s[i].0 < lo + 2 * SPAN as u64 {
                        sum += s[i].1;
                        c += 1;
                        i += 1;
                    }
                    assert_eq!(c, SPAN, "range returned the wrong count");
                }
            }
        }
        supported(op, inp).then_some(sum)
    }
}

struct HashBench(HashMap<u64, u64>);

impl Bencher for HashBench {
    fn run(&mut self, op: usize, inp: &Inputs) -> Option<u64> {
        let mut sum = 0;
        match op {
            0 => {
                for k in &inp.hits {
                    sum += self.0.get(k).copied().unwrap_or(0);
                }
            }
            1 => {
                for &k in &inp.fresh {
                    self.0.insert(k, k);
                    self.0.remove(&k);
                }
            }
            _ => {}
        }
        (op < 2).then_some(sum) // a map has no order, so no Range
    }
}

// bench returns ns per op for [structure][operation] at size n, each the best
// of 3 interleaved rounds (-1: not measured), and the levels of our tree. Keys
// are the even numbers 0..2n-2, so every odd number is a fresh key. The
// structures are built once, the trees from shuffled input (so the sorted Vec
// never pays for its own inserts).
fn bench(n: usize) -> ([[f64; 3]; 4], usize) {
    const GETS: usize = 200_000;
    const PAIRS: usize = 2_000;
    const SCANS: usize = 200;
    let mut rng = Pcg::new(9, 10);
    let mut order: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        order.swap(i, j);
    }
    let mut tree = OrderedMap::<u64, u64>::new(16);
    let mut std_map = BTreeMap::new();
    let mut hash = HashMap::with_capacity(n);
    for &i in &order {
        tree.put(2 * i, 2 * i);
        std_map.insert(2 * i, 2 * i);
        hash.insert(2 * i, 2 * i);
    }
    let levels = tree.height();
    let sorted: Vec<Entry> = (0..n as u64).map(|i| (2 * i, 2 * i)).collect();
    let hits = (0..GETS).map(|_| 2 * rng.int_n(n as u64)).collect();
    let fresh = (0..PAIRS).map(|_| 2 * rng.int_n(n as u64) + 1).collect();
    let starts = if n >= SPAN {
        // [start, start+2*SPAN) holds exactly SPAN keys
        let top = (n - SPAN + 1) as u64;
        (0..SCANS).map(|_| 2 * rng.int_n(top)).collect()
    } else {
        Vec::new() // a Range of 1,000 pairs needs 1,000 keys
    };
    let inp = Inputs {
        hits,
        fresh,
        starts,
    };
    let mut structs: [Box<dyn Bencher>; 4] = [
        Box::new(TreeBench(tree)),
        Box::new(StdBench(std_map)),
        Box::new(VecBench(sorted)),
        Box::new(HashBench(hash)),
    ];
    let per = [GETS, PAIRS, SCANS];
    let mut best = [[-1.0f64; 3]; 4];
    for _round in 0..3 {
        for (s, b) in structs.iter_mut().enumerate() {
            for (op, &count) in per.iter().enumerate() {
                let start = Instant::now();
                let sum = b.run(op, &inp);
                let ns = start.elapsed().as_nanos() as f64 / count as f64;
                if let Some(sum) = sum {
                    std::hint::black_box(sum);
                    if best[s][op] < 0.0 || ns < best[s][op] {
                        best[s][op] = ns;
                    }
                }
            }
        }
    }
    (best, levels)
}

fn fmt_ns(x: f64) -> String {
    if x >= 1e6 {
        format!("{:.2} ms", x / 1e6)
    } else if x >= 1e3 {
        format!("{:.2} us", x / 1e3)
    } else {
        format!("{x:.1} ns")
    }
}

const NAMES: [&str; 4] = [
    "B-tree t=16 (ours)",
    "BTreeMap (std)",
    "sorted Vec",
    "HashMap (std)",
];
const OPS: [&str; 3] = [
    "Get (random hit)",
    "Put+Delete pair",
    "Range of 1,000 pairs",
];

fn main() {
    println!("OrderedMap<K, V>: a B-tree with get, put, delete, min, max, floor, ceil, range, len");
    println!(
        "Rust's standard library has BTreeMap (a B-tree: B = 6, at most 11 keys per node, read in its source, node.rs); Go's has no ordered map."
    );
    println!(
        "The proof is against a sorted Vec of pairs and, in this program only, against BTreeMap too."
    );

    // ---- (a) the proof ----
    const OPS_PER_T: usize = 200_000;
    let degrees = [2usize, 3, 6, 16];
    println!(
        "\n== (a) proof: {OPS_PER_T} random operations per t, checker and oracle comparison after EVERY operation =="
    );
    println!(
        "operations drawn from PCG(1, 2), the same stream for every t: Put 40%, Delete 35%, Get 10%, Floor 5%, Ceil 5%, Range 5%, keys 0..599"
    );
    println!(
        "values are op*1000+key; the checker: keys per page t-1..2t-1 (root exempt), equal leaf depth, keys sorted and inside their separators,"
    );
    println!(
        "children = keys+1, Len() = reachable keys, every page reachable or on the free list, levels <= floor(log_t((n+1)/2))+1"
    );
    let runs: Vec<(Report, usize)> = degrees
        .iter()
        .map(|&t| run_ops(t, OPS_PER_T, false, 0))
        .collect();
    let r = &runs[0].0;
    println!(
        "workload: Put {} new keys + {} replacing; Delete {} hits + {} absent; {} Range scans returning {} pairs in all; the map held at most {} keys",
        r.put_new, r.put_old, r.del_hit, r.del_miss, r.ranges, r.range_keys, r.max_keys
    );
    let h = [
        "t",
        "splits",
        "merges",
        "borrowsL",
        "borrowsR",
        "root +/-",
        "max levels",
        "pages/Get",
        "mismatches",
        "inv. fails",
    ];
    println!(
        "{:>3} {:>8} {:>8} {:>8} {:>8} {:>9} {:>10} {:>10} {:>11} {:>11}",
        h[0], h[1], h[2], h[3], h[4], h[5], h[6], h[7], h[8], h[9]
    );
    let mut total = 0;
    for (&t, (r, _)) in degrees.iter().zip(&runs) {
        let s = &r.st;
        println!(
            "{:>3} {:>8} {:>8} {:>8} {:>8} {:>4}/{:<4} {:>10} {:>10.2} {:>11} {:>11}",
            t,
            s.splits,
            s.merges,
            s.borrows_l,
            s.borrows_r,
            s.root_grows,
            s.root_shrinks,
            r.max_levels,
            r.get_visits as f64 / r.gets as f64,
            r.mismatches,
            r.inv_failures
        );
        total += r.ops;
    }
    println!("{total} operations checked in all");
    let bt_bad: usize = runs.iter().map(|(_, b)| b).sum();
    println!(
        "three-way agreement, Rust only: our tree == sorted-Vec oracle == BTreeMap on every Put, Delete, Get, Floor (range(..=k).next_back()), Ceil (range(k..).next()), Range (range(k..hi)) and on the full contents after every operation: BTreeMap disagreed with the oracle {bt_bad} times"
    );

    // ---- (b) grow, shrink, grow ----
    println!(
        "\n== (b) grow to 5,000 keys, shrink to 0, grow again (checker after every operation; the empty map gets every query) =="
    );
    let h = [
        "t",
        "levels@5k",
        "levels@0",
        "levels@5k'",
        "max levels",
        "root +",
        "root -",
        "splits",
        "merges",
        "borrows",
    ];
    println!(
        "{:>3} {:>9} {:>9} {:>10} {:>10} {:>7} {:>7} {:>7} {:>8} {:>11}",
        h[0], h[1], h[2], h[3], h[4], h[5], h[6], h[7], h[8], h[9]
    );
    for &t in &degrees {
        let (lv, h) = grow_shrink(t);
        let s = &h.m.st;
        println!(
            "{:>3} {:>9} {:>9} {:>10} {:>10} {:>7} {:>7} {:>7} {:>8} {:>11}",
            t,
            lv[0],
            lv[1],
            lv[2],
            h.r.max_levels,
            s.root_grows,
            s.root_shrinks,
            s.splits,
            s.merges,
            s.borrows_l + s.borrows_r
        );
        println!(
            "    levels = 1 + root growths - root shrinks: {}; empty map has {} level; mismatches {}, invariant failures {}",
            1 + s.root_grows - s.root_shrinks == lv[2],
            lv[1],
            h.r.mismatches,
            h.r.inv_failures
        );
    }

    // ---- (c) planted bugs ----
    println!(
        "\n== (c) planted bugs at t = 2: the same operations, stopped at the first problem =="
    );
    let bugs = [
        "",
        "merge forgets to pull the separator down",
        "a left borrow moves the key up but leaves a stale value",
    ];
    for (bug, what) in bugs.iter().enumerate().skip(1) {
        println!("bug {bug}, {what}:");
        let (b, _) = run_ops(2, OPS_PER_T, true, bug as u8);
        if b.first_op == 0 {
            println!("  the checker did NOT notice the bug in {OPS_PER_T} operations");
            continue;
        }
        println!(
            "  first problem at operation #{}, {}",
            b.first_op, b.first_desc
        );
        if b.first_check.is_empty() {
            println!("  invariant checker: passed");
        } else {
            println!("  invariant checker: {}", b.first_check);
        }
        if b.first_result.is_empty() {
            println!("  the operation's own answer: agreed with the oracle");
        } else {
            println!("  the operation's own answer: {}", b.first_result);
        }
    }
    println!(
        "without a bug the same {OPS_PER_T} operations at t = 2 (part a): {} mismatches, {} invariant failures",
        runs[0].0.mismatches, runs[0].0.inv_failures
    );

    // ---- (d) the clock ----
    println!("\n== (d) time per operation, best of 3 interleaved rounds ==");
    println!(
        "Go's standard library has no ordered map; Rust's does, so here BTreeMap is a row next to our tree, a sorted Vec and a HashMap"
    );
    println!(
        "(std's default hasher, SipHash 1-3), which has no order, so no Range. Our tree and the sorted Vec are the rows Go's table has too."
    );
    println!(
        "keys are 0, 2, ..., 2n-2; Put+Delete is a pair on a key that is not in the map (the size stays n); a Range returns 1,000 pairs"
    );
    let sizes = [16usize, 64, 256, 1_000, 10_000, 100_000, 1_000_000];
    let (mut results, mut levels) = (Vec::new(), Vec::new());
    for &n in &sizes {
        let (res, lv) = bench(n);
        results.push(res);
        levels.push(lv);
    }
    print!("{:<22} {:<19}", "operation", "structure");
    for n in &sizes {
        print!(" {:>12}", format!("n={n}"));
    }
    println!();
    for (op, op_name) in OPS.iter().enumerate() {
        for (s, name) in NAMES.iter().enumerate() {
            let label = if s == 0 { *op_name } else { "" };
            print!("{label:<22} {name:<19}");
            for res in &results {
                if op == 2 && s == 3 {
                    print!(" {:>12}", "n/a: no order");
                } else if res[s][op] < 0.0 {
                    print!(" {:>12}", "n/a: n < 1000");
                } else {
                    print!(" {:>12}", fmt_ns(res[s][op]));
                }
            }
            println!();
        }
    }
    print!("{:<42}", "B-tree levels (ours)");
    for lv in &levels {
        print!(" {lv:>12}");
    }
    println!("\n");
    for (op, op_name) in OPS.iter().enumerate() {
        let mut at = Vec::new(); // the fastest structure at each size where the operation was measured
        let (mut first, mut same) = (None, true);
        for (res, &n) in results.iter().zip(&sizes) {
            let mut w: Option<usize> = None;
            for s in 0..4 {
                if res[s][op] >= 0.0 && w.is_none_or(|w| res[s][op] < res[w][op]) {
                    w = Some(s);
                }
            }
            let Some(w) = w else { continue };
            first = first.or(Some(w));
            same = same && first == Some(w);
            at.push(format!("n={n} {}", NAMES[w]));
        }
        if same {
            println!(
                "fastest, {op_name}: {} at every size measured",
                NAMES[first.expect("some size was measured")]
            );
        } else {
            println!("fastest, {op_name}: {}", at.join("; "));
        }
    }
    for (res, &n) in results.iter().zip(&sizes) {
        let q = res[2][1] / res[0][1]; // sorted Vec time over our B-tree time
        let verdict = if q > 1.25 {
            format!("our B-tree is faster, {q:.1}x")
        } else if q < 0.8 {
            format!("the sorted Vec is faster, {:.1}x", 1.0 / q)
        } else {
            "within 25%: no clear winner".to_string()
        };
        println!(
            "Put+Delete at n={n:<9}: sorted Vec {}, our B-tree {}: {verdict}",
            fmt_ns(res[2][1]),
            fmt_ns(res[0][1])
        );
    }
    // Rust only: ours against std's B-tree
    println!("our B-tree against BTreeMap, time(ours) / time(BTreeMap), per size:");
    for (op, op_name) in OPS.iter().enumerate() {
        print!("  {op_name:<22}");
        for res in &results {
            if res[0][op] < 0.0 {
                print!(" {:>8}", "n/a");
            } else {
                print!(" {:>7.2}x", res[0][op] / res[1][op]);
            }
        }
        println!();
    }
}
```

**Rust sample output:**

```
OrderedMap<K, V>: a B-tree with get, put, delete, min, max, floor, ceil, range, len
Rust's standard library has BTreeMap (a B-tree: B = 6, at most 11 keys per node, read in its source, node.rs); Go's has no ordered map.
The proof is against a sorted Vec of pairs and, in this program only, against BTreeMap too.

== (a) proof: 200000 random operations per t, checker and oracle comparison after EVERY operation ==
operations drawn from PCG(1, 2), the same stream for every t: Put 40%, Delete 35%, Get 10%, Floor 5%, Ceil 5%, Range 5%, keys 0..599
values are op*1000+key; the checker: keys per page t-1..2t-1 (root exempt), equal leaf depth, keys sorted and inside their separators,
children = keys+1, Len() = reachable keys, every page reachable or on the free list, levels <= floor(log_t((n+1)/2))+1
workload: Put 37377 new keys + 42769 replacing; Delete 37051 hits + 32874 absent; 9786 Range scans returning 285022 pairs in all; the map held at most 366 keys
  t   splits   merges borrowsL borrowsR  root +/- max levels  pages/Get  mismatches  inv. fails
  2    48760    48559    39840    33915  998/993           7       5.69           0           0
  3    17365    17255    20100    13528 6358/6355          5       3.91           0           0
  6     2578     2533     9622     4549    5/3             3       2.92           0           0
 16      367      352     4136     1000    2/1             2       1.98           0           0
800000 operations checked in all
three-way agreement, Rust only: our tree == sorted-Vec oracle == BTreeMap on every Put, Delete, Get, Floor (range(..=k).next_back()), Ceil (range(k..).next()), Range (range(k..hi)) and on the full contents after every operation: BTreeMap disagreed with the oracle 0 times

== (b) grow to 5,000 keys, shrink to 0, grow again (checker after every operation; the empty map gets every query) ==
  t levels@5k  levels@0 levels@5k' max levels  root +  root -  splits   merges     borrows
  2        10         1         12         12      20       9    7827     2846        4663
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0
  3         6         1          8          8      12       5    4048     1560        3185
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0
  6         4         1          5          5       7       3    1667      674        2161
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0
 16         3         1          3          3       4       2     562      233        1538
    levels = 1 + root growths - root shrinks: true; empty map has 1 level; mismatches 0, invariant failures 0

== (c) planted bugs at t = 2: the same operations, stopped at the first problem ==
bug 1, merge forgets to pull the separator down:
  first problem at operation #23, Delete(566)
  invariant checker: page 1 has 2 keys and 4 children
  the operation's own answer: agreed with the oracle
bug 2, a left borrow moves the key up but leaves a stale value:
  first problem at operation #16, Delete(301)
  invariant checker: contents differ at pair 1: map (24, 5110), oracle (24, 3024)
  the operation's own answer: agreed with the oracle
without a bug the same 200000 operations at t = 2 (part a): 0 mismatches, 0 invariant failures

== (d) time per operation, best of 3 interleaved rounds ==
Go's standard library has no ordered map; Rust's does, so here BTreeMap is a row next to our tree, a sorted Vec and a HashMap
(std's default hasher, SipHash 1-3), which has no order, so no Range. Our tree and the sorted Vec are the rows Go's table has too.
keys are 0, 2, ..., 2n-2; Put+Delete is a pair on a key that is not in the map (the size stays n); a Range returns 1,000 pairs
operation              structure                   n=16         n=64        n=256       n=1000      n=10000     n=100000    n=1000000
Get (random hit)       B-tree t=16 (ours)        2.3 ns       4.9 ns       8.7 ns      10.8 ns      18.3 ns      33.1 ns      85.6 ns
                       BTreeMap (std)           10.8 ns      14.6 ns      21.2 ns      28.1 ns      42.7 ns      64.5 ns     145.3 ns
                       sorted Vec                2.0 ns       3.0 ns       4.4 ns       5.9 ns      10.6 ns      19.0 ns      57.4 ns
                       HashMap (std)             3.9 ns       4.0 ns       4.0 ns       4.0 ns       4.4 ns       6.5 ns      19.0 ns
Put+Delete pair        B-tree t=16 (ours)       34.1 ns      48.9 ns      56.2 ns      70.6 ns      92.9 ns     141.0 ns     257.8 ns
                       BTreeMap (std)           32.2 ns      37.1 ns      60.3 ns      84.1 ns     111.0 ns     155.6 ns     260.2 ns
                       sorted Vec               32.5 ns      48.2 ns      84.5 ns     193.7 ns      1.45 us     21.78 us    226.73 us
                       HashMap (std)            17.4 ns      18.6 ns      17.5 ns      16.6 ns      19.6 ns      28.3 ns      18.5 ns
Range of 1,000 pairs   B-tree t=16 (ours)  n/a: n < 1000 n/a: n < 1000 n/a: n < 1000      2.53 us      2.60 us      2.67 us      6.29 us
                       BTreeMap (std)      n/a: n < 1000 n/a: n < 1000 n/a: n < 1000     881.5 ns      1.14 us      2.02 us      8.24 us
                       sorted Vec          n/a: n < 1000 n/a: n < 1000 n/a: n < 1000     306.5 ns     335.2 ns     407.7 ns     545.2 ns
                       HashMap (std)       n/a: no order n/a: no order n/a: no order n/a: no order n/a: no order n/a: no order n/a: no order
B-tree levels (ours)                                  1            2            2            3            3            4            5

fastest, Get (random hit): n=16 sorted Vec; n=64 sorted Vec; n=256 HashMap (std); n=1000 HashMap (std); n=10000 HashMap (std); n=100000 HashMap (std); n=1000000 HashMap (std)
fastest, Put+Delete pair: HashMap (std) at every size measured
fastest, Range of 1,000 pairs: sorted Vec at every size measured
Put+Delete at n=16       : sorted Vec 32.5 ns, our B-tree 34.1 ns: within 25%: no clear winner
Put+Delete at n=64       : sorted Vec 48.2 ns, our B-tree 48.9 ns: within 25%: no clear winner
Put+Delete at n=256      : sorted Vec 84.5 ns, our B-tree 56.2 ns: our B-tree is faster, 1.5x
Put+Delete at n=1000     : sorted Vec 193.7 ns, our B-tree 70.6 ns: our B-tree is faster, 2.7x
Put+Delete at n=10000    : sorted Vec 1.45 us, our B-tree 92.9 ns: our B-tree is faster, 15.6x
Put+Delete at n=100000   : sorted Vec 21.78 us, our B-tree 141.0 ns: our B-tree is faster, 154.5x
Put+Delete at n=1000000  : sorted Vec 226.73 us, our B-tree 257.8 ns: our B-tree is faster, 879.6x
our B-tree against BTreeMap, time(ours) / time(BTreeMap), per size:
  Get (random hit)          0.21x    0.33x    0.41x    0.39x    0.43x    0.51x    0.59x
  Put+Delete pair           1.06x    1.32x    0.93x    0.84x    0.84x    0.91x    0.99x
  Range of 1,000 pairs        n/a      n/a      n/a    2.87x    2.27x    1.32x    0.76x
```

> **Go vs Rust:** Parts (a)-(c) print the same numbers as the Go run line for line (same PCG stream, same CLRS algorithm: 48,760 splits at t = 2, the same two planted bugs caught at operations #23 and #16); the Rust-only agreement line adds `BTreeMap` as a third opinion, using `range(..=k).next_back()` for floor and `range(k..).next()` for ceiling, and it never disagreed. The tree is a `Vec<Page>` arena with `u32` ids and a free list (safe Rust, no `Rc`, no `unsafe`), the visit counter is a `Cell` because `get(&self)` has to count, and Go's `iter.Seq2` becomes a hand-written `Iterator` over the same explicit stack, with a `pending` page that is entered only when the next pair is asked for. The clock has a `BTreeMap` row that Go cannot have, but it is not like for like (our node holds 31 keys and is searched by binary search, `BTreeMap`'s holds 11 and is scanned linearly: read in its source, not run): the sorted `Vec` beat our tree on `Get` by 1.5x-2x from n = 64 up in every run, where Go's slice and tree tie, and our `Range` of 1,000 pairs took 2.2x-2.9x of `BTreeMap`'s time at n = 1,000-10,000 but 0.75x-0.85x at 10^6.

**Complexity:** `Get`, `Put`, `Delete`, `Floor` and `Ceil` cost Θ(log_t n) page visits (a `Get` visits 1.98 pages at t = 16 and 5.69 at t = 2 on the proof's map of at most 366 keys), `Range` Θ(log_t n + k), and a split or merge is amortised O(1) per update (t = 16: 367 splits and 352 merges for 37,377 new keys and 37,051 deletions). The proof is clean: **0 mismatches and 0 invariant failures over 4 × 200,000 operations** with the checker and a full comparison with the oracle after every one, and in the grow-shrink phase the levels always equal 1 + root growths − root shrinks. The planted bugs are caught at once: the forgotten separator at operation #23 (a page with 2 keys and 4 children), the stale value at operation #16 (5110 where the oracle has 3024). The clock (ranges over several runs; ratios only where they held every time): the built-in `map` is the fastest for `Get` and for a `Put`+`Delete` pair at every size, 3.5-7.3 ns per `Get` up to 10^5 keys and 22-24 ns at 10^6; the B-tree and the sorted slice are within about 25% of each other on `Get` at every size (46 vs 51 ns at n = 1,000, 155-188 vs 147-180 ns at 10^6). The sorted slice's O(n) insert is not cheap for long: for a `Put`+`Delete` pair the two are within 25% up to n = 64, the B-tree is about 1.5x faster at 256 and 2x at 1,000, about 9x at 10^4, 95-100x at 10^5 and 630-740x at 10^6 (the slice pair costs 227-233 us there, the B-tree pair 307-367 ns). The slice wins where its contiguity pays, a `Range` of 1,000 pairs: 3x faster than the B-tree at n = 1,000 and 6-7x at 10^6 (0.65-0.75 us against 4.0-5.1 us). Go has no ordered map, so there is no stdlib row to compare with; the measurements favour a sorted slice for data that is built once and scanned, and this B-tree once updates arrive and n passes a few hundred.

---

> Next lesson: [20 — Heaps & Priority Queues](../../20-heaps-priority-queues.md)

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
