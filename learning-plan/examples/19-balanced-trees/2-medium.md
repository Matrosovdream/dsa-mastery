# Step 19 — Balanced Trees & B-Trees · 🟡 Medium

Examples **7–11**: AVL, red-black insertion, a treap, a skip list and a B-tree, each built, checked against a
sorted-slice oracle and an invariant checker, and counted.

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

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. AVL tree, built and proved

`🟡 medium` · *AVL*

Examples 2 and 3 showed the rotation and the four rebalance cases in isolation; this one assembles them into a complete AVL tree with `Insert`, `Delete` (a node with two children takes its in-order successor, found by `takeMin`) and `Contains`, each node storing its height. AVL is the right tool when lookups dominate and a strict height guarantee is worth a rotation or two per update: the height is bounded by 1.4405·lg(n+2) − 0.3277 for every input order, so search, insert and delete are Θ(log n) in the worst case. The tree is proved the way this repo proves a structure, an invariant checker (stored heights, every balance factor in {−1, 0, +1}, in-order keys equal to a sorted-slice oracle) run after *every* operation of a PCG-seeded stream, and then the theorem is put next to the data: the tallest tree seen against the bound, rotations per insert, and the most fix-ups any single insert ever needed.

**Steps:**

1. Build the tree, a `Stats` counter (comparisons, single and double fix-ups, rotations) and the checker, then run 200,000 random operations (45% insert, 35% delete, 20% lookup, keys 0..499) against a sorted slice with the checker after each one.
2. The key move: plant one bug, a missing height update after the first rotation of a double rotation, replay the identical stream, and see which operation the checker catches it at; then build trees of 10³ to 10⁶ keys from sorted and shuffled input and delete a 100,000-key tree in random order.
3. Read the results against the bound, the ideal `⌈lg(n+1)⌉`, and the "at most one fix-up per insert" claim, which is measured here as a maximum, while deletes are allowed several.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"unsafe"
)

// An AVL tree: a BST in which every node stores its height and the heights of
// its two subtrees never differ by more than one.
type node struct {
	key         int
	h           int8 // height in nodes (empty = 0, leaf = 1): the only balance bookkeeping AVL needs
	left, right *node
}

// Stats counts the work, so the claims below are measured, not asserted.
//
//	cmps   nodes compared against (one three-way comparison each)
//	fixups rebalance steps that rotated (a node whose balance factor reached +-2)
//	rots   individual rotations (a double rotation is two)
type Stats struct {
	cmps, rots, single, double, fixups int
	plantBug                           bool // part (b): skip a height update (see rotLeft)
}

func ht(n *node) int {
	if n == nil {
		return 0
	}
	return int(n.h)
}

func upd(n *node) { n.h = int8(1 + max(ht(n.left), ht(n.right))) }

// rotRight lifts y's left child x above y and returns x, the new subtree root.
// fix=false skips the height recomputation; only the planted bug passes that.
func rotRight(y *node, st *Stats, fix bool) *node {
	x := y.left
	y.left = x.right
	x.right = y
	if fix {
		upd(y) // y went DOWN, so its height is recomputed first
		upd(x)
	}
	st.rots++
	return x
}

func rotLeft(x *node, st *Stats, fix bool) *node {
	y := x.right
	x.right = y.left
	y.left = x
	if fix {
		upd(x)
		upd(y)
	}
	st.rots++
	return y
}

// rebalance recomputes n's height and repairs a balance factor of +-2.
// The choice between one and two rotations is the sign of the heavy child's factor.
func rebalance(n *node, st *Stats) *node {
	upd(n)
	switch bf := ht(n.left) - ht(n.right); {
	case bf > 1:
		st.fixups++
		if ht(n.left.left) < ht(n.left.right) { // left-right: straighten the zig-zag first
			n.left = rotLeft(n.left, st, !st.plantBug)
			st.double++
		} else { // left-left, or (delete only) a balanced heavy child
			st.single++
		}
		return rotRight(n, st, true)
	case bf < -1:
		st.fixups++
		if ht(n.right.right) < ht(n.right.left) { // right-left
			n.right = rotRight(n.right, st, !st.plantBug)
			st.double++
		} else {
			st.single++
		}
		return rotLeft(n, st, true)
	}
	return n
}

// insert returns the new subtree root (rebalancing replaces the node we stand on).
func insert(n *node, key int, st *Stats) (*node, bool) {
	if n == nil {
		return &node{key: key, h: 1}, true
	}
	st.cmps++
	var ok bool
	switch {
	case key < n.key:
		n.left, ok = insert(n.left, key, st)
	case key > n.key:
		n.right, ok = insert(n.right, key, st)
	default:
		return n, false // already present: nothing changed, nothing to rebalance
	}
	if !ok {
		return n, false
	}
	return rebalance(n, st), true
}

// takeMin detaches the leftmost node of n's subtree and returns its key.
func takeMin(n *node, st *Stats) (*node, int) {
	if n.left == nil {
		return n.right, n.key
	}
	var k int
	n.left, k = takeMin(n.left, st)
	return rebalance(n, st), k
}

func remove(n *node, key int, st *Stats) (*node, bool) {
	if n == nil {
		return nil, false
	}
	st.cmps++
	var ok bool
	switch {
	case key < n.key:
		n.left, ok = remove(n.left, key, st)
	case key > n.key:
		n.right, ok = remove(n.right, key, st)
	default:
		ok = true
		if n.left == nil {
			return n.right, true
		}
		if n.right == nil {
			return n.left, true
		}
		// two children: the in-order successor takes this node's place
		n.right, n.key = takeMin(n.right, st)
	}
	if !ok {
		return n, false
	}
	return rebalance(n, st), true
}

func contains(n *node, key int) bool {
	for n != nil && n.key != key {
		if key < n.key {
			n = n.left
		} else {
			n = n.right
		}
	}
	return n != nil
}

func inorder(n *node, out []int) []int {
	if n == nil {
		return out
	}
	out = inorder(n.left, out)
	out = append(out, n.key)
	return inorder(n.right, out)
}

// walk checks every stored height and balance factor below n; it returns n's true height.
func walk(n *node) (int, error) {
	if n == nil {
		return 0, nil
	}
	hl, err := walk(n.left)
	if err != nil {
		return 0, err
	}
	hr, err := walk(n.right)
	if err != nil {
		return 0, err
	}
	h := 1 + max(hl, hr)
	if int(n.h) != h {
		return 0, fmt.Errorf("stale height at key %d: stored %d, actual %d", n.key, n.h, h)
	}
	if d := hl - hr; d > 1 || d < -1 {
		return 0, fmt.Errorf("balance factor %d at key %d", d, n.key)
	}
	return h, nil
}

// verify is the invariant checker: stored heights, |factor| <= 1, and the in-order
// key sequence equal to the oracle's (which covers BST order and the size).
func verify(root *node, want []int, buf []int) error {
	if _, err := walk(root); err != nil {
		return err
	}
	got := inorder(root, buf[:0])
	if len(got) != len(want) {
		return fmt.Errorf("tree holds %d keys, oracle holds %d", len(got), len(want))
	}
	for i := range got {
		if got[i] != want[i] {
			return fmt.Errorf("in-order[%d] = %d, oracle has %d", i, got[i], want[i])
		}
	}
	return nil
}

// bound is the AVL height theorem in nodes: h < 1.4405*lg(n+2) - 0.3277.
func bound(n int) float64 { return 1.4405*math.Log2(float64(n)+2) - 0.3277 }

func ideal(n int) int { return bits.Len(uint(n)) } // ceil(lg(n+1))

func verdict(ok bool) string {
	if ok {
		return "OK"
	}
	return "VIOLATED"
}

// opRun drives a PCG-seeded stream of operations against the tree and a sorted-slice
// oracle, and runs the checker after EVERY operation.
func opRun(ops int, plant bool) {
	rng := rand.New(rand.NewPCG(1, 2))
	var root *node
	var st Stats
	st.plantBug = plant
	oracle := []int{}
	buf := make([]int, 0, 1024)
	var mismatches, invFail, boundFail, maxH int
	var insFix [3]int
	var delFix [8]int
	maxIns, maxDel := 0, 0
	for op := 1; op <= ops; op++ {
		r, key := rng.IntN(100), rng.IntN(500)
		i, found := slices.BinarySearch(oracle, key)
		before := st.fixups
		var got, want bool
		kind := "lookup"
		switch {
		case r < 45:
			kind = "insert"
			root, got = insert(root, key, &st)
			want = !found
			if want {
				oracle = slices.Insert(oracle, i, key)
			}
		case r < 80:
			kind = "delete"
			root, got = remove(root, key, &st)
			want = found
			if found {
				oracle = slices.Delete(oracle, i, i+1)
			}
		default:
			got, want = contains(root, key), found
		}
		if got != want {
			mismatches++
		}
		d := st.fixups - before
		if kind == "insert" {
			insFix[d]++
			maxIns = max(maxIns, d)
		} else if kind == "delete" {
			delFix[d]++
			maxDel = max(maxDel, d)
		}
		maxH = max(maxH, ht(root))
		if float64(ht(root)) >= bound(len(oracle)) {
			boundFail++
		}
		if err := verify(root, oracle, buf); err != nil {
			invFail++
			if plant {
				fmt.Printf("  the checker fired at op %d (%s %d): %v\n", op, kind, key, err)
				fmt.Printf("  fix-ups up to and including that op: %d, of which %d were double rotations (the only place the bug lives)\n",
					st.fixups, st.double)
				return
			}
		}
	}
	fmt.Printf("  ops %d, mismatches vs oracle %d, invariant failures %d, final size %d\n", ops, mismatches, invFail, len(oracle))
	fmt.Printf("  max height seen %d; height >= bound(size) at %d ops (bound for 500 keys = %.2f)\n", maxH, boundFail, bound(500))
	fmt.Printf("  fix-ups per insert: 0 x %d, 1 x %d (max %d)\n", insFix[0], insFix[1], maxIns)
	fmt.Printf("  fix-ups per delete:")
	for k := 0; k <= maxDel; k++ {
		fmt.Printf(" %d x %d,", k, delFix[k])
	}
	fmt.Printf(" (max %d)\n", maxDel)
	fmt.Printf("  totals: %d single + %d double fix-ups, %d rotations, %d comparisons\n", st.single, st.double, st.rots, st.cmps)
}

func main() {
	fmt.Println("== (a) 200,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op ==")
	opRun(200_000, false)

	fmt.Println("\n== (b) the same stream with a planted bug: no height update after the first rotation of a double rotation ==")
	opRun(200_000, true)

	fmt.Println("\n== (c) height and rotation cost, sorted vs random insertion ==")
	fmt.Printf("node size: %d bytes with the height byte, %d without it (key + two pointers): padding swallows the saving\n",
		unsafe.Sizeof(node{}), unsafe.Sizeof(struct {
			key  int
			l, r *node
		}{}))
	fmt.Printf("%9s %-6s %6s %6s %6s %-4s %8s %8s %9s %9s  %s\n",
		"n", "order", "height", "ideal", "bound", "ok", "single", "double", "rots/ins", "max fix/i", "checker")
	rng := rand.New(rand.NewPCG(1, 2))
	var keep *node
	var keepKeys []int
	for _, n := range []int{1_000, 10_000, 100_000, 1_000_000} {
		sorted := make([]int, n)
		for i := range sorted {
			sorted[i] = i + 1
		}
		shuffled := slices.Clone(sorted)
		for i := n - 1; i >= 1; i-- { // Fisher-Yates by hand, so Go and Rust draw identically
			j := rng.IntN(i + 1)
			shuffled[i], shuffled[j] = shuffled[j], shuffled[i]
		}
		for _, in := range []struct {
			name string
			keys []int
		}{{"sorted", sorted}, {"random", shuffled}} {
			var root *node
			var st Stats
			maxFix := 0
			for _, k := range in.keys {
				before := st.fixups
				root, _ = insert(root, k, &st)
				maxFix = max(maxFix, st.fixups-before)
			}
			h := ht(root)
			chk := "ok"
			if err := verify(root, sorted, make([]int, 0, n)); err != nil {
				chk = err.Error()
			}
			fmt.Printf("%9d %-6s %6d %6d %6.2f %-4s %8d %8d %9.3f %9d  %s\n", n, in.name, h, ideal(n), bound(n),
				verdict(float64(h) < bound(n)), st.single, st.double, float64(st.rots)/float64(n), maxFix, chk)
			if in.name == "sorted" {
				fmt.Printf("%9s sorted rotations == n - bits.Len(n)? %v (%d vs %d)\n", "", st.rots == n-bits.Len(uint(n)), st.rots, n-bits.Len(uint(n)))
			}
			if n == 100_000 && in.name == "random" {
				keep, keepKeys = root, shuffled
			}
		}
	}

	fmt.Println("\n== (d) delete all 100,000 keys of the random tree, in random order ==")
	n := len(keepKeys)
	order := slices.Clone(keepKeys)
	for i := n - 1; i >= 1; i-- {
		j := rng.IntN(i + 1)
		order[i], order[j] = order[j], order[i]
	}
	present := make([]bool, n+1)
	for i := range present {
		present[i] = true
	}
	var dst Stats
	var fixHist [8]int
	maxRots, maxFix, missing, failures := 0, 0, 0, 0
	for idx, k := range order {
		rb, fb := dst.rots, dst.fixups
		var ok bool
		keep, ok = remove(keep, k, &dst)
		if !ok {
			missing++
		}
		present[k] = false
		maxRots = max(maxRots, dst.rots-rb)
		fixHist[dst.fixups-fb]++
		maxFix = max(maxFix, dst.fixups-fb)
		if (idx+1)%10_000 == 0 { // the checker on the 10 checkpoints (per-op would be O(n^2))
			var want []int
			for key := 1; key <= n; key++ {
				if present[key] {
					want = append(want, key)
				}
			}
			if err := verify(keep, want, make([]int, 0, n)); err != nil {
				failures++
				fmt.Println("  checker:", err)
			}
		}
	}
	fmt.Printf("  deletes %d, keys not found %d, checker failures at 10 checkpoints %d, tree empty at the end: %v\n", n, missing, failures, keep == nil)
	fmt.Printf("  rotations per delete: mean %.3f, max %d; fix-ups per delete: max %d\n", float64(dst.rots)/float64(n), maxRots, maxFix)
	fmt.Printf("  fix-ups per delete histogram:")
	for k := 0; k <= maxFix; k++ {
		fmt.Printf(" %d x %d,", k, fixHist[k])
	}
	fmt.Printf("\n  single %d, double %d, comparisons per delete %.2f\n", dst.single, dst.double, float64(dst.cmps)/float64(n))
}
```

**Output:**

```
== (a) 200,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op ==
  ops 200000, mismatches vs oracle 0, invariant failures 0, final size 303
  max height seen 10; height >= bound(size) at 0 ops (bound for 500 keys = 12.60)
  fix-ups per insert: 0 x 74467, 1 x 15619 (max 1)
  fix-ups per delete: 0 x 60688, 1 x 8441, 2 x 747, 3 x 25, (max 3)
  totals: 14404 single + 11225 double fix-ups, 36854 rotations, 1234102 comparisons

== (b) the same stream with a planted bug: no height update after the first rotation of a double rotation ==
  the checker fired at op 11 (insert 320): stale height at key 320: stored 3, actual 2
  fix-ups up to and including that op: 3, of which 2 were double rotations (the only place the bug lives)

== (c) height and rotation cost, sorted vs random insertion ==
node size: 32 bytes with the height byte, 24 without it (key + two pointers): padding swallows the saving
        n order  height  ideal  bound ok     single   double  rots/ins max fix/i  checker
     1000 sorted     10     10  14.03 OK        990        0     0.990         1  ok
          sorted rotations == n - bits.Len(n)? true (990 vs 990)
     1000 random     12     10  14.03 OK        239      227     0.693         1  ok
    10000 sorted     14     14  18.81 OK       9986        0     0.999         1  ok
          sorted rotations == n - bits.Len(n)? true (9986 vs 9986)
    10000 random     16     14  18.81 OK       2317     2240     0.680         1  ok
   100000 sorted     17     17  23.60 OK      99983        0     1.000         1  ok
          sorted rotations == n - bits.Len(n)? true (99983 vs 99983)
   100000 random     20     17  23.60 OK      23142    23353     0.698         1  ok
  1000000 sorted     20     20  28.38 OK     999980        0     1.000         1  ok
          sorted rotations == n - bits.Len(n)? true (999980 vs 999980)
  1000000 random     24     20  28.38 OK     233534   231997     0.698         1  ok

== (d) delete all 100,000 keys of the random tree, in random order ==
  deletes 100000, keys not found 0, checker failures at 10 checkpoints 0, tree empty at the end: true
  rotations per delete: mean 0.373, max 8; fix-ups per delete: max 4
  fix-ups per delete histogram: 0 x 76157, 1 x 21273, 2 x 2376, 3 x 183, 4 x 11,
  single 15905, double 10713, comparisons per delete 14.48
```

**Rust:**

```rust
use std::cmp::max;
use std::mem::size_of;

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

// An AVL tree: a BST in which every node stores its height and the heights of
// its two subtrees never differ by more than one. One owner per node, so a
// plain `Option<Box<Node>>` is enough; no parent links are needed.
type Link = Option<Box<Node>>;

struct Node {
    key: u64,
    h: i8, // height in nodes (empty = 0, leaf = 1): the only balance bookkeeping AVL needs
    left: Link,
    right: Link,
}

// Stats counts the work, so the claims below are measured, not asserted.
//   cmps   nodes compared against (one three-way comparison each)
//   fixups rebalance steps that rotated (a node whose balance factor reached +-2)
//   rots   individual rotations (a double rotation is two)
// Go keeps this in a struct passed by pointer; Rust threads it as `&mut Stats`.
#[derive(Default)]
struct Stats {
    cmps: usize,
    rots: usize,
    single: usize,
    double: usize,
    fixups: usize,
    plant_bug: bool, // part (b): skip a height update (see rot_left)
}

fn ht(link: &Link) -> i32 {
    link.as_ref().map_or(0, |n| i32::from(n.h))
}

fn upd(n: &mut Node) {
    n.h = (1 + max(ht(&n.left), ht(&n.right))) as i8;
}

// rot_right lifts y's left child x above y and returns x, the new subtree root.
// A rotation consumes the node and hands back the new root: the caller must store
// it. `take()` moves the child out of its field, because a field behind `&mut`
// cannot simply be moved. fix=false skips the height recomputation; only the
// planted bug passes that. With no left child the node comes back unchanged
// (total: no panic), where the Go version would dereference nil.
fn rot_right(mut y: Box<Node>, st: &mut Stats, fix: bool) -> Box<Node> {
    let Some(mut x) = y.left.take() else {
        return y;
    };
    y.left = x.right.take();
    if fix {
        upd(&mut y); // y went DOWN, so its height is recomputed first
    }
    x.right = Some(y);
    if fix {
        upd(&mut x);
    }
    st.rots += 1;
    x
}

fn rot_left(mut x: Box<Node>, st: &mut Stats, fix: bool) -> Box<Node> {
    let Some(mut y) = x.right.take() else {
        return x;
    };
    x.right = y.left.take();
    if fix {
        upd(&mut x);
    }
    y.left = Some(x);
    if fix {
        upd(&mut y);
    }
    st.rots += 1;
    y
}

// rebalance recomputes n's height and repairs a balance factor of +-2.
// The choice between one and two rotations is the sign of the heavy child's factor.
fn rebalance(mut n: Box<Node>, st: &mut Stats) -> Box<Node> {
    upd(&mut n);
    let bf = ht(&n.left) - ht(&n.right);
    let fix_first = !st.plant_bug;
    if bf > 1 {
        st.fixups += 1;
        // left-right: straighten the zig-zag first
        if n.left.as_ref().is_some_and(|l| ht(&l.left) < ht(&l.right)) {
            n.left = n.left.take().map(|c| rot_left(c, st, fix_first));
            st.double += 1;
        } else {
            // left-left, or (delete only) a balanced heavy child
            st.single += 1;
        }
        return rot_right(n, st, true);
    }
    if bf < -1 {
        st.fixups += 1;
        // right-left
        if n.right.as_ref().is_some_and(|r| ht(&r.right) < ht(&r.left)) {
            n.right = n.right.take().map(|c| rot_right(c, st, fix_first));
            st.double += 1;
        } else {
            st.single += 1;
        }
        return rot_left(n, st, true);
    }
    n
}

// Rebalance only when something below changed.
fn finish(n: Box<Node>, changed: bool, st: &mut Stats) -> (Link, bool) {
    if changed {
        (Some(rebalance(n, st)), true)
    } else {
        (Some(n), false)
    }
}

// insert returns the new subtree root (rebalancing replaces the node we stand
// on, which is why this is not a `&mut Link` function). The link is taken out
// of its parent, transformed, and put back by the caller.
fn insert(link: Link, key: u64, st: &mut Stats) -> (Link, bool) {
    let Some(mut n) = link else {
        let leaf = Node {
            key,
            h: 1,
            left: None,
            right: None,
        };
        return (Some(Box::new(leaf)), true);
    };
    st.cmps += 1;
    let ok;
    if key < n.key {
        (n.left, ok) = insert(n.left.take(), key, st);
    } else if key > n.key {
        (n.right, ok) = insert(n.right.take(), key, st);
    } else {
        return (Some(n), false); // already present: nothing changed
    }
    finish(n, ok, st)
}

// take_min detaches the leftmost node of n's subtree and returns its key.
fn take_min(mut n: Box<Node>, st: &mut Stats) -> (Link, u64) {
    match n.left.take() {
        None => (n.right.take(), n.key),
        Some(l) => {
            let (rest, k) = take_min(l, st);
            n.left = rest;
            (Some(rebalance(n, st)), k)
        }
    }
}

fn remove(link: Link, key: u64, st: &mut Stats) -> (Link, bool) {
    let Some(mut n) = link else {
        return (None, false);
    };
    st.cmps += 1;
    let ok;
    if key < n.key {
        (n.left, ok) = remove(n.left.take(), key, st);
    } else if key > n.key {
        (n.right, ok) = remove(n.right.take(), key, st);
    } else {
        return match (n.left.take(), n.right.take()) {
            (None, r) => (r, true),
            (l, None) => (l, true),
            (l, Some(r)) => {
                // two children: the in-order successor takes this node's place
                let (r, succ) = take_min(r, st);
                n.left = l;
                n.right = r;
                n.key = succ;
                (Some(rebalance(n, st)), true)
            }
        };
    }
    finish(n, ok, st)
}

fn contains(mut cur: &Link, key: u64) -> bool {
    while let Some(n) = cur {
        if key == n.key {
            return true;
        }
        cur = if key < n.key { &n.left } else { &n.right };
    }
    false
}

fn inorder(link: &Link, out: &mut Vec<u64>) {
    if let Some(n) = link {
        inorder(&n.left, out);
        out.push(n.key);
        inorder(&n.right, out);
    }
}

// walk checks every stored height and balance factor below n; it returns n's true height.
fn walk(link: &Link) -> Result<i32, String> {
    let Some(n) = link else {
        return Ok(0);
    };
    let hl = walk(&n.left)?;
    let hr = walk(&n.right)?;
    let h = 1 + max(hl, hr);
    if i32::from(n.h) != h {
        return Err(format!(
            "stale height at key {}: stored {}, actual {}",
            n.key, n.h, h
        ));
    }
    let d = hl - hr;
    if !(-1..=1).contains(&d) {
        return Err(format!("balance factor {} at key {}", d, n.key));
    }
    Ok(h)
}

// verify is the invariant checker: stored heights, |factor| <= 1, and the in-order
// key sequence equal to the oracle's (which covers BST order and the size).
fn verify(root: &Link, want: &[u64], buf: &mut Vec<u64>) -> Result<(), String> {
    walk(root)?;
    buf.clear();
    inorder(root, buf);
    if buf.len() != want.len() {
        return Err(format!(
            "tree holds {} keys, oracle holds {}",
            buf.len(),
            want.len()
        ));
    }
    for (i, (g, w)) in buf.iter().zip(want).enumerate() {
        if g != w {
            return Err(format!("in-order[{i}] = {g}, oracle has {w}"));
        }
    }
    Ok(())
}

// bound is the AVL height theorem in nodes: h < 1.4405*lg(n+2) - 0.3277.
fn bound(n: usize) -> f64 {
    1.4405 * (n as f64 + 2.0).log2() - 0.3277
}

fn ideal(n: usize) -> u32 {
    usize::BITS - n.leading_zeros() // ceil(lg(n+1))
}

fn verdict(ok: bool) -> &'static str {
    if ok { "OK" } else { "VIOLATED" }
}

// op_run drives a PCG-seeded stream of operations against the tree and a sorted-slice
// oracle, and runs the checker after EVERY operation.
fn op_run(ops: usize, plant: bool) {
    let mut rng = Pcg::new(1, 2);
    let mut root: Link = None;
    let mut st = Stats {
        plant_bug: plant,
        ..Stats::default()
    };
    let mut oracle: Vec<u64> = Vec::new();
    let mut buf: Vec<u64> = Vec::with_capacity(1024);
    let (mut mismatches, mut inv_fail, mut bound_fail, mut max_h) = (0, 0, 0, 0);
    let mut ins_fix = [0usize; 16];
    let mut del_fix = [0usize; 16];
    let (mut max_ins, mut max_del) = (0, 0);
    for op in 1..=ops {
        let r = rng.int_n(100);
        let key = rng.int_n(500);
        let i = oracle.partition_point(|&x| x < key);
        let found = oracle.get(i) == Some(&key);
        let before = st.fixups;
        let kind;
        let (got, want);
        if r < 45 {
            kind = "insert";
            (root, got) = insert(root.take(), key, &mut st);
            want = !found;
            if want {
                oracle.insert(i, key);
            }
        } else if r < 80 {
            kind = "delete";
            (root, got) = remove(root.take(), key, &mut st);
            want = found;
            if found {
                oracle.remove(i);
            }
        } else {
            kind = "lookup";
            got = contains(&root, key);
            want = found;
        }
        if got != want {
            mismatches += 1;
        }
        let d = st.fixups - before;
        if kind == "insert" {
            ins_fix[d] += 1;
            max_ins = max(max_ins, d);
        } else if kind == "delete" {
            del_fix[d] += 1;
            max_del = max(max_del, d);
        }
        max_h = max(max_h, ht(&root));
        if f64::from(ht(&root)) >= bound(oracle.len()) {
            bound_fail += 1;
        }
        if let Err(err) = verify(&root, &oracle, &mut buf) {
            inv_fail += 1;
            if plant {
                println!("  the checker fired at op {op} ({kind} {key}): {err}");
                println!(
                    "  fix-ups up to and including that op: {}, of which {} were double rotations (the only place the bug lives)",
                    st.fixups, st.double
                );
                return;
            }
        }
    }
    println!(
        "  ops {ops}, mismatches vs oracle {mismatches}, invariant failures {inv_fail}, final size {}",
        oracle.len()
    );
    println!(
        "  max height seen {max_h}; height >= bound(size) at {bound_fail} ops (bound for 500 keys = {:.2})",
        bound(500)
    );
    println!(
        "  fix-ups per insert: 0 x {}, 1 x {} (max {max_ins})",
        ins_fix[0], ins_fix[1]
    );
    print!("  fix-ups per delete:");
    for (k, c) in del_fix.iter().enumerate().take(max_del + 1) {
        print!(" {k} x {c},");
    }
    println!(" (max {max_del})");
    println!(
        "  totals: {} single + {} double fix-ups, {} rotations, {} comparisons",
        st.single, st.double, st.rots, st.cmps
    );
}

fn main() {
    println!(
        "== (a) 200,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op =="
    );
    op_run(200_000, false);

    println!(
        "\n== (b) the same stream with a planted bug: no height update after the first rotation of a double rotation =="
    );
    op_run(200_000, true);

    println!("\n== (c) height and rotation cost, sorted vs random insertion ==");
    println!(
        "node size: {} bytes with the height byte, {} without it (key + two pointers): padding swallows the saving",
        size_of::<Node>(),
        size_of::<(u64, Link, Link)>()
    );
    println!(
        "{:>9} {:<6} {:>6} {:>6} {:>6} {:<4} {:>8} {:>8} {:>9} {:>9}  checker",
        "n", "order", "height", "ideal", "bound", "ok", "single", "double", "rots/ins", "max fix/i"
    );
    let mut rng = Pcg::new(1, 2);
    let mut keep: Link = None;
    let mut keep_keys: Vec<u64> = Vec::new();
    for n in [1_000usize, 10_000, 100_000, 1_000_000] {
        let sorted: Vec<u64> = (1..=n as u64).collect();
        let mut shuffled = sorted.clone();
        for i in (1..n).rev() {
            // Fisher-Yates by hand, so Go and Rust draw identically
            let j = rng.int_n(i as u64 + 1) as usize;
            shuffled.swap(i, j);
        }
        for (name, keys) in [("sorted", &sorted), ("random", &shuffled)] {
            let mut root: Link = None;
            let mut st = Stats::default();
            let mut max_fix = 0;
            for &k in keys {
                let before = st.fixups;
                (root, _) = insert(root.take(), k, &mut st);
                max_fix = max(max_fix, st.fixups - before);
            }
            let h = ht(&root);
            let chk = match verify(&root, &sorted, &mut Vec::with_capacity(n)) {
                Ok(()) => "ok".to_string(),
                Err(e) => e,
            };
            println!(
                "{:>9} {:<6} {:>6} {:>6} {:>6.2} {:<4} {:>8} {:>8} {:>9.3} {:>9}  {}",
                n,
                name,
                h,
                ideal(n),
                bound(n),
                verdict(f64::from(h) < bound(n)),
                st.single,
                st.double,
                st.rots as f64 / n as f64,
                max_fix,
                chk
            );
            if name == "sorted" {
                let pred = n - ideal(n) as usize;
                println!(
                    "{:>9} sorted rotations == n - bits.Len(n)? {} ({} vs {})",
                    "",
                    st.rots == pred,
                    st.rots,
                    pred
                );
            }
            if n == 100_000 && name == "random" {
                keep = root;
                keep_keys = shuffled.clone();
            }
        }
    }

    println!("\n== (d) delete all 100,000 keys of the random tree, in random order ==");
    let n = keep_keys.len();
    let mut order = keep_keys.clone();
    for i in (1..n).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        order.swap(i, j);
    }
    let mut present = vec![true; n + 1];
    let mut dst = Stats::default();
    let mut fix_hist = [0usize; 16];
    let (mut max_rots, mut max_fix, mut missing, mut failures) = (0, 0, 0, 0);
    for (idx, &k) in order.iter().enumerate() {
        let (rb, fb) = (dst.rots, dst.fixups);
        let ok;
        (keep, ok) = remove(keep.take(), k, &mut dst);
        if !ok {
            missing += 1;
        }
        present[k as usize] = false;
        max_rots = max(max_rots, dst.rots - rb);
        fix_hist[dst.fixups - fb] += 1;
        max_fix = max(max_fix, dst.fixups - fb);
        if (idx + 1) % 10_000 == 0 {
            // the checker on the 10 checkpoints (per-op would be O(n^2))
            let want: Vec<u64> = (1..=n as u64)
                .filter(|&key| present[key as usize])
                .collect();
            if let Err(err) = verify(&keep, &want, &mut Vec::with_capacity(n)) {
                failures += 1;
                println!("  checker: {err}");
            }
        }
    }
    println!(
        "  deletes {n}, keys not found {missing}, checker failures at 10 checkpoints {failures}, tree empty at the end: {}",
        keep.is_none()
    );
    println!(
        "  rotations per delete: mean {:.3}, max {max_rots}; fix-ups per delete: max {max_fix}",
        dst.rots as f64 / n as f64
    );
    print!("  fix-ups per delete histogram:");
    for (k, c) in fix_hist.iter().enumerate().take(max_fix + 1) {
        print!(" {k} x {c},");
    }
    println!(
        "\n  single {}, double {}, comparisons per delete {:.2}",
        dst.single,
        dst.double,
        dst.cmps as f64 / n as f64
    );
}
```

**Rust output:**

```
== (a) 200,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op ==
  ops 200000, mismatches vs oracle 0, invariant failures 0, final size 303
  max height seen 10; height >= bound(size) at 0 ops (bound for 500 keys = 12.60)
  fix-ups per insert: 0 x 74467, 1 x 15619 (max 1)
  fix-ups per delete: 0 x 60688, 1 x 8441, 2 x 747, 3 x 25, (max 3)
  totals: 14404 single + 11225 double fix-ups, 36854 rotations, 1234102 comparisons

== (b) the same stream with a planted bug: no height update after the first rotation of a double rotation ==
  the checker fired at op 11 (insert 320): stale height at key 320: stored 3, actual 2
  fix-ups up to and including that op: 3, of which 2 were double rotations (the only place the bug lives)

== (c) height and rotation cost, sorted vs random insertion ==
node size: 32 bytes with the height byte, 24 without it (key + two pointers): padding swallows the saving
        n order  height  ideal  bound ok     single   double  rots/ins max fix/i  checker
     1000 sorted     10     10  14.03 OK        990        0     0.990         1  ok
          sorted rotations == n - bits.Len(n)? true (990 vs 990)
     1000 random     12     10  14.03 OK        239      227     0.693         1  ok
    10000 sorted     14     14  18.81 OK       9986        0     0.999         1  ok
          sorted rotations == n - bits.Len(n)? true (9986 vs 9986)
    10000 random     16     14  18.81 OK       2317     2240     0.680         1  ok
   100000 sorted     17     17  23.60 OK      99983        0     1.000         1  ok
          sorted rotations == n - bits.Len(n)? true (99983 vs 99983)
   100000 random     20     17  23.60 OK      23142    23353     0.698         1  ok
  1000000 sorted     20     20  28.38 OK     999980        0     1.000         1  ok
          sorted rotations == n - bits.Len(n)? true (999980 vs 999980)
  1000000 random     24     20  28.38 OK     233534   231997     0.698         1  ok

== (d) delete all 100,000 keys of the random tree, in random order ==
  deletes 100000, keys not found 0, checker failures at 10 checkpoints 0, tree empty at the end: true
  rotations per delete: mean 0.373, max 8; fix-ups per delete: max 4
  fix-ups per delete histogram: 0 x 76157, 1 x 21273, 2 x 2376, 3 x 183, 4 x 11,
  single 15905, double 10713, comparisons per delete 14.48
```

> **Go vs Rust:** Output is identical to Go, line for line (diffed: all 32 lines, so every count, height and the planted-bug op 11 match). Ownership shapes the updates: `insert` and `remove` take the subtree by value and return `(Link, bool)`, because rebalancing replaces the node you are standing on, and `(n.left, ok) = insert(n.left.take(), key, st)` moves the child out, recurses and writes the new root back, where Go assigns `n.left, ok = insert(n.left, key, st)` in place. The rotation is total (`let Some(..) = .. else { return y }`, no panic path) where Go's would dereference nil, and no iterative `Drop` is needed here: the default drop of the 10⁶-node tree recurses at most as deep as its height (24), unlike the degenerate chains of lesson 17 example 13. `size_of::<Node>()` is 32 in both languages.

**Complexity:** insert, delete and search Θ(log n) worst case, with height below 1.4405·lg(n+2) − 0.3277: the tallest tree built (24 levels for 10⁶ random keys, bound 28.38) is within 4 of the ideal 20, and sorted input gives exactly the ideal, 20 · **at most one fix-up per insert**, measured over 90,086 inserts in (a) and 2,222,000 in (c), while deletes needed up to 3 fix-ups in (a) and up to 4 (8 rotations in the worst one) among the 100,000 deletes of (d), which averaged 0.373 rotations per delete against 0.698 per insert when that same tree was built · sorted input costs exactly n − ⌈lg(n+1)⌉ single rotations · O(1) extra space per node, one height byte, which padding turns into 8 bytes (32 against 24).

---

## 8. Red-black insertion with the CLRS fix-up

`🟡 medium` · *Red-black insert*

The same job as example 7 with a different bill: instead of keeping heights exact, a red-black tree colours each new node red, recolours while the uncle is red (the trouble moves up two levels) and finishes with at most two rotations, so repairs are cheaper and the height is only guaranteed to stay below 2·lg(n+1). This is CLRS 13.3 line for line (parent links, one shared black sentinel for every nil leaf, the three cases and their mirror images), proved the same way as example 7: a checker that names the rule it finds broken (`root-red`, `parent-link`, `order`, `red-red`, `black-height`) and a sorted-slice oracle, run after every insert. Insert is Θ(log n); the "at most 3 rotations" for *delete* (CLRS 13.4) is quoted from the book and not measured here, because only insertion is built.

**Steps:**

1. Build the tree and the checker, then prove it with a checker and an oracle after every operation: 2,000 sorted keys, 2,000 shuffled keys, and 10,000 random operations (inserts, duplicates and lookups) on keys 0..999.
2. The key move: build 10³ and 10⁵ keys from sorted and shuffled input and print the height next to the ideal `⌈lg(n+1)⌉` and the bound `2·lg(n+1)`, then plant a bug (case 1 forgets to move `z` up) and see which rule the checker names, and at which operation.
3. Read the sorted row against the bound and against AVL's height in example 7, and the rotation and recolouring counts against the claim "at most 2 rotations per insert".

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"unsafe"
)

// A red-black tree, insertion only, written line for line from CLRS 13.3.
// Every node has a parent link, and one shared black sentinel stands in for
// every nil leaf, so "the uncle is black" never needs a nil check.
type color bool

const (
	black color = false
	red   color = true
)

func (c color) String() string {
	if c == red {
		return "red"
	}
	return "black"
}

type node struct {
	key                 int
	color               color
	left, right, parent *node
}

// Stats: rots counts rotations, recolours counts nodes whose colour actually
// changed, cmps counts nodes compared against while searching for the slot.
type Stats struct {
	rots, recolours, cmps int
	skipAdvance           bool // part (c): the planted bug, see fixup
}

type Tree struct {
	root, nilNode *node
	st            Stats
}

func newTree() *Tree {
	s := &node{color: black}
	s.left, s.right, s.parent = s, s, s
	return &Tree{root: s, nilNode: s}
}

func (t *Tree) paint(n *node, c color) {
	if n.color != c {
		n.color = c
		t.st.recolours++
	}
}

func (t *Tree) rotateLeft(x *node) {
	y := x.right
	x.right = y.left // y's left subtree becomes x's right subtree
	if y.left != t.nilNode {
		y.left.parent = x
	}
	y.parent = x.parent
	switch {
	case x.parent == t.nilNode:
		t.root = y
	case x == x.parent.left:
		x.parent.left = y
	default:
		x.parent.right = y
	}
	y.left = x
	x.parent = y
	t.st.rots++
}

func (t *Tree) rotateRight(y *node) {
	x := y.left
	y.left = x.right
	if x.right != t.nilNode {
		x.right.parent = y
	}
	x.parent = y.parent
	switch {
	case y.parent == t.nilNode:
		t.root = x
	case y == y.parent.right:
		y.parent.right = x
	default:
		y.parent.left = x
	}
	x.right = y
	y.parent = x
	t.st.rots++
}

func (t *Tree) has(key int) bool {
	x := t.root
	for x != t.nilNode && x.key != key {
		if key < x.key {
			x = x.left
		} else {
			x = x.right
		}
	}
	return x != t.nilNode
}

// insert is a plain BST insert of a RED node followed by fixup. A duplicate is a no-op.
func (t *Tree) insert(key int) bool {
	y, x := t.nilNode, t.root
	for x != t.nilNode {
		y = x
		t.st.cmps++
		switch {
		case key < x.key:
			x = x.left
		case key > x.key:
			x = x.right
		default:
			return false
		}
	}
	z := &node{key: key, color: red, left: t.nilNode, right: t.nilNode, parent: y}
	switch {
	case y == t.nilNode:
		t.root = z
	case key < y.key:
		y.left = z
	default:
		y.right = z
	}
	t.fixup(z)
	return true
}

// fixup restores the rules. The only rule a red z can break is "no red child under a red parent".
func (t *Tree) fixup(z *node) {
	for z.parent.color == red {
		g := z.parent.parent // the parent is red, so it is not the root, so g is a real node
		if z.parent == g.left {
			if u := g.right; u.color == red { // case 1: red uncle: recolour, trouble moves up two levels
				t.paint(z.parent, black)
				t.paint(u, black)
				t.paint(g, red)
				if !t.st.skipAdvance {
					z = g
				}
			} else {
				if z == z.parent.right { // case 2: zig-zag, straighten it into case 3
					z = z.parent
					t.rotateLeft(z)
				}
				t.paint(z.parent, black) // case 3: black uncle: one rotation ends the loop
				t.paint(g, red)
				t.rotateRight(g)
			}
		} else { // the mirror image
			if u := g.left; u.color == red {
				t.paint(z.parent, black)
				t.paint(u, black)
				t.paint(g, red)
				if !t.st.skipAdvance {
					z = g
				}
			} else {
				if z == z.parent.left {
					z = z.parent
					t.rotateRight(z)
				}
				t.paint(z.parent, black)
				t.paint(g, red)
				t.rotateLeft(g)
			}
		}
	}
	t.paint(t.root, black)
}

func (t *Tree) height(n *node) int {
	if n == t.nilNode {
		return 0
	}
	return 1 + max(t.height(n.left), t.height(n.right))
}

// walk checks node-local rules top-down, then returns the black height below n
// (black nodes on any path to a nil leaf, nil leaves not counted).
func (t *Tree) walk(n *node, lo, hi *int) (int, error) {
	if n == t.nilNode {
		return 0, nil
	}
	if (lo != nil && n.key <= *lo) || (hi != nil && n.key >= *hi) {
		return 0, fmt.Errorf("order: key %d breaks BST order", n.key)
	}
	for _, c := range []*node{n.left, n.right} {
		if c == t.nilNode {
			continue
		}
		if c.parent != n {
			return 0, fmt.Errorf("parent-link: key %d does not point back to its parent %d", c.key, n.key)
		}
		if n.color == red && c.color == red {
			return 0, fmt.Errorf("red-red: red key %d has red child %d", n.key, c.key)
		}
	}
	bl, err := t.walk(n.left, lo, &n.key)
	if err != nil {
		return 0, err
	}
	br, err := t.walk(n.right, &n.key, hi)
	if err != nil {
		return 0, err
	}
	if bl != br {
		return 0, fmt.Errorf("black-height: key %d has black height %d on its left, %d on its right", n.key, bl, br)
	}
	if n.color == black {
		bl++
	}
	return bl, nil
}

func inorder(t *Tree, n *node, out []int) []int {
	if n == t.nilNode {
		return out
	}
	out = inorder(t, n.left, out)
	out = append(out, n.key)
	return inorder(t, n.right, out)
}

// verify is the invariant checker: all five rules, parent links, and the in-order
// keys equal to the sorted-slice oracle. It returns the black height.
func (t *Tree) verify(oracle, buf []int) (int, error) {
	if t.root.color == red {
		return 0, fmt.Errorf("root-red: root key %d is red", t.root.key)
	}
	bh, err := t.walk(t.root, nil, nil)
	if err != nil {
		return 0, err
	}
	got := inorder(t, t.root, buf[:0])
	if !slices.Equal(got, oracle) {
		return 0, fmt.Errorf("oracle: tree holds %d keys, oracle holds %d (or they differ)", len(got), len(oracle))
	}
	return bh, nil
}

type op struct {
	key    int
	lookup bool
}

func inserts(keys []int) []op {
	ops := make([]op, len(keys))
	for i, k := range keys {
		ops[i] = op{key: k}
	}
	return ops
}

// prove runs the ops against the tree and a sorted-slice oracle with the checker after
// every one. With stopAtFail it stops at the first violation and reports it.
func prove(label string, ops []op, plant, stopAtFail bool) {
	t := newTree()
	t.st.skipAdvance = plant
	oracle := []int{}
	buf := make([]int, 0, len(ops))
	var hist [8]int
	maxRots, mismatches, failures, lookups, bh := 0, 0, 0, 0, 0
	for i, o := range ops {
		pos, found := slices.BinarySearch(oracle, o.key)
		if o.lookup {
			lookups++
			if t.has(o.key) != found {
				mismatches++
			}
		} else {
			before := t.st.rots
			if t.insert(o.key) == found {
				mismatches++
			}
			if !found {
				oracle = slices.Insert(oracle, pos, o.key)
			}
			hist[t.st.rots-before]++
			maxRots = max(maxRots, t.st.rots-before)
		}
		var err error
		if bh, err = t.verify(oracle, buf); err != nil {
			failures++
			if stopAtFail {
				fmt.Printf("  %s: the checker fired at op %d (insert %d): %v\n", label, i+1, o.key, err)
				return
			}
		}
	}
	fmt.Printf("%s: %d inserts, %d lookups\n", label, len(ops)-lookups, lookups)
	fmt.Printf("  mismatches vs oracle %d, invariant failures %d, final size %d, height %d, black height %d\n",
		mismatches, failures, len(oracle), t.height(t.root), bh)
	fmt.Printf("  rotations per insert: 0 x %d, 1 x %d, 2 x %d; max %d (theorem: at most 2, %s)\n",
		hist[0], hist[1], hist[2], maxRots, verdict(maxRots <= 2))
}

func verdict(ok bool) string {
	if ok {
		return "OK"
	}
	return "VIOLATED"
}

func seq(n int) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = i + 1
	}
	return a
}

func shuffled(n int, rng *rand.Rand) []int {
	a := seq(n)
	for i := n - 1; i >= 1; i-- { // Fisher-Yates by hand, so Go and Rust draw identically
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
	return a
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	fmt.Printf("node size: %d bytes (key, colour, three 8-byte pointers)\n", unsafe.Sizeof(node{}))

	fmt.Println("\n== (a) checker and sorted-slice oracle after EVERY operation ==")
	n := 2_000
	sortedKeys := seq(n)
	randKeys := shuffled(n, rng)
	prove("sorted 1..2000", inserts(sortedKeys), false, false)
	prove("random permutation of 1..2000", inserts(randKeys), false, false)
	mixed := make([]op, 10_000)
	for i := range mixed {
		r := rng.IntN(100)
		mixed[i] = op{key: rng.IntN(1000), lookup: r >= 70}
	}
	prove("random ops on keys 0..999 (70% insert, 30% lookup)", mixed, false, false)

	fmt.Println("\n== (b) height against the bound 2*lg(n+1), sorted vs random insertion ==")
	fmt.Printf("%7s %-6s %6s %3s %6s %6s %-4s %9s %9s %10s %8s  %s\n",
		"n", "order", "height", "bh", "ideal", "bound", "ok", "rotations", "rots/ins", "recolours", "max rot", "checker")
	var plain []int // sorted rows: n - rotations, then the final height
	for _, n := range []int{1_000, 100_000} {
		sorted := seq(n)
		for _, in := range []struct {
			name string
			keys []int
		}{{"sorted", sorted}, {"random", shuffled(n, rng)}} {
			t := newTree()
			maxRot := 0
			for _, k := range in.keys {
				before := t.st.rots
				t.insert(k)
				maxRot = max(maxRot, t.st.rots-before)
			}
			h, bound := t.height(t.root), 2*math.Log2(float64(n+1))
			chk := "ok"
			bh, err := t.verify(sorted, make([]int, 0, n))
			if err != nil {
				chk = err.Error()
			}
			fmt.Printf("%7d %-6s %6d %3d %6d %6.2f %-4s %9d %9.3f %10d %8d  %s\n", n, in.name, h, bh, bits.Len(uint(n)), bound,
				verdict(float64(h) <= bound), t.st.rots, float64(t.st.rots)/float64(n), t.st.recolours, maxRot, chk)
			if in.name == "sorted" {
				plain = append(plain, n-t.st.rots, h)
			}
		}
	}
	fmt.Printf("sorted rows: inserts that did not rotate = %d and %d, final heights = %d and %d\n", plain[0], plain[2], plain[1], plain[3])

	fmt.Println("\n== (c) planted bug: case 1 recolours but forgets to move z up to the grandparent ==")
	prove("sorted 1..2000", inserts(sortedKeys), true, true)
	prove("random permutation of 1..2000", inserts(randKeys), true, true)
}
```

**Output:**

```
node size: 40 bytes (key, colour, three 8-byte pointers)

== (a) checker and sorted-slice oracle after EVERY operation ==
sorted 1..2000: 2000 inserts, 0 lookups
  mismatches vs oracle 0, invariant failures 0, final size 2000, height 19, black height 10
  rotations per insert: 0 x 19, 1 x 1981, 2 x 0; max 1 (theorem: at most 2, OK)
random permutation of 1..2000: 2000 inserts, 0 lookups
  mismatches vs oracle 0, invariant failures 0, final size 2000, height 14, black height 7
  rotations per insert: 0 x 1245, 1 x 363, 2 x 392; max 2 (theorem: at most 2, OK)
random ops on keys 0..999 (70% insert, 30% lookup): 6953 inserts, 3047 lookups
  mismatches vs oracle 0, invariant failures 0, final size 998, height 12, black height 6
  rotations per insert: 0 x 6566, 1 x 206, 2 x 181; max 2 (theorem: at most 2, OK)

== (b) height against the bound 2*lg(n+1), sorted vs random insertion ==
      n order  height  bh  ideal  bound ok   rotations  rots/ins  recolours  max rot  checker
   1000 sorted     17   9     10  19.93 OK         983     0.983       4909        1  ok
   1000 random     13   7     10  19.93 OK         568     0.568       2325        2  ok
 100000 sorted     31  16     17  33.22 OK       99969     1.000     499846        1  ok
 100000 random     21  11     17  33.22 OK       58167     0.582     231692        2  ok
sorted rows: inserts that did not rotate = 17 and 31, final heights = 17 and 31

== (c) planted bug: case 1 recolours but forgets to move z up to the grandparent ==
  sorted 1..2000: the checker fired at op 8 (insert 8): red-red: red key 4 has red child 6
  random permutation of 1..2000: the checker fired at op 12 (insert 1989): red-red: red key 1790 has red child 1873
```

**Rust:**

```rust
use std::cmp::max;
use std::mem::size_of;
use std::ops::{Index, IndexMut};

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

// A red-black tree, insertion only, written line for line from CLRS 13.3.
// Every node has a parent link, which `Box` cannot express (a child would own its
// parent, and the parent owns the child), so the nodes live in a `Vec` arena and
// link to each other by `u32` index. Index 0 is the shared black sentinel that
// stands in for every nil leaf, so "the uncle is black" never needs a nil check.
const NIL: u32 = 0;

#[derive(Clone, Copy, PartialEq)]
enum Color {
    Black,
    Red,
}

struct RbNode {
    key: u64,
    left: u32,
    right: u32,
    parent: u32,
    color: Color,
}

// Stats: rots counts rotations, recolours counts nodes whose colour actually
// changed, cmps counts nodes compared against while searching for the slot.
#[derive(Default)]
struct Stats {
    rots: usize,
    recolours: usize,
    cmps: usize,
    skip_advance: bool, // part (c): the planted bug, see fixup
}

struct Tree {
    nodes: Vec<RbNode>,
    root: u32,
    st: Stats,
}

// `t[x]` reads like Go's `x.` field access: index the arena with a u32 id.
impl Index<u32> for Tree {
    type Output = RbNode;
    fn index(&self, i: u32) -> &RbNode {
        &self.nodes[i as usize]
    }
}

impl IndexMut<u32> for Tree {
    fn index_mut(&mut self, i: u32) -> &mut RbNode {
        &mut self.nodes[i as usize]
    }
}

impl Tree {
    fn new() -> Self {
        let sentinel = RbNode {
            key: 0,
            left: NIL,
            right: NIL,
            parent: NIL,
            color: Color::Black,
        };
        Tree {
            nodes: vec![sentinel],
            root: NIL,
            st: Stats::default(),
        }
    }

    fn paint(&mut self, n: u32, c: Color) {
        if self[n].color != c {
            self[n].color = c;
            self.st.recolours += 1;
        }
    }

    fn rotate_left(&mut self, x: u32) {
        let y = self[x].right;
        self[x].right = self[y].left; // y's left subtree becomes x's right subtree
        if self[y].left != NIL {
            let yl = self[y].left;
            self[yl].parent = x;
        }
        self[y].parent = self[x].parent;
        let p = self[x].parent;
        if p == NIL {
            self.root = y;
        } else if x == self[p].left {
            self[p].left = y;
        } else {
            self[p].right = y;
        }
        self[y].left = x;
        self[x].parent = y;
        self.st.rots += 1;
    }

    fn rotate_right(&mut self, y: u32) {
        let x = self[y].left;
        self[y].left = self[x].right;
        if self[x].right != NIL {
            let xr = self[x].right;
            self[xr].parent = y;
        }
        self[x].parent = self[y].parent;
        let p = self[y].parent;
        if p == NIL {
            self.root = x;
        } else if y == self[p].right {
            self[p].right = x;
        } else {
            self[p].left = x;
        }
        self[x].right = y;
        self[y].parent = x;
        self.st.rots += 1;
    }

    fn has(&self, key: u64) -> bool {
        let mut x = self.root;
        while x != NIL && self[x].key != key {
            x = if key < self[x].key {
                self[x].left
            } else {
                self[x].right
            };
        }
        x != NIL
    }

    // insert is a plain BST insert of a RED node followed by fixup. A duplicate is a no-op.
    fn insert(&mut self, key: u64) -> bool {
        let (mut y, mut x) = (NIL, self.root);
        while x != NIL {
            y = x;
            self.st.cmps += 1;
            if key < self[x].key {
                x = self[x].left;
            } else if key > self[x].key {
                x = self[x].right;
            } else {
                return false;
            }
        }
        let z = self.nodes.len() as u32; // the new node's index is its slot in the arena
        self.nodes.push(RbNode {
            key,
            left: NIL,
            right: NIL,
            parent: y,
            color: Color::Red,
        });
        if y == NIL {
            self.root = z;
        } else if key < self[y].key {
            self[y].left = z;
        } else {
            self[y].right = z;
        }
        self.fixup(z);
        true
    }

    // fixup restores the rules. The only rule a red z can break is "no red child under a red parent".
    fn fixup(&mut self, mut z: u32) {
        while self[self[z].parent].color == Color::Red {
            let p = self[z].parent;
            let g = self[p].parent; // p is red, so it is not the root, so g is a real node
            if p == self[g].left {
                let u = self[g].right;
                if self[u].color == Color::Red {
                    // case 1: red uncle: recolour, trouble moves up two levels
                    self.paint(p, Color::Black);
                    self.paint(u, Color::Black);
                    self.paint(g, Color::Red);
                    if !self.st.skip_advance {
                        z = g;
                    }
                } else {
                    if z == self[p].right {
                        // case 2: zig-zag, straighten it into case 3
                        z = p;
                        self.rotate_left(z);
                    }
                    let p = self[z].parent; // case 3: black uncle: one rotation ends the loop
                    self.paint(p, Color::Black);
                    self.paint(g, Color::Red);
                    self.rotate_right(g);
                }
            } else {
                // the mirror image
                let u = self[g].left;
                if self[u].color == Color::Red {
                    self.paint(p, Color::Black);
                    self.paint(u, Color::Black);
                    self.paint(g, Color::Red);
                    if !self.st.skip_advance {
                        z = g;
                    }
                } else {
                    if z == self[p].left {
                        z = p;
                        self.rotate_right(z);
                    }
                    let p = self[z].parent;
                    self.paint(p, Color::Black);
                    self.paint(g, Color::Red);
                    self.rotate_left(g);
                }
            }
        }
        let r = self.root;
        self.paint(r, Color::Black);
    }

    fn height(&self, n: u32) -> usize {
        if n == NIL {
            return 0;
        }
        1 + max(self.height(self[n].left), self.height(self[n].right))
    }

    // walk checks node-local rules top-down, then returns the black height below n
    // (black nodes on any path to a nil leaf, nil leaves not counted).
    fn walk(&self, n: u32, lo: Option<u64>, hi: Option<u64>) -> Result<usize, String> {
        if n == NIL {
            return Ok(0);
        }
        let node = &self[n];
        if lo.is_some_and(|l| node.key <= l) || hi.is_some_and(|h| node.key >= h) {
            return Err(format!("order: key {} breaks BST order", node.key));
        }
        for c in [node.left, node.right] {
            if c == NIL {
                continue;
            }
            if self[c].parent != n {
                return Err(format!(
                    "parent-link: key {} does not point back to its parent {}",
                    self[c].key, node.key
                ));
            }
            if node.color == Color::Red && self[c].color == Color::Red {
                return Err(format!(
                    "red-red: red key {} has red child {}",
                    node.key, self[c].key
                ));
            }
        }
        let bl = self.walk(node.left, lo, Some(node.key))?;
        let br = self.walk(node.right, Some(node.key), hi)?;
        if bl != br {
            return Err(format!(
                "black-height: key {} has black height {bl} on its left, {br} on its right",
                node.key
            ));
        }
        Ok(bl + usize::from(node.color == Color::Black))
    }

    fn inorder(&self, n: u32, out: &mut Vec<u64>) {
        if n != NIL {
            self.inorder(self[n].left, out);
            out.push(self[n].key);
            self.inorder(self[n].right, out);
        }
    }

    // verify is the invariant checker: all five rules, parent links, and the in-order
    // keys equal to the sorted-slice oracle. It returns the black height.
    fn verify(&self, oracle: &[u64], buf: &mut Vec<u64>) -> Result<usize, String> {
        if self[self.root].color == Color::Red {
            return Err(format!("root-red: root key {} is red", self[self.root].key));
        }
        let bh = self.walk(self.root, None, None)?;
        buf.clear();
        self.inorder(self.root, buf);
        if buf.as_slice() != oracle {
            return Err(format!(
                "oracle: tree holds {} keys, oracle holds {} (or they differ)",
                buf.len(),
                oracle.len()
            ));
        }
        Ok(bh)
    }
}

struct Op {
    key: u64,
    lookup: bool,
}

fn inserts(keys: &[u64]) -> Vec<Op> {
    keys.iter().map(|&key| Op { key, lookup: false }).collect()
}

// prove runs the ops against the tree and a sorted-slice oracle with the checker after
// every one. With stop_at_fail it stops at the first violation and reports it.
fn prove(label: &str, ops: &[Op], plant: bool, stop_at_fail: bool) {
    let mut t = Tree::new();
    t.st.skip_advance = plant;
    let mut oracle: Vec<u64> = Vec::new();
    let mut buf: Vec<u64> = Vec::with_capacity(ops.len());
    let mut hist = [0usize; 8];
    let (mut max_rots, mut mismatches, mut failures, mut lookups, mut bh) = (0, 0, 0, 0, 0);
    for (i, o) in ops.iter().enumerate() {
        let pos = oracle.partition_point(|&x| x < o.key);
        let found = oracle.get(pos) == Some(&o.key);
        if o.lookup {
            lookups += 1;
            if t.has(o.key) != found {
                mismatches += 1;
            }
        } else {
            let before = t.st.rots;
            if t.insert(o.key) == found {
                mismatches += 1;
            }
            if !found {
                oracle.insert(pos, o.key);
            }
            hist[t.st.rots - before] += 1;
            max_rots = max(max_rots, t.st.rots - before);
        }
        match t.verify(&oracle, &mut buf) {
            Ok(b) => bh = b,
            Err(err) => {
                failures += 1;
                if stop_at_fail {
                    println!(
                        "  {label}: the checker fired at op {} (insert {}): {err}",
                        i + 1,
                        o.key
                    );
                    return;
                }
            }
        }
    }
    println!(
        "{label}: {} inserts, {lookups} lookups",
        ops.len() - lookups
    );
    println!(
        "  mismatches vs oracle {mismatches}, invariant failures {failures}, final size {}, height {}, black height {bh}",
        oracle.len(),
        t.height(t.root)
    );
    println!(
        "  rotations per insert: 0 x {}, 1 x {}, 2 x {}; max {max_rots} (theorem: at most 2, {})",
        hist[0],
        hist[1],
        hist[2],
        verdict(max_rots <= 2)
    );
}

fn verdict(ok: bool) -> &'static str {
    if ok { "OK" } else { "VIOLATED" }
}

fn seq(n: usize) -> Vec<u64> {
    (1..=n as u64).collect()
}

fn shuffled(n: usize, rng: &mut Pcg) -> Vec<u64> {
    let mut a = seq(n);
    for i in (1..n).rev() {
        // Fisher-Yates by hand, so Go and Rust draw identically
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
    a
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    println!(
        "node size: {} bytes (key, colour, three 4-byte indices; Go's node is 40 with 8-byte pointers)",
        size_of::<RbNode>()
    );

    println!("\n== (a) checker and sorted-slice oracle after EVERY operation ==");
    let n = 2_000;
    let sorted_keys = seq(n);
    let rand_keys = shuffled(n, &mut rng);
    prove("sorted 1..2000", &inserts(&sorted_keys), false, false);
    prove(
        "random permutation of 1..2000",
        &inserts(&rand_keys),
        false,
        false,
    );
    let mixed: Vec<Op> = (0..10_000)
        .map(|_| {
            let r = rng.int_n(100);
            Op {
                key: rng.int_n(1000),
                lookup: r >= 70,
            }
        })
        .collect();
    prove(
        "random ops on keys 0..999 (70% insert, 30% lookup)",
        &mixed,
        false,
        false,
    );

    println!("\n== (b) height against the bound 2*lg(n+1), sorted vs random insertion ==");
    println!(
        "{:>7} {:<6} {:>6} {:>3} {:>6} {:>6} {:<4} {:>9} {:>9} {:>10} {:>8}  checker",
        "n",
        "order",
        "height",
        "bh",
        "ideal",
        "bound",
        "ok",
        "rotations",
        "rots/ins",
        "recolours",
        "max rot"
    );
    let mut plain: Vec<usize> = Vec::new(); // sorted rows: n - rotations, then the final height
    for n in [1_000usize, 100_000] {
        let sorted = seq(n);
        for (name, keys) in [
            ("sorted", sorted.clone()),
            ("random", shuffled(n, &mut rng)),
        ] {
            let mut t = Tree::new();
            let mut max_rot = 0;
            for &k in &keys {
                let before = t.st.rots;
                t.insert(k);
                max_rot = max(max_rot, t.st.rots - before);
            }
            let h = t.height(t.root);
            let bound = 2.0 * ((n + 1) as f64).log2();
            let (bh, chk) = match t.verify(&sorted, &mut Vec::with_capacity(n)) {
                Ok(bh) => (bh, "ok".to_string()),
                Err(e) => (0, e),
            };
            println!(
                "{:>7} {:<6} {:>6} {:>3} {:>6} {:>6.2} {:<4} {:>9} {:>9.3} {:>10} {:>8}  {}",
                n,
                name,
                h,
                bh,
                usize::BITS - n.leading_zeros(),
                bound,
                verdict(h as f64 <= bound),
                t.st.rots,
                t.st.rots as f64 / n as f64,
                t.st.recolours,
                max_rot,
                chk
            );
            if name == "sorted" {
                plain.extend([n - t.st.rots, h]);
            }
        }
    }
    println!(
        "sorted rows: inserts that did not rotate = {} and {}, final heights = {} and {}",
        plain[0], plain[2], plain[1], plain[3]
    );

    println!(
        "\n== (c) planted bug: case 1 recolours but forgets to move z up to the grandparent =="
    );
    prove("sorted 1..2000", &inserts(&sorted_keys), true, true);
    prove(
        "random permutation of 1..2000",
        &inserts(&rand_keys),
        true,
        true,
    );
}
```

**Rust output:**

```
node size: 24 bytes (key, colour, three 4-byte indices; Go's node is 40 with 8-byte pointers)

== (a) checker and sorted-slice oracle after EVERY operation ==
sorted 1..2000: 2000 inserts, 0 lookups
  mismatches vs oracle 0, invariant failures 0, final size 2000, height 19, black height 10
  rotations per insert: 0 x 19, 1 x 1981, 2 x 0; max 1 (theorem: at most 2, OK)
random permutation of 1..2000: 2000 inserts, 0 lookups
  mismatches vs oracle 0, invariant failures 0, final size 2000, height 14, black height 7
  rotations per insert: 0 x 1245, 1 x 363, 2 x 392; max 2 (theorem: at most 2, OK)
random ops on keys 0..999 (70% insert, 30% lookup): 6953 inserts, 3047 lookups
  mismatches vs oracle 0, invariant failures 0, final size 998, height 12, black height 6
  rotations per insert: 0 x 6566, 1 x 206, 2 x 181; max 2 (theorem: at most 2, OK)

== (b) height against the bound 2*lg(n+1), sorted vs random insertion ==
      n order  height  bh  ideal  bound ok   rotations  rots/ins  recolours  max rot  checker
   1000 sorted     17   9     10  19.93 OK         983     0.983       4909        1  ok
   1000 random     13   7     10  19.93 OK         568     0.568       2325        2  ok
 100000 sorted     31  16     17  33.22 OK       99969     1.000     499846        1  ok
 100000 random     21  11     17  33.22 OK       58167     0.582     231692        2  ok
sorted rows: inserts that did not rotate = 17 and 31, final heights = 17 and 31

== (c) planted bug: case 1 recolours but forgets to move z up to the grandparent ==
  sorted 1..2000: the checker fired at op 8 (insert 8): red-red: red key 4 has red child 6
  random permutation of 1..2000: the checker fired at op 12 (insert 1989): red-red: red key 1790 has red child 1873
```

> **Go vs Rust:** Output is identical to Go except the first line, the node size (24 bytes against Go's 40): every count, height, black height, the three proof runs and both planted-bug reports (op 8 and op 12, rule `red-red`) match (diffed). The arena is forced by the algorithm: the fix-up reads `z.parent.parent` and walks upward, and a `Box` tree has no way to hold a parent link (the child would own its owner), so nodes live in a `Vec` and refer to each other by `u32` index with 0 as the black sentinel, and `impl Index<u32> for Tree` lets `self[z].parent` read like Go's `z.parent`. The 4-byte indices are why `size_of::<RbNode>()` is 24 (8 key + 3 x 4 + 1 colour byte, padded) where Go's pointer node is 40.

**Complexity:** insert Θ(log n), **at most 2 rotations** per insert (the maximum was 1 for sorted input and 2 for random input) plus recolouring (about 5.0 recolourings per insert for 10⁵ sorted keys, 2.3 for random) · height ≤ 2·lg(n+1) and visibly looser than AVL's: 10⁵ sorted keys give height 31 against the bound 33.22 and the ideal 17 (example 7's AVL tree has exactly 17), random input gives 21 · cheaper repair: 0.58 rotations per random insert against AVL's 0.70 at the same n · the checker passed after every one of 2,000 + 2,000 + 10,000 operations, and the planted bug is caught by the `red-red` rule, at op 8 (sorted) and op 12 (random).

---

## 9. Treap: priorities instead of balance factors

`🟡 medium` · *Randomised BST*

A treap gives every key a random priority and keeps the tree a BST by key and a min-heap by priority, so no balance factor, colour or rebalancing rule exists at all. The pair of orders makes the tree **unique**: it is the BST you would get by inserting the keys in priority order, so the arrival order of the keys cannot change its shape, only the work done to reach it, and that is the experiment. Insert is a BST insert followed by rotations that lift the new node while it outranks its parent; delete rotates the node down toward the smaller-priority child until it can be spliced out. Depth is Θ(log n) *in expectation* for every key set (sorted input included), a guarantee that holds only while the priorities are random, and Go's own runtime uses exactly this structure in `runtime/sema.go`.

**Steps:**

1. Draw one priority per key, in key order, then build the same keys with the same priorities in three orders (sorted, reversed, shuffled) at n = 1,000 and 100,000 and compare the final shapes (pre-order key lists) and the work (rotations, comparisons).
2. The key move: redraw the priorities 200 times to see the spread of height and mean depth, run 100,000 random inserts, deletes and lookups against a sorted-slice oracle with an order-plus-heap checker after every operation, and finally feed sorted keys to a treap whose priorities are random and to one whose priority equals the key.
3. Read the equality line first (one shape, different work), then the heights against the random-BST expectation, then the last pair of rows, where the degenerate tree appears only when the priorities are chosen by someone who can see them.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// A treap: a BST by key and a MIN-heap by a random priority, the same orientation
// Go's own runtime uses in runtime/sema.go (s.ticket <= both children's tickets).
// Given the keys and their priorities the shape is unique: it is the tree you get by
// inserting the keys in priority order, whatever order they really arrive in.
type node struct {
	key         int
	pri         uint64
	left, right *node
}

// Stats counts nodes compared against and rotations performed.
type Stats struct{ cmps, rots int }

func rotRight(y *node, st *Stats) *node { // y's left child x becomes the subtree root
	x := y.left
	y.left = x.right
	x.right = y
	st.rots++
	return x
}

func rotLeft(x *node, st *Stats) *node {
	y := x.right
	x.right = y.left
	y.left = x
	st.rots++
	return y
}

// insert is a BST insert; on the way back up, a child whose priority is smaller than
// its parent's is rotated above it, which is "rotate the new node up while it outranks its parent".
func insert(n *node, key int, pri uint64, st *Stats) (*node, bool) {
	if n == nil {
		return &node{key: key, pri: pri}, true
	}
	st.cmps++
	var ok bool
	switch {
	case key < n.key:
		if n.left, ok = insert(n.left, key, pri, st); ok && n.left.pri < n.pri {
			n = rotRight(n, st)
		}
	case key > n.key:
		if n.right, ok = insert(n.right, key, pri, st); ok && n.right.pri < n.pri {
			n = rotLeft(n, st)
		}
	default:
		return n, false
	}
	return n, ok
}

// dropRoot removes the root of n's subtree: rotate it down toward the child with the smaller
// priority (that child must become the new root to keep heap order) until it has at most
// one child, then splice it out.
func dropRoot(n *node, st *Stats) *node {
	switch {
	case n.left == nil:
		return n.right
	case n.right == nil:
		return n.left
	case n.left.pri < n.right.pri:
		n = rotRight(n, st)
		n.right = dropRoot(n.right, st)
	default:
		n = rotLeft(n, st)
		n.left = dropRoot(n.left, st)
	}
	return n
}

func remove(n *node, key int, st *Stats) (*node, bool) {
	if n == nil {
		return nil, false
	}
	st.cmps++
	var ok bool
	switch {
	case key < n.key:
		n.left, ok = remove(n.left, key, st)
	case key > n.key:
		n.right, ok = remove(n.right, key, st)
	default:
		return dropRoot(n, st), true
	}
	return n, ok
}

func contains(n *node, key int) bool {
	for n != nil && n.key != key {
		if key < n.key {
			n = n.left
		} else {
			n = n.right
		}
	}
	return n != nil
}

// inorder walks with an explicit stack: a degenerate treap (part d) is thousands of levels deep.
func inorder(root *node, visit func(*node) error) error {
	var stack []*node
	for n := root; n != nil || len(stack) > 0; {
		for ; n != nil; n = n.left {
			stack = append(stack, n)
		}
		n = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if err := visit(n); err != nil {
			return err
		}
		n = n.right
	}
	return nil
}

// verify is the invariant checker: in-order keys equal the oracle (BST order and size)
// and every node's priority is <= both children's (heap order).
func verify(root *node, oracle []int) error {
	i := 0
	err := inorder(root, func(n *node) error {
		if i >= len(oracle) || n.key != oracle[i] {
			return fmt.Errorf("in-order key #%d is %d, oracle disagrees (oracle holds %d keys)", i, n.key, len(oracle))
		}
		i++
		for _, c := range []*node{n.left, n.right} {
			if c != nil && c.pri < n.pri {
				return fmt.Errorf("heap order: key %d outranks its parent %d", c.key, n.key)
			}
		}
		return nil
	})
	if err == nil && i != len(oracle) {
		err = fmt.Errorf("tree holds %d keys, oracle holds %d", i, len(oracle))
	}
	return err
}

// shape returns the height (nodes on the longest path) and the sum of node depths (root = 0).
func shape(root *node) (height, depthSum int) {
	type item struct {
		n *node
		d int
	}
	stack := []item{{root, 0}}
	for len(stack) > 0 {
		it := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if it.n == nil {
			continue
		}
		height = max(height, it.d+1)
		depthSum += it.d
		stack = append(stack, item{it.n.left, it.d + 1}, item{it.n.right, it.d + 1})
	}
	return height, depthSum
}

func preorder(root *node) []int {
	var out []int
	nodes := []*node{root}
	for len(nodes) > 0 {
		n := nodes[len(nodes)-1]
		nodes = nodes[:len(nodes)-1]
		if n == nil {
			continue
		}
		out = append(out, n.key)
		nodes = append(nodes, n.right, n.left)
	}
	return out
}

// spine counts the nodes on the rightmost (or leftmost) path from the root.
func spine(root *node, right bool) int {
	c := 0
	for n := root; n != nil; c++ {
		if right {
			n = n.right
		} else {
			n = n.left
		}
	}
	return c
}

func seq(n int) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = i + 1
	}
	return a
}

func shuffled(n int, rng *rand.Rand) []int {
	a := seq(n)
	for i := n - 1; i >= 1; i-- { // Fisher-Yates by hand, so Go and Rust draw identically
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
	return a
}

// priorities draws one priority per key 1..n, in key order, before any shuffle.
func priorities(n int, rng *rand.Rand) []uint64 {
	pri := make([]uint64, n+1)
	for k := 1; k <= n; k++ {
		pri[k] = rng.Uint64()
	}
	return pri
}

func build(keys []int, pri []uint64) (*node, Stats) {
	var root *node
	var st Stats
	for _, k := range keys {
		root, _ = insert(root, k, pri[k], &st)
	}
	return root, st
}

func verdict(ok bool) string {
	if ok {
		return "OK"
	}
	return "VIOLATED"
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("== (a) same keys, same priorities, three arrival orders ==")
	allSame, workDiffers := true, true
	for _, n := range []int{1_000, 100_000} {
		pri := priorities(n, rng)
		sorted := seq(n)
		reversed := slices.Clone(sorted)
		slices.Reverse(reversed)
		fmt.Printf("n = %d\n%-9s %10s %9s %7s %12s %11s  %s\n", n, "order", "rotations", "rots/ins", "height", "comparisons", "same shape", "checker")
		var ref []int
		var sortedRots int
		for _, in := range []struct {
			name string
			keys []int
		}{{"sorted", sorted}, {"reversed", reversed}, {"shuffled", shuffled(n, rng)}} {
			root, st := build(in.keys, pri)
			h, _ := shape(root)
			shapeKeys := preorder(root)
			if ref == nil {
				ref = shapeKeys
			}
			chk := "ok"
			if err := verify(root, sorted); err != nil {
				chk = err.Error()
			}
			same := slices.Equal(shapeKeys, ref) // the first row is the reference, so it is trivially true
			allSame = allSame && same
			if in.name == "sorted" {
				sortedRots = st.rots
			} else if in.name == "shuffled" {
				workDiffers = workDiffers && st.rots != sortedRots
			}
			fmt.Printf("%-9s %10d %9.3f %7d %12d %11v  %s\n", in.name, st.rots, float64(st.rots)/float64(n), h, st.cmps, same, chk)
			if in.name != "shuffled" {
				// Every rotation pushes exactly one node off the spine the new keys arrive along,
				// and every node joins that spine once: rotations = n - (final spine length).
				side := map[string]string{"sorted": "right", "reversed": "left"}[in.name]
				sp := spine(root, in.name == "sorted")
				fmt.Printf("%9s rotations == n - %s spine length? %v (%d vs %d - %d)\n", "", side, st.rots == n-sp, st.rots, n, sp)
			}
		}
	}
	fmt.Printf("all six shapes equal their sorted-order shape: %v; the shuffled rotation total differs from the sorted one: %v\n", allSame, workDiffers)
	if allSame && workDiffers {
		fmt.Println("the shape does not depend on the arrival order; the work (rotations, comparisons) does")
	}

	fmt.Println("\n== (b) 200 independent priority assignments, n = 1,000 ==")
	const n, trials = 1_000, 200
	heights := make([]int, trials)
	sumMean := 0.0
	for t := range trials {
		root, _ := build(seq(n), priorities(n, rng)) // arrival order is irrelevant to the shape
		h, ds := shape(root)
		heights[t] = h
		sumMean += float64(ds) / n
	}
	slices.Sort(heights)
	harmonic := 0.0
	for i := 1; i <= n; i++ {
		harmonic += 1 / float64(i)
	}
	fmt.Printf("height: min %d, median %d, max %d (ideal %d)\n", heights[0], heights[trials/2], heights[trials-1], bits.Len(uint(n)))
	fmt.Printf("mean node depth over %d treaps: %.3f; random-BST expectation 2(1+1/n)H_n - 4 = %.3f; 2 ln n = %.3f\n",
		trials, sumMean/trials, 2*(1+1.0/n)*harmonic-4, 2*math.Log(n))

	fmt.Println("\n== (c) 100,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op ==")
	crng := rand.New(rand.NewPCG(1, 2))
	cpri := make([]uint64, 500)
	for k := range cpri {
		cpri[k] = crng.Uint64()
	}
	var root *node
	var st Stats
	oracle := []int{}
	var mismatches, failures, maxH, ins, del, insRots, delRots int
	for op := 1; op <= 100_000; op++ {
		r, key := crng.IntN(100), crng.IntN(500)
		pos, found := slices.BinarySearch(oracle, key)
		var got, want bool
		before := st.rots
		switch {
		case r < 45:
			root, got = insert(root, key, cpri[key], &st)
			want = !found
			if want {
				oracle = slices.Insert(oracle, pos, key)
				ins++
				insRots += st.rots - before
			}
		case r < 80:
			root, got = remove(root, key, &st)
			want = found
			if found {
				oracle = slices.Delete(oracle, pos, pos+1)
				del++
				delRots += st.rots - before
			}
		default:
			got, want = contains(root, key), found
		}
		if got != want {
			mismatches++
		}
		if err := verify(root, oracle); err != nil {
			failures++
		}
		h, _ := shape(root)
		maxH = max(maxH, h)
	}
	fmt.Printf("  mismatches vs oracle %d, invariant failures %d, final size %d, max height %d\n", mismatches, failures, len(oracle), maxH)
	fmt.Printf("  rotations: %.3f per effective insert (%d inserts), %.3f per effective delete (%d deletes)\n",
		float64(insRots)/float64(ins), ins, float64(delRots)/float64(del), del)
	fmt.Println("  theory (Aragon and Seidel): the EXPECTED rotations per update is below 2; one run's mean can land on either side of a bound")

	fmt.Println("\n== (d) sorted insertion, n = 5,000: random priorities vs priority = key ==")
	const m = 5_000
	keys := seq(m)
	adversary := make([]uint64, m+1)
	for k := range adversary {
		adversary[k] = uint64(k) // someone who sees the priorities and makes them ascending
	}
	var cmps [2]int
	for i, c := range []struct {
		name string
		pri  []uint64
	}{{"random priorities", priorities(m, rng)}, {"priority = key", adversary}} {
		root, st := build(keys, c.pri)
		cmps[i] = st.cmps
		h, _ := shape(root)
		chk := "ok"
		if err := verify(root, keys); err != nil {
			chk = err.Error()
		}
		fmt.Printf("  %-18s height %5d, comparisons %9d, rotations %5d, checker %s\n", c.name, h, st.cmps, st.rots, chk)
	}
	fmt.Printf("  ideal height %d; a plain BST fed sorted input costs n(n-1)/2 = %d comparisons; the adversary's build costs %.0fx the random one\n",
		bits.Len(uint(m)), m*(m-1)/2, float64(cmps[1])/float64(cmps[0]))
}
```

**Output:**

```
== (a) same keys, same priorities, three arrival orders ==
n = 1000
order      rotations  rots/ins  height  comparisons  same shape  checker
sorted           992     0.992      21         6269        true  ok
          rotations == n - right spine length? true (992 vs 1000 - 8)
reversed         992     0.992      21         6324        true  ok
          rotations == n - left spine length? true (992 vs 1000 - 8)
shuffled        1804     1.804      21        10516        true  ok
n = 100000
order      rotations  rots/ins  height  comparisons  same shape  checker
sorted         99984     1.000      39      1053804        true  ok
          rotations == n - right spine length? true (99984 vs 100000 - 16)
reversed       99986     1.000      39      1116231        true  ok
          rotations == n - left spine length? true (99986 vs 100000 - 14)
shuffled      200107     2.001      39      2005297        true  ok
all six shapes equal their sorted-order shape: true; the shuffled rotation total differs from the sorted one: true
the shape does not depend on the arrival order; the work (rotations, comparisons) does

== (b) 200 independent priority assignments, n = 1,000 ==
height: min 18, median 22, max 28 (ideal 10)
mean node depth over 200 treaps: 10.995; random-BST expectation 2(1+1/n)H_n - 4 = 10.986; 2 ln n = 13.816

== (c) 100,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op ==
  mismatches vs oracle 0, invariant failures 0, final size 282, max height 25
  rotations: 1.953 per effective insert (19862 inserts), 0.947 per effective delete (19580 deletes)
  theory (Aragon and Seidel): the EXPECTED rotations per update is below 2; one run's mean can land on either side of a bound

== (d) sorted insertion, n = 5,000: random priorities vs priority = key ==
  random priorities  height    30, comparisons     43296, rotations  4984, checker ok
  priority = key     height  5000, comparisons  12497500, rotations     0, checker ok
  ideal height 13; a plain BST fed sorted input costs n(n-1)/2 = 12497500 comparisons; the adversary's build costs 289x the random one
```

**Rust:**

```rust
use std::cmp::max;

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

// A treap: a BST by key and a MIN-heap by a random priority, the same orientation
// Go's own runtime uses in runtime/sema.go (s.ticket <= both children's tickets).
// Given the keys and their priorities the shape is unique: it is the tree you get by
// inserting the keys in priority order, whatever order they really arrive in.
// Rust's std has no treap; like AVL, the one-owner `Option<Box<Node>>` + `take()` shape fits.
type Link = Option<Box<Node>>;

struct Node {
    key: u64,
    pri: u64,
    left: Link,
    right: Link,
}

// An iterative Drop, as in lesson 05 example 15. Treaps are shallow with overwhelming
// probability, but part (d) builds a deliberately degenerate 5,000-node chain on the
// same type, and the default drop glue would recurse once per level. (With a Drop impl
// the code below must `take()` children instead of moving them out of the Box.)
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

// Stats counts nodes compared against and rotations performed.
#[derive(Default)]
struct Stats {
    cmps: usize,
    rots: usize,
}

// y's left child x becomes the subtree root; the caller must store the returned root.
// With no left child the node comes back unchanged (total: no panic).
fn rot_right(mut y: Box<Node>, st: &mut Stats) -> Box<Node> {
    let Some(mut x) = y.left.take() else {
        return y;
    };
    y.left = x.right.take();
    x.right = Some(y);
    st.rots += 1;
    x
}

fn rot_left(mut x: Box<Node>, st: &mut Stats) -> Box<Node> {
    let Some(mut y) = x.right.take() else {
        return x;
    };
    x.right = y.left.take();
    y.left = Some(x);
    st.rots += 1;
    y
}

// insert is a BST insert; on the way back up, a child whose priority is smaller than
// its parent's is rotated above it, which is "rotate the new node up while it outranks its parent".
fn insert(link: Link, key: u64, pri: u64, st: &mut Stats) -> (Link, bool) {
    let Some(mut n) = link else {
        let leaf = Node {
            key,
            pri,
            left: None,
            right: None,
        };
        return (Some(Box::new(leaf)), true);
    };
    st.cmps += 1;
    let ok;
    if key < n.key {
        (n.left, ok) = insert(n.left.take(), key, pri, st);
        if ok && n.left.as_ref().is_some_and(|c| c.pri < n.pri) {
            n = rot_right(n, st);
        }
    } else if key > n.key {
        (n.right, ok) = insert(n.right.take(), key, pri, st);
        if ok && n.right.as_ref().is_some_and(|c| c.pri < n.pri) {
            n = rot_left(n, st);
        }
    } else {
        return (Some(n), false);
    }
    (Some(n), ok)
}

// drop_root removes the root of n's subtree: rotate it down toward the child with the smaller
// priority (that child must become the new root to keep heap order) until it has at most
// one child, then splice it out.
fn drop_root(mut n: Box<Node>, st: &mut Stats) -> Link {
    let lp = n.left.as_ref().map(|c| c.pri);
    let rp = n.right.as_ref().map(|c| c.pri);
    match (lp, rp) {
        (None, _) => n.right.take(),
        (_, None) => n.left.take(),
        (Some(l), Some(r)) => {
            if l < r {
                n = rot_right(n, st);
                n.right = n.right.take().and_then(|d| drop_root(d, st));
            } else {
                n = rot_left(n, st);
                n.left = n.left.take().and_then(|d| drop_root(d, st));
            }
            Some(n)
        }
    }
}

fn remove(link: Link, key: u64, st: &mut Stats) -> (Link, bool) {
    let Some(mut n) = link else {
        return (None, false);
    };
    st.cmps += 1;
    let ok;
    if key < n.key {
        (n.left, ok) = remove(n.left.take(), key, st);
    } else if key > n.key {
        (n.right, ok) = remove(n.right.take(), key, st);
    } else {
        return (drop_root(n, st), true);
    }
    (Some(n), ok)
}

fn contains(mut cur: &Link, key: u64) -> bool {
    while let Some(n) = cur {
        if key == n.key {
            return true;
        }
        cur = if key < n.key { &n.left } else { &n.right };
    }
    false
}

// inorder walks with an explicit stack: a degenerate treap (part d) is thousands of levels deep.
fn inorder(root: &Link, mut visit: impl FnMut(&Node) -> Result<(), String>) -> Result<(), String> {
    let mut stack: Vec<&Node> = Vec::new();
    let mut cur = root.as_deref();
    while cur.is_some() || !stack.is_empty() {
        while let Some(n) = cur {
            stack.push(n);
            cur = n.left.as_deref();
        }
        let Some(n) = stack.pop() else { break };
        visit(n)?;
        cur = n.right.as_deref();
    }
    Ok(())
}

// verify is the invariant checker: in-order keys equal the oracle (BST order and size)
// and every node's priority is <= both children's (heap order).
fn verify(root: &Link, oracle: &[u64]) -> Result<(), String> {
    let mut i = 0;
    inorder(root, |n| {
        if i >= oracle.len() || n.key != oracle[i] {
            return Err(format!(
                "in-order key #{i} is {}, oracle disagrees (oracle holds {} keys)",
                n.key,
                oracle.len()
            ));
        }
        i += 1;
        for c in [&n.left, &n.right].into_iter().flatten() {
            if c.pri < n.pri {
                return Err(format!(
                    "heap order: key {} outranks its parent {}",
                    c.key, n.key
                ));
            }
        }
        Ok(())
    })?;
    if i != oracle.len() {
        return Err(format!(
            "tree holds {i} keys, oracle holds {}",
            oracle.len()
        ));
    }
    Ok(())
}

// shape returns the height (nodes on the longest path) and the sum of node depths (root = 0).
fn shape(root: &Link) -> (usize, usize) {
    let (mut height, mut depth_sum) = (0, 0);
    let mut stack: Vec<(&Node, usize)> = root.as_deref().map(|n| (n, 0)).into_iter().collect();
    while let Some((n, d)) = stack.pop() {
        height = max(height, d + 1);
        depth_sum += d;
        for c in [&n.left, &n.right].into_iter().flatten() {
            stack.push((c, d + 1));
        }
    }
    (height, depth_sum)
}

fn preorder(root: &Link) -> Vec<u64> {
    let mut out = Vec::new();
    let mut stack: Vec<&Node> = root.as_deref().into_iter().collect();
    while let Some(n) = stack.pop() {
        out.push(n.key);
        stack.extend(n.right.as_deref());
        stack.extend(n.left.as_deref());
    }
    out
}

// spine counts the nodes on the rightmost (or leftmost) path from the root.
fn spine(root: &Link, right: bool) -> usize {
    let mut c = 0;
    let mut cur = root.as_deref();
    while let Some(n) = cur {
        c += 1;
        cur = if right {
            n.right.as_deref()
        } else {
            n.left.as_deref()
        };
    }
    c
}

fn seq(n: usize) -> Vec<u64> {
    (1..=n as u64).collect()
}

fn shuffled(n: usize, rng: &mut Pcg) -> Vec<u64> {
    let mut a = seq(n);
    for i in (1..n).rev() {
        // Fisher-Yates by hand, so Go and Rust draw identically
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
    a
}

// priorities draws one priority per key 1..n, in key order, before any shuffle.
fn priorities(n: usize, rng: &mut Pcg) -> Vec<u64> {
    let mut pri = vec![0u64; n + 1];
    for p in pri.iter_mut().skip(1) {
        *p = rng.next_u64();
    }
    pri
}

fn build(keys: &[u64], pri: &[u64]) -> (Link, Stats) {
    let mut root: Link = None;
    let mut st = Stats::default();
    for &k in keys {
        (root, _) = insert(root.take(), k, pri[k as usize], &mut st);
    }
    (root, st)
}

fn main() {
    let mut rng = Pcg::new(1, 2);

    println!("== (a) same keys, same priorities, three arrival orders ==");
    let (mut all_same, mut work_differs) = (true, true);
    for n in [1_000usize, 100_000] {
        let pri = priorities(n, &mut rng);
        let sorted = seq(n);
        let mut reversed = sorted.clone();
        reversed.reverse();
        println!(
            "n = {n}\n{:<9} {:>10} {:>9} {:>7} {:>12} {:>11}  checker",
            "order", "rotations", "rots/ins", "height", "comparisons", "same shape"
        );
        let mut reference: Option<Vec<u64>> = None;
        let mut sorted_rots = 0;
        let shuf = shuffled(n, &mut rng);
        for (name, keys) in [
            ("sorted", &sorted),
            ("reversed", &reversed),
            ("shuffled", &shuf),
        ] {
            let (root, st) = build(keys, &pri);
            let (h, _) = shape(&root);
            let shape_keys = preorder(&root);
            let same = *reference.get_or_insert_with(|| shape_keys.clone()) == shape_keys; // the first row is the reference, so it is trivially true
            all_same = all_same && same;
            if name == "sorted" {
                sorted_rots = st.rots;
            } else if name == "shuffled" {
                work_differs = work_differs && st.rots != sorted_rots;
            }
            let chk = match verify(&root, &sorted) {
                Ok(()) => "ok".to_string(),
                Err(e) => e,
            };
            println!(
                "{:<9} {:>10} {:>9.3} {:>7} {:>12} {:>11}  {}",
                name,
                st.rots,
                st.rots as f64 / n as f64,
                h,
                st.cmps,
                same,
                chk
            );
            if name != "shuffled" {
                // Every rotation pushes exactly one node off the spine the new keys arrive along,
                // and every node joins that spine once: rotations = n - (final spine length).
                let side = if name == "sorted" { "right" } else { "left" };
                let sp = spine(&root, name == "sorted");
                println!(
                    "{:>9} rotations == n - {side} spine length? {} ({} vs {n} - {sp})",
                    "",
                    st.rots == n - sp,
                    st.rots
                );
            }
        }
    }
    println!(
        "all six shapes equal their sorted-order shape: {all_same}; the shuffled rotation total differs from the sorted one: {work_differs}"
    );
    if all_same && work_differs {
        println!(
            "the shape does not depend on the arrival order; the work (rotations, comparisons) does"
        );
    }

    println!("\n== (b) 200 independent priority assignments, n = 1,000 ==");
    const N: usize = 1_000;
    const TRIALS: usize = 200;
    let mut heights = vec![0usize; TRIALS];
    let mut sum_mean = 0.0;
    for h_slot in heights.iter_mut() {
        let pri = priorities(N, &mut rng);
        let (root, _) = build(&seq(N), &pri); // arrival order is irrelevant to the shape
        let (h, ds) = shape(&root);
        *h_slot = h;
        sum_mean += ds as f64 / N as f64;
    }
    heights.sort();
    let mut harmonic = 0.0;
    for i in 1..=N {
        harmonic += 1.0 / i as f64;
    }
    println!(
        "height: min {}, median {}, max {} (ideal {})",
        heights[0],
        heights[TRIALS / 2],
        heights[TRIALS - 1],
        usize::BITS - N.leading_zeros()
    );
    println!(
        "mean node depth over {TRIALS} treaps: {:.3}; random-BST expectation 2(1+1/n)H_n - 4 = {:.3}; 2 ln n = {:.3}",
        sum_mean / TRIALS as f64,
        2.0 * (1.0 + 1.0 / N as f64) * harmonic - 4.0,
        2.0 * (N as f64).ln()
    );

    println!(
        "\n== (c) 100,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op =="
    );
    let mut crng = Pcg::new(1, 2);
    let cpri: Vec<u64> = (0..500).map(|_| crng.next_u64()).collect();
    let mut root: Link = None;
    let mut st = Stats::default();
    let mut oracle: Vec<u64> = Vec::new();
    let (mut mismatches, mut failures, mut max_h) = (0, 0, 0);
    let (mut ins, mut del, mut ins_rots, mut del_rots) = (0, 0, 0, 0);
    for _ in 1..=100_000 {
        let r = crng.int_n(100);
        let key = crng.int_n(500);
        let pos = oracle.partition_point(|&x| x < key);
        let found = oracle.get(pos) == Some(&key);
        let before = st.rots;
        let (got, want);
        if r < 45 {
            (root, got) = insert(root.take(), key, cpri[key as usize], &mut st);
            want = !found;
            if want {
                oracle.insert(pos, key);
                ins += 1;
                ins_rots += st.rots - before;
            }
        } else if r < 80 {
            (root, got) = remove(root.take(), key, &mut st);
            want = found;
            if found {
                oracle.remove(pos);
                del += 1;
                del_rots += st.rots - before;
            }
        } else {
            got = contains(&root, key);
            want = found;
        }
        if got != want {
            mismatches += 1;
        }
        if verify(&root, &oracle).is_err() {
            failures += 1;
        }
        max_h = max(max_h, shape(&root).0);
    }
    println!(
        "  mismatches vs oracle {mismatches}, invariant failures {failures}, final size {}, max height {max_h}",
        oracle.len()
    );
    println!(
        "  rotations: {:.3} per effective insert ({ins} inserts), {:.3} per effective delete ({del} deletes)",
        ins_rots as f64 / ins as f64,
        del_rots as f64 / del as f64
    );
    println!(
        "  theory (Aragon and Seidel): the EXPECTED rotations per update is below 2; one run's mean can land on either side of a bound"
    );

    println!("\n== (d) sorted insertion, n = 5,000: random priorities vs priority = key ==");
    const M: usize = 5_000;
    let keys = seq(M);
    // someone who sees the priorities and makes them ascending
    let adversary: Vec<u64> = (0..=M as u64).collect();
    let mut cmps = [0usize; 2];
    let random_pri = priorities(M, &mut rng);
    for (i, (name, pri)) in [
        ("random priorities", &random_pri),
        ("priority = key", &adversary),
    ]
    .into_iter()
    .enumerate()
    {
        let (root, st) = build(&keys, pri);
        cmps[i] = st.cmps;
        let (h, _) = shape(&root);
        let chk = match verify(&root, &keys) {
            Ok(()) => "ok".to_string(),
            Err(e) => e,
        };
        println!(
            "  {:<18} height {:>5}, comparisons {:>9}, rotations {:>5}, checker {}",
            name, h, st.cmps, st.rots, chk
        );
    }
    println!(
        "  ideal height {}; a plain BST fed sorted input costs n(n-1)/2 = {} comparisons; the adversary's build costs {:.0}x the random one",
        usize::BITS - M.leading_zeros(),
        M * (M - 1) / 2,
        cmps[1] as f64 / cmps[0] as f64
    );
}
```

**Rust output:**

```
== (a) same keys, same priorities, three arrival orders ==
n = 1000
order      rotations  rots/ins  height  comparisons  same shape  checker
sorted           992     0.992      21         6269        true  ok
          rotations == n - right spine length? true (992 vs 1000 - 8)
reversed         992     0.992      21         6324        true  ok
          rotations == n - left spine length? true (992 vs 1000 - 8)
shuffled        1804     1.804      21        10516        true  ok
n = 100000
order      rotations  rots/ins  height  comparisons  same shape  checker
sorted         99984     1.000      39      1053804        true  ok
          rotations == n - right spine length? true (99984 vs 100000 - 16)
reversed       99986     1.000      39      1116231        true  ok
          rotations == n - left spine length? true (99986 vs 100000 - 14)
shuffled      200107     2.001      39      2005297        true  ok
all six shapes equal their sorted-order shape: true; the shuffled rotation total differs from the sorted one: true
the shape does not depend on the arrival order; the work (rotations, comparisons) does

== (b) 200 independent priority assignments, n = 1,000 ==
height: min 18, median 22, max 28 (ideal 10)
mean node depth over 200 treaps: 10.995; random-BST expectation 2(1+1/n)H_n - 4 = 10.986; 2 ln n = 13.816

== (c) 100,000 random ops (45% insert, 35% delete, 20% lookup, keys 0..499), checker after every op ==
  mismatches vs oracle 0, invariant failures 0, final size 282, max height 25
  rotations: 1.953 per effective insert (19862 inserts), 0.947 per effective delete (19580 deletes)
  theory (Aragon and Seidel): the EXPECTED rotations per update is below 2; one run's mean can land on either side of a bound

== (d) sorted insertion, n = 5,000: random priorities vs priority = key ==
  random priorities  height    30, comparisons     43296, rotations  4984, checker ok
  priority = key     height  5000, comparisons  12497500, rotations     0, checker ok
  ideal height 13; a plain BST fed sorted input costs n(n-1)/2 = 12497500 comparisons; the adversary's build costs 289x the random one
```

> **Go vs Rust:** Output is identical to Go, all 31 lines (diffed): both draw the same priorities, shuffles and operations from the same PCG(1, 2) stream, so every rotation total, height, comparison count and the spine identity match. Rust's std has no treap, and the one-owner `Option<Box<Node>>` + `take()` shape of example 7 fits; the only extra is an iterative `Drop` (lesson 05 example 15), because part (d) builds a 5,000-deep chain on the same type, which also means the code must `take()` children instead of moving them out of a `Box<Node>` that has a destructor. The recursive `insert` still reaches 5,000 frames on that chain: in a separate scratch run (not part of this output) its stack span was about 560,000 bytes (roughly 112 per frame) of the 8 MB main stack, so it does not overflow, while `inorder`, `shape` and `preorder` use explicit stacks.

**Complexity:** expected depth Θ(log n) for every key set, sorted input included: 200 priority assignments at n = 1,000 gave heights 18 / 22 / 28 (min / median / max, ideal 10, so taller than AVL's 12 for the same size in example 7) and a mean node depth of 10.995 against the random-BST expectation 10.986 · the shape depends only on (key, priority): all six builds (three orders, n = 10³ and 10⁵) gave the same shape, while the work differed, at 0.99 to 1.00 rotations per insert for sorted input (exactly n minus the length of the final right spine) and 1.80 and 2.00 for shuffled input · expected rotations per update below 2 (Aragon and Seidel); the churn measured 1.95 per insert and 0.95 per delete · worst case Θ(n), reached here only by choosing the priorities: priority = key makes a 5,000-node path and 12,497,500 comparisons, 289x the random build's 43,296 · O(1) extra space, one 64-bit priority per node.

---

## 10. Skip list, built and measured in steps

`🟡 medium` · *Skip list*

A skip list gets the balance of a tree without a tree: it is a sorted linked list in which node i also sits on a geometric number of higher "express lanes" (a node reaches level k with probability pᵏ⁻¹), a search starts on the top lane and drops one lane whenever the next node would overshoot, and insert and delete just splice a node in or out of its lanes, so nothing is ever rebalanced and the shape does not depend on the order the keys arrive in. That makes it the right tool when you want ordered iteration and a simple structure, and the price is that the guarantee is only probabilistic: a search costs expected Θ((1/p)·log_{1/p} n) steps, each node holds an expected 1/(1−p) pointers, and the worst case is Θ(n) with vanishing probability. This example counts cost in *steps* (one `next` pointer examined, identical in both languages) and runs the lesson's trade-off: p = 1/2 against p = 1/4 cost the same expected steps at different memory.

**Steps:**

1. Write `Insert`, `Delete`, `Contains` and `Range` over a level generator (p = 1/2^pBits) and a step counter, plus a checker (bottom lane equals the oracle, each lane a subsequence of the lane below, a node of level L sits on exactly L lanes), and run 200,000 random operations against a sorted slice with the checker after every one, then drain the list to empty.
2. The key move: build the same 100,000 keys at p = 1/2 and p = 1/4, shuffled and sorted, count steps per successful search and pointers per node against `(1/p)·log_{1/p} n` and `1/(1−p)`, and repeat on 20 independent coin streams to see the spread and the worst single search.
3. Read the range cost (one search plus exactly k steps along the bottom lane) and the memory the build allocated, and compare the guarantee with what an AVL tree promises.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"runtime"
	"slices"
)

// A skip list: a sorted linked list in which node i also sits on a geometric
// number of higher "express lanes". Nothing is ever rebalanced; a coin flip
// per node decides its height, and the shape of the list is independent of the
// ORDER the keys arrive in.
const maxLevel = 32

type node struct {
	key  int
	next []*node // len(next) is this node's level: how many lanes it sits on
}

type skipList struct {
	head  *node // sentinel on every lane (len(next) == maxLevel); its key is unused
	level int   // lanes in use = the highest node level (0 while empty)
	n     int
	pBits int // a node climbs a lane with probability p = 1/2^pBits
	rng   *rand.Rand
	steps int // next pointers examined, cumulative: the unit every cost below is counted in
}

func newSkipList(pBits int, rng *rand.Rand) *skipList {
	return &skipList{head: &node{next: make([]*node, maxLevel)}, pBits: pBits, rng: rng}
}

// randomLevel: level 1 always, one more with probability p, capped at maxLevel.
func (s *skipList) randomLevel() int {
	lvl := 1
	for lvl < maxLevel && s.rng.IntN(1<<s.pBits) == 0 {
		lvl++
	}
	return lvl
}

// walk descends from the top lane. On each lane it keeps moving right while
// the next node is smaller than key, then drops one lane. Every look at a
// next pointer is one step (the failed look that ends a lane counts too).
// update[i] receives the last node on lane i with a key < key, which is
// exactly what insert and delete need to splice. It returns the first node
// whose key is >= key, or nil.
func (s *skipList) walk(key int, update *[maxLevel]*node) *node {
	x := s.head
	for lvl := s.level - 1; lvl >= 0; lvl-- {
		for {
			s.steps++
			nx := x.next[lvl]
			if nx == nil || nx.key >= key {
				break
			}
			x = nx
		}
		if update != nil {
			update[lvl] = x
		}
	}
	return x.next[0]
}

func (s *skipList) Contains(key int) bool {
	n := s.walk(key, nil)
	return n != nil && n.key == key
}

// Insert is a set insert: false if key is already there. The coin flips are
// drawn only for a key that really is new.
func (s *skipList) Insert(key int) bool {
	var update [maxLevel]*node
	if n := s.walk(key, &update); n != nil && n.key == key {
		return false
	}
	lvl := s.randomLevel()
	for i := s.level; i < lvl; i++ { // a taller node than any so far: new lanes start at head
		update[i] = s.head
	}
	s.level = max(s.level, lvl)
	nn := &node{key: key, next: make([]*node, lvl)}
	for i := range lvl {
		nn.next[i] = update[i].next[i]
		update[i].next[i] = nn
	}
	s.n++
	return true
}

func (s *skipList) Delete(key int) bool {
	var update [maxLevel]*node
	n := s.walk(key, &update)
	if n == nil || n.key != key {
		return false
	}
	for i := range n.next {
		update[i].next[i] = n.next[i]
	}
	for s.level > 0 && s.head.next[s.level-1] == nil {
		s.level--
	}
	s.n--
	return true
}

// Range returns the keys in [lo, hi]: one search to the first key >= lo, then
// a walk along the bottom lane. It reports the steps of each part separately.
func (s *skipList) Range(lo, hi int) (keys []int, search, walk int) {
	s0 := s.steps
	n := s.walk(lo, nil)
	search = s.steps - s0
	for n != nil && n.key <= hi {
		keys = append(keys, n.key)
		n = n.next[0]
		s.steps++
		walk++
	}
	return keys, search, walk
}

// check verifies every invariant and returns the first violation as text (it
// never panics): the bottom lane is strictly ascending and equals the oracle;
// each lane is a subsequence of the lane below; a node of level L is on
// exactly lanes 0..L-1; the list level is the highest node level.
func (s *skipList) check(oracle []int) error {
	var bottom []*node
	for x := s.head.next[0]; x != nil; x = x.next[0] {
		bottom = append(bottom, x)
		if len(bottom) > len(oracle) {
			return fmt.Errorf("bottom lane longer than the oracle (%d keys)", len(oracle))
		}
	}
	if len(bottom) != len(oracle) || s.n != len(oracle) {
		return fmt.Errorf("size: lane %d, n %d, oracle %d", len(bottom), s.n, len(oracle))
	}
	var onLane [maxLevel + 2]int // onLane[i] = how many nodes have level > i
	top := 0
	for i, x := range bottom {
		if x.key != oracle[i] {
			return fmt.Errorf("bottom lane position %d holds %d, oracle has %d", i, x.key, oracle[i])
		}
		if i > 0 && bottom[i-1].key >= x.key {
			return fmt.Errorf("bottom lane not ascending at %d", x.key)
		}
		if len(x.next) < 1 || len(x.next) > maxLevel {
			return fmt.Errorf("node %d has level %d", x.key, len(x.next))
		}
		for l := range len(x.next) {
			onLane[l]++
		}
		top = max(top, len(x.next))
	}
	if s.level != top {
		return fmt.Errorf("list level %d, highest node level %d", s.level, top)
	}
	for l := 1; l < maxLevel; l++ {
		j, count := 0, 0 // j walks the lane below
		for x := s.head.next[l]; x != nil; x = x.next[l] {
			for j < len(bottom) && bottom[j] != x {
				j++
			}
			if j == len(bottom) {
				return fmt.Errorf("lane %d is not a subsequence of the bottom lane", l)
			}
			if len(x.next) <= l {
				return fmt.Errorf("node %d is on lane %d but has level %d", x.key, l, len(x.next))
			}
			count++
		}
		if count != onLane[l] {
			return fmt.Errorf("lane %d holds %d nodes, %d nodes have level > %d", l, count, onLane[l], l)
		}
	}
	return nil
}

// shuffled returns 0..n-1 in a random order (Fisher-Yates, written out so the
// Rust twin can reproduce it draw for draw).
func shuffled(n int, rng *rand.Rand) []int {
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

func lg(x float64) float64 { return math.Log2(x) }

func pText(pBits int) string { return fmt.Sprintf("1/%d", 1<<pBits) }

// logBase returns log_{1/p}(n) for p = 1/2^pBits.
func logBase(pBits, n int) float64 { return lg(float64(n)) / float64(pBits) }

// ---------------------------------------------------------------- (a)

func oracleRun(pBits, ops int) {
	rng := rand.New(rand.NewPCG(1, 2))
	opRng := rand.New(rand.NewPCG(3, uint64(pBits)))
	s := newSkipList(pBits, rng)
	var oracle []int
	var insNew, insDup, delHit, delMiss, lookHit, lookMiss, ranges, rangeKeys int
	mismatches, failures, maxLevelSeen, peakLive := 0, 0, 0, 0
	firstFail := ""
	check := func(op int) {
		if err := s.check(oracle); err != nil {
			if failures == 0 {
				firstFail = fmt.Sprintf("op %d: %v", op, err)
			}
			failures++
		}
		maxLevelSeen = max(maxLevelSeen, s.level)
		peakLive = max(peakLive, s.n)
	}
	for op := 1; op <= ops; op++ {
		kind := opRng.IntN(100)
		key := opRng.IntN(1000)
		idx, found := slices.BinarySearch(oracle, key)
		switch {
		case kind < 40:
			if got := s.Insert(key); got == found {
				mismatches++
			}
			if found {
				insDup++
			} else {
				insNew++
				oracle = slices.Insert(oracle, idx, key)
			}
		case kind < 70:
			if got := s.Delete(key); got != found {
				mismatches++
			}
			if found {
				delHit++
				oracle = slices.Delete(oracle, idx, idx+1)
			} else {
				delMiss++
			}
		case kind < 90:
			if s.Contains(key) != found {
				mismatches++
			}
			if found {
				lookHit++
			} else {
				lookMiss++
			}
		default:
			hi := key + opRng.IntN(100)
			got, _, _ := s.Range(key, hi)
			end, _ := slices.BinarySearch(oracle, hi+1)
			if !slices.Equal(got, oracle[idx:end]) {
				mismatches++
			}
			ranges++
			rangeKeys += len(got)
		}
		check(op)
	}
	live := s.n
	// Drain: delete every remaining key in random order, checker after each,
	// so the shrinking of the list level and the empty list are exercised too.
	order := slices.Clone(oracle)
	for i := len(order) - 1; i >= 1; i-- {
		j := opRng.IntN(i + 1)
		order[i], order[j] = order[j], order[i]
	}
	for _, k := range order {
		idx, _ := slices.BinarySearch(oracle, k)
		if !s.Delete(k) {
			mismatches++
		}
		oracle = slices.Delete(oracle, idx, idx+1)
		check(ops + 1)
	}
	fmt.Printf("  p=%-3s inserts %d new + %d dup | deletes %d hit + %d miss | lookups %d hit + %d miss | %d ranges (%d keys)\n",
		pText(pBits), insNew, insDup, delHit, delMiss, lookHit, lookMiss, ranges, rangeKeys)
	fmt.Printf("        %d live keys at the end (peak %d), then drained to empty (n=%d, level=%d)\n", live, peakLive, s.n, s.level)
	fmt.Printf("        result mismatches vs the sorted slice: %d   invariant failures: %d   highest list level seen: %d\n",
		mismatches, failures, maxLevelSeen)
	if failures > 0 {
		fmt.Println("        first failure:", firstFail)
	}
}

// ---------------------------------------------------------------- (b), (c)

// searchAll looks up every key 0..n-1 (all present) and returns the mean and
// the maximum number of steps one search took.
func searchAll(s *skipList, n int) (mean float64, worst int) {
	total := 0
	for k := 0; k < n; k++ {
		before := s.steps
		if !s.Contains(k) {
			panic("a key that was inserted is missing")
		}
		d := s.steps - before
		total += d
		worst = max(worst, d)
	}
	return float64(total) / float64(n), worst
}

func buildList(pBits int, order []int, seed uint64) (s *skipList, buildSteps int) {
	s = newSkipList(pBits, rand.New(rand.NewPCG(1, seed)))
	for _, k := range order {
		s.Insert(k)
	}
	return s, s.steps
}

func pointersPerNode(s *skipList) float64 {
	total := 0
	for x := s.head.next[0]; x != nil; x = x.next[0] {
		total += len(x.next)
	}
	return float64(total) / float64(s.n)
}

func main() {
	fmt.Println("== (a) Oracle run: 200,000 random ops on keys 0..999, checker after every op ==")
	fmt.Println("  mix: 40% insert, 30% delete, 20% lookup, 10% range; the oracle is a sorted slice")
	oracleRun(1, 200000)
	oracleRun(2, 200000)

	const n = 100000
	keyRng := rand.New(rand.NewPCG(1, 2))
	shuf := shuffled(n, keyRng)
	sorted := make([]int, n)
	for i := range sorted {
		sorted[i] = i
	}

	fmt.Printf("\n== (b) n=%d keys, steps per successful search (every key looked up once) ==\n", n)
	fmt.Printf("  lg n = %.2f\n", lg(n))
	fmt.Println("  p    order     build/insert  search  search/lg n  (1/p)log_{1/p}n  ptrs/node  1/(1-p)  level  log_{1/p}n")
	meanSearch := map[string]float64{}
	var ptrs [3]float64
	for _, pBits := range []int{1, 2} {
		p := 1 / float64(int(1)<<pBits)
		for _, o := range []struct {
			name string
			keys []int
		}{{"shuffled", shuf}, {"sorted", sorted}} {
			s, build := buildList(pBits, o.keys, 7)
			mean, _ := searchAll(s, n)
			meanSearch[fmt.Sprint(pBits, o.name)] = mean
			pp := pointersPerNode(s)
			if o.name == "shuffled" {
				ptrs[pBits] = pp
			}
			fmt.Printf("  %-4s %-9s %12.2f %7.2f %12.2f %16.2f %10.3f %8.3f %6d %10.2f\n",
				pText(pBits), o.name, float64(build)/n, mean, mean/lg(n), logBase(pBits, n)/p, pp, 1/(1-p), s.level, logBase(pBits, n))
		}
	}
	fmt.Printf("  p=1/4 keeps %.0f%% fewer pointers than p=1/2 (%.3f vs %.3f per node) at %.2fx the steps\n",
		100*(1-ptrs[2]/ptrs[1]), ptrs[2], ptrs[1], meanSearch["2shuffled"]/meanSearch["1shuffled"])
	fmt.Printf("  sorted input costs %.2fx / %.2fx the steps of shuffled input (p=1/2 / p=1/4): the shape does not see the order\n",
		meanSearch["1sorted"]/meanSearch["1shuffled"], meanSearch["2sorted"]/meanSearch["2shuffled"])

	fmt.Println("\n== (c) The worst case is only likely: 20 independent level streams, same 100,000 keys ==")
	avlBound := 1.4405*lg(n+2) - 0.3277
	fmt.Printf("  a balanced BST of the same keys: a search needs at most ⌈lg(n+1)⌉ = %d comparisons; an AVL tree at most %d (height bound, a theorem)\n",
		int(math.Ceil(lg(n+1))), int(math.Floor(avlBound)))
	fmt.Println("  p    mean steps   (min..max over streams)   worst single search: min  median  max   max/mean")
	medianWorst, meanOver := map[int]int{}, map[int]float64{}
	for _, pBits := range []int{1, 2} {
		var means []float64
		var worsts []int
		for stream := 0; stream < 20; stream++ {
			s, _ := buildList(pBits, shuf, uint64(100+stream))
			m, w := searchAll(s, n)
			means = append(means, m)
			worsts = append(worsts, w)
		}
		slices.Sort(worsts)
		sum := 0.0
		for _, m := range means {
			sum += m
		}
		avg := sum / float64(len(means))
		medianWorst[pBits], meanOver[pBits] = worsts[len(worsts)/2], avg
		fmt.Printf("  %-4s %10.2f   (%.2f..%.2f) %27d %7d %5d %9.1fx\n",
			pText(pBits), avg, slices.Min(means), slices.Max(means), worsts[0], worsts[len(worsts)/2], worsts[len(worsts)-1], float64(worsts[len(worsts)-1])/avg)
	}
	fmt.Printf("  p=1/4 averages %.1f%% fewer steps than p=1/2, but its tail is heavier: the median stream's worst search is %d steps against %d\n",
		100*(1-meanOver[2]/meanOver[1]), medianWorst[2], medianWorst[1])
	for _, pBits := range []int{1, 2} {
		q := 1 - 1/float64(int(1)<<pBits)
		fmt.Printf("  the list degenerates into the bottom lane alone (every node level 1, searches cost ~n) with probability (1-p)^n = 10^-%.0f for p=%s\n",
			float64(n)*math.Log10(1/q), pText(pBits))
	}

	fmt.Println("\n== (d) Range(lo, hi) returning k keys: one search plus k bottom-lane steps (p=1/2, n=100,000) ==")
	s, _ := buildList(1, shuf, 7)
	fmt.Println("  k        mean search steps   walk steps   total   total/k   (1000 random starts each)")
	startRng := rand.New(rand.NewPCG(5, 6))
	for _, k := range []int{1, 100, 10000} {
		var searchSum, walkSum, wrong int
		const trials = 1000
		for range trials {
			lo := startRng.IntN(n - k + 1)
			keys, search, walk := s.Range(lo, lo+k-1)
			if len(keys) != k || keys[0] != lo || keys[k-1] != lo+k-1 {
				wrong++
			}
			searchSum += search
			walkSum += walk
		}
		fmt.Printf("  %-8d %17.2f %12.2f %7.2f %9.3f   wrong results: %d\n", k,
			float64(searchSum)/trials, float64(walkSum)/trials, float64(searchSum+walkSum)/trials, float64(searchSum+walkSum)/trials/float64(k), wrong)
	}

	fmt.Println("\n== (e) Memory: building the 100,000-key list (p=1/2, shuffled), measured by the runtime ==")
	var m0, m1 runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&m0)
	built, _ := buildList(1, shuf, 7)
	runtime.ReadMemStats(&m1)
	allocs := m1.Mallocs - m0.Mallocs
	nbytes := m1.TotalAlloc - m0.TotalAlloc
	fmt.Printf("  allocations: %d = %.3f per node (a node struct and a next slice each); bytes allocated: %.1f per node (the %.3f pointers per node are 8 bytes each on this 64-bit machine, rounded up to a size class)\n",
		allocs, float64(allocs)/n, float64(nbytes)/n, pointersPerNode(built))
	runtime.KeepAlive(built)
}
```

**Output:**

```
== (a) Oracle run: 200,000 random ops on keys 0..999, checker after every op ==
  mix: 40% insert, 30% delete, 20% lookup, 10% range; the oracle is a sorted slice
  p=1/2 inserts 34681 new + 45271 dup | deletes 34121 hit + 25834 miss | lookups 22823 hit + 17356 miss | 19914 ranges (551539 keys)
        560 live keys at the end (peak 628), then drained to empty (n=0, level=0)
        result mismatches vs the sorted slice: 0   invariant failures: 0   highest list level seen: 17
  p=1/4 inserts 34608 new + 45620 dup | deletes 34023 hit + 25772 miss | lookups 22472 hit + 17379 miss | 20126 ranges (560233 keys)
        585 live keys at the end (peak 618), then drained to empty (n=0, level=0)
        result mismatches vs the sorted slice: 0   invariant failures: 0   highest list level seen: 9

== (b) n=100000 keys, steps per successful search (every key looked up once) ==
  lg n = 16.61
  p    order     build/insert  search  search/lg n  (1/p)log_{1/p}n  ptrs/node  1/(1-p)  level  log_{1/p}n
  1/2  shuffled         30.86   32.92         1.98            33.22      1.995    2.000     16      16.61
  1/2  sorted           31.98   32.08         1.93            33.22      1.995    2.000     16      16.61
  1/4  shuffled         28.06   32.16         1.94            33.22      1.333    1.333     10       8.30
  1/4  sorted           29.79   31.19         1.88            33.22      1.333    1.333     10       8.30
  p=1/4 keeps 33% fewer pointers than p=1/2 (1.333 vs 1.995 per node) at 0.98x the steps
  sorted input costs 0.97x / 0.97x the steps of shuffled input (p=1/2 / p=1/4): the shape does not see the order

== (c) The worst case is only likely: 20 independent level streams, same 100,000 keys ==
  a balanced BST of the same keys: a search needs at most ⌈lg(n+1)⌉ = 17 comparisons; an AVL tree at most 23 (height bound, a theorem)
  p    mean steps   (min..max over streams)   worst single search: min  median  max   max/mean
  1/2       33.18   (29.60..37.21)                          56      60    68       2.0x
  1/4       31.18   (28.67..34.63)                          71      80    89       2.9x
  p=1/4 averages 6.0% fewer steps than p=1/2, but its tail is heavier: the median stream's worst search is 80 steps against 60
  the list degenerates into the bottom lane alone (every node level 1, searches cost ~n) with probability (1-p)^n = 10^-30103 for p=1/2
  the list degenerates into the bottom lane alone (every node level 1, searches cost ~n) with probability (1-p)^n = 10^-12494 for p=1/4

== (d) Range(lo, hi) returning k keys: one search plus k bottom-lane steps (p=1/2, n=100,000) ==
  k        mean search steps   walk steps   total   total/k   (1000 random starts each)
  1                    32.66         1.00   33.66    33.662   wrong results: 0
  100                  32.83       100.00  132.83     1.328   wrong results: 0
  10000                32.53     10000.00 10032.53     1.003   wrong results: 0

== (e) Memory: building the 100,000-key list (p=1/2, shuffled), measured by the runtime ==
  allocations: 200005 = 2.000 per node (a node struct and a next slice each); bytes allocated: 48.3 per node (the 1.995 pointers per node are 8 bytes each on this 64-bit machine, rounded up to a size class)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// ---- the kit's allocation counter (the stand-in for runtime.MemStats.Mallocs) ----
// The only `unsafe` in this program: a GlobalAlloc must be an unsafe impl. It
// forwards to the system allocator and counts calls, bytes requested and bytes live.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static TOTAL: AtomicUsize = AtomicUsize::new(0);
static LIVE: AtomicUsize = AtomicUsize::new(0);

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

/// bytes currently live on the heap.
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

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

// A skip list: a sorted linked list in which node i also sits on a geometric
// number of higher "express lanes". Nothing is ever rebalanced; a coin flip
// per node decides its height, and the shape of the list is independent of the
// ORDER the keys arrive in.
const MAX_LEVEL: usize = 32;

// A node is the target of up to `level` predecessors (one per lane), so no
// single owner exists and `Box` cannot express the list. The nodes live in an
// arena, `nodes: Vec<SNode>`, and a link is a u32 index into it. Index 0 is
// the head sentinel; it is never anybody's successor, so 0 in a `next` slot
// doubles as "nil" and no `Option` is needed.
struct SNode {
    key: u64,
    next: Vec<u32>, // next.len() is this node's level: how many lanes it sits on
}

struct SkipList {
    nodes: Vec<SNode>, // the arena; nodes[0] is the head, with MAX_LEVEL lanes
    free: Vec<u32>,    // slots of deleted nodes, reused by the next insert
    level: usize,      // lanes in use = the highest node level (0 while empty)
    n: usize,
    p_bits: u32, // a node climbs a lane with probability p = 1/2^p_bits
    rng: Pcg,
    // next pointers examined, cumulative. A Cell, because `contains` takes
    // &self (a lookup does not change the list) yet still has to count.
    steps: Cell<u64>,
}

impl SkipList {
    fn new(p_bits: u32, rng: Pcg) -> Self {
        let head = SNode {
            key: 0, // unused
            next: vec![0; MAX_LEVEL],
        };
        SkipList {
            nodes: vec![head],
            free: Vec::new(),
            level: 0,
            n: 0,
            p_bits,
            rng,
            steps: Cell::new(0),
        }
    }

    /// level 1 always, one more with probability p, capped at MAX_LEVEL.
    fn random_level(&mut self) -> usize {
        let mut lvl = 1;
        while lvl < MAX_LEVEL && self.rng.int_n(1 << self.p_bits) == 0 {
            lvl += 1;
        }
        lvl
    }

    /// Descends from the top lane. On each lane it keeps moving right while the
    /// next node is smaller than `key`, then drops one lane. Every look at a
    /// next pointer is one step (the failed look that ends a lane counts too).
    /// `update[i]` receives the last node on lane i with a key < `key`, which
    /// is exactly what insert and delete need to splice. Returns the first node
    /// whose key is >= `key`, or 0 (nil).
    fn walk(&self, key: u64, mut update: Option<&mut [u32; MAX_LEVEL]>) -> usize {
        let mut x = 0;
        for lvl in (0..self.level).rev() {
            loop {
                self.steps.set(self.steps.get() + 1);
                let nx = self.nodes[x].next[lvl] as usize;
                if nx == 0 || self.nodes[nx].key >= key {
                    break;
                }
                x = nx;
            }
            if let Some(u) = update.as_deref_mut() {
                u[lvl] = x as u32;
            }
        }
        self.nodes[x].next[0] as usize
    }

    fn contains(&self, key: u64) -> bool {
        let n = self.walk(key, None);
        n != 0 && self.nodes[n].key == key
    }

    /// A set insert: false if `key` is already there. The coin flips are drawn
    /// only for a key that really is new.
    fn insert(&mut self, key: u64) -> bool {
        // zeroed, and a zero is the head: lanes above the old level start there
        let mut update = [0u32; MAX_LEVEL];
        let n = self.walk(key, Some(&mut update));
        if n != 0 && self.nodes[n].key == key {
            return false;
        }
        let lvl = self.random_level();
        self.level = self.level.max(lvl);
        let node = SNode {
            key,
            next: vec![0; lvl],
        };
        let id = match self.free.pop() {
            Some(slot) => {
                self.nodes[slot as usize] = node;
                slot as usize
            }
            None => {
                self.nodes.push(node);
                self.nodes.len() - 1
            }
        };
        for (i, &pred) in update.iter().enumerate().take(lvl) {
            self.nodes[id].next[i] = self.nodes[pred as usize].next[i];
            self.nodes[pred as usize].next[i] = id as u32;
        }
        self.n += 1;
        true
    }

    fn delete(&mut self, key: u64) -> bool {
        let mut update = [0u32; MAX_LEVEL];
        let n = self.walk(key, Some(&mut update));
        if n == 0 || self.nodes[n].key != key {
            return false;
        }
        // take() leaves an empty Vec in the slot and frees the lane array
        let lanes = std::mem::take(&mut self.nodes[n].next);
        for (i, &succ) in lanes.iter().enumerate() {
            self.nodes[update[i] as usize].next[i] = succ;
        }
        self.free.push(n as u32);
        while self.level > 0 && self.nodes[0].next[self.level - 1] == 0 {
            self.level -= 1;
        }
        self.n -= 1;
        true
    }

    /// The keys in [lo, hi]: one search to the first key >= lo, then a walk
    /// along the bottom lane. Reports the steps of each part separately.
    fn range(&self, lo: u64, hi: u64) -> (Vec<u64>, u64, u64) {
        let s0 = self.steps.get();
        let mut n = self.walk(lo, None);
        let search = self.steps.get() - s0;
        let mut keys = Vec::new();
        let mut walk = 0;
        while n != 0 && self.nodes[n].key <= hi {
            keys.push(self.nodes[n].key);
            n = self.nodes[n].next[0] as usize;
            self.steps.set(self.steps.get() + 1);
            walk += 1;
        }
        (keys, search, walk)
    }

    /// the successor of node x on a lane, as an error (never a panic) if x has no such lane
    fn succ(&self, x: usize, lane: usize) -> Result<usize, String> {
        self.nodes[x]
            .next
            .get(lane)
            .map(|&s| s as usize)
            .ok_or_else(|| format!("node {} has no lane {lane}", self.nodes[x].key))
    }

    /// Verifies every invariant and returns the first violation as text (it
    /// never panics): the bottom lane is strictly ascending and equals the
    /// oracle; each lane is a subsequence of the lane below; a node of level L
    /// is on exactly lanes 0..L-1; the list level is the highest node level.
    /// Rust-only extra: the arena accounts for every slot (head + live + free).
    fn check(&self, oracle: &[u64]) -> Result<(), String> {
        let mut bottom: Vec<usize> = Vec::new();
        let mut x = self.succ(0, 0)?;
        while x != 0 {
            bottom.push(x);
            if bottom.len() > oracle.len() {
                return Err(format!(
                    "bottom lane longer than the oracle ({} keys)",
                    oracle.len()
                ));
            }
            x = self.succ(x, 0)?;
        }
        if bottom.len() != oracle.len() || self.n != oracle.len() {
            return Err(format!(
                "size: lane {}, n {}, oracle {}",
                bottom.len(),
                self.n,
                oracle.len()
            ));
        }
        let mut on_lane = [0usize; MAX_LEVEL + 2]; // on_lane[i] = how many nodes have level > i
        let mut top = 0;
        for (i, &x) in bottom.iter().enumerate() {
            let node = &self.nodes[x];
            if node.key != oracle[i] {
                return Err(format!(
                    "bottom lane position {i} holds {}, oracle has {}",
                    node.key, oracle[i]
                ));
            }
            if i > 0 && self.nodes[bottom[i - 1]].key >= node.key {
                return Err(format!("bottom lane not ascending at {}", node.key));
            }
            if node.next.is_empty() || node.next.len() > MAX_LEVEL {
                return Err(format!("node {} has level {}", node.key, node.next.len()));
            }
            for slot in on_lane.iter_mut().take(node.next.len()) {
                *slot += 1;
            }
            top = top.max(node.next.len());
        }
        if self.level != top {
            return Err(format!(
                "list level {}, highest node level {top}",
                self.level
            ));
        }
        for (l, &expected) in on_lane.iter().enumerate().take(MAX_LEVEL).skip(1) {
            let (mut j, mut count) = (0, 0); // j walks the lane below
            let mut x = self.succ(0, l)?;
            while x != 0 {
                while j < bottom.len() && bottom[j] != x {
                    j += 1;
                }
                if j == bottom.len() {
                    return Err(format!("lane {l} is not a subsequence of the bottom lane"));
                }
                if self.nodes[x].next.len() <= l {
                    return Err(format!(
                        "node {} is on lane {l} but has level {}",
                        self.nodes[x].key,
                        self.nodes[x].next.len()
                    ));
                }
                count += 1;
                x = self.succ(x, l)?;
            }
            if count != expected {
                return Err(format!(
                    "lane {l} holds {count} nodes, {expected} nodes have level > {l}"
                ));
            }
        }
        if self.nodes.len() != 1 + self.n + self.free.len() {
            return Err(format!(
                "arena: {} slots but 1 head + {} live + {} free",
                self.nodes.len(),
                self.n,
                self.free.len()
            ));
        }
        if self
            .free
            .iter()
            .any(|&f| !self.nodes[f as usize].next.is_empty())
        {
            return Err("a free slot still owns lanes".to_string());
        }
        Ok(())
    }
}

/// 0..n-1 in a random order (Fisher-Yates, written out so it matches the Go
/// program draw for draw).
fn shuffled(n: usize, rng: &mut Pcg) -> Vec<u64> {
    let mut a: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
    a
}

fn lg(x: f64) -> f64 {
    x.log2()
}

fn p_text(p_bits: u32) -> String {
    format!("1/{}", 1 << p_bits)
}

/// log_{1/p}(n) for p = 1/2^p_bits.
fn log_base(p_bits: u32, n: usize) -> f64 {
    lg(n as f64) / f64::from(p_bits)
}

// ---------------------------------------------------------------- (a)

fn oracle_run(p_bits: u32, ops: usize) {
    let mut s = SkipList::new(p_bits, Pcg::new(1, 2));
    let mut op_rng = Pcg::new(3, u64::from(p_bits));
    let mut oracle: Vec<u64> = Vec::new();
    let (mut ins_new, mut ins_dup, mut del_hit, mut del_miss) = (0, 0, 0, 0);
    let (mut look_hit, mut look_miss, mut ranges, mut range_keys) = (0, 0, 0, 0);
    let (mut mismatches, mut failures, mut max_level_seen, mut peak_live) = (0, 0, 0, 0);
    let mut first_fail = String::new();
    let mut check = |s: &SkipList, oracle: &[u64], op: usize| {
        if let Err(e) = s.check(oracle) {
            if failures == 0 {
                first_fail = format!("op {op}: {e}");
            }
            failures += 1;
        }
        max_level_seen = max_level_seen.max(s.level);
        peak_live = peak_live.max(s.n);
    };
    for op in 1..=ops {
        let kind = op_rng.int_n(100);
        let key = op_rng.int_n(1000);
        let (idx, found) = match oracle.binary_search(&key) {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        match kind {
            0..40 => {
                if s.insert(key) == found {
                    mismatches += 1;
                }
                if found {
                    ins_dup += 1;
                } else {
                    ins_new += 1;
                    oracle.insert(idx, key);
                }
            }
            40..70 => {
                if s.delete(key) != found {
                    mismatches += 1;
                }
                if found {
                    del_hit += 1;
                    oracle.remove(idx);
                } else {
                    del_miss += 1;
                }
            }
            70..90 => {
                if s.contains(key) != found {
                    mismatches += 1;
                }
                if found {
                    look_hit += 1;
                } else {
                    look_miss += 1;
                }
            }
            _ => {
                let hi = key + op_rng.int_n(100);
                let (got, _, _) = s.range(key, hi);
                let end = oracle.partition_point(|&x| x <= hi);
                if got != oracle[idx..end] {
                    mismatches += 1;
                }
                ranges += 1;
                range_keys += got.len();
            }
        }
        check(&s, &oracle, op);
    }
    let live = s.n;
    let slots = s.nodes.len();
    // Drain: delete every remaining key in random order, checker after each,
    // so the shrinking of the list level and the empty list are exercised too.
    let mut order = oracle.clone();
    for i in (1..order.len()).rev() {
        let j = op_rng.int_n(i as u64 + 1) as usize;
        order.swap(i, j);
    }
    for &k in &order {
        let idx = oracle.binary_search(&k).unwrap_or_else(|i| i);
        if !s.delete(k) {
            mismatches += 1;
        }
        oracle.remove(idx);
        check(&s, &oracle, ops + 1);
    }
    println!(
        "  p={:<3} inserts {ins_new} new + {ins_dup} dup | deletes {del_hit} hit + {del_miss} miss | lookups {look_hit} hit + {look_miss} miss | {ranges} ranges ({range_keys} keys)",
        p_text(p_bits)
    );
    println!(
        "        {live} live keys at the end (peak {peak_live}), then drained to empty (n={}, level={})",
        s.n, s.level
    );
    println!(
        "        result mismatches vs the sorted slice: {mismatches}   invariant failures: {failures}   highest list level seen: {max_level_seen}"
    );
    if failures > 0 {
        println!("        first failure: {first_fail}");
    }
    // Rust only: the arena never shrinks, so deleted slots go on a free list.
    println!(
        "        arena: {slots} slots at the end of the run = 1 head + peak {peak_live} live ({}), and {} slots sit on the free list after the drain",
        if slots == 1 + peak_live {
            "equal"
        } else {
            "NOT equal"
        },
        s.free.len()
    );
}

// ---------------------------------------------------------------- (b), (c)

/// Looks up every key 0..n-1 (all present); returns the mean and the maximum
/// number of steps one search took.
fn search_all(s: &SkipList, n: usize) -> (f64, u64) {
    let (mut total, mut worst) = (0, 0);
    for k in 0..n as u64 {
        let before = s.steps.get();
        assert!(s.contains(k), "a key that was inserted is missing");
        let d = s.steps.get() - before;
        total += d;
        worst = worst.max(d);
    }
    (total as f64 / n as f64, worst)
}

fn build_list(p_bits: u32, order: &[u64], seed: u64) -> (SkipList, u64) {
    let mut s = SkipList::new(p_bits, Pcg::new(1, seed));
    for &k in order {
        s.insert(k);
    }
    let build = s.steps.get();
    (s, build)
}

fn pointers_per_node(s: &SkipList) -> f64 {
    let mut total = 0;
    let mut x = s.nodes[0].next[0] as usize;
    while x != 0 {
        total += s.nodes[x].next.len();
        x = s.nodes[x].next[0] as usize;
    }
    total as f64 / s.n as f64
}

fn main() {
    println!("== (a) Oracle run: 200,000 random ops on keys 0..999, checker after every op ==");
    println!("  mix: 40% insert, 30% delete, 20% lookup, 10% range; the oracle is a sorted slice");
    oracle_run(1, 200000);
    oracle_run(2, 200000);

    const N: usize = 100000;
    let n = N as f64;
    let shuf = shuffled(N, &mut Pcg::new(1, 2));
    let sorted: Vec<u64> = (0..N as u64).collect();

    println!("\n== (b) n={N} keys, steps per successful search (every key looked up once) ==");
    println!("  lg n = {:.2}", lg(n));
    println!(
        "  p    order     build/insert  search  search/lg n  (1/p)log_{{1/p}}n  ptrs/node  1/(1-p)  level  log_{{1/p}}n"
    );
    let (mut mean_shuf, mut mean_sorted) = ([0.0; 3], [0.0; 3]);
    let mut ptrs = [0.0; 3];
    for p_bits in [1u32, 2] {
        let p = 1.0 / f64::from(1u32 << p_bits);
        for (name, keys) in [("shuffled", &shuf), ("sorted", &sorted)] {
            let (s, build) = build_list(p_bits, keys, 7);
            let (mean, _) = search_all(&s, N);
            let pp = pointers_per_node(&s);
            if name == "shuffled" {
                mean_shuf[p_bits as usize] = mean;
                ptrs[p_bits as usize] = pp;
            } else {
                mean_sorted[p_bits as usize] = mean;
            }
            println!(
                "  {:<4} {:<9} {:>12.2} {:>7.2} {:>12.2} {:>16.2} {:>10.3} {:>8.3} {:>6} {:>10.2}",
                p_text(p_bits),
                name,
                build as f64 / n,
                mean,
                mean / lg(n),
                log_base(p_bits, N) / p,
                pp,
                1.0 / (1.0 - p),
                s.level,
                log_base(p_bits, N)
            );
        }
    }
    println!(
        "  p=1/4 keeps {:.0}% fewer pointers than p=1/2 ({:.3} vs {:.3} per node) at {:.2}x the steps",
        100.0 * (1.0 - ptrs[2] / ptrs[1]),
        ptrs[2],
        ptrs[1],
        mean_shuf[2] / mean_shuf[1]
    );
    println!(
        "  sorted input costs {:.2}x / {:.2}x the steps of shuffled input (p=1/2 / p=1/4): the shape does not see the order",
        mean_sorted[1] / mean_shuf[1],
        mean_sorted[2] / mean_shuf[2]
    );

    println!(
        "\n== (c) The worst case is only likely: 20 independent level streams, same 100,000 keys =="
    );
    let avl_bound = 1.4405 * lg(n + 2.0) - 0.3277;
    println!(
        "  a balanced BST of the same keys: a search needs at most ⌈lg(n+1)⌉ = {} comparisons; an AVL tree at most {} (height bound, a theorem)",
        lg(n + 1.0).ceil() as u64,
        avl_bound.floor() as u64
    );
    println!(
        "  p    mean steps   (min..max over streams)   worst single search: min  median  max   max/mean"
    );
    let (mut median_worst, mut mean_over) = ([0u64; 3], [0.0; 3]);
    for p_bits in [1u32, 2] {
        let (mut means, mut worsts) = (Vec::new(), Vec::new());
        for stream in 0..20 {
            let (s, _) = build_list(p_bits, &shuf, 100 + stream);
            let (m, w) = search_all(&s, N);
            means.push(m);
            worsts.push(w);
        }
        worsts.sort_unstable();
        let avg = means.iter().sum::<f64>() / means.len() as f64;
        let lo = means.iter().copied().fold(f64::INFINITY, f64::min);
        let hi = means.iter().copied().fold(f64::NEG_INFINITY, f64::max);
        median_worst[p_bits as usize] = worsts[worsts.len() / 2];
        mean_over[p_bits as usize] = avg;
        println!(
            "  {:<4} {:>10.2}   ({lo:.2}..{hi:.2}) {:>27} {:>7} {:>5} {:>9.1}x",
            p_text(p_bits),
            avg,
            worsts[0],
            worsts[worsts.len() / 2],
            worsts[worsts.len() - 1],
            worsts[worsts.len() - 1] as f64 / avg
        );
    }
    println!(
        "  p=1/4 averages {:.1}% fewer steps than p=1/2, but its tail is heavier: the median stream's worst search is {} steps against {}",
        100.0 * (1.0 - mean_over[2] / mean_over[1]),
        median_worst[2],
        median_worst[1]
    );
    for p_bits in [1u32, 2] {
        let q = 1.0 - 1.0 / f64::from(1u32 << p_bits);
        println!(
            "  the list degenerates into the bottom lane alone (every node level 1, searches cost ~n) with probability (1-p)^n = 10^-{:.0} for p={}",
            n * (1.0 / q).log10(),
            p_text(p_bits)
        );
    }

    println!(
        "\n== (d) Range(lo, hi) returning k keys: one search plus k bottom-lane steps (p=1/2, n=100,000) =="
    );
    let (s, _) = build_list(1, &shuf, 7);
    println!(
        "  k        mean search steps   walk steps   total   total/k   (1000 random starts each)"
    );
    let mut start_rng = Pcg::new(5, 6);
    for k in [1usize, 100, 10000] {
        let (mut search_sum, mut walk_sum, mut wrong) = (0, 0, 0);
        const TRIALS: u64 = 1000;
        for _ in 0..TRIALS {
            let lo = start_rng.int_n((N - k + 1) as u64);
            let (keys, search, walk) = s.range(lo, lo + k as u64 - 1);
            if keys.len() != k || keys[0] != lo || keys[k - 1] != lo + k as u64 - 1 {
                wrong += 1;
            }
            search_sum += search;
            walk_sum += walk;
        }
        let t = TRIALS as f64;
        println!(
            "  {:<8} {:>17.2} {:>12.2} {:>7.2} {:>9.3}   wrong results: {wrong}",
            k,
            search_sum as f64 / t,
            walk_sum as f64 / t,
            (search_sum + walk_sum) as f64 / t,
            (search_sum + walk_sum) as f64 / t / k as f64
        );
    }

    println!(
        "\n== (e) Memory: building the 100,000-key list (p=1/2, shuffled), measured by the allocator =="
    );
    let live0 = live_bytes();
    let (allocs, _, built) = allocs_of(|| build_list(1, &shuf, 7).0);
    let live = live_bytes() - live0;
    let arena = built.nodes.capacity() * std::mem::size_of::<SNode>();
    let lanes = (pointers_per_node(&built) * n) as usize * 4;
    println!(
        "  allocations: {allocs} = {:.3} per node (a Vec of lanes per node, plus {} for the head's lanes and the arena's first allocation and regrowths)",
        allocs as f64 / n,
        allocs - N
    );
    println!(
        "  bytes live afterwards: {:.1} per node = {:.1} for the lane vectors ({:.3} pointers of 4 bytes) + {:.1} for the arena ({} slots reserved of {} bytes, {} used)",
        live as f64 / n,
        lanes as f64 / n,
        pointers_per_node(&built),
        arena as f64 / n,
        built.nodes.capacity(),
        std::mem::size_of::<SNode>(),
        built.nodes.len()
    );
}
```

**Rust output:**

```
== (a) Oracle run: 200,000 random ops on keys 0..999, checker after every op ==
  mix: 40% insert, 30% delete, 20% lookup, 10% range; the oracle is a sorted slice
  p=1/2 inserts 34681 new + 45271 dup | deletes 34121 hit + 25834 miss | lookups 22823 hit + 17356 miss | 19914 ranges (551539 keys)
        560 live keys at the end (peak 628), then drained to empty (n=0, level=0)
        result mismatches vs the sorted slice: 0   invariant failures: 0   highest list level seen: 17
        arena: 629 slots at the end of the run = 1 head + peak 628 live (equal), and 628 slots sit on the free list after the drain
  p=1/4 inserts 34608 new + 45620 dup | deletes 34023 hit + 25772 miss | lookups 22472 hit + 17379 miss | 20126 ranges (560233 keys)
        585 live keys at the end (peak 618), then drained to empty (n=0, level=0)
        result mismatches vs the sorted slice: 0   invariant failures: 0   highest list level seen: 9
        arena: 619 slots at the end of the run = 1 head + peak 618 live (equal), and 618 slots sit on the free list after the drain

== (b) n=100000 keys, steps per successful search (every key looked up once) ==
  lg n = 16.61
  p    order     build/insert  search  search/lg n  (1/p)log_{1/p}n  ptrs/node  1/(1-p)  level  log_{1/p}n
  1/2  shuffled         30.86   32.92         1.98            33.22      1.995    2.000     16      16.61
  1/2  sorted           31.98   32.08         1.93            33.22      1.995    2.000     16      16.61
  1/4  shuffled         28.06   32.16         1.94            33.22      1.333    1.333     10       8.30
  1/4  sorted           29.79   31.19         1.88            33.22      1.333    1.333     10       8.30
  p=1/4 keeps 33% fewer pointers than p=1/2 (1.333 vs 1.995 per node) at 0.98x the steps
  sorted input costs 0.97x / 0.97x the steps of shuffled input (p=1/2 / p=1/4): the shape does not see the order

== (c) The worst case is only likely: 20 independent level streams, same 100,000 keys ==
  a balanced BST of the same keys: a search needs at most ⌈lg(n+1)⌉ = 17 comparisons; an AVL tree at most 23 (height bound, a theorem)
  p    mean steps   (min..max over streams)   worst single search: min  median  max   max/mean
  1/2       33.18   (29.60..37.21)                          56      60    68       2.0x
  1/4       31.18   (28.67..34.63)                          71      80    89       2.9x
  p=1/4 averages 6.0% fewer steps than p=1/2, but its tail is heavier: the median stream's worst search is 80 steps against 60
  the list degenerates into the bottom lane alone (every node level 1, searches cost ~n) with probability (1-p)^n = 10^-30103 for p=1/2
  the list degenerates into the bottom lane alone (every node level 1, searches cost ~n) with probability (1-p)^n = 10^-12494 for p=1/4

== (d) Range(lo, hi) returning k keys: one search plus k bottom-lane steps (p=1/2, n=100,000) ==
  k        mean search steps   walk steps   total   total/k   (1000 random starts each)
  1                    32.66         1.00   33.66    33.662   wrong results: 0
  100                  32.83       100.00  132.83     1.328   wrong results: 0
  10000                32.53     10000.00 10032.53     1.003   wrong results: 0

== (e) Memory: building the 100,000-key list (p=1/2, shuffled), measured by the allocator ==
  allocations: 100018 = 1.000 per node (a Vec of lanes per node, plus 18 for the head's lanes and the arena's first allocation and regrowths)
  bytes live afterwards: 49.9 per node = 8.0 for the lane vectors (1.995 pointers of 4 bytes) + 41.9 for the arena (131072 slots reserved of 32 bytes, 100001 used)
```

> **Go vs Rust:** Sections (a) to (d) print the same counts as Go, line for line (diffed), except the one extra arena line per oracle run. A skip-list node has up to `level` predecessors and so no single owner: the nodes live in a `Vec<SNode>` arena addressed by `u32`, index 0 is both the head and "nil", `contains(&self)` counts steps through a `Cell<u64>`, and a free list reuses the slots of deleted nodes (Go leaves that to the collector); the arena ended each oracle run at exactly 1 + peak live slots. Section (e) differs by design: 100,018 allocations against Go's 200,005 (one `Vec` per node instead of a struct and a slice), yet about the same bytes per node, 49.9 live here against 48.3 allocated in Go (two different measures), because the arena reserves 131,072 slots of 32 bytes for 100,001 used.

**Complexity:** search, insert and delete cost expected `(1/p)·log_{1/p} n` steps, 33.2 for n = 100,000 at both p = 1/2 and p = 1/4, and a node holds an expected `1/(1−p)` pointers (2 against 1.333; measured 1.995 and 1.333, a third fewer at p = 1/4). The measured steps agree with the formula but are not equal across p: one stream gave 32.9 and 32.2, the average over 20 streams 33.18 and 31.18, so p = 1/4 was about 6% cheaper here. Sorted insertion cost the same as shuffled (0.97x): the shape never sees the order. The worst case is a probability, not a theorem: the worst search over 20 streams was 68 steps at p = 1/2 and 89 at p = 1/4 against a mean near 32, where an AVL tree guarantees at most 23 comparisons for these keys, and the degenerate list with no express lanes (every node level 1, searches cost about n) has probability 10^-30103 at p = 1/2 (10^-12494 at p = 1/4). `Range` returning k keys costs one search plus exactly k steps. Go allocates a node and a slice per node (2.000 allocations per node).

---

## 11. B-tree: fan-out against nodes touched

`🟡 medium` · *B-tree*

A B-tree does not reduce the number of comparisons, it reduces the number of **node visits**, and the node is the unit that costs: a disk page, or a few cache lines instead of a pointer chase per key. A B-tree of minimum degree t keeps every non-root node between t−1 and 2t−1 keys and all leaves at one depth, so a search visits one node per level, Θ(log_t n) of them, and spends Θ(log n) comparisons in total when each node is searched by binary search; insert splits full nodes on the way down and only a root split adds a level, at the top, which is why it cannot become unbalanced. This example builds one (CLRS, insert only, nodes in an arena so that a node id plays the part of a page number) at six values of t and counts levels, nodes, fill and the nodes and comparisons per search, with a linear and a binary scan inside the node side by side.

**Steps:**

1. Write the insert with its proactive split and a checker (keys ordered and between their separators, every non-root node holding t−1 to 2t−1 keys, one more child than keys, all leaves at one depth, in-order keys equal to a sorted-slice oracle), and tabulate on paper the levels the bound `⌊log_t((n+1)/2)⌋ + 1` allows for n = 10⁶ and 10⁹.
2. The key move: build 500,000 keys sorted and in random order at t = 2, 3, 6, 16, 64 and 256, search every present key and 100,000 absent ones, and count levels, nodes, fill, splits, nodes touched per search, and comparisons per search with each in-node scan; then rebuild 2,000 keys per t with the checker after every single insert.
3. Read the table against the bound and against a plain binary search of a sorted slice: nodes touched should fall as t grows while the comparisons do not, and the fill should show what sorted input does to a B-tree.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// A B-tree of minimum degree t (CLRS ch. 18): every non-root page holds
// between t-1 and 2t-1 keys, an internal page with k keys has k+1 children,
// and all leaves sit at the same depth. A page is the unit of cost: one disk
// read, or a few cache lines. The pages live in one slice and a page id is an
// index into it, which is exactly how a disk file addresses its pages.
type page struct {
	keys []int
	kids []int32 // child page ids; empty for a leaf
}

type btree struct {
	t      int
	pages  []page
	root   int32
	n      int
	splits int
}

func newBTree(t int) *btree {
	b := &btree{t: t}
	b.root = b.newPage(true)
	return b
}

func (b *btree) newPage(leaf bool) int32 {
	p := page{keys: make([]int, 0, 2*b.t-1)}
	if !leaf {
		p.kids = make([]int32, 0, 2*b.t)
	}
	b.pages = append(b.pages, p) // b.pages may move: never hold a *page across newPage
	return int32(len(b.pages) - 1)
}

// splitChild splits the full child kids[i] of parent around its median: the
// child keeps t-1 keys, a new right sibling takes the other t-1, and the
// median moves up into the parent (which the caller guarantees is not full).
func (b *btree) splitChild(parent int32, i int) {
	t := b.t
	yid := b.pages[parent].kids[i]
	zid := b.newPage(len(b.pages[yid].kids) == 0)
	y, z, par := &b.pages[yid], &b.pages[zid], &b.pages[parent]
	median := y.keys[t-1]
	z.keys = append(z.keys, y.keys[t:]...)
	y.keys = y.keys[:t-1]
	if len(y.kids) > 0 {
		z.kids = append(z.kids, y.kids[t:]...)
		y.kids = y.kids[:t]
	}
	par.keys = slices.Insert(par.keys, i, median)
	par.kids = slices.Insert(par.kids, i+1, zid)
	b.splits++
}

// Insert is a set insert (false if key exists). It splits every full page it
// meets on the way down, so a split never has to climb back up, and a full
// ROOT splits into a new root: the only way the tree gets taller, and it adds
// the level at the top, so all leaves stay at one depth.
func (b *btree) Insert(key int) bool {
	if len(b.pages[b.root].keys) == 2*b.t-1 {
		old := b.root
		b.root = b.newPage(false)
		b.pages[b.root].kids = append(b.pages[b.root].kids, old)
		b.splitChild(b.root, 0)
	}
	id := b.root
	for {
		p := &b.pages[id]
		i, found := slices.BinarySearch(p.keys, key)
		if found {
			return false
		}
		if len(p.kids) == 0 {
			p.keys = slices.Insert(p.keys, i, key)
			b.n++
			return true
		}
		if len(b.pages[p.kids[i]].keys) == 2*b.t-1 {
			b.splitChild(id, i)
			p = &b.pages[id] // newPage may have moved the slice
			if p.keys[i] == key {
				return false
			}
			if key > p.keys[i] {
				i++
			}
		}
		id = p.kids[i]
	}
}

// Two ways to search inside one page. Both count three-way comparisons
// (one per key examined) and return the same index: the position of key, or
// the child to descend into.
func scanLinear(keys []int, key int) (idx int, found bool, cmps int) {
	for i, k := range keys {
		cmps++
		switch c := cmp.Compare(key, k); {
		case c == 0:
			return i, true, cmps
		case c < 0:
			return i, false, cmps
		}
	}
	return len(keys), false, cmps
}

func scanBinary(keys []int, key int) (idx int, found bool, cmps int) {
	lo, hi := 0, len(keys)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		cmps++
		switch c := cmp.Compare(key, keys[mid]); {
		case c == 0:
			return mid, true, cmps
		case c < 0:
			hi = mid
		default:
			lo = mid + 1
		}
	}
	return lo, false, cmps
}

type probe struct {
	pages, linear, binary int // pages visited, comparisons with each in-page scan
	disagree              int // pages where the two scans returned different answers
}

// Search walks root to leaf, charging every page it touches for both scans.
func (b *btree) Search(key int) (found bool, pr probe) {
	id := b.root
	for {
		p := &b.pages[id]
		pr.pages++
		i, f, c := scanLinear(p.keys, key)
		pr.linear += c
		j, g, d := scanBinary(p.keys, key)
		pr.binary += d
		if i != j || f != g {
			pr.disagree++
		}
		if f {
			return true, pr
		}
		if len(p.kids) == 0 {
			return false, pr
		}
		id = p.kids[i]
	}
}

func (b *btree) levels() int {
	l := 1
	for id := b.root; len(b.pages[id].kids) > 0; id = b.pages[id].kids[0] {
		l++
	}
	return l
}

// check returns the first violated invariant as text (it never panics): keys
// ascending inside a page and strictly between the separators above it; every
// non-root page holds t-1..2t-1 keys; an internal page has one more child
// than keys; all leaves at one depth; every page reachable exactly once; and
// the in-order key sequence equals the oracle.
func (b *btree) check(oracle []int) error {
	seen := make([]bool, len(b.pages))
	leafDepth := -1
	var inorder []int
	var walk func(id int32, depth, lo, hi int) error
	walk = func(id int32, depth, lo, hi int) error {
		if id < 0 || int(id) >= len(b.pages) {
			return fmt.Errorf("page id %d out of range", id)
		}
		if seen[id] {
			return fmt.Errorf("page %d is reachable twice", id)
		}
		seen[id] = true
		p := &b.pages[id]
		nk := len(p.keys)
		if nk > 2*b.t-1 {
			return fmt.Errorf("page %d holds %d keys, more than %d", id, nk, 2*b.t-1)
		}
		if id != b.root && nk < b.t-1 {
			return fmt.Errorf("page %d holds %d keys, fewer than %d", id, nk, b.t-1)
		}
		for i, k := range p.keys {
			if i > 0 && p.keys[i-1] >= k {
				return fmt.Errorf("page %d keys not ascending at %d", id, k)
			}
			if k <= lo || k >= hi {
				return fmt.Errorf("page %d key %d outside its separators (%d, %d)", id, k, lo, hi)
			}
		}
		if len(p.kids) == 0 {
			if leafDepth < 0 {
				leafDepth = depth
			} else if leafDepth != depth {
				return fmt.Errorf("leaves at depths %d and %d", leafDepth, depth)
			}
			inorder = append(inorder, p.keys...)
			return nil
		}
		if len(p.kids) != nk+1 {
			return fmt.Errorf("page %d has %d keys and %d children", id, nk, len(p.kids))
		}
		for i, kid := range p.kids {
			clo, chi := lo, hi
			if i > 0 {
				clo = p.keys[i-1]
			}
			if i < nk {
				chi = p.keys[i]
			}
			if err := walk(kid, depth+1, clo, chi); err != nil {
				return err
			}
			if i < nk {
				inorder = append(inorder, p.keys[i])
			}
		}
		return nil
	}
	if err := walk(b.root, 0, math.MinInt, math.MaxInt); err != nil {
		return err
	}
	for id, s := range seen {
		if !s {
			return fmt.Errorf("page %d is unreachable", id)
		}
	}
	if len(inorder) != b.n || !slices.Equal(inorder, oracle) {
		return fmt.Errorf("in-order keys (%d) differ from the oracle (%d), n=%d", len(inorder), len(oracle), b.n)
	}
	return nil
}

// levelBound is the theorem levels <= floor(log_t((n+1)/2)) + 1, computed
// with integers: the largest h with 2*t^h - 1 <= n, plus one.
func levelBound(t, n int) int {
	h, pow := 0, 1
	for 2*pow*t-1 <= n {
		pow *= t
		h++
	}
	return h + 1
}

func shuffled(n int, rng *rand.Rand) []int {
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

func lg(x float64) float64 { return math.Log2(x) }

func main() {
	ts := []int{2, 3, 6, 16, 64, 256}

	fmt.Println("== (a) On paper: what minimum degree t buys (levels are an upper bound: floor(log_t((n+1)/2)) + 1) ==")
	fmt.Printf("  a balanced binary tree needs ⌈lg(n+1)⌉ levels: %d for n=10^6, %d for n=10^9\n",
		bits.Len(uint(1_000_000)), bits.Len(uint(1_000_000_000)))
	fmt.Println("  t     min keys  max keys  children  levels n=10^6  levels n=10^9")
	for _, t := range ts {
		fmt.Printf("  %-5d %8d %9d %5d..%-4d %14d %14d\n", t, t-1, 2*t-1, t, 2*t, levelBound(t, 1_000_000), levelBound(t, 1_000_000_000))
	}

	const n = 500000
	rng := rand.New(rand.NewPCG(1, 2))
	perm := shuffled(n, rng)
	sortedKeys := make([]int, n) // inserted keys are the even numbers 2x, so odd numbers are absent for sure
	randomKeys := make([]int, n)
	for i := range sortedKeys {
		sortedKeys[i] = 2 * i
		randomKeys[i] = 2 * perm[i]
	}
	absent := make([]int, 100000)
	for i := range absent {
		absent[i] = 2*rng.IntN(n) + 1
	}

	fmt.Printf("\n== (b) n=%d keys, inserted sorted and in random order; every present key and %d absent keys searched ==\n", n, len(absent))
	fmt.Printf("  lg n = %.2f, ⌈lg(n+1)⌉ = %d: what a perfectly balanced binary tree pays per search (comparisons, one key per node)\n",
		lg(n), bits.Len(uint(n)))
	fmt.Println("  fill = keys / (pages * (2t-1)); ref fill = (t-1)/(2t-1) for sorted input (a left page keeps t-1 keys after its split), ln 2 for random")
	fmt.Println("  t    order    levels bound  pages   fill  ref fill  splits  pages/search  linear cmps  binary cmps  checker  bound")
	type row struct{ pages, linear, binary float64 }
	rowsRandom := map[int]row{}
	var wrongPresent, wrongAbsent, disagree, builds, boundBroken, checkFailed int
	var minBin, maxBin = math.Inf(1), 0.0
	model6 := map[string]int{}
	for _, t := range ts {
		for _, o := range []struct {
			name string
			keys []int
		}{{"sorted", sortedKeys}, {"random", randomKeys}} {
			b := newBTree(t)
			for _, k := range o.keys {
				b.Insert(k)
			}
			builds++
			status := "ok"
			if err := b.check(sortedKeys); err != nil {
				status = "FAILED"
				checkFailed++
			}
			var tot probe
			for _, k := range sortedKeys {
				found, pr := b.Search(k)
				if !found {
					wrongPresent++
				}
				tot.pages += pr.pages
				tot.linear += pr.linear
				tot.binary += pr.binary
				tot.disagree += pr.disagree
			}
			for _, k := range absent {
				found, pr := b.Search(k)
				if found {
					wrongAbsent++
				}
				tot.disagree += pr.disagree
			}
			disagree += tot.disagree
			levels, bound := b.levels(), levelBound(t, n)
			okBound := "OK"
			if levels > bound {
				okBound = "VIOLATED"
				boundBroken++
			}
			fill := float64(b.n) / float64(len(b.pages)*(2*t-1))
			ref := math.Ln2
			if o.name == "sorted" {
				ref = float64(t-1) / float64(2*t-1)
			}
			r := row{float64(tot.pages) / n, float64(tot.linear) / n, float64(tot.binary) / n}
			if o.name == "random" {
				rowsRandom[t] = r
			}
			minBin, maxBin = min(minBin, r.binary), max(maxBin, r.binary)
			if t == 6 {
				model6[o.name] = len(b.pages)
			}
			fmt.Printf("  %-4d %-8s %5d %5d %7d %6.3f %9.3f %7d %13.2f %12.2f %12.2f  %-7s  %s\n",
				t, o.name, levels, bound, len(b.pages), fill, ref, b.splits, r.pages, r.linear, r.binary, status, okBound)
		}
	}
	fmt.Printf("  oracle over %d builds: %d present keys missed, %d absent keys found; %d node searches where the linear and binary scans disagreed; %d bound violations; %d checker failures\n",
		builds, wrongPresent, wrongAbsent, disagree, boundBroken, checkFailed)
	fmt.Printf("  random order, t=2 -> t=256: pages per search %.2f -> %.2f, linear comparisons %.2f -> %.2f, binary comparisons %.2f -> %.2f\n",
		rowsRandom[2].pages, rowsRandom[256].pages, rowsRandom[2].linear, rowsRandom[256].linear, rowsRandom[2].binary, rowsRandom[256].binary)

	fmt.Printf("\n== (c) 2,000 keys per t with the checker (and a sorted-slice oracle) after every insert ==\n")
	fmt.Println("       | sorted keys 0..1999        | random draws from 0..1499, so duplicates occur")
	fmt.Printf("  %-4s | %6s %6s %5s | %5s %4s %6s %6s %5s\n", "t", "failed", "levels", "pages", "new", "dup", "failed", "levels", "pages")
	for _, t := range ts {
		failS, failR := 0, 0
		bs := newBTree(t)
		var oracle []int
		for i := 0; i < 2000; i++ {
			bs.Insert(i)
			oracle = append(oracle, i)
			if bs.check(oracle) != nil {
				failS++
			}
		}
		br := newBTree(t)
		oracle = oracle[:0]
		newKeys, dups := 0, 0
		for i := 0; i < 2000; i++ {
			k := rng.IntN(1500)
			idx, found := slices.BinarySearch(oracle, k)
			if got := br.Insert(k); got == found {
				failR++
			}
			if found {
				dups++
			} else {
				newKeys++
				oracle = slices.Insert(oracle, idx, k)
			}
			if br.check(oracle) != nil {
				failR++
			}
		}
		fmt.Printf("  %-4d | %6d %6d %5d | %5d %4d %6d %6d %5d\n",
			t, failS, bs.levels(), len(bs.pages), newKeys, dups, failR, br.levels(), len(br.pages))
	}

	fmt.Println("\n== (d) The baseline in Go: a sorted slice (Go's standard library has no ordered map, read in the source: container/ is heap, list, ring) ==")
	var flat int
	for _, k := range sortedKeys {
		_, _, c := scanBinary(sortedKeys, k)
		flat += c
	}
	flatMean := float64(flat) / n
	fmt.Printf("  one binary search over one contiguous array of the same %d keys: %.2f comparisons per search, 1 \"node\" (lg n = %.2f)\n", n, flatMean, lg(n))
	fmt.Printf("  the B-trees' binary-scan comparisons per search ranged %.2f..%.2f over all %d builds: %.2fx to %.2fx a flat binary search\n",
		minBin, maxBin, builds, minBin/flatMean, maxBin/flatMean)
	fmt.Printf("  t=6 needs %d pages sorted and %d pages random for the same %d keys (reference for any other 11-key-page B-tree)\n", model6["sorted"], model6["random"], n)
}
```

**Output:**

```
== (a) On paper: what minimum degree t buys (levels are an upper bound: floor(log_t((n+1)/2)) + 1) ==
  a balanced binary tree needs ⌈lg(n+1)⌉ levels: 20 for n=10^6, 30 for n=10^9
  t     min keys  max keys  children  levels n=10^6  levels n=10^9
  2            1         3     2..4                19             29
  3            2         5     3..6                12             19
  6            5        11     6..12                8             12
  16          15        31    16..32                5              8
  64          63       127    64..128               4              5
  256        255       511   256..512               3              4

== (b) n=500000 keys, inserted sorted and in random order; every present key and 100000 absent keys searched ==
  lg n = 18.93, ⌈lg(n+1)⌉ = 19: what a perfectly balanced binary tree pays per search (comparisons, one key per node)
  fill = keys / (pages * (2t-1)); ref fill = (t-1)/(2t-1) for sorted input (a left page keeps t-1 keys after its split), ln 2 for random
  t    order    levels bound  pages   fill  ref fill  splits  pages/search  linear cmps  binary cmps  checker  bound
  2    sorted      18    18  499989  0.333     0.333  499971         17.00        18.27        17.95  ok       OK
  2    random      15    18  285036  0.585     0.693  285021         14.43        18.83        18.66  ok       OK
  3    sorted      12    12  249993  0.400     0.400  249981         11.50        19.29        18.86  ok       OK
  3    random      10    12  156472  0.639     0.693  156462          9.69        20.55        18.67  ok       OK
  6    sorted       7     7   99995  0.455     0.455   99988          6.80        25.02        18.49  ok       OK
  6    random       7     7   67311  0.675     0.693   67304          6.87        26.99        18.46  ok       OK
  16   sorted       5     5   33333  0.484     0.484   33328          4.93        37.87        17.95  ok       OK
  16   random       5     5   23529  0.685     0.693   23524          4.95        48.38        18.15  ok       OK
  64   sorted       3     3    7935  0.496     0.496    7932          2.98       125.52        17.95  ok       OK
  64   random       3     3    5653  0.696     0.693    5650          2.99       124.53        18.10  ok       OK
  256  sorted       3     3    1961  0.499     0.499    1958          3.00       277.24        17.95  ok       OK
  256  random       3     3    1359  0.720     0.693    1356          3.00       368.79        18.10  ok       OK
  oracle over 12 builds: 0 present keys missed, 0 absent keys found; 0 node searches where the linear and binary scans disagreed; 0 bound violations; 0 checker failures
  random order, t=2 -> t=256: pages per search 14.43 -> 3.00, linear comparisons 18.83 -> 368.79, binary comparisons 18.66 -> 18.10

== (c) 2,000 keys per t with the checker (and a sorted-slice oracle) after every insert ==
       | sorted keys 0..1999        | random draws from 0..1499, so duplicates occur
  t    | failed levels pages |   new  dup failed levels pages
  2    |      0     10  1991 |  1118  882      0      8   716
  3    |      0      7   997 |  1109  891      0      6   384
  6    |      0      4   398 |  1102  898      0      4   158
  16   |      0      3   133 |  1120  880      0      3    54
  64   |      0      2    32 |  1094  906      0      2    15
  256  |      0      2     8 |  1113  887      0      2     5

== (d) The baseline in Go: a sorted slice (Go's standard library has no ordered map, read in the source: container/ is heap, list, ring) ==
  one binary search over one contiguous array of the same 500000 keys: 17.95 comparisons per search, 1 "node" (lg n = 18.93)
  the B-trees' binary-scan comparisons per search ranged 17.95..18.86 over all 12 builds: 1.00x to 1.05x a flat binary search
  t=6 needs 99995 pages sorted and 67311 pages random for the same 500000 keys (reference for any other 11-key-page B-tree)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cmp::Ordering;
use std::collections::BTreeMap;
use std::sync::atomic::{AtomicU64, AtomicUsize, Ordering::Relaxed};

// ---- the kit's allocation counter (the stand-in for runtime.MemStats.Mallocs) ----
// The only `unsafe` in this program: a GlobalAlloc must be an unsafe impl. It
// forwards to the system allocator and counts calls and bytes requested. It is
// used in section (d), to count the nodes std's BTreeMap allocates.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static TOTAL: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(new, Relaxed);
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

// A B-tree of minimum degree t (CLRS ch. 18): every non-root node holds
// between t-1 and 2t-1 keys, an internal node with k keys has k+1 children,
// and all leaves sit at the same depth. A node is the unit of cost: one disk
// read, or a few cache lines. The nodes live in one Vec and a node id is an
// index into it, which is exactly how a disk file addresses its pages. A tree
// whose nodes are reached from several places (parent link, sibling, split)
// is where safe Rust reaches for an arena of indices instead of `Box`.
struct BNode {
    keys: Vec<u64>,
    kids: Vec<u32>, // child node ids; empty for a leaf
}

struct BTree {
    t: usize,
    nodes: Vec<BNode>,
    root: usize,
    n: usize,
    splits: usize,
}

impl BTree {
    fn new(t: usize) -> Self {
        let mut b = BTree {
            t,
            nodes: Vec::new(),
            root: 0,
            n: 0,
            splits: 0,
        };
        b.root = b.new_node(true);
        b
    }

    fn new_node(&mut self, leaf: bool) -> usize {
        self.nodes.push(BNode {
            keys: Vec::with_capacity(2 * self.t - 1),
            kids: if leaf {
                Vec::new()
            } else {
                Vec::with_capacity(2 * self.t)
            },
        });
        self.nodes.len() - 1
    }

    /// Splits the full child kids[i] of `parent` around its median: the child
    /// keeps t-1 keys, a new right sibling takes the other t-1, and the median
    /// moves up into the parent (which the caller guarantees is not full).
    fn split_child(&mut self, parent: usize, i: usize) {
        let t = self.t;
        let yid = self.nodes[parent].kids[i] as usize;
        let zid = self.new_node(self.nodes[yid].kids.is_empty());
        // the new node is the last one: split the Vec in two to hold &mut y and &mut z at once
        let (old, new) = self.nodes.split_at_mut(zid);
        let (y, z) = (&mut old[yid], &mut new[0]);
        let median = y.keys[t - 1];
        z.keys.extend_from_slice(&y.keys[t..]);
        y.keys.truncate(t - 1);
        if !y.kids.is_empty() {
            z.kids.extend_from_slice(&y.kids[t..]);
            y.kids.truncate(t);
        }
        let par = &mut self.nodes[parent];
        par.keys.insert(i, median);
        par.kids.insert(i + 1, zid as u32);
        self.splits += 1;
    }

    /// A set insert (false if `key` exists). It splits every full node it meets
    /// on the way down, so a split never has to climb back up, and a full ROOT
    /// splits into a new root: the only way the tree gets taller, and it adds
    /// the level at the top, so all leaves stay at one depth.
    fn insert(&mut self, key: u64) -> bool {
        if self.nodes[self.root].keys.len() == 2 * self.t - 1 {
            let old = self.root as u32;
            self.root = self.new_node(false);
            self.nodes[self.root].kids.push(old);
            self.split_child(self.root, 0);
        }
        let mut id = self.root;
        loop {
            let node = &self.nodes[id];
            let (mut i, found) = match node.keys.binary_search(&key) {
                Ok(i) => (i, true),
                Err(i) => (i, false),
            };
            if found {
                return false;
            }
            if node.kids.is_empty() {
                self.nodes[id].keys.insert(i, key);
                self.n += 1;
                return true;
            }
            let kid = node.kids[i] as usize;
            if self.nodes[kid].keys.len() == 2 * self.t - 1 {
                self.split_child(id, i);
                let separator = self.nodes[id].keys[i];
                if separator == key {
                    return false;
                }
                if key > separator {
                    i += 1;
                }
            }
            id = self.nodes[id].kids[i] as usize;
        }
    }

    /// Walks root to leaf, charging every node it touches for both scans.
    fn search(&self, key: u64) -> (bool, Probe) {
        let mut pr = Probe::default();
        let mut id = self.root;
        loop {
            let node = &self.nodes[id];
            pr.pages += 1;
            let (i, f, c) = scan_linear(&node.keys, key);
            pr.linear += c;
            let (j, g, d) = scan_binary(&node.keys, key);
            pr.binary += d;
            if i != j || f != g {
                pr.disagree += 1;
            }
            if f {
                return (true, pr);
            }
            if node.kids.is_empty() {
                return (false, pr);
            }
            id = node.kids[i] as usize;
        }
    }

    fn levels(&self) -> usize {
        let mut l = 1;
        let mut id = self.root;
        while !self.nodes[id].kids.is_empty() {
            id = self.nodes[id].kids[0] as usize;
            l += 1;
        }
        l
    }

    /// Returns the first violated invariant as text (it never panics): keys
    /// ascending inside a node and strictly between the separators above it;
    /// every non-root node holds t-1..2t-1 keys; an internal node has one more
    /// child than keys; all leaves at one depth; every node reachable exactly
    /// once; and the in-order key sequence equals the oracle.
    fn check(&self, oracle: &[u64]) -> Result<(), String> {
        let mut cx = Walk {
            seen: vec![false; self.nodes.len()],
            leaf_depth: None,
            inorder: Vec::new(),
        };
        self.walk(&mut cx, self.root, 0, None, None)?;
        if let Some(id) = cx.seen.iter().position(|&s| !s) {
            return Err(format!("page {id} is unreachable"));
        }
        if cx.inorder.len() != self.n || cx.inorder != oracle {
            return Err(format!(
                "in-order keys ({}) differ from the oracle ({}), n={}",
                cx.inorder.len(),
                oracle.len(),
                self.n
            ));
        }
        Ok(())
    }

    // lo and hi are exclusive bounds from the separators above (None = unbounded).
    fn walk(
        &self,
        cx: &mut Walk,
        id: usize,
        depth: usize,
        lo: Option<u64>,
        hi: Option<u64>,
    ) -> Result<(), String> {
        let Some(node) = self.nodes.get(id) else {
            return Err(format!("page id {id} out of range"));
        };
        if cx.seen[id] {
            return Err(format!("page {id} is reachable twice"));
        }
        cx.seen[id] = true;
        let nk = node.keys.len();
        if nk > 2 * self.t - 1 {
            return Err(format!(
                "page {id} holds {nk} keys, more than {}",
                2 * self.t - 1
            ));
        }
        if id != self.root && nk < self.t - 1 {
            return Err(format!(
                "page {id} holds {nk} keys, fewer than {}",
                self.t - 1
            ));
        }
        for (i, &k) in node.keys.iter().enumerate() {
            if i > 0 && node.keys[i - 1] >= k {
                return Err(format!("page {id} keys not ascending at {k}"));
            }
            if lo.is_some_and(|l| k <= l) || hi.is_some_and(|h| k >= h) {
                return Err(format!(
                    "page {id} key {k} outside its separators ({lo:?}, {hi:?})"
                ));
            }
        }
        if node.kids.is_empty() {
            match cx.leaf_depth {
                None => cx.leaf_depth = Some(depth),
                Some(d) if d != depth => return Err(format!("leaves at depths {d} and {depth}")),
                Some(_) => {}
            }
            cx.inorder.extend_from_slice(&node.keys);
            return Ok(());
        }
        if node.kids.len() != nk + 1 {
            return Err(format!(
                "page {id} has {nk} keys and {} children",
                node.kids.len()
            ));
        }
        for (i, &kid) in node.kids.iter().enumerate() {
            let clo = if i > 0 { Some(node.keys[i - 1]) } else { lo };
            let chi = if i < nk { Some(node.keys[i]) } else { hi };
            self.walk(cx, kid as usize, depth + 1, clo, chi)?;
            if i < nk {
                cx.inorder.push(node.keys[i]);
            }
        }
        Ok(())
    }
}

struct Walk {
    seen: Vec<bool>,
    leaf_depth: Option<usize>,
    inorder: Vec<u64>,
}

#[derive(Default)]
struct Probe {
    pages: usize,    // nodes visited
    linear: usize,   // comparisons with the linear in-node scan
    binary: usize,   // comparisons with the binary in-node scan
    disagree: usize, // nodes where the two scans returned different answers
}

// Two ways to search inside one node. Both count three-way comparisons (one
// per key examined) and return the same index: the position of the key, or
// the child to descend into.
fn scan_linear(keys: &[u64], key: u64) -> (usize, bool, usize) {
    let mut cmps = 0;
    for (i, &k) in keys.iter().enumerate() {
        cmps += 1;
        match key.cmp(&k) {
            Ordering::Equal => return (i, true, cmps),
            Ordering::Less => return (i, false, cmps),
            Ordering::Greater => {}
        }
    }
    (keys.len(), false, cmps)
}

fn scan_binary(keys: &[u64], key: u64) -> (usize, bool, usize) {
    let (mut lo, mut hi, mut cmps) = (0, keys.len(), 0);
    while lo < hi {
        let mid = (lo + hi) >> 1;
        cmps += 1;
        match key.cmp(&keys[mid]) {
            Ordering::Equal => return (mid, true, cmps),
            Ordering::Less => hi = mid,
            Ordering::Greater => lo = mid + 1,
        }
    }
    (lo, false, cmps)
}

/// The theorem levels <= floor(log_t((n+1)/2)) + 1, computed with integers:
/// the largest h with 2*t^h - 1 <= n, plus one.
fn level_bound(t: usize, n: usize) -> usize {
    let (mut h, mut pow) = (0, 1);
    while 2 * pow * t - 1 <= n {
        pow *= t;
        h += 1;
    }
    h + 1
}

fn shuffled(n: usize, rng: &mut Pcg) -> Vec<usize> {
    let mut a: Vec<usize> = (0..n).collect();
    for i in (1..n).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        a.swap(i, j);
    }
    a
}

fn lg(x: f64) -> f64 {
    x.log2()
}

/// ⌈lg(n+1)⌉, the same as the bit length of n
fn bit_len(n: usize) -> u32 {
    usize::BITS - n.leading_zeros()
}

// std's BTreeMap, with a key type that counts its own comparisons. The map
// only ever orders keys through `Ord::cmp`, so the counter sees every one.
static CMPS: AtomicU64 = AtomicU64::new(0);

#[derive(PartialEq, Eq, Clone, Copy)]
struct Counted(u64);

impl PartialOrd for Counted {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
impl Ord for Counted {
    fn cmp(&self, other: &Self) -> Ordering {
        CMPS.fetch_add(1, Relaxed);
        self.0.cmp(&other.0)
    }
}

fn get_cmps_per_search(map: &BTreeMap<Counted, u64>, keys: &[u64]) -> f64 {
    CMPS.store(0, Relaxed);
    for &k in keys {
        assert!(
            map.contains_key(&Counted(k)),
            "a key that was inserted is missing"
        );
    }
    CMPS.load(Relaxed) as f64 / keys.len() as f64
}

/// leaves and internal nodes from (allocations, bytes) and the two node sizes
/// L + I = allocs, leaf*L + internal*I = bytes, if that has an integer solution
fn split_nodes(
    allocs: usize,
    bytes: usize,
    leaf: usize,
    internal: usize,
) -> Option<(usize, usize)> {
    let extra = bytes.checked_sub(leaf * allocs)?;
    (extra % (internal - leaf) == 0).then(|| {
        let i = extra / (internal - leaf);
        (allocs - i, i)
    })
}

fn main() {
    let ts = [2usize, 3, 6, 16, 64, 256];

    println!(
        "== (a) On paper: what minimum degree t buys (levels are an upper bound: floor(log_t((n+1)/2)) + 1) =="
    );
    println!(
        "  a balanced binary tree needs ⌈lg(n+1)⌉ levels: {} for n=10^6, {} for n=10^9",
        bit_len(1_000_000),
        bit_len(1_000_000_000)
    );
    println!("  t     min keys  max keys  children  levels n=10^6  levels n=10^9");
    for t in ts {
        println!(
            "  {:<5} {:>8} {:>9} {:>5}..{:<4} {:>14} {:>14}",
            t,
            t - 1,
            2 * t - 1,
            t,
            2 * t,
            level_bound(t, 1_000_000),
            level_bound(t, 1_000_000_000)
        );
    }

    const N: usize = 500000;
    let n = N as f64;
    let mut rng = Pcg::new(1, 2);
    let perm = shuffled(N, &mut rng);
    // inserted keys are the even numbers 2x, so odd numbers are absent for sure
    let sorted_keys: Vec<u64> = (0..N as u64).map(|i| 2 * i).collect();
    let random_keys: Vec<u64> = perm.iter().map(|&p| 2 * p as u64).collect();
    let absent: Vec<u64> = (0..100000).map(|_| 2 * rng.int_n(N as u64) + 1).collect();

    println!(
        "\n== (b) n={N} keys, inserted sorted and in random order; every present key and {} absent keys searched ==",
        absent.len()
    );
    println!(
        "  lg n = {:.2}, ⌈lg(n+1)⌉ = {}: what a perfectly balanced binary tree pays per search (comparisons, one key per node)",
        lg(n),
        bit_len(N)
    );
    println!(
        "  fill = keys / (pages * (2t-1)); ref fill = (t-1)/(2t-1) for sorted input (a left page keeps t-1 keys after its split), ln 2 for random"
    );
    println!(
        "  t    order    levels bound  pages   fill  ref fill  splits  pages/search  linear cmps  binary cmps  checker  bound"
    );
    // (pages/search, linear, binary) of the random-order build at each t
    let mut rows_random = [(0.0f64, 0.0f64, 0.0f64); 257];
    let (mut wrong_present, mut wrong_absent, mut disagree, mut builds) = (0, 0, 0, 0);
    let (mut bound_broken, mut check_failed) = (0, 0);
    let (mut min_bin, mut max_bin) = (f64::INFINITY, 0.0f64);
    // what the t = 6 model built, for section (d): pages and linear comparisons
    let (mut model6_pages, mut model6_linear) = ([0usize; 2], [0.0f64; 2]);
    for t in ts {
        for (oi, (name, keys)) in [("sorted", &sorted_keys), ("random", &random_keys)]
            .into_iter()
            .enumerate()
        {
            let mut b = BTree::new(t);
            for &k in keys {
                b.insert(k);
            }
            builds += 1;
            let status = if b.check(&sorted_keys).is_ok() {
                "ok"
            } else {
                check_failed += 1;
                "FAILED"
            };
            let (mut pages, mut linear, mut binary, mut dis) = (0, 0, 0, 0);
            for &k in &sorted_keys {
                let (found, pr) = b.search(k);
                if !found {
                    wrong_present += 1;
                }
                pages += pr.pages;
                linear += pr.linear;
                binary += pr.binary;
                dis += pr.disagree;
            }
            for &k in &absent {
                let (found, pr) = b.search(k);
                if found {
                    wrong_absent += 1;
                }
                dis += pr.disagree;
            }
            disagree += dis;
            let (levels, bound) = (b.levels(), level_bound(t, N));
            let ok_bound = if levels <= bound {
                "OK"
            } else {
                bound_broken += 1;
                "VIOLATED"
            };
            let fill = b.n as f64 / (b.nodes.len() * (2 * t - 1)) as f64;
            let reference = if name == "sorted" {
                (t - 1) as f64 / (2 * t - 1) as f64
            } else {
                std::f64::consts::LN_2
            };
            let r = (pages as f64 / n, linear as f64 / n, binary as f64 / n);
            if name == "random" {
                rows_random[t] = r;
            }
            min_bin = min_bin.min(r.2);
            max_bin = max_bin.max(r.2);
            if t == 6 {
                model6_pages[oi] = b.nodes.len();
                model6_linear[oi] = r.1;
            }
            println!(
                "  {:<4} {:<8} {:>5} {:>5} {:>7} {:>6.3} {:>9.3} {:>7} {:>13.2} {:>12.2} {:>12.2}  {:<7}  {}",
                t,
                name,
                levels,
                bound,
                b.nodes.len(),
                fill,
                reference,
                b.splits,
                r.0,
                r.1,
                r.2,
                status,
                ok_bound
            );
        }
    }
    println!(
        "  oracle over {builds} builds: {wrong_present} present keys missed, {wrong_absent} absent keys found; {disagree} node searches where the linear and binary scans disagreed; {bound_broken} bound violations; {check_failed} checker failures"
    );
    println!(
        "  random order, t=2 -> t=256: pages per search {:.2} -> {:.2}, linear comparisons {:.2} -> {:.2}, binary comparisons {:.2} -> {:.2}",
        rows_random[2].0,
        rows_random[256].0,
        rows_random[2].1,
        rows_random[256].1,
        rows_random[2].2,
        rows_random[256].2
    );

    println!(
        "\n== (c) 2,000 keys per t with the checker (and a sorted-slice oracle) after every insert =="
    );
    println!(
        "       | sorted keys 0..1999        | random draws from 0..1499, so duplicates occur"
    );
    println!(
        "  {:<4} | {:>6} {:>6} {:>5} | {:>5} {:>4} {:>6} {:>6} {:>5}",
        "t", "failed", "levels", "pages", "new", "dup", "failed", "levels", "pages"
    );
    for t in ts {
        let (mut fail_s, mut fail_r) = (0, 0);
        let mut bs = BTree::new(t);
        let mut oracle: Vec<u64> = Vec::new();
        for i in 0..2000u64 {
            bs.insert(i);
            oracle.push(i);
            if bs.check(&oracle).is_err() {
                fail_s += 1;
            }
        }
        let mut br = BTree::new(t);
        oracle.clear();
        let (mut new_keys, mut dups) = (0, 0);
        for _ in 0..2000 {
            let k = rng.int_n(1500);
            let (idx, found) = match oracle.binary_search(&k) {
                Ok(i) => (i, true),
                Err(i) => (i, false),
            };
            if br.insert(k) == found {
                fail_r += 1;
            }
            if found {
                dups += 1;
            } else {
                new_keys += 1;
                oracle.insert(idx, k);
            }
            if br.check(&oracle).is_err() {
                fail_r += 1;
            }
        }
        println!(
            "  {:<4} | {:>6} {:>6} {:>5} | {:>5} {:>4} {:>6} {:>6} {:>5}",
            t,
            fail_s,
            bs.levels(),
            bs.nodes.len(),
            new_keys,
            dups,
            fail_r,
            br.levels(),
            br.nodes.len()
        );
    }

    println!("\n== (d) Baselines: a sorted slice, then (Rust only) std's BTreeMap ==");
    let flat: usize = sorted_keys
        .iter()
        .map(|&k| scan_binary(&sorted_keys, k).2)
        .sum();
    let flat_mean = flat as f64 / n;
    println!(
        "  one binary search over one contiguous array of the same {N} keys: {flat_mean:.2} comparisons per search, 1 \"node\" (lg n = {:.2})",
        lg(n)
    );
    println!(
        "  the B-trees' binary-scan comparisons per search ranged {min_bin:.2}..{max_bin:.2} over all {builds} builds: {:.2}x to {:.2}x a flat binary search",
        min_bin / flat_mean,
        max_bin / flat_mean
    );
    println!(
        "  t=6 needs {} pages sorted and {} pages random for the same {N} keys (reference for any other 11-key-page B-tree)",
        model6_pages[0], model6_pages[1]
    );

    // Rust only. std's BTreeMap allocates every node with one Box, so the
    // allocator's counts are node counts; a leaf and an internal node have
    // different sizes, which lets two numbers (allocations, bytes) give two
    // unknowns (leaves, internal nodes). First measure both sizes.
    let (a1, leaf_bytes, _) = allocs_of(|| {
        let mut m = BTreeMap::new();
        m.insert(Counted(0), 0u64);
        m
    });
    let (a12, bytes12, _) = allocs_of(|| {
        let mut m = BTreeMap::new();
        for k in 0..12u64 {
            m.insert(Counted(k), k);
        }
        m
    });
    let internal_bytes = bytes12 - 2 * leaf_bytes;
    println!("\n  std BTreeMap<u64, u64>, nodes counted by the allocator:");
    println!(
        "    calibration: 1 entry = {a1} allocation of {leaf_bytes} bytes (a leaf); 12 entries = {a12} allocations, {bytes12} bytes (two leaves and one internal node of {internal_bytes} bytes: the 12th insert splits the root)"
    );
    println!(
        "    built from the same {N} keys   allocations  leaves  internal  keys/node  fill (of 11)  t=6 model pages  model fill"
    );
    let build_sorted = || {
        let mut m = BTreeMap::new();
        for &k in &sorted_keys {
            m.insert(Counted(k), k / 2);
        }
        m
    };
    let build_random = || {
        let mut m = BTreeMap::new();
        for &k in &random_keys {
            m.insert(Counted(k), k / 2);
        }
        m
    };
    // collect() sorts its input first; measure that scratch buffer on its own and subtract it
    let input: Vec<(Counted, u64)> = sorted_keys.iter().map(|&k| (Counted(k), k / 2)).collect();
    let mut copy = input.clone();
    let (sort_allocs, sort_bytes, _) = allocs_of(|| copy.sort_by_key(|a| a.0));
    drop(copy);
    let mut cmps = Vec::new();
    for (name, built, model) in [
        ("sorted inserts", allocs_of(build_sorted), Some(0)),
        ("random inserts", allocs_of(build_random), Some(1)),
        (
            "collect() from a sorted iterator",
            allocs_of(|| input.into_iter().collect::<BTreeMap<Counted, u64>>()),
            None,
        ),
    ] {
        let (mut allocs, mut bytes, map) = built;
        if model.is_none() {
            allocs -= sort_allocs;
            bytes -= sort_bytes;
        }
        let (leaves, internal) = match split_nodes(allocs, bytes, leaf_bytes, internal_bytes) {
            Some((l, i)) => (l.to_string(), i.to_string()),
            None => ("?".to_string(), "?".to_string()),
        };
        let (model_pages, model_fill) = match model {
            Some(m) => (
                model6_pages[m].to_string(),
                format!("{:.3}", n / (model6_pages[m] * 11) as f64),
            ),
            None => ("-".to_string(), "-".to_string()),
        };
        println!(
            "    {:<32} {:>11} {:>7} {:>9} {:>10.2} {:>13.3} {:>16} {:>11}",
            name,
            allocs,
            leaves,
            internal,
            n / allocs as f64,
            n / (allocs * 11) as f64,
            model_pages,
            model_fill
        );
        cmps.push(get_cmps_per_search(&map, &sorted_keys));
    }
    println!(
        "    collect() also allocated {sort_allocs} scratch buffer(s) for its stable sort ({sort_bytes} bytes), subtracted above"
    );
    println!(
        "    get() comparisons per successful search (calls of Ord::cmp): sorted-built {:.2}, random-built {:.2}, collect-built {:.2}; the t=6 model above with a linear scan: {:.2} sorted, {:.2} random",
        cmps[0], cmps[1], cmps[2], model6_linear[0], model6_linear[1]
    );
}
```

**Rust output:**

```
== (a) On paper: what minimum degree t buys (levels are an upper bound: floor(log_t((n+1)/2)) + 1) ==
  a balanced binary tree needs ⌈lg(n+1)⌉ levels: 20 for n=10^6, 30 for n=10^9
  t     min keys  max keys  children  levels n=10^6  levels n=10^9
  2            1         3     2..4                19             29
  3            2         5     3..6                12             19
  6            5        11     6..12                8             12
  16          15        31    16..32                5              8
  64          63       127    64..128               4              5
  256        255       511   256..512               3              4

== (b) n=500000 keys, inserted sorted and in random order; every present key and 100000 absent keys searched ==
  lg n = 18.93, ⌈lg(n+1)⌉ = 19: what a perfectly balanced binary tree pays per search (comparisons, one key per node)
  fill = keys / (pages * (2t-1)); ref fill = (t-1)/(2t-1) for sorted input (a left page keeps t-1 keys after its split), ln 2 for random
  t    order    levels bound  pages   fill  ref fill  splits  pages/search  linear cmps  binary cmps  checker  bound
  2    sorted      18    18  499989  0.333     0.333  499971         17.00        18.27        17.95  ok       OK
  2    random      15    18  285036  0.585     0.693  285021         14.43        18.83        18.66  ok       OK
  3    sorted      12    12  249993  0.400     0.400  249981         11.50        19.29        18.86  ok       OK
  3    random      10    12  156472  0.639     0.693  156462          9.69        20.55        18.67  ok       OK
  6    sorted       7     7   99995  0.455     0.455   99988          6.80        25.02        18.49  ok       OK
  6    random       7     7   67311  0.675     0.693   67304          6.87        26.99        18.46  ok       OK
  16   sorted       5     5   33333  0.484     0.484   33328          4.93        37.87        17.95  ok       OK
  16   random       5     5   23529  0.685     0.693   23524          4.95        48.38        18.15  ok       OK
  64   sorted       3     3    7935  0.496     0.496    7932          2.98       125.52        17.95  ok       OK
  64   random       3     3    5653  0.696     0.693    5650          2.99       124.53        18.10  ok       OK
  256  sorted       3     3    1961  0.499     0.499    1958          3.00       277.24        17.95  ok       OK
  256  random       3     3    1359  0.720     0.693    1356          3.00       368.79        18.10  ok       OK
  oracle over 12 builds: 0 present keys missed, 0 absent keys found; 0 node searches where the linear and binary scans disagreed; 0 bound violations; 0 checker failures
  random order, t=2 -> t=256: pages per search 14.43 -> 3.00, linear comparisons 18.83 -> 368.79, binary comparisons 18.66 -> 18.10

== (c) 2,000 keys per t with the checker (and a sorted-slice oracle) after every insert ==
       | sorted keys 0..1999        | random draws from 0..1499, so duplicates occur
  t    | failed levels pages |   new  dup failed levels pages
  2    |      0     10  1991 |  1118  882      0      8   716
  3    |      0      7   997 |  1109  891      0      6   384
  6    |      0      4   398 |  1102  898      0      4   158
  16   |      0      3   133 |  1120  880      0      3    54
  64   |      0      2    32 |  1094  906      0      2    15
  256  |      0      2     8 |  1113  887      0      2     5

== (d) Baselines: a sorted slice, then (Rust only) std's BTreeMap ==
  one binary search over one contiguous array of the same 500000 keys: 17.95 comparisons per search, 1 "node" (lg n = 18.93)
  the B-trees' binary-scan comparisons per search ranged 17.95..18.86 over all 12 builds: 1.00x to 1.05x a flat binary search
  t=6 needs 99995 pages sorted and 67311 pages random for the same 500000 keys (reference for any other 11-key-page B-tree)

  std BTreeMap<u64, u64>, nodes counted by the allocator:
    calibration: 1 entry = 1 allocation of 192 bytes (a leaf); 12 entries = 3 allocations, 672 bytes (two leaves and one internal node of 288 bytes: the 12th insert splits the root)
    built from the same 500000 keys   allocations  leaves  internal  keys/node  fill (of 11)  t=6 model pages  model fill
    sorted inserts                         83331   71428     11903       6.00         0.545            99995       0.455
    random inserts                         66849   58990      7859       7.48         0.680            67311       0.675
    collect() from a sorted iterator       45459   41667      3792      11.00         1.000                -           -
    collect() also allocated 1 scratch buffer(s) for its stable sort (8000000 bytes), subtracted above
    get() comparisons per successful search (calls of Ord::cmp): sorted-built 24.82, random-built 28.92, collect-built 31.00; the t=6 model above with a linear scan: 25.02 sorted, 26.99 random
```

> **Go vs Rust:** Sections (a) to (c) and the sorted-slice lines of (d) are identical to Go's (diffed); the nodes are a `Vec<BNode>` addressed by `u32` ids, and `split_child` uses `split_at_mut` to hold the full child and its brand-new sibling as two `&mut` at once. The rest of (d) is Rust only, because Go has no ordered map: std's `BTreeMap` is a B-tree with B = 6 (`CAPACITY` = 11, non-root nodes hold at least 5, a plain linear scan inside the node, one `Box` per node, all read in `alloc/src/collections/btree/node.rs` and `search.rs`, not run), and a counting allocator plus a key type that counts `Ord::cmp` calls measure it. Its sorted inserts fill nodes to 6.00 keys (0.545) against the CLRS t = 6 model's 0.455, consistent with, not proven to be caused by, `splitpoint` leaving the left node 6 keys when the new key goes at the far right; random inserts land within 1% of the model (66,849 against 67,311 nodes), and `collect()` packs nodes full (45,459 nodes, 11.00 keys each).

**Complexity:** a B-tree of minimum degree t has at most `⌊log_t((n+1)/2)⌋ + 1` levels, a search visits one node per level, Θ(log_t n) node visits and Θ(log n) comparisons with binary search in the node, and an insert is amortised O(1) splits of Θ(t) work each. At n = 500,000 all 12 builds passed the checker and the bound (every sorted build sits exactly on it, the random builds are shallower at t = 2 and 3: 15 and 10 levels against 18 and 12). From t = 2 to t = 256 with random insertion the nodes touched per search fell from 14.43 to 3.00 and the linear in-node comparisons rose from 18.83 to 368.79, while the binary ones stayed at 17.95–18.86, between 1.00x and 1.05x of a flat binary search of a sorted array (17.95): fan-out saves node visits, not comparisons. Fill is exactly (t−1)/(2t−1), about ½, for ascending inserts (0.333 at t = 2, 0.499 at t = 256) and 0.585–0.720 for random ones (0.675–0.696 for t = 6 to 64, near ln 2 = 0.693). Splits per insert fell from 1.00 (t = 2, sorted) to 0.003 (t = 256, random).

---

> Next tier: [🔴 hard](3-hard.md) — the worst delete, the B+ tree, the page model, comparisons vs node visits, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
