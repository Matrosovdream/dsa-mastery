# Step 17 — Binary Trees & Traversals · 🟡 Medium

Examples **7–11**: the three path-sum problems, comparing two trees in lockstep, the lowest common
ancestor and the input that breaks it, serialization, and rebuilding a tree from two traversals.

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

## 7. Path sums: down, up and anywhere

`🟡 medium` · *Prefix-sum map on a path*

"Path sum" is three different problems. Root-to-leaf is a pre-order walk that carries what is left and lets only a leaf decide. Counting the downward paths that add up to k is lesson 10's `prefix[j] - k` identity applied to the current root path, with an **undo** on the way back up, which brings Θ(n·h) brute force down to Θ(n) map operations. The maximum path sum with negative values returns the best *downward* gain and records the best *bent* path on the side, and must not start its running best at 0. The program checks each against a brute-force oracle on thousands of small trees, then runs the two bugs (no undo, `best := 0`) against the same oracle.

**Steps:**

1. Run `hasPathSum` with and without the early exit on a 5,000-node tree, for a target that exists and one that does not, and compare the nodes visited.
2. Count the downward paths summing to k three ways (brute walk from every node, prefix map with undo, prefix map without) on 2,000 small trees, then on a 50,000-node random tree and a 2,000-node chain, and compare steps with map operations.
3. Check `maxPathSum` against all node pairs on 2,000 trees (500 of them all-negative) and read off what the `best := 0` bug returns.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func node(v int, l, r *Node) *Node { return &Node{v, l, r} }

// randomSplit builds a SHAPE only (every Val is 0). The left subtree gets a uniformly
// drawn k of the other n-1 nodes. The draw is made BEFORE the recursion, so the Rust twin
// consumes the PCG stream in the same order and builds the identical tree.
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelRange gives every node a value in [lo, hi], drawn in pre-order, in a pass of its own.
func labelRange(n *Node, r *rand.Rand, lo, hi int) {
	if n == nil {
		return
	}
	n.Val = r.IntN(hi-lo+1) + lo
	labelRange(n.Left, r, lo, hi)
	labelRange(n.Right, r, lo, hi)
}

// leftChain builds n nodes in a loop (never recursively); Val = depth.
func leftChain(n int) *Node {
	var root *Node
	for i := n - 1; i >= 0; i-- {
		root = &Node{i, root, nil}
	}
	return root
}

// show prints a tree as nested text: 1(2,3(-,4)); "-" is an empty child.
func show(n *Node) string {
	if n == nil {
		return "-"
	}
	if n.Left == nil && n.Right == nil {
		return fmt.Sprint(n.Val)
	}
	return fmt.Sprintf("%d(%s,%s)", n.Val, show(n.Left), show(n.Right))
}

// ---- (a) root to leaf ------------------------------------------------------------

// hasPathSum: is there a root-to-LEAF path whose values add up to rest? The remaining sum
// travels down as an argument. visits counts the real nodes entered.
func hasPathSum(n *Node, rest int, visits *int, early bool) bool {
	if n == nil {
		return false
	}
	*visits++
	rest -= n.Val
	if n.Left == nil && n.Right == nil {
		return rest == 0 // only a leaf decides: negative values make "rest < 0" prove nothing
	}
	l := hasPathSum(n.Left, rest, visits, early)
	if l && early {
		return true
	}
	r := hasPathSum(n.Right, rest, visits, early)
	return l || r
}

func leafSums(n *Node, sum int, out *[]int) {
	if n == nil {
		return
	}
	sum += n.Val
	if n.Left == nil && n.Right == nil {
		*out = append(*out, sum)
	}
	leafSums(n.Left, sum, out)
	leafSums(n.Right, sum, out)
}

// ---- (b) any downward path ---------------------------------------------------------

// countBrute starts a downward walk at every node; steps counts every node a walk enters.
func countBrute(root *Node, k int) (count, steps int) {
	var from func(n *Node, rest int) int
	from = func(n *Node, rest int) int {
		if n == nil {
			return 0
		}
		steps++
		rest -= n.Val
		c := 0
		if rest == 0 {
			c = 1
		}
		return c + from(n.Left, rest) + from(n.Right, rest)
	}
	var each func(n *Node) int
	each = func(n *Node) int {
		if n == nil {
			return 0
		}
		return from(n, k) + each(n.Left) + each(n.Right)
	}
	return each(root), steps
}

// countMap is lesson 10's prefix[j]-k identity on the CURRENT ROOT PATH: seen[s] = how many
// nodes on the path from the root to here have a prefix sum of s. ops counts map operations.
// With undo=false the entry is never removed: that is the bug.
func countMap(root *Node, k int, undo bool) (count, ops int) {
	seen := map[int]int{0: 1} // the empty prefix, so a path may start at the root
	var dfs func(n *Node, sum int) int
	dfs = func(n *Node, sum int) int {
		if n == nil {
			return 0
		}
		sum += n.Val
		got := seen[sum-k] // read BEFORE adding sum, or k = 0 would count an empty path
		seen[sum]++
		ops += 2
		got += dfs(n.Left, sum) + dfs(n.Right, sum)
		if undo {
			seen[sum]-- // leaving this node: siblings must not see our prefix
			ops++
		}
		return got
	}
	return dfs(root, 0), ops
}

// ---- (c) the best bent path ------------------------------------------------------

// maxPathSum: down(n) is the best DOWNWARD path starting at n, never below 0 (a negative
// branch is just not taken). The best BENT path through n is n + left gain + right gain.
// init is the starting value of the running best: MinInt is right, 0 is the bug.
func maxPathSum(root *Node, init int) int {
	best := init
	var down func(n *Node) int
	down = func(n *Node) int {
		if n == nil {
			return 0
		}
		l, r := down(n.Left), down(n.Right)
		best = max(best, n.Val+l+r)
		return max(0, n.Val+max(l, r))
	}
	down(root)
	return best
}

// maxPathOracle sums the path between EVERY pair of nodes (a node with itself included).
func maxPathOracle(root *Node) int {
	var vals, parent, depth []int
	var walk func(n *Node, p, d int)
	walk = func(n *Node, p, d int) {
		if n == nil {
			return
		}
		me := len(vals)
		vals, parent, depth = append(vals, n.Val), append(parent, p), append(depth, d)
		walk(n.Left, me, d+1)
		walk(n.Right, me, d+1)
	}
	walk(root, -1, 0)
	best := math.MinInt
	for a := range vals {
		for b := a; b < len(vals); b++ {
			x, y, s := a, b, 0
			for x != y { // climb the deeper end until the two ends meet
				if depth[x] >= depth[y] {
					s += vals[x]
					x = parent[x]
				} else {
					s += vals[y]
					y = parent[y]
				}
			}
			best = max(best, s+vals[x])
		}
	}
	return best
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	fmt.Println("== (a) root-to-leaf: hasPathSum on randomSplit(5000), values in [-5, 5]")
	big := randomSplit(5000, r)
	labelRange(big, r, -5, 5)
	var sums []int
	leafSums(big, 0, &sums)
	have, missing := sums[len(sums)/2], slices.Max(sums)+1
	fmt.Printf("%d leaves; the target that exists is the sum at the middle leaf; first leaf with that sum is #%d\n",
		len(sums), slices.Index(sums, have))
	fmt.Printf("%-8s %6s %6s %12s %10s\n", "target", "value", "found", "early exit", "full walk")
	for _, c := range []struct {
		name string
		t    int
	}{{"exists", have}, {"absent", missing}} {
		var early, full int
		ok := hasPathSum(big, c.t, &early, true)
		ok2 := hasPathSum(big, c.t, &full, false)
		fmt.Printf("%-8s %6d %6v %12d %10d   agree: %v\n", c.name, c.t, ok, early, full, ok == ok2)
	}

	fmt.Println("\n== (b) downward paths summing to k: brute vs prefix map, 2000 small trees (n <= 60, k in [-10, 10])")
	wrongUndo, wrongNo, smallest := 0, 0, 0
	var example string
	for t := 0; t < 2000; t++ {
		n := r.IntN(60) + 1
		tree := randomSplit(n, r)
		labelRange(tree, r, -5, 5)
		k := r.IntN(21) - 10
		want, _ := countBrute(tree, k)
		withUndo, _ := countMap(tree, k, true)
		noUndo, _ := countMap(tree, k, false)
		if withUndo != want {
			wrongUndo++
		}
		if noUndo != want {
			wrongNo++
			if smallest == 0 || n < smallest {
				smallest, example = n, fmt.Sprintf("%s  k=%d  brute=%d  no-undo=%d", show(tree), k, want, noUndo)
			}
		}
	}
	fmt.Printf("map with undo:    %d mismatches\n", wrongUndo)
	fmt.Printf("map without undo: wrong on %d of 2000 trees; smallest failing tree:\n  %s\n", wrongNo, example)

	fmt.Printf("\n%-19s %6s %3s %8s %12s %9s %12s %6s\n", "tree", "n", "k", "paths", "brute steps", "map ops", "no-undo says", "agree")
	chain := leftChain(2000)
	labelRange(chain, r, -5, 5)
	for _, c := range []struct {
		name string
		n    int
		root *Node
	}{{"randomSplit(50000)", 50000, nil}, {"leftChain(2000)", 2000, chain}} {
		if c.root == nil {
			c.root = randomSplit(c.n, r)
			labelRange(c.root, r, -5, 5)
		}
		const k = 7
		want, steps := countBrute(c.root, k)
		got, ops := countMap(c.root, k, true)
		bad, _ := countMap(c.root, k, false)
		fmt.Printf("%-19s %6d %3d %8d %12d %9d %12d %6v\n", c.name, c.n, k, want, steps, ops, bad, got == want)
		verdict := "right (a chain has no siblings)"
		if bad != want {
			verdict = fmt.Sprintf("%.0fx too big", float64(bad)/float64(want))
		}
		fmt.Printf("  brute steps / map ops = %.1f;  no-undo count is %s\n", float64(steps)/float64(ops), verdict)
	}
	fmt.Printf("n(n+1)/2 for the 2000-chain = %d\n", 2000*2001/2)

	fmt.Println("\n== (c) maximum path sum against all node pairs: 1500 mixed trees, 500 all-negative (n <= 40)")
	var wrongGood, wrongZeroMixed, wrongZeroNeg int
	for t := 0; t < 2000; t++ {
		tree := randomSplit(r.IntN(40)+1, r)
		if t < 1500 {
			labelRange(tree, r, -5, 5)
		} else {
			labelRange(tree, r, -9, -1)
		}
		want := maxPathOracle(tree)
		if maxPathSum(tree, math.MinInt) != want {
			wrongGood++
		}
		if maxPathSum(tree, 0) != want {
			if t < 1500 {
				wrongZeroMixed++
			} else {
				wrongZeroNeg++
			}
		}
	}
	fmt.Printf("best starts at MinInt: %d mismatches\n", wrongGood)
	fmt.Printf("best starts at 0:      wrong on %d of 1500 mixed trees and on %d of 500 all-negative trees\n", wrongZeroMixed, wrongZeroNeg)
	neg := node(-3, node(-2, node(-5, nil, nil), node(-1, nil, nil)), node(-4, nil, nil))
	fmt.Printf("tree %s: right answer %d (the single node -1), \"best := 0\" says %d\n",
		show(neg), maxPathSum(neg, math.MinInt), maxPathSum(neg, 0))
}
```

**Output:**

```
== (a) root-to-leaf: hasPathSum on randomSplit(5000), values in [-5, 5]
1700 leaves; the target that exists is the sum at the middle leaf; first leaf with that sum is #15
target    value  found   early exit  full walk
exists        1   true           49       5000   agree: true
absent       41  false         5000       5000   agree: true

== (b) downward paths summing to k: brute vs prefix map, 2000 small trees (n <= 60, k in [-10, 10])
map with undo:    0 mismatches
map without undo: wrong on 1603 of 2000 trees; smallest failing tree:
  -4(-5(-,-1),-4(-,-5))  k=2  brute=0  no-undo=1

tree                     n   k    paths  brute steps   map ops no-undo says  agree
randomSplit(50000)   50000   7    25515       952310    150000     25514647   true
  brute steps / map ops = 6.3;  no-undo count is 1000x too big
leftChain(2000)       2000   7    14826      2001000      6000        14826   true
  brute steps / map ops = 333.5;  no-undo count is right (a chain has no siblings)
n(n+1)/2 for the 2000-chain = 2001000

== (c) maximum path sum against all node pairs: 1500 mixed trees, 500 all-negative (n <= 40)
best starts at MinInt: 0 mismatches
best starts at 0:      wrong on 39 of 1500 mixed trees and on 500 of 500 all-negative trees
tree -3(-2(-5,-1),-4): right answer -1 (the single node -1), "best := 0" says 0
```

**Rust:**

```rust
use std::collections::HashMap;

// Pcg: the port of Go's math/rand/v2 PCG-DXSM generator and its IntN from lesson 01
// example 14, so seed (1, 2) builds exactly the trees the Go program builds.
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

// An empty tree is None; a node owns its two subtrees through Box.
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// A chain built in a loop is n levels deep, and the compiler's default drop glue
// would recurse once per level. This Drop moves the children onto a heap stack
// instead (lesson 05 example 15). Go has no destructor to write.
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

fn node(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

// random_split builds a SHAPE only (every val is 0). The left subtree gets a
// uniformly drawn k of the other n-1 nodes. The draw is made BEFORE the
// recursion, so both languages consume the PCG stream in the same order.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n as u64) as usize;
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    node(0, left, right)
}

// label_range gives every node a value in [lo, hi], drawn in pre-order, in a pass of its own.
fn label_range(n: &mut Link, r: &mut Pcg, lo: i64, hi: i64) {
    if let Some(b) = n {
        b.val = r.int_n((hi - lo + 1) as u64) as i64 + lo;
        label_range(&mut b.left, r, lo, hi);
        label_range(&mut b.right, r, lo, hi);
    }
}

// left_chain builds n nodes in a loop (never recursively); val = depth.
fn left_chain(n: usize) -> Link {
    let mut root = None;
    for i in (0..n).rev() {
        root = node(i as i64, root, None);
    }
    root
}

// show prints a tree as nested text: 1(2,3(-,4)); "-" is an empty child.
fn show(n: Option<&Node>) -> String {
    match n {
        None => "-".to_string(),
        Some(x) if x.left.is_none() && x.right.is_none() => x.val.to_string(),
        Some(x) => format!(
            "{}({},{})",
            x.val,
            show(x.left.as_deref()),
            show(x.right.as_deref())
        ),
    }
}

// ---- (a) root to leaf ------------------------------------------------------------

// has_path_sum: is there a root-to-LEAF path whose values add up to `rest`? The
// remaining sum travels down as an argument. `visits` counts the real nodes entered.
fn has_path_sum(n: Option<&Node>, rest: i64, visits: &mut u64, early: bool) -> bool {
    let Some(n) = n else { return false };
    *visits += 1;
    let rest = rest - n.val;
    if n.left.is_none() && n.right.is_none() {
        return rest == 0; // only a leaf decides: negative values make "rest < 0" prove nothing
    }
    let l = has_path_sum(n.left.as_deref(), rest, visits, early);
    if l && early {
        return true;
    }
    let r = has_path_sum(n.right.as_deref(), rest, visits, early);
    l || r
}

fn leaf_sums(n: Option<&Node>, sum: i64, out: &mut Vec<i64>) {
    let Some(n) = n else { return };
    let sum = sum + n.val;
    if n.left.is_none() && n.right.is_none() {
        out.push(sum);
    }
    leaf_sums(n.left.as_deref(), sum, out);
    leaf_sums(n.right.as_deref(), sum, out);
}

// ---- (b) any downward path ---------------------------------------------------------

// count_brute starts a downward walk at every node; steps counts every node a walk enters.
fn count_brute(root: &Link, k: i64) -> (i64, u64) {
    fn from(n: Option<&Node>, rest: i64, steps: &mut u64) -> i64 {
        let Some(n) = n else { return 0 };
        *steps += 1;
        let rest = rest - n.val;
        i64::from(rest == 0)
            + from(n.left.as_deref(), rest, steps)
            + from(n.right.as_deref(), rest, steps)
    }
    fn each(n: Option<&Node>, k: i64, steps: &mut u64) -> i64 {
        let Some(x) = n else { return 0 };
        from(n, k, steps) + each(x.left.as_deref(), k, steps) + each(x.right.as_deref(), k, steps)
    }
    let mut steps = 0;
    let count = each(root.as_deref(), k, &mut steps);
    (count, steps)
}

// PrefixWalk is lesson 10's prefix[j]-k identity on the CURRENT ROOT PATH:
// seen[s] = how many nodes on the path from the root to here have a prefix sum
// of s. `ops` counts map operations. With undo == false the entry is never
// removed: that is the bug. No globals in Rust: the state lives in a struct and
// the recursion borrows it mutably.
struct PrefixWalk {
    k: i64,
    undo: bool,
    seen: HashMap<i64, i64>,
    ops: u64,
}

impl PrefixWalk {
    fn dfs(&mut self, n: Option<&Node>, sum: i64) -> i64 {
        let Some(n) = n else { return 0 };
        let sum = sum + n.val;
        // read BEFORE adding sum, or k = 0 would count an empty path
        let mut got = self.seen.get(&(sum - self.k)).copied().unwrap_or(0);
        *self.seen.entry(sum).or_insert(0) += 1;
        self.ops += 2;
        got += self.dfs(n.left.as_deref(), sum) + self.dfs(n.right.as_deref(), sum);
        if self.undo {
            // leaving this node: siblings must not see our prefix
            *self.seen.entry(sum).or_insert(0) -= 1;
            self.ops += 1;
        }
        got
    }
}

fn count_map(root: &Link, k: i64, undo: bool) -> (i64, u64) {
    let mut w = PrefixWalk {
        k,
        undo,
        seen: HashMap::from([(0, 1)]), // the empty prefix, so a path may start at the root
        ops: 0,
    };
    let count = w.dfs(root.as_deref(), 0);
    (count, w.ops)
}

// ---- (c) the best bent path ------------------------------------------------------

// max_path_sum: down(n) is the best DOWNWARD path starting at n, never below 0
// (a negative branch is just not taken). The best BENT path through n is
// n + left gain + right gain. `init` is the starting value of the running best:
// None is right, Some(0) is the bug. The answer is an Option too: an empty tree
// has no path at all, which Go can only say with a sentinel.
fn max_path_sum(root: &Link, init: Option<i64>) -> Option<i64> {
    fn down(n: Option<&Node>, best: &mut Option<i64>) -> i64 {
        let Some(n) = n else { return 0 };
        let l = down(n.left.as_deref(), best);
        let r = down(n.right.as_deref(), best);
        let bent = n.val + l + r;
        *best = Some(best.map_or(bent, |b| b.max(bent)));
        0.max(n.val + l.max(r))
    }
    let mut best = init;
    down(root.as_deref(), &mut best);
    best
}

// max_path_oracle sums the path between EVERY pair of nodes (a node with itself included).
fn max_path_oracle(root: &Link) -> Option<i64> {
    fn walk(
        n: Option<&Node>,
        p: Option<usize>,
        d: usize,
        t: &mut Vec<(i64, Option<usize>, usize)>,
    ) {
        let Some(n) = n else { return };
        let me = t.len();
        t.push((n.val, p, d));
        walk(n.left.as_deref(), Some(me), d + 1, t);
        walk(n.right.as_deref(), Some(me), d + 1, t);
    }
    let mut t = Vec::new(); // (value, parent index, depth) in pre-order
    walk(root.as_deref(), None, 0, &mut t);
    let mut best = None;
    for a in 0..t.len() {
        for b in a..t.len() {
            let (mut x, mut y, mut s) = (a, b, 0);
            while x != y {
                // climb the deeper end until the two ends meet
                if t[x].2 >= t[y].2 {
                    s += t[x].0;
                    x = t[x].1.expect("a non-root node has a parent");
                } else {
                    s += t[y].0;
                    y = t[y].1.expect("a non-root node has a parent");
                }
            }
            let s = s + t[x].0;
            best = Some(best.map_or(s, |b: i64| b.max(s)));
        }
    }
    best
}

fn main() {
    let mut r = Pcg::new(1, 2);

    println!("== (a) root-to-leaf: has_path_sum on random_split(5000), values in [-5, 5]");
    let mut big = random_split(5000, &mut r);
    label_range(&mut big, &mut r, -5, 5);
    let mut sums = Vec::new();
    leaf_sums(big.as_deref(), 0, &mut sums);
    let have = sums[sums.len() / 2];
    let missing = sums.iter().max().expect("a tree has a leaf") + 1;
    println!(
        "{} leaves; the target that exists is the sum at the middle leaf; first leaf with that sum is #{}",
        sums.len(),
        sums.iter()
            .position(|&s| s == have)
            .expect("have is a leaf sum")
    );
    println!(
        "{:<8} {:>6} {:>6} {:>12} {:>10}",
        "target", "value", "found", "early exit", "full walk"
    );
    for (name, t) in [("exists", have), ("absent", missing)] {
        let (mut early, mut full) = (0, 0);
        let ok = has_path_sum(big.as_deref(), t, &mut early, true);
        let ok2 = has_path_sum(big.as_deref(), t, &mut full, false);
        println!(
            "{name:<8} {t:>6} {ok:>6} {early:>12} {full:>10}   agree: {}",
            ok == ok2
        );
    }

    println!(
        "\n== (b) downward paths summing to k: brute vs prefix map, 2000 small trees (n <= 60, k in [-10, 10])"
    );
    let (mut wrong_undo, mut wrong_no, mut smallest) = (0, 0, 0);
    let mut example = String::new();
    for _ in 0..2000 {
        let n = r.int_n(60) as usize + 1;
        let mut tree = random_split(n, &mut r);
        label_range(&mut tree, &mut r, -5, 5);
        let k = r.int_n(21) as i64 - 10;
        let (want, _) = count_brute(&tree, k);
        let (with_undo, _) = count_map(&tree, k, true);
        let (no_undo, _) = count_map(&tree, k, false);
        if with_undo != want {
            wrong_undo += 1;
        }
        if no_undo != want {
            wrong_no += 1;
            if smallest == 0 || n < smallest {
                smallest = n;
                example = format!(
                    "{}  k={k}  brute={want}  no-undo={no_undo}",
                    show(tree.as_deref())
                );
            }
        }
    }
    println!("map with undo:    {wrong_undo} mismatches");
    println!(
        "map without undo: wrong on {wrong_no} of 2000 trees; smallest failing tree:\n  {example}"
    );

    println!(
        "\n{:<19} {:>6} {:>3} {:>8} {:>12} {:>9} {:>12} {:>6}",
        "tree", "n", "k", "paths", "brute steps", "map ops", "no-undo says", "agree"
    );
    let mut chain = left_chain(2000);
    label_range(&mut chain, &mut r, -5, 5);
    let mut random = random_split(50000, &mut r);
    label_range(&mut random, &mut r, -5, 5);
    for (name, n, root) in [
        ("random_split(50000)", 50000, &random),
        ("left_chain(2000)", 2000, &chain),
    ] {
        const K: i64 = 7;
        let (want, steps) = count_brute(root, K);
        let (got, ops) = count_map(root, K, true);
        let (bad, _) = count_map(root, K, false);
        println!(
            "{name:<19} {n:>6} {K:>3} {want:>8} {steps:>12} {ops:>9} {bad:>12} {:>6}",
            got == want
        );
        let verdict = if bad == want {
            "right (a chain has no siblings)".to_string()
        } else {
            format!("{:.0}x too big", bad as f64 / want as f64)
        };
        println!(
            "  brute steps / map ops = {:.1};  no-undo count is {verdict}",
            steps as f64 / ops as f64
        );
    }
    println!("n(n+1)/2 for the 2000-chain = {}", 2000 * 2001 / 2);

    println!(
        "\n== (c) maximum path sum against all node pairs: 1500 mixed trees, 500 all-negative (n <= 40)"
    );
    let (mut wrong_good, mut wrong_zero_mixed, mut wrong_zero_neg) = (0, 0, 0);
    for t in 0..2000 {
        let mut tree = random_split(r.int_n(40) as usize + 1, &mut r);
        if t < 1500 {
            label_range(&mut tree, &mut r, -5, 5);
        } else {
            label_range(&mut tree, &mut r, -9, -1);
        }
        let want = max_path_oracle(&tree);
        if max_path_sum(&tree, None) != want {
            wrong_good += 1;
        }
        if max_path_sum(&tree, Some(0)) != want {
            if t < 1500 {
                wrong_zero_mixed += 1;
            } else {
                wrong_zero_neg += 1;
            }
        }
    }
    println!("best starts at None:   {wrong_good} mismatches");
    println!(
        "best starts at Some(0): wrong on {wrong_zero_mixed} of 1500 mixed trees and on {wrong_zero_neg} of 500 all-negative trees"
    );
    let neg = node(
        -3,
        node(-2, node(-5, None, None), node(-1, None, None)),
        node(-4, None, None),
    );
    println!(
        "tree {}: right answer {} (the single node -1), \"best = Some(0)\" says {}",
        show(neg.as_deref()),
        max_path_sum(&neg, None).expect("the tree has nodes"),
        max_path_sum(&neg, Some(0)).expect("the tree has nodes")
    );
}
```

**Rust output:**

```
== (a) root-to-leaf: has_path_sum on random_split(5000), values in [-5, 5]
1700 leaves; the target that exists is the sum at the middle leaf; first leaf with that sum is #15
target    value  found   early exit  full walk
exists        1   true           49       5000   agree: true
absent       41  false         5000       5000   agree: true

== (b) downward paths summing to k: brute vs prefix map, 2000 small trees (n <= 60, k in [-10, 10])
map with undo:    0 mismatches
map without undo: wrong on 1603 of 2000 trees; smallest failing tree:
  -4(-5(-,-1),-4(-,-5))  k=2  brute=0  no-undo=1

tree                     n   k    paths  brute steps   map ops no-undo says  agree
random_split(50000)  50000   7    25515       952310    150000     25514647   true
  brute steps / map ops = 6.3;  no-undo count is 1000x too big
left_chain(2000)      2000   7    14826      2001000      6000        14826   true
  brute steps / map ops = 333.5;  no-undo count is right (a chain has no siblings)
n(n+1)/2 for the 2000-chain = 2001000

== (c) maximum path sum against all node pairs: 1500 mixed trees, 500 all-negative (n <= 40)
best starts at None:   0 mismatches
best starts at Some(0): wrong on 39 of 1500 mixed trees and on 500 of 500 all-negative trees
tree -3(-2(-5,-1),-4): right answer -1 (the single node -1), "best = Some(0)" says 0
```

> **Go vs Rust:** Diffed against the Go run: every count, mismatch total, step count and map-operation count is identical, because both programs build the same trees from `PCG(1, 2)`; only the spelled names (`has_path_sum`, `random_split`) and the `None` / `Some(0)` wording of section (c) differ. A Rust closure cannot call itself, so the recursion is a method on a `PrefixWalk` struct that holds the map and the operation counter and is borrowed `&mut self` down the tree, where Go's recursive closure captures `seen` and `ops`. The running best is an `Option<i64>` (an empty tree has no path), where Go starts from `math.MinInt`; "start at 0" is the same bug in both.

**Complexity:** root-to-leaf Θ(n) time, Θ(h) stack — the early exit found its leaf after 49 of 5,000 nodes, an absent target costs all 5,000 · downward paths summing to k: brute force is O(n·h) steps (952,310 on the random 50,000-node tree, n(n+1)/2 = 2,001,000 on the 2,000-chain) against **Θ(n) map operations, 3n** (150,000 and 6,000); without the undo the map was wrong on 1,603 of 2,000 small trees, and 1000x too big on the 50,000-node tree, though right on the chain, which has no siblings · maximum path Θ(n): 0 mismatches against all pairs on 2,000 trees, while `best := 0` is wrong on all 500 all-negative trees (and on 39 of the 1,500 mixed ones).

---

## 8. Two trees at once: same, mirror, subtree

`🟡 medium` · *Lockstep recursion*

*Same*, *mirror* and *symmetric* are one recursion over a **pair** of nodes: both empty is true, exactly one empty is false, otherwise compare the values and recurse on the matching (or crossed) children, Θ(min(n, m)). "Is T a subtree of S" runs that test at every node of S, which is Θ(|S|·|T|) in the worst case, and an adversarial input reaches it. A subtree hash (a Merkle hash: each node's hash mixes its value and its children's hashes) is computed for every subtree in one post-order pass, Θ(n + m), and turns the candidate check into one integer compare, with a real comparison only to confirm a hit. The program checks `same` and `mirror` against an oracle built from serializations, then counts node comparisons.

**Steps:**

1. Check `same`, `mirror` and the language's own structural equality against serialization oracles on 3,000 random pairs (identical copies, one value changed, mirrored copies, unrelated trees) and count how many of each class.
2. Build the adversarial input (a 2,000-node chain of ones against a 1,000-node chain of ones ending in a 2), count the comparisons of the naive `isSubtree`, and compare with the hashed version, which does one hash per node and confirms only the nodes whose hash matches.
3. Truncate the hash to 8 bits to see what the confirmation step is for, and fuzz all three subtree tests against a serialization oracle.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"reflect"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func node(v int, l, r *Node) *Node { return &Node{v, l, r} }

// randomSplit builds a SHAPE only (every Val is 0); the draw comes BEFORE the recursion so
// the Rust twin consumes the PCG stream in the same order and builds the identical tree.
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelRange gives every node a value in [lo, hi], drawn in pre-order, in a pass of its own.
func labelRange(n *Node, r *rand.Rand, lo, hi int) {
	if n == nil {
		return
	}
	n.Val = r.IntN(hi-lo+1) + lo
	labelRange(n.Left, r, lo, hi)
	labelRange(n.Right, r, lo, hi)
}

// chain builds n nodes in a loop, each the Left child of the one above: every Val is 1
// except the bottom node's, which is `bottom`.
func chain(n, bottom int) *Node {
	root := &Node{bottom, nil, nil}
	for i := 1; i < n; i++ {
		root = &Node{1, root, nil}
	}
	return root
}

func show(n *Node) string {
	if n == nil {
		return "-"
	}
	if n.Left == nil && n.Right == nil {
		return fmt.Sprint(n.Val)
	}
	return fmt.Sprintf("%d(%s,%s)", n.Val, show(n.Left), show(n.Right))
}

func clone(n *Node) *Node {
	if n == nil {
		return nil
	}
	return &Node{n.Val, clone(n.Left), clone(n.Right)}
}

// invertCopy is the oracle's mirror image: a NEW tree with every pair of children swapped.
func invertCopy(n *Node) *Node {
	if n == nil {
		return nil
	}
	return &Node{n.Val, invertCopy(n.Right), invertCopy(n.Left)}
}

// nodes lists every node in pre-order.
func nodes(n *Node) []*Node {
	if n == nil {
		return nil
	}
	return append(append([]*Node{n}, nodes(n.Left)...), nodes(n.Right)...)
}

// serial is the oracle's view of a tree: pre-order with a '#' for every nil, so two trees
// are the same exactly when their strings are.
func serial(n *Node) string {
	if n == nil {
		return "#,"
	}
	return fmt.Sprintf("%d,", n.Val) + serial(n.Left) + serial(n.Right)
}

// ---- (a) one recursion over a PAIR of nodes -------------------------------------------

// same: both nil is true, exactly one nil is false (a == b compares the two pointers, so
// this is the pair of base cases in one line), otherwise compare the values and recurse on
// the matching children. cmps counts the node-against-node value comparisons.
func same(a, b *Node, cmps *int) bool {
	if a == nil || b == nil {
		return a == b
	}
	*cmps++
	return a.Val == b.Val && same(a.Left, b.Left, cmps) && same(a.Right, b.Right, cmps)
}

// mirror is the same recursion with the children crossed.
func mirror(a, b *Node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Val == b.Val && mirror(a.Left, b.Right) && mirror(a.Right, b.Left)
}

func symmetric(t *Node) bool { return t == nil || mirror(t.Left, t.Right) }

// ---- (b) is T a subtree of S: the same test at every node of S ------------------------

func isSubtree(s, t *Node, cmps *int) bool {
	if t == nil {
		return true
	}
	if s == nil {
		return false
	}
	return same(s, t, cmps) || isSubtree(s.Left, t, cmps) || isSubtree(s.Right, t, cmps)
}

// ---- (c) a subtree hash turns the candidate check into one integer compare ---------------

// mix is the splitmix64 finalizer, written out so both languages print the same numbers
// (not maphash and not Rust's DefaultHasher: their seed or algorithm is not ours to fix).
func mix(x uint64) uint64 {
	x ^= x >> 30
	x *= 0xbf58476d1ce4e5b9
	x ^= x >> 27
	x *= 0x94d049bb133111eb
	return x ^ x>>31
}

const nilHash = 1 // the hash of the empty tree: a constant

// nodeHash is order-sensitive: the left hash is XORed in, the right one added, so
// swapping the children changes the result.
func nodeHash(val int, hl, hr uint64) uint64 {
	h := mix(uint64(val) + 0x9e3779b97f4a7c15)
	h = mix(h ^ hl)
	return mix(h + hr)
}

type cand struct {
	n *Node
	h uint64
}

// hashAll computes the hash of EVERY subtree in one post-order pass and records (node, hash).
func hashAll(n *Node, out *[]cand, calls *int) uint64 {
	if n == nil {
		return nilHash
	}
	hl := hashAll(n.Left, out, calls)
	hr := hashAll(n.Right, out, calls)
	*calls++
	h := nodeHash(n.Val, hl, hr)
	*out = append(*out, cand{n, h})
	return h
}

type stats struct{ hashes, cands, matches, cmps int }

// isSubtreeHashed: hash all of S and T, then confirm every node of S whose hash equals T's
// (low `bits` bits only) with same(). bits=64 is the real thing; bits=8 is a deliberately
// weak hash that shows why the confirmation cannot be skipped.
func isSubtreeHashed(s, t *Node, bits uint) (found bool, st stats) {
	var ss, ts []cand
	hashAll(s, &ss, &st.hashes)
	hashAll(t, &ts, &st.hashes)
	mask := ^uint64(0) >> (64 - bits)
	want := ts[len(ts)-1].h & mask
	for _, c := range ss {
		if c.h&mask == want {
			st.cands++
			if same(c.n, t, &st.cmps) {
				st.matches++
			}
		}
	}
	return st.matches > 0, st
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	var scratch int

	fmt.Println("== (a) same / mirror against the oracle on 3000 random pairs (n <= 30, values 0..3)")
	classes := []string{"identical copy", "one value changed", "mirrored copy", "unrelated"}
	var pairs, sameTrue, mirrorTrue [4]int
	badSame, badMirror, badLib := 0, 0, 0
	for t := 0; t < 3000; t++ {
		n := r.IntN(30) + 1
		a := randomSplit(n, r)
		labelRange(a, r, 0, 3)
		var b *Node
		c := r.IntN(4)
		switch c {
		case 0:
			b = clone(a)
		case 1:
			b = clone(a)
			ns := nodes(b)
			x := ns[r.IntN(len(ns))]
			x.Val = (x.Val + 1 + r.IntN(3)) % 4 // a different value in 0..3
		case 2:
			b = invertCopy(a)
		case 3:
			b = randomSplit(r.IntN(30)+1, r)
			labelRange(b, r, 0, 3)
		}
		gotSame, gotMirror := same(a, b, &scratch), mirror(a, b)
		wantSame, wantMirror := serial(a) == serial(b), serial(a) == serial(invertCopy(b))
		pairs[c]++
		if gotSame {
			sameTrue[c]++
		}
		if gotMirror {
			mirrorTrue[c]++
		}
		if gotSame != wantSame {
			badSame++
		}
		if gotMirror != wantMirror {
			badMirror++
		}
		if reflect.DeepEqual(a, b) != gotSame {
			badLib++
		}
	}
	fmt.Printf("%-18s %6s %10s %12s\n", "class", "pairs", "same true", "mirror true")
	for c, name := range classes {
		fmt.Printf("%-18s %6d %10d %12d\n", name, pairs[c], sameTrue[c], mirrorTrue[c])
	}
	fmt.Printf("same vs equal serializations: %d mismatches; mirror vs serialization of the inverted copy: %d mismatches\n", badSame, badMirror)
	fmt.Printf("the language's own structural equality vs same(): %d disagreements\n", badLib)
	for _, t := range []*Node{
		node(1, node(2, node(3, nil, nil), node(4, nil, nil)), node(2, node(4, nil, nil), node(3, nil, nil))),
		node(1, node(2, node(3, nil, nil), node(4, nil, nil)), node(2, node(3, nil, nil), node(4, nil, nil))),
		node(1, node(2, nil, nil), nil),
	} {
		fmt.Printf("symmetric(%s) = %v\n", show(t), symmetric(t))
	}

	fmt.Println("\n== (b)+(c) is T a subtree of S: every start point vs a subtree hash")
	adv := [2]*Node{chain(2000, 1), chain(1000, 2)}
	rs := randomSplit(2000, r)
	labelRange(rs, r, 0, 3)
	var pick *Node // the node of S whose subtree is closest to 50 nodes
	for _, x := range nodes(rs) {
		if m := len(nodes(x)); pick == nil || abs(m-50) < abs(len(nodes(pick))-50) {
			pick = x
		}
	}
	other := randomSplit(50, r)
	labelRange(other, r, 0, 3)
	cases := []struct {
		name string
		s, t *Node
	}{
		{"adversarial chains", adv[0], adv[1]},
		{"random S, T inside", rs, clone(pick)},
		{"random S, T absent", rs, other},
	}
	fmt.Printf("%-19s %5s %5s %12s %5s %10s %6s %12s\n", "case", "|S|", "|T|", "naive cmps", "bits", "hash calls", "cands", "confirm cmps")
	allAgree, falseCands, advNaive := true, 0, 0
	for i, c := range cases {
		var naive int
		want := isSubtree(c.s, c.t, &naive)
		if i == 0 {
			advNaive = naive
		}
		for _, bits := range []uint{64, 8} {
			found, st := isSubtreeHashed(c.s, c.t, bits)
			fmt.Printf("%-19s %5d %5d %12d %5d %10d %6d %12d   matches=%d found=%v\n", c.name, len(nodes(c.s)), len(nodes(c.t)), naive, bits, st.hashes, st.cands, st.cmps, st.matches, found)
			allAgree = allAgree && found == want
			if bits == 8 {
				falseCands += st.cands - st.matches
			}
		}
	}
	fmt.Printf("hashed answers equal the naive answer in all 6 rows: %v; the 8-bit hash passed %d false candidates and same() rejected them\n", allAgree, falseCands)
	n, m := 2000, 1000
	fmt.Printf("adversarial n=%d m=%d: measured %d; (n-m+1)*m + m(m-1)/2 = %d; for scale n*m = %d, (n-m)*m = %d\n",
		n, m, advNaive, (n-m+1)*m+m*(m-1)/2, n*m, (n-m)*m)

	// Oracle: T is a subtree of S exactly when T's serialization equals that of some node of S.
	inside, bad := 0, 0
	for t := 0; t < 2000; t++ {
		s := randomSplit(r.IntN(40)+1, r)
		labelRange(s, r, 0, 1)
		var tt *Node
		if r.IntN(2) == 0 {
			ns := nodes(s)
			tt = clone(ns[r.IntN(len(ns))])
		} else {
			tt = randomSplit(r.IntN(6)+1, r)
			labelRange(tt, r, 0, 1)
		}
		want := false
		for _, x := range nodes(s) {
			want = want || serial(x) == serial(tt)
		}
		g1 := isSubtree(s, tt, &scratch)
		g2, _ := isSubtreeHashed(s, tt, 64)
		g3, _ := isSubtreeHashed(s, tt, 8)
		if want {
			inside++
		}
		if g1 != want || g2 != want || g3 != want {
			bad++
		}
	}
	fmt.Printf("fuzz: T inside S in %d of 2000 pairs; naive, 64-bit hash and 8-bit hash vs the serialization oracle: %d mismatches\n", inside, bad)
}

func abs(x int) int {
	if x < 0 {
		return -x
	}
	return x
}
```

**Output:**

```
== (a) same / mirror against the oracle on 3000 random pairs (n <= 30, values 0..3)
class               pairs  same true  mirror true
identical copy        711        711           28
one value changed     757          0            0
mirrored copy         758         19          758
unrelated             774          0            1
same vs equal serializations: 0 mismatches; mirror vs serialization of the inverted copy: 0 mismatches
the language's own structural equality vs same(): 0 disagreements
symmetric(1(2(3,4),2(4,3))) = true
symmetric(1(2(3,4),2(3,4))) = false
symmetric(1(2,-)) = false

== (b)+(c) is T a subtree of S: every start point vs a subtree hash
case                  |S|   |T|   naive cmps  bits hash calls  cands confirm cmps
adversarial chains   2000  1000      1500500    64       3000      0            0   matches=0 found=false
adversarial chains   2000  1000      1500500     8       3000      9         5482   matches=0 found=false
random S, T inside   2000    50          480    64       2050      1           50   matches=1 found=true
random S, T inside   2000    50          480     8       2050      3           52   matches=1 found=true
random S, T absent   2000    50         2308    64       2050      0            0   matches=0 found=false
random S, T absent   2000    50         2308     8       2050      8           11   matches=0 found=false
hashed answers equal the naive answer in all 6 rows: true; the 8-bit hash passed 19 false candidates and same() rejected them
adversarial n=2000 m=1000: measured 1500500; (n-m+1)*m + m(m-1)/2 = 1500500; for scale n*m = 2000000, (n-m)*m = 1000000
fuzz: T inside S in 1223 of 2000 pairs; naive, 64-bit hash and 8-bit hash vs the serialization oracle: 0 mismatches
```

**Rust:**

```rust
// Pcg: the port of Go's math/rand/v2 PCG-DXSM generator and its IntN from lesson 01
// example 14, so seed (1, 2) builds exactly the trees the Go program builds.
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

// An empty tree is None; a node owns its two subtrees through Box.
// derive(PartialEq) gives structural == for free: it compares val, then left, then
// right, recursively, so it is as deep as the tree (fine here; never run it on a chain).
#[derive(PartialEq)]
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// A chain built in a loop is n levels deep, and the compiler's default drop glue
// would recurse once per level. This Drop moves the children onto a heap stack
// instead (lesson 05 example 15). Go has no destructor to write.
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

fn node(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

fn leaf(val: i64) -> Link {
    node(val, None, None)
}

// random_split builds a SHAPE only (every val is 0); the draw comes BEFORE the recursion so
// both languages consume the PCG stream in the same order and build the identical tree.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n as u64) as usize;
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    node(0, left, right)
}

// label_range gives every node a value in [lo, hi], drawn in pre-order, in a pass of its own.
fn label_range(n: &mut Link, r: &mut Pcg, lo: i64, hi: i64) {
    if let Some(b) = n {
        b.val = r.int_n((hi - lo + 1) as u64) as i64 + lo;
        label_range(&mut b.left, r, lo, hi);
        label_range(&mut b.right, r, lo, hi);
    }
}

// chain builds n nodes in a loop, each the left child of the one above: every val is 1
// except the bottom node's, which is `bottom`.
fn chain(n: usize, bottom: i64) -> Link {
    let mut root = leaf(bottom);
    for _ in 1..n {
        root = node(1, root, None);
    }
    root
}

fn show(n: Option<&Node>) -> String {
    match n {
        None => "-".to_string(),
        Some(x) if x.left.is_none() && x.right.is_none() => x.val.to_string(),
        Some(x) => format!(
            "{}({},{})",
            x.val,
            show(x.left.as_deref()),
            show(x.right.as_deref())
        ),
    }
}

fn clone_tree(n: Option<&Node>) -> Link {
    n.map(|x| {
        Box::new(Node {
            val: x.val,
            left: clone_tree(x.left.as_deref()),
            right: clone_tree(x.right.as_deref()),
        })
    })
}

// invert_copy is the oracle's mirror image: a NEW tree with every pair of children swapped.
fn invert_copy(n: Option<&Node>) -> Link {
    n.map(|x| {
        Box::new(Node {
            val: x.val,
            left: invert_copy(x.right.as_deref()),
            right: invert_copy(x.left.as_deref()),
        })
    })
}

// nodes lists every node in pre-order, as references into the tree.
fn nodes(n: Option<&Node>) -> Vec<&Node> {
    let Some(x) = n else { return Vec::new() };
    let mut v = vec![x];
    v.extend(nodes(x.left.as_deref()));
    v.extend(nodes(x.right.as_deref()));
    v
}

// bump_nth changes the value of the j-th node in pre-order. Go takes the *Node out of a
// slice and assigns through it; a Vec<&Node> cannot be written through, so the
// mutation walks the tree with `&mut`.
fn bump_nth(n: &mut Link, j: &mut usize, bump: i64) -> bool {
    let Some(b) = n else { return false };
    if *j == 0 {
        b.val = (b.val + bump) % 4;
        return true;
    }
    *j -= 1;
    bump_nth(&mut b.left, j, bump) || bump_nth(&mut b.right, j, bump)
}

// serial is the oracle's view of a tree: pre-order with a '#' for every nil, so two trees
// are the same exactly when their strings are.
fn serial(n: Option<&Node>) -> String {
    match n {
        None => "#,".to_string(),
        Some(x) => format!(
            "{},{}{}",
            x.val,
            serial(x.left.as_deref()),
            serial(x.right.as_deref())
        ),
    }
}

// ---- (a) one recursion over a PAIR of nodes -------------------------------------------

// same: the four cases of a pair, in one match. cmps counts the node-against-node
// value comparisons.
fn same(a: Option<&Node>, b: Option<&Node>, cmps: &mut u64) -> bool {
    match (a, b) {
        (None, None) => true,
        (Some(x), Some(y)) => {
            *cmps += 1;
            x.val == y.val
                && same(x.left.as_deref(), y.left.as_deref(), cmps)
                && same(x.right.as_deref(), y.right.as_deref(), cmps)
        }
        _ => false,
    }
}

// mirror is the same recursion with the children crossed.
fn mirror(a: Option<&Node>, b: Option<&Node>) -> bool {
    match (a, b) {
        (None, None) => true,
        (Some(x), Some(y)) => {
            x.val == y.val
                && mirror(x.left.as_deref(), y.right.as_deref())
                && mirror(x.right.as_deref(), y.left.as_deref())
        }
        _ => false,
    }
}

fn symmetric(t: &Link) -> bool {
    match t {
        None => true,
        Some(x) => mirror(x.left.as_deref(), x.right.as_deref()),
    }
}

// ---- (b) is T a subtree of S: the same test at every node of S ------------------------

fn is_subtree(s: Option<&Node>, t: Option<&Node>, cmps: &mut u64) -> bool {
    if t.is_none() {
        return true;
    }
    let Some(sn) = s else { return false };
    same(s, t, cmps)
        || is_subtree(sn.left.as_deref(), t, cmps)
        || is_subtree(sn.right.as_deref(), t, cmps)
}

// ---- (c) a subtree hash turns the candidate check into one integer compare ---------------

// mix is the splitmix64 finalizer, written out so both languages print the same numbers
// (not Rust's DefaultHasher, whose algorithm is unspecified, nor Go's maphash, which is seeded).
fn mix(mut x: u64) -> u64 {
    x ^= x >> 30;
    x = x.wrapping_mul(0xbf58476d1ce4e5b9);
    x ^= x >> 27;
    x = x.wrapping_mul(0x94d049bb133111eb);
    x ^ (x >> 31)
}

const NIL_HASH: u64 = 1; // the hash of the empty tree: a constant

// node_hash is order-sensitive: the left hash is XORed in, the right one added, so
// swapping the children changes the result. Overflow must wrap: write it out.
fn node_hash(val: i64, hl: u64, hr: u64) -> u64 {
    let mut h = mix((val as u64).wrapping_add(0x9e3779b97f4a7c15));
    h = mix(h ^ hl);
    mix(h.wrapping_add(hr))
}

// hash_all computes the hash of EVERY subtree in one post-order pass and records
// (node, hash). The references borrow the tree, so the list cannot outlive it.
fn hash_all<'a>(n: Option<&'a Node>, out: &mut Vec<(&'a Node, u64)>, calls: &mut u64) -> u64 {
    let Some(x) = n else { return NIL_HASH };
    let hl = hash_all(x.left.as_deref(), out, calls);
    let hr = hash_all(x.right.as_deref(), out, calls);
    *calls += 1;
    let h = node_hash(x.val, hl, hr);
    out.push((x, h));
    h
}

#[derive(Default)]
struct Stats {
    hashes: u64,
    cands: u64,
    matches: u64,
    cmps: u64,
}

// is_subtree_hashed: hash all of S and T, then confirm every node of S whose hash equals T's
// (low `bits` bits only) with same(). bits=64 is the real thing; bits=8 is a deliberately
// weak hash that shows why the confirmation cannot be skipped.
fn is_subtree_hashed(s: &Link, t: &Link, bits: u32) -> (bool, Stats) {
    let mut st = Stats::default();
    let (mut ss, mut ts) = (Vec::new(), Vec::new());
    hash_all(s.as_deref(), &mut ss, &mut st.hashes);
    hash_all(t.as_deref(), &mut ts, &mut st.hashes);
    let mask = u64::MAX >> (64 - bits);
    let want = ts.last().expect("T is not empty").1 & mask;
    for &(n, h) in &ss {
        if h & mask == want {
            st.cands += 1;
            if same(Some(n), t.as_deref(), &mut st.cmps) {
                st.matches += 1;
            }
        }
    }
    (st.matches > 0, st)
}

fn main() {
    let mut r = Pcg::new(1, 2);
    let mut scratch = 0;

    println!("== (a) same / mirror against the oracle on 3000 random pairs (n <= 30, values 0..3)");
    let classes = [
        "identical copy",
        "one value changed",
        "mirrored copy",
        "unrelated",
    ];
    let (mut pairs, mut same_true, mut mirror_true) = ([0; 4], [0; 4], [0; 4]);
    let (mut bad_same, mut bad_mirror, mut bad_lib) = (0, 0, 0);
    for _ in 0..3000 {
        let n = r.int_n(30) as usize + 1;
        let mut a = random_split(n, &mut r);
        label_range(&mut a, &mut r, 0, 3);
        let c = r.int_n(4) as usize;
        let b = match c {
            0 => clone_tree(a.as_deref()),
            1 => {
                let mut b = clone_tree(a.as_deref());
                let mut j = r.int_n(n as u64) as usize;
                let bump = 1 + r.int_n(3) as i64; // a different value in 0..3
                bump_nth(&mut b, &mut j, bump);
                b
            }
            2 => invert_copy(a.as_deref()),
            _ => {
                let mut b = random_split(r.int_n(30) as usize + 1, &mut r);
                label_range(&mut b, &mut r, 0, 3);
                b
            }
        };
        let got_same = same(a.as_deref(), b.as_deref(), &mut scratch);
        let got_mirror = mirror(a.as_deref(), b.as_deref());
        let want_same = serial(a.as_deref()) == serial(b.as_deref());
        let want_mirror = serial(a.as_deref()) == serial(invert_copy(b.as_deref()).as_deref());
        pairs[c] += 1;
        same_true[c] += usize::from(got_same);
        mirror_true[c] += usize::from(got_mirror);
        bad_same += usize::from(got_same != want_same);
        bad_mirror += usize::from(got_mirror != want_mirror);
        bad_lib += usize::from((a == b) != got_same);
    }
    println!(
        "{:<18} {:>6} {:>10} {:>12}",
        "class", "pairs", "same true", "mirror true"
    );
    for (c, name) in classes.iter().enumerate() {
        println!(
            "{name:<18} {:>6} {:>10} {:>12}",
            pairs[c], same_true[c], mirror_true[c]
        );
    }
    println!(
        "same vs equal serializations: {bad_same} mismatches; mirror vs serialization of the inverted copy: {bad_mirror} mismatches"
    );
    println!("the language's own structural equality vs same(): {bad_lib} disagreements");
    for t in [
        node(1, node(2, leaf(3), leaf(4)), node(2, leaf(4), leaf(3))),
        node(1, node(2, leaf(3), leaf(4)), node(2, leaf(3), leaf(4))),
        node(1, leaf(2), None),
    ] {
        println!("symmetric({}) = {}", show(t.as_deref()), symmetric(&t));
    }

    println!("\n== (b)+(c) is T a subtree of S: every start point vs a subtree hash");
    let (adv_s, adv_t) = (chain(2000, 1), chain(1000, 2));
    let mut rs = random_split(2000, &mut r);
    label_range(&mut rs, &mut r, 0, 3);
    // the node of S whose subtree is closest to 50 nodes
    let mut pick: Option<&Node> = None;
    for x in nodes(rs.as_deref()) {
        let m = nodes(Some(x)).len() as i64;
        if pick.is_none_or(|p| (m - 50).abs() < (nodes(Some(p)).len() as i64 - 50).abs()) {
            pick = Some(x);
        }
    }
    let inside = clone_tree(pick);
    let mut other = random_split(50, &mut r);
    label_range(&mut other, &mut r, 0, 3);
    let cases = [
        ("adversarial chains", &adv_s, &adv_t),
        ("random S, T inside", &rs, &inside),
        ("random S, T absent", &rs, &other),
    ];
    println!(
        "{:<19} {:>5} {:>5} {:>12} {:>5} {:>10} {:>6} {:>12}",
        "case", "|S|", "|T|", "naive cmps", "bits", "hash calls", "cands", "confirm cmps"
    );
    let (mut all_agree, mut false_cands, mut adv_naive) = (true, 0, 0);
    for (i, &(name, s, t)) in cases.iter().enumerate() {
        let mut naive = 0;
        let want = is_subtree(s.as_deref(), t.as_deref(), &mut naive);
        if i == 0 {
            adv_naive = naive;
        }
        for bits in [64, 8] {
            let (found, st) = is_subtree_hashed(s, t, bits);
            println!(
                "{name:<19} {:>5} {:>5} {naive:>12} {bits:>5} {:>10} {:>6} {:>12}   matches={} found={found}",
                nodes(s.as_deref()).len(),
                nodes(t.as_deref()).len(),
                st.hashes,
                st.cands,
                st.cmps,
                st.matches
            );
            all_agree = all_agree && found == want;
            if bits == 8 {
                false_cands += st.cands - st.matches;
            }
        }
    }
    println!(
        "hashed answers equal the naive answer in all 6 rows: {all_agree}; the 8-bit hash passed {false_cands} false candidates and same() rejected them"
    );
    let (n, m) = (2000u64, 1000u64);
    println!(
        "adversarial n={n} m={m}: measured {adv_naive}; (n-m+1)*m + m(m-1)/2 = {}; for scale n*m = {}, (n-m)*m = {}",
        (n - m + 1) * m + m * (m - 1) / 2,
        n * m,
        (n - m) * m
    );

    // Oracle: T is a subtree of S exactly when T's serialization equals that of some node of S.
    let (mut inside_count, mut bad) = (0, 0);
    for _ in 0..2000 {
        let mut s = random_split(r.int_n(40) as usize + 1, &mut r);
        label_range(&mut s, &mut r, 0, 1);
        let tt = if r.int_n(2) == 0 {
            let ns = nodes(s.as_deref());
            clone_tree(Some(ns[r.int_n(ns.len() as u64) as usize]))
        } else {
            let mut t = random_split(r.int_n(6) as usize + 1, &mut r);
            label_range(&mut t, &mut r, 0, 1);
            t
        };
        let want = nodes(s.as_deref())
            .iter()
            .any(|&x| serial(Some(x)) == serial(tt.as_deref()));
        let g1 = is_subtree(s.as_deref(), tt.as_deref(), &mut scratch);
        let (g2, _) = is_subtree_hashed(&s, &tt, 64);
        let (g3, _) = is_subtree_hashed(&s, &tt, 8);
        inside_count += usize::from(want);
        bad += usize::from(g1 != want || g2 != want || g3 != want);
    }
    println!(
        "fuzz: T inside S in {inside_count} of 2000 pairs; naive, 64-bit hash and 8-bit hash vs the serialization oracle: {bad} mismatches"
    );
}
```

**Rust output:**

```
== (a) same / mirror against the oracle on 3000 random pairs (n <= 30, values 0..3)
class               pairs  same true  mirror true
identical copy        711        711           28
one value changed     757          0            0
mirrored copy         758         19          758
unrelated             774          0            1
same vs equal serializations: 0 mismatches; mirror vs serialization of the inverted copy: 0 mismatches
the language's own structural equality vs same(): 0 disagreements
symmetric(1(2(3,4),2(4,3))) = true
symmetric(1(2(3,4),2(3,4))) = false
symmetric(1(2,-)) = false

== (b)+(c) is T a subtree of S: every start point vs a subtree hash
case                  |S|   |T|   naive cmps  bits hash calls  cands confirm cmps
adversarial chains   2000  1000      1500500    64       3000      0            0   matches=0 found=false
adversarial chains   2000  1000      1500500     8       3000      9         5482   matches=0 found=false
random S, T inside   2000    50          480    64       2050      1           50   matches=1 found=true
random S, T inside   2000    50          480     8       2050      3           52   matches=1 found=true
random S, T absent   2000    50         2308    64       2050      0            0   matches=0 found=false
random S, T absent   2000    50         2308     8       2050      8           11   matches=0 found=false
hashed answers equal the naive answer in all 6 rows: true; the 8-bit hash passed 19 false candidates and same() rejected them
adversarial n=2000 m=1000: measured 1500500; (n-m+1)*m + m(m-1)/2 = 1500500; for scale n*m = 2000000, (n-m)*m = 1000000
fuzz: T inside S in 1223 of 2000 pairs; naive, 64-bit hash and 8-bit hash vs the serialization oracle: 0 mismatches
```

> **Go vs Rust:** Diffed against the Go run: the output is identical, byte for byte, because both programs build their trees from `PCG(1, 2)` and print counts only. The pair recursion is one `match (a, b)` that lists the four cases and needs no `a == b` pointer trick to say "both nil"; `#[derive(PartialEq)]` on `Node` gives the structural `==` that Go gets from `reflect.DeepEqual`, which is checked against `same` on every pair (both recursive, so as deep as the tree; not run on a chain). Go edits a node through a `*Node` it pulled out of a slice, while `bump_nth` must walk the tree with `&mut`; and `hash_all` returns `(&Node, u64)` pairs that borrow the tree, so the compiler proves the candidate list cannot outlive it.

**Complexity:** same / mirror Θ(min(n, m)) time, Θ(h) stack · naive `isSubtree` **Θ(n·m) worst case: 1,500,500 comparisons** for a 2,000-node S and a 1,000-node T, exactly (n−m+1)·m + m(m−1)/2 (a rougher guess of (n−m)·m = 1,000,000 undercounts it, because the start points near the bottom still match a prefix of T), against 480 and 2,308 on random trees whose values are 0..3 · hashed Θ(n + m): 3,000 hash computations, 0 candidates and 0 confirmation comparisons on the adversarial input; with an 8-bit hash the 9 false candidates cost 5,482 comparisons, still about 270 times fewer than naive, and every row agrees with the naive answer.

---

## 9. Lowest common ancestor, and the missing node

`🟡 medium` · *Post-order with a bug*

The one-pass LCA is an elegant post-order recursion: return the node if it is `p` or `q`, report the current node when both subtrees return something, otherwise pass up whichever side found one. It assumes both nodes are in the tree, and when one is not it silently returns the other as though it were the answer. The fix (count how many of the two nodes each subtree contains) still costs Θ(n) per query; parent pointers with depth equalisation cost Θ(n) once and Θ(h) per query, and comparing the two root paths from the top is the third way. The program runs all four against an oracle that shares no code with them (every node's `L`/`R` path string, LCA = longest common prefix), on queries where about one node in five is absent.

**Steps:**

1. Build the 9-node tree and print a table of queries, including `LCA(7, 42)` where 42 is absent, with the answer of every method next to the oracle.
2. Run 20,000 queries on a 2,000-node tree with unique labels, classify each query (both nodes present, exactly one absent, neither), and count how many the one-pass version gets wrong in each class.
3. Run the fixed version, the parent-pointer version and the root-path version against the same oracle, and count the work per query for each.

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func node(v int, l, r *Node) *Node { return &Node{v, l, r} }
func leaf(v int) *Node             { return &Node{v, nil, nil} }

// randomSplit builds a SHAPE only (every Val is 0); the draw comes BEFORE the recursion so
// the Rust twin consumes the PCG stream in the same order and builds the identical tree.
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelInOrder gives the nodes their in-order rank 0..n-1: unique labels, so a label names
// exactly one node (or none, when it is >= n).
func labelInOrder(n *Node, next *int) {
	if n == nil {
		return
	}
	labelInOrder(n.Left, next)
	n.Val = *next
	*next++
	labelInOrder(n.Right, next)
}

// label of an answer: -1 stands for "none" (real labels are >= 0).
func label(n *Node) int {
	if n == nil {
		return -1
	}
	return n.Val
}

func name(x int) string {
	if x < 0 {
		return "none"
	}
	return fmt.Sprint(x)
}

// ---- 1. the one-pass LCA: elegant, and it ASSUMES both nodes are in the tree -----------

// Return the node if it is p or q; if both sides return something this node is the answer;
// otherwise pass up whichever side found something. visits counts real nodes entered.
func lcaOnePass(n *Node, p, q int, visits *int) *Node {
	if n == nil {
		return nil
	}
	*visits++
	if n.Val == p || n.Val == q {
		return n // stops here: the subtree below is never searched for the other node
	}
	l := lcaOnePass(n.Left, p, q, visits)
	r := lcaOnePass(n.Right, p, q, visits)
	if l != nil && r != nil {
		return n
	}
	if l != nil {
		return l
	}
	return r
}

// ---- 2. the fix: also report how many of the two nodes each subtree contains -------------

// lcaChecked returns the answer AND how many of {p, q} lie in this subtree (p == q counts
// twice). The answer is non-nil only once the count reaches 2, so an absent node gives nil.
// The price: it cannot stop early, because an absent node could be anywhere.
func lcaChecked(n *Node, p, q int, visits *int) (*Node, int) {
	if n == nil {
		return nil, 0
	}
	*visits++
	lw, lc := lcaChecked(n.Left, p, q, visits)
	rw, rc := lcaChecked(n.Right, p, q, visits)
	c := lc + rc
	if n.Val == p {
		c++
	}
	if n.Val == q {
		c++
	}
	switch {
	case lw != nil:
		return lw, c
	case rw != nil:
		return rw, c
	case c == 2:
		return n, c
	}
	return nil, c
}

// ---- 3 and 4. side tables keyed by the unique label, and the oracle ---------------------

// tables is built in ONE walk of the tree: parent and depth (for the parent-pointer LCA)
// and the L/R path string of every label (for the oracle and nothing else).
type tables struct {
	root          *Node
	parent, depth map[int]int
	path          map[int]string
	byPath        map[string]int
}

func newTables(root *Node) *tables {
	t := &tables{root, map[int]int{}, map[int]int{}, map[int]string{}, map[string]int{}}
	var walk func(n *Node, parent, depth int, path string)
	walk = func(n *Node, parent, depth int, path string) {
		if n == nil {
			return
		}
		t.parent[n.Val], t.depth[n.Val], t.path[n.Val], t.byPath[path] = parent, depth, path, n.Val
		walk(n.Left, n.Val, depth+1, path+"L")
		walk(n.Right, n.Val, depth+1, path+"R")
	}
	walk(root, -1, 0, "")
	return t
}

// oracle: the LCA is the node at the longest common prefix of the two paths; none if either
// node is absent. It shares no code with any of the methods above.
func (t *tables) oracle(p, q int) int {
	pp, okp := t.path[p]
	pq, okq := t.path[q]
	if !okp || !okq {
		return -1
	}
	i := 0
	for i < len(pp) && i < len(pq) && pp[i] == pq[i] {
		i++
	}
	return t.byPath[pp[:i]]
}

// viaParents equalises the depths, then climbs both together. hops counts parent steps.
func (t *tables) viaParents(p, q int, hops *int) int {
	dp, okp := t.depth[p]
	dq, okq := t.depth[q]
	if !okp || !okq {
		return -1
	}
	for ; dp > dq; dp-- {
		p = t.parent[p]
		*hops++
	}
	for ; dq > dp; dq-- {
		q = t.parent[q]
		*hops++
	}
	for p != q {
		p, q = t.parent[p], t.parent[q]
		*hops += 2
	}
	return p
}

// rootPath finds target by DFS and leaves the labels from the root down to it in path.
func rootPath(n *Node, target int, path *[]int, visits *int) bool {
	if n == nil {
		return false
	}
	*visits++
	*path = append(*path, n.Val)
	if n.Val == target || rootPath(n.Left, target, path, visits) || rootPath(n.Right, target, path, visits) {
		return true
	}
	*path = (*path)[:len(*path)-1]
	return false
}

// viaPaths compares the two root paths from the top; the last label they share is the LCA.
// visits counts the nodes the two searches enter (a missing p skips the search for q).
func (t *tables) viaPaths(p, q int, visits *int) int {
	var pp, pq []int
	if !rootPath(t.root, p, &pp, visits) || !rootPath(t.root, q, &pq, visits) {
		return -1
	}
	i := 0
	for i < len(pp) && i < len(pq) && pp[i] == pq[i] {
		i++
	}
	return pp[i-1]
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	fmt.Println("== (a) the 9-node tree: 1(2(4,5(7,-)),3(-,6(8,9)))")
	hand := node(1, node(2, leaf(4), node(5, leaf(7), nil)), node(3, nil, node(6, leaf(8), leaf(9))))
	ht := newTables(hand)
	fmt.Printf("%-8s %9s %8s %8s %8s %8s\n", "query", "one-pass", "checked", "parents", "paths", "oracle")
	for _, q := range [][2]int{{4, 5}, {7, 8}, {2, 7}, {6, 6}, {7, 42}, {42, 7}, {42, 43}} {
		var v int
		onePass := label(lcaOnePass(hand, q[0], q[1], &v))
		checked, _ := lcaChecked(hand, q[0], q[1], &v)
		fmt.Printf("%-8s %9s %8s %8s %8s %8s\n", fmt.Sprintf("(%d,%d)", q[0], q[1]), name(onePass), name(label(checked)),
			name(ht.viaParents(q[0], q[1], &v)), name(ht.viaPaths(q[0], q[1], &v)), name(ht.oracle(q[0], q[1])))
	}

	big := randomSplit(2000, r)
	labelInOrder(big, new(int))
	bt := newTables(big)
	const queries = 20000
	classes := []string{"both present", "exactly one absent", "neither present"}
	var count, wrongOne [3]int
	var wrongChecked, wrongParents, wrongPaths int
	var visOne, visChecked, visPaths, hops [2]int // [0] = all queries, [1] = the both-present ones
	nPresent := 0
	for i := 0; i < queries; i++ {
		p, q := r.IntN(2500), r.IntN(2500)
		_, okp := bt.path[p]
		_, okq := bt.path[q]
		class := 2
		if okp && okq {
			class = 0
		} else if okp || okq {
			class = 1
		}
		count[class]++
		want := bt.oracle(p, q)

		var v1, v2, v3, h int
		got1 := label(lcaOnePass(big, p, q, &v1))
		n2, _ := lcaChecked(big, p, q, &v2)
		got3 := bt.viaParents(p, q, &h)
		got4 := bt.viaPaths(p, q, &v3)
		if got1 != want {
			wrongOne[class]++
		}
		if label(n2) != want {
			wrongChecked++
		}
		if got3 != want {
			wrongParents++
		}
		if got4 != want {
			wrongPaths++
		}
		visOne[0], visChecked[0], visPaths[0], hops[0] = visOne[0]+v1, visChecked[0]+v2, visPaths[0]+v3, hops[0]+h
		if class == 0 {
			visOne[1], visChecked[1], visPaths[1], hops[1] = visOne[1]+v1, visChecked[1]+v2, visPaths[1]+v3, hops[1]+h
			nPresent++
		}
	}

	fmt.Printf("\n== (b) %d queries on randomSplit(2000), labels 0..1999, p and q drawn from 0..2499\n", queries)
	fmt.Printf("%-20s %8s %20s\n", "class", "queries", "one-pass wrong")
	for c, nm := range classes {
		fmt.Printf("%-20s %8d %20d\n", nm, count[c], wrongOne[c])
	}
	fmt.Printf("one-pass wrong in total: %d; the size of the 'exactly one absent' class: %d; equal: %v\n",
		wrongOne[0]+wrongOne[1]+wrongOne[2], count[1], wrongOne[0]+wrongOne[1]+wrongOne[2] == count[1])

	fmt.Println("\n== (c) and (d) against the oracle, all queries")
	fmt.Printf("checked (node, count): %d wrong; parent pointers: %d wrong; root paths: %d wrong\n", wrongChecked, wrongParents, wrongPaths)

	fmt.Println("\n== (e) cost per query")
	fmt.Printf("%-24s %14s %14s   %s\n", "method", "all queries", "both present", "unit")
	mean := func(sum, n int) float64 { return float64(sum) / float64(n) }
	fmt.Printf("%-24s %14.1f %14.1f   nodes visited\n", "one-pass", mean(visOne[0], queries), mean(visOne[1], nPresent))
	fmt.Printf("%-24s %14.1f %14.1f   nodes visited\n", "checked (node, count)", mean(visChecked[0], queries), mean(visChecked[1], nPresent))
	fmt.Printf("%-24s %14.1f %14.1f   parent hops (after a setup walk of %d nodes)\n", "parent pointers", mean(hops[0], queries), mean(hops[1], nPresent), len(bt.depth))
	fmt.Printf("%-24s %14.1f %14.1f   nodes visited by two searches\n", "root paths", mean(visPaths[0], queries), mean(visPaths[1], nPresent))
	fmt.Printf("checked / one-pass nodes visited on the both-present queries: %.3f\n", mean(visChecked[1], nPresent)/mean(visOne[1], nPresent))
}
```

**Output:**

```
== (a) the 9-node tree: 1(2(4,5(7,-)),3(-,6(8,9)))
query     one-pass  checked  parents    paths   oracle
(4,5)            2        2        2        2        2
(7,8)            1        1        1        1        1
(2,7)            2        2        2        2        2
(6,6)            6        6        6        6        6
(7,42)           7     none     none     none     none
(42,7)           7     none     none     none     none
(42,43)       none     none     none     none     none

== (b) 20000 queries on randomSplit(2000), labels 0..1999, p and q drawn from 0..2499
class                 queries       one-pass wrong
both present            12804                    0
exactly one absent       6413                 6413
neither present           783                    0
one-pass wrong in total: 6413; the size of the 'exactly one absent' class: 6413; equal: true

== (c) and (d) against the oracle, all queries
checked (node, count): 0 wrong; parent pointers: 0 wrong; root paths: 0 wrong

== (e) cost per query
method                      all queries   both present   unit
one-pass                         1980.8         1976.0   nodes visited
checked (node, count)            2000.0         2000.0   nodes visited
parent pointers                    13.1           20.5   parent hops (after a setup walk of 2000 nodes)
root paths                       2159.8         1993.2   nodes visited by two searches
checked / one-pass nodes visited on the both-present queries: 1.012
```

**Rust:**

```rust
// Pcg: the port of Go's math/rand/v2 PCG-DXSM generator and its IntN from lesson 01
// example 14, so seed (1, 2) builds exactly the trees the Go program builds.
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

use std::collections::HashMap;

// An empty tree is None; a node owns its two subtrees through Box.
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// A chain built in a loop is n levels deep, and the compiler's default drop glue
// would recurse once per level. This Drop moves the children onto a heap stack
// instead (lesson 05 example 15). Go has no destructor to write.
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

fn node(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

fn leaf(val: i64) -> Link {
    node(val, None, None)
}

// random_split builds a SHAPE only (every val is 0); the draw comes BEFORE the recursion so
// both languages consume the PCG stream in the same order and build the identical tree.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n as u64) as usize;
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    node(0, left, right)
}

// label_in_order gives the nodes their in-order rank 0..n-1: unique labels, so a label
// names exactly one node (or none, when it is >= n).
fn label_in_order(n: &mut Link, next: &mut i64) {
    if let Some(b) = n {
        label_in_order(&mut b.left, next);
        b.val = *next;
        *next += 1;
        label_in_order(&mut b.right, next);
    }
}

// name prints an answer; Go says -1 for "none", Rust says None.
fn name(x: Option<i64>) -> String {
    x.map_or("none".to_string(), |v| v.to_string())
}

// ---- 1. the one-pass LCA: elegant, and it ASSUMES both nodes are in the tree -----------

// Return the node if it is p or q; if both sides return something this node is the answer;
// otherwise pass up whichever side found something. visits counts real nodes entered.
// The answer is a reference INTO the tree: the lifetime 'a says it cannot outlive the
// borrow of the tree it points into.
fn lca_one_pass<'a>(n: Option<&'a Node>, p: i64, q: i64, visits: &mut u64) -> Option<&'a Node> {
    let n = n?; // an empty tree answers None
    *visits += 1;
    if n.val == p || n.val == q {
        return Some(n); // stops here: the subtree below is never searched for the other node
    }
    let l = lca_one_pass(n.left.as_deref(), p, q, visits);
    let r = lca_one_pass(n.right.as_deref(), p, q, visits);
    match (l, r) {
        (Some(_), Some(_)) => Some(n),
        (Some(x), None) | (None, Some(x)) => Some(x),
        (None, None) => None,
    }
}

// ---- 2. the fix: also report how many of the two nodes each subtree contains -------------

// lca_checked returns the answer AND how many of {p, q} lie in this subtree (p == q counts
// twice). The answer is Some only once the count reaches 2, so an absent node gives None.
// The price: it cannot stop early, because an absent node could be anywhere.
fn lca_checked<'a>(
    n: Option<&'a Node>,
    p: i64,
    q: i64,
    visits: &mut u64,
) -> (Option<&'a Node>, u8) {
    let Some(n) = n else { return (None, 0) };
    *visits += 1;
    let (lw, lc) = lca_checked(n.left.as_deref(), p, q, visits);
    let (rw, rc) = lca_checked(n.right.as_deref(), p, q, visits);
    let c = lc + rc + u8::from(n.val == p) + u8::from(n.val == q);
    match (lw, rw) {
        (Some(x), _) | (None, Some(x)) => (Some(x), c),
        (None, None) if c == 2 => (Some(n), c),
        (None, None) => (None, c),
    }
}

// ---- 3 and 4. side tables keyed by the unique label, and the oracle ---------------------

// Tables is built in ONE walk of the tree: parent and depth (for the parent-pointer LCA)
// and the L/R path string of every label (for the oracle and nothing else). A Box tree
// cannot hold a parent pointer in its nodes (a node has one owner, and that owner is its
// parent), so the parents live beside the tree, keyed by label. The root simply has no
// entry in `parent`, where Go stores -1.
struct Tables<'a> {
    root: Option<&'a Node>,
    parent: HashMap<i64, i64>,
    depth: HashMap<i64, u64>,
    path: HashMap<i64, String>,
    by_path: HashMap<String, i64>,
}

impl<'a> Tables<'a> {
    fn new(root: Option<&'a Node>) -> Self {
        fn walk(n: Option<&Node>, parent: Option<i64>, depth: u64, path: &str, t: &mut Tables) {
            let Some(x) = n else { return };
            if let Some(p) = parent {
                t.parent.insert(x.val, p);
            }
            t.depth.insert(x.val, depth);
            t.path.insert(x.val, path.to_string());
            t.by_path.insert(path.to_string(), x.val);
            walk(
                x.left.as_deref(),
                Some(x.val),
                depth + 1,
                &format!("{path}L"),
                t,
            );
            walk(
                x.right.as_deref(),
                Some(x.val),
                depth + 1,
                &format!("{path}R"),
                t,
            );
        }
        let mut t = Tables {
            root,
            parent: HashMap::new(),
            depth: HashMap::new(),
            path: HashMap::new(),
            by_path: HashMap::new(),
        };
        walk(root, None, 0, "", &mut t);
        t
    }

    // oracle: the LCA is the node at the longest common prefix of the two paths; None if
    // either node is absent. It shares no code with any of the methods above.
    fn oracle(&self, p: i64, q: i64) -> Option<i64> {
        let (pp, pq) = (self.path.get(&p)?, self.path.get(&q)?);
        let i = pp
            .bytes()
            .zip(pq.bytes())
            .take_while(|(a, b)| a == b)
            .count();
        Some(self.by_path[&pp[..i]])
    }

    // via_parents equalises the depths, then climbs both together. hops counts parent steps.
    fn via_parents(&self, mut p: i64, mut q: i64, hops: &mut u64) -> Option<i64> {
        let (mut dp, mut dq) = (*self.depth.get(&p)?, *self.depth.get(&q)?);
        while dp > dq {
            p = self.parent[&p];
            dp -= 1;
            *hops += 1;
        }
        while dq > dp {
            q = self.parent[&q];
            dq -= 1;
            *hops += 1;
        }
        while p != q {
            (p, q) = (self.parent[&p], self.parent[&q]);
            *hops += 2;
        }
        Some(p)
    }

    // via_paths compares the two root paths from the top; the last label they share is the
    // LCA. visits counts the nodes the two searches enter (a missing p skips the search for q).
    fn via_paths(&self, p: i64, q: i64, visits: &mut u64) -> Option<i64> {
        let (mut pp, mut pq) = (Vec::new(), Vec::new());
        if !root_path(self.root, p, &mut pp, visits) || !root_path(self.root, q, &mut pq, visits) {
            return None;
        }
        let i = pp.iter().zip(&pq).take_while(|(a, b)| a == b).count();
        Some(pp[i - 1])
    }
}

// root_path finds target by DFS and leaves the labels from the root down to it in path.
fn root_path(n: Option<&Node>, target: i64, path: &mut Vec<i64>, visits: &mut u64) -> bool {
    let Some(x) = n else { return false };
    *visits += 1;
    path.push(x.val);
    if x.val == target
        || root_path(x.left.as_deref(), target, path, visits)
        || root_path(x.right.as_deref(), target, path, visits)
    {
        return true;
    }
    path.pop();
    false
}

fn main() {
    let mut r = Pcg::new(1, 2);

    println!("== (a) the 9-node tree: 1(2(4,5(7,-)),3(-,6(8,9)))");
    let hand = node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    );
    let ht = Tables::new(hand.as_deref());
    println!(
        "{:<8} {:>9} {:>8} {:>8} {:>8} {:>8}",
        "query", "one-pass", "checked", "parents", "paths", "oracle"
    );
    for (p, q) in [(4, 5), (7, 8), (2, 7), (6, 6), (7, 42), (42, 7), (42, 43)] {
        let mut v = 0;
        let one_pass = lca_one_pass(hand.as_deref(), p, q, &mut v).map(|n| n.val);
        let checked = lca_checked(hand.as_deref(), p, q, &mut v).0.map(|n| n.val);
        println!(
            "{:<8} {:>9} {:>8} {:>8} {:>8} {:>8}",
            format!("({p},{q})"),
            name(one_pass),
            name(checked),
            name(ht.via_parents(p, q, &mut v)),
            name(ht.via_paths(p, q, &mut v)),
            name(ht.oracle(p, q))
        );
    }

    let mut big = random_split(2000, &mut r);
    label_in_order(&mut big, &mut 0);
    let bt = Tables::new(big.as_deref());
    const QUERIES: u64 = 20000;
    let classes = ["both present", "exactly one absent", "neither present"];
    let (mut count, mut wrong_one) = ([0u64; 3], [0u64; 3]);
    let (mut wrong_checked, mut wrong_parents, mut wrong_paths) = (0, 0, 0);
    // [0] = all queries, [1] = the both-present ones
    let (mut vis_one, mut vis_checked, mut vis_paths, mut hops) =
        ([0u64; 2], [0u64; 2], [0u64; 2], [0u64; 2]);
    let mut n_present = 0u64;
    for _ in 0..QUERIES {
        let p = r.int_n(2500) as i64;
        let q = r.int_n(2500) as i64;
        let (okp, okq) = (bt.path.contains_key(&p), bt.path.contains_key(&q));
        let class = if okp && okq {
            0
        } else if okp || okq {
            1
        } else {
            2
        };
        count[class] += 1;
        let want = bt.oracle(p, q);

        let (mut v1, mut v2, mut v3, mut h) = (0, 0, 0, 0);
        let got1 = lca_one_pass(big.as_deref(), p, q, &mut v1).map(|n| n.val);
        let got2 = lca_checked(big.as_deref(), p, q, &mut v2).0.map(|n| n.val);
        let got3 = bt.via_parents(p, q, &mut h);
        let got4 = bt.via_paths(p, q, &mut v3);
        wrong_one[class] += u64::from(got1 != want);
        wrong_checked += u64::from(got2 != want);
        wrong_parents += u64::from(got3 != want);
        wrong_paths += u64::from(got4 != want);
        vis_one[0] += v1;
        vis_checked[0] += v2;
        vis_paths[0] += v3;
        hops[0] += h;
        if class == 0 {
            vis_one[1] += v1;
            vis_checked[1] += v2;
            vis_paths[1] += v3;
            hops[1] += h;
            n_present += 1;
        }
    }

    println!(
        "\n== (b) {QUERIES} queries on randomSplit(2000), labels 0..1999, p and q drawn from 0..2499"
    );
    println!("{:<20} {:>8} {:>20}", "class", "queries", "one-pass wrong");
    for (c, nm) in classes.iter().enumerate() {
        println!("{nm:<20} {:>8} {:>20}", count[c], wrong_one[c]);
    }
    let total_wrong: u64 = wrong_one.iter().sum();
    println!(
        "one-pass wrong in total: {total_wrong}; the size of the 'exactly one absent' class: {}; equal: {}",
        count[1],
        total_wrong == count[1]
    );

    println!("\n== (c) and (d) against the oracle, all queries");
    println!(
        "checked (node, count): {wrong_checked} wrong; parent pointers: {wrong_parents} wrong; root paths: {wrong_paths} wrong"
    );

    println!("\n== (e) cost per query");
    println!(
        "{:<24} {:>14} {:>14}   unit",
        "method", "all queries", "both present"
    );
    let mean = |sum: u64, n: u64| sum as f64 / n as f64;
    println!(
        "{:<24} {:>14.1} {:>14.1}   nodes visited",
        "one-pass",
        mean(vis_one[0], QUERIES),
        mean(vis_one[1], n_present)
    );
    println!(
        "{:<24} {:>14.1} {:>14.1}   nodes visited",
        "checked (node, count)",
        mean(vis_checked[0], QUERIES),
        mean(vis_checked[1], n_present)
    );
    println!(
        "{:<24} {:>14.1} {:>14.1}   parent hops (after a setup walk of {} nodes)",
        "parent pointers",
        mean(hops[0], QUERIES),
        mean(hops[1], n_present),
        bt.depth.len()
    );
    println!(
        "{:<24} {:>14.1} {:>14.1}   nodes visited by two searches",
        "root paths",
        mean(vis_paths[0], QUERIES),
        mean(vis_paths[1], n_present)
    );
    println!(
        "checked / one-pass nodes visited on the both-present queries: {:.3}",
        mean(vis_checked[1], n_present) / mean(vis_one[1], n_present)
    );
}
```

**Rust output:**

```
== (a) the 9-node tree: 1(2(4,5(7,-)),3(-,6(8,9)))
query     one-pass  checked  parents    paths   oracle
(4,5)            2        2        2        2        2
(7,8)            1        1        1        1        1
(2,7)            2        2        2        2        2
(6,6)            6        6        6        6        6
(7,42)           7     none     none     none     none
(42,7)           7     none     none     none     none
(42,43)       none     none     none     none     none

== (b) 20000 queries on randomSplit(2000), labels 0..1999, p and q drawn from 0..2499
class                 queries       one-pass wrong
both present            12804                    0
exactly one absent       6413                 6413
neither present           783                    0
one-pass wrong in total: 6413; the size of the 'exactly one absent' class: 6413; equal: true

== (c) and (d) against the oracle, all queries
checked (node, count): 0 wrong; parent pointers: 0 wrong; root paths: 0 wrong

== (e) cost per query
method                      all queries   both present   unit
one-pass                         1980.8         1976.0   nodes visited
checked (node, count)            2000.0         2000.0   nodes visited
parent pointers                    13.1           20.5   parent hops (after a setup walk of 2000 nodes)
root paths                       2159.8         1993.2   nodes visited by two searches
checked / one-pass nodes visited on the both-present queries: 1.012
```

> **Go vs Rust:** Diffed against the Go run: the output is identical, byte for byte (both build the tree and the 20,000 queries from `PCG(1, 2)`; "none" is `None` here and `-1` in Go). `lca_one_pass` returns `Option<&'a Node>`, a reference into the tree whose lifetime ties the answer to the borrow of the tree, and the `match (l, r)` states the three outcomes in one expression. A `Box` tree cannot hold parent pointers (a node has one owner, its parent), so the parents are a side table keyed by the unique label; keying by node identity would use `std::ptr::from_ref(n)` as a `*const Node` map key (compiled in a scratch program, not used here), and an arena stores a parent index (example 14).

**Complexity:** one-pass Θ(n) per query, Θ(h) stack — it visited 1,976 of 2,000 nodes on average when both nodes exist (the early stop saves about 24) — and it is **wrong whenever exactly one node is absent: 6,413 wrong answers, exactly the size of that class, and 0 wrong on the other 13,587 queries** · the `(node, count)` fix is right on all 20,000 and visits all 2,000 nodes (1.012 times the one-pass cost on present-present queries) · parent pointers Θ(n) setup + Θ(h) per query: 20.5 parent hops on average on present-present queries, 0 wrong · root paths: about 2,000 nodes visited per query by two DFS searches, 0 wrong · for many queries see binary lifting (28) / Euler tour + sparse table (36).

---

## 10. Serialize and deserialize: one marker per nil

`🟡 medium` · *2n+1 tokens*

Pre-order with a marker for every `nil` is a complete description of a tree: the markers say where each subtree ends,
so the reader never has to guess, and the length is exactly 2n + 1 tokens, the 2n + 1 calls of example 1. Drop the
markers and pre-order is ambiguous: all five 3-node shapes print `1,2,3`. The other half of the job is
deserializing text you did not write. The reader is a recursive descent over the tokens, so it returns an error for
truncated, trailing and non-numeric input and takes a depth cap, because the recursion is as deep as the input says.
The program round-trips thousands of random trees (repeated values included, since markers do not care about values),
compares with the level-order `[1,2,null,…]` format, and feeds the reader hostile and random strings. Θ(n) time both
ways, Θ(h) stack.

**Steps:**

1. Write `serialize` (value, left, right, `#` for nil) and `deserialize` (one token at a time, with a position, a
   depth and an error at every level), and round-trip the hand-built tree: 19 tokens for 9 nodes.
2. Round-trip 5,000 random trees, the empty one included, then list every 3-node shape with and without markers,
   then give the reader truncated, trailing, non-numeric, overflowing, 100,000-deep and random garbage input.
3. Read the counts: 2n + 1 tokens on every tree, 0 mismatches, one marker-free sequence for 1,430 shapes, an error
   (never a panic) for every bad string, and what the depth cap rejects even when the input is valid.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"strconv"
	"strings"
)

// Node is a binary tree node; nil is the empty tree.
type Node struct {
	Val         int
	Left, Right *Node
}

func mk(v int, l, r *Node) *Node { return &Node{v, l, r} }

// ---- generators (the Rust twin draws from the same PCG stream) ----

// randomSplit draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes. The draw happens before recursing, so the stream is
// consumed in one fixed order.
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelRange gives every node, in pre-order, a value from [lo, hi].
func labelRange(t *Node, lo, hi int, r *rand.Rand) {
	if t == nil {
		return
	}
	t.Val = lo + r.IntN(hi-lo+1)
	labelRange(t.Left, lo, hi, r)
	labelRange(t.Right, lo, hi, r)
}

func complete(i, n int) *Node { // heap-shaped, Val = index
	if i >= n {
		return nil
	}
	return &Node{i, complete(2*i+1, n), complete(2*i+2, n)}
}

func leftChain(n int) *Node { // built iteratively, Val = depth
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

func same(a, b *Node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Val == b.Val && same(a.Left, b.Left) && same(a.Right, b.Right)
}

// shapes lists every binary tree shape with n nodes (fresh nodes each call).
func shapes(n int) []*Node {
	if n == 0 {
		return []*Node{nil}
	}
	var out []*Node
	for k := 0; k < n; k++ {
		ls, rs := shapes(k), shapes(n-1-k)
		for _, l := range ls {
			for _, r := range rs {
				out = append(out, &Node{0, clone(l), clone(r)})
			}
		}
	}
	return out
}

func clone(t *Node) *Node {
	if t == nil {
		return nil
	}
	return &Node{t.Val, clone(t.Left), clone(t.Right)}
}

// relabel numbers the nodes 1, 2, 3, ... in pre-order.
func relabel(t *Node, next *int) {
	if t == nil {
		return
	}
	*next++
	t.Val = *next
	relabel(t.Left, next)
	relabel(t.Right, next)
}

func paren(t *Node) string { // 1(2(3,-),-): "-" is nil
	switch {
	case t == nil:
		return "-"
	case t.Left == nil && t.Right == nil:
		return strconv.Itoa(t.Val)
	}
	return fmt.Sprintf("%d(%s,%s)", t.Val, paren(t.Left), paren(t.Right))
}

// ---- pre-order with a marker for every nil ----

func serialize(t *Node, out []string) []string {
	if t == nil {
		return append(out, "#")
	}
	out = append(out, strconv.Itoa(t.Val))
	out = serialize(t.Left, out)
	return serialize(t.Right, out)
}

func ser(t *Node) string { return strings.Join(serialize(t, nil), ",") }

// preorder lists the values WITHOUT markers: the tempting, ambiguous format.
func preorder(t *Node, out []int) []int {
	if t == nil {
		return out
	}
	out = append(out, t.Val)
	out = preorder(t.Left, out)
	return preorder(t.Right, out)
}

func markerFree(t *Node) string {
	parts := []string{}
	for _, v := range preorder(t, nil) {
		parts = append(parts, strconv.Itoa(v))
	}
	return strings.Join(parts, ",")
}

type kind int

const (
	truncated kind = iota
	trailing
	badToken
	tooDeep
)

var kindNames = [...]string{"truncated", "trailing", "bad token", "too deep"}

type parseErr struct {
	kind kind
	msg  string
}

func (e *parseErr) Error() string { return e.msg }

type decoder struct {
	toks     []string
	pos      int
	maxDepth int
}

// node reads one subtree: a "#", or a value followed by two subtrees.
// Untrusted input gets errors, never panics, and a bound on the recursion.
func (d *decoder) node(depth int) (*Node, error) {
	if d.pos >= len(d.toks) {
		return nil, &parseErr{truncated, fmt.Sprintf("truncated: the input ends at position %d but a subtree is still open", d.pos)}
	}
	tok := d.toks[d.pos]
	if tok == "#" {
		d.pos++
		return nil, nil
	}
	if depth >= d.maxDepth {
		return nil, &parseErr{tooDeep, fmt.Sprintf("too deep: the node at position %d would nest beyond the cap of %d", d.pos, d.maxDepth)}
	}
	v, err := strconv.Atoi(tok)
	if err != nil {
		return nil, &parseErr{badToken, fmt.Sprintf("bad token %q at position %d", tok, d.pos)}
	}
	d.pos++
	left, err := d.node(depth + 1)
	if err != nil {
		return nil, err
	}
	right, err := d.node(depth + 1)
	if err != nil {
		return nil, err
	}
	return &Node{v, left, right}, nil
}

func deserialize(s string, maxDepth int) (*Node, error) {
	d := &decoder{toks: strings.Split(s, ","), maxDepth: maxDepth}
	root, err := d.node(0)
	if err != nil {
		return nil, err
	}
	if d.pos != len(d.toks) {
		return nil, &parseErr{trailing, fmt.Sprintf("trailing: %d unread token(s) start at position %d", len(d.toks)-d.pos, d.pos)}
	}
	return root, nil
}

// ---- level order with "null", trailing nulls trimmed ----

func serLevel(t *Node) string {
	var toks []string
	q := []*Node{t}
	for len(q) > 0 {
		x := q[0]
		q = q[1:]
		if x == nil {
			toks = append(toks, "null")
			continue
		}
		toks = append(toks, strconv.Itoa(x.Val))
		q = append(q, x.Left, x.Right)
	}
	for len(toks) > 0 && toks[len(toks)-1] == "null" {
		toks = toks[:len(toks)-1]
	}
	return "[" + strings.Join(toks, ",") + "]"
}

func levelTokens(s string) int {
	if s == "[]" {
		return 0
	}
	return strings.Count(s, ",") + 1
}

func deserLevel(s string) (*Node, error) {
	body := strings.TrimSuffix(strings.TrimPrefix(s, "["), "]")
	if body == "" {
		return nil, nil
	}
	toks := strings.Split(body, ",")
	next := 0
	take := func() (*Node, error) { // the next token as a node, nil past the end
		i := next
		next++
		if i >= len(toks) || toks[i] == "null" {
			return nil, nil
		}
		v, err := strconv.Atoi(toks[i])
		if err != nil {
			return nil, fmt.Errorf("bad token %q at position %d", toks[i], i)
		}
		return &Node{Val: v}, nil
	}
	root, err := take()
	if err != nil || root == nil {
		return nil, err
	}
	q := []*Node{root}
	for head := 0; head < len(q); head++ {
		l, err := take()
		if err != nil {
			return nil, err
		}
		r, err := take()
		if err != nil {
			return nil, err
		}
		q[head].Left, q[head].Right = l, r
		for _, c := range []*Node{l, r} {
			if c != nil {
				q = append(q, c)
			}
		}
	}
	return root, nil
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	hand := mk(1, mk(2, mk(4, nil, nil), mk(5, mk(7, nil, nil), nil)),
		mk(3, nil, mk(6, mk(8, nil, nil), mk(9, nil, nil))))

	fmt.Println("== (a) the hand-built tree, one marker per nil")
	s := ser(hand)
	back, err := deserialize(s, 10000)
	fmt.Printf("serialized: %s\n", s)
	fmt.Printf("tokens %d, 2n+1 = %d, round trip equal: %t\n",
		strings.Count(s, ",")+1, 2*size(hand)+1, err == nil && same(hand, back))
	e, err := deserialize(ser(nil), 10000)
	fmt.Printf("the empty tree serializes to %q and comes back empty: %t\n", ser(nil), err == nil && e == nil)

	roundTrips := func(t *Node) bool {
		got, err := deserialize(ser(t), 10000)
		return err == nil && same(t, got)
	}

	fmt.Println("\n== (b) round trip: 5000 random trees, n in [0, 80], values in [-1000, 1000]")
	bad, nodes, wrongCount, dupTrees, levelBad := 0, 0, 0, 0, 0
	lo, hi := math.Inf(1), 0.0
	for i := 0; i < 5000; i++ {
		t := randomSplit(r.IntN(81), r)
		labelRange(t, -1000, 1000, r)
		n := size(t)
		nodes += n
		toks := serialize(t, nil)
		if len(toks) != 2*n+1 {
			wrongCount++
		}
		if !roundTrips(t) {
			bad++
		}
		seen := map[int]bool{}
		for _, v := range preorder(t, nil) {
			seen[v] = true
		}
		if len(seen) < n {
			dupTrees++
		}
		lv := serLevel(t)
		got2, err := deserLevel(lv)
		if err != nil || !same(t, got2) {
			levelBad++
		}
		if n >= 1 {
			ratio := float64(levelTokens(lv)) / float64(2*n+1)
			lo, hi = math.Min(lo, ratio), math.Max(hi, ratio)
		}
	}
	fmt.Printf("pre-order + markers: %d trees, %d nodes in total, %d mismatches, %d trees with tokens != 2n+1\n", 5000, nodes, bad, wrongCount)
	fmt.Printf("%d of the 5000 trees contain a repeated value: the markers, not the values, carry the shape\n", dupTrees)
	allBad := 0
	for i := 0; i < 1000; i++ {
		t := randomSplit(r.IntN(81), r)
		labelRange(t, 7, 7, r)
		if !roundTrips(t) {
			allBad++
		}
	}
	fmt.Printf("every value equal (7): 1000 trees, %d mismatches\n", allBad)
	fmt.Printf("level-order format: %d mismatches; tokens/(2n+1) ranges over [%.3f, %.3f] (n >= 1)\n", levelBad, lo, hi)

	fmt.Println("\n== (c) why the markers: the 5 shapes of 3 nodes, labelled 1,2,3 in pre-order")
	fmt.Printf("%-14s %-12s %s\n", "shape", "no markers", "with markers")
	for _, t := range shapes(3) {
		c := 0
		relabel(t, &c)
		fmt.Printf("%-14s %-12s %s\n", paren(t), markerFree(t), ser(t))
	}
	fmt.Printf("\n%3s %7s %18s %18s\n", "n", "shapes", "distinct no-marker", "distinct marked")
	for n := 1; n <= 8; n++ {
		free, marked, all := map[string]bool{}, map[string]bool{}, shapes(n)
		for _, t := range all {
			c := 0
			relabel(t, &c)
			free[markerFree(t)] = true
			marked[ser(t)] = true
		}
		fmt.Printf("%3d %7d %18d %18d\n", n, len(all), len(free), len(marked))
	}

	fmt.Println("\n== (d) the level-order format: its length depends on the shape")
	fmt.Println("hand-built:", serLevel(hand), " tokens:", levelTokens(serLevel(hand)))
	fmt.Printf("%-18s %5s %6s | %9s %9s | %11s %11s\n", "tree", "n", "2n+1", "pre toks", "pre bytes", "level toks", "level bytes")
	rs := randomSplit(1000, r)
	labelRange(rs, -1000, 1000, r)
	preBytesWin, preTokensOK := 0, true
	for _, row := range []struct {
		name string
		t    *Node
	}{{"perfect(10)", complete(0, 1023)}, {"leftChain(100)", leftChain(100)}, {"randomSplit(1000)", rs}} {
		pre, lv := ser(row.t), serLevel(row.t)
		fmt.Printf("%-18s %5d %6d | %9d %9d | %11d %11d\n", row.name, size(row.t), 2*size(row.t)+1,
			strings.Count(pre, ",")+1, len(pre), levelTokens(lv), len(lv))
		if len(pre) < len(lv) {
			preBytesWin++
		}
		preTokensOK = preTokensOK && strings.Count(pre, ",")+1 == 2*size(row.t)+1
	}
	fmt.Printf("pre-order tokens equal 2n+1 in every row: %t; level-order drops its trailing nulls, so its count moves with the shape\n", preTokensOK)
	fmt.Printf("fewer tokens is not fewer bytes: pre-order is the smaller text on %d of the 3 rows (\"null\" is 4 bytes, \"#\" is 1)\n", preBytesWin)

	fmt.Println("\n== (e) hostile input: errors, not panics")
	deep := strings.TrimSuffix(strings.Repeat("1,", 100000), ",")
	for _, c := range []struct {
		label, in string
		cap       int
	}{
		{"truncated", "1,2", 10000},
		{"trailing", "#,5", 10000},
		{"extra # after a full tree", "1,#,#,#", 10000},
		{"non-numeric", "1,x,#", 10000},
		{"empty input", "", 10000},
		{"overflowing int", "99999999999999999999,#,#", 10000},
		{"100,000 ones, cap 10,000", deep, 10000},
	} {
		msg := "accepted"
		if _, err := deserialize(c.in, c.cap); err != nil {
			msg = err.Error()
		}
		fmt.Printf("%-26s -> %s\n", c.label, msg)
	}

	fmt.Println("\n== (f) the cap is policy: a VALID chain of 100,000 nodes")
	chain := strings.Repeat("1,", 100000) + strings.Repeat("#,", 100000) + "#"
	_, err = deserialize(chain, 10000)
	fmt.Printf("cap 10,000:  %v\n", err)
	big, err := deserialize(chain, 200000)
	h := 0
	for x := big; x != nil; x = x.Left {
		h++
	}
	fmt.Printf("cap 200,000: accepted: %t, height %d, size %d, serializes back to the same string: %t\n", err == nil, h, size(big), ser(big) == chain)

	fmt.Println("\n== (g) fuzz: 20000 random token strings of length 0..11 over {#, 1, -2, 7, x}, cap 5")
	alphabet := []string{"#", "1", "-2", "7", "x"}
	var byKind [4]int
	accepted, notCanonical, panics := 0, 0, 0
	for i := 0; i < 20000; i++ {
		toks := make([]string, r.IntN(12))
		for j := range toks {
			toks[j] = alphabet[r.IntN(len(alphabet))]
		}
		in := strings.Join(toks, ",")
		func() {
			defer func() {
				if recover() != nil {
					panics++
				}
			}()
			t, err := deserialize(in, 5)
			if err != nil {
				byKind[err.(*parseErr).kind]++
				return
			}
			accepted++
			if ser(t) != in {
				notCanonical++
			}
		}()
	}
	fmt.Printf("accepted %d (%d re-serialize to a different string), panics %d\n", accepted, notCanonical, panics)
	for k, name := range kindNames {
		fmt.Printf("  rejected as %-10s %d\n", name+":", byKind[k])
	}
}
```

**Output:**

```
== (a) the hand-built tree, one marker per nil
serialized: 1,2,4,#,#,5,7,#,#,#,3,#,6,8,#,#,9,#,#
tokens 19, 2n+1 = 19, round trip equal: true
the empty tree serializes to "#" and comes back empty: true

== (b) round trip: 5000 random trees, n in [0, 80], values in [-1000, 1000]
pre-order + markers: 5000 trees, 200303 nodes in total, 0 mismatches, 0 trees with tokens != 2n+1
1704 of the 5000 trees contain a repeated value: the markers, not the values, carry the shape
every value equal (7): 1000 trees, 0 mismatches
level-order format: 0 mismatches; tokens/(2n+1) ranges over [0.333, 0.988] (n >= 1)

== (c) why the markers: the 5 shapes of 3 nodes, labelled 1,2,3 in pre-order
shape          no markers   with markers
1(-,2(-,3))    1,2,3        1,#,2,#,3,#,#
1(-,2(3,-))    1,2,3        1,#,2,3,#,#,#
1(2,3)         1,2,3        1,2,#,#,3,#,#
1(2(-,3),-)    1,2,3        1,2,#,3,#,#,#
1(2(3,-),-)    1,2,3        1,2,3,#,#,#,#

  n  shapes distinct no-marker    distinct marked
  1       1                  1                  1
  2       2                  1                  2
  3       5                  1                  5
  4      14                  1                 14
  5      42                  1                 42
  6     132                  1                132
  7     429                  1                429
  8    1430                  1               1430

== (d) the level-order format: its length depends on the shape
hand-built: [1,2,3,4,5,null,6,null,null,7,null,8,9]  tokens: 13
tree                   n   2n+1 |  pre toks pre bytes |  level toks level bytes
perfect(10)         1023   2047 |      2047      6052 |        1023        4006
leftChain(100)       100    201 |       201       491 |         198         781
randomSplit(1000)   1000   2001 |      2001      6406 |        1998        9396
pre-order tokens equal 2n+1 in every row: true; level-order drops its trailing nulls, so its count moves with the shape
fewer tokens is not fewer bytes: pre-order is the smaller text on 2 of the 3 rows ("null" is 4 bytes, "#" is 1)

== (e) hostile input: errors, not panics
truncated                  -> truncated: the input ends at position 2 but a subtree is still open
trailing                   -> trailing: 1 unread token(s) start at position 1
extra # after a full tree  -> trailing: 1 unread token(s) start at position 3
non-numeric                -> bad token "x" at position 1
empty input                -> bad token "" at position 0
overflowing int            -> bad token "99999999999999999999" at position 0
100,000 ones, cap 10,000   -> too deep: the node at position 10000 would nest beyond the cap of 10000

== (f) the cap is policy: a VALID chain of 100,000 nodes
cap 10,000:  too deep: the node at position 10000 would nest beyond the cap of 10000
cap 200,000: accepted: true, height 100000, size 100000, serializes back to the same string: true

== (g) fuzz: 20000 random token strings of length 0..11 over {#, 1, -2, 7, x}, cap 5
accepted 356 (0 re-serialize to a different string), panics 0
  rejected as truncated: 3656
  rejected as trailing:  3772
  rejected as bad token: 10859
  rejected as too deep:  1357
```

**Rust:**

```rust
use std::collections::{HashSet, VecDeque};
use std::fmt;
use std::panic;
use std::thread;

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

// A binary tree node; None is the empty tree. Clone and PartialEq are derived
// (structural, recursive: as deep as the tree).
#[derive(Clone, PartialEq)]
struct Node {
    val: i64,
    left: Link,
    right: Link,
}

type Link = Option<Box<Node>>;

// Iterative Drop: the default drop glue recurses once per level, which a
// 100,000-node chain (section f) would turn into a stack overflow.
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

// ---- generators (same draws, same order as the Go program) ----

// random_split draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes. The draw happens before recursing.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    mk(0, left, right)
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
    // built iteratively, val = depth
    let mut root: Link = None;
    for i in (0..n).rev() {
        root = mk(i as i64, root, None);
    }
    root
}

fn size(t: &Link) -> usize {
    match t {
        None => 0,
        Some(n) => 1 + size(&n.left) + size(&n.right),
    }
}

// shapes lists every binary tree shape with n nodes (fresh nodes each call).
fn shapes(n: usize) -> Vec<Link> {
    if n == 0 {
        return vec![None];
    }
    let mut out = Vec::new();
    for k in 0..n {
        let (ls, rs) = (shapes(k), shapes(n - 1 - k));
        for l in &ls {
            for r in &rs {
                out.push(mk(0, l.clone(), r.clone()));
            }
        }
    }
    out
}

// relabel numbers the nodes 1, 2, 3, ... in pre-order.
fn relabel(t: &mut Link, next: &mut i64) {
    if let Some(n) = t {
        *next += 1;
        n.val = *next;
        relabel(&mut n.left, next);
        relabel(&mut n.right, next);
    }
}

fn paren(t: &Link) -> String {
    // 1(2(3,-),-): "-" is nil
    match t {
        None => "-".to_string(),
        Some(n) if n.left.is_none() && n.right.is_none() => n.val.to_string(),
        Some(n) => format!("{}({},{})", n.val, paren(&n.left), paren(&n.right)),
    }
}

// ---- pre-order with a marker for every nil ----

fn serialize(t: &Link, out: &mut Vec<String>) {
    match t {
        None => out.push("#".to_string()),
        Some(n) => {
            out.push(n.val.to_string());
            serialize(&n.left, out);
            serialize(&n.right, out);
        }
    }
}

fn ser(t: &Link) -> String {
    let mut toks = Vec::new();
    serialize(t, &mut toks);
    toks.join(",")
}

// preorder lists the values WITHOUT markers: the tempting, ambiguous format.
fn preorder(t: &Link, out: &mut Vec<i64>) {
    if let Some(n) = t {
        out.push(n.val);
        preorder(&n.left, out);
        preorder(&n.right, out);
    }
}

fn marker_free(t: &Link) -> String {
    let mut vals = Vec::new();
    preorder(t, &mut vals);
    let parts: Vec<String> = vals.iter().map(i64::to_string).collect();
    parts.join(",")
}

// One variant per way the input can be wrong. The messages match the Go ones.
enum ParseError {
    Truncated { pos: usize },
    Trailing { left: usize, pos: usize },
    BadToken { tok: String, pos: usize },
    TooDeep { pos: usize, cap: usize },
}

const KIND_NAMES: [&str; 4] = ["truncated", "trailing", "bad token", "too deep"];

impl ParseError {
    fn kind(&self) -> usize {
        match self {
            ParseError::Truncated { .. } => 0,
            ParseError::Trailing { .. } => 1,
            ParseError::BadToken { .. } => 2,
            ParseError::TooDeep { .. } => 3,
        }
    }
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ParseError::Truncated { pos } => write!(
                f,
                "truncated: the input ends at position {pos} but a subtree is still open"
            ),
            ParseError::Trailing { left, pos } => {
                write!(
                    f,
                    "trailing: {left} unread token(s) start at position {pos}"
                )
            }
            ParseError::BadToken { tok, pos } => write!(f, "bad token {tok:?} at position {pos}"),
            ParseError::TooDeep { pos, cap } => write!(
                f,
                "too deep: the node at position {pos} would nest beyond the cap of {cap}"
            ),
        }
    }
}

// The tokens are a lazy `split`: nothing is copied, `pos` counts what is consumed.
struct Decoder<'a> {
    toks: std::str::Split<'a, char>,
    pos: usize,
    max_depth: usize,
}

impl Decoder<'_> {
    // node reads one subtree: a "#", or a value followed by two subtrees.
    // Untrusted input gets errors, never panics, and a bound on the recursion.
    fn node(&mut self, depth: usize) -> Result<Link, ParseError> {
        let Some(tok) = self.toks.next() else {
            return Err(ParseError::Truncated { pos: self.pos });
        };
        if tok == "#" {
            self.pos += 1;
            return Ok(None);
        }
        if depth >= self.max_depth {
            return Err(ParseError::TooDeep {
                pos: self.pos,
                cap: self.max_depth,
            });
        }
        let Ok(val) = tok.parse::<i64>() else {
            return Err(ParseError::BadToken {
                tok: tok.to_string(),
                pos: self.pos,
            });
        };
        self.pos += 1;
        let left = self.node(depth + 1)?;
        let right = self.node(depth + 1)?;
        Ok(mk(val, left, right))
    }
}

fn deserialize(s: &str, max_depth: usize) -> Result<Link, ParseError> {
    let mut d = Decoder {
        toks: s.split(','),
        pos: 0,
        max_depth,
    };
    let root = d.node(0)?;
    let left = d.toks.by_ref().count();
    if left > 0 {
        return Err(ParseError::Trailing { left, pos: d.pos });
    }
    Ok(root)
}

// ---- level order with "null", trailing nulls trimmed ----

fn ser_level(t: &Link) -> String {
    let mut toks: Vec<String> = Vec::new();
    let mut q: VecDeque<Option<&Node>> = VecDeque::new();
    q.push_back(t.as_deref());
    while let Some(x) = q.pop_front() {
        match x {
            None => toks.push("null".to_string()),
            Some(n) => {
                toks.push(n.val.to_string());
                q.push_back(n.left.as_deref());
                q.push_back(n.right.as_deref());
            }
        }
    }
    while toks.last().is_some_and(|t| t == "null") {
        toks.pop();
    }
    format!("[{}]", toks.join(","))
}

fn level_tokens(s: &str) -> usize {
    if s == "[]" {
        0
    } else {
        s.matches(',').count() + 1
    }
}

// The k-th real node in level order owns the token slots 2k+1 and 2k+2, so the
// tree can be built bottom-up with no queue of `&mut` references (a borrow
// checker problem in Rust, a plain pointer queue in Go): go through the real
// nodes in REVERSE level order and take their children out of the slots.
fn deser_level(s: &str) -> Result<Link, String> {
    let body = s.strip_prefix('[').unwrap_or(s);
    let body = body.strip_suffix(']').unwrap_or(body);
    if body.is_empty() {
        return Ok(None);
    }
    let mut vals: Vec<Option<i64>> = Vec::new();
    for (i, tok) in body.split(',').enumerate() {
        vals.push(match tok {
            "null" => None,
            _ => Some(
                tok.parse::<i64>()
                    .map_err(|_| format!("bad token {tok:?} at position {i}"))?,
            ),
        });
    }
    let reals: Vec<usize> = (0..vals.len()).filter(|&i| vals[i].is_some()).collect();
    let mut slots: Vec<Link> = (0..vals.len()).map(|_| None).collect();
    for (k, &p) in reals.iter().enumerate().rev() {
        let left = slots.get_mut(2 * k + 1).and_then(Option::take);
        let right = slots.get_mut(2 * k + 2).and_then(Option::take);
        slots[p] = mk(vals[p].unwrap_or(0), left, right);
    }
    Ok(slots[0].take())
}

fn main() {
    let mut r = Pcg::new(1, 2);
    let hand = mk(
        1,
        mk(2, mk(4, None, None), mk(5, mk(7, None, None), None)),
        mk(3, None, mk(6, mk(8, None, None), mk(9, None, None))),
    );

    println!("== (a) the hand-built tree, one marker per nil");
    let s = ser(&hand);
    let back = deserialize(&s, 10000);
    println!("serialized: {s}");
    println!(
        "tokens {}, 2n+1 = {}, round trip equal: {}",
        s.matches(',').count() + 1,
        2 * size(&hand) + 1,
        back.is_ok_and(|b| b == hand)
    );
    println!(
        "the empty tree serializes to {:?} and comes back empty: {}",
        ser(&None),
        deserialize(&ser(&None), 10000).is_ok_and(|e| e.is_none())
    );

    let round_trips = |t: &Link| deserialize(&ser(t), 10000).is_ok_and(|g| g == *t);

    println!("\n== (b) round trip: 5000 random trees, n in [0, 80], values in [-1000, 1000]");
    let (mut bad, mut nodes, mut wrong_count, mut dup_trees, mut level_bad) = (0, 0, 0, 0, 0);
    let (mut lo, mut hi) = (f64::INFINITY, 0.0f64);
    for _ in 0..5000 {
        let n0 = r.int_n(81);
        let mut t = random_split(n0, &mut r);
        label_range(&mut t, -1000, 1000, &mut r);
        let n = size(&t);
        nodes += n;
        let mut toks = Vec::new();
        serialize(&t, &mut toks);
        if toks.len() != 2 * n + 1 {
            wrong_count += 1;
        }
        if !round_trips(&t) {
            bad += 1;
        }
        let mut vals = Vec::new();
        preorder(&t, &mut vals);
        if vals.iter().collect::<HashSet<_>>().len() < n {
            dup_trees += 1;
        }
        let lv = ser_level(&t);
        if !deser_level(&lv).is_ok_and(|g| g == t) {
            level_bad += 1;
        }
        if n >= 1 {
            let ratio = level_tokens(&lv) as f64 / (2 * n + 1) as f64;
            lo = lo.min(ratio);
            hi = hi.max(ratio);
        }
    }
    println!(
        "pre-order + markers: 5000 trees, {nodes} nodes in total, {bad} mismatches, {wrong_count} trees with tokens != 2n+1"
    );
    println!(
        "{dup_trees} of the 5000 trees contain a repeated value: the markers, not the values, carry the shape"
    );
    let mut all_bad = 0;
    for _ in 0..1000 {
        let n0 = r.int_n(81);
        let mut t = random_split(n0, &mut r);
        label_range(&mut t, 7, 7, &mut r);
        if !round_trips(&t) {
            all_bad += 1;
        }
    }
    println!("every value equal (7): 1000 trees, {all_bad} mismatches");
    println!(
        "level-order format: {level_bad} mismatches; tokens/(2n+1) ranges over [{lo:.3}, {hi:.3}] (n >= 1)"
    );

    println!("\n== (c) why the markers: the 5 shapes of 3 nodes, labelled 1,2,3 in pre-order");
    println!("{:<14} {:<12} with markers", "shape", "no markers");
    for mut t in shapes(3) {
        relabel(&mut t, &mut 0);
        println!("{:<14} {:<12} {}", paren(&t), marker_free(&t), ser(&t));
    }
    println!(
        "\n{:>3} {:>7} {:>18} {:>18}",
        "n", "shapes", "distinct no-marker", "distinct marked"
    );
    for n in 1..=8 {
        let all = shapes(n);
        let mut free = HashSet::new();
        let mut marked = HashSet::new();
        for mut t in all.clone() {
            relabel(&mut t, &mut 0);
            free.insert(marker_free(&t));
            marked.insert(ser(&t));
        }
        println!(
            "{:>3} {:>7} {:>18} {:>18}",
            n,
            all.len(),
            free.len(),
            marked.len()
        );
    }

    println!("\n== (d) the level-order format: its length depends on the shape");
    let hl = ser_level(&hand);
    println!("hand-built: {hl}  tokens: {}", level_tokens(&hl));
    println!(
        "{:<18} {:>5} {:>6} | {:>9} {:>9} | {:>11} {:>11}",
        "tree", "n", "2n+1", "pre toks", "pre bytes", "level toks", "level bytes"
    );
    let mut rs = random_split(1000, &mut r);
    label_range(&mut rs, -1000, 1000, &mut r);
    let (mut pre_bytes_win, mut pre_tokens_ok) = (0, true);
    let rows: [(&str, Link); 3] = [
        ("perfect(10)", complete(0, 1023)),
        ("leftChain(100)", left_chain(100)),
        ("randomSplit(1000)", rs),
    ];
    for (name, t) in &rows {
        let (pre, lv) = (ser(t), ser_level(t));
        println!(
            "{:<18} {:>5} {:>6} | {:>9} {:>9} | {:>11} {:>11}",
            name,
            size(t),
            2 * size(t) + 1,
            pre.matches(',').count() + 1,
            pre.len(),
            level_tokens(&lv),
            lv.len()
        );
        if pre.len() < lv.len() {
            pre_bytes_win += 1;
        }
        pre_tokens_ok = pre_tokens_ok && pre.matches(',').count() + 1 == 2 * size(t) + 1;
    }
    println!(
        "pre-order tokens equal 2n+1 in every row: {pre_tokens_ok}; level-order drops its trailing nulls, so its count moves with the shape"
    );
    println!(
        "fewer tokens is not fewer bytes: pre-order is the smaller text on {pre_bytes_win} of the 3 rows (\"null\" is 4 bytes, \"#\" is 1)"
    );

    println!("\n== (e) hostile input: errors, not panics");
    let deep = "1,".repeat(100000).trim_end_matches(',').to_string();
    let cases: [(&str, &str); 7] = [
        ("truncated", "1,2"),
        ("trailing", "#,5"),
        ("extra # after a full tree", "1,#,#,#"),
        ("non-numeric", "1,x,#"),
        ("empty input", ""),
        ("overflowing int", "99999999999999999999,#,#"),
        ("100,000 ones, cap 10,000", &deep),
    ];
    for (label, input) in cases {
        let msg = match deserialize(input, 10000) {
            Ok(_) => "accepted".to_string(),
            Err(e) => e.to_string(),
        };
        println!("{label:<26} -> {msg}");
    }

    println!("\n== (f) the cap is policy: a VALID chain of 100,000 nodes");
    let chain = "1,".repeat(100000) + &"#,".repeat(100000) + "#";
    match deserialize(&chain, 10000) {
        Ok(_) => println!("cap 10,000:  accepted"),
        Err(e) => println!("cap 10,000:  {e}"),
    }
    // size and ser recurse once per level: 100,000 levels need a stack we size
    // ourselves (the default 2 MiB spawned-thread stack would not do).
    thread::scope(|sc| {
        thread::Builder::new()
            .stack_size(256 << 20)
            .spawn_scoped(sc, || {
                let big = deserialize(&chain, 200000);
                let accepted = big.is_ok();
                let big = big.unwrap_or(None);
                let (mut h, mut cur) = (0, big.as_deref());
                while let Some(n) = cur {
                    h += 1;
                    cur = n.left.as_deref();
                }
                println!(
                    "cap 200,000: accepted: {accepted}, height {h}, size {}, serializes back to the same string: {}",
                    size(&big),
                    ser(&big) == chain
                );
            })
            .expect("spawn")
            .join()
            .expect("join");
    });

    println!(
        "\n== (g) fuzz: 20000 random token strings of length 0..11 over {{#, 1, -2, 7, x}}, cap 5"
    );
    let alphabet = ["#", "1", "-2", "7", "x"];
    let mut by_kind = [0usize; 4];
    let (mut accepted, mut not_canonical, mut panics) = (0, 0, 0);
    // a panic is caught and counted (silenced: the default hook would print each one)
    panic::set_hook(Box::new(|_| {}));
    for _ in 0..20000 {
        let len = r.int_n(12);
        let toks: Vec<&str> = (0..len)
            .map(|_| alphabet[r.int_n(alphabet.len())])
            .collect();
        let input = toks.join(",");
        match panic::catch_unwind(|| deserialize(&input, 5)) {
            Err(_) => panics += 1,
            Ok(Err(e)) => by_kind[e.kind()] += 1,
            Ok(Ok(t)) => {
                accepted += 1;
                if ser(&t) != input {
                    not_canonical += 1;
                }
            }
        }
    }
    let _ = panic::take_hook();
    println!(
        "accepted {accepted} ({not_canonical} re-serialize to a different string), panics {panics}"
    );
    for (name, count) in KIND_NAMES.iter().zip(by_kind) {
        println!("  rejected as {:<10} {count}", format!("{name}:"));
    }
}
```

**Rust output:**

```
== (a) the hand-built tree, one marker per nil
serialized: 1,2,4,#,#,5,7,#,#,#,3,#,6,8,#,#,9,#,#
tokens 19, 2n+1 = 19, round trip equal: true
the empty tree serializes to "#" and comes back empty: true

== (b) round trip: 5000 random trees, n in [0, 80], values in [-1000, 1000]
pre-order + markers: 5000 trees, 200303 nodes in total, 0 mismatches, 0 trees with tokens != 2n+1
1704 of the 5000 trees contain a repeated value: the markers, not the values, carry the shape
every value equal (7): 1000 trees, 0 mismatches
level-order format: 0 mismatches; tokens/(2n+1) ranges over [0.333, 0.988] (n >= 1)

== (c) why the markers: the 5 shapes of 3 nodes, labelled 1,2,3 in pre-order
shape          no markers   with markers
1(-,2(-,3))    1,2,3        1,#,2,#,3,#,#
1(-,2(3,-))    1,2,3        1,#,2,3,#,#,#
1(2,3)         1,2,3        1,2,#,#,3,#,#
1(2(-,3),-)    1,2,3        1,2,#,3,#,#,#
1(2(3,-),-)    1,2,3        1,2,3,#,#,#,#

  n  shapes distinct no-marker    distinct marked
  1       1                  1                  1
  2       2                  1                  2
  3       5                  1                  5
  4      14                  1                 14
  5      42                  1                 42
  6     132                  1                132
  7     429                  1                429
  8    1430                  1               1430

== (d) the level-order format: its length depends on the shape
hand-built: [1,2,3,4,5,null,6,null,null,7,null,8,9]  tokens: 13
tree                   n   2n+1 |  pre toks pre bytes |  level toks level bytes
perfect(10)         1023   2047 |      2047      6052 |        1023        4006
leftChain(100)       100    201 |       201       491 |         198         781
randomSplit(1000)   1000   2001 |      2001      6406 |        1998        9396
pre-order tokens equal 2n+1 in every row: true; level-order drops its trailing nulls, so its count moves with the shape
fewer tokens is not fewer bytes: pre-order is the smaller text on 2 of the 3 rows ("null" is 4 bytes, "#" is 1)

== (e) hostile input: errors, not panics
truncated                  -> truncated: the input ends at position 2 but a subtree is still open
trailing                   -> trailing: 1 unread token(s) start at position 1
extra # after a full tree  -> trailing: 1 unread token(s) start at position 3
non-numeric                -> bad token "x" at position 1
empty input                -> bad token "" at position 0
overflowing int            -> bad token "99999999999999999999" at position 0
100,000 ones, cap 10,000   -> too deep: the node at position 10000 would nest beyond the cap of 10000

== (f) the cap is policy: a VALID chain of 100,000 nodes
cap 10,000:  too deep: the node at position 10000 would nest beyond the cap of 10000
cap 200,000: accepted: true, height 100000, size 100000, serializes back to the same string: true

== (g) fuzz: 20000 random token strings of length 0..11 over {#, 1, -2, 7, x}, cap 5
accepted 356 (0 re-serialize to a different string), panics 0
  rejected as truncated: 3656
  rejected as trailing:  3772
  rejected as bad token: 10859
  rejected as too deep:  1357
```

> **Go vs Rust:** Identical output (diffed): the Rust `Pcg` port feeds both programs the same 5,000 trees, so every count and message matches. `deserialize` consumes a lazy `str::split` and returns `Result<Option<Box<Node>>, ParseError>` with `?` after each child, where Go threads `(*Node, error)` by hand, and `ParseError` is an enum that the fuzz counts with a `match` instead of Go's type assertion. Go's `recover` becomes `catch_unwind` (panics: 0, with the default hook silenced). The valid 100,000-node chain in section (f) needs the iterative `Drop` to go out of scope and a 256 MiB thread for `size` and `ser`, which recurse 100,000 deep, while a goroutine stack just grows; at the depth cap of 10,000 the parser's own recursion is about 110 bytes a level (measured with a throwaway address probe, not in this program), so it runs on the main thread. The level-order reader builds bottom-up from the token slots instead of keeping a queue of node pointers.

**Complexity:** Θ(n) time and Θ(h) stack both ways, and exactly **2n + 1 tokens** whatever the shape (5,000 random trees, 200,303 nodes, 0 mismatches, repeated values included). Level-order's token count moves with the shape (1,023 for the perfect tree of 1,023 nodes against 1,998 for a random 1,000-node tree). Without markers one sequence stands for all 1,430 shapes of 8 nodes. Every malformed string got an error; of 20,000 random strings none caused a panic, and the 356 that were valid trees re-serialized to the same text. A valid 100,000-node chain was refused at depth cap 10,000 and accepted at 200,000.

---

## 11. Rebuild a tree from two traversals

`🟡 medium` · *Which traversals are enough*

Pre-order plus in-order, and post-order plus in-order, identify a tree of distinct values; pre-order plus post-order do
not. The root is the first element of pre-order (the last of post-order), and its position in the in-order sequence
says how many nodes go left. Finding that position by scanning the window costs `size(left) + 1` per node, Θ(n·h) in
all and Θ(n²) on a left-leaning chain, while a value to index map (lesson 10) makes every lookup one probe: Θ(n)
for the whole build. The uniqueness claims are checked by brute force over every shape of up to 8 nodes, repeated
values are shown to break the pairing, and a checked builder (root inside its window, then re-traverse) is tested
against an exhaustive oracle on thousands of corrupted inputs, next to a builder that trusts its input.

**Steps:**

1. Write one builder that consumes the pre-order (or the reversed post-order) front to back and splits the in-order
   window at the root, with a scan or a map to find it, and rebuild the hand-built tree four ways.
2. Count the scan steps against `sum(size(left) + 1)` on random trees and on left and right chains, then enumerate
   every shape of 1 to 8 nodes and count the distinct (pre, in), (post, in) and (pre, post) pairs, with distinct
   labels and with values from {0, 1}.
3. Corrupt 10,000 in-orders (a swap or a non-member), and compare the checked builder, the exhaustive oracle and a
   builder that does not check: how many are rejected, how many crash it, how many it silently gets wrong.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strconv"
	"strings"
)

// Node is a binary tree node; nil is the empty tree.
type Node struct {
	Val         int
	Left, Right *Node
}

func mk(v int, l, r *Node) *Node { return &Node{v, l, r} }

// ---- generators (the Rust twin draws from the same PCG stream) ----

// randomSplit draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes; the draw happens before recursing.
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{0, left, right}
}

// labelShuffled gives the nodes the values 0..n-1, shuffled (a hand-written
// Fisher-Yates) and handed out in pre-order: distinct values, no order to lean on.
func labelShuffled(t *Node, n int, r *rand.Rand) {
	a := make([]int, n)
	for i := range a {
		a[i] = i
	}
	for i := n - 1; i >= 1; i-- {
		j := r.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
	next := 0
	var walk func(*Node)
	walk = func(t *Node) {
		if t == nil {
			return
		}
		t.Val = a[next]
		next++
		walk(t.Left)
		walk(t.Right)
	}
	walk(t)
}

func chain(n int, left bool) *Node { // built iteratively
	var root *Node
	for i := n - 1; i >= 0; i-- {
		if left {
			root = &Node{i, root, nil}
		} else {
			root = &Node{i, nil, root}
		}
	}
	return root
}

func size(t *Node) int {
	if t == nil {
		return 0
	}
	return 1 + size(t.Left) + size(t.Right)
}

func height(t *Node) int {
	if t == nil {
		return 0
	}
	return 1 + max(height(t.Left), height(t.Right))
}

func same(a, b *Node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Val == b.Val && same(a.Left, b.Left) && same(a.Right, b.Right)
}

func preorder(t *Node, out []int) []int {
	if t == nil {
		return out
	}
	out = append(out, t.Val)
	out = preorder(t.Left, out)
	return preorder(t.Right, out)
}

func inorder(t *Node, out []int) []int {
	if t == nil {
		return out
	}
	out = inorder(t.Left, out)
	out = append(out, t.Val)
	return inorder(t.Right, out)
}

func postorder(t *Node, out []int) []int {
	if t == nil {
		return out
	}
	out = postorder(t.Left, out)
	out = postorder(t.Right, out)
	return append(out, t.Val)
}

func str(xs []int) string {
	parts := make([]string, len(xs))
	for i, x := range xs {
		parts[i] = strconv.Itoa(x)
	}
	return strings.Join(parts, " ")
}

// leftWork is the sum over nodes of (size of the left subtree + 1): the number
// of comparisons a left-to-right scan for the root makes in total.
func leftWork(t *Node, acc *int) int {
	if t == nil {
		return 0
	}
	l := leftWork(t.Left, acc)
	r := leftWork(t.Right, acc)
	*acc += l + 1
	return l + r + 1
}

// ---- every shape, for the exhaustive checks ----

func shapes(n int) []*Node {
	if n == 0 {
		return []*Node{nil}
	}
	var out []*Node
	for k := 0; k < n; k++ {
		ls, rs := shapes(k), shapes(n-1-k)
		for _, l := range ls {
			for _, r := range rs {
				out = append(out, &Node{0, clone(l), clone(r)})
			}
		}
	}
	return out
}

func clone(t *Node) *Node {
	if t == nil {
		return nil
	}
	return &Node{t.Val, clone(t.Left), clone(t.Right)}
}

func relabel(t *Node, vals []int, next *int) { // values from vals, in pre-order
	if t == nil {
		return
	}
	t.Val = vals[*next]
	*next++
	relabel(t.Left, vals, next)
	relabel(t.Right, vals, next)
}

func paren(t *Node) string { // 1(2(3,-),-): "-" is nil
	switch {
	case t == nil:
		return "-"
	case t.Left == nil && t.Right == nil:
		return strconv.Itoa(t.Val)
	}
	return fmt.Sprintf("%d(%s,%s)", t.Val, paren(t.Left), paren(t.Right))
}

func oneChild(t *Node) int { // nodes with exactly one child
	if t == nil {
		return 0
	}
	k := oneChild(t.Left) + oneChild(t.Right)
	if (t.Left == nil) != (t.Right == nil) {
		k++
	}
	return k
}

func catalan(n int) int {
	c := 1
	for i := 0; i < n; i++ {
		c = c * 2 * (2*i + 1) / (i + 2)
	}
	return c
}

func motzkin(n int) int { // 1, 1, 2, 4, 9, 21, ...
	m := []int{1, 1}
	for k := 2; k <= n; k++ {
		m = append(m, ((2*k+1)*m[k-1]+(3*k-3)*m[k-2])/(k+2))
	}
	return m[n]
}

// ---- the builders: one routine, four switches ----

type buildErr int

const (
	errRanOut buildErr = iota
	errNotMember
	errOutside
	errTail
	errLength
	errRetrace
)

var buildErrNames = [...]string{
	"order ran out", "root not in the in-order", "root outside its window",
	"unused tail", "lengths differ", "re-traversal differs",
}

func (e buildErr) Error() string { return buildErrNames[e] }

// builder consumes `seq` front to back, one root per call, and uses the root's
// position in the in-order window [lo, hi) to size the left part. For
// post-order, seq is the REVERSED post-order (root, right, left) and the right
// subtree is built first.
type builder struct {
	seq, in       []int
	pos           map[int]int // value -> index in in; nil: scan the window
	rightFirst    bool
	checked       bool
	pi            int
	scan, lookups int
	inserts       int
}

func (b *builder) build(lo, hi int) (*Node, error) {
	if lo >= hi {
		return nil, nil
	}
	if b.checked && b.pi >= len(b.seq) {
		return nil, errRanOut
	}
	root := b.seq[b.pi] // an unchecked builder panics here when seq is short
	b.pi++
	var k int
	if b.pos != nil {
		b.lookups++
		i, ok := b.pos[root]
		if b.checked && !ok {
			return nil, errNotMember
		}
		k = i // unchecked: a missing key reads as 0, a legal index
	} else {
		for k = lo; k < hi; k++ {
			b.scan++
			if b.in[k] == root {
				break
			}
		}
	}
	if b.checked && (k < lo || k >= hi) {
		return nil, errOutside
	}
	n := &Node{Val: root}
	var err error
	if b.rightFirst {
		if n.Right, err = b.build(k+1, hi); err == nil {
			n.Left, err = b.build(lo, k)
		}
	} else {
		if n.Left, err = b.build(lo, k); err == nil {
			n.Right, err = b.build(k+1, hi)
		}
	}
	return n, err
}

func rebuild(order, in []int, post, useMap, checked bool) (*Node, *builder, error) {
	if checked && len(order) != len(in) {
		return nil, &builder{}, errLength
	}
	b := &builder{seq: order, in: in, rightFirst: post, checked: checked}
	if post {
		b.seq = slices.Clone(order)
		slices.Reverse(b.seq)
	}
	if useMap {
		b.pos = make(map[int]int, len(in))
		for i, v := range in {
			b.pos[v] = i
			b.inserts++
		}
	}
	root, err := b.build(0, len(in))
	if err != nil || !checked {
		return root, b, err
	}
	if b.pi != len(b.seq) {
		return nil, b, errTail
	}
	// the last line of defence: the tree must reproduce BOTH inputs
	want := preorder(root, nil)
	if post {
		want = postorder(root, nil)
	}
	if !slices.Equal(want, order) || !slices.Equal(inorder(root, nil), in) {
		return nil, b, errRetrace
	}
	return root, b, nil
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	hand := mk(1, mk(2, mk(4, nil, nil), mk(5, mk(7, nil, nil), nil)),
		mk(3, nil, mk(6, mk(8, nil, nil), mk(9, nil, nil))))

	fmt.Println("== (a) rebuild the hand-built tree from two traversals")
	pre, in, post := preorder(hand, nil), inorder(hand, nil), postorder(hand, nil)
	fmt.Printf("pre  %s\nin   %s\npost %s\n", str(pre), str(in), str(post))
	acc := 0
	leftWork(hand, &acc)
	for _, c := range []struct {
		name         string
		order        []int
		post, useMap bool
	}{{"pre+in  scan", pre, false, false}, {"pre+in  map", pre, false, true},
		{"post+in scan", post, true, false}, {"post+in map", post, true, true}} {
		t, b, err := rebuild(c.order, in, c.post, c.useMap, true)
		fmt.Printf("%-13s equal %t  scan steps %2d  map lookups %d\n", c.name, err == nil && same(hand, t), b.scan, b.lookups)
	}
	fmt.Printf("scan steps predicted by the sum of (left size + 1) over the nodes: %d\n", acc)

	bad, total := 0, 0
	for i := 0; i < 3000; i++ {
		n := r.IntN(61)
		t := randomSplit(n, r)
		labelShuffled(t, n, r)
		total += n
		p, io, po := preorder(t, nil), inorder(t, nil), postorder(t, nil)
		for _, useMap := range []bool{false, true} {
			a, _, ea := rebuild(p, io, false, useMap, true)
			b, _, eb := rebuild(po, io, true, useMap, true)
			if ea != nil || eb != nil || !same(t, a) || !same(t, b) {
				bad++
			}
		}
	}
	fmt.Printf("round trip over 3000 random trees (n in [0, 60], %d nodes in total, shuffled distinct labels): %d mismatches in 4 builders each\n", total, bad)

	fmt.Println("\n== (b) finding the root: scanning the window vs a value -> index map")
	fmt.Printf("%-20s %6s %4s | %13s %13s %13s | %8s %8s\n", "tree", "n", "h", "scan pre+in", "scan post+in", "sum(left+1)", "lookups", "inserts")
	rs1, rs2 := randomSplit(1000, r), randomSplit(100000, r)
	labelShuffled(rs1, 1000, r)
	labelShuffled(rs2, 100000, r)
	lc, rc := chain(2000, true), chain(2000, false)
	labelShuffled(lc, 2000, r)
	labelShuffled(rc, 2000, r)
	allEqual := true
	for _, row := range []struct {
		name string
		t    *Node
	}{{"randomSplit(1000)", rs1}, {"randomSplit(100000)", rs2}, {"leftChain(2000)", lc}, {"rightChain(2000)", rc}} {
		t := row.t
		n := size(t)
		p, io, po := preorder(t, nil), inorder(t, nil), postorder(t, nil)
		a, ba, _ := rebuild(p, io, false, false, false)
		b, bb, _ := rebuild(po, io, true, false, false)
		c, bc, _ := rebuild(p, io, false, true, false)
		d, bd, _ := rebuild(po, io, true, true, false)
		allEqual = allEqual && same(t, a) && same(t, b) && same(t, c) && same(t, d)
		w := 0
		leftWork(t, &w)
		fmt.Printf("%-20s %6d %4d | %13d %13d %13d | %8d %8d\n", row.name, n, height(t), ba.scan, bb.scan, w, bc.lookups+bd.lookups, bc.inserts+bd.inserts)
	}
	fmt.Printf("all 16 rebuilt trees equal their originals: %t\n", allEqual)
	fmt.Println("(lookups and inserts are summed over the pre+in and post+in map builds)")

	fmt.Println("\n== (c) which pairs of traversals identify a tree? every shape of n = 1..8, distinct labels")
	fmt.Println("columns 3-5: distinct pairs of sequences over all shapes; Motzkin: M(n-1) from its recurrence;")
	fmt.Println("full: (pre,post) pairs that belong to exactly one shape; 2^k ok: every (pre,post) pair is shared by exactly")
	fmt.Println("2^k shapes, k = nodes with one child, and full = Catalan((n-1)/2) for odd n, 0 for even n")
	fmt.Printf("%3s %7s | %9s %10s %11s | %9s %5s %6s\n", "n", "shapes", "(pre,in)", "(post,in)", "(pre,post)", "Motzkin", "full", "2^k ok")
	validIn := map[int]map[string]bool{} // n -> in-orders of the shapes labelled 1..n in pre-order
	for n := 1; n <= 8; n++ {
		vals := make([]int, n)
		for i := range vals {
			vals[i] = i + 1
		}
		pi, poi, pp := map[string]bool{}, map[string]bool{}, map[string][]int{}
		validIn[n] = map[string]bool{}
		for _, t := range shapes(n) {
			c := 0
			relabel(t, vals, &c)
			p, io, po := str(preorder(t, nil)), str(inorder(t, nil)), str(postorder(t, nil))
			pi[p+"|"+io], poi[po+"|"+io] = true, true
			pp[p+"|"+po] = append(pp[p+"|"+po], oneChild(t))
			validIn[n][io] = true
		}
		full, ok := 0, true
		for _, ks := range pp {
			for _, k := range ks {
				ok = ok && k == ks[0]
			}
			ok = ok && len(ks) == 1<<ks[0]
			if ks[0] == 0 {
				full++
			}
		}
		wantFull := 0
		if n%2 == 1 {
			wantFull = catalan((n - 1) / 2)
		}
		fmt.Printf("%3d %7d | %9d %10d %11d | %9d %5d %6t\n", n, catalan(n), len(pi), len(poi), len(pp), motzkin(n-1), full, ok && full == wantFull)
	}
	fmt.Println("one explicit collision: the 3-node shapes grouped by their (pre, post) pair")
	groups, order := map[string][]string{}, []string{}
	for _, t := range shapes(3) {
		c := 0
		relabel(t, []int{1, 2, 3}, &c)
		k := "pre " + str(preorder(t, nil)) + ", post " + str(postorder(t, nil))
		if groups[k] == nil {
			order = append(order, k)
		}
		groups[k] = append(groups[k], paren(t))
	}
	for _, k := range order {
		fmt.Printf("  %-24s %s\n", k, strings.Join(groups[k], "  "))
	}

	fmt.Println("\n== (d) repeated values break (pre, in) too: every shape of n nodes, every labelling over {0, 1}")
	fmt.Printf("%3s %7s %9s | %14s %24s | %20s\n", "n", "shapes", "labelled", "distinct pairs", "trees sharing a pair", "all-equal: pairs")
	for n := 1; n <= 8; n++ {
		count, equalPairs, labelled := map[string]int{}, map[string]bool{}, 0
		for _, t := range shapes(n) {
			for mask := 0; mask < 1<<n; mask++ {
				vals := make([]int, n)
				for i := range vals {
					vals[i] = mask >> i & 1
				}
				c := 0
				relabel(t, vals, &c)
				k := str(preorder(t, nil)) + "|" + str(inorder(t, nil))
				count[k]++
				labelled++
				if mask == 0 {
					equalPairs[k] = true
				}
			}
		}
		shared := 0
		for _, c := range count {
			if c > 1 {
				shared += c
			}
		}
		fmt.Printf("%3d %7d %9d | %14d %24d | %20d\n", n, catalan(n), labelled, len(count), shared, len(equalPairs))
	}
	var tally [2][3]int // [map, scan] x [the original, a different tree, rejected]
	dups := 0
	for i := 0; i < 5000; i++ {
		n := 1 + r.IntN(30)
		t := randomSplit(n, r)
		vals := make([]int, n)
		seen := map[int]bool{}
		for j := range vals {
			vals[j] = r.IntN(4)
			seen[vals[j]] = true
		}
		c := 0
		relabel(t, vals, &c)
		if len(seen) < n {
			dups++
		}
		for m, useMap := range []bool{true, false} {
			got, _, err := rebuild(preorder(t, nil), inorder(t, nil), false, useMap, true)
			switch {
			case err != nil:
				tally[m][2]++
			case same(t, got):
				tally[m][0]++
			default:
				tally[m][1]++
			}
		}
	}
	fmt.Printf("5000 random trees, values in {0..3}, n in [1, 30], %d with a repeated value; the checked builder returns:\n", dups)
	for m, name := range []string{"map (last occurrence)", "scan (first in window)"} {
		fmt.Printf("  %-24s the original %4d, a DIFFERENT tree with the same traversals %4d, an error %4d\n", name, tally[m][0], tally[m][1], tally[m][2])
	}

	fmt.Println("\n== (e) corrupt input: 10000 swaps or non-member replacements in a valid in-order, n in [2, 8]")
	var swaps, valid, panics, differ, accepted, disagree int
	var byErr [6]int
	retrace := 0
	for i := 0; i < 10000; i++ {
		n := 2 + r.IntN(7)
		t := randomSplit(n, r)
		vals := make([]int, n)
		for j := range vals {
			vals[j] = j + 1
		}
		c := 0
		relabel(t, vals, &c)
		p, io := preorder(t, nil), inorder(t, nil)
		if r.IntN(2) == 0 {
			swaps++
			a := r.IntN(n)
			b := r.IntN(n - 1)
			if b >= a {
				b++
			}
			io[a], io[b] = io[b], io[a]
		} else {
			io[r.IntN(n)] = n + 1 + r.IntN(5)
		}
		isValid := validIn[n][str(io)] // the exhaustive table of (c) is the oracle
		if isValid {
			valid++
		}
		_, _, err := rebuild(p, io, false, true, true)
		if (err == nil) != isValid {
			disagree++
		}
		if err != nil {
			byErr[err.(buildErr)]++
		} else {
			accepted++
		}
		if err == errRetrace {
			retrace++
		}
		if !isValid {
			func() {
				defer func() {
					if recover() != nil {
						panics++
					}
				}()
				got, _, _ := rebuild(p, io, false, true, false)
				if !slices.Equal(preorder(got, nil), p) || !slices.Equal(inorder(got, nil), io) {
					differ++
				}
			}()
		}
	}
	fmt.Printf("%d swaps and %d replacements; %d corrupted inputs are still a valid (pre, in) pair (a swap can land on another tree's in-order)\n", swaps, 10000-swaps, valid)
	fmt.Printf("checked builder: accepted %d, rejected %d, disagreements with the exhaustive oracle %d\n", accepted, 10000-accepted, disagree)
	for k, name := range buildErrNames {
		fmt.Printf("  rejected by %-26s %d\n", name+":", byErr[k])
	}
	fmt.Printf("unchecked builder on the %d invalid inputs: panicked %d, returned a tree whose traversals differ from the input %d, returned a matching tree %d\n",
		10000-valid, panics, differ, 10000-valid-panics-differ)
}
```

**Output:**

```
== (a) rebuild the hand-built tree from two traversals
pre  1 2 4 5 7 3 6 8 9
in   4 2 7 5 1 3 8 6 9
post 4 7 5 2 8 9 6 3 1
pre+in  scan  equal true  scan steps 16  map lookups 0
pre+in  map   equal true  scan steps  0  map lookups 9
post+in scan  equal true  scan steps 16  map lookups 0
post+in map   equal true  scan steps  0  map lookups 9
scan steps predicted by the sum of (left size + 1) over the nodes: 16
round trip over 3000 random trees (n in [0, 60], 91234 nodes in total, shuffled distinct labels): 0 mismatches in 4 builders each

== (b) finding the root: scanning the window vs a value -> index map
tree                      n    h |   scan pre+in  scan post+in   sum(left+1) |  lookups  inserts
randomSplit(1000)      1000   22 |          6860          6860          6860 |     2000     2000
randomSplit(100000)  100000   42 |       1103883       1103883       1103883 |   200000   200000
leftChain(2000)        2000 2000 |       2001000       2001000       2001000 |     4000     4000
rightChain(2000)       2000 2000 |          2000          2000          2000 |     4000     4000
all 16 rebuilt trees equal their originals: true
(lookups and inserts are summed over the pre+in and post+in map builds)

== (c) which pairs of traversals identify a tree? every shape of n = 1..8, distinct labels
columns 3-5: distinct pairs of sequences over all shapes; Motzkin: M(n-1) from its recurrence;
full: (pre,post) pairs that belong to exactly one shape; 2^k ok: every (pre,post) pair is shared by exactly
2^k shapes, k = nodes with one child, and full = Catalan((n-1)/2) for odd n, 0 for even n
  n  shapes |  (pre,in)  (post,in)  (pre,post) |   Motzkin  full 2^k ok
  1       1 |         1          1           1 |         1     1   true
  2       2 |         2          2           1 |         1     0   true
  3       5 |         5          5           2 |         2     1   true
  4      14 |        14         14           4 |         4     0   true
  5      42 |        42         42           9 |         9     2   true
  6     132 |       132        132          21 |        21     0   true
  7     429 |       429        429          51 |        51     5   true
  8    1430 |      1430       1430         127 |       127     0   true
one explicit collision: the 3-node shapes grouped by their (pre, post) pair
  pre 1 2 3, post 3 2 1    1(-,2(-,3))  1(-,2(3,-))  1(2(-,3),-)  1(2(3,-),-)
  pre 1 2 3, post 2 3 1    1(2,3)

== (d) repeated values break (pre, in) too: every shape of n nodes, every labelling over {0, 1}
  n  shapes  labelled | distinct pairs     trees sharing a pair |     all-equal: pairs
  1       1         2 |              2                        0 |                    1
  2       2         8 |              6                        4 |                    1
  3       5        40 |             20                       34 |                    1
  4      14       224 |             70                      212 |                    1
  5      42      1344 |            252                     1318 |                    1
  6     132      8448 |            924                     8392 |                    1
  7     429     54912 |           3432                    54794 |                    1
  8    1430    366080 |          12870                   365828 |                    1
5000 random trees, values in {0..3}, n in [1, 30], 4589 with a repeated value; the checked builder returns:
  map (last occurrence)    the original  411, a DIFFERENT tree with the same traversals    0, an error 4589
  scan (first in window)   the original  719, a DIFFERENT tree with the same traversals  980, an error 3301

== (e) corrupt input: 10000 swaps or non-member replacements in a valid in-order, n in [2, 8]
5068 swaps and 4932 replacements; 2971 corrupted inputs are still a valid (pre, in) pair (a swap can land on another tree's in-order)
checked builder: accepted 2971, rejected 7029, disagreements with the exhaustive oracle 0
  rejected by order ran out:             0
  rejected by root not in the in-order:  4932
  rejected by root outside its window:   2097
  rejected by unused tail:               0
  rejected by lengths differ:            0
  rejected by re-traversal differs:      0
unchecked builder on the 7029 invalid inputs: panicked 5805, returned a tree whose traversals differ from the input 1224, returned a matching tree 0
```

**Rust:**

```rust
use std::collections::{HashMap, HashSet};
use std::panic;

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

// A binary tree node; None is the empty tree. Clone and PartialEq are derived
// (structural, recursive: as deep as the tree, fine for the 2,000-node chains here).
#[derive(Clone, PartialEq)]
struct Node {
    val: i64,
    left: Link,
    right: Link,
}

type Link = Option<Box<Node>>;

// Iterative Drop: the default drop glue recurses once per level.
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

// ---- generators (same draws, same order as the Go program) ----

// random_split draws a random SHAPE: the left subtree gets a uniform k of the
// other n-1 nodes; the draw happens before recursing.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    mk(0, left, right)
}

// label_shuffled gives the nodes the values 0..n-1, shuffled (a hand-written
// Fisher-Yates) and handed out in pre-order: distinct values, no order to lean on.
fn label_shuffled(t: &mut Link, n: usize, r: &mut Pcg) {
    let mut a: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i + 1);
        a.swap(i, j);
    }
    fn walk(t: &mut Link, next: &mut impl Iterator<Item = i64>) {
        if let Some(n) = t {
            n.val = next.next().expect("one label per node");
            walk(&mut n.left, next);
            walk(&mut n.right, next);
        }
    }
    walk(t, &mut a.into_iter());
}

fn chain(n: usize, left: bool) -> Link {
    // built iteratively
    let mut root: Link = None;
    for i in (0..n).rev() {
        root = if left {
            mk(i as i64, root, None)
        } else {
            mk(i as i64, None, root)
        };
    }
    root
}

fn size(t: &Link) -> usize {
    t.as_ref().map_or(0, |n| 1 + size(&n.left) + size(&n.right))
}

fn height(t: &Link) -> usize {
    t.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

#[derive(Clone, Copy, PartialEq)]
enum Order {
    Pre,
    In,
    Post,
}

// One function with the visit position as a parameter (Go has three functions).
fn traverse(t: &Link, order: Order) -> Vec<i64> {
    fn go(t: &Link, order: Order, out: &mut Vec<i64>) {
        if let Some(n) = t {
            if order == Order::Pre {
                out.push(n.val);
            }
            go(&n.left, order, out);
            if order == Order::In {
                out.push(n.val);
            }
            go(&n.right, order, out);
            if order == Order::Post {
                out.push(n.val);
            }
        }
    }
    let mut out = Vec::new();
    go(t, order, &mut out);
    out
}

fn str_of(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    parts.join(" ")
}

// left_work is the sum over nodes of (size of the left subtree + 1): the number
// of comparisons a left-to-right scan for the root makes in total.
fn left_work(t: &Link, acc: &mut usize) -> usize {
    match t {
        None => 0,
        Some(n) => {
            let l = left_work(&n.left, acc);
            let r = left_work(&n.right, acc);
            *acc += l + 1;
            l + r + 1
        }
    }
}

// ---- every shape, for the exhaustive checks ----

fn shapes(n: usize) -> Vec<Link> {
    if n == 0 {
        return vec![None];
    }
    let mut out = Vec::new();
    for k in 0..n {
        let (ls, rs) = (shapes(k), shapes(n - 1 - k));
        for l in &ls {
            for r in &rs {
                out.push(mk(0, l.clone(), r.clone()));
            }
        }
    }
    out
}

fn relabel(t: &mut Link, vals: &[i64], next: &mut usize) {
    // values from vals, in pre-order
    if let Some(n) = t {
        n.val = vals[*next];
        *next += 1;
        relabel(&mut n.left, vals, next);
        relabel(&mut n.right, vals, next);
    }
}

fn paren(t: &Link) -> String {
    // 1(2(3,-),-): "-" is nil
    match t {
        None => "-".to_string(),
        Some(n) if n.left.is_none() && n.right.is_none() => n.val.to_string(),
        Some(n) => format!("{}({},{})", n.val, paren(&n.left), paren(&n.right)),
    }
}

fn one_child(t: &Link) -> usize {
    // nodes with exactly one child
    match t {
        None => 0,
        Some(n) => {
            one_child(&n.left)
                + one_child(&n.right)
                + usize::from(n.left.is_none() != n.right.is_none())
        }
    }
}

fn catalan(n: usize) -> usize {
    (0..n).fold(1, |c, i| c * 2 * (2 * i + 1) / (i + 2))
}

fn motzkin(n: usize) -> usize {
    // 1, 1, 2, 4, 9, 21, ...
    let mut m = vec![1, 1];
    for k in 2..=n {
        m.push(((2 * k + 1) * m[k - 1] + (3 * k - 3) * m[k - 2]) / (k + 2));
    }
    m[n]
}

// ---- the builders: one routine, four switches ----

#[derive(Clone, Copy, PartialEq)]
enum BuildErr {
    RanOut,
    NotMember,
    Outside,
    Tail,
    Length,
    Retrace,
}

const BUILD_ERR_NAMES: [&str; 6] = [
    "order ran out",
    "root not in the in-order",
    "root outside its window",
    "unused tail",
    "lengths differ",
    "re-traversal differs",
];

// Builder consumes `seq` front to back, one root per call, and uses the root's
// position in the in-order window [lo, hi) to size the left part. For
// post-order, seq is the REVERSED post-order (root, right, left) and the right
// subtree is built first.
struct Builder<'a> {
    seq: Vec<i64>,
    inord: &'a [i64],
    pos: Option<HashMap<i64, usize>>, // value -> index in inord; None: scan the window
    right_first: bool,
    checked: bool,
    pi: usize,
    scan: usize,
    lookups: usize,
}

impl Builder<'_> {
    fn build(&mut self, lo: usize, hi: usize) -> Result<Link, BuildErr> {
        if lo >= hi {
            return Ok(None);
        }
        if self.checked && self.pi >= self.seq.len() {
            return Err(BuildErr::RanOut);
        }
        let root = self.seq[self.pi]; // an unchecked builder panics here when seq is short
        self.pi += 1;
        let k = match &self.pos {
            Some(pos) => {
                self.lookups += 1;
                match pos.get(&root) {
                    Some(&i) => i,
                    None if self.checked => return Err(BuildErr::NotMember),
                    // unchecked: Go's map reads a missing key as 0, a legal index;
                    // Rust's get() returns None and this arm is where we must say so.
                    None => 0,
                }
            }
            None => {
                let mut k = lo;
                while k < hi {
                    self.scan += 1;
                    if self.inord[k] == root {
                        break;
                    }
                    k += 1;
                }
                k
            }
        };
        if self.checked && (k < lo || k >= hi) {
            return Err(BuildErr::Outside);
        }
        let (left, right);
        if self.right_first {
            right = self.build(k + 1, hi)?;
            left = self.build(lo, k)?;
        } else {
            left = self.build(lo, k)?;
            right = self.build(k + 1, hi)?;
        }
        Ok(mk(root, left, right))
    }
}

#[derive(Clone, Copy, Default)]
struct Stats {
    scan: usize,
    lookups: usize,
    inserts: usize,
}

// rebuild returns the tree (or why not) and what the build cost.
fn rebuild(
    order: &[i64],
    inord: &[i64],
    post: bool,
    use_map: bool,
    checked: bool,
) -> (Result<Link, BuildErr>, Stats) {
    if checked && order.len() != inord.len() {
        return (Err(BuildErr::Length), Stats::default());
    }
    let mut seq = order.to_vec();
    if post {
        seq.reverse();
    }
    let mut inserts = 0;
    let pos = use_map.then(|| {
        let mut m = HashMap::with_capacity(inord.len());
        for (i, &v) in inord.iter().enumerate() {
            m.insert(v, i);
            inserts += 1;
        }
        m
    });
    let mut b = Builder {
        seq,
        inord,
        pos,
        right_first: post,
        checked,
        pi: 0,
        scan: 0,
        lookups: 0,
    };
    let res = b.build(0, inord.len());
    let stats = Stats {
        scan: b.scan,
        lookups: b.lookups,
        inserts,
    };
    let root = match res {
        Err(e) => return (Err(e), stats),
        Ok(root) if !checked => return (Ok(root), stats),
        Ok(root) => root,
    };
    if b.pi != b.seq.len() {
        return (Err(BuildErr::Tail), stats);
    }
    // the last line of defence: the tree must reproduce BOTH inputs
    let want = traverse(&root, if post { Order::Post } else { Order::Pre });
    if want != order || traverse(&root, Order::In) != inord {
        return (Err(BuildErr::Retrace), stats);
    }
    (Ok(root), stats)
}

fn main() {
    let mut r = Pcg::new(1, 2);
    let hand = mk(
        1,
        mk(2, mk(4, None, None), mk(5, mk(7, None, None), None)),
        mk(3, None, mk(6, mk(8, None, None), mk(9, None, None))),
    );

    println!("== (a) rebuild the hand-built tree from two traversals");
    let (pre, inord, post) = (
        traverse(&hand, Order::Pre),
        traverse(&hand, Order::In),
        traverse(&hand, Order::Post),
    );
    println!(
        "pre  {}\nin   {}\npost {}",
        str_of(&pre),
        str_of(&inord),
        str_of(&post)
    );
    let mut acc = 0;
    left_work(&hand, &mut acc);
    for (name, order, is_post, use_map) in [
        ("pre+in  scan", &pre, false, false),
        ("pre+in  map", &pre, false, true),
        ("post+in scan", &post, true, false),
        ("post+in map", &post, true, true),
    ] {
        let (t, cost) = rebuild(order, &inord, is_post, use_map, true);
        println!(
            "{:<13} equal {}  scan steps {:>2}  map lookups {}",
            name,
            t.is_ok_and(|t| t == hand),
            cost.scan,
            cost.lookups
        );
    }
    println!("scan steps predicted by the sum of (left size + 1) over the nodes: {acc}");

    let (mut bad, mut total) = (0, 0);
    for _ in 0..3000 {
        let n = r.int_n(61);
        let mut t = random_split(n, &mut r);
        label_shuffled(&mut t, n, &mut r);
        total += n;
        let (p, io, po) = (
            traverse(&t, Order::Pre),
            traverse(&t, Order::In),
            traverse(&t, Order::Post),
        );
        for use_map in [false, true] {
            let a = rebuild(&p, &io, false, use_map, true).0;
            let b = rebuild(&po, &io, true, use_map, true).0;
            if !a.is_ok_and(|a| a == t) || !b.is_ok_and(|b| b == t) {
                bad += 1;
            }
        }
    }
    println!(
        "round trip over 3000 random trees (n in [0, 60], {total} nodes in total, shuffled distinct labels): {bad} mismatches in 4 builders each"
    );

    println!("\n== (b) finding the root: scanning the window vs a value -> index map");
    println!(
        "{:<20} {:>6} {:>4} | {:>13} {:>13} {:>13} | {:>8} {:>8}",
        "tree", "n", "h", "scan pre+in", "scan post+in", "sum(left+1)", "lookups", "inserts"
    );
    let mut rs1 = random_split(1000, &mut r);
    let mut rs2 = random_split(100000, &mut r);
    label_shuffled(&mut rs1, 1000, &mut r);
    label_shuffled(&mut rs2, 100000, &mut r);
    let (mut lc, mut rc) = (chain(2000, true), chain(2000, false));
    label_shuffled(&mut lc, 2000, &mut r);
    label_shuffled(&mut rc, 2000, &mut r);
    let mut all_equal = true;
    let rows = [
        ("randomSplit(1000)", rs1),
        ("randomSplit(100000)", rs2),
        ("leftChain(2000)", lc),
        ("rightChain(2000)", rc),
    ];
    for (name, t) in &rows {
        let n = size(t);
        let (p, io, po) = (
            traverse(t, Order::Pre),
            traverse(t, Order::In),
            traverse(t, Order::Post),
        );
        let (a, scan_pre) = rebuild(&p, &io, false, false, false);
        let (b, scan_post) = rebuild(&po, &io, true, false, false);
        let (c, map_pre) = rebuild(&p, &io, false, true, false);
        let (d, map_post) = rebuild(&po, &io, true, true, false);
        for built in [a, b, c, d] {
            all_equal = all_equal && built.is_ok_and(|x| x == *t);
        }
        let mut w = 0;
        left_work(t, &mut w);
        println!(
            "{:<20} {:>6} {:>4} | {:>13} {:>13} {:>13} | {:>8} {:>8}",
            name,
            n,
            height(t),
            scan_pre.scan,
            scan_post.scan,
            w,
            map_pre.lookups + map_post.lookups,
            map_pre.inserts + map_post.inserts
        );
    }
    println!("all 16 rebuilt trees equal their originals: {all_equal}");
    println!("(lookups and inserts are summed over the pre+in and post+in map builds)");

    println!(
        "\n== (c) which pairs of traversals identify a tree? every shape of n = 1..8, distinct labels"
    );
    println!(
        "columns 3-5: distinct pairs of sequences over all shapes; Motzkin: M(n-1) from its recurrence;"
    );
    println!(
        "full: (pre,post) pairs that belong to exactly one shape; 2^k ok: every (pre,post) pair is shared by exactly"
    );
    println!(
        "2^k shapes, k = nodes with one child, and full = Catalan((n-1)/2) for odd n, 0 for even n"
    );
    println!(
        "{:>3} {:>7} | {:>9} {:>10} {:>11} | {:>9} {:>5} {:>6}",
        "n", "shapes", "(pre,in)", "(post,in)", "(pre,post)", "Motzkin", "full", "2^k ok"
    );
    let mut valid_in: HashMap<usize, HashSet<String>> = HashMap::new(); // n -> in-orders of the shapes labelled 1..n
    for n in 1..=8 {
        let vals: Vec<i64> = (1..=n as i64).collect();
        let (mut pi, mut poi, mut valid) = (HashSet::new(), HashSet::new(), HashSet::new());
        let mut pp: HashMap<String, Vec<usize>> = HashMap::new();
        for mut t in shapes(n) {
            relabel(&mut t, &vals, &mut 0);
            let p = str_of(&traverse(&t, Order::Pre));
            let io = str_of(&traverse(&t, Order::In));
            let po = str_of(&traverse(&t, Order::Post));
            pi.insert(format!("{p}|{io}"));
            poi.insert(format!("{po}|{io}"));
            pp.entry(format!("{p}|{po}"))
                .or_default()
                .push(one_child(&t));
            valid.insert(io);
        }
        valid_in.insert(n, valid);
        let (mut full, mut ok) = (0, true);
        for ks in pp.values() {
            ok = ok && ks.iter().all(|&k| k == ks[0]) && ks.len() == 1 << ks[0];
            if ks[0] == 0 {
                full += 1;
            }
        }
        let want_full = if n % 2 == 1 { catalan((n - 1) / 2) } else { 0 };
        println!(
            "{:>3} {:>7} | {:>9} {:>10} {:>11} | {:>9} {:>5} {:>6}",
            n,
            catalan(n),
            pi.len(),
            poi.len(),
            pp.len(),
            motzkin(n - 1),
            full,
            ok && full == want_full
        );
    }
    println!("one explicit collision: the 3-node shapes grouped by their (pre, post) pair");
    let mut groups: Vec<(String, Vec<String>)> = Vec::new();
    for mut t in shapes(3) {
        relabel(&mut t, &[1, 2, 3], &mut 0);
        let key = format!(
            "pre {}, post {}",
            str_of(&traverse(&t, Order::Pre)),
            str_of(&traverse(&t, Order::Post))
        );
        match groups.iter_mut().find(|g| g.0 == key) {
            Some(g) => g.1.push(paren(&t)),
            None => groups.push((key, vec![paren(&t)])),
        }
    }
    for (key, members) in &groups {
        println!("  {:<24} {}", key, members.join("  "));
    }

    println!(
        "\n== (d) repeated values break (pre, in) too: every shape of n nodes, every labelling over {{0, 1}}"
    );
    println!(
        "{:>3} {:>7} {:>9} | {:>14} {:>24} | {:>20}",
        "n", "shapes", "labelled", "distinct pairs", "trees sharing a pair", "all-equal: pairs"
    );
    for n in 1..=8 {
        let mut count: HashMap<String, usize> = HashMap::new();
        let mut equal_pairs = HashSet::new();
        let mut labelled = 0;
        for mut t in shapes(n) {
            for mask in 0..1usize << n {
                let vals: Vec<i64> = (0..n).map(|i| (mask >> i & 1) as i64).collect();
                relabel(&mut t, &vals, &mut 0);
                let key = format!(
                    "{}|{}",
                    str_of(&traverse(&t, Order::Pre)),
                    str_of(&traverse(&t, Order::In))
                );
                if mask == 0 {
                    equal_pairs.insert(key.clone());
                }
                *count.entry(key).or_insert(0) += 1;
                labelled += 1;
            }
        }
        let shared: usize = count.values().filter(|&&c| c > 1).sum();
        println!(
            "{:>3} {:>7} {:>9} | {:>14} {:>24} | {:>20}",
            n,
            catalan(n),
            labelled,
            count.len(),
            shared,
            equal_pairs.len()
        );
    }
    let mut tally = [[0usize; 3]; 2]; // [map, scan] x [the original, a different tree, rejected]
    let mut dups = 0;
    for _ in 0..5000 {
        let n = 1 + r.int_n(30);
        let mut t = random_split(n, &mut r);
        let vals: Vec<i64> = (0..n).map(|_| r.int_n(4) as i64).collect();
        let distinct: HashSet<i64> = vals.iter().copied().collect();
        relabel(&mut t, &vals, &mut 0);
        if distinct.len() < n {
            dups += 1;
        }
        for (m, use_map) in [true, false].into_iter().enumerate() {
            let (p, io) = (traverse(&t, Order::Pre), traverse(&t, Order::In));
            match rebuild(&p, &io, false, use_map, true).0 {
                Err(_) => tally[m][2] += 1,
                Ok(got) if got == t => tally[m][0] += 1,
                Ok(_) => tally[m][1] += 1,
            }
        }
    }
    println!(
        "5000 random trees, values in {{0..3}}, n in [1, 30], {dups} with a repeated value; the checked builder returns:"
    );
    for (m, name) in ["map (last occurrence)", "scan (first in window)"]
        .iter()
        .enumerate()
    {
        println!(
            "  {:<24} the original {:>4}, a DIFFERENT tree with the same traversals {:>4}, an error {:>4}",
            name, tally[m][0], tally[m][1], tally[m][2]
        );
    }

    println!(
        "\n== (e) corrupt input: 10000 swaps or non-member replacements in a valid in-order, n in [2, 8]"
    );
    let (mut swaps, mut valid, mut panics, mut differ, mut accepted, mut disagree) =
        (0, 0, 0, 0, 0, 0);
    let mut by_err = [0usize; 6];
    // an unchecked panic is caught and counted; silence the default hook, which
    // would print one message per panic
    panic::set_hook(Box::new(|_| {}));
    for _ in 0..10000 {
        let n = 2 + r.int_n(7);
        let mut t = random_split(n, &mut r);
        let vals: Vec<i64> = (1..=n as i64).collect();
        relabel(&mut t, &vals, &mut 0);
        let (p, mut io) = (traverse(&t, Order::Pre), traverse(&t, Order::In));
        if r.int_n(2) == 0 {
            swaps += 1;
            let a = r.int_n(n);
            let mut b = r.int_n(n - 1);
            if b >= a {
                b += 1;
            }
            io.swap(a, b);
        } else {
            let i = r.int_n(n);
            io[i] = (n + 1 + r.int_n(5)) as i64;
        }
        let is_valid = valid_in[&n].contains(&str_of(&io)); // the exhaustive table of (c) is the oracle
        if is_valid {
            valid += 1;
        }
        let res = rebuild(&p, &io, false, true, true).0;
        if res.is_ok() != is_valid {
            disagree += 1;
        }
        match res {
            Ok(_) => accepted += 1,
            Err(e) => by_err[e as usize] += 1,
        }
        if !is_valid {
            match panic::catch_unwind(|| rebuild(&p, &io, false, true, false).0) {
                Err(_) => panics += 1,
                Ok(got) => {
                    let got = got.ok().flatten();
                    if traverse(&got, Order::Pre) != p || traverse(&got, Order::In) != io {
                        differ += 1;
                    }
                }
            }
        }
    }
    let _ = panic::take_hook();
    println!(
        "{swaps} swaps and {} replacements; {valid} corrupted inputs are still a valid (pre, in) pair (a swap can land on another tree's in-order)",
        10000 - swaps
    );
    println!(
        "checked builder: accepted {accepted}, rejected {}, disagreements with the exhaustive oracle {disagree}",
        10000 - accepted
    );
    for (name, count) in BUILD_ERR_NAMES.iter().zip(by_err) {
        println!("  rejected by {:<26} {count}", format!("{name}:"));
    }
    println!(
        "unchecked builder on the {} invalid inputs: panicked {panics}, returned a tree whose traversals differ from the input {differ}, returned a matching tree {}",
        10000 - valid,
        10000 - valid - panics - differ
    );
}
```

**Rust output:**

```
== (a) rebuild the hand-built tree from two traversals
pre  1 2 4 5 7 3 6 8 9
in   4 2 7 5 1 3 8 6 9
post 4 7 5 2 8 9 6 3 1
pre+in  scan  equal true  scan steps 16  map lookups 0
pre+in  map   equal true  scan steps  0  map lookups 9
post+in scan  equal true  scan steps 16  map lookups 0
post+in map   equal true  scan steps  0  map lookups 9
scan steps predicted by the sum of (left size + 1) over the nodes: 16
round trip over 3000 random trees (n in [0, 60], 91234 nodes in total, shuffled distinct labels): 0 mismatches in 4 builders each

== (b) finding the root: scanning the window vs a value -> index map
tree                      n    h |   scan pre+in  scan post+in   sum(left+1) |  lookups  inserts
randomSplit(1000)      1000   22 |          6860          6860          6860 |     2000     2000
randomSplit(100000)  100000   42 |       1103883       1103883       1103883 |   200000   200000
leftChain(2000)        2000 2000 |       2001000       2001000       2001000 |     4000     4000
rightChain(2000)       2000 2000 |          2000          2000          2000 |     4000     4000
all 16 rebuilt trees equal their originals: true
(lookups and inserts are summed over the pre+in and post+in map builds)

== (c) which pairs of traversals identify a tree? every shape of n = 1..8, distinct labels
columns 3-5: distinct pairs of sequences over all shapes; Motzkin: M(n-1) from its recurrence;
full: (pre,post) pairs that belong to exactly one shape; 2^k ok: every (pre,post) pair is shared by exactly
2^k shapes, k = nodes with one child, and full = Catalan((n-1)/2) for odd n, 0 for even n
  n  shapes |  (pre,in)  (post,in)  (pre,post) |   Motzkin  full 2^k ok
  1       1 |         1          1           1 |         1     1   true
  2       2 |         2          2           1 |         1     0   true
  3       5 |         5          5           2 |         2     1   true
  4      14 |        14         14           4 |         4     0   true
  5      42 |        42         42           9 |         9     2   true
  6     132 |       132        132          21 |        21     0   true
  7     429 |       429        429          51 |        51     5   true
  8    1430 |      1430       1430         127 |       127     0   true
one explicit collision: the 3-node shapes grouped by their (pre, post) pair
  pre 1 2 3, post 3 2 1    1(-,2(-,3))  1(-,2(3,-))  1(2(-,3),-)  1(2(3,-),-)
  pre 1 2 3, post 2 3 1    1(2,3)

== (d) repeated values break (pre, in) too: every shape of n nodes, every labelling over {0, 1}
  n  shapes  labelled | distinct pairs     trees sharing a pair |     all-equal: pairs
  1       1         2 |              2                        0 |                    1
  2       2         8 |              6                        4 |                    1
  3       5        40 |             20                       34 |                    1
  4      14       224 |             70                      212 |                    1
  5      42      1344 |            252                     1318 |                    1
  6     132      8448 |            924                     8392 |                    1
  7     429     54912 |           3432                    54794 |                    1
  8    1430    366080 |          12870                   365828 |                    1
5000 random trees, values in {0..3}, n in [1, 30], 4589 with a repeated value; the checked builder returns:
  map (last occurrence)    the original  411, a DIFFERENT tree with the same traversals    0, an error 4589
  scan (first in window)   the original  719, a DIFFERENT tree with the same traversals  980, an error 3301

== (e) corrupt input: 10000 swaps or non-member replacements in a valid in-order, n in [2, 8]
5068 swaps and 4932 replacements; 2971 corrupted inputs are still a valid (pre, in) pair (a swap can land on another tree's in-order)
checked builder: accepted 2971, rejected 7029, disagreements with the exhaustive oracle 0
  rejected by order ran out:             0
  rejected by root not in the in-order:  4932
  rejected by root outside its window:   2097
  rejected by unused tail:               0
  rejected by lengths differ:            0
  rejected by re-traversal differs:      0
unchecked builder on the 7029 invalid inputs: panicked 5805, returned a tree whose traversals differ from the input 1224, returned a matching tree 0
```

> **Go vs Rust:** Identical output (diffed): the Rust `Pcg` port gives both programs the same trees and corruptions, so every count matches, down to the 5,805 panics. The unchecked builder is where the languages differ in what they make you write: Go's `m[k]` reads a missing key as 0, a legal index, so the bug is silent, while Rust's `HashMap::get` returns an `Option` and the unchecked arm has to spell out `None => 0` (the checked arm is `None => return Err(NotMember)`). Go's `recover()` becomes `catch_unwind` with the default hook silenced; printing the panic messages in a scratch copy showed all 5,805 were `index out of bounds: the len is n but the index is n`, the pre-order read past its end. One `traverse(t, Order)` replaces Go's three traversal functions, and `Option<Box<Node>>` with `#[derive(PartialEq)]` makes `rebuilt == original` a one-token structural comparison (recursive, so only used on trees at most 2,000 deep here).

**Complexity:** scanning for the root costs sum(size(left) + 1) = O(n·h) steps: 6,860 on a random 1,000-node tree, 2,001,000 = n(n+1)/2 on a 2,000-node left chain, but only 2,000 on a right chain, because the lean decides how far into the window the root sits. The map costs exactly n lookups (after n inserts) on every shape. Over all 2,055 shapes of 1 to 8 nodes, (pre, in) and (post, in) give C(n) distinct pairs while (pre, post) gives only the Motzkin number M(n−1) (127 of 1,430 at n = 8), and with values from {0, 1} just 12,870 (pre, in) pairs stand for 366,080 trees at n = 8. On 10,000 corrupted inputs the checked builder agreed with the oracle every time; the unchecked builder panicked on 5,805 of the 7,029 invalid ones and returned a wrong tree for the other 1,224, and the final re-traversal rejected none that the window checks had let through.

---

> Next tier: [🔴 hard](3-hard.md) — Morris traversal, trees too deep to recurse over, three ways to own
> a tree in Rust, traversals as iterators, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
