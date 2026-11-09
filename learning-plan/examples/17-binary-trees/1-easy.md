# Step 17 — Binary Trees & Traversals · 🟢 Easy

Examples **1–6**: the node and its `nil` base case, the three depth-first orders, the call stack written as
a loop, level order, height, depth and diameter, and changing a tree by assigning what a child returns.

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

## 1. The node, and `nil` as the empty tree

`🟢 easy` · *Node / nil base case*

A binary tree is either empty or a node with two binary trees, so the empty tree is itself a value (`nil` in Go, `None` in Rust) and every function begins by asking whether it has been handed one. That single case is the whole base of the recursion, and counting calls makes the structural fact behind every traversal in this lesson visible: an n-node tree has n + 1 empty links, so a full walk makes exactly 2n + 1 calls, n that do work and n + 1 that hit the base case, whatever the shape. Time is Θ(n); what depends on the shape is the stack (the height `h`), and the table has four 1,000-node shapes whose heights run from 10 to 1000 and whose counts are identical.

**Steps:**

1. Build the hand-built tree and print it sideways; write `size` with three counters (calls, work, nil hits) and an independent scan that counts nil child links by looking at them.
2. Run both over seven shapes, from the empty tree to a 1,000-node chain, and compare `calls` with 2n + 1 and `nil hits` with n + 1; then call a version without the base case.
3. Read the `holds` column, and see that `h` changes by a factor of 100 while `calls` does not; see what the missing base case does, and what a node weighs.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"strings"
	"unsafe"
)

// A binary tree is either EMPTY, or a node holding a value and two binary
// trees. The empty tree is a value too -- nil -- and that is what lets every
// function begin with the same question: "is there anything here?"
type Node struct {
	Val         int
	Left, Right *Node
}

// Size runs happily on a nil receiver: Go lets a method be called on a nil
// pointer, so the base case can live inside the method itself.
func (n *Node) Size() int {
	if n == nil {
		return 0
	}
	return 1 + n.Left.Size() + n.Right.Size()
}

// counts says what a call did. Every call lands in exactly one of two cases.
type counts struct{ calls, work, nilHits int }

func (c *counts) size(n *Node) int {
	c.calls++
	if n == nil { // the empty tree: nothing to do
		c.nilHits++
		return 0
	}
	c.work++ // a real node: do one unit of work, ask both children
	return 1 + c.size(n.Left) + c.size(n.Right)
}

// nilLinks counts empty links by LOOKING at them: pop nodes off a worklist and
// test Left and Right. No recursive calls, so it cannot share a bug with size.
// The root pointer is a link too, so the empty tree has exactly one nil link.
func nilLinks(root *Node) int {
	if root == nil {
		return 1
	}
	nils, work := 0, []*Node{root}
	for len(work) > 0 {
		n := work[len(work)-1]
		work = work[:len(work)-1]
		for _, c := range []*Node{n.Left, n.Right} {
			if c == nil {
				nils++
			} else {
				work = append(work, c)
			}
		}
	}
	return nils
}

func height(n *Node) int { // nodes on the longest root-to-leaf path; empty = 0
	if n == nil {
		return 0
	}
	return 1 + max(height(n.Left), height(n.Right))
}

// badSize forgot the empty case. It works until it reaches a leaf, whose
// children are nil, and then it reads n.Left through a nil pointer.
func badSize(n *Node) int { return 1 + badSize(n.Left) + badSize(n.Right) }

func tryBad(t *Node) {
	defer func() { fmt.Println("  recovered:", recover()) }()
	fmt.Println("  badSize =", badSize(t))
}

// ---- shapes: the chains are built in a loop (no recursion), the others recurse
// only as deep as their own height, which is small ----

func leaf(v int) *Node { return &Node{Val: v} }

// handBuilt is the 9-node tree used by examples 1-6; its drawing is inside.
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

// complete is heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
func complete(i, n int) *Node {
	if i >= n {
		return nil
	}
	return &Node{Val: i, Left: complete(2*i+1, n), Right: complete(2*i+2, n)}
}

func leftChain(n int) *Node { // each node's only child is on the left; Val = depth
	var root *Node
	for i := n - 1; i >= 0; i-- {
		root = &Node{Val: i, Left: root}
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

// randomSplit: the left subtree gets a uniformly random share of the other n-1
// nodes. The draw happens BEFORE the recursion, so the stream is consumed in a
// fixed order (the Rust twin makes the same draws and builds the same tree).
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{Left: left, Right: right}
}

// sideways prints the right subtree ABOVE the node and the left BELOW it.
func sideways(n *Node, depth int) {
	if n == nil {
		return
	}
	sideways(n.Right, depth+1)
	fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.Val)
	sideways(n.Left, depth+1)
}

func main() {
	root := handBuilt()
	fmt.Println("the hand-built tree, sideways (right subtree above, left below):")
	sideways(root, 0)

	r := rand.New(rand.NewPCG(1, 2))
	shapes := []struct {
		name string
		root *Node
	}{
		{"empty", nil},
		{"hand-built", root},
		{"complete(15)", complete(0, 15)},
		{"complete(1000)", complete(0, 1000)},
		{"randomSplit(1000)", randomSplit(1000, r)},
		{"leftChain(1000)", leftChain(1000)},
		{"zigzag(1000)", zigzag(1000)},
	}

	fmt.Println()
	fmt.Printf("%-18s %5s %5s %6s %6s %8s %8s %6s %6s %6s\n",
		"shape", "n", "h", "calls", "work", "nil hits", "scanned", "2n+1", "n+1", "holds")
	for _, s := range shapes {
		var c counts
		n := c.size(s.root)
		scanned := nilLinks(s.root)
		holds := c.calls == 2*n+1 && c.nilHits == n+1 && scanned == c.nilHits
		fmt.Printf("%-18s %5d %5d %6d %6d %8d %8d %6d %6d %6v\n",
			s.name, n, height(s.root), c.calls, c.work, c.nilHits, scanned, 2*n+1, n+1, holds)
	}

	fmt.Println()
	fmt.Println("forgetting the base case:")
	tryBad(root)
	var empty *Node
	fmt.Println("  (*Node)(nil).Size() =", empty.Size(), "-- the method has the base case built in")

	fmt.Println()
	fmt.Printf("unsafe.Sizeof(Node{}) = %d bytes (Val %d + Left %d + Right %d)\n",
		unsafe.Sizeof(Node{}), unsafe.Sizeof(root.Val), unsafe.Sizeof(root.Left), unsafe.Sizeof(root.Right))
}
```

**Output:**

```
the hand-built tree, sideways (right subtree above, left below):
            9
        6
            8
    3
1
        5
            7
    2
        4

shape                  n     h  calls   work nil hits  scanned   2n+1    n+1  holds
empty                  0     0      1      0        1        1      1      1   true
hand-built             9     4     19      9       10       10     19     10   true
complete(15)          15     4     31     15       16       16     31     16   true
complete(1000)      1000    10   2001   1000     1001     1001   2001   1001   true
randomSplit(1000)   1000    23   2001   1000     1001     1001   2001   1001   true
leftChain(1000)     1000  1000   2001   1000     1001     1001   2001   1001   true
zigzag(1000)        1000  1000   2001   1000     1001     1001   2001   1001   true

forgetting the base case:
  recovered: runtime error: invalid memory address or nil pointer dereference
  (*Node)(nil).Size() = 0 -- the method has the base case built in

unsafe.Sizeof(Node{}) = 24 bytes (Val 8 + Left 8 + Right 8)
```

**Rust:**

```rust
use std::mem::size_of;

// A binary tree is either EMPTY, or a node holding a value and two binary
// trees. In Rust the empty tree is `None`: a link is `Option<Box<Node>>`.
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Go needs no destructor. Rust's default drop glue recurses once per level, so a
// chain of 1,000 nodes is fine but a chain of millions is not (example 13
// measures it). This Drop hands the children to a heap-allocated stack instead.
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

/// What a call did. Every call lands in exactly one of two cases.
#[derive(Default)]
struct Counts {
    calls: usize,
    work: usize,
    nil_hits: usize,
}

// The `match` must name both cases: leave out `None` and rustc refuses to
// compile (see the end of main), so the base case cannot be forgotten.
fn size(n: &Link, c: &mut Counts) -> usize {
    c.calls += 1;
    match n {
        None => {
            c.nil_hits += 1;
            0
        }
        Some(b) => {
            c.work += 1;
            1 + size(&b.left, c) + size(&b.right, c)
        }
    }
}

// Rust has no nil receiver (a method needs a `self` that exists). The same idea
// is a function over `Option<&Node>`, which `as_deref()` produces from a link.
fn size_ref(n: Option<&Node>) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + size_ref(b.left.as_deref()) + size_ref(b.right.as_deref()),
    }
}

/// Counts empty links by LOOKING at them: pop nodes off a worklist and test
/// left and right. No recursive calls, so it cannot share a bug with `size`.
/// The root pointer is a link too, so the empty tree has exactly one nil link.
fn nil_links(root: &Link) -> usize {
    let Some(r) = root else { return 1 };
    let (mut nils, mut work): (usize, Vec<&Node>) = (0, vec![r]);
    while let Some(n) = work.pop() {
        for c in [&n.left, &n.right] {
            match c {
                None => nils += 1,
                Some(b) => work.push(b),
            }
        }
    }
    nils
}

/// Nodes on the longest root-to-leaf path; the empty tree is 0.
fn height(n: &Link) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + height(&b.left).max(height(&b.right)),
    }
}

// ---- shapes: the chains are built in a loop (no recursion), the others recurse
// only as deep as their own height, which is small ----

fn node(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

fn leaf(val: i64) -> Link {
    node(val, None, None)
}

//         1
//       /   \
//      2     3
//     / \     \
//    4   5     6
//       /     / \
//      7     8   9
fn hand_built() -> Link {
    node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    )
}

/// Heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
fn complete(i: usize, n: usize) -> Link {
    if i >= n {
        None
    } else {
        node(i as i64, complete(2 * i + 1, n), complete(2 * i + 2, n))
    }
}

/// Each node's only child is on the left; val = depth.
fn left_chain(n: usize) -> Link {
    let mut root = None;
    for i in (0..n).rev() {
        root = node(i as i64, root, None);
    }
    root
}

/// Node i hangs off node i-1: right if i is odd, left if i is even.
fn zigzag(n: usize) -> Link {
    let mut root = None;
    for i in (0..n).rev() {
        root = if (i + 1) % 2 == 1 {
            node(i as i64, None, root)
        } else {
            node(i as i64, root, None)
        };
    }
    root
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of IntN, copied
// from lesson 01 example 14, so the same seed builds the same tree as in Go.
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

    /// Like rand.IntN(n): exactly uniform in [0, n), Lemire's method.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

/// The left subtree gets a uniformly random share of the other n-1 nodes. The
/// draw happens BEFORE the recursion, in the same order as the Go program.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    node(0, left, right)
}

/// Prints the right subtree ABOVE the node and the left BELOW it.
fn sideways(n: &Link, depth: usize) {
    if let Some(b) = n {
        sideways(&b.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), b.val);
        sideways(&b.left, depth + 1);
    }
}

fn main() {
    let root = hand_built();
    println!("the hand-built tree, sideways (right subtree above, left below):");
    sideways(&root, 0);

    let mut r = Pcg::new(1, 2);
    let shapes: Vec<(&str, Link)> = vec![
        ("empty", None),
        ("hand-built", hand_built()),
        ("complete(15)", complete(0, 15)),
        ("complete(1000)", complete(0, 1000)),
        ("randomSplit(1000)", random_split(1000, &mut r)),
        ("leftChain(1000)", left_chain(1000)),
        ("zigzag(1000)", zigzag(1000)),
    ];

    println!();
    println!(
        "{:<18} {:>5} {:>5} {:>6} {:>6} {:>8} {:>8} {:>6} {:>6} {:>6}",
        "shape", "n", "h", "calls", "work", "nil hits", "scanned", "2n+1", "n+1", "holds"
    );
    for (name, tree) in &shapes {
        let mut c = Counts::default();
        let n = size(tree, &mut c);
        let scanned = nil_links(tree);
        let holds = c.calls == 2 * n + 1 && c.nil_hits == n + 1 && scanned == c.nil_hits;
        println!(
            "{:<18} {:>5} {:>5} {:>6} {:>6} {:>8} {:>8} {:>6} {:>6} {:>6}",
            name,
            n,
            height(tree),
            c.calls,
            c.work,
            c.nil_hits,
            scanned,
            2 * n + 1,
            n + 1,
            holds
        );
    }

    println!();
    println!("forgetting the base case:");
    println!("  a `match` with only the `Some(b) => ..` arm does not compile (error[E0004]:");
    println!(
        "  non-exhaustive patterns: `&None` not covered), so there is nothing to recover from"
    );
    println!(
        "  size_ref(None) = {} -- the same idea as a nil receiver, written as a function",
        size_ref(None)
    );
    println!(
        "  size_ref(hand-built.as_deref()) = {}",
        size_ref(root.as_deref())
    );

    println!();
    println!(
        "size_of::<Node>() = {} bytes (val {} + left {} + right {})",
        size_of::<Node>(),
        size_of::<i64>(),
        size_of::<Link>(),
        size_of::<Link>()
    );
    println!(
        "size_of::<Option<Box<Node>>>() = {} and size_of::<Box<Node>>() = {}: None is the null pointer, for free",
        size_of::<Option<Box<Node>>>(),
        size_of::<Box<Node>>()
    );
}
```

**Rust output:**

```
the hand-built tree, sideways (right subtree above, left below):
            9
        6
            8
    3
1
        5
            7
    2
        4

shape                  n     h  calls   work nil hits  scanned   2n+1    n+1  holds
empty                  0     0      1      0        1        1      1      1   true
hand-built             9     4     19      9       10       10     19     10   true
complete(15)          15     4     31     15       16       16     31     16   true
complete(1000)      1000    10   2001   1000     1001     1001   2001   1001   true
randomSplit(1000)   1000    23   2001   1000     1001     1001   2001   1001   true
leftChain(1000)     1000  1000   2001   1000     1001     1001   2001   1001   true
zigzag(1000)        1000  1000   2001   1000     1001     1001   2001   1001   true

forgetting the base case:
  a `match` with only the `Some(b) => ..` arm does not compile (error[E0004]:
  non-exhaustive patterns: `&None` not covered), so there is nothing to recover from
  size_ref(None) = 0 -- the same idea as a nil receiver, written as a function
  size_ref(hand-built.as_deref()) = 9

size_of::<Node>() = 24 bytes (val 8 + left 8 + right 8)
size_of::<Option<Box<Node>>>() = 8 and size_of::<Box<Node>>() = 8: None is the null pointer, for free
```

> **Go vs Rust:** All seven table rows are identical to the Go run (the `PCG(1, 2)` port builds the same `randomSplit(1000)`, height 23). Go lets a method run on a nil receiver, which Rust cannot (a method needs a `self`), so the same idea is a function over `Option<&Node>`; the forgotten base case is a recovered runtime panic in Go and, in Rust, a compile error — the `E0004` text above is what `rustc` printed when I compiled a `match` without the `None` arm separately, it is not produced by this program. `Option<Box<Node>>` is 8 bytes like `Box<Node>`, so `None` costs nothing, and the `Drop` impl is Rust-only (Go has no destructor to write).

**Complexity:** Θ(n) time and Θ(h) stack for `size`; **n + 1 nil links and 2n + 1 calls, exactly**, on every shape in the table: 2,001 calls and 1,001 nil hits for each of the four 1,000-node trees even though their heights are 10, 23, 1000 and 1000, and 1 call for the empty tree. The independent scan of the child links agrees with the call counts in all seven rows. A node is 24 bytes (three 8-byte fields).

---

## 2. Pre, in and post: where the visit goes

`🟢 easy` · *Visit position*

One recursive skeleton touches every real node three times (on the way down, between the two children, on the way up), and the three depth-first orders are only which touch you keep. Printing the event trace first makes the orders fall out of the trace instead of being memorised, and a short expression tree shows what each order is for: pre-order is prefix notation, in-order is infix, post-order is postfix and is what evaluation needs. All three are Θ(n) time and Θ(h) stack, and all three make the same 2n + 1 calls, because moving the visit changes no call.

**Steps:**

1. Write `walk(n, depth, emit)` with three `emit` calls around the two recursive calls; run it on the hand-built tree and print the 27 events (3 per node), indented by depth.
2. Derive each order by keeping one kind of event, then write the three tiny recursions where only the line that appends to `out` moves, and compare the two.
3. Read the counts: calls, calls on real nodes and the deepest stack are the same for all three orders; then read the expression tree three ways and evaluate it in post-order.

```go
package main

import (
	"fmt"
	"slices"
	"strconv"
	"strings"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func leaf(v int) *Node { return &Node{Val: v} }

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
func handBuilt() *Node {
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// walk is THE skeleton. Each real node is touched three times: on the way
// down (before either child), between the children, and on the way up. Pre-,
// in- and post-order differ only in which of the three touches you keep.
func walk(n *Node, depth int, emit func(kind string, val, depth int)) {
	if n == nil {
		return // a nil call touches nothing
	}
	emit("enter", n.Val, depth)
	walk(n.Left, depth+1, emit)
	emit("between", n.Val, depth)
	walk(n.Right, depth+1, emit)
	emit("leave", n.Val, depth)
}

// Three tiny recursions: the SAME four lines, with the visit moved. stats
// counts calls, calls on real nodes, and the deepest stack (the root is 1).
type stats struct{ calls, work, deepest int }

func (s *stats) enter(n *Node, depth int) bool {
	s.calls++
	s.deepest = max(s.deepest, depth+1)
	if n != nil {
		s.work++
	}
	return n != nil
}

func pre(n *Node, depth int, s *stats, out *[]int) {
	if !s.enter(n, depth) {
		return
	}
	*out = append(*out, n.Val) // visit BEFORE the children
	pre(n.Left, depth+1, s, out)
	pre(n.Right, depth+1, s, out)
}

func in(n *Node, depth int, s *stats, out *[]int) {
	if !s.enter(n, depth) {
		return
	}
	in(n.Left, depth+1, s, out)
	*out = append(*out, n.Val) // visit BETWEEN the children
	in(n.Right, depth+1, s, out)
}

func post(n *Node, depth int, s *stats, out *[]int) {
	if !s.enter(n, depth) {
		return
	}
	post(n.Left, depth+1, s, out)
	post(n.Right, depth+1, s, out)
	*out = append(*out, n.Val) // visit AFTER the children
}

func height(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.Left), height(n.Right))
}

// An expression tree: a leaf holds a number (Op == 0), every other node an
// operator with two operands. Go needs a kind tag and nil children for this.
type Expr struct {
	Op   byte
	Num  int
	L, R *Expr
}

func num(v int) *Expr               { return &Expr{Num: v} }
func bin(op byte, l, r *Expr) *Expr { return &Expr{Op: op, L: l, R: r} }

func prefix(e *Expr) string { // pre-order: operator first
	if e.Op == 0 {
		return strconv.Itoa(e.Num)
	}
	return fmt.Sprintf("%c %s %s", e.Op, prefix(e.L), prefix(e.R))
}

func infix(e *Expr) string { // in-order: operator between its operands
	if e.Op == 0 {
		return strconv.Itoa(e.Num)
	}
	return fmt.Sprintf("(%s %c %s)", infix(e.L), e.Op, infix(e.R))
}

func postfix(e *Expr) string { // post-order: operator after its operands
	if e.Op == 0 {
		return strconv.Itoa(e.Num)
	}
	return fmt.Sprintf("%s %s %c", postfix(e.L), postfix(e.R), e.Op)
}

func eval(e *Expr) int { // post-order: a node needs its children's answers
	if e.Op == 0 {
		return e.Num
	}
	l, r := eval(e.L), eval(e.R)
	switch e.Op {
	case '+':
		return l + r
	case '-':
		return l - r
	}
	return l * r
}

func main() {
	root := handBuilt()
	type event struct {
		kind       string
		val, depth int
	}
	var trace []event
	fmt.Println("the event trace (indent = depth, nil calls print nothing):")
	walk(root, 0, func(kind string, val, depth int) {
		trace = append(trace, event{kind, val, depth})
		fmt.Printf("  %s%-7s %d\n", strings.Repeat("  ", depth), kind, val)
	})

	// Keeping one kind of event IS the traversal.
	keep := func(kind string) (out []int) {
		for _, e := range trace {
			if e.kind == kind {
				out = append(out, e.val)
			}
		}
		return
	}
	var s [3]stats
	var seq [3][]int
	pre(root, 0, &s[0], &seq[0])
	in(root, 0, &s[1], &seq[1])
	post(root, 0, &s[2], &seq[2])

	fmt.Println()
	fmt.Printf("events: %d = 3 x %d nodes\n", len(trace), s[0].work)
	for i, name := range []string{"pre", "in", "post"} {
		kind := []string{"enter", "between", "leave"}[i]
		fmt.Printf("%-5s keep %-8s -> %v   == recursion: %v\n",
			name, kind, keep(kind), slices.Equal(keep(kind), seq[i]))
	}

	fmt.Println()
	fmt.Printf("%-5s %6s %6s %10s\n", "order", "calls", "work", "max stack")
	for i, name := range []string{"pre", "in", "post"} {
		fmt.Printf("%-5s %6d %6d %10d\n", name, s[i].calls, s[i].work, s[i].deepest)
	}
	fmt.Printf("2n+1 = %d, height+1 = %d, the same for all three orders: %v\n",
		2*s[0].work+1, height(root)+1, s[0] == s[1] && s[1] == s[2])

	// (3 + 4) * (10 - 2)
	e := bin('*', bin('+', num(3), num(4)), bin('-', num(10), num(2)))
	fmt.Println()
	fmt.Println("the expression tree (3 + 4) * (10 - 2):")
	fmt.Println("  pre-order  (prefix): ", prefix(e))
	fmt.Println("  in-order   (infix):  ", infix(e))
	fmt.Println("  post-order (postfix):", postfix(e))
	fmt.Println("  evaluated by post-order:", eval(e))
}
```

**Output:**

```
the event trace (indent = depth, nil calls print nothing):
  enter   1
    enter   2
      enter   4
      between 4
      leave   4
    between 2
      enter   5
        enter   7
        between 7
        leave   7
      between 5
      leave   5
    leave   2
  between 1
    enter   3
    between 3
      enter   6
        enter   8
        between 8
        leave   8
      between 6
        enter   9
        between 9
        leave   9
      leave   6
    leave   3
  leave   1

events: 27 = 3 x 9 nodes
pre   keep enter    -> [1 2 4 5 7 3 6 8 9]   == recursion: true
in    keep between  -> [4 2 7 5 1 3 8 6 9]   == recursion: true
post  keep leave    -> [4 7 5 2 8 9 6 3 1]   == recursion: true

order  calls   work  max stack
pre       19      9          5
in        19      9          5
post      19      9          5
2n+1 = 19, height+1 = 5, the same for all three orders: true

the expression tree (3 + 4) * (10 - 2):
  pre-order  (prefix):  * + 3 4 - 10 2
  in-order   (infix):   ((3 + 4) * (10 - 2))
  post-order (postfix): 3 4 + 10 2 - *
  evaluated by post-order: 56
```

**Rust:**

```rust
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

fn node(val: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { val, left, right }))
}

fn leaf(val: i64) -> Link {
    node(val, None, None)
}

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
// (Only 9 nodes, so the default recursive drop is fine here.)
fn hand_built() -> Link {
    node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    )
}

// The three touches. Go passes a string; Rust names them in an enum, so a
// typo is a compile error and `match` can be checked for completeness.
#[derive(Clone, Copy, PartialEq)]
enum Event {
    Enter,
    Between,
    Leave,
}

impl Event {
    fn name(self) -> &'static str {
        match self {
            Event::Enter => "enter",
            Event::Between => "between",
            Event::Leave => "leave",
        }
    }
}

// walk is THE skeleton. Each real node is touched three times: on the way
// down (before either child), between the children, and on the way up. Pre-,
// in- and post-order differ only in which of the three touches you keep.
// `emit` is a closure that may mutate what it captured (FnMut); recursion
// reborrows it for each call.
fn walk(n: &Link, depth: usize, emit: &mut impl FnMut(Event, i64, usize)) {
    let Some(b) = n else { return }; // a nil call touches nothing
    emit(Event::Enter, b.val, depth);
    walk(&b.left, depth + 1, emit);
    emit(Event::Between, b.val, depth);
    walk(&b.right, depth + 1, emit);
    emit(Event::Leave, b.val, depth);
}

// Three tiny recursions: the SAME four lines, with the visit moved. Stats
// counts calls, calls on real nodes, and the deepest stack (the root is 1).
#[derive(Default, PartialEq)]
struct Stats {
    calls: usize,
    work: usize,
    deepest: usize,
}

impl Stats {
    fn enter(&mut self, is_node: bool, depth: usize) {
        self.calls += 1;
        self.deepest = self.deepest.max(depth + 1);
        if is_node {
            self.work += 1;
        }
    }
}

fn pre(n: &Link, depth: usize, s: &mut Stats, out: &mut Vec<i64>) {
    s.enter(n.is_some(), depth);
    let Some(b) = n else { return };
    out.push(b.val); // visit BEFORE the children
    pre(&b.left, depth + 1, s, out);
    pre(&b.right, depth + 1, s, out);
}

fn in_order(n: &Link, depth: usize, s: &mut Stats, out: &mut Vec<i64>) {
    s.enter(n.is_some(), depth);
    let Some(b) = n else { return };
    in_order(&b.left, depth + 1, s, out);
    out.push(b.val); // visit BETWEEN the children
    in_order(&b.right, depth + 1, s, out);
}

fn post(n: &Link, depth: usize, s: &mut Stats, out: &mut Vec<i64>) {
    s.enter(n.is_some(), depth);
    let Some(b) = n else { return };
    post(&b.left, depth + 1, s, out);
    post(&b.right, depth + 1, s, out);
    out.push(b.val); // visit AFTER the children
}

fn height(n: &Link) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + height(&b.left).max(height(&b.right)),
    }
}

// An expression tree is a sum type: a number, or an operator with two
// operands. No kind tag and no nil children as in Go: a Num has no operands to
// forget, and every `match` below must handle both variants.
#[derive(Clone, Copy)]
enum Op {
    Add,
    Sub,
    Mul,
}

impl Op {
    fn symbol(self) -> char {
        match self {
            Op::Add => '+',
            Op::Sub => '-',
            Op::Mul => '*',
        }
    }

    fn apply(self, l: i64, r: i64) -> i64 {
        match self {
            Op::Add => l + r,
            Op::Sub => l - r,
            Op::Mul => l * r,
        }
    }
}

enum Expr {
    Num(i64),
    Bin(Op, Box<Expr>, Box<Expr>),
}

fn bin(op: Op, l: Expr, r: Expr) -> Expr {
    Expr::Bin(op, Box::new(l), Box::new(r))
}

fn prefix(e: &Expr) -> String {
    // pre-order: operator first
    match e {
        Expr::Num(v) => v.to_string(),
        Expr::Bin(op, l, r) => format!("{} {} {}", op.symbol(), prefix(l), prefix(r)),
    }
}

fn infix(e: &Expr) -> String {
    // in-order: operator between its operands
    match e {
        Expr::Num(v) => v.to_string(),
        Expr::Bin(op, l, r) => format!("({} {} {})", infix(l), op.symbol(), infix(r)),
    }
}

fn postfix(e: &Expr) -> String {
    // post-order: operator after its operands
    match e {
        Expr::Num(v) => v.to_string(),
        Expr::Bin(op, l, r) => format!("{} {} {}", postfix(l), postfix(r), op.symbol()),
    }
}

fn eval(e: &Expr) -> i64 {
    // post-order: a node needs its children's answers
    match e {
        Expr::Num(v) => *v,
        Expr::Bin(op, l, r) => op.apply(eval(l), eval(r)),
    }
}

/// Go's %v of a slice: [1 2 3]
fn go_list(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let root = hand_built();
    let mut trace: Vec<(Event, i64, usize)> = Vec::new();
    println!("the event trace (indent = depth, nil calls print nothing):");
    walk(&root, 0, &mut |kind, val, depth| {
        trace.push((kind, val, depth));
        println!("  {}{:<7} {}", "  ".repeat(depth), kind.name(), val);
    });

    // Keeping one kind of event IS the traversal.
    let keep =
        |kind: Event| -> Vec<i64> { trace.iter().filter(|e| e.0 == kind).map(|e| e.1).collect() };
    let mut s: [Stats; 3] = Default::default();
    let mut seq: [Vec<i64>; 3] = Default::default();
    pre(&root, 0, &mut s[0], &mut seq[0]);
    in_order(&root, 0, &mut s[1], &mut seq[1]);
    post(&root, 0, &mut s[2], &mut seq[2]);

    println!();
    println!("events: {} = 3 x {} nodes", trace.len(), s[0].work);
    let orders = [
        ("pre", Event::Enter),
        ("in", Event::Between),
        ("post", Event::Leave),
    ];
    for (i, (name, kind)) in orders.iter().enumerate() {
        println!(
            "{:<5} keep {:<8} -> {}   == recursion: {}",
            name,
            kind.name(),
            go_list(&keep(*kind)),
            keep(*kind) == seq[i]
        );
    }

    println!();
    println!(
        "{:<5} {:>6} {:>6} {:>10}",
        "order", "calls", "work", "max stack"
    );
    for (i, (name, _)) in orders.iter().enumerate() {
        println!(
            "{:<5} {:>6} {:>6} {:>10}",
            name, s[i].calls, s[i].work, s[i].deepest
        );
    }
    println!(
        "2n+1 = {}, height+1 = {}, the same for all three orders: {}",
        2 * s[0].work + 1,
        height(&root) + 1,
        s[0] == s[1] && s[1] == s[2]
    );

    // (3 + 4) * (10 - 2)
    let e = bin(
        Op::Mul,
        bin(Op::Add, Expr::Num(3), Expr::Num(4)),
        bin(Op::Sub, Expr::Num(10), Expr::Num(2)),
    );
    println!();
    println!("the expression tree (3 + 4) * (10 - 2):");
    println!("  pre-order  (prefix):  {}", prefix(&e));
    println!("  in-order   (infix):   {}", infix(&e));
    println!("  post-order (postfix): {}", postfix(&e));
    println!("  evaluated by post-order: {}", eval(&e));
}
```

**Rust output:**

```
the event trace (indent = depth, nil calls print nothing):
  enter   1
    enter   2
      enter   4
      between 4
      leave   4
    between 2
      enter   5
        enter   7
        between 7
        leave   7
      between 5
      leave   5
    leave   2
  between 1
    enter   3
    between 3
      enter   6
        enter   8
        between 8
        leave   8
      between 6
        enter   9
        between 9
        leave   9
      leave   6
    leave   3
  leave   1

events: 27 = 3 x 9 nodes
pre   keep enter    -> [1 2 4 5 7 3 6 8 9]   == recursion: true
in    keep between  -> [4 2 7 5 1 3 8 6 9]   == recursion: true
post  keep leave    -> [4 7 5 2 8 9 6 3 1]   == recursion: true

order  calls   work  max stack
pre       19      9          5
in        19      9          5
post      19      9          5
2n+1 = 19, height+1 = 5, the same for all three orders: true

the expression tree (3 + 4) * (10 - 2):
  pre-order  (prefix):  * + 3 4 - 10 2
  in-order   (infix):   ((3 + 4) * (10 - 2))
  post-order (postfix): 3 4 + 10 2 - *
  evaluated by post-order: 56
```

> **Go vs Rust:** Identical output, byte for byte. The three touches are an `enum Event` here (Go passes a string), and the expression tree is a sum type, `enum Expr { Num, Bin(Op, Box<Expr>, Box<Expr>) }`: a number has no operands to forget, and an `Op` enum makes `eval`'s `match` exhaustive, where Go's `eval` ends with an unchecked `return l * r` for every operator byte it did not list. `walk` takes `&mut impl FnMut(..)` so the callback can push onto the trace it captured.

**Complexity:** all three orders are Θ(n) time and Θ(h) stack, and make **2n + 1 calls whatever the order**: 19 calls (9 on real nodes, 10 on nil) and a deepest stack of 5 (= height + 1, counting the nil call below the deepest leaf) for pre, in and post alike. Only the visit position moves, and the event trace holds exactly 3n = 27 events. The expression tree reads `* + 3 4 - 10 2` in pre-order, `((3 + 4) * (10 - 2))` in in-order, `3 4 + 10 2 - *` in post-order, and evaluates to 56.

---

## 3. The call stack, written as a loop

`🟢 easy` · *Explicit stack*

[Lesson 05](../../05-recursion.md) example 7 converted pre- and in-order on a small tree; the missing piece is post-order, which has to know which child it is returning from, and the surprise is what the stack *holds*: not the height, but a number that depends on the method and on which way the tree leans. A stack of `(node, state)` frames does all three orders with one loop because it is a literal copy of the call stack. Every variant is Θ(n) time, and the example prints each one's high-water mark (the most entries held at once) so the extra space can be read off instead of assumed.

**Steps:**

1. Write the recursive reference and five iterative walks (pre-order pushing right then left, in-order walking left, post-order with two stacks, post-order with a `last` pointer, and the frame stack, which handles all three orders), each returning its sequence and the high-water mark of its stack.
2. Check every variant against the recursion on the hand-built tree and on 2,000 random trees of 0 to 60 nodes (the empty tree included).
3. Read the high-water table over five shapes and compare each column with the height `h`, which is what the recursion's call stack would need.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func leaf(v int) *Node { return &Node{Val: v} }

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
func handBuilt() *Node {
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// recursive is the reference: the skeleton of example 2. order 0 = pre, 1 = in, 2 = post.
func recursive(n *Node, order int, out *[]int) {
	if n == nil {
		return
	}
	if order == 0 {
		*out = append(*out, n.Val)
	}
	recursive(n.Left, order, out)
	if order == 1 {
		*out = append(*out, n.Val)
	}
	recursive(n.Right, order, out)
	if order == 2 {
		*out = append(*out, n.Val)
	}
}

// Every iterative method returns its sequence and its HIGH-WATER mark: the most
// entries its stack(s) held at once, measured right after the pushes.

// preIter: pop, visit, push right THEN left (a stack reverses, so left pops first).
func preIter(root *Node) (seq []int, hw int) {
	if root == nil {
		return
	}
	st := []*Node{root}
	hw = 1 // the root is on the stack before the first pop
	for len(st) > 0 {
		n := st[len(st)-1]
		st = st[:len(st)-1]
		seq = append(seq, n.Val)
		if n.Right != nil {
			st = append(st, n.Right)
		}
		if n.Left != nil {
			st = append(st, n.Left)
		}
		hw = max(hw, len(st))
	}
	return
}

// inIter: walk left pushing every node, pop and visit, then step to the right child.
func inIter(root *Node) (seq []int, hw int) {
	var st []*Node
	for cur := root; cur != nil || len(st) > 0; {
		for ; cur != nil; cur = cur.Left {
			st = append(st, cur)
		}
		hw = max(hw, len(st))
		cur = st[len(st)-1]
		st = st[:len(st)-1]
		seq = append(seq, cur.Val)
		cur = cur.Right
	}
	return
}

// postTwo: a root-right-left walk with one stack puts nodes on a second stack;
// reversed, that is post-order. The second stack receives EVERY node.
func postTwo(root *Node) (seq []int, hw int) {
	if root == nil {
		return
	}
	s1, s2 := []*Node{root}, []*Node(nil)
	for len(s1) > 0 {
		n := s1[len(s1)-1]
		s1 = s1[:len(s1)-1]
		s2 = append(s2, n)
		for _, c := range []*Node{n.Left, n.Right} {
			if c != nil {
				s1 = append(s1, c)
			}
		}
		hw = max(hw, len(s1)+len(s2))
	}
	for i := len(s2) - 1; i >= 0; i-- {
		seq = append(seq, s2[i].Val)
	}
	return
}

// postLast: one stack, plus `last` = the node just finished. A node is done when
// it has no right child, or we are returning from it.
func postLast(root *Node) (seq []int, hw int) {
	var st []*Node
	var last *Node
	for cur := root; cur != nil || len(st) > 0; {
		for ; cur != nil; cur = cur.Left {
			st = append(st, cur)
		}
		hw = max(hw, len(st))
		top := st[len(st)-1]
		if top.Right != nil && top.Right != last {
			cur = top.Right // the right subtree is not done yet
		} else {
			seq = append(seq, top.Val)
			st = st[:len(st)-1]
			last = top
		}
	}
	return
}

// frames: a stack of (node, state) is a literal copy of the call stack. state 0 =
// just entered, 1 = left child done, 2 = right child done. One loop does all
// three orders: emit when state == order.
type frame struct {
	n     *Node
	state int
}

func frames(root *Node, order int) (seq []int, hw int) {
	if root == nil {
		return
	}
	st := []frame{{root, 0}}
	hw = 1
	for len(st) > 0 {
		f := &st[len(st)-1]
		n, state := f.n, f.state
		f.state++ // this frame resumes at the next state
		if state == order {
			seq = append(seq, n.Val)
		}
		var child *Node
		switch state {
		case 0:
			child = n.Left
		case 1:
			child = n.Right
		default:
			st = st[:len(st)-1] // return: pop the frame
		}
		if child != nil {
			st = append(st, frame{child, 0})
			hw = max(hw, len(st))
		}
	}
	return
}

type method struct {
	name  string
	order int // the recursive order it must reproduce
	run   func(*Node) ([]int, int)
}

var methods = []method{
	{"pre: push right, then left", 0, preIter},
	{"in: walk left pushing", 1, inIter},
	{"post: two stacks", 2, postTwo},
	{"post: last pointer", 2, postLast},
	{"pre: frame stack", 0, func(r *Node) ([]int, int) { return frames(r, 0) }},
	{"in: frame stack", 1, func(r *Node) ([]int, int) { return frames(r, 1) }},
	{"post: frame stack", 2, func(r *Node) ([]int, int) { return frames(r, 2) }},
}

// ---- shapes ----

// randomSplit: the left subtree gets a uniformly random share of the other n-1
// nodes. The draw comes BEFORE the recursion (the Rust twin makes the same draws),
// and Val is the node's in-order rank, so all three orders give different sequences.
func randomSplit(n int, r *rand.Rand, next *int) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r, next)
	nd := &Node{Val: *next, Left: left}
	*next++
	nd.Right = randomSplit(n-1-k, r, next)
	return nd
}

func random(n int, r *rand.Rand) *Node { return randomSplit(n, r, new(int)) }

func perfect(h int) *Node { return complete(0, 1<<h-1) }

func complete(i, n int) *Node { // node i has children 2i+1 and 2i+2 while they are < n
	if i >= n {
		return nil
	}
	return &Node{Val: i, Left: complete(2*i+1, n), Right: complete(2*i+2, n)}
}

// chain builds n nodes in a loop; side(i) says whether node i hangs on the right of node i-1.
func chain(n int, side func(i int) bool) *Node {
	var root *Node
	for i := n - 1; i >= 0; i-- {
		if side(i + 1) {
			root = &Node{Val: i, Right: root}
		} else {
			root = &Node{Val: i, Left: root}
		}
	}
	return root
}

func height(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.Left), height(n.Right))
}

func main() {
	root := handBuilt()
	fmt.Println("the hand-built tree, every method:")
	for _, m := range methods {
		var want []int
		recursive(root, m.order, &want)
		got, hw := m.run(root)
		fmt.Printf("  %-27s %v  stack %d  == recursive: %v\n", m.name, got, hw, slices.Equal(got, want))
	}

	fz := rand.New(rand.NewPCG(1, 2))
	mismatches, nodes := 0, 0
	for t := 0; t < 2000; t++ {
		n := fz.IntN(61) // 0..60, the empty tree included
		tree := random(n, fz)
		nodes += n
		for _, m := range methods {
			var want []int
			recursive(tree, m.order, &want)
			if got, _ := m.run(tree); !slices.Equal(got, want) {
				mismatches++
			}
		}
	}
	fmt.Printf("\nfuzz: %d mismatches over 2000 random trees x %d methods, %d nodes in total\n",
		mismatches, len(methods), nodes)

	// How much does each stack hold? The recursion's call stack is the height h.
	r := rand.New(rand.NewPCG(1, 2))
	right := func(i int) bool { return true }
	left := func(i int) bool { return false }
	zig := func(i int) bool { return i%2 == 1 }
	shapes := []struct {
		name string
		root *Node
	}{
		{"randomSplit(100000)", random(100000, r)},
		{"perfect(16)", perfect(16)},
		{"leftChain(1000)", chain(1000, left)},
		{"rightChain(1000)", chain(1000, right)},
		{"zigzag(1000)", chain(1000, zig)},
	}
	names := []string{"pre", "in", "post-last", "post-2stk", "frames"}
	cols := []int{0, 1, 3, 2, 6} // indexes into methods (the frame column is the post-order one)
	type row struct {
		n, h int
		hw   []int
	}
	rows := make([]row, len(shapes))
	fmt.Println()
	fmt.Printf("%-20s %7s %5s", "shape", "n", "h")
	for _, name := range names {
		fmt.Printf(" %10s", name)
	}
	fmt.Println()
	for i, s := range shapes {
		seq, _ := postLast(s.root)
		rows[i] = row{len(seq), height(s.root), nil}
		fmt.Printf("%-20s %7d %5d", s.name, rows[i].n, rows[i].h)
		for _, c := range cols {
			_, hw := methods[c].run(s.root)
			rows[i].hw = append(rows[i].hw, hw)
			fmt.Printf(" %10d", hw)
		}
		fmt.Println()
	}

	// Compare each stack with the height h, over the five shapes.
	fmt.Println()
	for c, name := range names {
		below, above := 0, 0
		for _, rw := range rows {
			if rw.hw[c] < rw.h {
				below++
			} else if rw.hw[c] > rw.h {
				above++
			}
		}
		fmt.Printf("%-10s below h on %d of %d shapes, above h on %d\n", name, below, len(rows), above)
	}
	allN, allH := true, true
	for _, rw := range rows {
		allN = allN && rw.hw[3] == rw.n
		allH = allH && rw.hw[4] == rw.h
	}
	fmt.Printf("two-stack post-order holds all n on every shape: %v\n", allN)
	fmt.Printf("frame stack equals the recursion depth h on every shape: %v\n", allH)
}
```

**Output:**

```
the hand-built tree, every method:
  pre: push right, then left  [1 2 4 5 7 3 6 8 9]  stack 3  == recursive: true
  in: walk left pushing       [4 2 7 5 1 3 8 6 9]  stack 3  == recursive: true
  post: two stacks            [4 7 5 2 8 9 6 3 1]  stack 9  == recursive: true
  post: last pointer          [4 7 5 2 8 9 6 3 1]  stack 4  == recursive: true
  pre: frame stack            [1 2 4 5 7 3 6 8 9]  stack 4  == recursive: true
  in: frame stack             [4 2 7 5 1 3 8 6 9]  stack 4  == recursive: true
  post: frame stack           [4 7 5 2 8 9 6 3 1]  stack 4  == recursive: true

fuzz: 0 mismatches over 2000 random trees x 7 methods, 61262 nodes in total

shape                      n     h        pre         in  post-last  post-2stk     frames
randomSplit(100000)   100000    38         25         25         38     100000         38
perfect(16)            65535    16         16         16         16      65535         16
leftChain(1000)         1000  1000          1       1000       1000       1000       1000
rightChain(1000)        1000  1000          1          1       1000       1000       1000
zigzag(1000)            1000  1000          1        500       1000       1000       1000

pre        below h on 4 of 5 shapes, above h on 0
in         below h on 3 of 5 shapes, above h on 0
post-last  below h on 0 of 5 shapes, above h on 0
post-2stk  below h on 0 of 5 shapes, above h on 2
frames     below h on 0 of 5 shapes, above h on 0
two-stack post-order holds all n on every shape: true
frame stack equals the recursion depth h on every shape: true
```

**Rust:**

```rust
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (a chain of 1,000 nodes would be fine, a chain of millions
// would not: example 13). Go has no destructor to write.
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

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
fn hand_built() -> Link {
    node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    )
}

/// The reference: the skeleton of example 2. order 0 = pre, 1 = in, 2 = post.
fn recursive(n: &Link, order: u8, out: &mut Vec<i64>) {
    let Some(b) = n else { return };
    if order == 0 {
        out.push(b.val);
    }
    recursive(&b.left, order, out);
    if order == 1 {
        out.push(b.val);
    }
    recursive(&b.right, order, out);
    if order == 2 {
        out.push(b.val);
    }
}

// Every iterative method returns its sequence and its HIGH-WATER mark: the most
// entries its stack(s) held at once, measured right after the pushes. The
// stacks hold `&Node`: references INTO the tree, so the borrow checker proves
// they cannot outlive it. `as_deref()` turns `&Option<Box<Node>>` into
// `Option<&Node>`, and an Option is an iterator of 0 or 1 items, so
// `st.extend(child)` pushes only if there is one.

/// pop, visit, push right THEN left (a stack reverses, so left pops first).
fn pre_iter(root: &Link) -> (Vec<i64>, usize) {
    let mut seq = Vec::new();
    let mut st: Vec<&Node> = root.as_deref().into_iter().collect();
    let mut hw = st.len(); // the root is on the stack before the first pop
    while let Some(n) = st.pop() {
        seq.push(n.val);
        st.extend(n.right.as_deref());
        st.extend(n.left.as_deref());
        hw = hw.max(st.len());
    }
    (seq, hw)
}

/// walk left pushing every node, pop and visit, then step to the right child.
fn in_iter(root: &Link) -> (Vec<i64>, usize) {
    let (mut seq, mut hw) = (Vec::new(), 0);
    let mut st: Vec<&Node> = Vec::new();
    let mut cur = root.as_deref();
    while cur.is_some() || !st.is_empty() {
        while let Some(n) = cur {
            st.push(n);
            cur = n.left.as_deref();
        }
        hw = hw.max(st.len());
        let n = st
            .pop()
            .expect("cur is None, so the loop condition says st is not empty");
        seq.push(n.val);
        cur = n.right.as_deref();
    }
    (seq, hw)
}

/// A root-right-left walk with one stack puts nodes on a second stack;
/// reversed, that is post-order. The second stack receives EVERY node.
fn post_two(root: &Link) -> (Vec<i64>, usize) {
    let mut s1: Vec<&Node> = root.as_deref().into_iter().collect();
    let mut s2: Vec<&Node> = Vec::new();
    let mut hw = 0;
    while let Some(n) = s1.pop() {
        s2.push(n);
        s1.extend(n.left.as_deref());
        s1.extend(n.right.as_deref());
        hw = hw.max(s1.len() + s2.len());
    }
    (s2.iter().rev().map(|n| n.val).collect(), hw)
}

/// One stack, plus `last` = the node just finished. A node is done when it has
/// no right child, or we are returning from it. Go compares two pointers; Rust
/// compares two references' addresses with `ptr::eq` (`==` on `&Node` would compare
/// the pointed-to values, and `Node` has no `PartialEq` here).
fn post_last(root: &Link) -> (Vec<i64>, usize) {
    let (mut seq, mut hw) = (Vec::new(), 0);
    let mut st: Vec<&Node> = Vec::new();
    let mut last: Option<&Node> = None;
    let mut cur = root.as_deref();
    while cur.is_some() || !st.is_empty() {
        while let Some(n) = cur {
            st.push(n);
            cur = n.left.as_deref();
        }
        hw = hw.max(st.len());
        let top = *st
            .last()
            .expect("cur is None, so the loop condition says st is not empty");
        match top.right.as_deref() {
            // the right subtree is not done yet
            Some(r) if !last.is_some_and(|l| std::ptr::eq(l, r)) => cur = Some(r),
            _ => {
                seq.push(top.val);
                st.pop();
                last = Some(top);
            }
        }
    }
    (seq, hw)
}

/// A stack of (node, state) is a literal copy of the call stack. state 0 = just
/// entered, 1 = left child done, 2 = right child done. One loop does all three
/// orders: emit when state == order.
fn frames(root: &Link, order: u8) -> (Vec<i64>, usize) {
    let mut seq = Vec::new();
    let mut st: Vec<(&Node, u8)> = root.as_deref().map(|n| (n, 0)).into_iter().collect();
    let mut hw = st.len();
    while let Some(top) = st.last_mut() {
        let (n, state) = *top;
        top.1 += 1; // this frame resumes at the next state
        if state == order {
            seq.push(n.val);
        }
        let child = match state {
            0 => n.left.as_deref(),
            1 => n.right.as_deref(),
            _ => {
                st.pop(); // return: pop the frame
                None
            }
        };
        if let Some(c) = child {
            st.push((c, 0));
            hw = hw.max(st.len());
        }
    }
    (seq, hw)
}

struct Method {
    name: &'static str,
    order: u8, // the recursive order it must reproduce
    run: fn(&Link) -> (Vec<i64>, usize),
}

// ---- shapes ----

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of IntN, copied
// from lesson 01 example 14, so the same seed builds the same tree as in Go.
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

    /// Like rand.IntN(n): exactly uniform in [0, n), Lemire's method.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

/// The left subtree gets a uniformly random share of the other n-1 nodes. The
/// draw comes BEFORE the recursion (as in Go), and val is the node's in-order
/// rank, so all three orders give different sequences.
fn random_split(n: usize, r: &mut Pcg, next: &mut i64) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r, next);
    let val = *next;
    *next += 1;
    let right = random_split(n - 1 - k, r, next);
    node(val, left, right)
}

fn random(n: usize, r: &mut Pcg) -> Link {
    random_split(n, r, &mut 0)
}

/// Heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
fn complete(i: usize, n: usize) -> Link {
    if i >= n {
        None
    } else {
        node(i as i64, complete(2 * i + 1, n), complete(2 * i + 2, n))
    }
}

/// n nodes built in a loop; side(i) says whether node i hangs on the right of node i-1.
fn chain(n: usize, side: impl Fn(usize) -> bool) -> Link {
    let mut root = None;
    for i in (0..n).rev() {
        root = if side(i + 1) {
            node(i as i64, None, root)
        } else {
            node(i as i64, root, None)
        };
    }
    root
}

fn height(n: &Link) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + height(&b.left).max(height(&b.right)),
    }
}

/// Go's %v of a slice: [1 2 3]
fn go_list(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let methods = [
        Method {
            name: "pre: push right, then left",
            order: 0,
            run: pre_iter,
        },
        Method {
            name: "in: walk left pushing",
            order: 1,
            run: in_iter,
        },
        Method {
            name: "post: two stacks",
            order: 2,
            run: post_two,
        },
        Method {
            name: "post: last pointer",
            order: 2,
            run: post_last,
        },
        Method {
            name: "pre: frame stack",
            order: 0,
            run: |r| frames(r, 0),
        },
        Method {
            name: "in: frame stack",
            order: 1,
            run: |r| frames(r, 1),
        },
        Method {
            name: "post: frame stack",
            order: 2,
            run: |r| frames(r, 2),
        },
    ];
    let reference = |tree: &Link, order: u8| {
        let mut want = Vec::new();
        recursive(tree, order, &mut want);
        want
    };

    let root = hand_built();
    println!("the hand-built tree, every method:");
    for m in &methods {
        let (got, hw) = (m.run)(&root);
        println!(
            "  {:<27} {}  stack {}  == recursive: {}",
            m.name,
            go_list(&got),
            hw,
            got == reference(&root, m.order)
        );
    }

    let mut fz = Pcg::new(1, 2);
    let (mut mismatches, mut nodes) = (0, 0);
    for _ in 0..2000 {
        let n = fz.int_n(61); // 0..60, the empty tree included
        let tree = random(n, &mut fz);
        nodes += n;
        for m in &methods {
            if (m.run)(&tree).0 != reference(&tree, m.order) {
                mismatches += 1;
            }
        }
    }
    println!(
        "\nfuzz: {} mismatches over 2000 random trees x {} methods, {} nodes in total",
        mismatches,
        methods.len(),
        nodes
    );

    // How much does each stack hold? The recursion's call stack is the height h.
    let mut r = Pcg::new(1, 2);
    let shapes: Vec<(&str, Link)> = vec![
        ("randomSplit(100000)", random(100000, &mut r)),
        ("perfect(16)", complete(0, (1 << 16) - 1)),
        ("leftChain(1000)", chain(1000, |_| false)),
        ("rightChain(1000)", chain(1000, |_| true)),
        ("zigzag(1000)", chain(1000, |i| i % 2 == 1)),
    ];
    let names = ["pre", "in", "post-last", "post-2stk", "frames"];
    let cols = [0, 1, 3, 2, 6]; // indexes into methods (the frame column is the post-order one)
    println!();
    print!("{:<20} {:>7} {:>5}", "shape", "n", "h");
    for name in names {
        print!(" {:>10}", name);
    }
    println!();
    let mut rows: Vec<(usize, usize, Vec<usize>)> = Vec::new(); // (n, h, high-water per column)
    for (name, tree) in &shapes {
        let n = post_last(tree).0.len();
        let h = height(tree);
        let hw: Vec<usize> = cols.iter().map(|&c| (methods[c].run)(tree).1).collect();
        print!("{:<20} {:>7} {:>5}", name, n, h);
        for v in &hw {
            print!(" {:>10}", v);
        }
        println!();
        rows.push((n, h, hw));
    }

    // Compare each stack with the height h, over the five shapes.
    println!();
    for (c, name) in names.iter().enumerate() {
        let below = rows.iter().filter(|(_, h, hw)| hw[c] < *h).count();
        let above = rows.iter().filter(|(_, h, hw)| hw[c] > *h).count();
        println!(
            "{:<10} below h on {} of {} shapes, above h on {}",
            name,
            below,
            rows.len(),
            above
        );
    }
    let all_n = rows.iter().all(|(n, _, hw)| hw[3] == *n);
    let all_h = rows.iter().all(|(_, h, hw)| hw[4] == *h);
    println!("two-stack post-order holds all n on every shape: {all_n}");
    println!("frame stack equals the recursion depth h on every shape: {all_h}");
}
```

**Rust output:**

```
the hand-built tree, every method:
  pre: push right, then left  [1 2 4 5 7 3 6 8 9]  stack 3  == recursive: true
  in: walk left pushing       [4 2 7 5 1 3 8 6 9]  stack 3  == recursive: true
  post: two stacks            [4 7 5 2 8 9 6 3 1]  stack 9  == recursive: true
  post: last pointer          [4 7 5 2 8 9 6 3 1]  stack 4  == recursive: true
  pre: frame stack            [1 2 4 5 7 3 6 8 9]  stack 4  == recursive: true
  in: frame stack             [4 2 7 5 1 3 8 6 9]  stack 4  == recursive: true
  post: frame stack           [4 7 5 2 8 9 6 3 1]  stack 4  == recursive: true

fuzz: 0 mismatches over 2000 random trees x 7 methods, 61262 nodes in total

shape                      n     h        pre         in  post-last  post-2stk     frames
randomSplit(100000)   100000    38         25         25         38     100000         38
perfect(16)            65535    16         16         16         16      65535         16
leftChain(1000)         1000  1000          1       1000       1000       1000       1000
rightChain(1000)        1000  1000          1          1       1000       1000       1000
zigzag(1000)            1000  1000          1        500       1000       1000       1000

pre        below h on 4 of 5 shapes, above h on 0
in         below h on 3 of 5 shapes, above h on 0
post-last  below h on 0 of 5 shapes, above h on 0
post-2stk  below h on 0 of 5 shapes, above h on 2
frames     below h on 0 of 5 shapes, above h on 0
two-stack post-order holds all n on every shape: true
frame stack equals the recursion depth h on every shape: true
```

> **Go vs Rust:** Identical output: every high-water mark, the fuzz node total (61,262) and the 100,000-node tree (height 38) match, because the `PCG(1, 2)` port draws the same stream. The stacks are `Vec<&Node>` and `Vec<(&Node, u8)>`: references into the tree, so the compiler proves they cannot outlive it, and `st.extend(n.left.as_deref())` pushes a child only if it exists (an `Option` is an iterator of 0 or 1 items). The `last` pointer of the post-order is compared with `std::ptr::eq` (identity); `==` on two `&Node` would compare the pointed-to values, and `Node` does not even implement `PartialEq` here.

**Complexity:** Θ(n) time for every variant; extra space is the stack high-water, and it is **not the height**. On the 5 shapes (n = 100,000 random with h = 38, perfect(16) with h = 16, and three 1,000-node chains), the push-both pre-order stack was below h on 4 and in-order below h on 3 (a left chain needs 1,000 for in-order and 1 for pre-order; a right chain needs 1 for both), the `last`-pointer post-order and the frame stack equalled h on all 5, and **two-stack post-order held all n on every shape** (100,000 and 65,535 where h is 38 and 16). 2,000 random trees gave 0 mismatches against the recursion.

---

## 4. Level order: the queue holds one layer

`🟢 easy` · *BFS / width vs height*

Level order is BFS with a queue, and the `n := len(q)` snapshot that [lesson 09](../../09-queues-deques.md) showed on a grid separates the layers; on a tree there is no `seen` set, because a tree has no cycles. What is new is the cost: the queue holds about one layer, so BFS memory follows the tree's **width** while recursive DFS memory follows its **height**, and the two move in opposite directions across shapes (a perfect tree is the best case for DFS and the worst for BFS, a chain the reverse). Time is Θ(n) either way; the example measures both memories and checks the width bound `w <= (n+1)/2` on 2,000 random trees.

**Steps:**

1. Write `levels` with the snapshot and a queue, and `levelsTwo` with two slices that swap per layer (no queue, no snapshot); print the layers, the right-side view and the zigzag order of the hand-built tree.
2. Check on 2,000 random trees that both versions agree and that no layer is wider than `(n+1)/2`.
3. Read the table over five shapes: the queue's high-water mark next to the width `w` and the height `h`, and which of BFS (queue) and DFS (stack) needs less.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func leaf(v int) *Node { return &Node{Val: v} }

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
func handBuilt() *Node {
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// levels is BFS with a queue. `n := len(q)` is the SNAPSHOT of how many nodes
// are in the current layer: it is read once, before the loop, because the loop
// itself keeps pushing the next layer onto the same queue. A tree has no cycles,
// so unlike a graph BFS there is no `seen` set. hw is the queue's high-water mark.
func levels(root *Node) (layers [][]int, hw int) {
	if root == nil {
		return
	}
	q := []*Node{root}
	hw = 1
	for len(q) > 0 {
		n := len(q)
		layer := make([]int, 0, n)
		for range n {
			x := q[0]
			q = q[1:]
			layer = append(layer, x.Val)
			if x.Left != nil {
				q = append(q, x.Left)
			}
			if x.Right != nil {
				q = append(q, x.Right)
			}
			hw = max(hw, len(q))
		}
		layers = append(layers, layer)
	}
	return
}

// levelsTwo needs no queue and no snapshot: `cur` is this layer, `next` is the
// one being built, and the two slices swap at the end of each layer.
func levelsTwo(root *Node) (layers [][]int) {
	if root == nil {
		return
	}
	cur, next := []*Node{root}, []*Node(nil)
	for len(cur) > 0 {
		layer := make([]int, 0, len(cur))
		next = next[:0]
		for _, x := range cur {
			layer = append(layer, x.Val)
			if x.Left != nil {
				next = append(next, x.Left)
			}
			if x.Right != nil {
				next = append(next, x.Right)
			}
		}
		layers = append(layers, layer)
		cur, next = next, cur
	}
	return
}

func height(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.Left), height(n.Right))
}

// randomSplit: the left subtree gets a uniformly random share of the other n-1
// nodes; the draw comes BEFORE the recursion (the Rust twin makes the same draws).
func randomSplit(n int, r *rand.Rand, next *int) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r, next)
	nd := &Node{Val: *next, Left: left}
	*next++
	nd.Right = randomSplit(n-1-k, r, next)
	return nd
}

func complete(i, n int) *Node { // node i has children 2i+1 and 2i+2 while they are < n
	if i >= n {
		return nil
	}
	return &Node{Val: i, Left: complete(2*i+1, n), Right: complete(2*i+2, n)}
}

// chain builds n nodes in a loop; side(i) says whether node i hangs on the right of node i-1.
func chain(n int, side func(i int) bool) *Node {
	var root *Node
	for i := n - 1; i >= 0; i-- {
		if side(i + 1) {
			root = &Node{Val: i, Right: root}
		} else {
			root = &Node{Val: i, Left: root}
		}
	}
	return root
}

func main() {
	root := handBuilt()
	layers, _ := levels(root)
	fmt.Println("layers:     ", layers)

	var view []int // right-side view: the last node of every layer
	zig := make([][]int, len(layers))
	for i, l := range layers {
		view = append(view, l[len(l)-1])
		zig[i] = slices.Clone(l)
		if i%2 == 1 {
			slices.Reverse(zig[i]) // zigzag: every other layer runs right to left
		}
	}
	fmt.Println("right view: ", view)
	fmt.Println("zigzag:     ", zig)

	r := rand.New(rand.NewPCG(1, 2))
	same, violations, worst := 0, 0, 0.0
	for range 2000 {
		n := 20 + r.IntN(181) // 20..200 nodes
		t := randomSplit(n, r, new(int))
		a, _ := levels(t)
		if slices.EqualFunc(a, levelsTwo(t), slices.Equal) {
			same++
		}
		w := 0
		for _, l := range a {
			w = max(w, len(l))
		}
		if 2*w > n+1 {
			violations++
		}
		worst = max(worst, float64(w)/(float64(n+1)/2))
	}
	fmt.Printf("\n== queue version: %d of 2000 random trees agree with the two-slice version\n", same)
	fmt.Printf("width bound w <= (n+1)/2: violations: %d; largest w / ((n+1)/2) seen: %.3f\n", violations, worst)

	r = rand.New(rand.NewPCG(1, 2))
	shapes := []struct {
		name string
		root *Node
	}{
		{"perfect(16)", complete(0, 1<<16-1)},
		{"randomSplit(100000)", randomSplit(100000, r, new(int))},
		{"leftChain(1000)", chain(1000, func(int) bool { return false })},
		{"rightChain(1000)", chain(1000, func(int) bool { return true })},
		{"zigzag(1000)", chain(1000, func(i int) bool { return i%2 == 1 })},
	}
	fmt.Println()
	fmt.Printf("%-20s %7s %5s %7s %7s %9s  %s\n", "shape", "n", "h", "width", "queue", "(n+1)/2", "less memory")
	over := 0
	for _, s := range shapes {
		ls, hw := levels(s.root)
		n, w := 0, 0
		for _, l := range ls {
			n += len(l)
			w = max(w, len(l))
		}
		h := height(s.root)
		less := "tie"
		if hw < h {
			less = "BFS queue"
		} else if h < hw {
			less = "DFS stack"
		}
		fmt.Printf("%-20s %7d %5d %7d %7d %9d  %s\n", s.name, n, h, w, hw, (n+1)/2, less)
		if hw > w {
			over++
		}
	}
	fmt.Printf("the queue's high-water mark exceeded the widest layer on %d of %d shapes\n", over, len(shapes))
}
```

**Output:**

```
layers:      [[1] [2 3] [4 5 6] [7 8 9]]
right view:  [1 3 6 9]
zigzag:      [[1] [3 2] [4 5 6] [9 8 7]]

== queue version: 2000 of 2000 random trees agree with the two-slice version
width bound w <= (n+1)/2: violations: 0; largest w / ((n+1)/2) seen: 0.762

shape                      n     h   width   queue   (n+1)/2  less memory
perfect(16)            65535    16   32768   32768     32768  DFS stack
randomSplit(100000)   100000    38    9075    9305     50000  DFS stack
leftChain(1000)         1000  1000       1       1       500  BFS queue
rightChain(1000)        1000  1000       1       1       500  BFS queue
zigzag(1000)            1000  1000       1       1       500  BFS queue
the queue's high-water mark exceeded the widest layer on 1 of 5 shapes
```

**Rust:**

```rust
use std::collections::VecDeque;

struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (a chain of 1,000 nodes would be fine, a chain of millions
// would not: example 13). Go has no destructor to write.
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

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
fn hand_built() -> Link {
    node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    )
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of IntN, copied
// from lesson 01 example 14, so the same seed builds the same tree as in Go.
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

    /// Like rand.IntN(n): exactly uniform in [0, n), Lemire's method.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

/// Heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
fn complete(i: usize, n: usize) -> Link {
    if i >= n {
        None
    } else {
        node(i as i64, complete(2 * i + 1, n), complete(2 * i + 2, n))
    }
}

/// n nodes built in a loop; side(i) says whether node i hangs on the right of node i-1.
fn chain(n: usize, side: impl Fn(usize) -> bool) -> Link {
    let mut root = None;
    for i in (0..n).rev() {
        root = if side(i + 1) {
            node(i as i64, None, root)
        } else {
            node(i as i64, root, None)
        };
    }
    root
}

fn height(n: &Link) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + height(&b.left).max(height(&b.right)),
    }
}

/// BFS with a queue. `n = q.len()` is the SNAPSHOT of how many nodes are in the
/// current layer: it is read once, before the loop, because the loop itself keeps
/// pushing the next layer onto the same queue. A tree has no cycles, so unlike a
/// graph BFS there is no `seen` set. The second result is the queue's high-water mark.
fn levels(root: &Link) -> (Vec<Vec<i64>>, usize) {
    let mut layers = Vec::new();
    // `q.extend(child)`: an Option is an iterator of 0 or 1 items, so a missing child pushes nothing.
    let mut q: VecDeque<&Node> = root.as_deref().into_iter().collect();
    let mut hw = q.len();
    while !q.is_empty() {
        let n = q.len();
        let mut layer = Vec::with_capacity(n);
        for _ in 0..n {
            let x = q.pop_front().expect("the snapshot counted n entries");
            layer.push(x.val);
            q.extend(x.left.as_deref());
            q.extend(x.right.as_deref());
            hw = hw.max(q.len());
        }
        layers.push(layer);
    }
    (layers, hw)
}

/// No queue and no snapshot: `cur` is this layer, `next` is the one being built,
/// and the two vectors swap at the end of each layer.
fn levels_two(root: &Link) -> Vec<Vec<i64>> {
    let mut layers = Vec::new();
    let mut cur: Vec<&Node> = root.as_deref().into_iter().collect();
    let mut next: Vec<&Node> = Vec::new();
    while !cur.is_empty() {
        layers.push(cur.iter().map(|x| x.val).collect());
        next.clear();
        for x in &cur {
            next.extend(x.left.as_deref());
            next.extend(x.right.as_deref());
        }
        std::mem::swap(&mut cur, &mut next);
    }
    layers
}

/// The left subtree gets a uniformly random share of the other n-1 nodes; the
/// draw comes BEFORE the recursion (as in Go), and val is the in-order rank.
fn random_split(n: usize, r: &mut Pcg, next: &mut i64) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r, next);
    let val = *next;
    *next += 1;
    let right = random_split(n - 1 - k, r, next);
    node(val, left, right)
}

/// Go's %v of a slice of slices: [[1] [2 3]]
fn go_layers(ls: &[Vec<i64>]) -> String {
    let parts: Vec<String> = ls.iter().map(|l| go_list(l)).collect();
    format!("[{}]", parts.join(" "))
}

fn go_list(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let root = hand_built();
    let (layers, _) = levels(&root);
    println!("layers:      {}", go_layers(&layers));

    // right-side view: the last node of every layer
    let view: Vec<i64> = layers.iter().filter_map(|l| l.last().copied()).collect();
    println!("right view:  {}", go_list(&view));
    // zigzag: every other layer runs right to left
    let zig: Vec<Vec<i64>> = layers
        .iter()
        .enumerate()
        .map(|(i, l)| {
            let mut l = l.clone();
            if i % 2 == 1 {
                l.reverse();
            }
            l
        })
        .collect();
    println!("zigzag:      {}", go_layers(&zig));

    let mut r = Pcg::new(1, 2);
    let (mut same, mut violations, mut worst) = (0, 0, 0.0_f64);
    for _ in 0..2000 {
        let n = 20 + r.int_n(181); // 20..200 nodes
        let t = random_split(n, &mut r, &mut 0);
        let (a, _) = levels(&t);
        if a == levels_two(&t) {
            same += 1;
        }
        let w = a.iter().map(Vec::len).max().unwrap_or(0);
        if 2 * w > n + 1 {
            violations += 1;
        }
        worst = worst.max(w as f64 / ((n + 1) as f64 / 2.0));
    }
    println!("\n== queue version: {same} of 2000 random trees agree with the two-slice version");
    println!(
        "width bound w <= (n+1)/2: violations: {violations}; largest w / ((n+1)/2) seen: {worst:.3}"
    );

    let mut r = Pcg::new(1, 2);
    let shapes: Vec<(&str, Link)> = vec![
        ("perfect(16)", complete(0, (1 << 16) - 1)),
        ("randomSplit(100000)", random_split(100000, &mut r, &mut 0)),
        ("leftChain(1000)", chain(1000, |_| false)),
        ("rightChain(1000)", chain(1000, |_| true)),
        ("zigzag(1000)", chain(1000, |i| i % 2 == 1)),
    ];
    println!();
    println!(
        "{:<20} {:>7} {:>5} {:>7} {:>7} {:>9}  less memory",
        "shape", "n", "h", "width", "queue", "(n+1)/2"
    );
    let mut over = 0;
    for (name, tree) in &shapes {
        let (ls, hw) = levels(tree);
        let n: usize = ls.iter().map(Vec::len).sum();
        let w = ls.iter().map(Vec::len).max().unwrap_or(0);
        let h = height(tree);
        let less = match hw.cmp(&h) {
            std::cmp::Ordering::Less => "BFS queue",
            std::cmp::Ordering::Greater => "DFS stack",
            std::cmp::Ordering::Equal => "tie",
        };
        println!(
            "{:<20} {:>7} {:>5} {:>7} {:>7} {:>9}  {}",
            name,
            n,
            h,
            w,
            hw,
            n.div_ceil(2), // = (n+1)/2 in integer division
            less
        );
        if hw > w {
            over += 1;
        }
    }
    println!(
        "the queue's high-water mark exceeded the widest layer on {over} of {} shapes",
        shapes.len()
    );
}
```

**Rust output:**

```
layers:      [[1] [2 3] [4 5 6] [7 8 9]]
right view:  [1 3 6 9]
zigzag:      [[1] [3 2] [4 5 6] [9 8 7]]

== queue version: 2000 of 2000 random trees agree with the two-slice version
width bound w <= (n+1)/2: violations: 0; largest w / ((n+1)/2) seen: 0.762

shape                      n     h   width   queue   (n+1)/2  less memory
perfect(16)            65535    16   32768   32768     32768  DFS stack
randomSplit(100000)   100000    38    9075    9305     50000  DFS stack
leftChain(1000)         1000  1000       1       1       500  BFS queue
rightChain(1000)        1000  1000       1       1       500  BFS queue
zigzag(1000)            1000  1000       1       1       500  BFS queue
the queue's high-water mark exceeded the widest layer on 1 of 5 shapes
```

> **Go vs Rust:** Identical output. `q.extend(x.left.as_deref())` is the whole "push the child if it exists" (an `Option` is an iterator of 0 or 1 items), and the `Vec` + `mem::swap` frontier form needs neither a queue nor the snapshot variable; here the borrowed `&Node` entries mean neither queue can outlive the tree. Rust's `(n+1)/2` is spelled `n.div_ceil(2)` because clippy flags the manual form.

**Complexity:** Θ(n) time · queue Θ(w) with w <= (n+1)/2 (0 violations and a largest ratio of 0.762 on 2,000 random trees of 20 to 200 nodes), recursive DFS Θ(h). **The two memories run in opposite directions:** perfect(16) needs a queue of 32,768 against h = 16, while each 1,000-node chain needs a queue of 1 against h = 1000. The queue can also exceed the widest layer, since it holds the unfinished layer plus the part of the next one already pushed: 9,305 against a width of 9,075 on `randomSplit(100000)`, equal to the width on the other four shapes.

---

## 5. Height, depth and diameter

`🟢 easy` · *Return one, record another*

Height is a post-order recursion (a node needs its children's answers), depth is a number passed *down*, and the diameter needs the height of both children at every node, which tempts you into calling `height` at every node and paying the sum of all subtree sizes. One pass that **returns** the height and **records** the best `left + right` on the side is Θ(n). The example counts both costs, and checks three independent diameters (the naive one, the one-pass one, and a BFS-from-every-node oracle) against each other on 3,000 random trees.

**Steps:**

1. On the hand-built tree print the height (in nodes), the depth of every node, and the diameter (in edges), and state the conventions once.
2. Write `naive` (calls `height` at every node and counts the nodes it walks), `onePass` (returns the height, updates `best`) and an `oracle` (BFS from every node over the undirected tree); compare all three on 3,000 random trees of 1 to 60 nodes.
3. Tabulate the work on a 1,000-node chain, a perfect tree and a 100,000-node random tree: the naive visits next to the sum of the subtree sizes, and next to the one-pass calls (2n + 1).

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

func leaf(v int) *Node { return &Node{Val: v} }

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
func handBuilt() *Node {
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// Convention, written once: height counts NODES on the longest root-to-leaf
// path (empty tree = 0, a leaf = 1). Depth counts EDGES from the root (root = 0).
// Diameter counts EDGES on the longest path between any two nodes.

// height is a post-order recursion: a node needs its children's answers.
func height(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.Left), height(n.Right))
}

// depths is the opposite direction: the depth is a number passed DOWN.
func depths(n *Node, d int, out *[]string) {
	if n == nil {
		return
	}
	*out = append(*out, fmt.Sprintf("%d:%d", n.Val, d))
	depths(n.Left, d+1, out)
	depths(n.Right, d+1, out)
}

// The longest path THROUGH a node goes down both sides: height(left) + height(right)
// edges, since heights count nodes. The diameter is the best of that over all nodes.

// heightCount is height that also counts the nodes it walks.
func heightCount(n *Node, visits *int) int {
	if n == nil {
		return 0
	}
	*visits++
	return 1 + max(heightCount(n.Left, visits), heightCount(n.Right, visits))
}

// naive calls height at EVERY node: the cost is the sum of the subtree sizes.
func naive(n *Node, visits *int) int {
	if n == nil {
		return 0
	}
	through := heightCount(n.Left, visits) + heightCount(n.Right, visits)
	return max(through, naive(n.Left, visits), naive(n.Right, visits))
}

// onePass RETURNS the height and RECORDS the best bend on the side: one call per
// node and per nil link, 2n+1 in all.
func onePass(n *Node, best, calls *int) int {
	*calls++
	if n == nil {
		return 0
	}
	l, r := onePass(n.Left, best, calls), onePass(n.Right, best, calls)
	*best = max(*best, l+r)
	return 1 + max(l, r)
}

// oracle shares nothing with the two above: number the nodes, make the tree an
// undirected graph, BFS from every node, take the largest distance seen.
func oracle(root *Node) int {
	var adj [][]int
	var add func(n *Node) int
	add = func(n *Node) int {
		id := len(adj)
		adj = append(adj, nil)
		for _, c := range []*Node{n.Left, n.Right} {
			if c != nil {
				cid := add(c)
				adj[id] = append(adj[id], cid)
				adj[cid] = append(adj[cid], id)
			}
		}
		return id
	}
	add(root)
	best := 0
	for src := range adj {
		dist := make([]int, len(adj))
		for i := range dist {
			dist[i] = -1
		}
		dist[src] = 0
		for q := []int{src}; len(q) > 0; q = q[1:] {
			for _, v := range adj[q[0]] {
				if dist[v] < 0 {
					dist[v] = dist[q[0]] + 1
					best = max(best, dist[v])
					q = append(q, v)
				}
			}
		}
	}
	return best
}

// sumSizes returns the size of n's subtree and the sum of ALL subtree sizes in it.
func sumSizes(n *Node) (size, total int) {
	if n == nil {
		return 0, 0
	}
	ls, lt := sumSizes(n.Left)
	rs, rt := sumSizes(n.Right)
	size = 1 + ls + rs
	return size, size + lt + rt
}

// ---- shapes (random: the draw comes BEFORE the recursion, as in the Rust twin) ----

func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{Left: left, Right: right}
}

func complete(i, n int) *Node { // node i has children 2i+1 and 2i+2 while they are < n
	if i >= n {
		return nil
	}
	return &Node{Val: i, Left: complete(2*i+1, n), Right: complete(2*i+2, n)}
}

func leftChain(n int) *Node {
	var root *Node
	for i := n - 1; i >= 0; i-- {
		root = &Node{Val: i, Left: root}
	}
	return root
}

func main() {
	root := handBuilt()
	var ds []string
	depths(root, 0, &ds)
	var best, calls int
	onePass(root, &best, &calls)
	fmt.Println("hand-built: height", height(root), "nodes; depth of every node (val:depth)", ds)
	fmt.Println("            diameter", best, "edges (one pass), oracle says", oracle(root))
	fmt.Printf("convention: height %d nodes = %d edges; diameter %d edges = %d nodes on the path\n",
		height(root), height(root)-1, best, best+1)

	r := rand.New(rand.NewPCG(1, 2))
	mismatches := 0
	for range 3000 {
		t := randomSplit(r.IntN(60)+1, r)
		var v, b, c int
		onePass(t, &b, &c)
		if o := oracle(t); naive(t, &v) != o || b != o {
			mismatches++
		}
	}
	fmt.Printf("\nnaive vs one-pass vs oracle: %d mismatches over 3000 trees\n", mismatches)

	r = rand.New(rand.NewPCG(1, 2))
	shapes := []struct {
		name string
		root *Node
	}{
		{"leftChain(1000)", leftChain(1000)},
		{"perfect(10)", complete(0, 1<<10-1)},
		{"randomSplit(100000)", randomSplit(100000, r)},
	}
	fmt.Println()
	fmt.Printf("%-20s %7s %5s %9s %12s %12s %10s %8s\n",
		"shape", "n", "h", "diameter", "naive visits", "sizes sum-n", "one-pass", "ratio")
	allMatch := true
	for _, s := range shapes {
		var visits, best, calls int
		d := naive(s.root, &visits)
		onePass(s.root, &best, &calls)
		n, total := sumSizes(s.root)
		allMatch = allMatch && visits == total-n && d == best && calls == 2*n+1
		fmt.Printf("%-20s %7d %5d %9d %12d %12d %10d %7.1fx\n",
			s.name, n, height(s.root), d, visits, total-n, calls, float64(visits)/float64(calls))
	}
	fmt.Println("naive visits == (sum of all subtree sizes) - n, one-pass calls == 2n+1, same diameter:", allMatch)
}
```

**Output:**

```
hand-built: height 4 nodes; depth of every node (val:depth) [1:0 2:1 4:2 5:2 7:3 3:1 6:2 8:3 9:3]
            diameter 6 edges (one pass), oracle says 6
convention: height 4 nodes = 3 edges; diameter 6 edges = 7 nodes on the path

naive vs one-pass vs oracle: 0 mismatches over 3000 trees

shape                      n     h  diameter naive visits  sizes sum-n   one-pass    ratio
leftChain(1000)         1000  1000       999       499500       499500       2001   249.6x
perfect(10)             1023    10        18         8194         8194       2047     4.0x
randomSplit(100000)   100000    38        71      1974106      1974106     200001     9.9x
naive visits == (sum of all subtree sizes) - n, one-pass calls == 2n+1, same diameter: true
```

**Rust:**

```rust
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (a chain of 1,000 nodes would be fine, a chain of millions
// would not: example 13). Go has no destructor to write.
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

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
fn hand_built() -> Link {
    node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    )
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of IntN, copied
// from lesson 01 example 14, so the same seed builds the same tree as in Go.
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

    /// Like rand.IntN(n): exactly uniform in [0, n), Lemire's method.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

/// Convention, written once: height counts NODES on the longest root-to-leaf
/// path (empty tree = 0, a leaf = 1). Depth counts EDGES from the root (root = 0).
/// Diameter counts EDGES on the longest path between any two nodes. With `usize`
/// there is no -1, so "empty = 0 nodes" is the convention that needs no Option.
///
/// `height` is a post-order recursion: a node needs its children's answers.
fn height(n: &Link) -> usize {
    match n {
        None => 0,
        Some(b) => 1 + height(&b.left).max(height(&b.right)),
    }
}

/// `depths` is the opposite direction: the depth is a number passed DOWN.
fn depths(n: &Link, d: usize, out: &mut Vec<String>) {
    let Some(b) = n else { return };
    out.push(format!("{}:{}", b.val, d));
    depths(&b.left, d + 1, out);
    depths(&b.right, d + 1, out);
}

// The longest path THROUGH a node goes down both sides: height(left) + height(right)
// edges, since heights count nodes. The diameter is the best of that over all nodes.

/// height that also counts the nodes it walks.
fn height_count(n: &Link, visits: &mut usize) -> usize {
    let Some(b) = n else { return 0 };
    *visits += 1;
    1 + height_count(&b.left, visits).max(height_count(&b.right, visits))
}

/// Calls height at EVERY node: the cost is the sum of the subtree sizes.
fn naive(n: &Link, visits: &mut usize) -> usize {
    let Some(b) = n else { return 0 };
    let through = height_count(&b.left, visits) + height_count(&b.right, visits);
    through
        .max(naive(&b.left, visits))
        .max(naive(&b.right, visits))
}

/// RETURNS the height and RECORDS the best bend on the side. Rust has no mutable
/// globals and no free pointers to a counter: `best` and `calls` are `&mut`
/// parameters threaded through every call. One call per node and per nil link.
fn one_pass(n: &Link, best: &mut usize, calls: &mut usize) -> usize {
    *calls += 1;
    let Some(b) = n else { return 0 };
    let (l, r) = (
        one_pass(&b.left, best, calls),
        one_pass(&b.right, best, calls),
    );
    *best = (*best).max(l + r);
    1 + l.max(r)
}

/// The same pass with nothing threaded: return BOTH numbers as a tuple.
fn pair(n: &Link) -> (usize, usize) {
    let Some(b) = n else { return (0, 0) };
    let ((lh, ld), (rh, rd)) = (pair(&b.left), pair(&b.right));
    (1 + lh.max(rh), ld.max(rd).max(lh + rh)) // (height, diameter)
}

/// Shares nothing with the passes above: number the nodes, make the tree an
/// undirected graph, BFS from every node, take the largest distance seen.
fn add_to_graph(n: &Node, adj: &mut Vec<Vec<usize>>) -> usize {
    let id = adj.len();
    adj.push(Vec::new());
    for c in [n.left.as_deref(), n.right.as_deref()]
        .into_iter()
        .flatten()
    {
        let cid = add_to_graph(c, adj);
        adj[id].push(cid);
        adj[cid].push(id);
    }
    id
}

fn oracle(root: &Link) -> usize {
    let mut adj = Vec::new();
    if let Some(r) = root {
        add_to_graph(r, &mut adj);
    }
    let mut best = 0;
    for src in 0..adj.len() {
        let mut dist = vec![usize::MAX; adj.len()]; // MAX = not reached yet
        dist[src] = 0;
        let mut q = std::collections::VecDeque::from([src]);
        while let Some(u) = q.pop_front() {
            for &v in &adj[u] {
                if dist[v] == usize::MAX {
                    dist[v] = dist[u] + 1;
                    best = best.max(dist[v]);
                    q.push_back(v);
                }
            }
        }
    }
    best
}

/// The size of n's subtree and the sum of ALL subtree sizes in it.
fn sum_sizes(n: &Link) -> (usize, usize) {
    let Some(b) = n else { return (0, 0) };
    let ((ls, lt), (rs, rt)) = (sum_sizes(&b.left), sum_sizes(&b.right));
    let size = 1 + ls + rs;
    (size, size + lt + rt)
}

// ---- shapes (random: the draw comes BEFORE the recursion, as in Go) ----

fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    node(0, left, right)
}

/// Heap-shaped: node i has children 2i+1 and 2i+2 while they are < n.
fn complete(i: usize, n: usize) -> Link {
    if i >= n {
        None
    } else {
        node(i as i64, complete(2 * i + 1, n), complete(2 * i + 2, n))
    }
}

fn left_chain(n: usize) -> Link {
    let mut root = None;
    for i in (0..n).rev() {
        root = node(i as i64, root, None);
    }
    root
}

fn main() {
    let root = hand_built();
    let mut ds = Vec::new();
    depths(&root, 0, &mut ds);
    let (mut best, mut calls) = (0, 0);
    one_pass(&root, &mut best, &mut calls);
    println!(
        "hand-built: height {} nodes; depth of every node (val:depth) [{}]",
        height(&root),
        ds.join(" ")
    );
    println!(
        "            diameter {} edges (one pass), oracle says {}",
        best,
        oracle(&root)
    );
    println!(
        "convention: height {} nodes = {} edges; diameter {} edges = {} nodes on the path",
        height(&root),
        height(&root) - 1,
        best,
        best + 1
    );

    let mut r = Pcg::new(1, 2);
    let (mut mismatches, mut tuple_mismatches) = (0, 0);
    for _ in 0..3000 {
        let n = r.int_n(60) + 1;
        let t = random_split(n, &mut r);
        let (mut v, mut b, mut c) = (0, 0, 0);
        one_pass(&t, &mut b, &mut c);
        let o = oracle(&t);
        if naive(&t, &mut v) != o || b != o {
            mismatches += 1;
        }
        if pair(&t).1 != o {
            tuple_mismatches += 1;
        }
    }
    println!("\nnaive vs one-pass vs oracle: {mismatches} mismatches over 3000 trees");
    println!(
        "the (height, diameter) tuple form vs oracle: {tuple_mismatches} mismatches over the same 3000 trees"
    );

    let mut r = Pcg::new(1, 2);
    let shapes: Vec<(&str, Link)> = vec![
        ("leftChain(1000)", left_chain(1000)),
        ("perfect(10)", complete(0, (1 << 10) - 1)),
        ("randomSplit(100000)", random_split(100000, &mut r)),
    ];
    println!();
    println!(
        "{:<20} {:>7} {:>5} {:>9} {:>12} {:>12} {:>10} {:>8}",
        "shape", "n", "h", "diameter", "naive visits", "sizes sum-n", "one-pass", "ratio"
    );
    let mut all_match = true;
    for (name, tree) in &shapes {
        let (mut visits, mut best, mut calls) = (0, 0, 0);
        let d = naive(tree, &mut visits);
        one_pass(tree, &mut best, &mut calls);
        let (n, total) = sum_sizes(tree);
        all_match = all_match && visits == total - n && d == best && calls == 2 * n + 1;
        println!(
            "{:<20} {:>7} {:>5} {:>9} {:>12} {:>12} {:>10} {:>7.1}x",
            name,
            n,
            height(tree),
            d,
            visits,
            total - n,
            calls,
            visits as f64 / calls as f64
        );
    }
    println!(
        "naive visits == (sum of all subtree sizes) - n, one-pass calls == 2n+1, same diameter: {all_match}"
    );
}
```

**Rust output:**

```
hand-built: height 4 nodes; depth of every node (val:depth) [1:0 2:1 4:2 5:2 7:3 3:1 6:2 8:3 9:3]
            diameter 6 edges (one pass), oracle says 6
convention: height 4 nodes = 3 edges; diameter 6 edges = 7 nodes on the path

naive vs one-pass vs oracle: 0 mismatches over 3000 trees
the (height, diameter) tuple form vs oracle: 0 mismatches over the same 3000 trees

shape                      n     h  diameter naive visits  sizes sum-n   one-pass    ratio
leftChain(1000)         1000  1000       999       499500       499500       2001   249.6x
perfect(10)             1023    10        18         8194         8194       2047     4.0x
randomSplit(100000)   100000    38        71      1974106      1974106     200001     9.9x
naive visits == (sum of all subtree sizes) - n, one-pass calls == 2n+1, same diameter: true
```

> **Go vs Rust:** Identical output for every line the two programs share (the Rust run adds one line, the tuple form). Rust has no mutable globals, so `best` and `calls` are `&mut usize` parameters threaded through the recursion; the alternative shown here is `fn pair(n) -> (usize, usize)`, which returns `(height, diameter)` and threads nothing, and agrees with the oracle on the same 3,000 trees. With `usize` there is no `-1`, so height in nodes (empty = 0) is the convention that needs no `Option`.

**Complexity:** height and depth Θ(n) time and Θ(h) stack · diameter in one pass Θ(n), with exactly 2n + 1 calls; the naive version walks **the sum of all subtree sizes, minus n** nodes (it calls `height` on the two children of every node), which is **Θ(n²) on a chain**: 499,500 node visits against 2,001 one-pass calls on the 1,000-node chain (249.6x), 8,194 against 2,047 on perfect(10) (4.0x), and 1,974,106 against 200,001 on the 100,000-node random tree (9.9x). All three diameters agreed on 3,000 random trees (0 mismatches); the hand-built tree has height 4 nodes (3 edges) and diameter 6 edges.

---

## 6. Invert and prune: assign what the child returns

`🟢 easy` · *Mutation by return value*

Every tree mutation is `n.Left = fix(n.Left)`: the recursive call returns what the subtree should now be, and the parent stores it. **Invert** swaps the children at every node, in place, so it allocates nothing; **prune** drops every subtree with no positive value by returning `nil` from the post-order recursion; attaching a leaf at a path is the same move with a pointer to the link in place of a recursion. Invert and prune are Θ(n) time and Θ(h) stack, attach is Θ(path length), and the example counts allocations (in place against a copy) and checks prune against an oracle that shares no code with it: a node survives if and only if its subtree contains a positive value, decided naively and compared by `L`/`R` path.

**Steps:**

1. Write `invert` as `n.Left, n.Right = invert(n.Right), invert(n.Left)`; print the hand-built tree before and after, check on 1,000 random trees that inverting twice restores the tree and that the in-order sequence reverses, and count allocations against the copying `invertCopy` on 10,000 nodes.
2. Write `prune` (post-order: decide after both children have) and compare the surviving paths with the O(n²) oracle on 2,000 random trees with values in [-2, 2]; print one small tree before and after.
3. Write `attach` with a `**Node` and replace a 5-node subtree by a leaf, and see what each language can say about the 5 old nodes.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
)

type Node struct {
	Val         int
	Left, Right *Node
}

func leaf(v int) *Node { return &Node{Val: v} }

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
func handBuilt() *Node {
	return &Node{1,
		&Node{2, leaf(4), &Node{5, leaf(7), nil}},
		&Node{3, nil, &Node{6, leaf(8), leaf(9)}}}
}

// Every tree mutation has one shape: n.Left = fix(n.Left). The call RETURNS
// what the subtree should now be, and the parent stores it.

// invert swaps the children at every node, in place. It reuses every node, so it
// allocates nothing; the swap is the same move, with two returned subtrees.
func invert(n *Node) *Node {
	if n == nil {
		return nil
	}
	n.Left, n.Right = invert(n.Right), invert(n.Left)
	return n
}

// invertCopy leaves the input alone and builds a new tree: one allocation per node.
func invertCopy(n *Node) *Node {
	if n == nil {
		return nil
	}
	return &Node{n.Val, invertCopy(n.Right), invertCopy(n.Left)}
}

// prune drops every subtree in which no value is positive and returns the new root.
// It is post-order: a node can only decide after its children have.
func prune(n *Node) *Node {
	if n == nil {
		return nil
	}
	n.Left = prune(n.Left)
	n.Right = prune(n.Right)
	if n.Val <= 0 && n.Left == nil && n.Right == nil {
		return nil // not positive itself, and nothing positive below it
	}
	return n
}

// attach replaces the link at `path` (L/R letters) with a new leaf. It holds a
// **Node, a pointer to a LINK, so it can overwrite the link itself.
func attach(root **Node, path string, v int) {
	link := root
	for _, c := range path {
		if c == 'L' {
			link = &(*link).Left
		} else {
			link = &(*link).Right
		}
	}
	*link = leaf(v)
}

// ---- helpers and oracles ----

func clone(n *Node) *Node {
	if n == nil {
		return nil
	}
	return &Node{n.Val, clone(n.Left), clone(n.Right)}
}

func equal(a, b *Node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Val == b.Val && equal(a.Left, b.Left) && equal(a.Right, b.Right)
}

func size(n *Node) int {
	if n == nil {
		return 0
	}
	return 1 + size(n.Left) + size(n.Right)
}

func inOrder(n *Node) (out []int) {
	if n != nil {
		out = append(inOrder(n.Left), n.Val)
		out = append(out, inOrder(n.Right)...)
	}
	return
}

// paths lists the L/R path of every node that satisfies keep, in pre-order. A
// node IS its path, which makes it an oracle that shares no code with prune.
func paths(n *Node, p string, keep func(*Node) bool, out *[]string) {
	if n == nil {
		return
	}
	if keep(n) {
		*out = append(*out, p)
	}
	paths(n.Left, p+"L", keep, out)
	paths(n.Right, p+"R", keep, out)
}

// hasPositive is the naive definition: look at every value in the subtree.
func hasPositive(n *Node) bool {
	return n != nil && (n.Val > 0 || hasPositive(n.Left) || hasPositive(n.Right))
}

func sideways(n *Node, depth int) {
	if n == nil {
		return
	}
	sideways(n.Right, depth+1)
	fmt.Printf("%s%d\n", strings.Repeat("    ", depth), n.Val)
	sideways(n.Left, depth+1)
}

// randomSplit: the draw comes BEFORE the recursion (the Rust twin makes the same
// draws); labels are a separate pass, so one seed gives one shape.
func randomSplit(n int, r *rand.Rand) *Node {
	if n == 0 {
		return nil
	}
	k := r.IntN(n)
	left := randomSplit(k, r)
	right := randomSplit(n-1-k, r)
	return &Node{Left: left, Right: right}
}

func labelInOrder(n *Node, next *int) {
	if n != nil {
		labelInOrder(n.Left, next)
		n.Val = *next
		*next++
		labelInOrder(n.Right, next)
	}
}

func labelRange(n *Node, lo, hi int, r *rand.Rand) { // pre-order draws
	if n != nil {
		n.Val = r.IntN(hi-lo+1) + lo
		labelRange(n.Left, lo, hi, r)
		labelRange(n.Right, lo, hi, r)
	}
}

var sink *Node

func main() {
	root := handBuilt()
	before := inOrder(root)
	fmt.Println("hand-built tree:")
	sideways(root, 0)
	invert(root)
	fmt.Println("after invert (in place):")
	sideways(root, 0)
	fmt.Println("in-order before:", before)
	fmt.Println("in-order after: ", inOrder(root), " (the reverse)")

	r := rand.New(rand.NewPCG(1, 2))
	bad := 0
	for range 1000 {
		t := randomSplit(r.IntN(61), r) // 0..60 nodes, the empty tree included
		labelInOrder(t, new(int))
		orig := clone(t)
		in := inOrder(t)
		invert(t)
		slices.Reverse(in)
		ok := slices.Equal(inOrder(t), in)
		invert(t)
		if !ok || !equal(t, orig) {
			bad++
		}
	}
	fmt.Printf("\ninvert twice == original and in-order == reverse: %d mismatches over 1000 random trees\n", bad)

	big := randomSplit(10000, r)
	n := size(big)
	inPlace := testing.AllocsPerRun(10, func() { invert(big) })
	copied := testing.AllocsPerRun(10, func() { sink = invertCopy(big) })
	fmt.Printf("allocations to invert a %d-node tree: in place %.0f, copy %.0f\n", n, inPlace, copied)

	r = rand.New(rand.NewPCG(1, 2))
	bad = 0
	for range 2000 {
		t := randomSplit(r.IntN(61), r)
		labelRange(t, -2, 2, r)
		var want, got []string
		paths(t, "", hasPositive, &want)
		t = prune(t)
		paths(t, "", func(*Node) bool { return true }, &got)
		if !slices.Equal(want, got) {
			bad++
		}
	}
	fmt.Printf("\nprune vs the O(n^2) oracle (a node survives iff its subtree holds a positive value): %d mismatches over 2000 trees\n", bad)

	t := randomSplit(10, r)
	labelRange(t, -2, 2, r)
	fmt.Println("one tree, values in [-2, 2]:")
	sideways(t, 0)
	n = size(t)
	t = prune(t)
	fmt.Printf("after prune: kept %d, removed %d\n", size(t), n-size(t))
	sideways(t, 0)

	// attach: a path is a list of choices, and a **Node can replace what it ends on.
	root = handBuilt()
	n0 := size(root)
	attach(&root, "RL", 10)
	n1 := size(root)
	fmt.Printf("\nattach 10 at path RL: size %d -> %d, in-order %v\n", n0, n1, inOrder(root))
	old := size(root.Right)
	attach(&root, "R", 99)
	fmt.Printf("replace the %d-node subtree at path R by a leaf: size %d -> %d, in-order %v\n",
		old, n1, size(root), inOrder(root))
	fmt.Printf("(Go has no destructor to count: the %d old nodes are the collector's to find)\n", old)
}
```

**Output:**

```
hand-built tree:
            9
        6
            8
    3
1
        5
            7
    2
        4
after invert (in place):
        4
    2
            7
        5
1
    3
            8
        6
            9
in-order before: [4 2 7 5 1 3 8 6 9]
in-order after:  [9 6 8 3 1 5 7 2 4]  (the reverse)

invert twice == original and in-order == reverse: 0 mismatches over 1000 random trees
allocations to invert a 10000-node tree: in place 0, copy 10000

prune vs the O(n^2) oracle (a node survives iff its subtree holds a positive value): 0 mismatches over 2000 trees
one tree, values in [-2, 2]:
        2
    -2
-2
            -2
        -2
    0
                -1
            -1
                -2
        0
after prune: kept 3, removed 7
        2
    -2
-2

attach 10 at path RL: size 9 -> 10, in-order [4 2 7 5 1 10 3 8 6 9]
replace the 5-node subtree at path R by a leaf: size 10 -> 6, in-order [4 2 7 5 1 99]
(Go has no destructor to count: the 5 old nodes are the collector's to find)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

#[derive(Clone, PartialEq)]
struct Node {
    val: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

thread_local! {
    // Counts Node::drop calls: every node that is freed runs Drop exactly once.
    static DROPS: Cell<usize> = const { Cell::new(0) };
}

// Iterative Drop (a degenerate chain would overflow the default recursive glue:
// example 13). Go has no destructor, so it has nothing to count here.
impl Drop for Node {
    fn drop(&mut self) {
        DROPS.set(DROPS.get() + 1);
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

// The hand-built 9-node tree: 1(2(4, 5(7, -)), 3(-, 6(8, 9))).
fn hand_built() -> Link {
    node(
        1,
        node(2, leaf(4), node(5, leaf(7), None)),
        node(3, None, node(6, leaf(8), leaf(9))),
    )
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of IntN, copied
// from lesson 01 example 14, so the same seed builds the same tree as in Go.
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

    /// Like rand.IntN(n): exactly uniform in [0, n), Lemire's method.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

// ---- allocation counter: stand-in for runtime.MemStats / testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes ever requested

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

/// (allocation events, bytes requested, result) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

// Every tree mutation has one shape: the subtree becomes whatever the call
// returns. Rust gives that shape three spellings, depending on who owns the tree.

/// Borrowed: `&mut Link` and `mem::swap` the two links; reuses every node.
fn invert(n: &mut Link) {
    if let Some(b) = n {
        std::mem::swap(&mut b.left, &mut b.right);
        invert(&mut b.left);
        invert(&mut b.right);
    }
}

/// Owned: take the tree in, take each child out with `take()`, store what the
/// recursive calls return, hand the same boxes back. Go's `n.Left, n.Right =
/// invert(n.Right), invert(n.Left)` is exactly this, with the moves written out.
fn invert_owned(n: Link) -> Link {
    n.map(|mut b| {
        let (l, r) = (b.left.take(), b.right.take());
        b.left = invert_owned(r);
        b.right = invert_owned(l);
        b
    })
}

/// Copy: leaves the input alone and builds a new tree, one Box per node.
fn invert_copy(n: &Link) -> Link {
    n.as_ref().map(|b| {
        Box::new(Node {
            val: b.val,
            left: invert_copy(&b.right),
            right: invert_copy(&b.left),
        })
    })
}

/// Drops every subtree in which no value is positive and returns the new root.
/// Post-order: a node can only decide after its children have. It takes
/// ownership, so a dropped subtree is freed right here.
fn prune(n: Link) -> Link {
    let mut b = n?; // None in, None out
    b.left = prune(b.left.take());
    b.right = prune(b.right.take());
    if b.val <= 0 && b.left.is_none() && b.right.is_none() {
        return None; // not positive itself, and nothing positive below it
    }
    Some(b)
}

/// Replaces the link at `path` (L/R letters) with a new leaf. `link` is a cursor:
/// a `&mut Link` that is re-pointed at a child link on every step. Assigning
/// through it DROPS whatever subtree was there.
fn attach(root: &mut Link, path: &str, val: i64) {
    let mut link = root;
    for c in path.chars() {
        let node = link.as_mut().expect("the path must stay inside the tree");
        link = if c == 'L' {
            &mut node.left
        } else {
            &mut node.right
        };
    }
    *link = leaf(val);
}

// ---- helpers and oracles ----

fn size(n: &Link) -> usize {
    n.as_ref().map_or(0, |b| 1 + size(&b.left) + size(&b.right))
}

fn in_order(n: &Link) -> Vec<i64> {
    fn go(n: &Link, out: &mut Vec<i64>) {
        if let Some(b) = n {
            go(&b.left, out);
            out.push(b.val);
            go(&b.right, out);
        }
    }
    let mut out = Vec::new();
    go(n, &mut out);
    out
}

/// Lists the L/R path of every node that satisfies `keep`, in pre-order. A node
/// IS its path, which makes it an oracle that shares no code with `prune`.
fn paths(n: &Link, p: String, keep: &impl Fn(&Node) -> bool, out: &mut Vec<String>) {
    let Some(b) = n else { return };
    if keep(b) {
        out.push(p.clone());
    }
    paths(&b.left, format!("{p}L"), keep, out);
    paths(&b.right, format!("{p}R"), keep, out);
}

/// The naive definition: look at every value in the subtree.
fn has_positive(n: &Node) -> bool {
    n.val > 0
        || n.left.as_deref().is_some_and(has_positive)
        || n.right.as_deref().is_some_and(has_positive)
}

fn sideways(n: &Link, depth: usize) {
    if let Some(b) = n {
        sideways(&b.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), b.val);
        sideways(&b.left, depth + 1);
    }
}

/// The draw comes BEFORE the recursion (as in Go); labels are a separate pass,
/// so one seed gives one shape.
fn random_split(n: usize, r: &mut Pcg) -> Link {
    if n == 0 {
        return None;
    }
    let k = r.int_n(n);
    let left = random_split(k, r);
    let right = random_split(n - 1 - k, r);
    node(0, left, right)
}

fn label_in_order(n: &mut Link, next: &mut i64) {
    if let Some(b) = n {
        label_in_order(&mut b.left, next);
        b.val = *next;
        *next += 1;
        label_in_order(&mut b.right, next);
    }
}

fn label_range(n: &mut Link, lo: i64, hi: i64, r: &mut Pcg) {
    // pre-order draws
    if let Some(b) = n {
        b.val = r.int_n((hi - lo + 1) as usize) as i64 + lo;
        label_range(&mut b.left, lo, hi, r);
        label_range(&mut b.right, lo, hi, r);
    }
}

/// Go's %v of a slice: [1 2 3]
fn go_list(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let mut root = hand_built();
    let before = in_order(&root);
    println!("hand-built tree:");
    sideways(&root, 0);
    invert(&mut root);
    println!("after invert (in place):");
    sideways(&root, 0);
    println!("in-order before: {}", go_list(&before));
    println!(
        "in-order after:  {}  (the reverse)",
        go_list(&in_order(&root))
    );

    let mut r = Pcg::new(1, 2);
    let mut bad = 0;
    for _ in 0..1000 {
        let mut t = random_split(r.int_n(61), &mut r); // 0..60 nodes, the empty tree included
        label_in_order(&mut t, &mut 0);
        let orig = t.clone();
        let mut rev = in_order(&t);
        rev.reverse();
        invert(&mut t);
        // the by-value form must agree with the in-place one
        let ok = in_order(&t) == rev && t == invert_owned(orig.clone());
        invert(&mut t);
        if !ok || t != orig {
            bad += 1;
        }
    }
    println!(
        "\ninvert twice == original and in-order == reverse: {bad} mismatches over 1000 random trees"
    );

    let mut big = random_split(10000, &mut r);
    let n = size(&big);
    let (in_place, _, ()) = allocs_of(|| invert(&mut big));
    let (copied, bytes, copy) = allocs_of(|| invert_copy(&big));
    let (owned, _, inverted) = allocs_of(|| invert_owned(big.take()));
    println!("allocations to invert a {n}-node tree: in place {in_place}, copy {copied}");
    println!(
        "  Rust adds: the by-value form {owned}; the copy requested {bytes} bytes = {} per node",
        bytes / n
    );
    drop((copy, inverted));

    let mut r = Pcg::new(1, 2);
    let mut bad = 0;
    for _ in 0..2000 {
        let mut t = random_split(r.int_n(61), &mut r);
        label_range(&mut t, -2, 2, &mut r);
        let (mut want, mut got) = (Vec::new(), Vec::new());
        paths(&t, String::new(), &has_positive, &mut want);
        let t = prune(t);
        paths(&t, String::new(), &|_| true, &mut got);
        if want != got {
            bad += 1;
        }
    }
    println!(
        "\nprune vs the O(n^2) oracle (a node survives iff its subtree holds a positive value): {bad} mismatches over 2000 trees"
    );

    let mut t = random_split(10, &mut r);
    label_range(&mut t, -2, 2, &mut r);
    println!("one tree, values in [-2, 2]:");
    sideways(&t, 0);
    let n = size(&t);
    let t = prune(t);
    println!("after prune: kept {}, removed {}", size(&t), n - size(&t));
    sideways(&t, 0);

    // attach: a path is a list of choices, and a cursor can replace what it ends on.
    let mut root = hand_built();
    let n0 = size(&root);
    attach(&mut root, "RL", 10);
    let n1 = size(&root);
    println!(
        "\nattach 10 at path RL: size {n0} -> {n1}, in-order {}",
        go_list(&in_order(&root))
    );
    let old = size(&root.as_ref().expect("root exists").right);
    let drops_before = DROPS.get();
    attach(&mut root, "R", 99);
    println!(
        "replace the {old}-node subtree at path R by a leaf: size {n1} -> {}, in-order {}",
        size(&root),
        go_list(&in_order(&root))
    );
    println!(
        "Rust drops them at once: Node::drop ran {} times during that one assignment",
        DROPS.get() - drops_before
    );
}
```

**Rust output:**

```
hand-built tree:
            9
        6
            8
    3
1
        5
            7
    2
        4
after invert (in place):
        4
    2
            7
        5
1
    3
            8
        6
            9
in-order before: [4 2 7 5 1 3 8 6 9]
in-order after:  [9 6 8 3 1 5 7 2 4]  (the reverse)

invert twice == original and in-order == reverse: 0 mismatches over 1000 random trees
allocations to invert a 10000-node tree: in place 0, copy 10000
  Rust adds: the by-value form 0; the copy requested 240000 bytes = 24 per node

prune vs the O(n^2) oracle (a node survives iff its subtree holds a positive value): 0 mismatches over 2000 trees
one tree, values in [-2, 2]:
        2
    -2
-2
            -2
        -2
    0
                -1
            -1
                -2
        0
after prune: kept 3, removed 7
        2
    -2
-2

attach 10 at path RL: size 9 -> 10, in-order [4 2 7 5 1 10 3 8 6 9]
replace the 5-node subtree at path R by a leaf: size 10 -> 6, in-order [4 2 7 5 1 99]
Rust drops them at once: Node::drop ran 5 times during that one assignment
```

> **Go vs Rust:** Identical for every line the two programs share (0 and 10,000 allocations, the same pruned tree); the Rust run adds three facts Go cannot show. `invert_owned` takes the tree by value and hands the same boxes back, so it also allocates 0 (and agrees with the in-place `&mut` form on the 1,000 random trees); the copy asked the allocator for 240,000 bytes, 24 per node; and assigning through the `&mut Link` cursor drops the replaced subtree at once, which a `thread_local` counter in `Drop` saw as 5 `Node::drop` calls. Go has no destructor to count, so it can only say the old nodes are the collector's to find. The allocation counts come from the counting allocator in the shared kit (a `#[global_allocator]`, hence the `unsafe impl`), standing in for `testing.AllocsPerRun`.

**Complexity:** invert Θ(n) time, Θ(h) stack, **0 allocations in place against 10,000 for the copy** of a 10,000-node tree (one per node) · prune Θ(n), 0 mismatches against the O(n²) oracle on 2,000 trees (on the example tree it kept 3 of 10 nodes: a non-positive ancestor stays when something positive is below it) · attach Θ(path length). Inverting twice restored the tree and the in-order sequence reversed on all 1,000 random trees.

---

> Next tier: [🟡 medium](2-medium.md) — path sums, two trees at once, the lowest common ancestor,
> serialization, and rebuilding a tree from its traversals.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
