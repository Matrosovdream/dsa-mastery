# Step 18 — Binary Search Trees · 🔴 Hard

Examples **12–16**: delete without recursion and what a held pointer sees, Hibbard's drift under random
churn, the tree against a sorted slice, the problems BSTs own, and the capstone.

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

## 12. Delete without recursion: a cursor and a held pointer

`🔴 hard` · *Pointer-to-pointer delete*

A recursive delete stores a link at every level on its way back up and needs a call stack as deep as the tree; two loops do the same job with a constant number of link stores and no stack. One tracks the **parent** and which side the node hangs on, which forces a three-way choice (root, left child, right child) wherever a link is rewritten; the other keeps a cursor that points at the **link** itself — in Go `link := &t.root`, a pointer to a pointer, in Rust a `&mut Option<Box<Node>>` — so root, middle node and leaf are one case, as in lesson 07 example 5 for lists. The second question is what the two-child case does with the successor: *copy* its key up (and any pointer someone held to the successor now points at a detached node), or *relink* the node, which keeps every surviving node's identity and costs more link stores. Iterative delete is Θ(h) time and Θ(1) extra space; the recursive one needs Θ(h) stack.

**Steps:**

1. Run the same 100,000 random operations through three deletes (recursive, iterative with parent tracking, iterative with a link cursor) and compare their results, comparison counts, trees and checker verdicts.
2. Tabulate which of the five delete cases each delete hits and how many link stores each version makes per delete; then hold a pointer to the successor of 50 across a two-child delete, copying and relinking.
3. Delete the largest key of a 4,000-node sorted chain with each version and read the call depth against the loop.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// A node with a payload (val) so that part (c) can ask which NODE a write hit.
type node struct {
	key         int
	val         string
	left, right *node
}

type tree struct{ root *node }

// insert is example 2's link-cursor insert; all three trees below are built
// with it, so only DELETE differs between them.
func (t *tree) insert(key int, val string) bool {
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
	*link = &node{key: key, val: val}
	return true
}

// What one delete did and cost. A "link store" is one executed assignment to a
// left/right field or to the root; a "key copy" moves a key and value between nodes.
type cost struct {
	kind   int
	cmps   int // key comparisons on the way down
	extra  int // left links followed to reach the successor (compares nothing)
	links  int
	copies int
}

const (
	absent    = iota
	leaf      // no children: clear the link
	onlyLeft  // one child: point the link at it
	onlyRight // one child: point the link at it
	succChild // two children, successor IS the right child
	succDeep  // two children, successor is deeper in the right subtree
	numKinds
)

var kindName = [numKinds]string{
	"key absent", "leaf", "only a left child", "only a right child",
	"two children, successor = right child", "two children, deeper successor",
}

// ---------------------------------------------------------------------------
// 1. Recursive (example 5): every level returns the new subtree root and its
//    caller stores it back, so a delete makes one link store per level.
//    The two-child case copies the successor's key up. d = nested calls so far.
// ---------------------------------------------------------------------------

func takeMin(n *node, c *cost) (key int, val string, rest *node) {
	if n.left == nil {
		return n.key, n.val, n.right
	}
	c.extra++
	k, v, r := takeMin(n.left, c)
	n.left = r
	c.links++
	return k, v, n
}

func deleteRec(n *node, key, d int, c *cost, deepest *int) *node {
	*deepest = max(*deepest, d)
	if n == nil {
		return nil
	}
	c.cmps++
	switch x := cmp.Compare(key, n.key); {
	case x < 0:
		n.left = deleteRec(n.left, key, d+1, c, deepest)
		c.links++
	case x > 0:
		n.right = deleteRec(n.right, key, d+1, c, deepest)
		c.links++
	case n.left == nil && n.right == nil:
		c.kind = leaf
		return nil
	case n.left == nil:
		c.kind = onlyRight
		return n.right
	case n.right == nil:
		c.kind = onlyLeft
		return n.left
	default:
		var rest *node
		n.key, n.val, rest = takeMin(n.right, c)
		n.right = rest
		c.copies++
		c.links++
		c.kind = succChild
		if c.extra > 0 {
			c.kind = succDeep
		}
	}
	return n
}

// ---------------------------------------------------------------------------
// 2. Iterative with explicit parent tracking. No recursion, but the root has
//    no parent, so every place that rewrites "the link to n" is a three-way
//    choice: root, left child of parent, or right child of parent.
//    The two-child case copies the successor's key up, as the recursion does.
// ---------------------------------------------------------------------------

func (t *tree) deleteParent(key int) (c cost) {
	var parent *node
	n, isLeft := t.root, false
	for n != nil {
		c.cmps++
		d := cmp.Compare(key, n.key)
		if d == 0 {
			break
		}
		parent = n
		if d < 0 {
			n, isLeft = n.left, true
		} else {
			n, isLeft = n.right, false
		}
	}
	if n == nil {
		return c
	}
	replace := func(child *node) { // store `child` where n hung
		switch {
		case parent == nil:
			t.root = child
		case isLeft:
			parent.left = child
		default:
			parent.right = child
		}
		c.links++
	}
	switch {
	case n.left == nil && n.right == nil:
		c.kind = leaf
		replace(nil)
	case n.right == nil:
		c.kind = onlyLeft
		replace(n.left)
	case n.left == nil:
		c.kind = onlyRight
		replace(n.right)
	default:
		sp, s := n, n.right // the successor and ITS parent: a second pair to track
		for s.left != nil {
			sp, s = s, s.left
			c.extra++
		}
		n.key, n.val = s.key, s.val
		c.copies++
		if sp == n {
			sp.right = s.right
		} else {
			sp.left = s.right
		}
		c.links++
		c.kind = succChild
		if c.extra > 0 {
			c.kind = succDeep
		}
	}
	return c
}

// ---------------------------------------------------------------------------
// 3. Iterative with a link cursor: `link` is the ADDRESS of whichever pointer
//    the descent stands on (&t.root, then &n.left or &n.right). Root, middle
//    and leaf are one case and there is no parent variable. The two-child case
//    RELINKS the successor node into n's place; no key is copied.
// ---------------------------------------------------------------------------

func (t *tree) delete(key int) (c cost) {
	link := &t.root
	for *link != nil {
		c.cmps++
		d := cmp.Compare(key, (*link).key)
		if d == 0 {
			break
		}
		if d < 0 {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	n := *link
	switch {
	case n == nil:
		return c
	case n.left == nil && n.right == nil:
		*link, c.kind, c.links = nil, leaf, 1
	case n.right == nil:
		*link, c.kind, c.links = n.left, onlyLeft, 1
	case n.left == nil:
		*link, c.kind, c.links = n.right, onlyRight, 1
	default:
		succ := &n.right // a second cursor, on the link that holds the successor
		for (*succ).left != nil {
			succ = &(*succ).left
			c.extra++
		}
		s := *succ
		*succ = s.right // detach s: it has no left child, so this is case 1 or 2
		s.left, s.right = n.left, n.right
		*link = s
		c.links = 4
		c.kind = succChild
		if c.extra > 0 {
			c.kind = succDeep
		}
	}
	return c
}

// ---------------------------------------------------------------------------
// Checkers and helpers
// ---------------------------------------------------------------------------

func same(a, b *node) bool { // identical shape AND identical keys
	if a == nil || b == nil {
		return a == b
	}
	return a.key == b.key && same(a.left, b.left) && same(a.right, b.right)
}

// check: the bounds invariant (nil bound = unbounded), which implies a
// strictly ascending in-order.
func check(t *tree) string {
	var walk func(n *node, lo, hi *int) string
	walk = func(n *node, lo, hi *int) string {
		if n == nil {
			return ""
		}
		if (lo != nil && n.key <= *lo) || (hi != nil && n.key >= *hi) {
			return fmt.Sprintf("key %d violates the bounds", n.key)
		}
		if m := walk(n.left, lo, &n.key); m != "" {
			return m
		}
		return walk(n.right, &n.key, hi)
	}
	return walk(t.root, nil, nil)
}

func reachable(root, target *node) bool {
	return root != nil && (root == target || reachable(root.left, target) || reachable(root.right, target))
}

func find(root *node, key int) *node {
	for root != nil && root.key != key {
		if key < root.key {
			root = root.left
		} else {
			root = root.right
		}
	}
	return root
}

func show(n *node, depth int) {
	if n == nil {
		return
	}
	show(n.right, depth+1)
	fmt.Printf("      %s%d\n", strings.Repeat("    ", depth), n.key)
	show(n.left, depth+1)
}

func pct(a, b int) float64 { return 100 * float64(a) / float64(b) }

func main() {
	// (a) Three implementations, one random history.
	const ops, span = 100_000, 200
	var rec, par, cur tree // recursive / parent tracking (both copy) / link cursor (relink)
	var oracle []int
	deepest := 0
	rng := rand.New(rand.NewPCG(1, 2))

	// per case: how many deletes, and the summed cost of each implementation
	var n, cmps, extra [numKinds]int
	var links, copies [3][numKinds]int
	disagree, shapeDiff, checks, checkFails := 0, 0, 0, 0
	for i := 1; i <= ops; i++ {
		key := rng.IntN(span)
		pos, found := slices.BinarySearch(oracle, key)
		if rng.IntN(2) == 0 {
			for _, t := range []*tree{&rec, &par, &cur} {
				if t.insert(key, "v") == found {
					disagree++
				}
			}
			if !found {
				oracle = slices.Insert(oracle, pos, key)
			}
		} else {
			var cr cost
			rec.root = deleteRec(rec.root, key, 1, &cr, &deepest)
			cr.links++ // the store into rec.root
			cp, cc := par.deleteParent(key), cur.delete(key)
			if cr.kind != cp.kind || cr.kind != cc.kind || cr.cmps != cp.cmps || cr.cmps != cc.cmps ||
				cr.extra != cp.extra || cr.extra != cc.extra || (cr.kind != absent) != found {
				disagree++
			}
			if found {
				oracle = slices.Delete(oracle, pos, pos+1)
			}
			k := cr.kind
			n[k]++
			cmps[k] += cr.cmps
			extra[k] += cr.extra
			for j, c := range []cost{cr, cp, cc} {
				links[j][k] += c.links
				copies[j][k] += c.copies
			}
		}
		if i <= 2000 || i%1000 == 0 {
			checks++
			if !same(rec.root, par.root) || !same(par.root, cur.root) {
				shapeDiff++
			}
			if check(&rec) != "" || check(&par) != "" || check(&cur) != "" {
				checkFails++
			}
		}
	}
	fmt.Printf("(a) %d random insert/delete ops on keys in [0, %d) against three deletes:\n", ops, span)
	fmt.Println("    recursive (copy the successor's key)  |  iterative, parent tracking (copy)  |  iterative, link cursor (relink)")
	fmt.Printf("  op results, delete case, comparisons and left-steps agree across the three and with a sorted slice: %d disagreements\n", disagree)
	fmt.Printf("  the three trees compared key for key at %d points (every op up to 2000, then every 1000th): %d differences\n", checks, shapeDiff)
	fmt.Printf("  checker (bounds, strictly ascending) on all three at the same points: %d failures; the oracle ends with %d keys\n\n", checkFails, len(oracle))

	// (b) Which case a random delete hits, and what each version spends on it.
	deletes, total := 0, [3]int{}
	for k := range numKinds {
		deletes += n[k]
		for j := range 3 {
			total[j] += links[j][k]
		}
	}
	fmt.Printf("(b) the %d deletes of that run, by case; link stores per delete (recursive / parent / cursor):\n", deletes)
	fmt.Printf("  %-38s %6s %6s %6s  %9s %7s %7s\n", "case", "count", "share", "cmps", "recursive", "parent", "cursor")
	mean := func(sum, count int) float64 { return float64(sum) / float64(count) }
	for k := range numKinds {
		fmt.Printf("  %-38s %6d %5.1f%% %6.2f  %9.2f %7.2f %7.2f\n", kindName[k], n[k], pct(n[k], deletes), mean(cmps[k], n[k]),
			mean(links[0][k], n[k]), mean(links[1][k], n[k]), mean(links[2][k], n[k]))
	}
	fmt.Printf("  all deletes: %d link stores recursive, %d parent, %d cursor (%.2f, %.2f, %.2f per delete)\n",
		total[0], total[1], total[2], mean(total[0], deletes), mean(total[1], deletes), mean(total[2], deletes))
	two := n[succChild] + n[succDeep]
	copySum := func(j int) (s int) {
		for k := range numKinds {
			s += copies[j][k]
		}
		return s
	}
	fmt.Printf("  key copies: %d recursive, %d parent, %d cursor; successor search: %.2f left-steps per two-child delete\n\n",
		copySum(0), copySum(1), copySum(2), mean(extra[succChild]+extra[succDeep], two))

	// (c) Copy the successor's key, or relink the successor node?
	fmt.Println("(c) hold a pointer h to the node with key 60 (the successor of 50), then delete 50:")
	build := func() *tree {
		t := new(tree)
		for _, k := range []int{50, 30, 70, 20, 40, 60, 80, 65} {
			t.insert(k, fmt.Sprintf("v%d", k))
		}
		return t
	}
	fmt.Println("  before:")
	show(build().root, 0)
	var results [2]*tree
	for i, relink := range []bool{false, true} {
		t := build()
		results[i] = t
		h, old50 := find(t.root, 60), find(t.root, 50)
		name := "copy the key up, unlink the successor"
		if relink {
			t.delete(50)
			name = "relink the successor node"
		} else {
			t.deleteParent(50)
		}
		holder := find(t.root, 60)
		h.val = "x" // write through the pointer we held
		fmt.Printf("  %s:\n", name)
		show(t.root, 0)
		fmt.Printf("    h.key = %d, h reachable from the root: %v, node now carrying key 60 is h: %v (the old 50 node: %v)\n",
			h.key, reachable(t.root, h), holder == h, holder == old50)
		fmt.Printf("    after h.val = \"x\": Get(60) = %q, so the write was %s; checker: %q\n",
			holder.val, map[bool]string{true: "visible", false: "lost"}[holder.val == "x"], check(t))
	}
	fmt.Printf("  the two results hold the same keys in the same shape: %v; what differs is which node carries 60\n", same(results[0].root, results[1].root))

	// (d) Stack depth: a sorted right chain, delete its largest key.
	const chain = 4000
	var rc, pc, cc tree
	for k := range chain {
		rc.insert(k, "v") // sorted input: a right chain of 4,000 nodes (example 6)
		pc.insert(k, "v")
		cc.insert(k, "v")
	}
	deep := 0
	var cr cost
	rc.root = deleteRec(rc.root, chain-1, 1, &cr, &deep)
	cp, cl := pc.deleteParent(chain-1), cc.delete(chain-1)
	fmt.Printf("\n(d) chain of %d nodes, delete the largest key:\n", chain)
	fmt.Printf("  recursive:        deepest call stack %d frames, %d link stores\n", deep, cr.links+1)
	fmt.Printf("  parent tracking:  %d loop steps in one call, 0 recursive frames, %d link store\n", cp.cmps, cp.links)
	fmt.Printf("  link cursor:      %d loop steps in one call, 0 recursive frames, %d link store\n", cl.cmps, cl.links)
}
```

**Output:**

```
(a) 100000 random insert/delete ops on keys in [0, 200) against three deletes:
    recursive (copy the successor's key)  |  iterative, parent tracking (copy)  |  iterative, link cursor (relink)
  op results, delete case, comparisons and left-steps agree across the three and with a sorted slice: 0 disagreements
  the three trees compared key for key at 2098 points (every op up to 2000, then every 1000th): 0 differences
  checker (bounds, strictly ascending) on all three at the same points: 0 failures; the oracle ends with 105 keys

(b) the 49923 deletes of that run, by case; link stores per delete (recursive / parent / cursor):
  case                                    count  share   cmps  recursive  parent  cursor
  key absent                              24921  49.9%   7.95       8.95    0.00    0.00
  leaf                                     8796  17.6%   8.50       8.50    1.00    1.00
  only a left child                        3433   6.9%   7.57       7.57    1.00    1.00
  only a right child                       4315   8.6%   7.70       7.70    1.00    1.00
  two children, successor = right child    3625   7.3%   6.68       7.68    1.00    4.00
  two children, deeper successor           4833   9.7%   5.18       7.94    1.00    4.00
  all deletes: 423272 link stores recursive, 25002 parent, 50376 cursor (8.48, 0.50, 1.01 per delete)
  key copies: 8458 recursive, 8458 parent, 0 cursor; successor search: 1.01 left-steps per two-child delete

(c) hold a pointer h to the node with key 60 (the successor of 50), then delete 50:
  before:
              80
          70
                  65
              60
      50
              40
          30
              20
  copy the key up, unlink the successor:
              80
          70
              65
      60
              40
          30
              20
    h.key = 60, h reachable from the root: false, node now carrying key 60 is h: false (the old 50 node: true)
    after h.val = "x": Get(60) = "v60", so the write was lost; checker: ""
  relink the successor node:
              80
          70
              65
      60
              40
          30
              20
    h.key = 60, h reachable from the root: true, node now carrying key 60 is h: true (the old 50 node: false)
    after h.val = "x": Get(60) = "x", so the write was visible; checker: ""
  the two results hold the same keys in the same shape: true; what differs is which node carries 60

(d) chain of 4000 nodes, delete the largest key:
  recursive:        deepest call stack 4000 frames, 4000 link stores
  parent tracking:  4000 loop steps in one call, 0 recursive frames, 1 link store
  link cursor:      4000 loop steps in one call, 0 recursive frames, 1 link store
```

**Rust:**

```rust
use std::cmp::Ordering;

// A node with a payload (val) so that part (c) can ask which NODE a write hit.
struct Node {
    key: i64,
    val: String,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (the default drop glue recurses once per level): part (d)
// builds a 4,000-node chain, and no tree here may overflow the stack.
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
}

// What one delete did and cost. A "link store" is one executed assignment to a
// left/right field or to the root; a "key copy" moves a key and value between nodes.
#[derive(Default)]
struct Cost {
    kind: Kind,
    cmps: usize,  // key comparisons on the way down
    extra: usize, // left links followed to reach the successor (compares nothing)
    links: usize,
    copies: usize,
}

#[derive(Clone, Copy, Default, PartialEq)]
enum Kind {
    #[default]
    Absent,
    Leaf,      // no children: clear the link
    OnlyLeft,  // one child: point the link at it
    OnlyRight, // one child: point the link at it
    SuccChild, // two children, successor IS the right child
    SuccDeep,  // two children, successor is deeper in the right subtree
}
const KINDS: [(Kind, &str); 6] = [
    (Kind::Absent, "key absent"),
    (Kind::Leaf, "leaf"),
    (Kind::OnlyLeft, "only a left child"),
    (Kind::OnlyRight, "only a right child"),
    (Kind::SuccChild, "two children, successor = right child"),
    (Kind::SuccDeep, "two children, deeper successor"),
];

fn new_node(key: i64, val: &str) -> Link {
    Some(Box::new(Node {
        key,
        val: val.to_string(),
        left: None,
        right: None,
    }))
}

// ---------------------------------------------------------------------------
// 1. Recursive (example 5): every level takes its subtree BY VALUE (`take()`),
//    returns the new subtree root, and its caller stores it back: one link
//    store per level. The two-child case copies the successor's key up.
//    `d` = nested calls so far.
// ---------------------------------------------------------------------------

fn take_min(mut n: Box<Node>, c: &mut Cost) -> (i64, String, Link) {
    match n.left.take() {
        None => {
            let rest = n.right.take();
            (n.key, std::mem::take(&mut n.val), rest)
        }
        Some(l) => {
            c.extra += 1;
            let (k, v, rest) = take_min(l, c);
            n.left = rest;
            c.links += 1;
            (k, v, Some(n))
        }
    }
}

fn delete_rec(link: Link, key: i64, d: usize, c: &mut Cost, deepest: &mut usize) -> Link {
    *deepest = (*deepest).max(d);
    let mut n = link?;
    c.cmps += 1;
    match key.cmp(&n.key) {
        Ordering::Less => {
            n.left = delete_rec(n.left.take(), key, d + 1, c, deepest);
            c.links += 1;
        }
        Ordering::Greater => {
            n.right = delete_rec(n.right.take(), key, d + 1, c, deepest);
            c.links += 1;
        }
        Ordering::Equal => match (n.left.take(), n.right.take()) {
            (None, None) => {
                c.kind = Kind::Leaf;
                return None;
            }
            (None, right) => {
                c.kind = Kind::OnlyRight;
                return right;
            }
            (left, None) => {
                c.kind = Kind::OnlyLeft;
                return left;
            }
            (left, Some(right)) => {
                let (k, v, rest) = take_min(right, c);
                (n.key, n.val, n.left, n.right) = (k, v, left, rest);
                c.copies += 1;
                c.links += 1;
                c.kind = if c.extra > 0 {
                    Kind::SuccDeep
                } else {
                    Kind::SuccChild
                };
            }
        },
    }
    Some(n)
}

impl Tree {
    // insert is example 2's cursor insert; all three trees below are built
    // with it, so only DELETE differs between them.
    fn insert(&mut self, key: i64, val: &str) -> bool {
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            match key.cmp(&n.key) {
                Ordering::Less => cur = &mut n.left,
                Ordering::Greater => cur = &mut n.right,
                Ordering::Equal => return false,
            }
        }
        *cur = new_node(key, val);
        true
    }

    // -----------------------------------------------------------------------
    // 2 and 3. Iterative with a link cursor. Rust has no parent pointers (no
    // `parent *node` as in the Go program: that needs Rc/Weak, an arena or
    // unsafe), and needs none: `cur` is a `&mut Option<Box<Node>>`, the
    // mutable reference to the link itself, which is exactly what the Go
    // parent+direction pair stands for. Root, middle and leaf are one case.
    //
    // `relink = false` copies the successor's key and value up and unlinks
    // the successor (what the recursion does); `relink = true` moves the
    // successor NODE into place and copies nothing.
    //
    // The loop asks with a SHORT-LIVED shared borrow (`as_ref`, over by the
    // next line) and only then reborrows `cur` to move it; example 5 (d)
    // shows the natural loop that does not compile, the note below the two
    // that people try next.
    // -----------------------------------------------------------------------
    fn delete(&mut self, key: i64, relink: bool) -> Cost {
        let mut c = Cost::default();
        let mut cur = &mut self.root;
        loop {
            let ord = match cur.as_ref() {
                None => break,
                Some(n) => key.cmp(&n.key),
            };
            c.cmps += 1;
            match ord {
                Ordering::Equal => break,
                Ordering::Less => cur = &mut cur.as_mut().unwrap().left,
                Ordering::Greater => cur = &mut cur.as_mut().unwrap().right,
            }
        }
        // `cur` now points at the link that holds the key (or at a nil link).
        let Some(mut node) = cur.take() else {
            return c;
        };
        *cur = match (node.left.take(), node.right.take()) {
            (None, None) => {
                (c.kind, c.links) = (Kind::Leaf, 1);
                None
            }
            (left @ Some(_), None) => {
                (c.kind, c.links) = (Kind::OnlyLeft, 1);
                left
            }
            (None, right @ Some(_)) => {
                (c.kind, c.links) = (Kind::OnlyRight, 1);
                right
            }
            (left, Some(right)) => {
                let mut right = Some(right);
                let mut succ = &mut right; // a second cursor, on the link holding the successor
                while succ.as_ref().unwrap().left.is_some() {
                    succ = &mut succ.as_mut().unwrap().left;
                    c.extra += 1;
                }
                let mut s = succ.take().unwrap();
                *succ = s.right.take(); // detach s: no left child, so case 1 or 2
                c.kind = if c.extra > 0 {
                    Kind::SuccDeep
                } else {
                    Kind::SuccChild
                };
                if relink {
                    (s.left, s.right) = (left, right);
                    c.links = 4;
                    Some(s)
                } else {
                    node.key = s.key;
                    node.val = std::mem::take(&mut s.val); // Node has a Drop impl: fields cannot be moved out
                    (node.left, node.right) = (left, right);
                    (c.copies, c.links) = (1, 1);
                    Some(node)
                }
            }
        };
        c
    }
}

// ---------------------------------------------------------------------------
// Checkers and helpers
// ---------------------------------------------------------------------------

fn same(a: &Link, b: &Link) -> bool {
    // identical shape AND identical keys
    match (a, b) {
        (None, None) => true,
        (Some(x), Some(y)) => x.key == y.key && same(&x.left, &y.left) && same(&x.right, &y.right),
        _ => false,
    }
}

// check: the bounds invariant (None = unbounded), which implies a strictly
// ascending in-order.
fn check(t: &Tree) -> String {
    fn walk(n: &Link, lo: Option<i64>, hi: Option<i64>) -> String {
        let Some(n) = n else { return String::new() };
        if lo.is_some_and(|lo| n.key <= lo) || hi.is_some_and(|hi| n.key >= hi) {
            return format!("key {} violates the bounds", n.key);
        }
        let m = walk(&n.left, lo, Some(n.key));
        if !m.is_empty() {
            return m;
        }
        walk(&n.right, Some(n.key), hi)
    }
    walk(&t.root, None, None)
}

// Where does the node with this key live? An ADDRESS, not a reference: a raw
// pointer carries no borrow, so the compiler lets the tree be mutated while we
// still hold it. We only ever compare it, never dereference it.
fn find(root: &Link, key: i64) -> Option<&Node> {
    let mut cur = root.as_deref();
    while let Some(n) = cur {
        match key.cmp(&n.key) {
            Ordering::Less => cur = n.left.as_deref(),
            Ordering::Greater => cur = n.right.as_deref(),
            Ordering::Equal => return Some(n),
        }
    }
    None
}

fn addr_of(root: &Link, key: i64) -> *const Node {
    find(root, key).map_or(std::ptr::null(), std::ptr::from_ref)
}

fn reachable(n: &Link, target: *const Node) -> bool {
    n.as_ref().is_some_and(|n| {
        std::ptr::eq(&**n, target) || reachable(&n.left, target) || reachable(&n.right, target)
    })
}

fn show(n: &Link, depth: usize) {
    if let Some(n) = n {
        show(&n.right, depth + 1);
        println!("      {}{}", "    ".repeat(depth), n.key);
        show(&n.left, depth + 1);
    }
}

fn pct(a: usize, b: usize) -> f64 {
    100.0 * a as f64 / b as f64
}

fn mean(sum: usize, count: usize) -> f64 {
    sum as f64 / count as f64
}

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the same
// seed gives the same draws and every count below matches the Go program's.
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

fn main() {
    // (a) Three implementations, one random history.
    const OPS: usize = 100_000;
    const SPAN: u64 = 200;
    // recursive / cursor copying the successor's key / cursor relinking it
    let (mut rec, mut cpy, mut rel) = (Tree::default(), Tree::default(), Tree::default());
    let mut oracle: Vec<i64> = Vec::new();
    let mut deepest = 0usize;
    let mut rng = Pcg::new(1, 2);

    // per case: how many deletes, and the summed cost of each implementation
    let (mut n, mut cmps, mut extra) = ([0usize; 6], [0usize; 6], [0usize; 6]);
    let (mut links, mut copies) = ([[0usize; 6]; 3], [[0usize; 6]; 3]);
    let (mut disagree, mut shape_diff, mut checks, mut check_fails) = (0, 0, 0, 0);
    for i in 1..=OPS {
        let key = rng.int_n(SPAN) as i64;
        let pos = oracle.partition_point(|&k| k < key);
        let found = oracle.get(pos) == Some(&key);
        if rng.int_n(2) == 0 {
            for t in [&mut rec, &mut cpy, &mut rel] {
                if t.insert(key, "v") == found {
                    disagree += 1;
                }
            }
            if !found {
                oracle.insert(pos, key);
            }
        } else {
            let mut cr = Cost::default();
            rec.root = delete_rec(rec.root.take(), key, 1, &mut cr, &mut deepest);
            cr.links += 1; // the store into rec.root
            let (cp, cc) = (cpy.delete(key, false), rel.delete(key, true));
            if cr.kind != cp.kind
                || cr.kind != cc.kind
                || cr.cmps != cp.cmps
                || cr.cmps != cc.cmps
                || cr.extra != cp.extra
                || cr.extra != cc.extra
                || (cr.kind != Kind::Absent) != found
            {
                disagree += 1;
            }
            if found {
                oracle.remove(pos);
            }
            let k = cr.kind as usize;
            n[k] += 1;
            cmps[k] += cr.cmps;
            extra[k] += cr.extra;
            for (j, c) in [&cr, &cp, &cc].into_iter().enumerate() {
                links[j][k] += c.links;
                copies[j][k] += c.copies;
            }
        }
        if i <= 2000 || i % 1000 == 0 {
            checks += 1;
            if !same(&rec.root, &cpy.root) || !same(&cpy.root, &rel.root) {
                shape_diff += 1;
            }
            if !check(&rec).is_empty() || !check(&cpy).is_empty() || !check(&rel).is_empty() {
                check_fails += 1;
            }
        }
    }
    println!("(a) {OPS} random insert/delete ops on keys in [0, {SPAN}) against three deletes:");
    println!(
        "    recursive (copy the successor's key)  |  iterative, cursor (copy)  |  iterative, cursor (relink)"
    );
    println!(
        "  op results, delete case, comparisons and left-steps agree across the three and with a sorted slice: {disagree} disagreements"
    );
    println!(
        "  the three trees compared key for key at {checks} points (every op up to 2000, then every 1000th): {shape_diff} differences"
    );
    println!(
        "  checker (bounds, strictly ascending) on all three at the same points: {check_fails} failures; the oracle ends with {} keys\n",
        oracle.len()
    );

    // (b) Which case a random delete hits, and what each version spends on it.
    let deletes: usize = n.iter().sum();
    let total: Vec<usize> = links.iter().map(|l| l.iter().sum()).collect();
    println!(
        "(b) the {deletes} deletes of that run, by case; link stores per delete (recursive / copy cursor / relink cursor):"
    );
    println!(
        "  {:<38} {:>6} {:>6} {:>6}  {:>9} {:>7} {:>7}",
        "case", "count", "share", "cmps", "recursive", "copy", "relink"
    );
    for (kind, name) in KINDS {
        let k = kind as usize;
        println!(
            "  {name:<38} {:>6} {:>5.1}% {:>6.2}  {:>9.2} {:>7.2} {:>7.2}",
            n[k],
            pct(n[k], deletes),
            mean(cmps[k], n[k]),
            mean(links[0][k], n[k]),
            mean(links[1][k], n[k]),
            mean(links[2][k], n[k])
        );
    }
    println!(
        "  all deletes: {} link stores recursive, {} copy, {} relink ({:.2}, {:.2}, {:.2} per delete)",
        total[0],
        total[1],
        total[2],
        mean(total[0], deletes),
        mean(total[1], deletes),
        mean(total[2], deletes)
    );
    let two = n[Kind::SuccChild as usize] + n[Kind::SuccDeep as usize];
    let copy_sum = |j: usize| copies[j].iter().sum::<usize>();
    println!(
        "  key copies: {} recursive, {} copy, {} relink; successor search: {:.2} left-steps per two-child delete\n",
        copy_sum(0),
        copy_sum(1),
        copy_sum(2),
        mean(
            extra[Kind::SuccChild as usize] + extra[Kind::SuccDeep as usize],
            two
        )
    );

    // (c) Copy the successor's key, or relink the successor node?
    println!(
        "(c) hold an address h of the node with key 60 (the successor of 50), then delete 50:"
    );
    println!(
        "    (Rust refuses to hold a &Node across the delete, E0502; h is a raw address, compared and never read)"
    );
    let build = || {
        let mut t = Tree::default();
        for k in [50, 30, 70, 20, 40, 60, 80, 65] {
            t.insert(k, &format!("v{k}"));
        }
        t
    };
    println!("  before:");
    show(&build().root, 0);
    let mut results = Vec::new();
    for relink in [false, true] {
        let mut t = build();
        let (h, old50) = (addr_of(&t.root, 60), addr_of(&t.root, 50));
        t.delete(50, relink);
        let name = if relink {
            "relink the successor node"
        } else {
            "copy the key up, unlink the successor"
        };
        // Compare the addresses BEFORE anything else allocates: in the copy
        // variant h is the address of a node that has just been freed.
        let holder = addr_of(&t.root, 60);
        let (h_reachable, is_h, is_old50) = (reachable(&t.root, h), holder == h, holder == old50);
        println!("  {name}:");
        show(&t.root, 0);
        println!(
            "    h reachable from the root: {h_reachable}, node now carrying key 60 is h: {is_h} (the old 50 node: {is_old50})"
        );
        // What the Go program did through its pointer, Rust does through the KEY:
        // look the node up again. That is always the live node.
        let before = find(&t.root, 60).map(|n| n.val.clone());
        let mut cur = &mut t.root;
        while cur.as_ref().is_some_and(|n| n.key != 60) {
            let n = cur.as_mut().unwrap();
            cur = if 60 < n.key {
                &mut n.left
            } else {
                &mut n.right
            };
        }
        if let Some(n) = cur {
            n.val = "x".to_string();
        }
        println!(
            "    re-lookup of key 60: {before:?}; after writing \"x\" through a fresh lookup: {:?}; checker: {:?}",
            find(&t.root, 60).map(|n| n.val.as_str()),
            check(&t)
        );
        results.push(t);
    }
    println!(
        "  the two results hold the same keys in the same shape: {}; what differs is which node carries 60",
        same(&results[0].root, &results[1].root)
    );

    // (d) Stack depth: a sorted right chain, delete its largest key.
    const CHAIN: i64 = 4000;
    let (mut rc, mut pc, mut cc) = (Tree::default(), Tree::default(), Tree::default());
    for k in 0..CHAIN {
        rc.insert(k, "v"); // sorted input: a right chain of 4,000 nodes (example 6)
        pc.insert(k, "v");
        cc.insert(k, "v");
    }
    let (mut deep, mut cr) = (0, Cost::default());
    rc.root = delete_rec(rc.root.take(), CHAIN - 1, 1, &mut cr, &mut deep);
    let (cp, cl) = (pc.delete(CHAIN - 1, false), cc.delete(CHAIN - 1, true));
    println!("\n(d) chain of {CHAIN} nodes, delete the largest key:");
    println!(
        "  recursive:       deepest call stack {deep} frames, {} link stores",
        cr.links + 1
    );
    println!(
        "  cursor (copy):   {} loop steps in one call, 0 recursive frames, {} link store",
        cp.cmps, cp.links
    );
    println!(
        "  cursor (relink): {} loop steps in one call, 0 recursive frames, {} link store",
        cl.cmps, cl.links
    );
}
```

**Rust output:**

```
(a) 100000 random insert/delete ops on keys in [0, 200) against three deletes:
    recursive (copy the successor's key)  |  iterative, cursor (copy)  |  iterative, cursor (relink)
  op results, delete case, comparisons and left-steps agree across the three and with a sorted slice: 0 disagreements
  the three trees compared key for key at 2098 points (every op up to 2000, then every 1000th): 0 differences
  checker (bounds, strictly ascending) on all three at the same points: 0 failures; the oracle ends with 105 keys

(b) the 49923 deletes of that run, by case; link stores per delete (recursive / copy cursor / relink cursor):
  case                                    count  share   cmps  recursive    copy  relink
  key absent                              24921  49.9%   7.95       8.95    0.00    0.00
  leaf                                     8796  17.6%   8.50       8.50    1.00    1.00
  only a left child                        3433   6.9%   7.57       7.57    1.00    1.00
  only a right child                       4315   8.6%   7.70       7.70    1.00    1.00
  two children, successor = right child    3625   7.3%   6.68       7.68    1.00    4.00
  two children, deeper successor           4833   9.7%   5.18       7.94    1.00    4.00
  all deletes: 423272 link stores recursive, 25002 copy, 50376 relink (8.48, 0.50, 1.01 per delete)
  key copies: 8458 recursive, 8458 copy, 0 relink; successor search: 1.01 left-steps per two-child delete

(c) hold an address h of the node with key 60 (the successor of 50), then delete 50:
    (Rust refuses to hold a &Node across the delete, E0502; h is a raw address, compared and never read)
  before:
              80
          70
                  65
              60
      50
              40
          30
              20
  copy the key up, unlink the successor:
              80
          70
              65
      60
              40
          30
              20
    h reachable from the root: false, node now carrying key 60 is h: false (the old 50 node: true)
    re-lookup of key 60: Some("v60"); after writing "x" through a fresh lookup: Some("x"); checker: ""
  relink the successor node:
              80
          70
              65
      60
              40
          30
              20
    h reachable from the root: true, node now carrying key 60 is h: true (the old 50 node: false)
    re-lookup of key 60: Some("v60"); after writing "x" through a fresh lookup: Some("x"); checker: ""
  the two results hold the same keys in the same shape: true; what differs is which node carries 60

(d) chain of 4000 nodes, delete the largest key:
  recursive:       deepest call stack 4000 frames, 4000 link stores
  cursor (copy):   4000 loop steps in one call, 0 recursive frames, 1 link store
  cursor (relink): 4000 loop steps in one call, 0 recursive frames, 1 link store
```

> **Go vs Rust:** Parts (a), (b) and (d) print the same numbers as the Go program (diffed). Only the labels differ: Go's "parent" column is Rust's "copy" column, because Rust has no parent-tracking version. A `parent *node` plus a direction flag is Go's way of naming a link; in Rust the way to name a link is a `&mut Option<Box<Node>>`, so the cursor loop replaces it, and the copy/relink difference is a flag on the same loop. Part (c) differs on purpose (below).
> 
> The cursor loop is example 5 (d)'s, extended with comparison counting and the relink. Example 5's note shows the first shape that fails to compile (`while let Some(node) = cur { … *cur = node.left.take() … }`, **E0506**). Two more that people try next, each compiled on rustc 1.98.1 and each rejected with **E0499** (a second mutable borrow of `*cur`) **and E0506**:
> 
> ```rust
> // does not compile
> loop {
>     match cur {
>         None => return false,
>         Some(node) => match key.cmp(&node.key) {
>             Ordering::Less => cur = &mut node.left,
>             Ordering::Greater => cur = &mut node.right,
>             Ordering::Equal => break,
>         },
>     }
> }
> let node = cur.as_mut().unwrap();
> *cur = node.left.take();
> 
> // does not compile
> while let Some(node) = cur.as_mut() {
>     match key.cmp(&node.key) { /* … Ordering::Equal => break */ }
> }
> match cur.take() { /* … */ }
> ```
> 
> The working loop above asks with a short-lived shared borrow (`cur.as_ref()` yields only an `Ordering`, so no borrow of `*cur` is alive on the next line), reborrows to move the cursor with `cur = &mut cur.as_mut().unwrap().left`, and calls `cur.take()` only after the loop. I do not explain it beyond what the compiler's messages say.
> 
> Part (c) cannot be written as in Go: `let h = t.find(60).unwrap(); t.delete(50); println!("{}", h.key)` is **E0502** ("cannot borrow `t` as mutable because it is also borrowed as immutable"), so the stale-pointer bug has no safe spelling. The twin keeps an address (`*const Node`, compared and never read) to ask the same identity question, and then does what Rust leaves you: hold the **key** and look the node up again, which is always the live node (the write is visible in both variants). The three verdict lines match Go's. What differs is what the old address means: Rust drops the detached successor, so after the copy variant a dereference of `h` would be a use-after-free, where Go's `h` keeps the node alive and `h.key` still reads 60.

**Complexity:** iterative delete is Θ(h) time and Θ(1) extra space (one cursor for the node, one for its successor); the recursive version used a 4,000-deep call stack on the chain where both loops used none. Over the 100,000 operations the three versions never disagreed (same results, same case, same comparisons and left-steps on every delete; 0 tree differences at 2,098 comparison points; 0 checker failures). The recursive delete made a link store at every level (8.48 per delete on average here) while the loops made a constant number per case: 0 for an absent key, 1 for a leaf or one child, 1 for the two-child copy and 4 for the two-child relink (0.50 per delete on average for the parent-tracking copy, 1.01 for the relink cursor). The relink trades 3 extra link stores for copying no key; copying left a held pointer detached and its write lost, relinking kept it in the tree.

---

## 13. Hibbard's drift: random deletes unbalance the tree

`🔴 hard` · *Deletion asymmetry*

Replacing a deleted two-child node with its successor, *always*, is lopsided: it only ever shortens right subtrees. A tree built by random insertions is balanced on average, and one such deletion leaves it random; the literature the lesson cites (Knuth; Culberson, 1985) reports that a long run of random delete-then-insert pairs on a tree maintained this way drifts away from the random-tree average, and that choosing successor or predecessor by a fair coin does not. This example measures how far, at sizes you can run: 8 independent trees of 1,000 keys churned for 3,000,000 steps under each policy, the mean comparisons per hit and the height at checkpoints along the way, set against the random-tree formula `2(1+1/n)H_n − 3`, and then the same recipe at smaller sizes to see whether the effect depends on n. Each churn step is Θ(h).

**Steps:**

1. Build 8 trees of 1,000 distinct random keys per policy (always successor; fair coin between successor and predecessor), each from its own PCG stream, and print the random-tree baseline.
2. Churn each tree — delete a random live key, insert a fresh random key in its slot — recording the mean depth `I/n + 1` and the height at checkpoints up to 3,000,000 steps; check every final tree.
3. Read the two columns against the baseline, then read the 3n²-steps table at n = 125, 250, 500, 1,000 to see how the effect grows with the size.

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

type tree struct{ root *node }

// insert: the link-cursor insert of example 2. Reports whether it added.
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
	return true
}

// The two policies for a node with two children.
const (
	alwaysSuccessor = iota // Hibbard's rule: always the in-order successor
	fairCoin               // successor or predecessor, decided by a fair coin
)

// delete is example 12's link-cursor delete, relinking a node (no key copy),
// with one addition: with two children it takes the successor (leftmost of the
// right subtree) or, under fairCoin and heads-or-tails, the predecessor
// (rightmost of the left subtree). The coin is drawn from the same PCG stream
// as the keys, and only when a node actually has two children.
func (t *tree) delete(key int, policy int, rng *rand.Rand) {
	link := &t.root
	for *link != nil && (*link).key != key {
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	n := *link
	switch {
	case n == nil:
		return
	case n.right == nil:
		*link = n.left
	case n.left == nil:
		*link = n.right
	default:
		var s *node
		if policy == fairCoin && rng.IntN(2) == 1 {
			p := &n.left // predecessor: rightmost node of the LEFT subtree
			for (*p).right != nil {
				p = &(*p).right
			}
			s = *p
			*p = s.left
		} else {
			p := &n.right // successor: leftmost node of the RIGHT subtree
			for (*p).left != nil {
				p = &(*p).left
			}
			s = *p
			*p = s.right
		}
		s.left, s.right = n.left, n.right
		*link = s
	}
}

// shape returns I (the sum of node depths), the node count and the height
// (nodes on the longest root-to-nil path). Depth is below 100 here: recursion is fine.
func shape(n *node, d int) (sum, count, height int) {
	if n == nil {
		return 0, 0, 0
	}
	ls, lc, lh := shape(n.left, d+1)
	rs, rc, rh := shape(n.right, d+1)
	return d + ls + rs, 1 + lc + rc, 1 + max(lh, rh)
}

// valid: strictly ascending keys in the (lo, hi) bounds, nil bound = unbounded.
func valid(n *node, lo, hi *int) bool {
	return n == nil || (!(lo != nil && n.key <= *lo) && !(hi != nil && n.key >= *hi) &&
		valid(n.left, lo, &n.key) && valid(n.right, &n.key, hi))
}

// result of one policy: per checkpoint, summed (or min/max-ed) over the trials.
type result struct {
	depth, minDepth, maxDepth, height []float64
	finalOK                           int
}

const trials = 8

// run grows `trials` trees of n random keys and churns each one until the last
// checkpoint: a step deletes a random live key and inserts a fresh random key
// in its slot. At every checkpoint it records the mean comparisons per hit
// (I/n + 1) and the height.
func run(policy, n int, checkpoints []int) result {
	r := result{make([]float64, len(checkpoints)), make([]float64, len(checkpoints)),
		make([]float64, len(checkpoints)), make([]float64, len(checkpoints)), 0}
	for i := range r.minDepth {
		r.minDepth[i] = 1e9
	}
	for trial := range trials {
		rng := rand.New(rand.NewPCG(uint64(trial+1), 2))
		var t tree
		live := make([]int, 0, n)
		for len(live) < n {
			if k := rng.IntN(1 << 30); t.insert(k) {
				live = append(live, k)
			}
		}
		next := 0 // index of the next checkpoint to record
		record := func(step int) {
			for next < len(checkpoints) && checkpoints[next] == step {
				sum, _, h := shape(t.root, 0)
				d := float64(sum)/float64(n) + 1
				r.depth[next] += d
				r.minDepth[next] = min(r.minDepth[next], d)
				r.maxDepth[next] = max(r.maxDepth[next], d)
				r.height[next] += float64(h)
				next++
			}
		}
		record(0)
		for step := 1; step <= checkpoints[len(checkpoints)-1]; step++ {
			i := rng.IntN(n)
			t.delete(live[i], policy, rng)
			for { // a fresh key that is not in the tree
				k := rng.IntN(1 << 30)
				if t.insert(k) {
					live[i] = k
					break
				}
			}
			record(step)
		}
		if _, count, _ := shape(t.root, 0); count == n && valid(t.root, nil, nil) {
			r.finalOK++
		}
	}
	for i := range r.depth {
		r.depth[i] /= trials
		r.height[i] /= trials
	}
	return r
}

// knuth: the random-tree average 2(1+1/n)H_n - 3 comparisons per hit.
func knuth(n int) float64 {
	h := 0.0
	for i := 1; i <= n; i++ {
		h += 1 / float64(i)
	}
	return 2*(1+1/float64(n))*h - 3
}

func main() {
	const n, steps = 1000, 3_000_000
	checkpoints := []int{0, 10_000, 50_000, 250_000, 500_000, 1_000_000, 2_000_000, steps}
	baseline := knuth(n)
	fmt.Printf("n = %d keys, %d independent trees per policy (PCG streams 1..%d), %d churn steps each\n", n, trials, trials, steps)
	fmt.Println("step = delete a random live key, then insert a fresh random key in its place")
	fmt.Printf("random-tree baseline 2(1+1/n)H_n - 3 = %.2f comparisons per hit; sorted insertion would cost (n+1)/2 = %.1f\n\n", baseline, float64(n+1)/2)

	succ, coin := run(alwaysSuccessor, n, checkpoints), run(fairCoin, n, checkpoints)

	fmt.Println("mean comparisons per hit, I/n + 1 (min-max over the trees), and mean height:")
	fmt.Printf("%10s | %-26s | %s\n", "", "always successor", "fair coin")
	fmt.Printf("%10s | %-18s %7s | %-18s %7s\n", "steps", "depth (min-max)", "height", "depth (min-max)", "height")
	cell := func(r *result, i int) string {
		return fmt.Sprintf("%-18s %7.1f", fmt.Sprintf("%.2f (%.1f-%.1f)", r.depth[i], r.minDepth[i], r.maxDepth[i]), r.height[i])
	}
	for i, s := range checkpoints {
		fmt.Printf("%10d | %s | %s\n", s, cell(&succ, i), cell(&coin, i))
	}
	last := len(checkpoints) - 1
	fmt.Printf("\nfinal trees passing the checker (strict order, bounds, %d keys): successor %d/%d, coin %d/%d\n", n, succ.finalOK, trials, coin.finalOK, trials)

	// Read the table: computed from the numbers above, not asserted.
	sd, cd := succ.depth[last], coin.depth[last]
	fmt.Printf("\nat step %d: successor %.2f, coin %.2f, baseline %.2f\n", steps, sd, cd, baseline)
	fmt.Printf("  successor is %+.1f%% against the baseline and %+.1f%% against the coin policy\n", 100*(sd/baseline-1), 100*(sd/cd-1))
	fmt.Printf("  coin is %+.1f%% against the baseline\n", 100*(cd/baseline-1))
	fmt.Printf("  successor, steps %d -> %d -> %d: %.2f -> %.2f -> %.2f (the last interval moved it %+.2f)\n",
		checkpoints[last-2], checkpoints[last-1], steps, succ.depth[last-2], succ.depth[last-1], sd, sd-succ.depth[last-1])

	// Does the effect depend on n? The same recipe at 3n^2 steps for smaller n.
	fmt.Println("\nthe same churn for 3n^2 steps at other sizes (mean over the same 8 PCG streams):")
	fmt.Printf("%6s %10s %9s | %15s %9s | %15s %9s\n", "n", "steps", "baseline", "successor depth", "vs base", "coin depth", "vs base")
	for _, m := range []int{125, 250, 500, n} {
		s := 3 * m * m
		cps := []int{0, s}
		a, b := succ, coin
		if m != n {
			a, b = run(alwaysSuccessor, m, cps), run(fairCoin, m, cps)
			s = cps[1]
		}
		base := knuth(m)
		fmt.Printf("%6d %10d %9.2f | %15.2f %+8.1f%% | %15.2f %+8.1f%%\n", m, s, base,
			a.depth[len(a.depth)-1], 100*(a.depth[len(a.depth)-1]/base-1),
			b.depth[len(b.depth)-1], 100*(b.depth[len(b.depth)-1]/base-1))
	}
}
```

**Output:**

```
n = 1000 keys, 8 independent trees per policy (PCG streams 1..8), 3000000 churn steps each
step = delete a random live key, then insert a fresh random key in its place
random-tree baseline 2(1+1/n)H_n - 3 = 11.99 comparisons per hit; sorted insertion would cost (n+1)/2 = 500.5

mean comparisons per hit, I/n + 1 (min-max over the trees), and mean height:
           | always successor           | fair coin
     steps | depth (min-max)     height | depth (min-max)     height
         0 | 11.71 (11.1-13.1)     22.1 | 11.71 (11.1-13.1)     22.1
     10000 | 11.22 (10.6-12.8)     19.9 | 11.18 (10.6-12.9)     19.8
     50000 | 10.91 (10.6-11.5)     18.4 | 10.93 (10.4-12.5)     18.8
    250000 | 11.26 (10.8-12.5)     21.5 | 10.75 (10.3-12.2)     18.6
    500000 | 12.30 (10.8-14.1)     25.6 | 10.69 (10.1-11.9)     18.0
   1000000 | 15.01 (14.0-17.0)     30.0 | 10.58 (10.2-11.5)     17.4
   2000000 | 15.58 (13.4-17.3)     30.9 | 10.43 (10.2-10.9)     18.1
   3000000 | 15.85 (14.7-17.0)     31.6 | 10.46 (10.1-11.2)     17.6

final trees passing the checker (strict order, bounds, 1000 keys): successor 8/8, coin 8/8

at step 3000000: successor 15.85, coin 10.46, baseline 11.99
  successor is +32.2% against the baseline and +51.5% against the coin policy
  coin is -12.7% against the baseline
  successor, steps 1000000 -> 2000000 -> 3000000: 15.01 -> 15.58 -> 15.85 (the last interval moved it +0.26)

the same churn for 3n^2 steps at other sizes (mean over the same 8 PCG streams):
     n      steps  baseline | successor depth   vs base |      coin depth   vs base
   125      46875      7.91 |            7.72     -2.3% |            7.33     -7.2%
   250     187500      9.25 |            9.85     +6.5% |            8.11    -12.3%
   500     750000     10.61 |           12.26    +15.5% |            9.38    -11.6%
  1000    3000000     11.99 |           15.85    +32.2% |           10.46    -12.7%
```

**Rust:**

```rust
use std::cmp::Ordering;

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (the default drop glue recurses once per level). These trees
// stay under ~60 deep, so it is not needed here, but every tree in this
// lesson carries the same node and the same Drop.
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
}

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the same
// stream (seed (trial+1, 2)) gives the same keys, the same coin flips and the
// same trees as the Go program.
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

// The two policies for a node with two children.
#[derive(Clone, Copy, PartialEq)]
enum Policy {
    AlwaysSuccessor, // Hibbard's rule: always the in-order successor
    FairCoin,        // successor or predecessor, decided by a fair coin
}

impl Tree {
    // insert: the cursor insert of example 2. Reports whether it added.
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
        true
    }

    // delete is example 12's cursor delete, relinking a node (no key copy), with
    // one addition: with two children it takes the successor (leftmost of the
    // right subtree) or, under FairCoin and a draw of 1, the predecessor
    // (rightmost of the left subtree). The coin comes from the same PCG stream
    // as the keys, and only when a node actually has two children.
    fn delete(&mut self, key: i64, policy: Policy, rng: &mut Pcg) {
        let mut cur = &mut self.root;
        while cur.as_ref().is_some_and(|n| n.key != key) {
            let n = cur.as_mut().unwrap(); // short-lived shared test, then a reborrow
            cur = if key < n.key {
                &mut n.left
            } else {
                &mut n.right
            };
        }
        let Some(mut node) = cur.take() else { return };
        *cur = match (node.left.take(), node.right.take()) {
            (None, right) => right,
            (left, None) => left,
            (Some(l), Some(r)) => {
                let (mut left, mut right) = (Some(l), Some(r));
                let s = if policy == Policy::FairCoin && rng.int_n(2) == 1 {
                    let mut p = &mut left; // predecessor: rightmost node of the LEFT subtree
                    while p.as_ref().unwrap().right.is_some() {
                        p = &mut p.as_mut().unwrap().right;
                    }
                    let mut s = p.take().unwrap();
                    *p = s.left.take();
                    s
                } else {
                    let mut p = &mut right; // successor: leftmost node of the RIGHT subtree
                    while p.as_ref().unwrap().left.is_some() {
                        p = &mut p.as_mut().unwrap().left;
                    }
                    let mut s = p.take().unwrap();
                    *p = s.right.take();
                    s
                };
                let mut s = s;
                (s.left, s.right) = (left, right);
                Some(s)
            }
        };
    }
}

// shape returns I (the sum of node depths), the node count and the height
// (nodes on the longest root-to-nil path). Depth is below 100 here: recursion is fine.
fn shape(n: &Link, d: usize) -> (usize, usize, usize) {
    match n {
        None => (0, 0, 0),
        Some(n) => {
            let (ls, lc, lh) = shape(&n.left, d + 1);
            let (rs, rc, rh) = shape(&n.right, d + 1);
            (d + ls + rs, 1 + lc + rc, 1 + lh.max(rh))
        }
    }
}

// valid: strictly ascending keys in the (lo, hi) bounds, None = unbounded.
fn valid(n: &Link, lo: Option<i64>, hi: Option<i64>) -> bool {
    n.as_ref().is_none_or(|n| {
        lo.is_none_or(|lo| n.key > lo)
            && hi.is_none_or(|hi| n.key < hi)
            && valid(&n.left, lo, Some(n.key))
            && valid(&n.right, Some(n.key), hi)
    })
}

// result of one policy: per checkpoint, summed (or min/max-ed) over the trials.
struct Outcome {
    depth: Vec<f64>,
    min_depth: Vec<f64>,
    max_depth: Vec<f64>,
    height: Vec<f64>,
    final_ok: usize,
}

const TRIALS: usize = 8;

// run grows TRIALS trees of n random keys and churns each one until the last
// checkpoint: a step deletes a random live key and inserts a fresh random key
// in its slot. At every checkpoint it records the mean comparisons per hit
// (I/n + 1) and the height.
fn run(policy: Policy, n: usize, checkpoints: &[usize]) -> Outcome {
    let k = checkpoints.len();
    let mut r = Outcome {
        depth: vec![0.0; k],
        min_depth: vec![1e9; k],
        max_depth: vec![0.0; k],
        height: vec![0.0; k],
        final_ok: 0,
    };
    for trial in 0..TRIALS {
        let mut rng = Pcg::new(trial as u64 + 1, 2);
        let mut t = Tree::default();
        let mut live: Vec<i64> = Vec::with_capacity(n);
        while live.len() < n {
            let key = rng.int_n(1 << 30) as i64;
            if t.insert(key) {
                live.push(key);
            }
        }
        let mut next = 0; // index of the next checkpoint to record
        let mut record = |t: &Tree, step: usize| {
            while next < k && checkpoints[next] == step {
                let (sum, _, h) = shape(&t.root, 0);
                let d = sum as f64 / n as f64 + 1.0;
                r.depth[next] += d;
                r.min_depth[next] = r.min_depth[next].min(d);
                r.max_depth[next] = r.max_depth[next].max(d);
                r.height[next] += h as f64;
                next += 1;
            }
        };
        record(&t, 0);
        for step in 1..=checkpoints[k - 1] {
            let i = rng.int_n(n as u64) as usize;
            t.delete(live[i], policy, &mut rng);
            loop {
                // a fresh key that is not in the tree
                let key = rng.int_n(1 << 30) as i64;
                if t.insert(key) {
                    live[i] = key;
                    break;
                }
            }
            record(&t, step);
        }
        let (_, count, _) = shape(&t.root, 0);
        if count == n && valid(&t.root, None, None) {
            r.final_ok += 1;
        }
    }
    for i in 0..k {
        r.depth[i] /= TRIALS as f64;
        r.height[i] /= TRIALS as f64;
    }
    r
}

// knuth: the random-tree average 2(1+1/n)H_n - 3 comparisons per hit.
fn knuth(n: usize) -> f64 {
    let mut h = 0.0;
    for i in 1..=n {
        h += 1.0 / i as f64;
    }
    2.0 * (1.0 + 1.0 / n as f64) * h - 3.0
}

fn main() {
    const N: usize = 1000;
    const STEPS: usize = 3_000_000;
    let checkpoints = [
        0, 10_000, 50_000, 250_000, 500_000, 1_000_000, 2_000_000, STEPS,
    ];
    let baseline = knuth(N);
    println!(
        "n = {N} keys, {TRIALS} independent trees per policy (PCG streams 1..{TRIALS}), {STEPS} churn steps each"
    );
    println!("step = delete a random live key, then insert a fresh random key in its place");
    println!(
        "random-tree baseline 2(1+1/n)H_n - 3 = {baseline:.2} comparisons per hit; sorted insertion would cost (n+1)/2 = {:.1}\n",
        (N + 1) as f64 / 2.0
    );

    let succ = run(Policy::AlwaysSuccessor, N, &checkpoints);
    let coin = run(Policy::FairCoin, N, &checkpoints);

    println!("mean comparisons per hit, I/n + 1 (min-max over the trees), and mean height:");
    println!("{:>10} | {:<26} | fair coin", "", "always successor");
    println!(
        "{:>10} | {:<18} {:>7} | {:<18} {:>7}",
        "steps", "depth (min-max)", "height", "depth (min-max)", "height"
    );
    let cell = |r: &Outcome, i: usize| {
        let d = format!(
            "{:.2} ({:.1}-{:.1})",
            r.depth[i], r.min_depth[i], r.max_depth[i]
        );
        format!("{d:<18} {:>7.1}", r.height[i])
    };
    for (i, s) in checkpoints.iter().enumerate() {
        println!("{s:>10} | {} | {}", cell(&succ, i), cell(&coin, i));
    }
    let last = checkpoints.len() - 1;
    println!(
        "\nfinal trees passing the checker (strict order, bounds, {N} keys): successor {}/{TRIALS}, coin {}/{TRIALS}",
        succ.final_ok, coin.final_ok
    );

    // Read the table: computed from the numbers above, not asserted.
    let (sd, cd) = (succ.depth[last], coin.depth[last]);
    println!("\nat step {STEPS}: successor {sd:.2}, coin {cd:.2}, baseline {baseline:.2}");
    println!(
        "  successor is {:+.1}% against the baseline and {:+.1}% against the coin policy",
        100.0 * (sd / baseline - 1.0),
        100.0 * (sd / cd - 1.0)
    );
    println!(
        "  coin is {:+.1}% against the baseline",
        100.0 * (cd / baseline - 1.0)
    );
    println!(
        "  successor, steps {} -> {} -> {STEPS}: {:.2} -> {:.2} -> {sd:.2} (the last interval moved it {:+.2})",
        checkpoints[last - 2],
        checkpoints[last - 1],
        succ.depth[last - 2],
        succ.depth[last - 1],
        sd - succ.depth[last - 1]
    );

    // Does the effect depend on n? The same recipe at 3n^2 steps for smaller n.
    println!(
        "\nthe same churn for 3n^2 steps at other sizes (mean over the same {TRIALS} PCG streams):"
    );
    println!(
        "{:>6} {:>10} {:>9} | {:>15} {:>9} | {:>15} {:>9}",
        "n", "steps", "baseline", "successor depth", "vs base", "coin depth", "vs base"
    );
    for m in [125, 250, 500, N] {
        let s = 3 * m * m;
        let owned;
        let (a, b) = if m == N {
            (&succ, &coin)
        } else {
            let cps = [0, s];
            owned = (
                run(Policy::AlwaysSuccessor, m, &cps),
                run(Policy::FairCoin, m, &cps),
            );
            (&owned.0, &owned.1)
        };
        let base = knuth(m);
        let (da, db) = (a.depth[a.depth.len() - 1], b.depth[b.depth.len() - 1]);
        println!(
            "{m:>6} {s:>10} {base:>9.2} | {da:>15.2} {:>+8.1}% | {db:>15.2} {:>+8.1}%",
            100.0 * (da / base - 1.0),
            100.0 * (db / base - 1.0)
        );
    }
}
```

**Rust output:**

```
n = 1000 keys, 8 independent trees per policy (PCG streams 1..8), 3000000 churn steps each
step = delete a random live key, then insert a fresh random key in its place
random-tree baseline 2(1+1/n)H_n - 3 = 11.99 comparisons per hit; sorted insertion would cost (n+1)/2 = 500.5

mean comparisons per hit, I/n + 1 (min-max over the trees), and mean height:
           | always successor           | fair coin
     steps | depth (min-max)     height | depth (min-max)     height
         0 | 11.71 (11.1-13.1)     22.1 | 11.71 (11.1-13.1)     22.1
     10000 | 11.22 (10.6-12.8)     19.9 | 11.18 (10.6-12.9)     19.8
     50000 | 10.91 (10.6-11.5)     18.4 | 10.93 (10.4-12.5)     18.8
    250000 | 11.26 (10.8-12.5)     21.5 | 10.75 (10.3-12.2)     18.6
    500000 | 12.30 (10.8-14.1)     25.6 | 10.69 (10.1-11.9)     18.0
   1000000 | 15.01 (14.0-17.0)     30.0 | 10.58 (10.2-11.5)     17.4
   2000000 | 15.58 (13.4-17.3)     30.9 | 10.43 (10.2-10.9)     18.1
   3000000 | 15.85 (14.7-17.0)     31.6 | 10.46 (10.1-11.2)     17.6

final trees passing the checker (strict order, bounds, 1000 keys): successor 8/8, coin 8/8

at step 3000000: successor 15.85, coin 10.46, baseline 11.99
  successor is +32.2% against the baseline and +51.5% against the coin policy
  coin is -12.7% against the baseline
  successor, steps 1000000 -> 2000000 -> 3000000: 15.01 -> 15.58 -> 15.85 (the last interval moved it +0.26)

the same churn for 3n^2 steps at other sizes (mean over the same 8 PCG streams):
     n      steps  baseline | successor depth   vs base |      coin depth   vs base
   125      46875      7.91 |            7.72     -2.3% |            7.33     -7.2%
   250     187500      9.25 |            9.85     +6.5% |            8.11    -12.3%
   500     750000     10.61 |           12.26    +15.5% |            9.38    -11.6%
  1000    3000000     11.99 |           15.85    +32.2% |           10.46    -12.7%
```

> **Go vs Rust:** Output identical to the Go program's, byte for byte (diffed): same PCG streams, the same relink loop as example 12's cursor delete, and the coin drawn only for two-child nodes, so a different number of draws in either language would have produced different trees. Both programs ran for several seconds here (8 trees x 3,000,000 steps x 2 policies, plus the smaller sizes). `BTreeMap` has no such issue because a B-tree rebalances on every update (lesson 19); that is stated, not measured here.

**Complexity:** each churn step is Θ(h) (one descent to delete, one to insert). Measured over 8 trees of 1,000 keys: always taking the successor moved the mean comparisons per hit from 11.71 (the random-tree formula says 11.99) down to 10.91 after 50,000 steps and then up to 15.85 after 3,000,000, +32% over the formula, with mean height 22.1 → 31.6; the 8 trees ranged 14.7–17.0 at the end and the curve was still creeping up (+0.26 over the last million steps), so this run shows a drift but not a plateau. The fair coin did not drift: from 1,000,000 steps on it stayed at 10.4–10.6 (about 13% *below* the random-tree formula) with mean height about 18. At 3n² steps the successor policy's excess over the formula grew with n (−2.3% at n = 125, +6.5% at 250, +15.5% at 500, +32.2% at 1,000), so at small sizes there is no drift to see. The program measures the effect, not the mechanism (that deletion only ever shortens right subtrees is the lesson's explanation).

---

## 14. Ordered and dynamic: a BST against a sorted slice

`🔴 hard` · *The workload both earlier lessons deferred*

[Lesson 04 example 14](../04-go-memory-model/3-hard.md) and [lesson 06 example 12](../06-arrays-slices/3-hard.md) each ended on the same point: a sorted slice cannot be updated cheaply, so data that is ordered *and* changing wants a tree. This runs that workload: n random keys, then update pairs (delete a random live key, insert a fresh one), plus predecessor queries and 50-key range scans. A sorted slice pays Θ(log n) comparisons for the binary search plus Θ(n) element moves per update; a random-order BST pays Θ(h) comparisons (printed below) and moves nothing, but follows a pointer per step, and fed sorted keys it degrades to Θ(n) comparisons per operation. The example counts first (deterministic, and the reason behind the timings) and then times each workload at sizes from 16 to 65,536 as best of three interleaved rounds. Go has no ordered map in its standard library, so the opponent is a hand-written tree.

**Steps:**

1. Prefill n random distinct keys and apply 10,000 update pairs to a sorted slice and to a BST, counting binary-search probes, element moves and tree comparisons; add a BST fed the same keys in sorted order.
2. For n from 16 to 65,536 build both structures outside the timer, check that they answer identically, then time update pairs, predecessor queries and 50-key scans, rebuilding the input before every timed pass.
3. Read the counts against the three timing tables and find, for each workload, the sizes at which the tree is faster.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"os"
	"slices"
	"time"
)

// ---------------------------------------------------------------------------
// The sorted slice. A probe is one comparison. Every update pays the binary
// search AND a memmove of everything behind the position; copy() returns the
// number of elements it moved, so the moves are counted, not computed.
// ---------------------------------------------------------------------------

func lowerBound(s []int, key int) (pos, probes int) { // first index with s[i] >= key
	lo, hi := 0, len(s)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		probes++
		if s[mid] < key {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo, probes
}

func sliceDelete(s []int, key int) (_ []int, probes, moves int) {
	pos, probes := lowerBound(s, key)
	moves = copy(s[pos:], s[pos+1:])
	return s[:len(s)-1], probes, moves
}

// sliceInsert needs spare capacity (the caller makes it), so no reallocation is timed.
func sliceInsert(s []int, key int) (_ []int, probes, moves int) {
	pos, probes := lowerBound(s, key)
	s = append(s, 0)
	moves = copy(s[pos+1:], s[pos:])
	s[pos] = key
	return s, probes, moves
}

// ---------------------------------------------------------------------------
// The BST: link-cursor insert (example 2) and relinking delete (example 12).
// A comparison is one three-way compare against one node's key; both are
// iterative, so a degenerate tree cannot overflow the stack.
// ---------------------------------------------------------------------------

type node struct {
	key         int
	left, right *node
}

type bst struct{ root *node }

func (t *bst) insert(key int) (cmps int) {
	link := &t.root
	for *link != nil {
		cmps++
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	*link = &node{key: key}
	return cmps
}

func (t *bst) delete(key int) (cmps int) {
	link := &t.root
	for *link != nil && (*link).key != key {
		cmps++
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	n := *link
	if n == nil {
		return cmps
	}
	cmps++ // the comparison that found it
	switch {
	case n.right == nil:
		*link = n.left
	case n.left == nil:
		*link = n.right
	default:
		succ := &n.right
		for (*succ).left != nil {
			succ = &(*succ).left
		}
		s := *succ
		*succ = s.right
		s.left, s.right = n.left, n.right
		*link = s
	}
	return cmps
}

// pred: the largest key below `key`: one descent that remembers the last right turn.
func (t *bst) pred(key int) (best int, ok bool) {
	for n := t.root; n != nil; {
		if n.key < key {
			best, ok = n.key, true
			n = n.right
		} else {
			n = n.left
		}
	}
	return best, ok
}

// scan sums the first `count` keys >= lo, in order, with an explicit stack that
// the caller reuses (so no allocation is timed): seed it with the left spine of
// the nodes that are >= lo, pop, then push the left spine of the right child.
func (t *bst) scan(lo, count int, stack []*node) (sum int, _ []*node) {
	stack = stack[:0]
	for n := t.root; n != nil; {
		if n.key >= lo {
			stack = append(stack, n)
			n = n.left
		} else {
			n = n.right
		}
	}
	for i := 0; i < count && len(stack) > 0; i++ {
		n := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		sum += n.key
		for m := n.right; m != nil; m = m.left {
			stack = append(stack, m)
		}
	}
	return sum, stack
}

func (t *bst) keys() []int { // sorted contents, iteratively
	var out []int
	var stack []*node
	for n := t.root; n != nil || len(stack) > 0; {
		for ; n != nil; n = n.left {
			stack = append(stack, n)
		}
		n = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		out = append(out, n.key)
		n = n.right
	}
	return out
}

func (t *bst) height() int { // nodes on the longest root-to-nil path, counted level by level
	h := 0
	for level := []*node{t.root}; level[0] != nil && len(level) > 0; h++ {
		var next []*node
		for _, n := range level {
			for _, c := range []*node{n.left, n.right} {
				if c != nil {
					next = append(next, c)
				}
			}
		}
		if len(next) == 0 {
			return h + 1
		}
		level = next
	}
	return h
}

// ---------------------------------------------------------------------------
// The workload: n distinct random keys, then update PAIRS: delete a random live
// key, insert a fresh random key in its place. Same draws in the Rust twin.
// ---------------------------------------------------------------------------

type pair struct{ del, ins int }

func workload(rng *rand.Rand, n, pairs int) (prefill []int, ps []pair) {
	live := make([]int, 0, n)
	set := make(map[int]struct{}, n)
	draw := func() int {
		for {
			if k := rng.IntN(1 << 30); !has(set, k) {
				set[k] = struct{}{}
				return k
			}
		}
	}
	for len(live) < n {
		live = append(live, draw())
	}
	prefill = slices.Clone(live)
	for range pairs {
		i := rng.IntN(n)
		del := live[i]
		delete(set, del)
		live[i] = draw()
		ps = append(ps, pair{del, live[i]})
	}
	return prefill, ps
}

func has(set map[int]struct{}, k int) bool { _, ok := set[k]; return ok }

// ---------------------------------------------------------------------------
// (a) counts
// ---------------------------------------------------------------------------

type counts struct{ delCmps, insCmps, delMoves, insMoves, ops int }

func countSlice(prefill []int, ps []pair) counts {
	s := make([]int, len(prefill), len(prefill)+1)
	copy(s, prefill)
	slices.Sort(s)
	var c counts
	for _, p := range ps {
		var pr, mv int
		s, pr, mv = sliceDelete(s, p.del)
		c.delCmps, c.delMoves = c.delCmps+pr, c.delMoves+mv
		s, pr, mv = sliceInsert(s, p.ins)
		c.insCmps, c.insMoves = c.insCmps+pr, c.insMoves+mv
	}
	c.ops = len(ps)
	return c
}

// countBST returns the counts, the final tree and the height before the updates.
func countBST(prefill []int, ps []pair) (counts, *bst, int) {
	t := new(bst)
	for _, k := range prefill {
		t.insert(k)
	}
	h := t.height()
	var c counts
	for _, p := range ps {
		c.delCmps += t.delete(p.del)
		c.insCmps += t.insert(p.ins)
	}
	c.ops = len(ps)
	return c, t, h
}

// ---------------------------------------------------------------------------
// (b) timing: each structure is a runner whose timed loops are closures, so the
// hot loops make direct calls. Inputs are rebuilt OUTSIDE the timer every round
// (the update workload mutates its structure; the queries do not).
// ---------------------------------------------------------------------------

type runner struct {
	name  string
	build func(prefill []int)      // untimed: a fresh structure from the pristine keys
	upd   func(ps []pair)          // timed
	pred  func(qs []int) (sum int) // timed: sum of (predecessor + 1, or 0)
	scan  func(qs []int) (sum int) // timed: sum of the first 50 keys >= q
	keys  func() []int
}

const scanLen = 50

func sliceRunner() runner {
	var s []int
	return runner{
		name: "sorted slice",
		build: func(prefill []int) {
			s = make([]int, len(prefill), len(prefill)+1)
			copy(s, prefill)
			slices.Sort(s)
		},
		upd: func(ps []pair) {
			for _, p := range ps {
				s, _, _ = sliceDelete(s, p.del)
				s, _, _ = sliceInsert(s, p.ins)
			}
		},
		pred: func(qs []int) (sum int) {
			for _, q := range qs {
				if pos, _ := lowerBound(s, q); pos > 0 {
					sum += s[pos-1] + 1
				}
			}
			return sum
		},
		scan: func(qs []int) (sum int) {
			for _, q := range qs {
				pos, _ := lowerBound(s, q)
				for _, k := range s[pos:min(pos+scanLen, len(s))] {
					sum += k
				}
			}
			return sum
		},
		keys: func() []int { return s },
	}
}

func bstRunner() runner {
	t := new(bst)
	stack := make([]*node, 0, 256)
	return runner{
		name: "BST",
		build: func(prefill []int) {
			t = new(bst)
			for _, k := range prefill {
				t.insert(k)
			}
		},
		upd: func(ps []pair) {
			for _, p := range ps {
				t.delete(p.del)
				t.insert(p.ins)
			}
		},
		pred: func(qs []int) (sum int) {
			for _, q := range qs {
				if k, ok := t.pred(q); ok {
					sum += k + 1
				}
			}
			return sum
		},
		scan: func(qs []int) (sum int) {
			for _, q := range qs {
				var s int
				s, stack = t.scan(q, scanLen, stack)
				sum += s
			}
			return sum
		},
		keys: func() []int { return t.keys() }, // t is rebuilt by build: do not bind the old one
	}
}

var sink int // keeps the compiler from discarding the timed work

// bestOf runs `rounds` interleaved rounds: in each round every runner gets a
// freshly built structure and one timed pass. It returns the best ns/op per runner.
func bestOf(rounds, ops int, rs []runner, prefill []int, work func(r runner) int) []float64 {
	best := make([]float64, len(rs))
	for i := range best {
		best[i] = math.Inf(1)
	}
	for range rounds {
		for i, r := range rs {
			r.build(prefill)
			t0 := time.Now()
			sink += work(r)
			best[i] = min(best[i], float64(time.Since(t0).Nanoseconds())/float64(ops))
		}
	}
	return best
}

func main() {
	big := len(os.Args) > 1 && os.Args[1] == "--big"
	rng := rand.New(rand.NewPCG(1, 2))

	// (a) Counts first: deterministic, and the reason the timings come out as they do.
	fmt.Println("(a) counts: n random keys, then 10,000 update pairs (delete a random live key, insert a fresh one)")
	fmt.Println("    slice: a comparison is one probe of the binary search; a move is one element shifted by copy()")
	fmt.Println("    BST:   a comparison is one node's key looked at; a BST update moves nothing")
	fmt.Printf("  %-36s %6s %7s %7s %9s %9s %12s\n", "structure", "n", "cmp/del", "cmp/ins", "moves/del", "moves/ins", "moves per cmp")
	row := func(name string, n int, c counts) {
		f := func(x int) float64 { return float64(x) / float64(c.ops) }
		fmt.Printf("  %-36s %6d %7.2f %7.2f %9.1f %9.1f %12.1f\n", name, n, f(c.delCmps), f(c.insCmps), f(c.delMoves), f(c.insMoves),
			float64(c.delMoves+c.insMoves)/float64(c.delCmps+c.insCmps))
	}
	prefill, ps := workload(rng, 10_000, 10_000)
	row("sorted slice", 10_000, countSlice(prefill, ps))
	c, t, h0 := countBST(prefill, ps)
	row("BST, keys inserted in random order", 10_000, c)
	fmt.Printf("    that BST had height %d before the updates and %d after, against log2(n) = %.1f\n", h0, t.height(), math.Log2(10_000))

	prefill, ps = workload(rng, 5_000, 10_000)
	row("sorted slice", 5_000, countSlice(prefill, ps))
	cr, _, _ := countBST(prefill, ps)
	row("BST, keys inserted in random order", 5_000, cr)
	sorted := slices.Sorted(slices.Values(prefill))
	cs, ts, hs := countBST(sorted, ps)
	row("BST, keys inserted in sorted order", 5_000, cs)
	fmt.Printf("    the sorted-fill BST had height %d (a list) and %d after the updates; per update pair it spends %.0f comparisons;\n", hs, ts.height(), float64(cs.delCmps+cs.insCmps)/float64(cs.ops))
	cl := countSlice(prefill, ps)
	fmt.Printf("    the slice at the same n spends %.0f element moves plus a binary search\n", float64(cl.delMoves+cl.insMoves)/float64(cl.ops))
	fmt.Println("    Go has no ordered map to put in this table; slices.Sorted(maps.Keys(m)) would cost Θ(n log n) per ordered dump and is not run")

	// (b) One timing table per workload.
	sizes := []int{16, 64, 256, 1024, 4096, 16384, 65536}
	if big {
		sizes = append(sizes, 1_000_000)
	}
	names := []string{
		"update pair (delete a live key, insert a fresh one)",
		"predecessor query (largest key below a random q)",
		fmt.Sprintf("%d-key range scan (the %d keys from a random q)", scanLen, scanLen),
	}
	table := make([][]float64, 3) // per workload: flattened [size][slice, bst]
	verified := true
	for _, n := range sizes {
		r := rand.New(rand.NewPCG(uint64(n), 2))
		pairsN := max(20_000, 2_000_000/n)
		if n > 100_000 {
			pairsN = 5_000
		}
		prefill, ps := workload(r, n, pairsN)
		qs := make([]int, 200_000)
		for i := range qs {
			qs[i] = r.IntN(1 << 30)
		}
		scanQs := qs[:50_000]
		rs := []runner{sliceRunner(), bstRunner()}

		// Verify before timing: every structure answers the same.
		for _, r := range rs {
			r.build(prefill)
		}
		for _, q := range qs[:2_000] {
			if rs[0].pred([]int{q}) != rs[1].pred([]int{q}) || rs[0].scan([]int{q}) != rs[1].scan([]int{q}) {
				verified = false
			}
		}
		for _, r := range rs {
			r.upd(ps)
		}
		verified = verified && slices.Equal(rs[0].keys(), rs[1].keys()) && slices.IsSorted(rs[0].keys())

		for w, res := range [][]float64{
			bestOf(3, len(ps), rs, prefill, func(r runner) int { r.upd(ps); return 1 }),
			bestOf(3, len(qs), rs, prefill, func(r runner) int { return r.pred(qs) }),
			bestOf(3, len(scanQs), rs, prefill, func(r runner) int { return r.scan(scanQs) }),
		} {
			table[w] = append(table[w], res...)
		}
	}
	fmt.Printf("\n(b) ns per operation, best of 3 interleaved rounds (structures rebuilt outside the timer each round).\n")
	fmt.Printf("    Before timing, all structures gave the same predecessors and 50-key sums for 2,000 queries per size,\n")
	fmt.Printf("    and the same contents after the update pairs: %v\n", verified)
	for w, name := range names {
		fmt.Printf("\n  %s\n", name)
		fmt.Printf("  %9s %12s %12s   %s\n", "n", "slice", "BST", "faster")
		var wins []int
		for i, n := range sizes {
			a, b := table[w][2*i], table[w][2*i+1]
			who := fmt.Sprintf("slice %.1fx", b/a)
			if b < a {
				who = fmt.Sprintf("BST %.1fx", a/b)
				wins = append(wins, n)
			}
			fmt.Printf("  %9d %12.1f %12.1f   %s\n", n, a, b, who)
		}
		fmt.Printf("  the BST was faster at n = %v\n", wins)
	}
	_ = sink
}
```

**Sample output:**

```
(a) counts: n random keys, then 10,000 update pairs (delete a random live key, insert a fresh one)
    slice: a comparison is one probe of the binary search; a move is one element shifted by copy()
    BST:   a comparison is one node's key looked at; a BST update moves nothing
  structure                                 n cmp/del cmp/ins moves/del moves/ins moves per cmp
  sorted slice                          10000   13.36   13.37    5005.9    5002.2        374.4
  BST, keys inserted in random order    10000   15.84   16.73       0.0       0.0          0.0
    that BST had height 31 before the updates and 29 after, against log2(n) = 13.3
  sorted slice                           5000   12.36   12.36    2476.4    2482.8        200.6
  BST, keys inserted in random order     5000   15.33   16.22       0.0       0.0          0.0
  BST, keys inserted in sorted order     5000 1264.14 1256.80       0.0       0.0          0.0
    the sorted-fill BST had height 5000 (a list) and 1368 after the updates; per update pair it spends 2521 comparisons;
    the slice at the same n spends 4959 element moves plus a binary search
    Go has no ordered map to put in this table; slices.Sorted(maps.Keys(m)) would cost Θ(n log n) per ordered dump and is not run

(b) ns per operation, best of 3 interleaved rounds (structures rebuilt outside the timer each round).
    Before timing, all structures gave the same predecessors and 50-key sums for 2,000 queries per size,
    and the same contents after the update pairs: true

  update pair (delete a live key, insert a fresh one)
          n        slice          BST   faster
         16         34.5         44.3   slice 1.3x
         64         51.6         58.2   slice 1.1x
        256         80.4         72.4   BST 1.1x
       1024        151.6         87.9   BST 1.7x
       4096        367.7        117.5   BST 3.1x
      16384       1305.6        168.7   BST 7.7x
      65536       7511.6        230.6   BST 32.6x
  the BST was faster at n = [256 1024 4096 16384 65536]

  predecessor query (largest key below a random q)
          n        slice          BST   faster
         16         12.0         10.7   BST 1.1x
         64         16.4         15.3   BST 1.1x
        256         24.3         23.5   BST 1.0x
       1024         35.6         32.0   BST 1.1x
       4096         42.6         37.7   BST 1.1x
      16384         53.8         67.1   slice 1.2x
      65536         79.8         94.8   slice 1.2x
  the BST was faster at n = [16 64 256 1024 4096]

  50-key range scan (the 50 keys from a random q)
          n        slice          BST   faster
         16         13.9         21.0   slice 1.5x
         64         25.5         52.8   slice 2.1x
        256         38.4         93.1   slice 2.4x
       1024         48.4        118.9   slice 2.5x
       4096         55.8        147.4   slice 2.6x
      16384         66.2        240.1   slice 3.6x
      65536         94.9        445.1   slice 4.7x
  the BST was faster at n = []
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::Ordering;
use std::collections::{BTreeMap, HashSet};
use std::hint::black_box;
use std::time::Instant;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the same
// seeds give the same keys and update pairs, and every count in (a) matches Go's.
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

// ---------------------------------------------------------------------------
// The sorted slice. A probe is one comparison. Every update pays the binary
// search AND a memmove of everything behind the position. Vec::insert and
// Vec::remove do not return how many elements they moved; the std source
// (read, not run) does `ptr::copy(p, p.add(1), len - index)` for insert and
// `ptr::copy(ptr.add(1), ptr, len - index - 1)` for remove, so the move counts
// below are exactly those two lengths.
// ---------------------------------------------------------------------------

fn lower_bound(s: &[i64], key: i64) -> (usize, u64) {
    // first index with s[i] >= key, and the probes spent
    let (mut lo, mut hi, mut probes) = (0, s.len(), 0);
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        probes += 1;
        if s[mid] < key {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    (lo, probes)
}

fn slice_delete(s: &mut Vec<i64>, key: i64) -> (u64, usize) {
    let (pos, probes) = lower_bound(s, key);
    let moves = s.len() - pos - 1;
    s.remove(pos);
    (probes, moves)
}

fn slice_insert(s: &mut Vec<i64>, key: i64) -> (u64, usize) {
    let (pos, probes) = lower_bound(s, key);
    let moves = s.len() - pos;
    s.insert(pos, key);
    (probes, moves)
}

// ---------------------------------------------------------------------------
// The BST: cursor insert (example 2) and relinking delete (example 12). A
// comparison is one three-way compare against one node's key; both are
// iterative, so a degenerate tree cannot overflow the stack, and the Drop
// below is iterative for the same reason.
// ---------------------------------------------------------------------------

struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

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
struct Bst {
    root: Link,
}

impl Bst {
    fn insert(&mut self, key: i64) -> u64 {
        let mut cmps = 0;
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            cmps += 1;
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
        cmps
    }

    fn delete(&mut self, key: i64) -> u64 {
        let mut cmps = 0;
        let mut cur = &mut self.root;
        while cur.as_ref().is_some_and(|n| n.key != key) {
            cmps += 1;
            let n = cur.as_mut().unwrap(); // short-lived shared test, then a reborrow
            cur = if key < n.key {
                &mut n.left
            } else {
                &mut n.right
            };
        }
        let Some(mut node) = cur.take() else {
            return cmps;
        };
        cmps += 1; // the comparison that found it
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
                *succ = s.right.take();
                (s.left, s.right) = (left, right);
                Some(s)
            }
        };
        cmps
    }

    // pred: the largest key below `key`: one descent that remembers the last right turn.
    fn pred(&self, key: i64) -> Option<i64> {
        let (mut best, mut cur) = (None, self.root.as_deref());
        while let Some(n) = cur {
            if n.key < key {
                best = Some(n.key);
                cur = n.right.as_deref();
            } else {
                cur = n.left.as_deref();
            }
        }
        best
    }

    // scan sums the first `count` keys >= lo, in order, with an explicit stack that
    // the caller reuses (so no allocation is timed): seed it with the left spine of
    // the nodes that are >= lo, pop, then push the left spine of the right child.
    fn scan<'a>(&'a self, lo: i64, count: usize, stack: &mut Vec<&'a Node>) -> i64 {
        stack.clear();
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            if n.key >= lo {
                stack.push(n);
                cur = n.left.as_deref();
            } else {
                cur = n.right.as_deref();
            }
        }
        let mut sum = 0;
        for _ in 0..count {
            let Some(n) = stack.pop() else { break };
            sum += n.key;
            let mut m = n.right.as_deref();
            while let Some(x) = m {
                stack.push(x);
                m = x.left.as_deref();
            }
        }
        sum
    }

    fn keys(&self) -> Vec<i64> {
        // sorted contents, iteratively
        let (mut out, mut stack, mut cur) = (Vec::new(), Vec::new(), self.root.as_deref());
        while cur.is_some() || !stack.is_empty() {
            while let Some(n) = cur {
                stack.push(n);
                cur = n.left.as_deref();
            }
            let n: &Node = stack.pop().unwrap();
            out.push(n.key);
            cur = n.right.as_deref();
        }
        out
    }

    fn height(&self) -> usize {
        // nodes on the longest root-to-nil path, counted level by level
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

// ---------------------------------------------------------------------------
// A BTreeMap key that counts every comparison BTreeMap makes. BTreeMap only
// ever calls Ord::cmp, so the count is complete. (Rust has no mutable globals:
// a thread_local Cell is the counter.)
// ---------------------------------------------------------------------------

thread_local! {
    static CMPS: Cell<u64> = const { Cell::new(0) };
}

#[derive(Clone, Copy, PartialEq, Eq)]
struct K(i64);

impl Ord for K {
    fn cmp(&self, other: &Self) -> Ordering {
        CMPS.with(|c| c.set(c.get() + 1));
        self.0.cmp(&other.0)
    }
}

impl PartialOrd for K {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn cmps_of<R>(f: impl FnOnce() -> R) -> (u64, R) {
    let before = CMPS.with(Cell::get);
    let r = f();
    (CMPS.with(Cell::get) - before, r)
}

// ---------------------------------------------------------------------------
// The workload: n distinct random keys, then update PAIRS: delete a random live
// key, insert a fresh random key in its place. Same draws as the Go program.
// ---------------------------------------------------------------------------

struct Pair {
    del: i64,
    ins: i64,
}

fn workload(rng: &mut Pcg, n: usize, pairs: usize) -> (Vec<i64>, Vec<Pair>) {
    let mut set: HashSet<i64> = HashSet::with_capacity(n);
    let draw = |rng: &mut Pcg, set: &mut HashSet<i64>| loop {
        let k = rng.int_n(1 << 30) as i64;
        if set.insert(k) {
            return k;
        }
    };
    let mut live: Vec<i64> = Vec::with_capacity(n);
    while live.len() < n {
        live.push(draw(rng, &mut set));
    }
    let prefill = live.clone();
    let mut ps = Vec::with_capacity(pairs);
    for _ in 0..pairs {
        let i = rng.int_n(n as u64) as usize;
        let del = live[i];
        set.remove(&del);
        live[i] = draw(rng, &mut set);
        ps.push(Pair { del, ins: live[i] });
    }
    (prefill, ps)
}

// ---------------------------------------------------------------------------
// (a) counts
// ---------------------------------------------------------------------------

#[derive(Default)]
struct Counts {
    del_cmps: u64,
    ins_cmps: u64,
    del_moves: usize,
    ins_moves: usize,
    ops: usize,
}

fn count_slice(prefill: &[i64], ps: &[Pair]) -> Counts {
    let mut s = prefill.to_vec();
    s.sort_unstable();
    let mut c = Counts::default();
    for p in ps {
        let (pr, mv) = slice_delete(&mut s, p.del);
        (c.del_cmps, c.del_moves) = (c.del_cmps + pr, c.del_moves + mv);
        let (pr, mv) = slice_insert(&mut s, p.ins);
        (c.ins_cmps, c.ins_moves) = (c.ins_cmps + pr, c.ins_moves + mv);
    }
    c.ops = ps.len();
    c
}

// count_bst returns the counts, the final tree and the height before the updates.
fn count_bst(prefill: &[i64], ps: &[Pair]) -> (Counts, Bst, usize) {
    let mut t = Bst::default();
    for &k in prefill {
        t.insert(k);
    }
    let h = t.height();
    let mut c = Counts::default();
    for p in ps {
        c.del_cmps += t.delete(p.del);
        c.ins_cmps += t.insert(p.ins);
    }
    c.ops = ps.len();
    (c, t, h)
}

// The same workload on a BTreeMap whose keys count their comparisons.
fn count_btree(prefill: &[i64], ps: &[Pair]) -> Counts {
    let mut m: BTreeMap<K, ()> = BTreeMap::new();
    for &k in prefill {
        m.insert(K(k), ());
    }
    let mut c = Counts::default();
    for p in ps {
        c.del_cmps += cmps_of(|| m.remove(&K(p.del))).0;
        c.ins_cmps += cmps_of(|| m.insert(K(p.ins), ())).0;
    }
    c.ops = ps.len();
    c
}

fn row(name: &str, n: usize, c: &Counts, moves: bool) {
    let f = |x: u64| x as f64 / c.ops as f64;
    let g = |x: usize| x as f64 / c.ops as f64;
    if moves {
        println!(
            "  {name:<36} {n:>6} {:>7.2} {:>7.2} {:>9.1} {:>9.1} {:>12.1}",
            f(c.del_cmps),
            f(c.ins_cmps),
            g(c.del_moves),
            g(c.ins_moves),
            (c.del_moves + c.ins_moves) as f64 / (c.del_cmps + c.ins_cmps) as f64
        );
    } else {
        println!(
            "  {name:<36} {n:>6} {:>7.2} {:>7.2} {:>9} {:>9} {:>12}",
            f(c.del_cmps),
            f(c.ins_cmps),
            "n/a",
            "n/a",
            "n/a"
        );
    }
}

// ---------------------------------------------------------------------------
// (b) timing: each structure is a Runner; the timed loops live inside its
// methods, so the hot loops make direct calls. Inputs are rebuilt OUTSIDE the
// timer every round (the update workload mutates its structure; the queries
// do not).
// ---------------------------------------------------------------------------

const SCAN_LEN: usize = 50;

trait Runner {
    fn build(&mut self, prefill: &[i64]); // untimed: a fresh structure from the pristine keys
    fn upd(&mut self, ps: &[Pair]); // timed
    fn pred(&self, qs: &[i64]) -> i64; // timed: sum of (predecessor + 1, or 0)
    fn scan(&self, qs: &[i64]) -> i64; // timed: sum of the first 50 keys >= q
    fn keys(&self) -> Vec<i64>;
}

struct SliceRunner(Vec<i64>);
struct BstRunner(Bst);
struct BTreeRunner(BTreeMap<i64, ()>);

impl Runner for SliceRunner {
    fn build(&mut self, prefill: &[i64]) {
        self.0 = Vec::with_capacity(prefill.len() + 1);
        self.0.extend_from_slice(prefill);
        self.0.sort_unstable();
    }
    fn upd(&mut self, ps: &[Pair]) {
        for p in ps {
            slice_delete(&mut self.0, black_box(p.del));
            slice_insert(&mut self.0, black_box(p.ins));
        }
    }
    fn pred(&self, qs: &[i64]) -> i64 {
        let mut sum = 0;
        for &q in qs {
            let (pos, _) = lower_bound(&self.0, black_box(q));
            if pos > 0 {
                sum += self.0[pos - 1] + 1;
            }
        }
        sum
    }
    fn scan(&self, qs: &[i64]) -> i64 {
        let mut sum = 0;
        for &q in qs {
            let (pos, _) = lower_bound(&self.0, black_box(q));
            sum += self.0[pos..(pos + SCAN_LEN).min(self.0.len())]
                .iter()
                .sum::<i64>();
        }
        sum
    }
    fn keys(&self) -> Vec<i64> {
        self.0.clone()
    }
}

impl Runner for BstRunner {
    fn build(&mut self, prefill: &[i64]) {
        self.0 = Bst::default();
        for &k in prefill {
            self.0.insert(k);
        }
    }
    fn upd(&mut self, ps: &[Pair]) {
        for p in ps {
            self.0.delete(black_box(p.del));
            self.0.insert(black_box(p.ins));
        }
    }
    fn pred(&self, qs: &[i64]) -> i64 {
        qs.iter()
            .map(|&q| self.0.pred(black_box(q)).map_or(0, |k| k + 1))
            .sum()
    }
    fn scan(&self, qs: &[i64]) -> i64 {
        let mut stack = Vec::with_capacity(256);
        qs.iter()
            .map(|&q| self.0.scan(black_box(q), SCAN_LEN, &mut stack))
            .sum()
    }
    fn keys(&self) -> Vec<i64> {
        self.0.keys()
    }
}

impl Runner for BTreeRunner {
    fn build(&mut self, prefill: &[i64]) {
        // one insert at a time, in the drawn order, like the BST (collect() would
        // sort and bulk-build fully packed nodes, a different shape)
        self.0 = BTreeMap::new();
        for &k in prefill {
            self.0.insert(k, ());
        }
    }
    fn upd(&mut self, ps: &[Pair]) {
        for p in ps {
            self.0.remove(&black_box(p.del));
            self.0.insert(black_box(p.ins), ());
        }
    }
    fn pred(&self, qs: &[i64]) -> i64 {
        qs.iter()
            .map(|&q| {
                self.0
                    .range(..black_box(q))
                    .next_back()
                    .map_or(0, |(k, _)| k + 1)
            })
            .sum()
    }
    fn scan(&self, qs: &[i64]) -> i64 {
        qs.iter()
            .map(|&q| {
                self.0
                    .range(black_box(q)..)
                    .take(SCAN_LEN)
                    .map(|(k, _)| k)
                    .sum::<i64>()
            })
            .sum()
    }
    fn keys(&self) -> Vec<i64> {
        self.0.keys().copied().collect()
    }
}

// best_of runs `rounds` interleaved rounds: in each round every runner gets a
// freshly built structure and one timed pass. It returns the best ns/op per runner.
fn best_of(
    rounds: usize,
    ops: usize,
    rs: &mut [Box<dyn Runner>],
    prefill: &[i64],
    work: impl Fn(&mut dyn Runner) -> i64,
) -> Vec<f64> {
    let mut best = vec![f64::INFINITY; rs.len()];
    for _ in 0..rounds {
        for (i, r) in rs.iter_mut().enumerate() {
            r.build(prefill);
            let t0 = Instant::now();
            black_box(work(r.as_mut()));
            best[i] = best[i].min(t0.elapsed().as_nanos() as f64 / ops as f64);
        }
    }
    best
}

fn main() {
    let big = std::env::args().nth(1).as_deref() == Some("--big");
    let mut rng = Pcg::new(1, 2);

    // (a) Counts first: deterministic, and the reason the timings come out as they do.
    println!(
        "(a) counts: n random keys, then 10,000 update pairs (delete a random live key, insert a fresh one)"
    );
    println!(
        "    slice: a comparison is one probe of the binary search; a move is one element shifted (the memmove length)"
    );
    println!("    BST:   a comparison is one node's key looked at; a BST update moves nothing");
    println!(
        "    BTreeMap: a comparison is one call of Ord::cmp on a counting key; the keys it shifts inside a node are not counted"
    );
    println!(
        "  {:<36} {:>6} {:>7} {:>7} {:>9} {:>9} {:>12}",
        "structure", "n", "cmp/del", "cmp/ins", "moves/del", "moves/ins", "moves per cmp"
    );
    let (prefill, ps) = workload(&mut rng, 10_000, 10_000);
    row("sorted slice", 10_000, &count_slice(&prefill, &ps), true);
    let (c, t, h0) = count_bst(&prefill, &ps);
    row("BST, keys inserted in random order", 10_000, &c, true);
    println!(
        "    that BST had height {h0} before the updates and {} after, against log2(n) = {:.1}",
        t.height(),
        (10_000f64).log2()
    );
    row(
        "BTreeMap (a B-tree, not a BST)",
        10_000,
        &count_btree(&prefill, &ps),
        false,
    );

    let (prefill, ps) = workload(&mut rng, 5_000, 10_000);
    row("sorted slice", 5_000, &count_slice(&prefill, &ps), true);
    row(
        "BST, keys inserted in random order",
        5_000,
        &count_bst(&prefill, &ps).0,
        true,
    );
    let mut sorted = prefill.clone();
    sorted.sort_unstable();
    let (cs, ts, hs) = count_bst(&sorted, &ps);
    row("BST, keys inserted in sorted order", 5_000, &cs, true);
    row(
        "BTreeMap, keys inserted in random order",
        5_000,
        &count_btree(&prefill, &ps),
        false,
    );
    row(
        "BTreeMap, keys inserted in sorted order",
        5_000,
        &count_btree(&sorted, &ps),
        false,
    );
    println!(
        "    the sorted-fill BST had height {hs} (a list) and {} after the updates; per update pair it spends {:.0} comparisons;",
        ts.height(),
        (cs.del_cmps + cs.ins_cmps) as f64 / cs.ops as f64
    );
    let cl = count_slice(&prefill, &ps);
    println!(
        "    the slice at the same n spends {:.0} element moves plus a binary search",
        (cl.del_moves + cl.ins_moves) as f64 / cl.ops as f64
    );

    // (a2) Rust only: what a lookup costs a BTreeMap, counted, against the BST.
    println!(
        "\n(a2) comparisons per successful lookup of a random present key (20,000 lookups, keys inserted in random order):"
    );
    println!(
        "  {:>9} {:>9} {:>14} {:>9} {:>14} {:>9}",
        "n", "log2 n", "BST", "BST/log2", "BTreeMap", "BTM/log2"
    );
    for n in [1_000usize, 10_000, 100_000, 1_000_000] {
        let mut r = Pcg::new(n as u64, 2);
        let (keys, _) = workload(&mut r, n, 0);
        let mut bst = Bst::default();
        let mut map: BTreeMap<K, ()> = BTreeMap::new();
        for &k in &keys {
            bst.insert(k);
            map.insert(K(k), ());
        }
        let probes: Vec<i64> = (0..20_000)
            .map(|_| keys[r.int_n(n as u64) as usize])
            .collect();
        // BST comparisons for a hit = depth + 1: count them with a read-only descent
        let (mut bst_c, mut map_c) = (0u64, 0u64);
        for &q in &probes {
            let mut cur = bst.root.as_deref();
            while let Some(nd) = cur {
                bst_c += 1;
                match q.cmp(&nd.key) {
                    Ordering::Less => cur = nd.left.as_deref(),
                    Ordering::Greater => cur = nd.right.as_deref(),
                    Ordering::Equal => break,
                }
            }
            map_c += cmps_of(|| map.contains_key(&K(q))).0;
        }
        let (b, m, l2) = (
            bst_c as f64 / 20_000.0,
            map_c as f64 / 20_000.0,
            (n as f64).log2(),
        );
        println!(
            "  {n:>9} {l2:>9.1} {b:>14.2} {:>9.2} {m:>14.2} {:>9.2}",
            b / l2,
            m / l2
        );
    }
    println!(
        "  BTreeMap: B = 6, so a node holds 5 to 11 keys (node.rs: B = 6, CAPACITY = 2B - 1; read in the source, not run); lesson 19 builds the structure"
    );

    // (b) One timing table per workload.
    let mut sizes = vec![16usize, 64, 256, 1024, 4096, 16384, 65536];
    if big {
        sizes.push(1_000_000);
    }
    let names = [
        "update pair (delete a live key, insert a fresh one)".to_string(),
        "predecessor query (largest key below a random q)".to_string(),
        format!("{SCAN_LEN}-key range scan (the {SCAN_LEN} keys from a random q)"),
    ];
    let labels = ["slice", "BST", "BTreeMap"];
    let mut table: Vec<Vec<Vec<f64>>> = vec![Vec::new(); 3]; // [workload][size] -> ns per runner
    let mut verified = true;
    for &n in &sizes {
        let mut r = Pcg::new(n as u64, 2);
        let pairs_n = if n > 100_000 {
            5_000
        } else {
            20_000.max(2_000_000 / n)
        };
        let (prefill, ps) = workload(&mut r, n, pairs_n);
        let qs: Vec<i64> = (0..200_000).map(|_| r.int_n(1 << 30) as i64).collect();
        let scan_qs = &qs[..50_000];
        let mut rs: Vec<Box<dyn Runner>> = vec![
            Box::new(SliceRunner(Vec::new())),
            Box::new(BstRunner(Bst::default())),
            Box::new(BTreeRunner(BTreeMap::new())),
        ];

        // Verify before timing: every structure answers the same.
        for r in rs.iter_mut() {
            r.build(&prefill);
        }
        let mut sorted = prefill.clone();
        sorted.sort_unstable();
        for &q in &qs[..2_000] {
            let one = [q];
            verified &= rs.iter().all(|r| r.pred(&one) == rs[0].pred(&one));
            verified &= rs.iter().all(|r| r.scan(&one) == rs[0].scan(&one));
            // std's own binary search agrees with the hand-written lower bound
            verified &= lower_bound(&sorted, q).0 == sorted.partition_point(|&k| k < q);
        }
        for r in rs.iter_mut() {
            r.upd(&ps);
        }
        let first = rs[0].keys();
        verified &= first.is_sorted() && rs.iter().all(|r| r.keys() == first);

        let u = best_of(3, ps.len(), &mut rs, &prefill, |r| {
            r.upd(&ps);
            1
        });
        let p = best_of(3, qs.len(), &mut rs, &prefill, |r| r.pred(&qs));
        let s = best_of(3, scan_qs.len(), &mut rs, &prefill, |r| r.scan(scan_qs));
        for (w, res) in [u, p, s].into_iter().enumerate() {
            table[w].push(res);
        }
    }
    println!(
        "\n(b) ns per operation, best of 3 interleaved rounds (structures rebuilt outside the timer each round)."
    );
    println!(
        "    Before timing, all structures gave the same predecessors and 50-key sums for 2,000 queries per size,"
    );
    println!(
        "    and the same contents after the update pairs (and Vec's partition_point agreed with the hand-written lower bound): {verified}"
    );
    for (w, name) in names.iter().enumerate() {
        println!("\n  {name}");
        println!(
            "  {:>9} {:>12} {:>12} {:>12}   fastest (vs the runner-up)",
            "n", "slice", "BST", "BTreeMap"
        );
        let mut wins: Vec<Vec<usize>> = vec![Vec::new(); 3];
        for (i, &n) in sizes.iter().enumerate() {
            let t = &table[w][i];
            let fastest = (0..3).min_by(|&a, &b| t[a].total_cmp(&t[b])).unwrap();
            let second = (0..3)
                .filter(|&x| x != fastest)
                .map(|x| t[x])
                .fold(f64::INFINITY, f64::min);
            wins[fastest].push(n);
            println!(
                "  {n:>9} {:>12.1} {:>12.1} {:>12.1}   {} {:.1}x",
                t[0],
                t[1],
                t[2],
                labels[fastest],
                second / t[fastest]
            );
        }
        for (j, l) in labels.iter().enumerate() {
            println!("  {l} was fastest at n = {:?}", wins[j]);
        }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O main.rs -o main && ./main      # ./main --big adds n = 1,000,000
```

**Rust sample output:**

```
(a) counts: n random keys, then 10,000 update pairs (delete a random live key, insert a fresh one)
    slice: a comparison is one probe of the binary search; a move is one element shifted (the memmove length)
    BST:   a comparison is one node's key looked at; a BST update moves nothing
    BTreeMap: a comparison is one call of Ord::cmp on a counting key; the keys it shifts inside a node are not counted
  structure                                 n cmp/del cmp/ins moves/del moves/ins moves per cmp
  sorted slice                          10000   13.36   13.37    5005.9    5002.2        374.4
  BST, keys inserted in random order    10000   15.84   16.73       0.0       0.0          0.0
    that BST had height 31 before the updates and 29 after, against log2(n) = 13.3
  BTreeMap (a B-tree, not a BST)        10000   18.99   19.94       n/a       n/a          n/a
  sorted slice                           5000   12.36   12.36    2476.4    2482.8        200.6
  BST, keys inserted in random order     5000   15.33   16.22       0.0       0.0          0.0
  BST, keys inserted in sorted order     5000 1264.14 1256.80       0.0       0.0          0.0
  BTreeMap, keys inserted in random order   5000   17.92   18.77       n/a       n/a          n/a
  BTreeMap, keys inserted in sorted order   5000   17.74   18.66       n/a       n/a          n/a
    the sorted-fill BST had height 5000 (a list) and 1368 after the updates; per update pair it spends 2521 comparisons;
    the slice at the same n spends 4959 element moves plus a binary search

(a2) comparisons per successful lookup of a random present key (20,000 lookups, keys inserted in random order):
          n    log2 n            BST  BST/log2       BTreeMap  BTM/log2
       1000      10.0          12.43      1.25          13.48      1.35
      10000      13.3          15.99      1.20          18.89      1.42
     100000      16.6          20.70      1.25          24.69      1.49
    1000000      19.9          26.41      1.33          29.88      1.50
  BTreeMap: B = 6, so a node holds 5 to 11 keys (node.rs: B = 6, CAPACITY = 2B - 1; read in the source, not run); lesson 19 builds the structure

(b) ns per operation, best of 3 interleaved rounds (structures rebuilt outside the timer each round).
    Before timing, all structures gave the same predecessors and 50-key sums for 2,000 queries per size,
    and the same contents after the update pairs (and Vec's partition_point agreed with the hand-written lower bound): true

  update pair (delete a live key, insert a fresh one)
          n        slice          BST     BTreeMap   fastest (vs the runner-up)
         16         36.1         50.5         38.1   slice 1.1x
         64         47.8         66.8         58.0   slice 1.2x
        256         67.9         77.9         71.6   slice 1.1x
       1024        130.7         94.1         84.5   BTreeMap 1.1x
       4096        367.7        118.4         93.4   BTreeMap 1.3x
      16384       1354.7        170.9        125.2   BTreeMap 1.4x
      65536       7603.8        208.5        151.3   BTreeMap 1.4x
  slice was fastest at n = [16, 64, 256]
  BST was fastest at n = []
  BTreeMap was fastest at n = [1024, 4096, 16384, 65536]

  predecessor query (largest key below a random q)
          n        slice          BST     BTreeMap   fastest (vs the runner-up)
         16          2.4          7.7         13.6   slice 3.1x
         64          4.2         16.0         17.5   slice 3.8x
        256          6.0         25.3         24.2   slice 4.0x
       1024          8.2         36.2         31.1   slice 3.8x
       4096         10.7         45.6         35.7   slice 3.3x
      16384         13.7         66.5         47.8   slice 3.5x
      65536         31.3         89.9         60.0   slice 1.9x
  slice was fastest at n = [16, 64, 256, 1024, 4096, 16384, 65536]
  BST was fastest at n = []
  BTreeMap was fastest at n = []

  50-key range scan (the 50 keys from a random q)
          n        slice          BST     BTreeMap   fastest (vs the runner-up)
         16         14.6         16.0         18.8   slice 1.1x
         64         22.0         38.5         38.2   slice 1.7x
        256         13.2         72.4         62.9   slice 4.8x
       1024         13.3        102.1         78.4   slice 5.9x
       4096         15.7        134.9         95.4   slice 6.1x
      16384         19.2        267.6        128.7   slice 6.7x
      65536         41.5        434.5        158.4   slice 3.8x
  slice was fastest at n = [16, 64, 256, 1024, 4096, 16384, 65536]
  BST was fastest at n = []
  BTreeMap was fastest at n = []
```

> **Go vs Rust:** The counts for the slice and the BST in (a) are identical to Go's (diffed): same PCG draws, same probe loop, same cursor insert and relinking delete; the `BTreeMap` rows, (a2) and the third timing column are new. `Vec::insert` and `Vec::remove` do not report how many elements they shifted, so the move counts are the lengths the std source passes to `ptr::copy` (`len - index` for insert, `len - index - 1` for remove; read in `alloc/src/vec/mod.rs`, not run), where Go's `copy` returns the count.
> 
> `BTreeMap` is a B-tree, not a binary search tree: nodes hold 5 to 11 keys in an array (`B = 6`, `CAPACITY = 2B - 1` in `node.rs`; read, not run) and every leaf is at the same depth. Counted with a key whose `Ord::cmp` increments a thread-local counter (it spent 19.0 and 19.9 comparisons per delete and insert at n = 10,000 against the BST's 15.8 and 16.7), and measured in (a2) at 1.35 to 1.50 times log₂ n per lookup (1.50 at n = 10⁶, so the ~1.5 hypothesis held) against the BST's 1.20 to 1.33: the docs' remark that a B-tree "will have to do more comparisons" shows in the counts, so whatever it wins is not comparisons. Fed sorted keys it still spends 17.7 and 18.7 per operation where the BST spent about 1,260: a B-tree cannot degenerate (lesson 19).
> 
> Timings (this run and three more, ns per operation; every verdict below held in all four): **updates** — the slice is fastest at n ≤ 256 (at n = 16 and 256 by only 2–8% over `BTreeMap`), `BTreeMap` from n = 1,024 on, 1.1x ahead of the BST there and 1.3–1.4x at 65,536, where a slice update costs about 7.5 µs, the BST about 0.2 µs and `BTreeMap` about 0.15 µs; the BST is never fastest in Rust but beats the slice from n = 1,024 (1.4x), where Go's tree beat Go's slice from n = 256. **Predecessor and scan** — the slice is fastest at every size: by 1.9–4.3x for predecessor (2.5 ns at n = 16, 31 ns at 65,536), and for the 50-key scan by 1.1x at n = 16, 1.7–1.8x at 64, 4.6–6.7x from 256 to 16,384 and 3.2–3.8x at 65,536. That is not Go's verdict: Go's slice and tree were within about 1.4x of each other on predecessor, because Rust's slice search is 2.6–5x faster than Go's (2.5 against about 12 ns at n = 16, 31 against about 85 ns at 65,536) while its tree is within 1.3x of Go's tree at every size except n = 16, where it is 1.4x faster. A branch-free binary search and a vectorised 50-element sum are plausible mechanisms; neither was measured. One `--big` run (not in the output above) at n = 1,000,000: `BTreeMap` 216 ns per update pair, the BST 563 ns, the slice 124 µs; the slice was still fastest on predecessor (45 ns against `BTreeMap`'s 111 and the BST's 418) and on scans.

**Complexity:** slice update Θ(log n) comparisons plus Θ(n) element moves: at n = 10,000 about 13 probes and about 5,000 moves per operation, roughly 370 moves per comparison (2,500 moves at n = 5,000); BST update Θ(h) comparisons and no moves: about 16–17 per operation at n = 10,000 (height 31 against log₂ n = 13.3), but about 1,260 per operation when the n = 5,000 keys were inserted in sorted order (height 5,000 before the updates, 1,368 after). Timings (four full runs; ns per operation, so read them as ratios and ranges): **updates** — slice faster at n = 16 and 64 (1.1–1.3x), tree faster from n = 256 (1.1–1.2x), 1.6–1.7x at 1,024, about 3x at 4,096, 7–8x at 16,384 and about 30x at 65,536 (the slice paid about 7.5 µs per pair there, the tree about 0.23 µs); **predecessor** — within 1.0–1.4x either way at every size: the tree ahead by 1.0–1.2x up to n = 1,024, within 10% either way at 4,096, the slice ahead by 1.1–1.4x at 16,384 and 65,536; **50-key scan** — the slice faster at every size, 1.4–2.6x up to n = 1,024 and 3.5–5x from 16,384 on. So on this machine the tree overtakes the slice on updates between n = 64 and 256, and it does not overtake it on ordered scans at any size. `go run . --big` adds n = 1,000,000 (one run: the slice paid about 120 µs per update pair against about 0.5 µs, and was about 10x faster on scans).

---

## 15. The problems BSTs own: recover, closest, count smaller

`🔴 hard` · *Problem set*

Three classic problems where the ordered structure does the thinking, each checked against a brute-force oracle on seeded random input. **Recover** a BST in which two keys were swapped: one in-order pass sees one descent (`prev > cur`) if the keys were in-order neighbours and two otherwise, and the first `prev` and the last `cur` are the culprits, in Θ(n) time and Θ(h) space. **Closest** key to a target: one descent whose path always contains the floor and the ceiling. **Count smaller after self**: insert from the right into a tree that stores a subtree size and a copy count, and read the answer off the same descent that does the insert, which is Θ(n log n) on random input and Θ(n²) on sorted input, the failure mode of example 6 inside an algorithm; a merge-sort version is the Θ(n log n) guarantee.

**Steps:**

1. Run the recover pass on a 7-node tree twice (a far-apart swap, then neighbours), then on 2,000 random trees, and read the failure count and how many cases had one descent and how many two.
2. Check the closest-key descent against a scan on 10,000 targets, on keys chosen so that exact ties are common, and note the tie rule (the smaller key wins).
3. Count smaller-after-self three ways (brute force, BST, merge sort), confirm they agree including on duplicates, then read the comparison table: random, sorted and reversed input against `n log2 n` and `n(n-1)/2`.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
)

// One node type for all three problems. key is what the tree orders by; cnt is
// how many copies of the key were inserted (always 1 in (a) and (b), where the
// keys are distinct); size counts the elements in the subtree, copies included.
type node struct {
	key, cnt, size int
	left, right    *node
}

type bst struct {
	root *node
	cmps int // three-way comparisons made by add
}

func size(n *node) int {
	if n == nil {
		return 0
	}
	return n.size
}

// add inserts key and, in the SAME descent, counts the elements already in the
// tree that are strictly smaller. Every time the descent goes right, the node's
// copies and its whole left subtree are smaller than key. An equal key stops the
// descent and bumps cnt. The loop is iterative: it must survive a 4,000-node path.
func (t *bst) add(key int) (smaller int) {
	link := &t.root
	for *link != nil {
		n := *link
		n.size++ // the new element lands somewhere below n
		t.cmps++
		switch c := cmp.Compare(key, n.key); {
		case c < 0:
			link = &n.left
		case c > 0:
			smaller += size(n.left) + n.cnt
			link = &n.right
		default:
			n.cnt++
			return smaller + size(n.left)
		}
	}
	*link = &node{key: key, cnt: 1, size: 1}
	return smaller
}

// inorder walks left, node, right with an explicit stack and reports how deep
// the stack got.
func (t *bst) inorder(visit func(*node)) (maxStack int) {
	var stack []*node
	for n := t.root; n != nil || len(stack) > 0; {
		for ; n != nil; n = n.left {
			stack = append(stack, n)
		}
		maxStack = max(maxStack, len(stack))
		n = stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		visit(n)
		n = n.right
	}
	return
}

func (t *bst) nodes() (out []*node) {
	t.inorder(func(n *node) { out = append(out, n) })
	return
}

func (t *bst) keys() (out []int) {
	t.inorder(func(n *node) { out = append(out, n.key) })
	return
}

func height(n *node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.left), height(n.right))
}

// ser writes shape AND keys: "((. 1 .) 2 (. 3 .))".
func ser(n *node) string {
	if n == nil {
		return "."
	}
	return fmt.Sprintf("(%s %d %s)", ser(n.left), n.key, ser(n.right))
}

// distinct returns n distinct keys from [0, keyRange), in random order.
func distinct(rng *rand.Rand, n, keyRange int) []int {
	return shuffled(rng, keyRange)[:n]
}

func shuffled(rng *rand.Rand, n int) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = i
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates, written out so both languages draw the same numbers
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
	return a
}

// ------------------------------------------------------------ (a) recover

// findSwapped is ONE in-order pass. A descent is a place where prev > cur, which
// a sorted sequence never has. Swapping two keys that were neighbours in order
// makes ONE descent; swapping two that were not makes TWO. Either way the first
// descent's prev and the last descent's cur are the two wrong nodes.
func (t *bst) findSwapped() (first, second *node, descents, maxStack int) {
	var prev *node
	maxStack = t.inorder(func(cur *node) {
		if prev != nil && prev.key > cur.key {
			descents++
			if first == nil {
				first = prev
			}
			second = cur
		}
		prev = cur
	})
	return
}

func descentPairs(seq []int) (parts []string) {
	for i := 1; i < len(seq); i++ {
		if seq[i-1] > seq[i] {
			parts = append(parts, fmt.Sprintf("(%d > %d)", seq[i-1], seq[i]))
		}
	}
	return
}

func recoverDemo() {
	for _, c := range []struct{ i, j int }{{1, 5}, {2, 3}} {
		t := &bst{}
		for _, k := range []int{4, 2, 6, 1, 3, 5, 7} {
			t.add(k)
		}
		nodes := t.nodes()
		nodes[c.i].key, nodes[c.j].key = nodes[c.j].key, nodes[c.i].key
		seq := t.keys()
		first, second, d, _ := t.findSwapped()
		fmt.Printf("  swap the keys at in-order positions %d and %d: in-order %v\n", c.i, c.j, seq)
		fmt.Printf("    descents %s: %d, so first = %d and second = %d\n", strings.Join(descentPairs(seq), " "), d, first.key, second.key)
		first.key, second.key = second.key, first.key
		fmt.Printf("    swapped back: in-order %v\n", t.keys())
	}
}

func recoverTrials(rng *rand.Rand) {
	const trials = 2000
	failures, one, two, maxStack, maxHeight := 0, 0, 0, 0, 0
	for range trials {
		n := 2 + rng.IntN(39)
		t := &bst{}
		for _, k := range distinct(rng, n, 1000) {
			t.add(k)
		}
		want := ser(t.root)
		i := rng.IntN(n)
		j := rng.IntN(n - 1)
		if j >= i {
			j++
		}
		nodes := t.nodes()
		nodes[i].key, nodes[j].key = nodes[j].key, nodes[i].key
		wantDescents := 2
		if i-j == 1 || j-i == 1 {
			wantDescents = 1
		}
		first, second, d, stack := t.findSwapped()
		if d != wantDescents {
			failures++
			continue
		}
		first.key, second.key = second.key, first.key
		if ser(t.root) != want {
			failures++
		}
		one += btoi(d == 1)
		two += btoi(d == 2)
		maxStack, maxHeight = max(maxStack, stack), max(maxHeight, height(t.root))
	}
	fmt.Printf("  %d random trees (n = 2..40), two random keys swapped, one in-order pass, swapped back:\n", trials)
	fmt.Printf("  %d failures (the repaired tree equals the original, shape and keys)\n", failures)
	fmt.Printf("  1 descent in %d cases (the keys were in-order neighbours), 2 descents in %d\n", one, two)
	fmt.Printf("  deepest in-order stack = %d, tallest tree = %d: one pass visits each node once and holds one root-to-node path\n", maxStack, maxHeight)
}

func btoi(b bool) int {
	if b {
		return 1
	}
	return 0
}

// ------------------------------------------------------------ (b) closest

func abs(x int) int { return max(x, -x) }

// closest is one descent. Every node on the search path is a candidate (the
// path contains both the floor and the ceiling of target, and the closest key
// is one of those two). Ties go to the SMALLER key.
func (t *bst) closest(target int) (best, visited int) {
	bestD := math.MaxInt
	for n := t.root; n != nil; {
		visited++
		if d := abs(n.key - target); d < bestD || d == bestD && n.key < best {
			best, bestD = n.key, d
		}
		switch {
		case target < n.key:
			n = n.left
		case target > n.key:
			n = n.right
		default:
			return
		}
	}
	return
}

// closestBrute scans every key. tied is how many keys share the winning distance.
func closestBrute(keys []int, target int) (best, tied int) {
	bestD := math.MaxInt
	for _, k := range keys {
		switch d := abs(k - target); {
		case d < bestD:
			best, bestD, tied = k, d, 1
		case d == bestD:
			best, tied = min(best, k), tied+1
		}
	}
	return
}

func closestTrials(rng *rand.Rand) {
	t := &bst{}
	for _, k := range []int{8, 4, 12, 2, 6, 10, 14} {
		t.add(k)
	}
	for _, target := range []int{7, 11, -5, 100, 6} {
		best, _ := t.closest(target)
		fmt.Printf("  closest(%d) in {2 4 6 8 10 12 14} = %d\n", target, best)
	}

	// even keys only: two neighbouring keys are an even distance apart, so the
	// target in the middle of their gap is an exact tie
	t = &bst{}
	keys := distinct(rng, 300, 1000)
	for i, k := range keys {
		keys[i] = 2 * k
		t.add(2 * k)
	}
	lo, hi := slices.Min(keys), slices.Max(keys)
	const trials = 10000
	mismatches, ties, outside, maxVisited := 0, 0, 0, 0
	for range trials {
		target := rng.IntN(2100) - 50
		got, visited := t.closest(target)
		want, tied := closestBrute(keys, target)
		mismatches += btoi(got != want)
		ties += btoi(tied > 1)
		outside += btoi(target < lo || target > hi)
		maxVisited = max(maxVisited, visited)
	}
	fmt.Printf("  %d random targets in [-50, 2050) against %d random even keys (%d..%d):\n", trials, len(keys), lo, hi)
	fmt.Printf("  %d mismatches with the brute-force scan; %d targets were exact ties (smaller key wins), %d lay outside the keys\n", mismatches, ties, outside)
	fmt.Printf("  the most nodes one query visited = %d, tree height = %d (visited <= height: %v): one descent, Theta(h)\n",
		maxVisited, height(t.root), maxVisited <= height(t.root))
}

// ------------------------------------------------------------ (c) count smaller

// countsBrute: the definition. n(n-1)/2 comparisons, always.
func countsBrute(a []int) (out []int, cmps int) {
	out = make([]int, len(a))
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			cmps++
			out[i] += btoi(a[j] < a[i])
		}
	}
	return
}

// countsBST goes right to left: before a[i] is inserted, the tree holds exactly
// a[i+1:], so the "smaller" count of the insert descent IS the answer.
func countsBST(a []int) (out []int, cmps int) {
	t := &bst{}
	out = make([]int, len(a))
	for i := len(a) - 1; i >= 0; i-- {
		out[i] = t.add(a[i])
	}
	return out, t.cmps
}

// countsMerge sorts indices by value. When an element of the left half is
// emitted, the right-half elements emitted before it are exactly the ones
// strictly smaller (a right element goes first only if it is strictly smaller,
// so equal keys never count). Stable, and n log n comparisons whatever the input.
func countsMerge(a []int) (out []int, cmps int) {
	idx := make([]int, len(a))
	for i := range idx {
		idx[i] = i
	}
	tmp := make([]int, len(a))
	out = make([]int, len(a))
	var sortRange func(lo, hi int)
	sortRange = func(lo, hi int) {
		if hi-lo < 2 {
			return
		}
		mid := (lo + hi) / 2
		sortRange(lo, mid)
		sortRange(mid, hi)
		i, j, k := lo, mid, lo
		for i < mid {
			if j < hi {
				cmps++
				if a[idx[j]] < a[idx[i]] {
					tmp[k] = idx[j]
					j++
					k++
					continue
				}
			}
			out[idx[i]] += j - mid
			tmp[k] = idx[i]
			i++
			k++
		}
		copy(tmp[k:hi], idx[j:hi]) // left half exhausted: the rest of the right half
		copy(idx[lo:hi], tmp[lo:hi])
	}
	sortRange(0, len(a))
	return
}

func allEqual(a []int) bool {
	b, _ := countsBrute(a)
	c, _ := countsBST(a)
	d, _ := countsMerge(a)
	return slices.Equal(b, c) && slices.Equal(b, d)
}

func countSmaller(rng *rand.Rand) {
	small := []int{3, 3, 1, 3, 2}
	b, _ := countsBrute(small)
	t, _ := countsBST(small)
	m, _ := countsMerge(small)
	fmt.Printf("  %v: brute %v, tree %v, merge sort %v\n", small, b, t, m)

	narrow := make([]int, 200)
	for i := range narrow {
		narrow[i] = rng.IntN(20)
	}
	fmt.Printf("  n = 200, values in [0, 20) (about 10 copies of each value): all three agree = %v\n", allEqual(narrow))

	arrays, positions, bad := 1000, 0, 0
	for trial := range arrays {
		width := []int{8, 64, 1 << 30}[trial%3]
		a := make([]int, rng.IntN(100))
		for i := range a {
			a[i] = rng.IntN(width)
		}
		positions += len(a)
		bad += btoi(!allEqual(a))
	}
	fmt.Printf("  %d random arrays (n < 100, value ranges 8, 64 and 2^30): %d positions, %d arrays where the methods disagree\n", arrays, positions, bad)

	// The array is a shuffled 0..n-1 ("random"), 0..n-1 ("sorted") or n-1..0
	// ("reversed"). The BST goes right to left, so a sorted array inserts
	// DEscending keys (a left path) and a reversed one ascending keys (a right path).
	fmt.Println("\n  comparisons to count smaller-after-self (merge sort and Knuth are for the random array):")
	fmt.Printf("  %6s %11s %11s %11s %11s | %9s %9s %10s | %s\n", "n", "BST random", "BST sorted", "BST revers.", "merge sort", "n log2 n", "Knuth", "n(n-1)/2", "random / n log2 n")
	pathsCost, agree := true, true
	for _, n := range []int{1000, 2000, 4000} {
		rnd := shuffled(rng, n)
		asc, desc := make([]int, n), make([]int, n)
		for i := range asc {
			asc[i], desc[i] = i, n-1-i
		}
		_, cr := countsBST(rnd)
		_, ca := countsBST(asc)
		_, cd := countsBST(desc)
		_, cm := countsMerge(rnd)
		_, cb := countsBrute(rnd)
		h := 0.0
		for i := 1; i <= n; i++ {
			h += 1 / float64(i)
		}
		fn, pairs := float64(n), n*(n-1)/2
		fmt.Printf("  %6d %11d %11d %11d %11d | %9.0f %9.0f %10d | %17.2f\n", n, cr, ca, cd, cm, fn*math.Log2(fn), 2*(fn+1)*h-4*fn, pairs, float64(cr)/(fn*math.Log2(fn)))
		pathsCost = pathsCost && ca == pairs && cd == pairs && cb == pairs
		agree = agree && allEqual(rnd) && allEqual(asc) && allEqual(desc)
	}
	fmt.Printf("  sorted, reversed and brute force cost exactly n(n-1)/2 at every n: %v; the three methods agree on all nine arrays: %v\n", pathsCost, agree)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	fmt.Println("(a) recover a BST with two keys swapped")
	recoverDemo()
	recoverTrials(rng)
	fmt.Println("\n(b) the closest key to a target")
	closestTrials(rng)
	fmt.Println("\n(c) count smaller elements after self")
	countSmaller(rng)
}
```

**Output:**

```
(a) recover a BST with two keys swapped
  swap the keys at in-order positions 1 and 5: in-order [1 6 3 4 5 2 7]
    descents (6 > 3) (5 > 2): 2, so first = 6 and second = 2
    swapped back: in-order [1 2 3 4 5 6 7]
  swap the keys at in-order positions 2 and 3: in-order [1 2 4 3 5 6 7]
    descents (4 > 3): 1, so first = 4 and second = 3
    swapped back: in-order [1 2 3 4 5 6 7]
  2000 random trees (n = 2..40), two random keys swapped, one in-order pass, swapped back:
  0 failures (the repaired tree equals the original, shape and keys)
  1 descent in 335 cases (the keys were in-order neighbours), 2 descents in 1665
  deepest in-order stack = 11, tallest tree = 14: one pass visits each node once and holds one root-to-node path

(b) the closest key to a target
  closest(7) in {2 4 6 8 10 12 14} = 6
  closest(11) in {2 4 6 8 10 12 14} = 10
  closest(-5) in {2 4 6 8 10 12 14} = 2
  closest(100) in {2 4 6 8 10 12 14} = 14
  closest(6) in {2 4 6 8 10 12 14} = 6
  10000 random targets in [-50, 2050) against 300 random even keys (10..1998):
  0 mismatches with the brute-force scan; 1383 targets were exact ties (smaller key wins), 527 lay outside the keys
  the most nodes one query visited = 18, tree height = 18 (visited <= height: true): one descent, Theta(h)

(c) count smaller elements after self
  [3 3 1 3 2]: brute [2 2 0 1 0], tree [2 2 0 1 0], merge sort [2 2 0 1 0]
  n = 200, values in [0, 20) (about 10 copies of each value): all three agree = true
  1000 random arrays (n < 100, value ranges 8, 64 and 2^30): 47740 positions, 0 arrays where the methods disagree

  comparisons to count smaller-after-self (merge sort and Knuth are for the random array):
       n  BST random  BST sorted BST revers.  merge sort |  n log2 n     Knuth   n(n-1)/2 | random / n log2 n
    1000       10570      499500      499500        8725 |      9966     10986     499500 |              1.06
    2000       24696     1999000     1999000       19429 |     21932     24730    1999000 |              1.13
    4000       55507     7998000     7998000       42925 |     47863     54989    7998000 |              1.16
  sorted, reversed and brute force cost exactly n(n-1)/2 at every n: true; the three methods agree on all nine arrays: true
```

**Rust:**

```rust
use std::cmp::Ordering;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN (the same
// port as lesson 01, example 14), so both programs draw the same numbers.
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

// One node type for all three problems. key is what the tree orders by; cnt is
// how many copies of the key were inserted (always 1 in (a) and (b), where the
// keys are distinct); size counts the elements in the subtree, copies included.
// A link is Option<Box<Node>>: None is the empty tree, Go's nil.
struct Node {
    key: i64,
    cnt: usize,
    size: usize,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Rust's default drop glue recurses once per level. The 4,000-node paths below
// are fine on an 8 MB stack, but this Drop (the one from example 1) hands the
// children to a heap-allocated stack instead, so no tree here can overflow.
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
struct Bst {
    root: Link,
    cmps: usize, // three-way comparisons made by add
}

fn size(n: &Link) -> usize {
    n.as_ref().map_or(0, |n| n.size)
}

impl Bst {
    // add inserts key and, in the SAME descent, counts the elements already in
    // the tree that are strictly smaller. Every time the descent goes right, the
    // node's copies and its whole left subtree are smaller than key. An equal key
    // stops the descent and bumps cnt. The loop is iterative: it must survive a
    // 4,000-node path. `cur` is a `&mut Option<Box<Node>>`, Go's `link := &t.root`.
    fn add(&mut self, key: i64) -> usize {
        let mut smaller = 0;
        let mut cur = &mut self.root;
        while let Some(n) = cur {
            n.size += 1; // the new element lands somewhere below n
            self.cmps += 1;
            match key.cmp(&n.key) {
                Ordering::Less => cur = &mut n.left,
                Ordering::Greater => {
                    smaller += size(&n.left) + n.cnt;
                    cur = &mut n.right;
                }
                Ordering::Equal => {
                    n.cnt += 1;
                    return smaller + size(&n.left);
                }
            }
        }
        *cur = Some(Box::new(Node {
            key,
            cnt: 1,
            size: 1,
            left: None,
            right: None,
        }));
        smaller
    }

    // inorder walks left, node, right with an explicit stack and reports how
    // deep the stack got. The visitor gets a shared reference, so it can read
    // keys but not change them.
    fn inorder(&self, mut visit: impl FnMut(&Node)) -> usize {
        let mut stack: Vec<&Node> = Vec::new();
        let mut max_stack = 0;
        let mut cur = self.root.as_deref();
        while cur.is_some() || !stack.is_empty() {
            while let Some(n) = cur {
                stack.push(n);
                cur = n.left.as_deref();
            }
            max_stack = max_stack.max(stack.len());
            let n = stack.pop().unwrap();
            visit(n);
            cur = n.right.as_deref();
        }
        max_stack
    }

    fn keys(&self) -> Vec<i64> {
        let mut out = Vec::new();
        self.inorder(|n| out.push(n.key));
        out
    }

    // Rust cannot hold two `&mut Node` into one tree, so Go's "swap the two
    // nodes' keys" becomes "swap two key VALUES": one walk that rewrites every
    // node whose key is a or b. (It never searches, so it works on a tree that
    // is no longer a BST.) `Vec<&mut Node>` holds one live borrow per pending node.
    fn swap_values(&mut self, a: i64, b: i64) {
        let mut stack: Vec<&mut Node> = self.root.as_deref_mut().into_iter().collect();
        while let Some(n) = stack.pop() {
            if n.key == a {
                n.key = b;
            } else if n.key == b {
                n.key = a;
            }
            stack.extend(n.left.as_deref_mut());
            stack.extend(n.right.as_deref_mut());
        }
    }

    // find_swapped is ONE in-order pass. A descent is a place where prev > cur,
    // which a sorted sequence never has. Swapping two keys that were neighbours
    // in order makes ONE descent; swapping two that were not makes TWO. Either
    // way the first descent's prev and the last descent's cur are the two wrong
    // keys. It returns KEYS (a second pass swaps them by value), not nodes.
    fn find_swapped(&self) -> (Option<(i64, i64)>, usize, usize) {
        let (mut prev, mut first, mut second) = (None::<i64>, None, None);
        let mut descents = 0;
        let max_stack = self.inorder(|cur| {
            if let Some(p) = prev.filter(|&p| p > cur.key) {
                descents += 1;
                first = first.or(Some(p));
                second = Some(cur.key);
            }
            prev = Some(cur.key);
        });
        (first.zip(second), descents, max_stack)
    }

    // closest is one descent. Every node on the search path is a candidate (the
    // path contains both the floor and the ceiling of target, and the closest key
    // is one of those two). Ties go to the SMALLER key.
    fn closest(&self, target: i64) -> (Option<i64>, usize) {
        let (mut best, mut best_d, mut visited) = (None, i64::MAX, 0);
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            visited += 1;
            let d = (n.key - target).abs();
            if d < best_d || d == best_d && Some(n.key) < best {
                best = Some(n.key);
                best_d = d;
            }
            cur = match target.cmp(&n.key) {
                Ordering::Less => n.left.as_deref(),
                Ordering::Greater => n.right.as_deref(),
                Ordering::Equal => break,
            };
        }
        (best, visited)
    }
}

fn height(n: &Link) -> usize {
    n.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

// ser writes shape AND keys: "((. 1 .) 2 (. 3 .))".
fn ser(n: &Link) -> String {
    match n {
        None => ".".to_string(),
        Some(n) => format!("({} {} {})", ser(&n.left), n.key, ser(&n.right)),
    }
}

// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
// so the output reads the same in both languages.
fn go_slice<T: ToString>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn shuffled(rng: &mut Pcg, n: usize) -> Vec<usize> {
    let mut a: Vec<usize> = (0..n).collect();
    for i in (1..n).rev() {
        // Fisher-Yates, written out so both languages draw the same numbers
        let j = rng.int_n(i + 1);
        a.swap(i, j);
    }
    a
}

// distinct returns n distinct keys from [0, key_range), in random order.
fn distinct(rng: &mut Pcg, n: usize, key_range: usize) -> Vec<i64> {
    shuffled(rng, key_range)[..n]
        .iter()
        .map(|&k| k as i64)
        .collect()
}

// ------------------------------------------------------------ (a) recover

fn descent_pairs(seq: &[i64]) -> Vec<String> {
    seq.windows(2)
        .filter(|w| w[0] > w[1])
        .map(|w| format!("({} > {})", w[0], w[1]))
        .collect()
}

fn recover_demo() {
    for (i, j) in [(1, 5), (2, 3)] {
        let mut t = Bst::default();
        for k in [4, 2, 6, 1, 3, 5, 7] {
            t.add(k);
        }
        let seq = t.keys();
        t.swap_values(seq[i], seq[j]);
        let seq = t.keys();
        let (pair, d, _) = t.find_swapped();
        let (first, second) = pair.unwrap();
        println!(
            "  swap the keys at in-order positions {i} and {j}: in-order {}",
            go_slice(&seq)
        );
        println!(
            "    descents {}: {d}, so first = {first} and second = {second}",
            descent_pairs(&seq).join(" ")
        );
        t.swap_values(first, second);
        println!("    swapped back: in-order {}", go_slice(&t.keys()));
    }
}

fn recover_trials(rng: &mut Pcg) {
    const TRIALS: usize = 2000;
    let (mut failures, mut one, mut two, mut max_stack, mut max_height) = (0, 0, 0, 0, 0);
    for _ in 0..TRIALS {
        let n = 2 + rng.int_n(39);
        let mut t = Bst::default();
        for k in distinct(rng, n, 1000) {
            t.add(k);
        }
        let want = ser(&t.root);
        let i = rng.int_n(n);
        let mut j = rng.int_n(n - 1);
        if j >= i {
            j += 1;
        }
        let seq = t.keys();
        t.swap_values(seq[i], seq[j]);
        let want_descents = if i.abs_diff(j) == 1 { 1 } else { 2 };
        let (pair, d, stack) = t.find_swapped();
        let Some((first, second)) = pair.filter(|_| d == want_descents) else {
            failures += 1;
            continue;
        };
        t.swap_values(first, second);
        if ser(&t.root) != want {
            failures += 1;
        }
        if d == 1 {
            one += 1;
        } else {
            two += 1;
        }
        max_stack = max_stack.max(stack);
        max_height = max_height.max(height(&t.root));
    }
    println!(
        "  {TRIALS} random trees (n = 2..40), two random keys swapped, one in-order pass, swapped back:"
    );
    println!("  {failures} failures (the repaired tree equals the original, shape and keys)");
    println!("  1 descent in {one} cases (the keys were in-order neighbours), 2 descents in {two}");
    println!(
        "  deepest in-order stack = {max_stack}, tallest tree = {max_height}: one pass visits each node once and holds one root-to-node path"
    );
}

// ------------------------------------------------------------ (b) closest

// closest_brute scans every key. tied is how many keys share the winning distance.
fn closest_brute(keys: &[i64], target: i64) -> (i64, usize) {
    let (mut best, mut best_d, mut tied) = (0, i64::MAX, 0);
    for &k in keys {
        let d = (k - target).abs();
        if d < best_d {
            (best, best_d, tied) = (k, d, 1);
        } else if d == best_d {
            (best, tied) = (best.min(k), tied + 1);
        }
    }
    (best, tied)
}

fn closest_trials(rng: &mut Pcg) {
    let mut t = Bst::default();
    for k in [8, 4, 12, 2, 6, 10, 14] {
        t.add(k);
    }
    for target in [7, 11, -5, 100, 6] {
        let (best, _) = t.closest(target);
        println!(
            "  closest({target}) in {{2 4 6 8 10 12 14}} = {}",
            best.unwrap()
        );
    }

    // even keys only: two neighbouring keys are an even distance apart, so the
    // target in the middle of their gap is an exact tie
    let mut t = Bst::default();
    let keys: Vec<i64> = distinct(rng, 300, 1000).iter().map(|k| 2 * k).collect();
    for &k in &keys {
        t.add(k);
    }
    let (lo, hi) = (*keys.iter().min().unwrap(), *keys.iter().max().unwrap());
    const TRIALS: usize = 10000;
    let (mut mismatches, mut ties, mut outside, mut max_visited) = (0, 0, 0, 0);
    for _ in 0..TRIALS {
        let target = rng.int_n(2100) as i64 - 50;
        let (got, visited) = t.closest(target);
        let (want, tied) = closest_brute(&keys, target);
        mismatches += usize::from(got != Some(want));
        ties += usize::from(tied > 1);
        outside += usize::from(target < lo || target > hi);
        max_visited = max_visited.max(visited);
    }
    println!(
        "  {TRIALS} random targets in [-50, 2050) against {} random even keys ({lo}..{hi}):",
        keys.len()
    );
    println!(
        "  {mismatches} mismatches with the brute-force scan; {ties} targets were exact ties (smaller key wins), {outside} lay outside the keys"
    );
    println!(
        "  the most nodes one query visited = {max_visited}, tree height = {} (visited <= height: {}): one descent, Theta(h)",
        height(&t.root),
        max_visited <= height(&t.root)
    );
}

// ------------------------------------------------------------ (c) count smaller

// counts_brute: the definition. n(n-1)/2 comparisons, always.
fn counts_brute(a: &[i64]) -> (Vec<usize>, usize) {
    let mut out = vec![0; a.len()];
    let mut cmps = 0;
    for i in 0..a.len() {
        for j in i + 1..a.len() {
            cmps += 1;
            out[i] += usize::from(a[j] < a[i]);
        }
    }
    (out, cmps)
}

// counts_bst goes right to left: before a[i] is inserted, the tree holds exactly
// a[i+1..], so the "smaller" count of the insert descent IS the answer.
fn counts_bst(a: &[i64]) -> (Vec<usize>, usize) {
    let mut t = Bst::default();
    let mut out = vec![0; a.len()];
    for i in (0..a.len()).rev() {
        out[i] = t.add(a[i]);
    }
    (out, t.cmps)
}

// counts_merge sorts indices by value. When an element of the left half is
// emitted, the right-half elements emitted before it are exactly the ones
// strictly smaller (a right element goes first only if it is strictly smaller,
// so equal keys never count). Stable, and n log n comparisons whatever the input.
fn counts_merge(a: &[i64]) -> (Vec<usize>, usize) {
    // Go uses a closure that captures idx, tmp, out and cmps; a Rust closure
    // cannot call itself, so the shared state travels in a struct.
    struct Sorter<'a> {
        a: &'a [i64],
        idx: Vec<usize>,
        tmp: Vec<usize>,
        out: Vec<usize>,
        cmps: usize,
    }
    impl Sorter<'_> {
        fn sort_range(&mut self, lo: usize, hi: usize) {
            if hi - lo < 2 {
                return;
            }
            let mid = usize::midpoint(lo, hi);
            self.sort_range(lo, mid);
            self.sort_range(mid, hi);
            let (mut i, mut j, mut k) = (lo, mid, lo);
            while i < mid {
                if j < hi {
                    self.cmps += 1;
                    if self.a[self.idx[j]] < self.a[self.idx[i]] {
                        self.tmp[k] = self.idx[j];
                        j += 1;
                        k += 1;
                        continue;
                    }
                }
                self.out[self.idx[i]] += j - mid;
                self.tmp[k] = self.idx[i];
                i += 1;
                k += 1;
            }
            self.tmp[k..hi].copy_from_slice(&self.idx[j..hi]); // left half exhausted: the rest of the right half
            self.idx[lo..hi].copy_from_slice(&self.tmp[lo..hi]);
        }
    }
    let mut s = Sorter {
        a,
        idx: (0..a.len()).collect(),
        tmp: vec![0; a.len()],
        out: vec![0; a.len()],
        cmps: 0,
    };
    s.sort_range(0, a.len());
    (s.out, s.cmps)
}

fn all_equal(a: &[i64]) -> bool {
    let b = counts_brute(a).0;
    b == counts_bst(a).0 && b == counts_merge(a).0
}

fn count_smaller(rng: &mut Pcg) {
    let small = [3, 3, 1, 3, 2];
    println!(
        "  {}: brute {}, tree {}, merge sort {}",
        go_slice(&small),
        go_slice(&counts_brute(&small).0),
        go_slice(&counts_bst(&small).0),
        go_slice(&counts_merge(&small).0)
    );

    let narrow: Vec<i64> = (0..200).map(|_| rng.int_n(20) as i64).collect();
    println!(
        "  n = 200, values in [0, 20) (about 10 copies of each value): all three agree = {}",
        all_equal(&narrow)
    );

    let (arrays, mut positions, mut bad) = (1000, 0, 0);
    for trial in 0..arrays {
        let width = [8, 64, 1 << 30][trial % 3];
        let len = rng.int_n(100);
        let a: Vec<i64> = (0..len).map(|_| rng.int_n(width) as i64).collect();
        positions += a.len();
        bad += usize::from(!all_equal(&a));
    }
    println!(
        "  {arrays} random arrays (n < 100, value ranges 8, 64 and 2^30): {positions} positions, {bad} arrays where the methods disagree"
    );

    // The array is a shuffled 0..n-1 ("random"), 0..n-1 ("sorted") or n-1..0
    // ("reversed"). The BST goes right to left, so a sorted array inserts
    // DEscending keys (a left path) and a reversed one ascending keys (a right path).
    println!(
        "\n  comparisons to count smaller-after-self (merge sort and Knuth are for the random array):"
    );
    println!(
        "  {:>6} {:>11} {:>11} {:>11} {:>11} | {:>9} {:>9} {:>10} | random / n log2 n",
        "n",
        "BST random",
        "BST sorted",
        "BST revers.",
        "merge sort",
        "n log2 n",
        "Knuth",
        "n(n-1)/2"
    );
    let (mut paths_cost, mut agree) = (true, true);
    for n in [1000usize, 2000, 4000] {
        let rnd: Vec<i64> = shuffled(rng, n).iter().map(|&k| k as i64).collect();
        let asc: Vec<i64> = (0..n as i64).collect();
        let desc: Vec<i64> = (0..n as i64).rev().collect();
        let cr = counts_bst(&rnd).1;
        let ca = counts_bst(&asc).1;
        let cd = counts_bst(&desc).1;
        let cm = counts_merge(&rnd).1;
        let cb = counts_brute(&rnd).1;
        let h: f64 = (1..=n).map(|i| 1.0 / i as f64).sum();
        let (fn_, pairs) = (n as f64, n * (n - 1) / 2);
        println!(
            "  {n:>6} {cr:>11} {ca:>11} {cd:>11} {cm:>11} | {:>9.0} {:>9.0} {pairs:>10} | {:>17.2}",
            fn_ * fn_.log2(),
            2.0 * (fn_ + 1.0) * h - 4.0 * fn_,
            cr as f64 / (fn_ * fn_.log2())
        );
        paths_cost = paths_cost && ca == pairs && cd == pairs && cb == pairs;
        agree = agree && all_equal(&rnd) && all_equal(&asc) && all_equal(&desc);
    }
    println!(
        "  sorted, reversed and brute force cost exactly n(n-1)/2 at every n: {paths_cost}; the three methods agree on all nine arrays: {agree}"
    );
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    println!("(a) recover a BST with two keys swapped");
    recover_demo();
    recover_trials(&mut rng);
    println!("\n(b) the closest key to a target");
    closest_trials(&mut rng);
    println!("\n(c) count smaller elements after self");
    count_smaller(&mut rng);
}
```

**Rust output:**

```
(a) recover a BST with two keys swapped
  swap the keys at in-order positions 1 and 5: in-order [1 6 3 4 5 2 7]
    descents (6 > 3) (5 > 2): 2, so first = 6 and second = 2
    swapped back: in-order [1 2 3 4 5 6 7]
  swap the keys at in-order positions 2 and 3: in-order [1 2 4 3 5 6 7]
    descents (4 > 3): 1, so first = 4 and second = 3
    swapped back: in-order [1 2 3 4 5 6 7]
  2000 random trees (n = 2..40), two random keys swapped, one in-order pass, swapped back:
  0 failures (the repaired tree equals the original, shape and keys)
  1 descent in 335 cases (the keys were in-order neighbours), 2 descents in 1665
  deepest in-order stack = 11, tallest tree = 14: one pass visits each node once and holds one root-to-node path

(b) the closest key to a target
  closest(7) in {2 4 6 8 10 12 14} = 6
  closest(11) in {2 4 6 8 10 12 14} = 10
  closest(-5) in {2 4 6 8 10 12 14} = 2
  closest(100) in {2 4 6 8 10 12 14} = 14
  closest(6) in {2 4 6 8 10 12 14} = 6
  10000 random targets in [-50, 2050) against 300 random even keys (10..1998):
  0 mismatches with the brute-force scan; 1383 targets were exact ties (smaller key wins), 527 lay outside the keys
  the most nodes one query visited = 18, tree height = 18 (visited <= height: true): one descent, Theta(h)

(c) count smaller elements after self
  [3 3 1 3 2]: brute [2 2 0 1 0], tree [2 2 0 1 0], merge sort [2 2 0 1 0]
  n = 200, values in [0, 20) (about 10 copies of each value): all three agree = true
  1000 random arrays (n < 100, value ranges 8, 64 and 2^30): 47740 positions, 0 arrays where the methods disagree

  comparisons to count smaller-after-self (merge sort and Knuth are for the random array):
       n  BST random  BST sorted BST revers.  merge sort |  n log2 n     Knuth   n(n-1)/2 | random / n log2 n
    1000       10570      499500      499500        8725 |      9966     10986     499500 |              1.06
    2000       24696     1999000     1999000       19429 |     21932     24730    1999000 |              1.13
    4000       55507     7998000     7998000       42925 |     47863     54989    7998000 |              1.16
  sorted, reversed and brute force cost exactly n(n-1)/2 at every n: true; the three methods agree on all nine arrays: true
```

> **Go vs Rust:** Identical output, byte for byte (diffed against a live run of the Go program; `Pcg` is the PCG-DXSM port from lesson 01, so both draw the same numbers). Rust cannot hold two `&mut Node` into one tree, so `find_swapped` returns the two wrong *keys* and `swap_values` repairs the tree by value in a second walk, where Go swaps through two node pointers. The insert loop with a `&mut Option<Box<Node>>` cursor (Go's `link := &t.root`) compiles and updates `size` on the way down; the 4,000-node paths are freed by the iterative `Drop` from example 1, not by recursive drop glue.

**Complexity:** recover Θ(n) time and Θ(h) space (one in-order pass; 1 descent when the swapped keys were neighbours, otherwise 2) · closest Θ(h) · count-smaller by BST Θ(n log n) expected but **Θ(n²) worst case**, merge sort Θ(n log n) always. The run: 0 recover failures in 2,000 trees; 0 closest mismatches in 10,000 targets (about 1,400 of them exact ties); and at n = 4,000 the BST needs 55,507 comparisons on a random array (1.16 × n log2 n, close to Knuth's 54,989 expectation) but exactly n(n-1)/2 = 7,998,000 on a sorted one and on a reversed one, the same as brute force, against 42,925 for merge sort. A sorted array and a reversed array both degenerate, because the right-to-left scan turns one into a left path and the other into a right path.

---

## 16. Capstone: `Tree[K,V]`, built, proved, measured

`🔴 hard` · *Applied mini-build*

The lesson assembled into one ordered map with no rebalancing: `Put` (replace on an equal key), `Get`, `Delete` (the successor node is relinked, not copied), `Min`/`Max`, `Floor`/`Ceil`/`Pred`/`Succ`, `Rank`/`Select` from a per-node size, a `Range(lo, hi)` iterator, `Len`, `Height`, and a `Check()` that verifies the `(lo, hi)` bounds invariant (which makes the in-order sequence strictly ascending) and recounts every size. Every operation is Θ(h), which is Θ(log n) for random insertion order and Θ(n) for sorted order, and the program measures both. It is proved against a sorted-slice model on seeded random operation streams, including duplicates, re-deletes and the empty tree, exhaustively for every insertion order and deletion order up to n = 6, and with a planted bug that `Check()` has to catch. It is a capstone, so the Go program is longer than the usual hard-tier size.

**Steps:**

1. Read `Tree[K, V]` (sizes repaired on the way back up in `Put` and `Delete`, one-descent `Floor`/`Ceil`/`Rank`/`Select`, a pruned `Range`) and the demo on a 7-key tree.
2. Run the proofs: 200,000 narrow-range operations with `Check()` after every one, 100,000 wide-range ones, all 720 × 720 order pairs at n = 6, then the planted bug and the operation at which `Check()` reports it.
3. Read the measurements: height and comparison counts for random, sorted and zig-zag order at n = 4,000 (identical in both languages), and one timing table at n = 100,000 against the sorted slice.

```go
package main

import (
	"cmp"
	"fmt"
	"iter"
	"math/bits"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// ===================================================================
// Tree[K, V]: an ordered map built from the lesson's parts, with NO
// rebalancing. Every operation is Theta(h), where h is the height.
// ===================================================================

type node[K cmp.Ordered, V any] struct {
	key         K
	val         V
	size        int // nodes in this subtree, itself included
	left, right *node[K, V]
}

type Tree[K cmp.Ordered, V any] struct {
	root  *node[K, V]
	Cmps  int  // three-way key comparisons made so far: every descent goes through compare
	fault bool // test hook for part (2d): detachMin forgets to repair sizes
}

func size[K cmp.Ordered, V any](n *node[K, V]) int {
	if n == nil {
		return 0
	}
	return n.size
}

func (n *node[K, V]) fix() { n.size = 1 + size(n.left) + size(n.right) } // the size invariant, in one place

func (t *Tree[K, V]) compare(a, b K) int { t.Cmps++; return cmp.Compare(a, b) }

func (t *Tree[K, V]) Len() int { return size(t.root) }

// Put stores v under k and reports whether k was already there. It returns the
// new root of each subtree, so sizes are recomputed on the way back up.
func (t *Tree[K, V]) Put(k K, v V) (replaced bool) {
	t.root = t.put(t.root, k, v, &replaced)
	return
}

func (t *Tree[K, V]) put(n *node[K, V], k K, v V, replaced *bool) *node[K, V] {
	if n == nil {
		return &node[K, V]{key: k, val: v, size: 1}
	}
	switch c := t.compare(k, n.key); {
	case c < 0:
		n.left = t.put(n.left, k, v, replaced)
	case c > 0:
		n.right = t.put(n.right, k, v, replaced)
	default:
		n.val, *replaced = v, true
	}
	n.fix()
	return n
}

func (t *Tree[K, V]) Get(k K) (v V, ok bool) {
	for n := t.root; n != nil; {
		switch c := t.compare(k, n.key); {
		case c < 0:
			n = n.left
		case c > 0:
			n = n.right
		default:
			return n.val, true
		}
	}
	return
}

// Delete removes k. A node with two children is replaced by its in-order
// successor NODE, relinked (not copied), so every surviving node keeps its identity.
func (t *Tree[K, V]) Delete(k K) (deleted bool) {
	t.root = t.del(t.root, k, &deleted)
	return
}

func (t *Tree[K, V]) del(n *node[K, V], k K, deleted *bool) *node[K, V] {
	if n == nil {
		return nil
	}
	switch c := t.compare(k, n.key); {
	case c < 0:
		n.left = t.del(n.left, k, deleted)
	case c > 0:
		n.right = t.del(n.right, k, deleted)
	default:
		*deleted = true
		if n.left == nil {
			return n.right
		}
		if n.right == nil {
			return n.left
		}
		var succ *node[K, V]
		n.right, succ = t.detachMin(n.right)
		succ.left, succ.right = n.left, n.right
		n = succ
	}
	n.fix()
	return n
}

// detachMin unlinks the leftmost node of n's subtree and returns the rest plus that node.
func (t *Tree[K, V]) detachMin(n *node[K, V]) (rest, min *node[K, V]) {
	if n.left == nil {
		return n.right, n
	}
	n.left, min = t.detachMin(n.left)
	if !t.fault {
		n.fix()
	}
	return n, min
}

func kv[K cmp.Ordered, V any](n *node[K, V]) (k K, v V, ok bool) {
	if n != nil {
		return n.key, n.val, true
	}
	return
}

// floor: the largest key <= k (strict: < k). One descent; remember every right turn.
func (t *Tree[K, V]) floor(k K, strict bool) (best *node[K, V]) {
	for n := t.root; n != nil; {
		if c := t.compare(k, n.key); c > 0 || c == 0 && !strict {
			if best = n; c == 0 {
				return
			}
			n = n.right
		} else {
			n = n.left
		}
	}
	return
}

// ceil: the smallest key >= k (strict: > k). The mirror image.
func (t *Tree[K, V]) ceil(k K, strict bool) (best *node[K, V]) {
	for n := t.root; n != nil; {
		if c := t.compare(k, n.key); c < 0 || c == 0 && !strict {
			if best = n; c == 0 {
				return
			}
			n = n.left
		} else {
			n = n.right
		}
	}
	return
}

func (t *Tree[K, V]) Floor(k K) (K, V, bool) { return kv(t.floor(k, false)) }
func (t *Tree[K, V]) Pred(k K) (K, V, bool)  { return kv(t.floor(k, true)) }
func (t *Tree[K, V]) Ceil(k K) (K, V, bool)  { return kv(t.ceil(k, false)) }
func (t *Tree[K, V]) Succ(k K) (K, V, bool)  { return kv(t.ceil(k, true)) }

func (t *Tree[K, V]) Min() (K, V, bool) {
	n := t.root
	for n != nil && n.left != nil {
		n = n.left
	}
	return kv(n)
}

func (t *Tree[K, V]) Max() (K, V, bool) {
	n := t.root
	for n != nil && n.right != nil {
		n = n.right
	}
	return kv(n)
}

// Rank is the number of keys smaller than k; Select is its inverse. Both are
// one descent, because every node knows how many keys sit in its left subtree.
func (t *Tree[K, V]) Rank(k K) (r int) {
	for n := t.root; n != nil; {
		if t.compare(k, n.key) <= 0 {
			n = n.left
		} else {
			r += size(n.left) + 1
			n = n.right
		}
	}
	return
}

func (t *Tree[K, V]) Select(i int) (K, V, bool) {
	if i < 0 || i >= t.Len() {
		return kv[K, V](nil)
	}
	for n := t.root; ; {
		switch l := size(n.left); {
		case i < l:
			n = n.left
		case i > l:
			i -= l + 1
			n = n.right
		default:
			return kv(n)
		}
	}
}

// Range yields the pairs with lo <= key <= hi in ascending order; the caller can
// break out early. It descends only where the answer can be: Theta(h + m).
func (t *Tree[K, V]) Range(lo, hi K) iter.Seq2[K, V] {
	return func(yield func(K, V) bool) {
		var walk func(n *node[K, V]) bool
		walk = func(n *node[K, V]) bool {
			if n == nil {
				return true
			}
			if cmp.Compare(lo, n.key) < 0 && !walk(n.left) {
				return false
			}
			if cmp.Compare(lo, n.key) <= 0 && cmp.Compare(n.key, hi) <= 0 && !yield(n.key, n.val) {
				return false
			}
			return cmp.Compare(n.key, hi) >= 0 || walk(n.right)
		}
		walk(t.root)
	}
}

// Height is the number of nodes on the longest root-to-nil path. It uses an
// explicit stack, so it also works on a 4,000-node path.
func (t *Tree[K, V]) Height() (h int) {
	type item struct {
		n *node[K, V]
		d int
	}
	for stack := []item{{t.root, 1}}; len(stack) > 0; {
		it := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if it.n != nil {
			h = max(h, it.d)
			stack = append(stack, item{it.n.left, it.d + 1}, item{it.n.right, it.d + 1})
		}
	}
	return
}

// Check verifies the two invariants and returns the first violation: every key
// lies strictly inside the open interval its ancestors allow (which makes the
// in-order sequence strictly ascending), and every size field equals the
// recounted size. It returns an error rather than panicking.
func (t *Tree[K, V]) Check() error {
	_, err := check(t.root, nil, nil)
	return err
}

func check[K cmp.Ordered, V any](n *node[K, V], lo, hi *K) (int, error) {
	if n == nil {
		return 0, nil
	}
	if lo != nil && cmp.Compare(n.key, *lo) <= 0 || hi != nil && cmp.Compare(n.key, *hi) >= 0 {
		return 0, fmt.Errorf("key %v breaks the bounds invariant", n.key)
	}
	l, err := check(n.left, lo, &n.key)
	if err != nil {
		return 0, err
	}
	r, err := check(n.right, &n.key, hi)
	if err != nil {
		return 0, err
	}
	if n.size != 1+l+r {
		return 0, fmt.Errorf("key %v: size field %d, recounted 1 + %d + %d = %d", n.key, n.size, l, r, 1+l+r)
	}
	return 1 + l + r, nil
}

// ===================================================================
// The sorted-slice model, and the random-operation harness
// ===================================================================

type pair struct{ k, v int }
type model []pair

type res struct { // one answer, comparable with ==
	k, v int
	ok   bool
}

func toRes(k, v int, ok bool) res { return res{k, v, ok} }

func (m model) find(k int) (int, bool) { // lower bound, and whether it is k
	return slices.BinarySearchFunc(m, k, func(p pair, k int) int { return cmp.Compare(p.k, k) })
}

func (m model) at(j int) res {
	if j < 0 || j >= len(m) {
		return res{}
	}
	return res{m[j].k, m[j].v, true}
}

func btoi(b bool) int {
	if b {
		return 1
	}
	return 0
}

// agrees: the tree's in-order pairs, and Len, Min and Max, equal the model's.
func agrees(t *Tree[int, int], m model) bool {
	i := 0
	for k, v := range t.Range(-1<<62, 1<<62) {
		if i >= len(m) || m[i] != (pair{k, v}) {
			return false
		}
		i++
	}
	return i == len(m) && t.Len() == len(m) && toRes(t.Min()) == m.at(0) && toRes(t.Max()) == m.at(len(m)-1)
}

// rangeOK compares Range(lo, hi) with the model's pairs, including lo > hi.
func rangeOK(t *Tree[int, int], m model, lo, hi int) bool {
	i, _ := m.find(lo)
	for k, v := range t.Range(lo, hi) {
		if i >= len(m) || m[i] != (pair{k, v}) {
			return false
		}
		i++
	}
	return i == len(m) || m[i].k > hi
}

var opNames = []string{"put", "delete", "get", "floor", "ceil", "pred", "succ", "rank", "select", "range"}
var opMix = []int{0, 0, 0, 1, 1, 1, 2, 3, 4, 5, 6, 7, 8, 9} // put and delete 3 in 14 each

type stats struct {
	counts                                    [10]int
	mismatches, failures, replaced, hit, miss int
	empty, maxLen, firstOp                    int
	firstErr                                  error
	firstKind                                 string
}

// run applies ops random operations to t and to the sorted-slice model and compares
// every answer. Check() runs after every `every` operations, and a short burst of
// deletes every 5,000 operations (drain) empties the narrow tree again and again.
func run(t *Tree[int, int], rng *rand.Rand, ops, keyRange, width, every int, wide, drain, stop bool) (s stats) {
	var m model
	for i := 1; i <= ops; i++ {
		kind := opMix[rng.IntN(len(opMix))]
		if drain && i%5000 < 400 {
			kind = 1
		}
		k := rng.IntN(keyRange)
		if wide && kind == 1 && len(m) > 0 && rng.IntN(4) == 0 {
			k = m[rng.IntN(len(m))].k // a quarter of the deletes aim at a key that exists
		}
		var want, got res
		switch kind {
		case 0:
			j, found := m.find(k)
			if found {
				m[j].v = i
			} else {
				m = slices.Insert(m, j, pair{k, i})
			}
			want, got = res{ok: found}, res{ok: t.Put(k, i)}
			s.replaced += btoi(found)
		case 1:
			j, found := m.find(k)
			if found {
				m = slices.Delete(m, j, j+1)
			}
			want, got = res{ok: found}, res{ok: t.Delete(k)}
			s.hit, s.miss = s.hit+btoi(found), s.miss+btoi(!found)
		case 2:
			if j, found := m.find(k); found {
				want = m.at(j)
			}
			got = res{}
			if v, ok := t.Get(k); ok {
				got = res{k, v, true}
			}
		case 3: // floor: the last key <= k
			j, found := m.find(k)
			want, got = m.at(j-1+btoi(found)), toRes(t.Floor(k))
		case 4: // ceil: the first key >= k
			j, _ := m.find(k)
			want, got = m.at(j), toRes(t.Ceil(k))
		case 5: // pred: the last key < k
			j, _ := m.find(k)
			want, got = m.at(j-1), toRes(t.Pred(k))
		case 6: // succ: the first key > k
			j, found := m.find(k)
			want, got = m.at(j+btoi(found)), toRes(t.Succ(k))
		case 7:
			j, _ := m.find(k)
			want, got = res{k: j}, res{k: t.Rank(k)}
		case 8:
			idx := k % (len(m) + 2) // sometimes one or two past the end
			want, got = m.at(idx), toRes(t.Select(idx))
		case 9:
			hi := k + rng.IntN(width) - width/8 // sometimes lo > hi
			want, got = res{ok: true}, res{ok: rangeOK(t, m, k, hi)}
		}
		s.counts[kind]++
		s.mismatches += btoi(want != got || t.Len() != len(m))
		s.empty += btoi(len(m) == 0)
		s.maxLen = max(s.maxLen, len(m))
		if i%every == 0 {
			err := t.Check()
			s.failures += btoi(err != nil)
			s.mismatches += btoi(!agrees(t, m))
			if err != nil && s.firstErr == nil {
				s.firstOp, s.firstErr, s.firstKind = i, err, opNames[kind]
			}
		}
		if stop && s.firstErr != nil {
			break
		}
	}
	return
}

// ===================================================================

func demo() {
	t := &Tree[int, string]{}
	for _, k := range []int{40, 20, 60, 10, 30, 50, 70} {
		t.Put(k, fmt.Sprint("v", k))
	}
	show := func(k int, v string, ok bool) string {
		if !ok {
			return "none"
		}
		return fmt.Sprintf("%d=%s", k, v)
	}
	pairs := func(lo, hi int) string {
		var out []string
		for k, v := range t.Range(lo, hi) {
			out = append(out, show(k, v, true))
		}
		return strings.Join(out, " ")
	}
	v30, _ := t.Get(30)
	_, ok35 := t.Get(35)
	fmt.Printf("  put 40 20 60 10 30 50 70: Len = %d, Height = %d, Check passes = %v\n", t.Len(), t.Height(), t.Check() == nil)
	fmt.Printf("  Get(30) = %s, Get(35) found = %v, Min = %s, Max = %s\n", v30, ok35, show(t.Min()), show(t.Max()))
	fmt.Printf("  Floor(35) = %s, Ceil(35) = %s, Pred(40) = %s, Succ(40) = %s, Floor(5) = %s\n",
		show(t.Floor(35)), show(t.Ceil(35)), show(t.Pred(40)), show(t.Succ(40)), show(t.Floor(5)))
	fmt.Printf("  Rank(35) = %d keys below it, Select(3) = %s, Range(25, 60) = %s\n", t.Rank(35), show(t.Select(3)), pairs(25, 60))
	fmt.Printf("  Put(30) again replaced = %v; Delete(40), a node with two children = %v; Delete(40) again = %v\n", t.Put(30, "new"), t.Delete(40), t.Delete(40))
	fmt.Printf("  now %s, Len = %d, Check passes = %v\n", pairs(0, 100), t.Len(), t.Check() == nil)
}

// (2c) every insertion order x every deletion order of the keys 0..n-1.
func permutations(n int) [][]int {
	if n == 0 {
		return [][]int{{}}
	}
	var out [][]int
	for _, p := range permutations(n - 1) {
		for i := 0; i <= len(p); i++ {
			out = append(out, slices.Insert(slices.Clone(p), i, n-1))
		}
	}
	return out
}

// present: the tree holds exactly the keys whose bit is set in mask, and passes Check.
func present(t *Tree[int, int], mask int) bool {
	if t.Check() != nil || t.Len() != bits.OnesCount(uint(mask)) {
		return false
	}
	for k := range t.Range(0, 63) {
		if mask == 0 || bits.TrailingZeros(uint(mask)) != k {
			return false
		}
		mask &= mask - 1
	}
	return mask == 0
}

func exhaustive() {
	failures := 0
	for n := 1; n <= 6; n++ {
		perms := permutations(n)
		for _, ins := range perms {
			for _, del := range perms {
				t, mask := &Tree[int, int]{}, 0
				for _, k := range ins {
					mask |= 1 << k
					failures += btoi(t.Put(k, k) || !present(t, mask))
				}
				for _, k := range del {
					mask &^= 1 << k
					failures += btoi(!t.Delete(k) || !present(t, mask))
				}
			}
		}
		fmt.Printf("  n = %d: %d x %d = %6d (insertion order, deletion order) pairs, %7d operations checked\n",
			n, len(perms), len(perms), len(perms)*len(perms), len(perms)*len(perms)*2*n)
	}
	fmt.Printf("  %d failures\n", failures)
}

func proofs() {
	fmt.Println("(2a) narrow keys [0, 64): 200000 random operations (plus a burst of 400 deletes every 5000) against the sorted-slice model, Check() after every one")
	s := run(&Tree[int, int]{}, rand.New(rand.NewPCG(1, 2)), 200_000, 64, 16, 1, false, true, false)
	fmt.Printf("  %d mismatches, %d invariant failures over %d operations\n", s.mismatches, s.failures, 200_000)
	var mix []string
	for i, c := range s.counts {
		mix = append(mix, fmt.Sprintf("%s %d", opNames[i], c))
	}
	fmt.Println("  mix:", strings.Join(mix, ", "))
	fmt.Printf("  coverage: %d puts replaced a key, %d deletes hit and %d missed, the tree was empty after %d operations, at most %d keys\n",
		s.replaced, s.hit, s.miss, s.empty, s.maxLen)

	fmt.Println("\n(2b) wide keys [0, 2^20): 100000 operations, Check() and a full in-order comparison every 1000")
	s = run(&Tree[int, int]{}, rand.New(rand.NewPCG(2, 2)), 100_000, 1<<20, 1<<14, 1000, true, false, false)
	fmt.Printf("  %d mismatches, %d invariant failures over %d operations; %d deletes hit and %d missed, at most %d keys\n",
		s.mismatches, s.failures, 100_000, s.hit, s.miss, s.maxLen)

	fmt.Println("\n(2c) every insertion order x every deletion order of the keys 0..n-1, Check() and the key set verified after every operation:")
	exhaustive()

	fmt.Println("\n(2d) a planted bug: detachMin skips the size repair; the same stream as (2a), stopped at the first failure")
	bad := &Tree[int, int]{fault: true}
	s = run(bad, rand.New(rand.NewPCG(1, 2)), 200_000, 64, 16, 1, false, true, true)
	fmt.Printf("  Check() failed at operation %d (a %s): %v\n", s.firstOp, s.firstKind, s.firstErr)
}

// ===================================================================
// (3) measure
// ===================================================================

func drawDistinct(rng *rand.Rand, n, keyRange int) []int {
	seen := make(map[int]bool, n)
	keys := make([]int, 0, n)
	for len(keys) < n {
		if k := rng.IntN(keyRange); !seen[k] {
			seen[k] = true
			keys = append(keys, k)
		}
	}
	return keys
}

func shape() {
	const n = 4000
	keys := drawDistinct(rand.New(rand.NewPCG(3, 2)), n, 1<<30)
	sorted := slices.Sorted(slices.Values(keys))
	zigzag := make([]int, 0, n)
	for i := range n / 2 {
		zigzag = append(zigzag, sorted[i], sorted[n-1-i])
	}
	fmt.Printf("  the same %d keys put in three orders (zig-zag = smallest, largest, 2nd smallest, 2nd largest, ...):\n", n)
	fmt.Printf("  %-8s %7s %14s %14s %13s %6s\n", "order", "height", "Put cmps", "Get cmps (all)", "mean per Get", "Check")
	identity := true
	for _, c := range []struct {
		name string
		seq  []int
	}{{"random", keys}, {"sorted", sorted}, {"zig-zag", zigzag}} {
		t := &Tree[int, int]{}
		for _, k := range c.seq {
			t.Put(k, k)
		}
		put := t.Cmps
		t.Cmps = 0
		for _, k := range c.seq {
			t.Get(k)
		}
		identity = identity && t.Cmps == put+n
		fmt.Printf("  %-8s %7d %14d %14d %13.2f %6v\n", c.name, t.Height(), put, t.Cmps, float64(t.Cmps)/n, t.Check() == nil)
	}
	fmt.Printf("  Get cmps (all) = Put cmps + n in every row: %v (a key's Put cost is its depth, its Get cost is depth + 1)\n", identity)
}

func timeIt(reps int, f func()) float64 {
	start := time.Now()
	f()
	return float64(time.Since(start).Nanoseconds()) / float64(reps)
}

// bench is one row of the table: the same operation on the Tree and on the sorted slice.
type bench struct {
	name        string
	reps        int
	tree, slice func()
}

func timings() {
	const N, M, G, S, R, W = 100_000, 10_000, 200_000, 1_000_000, 100_000, 50
	var best [6][2]float64
	for i := range best {
		best[i] = [2]float64{1e18, 1e18}
	}
	var names []string
	var reps []int
	agree := true
	for round := range 3 { // best of 3 rounds, each from the same seeded data
		rng := rand.New(rand.NewPCG(4, 2))
		keys := drawDistinct(rng, N+M, 1<<30) // the first N are built, the last M are put later
		probes, sel, starts := make([]int, G), make([]int, S), make([]int, R)
		for i := range probes {
			probes[i] = keys[rng.IntN(N+M)]
		}
		for i := range sel {
			sel[i] = rng.IntN(N)
		}
		for i := range starts {
			starts[i] = rng.IntN(N - 100)
		}
		victims := slices.Clone(keys)
		for i := range M { // partial Fisher-Yates: M random keys to delete
			j := i + rng.IntN(len(victims)-i)
			victims[i], victims[j] = victims[j], victims[i]
		}
		victims = victims[:M]

		t := &Tree[int, int]{}
		m := make(model, 0, N+M)
		for _, k := range keys[:N] {
			t.Put(k, k)
			m = append(m, pair{k, k})
		}
		slices.SortFunc(m, func(a, b pair) int { return cmp.Compare(a.k, b.k) })

		var st, ss int // checksums: what each side read
		rows := []bench{
			{"Put", M, func() {
				for _, k := range keys[N:] {
					t.Put(k, k)
				}
			}, func() {
				for _, k := range keys[N:] {
					j, _ := m.find(k)
					m = slices.Insert(m, j, pair{k, k})
				}
			}},
			{"Get", G, func() {
				for _, k := range probes {
					v, _ := t.Get(k)
					st += v
				}
			}, func() {
				for _, k := range probes {
					j, _ := m.find(k)
					ss += m[j].v
				}
			}},
			{"Delete", M, func() {
				for _, k := range victims {
					t.Delete(k)
				}
			}, func() {
				for _, k := range victims {
					if j, ok := m.find(k); ok {
						m = slices.Delete(m, j, j+1)
					}
				}
			}},
			{"Select(random i)", S, func() {
				for _, i := range sel {
					k, _, _ := t.Select(i)
					st += k
				}
			}, func() {
				for _, i := range sel {
					ss += m[i].k
				}
			}},
			{"Select(n/2) by walking in order", W, func() { // the plain in-order walk, stopped at n/2
				for range W {
					i := 0
					for k := range t.Range(m[0].k, m[N-1].k) {
						if i == N/2 {
							st += k
							break
						}
						i++
					}
				}
			}, nil},
			{"Range of 100 keys", R, func() {
				for _, r := range starts {
					for k, v := range t.Range(m[r].k, m[r+99].k) {
						st += k + v
					}
				}
			}, func() {
				for _, r := range starts {
					j, _ := m.find(m[r].k)
					for _, p := range m[j : j+100] {
						ss += p.k + p.v
					}
				}
			}},
		}
		for i, r := range rows {
			st, ss = 0, 0
			best[i][0] = min(best[i][0], timeIt(r.reps, r.tree))
			if r.slice != nil {
				best[i][1] = min(best[i][1], timeIt(r.reps, r.slice))
				agree = agree && st == ss
			}
			if round == 0 {
				names, reps = append(names, r.name), append(reps, r.reps)
			}
		}
		agree = agree && t.Len() == N && len(m) == N && t.Check() == nil
	}
	fmt.Printf("  n = %d random keys; Put grows the map to %d, Delete brings it back to %d; best of 3 rounds, ns per operation:\n", N, N+M, N)
	fmt.Printf("  %-34s %8s %10s %12s %10s\n", "operation", "reps", "Tree", "sorted slice", "slice/Tree")
	for i, name := range names {
		if i == 4 {
			fmt.Printf("  %-34s %8d %10.0f %12s %10s\n", name, reps[i], best[i][0], "-", "-")
		} else {
			fmt.Printf("  %-34s %8d %10.0f %12.1f %10.3f\n", name, reps[i], best[i][0], best[i][1], best[i][1]/best[i][0])
		}
	}
	fmt.Printf("  tree and slice read the same checksums and ended with the same sizes in every round: %v\n", agree)
	fmt.Printf("  walking in order to rank n/2 costs %.0fx a Select descent\n", best[4][0]/best[3][0])
}

func main() {
	fmt.Println("(1) what it does")
	demo()
	fmt.Println()
	proofs()
	fmt.Println("\n(3a) the capstone has no balancing: n = 4000 (lesson 19 fixes this)")
	shape()
	fmt.Println("\n(3b) timings")
	timings()
}
```

**Sample output:**

```
(1) what it does
  put 40 20 60 10 30 50 70: Len = 7, Height = 3, Check passes = true
  Get(30) = v30, Get(35) found = false, Min = 10=v10, Max = 70=v70
  Floor(35) = 30=v30, Ceil(35) = 40=v40, Pred(40) = 30=v30, Succ(40) = 50=v50, Floor(5) = none
  Rank(35) = 3 keys below it, Select(3) = 40=v40, Range(25, 60) = 30=v30 40=v40 50=v50 60=v60
  Put(30) again replaced = true; Delete(40), a node with two children = true; Delete(40) again = false
  now 10=v10 20=v20 30=new 50=v50 60=v60 70=v70, Len = 6, Check passes = true

(2a) narrow keys [0, 64): 200000 random operations (plus a burst of 400 deletes every 5000) against the sorted-slice model, Check() after every one
  0 mismatches, 0 invariant failures over 200000 operations
  mix: put 39461, delete 55427, get 13225, floor 13000, ceil 13154, pred 13215, succ 13071, rank 13260, select 13162, range 13025
  coverage: 19147 puts replaced a key, 20280 deletes hit and 35147 missed, the tree was empty after 6814 operations, at most 47 keys

(2b) wide keys [0, 2^20): 100000 operations, Check() and a full in-order comparison every 1000
  0 mismatches, 0 invariant failures over 100000 operations; 5365 deletes hit and 16038 missed, at most 16033 keys

(2c) every insertion order x every deletion order of the keys 0..n-1, Check() and the key set verified after every operation:
  n = 1: 1 x 1 =      1 (insertion order, deletion order) pairs,       2 operations checked
  n = 2: 2 x 2 =      4 (insertion order, deletion order) pairs,      16 operations checked
  n = 3: 6 x 6 =     36 (insertion order, deletion order) pairs,     216 operations checked
  n = 4: 24 x 24 =    576 (insertion order, deletion order) pairs,    4608 operations checked
  n = 5: 120 x 120 =  14400 (insertion order, deletion order) pairs,  144000 operations checked
  n = 6: 720 x 720 = 518400 (insertion order, deletion order) pairs, 6220800 operations checked
  0 failures

(2d) a planted bug: detachMin skips the size repair; the same stream as (2a), stopped at the first failure
  Check() failed at operation 517 (a delete): key 26: size field 6, recounted 1 + 2 + 2 = 5

(3a) the capstone has no balancing: n = 4000 (lesson 19 fixes this)
  the same 4000 keys put in three orders (zig-zag = smallest, largest, 2nd smallest, 2nd largest, ...):
  order     height       Put cmps Get cmps (all)  mean per Get  Check
  random        28          52102          56102         14.03   true
  sorted      4000        7998000        8002000       2000.50   true
  zig-zag     4000        7998000        8002000       2000.50   true
  Get cmps (all) = Put cmps + n in every row: true (a key's Put cost is its depth, its Get cost is depth + 1)

(3b) timings
  n = 100000 random keys; Put grows the map to 110000, Delete brings it back to 100000; best of 3 rounds, ns per operation:
  operation                              reps       Tree sorted slice slice/Tree
  Put                                   10000        188      11882.4     63.205
  Get                                  200000        121         99.0      0.818
  Delete                                10000        198      11903.5     60.264
  Select(random i)                    1000000        139          0.8      0.006
  Select(n/2) by walking in order          50     725138            -          -
  Range of 100 keys                    100000       1789        144.4      0.081
  tree and slice read the same checksums and ended with the same sizes in every round: true
  walking in order to rank n/2 costs 5201x a Select descent
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::Ordering;
use std::collections::{BTreeMap, HashSet};
use std::fmt::Debug;
use std::hint::black_box;
use std::ops::Bound::{Excluded, Unbounded};
use std::time::Instant;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN (the same
// port as lesson 01, example 14), so both programs draw the same numbers.
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

// ===================================================================
// Tree<K, V>: an ordered map built from the lesson's parts, with NO
// rebalancing. Every operation is Theta(h), where h is the height.
// ===================================================================

struct Node<K, V> {
    key: K,
    val: V,
    size: usize, // nodes in this subtree, itself included
    left: Link<K, V>,
    right: Link<K, V>,
}
type Link<K, V> = Option<Box<Node<K, V>>>;

fn size<K, V>(n: &Link<K, V>) -> usize {
    n.as_ref().map_or(0, |n| n.size)
}

impl<K, V> Node<K, V> {
    // the size invariant, in one place
    fn fix(&mut self) {
        self.size = 1 + size(&self.left) + size(&self.right);
    }
}

struct Tree<K, V> {
    root: Link<K, V>,
    cmps: Cell<u64>, // three-way key comparisons made so far; a Cell so `&self` methods can count
    fault: bool,     // test hook for part (2d): detach_min forgets to repair sizes
}

// The Drop lives on the TREE, not on Node (example 1 put it on Node). Rust's default
// drop glue recurses once per level, which a 4,000-node path would survive but a
// longer one would not; this hands the nodes to a heap-allocated stack instead.
// Putting it on Tree leaves Node free of a destructor, and a type with a Drop impl
// cannot have a field moved out of it: `delete` moves the value out and returns it.
impl<K, V> Drop for Tree<K, V> {
    fn drop(&mut self) {
        let mut stack: Vec<Box<Node<K, V>>> = self.root.take().into_iter().collect();
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
        }
    }
}

// Range yields the pairs with lo <= key <= hi in ascending order. The stack holds
// the left spine from the root down to the first key >= lo; next() pops, then
// pushes the left spine of the popped node's right child. It stops at the first
// key above hi. Theta(h + m) for m results, and dropping it early costs nothing.
struct Range<'a, K, V> {
    stack: Vec<&'a Node<K, V>>,
    hi: &'a K,
}

impl<'a, K: Ord, V> Iterator for Range<'a, K, V> {
    type Item = (&'a K, &'a V);

    fn next(&mut self) -> Option<Self::Item> {
        let n = self.stack.pop()?;
        if n.key > *self.hi {
            self.stack.clear();
            return None;
        }
        let mut cur = n.right.as_deref();
        while let Some(c) = cur {
            self.stack.push(c);
            cur = c.left.as_deref();
        }
        Some((&n.key, &n.val))
    }
}

fn entry<K, V>(n: Option<&Node<K, V>>) -> Option<(&K, &V)> {
    n.map(|n| (&n.key, &n.val))
}

impl<K: Ord, V> Tree<K, V> {
    fn new() -> Self {
        Tree {
            root: None,
            cmps: Cell::new(0),
            fault: false,
        }
    }

    fn compare(&self, a: &K, b: &K) -> Ordering {
        self.cmps.set(self.cmps.get() + 1);
        a.cmp(b)
    }

    fn len(&self) -> usize {
        size(&self.root)
    }

    // put stores val under key and returns the old value if the key was there.
    // put_at takes a subtree BY VALUE and returns its new root, so sizes are
    // recomputed on the way back up and nothing holds a reference into the tree.
    fn put(&mut self, key: K, val: V) -> Option<V> {
        let mut old = None;
        let root = self.root.take();
        self.root = self.put_at(root, key, val, &mut old);
        old
    }

    fn put_at(&self, link: Link<K, V>, key: K, val: V, old: &mut Option<V>) -> Link<K, V> {
        let Some(mut n) = link else {
            return Some(Box::new(Node {
                key,
                val,
                size: 1,
                left: None,
                right: None,
            }));
        };
        match self.compare(&key, &n.key) {
            Ordering::Less => n.left = self.put_at(n.left.take(), key, val, old),
            Ordering::Greater => n.right = self.put_at(n.right.take(), key, val, old),
            Ordering::Equal => *old = Some(std::mem::replace(&mut n.val, val)),
        }
        n.fix();
        Some(n)
    }

    fn get(&self, key: &K) -> Option<&V> {
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            match self.compare(key, &n.key) {
                Ordering::Less => cur = n.left.as_deref(),
                Ordering::Greater => cur = n.right.as_deref(),
                Ordering::Equal => return Some(&n.val),
            }
        }
        None
    }

    // delete removes key and hands its value back. A node with two children is
    // replaced by its in-order successor NODE, relinked (not copied), so every
    // surviving node keeps its identity.
    fn delete(&mut self, key: &K) -> Option<V> {
        let mut out = None;
        let root = self.root.take();
        self.root = self.del_at(root, key, &mut out);
        out
    }

    fn del_at(&self, link: Link<K, V>, key: &K, out: &mut Option<V>) -> Link<K, V> {
        let mut n = link?; // None: fell off the tree, the key is absent
        match self.compare(key, &n.key) {
            Ordering::Less => n.left = self.del_at(n.left.take(), key, out),
            Ordering::Greater => n.right = self.del_at(n.right.take(), key, out),
            Ordering::Equal => {
                let (left, right) = (n.left.take(), n.right.take());
                *out = Some(n.val); // moves the value out of the box; n is gone after this arm
                return match (left, right) {
                    (None, r) => r,
                    (l, None) => l,
                    (l, Some(r)) => {
                        let (rest, mut succ) = self.detach_min(r);
                        succ.left = l;
                        succ.right = rest;
                        succ.fix();
                        Some(succ)
                    }
                };
            }
        }
        n.fix();
        Some(n)
    }

    // detach_min unlinks the leftmost node of n's subtree and returns the rest plus that node.
    fn detach_min(&self, mut n: Box<Node<K, V>>) -> (Link<K, V>, Box<Node<K, V>>) {
        match n.left.take() {
            None => (n.right.take(), n),
            Some(l) => {
                let (rest, min) = self.detach_min(l);
                n.left = rest;
                if !self.fault {
                    n.fix();
                }
                (Some(n), min)
            }
        }
    }

    // floor: the largest key <= k (strict: < k). One descent; remember every right turn.
    fn floor_node(&self, key: &K, strict: bool) -> Option<&Node<K, V>> {
        let (mut best, mut cur) = (None, self.root.as_deref());
        while let Some(n) = cur {
            match self.compare(key, &n.key) {
                Ordering::Equal if !strict => return Some(n),
                Ordering::Greater => {
                    best = Some(n);
                    cur = n.right.as_deref();
                }
                _ => cur = n.left.as_deref(), // key < n.key, or equal and strict
            }
        }
        best
    }

    // ceil: the smallest key >= k (strict: > k). The mirror image.
    fn ceil_node(&self, key: &K, strict: bool) -> Option<&Node<K, V>> {
        let (mut best, mut cur) = (None, self.root.as_deref());
        while let Some(n) = cur {
            match self.compare(key, &n.key) {
                Ordering::Equal if !strict => return Some(n),
                Ordering::Less => {
                    best = Some(n);
                    cur = n.left.as_deref();
                }
                _ => cur = n.right.as_deref(),
            }
        }
        best
    }

    fn floor(&self, key: &K) -> Option<(&K, &V)> {
        entry(self.floor_node(key, false))
    }

    fn pred(&self, key: &K) -> Option<(&K, &V)> {
        entry(self.floor_node(key, true))
    }

    fn ceil(&self, key: &K) -> Option<(&K, &V)> {
        entry(self.ceil_node(key, false))
    }

    fn succ(&self, key: &K) -> Option<(&K, &V)> {
        entry(self.ceil_node(key, true))
    }

    fn min(&self) -> Option<(&K, &V)> {
        let mut n = self.root.as_deref()?;
        while let Some(l) = n.left.as_deref() {
            n = l;
        }
        entry(Some(n))
    }

    fn max(&self) -> Option<(&K, &V)> {
        let mut n = self.root.as_deref()?;
        while let Some(r) = n.right.as_deref() {
            n = r;
        }
        entry(Some(n))
    }

    // rank is the number of keys smaller than key; select is its inverse. Both are
    // one descent, because every node knows how many keys sit in its left subtree.
    fn rank(&self, key: &K) -> usize {
        let (mut r, mut cur) = (0, self.root.as_deref());
        while let Some(n) = cur {
            if self.compare(key, &n.key) != Ordering::Greater {
                cur = n.left.as_deref();
            } else {
                r += size(&n.left) + 1;
                cur = n.right.as_deref();
            }
        }
        r
    }

    fn select(&self, mut i: usize) -> Option<(&K, &V)> {
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            let l = size(&n.left);
            match i.cmp(&l) {
                Ordering::Less => cur = n.left.as_deref(),
                Ordering::Greater => {
                    i -= l + 1;
                    cur = n.right.as_deref();
                }
                Ordering::Equal => return entry(Some(n)),
            }
        }
        None // i >= len: the descent falls off the tree
    }

    fn range<'a>(&'a self, lo: &K, hi: &'a K) -> Range<'a, K, V> {
        let mut stack = Vec::new();
        let mut cur = self.root.as_deref();
        while let Some(n) = cur {
            if n.key >= *lo {
                stack.push(n);
                cur = n.left.as_deref();
            } else {
                cur = n.right.as_deref();
            }
        }
        Range { stack, hi }
    }

    // height is the number of nodes on the longest root-to-nil path. It uses an
    // explicit stack, so it also works on a 4,000-node path.
    fn height(&self) -> usize {
        let mut h = 0;
        let mut stack: Vec<(&Node<K, V>, usize)> =
            self.root.as_deref().map(|n| (n, 1)).into_iter().collect();
        while let Some((n, d)) = stack.pop() {
            h = h.max(d);
            stack.extend(n.left.as_deref().map(|c| (c, d + 1)));
            stack.extend(n.right.as_deref().map(|c| (c, d + 1)));
        }
        h
    }
}

impl<K: Ord + Debug, V> Tree<K, V> {
    // check verifies the two invariants and returns the first violation: every key
    // lies strictly inside the open interval its ancestors allow (which makes the
    // in-order sequence strictly ascending), and every size field equals the
    // recounted size. It returns a Result rather than panicking.
    fn check(&self) -> Result<(), String> {
        check_at(&self.root, None, None).map(|_| ())
    }
}

fn check_at<K: Ord + Debug, V>(
    n: &Link<K, V>,
    lo: Option<&K>,
    hi: Option<&K>,
) -> Result<usize, String> {
    let Some(n) = n else { return Ok(0) };
    if lo.is_some_and(|lo| n.key <= *lo) || hi.is_some_and(|hi| n.key >= *hi) {
        return Err(format!("key {:?} breaks the bounds invariant", n.key));
    }
    let l = check_at(&n.left, lo, Some(&n.key))?;
    let r = check_at(&n.right, Some(&n.key), hi)?;
    if n.size != 1 + l + r {
        return Err(format!(
            "key {:?}: size field {}, recounted 1 + {l} + {r} = {}",
            n.key,
            n.size,
            1 + l + r
        ));
    }
    Ok(1 + l + r)
}

// ===================================================================
// The sorted-slice model, and the random-operation harness
// ===================================================================

type Pair = (i64, i64);

#[derive(Clone, Copy, Default, PartialEq)]
struct Res {
    // one answer, comparable with ==
    k: i64,
    v: i64,
    ok: bool,
}

impl Res {
    fn flag(ok: bool) -> Res {
        Res {
            ok,
            ..Res::default()
        }
    }

    fn of(e: Option<(&i64, &i64)>) -> Res {
        e.map_or(Res::default(), |(&k, &v)| Res { k, v, ok: true })
    }
}

// lower bound in the sorted slice, and whether it is k
fn find(m: &[Pair], k: i64) -> (usize, bool) {
    match m.binary_search_by_key(&k, |p| p.0) {
        Ok(j) => (j, true),
        Err(j) => (j, false),
    }
}

fn at(m: &[Pair], j: Option<usize>) -> Res {
    j.and_then(|j| m.get(j))
        .map_or(Res::default(), |&(k, v)| Res { k, v, ok: true })
}

// agrees: the tree's in-order pairs, and len, min and max, equal the model's.
fn agrees(t: &Tree<i64, i64>, m: &[Pair]) -> bool {
    t.range(&-(1 << 62), &(1 << 62))
        .map(|(&k, &v)| (k, v))
        .eq(m.iter().copied())
        && t.len() == m.len()
        && Res::of(t.min()) == at(m, Some(0))
        && Res::of(t.max()) == at(m, m.len().checked_sub(1))
}

// range_ok compares range(lo, hi) with the model's pairs, including lo > hi.
fn range_ok(t: &Tree<i64, i64>, m: &[Pair], lo: i64, hi: i64) -> bool {
    let (mut i, _) = find(m, lo);
    for (&k, &v) in t.range(&lo, &hi) {
        if m.get(i) != Some(&(k, v)) {
            return false;
        }
        i += 1;
    }
    m.get(i).is_none_or(|p| p.0 > hi)
}

const OP_NAMES: [&str; 10] = [
    "put", "delete", "get", "floor", "ceil", "pred", "succ", "rank", "select", "range",
];
// put and delete 3 in 14 each
const OP_MIX: [usize; 14] = [0, 0, 0, 1, 1, 1, 2, 3, 4, 5, 6, 7, 8, 9];

#[derive(Default)]
struct Stats {
    counts: [usize; 10],
    mismatches: usize,
    failures: usize,
    replaced: usize,
    hit: usize,
    miss: usize,
    empty: usize,
    max_len: usize,
    bt_mismatches: usize,
    first: Option<(usize, &'static str, String)>,
}

struct Cfg {
    ops: usize,
    key_range: usize,
    width: usize,
    every: usize,
    wide: bool,
    drain: bool,
    stop: bool,
}

// The third oracle, Rust only: the same operation on a std BTreeMap, in the
// shapes the lesson's Rust notes give (floor = range(..=k).next_back() ...).
// BTreeMap has no rank and no select: rank counts a range, select is iter().nth(i).
fn bt_step(
    bt: &mut BTreeMap<i64, i64>,
    t: &Tree<i64, i64>,
    kind: usize,
    (k, hi, i, idx): (i64, i64, i64, usize),
) -> Res {
    match kind {
        0 => Res::flag(bt.insert(k, i).is_some()),
        1 => Res::flag(bt.remove(&k).is_some()),
        2 => bt
            .get(&k)
            .map_or(Res::default(), |&v| Res { k, v, ok: true }),
        3 => Res::of(bt.range(..=k).next_back()),
        4 => Res::of(bt.range(k..).next()),
        5 => Res::of(bt.range(..k).next_back()),
        6 => Res::of(bt.range((Excluded(k), Unbounded)).next()),
        7 => Res {
            k: bt.range(..k).count() as i64,
            ..Res::default()
        },
        8 => Res::of(bt.iter().nth(idx)),
        // BTreeMap::range panics when start > end, so guard it ourselves
        _ => Res::flag(if k <= hi {
            t.range(&k, &hi)
                .map(|(&a, &b)| (a, b))
                .eq(bt.range(k..=hi).map(|(&a, &b)| (a, b)))
        } else {
            t.range(&k, &hi).next().is_none()
        }),
    }
}

// run applies cfg.ops random operations to t and to the sorted-slice model and
// compares every answer. check() runs after every cfg.every operations, and a short
// burst of deletes every 5,000 operations (drain) empties the narrow tree again and again.
fn run(
    t: &mut Tree<i64, i64>,
    rng: &mut Pcg,
    cfg: &Cfg,
    mut bt: Option<&mut BTreeMap<i64, i64>>,
) -> Stats {
    let mut s = Stats::default();
    let mut m: Vec<Pair> = Vec::new();
    for i in 1..=cfg.ops {
        let mut kind = OP_MIX[rng.int_n(OP_MIX.len())];
        if cfg.drain && i % 5000 < 400 {
            kind = 1;
        }
        let mut k = rng.int_n(cfg.key_range) as i64;
        if cfg.wide && kind == 1 && !m.is_empty() && rng.int_n(4) == 0 {
            k = m[rng.int_n(m.len())].0; // a quarter of the deletes aim at a key that exists
        }
        let idx = k as usize % (m.len() + 2); // sometimes one or two past the end
        let hi = if kind == 9 {
            k + rng.int_n(cfg.width) as i64 - (cfg.width / 8) as i64 // sometimes lo > hi
        } else {
            0
        };
        let val = i as i64;
        let (want, got) = match kind {
            0 => {
                let (j, found) = find(&m, k);
                if found {
                    m[j].1 = val;
                } else {
                    m.insert(j, (k, val));
                }
                s.replaced += usize::from(found);
                (Res::flag(found), Res::flag(t.put(k, val).is_some()))
            }
            1 => {
                let (j, found) = find(&m, k);
                if found {
                    m.remove(j);
                }
                s.hit += usize::from(found);
                s.miss += usize::from(!found);
                (Res::flag(found), Res::flag(t.delete(&k).is_some()))
            }
            2 => {
                let (j, found) = find(&m, k);
                let want = if found {
                    at(&m, Some(j))
                } else {
                    Res::default()
                };
                let got = t
                    .get(&k)
                    .map_or(Res::default(), |&v| Res { k, v, ok: true });
                (want, got)
            }
            3 => {
                // floor: the last key <= k
                let (j, found) = find(&m, k);
                let j = if found { Some(j) } else { j.checked_sub(1) };
                (at(&m, j), Res::of(t.floor(&k)))
            }
            4 => (at(&m, Some(find(&m, k).0)), Res::of(t.ceil(&k))), // ceil: the first key >= k
            5 => (at(&m, find(&m, k).0.checked_sub(1)), Res::of(t.pred(&k))), // pred: the last key < k
            6 => {
                // succ: the first key > k
                let (j, found) = find(&m, k);
                (at(&m, Some(j + usize::from(found))), Res::of(t.succ(&k)))
            }
            7 => (
                Res {
                    k: find(&m, k).0 as i64,
                    ..Res::default()
                },
                Res {
                    k: t.rank(&k) as i64,
                    ..Res::default()
                },
            ),
            8 => (at(&m, Some(idx)), Res::of(t.select(idx))),
            _ => (Res::flag(true), Res::flag(range_ok(t, &m, k, hi))),
        };
        if let Some(bt) = bt.as_deref_mut() {
            s.bt_mismatches += usize::from(bt_step(bt, t, kind, (k, hi, val, idx)) != got);
        }
        s.counts[kind] += 1;
        s.mismatches += usize::from(want != got || t.len() != m.len());
        s.empty += usize::from(m.is_empty());
        s.max_len = s.max_len.max(m.len());
        if i % cfg.every == 0 {
            let r = t.check();
            s.failures += usize::from(r.is_err());
            s.mismatches += usize::from(!agrees(t, &m));
            if let Err(e) = r
                && s.first.is_none()
            {
                s.first = Some((i, OP_NAMES[kind], e));
            }
        }
        if cfg.stop && s.first.is_some() {
            break;
        }
    }
    s
}

// ===================================================================

fn demo() {
    let mut t: Tree<i64, String> = Tree::new();
    for k in [40, 20, 60, 10, 30, 50, 70] {
        t.put(k, format!("v{k}"));
    }
    let show =
        |e: Option<(&i64, &String)>| e.map_or("none".to_string(), |(k, v)| format!("{k}={v}"));
    let pairs = |t: &Tree<i64, String>, lo: i64, hi: i64| {
        t.range(&lo, &hi)
            .map(|(k, v)| format!("{k}={v}"))
            .collect::<Vec<_>>()
            .join(" ")
    };
    println!(
        "  put 40 20 60 10 30 50 70: Len = {}, Height = {}, Check passes = {}",
        t.len(),
        t.height(),
        t.check().is_ok()
    );
    println!(
        "  Get(30) = {}, Get(35) found = {}, Min = {}, Max = {}",
        t.get(&30).unwrap(),
        t.get(&35).is_some(),
        show(t.min()),
        show(t.max())
    );
    println!(
        "  Floor(35) = {}, Ceil(35) = {}, Pred(40) = {}, Succ(40) = {}, Floor(5) = {}",
        show(t.floor(&35)),
        show(t.ceil(&35)),
        show(t.pred(&40)),
        show(t.succ(&40)),
        show(t.floor(&5))
    );
    println!(
        "  Rank(35) = {} keys below it, Select(3) = {}, Range(25, 60) = {}",
        t.rank(&35),
        show(t.select(3)),
        pairs(&t, 25, 60)
    );
    let replaced = t.put(30, "new".to_string()).is_some();
    let first = t.delete(&40).is_some();
    let again = t.delete(&40).is_some();
    println!(
        "  Put(30) again replaced = {replaced}; Delete(40), a node with two children = {first}; Delete(40) again = {again}"
    );
    println!(
        "  now {}, Len = {}, Check passes = {}",
        pairs(&t, 0, 100),
        t.len(),
        t.check().is_ok()
    );
}

// (2c) every insertion order x every deletion order of the keys 0..n-1.
fn permutations(n: usize) -> Vec<Vec<i64>> {
    if n == 0 {
        return vec![vec![]];
    }
    let mut out = Vec::new();
    for p in permutations(n - 1) {
        for i in 0..=p.len() {
            let mut q = p.clone();
            q.insert(i, (n - 1) as i64);
            out.push(q);
        }
    }
    out
}

// present: the tree holds exactly the keys whose bit is set in mask, and passes check.
fn present(t: &Tree<i64, i64>, mut mask: u64) -> bool {
    if t.check().is_err() || t.len() != mask.count_ones() as usize {
        return false;
    }
    for (&k, _) in t.range(&0, &63) {
        if mask == 0 || i64::from(mask.trailing_zeros()) != k {
            return false;
        }
        mask &= mask - 1;
    }
    mask == 0
}

fn exhaustive() {
    let mut failures = 0;
    for n in 1..=6 {
        let perms = permutations(n);
        for ins in &perms {
            for del in &perms {
                let (mut t, mut mask) = (Tree::new(), 0u64);
                for &k in ins {
                    mask |= 1 << k;
                    failures += usize::from(t.put(k, k).is_some() || !present(&t, mask));
                }
                for &k in del {
                    mask &= !(1 << k);
                    failures += usize::from(t.delete(&k).is_none() || !present(&t, mask));
                }
            }
        }
        let p = perms.len();
        println!(
            "  n = {n}: {p} x {p} = {:>6} (insertion order, deletion order) pairs, {:>7} operations checked",
            p * p,
            p * p * 2 * n
        );
    }
    println!("  {failures} failures");
}

fn proofs() {
    println!(
        "(2a) narrow keys [0, 64): 200000 random operations (plus a burst of 400 deletes every 5000) against the sorted-slice model, Check() after every one"
    );
    let cfg = Cfg {
        ops: 200_000,
        key_range: 64,
        width: 16,
        every: 1,
        wide: false,
        drain: true,
        stop: false,
    };
    let mut bt = BTreeMap::new();
    let s = run(&mut Tree::new(), &mut Pcg::new(1, 2), &cfg, Some(&mut bt));
    println!(
        "  {} mismatches, {} invariant failures over {} operations",
        s.mismatches, s.failures, cfg.ops
    );
    let mix: Vec<String> = OP_NAMES
        .iter()
        .zip(s.counts)
        .map(|(name, c)| format!("{name} {c}"))
        .collect();
    println!("  mix: {}", mix.join(", "));
    println!(
        "  coverage: {} puts replaced a key, {} deletes hit and {} missed, the tree was empty after {} operations, at most {} keys",
        s.replaced, s.hit, s.miss, s.empty, s.max_len
    );
    println!(
        "  Rust only, the same stream against std's BTreeMap (floor = range(..=k).next_back(), select = iter().nth(i)): {} mismatches",
        s.bt_mismatches
    );

    println!(
        "\n(2b) wide keys [0, 2^20): 100000 operations, Check() and a full in-order comparison every 1000"
    );
    let cfg = Cfg {
        ops: 100_000,
        key_range: 1 << 20,
        width: 1 << 14,
        every: 1000,
        wide: true,
        drain: false,
        stop: false,
    };
    let s = run(&mut Tree::new(), &mut Pcg::new(2, 2), &cfg, None);
    println!(
        "  {} mismatches, {} invariant failures over {} operations; {} deletes hit and {} missed, at most {} keys",
        s.mismatches, s.failures, cfg.ops, s.hit, s.miss, s.max_len
    );

    println!(
        "\n(2c) every insertion order x every deletion order of the keys 0..n-1, Check() and the key set verified after every operation:"
    );
    exhaustive();

    println!(
        "\n(2d) a planted bug: detach_min skips the size repair; the same stream as (2a), stopped at the first failure"
    );
    let cfg = Cfg {
        ops: 200_000,
        key_range: 64,
        width: 16,
        every: 1,
        wide: false,
        drain: true,
        stop: true,
    };
    let mut bad = Tree::new();
    bad.fault = true;
    let s = run(&mut bad, &mut Pcg::new(1, 2), &cfg, None);
    if let Some((op, kind, err)) = s.first {
        println!("  Check() failed at operation {op} (a {kind}): {err}");
    }

    // Rust only: BTreeMap has no select. iter().nth(k) is the default Iterator::nth,
    // k + 1 calls to next() (read in the source, and counted here).
    let bt: BTreeMap<i64, i64> = (0..100).map(|k| (k, k)).collect();
    let mut calls = 0;
    let found = bt.iter().inspect(|_| calls += 1).nth(40);
    println!(
        "\n(2e) Rust only: BTreeMap has no select. iter().nth(40) on 100 keys returned {:?} after {calls} calls to next()",
        found.map(|(&k, _)| k)
    );
}

// ===================================================================
// (3) measure
// ===================================================================

fn draw_distinct(rng: &mut Pcg, n: usize, key_range: usize) -> Vec<i64> {
    let mut seen = HashSet::with_capacity(n);
    let mut keys = Vec::with_capacity(n);
    while keys.len() < n {
        let k = rng.int_n(key_range) as i64;
        if seen.insert(k) {
            keys.push(k);
        }
    }
    keys
}

fn shape() {
    const N: usize = 4000;
    let keys = draw_distinct(&mut Pcg::new(3, 2), N, 1 << 30);
    let mut sorted = keys.clone();
    sorted.sort_unstable();
    let mut zigzag = Vec::with_capacity(N);
    for i in 0..N / 2 {
        zigzag.push(sorted[i]);
        zigzag.push(sorted[N - 1 - i]);
    }
    println!(
        "  the same {N} keys put in three orders (zig-zag = smallest, largest, 2nd smallest, 2nd largest, ...):"
    );
    println!(
        "  {:<8} {:>7} {:>14} {:>14} {:>13} {:>6}",
        "order", "height", "Put cmps", "Get cmps (all)", "mean per Get", "Check"
    );
    let mut identity = true;
    for (name, seq) in [("random", &keys), ("sorted", &sorted), ("zig-zag", &zigzag)] {
        let mut t = Tree::new();
        for &k in seq {
            t.put(k, k);
        }
        let put = t.cmps.replace(0);
        for k in seq {
            t.get(k);
        }
        let get = t.cmps.get();
        identity = identity && get == put + N as u64;
        println!(
            "  {name:<8} {:>7} {put:>14} {get:>14} {:>13.2} {:>6}",
            t.height(),
            get as f64 / N as f64,
            t.check().is_ok()
        );
    }
    println!(
        "  Get cmps (all) = Put cmps + n in every row: {identity} (a key's Put cost is its depth, its Get cost is depth + 1)"
    );
}

// Bests collects, per table row and column (Tree, sorted slice, BTreeMap), the best
// ns per operation over the rounds, and the checksum the last run computed.
struct Bests {
    ns: [[f64; 3]; 6],
    sum: [i64; 3],
}

impl Bests {
    // run times f once and keeps the best time; f returns a checksum of what it read.
    fn run(&mut self, row: usize, col: usize, reps: usize, f: impl FnOnce() -> i64) {
        let start = Instant::now();
        let sum = black_box(f());
        let ns = start.elapsed().as_nanos() as f64 / reps as f64;
        self.ns[row][col] = self.ns[row][col].min(ns);
        self.sum[col] = sum;
    }

    fn agree(&self) -> bool {
        self.sum[0] == self.sum[1] && self.sum[0] == self.sum[2]
    }
}

fn timings() {
    const N: usize = 100_000;
    const M: usize = 10_000;
    const G: usize = 200_000;
    const S: usize = 1_000_000;
    const BT_S: usize = 1000; // BTreeMap select is Theta(i): fewer calls
    const R: usize = 100_000;
    const W: usize = 50;
    let mut b = Bests {
        ns: [[f64::MAX; 3]; 6],
        sum: [0; 3],
    };
    let mut ok = true;
    for _ in 0..3 {
        // best of 3 rounds, each from the same seeded data
        let mut rng = Pcg::new(4, 2);
        let keys = draw_distinct(&mut rng, N + M, 1 << 30); // the first N are built, the last M are put later
        let probes: Vec<i64> = (0..G).map(|_| keys[rng.int_n(N + M)]).collect();
        let sel: Vec<usize> = (0..S).map(|_| rng.int_n(N)).collect();
        let starts: Vec<usize> = (0..R).map(|_| rng.int_n(N - 100)).collect();
        let mut victims = keys.clone();
        for i in 0..M {
            // partial Fisher-Yates: M random keys to delete
            let j = i + rng.int_n(victims.len() - i);
            victims.swap(i, j);
        }
        victims.truncate(M);

        let mut t = Tree::new();
        let mut m: Vec<Pair> = Vec::with_capacity(N + M);
        let mut bt = BTreeMap::new();
        for &k in &keys[..N] {
            t.put(k, k);
            bt.insert(k, k);
            m.push((k, k));
        }
        m.sort_unstable_by_key(|p| p.0);

        // Go keeps the phases as a table of closures that share t and m; two Rust
        // closures cannot both borrow t mutably, so each phase is one call, in order.
        b.run(0, 0, M, || {
            keys[N..]
                .iter()
                .map(|&k| t.put(k, k).map_or(0, |_| 1))
                .sum()
        });
        b.run(0, 1, M, || {
            keys[N..]
                .iter()
                .map(|&k| {
                    m.insert(find(&m, k).0, (k, k));
                    0
                })
                .sum()
        });
        b.run(0, 2, M, || {
            keys[N..]
                .iter()
                .map(|&k| bt.insert(k, k).map_or(0, |_| 1))
                .sum()
        });
        b.run(1, 0, G, || probes.iter().map(|k| *t.get(k).unwrap()).sum());
        b.run(1, 1, G, || probes.iter().map(|&k| m[find(&m, k).0].1).sum());
        b.run(1, 2, G, || probes.iter().map(|k| bt[k]).sum());
        ok = ok && b.agree();
        b.run(2, 0, M, || {
            victims
                .iter()
                .map(|k| i64::from(t.delete(k).is_some()))
                .sum()
        });
        b.run(2, 1, M, || {
            victims
                .iter()
                .map(|&k| match find(&m, k) {
                    (j, true) => {
                        m.remove(j);
                        1
                    }
                    _ => 0,
                })
                .sum()
        });
        b.run(2, 2, M, || {
            victims
                .iter()
                .map(|k| i64::from(bt.remove(k).is_some()))
                .sum()
        });
        ok = ok && b.agree() && t.len() == N && m.len() == N && bt.len() == N && t.check().is_ok();
        b.run(3, 0, S, || {
            sel.iter().map(|&i| *t.select(i).unwrap().0).sum()
        });
        b.run(3, 1, S, || sel.iter().map(|&i| m[i].0).sum());
        let tree_sum: i64 = sel[..BT_S].iter().map(|&i| *t.select(i).unwrap().0).sum();
        b.run(3, 2, BT_S, || {
            sel[..BT_S]
                .iter()
                .map(|&i| *bt.iter().nth(i).unwrap().0)
                .sum()
        });
        ok = ok && b.sum[1] == b.sum[0] && b.sum[2] == tree_sum;
        // select n/2 by walking in order: our Range iterator, and BTreeMap's iter
        let (lo, hi) = (m[0].0, m[N - 1].0);
        b.run(4, 0, W, || {
            (0..W)
                .map(|_| *t.range(&lo, &hi).nth(N / 2).unwrap().0)
                .sum()
        });
        b.run(4, 2, W, || {
            (0..W).map(|_| *bt.iter().nth(N / 2).unwrap().0).sum()
        });
        ok = ok && b.sum[0] == b.sum[2];
        b.run(5, 0, R, || {
            starts
                .iter()
                .map(|&r| {
                    t.range(&m[r].0, &m[r + 99].0)
                        .map(|(k, v)| k + v)
                        .sum::<i64>()
                })
                .sum()
        });
        b.run(5, 1, R, || {
            starts
                .iter()
                .map(|&r| {
                    let j = find(&m, m[r].0).0;
                    m[j..j + 100].iter().map(|p| p.0 + p.1).sum::<i64>()
                })
                .sum()
        });
        b.run(5, 2, R, || {
            starts
                .iter()
                .map(|&r| {
                    bt.range(m[r].0..=m[r + 99].0)
                        .map(|(k, v)| k + v)
                        .sum::<i64>()
                })
                .sum()
        });
        ok = ok && b.agree();
    }
    println!(
        "  n = {N} random keys; Put grows the map to {}, Delete brings it back to {N}; best of 3 rounds, ns per operation:",
        N + M
    );
    println!(
        "  {:<34} {:>8} {:>10} {:>12} {:>10} {:>10}",
        "operation", "reps", "Tree", "sorted slice", "slice/Tree", "BTreeMap"
    );
    let rows = [
        ("Put", M),
        ("Get", G),
        ("Delete", M),
        ("Select(random i)", S),
        ("Select(n/2) by walking in order", W),
        ("Range of 100 keys", R),
    ];
    for (i, (name, reps)) in rows.iter().enumerate() {
        let [tree, slice, btree] = b.ns[i];
        if i == 4 {
            println!(
                "  {name:<34} {reps:>8} {tree:>10.0} {:>12} {:>10} {btree:>10.0}",
                "-", "-"
            );
        } else {
            println!(
                "  {name:<34} {reps:>8} {tree:>10.0} {slice:>12.1} {:>10.3} {btree:>10.1}",
                slice / tree
            );
        }
    }
    println!(
        "  (BTreeMap Select(random i) is iter().nth(i), Theta(i), so it ran {BT_S} calls instead of {S}; its n/2 row is iter().nth(n/2))"
    );
    println!(
        "  tree, slice and BTreeMap read the same checksums and ended with the same sizes in every round: {ok}"
    );
    println!(
        "  walking in order to rank n/2 costs {:.0}x a Select descent",
        b.ns[4][0] / b.ns[3][0]
    );
}

fn main() {
    println!("(1) what it does");
    demo();
    println!();
    proofs();
    println!("\n(3a) the capstone has no balancing: n = 4000 (lesson 19 fixes this)");
    shape();
    println!("\n(3b) timings");
    timings();
}
```

**Rust sample output:**

```
(1) what it does
  put 40 20 60 10 30 50 70: Len = 7, Height = 3, Check passes = true
  Get(30) = v30, Get(35) found = false, Min = 10=v10, Max = 70=v70
  Floor(35) = 30=v30, Ceil(35) = 40=v40, Pred(40) = 30=v30, Succ(40) = 50=v50, Floor(5) = none
  Rank(35) = 3 keys below it, Select(3) = 40=v40, Range(25, 60) = 30=v30 40=v40 50=v50 60=v60
  Put(30) again replaced = true; Delete(40), a node with two children = true; Delete(40) again = false
  now 10=v10 20=v20 30=new 50=v50 60=v60 70=v70, Len = 6, Check passes = true

(2a) narrow keys [0, 64): 200000 random operations (plus a burst of 400 deletes every 5000) against the sorted-slice model, Check() after every one
  0 mismatches, 0 invariant failures over 200000 operations
  mix: put 39461, delete 55427, get 13225, floor 13000, ceil 13154, pred 13215, succ 13071, rank 13260, select 13162, range 13025
  coverage: 19147 puts replaced a key, 20280 deletes hit and 35147 missed, the tree was empty after 6814 operations, at most 47 keys
  Rust only, the same stream against std's BTreeMap (floor = range(..=k).next_back(), select = iter().nth(i)): 0 mismatches

(2b) wide keys [0, 2^20): 100000 operations, Check() and a full in-order comparison every 1000
  0 mismatches, 0 invariant failures over 100000 operations; 5365 deletes hit and 16038 missed, at most 16033 keys

(2c) every insertion order x every deletion order of the keys 0..n-1, Check() and the key set verified after every operation:
  n = 1: 1 x 1 =      1 (insertion order, deletion order) pairs,       2 operations checked
  n = 2: 2 x 2 =      4 (insertion order, deletion order) pairs,      16 operations checked
  n = 3: 6 x 6 =     36 (insertion order, deletion order) pairs,     216 operations checked
  n = 4: 24 x 24 =    576 (insertion order, deletion order) pairs,    4608 operations checked
  n = 5: 120 x 120 =  14400 (insertion order, deletion order) pairs,  144000 operations checked
  n = 6: 720 x 720 = 518400 (insertion order, deletion order) pairs, 6220800 operations checked
  0 failures

(2d) a planted bug: detach_min skips the size repair; the same stream as (2a), stopped at the first failure
  Check() failed at operation 517 (a delete): key 26: size field 6, recounted 1 + 2 + 2 = 5

(2e) Rust only: BTreeMap has no select. iter().nth(40) on 100 keys returned Some(40) after 41 calls to next()

(3a) the capstone has no balancing: n = 4000 (lesson 19 fixes this)
  the same 4000 keys put in three orders (zig-zag = smallest, largest, 2nd smallest, 2nd largest, ...):
  order     height       Put cmps Get cmps (all)  mean per Get  Check
  random        28          52102          56102         14.03   true
  sorted      4000        7998000        8002000       2000.50   true
  zig-zag     4000        7998000        8002000       2000.50   true
  Get cmps (all) = Put cmps + n in every row: true (a key's Put cost is its depth, its Get cost is depth + 1)

(3b) timings
  n = 100000 random keys; Put grows the map to 110000, Delete brings it back to 100000; best of 3 rounds, ns per operation:
  operation                              reps       Tree sorted slice slice/Tree   BTreeMap
  Put                                   10000        211      11903.2     56.480      133.1
  Get                                  200000        129         18.9      0.147       68.2
  Delete                                10000        201      11901.7     59.326      111.2
  Select(random i)                    1000000        148          0.7      0.005    91350.8
  Select(n/2) by walking in order          50     373349            -          -      94299
  Range of 100 keys                    100000       1042         83.6      0.080      318.9
  (BTreeMap Select(random i) is iter().nth(i), Theta(i), so it ran 1000 calls instead of 1000000; its n/2 row is iter().nth(n/2))
  tree, slice and BTreeMap read the same checksums and ended with the same sizes in every round: true
  walking in order to rank n/2 costs 2519x a Select descent
```

> **Go vs Rust:** Every section except the timings prints exactly what the Go program prints (diffed against a live run), apart from the Rust-only lines and `detachMin` spelled `detach_min`; the twin adds a `BTreeMap` oracle that agreed on all 200,000 narrow-range operations (floor is `range(..=k).next_back()`, select is `iter().nth(i)`, which made 41 `next()` calls for index 40) and a `BTreeMap` timing column. The `Drop` sits on `Tree` rather than `Node` because a type with a `Drop` impl cannot have a field moved out of it, and `delete` hands the value back as `Option<V>` where Go returns a bool. The timings are not Go's: the sorted slice's `Get` took 18-20 ns here against 96-101 ns in Go, so the slice beat the tree's `Get` by 6-7x in Rust and only by 1.1-1.2x in Go, and walking in order to rank n/2 took 190-380 µs here against 670-700 µs in Go; `binary_search_by` is written with no early exit and with conditional moves (read in the source, not run), while Go's `BinarySearchFunc` is a plain if/else, and what either compiler emitted was not checked.

**Complexity:** every operation Θ(h): Θ(log n) expected for random insertion order, Θ(n) for sorted order · `Select`/`Rank` Θ(h) through the size field · `Range` Θ(h + m) · `Check()` Θ(n). Proofs (counts, identical in Go and Rust): 0 mismatches and 0 invariant failures over 200,000 narrow-range operations (the tree was empty after about 6,800 of them) and 100,000 wide-range ones; 0 failures over all 533,417 (insertion order, deletion order) pairs for n = 1..6, 518,400 of them at n = 6; the planted size bug is reported by `Check()` at operation 517 with no panic. At n = 4,000 the same keys give height 28 and 14.03 comparisons per `Get` in random order, and height 4,000 and 2,000.50 per `Get` in sorted and zig-zag order (7,998,000 comparisons to build), the missing rebalancing that lesson 19 repairs. Timings at n = 100,000, best of 3 rounds in each of several runs on a loaded machine (the ranges, not one run): inserting or deleting in the sorted slice cost about 11.8-12.1 µs against 175-230 ns in the tree, 53-68x; `Get` was 108-123 ns in the tree and 96-101 ns in the slice, the slice ahead in every run; `Select` by descent 135-145 ns against 670-700 µs for walking in order, about 4,700-5,200x; a 100-key `Range` 1.65-1.8 µs in the tree against 142-147 ns in the slice, which was 11-12x faster. The tree wins where the slice must move elements (`Put`, `Delete`) and loses where it only reads.

---

> Next lesson: [19 — Balanced Trees & B-Trees](../../19-balanced-trees.md) — rotations, AVL and red-black
> trees, and the B-tree behind Rust's `BTreeMap`.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
