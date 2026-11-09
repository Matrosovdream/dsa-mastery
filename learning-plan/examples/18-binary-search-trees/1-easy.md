# Step 18 — Binary Search Trees · 🟢 Easy

Examples **1–6**: the invariant and what a search costs, insert as a failed search, the in-order walk, the
neighbour queries, delete in its three cases, and the input that turns the tree into a linked list.

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

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. The invariant, and search as a descent

`🟢 easy` · *The BST invariant*

A binary search tree is a binary tree with one rule about keys: **every** key in a node's left subtree is smaller and **every** key in its right subtree is larger — a rule about subtrees, not about children. That one rule lets a search throw away a whole subtree per comparison, which is why a tree is the right tool when data must stay ordered *and* change, and it makes the cost of a search exactly countable: a hit costs depth + 1 comparisons, a miss costs the depth of the nil link where the descent fell off, and the totals over all nodes (`I`) and all nil links (`E`) are tied by `E = I + 2n`. Search is Θ(h), where h is the height; this example measures it on a 12-key tree and then shows a 3-node tree that obeys the weaker "child versus parent" rule and still is not a BST.

**Steps:**

1. Insert the 12 keys in a fixed order, print the tree sideways, and trace five searches (a root hit, a deep hit, and three misses) with the keys they visit.
2. Sum depths: compute `I` and `E` in one walk, then confirm them by really searching for every key and for one key inside every gap, and compare with a sorted slice.
3. Check the invariant the slow way (collect every key of each subtree) on the tree and on the 3-node counterexample, where the parent/child check says `true` and the definition says `false`.

```go
package main

import (
	"cmp"
	"fmt"
	"strings"
	"unsafe"
)

// A binary search tree is a binary tree (lesson 17) with ONE rule about keys:
// for every node, EVERY key in its left subtree is smaller and EVERY key in
// its right subtree is larger. "Every key in the subtree" -- not "the child".
type node struct {
	key         int
	left, right *node
}

// insert hangs a new key on the nil link where a search for it falls off.
// (Example 2 is about insert; here it only builds the trees.)
func insert(n *node, key int) *node {
	if n == nil {
		return &node{key: key}
	}
	if key < n.key {
		n.left = insert(n.left, key)
	} else if key > n.key {
		n.right = insert(n.right, key)
	}
	return n
}

// search is a descent. Every iteration is ONE comparison and it throws away a
// whole subtree: go left and everything on the right is gone, and vice versa.
// It returns the keys it visited; the comparison count is len(path).
func search(root *node, key int) (path []int, found bool) {
	for n := root; n != nil; {
		path = append(path, n.key)
		switch c := cmp.Compare(key, n.key); {
		case c < 0:
			n = n.left
		case c > 0:
			n = n.right
		default:
			return path, true
		}
	}
	return path, false
}

// show prints the tree sideways: right subtree on top, four spaces per level.
func show(n *node, depth int) {
	if n == nil {
		return
	}
	show(n.right, depth+1)
	fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.key)
	show(n.left, depth+1)
}

func height(n *node) int { // nodes on the longest root-to-nil path
	if n == nil {
		return 0
	}
	return 1 + max(height(n.left), height(n.right))
}

func collect(n *node, out *[]int) { // in-order: left, node, right
	if n != nil {
		collect(n.left, out)
		*out = append(*out, n.key)
		collect(n.right, out)
	}
}

// sums walks the tree once. A node at depth d is found with d+1 comparisons.
// A nil link at depth d is a miss that cost d comparisons (the root is depth 0).
func sums(n *node, d int) (I, nils, E int) {
	if n == nil {
		return 0, 1, d
	}
	li, ln, le := sums(n.left, d+1)
	ri, rn, re := sums(n.right, d+1)
	return d + li + ri, ln + rn, le + re
}

// satisfies is the definition, the slow way: for every node collect EVERY key
// of each subtree and compare it with the node's key. O(n^2) on a path.
func satisfies(n *node) bool {
	if n == nil {
		return true
	}
	var l, r []int
	collect(n.left, &l)
	collect(n.right, &r)
	for _, k := range l {
		if k >= n.key {
			return false
		}
	}
	for _, k := range r {
		if k <= n.key {
			return false
		}
	}
	return satisfies(n.left) && satisfies(n.right)
}

// childrenOK is the tempting weaker rule: each child compares correctly with
// its own parent, and nothing else.
func childrenOK(n *node) bool {
	if n == nil {
		return true
	}
	if n.left != nil && n.left.key >= n.key || n.right != nil && n.right.key <= n.key {
		return false
	}
	return childrenOK(n.left) && childrenOK(n.right)
}

// probes counts the array elements a lower-bound binary search reads.
func probes(a []int, key int) int {
	lo, hi, p := 0, len(a), 0
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		p++
		if a[mid] < key {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return p
}

func main() {
	order := []int{50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25}
	var root *node
	for _, k := range order {
		root = insert(root, k)
	}
	n := len(order)

	fmt.Println("(a) inserted in the order", order, "- sideways, right subtree on top:")
	show(root, 0)
	fmt.Printf("n = %d, height = %d, one node = %d bytes (key + two pointers)\n", n, height(root), unsafe.Sizeof(node{}))

	fmt.Println("\n(b) five searches:")
	for _, k := range []int{50, 25, 36, 5, 99} {
		path, found := search(root, k)
		if found {
			fmt.Printf("  search %2d: hit  comparisons = %d (node depth %d), visited %v\n", k, len(path), len(path)-1, path)
		} else {
			fmt.Printf("  search %2d: miss comparisons = %d (fell off at depth %d), visited %v\n", k, len(path), len(path), path)
		}
	}

	// Totals. The structure gives I and E in one walk; to check it we also
	// SEARCH for every key and for one key inside every gap (every key here is
	// a multiple of 5, so k-1 lies in the gap just below k).
	var keys []int
	collect(root, &keys)
	I, nils, E := sums(root, 0)
	hits, misses := 0, 0
	hitP, missP, minP, maxP := 0, 0, len(keys), 0 // sorted-slice probes
	for _, k := range keys {
		p, _ := search(root, k)
		hits += len(p)
		q := probes(keys, k)
		hitP, minP, maxP = hitP+q, min(minP, q), max(maxP, q)
	}
	gaps := []int{keys[n-1] + 1}
	for _, k := range keys {
		gaps = append(gaps, k-1)
	}
	for _, g := range gaps {
		p, found := search(root, g)
		if found {
			panic("a gap key was found")
		}
		misses += len(p)
		q := probes(keys, g)
		missP, minP, maxP = missP+q, min(minP, q), max(maxP, q)
	}

	fmt.Println("\n(c) totals over every key and every gap:")
	fmt.Printf("  I = sum of node depths       = %d   (n = %d nodes)\n", I, n)
	fmt.Printf("  E = sum of nil-link depths   = %d   (%d nil links = n + 1)\n", E, nils)
	fmt.Printf("  E - I = %d, and 2n = %d: equal = %v\n", E-I, 2*n, E-I == 2*n)
	fmt.Printf("  comparisons over all %d hits   = %d (I + n = %d)\n", n, hits, I+n)
	fmt.Printf("  comparisons over all %d misses = %d (E = %d)\n", nils, misses, E)
	fmt.Printf("  mean per hit  = I/n + 1   = %.3f\n", float64(I)/float64(n)+1)
	fmt.Printf("  mean per miss = E/(n + 1) = %.3f\n", float64(E)/float64(n+1))
	fmt.Printf("  sorted slice of the same keys: %d to %d probes per search; mean %.3f per hit, %.3f per miss\n",
		minP, maxP, float64(hitP)/float64(n), float64(missP)/float64(n+1))

	fmt.Println("\n(d) the invariant, checked the slow way:")
	fmt.Println("  satisfies(tree above)       =", satisfies(root))
	bad := &node{key: 2, right: &node{key: 3, left: &node{key: 1}}}
	fmt.Println("  the 3-node tree (2, right child 3, whose left child is 1):")
	show(bad, 0)
	fmt.Println("  each child vs its parent    =", childrenOK(bad))
	fmt.Println("  satisfies(3-node tree)      =", satisfies(bad), " <- 1 is in 2's RIGHT subtree")
}
```

**Output:**

```
(a) inserted in the order [50 30 70 20 40 60 80 35 45 65 10 25] - sideways, right subtree on top:
        80
    70
            65
        60
50
            45
        40
            35
    30
            25
        20
            10
n = 12, height = 4, one node = 24 bytes (key + two pointers)

(b) five searches:
  search 50: hit  comparisons = 1 (node depth 0), visited [50]
  search 25: hit  comparisons = 4 (node depth 3), visited [50 30 20 25]
  search 36: miss comparisons = 4 (fell off at depth 4), visited [50 30 40 35]
  search  5: miss comparisons = 4 (fell off at depth 4), visited [50 30 20 10]
  search 99: miss comparisons = 3 (fell off at depth 3), visited [50 70 80]

(c) totals over every key and every gap:
  I = sum of node depths       = 25   (n = 12 nodes)
  E = sum of nil-link depths   = 49   (13 nil links = n + 1)
  E - I = 24, and 2n = 24: equal = true
  comparisons over all 12 hits   = 37 (I + n = 37)
  comparisons over all 13 misses = 49 (E = 49)
  mean per hit  = I/n + 1   = 3.083
  mean per miss = E/(n + 1) = 3.769
  sorted slice of the same keys: 3 to 4 probes per search; mean 3.833 per hit, 3.769 per miss

(d) the invariant, checked the slow way:
  satisfies(tree above)       = true
  the 3-node tree (2, right child 3, whose left child is 1):
    3
        1
2
  each child vs its parent    = true
  satisfies(3-node tree)      = false  <- 1 is in 2's RIGHT subtree
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::mem::size_of;

// A binary search tree is a binary tree (lesson 17) with ONE rule about keys:
// for every node, EVERY key in its left subtree is smaller and EVERY key in
// its right subtree is larger. "Every key in the subtree" -- not "the child".
// A link is Option<Box<Node>>: None is the empty tree, Go's nil.
struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Go needs no destructor. Rust's default drop glue recurses once per level, which
// is harmless on a 12-node tree but not on a 100,000-node chain. This Drop hands
// the children to a heap-allocated stack instead, so no tree here can overflow.
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

fn node(key: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { key, left, right }))
}

// insert hangs a new key on the nil link where a search for it falls off.
// (Example 2 is about insert; here it only builds the trees.)
fn insert(link: &mut Link, key: i64) {
    match link {
        None => *link = node(key, None, None),
        Some(n) => match key.cmp(&n.key) {
            Ordering::Less => insert(&mut n.left, key),
            Ordering::Greater => insert(&mut n.right, key),
            Ordering::Equal => {}
        },
    }
}

// search is a descent. Every iteration is ONE comparison and it throws away a
// whole subtree. The cursor is a shared borrow, `Option<&Node>`: as_deref()
// turns `&Option<Box<Node>>` into `Option<&Node>`, so nothing is ever moved.
// It returns the keys it visited; the comparison count is path.len().
fn search(root: &Link, key: i64) -> (Vec<i64>, bool) {
    let mut path = Vec::new();
    let mut cur = root.as_deref();
    while let Some(n) = cur {
        path.push(n.key);
        match key.cmp(&n.key) {
            Ordering::Less => cur = n.left.as_deref(),
            Ordering::Greater => cur = n.right.as_deref(),
            Ordering::Equal => return (path, true),
        }
    }
    (path, false)
}

// show prints the tree sideways: right subtree on top, four spaces per level.
fn show(n: &Link, depth: usize) {
    if let Some(n) = n {
        show(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.key);
        show(&n.left, depth + 1);
    }
}

// the number of nodes on the longest root-to-nil path
fn height(n: &Link) -> usize {
    n.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

// in-order: left, node, right
fn collect(n: &Link, out: &mut Vec<i64>) {
    if let Some(n) = n {
        collect(&n.left, out);
        out.push(n.key);
        collect(&n.right, out);
    }
}

// sums walks the tree once. A node at depth d is found with d+1 comparisons.
// A nil link at depth d is a miss that cost d comparisons (the root is depth 0).
fn sums(n: &Link, d: usize) -> (usize, usize, usize) {
    match n {
        None => (0, 1, d),
        Some(n) => {
            let (li, ln, le) = sums(&n.left, d + 1);
            let (ri, rn, re) = sums(&n.right, d + 1);
            (d + li + ri, ln + rn, le + re)
        }
    }
}

// satisfies is the definition, the slow way: for every node collect EVERY key
// of each subtree and compare it with the node's key. O(n^2) on a path.
fn satisfies(n: &Link) -> bool {
    let Some(n) = n else { return true };
    let (mut l, mut r) = (Vec::new(), Vec::new());
    collect(&n.left, &mut l);
    collect(&n.right, &mut r);
    l.iter().all(|&k| k < n.key)
        && r.iter().all(|&k| k > n.key)
        && satisfies(&n.left)
        && satisfies(&n.right)
}

// children_ok is the tempting weaker rule: each child compares correctly with
// its own parent, and nothing else.
fn children_ok(n: &Link) -> bool {
    let Some(n) = n else { return true };
    let left_bad = n.left.as_ref().is_some_and(|l| l.key >= n.key);
    let right_bad = n.right.as_ref().is_some_and(|r| r.key <= n.key);
    !left_bad && !right_bad && children_ok(&n.left) && children_ok(&n.right)
}

// probes counts the array elements a lower-bound binary search reads.
fn probes(a: &[i64], key: i64) -> usize {
    let (mut lo, mut hi, mut p) = (0, a.len(), 0);
    while lo < hi {
        let mid = (lo + hi) / 2;
        p += 1;
        if a[mid] < key {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    p
}

// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    let order = [50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25];
    let mut root: Link = None;
    for k in order {
        insert(&mut root, k);
    }
    let n = order.len();

    println!(
        "(a) inserted in the order {} - sideways, right subtree on top:",
        go_slice(&order)
    );
    show(&root, 0);
    println!(
        "n = {n}, height = {}, one node = {} bytes (key + two pointers)",
        height(&root),
        size_of::<Node>()
    );
    println!(
        "a link, Option<Box<Node>>, = {} bytes: None is the null pointer, no tag",
        size_of::<Link>()
    );

    println!("\n(b) five searches:");
    for k in [50, 25, 36, 5, 99] {
        let (path, found) = search(&root, k);
        if found {
            println!(
                "  search {k:2}: hit  comparisons = {} (node depth {}), visited {}",
                path.len(),
                path.len() - 1,
                go_slice(&path)
            );
        } else {
            println!(
                "  search {k:2}: miss comparisons = {} (fell off at depth {}), visited {}",
                path.len(),
                path.len(),
                go_slice(&path)
            );
        }
    }

    // Totals. The structure gives I and E in one walk; to check it we also
    // SEARCH for every key and for one key inside every gap (every key here is
    // a multiple of 5, so k-1 lies in the gap just below k).
    let mut keys = Vec::new();
    collect(&root, &mut keys);
    let (i, nils, e) = sums(&root, 0);
    let (mut hits, mut misses) = (0, 0);
    let (mut hit_p, mut miss_p, mut min_p, mut max_p) = (0, 0, keys.len(), 0); // sorted-slice probes
    for &k in &keys {
        hits += search(&root, k).0.len();
        let q = probes(&keys, k);
        hit_p += q;
        (min_p, max_p) = (min_p.min(q), max_p.max(q));
    }
    let mut gaps = vec![keys[n - 1] + 1];
    gaps.extend(keys.iter().map(|k| k - 1));
    for g in gaps {
        let (path, found) = search(&root, g);
        assert!(!found, "a gap key was found");
        misses += path.len();
        let q = probes(&keys, g);
        miss_p += q;
        (min_p, max_p) = (min_p.min(q), max_p.max(q));
    }

    println!("\n(c) totals over every key and every gap:");
    println!("  I = sum of node depths       = {i}   (n = {n} nodes)");
    println!("  E = sum of nil-link depths   = {e}   ({nils} nil links = n + 1)");
    println!(
        "  E - I = {}, and 2n = {}: equal = {}",
        e - i,
        2 * n,
        e - i == 2 * n
    );
    println!(
        "  comparisons over all {n} hits   = {hits} (I + n = {})",
        i + n
    );
    println!("  comparisons over all {nils} misses = {misses} (E = {e})");
    println!(
        "  mean per hit  = I/n + 1   = {:.3}",
        i as f64 / n as f64 + 1.0
    );
    println!(
        "  mean per miss = E/(n + 1) = {:.3}",
        e as f64 / (n + 1) as f64
    );
    println!(
        "  sorted slice of the same keys: {min_p} to {max_p} probes per search; mean {:.3} per hit, {:.3} per miss",
        hit_p as f64 / n as f64,
        miss_p as f64 / (n + 1) as f64
    );

    println!("\n(d) the invariant, checked the slow way:");
    println!("  satisfies(tree above)       = {}", satisfies(&root));
    let bad = node(2, None, node(3, node(1, None, None), None));
    println!("  the 3-node tree (2, right child 3, whose left child is 1):");
    show(&bad, 0);
    println!("  each child vs its parent    = {}", children_ok(&bad));
    println!(
        "  satisfies(3-node tree)      = {}  <- 1 is in 2's RIGHT subtree",
        satisfies(&bad)
    );
}
```

**Rust output:**

```
(a) inserted in the order [50 30 70 20 40 60 80 35 45 65 10 25] - sideways, right subtree on top:
        80
    70
            65
        60
50
            45
        40
            35
    30
            25
        20
            10
n = 12, height = 4, one node = 24 bytes (key + two pointers)
a link, Option<Box<Node>>, = 8 bytes: None is the null pointer, no tag

(b) five searches:
  search 50: hit  comparisons = 1 (node depth 0), visited [50]
  search 25: hit  comparisons = 4 (node depth 3), visited [50 30 20 25]
  search 36: miss comparisons = 4 (fell off at depth 4), visited [50 30 40 35]
  search  5: miss comparisons = 4 (fell off at depth 4), visited [50 30 20 10]
  search 99: miss comparisons = 3 (fell off at depth 3), visited [50 70 80]

(c) totals over every key and every gap:
  I = sum of node depths       = 25   (n = 12 nodes)
  E = sum of nil-link depths   = 49   (13 nil links = n + 1)
  E - I = 24, and 2n = 24: equal = true
  comparisons over all 12 hits   = 37 (I + n = 37)
  comparisons over all 13 misses = 49 (E = 49)
  mean per hit  = I/n + 1   = 3.083
  mean per miss = E/(n + 1) = 3.769
  sorted slice of the same keys: 3 to 4 probes per search; mean 3.833 per hit, 3.769 per miss

(d) the invariant, checked the slow way:
  satisfies(tree above)       = true
  the 3-node tree (2, right child 3, whose left child is 1):
    3
        1
2
  each child vs its parent    = true
  satisfies(3-node tree)      = false  <- 1 is in 2's RIGHT subtree
```

> **Go vs Rust:** Identical to Go line for line (diffed), except the extra line that prints `size_of::<Option<Box<Node>>>()`: 8 bytes, the same as a bare `Box`, because `None` is stored as the null pointer; a `Node` is 24 bytes in both languages. The descent keeps only a shared borrow (`Option<&Node>` from `as_deref()`), `Ord::cmp` returns an `Ordering` you `match` where `cmp.Compare` returns -1/0/+1, and the `Drop` impl (iterative) has no Go counterpart because Go has no destructor to recurse in.

**Complexity:** search Θ(h) comparisons: a hit costs depth + 1, a miss costs the depth of the nil link it fell off, and `E = I + 2n` holds for every binary tree (here 49 = 25 + 24) · h ≥ ⌊log₂ n⌋ + 1 and h ≤ n · measured: 3.083 comparisons per hit and 3.769 per miss on this 12-key tree (height 4), against 3.833 and 3.769 probes on a sorted slice · checking only child against parent accepts a tree that is not a BST.

---

## 2. Insert is a failed search

`🟢 easy` · *Insert and insertion order*

Insert walks down exactly as a search does, falls off the tree, and hangs the new node on the nil link where it fell; nothing already in the tree ever moves. That makes the shape of a BST a pure function of the insertion order, so the same keys in different orders give different trees, and the comparisons an insert spends equal the new node's depth, which sums to the final tree's `I`. Each insert is Θ(h). The example builds 1..7 in three orders, shows what a duplicate insert does, and then runs **all** orders of 1..n for n = 5, 6 and 7 to count the shapes and heights they produce.

**Steps:**

1. Insert 1..7 in a balanced, a sorted and a zig-zag order with a pointer to the *link* (`link := &t.root`), print each tree with the comparisons of every insert, and compare their total with `I`.
2. Insert keys that already exist: the tree is left alone and the size does not change.
3. Generate every permutation of 1..n, build each tree, and tabulate heights and distinct shapes (a shape written as an s-expression), next to Catalan(n) and 2^(n−1) computed.

```go
package main

import (
	"cmp"
	"fmt"
	"slices"
	"strings"
)

type node struct {
	key         int
	left, right *node
}

type tree struct {
	root *node
	size int
}

// insert is a FAILED search: walk down until you fall off the tree, then hang
// the new node on the nil link where you fell off. `link` points at the LINK
// (the root field, or some node's left/right field), not at a node -- so the
// empty tree, the root and a leaf's child are all the same case. Nothing that
// is already in the tree moves, so the shape depends only on the order of the
// inserts. It returns whether it inserted, and the comparisons it spent.
func (t *tree) insert(key int) (inserted bool, cmps int) {
	link := &t.root
	for *link != nil {
		cmps++
		switch c := cmp.Compare(key, (*link).key); {
		case c < 0:
			link = &(*link).left
		case c > 0:
			link = &(*link).right
		default:
			return false, cmps // already there: policy = leave the tree alone
		}
	}
	*link = &node{key: key}
	t.size++
	return true, cmps
}

func show(n *node, depth int) {
	if n != nil {
		show(n.right, depth+1)
		fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.key)
		show(n.left, depth+1)
	}
}

func height(n *node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.left), height(n.right))
}

func sumDepths(n *node, d int) int { // I
	if n == nil {
		return 0
	}
	return d + sumDepths(n.left, d+1) + sumDepths(n.right, d+1)
}

// shape writes the tree without its keys: "." is nil, "(l r)" is a node.
func shape(n *node) string {
	if n == nil {
		return "."
	}
	return "(" + shape(n.left) + shape(n.right) + ")"
}

func permute(a []int, k int, visit func([]int)) {
	if k == len(a) {
		visit(a)
		return
	}
	for i := k; i < len(a); i++ {
		a[k], a[i] = a[i], a[k]
		permute(a, k+1, visit)
		a[k], a[i] = a[i], a[k]
	}
}

func catalan(n int) int {
	c := 1
	for i := 0; i < n; i++ {
		c = c * 2 * (2*i + 1) / (i + 2)
	}
	return c
}

func main() {
	fmt.Println("(a) the same keys 1..7, three insertion orders:")
	for _, o := range []struct {
		name string
		keys []int
	}{
		{"balanced", []int{4, 2, 6, 1, 3, 5, 7}},
		{"sorted", []int{1, 2, 3, 4, 5, 6, 7}},
		{"zig-zag", []int{1, 7, 2, 6, 3, 5, 4}},
	} {
		var t tree
		var per []int
		total := 0
		for _, k := range o.keys {
			_, c := t.insert(k)
			per = append(per, c)
			total += c
		}
		fmt.Printf("\n%s order %v\n", o.name, o.keys)
		show(t.root, 0)
		fmt.Printf("height %d, comparisons per insert %v, total %d, I of the final tree = %d, equal = %v\n",
			height(t.root), per, total, sumDepths(t.root, 0), total == sumDepths(t.root, 0))
	}

	fmt.Println("\n(b) inserting a key that is already there:")
	var t tree
	for _, k := range []int{4, 2, 6, 1, 3, 5, 7} {
		t.insert(k)
	}
	before := shape(t.root)
	for _, k := range []int{4, 3, 7} {
		sizeBefore := t.size
		ok, c := t.insert(k)
		fmt.Printf("  insert(%d): inserted = %-5v comparisons = %d, size %d -> %d\n", k, ok, c, sizeBefore, t.size)
	}
	fmt.Println("  shape unchanged:", shape(t.root) == before)

	fmt.Println("\n(c) EVERY insertion order of 1..n, and what each one builds:")
	for _, n := range []int{5, 6, 7} {
		keys := make([]int, n)
		for i := range keys {
			keys[i] = i + 1
		}
		orders := 0
		byHeight := map[int]int{}    // height -> number of orders
		perShape := map[string]int{} // shape -> number of orders
		shapeHeight := map[string]int{}
		sortedHeight := 0
		permute(keys, 0, func(p []int) {
			var t tree
			for _, k := range p {
				t.insert(k)
			}
			h := height(t.root)
			orders++
			byHeight[h]++
			s := shape(t.root)
			perShape[s]++
			shapeHeight[s] = h
			if slices.IsSorted(p) {
				sortedHeight = h
			}
		})
		shapesAt := map[int]int{}
		mostOrders := 0
		maxPerPath := 0
		for s, c := range perShape {
			shapesAt[shapeHeight[s]]++
			mostOrders = max(mostOrders, c)
			if shapeHeight[s] == n {
				maxPerPath = max(maxPerPath, c)
			}
		}
		fmt.Printf("\nn = %d: %d orders build %d distinct shapes; Catalan(%d) = %d, so every shape is reachable: %v\n",
			n, orders, len(perShape), n, catalan(n), len(perShape) == catalan(n))
		fmt.Println("  height  orders  shapes")
		for _, h := range slices.Sorted(func(yield func(int) bool) {
			for h := range byHeight {
				if !yield(h) {
					return
				}
			}
		}) {
			fmt.Printf("  %6d  %6d  %6d\n", h, byHeight[h], shapesAt[h])
		}
		fmt.Printf("  the most orders any one shape gets: %d\n", mostOrders)
		fmt.Printf("  height %d (a path): %d orders, %d shapes, at most %d order per shape; 2^(n-1) = %d; sorted order has height %d\n",
			n, byHeight[n], shapesAt[n], maxPerPath, 1<<(n-1), sortedHeight)
	}
}
```

**Output:**

```
(a) the same keys 1..7, three insertion orders:

balanced order [4 2 6 1 3 5 7]
        7
    6
        5
4
        3
    2
        1
height 3, comparisons per insert [0 1 1 2 2 2 2], total 10, I of the final tree = 10, equal = true

sorted order [1 2 3 4 5 6 7]
                        7
                    6
                5
            4
        3
    2
1
height 7, comparisons per insert [0 1 2 3 4 5 6], total 21, I of the final tree = 21, equal = true

zig-zag order [1 7 2 6 3 5 4]
    7
            6
                    5
                        4
                3
        2
1
height 7, comparisons per insert [0 1 2 3 4 5 6], total 21, I of the final tree = 21, equal = true

(b) inserting a key that is already there:
  insert(4): inserted = false comparisons = 1, size 7 -> 7
  insert(3): inserted = false comparisons = 3, size 7 -> 7
  insert(7): inserted = false comparisons = 3, size 7 -> 7
  shape unchanged: true

(c) EVERY insertion order of 1..n, and what each one builds:

n = 5: 120 orders build 42 distinct shapes; Catalan(5) = 42, so every shape is reachable: true
  height  orders  shapes
       3      40       6
       4      64      20
       5      16      16
  the most orders any one shape gets: 8
  height 5 (a path): 16 orders, 16 shapes, at most 1 order per shape; 2^(n-1) = 16; sorted order has height 5

n = 6: 720 orders build 132 distinct shapes; Catalan(6) = 132, so every shape is reachable: true
  height  orders  shapes
       3      80       4
       4     400      40
       5     208      56
       6      32      32
  the most orders any one shape gets: 20
  height 6 (a path): 32 orders, 32 shapes, at most 1 order per shape; 2^(n-1) = 32; sorted order has height 6

n = 7: 5040 orders build 429 distinct shapes; Catalan(7) = 429, so every shape is reachable: true
  height  orders  shapes
       3      80       1
       4    2240      68
       5    2048     152
       6     608     144
       7      64      64
  the most orders any one shape gets: 80
  height 7 (a path): 64 orders, 64 shapes, at most 1 order per shape; 2^(n-1) = 64; sorted order has height 7
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::{BTreeMap, HashMap};

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// An iterative Drop: the default drop glue recurses once per level, and the
// sorted-input tree in (a) is a chain. (Example 6 builds chains of 4,000.)
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

#[derive(Default)]
struct Tree {
    root: Link,
    size: usize,
}

impl Tree {
    // insert is a FAILED search: walk down until you fall off the tree, then
    // hang the new node on the nil link where you fell off. `cur` is a
    // `&mut Option<Box<Node>>`: a mutable reference to the LINK (the root field,
    // or some node's left/right field), not to a node -- so the empty tree, the
    // root and a leaf's child are all the same case. This loop COMPILES for
    // insert; the same shape for delete does not (example 12). Nothing already
    // in the tree moves, so the shape depends only on the order of the inserts.
    // It returns whether it inserted, and the comparisons it spent.
    fn insert(&mut self, key: i64) -> (bool, u64) {
        let mut cmps = 0;
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            cmps += 1;
            match key.cmp(&n.key) {
                Ordering::Less => cur = &mut n.left,
                Ordering::Greater => cur = &mut n.right,
                Ordering::Equal => return (false, cmps), // policy: leave the tree alone
            }
        }
        *cur = Some(Box::new(Node {
            key,
            left: None,
            right: None,
        }));
        self.size += 1;
        (true, cmps)
    }
}

fn show(n: &Link, depth: usize) {
    if let Some(n) = n {
        show(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.key);
        show(&n.left, depth + 1);
    }
}

fn height(n: &Link) -> usize {
    n.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

// I: the sum of the depths of all nodes
fn sum_depths(n: &Link, d: u64) -> u64 {
    n.as_ref().map_or(0, |n| {
        d + sum_depths(&n.left, d + 1) + sum_depths(&n.right, d + 1)
    })
}

// shape writes the tree without its keys: "." is nil, "(l r)" is a node.
fn shape(n: &Link) -> String {
    match n {
        None => ".".to_string(),
        Some(n) => format!("({}{})", shape(&n.left), shape(&n.right)),
    }
}

fn permute(a: &mut [i64], k: usize, visit: &mut impl FnMut(&[i64])) {
    if k == a.len() {
        visit(a);
        return;
    }
    for i in k..a.len() {
        a.swap(k, i);
        permute(a, k + 1, visit);
        a.swap(k, i);
    }
}

fn catalan(n: u64) -> u64 {
    (0..n).fold(1, |c, i| c * 2 * (2 * i + 1) / (i + 2))
}

// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand.
fn go_slice<T: ToString>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    println!("(a) the same keys 1..7, three insertion orders:");
    let orders: [(&str, [i64; 7]); 3] = [
        ("balanced", [4, 2, 6, 1, 3, 5, 7]),
        ("sorted", [1, 2, 3, 4, 5, 6, 7]),
        ("zig-zag", [1, 7, 2, 6, 3, 5, 4]),
    ];
    for (name, keys) in orders {
        let mut t = Tree::default();
        let per: Vec<u64> = keys.iter().map(|&k| t.insert(k).1).collect();
        let total: u64 = per.iter().sum();
        let i = sum_depths(&t.root, 0);
        println!("\n{name} order {}", go_slice(&keys));
        show(&t.root, 0);
        println!(
            "height {}, comparisons per insert {}, total {total}, I of the final tree = {i}, equal = {}",
            height(&t.root),
            go_slice(&per),
            total == i
        );
    }

    println!("\n(b) inserting a key that is already there:");
    let mut t = Tree::default();
    for k in [4, 2, 6, 1, 3, 5, 7] {
        t.insert(k);
    }
    let before = shape(&t.root);
    for k in [4, 3, 7] {
        let size_before = t.size;
        let (ok, c) = t.insert(k);
        println!(
            "  insert({k}): inserted = {ok:<5} comparisons = {c}, size {size_before} -> {}",
            t.size
        );
    }
    println!("  shape unchanged: {}", shape(&t.root) == before);

    println!("\n(c) EVERY insertion order of 1..n, and what each one builds:");
    for n in [5usize, 6, 7] {
        let mut keys: Vec<i64> = (1..=n as i64).collect();
        let mut count = 0;
        let mut by_height: BTreeMap<usize, usize> = BTreeMap::new(); // height -> orders
        let mut per_shape: HashMap<String, usize> = HashMap::new(); // shape -> orders
        let mut shape_height: HashMap<String, usize> = HashMap::new();
        let mut sorted_height = 0;
        permute(&mut keys, 0, &mut |p| {
            let mut t = Tree::default();
            for &k in p {
                t.insert(k);
            }
            let h = height(&t.root);
            count += 1;
            *by_height.entry(h).or_default() += 1;
            let s = shape(&t.root);
            shape_height.insert(s.clone(), h);
            *per_shape.entry(s).or_default() += 1;
            if p.is_sorted() {
                sorted_height = h;
            }
        });
        let mut shapes_at: HashMap<usize, usize> = HashMap::new();
        let (mut most_orders, mut max_per_path) = (0, 0);
        for (s, &c) in &per_shape {
            *shapes_at.entry(shape_height[s]).or_default() += 1;
            most_orders = most_orders.max(c);
            if shape_height[s] == n {
                max_per_path = max_per_path.max(c);
            }
        }
        let cat = catalan(n as u64);
        println!(
            "\nn = {n}: {count} orders build {} distinct shapes; Catalan({n}) = {cat}, so every shape is reachable: {}",
            per_shape.len(),
            per_shape.len() as u64 == cat
        );
        println!("  height  orders  shapes");
        for (h, orders) in &by_height {
            println!("  {h:6}  {orders:6}  {:6}", shapes_at[h]);
        }
        println!("  the most orders any one shape gets: {most_orders}");
        println!(
            "  height {n} (a path): {} orders, {} shapes, at most {max_per_path} order per shape; 2^(n-1) = {}; sorted order has height {sorted_height}",
            by_height[&n],
            shapes_at[&n],
            1u64 << (n - 1)
        );
    }
}
```

**Rust output:**

```
(a) the same keys 1..7, three insertion orders:

balanced order [4 2 6 1 3 5 7]
        7
    6
        5
4
        3
    2
        1
height 3, comparisons per insert [0 1 1 2 2 2 2], total 10, I of the final tree = 10, equal = true

sorted order [1 2 3 4 5 6 7]
                        7
                    6
                5
            4
        3
    2
1
height 7, comparisons per insert [0 1 2 3 4 5 6], total 21, I of the final tree = 21, equal = true

zig-zag order [1 7 2 6 3 5 4]
    7
            6
                    5
                        4
                3
        2
1
height 7, comparisons per insert [0 1 2 3 4 5 6], total 21, I of the final tree = 21, equal = true

(b) inserting a key that is already there:
  insert(4): inserted = false comparisons = 1, size 7 -> 7
  insert(3): inserted = false comparisons = 3, size 7 -> 7
  insert(7): inserted = false comparisons = 3, size 7 -> 7
  shape unchanged: true

(c) EVERY insertion order of 1..n, and what each one builds:

n = 5: 120 orders build 42 distinct shapes; Catalan(5) = 42, so every shape is reachable: true
  height  orders  shapes
       3      40       6
       4      64      20
       5      16      16
  the most orders any one shape gets: 8
  height 5 (a path): 16 orders, 16 shapes, at most 1 order per shape; 2^(n-1) = 16; sorted order has height 5

n = 6: 720 orders build 132 distinct shapes; Catalan(6) = 132, so every shape is reachable: true
  height  orders  shapes
       3      80       4
       4     400      40
       5     208      56
       6      32      32
  the most orders any one shape gets: 20
  height 6 (a path): 32 orders, 32 shapes, at most 1 order per shape; 2^(n-1) = 32; sorted order has height 6

n = 7: 5040 orders build 429 distinct shapes; Catalan(7) = 429, so every shape is reachable: true
  height  orders  shapes
       3      80       1
       4    2240      68
       5    2048     152
       6     608     144
       7      64      64
  the most orders any one shape gets: 80
  height 7 (a path): 64 orders, 64 shapes, at most 1 order per shape; 2^(n-1) = 64; sorted order has height 7
```

> **Go vs Rust:** Identical to Go line for line (diffed). The insert loop with a `&mut Option<Box<Node>>` cursor compiles as written (`while let Some(n) = cur { … cur = &mut n.left … } *cur = Some(Box::new(…))`), which is Rust's version of Go's `link := &t.root`; the same loop shape does not compile for delete (examples 5 and 12). The permutation visitor is a `&mut impl FnMut(&[i64])` and `Vec::is_sorted` replaces `slices.IsSorted`.

**Complexity:** insert Θ(h) comparisons = the depth of the new node, so the n inserts cost `I` of the final tree in total (10 for the balanced 1..7 tree, 21 for both degenerate ones) · a duplicate insert costs depth + 1 comparisons and changes nothing · shapes: all 5,040 orders of 1..7 build 429 = Catalan(7) distinct shapes, but not equally often — the one perfect shape (height 3) is built by 80 orders, and each of the 64 = 2^(n−1) path shapes (height 7) by exactly one, the sorted order being one of them.

---

## 3. In-order is sorted

`🟢 easy` · *In-order = ascending*

The invariant, restated as an algorithm: visit the left subtree, the node, then the right subtree, and the keys come out in ascending order. That gives three things for free — an ordered dump of the keys, a **tree sort** (insert, then walk once), and the cheapest correctness test a BST has: after any operation the in-order sequence must be *strictly* ascending. The walk is Θ(n) time and Θ(h) stack; the tree sort costs the sum of the insert comparisons, Θ(n log n) expected for random order and Θ(n²) for sorted input (example 6). Traversals themselves are lesson 17's subject, and stopping a walk early is 17 example 15's; this one uses the walk as an oracle.

**Steps:**

1. Build the 12-key tree of example 1 and print its in-order sequence; in Go this is an `iter.Seq`, consumed with `slices.Collect` and a plain `for … range`.
2. Compare the in-order sequence of 1,000 random trees (duplicates drawn on purpose, so the tree must reject them) with the same input sorted and de-duplicated.
3. Tree-sort 1,000 distinct random keys and compare the insert comparisons with `n·log2 n`; then store duplicates on the right to see a *non-decreasing* in-order sequence that a strict checker rejects.

```go
package main

import (
	"cmp"
	"fmt"
	"iter"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
)

type node struct {
	key         int
	left, right *node
}

type tree struct {
	root *node
	size int
}

// insert rejects duplicates: it returns false and the comparisons spent.
func (t *tree) insert(key int) (inserted bool, cmps int) {
	link := &t.root
	for *link != nil {
		cmps++
		switch c := cmp.Compare(key, (*link).key); {
		case c < 0:
			link = &(*link).left
		case c > 0:
			link = &(*link).right
		default:
			return false, cmps
		}
	}
	*link = &node{key: key}
	t.size++
	return true, cmps
}

// insertRight is the other policy: an equal key goes into the right subtree.
func (t *tree) insertRight(key int) {
	link := &t.root
	for *link != nil {
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	*link = &node{key: key}
	t.size++
}

// All yields the keys ascending: left subtree, the node, right subtree. This is
// the invariant turned into an algorithm. (The yield protocol also lets a
// caller stop early; that mechanism is lesson 17, example 15.)
func (t *tree) All() iter.Seq[int] {
	return func(yield func(int) bool) {
		var walk func(n *node) bool
		walk = func(n *node) bool {
			return n == nil || walk(n.left) && yield(n.key) && walk(n.right)
		}
		walk(t.root)
	}
}

func show(n *node, depth int) {
	if n != nil {
		show(n.right, depth+1)
		fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.key)
		show(n.left, depth+1)
	}
}

func sumDepths(n *node, d int) int { // I
	if n == nil {
		return 0
	}
	return d + sumDepths(n.left, d+1) + sumDepths(n.right, d+1)
}

// strictlyAscending is the cheapest BST test there is: after ANY operation
// the in-order sequence must pass it.
func strictlyAscending(xs []int) bool {
	for i := 1; i < len(xs); i++ {
		if xs[i-1] >= xs[i] {
			return false
		}
	}
	return true
}

func main() {
	var t tree
	for _, k := range []int{50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25} {
		t.insert(k)
	}
	fmt.Println("(a) the tree of example 1, in-order:")
	fmt.Println("  slices.Collect(t.All()) =", slices.Collect(t.All()))
	fmt.Print("  for k := range t.All()  =")
	for k := range t.All() {
		fmt.Print(" ", k)
	}
	fmt.Println()

	// (b) 1,000 random trees against sort + dedup. Keys come from 0..99 and a
	// tree holds up to 59 of them, so duplicates are drawn and must be rejected.
	rng := rand.New(rand.NewPCG(1, 2))
	mismatches, stored, rejected, empty := 0, 0, 0, 0
	for range 1000 {
		var tr tree
		in := make([]int, rng.IntN(60))
		for i := range in {
			in[i] = rng.IntN(100)
			if ok, _ := tr.insert(in[i]); !ok {
				rejected++
			}
		}
		want := slices.Clone(in)
		slices.Sort(want)
		want = slices.Compact(want)
		if got := slices.Collect(tr.All()); !slices.Equal(got, want) || !strictlyAscending(got) {
			mismatches++
		}
		stored += tr.size
		if tr.size == 0 {
			empty++
		}
	}
	fmt.Printf("\n(b) %d mismatches over 1000 trees, %d keys stored (%d duplicate draws rejected, %d empty trees)\n",
		mismatches, stored, rejected, empty)

	// (c) tree sort: insert n distinct random keys, walk once.
	const n = 1000
	var ts tree
	var in []int
	cmpsTotal, draws := 0, 0
	for ts.size < n {
		k := rng.IntN(1 << 30)
		draws++
		if ok, c := ts.insert(k); ok {
			in = append(in, k)
			cmpsTotal += c
		}
	}
	out := slices.Collect(ts.All())
	want := slices.Clone(in)
	slices.Sort(want)
	nlogn := float64(n) * math.Log2(n)
	fmt.Printf("\n(c) tree sort of %d distinct random keys (%d draws): output equals sort = %v\n", n, draws, slices.Equal(out, want))
	fmt.Printf("  comparisons while inserting = %d, I of the final tree = %d\n", cmpsTotal, sumDepths(ts.root, 0))
	fmt.Printf("  n*log2(n) = %.0f, ratio = %.3f, mean per insert = %.2f\n", nlogn, float64(cmpsTotal)/nlogn, float64(cmpsTotal)/n)

	// (d) why the checker says < and not <=
	var d tree
	for _, k := range []int{5, 3, 5, 5, 7} {
		d.insertRight(k)
	}
	xs := slices.Collect(d.All())
	fmt.Println("\n(d) a tree that keeps equal keys on the right, built from 5 3 5 5 7:")
	show(d.root, 0)
	fmt.Println("  in-order:", xs)
	fmt.Println("  non-decreasing:", slices.IsSorted(xs), " strictly ascending:", strictlyAscending(xs))
}
```

**Output:**

```
(a) the tree of example 1, in-order:
  slices.Collect(t.All()) = [10 20 25 30 35 40 45 50 60 65 70 80]
  for k := range t.All()  = 10 20 25 30 35 40 45 50 60 65 70 80

(b) 0 mismatches over 1000 trees, 24427 keys stored (4878 duplicate draws rejected, 19 empty trees)

(c) tree sort of 1000 distinct random keys (1000 draws): output equals sort = true
  comparisons while inserting = 11815, I of the final tree = 11815
  n*log2(n) = 9966, ratio = 1.186, mean per insert = 11.81

(d) a tree that keeps equal keys on the right, built from 5 3 5 5 7:
            7
        5
    5
5
    3
  in-order: [3 5 5 5 7]
  non-decreasing: true  strictly ascending: false
```

**Rust:**

```rust
use std::cmp::Ordering;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the keys the Go program draws (seed 1, 2).
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

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (see example 1): safe for any shape of tree.
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

#[derive(Default)]
struct Tree {
    root: Link,
    size: usize,
}

impl Tree {
    // insert rejects duplicates: it returns false and the comparisons spent.
    fn insert(&mut self, key: i64) -> (bool, u64) {
        let mut cmps = 0;
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            cmps += 1;
            match key.cmp(&n.key) {
                Ordering::Less => cur = &mut n.left,
                Ordering::Greater => cur = &mut n.right,
                Ordering::Equal => return (false, cmps),
            }
        }
        *cur = Some(Box::new(Node {
            key,
            left: None,
            right: None,
        }));
        self.size += 1;
        (true, cmps)
    }

    // insert_right is the other policy: an equal key goes into the right subtree.
    fn insert_right(&mut self, key: i64) {
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            cur = if key < n.key {
                &mut n.left
            } else {
                &mut n.right
            };
        }
        *cur = Some(Box::new(Node {
            key,
            left: None,
            right: None,
        }));
        self.size += 1;
    }

    // The keys ascending: left subtree, the node, right subtree. This is the
    // invariant turned into an algorithm. (A lazy iterator that can stop early,
    // Go's iter.Seq, is lesson 17, example 15; here the walk fills a Vec.)
    fn keys(&self) -> Vec<i64> {
        fn walk(n: &Link, out: &mut Vec<i64>) {
            if let Some(n) = n {
                walk(&n.left, out);
                out.push(n.key);
                walk(&n.right, out);
            }
        }
        let mut out = Vec::with_capacity(self.size);
        walk(&self.root, &mut out);
        out
    }
}

fn show(n: &Link, depth: usize) {
    if let Some(n) = n {
        show(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.key);
        show(&n.left, depth + 1);
    }
}

// I: the sum of the depths of all nodes
fn sum_depths(n: &Link, d: u64) -> u64 {
    n.as_ref().map_or(0, |n| {
        d + sum_depths(&n.left, d + 1) + sum_depths(&n.right, d + 1)
    })
}

// strictly_ascending is the cheapest BST test there is: after ANY operation
// the in-order sequence must pass it.
fn strictly_ascending(xs: &[i64]) -> bool {
    xs.windows(2).all(|w| w[0] < w[1])
}

// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    let mut t = Tree::default();
    for k in [50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25] {
        t.insert(k);
    }
    println!("(a) the tree of example 1, in-order:");
    let keys = t.keys();
    println!("  t.keys() into a Vec     = {}", go_slice(&keys));
    print!("  for k in &keys          =");
    for k in &keys {
        print!(" {k}");
    }
    println!();

    // (b) 1,000 random trees against sort + dedup. Keys come from 0..99 and a
    // tree holds up to 59 of them, so duplicates are drawn and must be rejected.
    let mut rng = Pcg::new(1, 2);
    let (mut mismatches, mut stored, mut rejected, mut empty) = (0, 0, 0, 0);
    for _ in 0..1000 {
        let mut tr = Tree::default();
        let len = rng.int_n(60) as usize;
        let mut input = Vec::with_capacity(len);
        for _ in 0..len {
            let k = rng.int_n(100) as i64;
            input.push(k);
            if !tr.insert(k).0 {
                rejected += 1;
            }
        }
        let mut want = input.clone();
        want.sort();
        want.dedup();
        let got = tr.keys();
        if got != want || !strictly_ascending(&got) {
            mismatches += 1;
        }
        stored += tr.size;
        if tr.size == 0 {
            empty += 1;
        }
    }
    println!(
        "\n(b) {mismatches} mismatches over 1000 trees, {stored} keys stored ({rejected} duplicate draws rejected, {empty} empty trees)"
    );

    // (c) tree sort: insert n distinct random keys, walk once.
    const N: usize = 1000;
    let mut ts = Tree::default();
    let mut input = Vec::new();
    let (mut cmps_total, mut draws) = (0u64, 0);
    while ts.size < N {
        let k = rng.int_n(1 << 30) as i64;
        draws += 1;
        let (ok, c) = ts.insert(k);
        if ok {
            input.push(k);
            cmps_total += c;
        }
    }
    let out = ts.keys();
    let mut want = input.clone();
    want.sort();
    let nlogn = N as f64 * (N as f64).log2();
    println!(
        "\n(c) tree sort of {N} distinct random keys ({draws} draws): output equals sort = {}",
        out == want
    );
    println!(
        "  comparisons while inserting = {cmps_total}, I of the final tree = {}",
        sum_depths(&ts.root, 0)
    );
    println!(
        "  n*log2(n) = {nlogn:.0}, ratio = {:.3}, mean per insert = {:.2}",
        cmps_total as f64 / nlogn,
        cmps_total as f64 / N as f64
    );

    // (d) why the checker says < and not <=
    let mut d = Tree::default();
    for k in [5, 3, 5, 5, 7] {
        d.insert_right(k);
    }
    let xs = d.keys();
    println!("\n(d) a tree that keeps equal keys on the right, built from 5 3 5 5 7:");
    show(&d.root, 0);
    println!("  in-order: {}", go_slice(&xs));
    println!(
        "  non-decreasing: {}  strictly ascending: {}",
        xs.is_sorted(),
        strictly_ascending(&xs)
    );
}
```

**Rust output:**

```
(a) the tree of example 1, in-order:
  t.keys() into a Vec     = [10 20 25 30 35 40 45 50 60 65 70 80]
  for k in &keys          = 10 20 25 30 35 40 45 50 60 65 70 80

(b) 0 mismatches over 1000 trees, 24427 keys stored (4878 duplicate draws rejected, 19 empty trees)

(c) tree sort of 1000 distinct random keys (1000 draws): output equals sort = true
  comparisons while inserting = 11815, I of the final tree = 11815
  n*log2(n) = 9966, ratio = 1.186, mean per insert = 11.81

(d) a tree that keeps equal keys on the right, built from 5 3 5 5 7:
            7
        5
    5
5
    3
  in-order: [3 5 5 5 7]
  non-decreasing: true  strictly ascending: false
```

> **Go vs Rust:** The same keys, the same counts (24,427 stored, 4,878 rejected, 11,815 comparisons): the PCG-DXSM port draws what Go's `rand.New(rand.NewPCG(1, 2))` draws, and only the two labels in (a) differ. Go's walk is an `iter.Seq` the caller can `range` over; the Rust twin fills a `Vec` with a recursive `walk` (a borrowing `Iterator` is lesson 17 example 15), and `sort` + `dedup` and `Vec::is_sorted` play the roles of `slices.Sort` + `slices.Compact` and `slices.IsSorted`.

**Complexity:** in-order Θ(n) time, Θ(h) stack, strictly ascending · tree sort = the insert comparisons, which equal `I` of the final tree: 11,815 for 1,000 random keys (one tree), about 1.19 × `n·log2 n` = 9,966, i.e. 11.8 per insert · Θ(n²) for sorted input (example 6) · 0 mismatches against sort + dedup over 1,000 random trees (24,427 keys, 4,878 duplicate draws rejected) · "strictly" is a policy: keeping equal keys on the right makes the in-order sequence non-decreasing, and only `<=` accepts it.

---

## 4. Floor, ceiling, successor and predecessor

`🟢 easy` · *Navigating by order*

Floor (largest key ≤ k), ceiling (smallest key ≥ k), predecessor (largest key < k), successor (smallest key > k), min and max are each **one descent** from the root that remembers the last node where the descent turned away from a qualifying key. There are no parent pointers and no recursion, the cost is Θ(h) comparisons and Θ(1) space, and the query key does not have to be in the tree, which is what a hash map cannot do. The textbook "successor of a *node*" (leftmost node of the right subtree, else the lowest ancestor whose left subtree holds it) is the same answer found from below, and the example checks that too.

**Steps:**

1. Write the four descents, trace them on the 12-key tree for keys below the minimum, present, in a gap and above the maximum, and print the comparisons each one spent.
2. Check every integer from −1 to 1,001 on a random 200-key tree against a linear scan of the sorted keys that never touches the tree.
3. Compare the node-based successor with the descent for every key, then walk the whole tree by calling `succ` n times and look at the total cost against the height.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
)

type node struct {
	key         int
	left, right *node
}

type tree struct {
	root *node
	size int
}

func (t *tree) insert(key int) bool {
	link := &t.root
	for *link != nil {
		switch c := cmp.Compare(key, (*link).key); {
		case c < 0:
			link = &(*link).left
		case c > 0:
			link = &(*link).right
		default:
			return false
		}
	}
	*link = &node{key: key}
	t.size++
	return true
}

// Every query below is ONE descent from the root. Whenever the descent turns
// away from a node that still qualifies as an answer, it remembers that node;
// when it falls off the tree, the last one remembered is the answer. No parent
// pointers, no recursion, Theta(1) space, and k need not be in the tree.
// Each returns (key, ok) and the comparisons spent.

// floor: the largest key <= k.
func (t *tree) floor(k int) (key int, ok bool, cmps int) {
	for n := t.root; n != nil; {
		cmps++
		switch c := cmp.Compare(k, n.key); {
		case c == 0:
			return n.key, true, cmps // exact hit, nothing can be closer
		case c < 0:
			n = n.left // n is too big: the answer is in the left subtree, if any
		default:
			key, ok = n.key, true // n qualifies; maybe something bigger does too
			n = n.right
		}
	}
	return key, ok, cmps
}

// ceil: the smallest key >= k.
func (t *tree) ceil(k int) (key int, ok bool, cmps int) {
	for n := t.root; n != nil; {
		cmps++
		switch c := cmp.Compare(k, n.key); {
		case c == 0:
			return n.key, true, cmps
		case c > 0:
			n = n.right
		default:
			key, ok = n.key, true
			n = n.left
		}
	}
	return key, ok, cmps
}

// pred: the largest key < k. An equal key does not qualify, so keep going left.
func (t *tree) pred(k int) (key int, ok bool, cmps int) {
	for n := t.root; n != nil; {
		cmps++
		if cmp.Compare(k, n.key) <= 0 {
			n = n.left
		} else {
			key, ok = n.key, true
			n = n.right
		}
	}
	return key, ok, cmps
}

// succ: the smallest key > k.
func (t *tree) succ(k int) (key int, ok bool, cmps int) {
	for n := t.root; n != nil; {
		cmps++
		if cmp.Compare(k, n.key) >= 0 {
			n = n.right
		} else {
			key, ok = n.key, true
			n = n.left
		}
	}
	return key, ok, cmps
}

// min and max follow one link per step and compare no keys.
func (t *tree) min() (key, steps int) {
	n := t.root
	for ; n.left != nil; n = n.left {
		steps++
	}
	return n.key, steps
}

func (t *tree) max() (key, steps int) {
	n := t.root
	for ; n.right != nil; n = n.right {
		steps++
	}
	return n.key, steps
}

func height(n *node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.left), height(n.right))
}

func inorder(n *node, out *[]*node) {
	if n != nil {
		inorder(n.left, out)
		*out = append(*out, n)
		inorder(n.right, out)
	}
}

// succOfNode is the textbook successor of a NODE: the leftmost node of its right
// subtree, or, if there is no right subtree, the lowest ancestor whose LEFT
// subtree contains it. Without parent pointers that ancestor is found by a
// descent from the root that remembers the last left turn.
func (t *tree) succOfNode(x *node) (succ *node, viaRight bool) {
	if x.right != nil {
		n := x.right
		for n.left != nil {
			n = n.left
		}
		return n, true
	}
	for n := t.root; n != x; {
		if x.key < n.key {
			succ = n
			n = n.left
		} else {
			n = n.right
		}
	}
	return succ, false
}

type answer struct {
	key int
	ok  bool
}

func (a answer) String() string { return fmt.Sprintf("(%d, %v)", a.key, a.ok) }

// scan answers the same four questions with a linear scan of the sorted keys:
// the oracle does not use the tree at all.
func scan(sorted []int, k int) (fl, ce, pr, su answer) {
	for _, x := range sorted {
		if x <= k {
			fl = answer{x, true}
		}
		if x >= k && !ce.ok {
			ce = answer{x, true}
		}
		if x < k {
			pr = answer{x, true}
		}
		if x > k && !su.ok {
			su = answer{x, true}
		}
	}
	return
}

func main() {
	var t tree
	for _, k := range []int{50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25} {
		t.insert(k)
	}
	lo, loSteps := t.min()
	hi, hiSteps := t.max()
	fmt.Printf("(a) the tree of example 1 (height %d): min = %d after %d link steps, max = %d after %d\n",
		height(t.root), lo, loSteps, hi, hiSteps)
	fmt.Printf("  %4s  %-10s %-10s %-10s %-10s  %s\n", "k", "floor", "ceil", "pred", "succ", "comparisons f/c/p/s")
	for _, k := range []int{5, 10, 36, 50, 25, 62, 80, 99} {
		f, fok, fc := t.floor(k)
		c, cok, cc := t.ceil(k)
		p, pok, pc := t.pred(k)
		s, sok, sc := t.succ(k)
		fmt.Printf("  %4d  %-10v %-10v %-10v %-10v  %d/%d/%d/%d\n", k,
			answer{f, fok}, answer{c, cok}, answer{p, pok}, answer{s, sok}, fc, cc, pc, sc)
	}

	// (b) every integer query against the oracle, on a random tree.
	rng := rand.New(rand.NewPCG(1, 2))
	var r tree
	var sorted []int
	for r.size < 200 {
		k := rng.IntN(1000)
		if r.insert(k) {
			sorted = append(sorted, k)
		}
	}
	slices.Sort(sorted)
	queries, mismatches, total, worst := 0, 0, 0, 0
	for k := -1; k <= 1001; k++ {
		f, fok, fc := r.floor(k)
		c, cok, cc := r.ceil(k)
		p, pok, pc := r.pred(k)
		s, sok, sc := r.succ(k)
		wf, wc, wp, ws := scan(sorted, k)
		for _, pair := range [][2]answer{{{f, fok}, wf}, {{c, cok}, wc}, {{p, pok}, wp}, {{s, sok}, ws}} {
			if pair[0] != pair[1] {
				mismatches++
			}
		}
		queries++
		total += fc + cc + pc + sc
		worst = max(worst, fc, cc, pc, sc)
	}
	fmt.Printf("\n(b) random tree, %d keys in [0, 1000), height %d: every k in [-1, 1001]\n", r.size, height(r.root))
	fmt.Printf("  %d mismatches over %d queries (%d answers)\n", mismatches, queries, 4*queries)
	fmt.Printf("  comparisons per descent: mean %.2f, max %d, tree height %d\n",
		float64(total)/float64(4*queries), worst, height(r.root))

	// (c) the textbook node-based successor agrees for every key present.
	var nodes []*node
	inorder(r.root, &nodes)
	disagree, viaRight := 0, 0
	for i, x := range nodes {
		got, right := r.succOfNode(x)
		want, wok, _ := r.succ(x.key)
		if right {
			viaRight++
		}
		if (got == nil) != !wok || got != nil && got.key != want {
			disagree++
		}
		if i+1 < len(nodes) && got != nodes[i+1] {
			disagree++ // the next node in-order is the successor, as a node
		}
	}
	fmt.Printf("\n(c) successor of a node, %d keys: %d disagreements (%d answered from the right subtree, %d from an ancestor)\n",
		len(nodes), disagree, viaRight, len(nodes)-viaRight)

	// (d) walk the whole tree by calling succ again and again.
	calls, found, cmps := 0, 0, 0
	k, _ := r.min()
	for {
		next, ok, c := r.succ(k)
		calls++
		cmps += c
		if !ok {
			break
		}
		found++
		k = next
	}
	fmt.Printf("\n(d) Min, then Succ until none: %d calls (%d returned a key), %d comparisons in total,\n", calls, found, cmps)
	fmt.Printf("  mean %.2f per call, against a height of %d\n", float64(cmps)/float64(calls), height(r.root))
	fmt.Println("  an iterator keeps a stack instead and pays amortized Theta(1) per step (example 11); not measured here")
}
```

**Output:**

```
(a) the tree of example 1 (height 4): min = 10 after 3 link steps, max = 80 after 2
     k  floor      ceil       pred       succ        comparisons f/c/p/s
     5  (0, false) (10, true) (0, false) (10, true)  4/4/4/4
    10  (10, true) (10, true) (0, false) (20, true)  4/4/4/4
    36  (35, true) (40, true) (35, true) (40, true)  4/4/4/4
    50  (50, true) (50, true) (45, true) (60, true)  1/1/4/3
    25  (25, true) (25, true) (20, true) (30, true)  4/4/4/4
    62  (60, true) (65, true) (60, true) (65, true)  4/4/4/4
    80  (80, true) (80, true) (70, true) (0, false)  3/3/3/3
    99  (80, true) (0, false) (80, true) (0, false)  3/3/3/3

(b) random tree, 200 keys in [0, 1000), height 17: every k in [-1, 1001]
  0 mismatches over 1003 queries (4012 answers)
  comparisons per descent: mean 9.50, max 17, tree height 17

(c) successor of a node, 200 keys: 0 disagreements (104 answered from the right subtree, 96 from an ancestor)

(d) Min, then Succ until none: 200 calls (199 returned a key), 1898 comparisons in total,
  mean 9.49 per call, against a height of 17
  an iterator keeps a stack instead and pays amortized Theta(1) per step (example 11); not measured here
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::BTreeSet;
use std::ops::Bound::{Excluded, Unbounded};

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the keys the Go program draws (seed 1, 2).
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

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (see example 1): safe for any shape of tree.
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

#[derive(Default)]
struct Tree {
    root: Link,
    size: usize,
}

// Every query below is ONE descent from the root. Whenever the descent turns
// away from a node that still qualifies as an answer, it remembers that node;
// when it falls off the tree, the last one remembered is the answer. No parent
// pointers, no recursion, Theta(1) space, and k need not be in the tree. The
// candidate is an Option<i64> (Go returns (key, ok)) and the cursor is a shared
// borrow, so the whole descent is safe code that moves nothing.
// Each returns the answer and the comparisons spent.
impl Tree {
    fn insert(&mut self, key: i64) -> bool {
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            match key.cmp(&n.key) {
                Ordering::Less => cur = &mut n.left,
                Ordering::Greater => cur = &mut n.right,
                Ordering::Equal => return false,
            }
        }
        *cur = Some(Box::new(Node {
            key,
            left: None,
            right: None,
        }));
        self.size += 1;
        true
    }

    // floor: the largest key <= k.
    fn floor(&self, k: i64) -> (Option<i64>, u64) {
        let (mut best, mut cmps) = (None, 0);
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            cmps += 1;
            match k.cmp(&n.key) {
                Ordering::Equal => return (Some(n.key), cmps), // exact hit, nothing can be closer
                Ordering::Less => cur = n.left.as_deref(),     // n is too big: look left
                Ordering::Greater => {
                    best = Some(n.key); // n qualifies; maybe something bigger does too
                    cur = n.right.as_deref();
                }
            }
        }
        (best, cmps)
    }

    // ceil: the smallest key >= k.
    fn ceil(&self, k: i64) -> (Option<i64>, u64) {
        let (mut best, mut cmps) = (None, 0);
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            cmps += 1;
            match k.cmp(&n.key) {
                Ordering::Equal => return (Some(n.key), cmps),
                Ordering::Greater => cur = n.right.as_deref(),
                Ordering::Less => {
                    best = Some(n.key);
                    cur = n.left.as_deref();
                }
            }
        }
        (best, cmps)
    }

    // pred: the largest key < k. An equal key does not qualify, so keep going left.
    fn pred(&self, k: i64) -> (Option<i64>, u64) {
        let (mut best, mut cmps) = (None, 0);
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            cmps += 1;
            if k.cmp(&n.key) != Ordering::Greater {
                cur = n.left.as_deref();
            } else {
                best = Some(n.key);
                cur = n.right.as_deref();
            }
        }
        (best, cmps)
    }

    // succ: the smallest key > k.
    fn succ(&self, k: i64) -> (Option<i64>, u64) {
        let (mut best, mut cmps) = (None, 0);
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            cmps += 1;
            if k.cmp(&n.key) != Ordering::Less {
                cur = n.right.as_deref();
            } else {
                best = Some(n.key);
                cur = n.left.as_deref();
            }
        }
        (best, cmps)
    }

    // min and max follow one link per step and compare no keys.
    fn min(&self) -> (i64, u64) {
        let mut n = self.root.as_deref().expect("empty tree");
        let mut steps = 0;
        while let Some(l) = n.left.as_deref() {
            n = l;
            steps += 1;
        }
        (n.key, steps)
    }

    fn max(&self) -> (i64, u64) {
        let mut n = self.root.as_deref().expect("empty tree");
        let mut steps = 0;
        while let Some(r) = n.right.as_deref() {
            n = r;
            steps += 1;
        }
        (n.key, steps)
    }

    // succ_of_node is the textbook successor of a NODE: the leftmost node of its
    // right subtree, or, if there is no right subtree, the lowest ancestor whose
    // LEFT subtree contains it. Without parent pointers that ancestor is found by
    // a descent from the root that remembers the last left turn. (A parent link
    // would need Rc/Weak, an arena of indices or unsafe: BTreeMap uses unsafe.)
    fn succ_of_node<'a>(&'a self, x: &'a Node) -> (Option<&'a Node>, bool) {
        if let Some(r) = x.right.as_deref() {
            let mut n = r;
            while let Some(l) = n.left.as_deref() {
                n = l;
            }
            return (Some(n), true);
        }
        let (mut best, mut cur) = (None, self.root.as_deref());
        while let Some(n) = cur {
            if std::ptr::eq(n, x) {
                break;
            }
            if x.key < n.key {
                best = Some(n);
                cur = n.left.as_deref();
            } else {
                cur = n.right.as_deref();
            }
        }
        (best, false)
    }
}

fn height(n: &Link) -> usize {
    n.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

fn inorder<'a>(n: &'a Link, out: &mut Vec<&'a Node>) {
    if let Some(n) = n {
        inorder(&n.left, out);
        out.push(n);
        inorder(&n.right, out);
    }
}

// Go prints (key, ok); an Option prints the same way.
fn answer(a: Option<i64>) -> String {
    match a {
        Some(k) => format!("({k}, true)"),
        None => "(0, false)".to_string(),
    }
}

// scan answers the same four questions with a linear scan of the sorted keys:
// the oracle does not use the tree at all.
fn scan(sorted: &[i64], k: i64) -> [Option<i64>; 4] {
    let mut r = [None; 4];
    for &x in sorted {
        if x <= k {
            r[0] = Some(x);
        }
        if x >= k && r[1].is_none() {
            r[1] = Some(x);
        }
        if x < k {
            r[2] = Some(x);
        }
        if x > k && r[3].is_none() {
            r[3] = Some(x);
        }
    }
    r
}

// The four answers from a BTreeSet, all through range(). Unbounded on one side
// never panics; the panic is for an empty-looking range such as (5..3).
fn btree_four(s: &BTreeSet<i64>, k: i64) -> [Option<i64>; 4] {
    [
        s.range(..=k).next_back().copied(),
        s.range(k..).next().copied(),
        s.range(..k).next_back().copied(),
        s.range((Excluded(k), Unbounded)).next().copied(),
    ]
}

fn main() {
    let mut t = Tree::default();
    for k in [50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25] {
        t.insert(k);
    }
    let (lo, lo_steps) = t.min();
    let (hi, hi_steps) = t.max();
    println!(
        "(a) the tree of example 1 (height {}): min = {lo} after {lo_steps} link steps, max = {hi} after {hi_steps}",
        height(&t.root)
    );
    println!(
        "  {:>4}  {:<10} {:<10} {:<10} {:<10}  comparisons f/c/p/s",
        "k", "floor", "ceil", "pred", "succ"
    );
    for k in [5, 10, 36, 50, 25, 62, 80, 99] {
        let ((f, fc), (c, cc), (p, pc), (s, sc)) = (t.floor(k), t.ceil(k), t.pred(k), t.succ(k));
        println!(
            "  {k:4}  {:<10} {:<10} {:<10} {:<10}  {fc}/{cc}/{pc}/{sc}",
            answer(f),
            answer(c),
            answer(p),
            answer(s)
        );
    }

    // (b) every integer query against the oracle, on a random tree.
    let mut rng = Pcg::new(1, 2);
    let mut r = Tree::default();
    let mut sorted = Vec::new();
    while r.size < 200 {
        let k = rng.int_n(1000) as i64;
        if r.insert(k) {
            sorted.push(k);
        }
    }
    sorted.sort();
    let set: BTreeSet<i64> = sorted.iter().copied().collect();
    let (mut queries, mut mismatches, mut set_mismatches) = (0u64, 0, 0);
    let (mut total, mut worst) = (0u64, 0u64);
    for k in -1..=1001 {
        let ((f, fc), (c, cc), (p, pc), (s, sc)) = (r.floor(k), r.ceil(k), r.pred(k), r.succ(k));
        let want = scan(&sorted, k);
        mismatches += [f, c, p, s]
            .iter()
            .zip(&want)
            .filter(|(got, want)| got != want)
            .count();
        set_mismatches += btree_four(&set, k)
            .iter()
            .zip(&want)
            .filter(|(got, want)| got != want)
            .count();
        queries += 1;
        total += fc + cc + pc + sc;
        worst = worst.max(fc).max(cc).max(pc).max(sc);
    }
    println!(
        "\n(b) random tree, {} keys in [0, 1000), height {}: every k in [-1, 1001]",
        r.size,
        height(&r.root)
    );
    println!(
        "  {mismatches} mismatches over {queries} queries ({} answers)",
        4 * queries
    );
    println!(
        "  comparisons per descent: mean {:.2}, max {worst}, tree height {}",
        total as f64 / (4 * queries) as f64,
        height(&r.root)
    );

    // (c) the textbook node-based successor agrees for every key present.
    let mut nodes = Vec::new();
    inorder(&r.root, &mut nodes);
    let (mut disagree, mut via_right) = (0, 0);
    for (i, &x) in nodes.iter().enumerate() {
        let (got, right) = r.succ_of_node(x);
        let (want, _) = r.succ(x.key);
        via_right += usize::from(right);
        if got.map(|g| g.key) != want {
            disagree += 1;
        }
        // the next node in-order is the successor, as a node
        if i + 1 < nodes.len() && !got.is_some_and(|g| std::ptr::eq(g, nodes[i + 1])) {
            disagree += 1;
        }
    }
    println!(
        "\n(c) successor of a node, {} keys: {disagree} disagreements ({via_right} answered from the right subtree, {} from an ancestor)",
        nodes.len(),
        nodes.len() - via_right
    );

    // (d) walk the whole tree by calling succ again and again.
    let (mut calls, mut found, mut cmps) = (0, 0, 0);
    let mut k = r.min().0;
    loop {
        let (next, c) = r.succ(k);
        calls += 1;
        cmps += c;
        let Some(next) = next else { break };
        found += 1;
        k = next;
    }
    println!(
        "\n(d) Min, then Succ until none: {calls} calls ({found} returned a key), {cmps} comparisons in total,"
    );
    println!(
        "  mean {:.2} per call, against a height of {}",
        cmps as f64 / f64::from(calls),
        height(&r.root)
    );
    println!(
        "  an iterator keeps a stack instead and pays amortized Theta(1) per step (example 11); not measured here"
    );

    // (e) Rust only: std's ordered set answers all four with range().
    println!("\n(e) BTreeSet<i64> over the same 200 keys, the same {queries} queries:");
    println!(
        "  {set_mismatches} mismatches (floor range(..=k).next_back(), ceil range(k..).next(),"
    );
    println!("  pred range(..k).next_back(), succ range((Excluded(k), Unbounded)).next())");
    let evens: BTreeSet<i64> = (0..30).step_by(2).collect();
    let [f, c, p, s] = btree_four(&evens, 7);
    println!(
        "  even keys 0..28, k = 7: floor {}, ceil {}, pred {}, succ {}",
        f.unwrap(),
        c.unwrap(),
        p.unwrap(),
        s.unwrap()
    );
}
```

**Rust output:**

```
(a) the tree of example 1 (height 4): min = 10 after 3 link steps, max = 80 after 2
     k  floor      ceil       pred       succ        comparisons f/c/p/s
     5  (0, false) (10, true) (0, false) (10, true)  4/4/4/4
    10  (10, true) (10, true) (0, false) (20, true)  4/4/4/4
    36  (35, true) (40, true) (35, true) (40, true)  4/4/4/4
    50  (50, true) (50, true) (45, true) (60, true)  1/1/4/3
    25  (25, true) (25, true) (20, true) (30, true)  4/4/4/4
    62  (60, true) (65, true) (60, true) (65, true)  4/4/4/4
    80  (80, true) (80, true) (70, true) (0, false)  3/3/3/3
    99  (80, true) (0, false) (80, true) (0, false)  3/3/3/3

(b) random tree, 200 keys in [0, 1000), height 17: every k in [-1, 1001]
  0 mismatches over 1003 queries (4012 answers)
  comparisons per descent: mean 9.50, max 17, tree height 17

(c) successor of a node, 200 keys: 0 disagreements (104 answered from the right subtree, 96 from an ancestor)

(d) Min, then Succ until none: 200 calls (199 returned a key), 1898 comparisons in total,
  mean 9.49 per call, against a height of 17
  an iterator keeps a stack instead and pays amortized Theta(1) per step (example 11); not measured here

(e) BTreeSet<i64> over the same 200 keys, the same 1003 queries:
  0 mismatches (floor range(..=k).next_back(), ceil range(k..).next(),
  pred range(..k).next_back(), succ range((Excluded(k), Unbounded)).next())
  even keys 0..28, k = 7: floor 6, ceil 8, pred 6, succ 8
```

> **Go vs Rust:** Sections (a) to (d) print the same text as Go, comparison counts included (diffed); (e) is Rust-only. The descents keep the candidate as an `Option<i64>` where Go returns `(key, ok)` (printed the same way here, `(0, false)` for none), and `BTreeSet` answers all four questions through `range(..)` with 0 mismatches on the same 1,003 queries. The Rust twin navigates from the root because a parent link would need `Rc`/`Weak`, an arena of indices or `unsafe`; `BTreeMap` takes the `unsafe` route (its nodes hold an `Option<NonNull<..>>` parent pointer, read in `node.rs`, not run). `range` panics on an empty-looking range such as `5..3`, but never with `Unbounded` on one side, which is all these four use.

**Complexity:** each query Θ(h) comparisons and Θ(1) space, whether or not the key is present (here at most 17 = the height, mean 9.50 per descent on the 200-key random tree; 0 mismatches against the scan over 1,003 queries) · min/max Θ(h) link steps with no key comparison · n successor calls from the root cost Θ(n·h): 1,898 comparisons for 200 keys, 9.49 per call, against a height of 17 · the node-based successor agreed with the descent for all 200 keys.

---

## 5. Delete, case by case

`🟢 easy` · *Delete: three cases*

Delete finds the node and then splits three ways: no children, so clear the link; one child, so point the link at the child; two children, so the node cannot vanish and its in-order **successor** (the leftmost node of its right subtree) takes its place — and that successor has no left child, so detaching it is the first or second case. The in-order sequence loses exactly one key and stays ascending. The cost is Θ(h): one descent to the node, plus a walk down the left spine that compares no keys. This version is recursive and returns the new root of each subtree; it moves the successor's *key* up and detaches the successor node (copy-versus-relink, and the loop form, are example 12).

**Steps:**

1. Build an 11-node tree and delete, in order, a leaf, a node with only a right child, one with only a left child, a two-child node whose successor is its right child, one whose successor is deeper, the root, and a key that is absent; print the tree, the in-order sequence, the case and the comparisons after each.
2. Run 2,000 random inserts and deletes on keys 0..39 against a sorted slice, with a checker (bounds invariant plus strictly ascending in-order) after every operation, and count which cases the deletes hit.
3. Replay the same operations with a delete that mishandles one-child nodes, and see which test catches it and which one does not.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

type node struct {
	key         int
	left, right *node
}

// insert returns the new root of the subtree and whether it added the key.
func insert(n *node, key int) (*node, bool) {
	if n == nil {
		return &node{key: key}, true
	}
	var ok bool
	switch c := cmp.Compare(key, n.key); {
	case c < 0:
		n.left, ok = insert(n.left, key)
	case c > 0:
		n.right, ok = insert(n.right, key)
	}
	return n, ok
}

// stats says what one delete did.
type stats struct {
	found    bool
	kind     string
	cmps     int // key comparisons on the way down
	minSteps int // left links followed to reach the successor (no comparisons)
}

// takeMin detaches the leftmost node of the subtree n and returns its key and
// the subtree that is left. The leftmost node has no left child, so detaching it
// is the leaf case or the one-child case: its right subtree takes its place.
func takeMin(n *node, s *stats) (int, *node) {
	if n.left == nil {
		return n.key, n.right
	}
	s.minSteps++
	k, rest := takeMin(n.left, s)
	n.left = rest
	return k, n
}

// remove deletes key from the subtree rooted at n and returns that subtree's new
// root: the caller stores the result back into the link it came from.
func remove(n *node, key int, s *stats) *node {
	if n == nil {
		return nil // fell off the tree: the key is absent
	}
	s.cmps++
	switch c := cmp.Compare(key, n.key); {
	case c < 0:
		n.left = remove(n.left, key, s)
	case c > 0:
		n.right = remove(n.right, key, s)
	default: // found it: three cases
		s.found = true
		switch {
		case n.left == nil && n.right == nil:
			s.kind = "leaf"
			return nil // 1. no children: clear the link
		case n.left == nil:
			s.kind = "only a right child"
			return n.right // 2. one child: the link points at the child
		case n.right == nil:
			s.kind = "only a left child"
			return n.left
		}
		// 3. two children: the node cannot vanish. Its in-order successor (the
		// leftmost node of the right subtree) has no left child; move that key up
		// and detach the successor from the right subtree.
		n.key, n.right = takeMin(n.right, s)
		s.kind = "two children, successor is its right child"
		if s.minSteps > 0 {
			s.kind = "two children, successor is deeper"
		}
	}
	return n
}

// removeBuggy forgets that a node can have ONE child: for "one child" it does
// what the leaf case does and clears the link, so the whole subtree goes too.
func removeBuggy(n *node, key int) *node {
	if n == nil {
		return nil
	}
	switch {
	case key < n.key:
		n.left = removeBuggy(n.left, key)
	case key > n.key:
		n.right = removeBuggy(n.right, key)
	case n.left == nil || n.right == nil:
		return nil // BUG
	default:
		n.key, n.right = takeMin(n.right, new(stats))
	}
	return n
}

func show(n *node, depth int) {
	if n != nil {
		show(n.right, depth+1)
		fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.key)
		show(n.left, depth+1)
	}
}

func keys(n *node, out *[]int) { // in-order
	if n != nil {
		keys(n.left, out)
		*out = append(*out, n.key)
		keys(n.right, out)
	}
}

func inorder(n *node) []int {
	var xs []int
	keys(n, &xs)
	return xs
}

// bounds checks the invariant with an open interval (lo, hi) per subtree.
func bounds(n *node, lo, hi *int) string {
	if n == nil {
		return ""
	}
	if lo != nil && n.key <= *lo || hi != nil && n.key >= *hi {
		return fmt.Sprintf("key %d is outside its subtree's bounds", n.key)
	}
	if msg := bounds(n.left, lo, &n.key); msg != "" {
		return msg
	}
	return bounds(n.right, &n.key, hi)
}

// check: the bounds invariant AND a strictly ascending in-order sequence.
func check(root *node) string {
	if msg := bounds(root, nil, nil); msg != "" {
		return msg
	}
	xs := inorder(root)
	for i := 1; i < len(xs); i++ {
		if xs[i-1] >= xs[i] {
			return fmt.Sprintf("in-order not strictly ascending: %d then %d", xs[i-1], xs[i])
		}
	}
	return ""
}

func main() {
	var root *node
	order := []int{50, 30, 70, 20, 40, 60, 80, 35, 65, 75, 85}
	for _, k := range order {
		root, _ = insert(root, k)
	}
	fmt.Println("(a) an 11-node tree, built by inserting", order)
	show(root, 0)
	fmt.Println("in-order", inorder(root))
	for _, key := range []int{20, 60, 40, 80, 70, 50, 99} {
		var s stats
		root = remove(root, key, &s)
		if !s.found {
			s.kind = "absent, the tree is unchanged"
		}
		xs := inorder(root)
		fmt.Printf("\ndelete %d: %s (comparisons %d, successor steps %d)\n", key, s.kind, s.cmps, s.minSteps)
		show(root, 0)
		fmt.Printf("in-order %v (%d keys, strictly ascending: %v)\n", xs, len(xs), check(root) == "")
	}

	// (b) 2,000 random operations against a sorted slice, checking the tree after every one.
	rng := rand.New(rand.NewPCG(1, 2))
	var t *node
	var oracle []int
	mismatches, failures := 0, 0
	kinds := map[string]int{}
	for op := 0; op < 2000; op++ {
		insertOp, key := rng.IntN(2) == 0, rng.IntN(40)
		i, present := slices.BinarySearch(oracle, key)
		var got bool
		if insertOp {
			t, got = insert(t, key)
			if !present {
				oracle = slices.Insert(oracle, i, key)
			}
			if got == present { // inserted exactly when it was absent
				mismatches++
			}
		} else {
			var s stats
			t = remove(t, key, &s)
			if !s.found {
				s.kind = "absent"
			}
			kinds[s.kind]++
			if present {
				oracle = slices.Delete(oracle, i, i+1)
			}
			if s.found != present {
				mismatches++
			}
		}
		if !slices.Equal(inorder(t), oracle) {
			mismatches++
		}
		if check(t) != "" {
			failures++
		}
	}
	fmt.Printf("\n(b) %d mismatches, %d invariant failures over 2000 random operations on keys 0..39\n", mismatches, failures)
	fmt.Printf("  deletes by case: leaf %d, only a right child %d, only a left child %d,\n", kinds["leaf"], kinds["only a right child"], kinds["only a left child"])
	fmt.Printf("  two children with the successor as right child %d, deeper %d; absent key %d\n",
		kinds["two children, successor is its right child"], kinds["two children, successor is deeper"], kinds["absent"])

	// (c) the same operations against a delete that mishandles one child.
	rng = rand.New(rand.NewPCG(1, 2))
	t, oracle = nil, nil
	for op := 0; op < 2000; op++ {
		insertOp, key := rng.IntN(2) == 0, rng.IntN(40)
		i, present := slices.BinarySearch(oracle, key)
		if insertOp {
			t, _ = insert(t, key)
			if !present {
				oracle = slices.Insert(oracle, i, key)
			}
			continue
		}
		t = removeBuggy(t, key)
		if present {
			oracle = slices.Delete(oracle, i, i+1)
		}
		if got := inorder(t); !slices.Equal(got, oracle) {
			fmt.Printf("\n(c) the buggy delete is caught at operation %d, delete(%d):\n", op+1, key)
			fmt.Println("  the tree's in-order ", got)
			fmt.Println("  the oracle's        ", oracle)
			fmt.Printf("  the invariant checker passes the damaged tree: %v (it is still a valid BST)\n", check(t) == "")
			break
		}
	}
}
```

**Output:**

```
(a) an 11-node tree, built by inserting [50 30 70 20 40 60 80 35 65 75 85]
            85
        80
            75
    70
            65
        60
50
        40
            35
    30
        20
in-order [20 30 35 40 50 60 65 70 75 80 85]

delete 20: leaf (comparisons 3, successor steps 0)
            85
        80
            75
    70
            65
        60
50
        40
            35
    30
in-order [30 35 40 50 60 65 70 75 80 85] (10 keys, strictly ascending: true)

delete 60: only a right child (comparisons 3, successor steps 0)
            85
        80
            75
    70
        65
50
        40
            35
    30
in-order [30 35 40 50 65 70 75 80 85] (9 keys, strictly ascending: true)

delete 40: only a left child (comparisons 3, successor steps 0)
            85
        80
            75
    70
        65
50
        35
    30
in-order [30 35 50 65 70 75 80 85] (8 keys, strictly ascending: true)

delete 80: two children, successor is its right child (comparisons 3, successor steps 0)
        85
            75
    70
        65
50
        35
    30
in-order [30 35 50 65 70 75 85] (7 keys, strictly ascending: true)

delete 70: two children, successor is deeper (comparisons 2, successor steps 1)
        85
    75
        65
50
        35
    30
in-order [30 35 50 65 75 85] (6 keys, strictly ascending: true)

delete 50: two children, successor is deeper (comparisons 1, successor steps 1)
        85
    75
65
        35
    30
in-order [30 35 65 75 85] (5 keys, strictly ascending: true)

delete 99: absent, the tree is unchanged (comparisons 3, successor steps 0)
        85
    75
65
        35
    30
in-order [30 35 65 75 85] (5 keys, strictly ascending: true)

(b) 0 mismatches, 0 invariant failures over 2000 random operations on keys 0..39
  deletes by case: leaf 169, only a right child 82, only a left child 80,
  two children with the successor as right child 88, deeper 69; absent key 516

(c) the buggy delete is caught at operation 30, delete(18):
  the tree's in-order  [0 1 24 28 31 32 35 36 37]
  the oracle's         [0 1 2 24 28 31 32 35 36 37]
  the invariant checker passes the damaged tree: true (it is still a valid BST)
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::{BTreeSet, HashMap};

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the operations the Go program draws (seed 1, 2).
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

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (see example 1): safe for any shape of tree.
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

fn leaf(key: i64) -> Box<Node> {
    Box::new(Node {
        key,
        left: None,
        right: None,
    })
}

// insert reports whether it added the key.
fn insert(link: &mut Link, key: i64) -> bool {
    match link {
        None => {
            *link = Some(leaf(key));
            true
        }
        Some(n) => match key.cmp(&n.key) {
            Ordering::Less => insert(&mut n.left, key),
            Ordering::Greater => insert(&mut n.right, key),
            Ordering::Equal => false,
        },
    }
}

// Stats says what one delete did.
#[derive(Default)]
struct Stats {
    found: bool,
    kind: &'static str,
    cmps: u64,      // key comparisons on the way down
    min_steps: u64, // left links followed to reach the successor (no comparisons)
}

// take_min detaches the leftmost node of the subtree `n` and returns its key and
// the subtree that is left. It OWNS the subtree (a Box<Node>) and hands the
// remainder back, so there is no borrow to fight: take() moves a child out of
// its Option, leaving None behind. The leftmost node has no left child, so
// detaching it is the leaf case or the one-child case.
fn take_min(mut n: Box<Node>, s: &mut Stats) -> (i64, Link) {
    match n.left.take() {
        None => (n.key, n.right.take()),
        Some(l) => {
            s.min_steps += 1;
            let (k, rest) = take_min(l, s);
            n.left = rest;
            (k, Some(n))
        }
    }
}

// remove deletes key from the subtree `link` and returns that subtree's new
// root: the caller stores the result back into the link it came from. The
// subtree is passed BY VALUE, `n.left = remove(n.left.take(), ..)`: take() hands
// it to the recursive call and the result is written back, so the call never
// holds a reference into the tree it is rebuilding.
fn remove(link: Link, key: i64, s: &mut Stats) -> Link {
    let mut n = link?; // None: fell off the tree, the key is absent
    s.cmps += 1;
    match key.cmp(&n.key) {
        Ordering::Less => n.left = remove(n.left.take(), key, s),
        Ordering::Greater => n.right = remove(n.right.take(), key, s),
        Ordering::Equal => {
            // found it: three cases
            s.found = true;
            match (n.left.take(), n.right.take()) {
                (None, None) => {
                    s.kind = "leaf";
                    return None; // 1. no children: clear the link
                }
                (None, right) => {
                    s.kind = "only a right child";
                    return right; // 2. one child: the link points at the child
                }
                (left, None) => {
                    s.kind = "only a left child";
                    return left;
                }
                (left, Some(right)) => {
                    // 3. two children: the node cannot vanish. Its in-order
                    // successor (leftmost of the right subtree) has no left
                    // child; move that key up and detach the successor.
                    let (k, rest) = take_min(right, s);
                    (n.key, n.left, n.right) = (k, left, rest);
                    s.kind = if s.min_steps > 0 {
                        "two children, successor is deeper"
                    } else {
                        "two children, successor is its right child"
                    };
                }
            }
        }
    }
    Some(n)
}

// remove_buggy forgets that a node can have ONE child: for "one child" it does
// what the leaf case does and clears the link, so the whole subtree goes too.
fn remove_buggy(link: Link, key: i64) -> Link {
    let mut n = link?;
    match key.cmp(&n.key) {
        Ordering::Less => n.left = remove_buggy(n.left.take(), key),
        Ordering::Greater => n.right = remove_buggy(n.right.take(), key),
        Ordering::Equal => match n.right.take() {
            Some(right) if n.left.is_some() => {
                (n.key, n.right) = take_min(right, &mut Stats::default());
            }
            _ => return None, // BUG: a one-child node takes its child's subtree with it
        },
    }
    Some(n)
}

// remove_cursor is delete WITHOUT recursion: `cur` is a `&mut Option<Box<Node>>`
// aimed at a link (the root, or some node's left/right field), so root, middle
// and leaf are one case. The loop shape that looks natural does not compile
// (see the note under this program); this one tests with a short-lived shared
// borrow, then reborrows. It relinks the successor NODE instead of copying its
// key, and builds the same tree as `remove`.
fn remove_cursor(root: &mut Link, key: i64) -> bool {
    let mut cur = root;
    while cur.as_ref().is_some_and(|n| n.key != key) {
        let n = cur.as_mut().unwrap(); // a fresh reborrow, then move the cursor
        cur = if key < n.key {
            &mut n.left
        } else {
            &mut n.right
        };
    }
    let Some(mut node) = cur.take() else {
        return false;
    };
    *cur = match (node.left.take(), node.right.take()) {
        (None, right) => right,
        (left, None) => left,
        (left, Some(right)) => {
            let mut right = Some(right);
            let mut succ = &mut right;
            while succ.as_ref().unwrap().left.is_some() {
                succ = &mut succ.as_mut().unwrap().left;
            }
            let mut s = succ.take().unwrap();
            *succ = s.right.take(); // detach the successor: its right child moves up
            s.left = left;
            s.right = right;
            Some(s)
        }
    };
    true
}

fn show(n: &Link, depth: usize) {
    if let Some(n) = n {
        show(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.key);
        show(&n.left, depth + 1);
    }
}

fn inorder(n: &Link) -> Vec<i64> {
    fn walk(n: &Link, out: &mut Vec<i64>) {
        if let Some(n) = n {
            walk(&n.left, out);
            out.push(n.key);
            walk(&n.right, out);
        }
    }
    let mut out = Vec::new();
    walk(n, &mut out);
    out
}

// shape writes the tree with its keys, nil as ".": equal trees, equal strings.
fn shape(n: &Link) -> String {
    match n {
        None => ".".to_string(),
        Some(n) => format!("({} {} {})", shape(&n.left), n.key, shape(&n.right)),
    }
}

// bounds checks the invariant with an open interval (lo, hi) per subtree.
// Option<i64> is "unbounded or a bound": no sentinel value to collide with a key.
fn bounds(n: &Link, lo: Option<i64>, hi: Option<i64>) -> Result<(), String> {
    let Some(n) = n else { return Ok(()) };
    if lo.is_some_and(|l| n.key <= l) || hi.is_some_and(|h| n.key >= h) {
        return Err(format!("key {} is outside its subtree's bounds", n.key));
    }
    bounds(&n.left, lo, Some(n.key))?;
    bounds(&n.right, Some(n.key), hi)
}

// check: the bounds invariant AND a strictly ascending in-order sequence.
fn check(root: &Link) -> Result<(), String> {
    bounds(root, None, None)?;
    let xs = inorder(root);
    match xs.windows(2).find(|w| w[0] >= w[1]) {
        Some(w) => Err(format!(
            "in-order not strictly ascending: {} then {}",
            w[0], w[1]
        )),
        None => Ok(()),
    }
}

// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    let mut root: Link = None;
    let order = [50, 30, 70, 20, 40, 60, 80, 35, 65, 75, 85];
    for k in order {
        insert(&mut root, k);
    }
    println!(
        "(a) an 11-node tree, built by inserting {}",
        go_slice(&order)
    );
    show(&root, 0);
    println!("in-order {}", go_slice(&inorder(&root)));
    for key in [20, 60, 40, 80, 70, 50, 99] {
        let mut s = Stats::default();
        root = remove(root.take(), key, &mut s);
        if !s.found {
            s.kind = "absent, the tree is unchanged";
        }
        let xs = inorder(&root);
        println!(
            "\ndelete {key}: {} (comparisons {}, successor steps {})",
            s.kind, s.cmps, s.min_steps
        );
        show(&root, 0);
        println!(
            "in-order {} ({} keys, strictly ascending: {})",
            go_slice(&xs),
            xs.len(),
            check(&root).is_ok()
        );
    }

    // (b) 2,000 random operations against a sorted Vec, checking the tree after every one.
    let mut rng = Pcg::new(1, 2);
    let mut t: Link = None;
    let mut oracle: Vec<i64> = Vec::new();
    let mut set = BTreeSet::new(); // a second oracle: std's ordered set
    let (mut mismatches, mut failures, mut set_mismatches) = (0, 0, 0);
    let mut kinds: HashMap<&str, usize> = HashMap::new();
    for _ in 0..2000 {
        let insert_op = rng.int_n(2) == 0;
        let key = rng.int_n(40) as i64;
        let (i, present) = match oracle.binary_search(&key) {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        if insert_op {
            let got = insert(&mut t, key);
            if !present {
                oracle.insert(i, key);
            }
            mismatches += usize::from(got == present); // inserted exactly when it was absent
            set_mismatches += usize::from(set.insert(key) == present);
        } else {
            let mut s = Stats::default();
            t = remove(t.take(), key, &mut s);
            *kinds
                .entry(if s.found { s.kind } else { "absent" })
                .or_default() += 1;
            if present {
                oracle.remove(i);
            }
            mismatches += usize::from(s.found != present);
            set_mismatches += usize::from(set.remove(&key) != present);
        }
        mismatches += usize::from(inorder(&t) != oracle);
        set_mismatches += usize::from(!inorder(&t).iter().eq(set.iter()));
        failures += usize::from(check(&t).is_err());
    }
    println!(
        "\n(b) {mismatches} mismatches, {failures} invariant failures over 2000 random operations on keys 0..39"
    );
    let n = |k: &str| kinds.get(k).copied().unwrap_or(0);
    println!(
        "  deletes by case: leaf {}, only a right child {}, only a left child {},",
        n("leaf"),
        n("only a right child"),
        n("only a left child")
    );
    println!(
        "  two children with the successor as right child {}, deeper {}; absent key {}",
        n("two children, successor is its right child"),
        n("two children, successor is deeper"),
        n("absent")
    );
    println!("  BTreeSet as a second oracle: {set_mismatches} mismatches");

    // (c) the same operations against a delete that mishandles one child.
    let mut rng = Pcg::new(1, 2);
    let mut t: Link = None;
    let mut oracle: Vec<i64> = Vec::new();
    for op in 0..2000 {
        let insert_op = rng.int_n(2) == 0;
        let key = rng.int_n(40) as i64;
        let (i, present) = match oracle.binary_search(&key) {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        if insert_op {
            insert(&mut t, key);
            if !present {
                oracle.insert(i, key);
            }
            continue;
        }
        t = remove_buggy(t.take(), key);
        if present {
            oracle.remove(i);
        }
        let got = inorder(&t);
        if got != oracle {
            println!(
                "\n(c) the buggy delete is caught at operation {}, delete({key}):",
                op + 1
            );
            println!("  the tree's in-order  {}", go_slice(&got));
            println!("  the oracle's         {}", go_slice(&oracle));
            println!(
                "  the invariant checker passes the damaged tree: {} (it is still a valid BST)",
                check(&t).is_ok()
            );
            break;
        }
    }

    // (d) Rust only: the iterative cursor delete builds the very same trees.
    let mut rng = Pcg::new(1, 2);
    let (mut a, mut b): (Link, Link) = (None, None);
    let (mut shape_diffs, mut bad_found, mut deletes) = (0, 0, 0);
    for _ in 0..2000 {
        let insert_op = rng.int_n(2) == 0;
        let key = rng.int_n(40) as i64;
        if insert_op {
            insert(&mut a, key);
            insert(&mut b, key);
        } else {
            let mut s = Stats::default();
            a = remove(a.take(), key, &mut s);
            bad_found += usize::from(remove_cursor(&mut b, key) != s.found);
            deletes += 1;
        }
        shape_diffs += usize::from(shape(&a) != shape(&b));
    }
    println!(
        "\n(d) remove_cursor against remove, same {deletes} deletes: {shape_diffs} shape differences after any operation, {bad_found} disagreements on found"
    );
}
```

**Rust output:**

```
(a) an 11-node tree, built by inserting [50 30 70 20 40 60 80 35 65 75 85]
            85
        80
            75
    70
            65
        60
50
        40
            35
    30
        20
in-order [20 30 35 40 50 60 65 70 75 80 85]

delete 20: leaf (comparisons 3, successor steps 0)
            85
        80
            75
    70
            65
        60
50
        40
            35
    30
in-order [30 35 40 50 60 65 70 75 80 85] (10 keys, strictly ascending: true)

delete 60: only a right child (comparisons 3, successor steps 0)
            85
        80
            75
    70
        65
50
        40
            35
    30
in-order [30 35 40 50 65 70 75 80 85] (9 keys, strictly ascending: true)

delete 40: only a left child (comparisons 3, successor steps 0)
            85
        80
            75
    70
        65
50
        35
    30
in-order [30 35 50 65 70 75 80 85] (8 keys, strictly ascending: true)

delete 80: two children, successor is its right child (comparisons 3, successor steps 0)
        85
            75
    70
        65
50
        35
    30
in-order [30 35 50 65 70 75 85] (7 keys, strictly ascending: true)

delete 70: two children, successor is deeper (comparisons 2, successor steps 1)
        85
    75
        65
50
        35
    30
in-order [30 35 50 65 75 85] (6 keys, strictly ascending: true)

delete 50: two children, successor is deeper (comparisons 1, successor steps 1)
        85
    75
65
        35
    30
in-order [30 35 65 75 85] (5 keys, strictly ascending: true)

delete 99: absent, the tree is unchanged (comparisons 3, successor steps 0)
        85
    75
65
        35
    30
in-order [30 35 65 75 85] (5 keys, strictly ascending: true)

(b) 0 mismatches, 0 invariant failures over 2000 random operations on keys 0..39
  deletes by case: leaf 169, only a right child 82, only a left child 80,
  two children with the successor as right child 88, deeper 69; absent key 516
  BTreeSet as a second oracle: 0 mismatches

(c) the buggy delete is caught at operation 30, delete(18):
  the tree's in-order  [0 1 24 28 31 32 35 36 37]
  the oracle's         [0 1 2 24 28 31 32 35 36 37]
  the invariant checker passes the damaged tree: true (it is still a valid BST)

(d) remove_cursor against remove, same 1004 deletes: 0 shape differences after any operation, 0 disagreements on found
```

> **Go vs Rust:** Same output as Go in (a) to (c) (diffed); (b) adds a `BTreeSet` oracle and (d) is Rust-only. The recursive delete passes each subtree **by value** — `n.left = remove(n.left.take(), key, s)`: `take()` moves a child out of its `Option`, the call returns the rebuilt subtree, and nothing is borrowed while it works. The iterative cursor in (d), `&mut Option<Box<Node>>` (Go's `link := &t.root`), compiles and builds exactly the same trees, but the shape that looks natural does not (verified on rustc 1.98.1):
> 
> ```rust
> // does not compile
> let mut cur = root;
> while let Some(node) = cur {
>     match key.cmp(&node.key) {
>         Ordering::Less => cur = &mut node.left,
>         Ordering::Greater => cur = &mut node.right,
>         Ordering::Equal => { *cur = node.left.take(); return true; }
>     }
> }
> ```
> 
> `error[E0506]: cannot assign to *cur because it is borrowed` — the `while let` pattern borrows `*cur` and the borrow is still used when `*cur` is assigned. The working form in (d) tests with a short-lived shared borrow (`cur.as_ref().is_some_and(..)`) and reborrows afterwards; example 12 takes the loop apart.

**Complexity:** delete Θ(h): one descent of comparisons to the node, plus a left-spine walk of up to h steps that compares no keys (1 step in the two deeper-successor deletes above, 0 in the others) · in-order loses exactly one key and stays strictly ascending · three cases, five sub-cases here, and the random run hit all of them (leaf 169, only a right child 82, only a left child 80, two children with the successor as right child 88, deeper 69, absent 516) with 0 mismatches and 0 invariant failures · the planted one-child bug leaves a **valid** BST, so the invariant checker passes it and only the oracle comparison catches it (operation 30).

---

## 6. Sorted input makes a linked list

`🟢 easy` · *The failure mode*

The BST invariant says nothing about shape. Feed the tree keys in order and every node gets exactly one child: the tree is a linked list of height n, the i-th insert costs i comparisons, so building it costs n(n−1)/2 and looking every key up once costs n(n+1)/2. Reversed input and the zig-zag `lo, hi, lo+1, hi−1, …` do the same, and these inputs are not exotic — timestamps, auto-increment IDs, a file read in order. A randomly ordered input is the opposite, with a mean search cost that Knuth's formula `2(1 + 1/n)H(n) − 3` predicts. Search and insert are Θ(h), and h is the whole question; all trees here are built and searched iteratively, and none has more than 4,000 nodes.

**Steps:**

1. For n = 500 to 4,000 and four input orders (sorted, reversed, zig-zag, shuffled with a hand-written Fisher–Yates), build the tree and print its height, the build comparisons and the comparisons to find every key once, next to `n(n−1)/2` and `n(n+1)/2`.
2. Print the ratio of build comparisons as n doubles: about 4× for the degenerate orders, a little over 2× for the shuffled one.
3. For random order, build 100 trees per n and compare the measured mean comparisons per hit with Knuth's formula; finish with the sorted and random cost per search at n = 4,000.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
)

type node struct {
	key         int
	left, right *node
}

type tree struct {
	root *node
	size int
}

// insert and search are ITERATIVE: on a degenerate tree a recursive version
// would recurse n deep. Each returns the comparisons it spent.
func (t *tree) insert(key int) (inserted bool, cmps int) {
	link := &t.root
	for *link != nil {
		cmps++
		switch c := cmp.Compare(key, (*link).key); {
		case c < 0:
			link = &(*link).left
		case c > 0:
			link = &(*link).right
		default:
			return false, cmps
		}
	}
	*link = &node{key: key}
	t.size++
	return true, cmps
}

func (t *tree) search(key int) (found bool, cmps int) {
	for n := t.root; n != nil; {
		cmps++
		switch c := cmp.Compare(key, n.key); {
		case c < 0:
			n = n.left
		case c > 0:
			n = n.right
		default:
			return true, cmps
		}
	}
	return false, cmps
}

// height by level order: count the levels, no recursion.
func (t *tree) height() int {
	level, h := []*node{}, 0
	if t.root != nil {
		level = append(level, t.root)
	}
	for len(level) > 0 {
		h++
		var next []*node
		for _, n := range level {
			if n.left != nil {
				next = append(next, n.left)
			}
			if n.right != nil {
				next = append(next, n.right)
			}
		}
		level = next
	}
	return h
}

// The four input orders of the keys 0..n-1.
func sorted(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

func reversed(n int) []int {
	xs := sorted(n)
	for i, j := 0, n-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
	}
	return xs
}

// zigzag: lo, hi, lo+1, hi-1, ... the tree alternates right, left, right, ...
func zigzag(n int) []int {
	var xs []int
	for lo, hi := 0, n-1; lo <= hi; {
		xs = append(xs, lo)
		lo++
		if lo <= hi {
			xs = append(xs, hi)
			hi--
		}
	}
	return xs
}

// shuffled: Fisher-Yates by hand, so the Rust twin shuffles identically.
func shuffled(n int, rng *rand.Rand) []int {
	xs := sorted(n)
	for i := n - 1; i > 0; i-- {
		j := rng.IntN(i + 1)
		xs[i], xs[j] = xs[j], xs[i]
	}
	return xs
}

// measure builds a tree from keys in the given order and returns its height,
// the comparisons spent building it, and the comparisons to find every key once.
func measure(keys []int) (height, build, findAll int) {
	var t tree
	for _, k := range keys {
		_, c := t.insert(k)
		build += c
	}
	for _, k := range keys {
		_, c := t.search(k)
		findAll += c
	}
	return t.height(), build, findAll
}

func main() {
	sizes := []int{500, 1000, 2000, 4000}
	rng := rand.New(rand.NewPCG(1, 2))
	names := []string{"sorted", "reversed", "zig-zag", "shuffled"}
	builds := map[string][]int{}

	fmt.Println("(a) height and comparisons by input order (keys 0..n-1, one tree per row):")
	fmt.Printf("%6s  %-8s %6s %10s %10s %-5s %12s %10s %s\n",
		"n", "order", "height", "build", "n(n-1)/2", "equal", "find all", "n(n+1)/2", "equal")
	for _, n := range sizes {
		inputs := [][]int{sorted(n), reversed(n), zigzag(n), shuffled(n, rng)}
		for i, keys := range inputs {
			h, b, f := measure(keys)
			builds[names[i]] = append(builds[names[i]], b)
			fmt.Printf("%6d  %-8s %6d %10d %10d %-5v %12d %10d %v\n",
				n, names[i], h, b, n*(n-1)/2, b == n*(n-1)/2, f, n*(n+1)/2, f == n*(n+1)/2)
		}
	}

	fmt.Println("\n(b) build comparisons: the ratio from each n to the next (n doubles):")
	fmt.Printf("%-8s", "order")
	for i := 0; i+1 < len(sizes); i++ {
		fmt.Printf(" %11s", fmt.Sprintf("%d->%d", sizes[i], sizes[i+1]))
	}
	fmt.Println()
	for _, name := range names {
		fmt.Printf("%-8s", name)
		for i := 0; i+1 < len(sizes); i++ {
			fmt.Printf(" %11.3f", float64(builds[name][i+1])/float64(builds[name][i]))
		}
		fmt.Println()
	}

	fmt.Println("\n(c) random order, 100 trees per n (distinct keys from IntN(1<<30)): mean height, and mean")
	fmt.Println("    comparisons per successful search against Knuth's 2(1+1/n)H(n) - 3:")
	fmt.Printf("%6s %12s %10s %10s %9s\n", "n", "mean height", "mean hit", "formula", "gap")
	var lastHit float64
	for _, n := range sizes {
		hsum, csum := 0, 0
		for trial := range 100 {
			r := rand.New(rand.NewPCG(uint64(trial+1), 2))
			var t tree
			var keys []int
			for t.size < n {
				k := r.IntN(1 << 30)
				if ok, _ := t.insert(k); ok {
					keys = append(keys, k)
				}
			}
			hsum += t.height()
			for _, k := range keys {
				_, c := t.search(k)
				csum += c
			}
		}
		hn := 0.0 // H(n), summed in index order
		for i := 1; i <= n; i++ {
			hn += 1 / float64(i)
		}
		formula := 2*(1+1/float64(n))*hn - 3
		hit := float64(csum) / float64(100*n)
		lastHit = hit
		fmt.Printf("%6d %12.2f %10.3f %10.3f %+8.2f%%\n", n, float64(hsum)/100, hit, formula, (hit-formula)/formula*100)
	}

	n := sizes[len(sizes)-1]
	h, _, _ := measure(sorted(n))
	fmt.Printf("\nAt n = %d a successful search costs %.1f comparisons on average in the sorted tree (height %d)\n",
		n, float64(n*(n+1)/2)/float64(n), h)
	fmt.Printf("and %.2f in a random one: a factor of %.0f.\n", lastHit, float64(n*(n+1)/2)/float64(n)/lastHit)
	fmt.Printf("A recursive search on the sorted tree would recurse %d deep (its height); these programs iterate.\n", h)
}
```

**Output:**

```
(a) height and comparisons by input order (keys 0..n-1, one tree per row):
     n  order    height      build   n(n-1)/2 equal     find all   n(n+1)/2 equal
   500  sorted      500     124750     124750 true        125250     125250 true
   500  reversed    500     124750     124750 true        125250     125250 true
   500  zig-zag     500     124750     124750 true        125250     125250 true
   500  shuffled     19       5070     124750 false         5570     125250 false
  1000  sorted     1000     499500     499500 true        500500     500500 true
  1000  reversed   1000     499500     499500 true        500500     500500 true
  1000  zig-zag    1000     499500     499500 true        500500     500500 true
  1000  shuffled     24      10720     499500 false        11720     500500 false
  2000  sorted     2000    1999000    1999000 true       2001000    2001000 true
  2000  reversed   2000    1999000    1999000 true       2001000    2001000 true
  2000  zig-zag    2000    1999000    1999000 true       2001000    2001000 true
  2000  shuffled     25      23588    1999000 false        25588    2001000 false
  4000  sorted     4000    7998000    7998000 true       8002000    8002000 true
  4000  reversed   4000    7998000    7998000 true       8002000    8002000 true
  4000  zig-zag    4000    7998000    7998000 true       8002000    8002000 true
  4000  shuffled     25      51886    7998000 false        55886    8002000 false

(b) build comparisons: the ratio from each n to the next (n doubles):
order      500->1000  1000->2000  2000->4000
sorted         4.004       4.002       4.001
reversed       4.004       4.002       4.001
zig-zag        4.004       4.002       4.001
shuffled       2.114       2.200       2.200

(c) random order, 100 trees per n (distinct keys from IntN(1<<30)): mean height, and mean
    comparisons per successful search against Knuth's 2(1+1/n)H(n) - 3:
     n  mean height   mean hit    formula       gap
   500        19.24     10.518     10.613    -0.89%
  1000        21.70     11.877     11.986    -0.91%
  2000        24.44     13.256     13.365    -0.82%
  4000        27.06     14.637     14.747    -0.75%

At n = 4000 a successful search costs 2000.5 comparisons on average in the sorted tree (height 4000)
and 14.64 in a random one: a factor of 137.
A recursive search on the sorted tree would recurse 4000 deep (its height); these programs iterate.
```

**Rust:**

```rust
use std::cmp::Ordering;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program shuffles and draws exactly what the Go program does.
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

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// An iterative Drop. Rust's default drop glue recurses once per level, and every
// tree in (a) but the shuffled one is a chain of up to 4,000 nodes. That depth is
// harmless here, but this keeps the code safe at any n (lesson 17, example 13,
// counts the nesting depth of the default Drop). Go has no destructor to recurse in.
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

#[derive(Default)]
struct Tree {
    root: Link,
    size: usize,
}

impl Tree {
    // insert and search are ITERATIVE: on a degenerate tree a recursive version
    // would recurse n deep. Each returns the comparisons it spent.
    fn insert(&mut self, key: i64) -> (bool, u64) {
        let mut cmps = 0;
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            cmps += 1;
            match key.cmp(&n.key) {
                Ordering::Less => cur = &mut n.left,
                Ordering::Greater => cur = &mut n.right,
                Ordering::Equal => return (false, cmps),
            }
        }
        *cur = Some(Box::new(Node {
            key,
            left: None,
            right: None,
        }));
        self.size += 1;
        (true, cmps)
    }

    fn search(&self, key: i64) -> (bool, u64) {
        let mut cmps = 0;
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            cmps += 1;
            match key.cmp(&n.key) {
                Ordering::Less => cur = n.left.as_deref(),
                Ordering::Greater => cur = n.right.as_deref(),
                Ordering::Equal => return (true, cmps),
            }
        }
        (false, cmps)
    }

    // height by level order: count the levels, no recursion.
    fn height(&self) -> usize {
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
}

// The four input orders of the keys 0..n-1.
fn sorted(n: usize) -> Vec<i64> {
    (0..n as i64).collect()
}

fn reversed(n: usize) -> Vec<i64> {
    (0..n as i64).rev().collect()
}

// zigzag: lo, hi, lo+1, hi-1, ... the tree alternates right, left, right, ...
fn zigzag(n: usize) -> Vec<i64> {
    let mut xs = Vec::with_capacity(n);
    let (mut lo, mut hi) = (0, n as i64 - 1);
    while lo <= hi {
        xs.push(lo);
        lo += 1;
        if lo <= hi {
            xs.push(hi);
            hi -= 1;
        }
    }
    xs
}

// shuffled: Fisher-Yates by hand, so it consumes the stream exactly like Go's.
fn shuffled(n: usize, rng: &mut Pcg) -> Vec<i64> {
    let mut xs = sorted(n);
    for i in (1..n).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        xs.swap(i, j);
    }
    xs
}

// measure builds a tree from keys in the given order and returns its height,
// the comparisons spent building it, and the comparisons to find every key once.
fn measure(keys: &[i64]) -> (usize, u64, u64) {
    let mut t = Tree::default();
    let build = keys.iter().map(|&k| t.insert(k).1).sum();
    let find_all = keys.iter().map(|&k| t.search(k).1).sum();
    (t.height(), build, find_all)
}

fn main() {
    let sizes = [500usize, 1000, 2000, 4000];
    let mut rng = Pcg::new(1, 2);
    let names = ["sorted", "reversed", "zig-zag", "shuffled"];
    let mut builds: [Vec<u64>; 4] = Default::default();

    println!("(a) height and comparisons by input order (keys 0..n-1, one tree per row):");
    println!(
        "{:>6}  {:<8} {:>6} {:>10} {:>10} {:<5} {:>12} {:>10} equal",
        "n", "order", "height", "build", "n(n-1)/2", "equal", "find all", "n(n+1)/2"
    );
    for n in sizes {
        let inputs = [sorted(n), reversed(n), zigzag(n), shuffled(n, &mut rng)];
        let n64 = n as u64;
        for (i, keys) in inputs.iter().enumerate() {
            let (h, b, f) = measure(keys);
            builds[i].push(b);
            println!(
                "{n:6}  {:<8} {h:6} {b:10} {:10} {:<5} {f:12} {:10} {}",
                names[i],
                n64 * (n64 - 1) / 2,
                b == n64 * (n64 - 1) / 2,
                n64 * (n64 + 1) / 2,
                f == n64 * (n64 + 1) / 2
            );
        }
    }

    println!("\n(b) build comparisons: the ratio from each n to the next (n doubles):");
    print!("{:<8}", "order");
    for w in sizes.windows(2) {
        print!(" {:>11}", format!("{}->{}", w[0], w[1]));
    }
    println!();
    for (name, b) in names.iter().zip(&builds) {
        print!("{name:<8}");
        for w in b.windows(2) {
            print!(" {:11.3}", w[1] as f64 / w[0] as f64);
        }
        println!();
    }

    println!(
        "\n(c) random order, 100 trees per n (distinct keys from IntN(1<<30)): mean height, and mean"
    );
    println!("    comparisons per successful search against Knuth's 2(1+1/n)H(n) - 3:");
    println!(
        "{:>6} {:>12} {:>10} {:>10} {:>9}",
        "n", "mean height", "mean hit", "formula", "gap"
    );
    let mut last_hit = 0.0;
    for n in sizes {
        let (mut hsum, mut csum) = (0, 0);
        for trial in 0..100u64 {
            let mut r = Pcg::new(trial + 1, 2);
            let mut t = Tree::default();
            let mut keys = Vec::with_capacity(n);
            while t.size < n {
                let k = r.int_n(1 << 30) as i64;
                if t.insert(k).0 {
                    keys.push(k);
                }
            }
            hsum += t.height();
            csum += keys.iter().map(|&k| t.search(k).1).sum::<u64>();
        }
        let mut hn = 0.0; // H(n), summed in index order
        for i in 1..=n {
            hn += 1.0 / i as f64;
        }
        let formula = 2.0 * (1.0 + 1.0 / n as f64) * hn - 3.0;
        let hit = csum as f64 / (100 * n) as f64;
        last_hit = hit;
        println!(
            "{n:6} {:12.2} {hit:10.3} {formula:10.3} {:+8.2}%",
            hsum as f64 / 100.0,
            (hit - formula) / formula * 100.0
        );
    }

    let n = sizes[sizes.len() - 1] as u64;
    let (h, _, _) = measure(&sorted(n as usize));
    let sorted_hit = (n * (n + 1) / 2) as f64 / n as f64;
    println!(
        "\nAt n = {n} a successful search costs {sorted_hit:.1} comparisons on average in the sorted tree (height {h})"
    );
    println!(
        "and {last_hit:.2} in a random one: a factor of {:.0}.",
        sorted_hit / last_hit
    );
    println!(
        "A recursive search on the sorted tree would recurse {h} deep (its height); these programs iterate."
    );
}
```

**Rust output:**

```
(a) height and comparisons by input order (keys 0..n-1, one tree per row):
     n  order    height      build   n(n-1)/2 equal     find all   n(n+1)/2 equal
   500  sorted      500     124750     124750 true        125250     125250 true
   500  reversed    500     124750     124750 true        125250     125250 true
   500  zig-zag     500     124750     124750 true        125250     125250 true
   500  shuffled     19       5070     124750 false         5570     125250 false
  1000  sorted     1000     499500     499500 true        500500     500500 true
  1000  reversed   1000     499500     499500 true        500500     500500 true
  1000  zig-zag    1000     499500     499500 true        500500     500500 true
  1000  shuffled     24      10720     499500 false        11720     500500 false
  2000  sorted     2000    1999000    1999000 true       2001000    2001000 true
  2000  reversed   2000    1999000    1999000 true       2001000    2001000 true
  2000  zig-zag    2000    1999000    1999000 true       2001000    2001000 true
  2000  shuffled     25      23588    1999000 false        25588    2001000 false
  4000  sorted     4000    7998000    7998000 true       8002000    8002000 true
  4000  reversed   4000    7998000    7998000 true       8002000    8002000 true
  4000  zig-zag    4000    7998000    7998000 true       8002000    8002000 true
  4000  shuffled     25      51886    7998000 false        55886    8002000 false

(b) build comparisons: the ratio from each n to the next (n doubles):
order      500->1000  1000->2000  2000->4000
sorted         4.004       4.002       4.001
reversed       4.004       4.002       4.001
zig-zag        4.004       4.002       4.001
shuffled       2.114       2.200       2.200

(c) random order, 100 trees per n (distinct keys from IntN(1<<30)): mean height, and mean
    comparisons per successful search against Knuth's 2(1+1/n)H(n) - 3:
     n  mean height   mean hit    formula       gap
   500        19.24     10.518     10.613    -0.89%
  1000        21.70     11.877     11.986    -0.91%
  2000        24.44     13.256     13.365    -0.82%
  4000        27.06     14.637     14.747    -0.75%

At n = 4000 a successful search costs 2000.5 comparisons on average in the sorted tree (height 4000)
and 14.64 in a random one: a factor of 137.
A recursive search on the sorted tree would recurse 4000 deep (its height); these programs iterate.
```

> **Go vs Rust:** Identical to Go (diffed): heights, comparison counts, ratios and the means over 100 trees all agree, because the PCG-DXSM port draws the same keys, and the `f64` sums run in the same order. The `Drop` impl is iterative so a 4,000-node chain (or any longer one) cannot overflow the stack when it is freed; Rust's default `Drop` recurses once per level and Go has no destructor to recurse in. Nothing here recurses on a degenerate tree, so no overflow is demonstrated or needed.

**Complexity:** sorted, reversed and zig-zag input: height n, build exactly n(n−1)/2 comparisons, find-all exactly n(n+1)/2 (every row says `true`), about n/2 per search; build cost grows 4.00× per doubling · random order: expected height Θ(log n) (mean 27.06 at n = 4,000) and expected comparisons per hit `2(1+1/n)H(n) − 3` ≈ 1.39 log₂ n; measured means were 0.75% to 0.91% below the formula at every n (14.637 vs 14.747 at n = 4,000), and one shuffled build grows 2.11× to 2.20× per doubling · at n = 4,000 the sorted tree costs 2000.5 comparisons per search against 14.64, a factor of 137.

---

> Next tier: [🟡 medium](2-medium.md) — validation with bounds, the k-th smallest, range queries,
> building a balanced tree, and two-sum with two iterators.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
