# Step 18 — Binary Search Trees · 🟡 Medium

Examples **7–11**: validation with the min/max bounds, the k-th smallest and what a size field buys, range
queries, building a balanced tree from a sorted slice, and two-sum with two iterators.

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

## 7. Validation: the bounds trick, not the naive check

`🟡 medium` · *Validate a BST*

The check everyone writes first compares each node with its two children, and it is wrong: a grandchild can sit on the wrong side of an ancestor. The smallest counterexample has three nodes (root 2, right child 3, whose left child is 1), and the program then measures how common the mistake is by brute force: every labelled binary tree on 1 to 7 nodes (2,162,160 of them at n = 7) is judged by five validators and compared with the O(n²) definition. The fix is to hand each subtree the open interval `(lo, hi)` its keys must lie in, going left tightens `hi` and going right tightens `lo`: Θ(n) time, Θ(h) stack. Two traps follow, sentinel bounds (`MinInt`/`MaxInt` as "unbounded" rejects valid trees) and what a `NaN` key does to a descent.

**Steps:**

1. Write the five validators (neighbours only, bounds, in-order scan, streaming scan with a `prev` pointer, and the literal O(n²) definition); print the 3-node and a 7-node counterexample sideways with the interval each key must lie in.
2. Enumerate every shape (Catalan(n)) and write every permutation of 1..n onto it in pre-order, and tally each validator's disagreements with the definition for n = 1..7; then try the sentinel version on trees holding `MinInt` and `MaxInt`.
3. Read the table: the naive column is the only one that is ever wrong, and only in one direction; read the sentinel rows and the NaN rows (inserting 5, NaN, 3, NaN under two comparisons).

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"strings"
)

// A node holds one key; nil is the empty tree.
type node struct {
	key         int
	left, right *node
}

func mk(key int, l, r *node) *node { return &node{key, l, r} }

// 1. The check everyone writes first: each node against its two CHILDREN.
func naive(n *node) bool {
	if n == nil {
		return true
	}
	if n.left != nil && n.left.key >= n.key || n.right != nil && n.right.key <= n.key {
		return false
	}
	return naive(n.left) && naive(n.right)
}

// 2. The bounds check: every key must lie in the open interval (lo, hi) its
// ancestors hand down. Going left tightens hi, going right tightens lo. nil
// means "unbounded", so no real key can be mistaken for a sentinel.
func bounds(n *node, lo, hi *int) bool {
	if n == nil {
		return true
	}
	if lo != nil && n.key <= *lo || hi != nil && n.key >= *hi {
		return false
	}
	return bounds(n.left, lo, &n.key) && bounds(n.right, &n.key, hi)
}

// 3. In-order must be strictly ascending (collect, then scan).
func inOrder(n *node, out *[]int) {
	if n != nil {
		inOrder(n.left, out)
		*out = append(*out, n.key)
		inOrder(n.right, out)
	}
}

func ascending(root *node) bool {
	var xs []int
	inOrder(root, &xs)
	for i := 1; i < len(xs); i++ {
		if xs[i-1] >= xs[i] {
			return false
		}
	}
	return true
}

// 4. The same test with no slice: remember only the previously visited node.
func streaming(n *node, prev **node) bool {
	if n == nil {
		return true
	}
	if !streaming(n.left, prev) || *prev != nil && (*prev).key >= n.key {
		return false
	}
	*prev = n
	return streaming(n.right, prev)
}

// 5. The definition, taken literally: every key in the left subtree is
// smaller and every key in the right subtree larger. O(n^2): the ground truth.
func all(n *node, ok func(int) bool) bool {
	return n == nil || ok(n.key) && all(n.left, ok) && all(n.right, ok)
}

func definition(n *node) bool {
	if n == nil {
		return true
	}
	return all(n.left, func(k int) bool { return k < n.key }) &&
		all(n.right, func(k int) bool { return k > n.key }) &&
		definition(n.left) && definition(n.right)
}

// The sentinel trap: "unbounded" spelled as the extreme ints.
func sentinel(n *node, lo, hi int) bool {
	if n == nil {
		return true
	}
	return n.key > lo && n.key < hi && sentinel(n.left, lo, n.key) && sentinel(n.right, n.key, hi)
}

func interval(lo, hi *int) string {
	l, h := "-inf", "+inf"
	if lo != nil {
		l = fmt.Sprint(*lo)
	}
	if hi != nil {
		h = fmt.Sprint(*hi)
	}
	return "(" + l + ", " + h + ")"
}

// explain prints the tree sideways with the interval each key must lie in.
func explain(n *node, lo, hi *int, depth int) {
	if n == nil {
		return
	}
	explain(n.right, &n.key, hi, depth+1)
	verdict := "ok"
	if lo != nil && n.key <= *lo || hi != nil && n.key >= *hi {
		verdict = "VIOLATION"
	}
	fmt.Printf("  %-14s must lie in %-12s %s\n", strings.Repeat("    ", depth)+fmt.Sprint(n.key), interval(lo, hi), verdict)
	explain(n.left, lo, &n.key, depth+1)
}

func verdicts(name string, t *node) {
	var prev *node
	fmt.Printf("  %-26s naive=%-5t bounds=%-5t in-order=%-5t streaming=%-5t definition=%t\n",
		name, naive(t), bounds(t, nil, nil), ascending(t), streaming(t, &prev), definition(t))
}

// ---- every labelled tree on n nodes ----

func clone(n *node) *node {
	if n == nil {
		return nil
	}
	return &node{n.key, clone(n.left), clone(n.right)}
}

// shapes returns every binary-tree shape with n nodes (Catalan(n) of them).
func shapes(n int) []*node {
	if n == 0 {
		return []*node{nil}
	}
	var out []*node
	for l := 0; l < n; l++ {
		for _, ls := range shapes(l) {
			for _, rs := range shapes(n - 1 - l) {
				out = append(out, &node{left: clone(ls), right: clone(rs)})
			}
		}
	}
	return out
}

// label writes p onto the tree in pre-order.
func label(n *node, p []int, i *int) {
	if n != nil {
		n.key = p[*i]
		*i++
		label(n.left, p, i)
		label(n.right, p, i)
	}
}

func permute(a []int, k int, visit func()) {
	if k == len(a) {
		visit()
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

// ---- NaN: Go lets you build the tree, so the comparison decides ----

type fnode struct {
	key         float64
	left, right *fnode
}

// insertF reports whether it added a key; c is the three-way comparison.
func insertF(link **fnode, key float64, c func(a, b float64) int) bool {
	for *link != nil {
		switch d := c(key, (*link).key); {
		case d < 0:
			link = &(*link).left
		case d > 0:
			link = &(*link).right
		default:
			return false
		}
	}
	*link = &fnode{key: key}
	return true
}

func keysF(n *fnode, out *[]float64) {
	if n != nil {
		keysF(n.left, out)
		*out = append(*out, n.key)
		keysF(n.right, out)
	}
}

// operators is the descent written with < and >, anything else "found".
func operators(a, b float64) int {
	if a < b {
		return -1
	}
	if a > b {
		return 1
	}
	return 0
}

func main() {
	fmt.Println("(a) two trees the neighbour check accepts")
	t3 := mk(2, nil, mk(3, mk(1, nil, nil), nil))
	t7 := mk(50, mk(30, mk(20, nil, nil), mk(40, nil, mk(55, nil, nil))), mk(70, mk(60, nil, nil), nil))
	for _, c := range []struct {
		name string
		t    *node
	}{{"3 nodes", t3}, {"7 nodes", t7}} {
		explain(c.t, nil, nil, 0)
		verdicts(c.name, c.t)
		fmt.Println()
	}

	fmt.Println("(b) every labelled tree on n nodes, against the O(n^2) definition")
	fmt.Println("  (\"wrong\" = the verdict differs from the definition's)")
	fmt.Printf("  %s %8s %9s %7s %14s %12s %13s %15s %16s\n", "n", "shapes", "trees", "valid", "naive accepts", "naive wrong", "bounds wrong", "in-order wrong", "streaming wrong")
	onlyFalseAccepts, validIsShapes, shapesIsCatalan := true, true, true
	for n := 1; n <= 7; n++ {
		var trees, valid, naiveOK, naiveBad, boundsBad, orderBad, streamBad int
		sh := shapes(n)
		p := make([]int, n)
		for _, t := range sh {
			for i := range p {
				p[i] = i + 1
			}
			permute(p, 0, func() {
				i := 0
				label(t, p, &i)
				truth := definition(t)
				trees++
				var prev *node
				nv := naive(t)
				for _, c := range []struct {
					verdict bool
					bad     *int
				}{{nv, &naiveBad}, {bounds(t, nil, nil), &boundsBad}, {ascending(t), &orderBad}, {streaming(t, &prev), &streamBad}} {
					if c.verdict != truth {
						*c.bad++
					}
				}
				if truth {
					valid++
				}
				if nv {
					naiveOK++
				}
			})
		}
		fmt.Printf("  %d %8d %9d %7d %14d %12d %13d %15d %16d\n", n, len(sh), trees, valid, naiveOK, naiveBad, boundsBad, orderBad, streamBad)
		onlyFalseAccepts = onlyFalseAccepts && naiveOK-valid == naiveBad
		validIsShapes = validIsShapes && valid == len(sh)
		shapesIsCatalan = shapesIsCatalan && len(sh) == catalan(n)
	}
	fmt.Printf("  naive accepts - valid == naive wrong at every n (it only ever wrongly ACCEPTS): %t\n", onlyFalseAccepts)
	fmt.Printf("  valid labellings == shapes at every n (one valid labelling per shape): %t\n", validIsShapes)
	fmt.Printf("  shapes == Catalan(n) at every n: %t\n\n", shapesIsCatalan)

	fmt.Println("(c) sentinel bounds: MinInt and MaxInt are legal keys (last row: a duplicate, invalid by this policy)")
	for _, c := range []struct {
		name string
		t    *node
	}{
		{"one node MinInt", mk(math.MinInt, nil, nil)},
		{"one node MaxInt", mk(math.MaxInt, nil, nil)},
		{"0 with MinInt, MaxInt", mk(0, mk(math.MinInt, nil, nil), mk(math.MaxInt, nil, nil))},
		{"5 with left child 5", mk(5, mk(5, nil, nil), nil)},
	} {
		sen, truth := sentinel(c.t, math.MinInt, math.MaxInt), definition(c.t)
		fmt.Printf("  %-24s sentinel=%-5t bounds=%-5t definition=%-5t sentinel right: %t\n", c.name, sen, bounds(c.t, nil, nil), truth, sen == truth)
	}
	fmt.Println()

	fmt.Println("(e) NaN keys (Go accepts float keys; the comparison decides what a tree means)")
	keys := []float64{5, math.NaN(), 3, math.NaN()}
	for _, c := range []struct {
		name string
		cmp  func(a, b float64) int
	}{{"cmp.Compare", cmp.Compare[float64]}, {"<, >, else found", operators}} {
		var root *fnode
		added := make([]bool, len(keys))
		for i, k := range keys {
			added[i] = insertF(&root, k, c.cmp)
		}
		var ks []float64
		keysF(root, &ks)
		stored := false
		for _, k := range ks {
			stored = stored || math.IsNaN(k)
		}
		found := false
		for l := root; l != nil && !found; {
			switch d := c.cmp(math.NaN(), l.key); {
			case d < 0:
				l = l.left
			case d > 0:
				l = l.right
			default:
				found = true
			}
		}
		fmt.Printf("  %-17s inserted 5,NaN,3,NaN -> %v  in-order=%v  stores a NaN: %t  search(NaN): %t\n", c.name, added, ks, stored, found)
	}
}
```

**Output:**

```
(a) two trees the neighbour check accepts
      3          must lie in (2, +inf)    ok
          1      must lie in (2, 3)       VIOLATION
  2              must lie in (-inf, +inf) ok
  3 nodes                    naive=true  bounds=false in-order=false streaming=false definition=false

      70         must lie in (50, +inf)   ok
          60     must lie in (50, 70)     ok
  50             must lie in (-inf, +inf) ok
              55 must lie in (40, 50)     VIOLATION
          40     must lie in (30, 50)     ok
      30         must lie in (-inf, 50)   ok
          20     must lie in (-inf, 30)   ok
  7 nodes                    naive=true  bounds=false in-order=false streaming=false definition=false

(b) every labelled tree on n nodes, against the O(n^2) definition
  ("wrong" = the verdict differs from the definition's)
  n   shapes     trees   valid  naive accepts  naive wrong  bounds wrong  in-order wrong  streaming wrong
  1        1         1       1              1            0             0               0                0
  2        2         4       2              2            0             0               0                0
  3        5        30       5              7            2             0               0                0
  4       14       336      14             36           22             0               0                0
  5       42      5040      42            246          204             0               0                0
  6      132     95040     132           2104         1972             0               0                0
  7      429   2162160     429          21652        21223             0               0                0
  naive accepts - valid == naive wrong at every n (it only ever wrongly ACCEPTS): true
  valid labellings == shapes at every n (one valid labelling per shape): true
  shapes == Catalan(n) at every n: true

(c) sentinel bounds: MinInt and MaxInt are legal keys (last row: a duplicate, invalid by this policy)
  one node MinInt          sentinel=false bounds=true  definition=true  sentinel right: false
  one node MaxInt          sentinel=false bounds=true  definition=true  sentinel right: false
  0 with MinInt, MaxInt    sentinel=false bounds=true  definition=true  sentinel right: false
  5 with left child 5      sentinel=false bounds=false definition=false sentinel right: true

(e) NaN keys (Go accepts float keys; the comparison decides what a tree means)
  cmp.Compare       inserted 5,NaN,3,NaN -> [true true true false]  in-order=[NaN 3 5]  stores a NaN: true  search(NaN): true
  <, >, else found  inserted 5,NaN,3,NaN -> [true false true false]  in-order=[3 5]  stores a NaN: false  search(NaN): true
```

**Rust:**

```rust
use std::cmp::Ordering;

// A node holds one key; None is the empty tree. (Trees here have at most 7
// nodes, so the default recursive drop is fine; examples 8 and 9 hold 100,000
// nodes and add an iterative Drop.)
#[derive(Clone)]
struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

fn mk(key: i64, left: Link, right: Link) -> Link {
    Some(Box::new(Node { key, left, right }))
}

// 1. The check everyone writes first: each node against its two CHILDREN.
fn naive(n: &Link) -> bool {
    let Some(n) = n else { return true };
    n.left.as_ref().is_none_or(|l| l.key < n.key)
        && n.right.as_ref().is_none_or(|r| r.key > n.key)
        && naive(&n.left)
        && naive(&n.right)
}

// 2. The bounds check: every key must lie in the open interval (lo, hi) its
// ancestors hand down. Going left tightens hi, going right tightens lo. None
// means "unbounded", so no real key can be mistaken for a sentinel.
fn bounds(n: &Link, lo: Option<i64>, hi: Option<i64>) -> bool {
    let Some(n) = n else { return true };
    lo.is_none_or(|lo| n.key > lo)
        && hi.is_none_or(|hi| n.key < hi)
        && bounds(&n.left, lo, Some(n.key))
        && bounds(&n.right, Some(n.key), hi)
}

// 3. In-order must be strictly ascending (collect, then scan).
fn in_order(n: &Link, out: &mut Vec<i64>) {
    if let Some(n) = n {
        in_order(&n.left, out);
        out.push(n.key);
        in_order(&n.right, out);
    }
}

fn ascending(root: &Link) -> bool {
    let mut xs = Vec::new();
    in_order(root, &mut xs);
    xs.windows(2).all(|w| w[0] < w[1])
}

// 4. The same test with no vector: remember only the previously visited key.
fn streaming(n: &Link, prev: &mut Option<i64>) -> bool {
    let Some(n) = n else { return true };
    if !streaming(&n.left, prev) || prev.is_some_and(|p| p >= n.key) {
        return false;
    }
    *prev = Some(n.key);
    streaming(&n.right, prev)
}

// 5. The definition, taken literally: every key in the left subtree is
// smaller and every key in the right subtree larger. O(n^2): the ground truth.
fn all(n: &Link, ok: &impl Fn(i64) -> bool) -> bool {
    n.as_ref()
        .is_none_or(|n| ok(n.key) && all(&n.left, ok) && all(&n.right, ok))
}

fn definition(n: &Link) -> bool {
    let Some(node) = n else { return true };
    all(&node.left, &|k| k < node.key)
        && all(&node.right, &|k| k > node.key)
        && definition(&node.left)
        && definition(&node.right)
}

// The sentinel trap: "unbounded" spelled as the extreme ints.
fn sentinel(n: &Link, lo: i64, hi: i64) -> bool {
    let Some(n) = n else { return true };
    n.key > lo && n.key < hi && sentinel(&n.left, lo, n.key) && sentinel(&n.right, n.key, hi)
}

fn interval(lo: Option<i64>, hi: Option<i64>) -> String {
    let l = lo.map_or("-inf".to_string(), |v| v.to_string());
    let h = hi.map_or("+inf".to_string(), |v| v.to_string());
    format!("({l}, {h})")
}

// explain prints the tree sideways with the interval each key must lie in.
fn explain(n: &Link, lo: Option<i64>, hi: Option<i64>, depth: usize) {
    let Some(n) = n else { return };
    explain(&n.right, Some(n.key), hi, depth + 1);
    let bad = lo.is_some_and(|l| n.key <= l) || hi.is_some_and(|h| n.key >= h);
    let label = format!("{}{}", "    ".repeat(depth), n.key);
    let verdict = if bad { "VIOLATION" } else { "ok" };
    println!(
        "  {label:<14} must lie in {:<12} {verdict}",
        interval(lo, hi)
    );
    explain(&n.left, lo, Some(n.key), depth + 1);
}

fn verdicts(name: &str, t: &Link) {
    println!(
        "  {name:<26} naive={:<5} bounds={:<5} in-order={:<5} streaming={:<5} definition={}",
        naive(t),
        bounds(t, None, None),
        ascending(t),
        streaming(t, &mut None),
        definition(t)
    );
}

// ---- every labelled tree on n nodes ----

// shapes returns every binary-tree shape with n nodes (Catalan(n) of them).
fn shapes(n: usize) -> Vec<Link> {
    if n == 0 {
        return vec![None];
    }
    let mut out = Vec::new();
    for l in 0..n {
        for ls in shapes(l) {
            for rs in shapes(n - 1 - l) {
                out.push(mk(0, ls.clone(), rs));
            }
        }
    }
    out
}

// label writes p onto the tree in pre-order.
fn label(n: &mut Link, p: &[i64], i: &mut usize) {
    if let Some(n) = n {
        n.key = p[*i];
        *i += 1;
        label(&mut n.left, p, i);
        label(&mut n.right, p, i);
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

fn catalan(n: usize) -> usize {
    (0..n).fold(1, |c, i| c * 2 * (2 * i + 1) / (i + 2))
}

#[derive(Default)]
struct Tally {
    trees: usize,
    valid: usize,
    naive_ok: usize,
    naive_bad: usize,
    bounds_bad: usize,
    order_bad: usize,
    stream_bad: usize,
}

// ---- NaN: f64 is not Ord, so the comparison is something you must supply ----

// a three-way comparison of two keys
type Cmp = fn(f64, f64) -> Ordering;

struct FNode {
    key: f64,
    left: Option<Box<FNode>>,
    right: Option<Box<FNode>>,
}

// insert_f reports whether it added a key; c is the three-way comparison.
// A `&mut Option<Box<_>>` cursor compiles for insert (delete is example 12).
fn insert_f(root: &mut Option<Box<FNode>>, key: f64, c: Cmp) -> bool {
    let mut cur = root;
    while let Some(n) = cur {
        match c(key, n.key) {
            Ordering::Less => cur = &mut n.left,
            Ordering::Greater => cur = &mut n.right,
            Ordering::Equal => return false,
        }
    }
    *cur = Some(Box::new(FNode {
        key,
        left: None,
        right: None,
    }));
    true
}

fn keys_f(n: &Option<Box<FNode>>, out: &mut Vec<f64>) {
    if let Some(n) = n {
        keys_f(&n.left, out);
        out.push(n.key);
        keys_f(&n.right, out);
    }
}

// total is IEEE 754 totalOrder: every f64, NaN included, has a place. (The
// method takes references, so it needs this wrapper to match the fn type.)
fn total(a: f64, b: f64) -> Ordering {
    a.total_cmp(&b)
}

// operators is the descent written with < and >, anything else "found".
fn operators(a: f64, b: f64) -> Ordering {
    a.partial_cmp(&b).unwrap_or(Ordering::Equal)
}

// Go prints a slice as [true false]; Rust's {:?} would add commas.
fn go_list<T: ToString>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    println!("(a) two trees the neighbour check accepts");
    let t3 = mk(2, None, mk(3, mk(1, None, None), None));
    let t7 = mk(
        50,
        mk(30, mk(20, None, None), mk(40, None, mk(55, None, None))),
        mk(70, mk(60, None, None), None),
    );
    for (name, t) in [("3 nodes", &t3), ("7 nodes", &t7)] {
        explain(t, None, None, 0);
        verdicts(name, t);
        println!();
    }

    println!("(b) every labelled tree on n nodes, against the O(n^2) definition");
    println!("  (\"wrong\" = the verdict differs from the definition's)");
    println!(
        "  n {:>8} {:>9} {:>7} {:>14} {:>12} {:>13} {:>15} {:>16}",
        "shapes",
        "trees",
        "valid",
        "naive accepts",
        "naive wrong",
        "bounds wrong",
        "in-order wrong",
        "streaming wrong"
    );
    let (mut only_false_accepts, mut valid_is_shapes, mut shapes_is_catalan) = (true, true, true);
    for n in 1..=7 {
        let mut s = Tally::default();
        let sh = shapes(n);
        let mut p: Vec<i64> = (1..=n as i64).collect();
        for mut t in sh.iter().cloned() {
            permute(&mut p, 0, &mut |p| {
                label(&mut t, p, &mut 0);
                let truth = definition(&t);
                let nv = naive(&t);
                s.trees += 1;
                s.valid += usize::from(truth);
                s.naive_ok += usize::from(nv);
                s.naive_bad += usize::from(nv != truth);
                s.bounds_bad += usize::from(bounds(&t, None, None) != truth);
                s.order_bad += usize::from(ascending(&t) != truth);
                s.stream_bad += usize::from(streaming(&t, &mut None) != truth);
            });
        }
        println!(
            "  {n} {:>8} {:>9} {:>7} {:>14} {:>12} {:>13} {:>15} {:>16}",
            sh.len(),
            s.trees,
            s.valid,
            s.naive_ok,
            s.naive_bad,
            s.bounds_bad,
            s.order_bad,
            s.stream_bad
        );
        only_false_accepts &= s.naive_ok - s.valid == s.naive_bad;
        valid_is_shapes &= s.valid == sh.len();
        shapes_is_catalan &= sh.len() == catalan(n);
    }
    println!(
        "  naive accepts - valid == naive wrong at every n (it only ever wrongly ACCEPTS): {only_false_accepts}"
    );
    println!(
        "  valid labellings == shapes at every n (one valid labelling per shape): {valid_is_shapes}"
    );
    println!("  shapes == Catalan(n) at every n: {shapes_is_catalan}\n");

    println!(
        "(c) sentinel bounds: MinInt and MaxInt are legal keys (last row: a duplicate, invalid by this policy)"
    );
    for (name, t) in [
        ("one node MinInt", mk(i64::MIN, None, None)),
        ("one node MaxInt", mk(i64::MAX, None, None)),
        (
            "0 with MinInt, MaxInt",
            mk(0, mk(i64::MIN, None, None), mk(i64::MAX, None, None)),
        ),
        ("5 with left child 5", mk(5, mk(5, None, None), None)),
    ] {
        let (sen, truth) = (sentinel(&t, i64::MIN, i64::MAX), definition(&t));
        println!(
            "  {name:<24} sentinel={sen:<5} bounds={:<5} definition={truth:<5} sentinel right: {}",
            bounds(&t, None, None),
            sen == truth
        );
    }
    println!();

    println!("(e) NaN keys (f64 is not Ord, so no BTreeSet<f64>; here the comparison is supplied)");
    let keys = [5.0, f64::NAN, 3.0, f64::NAN];
    let comparisons: [(&str, Cmp); 2] = [("total_cmp", total), ("<, >, else found", operators)];
    for (name, c) in comparisons {
        let mut root = None;
        let added: Vec<bool> = keys.iter().map(|&k| insert_f(&mut root, k, c)).collect();
        let mut ks = Vec::new();
        keys_f(&root, &mut ks);
        // search(NaN): the same descent, stopping on Equal
        let mut found = false;
        let mut cur = root.as_deref();
        while let Some(n) = cur {
            cur = match c(f64::NAN, n.key) {
                Ordering::Less => n.left.as_deref(),
                Ordering::Greater => n.right.as_deref(),
                Ordering::Equal => {
                    found = true;
                    None
                }
            };
        }
        println!(
            "  {name:<17} inserted 5,NaN,3,NaN -> {}  in-order={}  stores a NaN: {}  search(NaN): {found}",
            go_list(&added),
            go_list(&ks),
            ks.iter().any(|k| k.is_nan())
        );
    }
}
```

**Rust output:**

```
(a) two trees the neighbour check accepts
      3          must lie in (2, +inf)    ok
          1      must lie in (2, 3)       VIOLATION
  2              must lie in (-inf, +inf) ok
  3 nodes                    naive=true  bounds=false in-order=false streaming=false definition=false

      70         must lie in (50, +inf)   ok
          60     must lie in (50, 70)     ok
  50             must lie in (-inf, +inf) ok
              55 must lie in (40, 50)     VIOLATION
          40     must lie in (30, 50)     ok
      30         must lie in (-inf, 50)   ok
          20     must lie in (-inf, 30)   ok
  7 nodes                    naive=true  bounds=false in-order=false streaming=false definition=false

(b) every labelled tree on n nodes, against the O(n^2) definition
  ("wrong" = the verdict differs from the definition's)
  n   shapes     trees   valid  naive accepts  naive wrong  bounds wrong  in-order wrong  streaming wrong
  1        1         1       1              1            0             0               0                0
  2        2         4       2              2            0             0               0                0
  3        5        30       5              7            2             0               0                0
  4       14       336      14             36           22             0               0                0
  5       42      5040      42            246          204             0               0                0
  6      132     95040     132           2104         1972             0               0                0
  7      429   2162160     429          21652        21223             0               0                0
  naive accepts - valid == naive wrong at every n (it only ever wrongly ACCEPTS): true
  valid labellings == shapes at every n (one valid labelling per shape): true
  shapes == Catalan(n) at every n: true

(c) sentinel bounds: MinInt and MaxInt are legal keys (last row: a duplicate, invalid by this policy)
  one node MinInt          sentinel=false bounds=true  definition=true  sentinel right: false
  one node MaxInt          sentinel=false bounds=true  definition=true  sentinel right: false
  0 with MinInt, MaxInt    sentinel=false bounds=true  definition=true  sentinel right: false
  5 with left child 5      sentinel=false bounds=false definition=false sentinel right: true

(e) NaN keys (f64 is not Ord, so no BTreeSet<f64>; here the comparison is supplied)
  total_cmp         inserted 5,NaN,3,NaN -> [true true true false]  in-order=[3 5 NaN]  stores a NaN: true  search(NaN): true
  <, >, else found  inserted 5,NaN,3,NaN -> [true false true false]  in-order=[3 5]  stores a NaN: false  search(NaN): true
```

> **Go vs Rust:** Output is identical to Go through the end of (c), including all seven rows of the exhaustive table (21,223 wrong accepts at n = 7). Rust spells "unbounded" as `Option<i64>` with `is_none_or`, so no key value can be a sentinel, and the streaming validator keeps the previous `Option<i64>` key instead of a node pointer. Only (e) differs, and that is the finding: `BTreeSet<f64>` does not compile (E0277, "the trait bound `f64: Ord` is not satisfied", verified on rustc 1.98.1), so the twin supplies the comparison itself; `f64::total_cmp` puts `f64::NAN` (sign bit clear in this build) after every number (`[3 5 NaN]`), where Go's `cmp.Compare` puts NaN first (`[NaN 3 5]`; a scratch sort showed `-f64::NAN` sorting before -inf under `total_cmp`), and the `<`/`>`/else-found descent loses the NaN in both languages.

**Complexity:** the bounds validator is Θ(n) time and Θ(h) stack; the neighbour check is also Θ(n) and wrong, and the exhaustive run shows how wrong: it wrongly accepts 2 of 30 labelled trees at n = 3 (the smallest failure) and 21,223 of 2,162,160 at n = 7 (21,652 accepted, 429 valid), while the bounds, in-order and streaming validators are wrong on none. There is exactly one valid labelling per shape, so the number of valid trees is Catalan(n). Sentinel bounds reject a valid tree holding `MinInt` or `MaxInt`; the `nil` version accepts both. A `NaN` key is only coherent under a three-way comparison that orders it (`cmp.Compare` puts it first).

---

## 8. The k-th smallest, and what a size field buys

`🟡 medium` · *Order statistics*

Without help the k-th smallest key is an in-order walk that stops after k keys, Θ(h + k). Store the **subtree size** in every node and it becomes a descent: compare k with `size(left)` and go left, stop, or go right with k reduced, Θ(h); `Rank(x)` (the keys below x) is the same walk. On a random 100,000-key tree the table puts the two side by side at k = 0, n/4, n/2 and n − 1, counting nodes entered, and the walk's count turns out to be exactly `k + 1` plus the number of left turns on the path. The price is one more invariant, `size = 1 + size(left) + size(right)`, which every insert and delete must repair on the way back up; the program runs 5,000 random updates with a recount after each, then plants the classic way to break it.

**Steps:**

1. Build the tree from 100,000 distinct random keys; write `Kth` (in-order with a stop), `Select` (descent by sizes), `Rank`, and a recounting `check`.
2. Print the table for four values of k and check every rank (`Rank(Select(r)) == r`) and 10,000 probe keys against a sorted slice.
3. Run the 5,000 random inserts and deletes with `check` after every op; then swap in the insert that bumps `size` before it knows the key is new, and read the op number and message where the recount first catches it.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"unsafe"
)

// node holds a key and the number of nodes in its subtree (itself included).
type node struct {
	key, size   int
	left, right *node
}

func size(n *node) int {
	if n == nil {
		return 0
	}
	return n.size
}

// fix recomputes size from the children: the repair every update must make.
func fix(n *node) { n.size = 1 + size(n.left) + size(n.right) }

func height(n *node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.left), height(n.right))
}

// insert returns the new subtree root and whether it added the key. The sizes
// are repaired on the way back up, once we know what happened below.
func insert(n *node, key int) (*node, bool) {
	if n == nil {
		return &node{key: key, size: 1}, true
	}
	var added bool
	switch {
	case key < n.key:
		n.left, added = insert(n.left, key)
	case key > n.key:
		n.right, added = insert(n.right, key)
	default:
		return n, false
	}
	fix(n)
	return n, added
}

// insertBuggy is the planted bug: it bumps size on the way DOWN, before it
// knows whether the key is new, so a duplicate inflates every ancestor.
func insertBuggy(n *node, key int) (*node, bool) {
	if n == nil {
		return &node{key: key, size: 1}, true
	}
	n.size++
	var added bool
	switch {
	case key < n.key:
		n.left, added = insertBuggy(n.left, key)
	case key > n.key:
		n.right, added = insertBuggy(n.right, key)
	default:
		return n, false
	}
	return n, added
}

// remove: the two-child case moves the successor's key up and removes the
// successor from the right subtree.
func remove(n *node, key int) (*node, bool) {
	if n == nil {
		return nil, false
	}
	var found bool
	switch {
	case key < n.key:
		n.left, found = remove(n.left, key)
	case key > n.key:
		n.right, found = remove(n.right, key)
	default:
		if n.left == nil {
			return n.right, true
		}
		if n.right == nil {
			return n.left, true
		}
		m := n.right
		for m.left != nil {
			m = m.left
		}
		n.key = m.key
		n.right, _ = remove(n.right, m.key)
		found = true
	}
	fix(n)
	return n, found
}

// kth walks in order and stops at the k-th key (0-based). entered counts nodes.
func kth(n *node, k, entered *int) (int, bool) {
	if n == nil {
		return 0, false
	}
	*entered++
	if key, ok := kth(n.left, k, entered); ok {
		return key, true
	}
	if *k == 0 {
		return n.key, true
	}
	*k--
	return kth(n.right, k, entered)
}

// selectK uses the sizes: one comparison of k with size(left) per level.
func selectK(n *node, k int, entered *int) (int, bool) {
	for n != nil {
		*entered++
		switch l := size(n.left); {
		case k < l:
			n = n.left
		case k == l:
			return n.key, true
		default:
			k -= l + 1
			n = n.right
		}
	}
	return 0, false
}

// leftTurns counts the ancestors of the k-th key where that descent goes left:
// the in-order walk enters them, but they come after the k-th key.
func leftTurns(n *node, k int) (turns int) {
	for n != nil {
		switch l := size(n.left); {
		case k < l:
			turns++
			n = n.left
		case k == l:
			return
		default:
			k -= l + 1
			n = n.right
		}
	}
	return
}

// rank counts the keys smaller than x; present or not.
func rank(n *node, x int, entered *int) int {
	r := 0
	for n != nil {
		*entered++
		if x <= n.key {
			n = n.left
		} else {
			r += size(n.left) + 1
			n = n.right
		}
	}
	return r
}

// check recounts every size and verifies the (lo, hi) bounds; it returns the
// recounted size and the first problem found ("" if none). It never panics.
func check(n *node, lo, hi *int) (int, string) {
	if n == nil {
		return 0, ""
	}
	if lo != nil && n.key <= *lo || hi != nil && n.key >= *hi {
		return 0, fmt.Sprintf("key %d is outside its bounds", n.key)
	}
	ls, msg := check(n.left, lo, &n.key)
	if msg != "" {
		return 0, msg
	}
	rs, msg := check(n.right, &n.key, hi)
	if msg != "" {
		return 0, msg
	}
	if n.size != 1+ls+rs {
		return 0, fmt.Sprintf("key %d stores size %d but its subtrees hold %d", n.key, n.size, 1+ls+rs)
	}
	return 1 + ls + rs, ""
}

type report struct {
	checkOp, selectOp, mismatches, inserts, added, removed, finalLen int
	msg                                                              string
}

// churn runs 5,000 random inserts and deletes on keys in [0, 40) against a
// sorted slice, checking sizes after every op. Both op streams are identical.
func churn(ins func(*node, int) (*node, bool)) report {
	rng := rand.New(rand.NewPCG(3, 4))
	r := report{checkOp: -1, selectOp: -1}
	var root *node
	var oracle []int
	for op := 1; op <= 5000; op++ {
		insertOp, key := rng.IntN(2) == 0, rng.IntN(40)
		i, present := slices.BinarySearch(oracle, key)
		var did bool
		if insertOp {
			root, did = ins(root, key)
			if !present {
				oracle = slices.Insert(oracle, i, key)
			}
			r.inserts++
			r.added += btoi(did)
			if did == present {
				r.mismatches++
			}
		} else {
			root, did = remove(root, key)
			if present {
				oracle = slices.Delete(oracle, i, i+1)
			}
			r.removed += btoi(did)
			if did != present {
				r.mismatches++
			}
		}
		if _, msg := check(root, nil, nil); msg != "" && r.checkOp < 0 {
			r.checkOp, r.msg = op, msg
		}
		for j, want := range oracle {
			if got, ok := selectK(root, j, new(int)); (!ok || got != want) && r.selectOp < 0 {
				r.selectOp = op
			}
		}
	}
	r.finalLen = len(oracle)
	return r
}

func btoi(b bool) int {
	if b {
		return 1
	}
	return 0
}

func opText(op int) string {
	if op < 0 {
		return "never"
	}
	return fmt.Sprint("op ", op)
}

func main() {
	const n = 100_000
	rng := rand.New(rand.NewPCG(1, 2))
	var root *node
	keys := make([]int, 0, n)
	for len(keys) < n {
		var added bool
		key := rng.IntN(1 << 30)
		if root, added = insert(root, key); added {
			keys = append(keys, key)
		}
	}
	sorted := slices.Sorted(slices.Values(keys))
	h := height(root)
	fmt.Printf("(a) the tree: %d distinct random keys in draw order: height %d, node = %d bytes, of which the size field is %d\n",
		n, h, unsafe.Sizeof(node{}), unsafe.Sizeof(node{}.size))

	fmt.Println("\n(b) k-th smallest: in-order walk with a stop vs descent by sizes (nodes entered)")
	fmt.Printf("%8s %11s %12s %14s %11s %22s  %s\n", "k", "key", "Kth entered", "Select entered", "left turns", "Kth = k + 1 + turns", "both = sorted[k]")
	for _, k := range []int{0, n / 4, n / 2, n - 1} {
		kk, e1, e2 := k, 0, 0
		k1, _ := kth(root, &kk, &e1)
		k2, _ := selectK(root, k, &e2)
		turns := leftTurns(root, k)
		fmt.Printf("%8d %11d %12d %14d %11d %22t  %t\n", k, k1, e1, e2, turns, e1 == k+1+turns, k1 == sorted[k] && k2 == sorted[k])
	}

	fmt.Println("\n(c) consistency over every rank, and 10,000 probes (half random, half near a present key)")
	bad, worst, total := 0, 0, 0
	for r := 0; r < n; r++ {
		e := 0
		key, ok := selectK(root, r, &e)
		if !ok || key != sorted[r] || rank(root, key, new(int)) != r {
			bad++
		}
		worst, total = max(worst, e), total+e
	}
	present, worstRank := 0, 0
	for i := 0; i < 10_000; i++ {
		x := rng.IntN(1 << 30)
		if i%2 == 1 {
			x = sorted[rng.IntN(n)] + rng.IntN(3) - 1
		}
		want, found := slices.BinarySearch(sorted, x)
		e := 0
		if rank(root, x, &e) != want {
			bad++
		}
		present, worstRank = present+btoi(found), max(worstRank, e)
	}
	fmt.Printf("Rank(Select(r)) == r and Select(r) == sorted[r] for all %d ranks, Rank(x) == lower bound for 10,000 probes (%d present): %d mismatches\n", n, present, bad)
	fmt.Printf("Select entered at most %d nodes (mean %.2f), Rank at most %d, height %d\n", worst, float64(total)/n, worstRank, h)

	fmt.Println("\n(d) 5,000 random inserts and deletes on keys in [0, 40), sizes recounted after every op")
	good := churn(insert)
	fmt.Printf("%d inserts (%d added a key), %d deletes (%d removed one), %d keys left; %d result mismatches vs the sorted slice\ncheck failed at: %s; Select vs sorted slice failed at: %s\n",
		good.inserts, good.added, 5000-good.inserts, good.removed, good.finalLen, good.mismatches, opText(good.checkOp), opText(good.selectOp))

	fmt.Println("\n(e) planted bug: size bumped before the duplicate is known")
	bug := churn(insertBuggy)
	fmt.Printf("check fires at %s: %s\nSelect vs sorted slice first disagrees at %s\n", opText(bug.checkOp), bug.msg, opText(bug.selectOp))
}
```

**Output:**

```
(a) the tree: 100000 distinct random keys in draw order: height 38, node = 32 bytes, of which the size field is 8

(b) k-th smallest: in-order walk with a stop vs descent by sizes (nodes entered)
       k         key  Kth entered Select entered  left turns    Kth = k + 1 + turns  both = sorted[k]
       0        3543           11             11          10                   true  true
   25000   269200296        25014             21          13                   true  true
   50000   539634904        50015             21          14                   true  true
   99999  1073736641       100000             11           0                   true  true

(c) consistency over every rank, and 10,000 probes (half random, half near a present key)
Rank(Select(r)) == r and Select(r) == sorted[r] for all 100000 ranks, Rank(x) == lower bound for 10,000 probes (1703 present): 0 mismatches
Select entered at most 38 nodes (mean 20.51), Rank at most 38, height 38

(d) 5,000 random inserts and deletes on keys in [0, 40), sizes recounted after every op
2444 inserts (1277 added a key), 2556 deletes (1256 removed one), 21 keys left; 0 result mismatches vs the sorted slice
check failed at: never; Select vs sorted slice failed at: never

(e) planted bug: size bumped before the duplicate is known
check fires at op 23: key 21 stores size 3 but its subtrees hold 2
Select vs sorted slice first disagrees at op 23
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::BTreeSet;
use std::mem::size_of;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the same
// seeds give the same keys and the same operation stream as the Go program.
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

// Node holds a key and the number of nodes in its subtree (itself included).
struct Node {
    key: i64,
    size: usize,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// The default drop of a Box tree recurses once per level. Random trees are
// ~40 deep, so it would be fine here, but this Drop walks with a heap-allocated
// stack so that no input, however lopsided, can overflow the thread's stack.
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

fn size(n: &Link) -> usize {
    n.as_ref().map_or(0, |n| n.size)
}

// fix recomputes size from the children: the repair every update must make.
fn fix(n: &mut Node) {
    n.size = 1 + size(&n.left) + size(&n.right);
}

fn height(n: &Link) -> usize {
    n.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

// insert returns the new subtree root and sets `added`. The sizes are repaired
// on the way back up, once we know what happened below. `take()` hands the
// subtree to the recursive call and the result is written back.
fn insert(link: Link, key: i64, added: &mut bool) -> Link {
    let Some(mut n) = link else {
        *added = true;
        return Some(Box::new(Node {
            key,
            size: 1,
            left: None,
            right: None,
        }));
    };
    match key.cmp(&n.key) {
        Ordering::Less => n.left = insert(n.left.take(), key, added),
        Ordering::Greater => n.right = insert(n.right.take(), key, added),
        Ordering::Equal => return Some(n),
    }
    fix(&mut n);
    Some(n)
}

// insert_buggy is the planted bug: it bumps size on the way DOWN, before it
// knows whether the key is new, so a duplicate inflates every ancestor.
fn insert_buggy(link: Link, key: i64, added: &mut bool) -> Link {
    let Some(mut n) = link else {
        *added = true;
        return Some(Box::new(Node {
            key,
            size: 1,
            left: None,
            right: None,
        }));
    };
    n.size += 1;
    match key.cmp(&n.key) {
        Ordering::Less => n.left = insert_buggy(n.left.take(), key, added),
        Ordering::Greater => n.right = insert_buggy(n.right.take(), key, added),
        Ordering::Equal => {}
    }
    Some(n)
}

// take_min detaches the leftmost node of a subtree and returns its key.
fn take_min(mut n: Box<Node>) -> (i64, Link) {
    match n.left.take() {
        None => (n.key, n.right.take()),
        Some(l) => {
            let (key, rest) = take_min(l);
            n.left = rest;
            fix(&mut n);
            (key, Some(n))
        }
    }
}

// remove: the two-child case moves the successor's key up and removes the
// successor from the right subtree.
fn remove(link: Link, key: i64, found: &mut bool) -> Link {
    let mut n = link?;
    match key.cmp(&n.key) {
        Ordering::Less => n.left = remove(n.left.take(), key, found),
        Ordering::Greater => n.right = remove(n.right.take(), key, found),
        Ordering::Equal => {
            *found = true;
            match (n.left.take(), n.right.take()) {
                (None, r) => return r,
                (l, None) => return l,
                (l, Some(r)) => {
                    let (min_key, r) = take_min(r);
                    n.key = min_key;
                    n.left = l;
                    n.right = r;
                }
            }
        }
    }
    fix(&mut n);
    Some(n)
}

// kth walks in order and stops at the k-th key (0-based). entered counts nodes.
fn kth(n: &Link, k: &mut usize, entered: &mut usize) -> Option<i64> {
    let n = n.as_ref()?;
    *entered += 1;
    if let Some(key) = kth(&n.left, k, entered) {
        return Some(key);
    }
    if *k == 0 {
        return Some(n.key);
    }
    *k -= 1;
    kth(&n.right, k, entered)
}

// select uses the sizes: one comparison of k with size(left) per level.
fn select(n: &Link, mut k: usize, entered: &mut usize) -> Option<i64> {
    let mut cur = n.as_deref();
    while let Some(n) = cur {
        *entered += 1;
        let l = size(&n.left);
        match k.cmp(&l) {
            Ordering::Less => cur = n.left.as_deref(),
            Ordering::Equal => return Some(n.key),
            Ordering::Greater => {
                k -= l + 1;
                cur = n.right.as_deref();
            }
        }
    }
    None
}

// left_turns counts the ancestors of the k-th key where that descent goes left:
// the in-order walk enters them, but they come after the k-th key.
fn left_turns(n: &Link, mut k: usize) -> usize {
    let (mut turns, mut cur) = (0, n.as_deref());
    while let Some(n) = cur {
        let l = size(&n.left);
        match k.cmp(&l) {
            Ordering::Less => {
                turns += 1;
                cur = n.left.as_deref();
            }
            Ordering::Equal => break,
            Ordering::Greater => {
                k -= l + 1;
                cur = n.right.as_deref();
            }
        }
    }
    turns
}

// rank counts the keys smaller than x; present or not.
fn rank(n: &Link, x: i64, entered: &mut usize) -> usize {
    let (mut r, mut cur) = (0, n.as_deref());
    while let Some(n) = cur {
        *entered += 1;
        if x <= n.key {
            cur = n.left.as_deref();
        } else {
            r += size(&n.left) + 1;
            cur = n.right.as_deref();
        }
    }
    r
}

// check recounts every size and verifies the (lo, hi) bounds; it returns the
// recounted size or the first problem found. It never panics.
fn check(n: &Link, lo: Option<i64>, hi: Option<i64>) -> Result<usize, String> {
    let Some(n) = n else { return Ok(0) };
    if lo.is_some_and(|l| n.key <= l) || hi.is_some_and(|h| n.key >= h) {
        return Err(format!("key {} is outside its bounds", n.key));
    }
    let ls = check(&n.left, lo, Some(n.key))?;
    let rs = check(&n.right, Some(n.key), hi)?;
    if n.size != 1 + ls + rs {
        return Err(format!(
            "key {} stores size {} but its subtrees hold {}",
            n.key,
            n.size,
            1 + ls + rs
        ));
    }
    Ok(1 + ls + rs)
}

#[derive(Default)]
struct Report {
    check_op: Option<usize>,
    select_op: Option<usize>,
    mismatches: usize,
    inserts: usize,
    added: usize,
    removed: usize,
    final_len: usize,
    msg: String,
}

type Insert = fn(Link, i64, &mut bool) -> Link;

// churn runs 5,000 random inserts and deletes on keys in [0, 40) against a
// sorted Vec, checking sizes after every op. Both op streams are identical.
fn churn(ins: Insert) -> Report {
    let mut rng = Pcg::new(3, 4);
    let mut r = Report::default();
    let mut root: Link = None;
    let mut oracle: Vec<i64> = Vec::new();
    for op in 1..=5000 {
        let insert_op = rng.int_n(2) == 0;
        let key = rng.int_n(40) as i64;
        let (i, present) = match oracle.binary_search(&key) {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        let mut did = false;
        if insert_op {
            root = ins(root.take(), key, &mut did);
            if !present {
                oracle.insert(i, key);
            }
            r.inserts += 1;
            r.added += usize::from(did);
            r.mismatches += usize::from(did == present);
        } else {
            root = remove(root.take(), key, &mut did);
            if present {
                oracle.remove(i);
            }
            r.removed += usize::from(did);
            r.mismatches += usize::from(did != present);
        }
        if let Err(msg) = check(&root, None, None)
            && r.check_op.is_none()
        {
            r.check_op = Some(op);
            r.msg = msg;
        }
        if r.select_op.is_none()
            && oracle
                .iter()
                .enumerate()
                .any(|(j, &want)| select(&root, j, &mut 0) != Some(want))
        {
            r.select_op = Some(op);
        }
    }
    r.final_len = oracle.len();
    r
}

fn op_text(op: Option<usize>) -> String {
    op.map_or("never".to_string(), |op| format!("op {op}"))
}

fn main() {
    const N: usize = 100_000;
    let mut rng = Pcg::new(1, 2);
    let mut root: Link = None;
    let mut keys: Vec<i64> = Vec::with_capacity(N);
    while keys.len() < N {
        let mut added = false;
        let key = rng.int_n(1 << 30) as i64;
        root = insert(root.take(), key, &mut added);
        if added {
            keys.push(key);
        }
    }
    let mut sorted = keys.clone();
    sorted.sort_unstable();
    let h = height(&root);
    println!(
        "(a) the tree: {N} distinct random keys in draw order: height {h}, node = {} bytes, of which the size field is {}",
        size_of::<Node>(),
        size_of::<usize>()
    );

    println!("\n(b) k-th smallest: in-order walk with a stop vs descent by sizes (nodes entered)");
    println!(
        "{:>8} {:>11} {:>12} {:>14} {:>11} {:>22}  both = sorted[k]",
        "k", "key", "Kth entered", "Select entered", "left turns", "Kth = k + 1 + turns"
    );
    for k in [0, N / 4, N / 2, N - 1] {
        let (mut kk, mut e1, mut e2) = (k, 0, 0);
        let k1 = kth(&root, &mut kk, &mut e1);
        let k2 = select(&root, k, &mut e2);
        let turns = left_turns(&root, k);
        println!(
            "{k:>8} {:>11} {e1:>12} {e2:>14} {turns:>11} {:>22}  {}",
            k1.unwrap_or(-1),
            e1 == k + 1 + turns,
            k1 == Some(sorted[k]) && k2 == Some(sorted[k])
        );
    }

    println!(
        "\n(c) consistency over every rank, and 10,000 probes (half random, half near a present key)"
    );
    let (mut bad, mut worst, mut total) = (0, 0, 0);
    for (r, &want) in sorted.iter().enumerate() {
        let mut e = 0;
        match select(&root, r, &mut e) {
            Some(key) if key == want && rank(&root, key, &mut 0) == r => {}
            _ => bad += 1,
        }
        worst = worst.max(e);
        total += e;
    }
    let (mut present, mut worst_rank) = (0, 0);
    for i in 0..10_000 {
        let mut x = rng.int_n(1 << 30) as i64;
        if i % 2 == 1 {
            x = sorted[rng.int_n(N as u64) as usize] + rng.int_n(3) as i64 - 1;
        }
        let (want, found) = match sorted.binary_search(&x) {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        let mut e = 0;
        bad += usize::from(rank(&root, x, &mut e) != want);
        present += usize::from(found);
        worst_rank = worst_rank.max(e);
    }
    println!(
        "Rank(Select(r)) == r and Select(r) == sorted[r] for all {N} ranks, Rank(x) == lower bound for 10,000 probes ({present} present): {bad} mismatches"
    );
    println!(
        "Select entered at most {worst} nodes (mean {:.2}), Rank at most {worst_rank}, height {h}",
        total as f64 / N as f64
    );

    println!(
        "\n(d) 5,000 random inserts and deletes on keys in [0, 40), sizes recounted after every op"
    );
    let good = churn(insert);
    println!(
        "{} inserts ({} added a key), {} deletes ({} removed one), {} keys left; {} result mismatches vs the sorted slice",
        good.inserts,
        good.added,
        5000 - good.inserts,
        good.removed,
        good.final_len,
        good.mismatches
    );
    println!(
        "check failed at: {}; Select vs sorted slice failed at: {}",
        op_text(good.check_op),
        op_text(good.select_op)
    );

    println!("\n(e) planted bug: size bumped before the duplicate is known");
    let bug = churn(insert_buggy);
    println!(
        "check fires at {}: {}\nSelect vs sorted slice first disagrees at {}",
        op_text(bug.check_op),
        bug.msg,
        op_text(bug.select_op)
    );

    // Rust's BTreeSet has no rank/select. iter().nth(k) is the default
    // Iterator::nth (the btree Iter overrides next, last, min, max, size_hint,
    // not nth: read in the source), so it makes k + 1 calls to next.
    println!("\n(f) BTreeSet<i64> over the same keys: no select, so iter().nth(k) walks");
    let set: BTreeSet<i64> = keys.iter().copied().collect();
    println!(
        "{:>8} {:>11} {:>12}  nth(k) == Select(k) and calls == k + 1",
        "k", "key", "next() calls"
    );
    for k in [0, N / 4, N / 2, N - 1] {
        let mut calls = 0;
        let got = set.iter().inspect(|_| calls += 1).nth(k).copied();
        println!(
            "{k:>8} {:>11} {calls:>12}  {}",
            got.unwrap_or(-1),
            got == select(&root, k, &mut 0) && calls == k + 1
        );
    }
}
```

**Rust output:**

```
(a) the tree: 100000 distinct random keys in draw order: height 38, node = 32 bytes, of which the size field is 8

(b) k-th smallest: in-order walk with a stop vs descent by sizes (nodes entered)
       k         key  Kth entered Select entered  left turns    Kth = k + 1 + turns  both = sorted[k]
       0        3543           11             11          10                   true  true
   25000   269200296        25014             21          13                   true  true
   50000   539634904        50015             21          14                   true  true
   99999  1073736641       100000             11           0                   true  true

(c) consistency over every rank, and 10,000 probes (half random, half near a present key)
Rank(Select(r)) == r and Select(r) == sorted[r] for all 100000 ranks, Rank(x) == lower bound for 10,000 probes (1703 present): 0 mismatches
Select entered at most 38 nodes (mean 20.51), Rank at most 38, height 38

(d) 5,000 random inserts and deletes on keys in [0, 40), sizes recounted after every op
2444 inserts (1277 added a key), 2556 deletes (1256 removed one), 21 keys left; 0 result mismatches vs the sorted slice
check failed at: never; Select vs sorted slice failed at: never

(e) planted bug: size bumped before the duplicate is known
check fires at op 23: key 21 stores size 3 but its subtrees hold 2
Select vs sorted slice first disagrees at op 23

(f) BTreeSet<i64> over the same keys: no select, so iter().nth(k) walks
       k         key next() calls  nth(k) == Select(k) and calls == k + 1
       0        3543            1  true
   25000   269200296        25001  true
   50000   539634904        50001  true
   99999  1073736641       100000  true
```

> **Go vs Rust:** Output is identical to Go through (e), including the 5,000-op churn and the planted bug (op 23, "key 21 stores size 3 but its subtrees hold 2"). Ownership shapes the updates: `n.left = insert(n.left.take(), key, added)` hands the subtree to the recursive call and writes the result back, and `check` returns `Result<usize, String>` where Go returns `(int, string)`. Part (f) is Rust-only: `BTreeSet` has no rank/select, and `iter().nth(k)` made k + 1 calls to `next` (1, 25,001, 50,001 and 100,000, counted with `inspect`); the btree `Iter` overrides `next`, `last`, `min` and `max` but not `nth` (read in the source, not run), so that cost is the default.

**Complexity:** plain k-th is Θ(h + k): the walk enters exactly k + 1 + (the left turns on the path to the k-th key) nodes, which is 25,014 at k = 25,000 and all 100,000 at k = n − 1, while `Select` entered at most 38 nodes for any of the 100,000 ranks (mean 20.51, height 38) and `Rank` at most 38. The sizes cost one word per node (an 8-byte field in a 32-byte node) and must be repaired on every update: 0 failures over 5,000 random operations, and the duplicate-bump bug is caught at op 23 by the recount (the sorted-slice comparison fires at the same op). Lesson 13's quickselect is the one-shot Θ(n) answer on an unsorted slice.

---

## 9. Range queries: the path plus the answer

`🟡 medium` · *Range queries*

To report the keys in `[lo, hi]`, recurse left only if `lo < key`, emit if the key is in range, recurse right only if `key < hi`. The nodes entered are the m reported ones plus the nodes on the two root-to-boundary paths, so the cost is Θ(h + m) rather than Θ(n); the paths part at the **split node**, the first node on the way down with `lo <= key <= hi`, which is the lowest common ancestor of `lo` and `hi` found without parent pointers. With the sizes of example 8, *counting* a range is `Rank(hi+1) − Rank(lo)`, Θ(h) whatever m is. The program runs 10,000 random queries on a 100,000-key tree, tests the bound `visited − m ≤ 2h` as a hypothesis, removes the two `if`s to see what they buy, checks three independent answers against each other, and ends on the edge cases an oracle must cover (an inverted range, a single key, bounds beyond the keys, an empty tree).

**Steps:**

1. Build the tree from 100,000 distinct random keys; write the pruned walk, the unpruned walk, `split`, `Rank` and a hand-written `lowerBound` that counts its comparisons.
2. Run 10,000 queries: print eight of them with `m`, nodes entered and split depth, then the extremes over all of them; compare `visited − m` with `2h` and the split node with the last node the searches for `lo` and `hi` share.
3. Compare the three answers on every query, rerun 200 queries without the pruning, and read the edge-case table, where the raw count formulas give −999 for an inverted range that holds no keys.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// node holds a key and the number of nodes in its subtree (itself included).
type node struct {
	key, size   int
	left, right *node
}

func size(n *node) int {
	if n == nil {
		return 0
	}
	return n.size
}

func insert(n *node, key int) (*node, bool) {
	if n == nil {
		return &node{key: key, size: 1}, true
	}
	var added bool
	switch {
	case key < n.key:
		n.left, added = insert(n.left, key)
	case key > n.key:
		n.right, added = insert(n.right, key)
	default:
		return n, false
	}
	n.size = 1 + size(n.left) + size(n.right)
	return n, added
}

func height(n *node) int {
	if n == nil {
		return 0
	}
	return 1 + max(height(n.left), height(n.right))
}

// rangeKeys reports the keys in [lo, hi], pruned: go left only if lo < key,
// right only if key < hi. visited counts the nodes entered.
func rangeKeys(n *node, lo, hi int, out *[]int, visited *int) {
	if n == nil {
		return
	}
	*visited++
	if lo < n.key {
		rangeKeys(n.left, lo, hi, out, visited)
	}
	if lo <= n.key && n.key <= hi {
		*out = append(*out, n.key)
	}
	if n.key < hi {
		rangeKeys(n.right, lo, hi, out, visited)
	}
}

// rangeAll is the same walk without the two ifs: it enters every node.
func rangeAll(n *node, lo, hi int, out *[]int, visited *int) {
	if n == nil {
		return
	}
	*visited++
	rangeAll(n.left, lo, hi, out, visited)
	if lo <= n.key && n.key <= hi {
		*out = append(*out, n.key)
	}
	rangeAll(n.right, lo, hi, out, visited)
}

// split finds the first node on the way down with lo <= key <= hi: where the
// paths to lo and to hi part ways. No parent pointers, no ordering tricks.
func split(n *node, lo, hi int) (depth int, ok bool) {
	for ; n != nil; depth++ {
		switch {
		case hi < n.key:
			n = n.left
		case n.key < lo:
			n = n.right
		default:
			return depth, true
		}
	}
	return 0, false
}

// path returns the nodes a search for x passes through.
func path(n *node, x int) []*node {
	var p []*node
	for n != nil {
		p = append(p, n)
		switch {
		case x < n.key:
			n = n.left
		case x > n.key:
			n = n.right
		default:
			return p
		}
	}
	return p
}

// sharedDepth is the depth of the last node the searches for lo and hi share.
func sharedDepth(root *node, lo, hi int) int {
	pl, ph := path(root, lo), path(root, hi)
	d := 0
	for d+1 < min(len(pl), len(ph)) && pl[d+1] == ph[d+1] {
		d++
	}
	return d
}

// rank counts the keys smaller than x, using the sizes.
func rank(n *node, x int, entered *int) int {
	r := 0
	for n != nil {
		*entered++
		if x <= n.key {
			n = n.left
		} else {
			r += size(n.left) + 1
			n = n.right
		}
	}
	return r
}

// lowerBound is binary search for the first index with xs[i] >= x.
func lowerBound(xs []int, x int, cmps *int) int {
	lo, hi := 0, len(xs)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		*cmps++
		if xs[mid] < x {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

func edge(name string, root *node, sorted []int, lo, hi int) {
	var keys []int
	visited, e, c := 0, 0, 0
	rangeKeys(root, lo, hi, &keys, &visited)
	byRank := rank(root, hi+1, &e) - rank(root, lo, &e)
	bySlice := lowerBound(sorted, hi+1, &c) - lowerBound(sorted, lo, &c)
	fmt.Printf("%-26s %11d %11d %7d %8d %10d %8d %14t\n", name, lo, hi, len(keys), visited, byRank, bySlice,
		max(byRank, 0) == len(keys) && max(bySlice, 0) == len(keys))
}

func main() {
	const n = 100_000
	rng := rand.New(rand.NewPCG(1, 2))
	var root *node
	keys := make([]int, 0, n)
	for len(keys) < n {
		var added bool
		key := rng.IntN(1 << 30)
		if root, added = insert(root, key); added {
			keys = append(keys, key)
		}
	}
	sorted := slices.Sorted(slices.Values(keys))
	h := height(root)
	fmt.Printf("%d distinct random keys over [0, 2^30), height %d\n", n, h)

	fmt.Println("\n(a) the first eight of 10,000 random queries [lo, lo + width], width < 2^24")
	fmt.Printf("%11s %11s %6s %8s %10s %12s\n", "lo", "hi", "m", "visited", "visited-m", "split depth")
	const queries = 10_000
	worstExtra, sumM, sumVisited, maxM, minM := 0, 0, 0, 0, n
	worstRank, minCmps, maxCmps, mismatches, splitBad := 0, 1<<30, 0, 0, 0
	type query struct{ lo, hi int }
	qs := make([]query, queries)
	for q := range qs {
		lo := rng.IntN(1 << 30)
		qs[q] = query{lo, lo + rng.IntN(1<<24)}
		var got []int
		visited, e, c := 0, 0, 0
		rangeKeys(root, qs[q].lo, qs[q].hi, &got, &visited)
		m := len(got)
		d, ok := split(root, qs[q].lo, qs[q].hi)
		if ok != (m > 0) || ok && d != sharedDepth(root, qs[q].lo, qs[q].hi) {
			splitBad++
		}
		if q < 8 {
			if ok {
				fmt.Printf("%11d %11d %6d %8d %10d %12d\n", qs[q].lo, qs[q].hi, m, visited, visited-m, d)
			} else {
				fmt.Printf("%11d %11d %6d %8d %10d %12s\n", qs[q].lo, qs[q].hi, m, visited, visited-m, "none")
			}
		}
		worstExtra = max(worstExtra, visited-m)
		sumM, sumVisited, maxM, minM = sumM+m, sumVisited+visited, max(maxM, m), min(minM, m)

		i := lowerBound(sorted, qs[q].lo, &c)
		j := lowerBound(sorted, qs[q].hi+1, &c)
		if !slices.Equal(got, sorted[i:j]) || rank(root, qs[q].hi+1, &e)-rank(root, qs[q].lo, &e) != len(got) {
			mismatches++
		}
		worstRank, minCmps, maxCmps = max(worstRank, e), min(minCmps, c), max(maxCmps, c)
	}

	verdict := "VIOLATED"
	if worstExtra <= 2*h {
		verdict = "OK"
	}
	fmt.Println("\n(b) the pruned walk over all 10,000 queries")
	fmt.Printf("m ranged from %d to %d (mean %.1f); the walk entered %.1f nodes on average\n",
		minM, maxM, float64(sumM)/queries, float64(sumVisited)/queries)
	fmt.Printf("max visited - m = %d, 2 x height = %d: hypothesis visited - m <= 2h %s\n", worstExtra, 2*h, verdict)
	fmt.Printf("split node found exactly when m > 0, and equal to the last node shared by the searches for lo and hi: %d disagreements\n", splitBad)

	fmt.Println("\n(c) the same query without the two ifs, first 200 queries")
	minAll, maxAll, same, prunedSum := n, 0, true, 0
	for _, q := range qs[:200] {
		var a, b []int
		va, vb := 0, 0
		rangeAll(root, q.lo, q.hi, &a, &va)
		rangeKeys(root, q.lo, q.hi, &b, &vb)
		minAll, maxAll, same, prunedSum = min(minAll, va), max(maxAll, va), same && slices.Equal(a, b), prunedSum+vb
	}
	fmt.Printf("unpruned visited %d to %d nodes (n = %d); same keys as the pruned walk: %t; pruned visited %.1f on average\n",
		minAll, maxAll, n, same, float64(prunedSum)/200)

	fmt.Println("\n(d) three answers on all 10,000 queries: pruned walk, Rank(hi+1) - Rank(lo), sorted slice with two lower bounds")
	fmt.Printf("%d mismatches\n", mismatches)
	fmt.Printf("the two Rank calls entered at most %d nodes whatever m was (2h = %d); the two lower bounds took %d to %d comparisons\n",
		worstRank, 2*h, minCmps, maxCmps)

	fmt.Println("\n(e) edge cases (raw = Rank(hi+1) - Rank(lo) and the slice's j - i, before any guard)")
	fmt.Printf("%-26s %11s %11s %7s %8s %10s %8s %14s\n", "case", "lo", "hi", "m", "visited", "rank raw", "slice raw", "guarded agree")
	a, b := sorted[500], sorted[1500]
	gap := a + 1
	for _, found := slices.BinarySearch(sorted, gap); found; _, found = slices.BinarySearch(sorted, gap) {
		gap++
	}
	edge("lo > hi", root, sorted, b, a)
	edge("lo == hi, a present key", root, sorted, a, a)
	edge("lo == hi, an absent key", root, sorted, gap, gap)
	edge("bounds beyond all keys", root, sorted, -1<<20, 1<<31)
	edge("entirely above all keys", root, sorted, 1<<31, 1<<32)
	edge("empty tree", nil, nil, 0, 1<<30)
}
```

**Output:**

```
100000 distinct random keys over [0, 2^30), height 38

(a) the first eight of 10,000 random queries [lo, lo + width], width < 2^24
         lo          hi      m  visited  visited-m  split depth
  255501120   261716992    565      585         20            6
  109758076   113219107    314      333         19            5
  251386222   253592887    222      241         19            8
  521924318   534499810   1203     1236         33            6
  351679941   367682011   1445     1467         22            5
  662557371   669223641    600      626         26            4
  300434986   305108231    441      471         30           13
  644466172   654636726    942      960         18            7

(b) the pruned walk over all 10,000 queries
m ranged from 0 to 1644 (mean 785.0); the walk entered 806.4 nodes on average
max visited - m = 39, 2 x height = 76: hypothesis visited - m <= 2h OK
split node found exactly when m > 0, and equal to the last node shared by the searches for lo and hi: 0 disagreements

(c) the same query without the two ifs, first 200 queries
unpruned visited 100000 to 100000 nodes (n = 100000); same keys as the pruned walk: true; pruned visited 786.6 on average

(d) three answers on all 10,000 queries: pruned walk, Rank(hi+1) - Rank(lo), sorted slice with two lower bounds
0 mismatches
the two Rank calls entered at most 68 nodes whatever m was (2h = 76); the two lower bounds took 32 to 34 comparisons

(e) edge cases (raw = Rank(hi+1) - Rank(lo) and the slice's j - i, before any guard)
case                                lo          hi       m  visited   rank raw slice raw  guarded agree
lo > hi                       16617983     5251599       0        6       -999     -999           true
lo == hi, a present key        5251599     5251599       1       13          1        1           true
lo == hi, an absent key        5251600     5251600       0       14          0        0           true
bounds beyond all keys        -1048576  2147483648  100000   100000     100000   100000           true
entirely above all keys     2147483648  4294967296       0       11          0        0           true
empty tree                           0  1073741824       0        0          0        0           true
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::BTreeSet;
use std::ops::Bound::Excluded;
use std::panic::{AssertUnwindSafe, catch_unwind};

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the same
// seeds give the same keys and the same queries as the Go program.
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

// Node holds a key and the number of nodes in its subtree (itself included).
struct Node {
    key: i64,
    size: usize,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// The default drop of a Box tree recurses once per level. Random trees are
// ~40 deep, so it would be fine here, but this Drop walks with a heap-allocated
// stack so that no input, however lopsided, can overflow the thread's stack.
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

fn size(n: &Link) -> usize {
    n.as_ref().map_or(0, |n| n.size)
}

// insert returns the new subtree root and sets `added`; sizes are repaired on
// the way back up. `take()` hands the subtree to the recursive call.
fn insert(link: Link, key: i64, added: &mut bool) -> Link {
    let Some(mut n) = link else {
        *added = true;
        return Some(Box::new(Node {
            key,
            size: 1,
            left: None,
            right: None,
        }));
    };
    match key.cmp(&n.key) {
        Ordering::Less => n.left = insert(n.left.take(), key, added),
        Ordering::Greater => n.right = insert(n.right.take(), key, added),
        Ordering::Equal => return Some(n),
    }
    n.size = 1 + size(&n.left) + size(&n.right);
    Some(n)
}

fn height(n: &Link) -> usize {
    n.as_ref()
        .map_or(0, |n| 1 + height(&n.left).max(height(&n.right)))
}

// range_keys reports the keys in [lo, hi], pruned: go left only if lo < key,
// right only if key < hi. visited counts the nodes entered.
fn range_keys(n: &Link, lo: i64, hi: i64, out: &mut Vec<i64>, visited: &mut usize) {
    let Some(n) = n else { return };
    *visited += 1;
    if lo < n.key {
        range_keys(&n.left, lo, hi, out, visited);
    }
    if lo <= n.key && n.key <= hi {
        out.push(n.key);
    }
    if n.key < hi {
        range_keys(&n.right, lo, hi, out, visited);
    }
}

// range_all is the same walk without the two ifs: it enters every node.
fn range_all(n: &Link, lo: i64, hi: i64, out: &mut Vec<i64>, visited: &mut usize) {
    let Some(n) = n else { return };
    *visited += 1;
    range_all(&n.left, lo, hi, out, visited);
    if lo <= n.key && n.key <= hi {
        out.push(n.key);
    }
    range_all(&n.right, lo, hi, out, visited);
}

// split finds the first node on the way down with lo <= key <= hi: where the
// paths to lo and to hi part ways. No parent pointers, no ordering tricks.
fn split(n: &Link, lo: i64, hi: i64) -> Option<usize> {
    let mut cur = n.as_deref();
    let mut depth = 0;
    while let Some(n) = cur {
        if hi < n.key {
            cur = n.left.as_deref();
        } else if n.key < lo {
            cur = n.right.as_deref();
        } else {
            return Some(depth);
        }
        depth += 1;
    }
    None
}

// path returns the nodes a search for x passes through: shared borrows only.
fn path(root: &Link, x: i64) -> Vec<&Node> {
    let mut p = Vec::new();
    let mut cur = root.as_deref();
    while let Some(n) = cur {
        p.push(n);
        cur = match x.cmp(&n.key) {
            Ordering::Less => n.left.as_deref(),
            Ordering::Greater => n.right.as_deref(),
            Ordering::Equal => None,
        };
    }
    p
}

// shared_depth is the depth of the last node the searches for lo and hi share.
fn shared_depth(root: &Link, lo: i64, hi: i64) -> usize {
    let (pl, ph) = (path(root, lo), path(root, hi));
    let mut d = 0;
    while d + 1 < pl.len().min(ph.len()) && std::ptr::eq(pl[d + 1], ph[d + 1]) {
        d += 1;
    }
    d
}

// rank counts the keys smaller than x, using the sizes.
fn rank(n: &Link, x: i64, entered: &mut usize) -> i64 {
    let (mut r, mut cur) = (0, n.as_deref());
    while let Some(n) = cur {
        *entered += 1;
        if x <= n.key {
            cur = n.left.as_deref();
        } else {
            r += size(&n.left) as i64 + 1;
            cur = n.right.as_deref();
        }
    }
    r
}

// lower_bound is binary search for the first index with xs[i] >= x.
fn lower_bound(xs: &[i64], x: i64, cmps: &mut usize) -> i64 {
    let (mut lo, mut hi) = (0, xs.len());
    while lo < hi {
        let mid = (lo + hi) / 2;
        *cmps += 1;
        if xs[mid] < x {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo as i64
}

// The guard: BTreeSet::range PANICS on lo > hi when the set is non-empty, so
// every call in this program goes through here.
fn range_count(set: &BTreeSet<i64>, lo: i64, hi: i64) -> usize {
    if lo > hi {
        0
    } else {
        set.range(lo..=hi).count()
    }
}

fn edge(name: &str, root: &Link, sorted: &[i64], lo: i64, hi: i64) {
    let mut keys = Vec::new();
    let (mut visited, mut e, mut c) = (0, 0, 0);
    range_keys(root, lo, hi, &mut keys, &mut visited);
    let by_rank = rank(root, hi + 1, &mut e) - rank(root, lo, &mut e);
    let by_slice = lower_bound(sorted, hi + 1, &mut c) - lower_bound(sorted, lo, &mut c);
    println!(
        "{name:<26} {lo:>11} {hi:>11} {:>7} {visited:>8} {by_rank:>10} {by_slice:>8} {:>14}",
        keys.len(),
        by_rank.max(0) == keys.len() as i64 && by_slice.max(0) == keys.len() as i64
    );
}

// What the unguarded call does: run with --panic. catch_unwind keeps the
// program alive so all three cases print; the real message also goes to stderr.
fn show_panic(label: &str, f: impl FnOnce() -> usize) {
    match catch_unwind(AssertUnwindSafe(f)) {
        Ok(count) => println!("  {label:<34} -> {count} keys, no panic"),
        Err(payload) => {
            let msg = payload
                .downcast_ref::<&str>()
                .map(|s| s.to_string())
                .or_else(|| payload.downcast_ref::<String>().cloned());
            println!(
                "  {label:<34} -> PANIC: {}",
                msg.unwrap_or_else(|| "?".into())
            );
        }
    }
}

fn main() {
    const N: usize = 100_000;
    let mut rng = Pcg::new(1, 2);
    let mut root: Link = None;
    let mut keys: Vec<i64> = Vec::with_capacity(N);
    while keys.len() < N {
        let mut added = false;
        let key = rng.int_n(1 << 30) as i64;
        root = insert(root.take(), key, &mut added);
        if added {
            keys.push(key);
        }
    }
    let mut sorted = keys.clone();
    sorted.sort_unstable();
    let h = height(&root);
    println!("{N} distinct random keys over [0, 2^30), height {h}");

    println!("\n(a) the first eight of 10,000 random queries [lo, lo + width], width < 2^24");
    println!(
        "{:>11} {:>11} {:>6} {:>8} {:>10} {:>12}",
        "lo", "hi", "m", "visited", "visited-m", "split depth"
    );
    const QUERIES: usize = 10_000;
    let (mut worst_extra, mut sum_m, mut sum_visited, mut max_m, mut min_m) = (0, 0, 0, 0, N);
    let (mut worst_rank, mut min_cmps, mut max_cmps) = (0, usize::MAX, 0);
    let (mut mismatches, mut split_bad) = (0, 0);
    let mut qs: Vec<(i64, i64)> = Vec::with_capacity(QUERIES);
    for q in 0..QUERIES {
        let lo = rng.int_n(1 << 30) as i64;
        let hi = lo + rng.int_n(1 << 24) as i64;
        qs.push((lo, hi));
        let mut got = Vec::new();
        let (mut visited, mut e, mut c) = (0, 0, 0);
        range_keys(&root, lo, hi, &mut got, &mut visited);
        let m = got.len();
        let d = split(&root, lo, hi);
        if d.is_some() != (m > 0) || d.is_some_and(|d| d != shared_depth(&root, lo, hi)) {
            split_bad += 1;
        }
        if q < 8 {
            let depth = d.map_or("none".to_string(), |d| d.to_string());
            println!(
                "{lo:>11} {hi:>11} {m:>6} {visited:>8} {:>10} {depth:>12}",
                visited - m
            );
        }
        worst_extra = worst_extra.max(visited - m);
        sum_m += m;
        sum_visited += visited;
        max_m = max_m.max(m);
        min_m = min_m.min(m);

        let i = lower_bound(&sorted, lo, &mut c) as usize;
        let j = lower_bound(&sorted, hi + 1, &mut c) as usize;
        let count = rank(&root, hi + 1, &mut e) - rank(&root, lo, &mut e);
        if got != sorted[i..j] || count != got.len() as i64 {
            mismatches += 1;
        }
        worst_rank = worst_rank.max(e);
        min_cmps = min_cmps.min(c);
        max_cmps = max_cmps.max(c);
    }

    let verdict = if worst_extra <= 2 * h {
        "OK"
    } else {
        "VIOLATED"
    };
    println!("\n(b) the pruned walk over all 10,000 queries");
    println!(
        "m ranged from {min_m} to {max_m} (mean {:.1}); the walk entered {:.1} nodes on average",
        sum_m as f64 / QUERIES as f64,
        sum_visited as f64 / QUERIES as f64
    );
    println!(
        "max visited - m = {worst_extra}, 2 x height = {}: hypothesis visited - m <= 2h {verdict}",
        2 * h
    );
    println!(
        "split node found exactly when m > 0, and equal to the last node shared by the searches for lo and hi: {split_bad} disagreements"
    );

    println!("\n(c) the same query without the two ifs, first 200 queries");
    let (mut min_all, mut max_all, mut same, mut pruned_sum) = (N, 0, true, 0);
    for &(lo, hi) in &qs[..200] {
        let (mut a, mut b) = (Vec::new(), Vec::new());
        let (mut va, mut vb) = (0, 0);
        range_all(&root, lo, hi, &mut a, &mut va);
        range_keys(&root, lo, hi, &mut b, &mut vb);
        min_all = min_all.min(va);
        max_all = max_all.max(va);
        same &= a == b;
        pruned_sum += vb;
    }
    println!(
        "unpruned visited {min_all} to {max_all} nodes (n = {N}); same keys as the pruned walk: {same}; pruned visited {:.1} on average",
        pruned_sum as f64 / 200.0
    );

    println!(
        "\n(d) three answers on all 10,000 queries: pruned walk, Rank(hi+1) - Rank(lo), sorted slice with two lower bounds"
    );
    println!("{mismatches} mismatches");
    println!(
        "the two Rank calls entered at most {worst_rank} nodes whatever m was (2h = {}); the two lower bounds took {min_cmps} to {max_cmps} comparisons",
        2 * h
    );

    println!(
        "\n(e) edge cases (raw = Rank(hi+1) - Rank(lo) and the slice's j - i, before any guard)"
    );
    println!(
        "{:<26} {:>11} {:>11} {:>7} {:>8} {:>10} {:>8} {:>14}",
        "case", "lo", "hi", "m", "visited", "rank raw", "slice raw", "guarded agree"
    );
    let (a, b) = (sorted[500], sorted[1500]);
    let mut gap = a + 1;
    while sorted.binary_search(&gap).is_ok() {
        gap += 1;
    }
    edge("lo > hi", &root, &sorted, b, a);
    edge("lo == hi, a present key", &root, &sorted, a, a);
    edge("lo == hi, an absent key", &root, &sorted, gap, gap);
    edge(
        "bounds beyond all keys",
        &root,
        &sorted,
        -(1 << 20),
        1 << 31,
    );
    edge("entirely above all keys", &root, &sorted, 1 << 31, 1 << 32);
    edge("empty tree", &None, &[], 0, 1 << 30);

    // Rust's std ordered set: BTreeSet::range, behind the guard.
    println!("\n(f) BTreeSet<i64> over the same keys: range(lo..=hi), behind the lo > hi guard");
    let set: BTreeSet<i64> = keys.iter().copied().collect();
    let range_bad = qs
        .iter()
        .filter(|&&(lo, hi)| {
            let i = sorted.partition_point(|&k| k < lo);
            let j = sorted.partition_point(|&k| k <= hi);
            !set.range(lo..=hi).copied().eq(sorted[i..j].iter().copied())
        })
        .count();
    println!("{range_bad} mismatches over {QUERIES} queries against the sorted slice");
    let edges = [
        ("lo > hi", b, a),
        ("lo == hi, a present key", a, a),
        ("lo == hi, an absent key", gap, gap),
        ("bounds beyond all keys", -(1 << 20), 1 << 31),
        ("entirely above all keys", 1 << 31, 1 << 32),
    ];
    for (name, lo, hi) in edges {
        println!("  {name:<26} range_count = {}", range_count(&set, lo, hi));
    }
    println!(
        "unguarded, lo > hi or (Excluded(x), Excluded(x)) on a non-empty set panics: run with --panic"
    );

    if std::env::args().any(|a| a == "--panic") {
        println!("\n(g) --panic: the real BTreeSet::range panics");
        let empty: BTreeSet<i64> = BTreeSet::new();
        show_panic("range(b..=a) with b > a", || set.range(b..=a).count());
        show_panic("(Excluded(a), Excluded(a))", || {
            set.range((Excluded(a), Excluded(a))).count()
        });
        show_panic("range(b..=a) on an EMPTY set", || {
            empty.range(b..=a).count()
        });
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O main.rs -o main && ./main          # add --panic to see the panics BTreeSet::range raises
```

**Rust output:**

```
100000 distinct random keys over [0, 2^30), height 38

(a) the first eight of 10,000 random queries [lo, lo + width], width < 2^24
         lo          hi      m  visited  visited-m  split depth
  255501120   261716992    565      585         20            6
  109758076   113219107    314      333         19            5
  251386222   253592887    222      241         19            8
  521924318   534499810   1203     1236         33            6
  351679941   367682011   1445     1467         22            5
  662557371   669223641    600      626         26            4
  300434986   305108231    441      471         30           13
  644466172   654636726    942      960         18            7

(b) the pruned walk over all 10,000 queries
m ranged from 0 to 1644 (mean 785.0); the walk entered 806.4 nodes on average
max visited - m = 39, 2 x height = 76: hypothesis visited - m <= 2h OK
split node found exactly when m > 0, and equal to the last node shared by the searches for lo and hi: 0 disagreements

(c) the same query without the two ifs, first 200 queries
unpruned visited 100000 to 100000 nodes (n = 100000); same keys as the pruned walk: true; pruned visited 786.6 on average

(d) three answers on all 10,000 queries: pruned walk, Rank(hi+1) - Rank(lo), sorted slice with two lower bounds
0 mismatches
the two Rank calls entered at most 68 nodes whatever m was (2h = 76); the two lower bounds took 32 to 34 comparisons

(e) edge cases (raw = Rank(hi+1) - Rank(lo) and the slice's j - i, before any guard)
case                                lo          hi       m  visited   rank raw slice raw  guarded agree
lo > hi                       16617983     5251599       0        6       -999     -999           true
lo == hi, a present key        5251599     5251599       1       13          1        1           true
lo == hi, an absent key        5251600     5251600       0       14          0        0           true
bounds beyond all keys        -1048576  2147483648  100000   100000     100000   100000           true
entirely above all keys     2147483648  4294967296       0       11          0        0           true
empty tree                           0  1073741824       0        0          0        0           true

(f) BTreeSet<i64> over the same keys: range(lo..=hi), behind the lo > hi guard
0 mismatches over 10000 queries against the sorted slice
  lo > hi                    range_count = 0
  lo == hi, a present key    range_count = 1
  lo == hi, an absent key    range_count = 0
  bounds beyond all keys     range_count = 100000
  entirely above all keys    range_count = 0
unguarded, lo > hi or (Excluded(x), Excluded(x)) on a non-empty set panics: run with --panic
```

> **Go vs Rust:** Output is identical to Go through (e), edge-case table included. `BTreeSet::range(lo..=hi)` is a second oracle in (f), agreeing on all 10,000 queries. Go has no range API at all, and the Rust call needs a guard: on a non-empty set `range(b..=a)` with `b > a` panics with "range start is greater than range end in BTreeSet" and `(Excluded(a), Excluded(a))` with "range start and end are equal and excluded in BTreeSet", while the same inverted range on an empty set returns 0 keys (a separate `--panic` run, caught with `catch_unwind`; not part of the default output).

**Complexity:** the pruned range walk is Θ(h + m): it entered 806.4 nodes on average for a mean m of 785.0, and `visited − m` never exceeded 39 against 2h = 76 (height 38), so the hypothesis held with room to spare; without the two `if`s it entered all 100,000 nodes on each of 200 queries. Counting with sizes is Θ(h) whatever m is: the two `Rank` calls entered at most 68 nodes while m ranged from 0 to 1,644, and the pruned walk, the rank difference and the sorted slice (two lower bounds, 32 to 34 comparisons) agreed on all 10,000 queries. The slice does the same in Θ(log n + m), but only while nothing changes. Raw `Rank(hi+1) − Rank(lo)` is negative for an inverted range, so guard `lo > hi` first.

---

## 10. Building a balanced tree from a sorted slice

`🟡 medium` · *Balanced construction*

If you already have the keys, do not insert them one by one: make the middle element the root and recurse on each half. That is Θ(n) time and **zero key comparisons**, because the build only ever looks at slice *positions*, never at a key's value, and the height comes out as exactly `⌊log₂ n⌋ + 1` with every level but the last full. Inserting the same keys in sorted order gives a list, and inserting them shuffled gives a tree that is balanced only on average, so the program counts comparisons for all three. It then rebuilds an existing tree by flattening it with an in-order walk and building again (the crudest rebalance there is, and the idea behind scapegoat trees), and finally rebuilds a BST from its preorder alone: the invariant supplies the nil markers that a general tree would need a second traversal for.

**Steps:**

1. Write `build` and check it for every n from 1 to 2,000: height, balance of every node, in-order equals the input, the depth of the shallowest nil link, and the number of calls. Print the key comparisons it spent, counted by the same `compare` that `insert` uses.
2. Build the same 4,000 keys three ways, and 1,000,000 keys two ways (sorted-order insertion there is only priced, not run), then flatten the two 4,000-key insertion trees and rebuild them.
3. Rebuild 1,000 random BSTs from their preorder with one pass over the keys, keeping an open interval `(lo, hi)` per link, and compare shapes.

```go
package main

import (
	"cmp"
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"strings"
)

type node struct {
	key         int
	left, right *node
}

// Every key-versus-key comparison in this program goes through compare(), so the
// counter is the evidence: build() below never calls it.
var comparisons int

func compare(a, b int) int {
	comparisons++
	return cmp.Compare(a, b)
}

// build makes the middle element the root and recurses on each half. It looks at
// no key's value, only at slice indexes. calls counts invocations: one per node
// plus one per empty range, so 2n+1 for n keys.
func build(xs []int, calls *int) *node {
	*calls++
	if len(xs) == 0 {
		return nil
	}
	mid := len(xs) / 2
	return &node{xs[mid], build(xs[:mid], calls), build(xs[mid+1:], calls)}
}

// insert is the ordinary BST insert (iterative, `link` points at the nil link).
func insert(root **node, key int) bool {
	link := root
	for *link != nil {
		switch c := compare(key, (*link).key); {
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

// meanHit searches for every key once and returns comparisons per search.
func meanHit(root *node, keys []int) float64 {
	before := comparisons
	for _, k := range keys {
		n := root
		for n != nil {
			c := compare(k, n.key)
			if c == 0 {
				break
			}
			if c < 0 {
				n = n.left
			} else {
				n = n.right
			}
		}
	}
	return float64(comparisons-before) / float64(len(keys))
}

// shape does one level-order pass (no recursion, so a 4,000-deep chain is safe):
// the height in nodes, and the depth of the shallowest nil link.
func shape(root *node) (height, minNil int) {
	for level := []*node{root}; root != nil && len(level) > 0; height++ {
		var next []*node
		for _, n := range level {
			if (n.left == nil || n.right == nil) && minNil == 0 {
				minNil = height + 1
			}
			for _, c := range []*node{n.left, n.right} {
				if c != nil {
					next = append(next, c)
				}
			}
		}
		level = next
	}
	return
}

// balanced: every node's two subtree heights differ by at most 1. Recursive, so
// only used on trees whose height is logarithmic.
func balanced(n *node) (int, bool) {
	if n == nil {
		return 0, true
	}
	hl, okl := balanced(n.left)
	hr, okr := balanced(n.right)
	return 1 + max(hl, hr), okl && okr && max(hl-hr, hr-hl) <= 1
}

// inOrder flattens with an explicit stack.
func inOrder(root *node) []int {
	var out []int
	var stack []*node
	for n := root; n != nil || len(stack) > 0; {
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

func printTree(n *node, depth int) {
	if n == nil {
		return
	}
	printTree(n.right, depth+1)
	fmt.Println(strings.Repeat("    ", depth) + fmt.Sprint(n.key))
	printTree(n.left, depth+1)
}

// failed turns a check into a counter increment: 0 when it held, 1 when not.
func failed(ok bool) int {
	if ok {
		return 0
	}
	return 1
}

func ints(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

func shuffled(rng *rand.Rand, xs []int) []int {
	out := slices.Clone(xs)
	for i := len(out) - 1; i > 0; i-- {
		j := rng.IntN(i + 1)
		out[i], out[j] = out[j], out[i]
	}
	return out
}

// fromPreorder rebuilds a BST from its preorder alone, in one pass over an index.
// The invariant supplies the nil markers: a key outside the open interval
// (lo, hi) cannot belong here, so this link is nil. nil bounds mean "unbounded".
func fromPreorder(pre []int, looks *int) *node {
	i := 0
	var build func(lo, hi *int) *node
	build = func(lo, hi *int) *node {
		if i == len(pre) {
			return nil
		}
		*looks++
		k := pre[i]
		if lo != nil && k <= *lo || hi != nil && k >= *hi {
			return nil
		}
		i++
		n := &node{key: k}
		n.left = build(lo, &n.key)
		n.right = build(&n.key, hi)
		return n
	}
	return build(nil, nil)
}

func preorder(n *node, out *[]int) {
	if n != nil {
		*out = append(*out, n.key)
		preorder(n.left, out)
		preorder(n.right, out)
	}
}

func sameShape(a, b *node) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.key == b.key && sameShape(a.left, b.left) && sameShape(a.right, b.right)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("== (a) middle-element build, every n from 1 to 2000 ==")
	var fails [5]int
	for n := 1; n <= 2000; n++ {
		xs, calls := ints(n), 0
		root := build(xs, &calls)
		h, minNil := shape(root)
		_, bal := balanced(root)
		fails[0] += failed(h == bits.Len(uint(n)))
		fails[1] += failed(bal)
		fails[2] += failed(slices.Equal(inOrder(root), xs))
		fails[3] += failed(minNil >= h-1)
		fails[4] += failed(calls == 2*n+1)
	}
	fmt.Printf("failures over 2000 sizes: height==floor(log2 n)+1 %d, balanced %d, in-order==input %d, nil links at depth >= h-1 %d, calls==2n+1 %d\n",
		fails[0], fails[1], fails[2], fails[3], fails[4])
	fmt.Printf("key comparisons spent by all 2000 builds: %d\n\n", comparisons)

	fmt.Printf("%6s %7s %13s %13s\n", "n", "height", "floor(log2)+1", "min nil depth")
	for _, n := range []int{1, 2, 3, 4, 7, 8, 15, 16, 1000, 1023, 1024, 2000} {
		root := build(ints(n), new(int))
		h, minNil := shape(root)
		fmt.Printf("%6d %7d %13d %13d\n", n, h, bits.Len(uint(n)), minNil)
	}
	for _, n := range []int{7, 10} {
		root := build(ints(n), new(int))
		h, _ := shape(root)
		fmt.Printf("\nn = %d, height %d:\n", n, h)
		printTree(root, 0)
	}

	fmt.Println("\n== (b) three ways to get a tree from the same sorted keys ==")
	fmt.Printf("%-24s %7s %12s %14s\n", "n = 4000", "height", "build cmps", "cmps per hit")
	const n = 4000
	keys := ints(n)
	report := func(name string, root *node, buildCmps int) {
		h, _ := shape(root)
		fmt.Printf("%-24s %7d %12d %14.3f\n", name, h, buildCmps, meanHit(root, keys))
	}
	before := comparisons
	balancedTree := build(keys, new(int))
	report("balanced build", balancedTree, comparisons-before)
	var sortedTree, shuffledTree *node
	before = comparisons
	for _, k := range keys {
		insert(&sortedTree, k)
	}
	report("insert in sorted order", sortedTree, comparisons-before)
	order := shuffled(rng, keys)
	before = comparisons
	for _, k := range order {
		insert(&shuffledTree, k)
	}
	report("insert shuffled", shuffledTree, comparisons-before)

	const big = 1_000_000
	bigKeys := ints(big)
	bigOrder := shuffled(rng, bigKeys)
	before = comparisons
	bigBalanced := build(bigKeys, new(int))
	bigBuild := comparisons - before
	var bigShuffled *node
	before = comparisons
	for _, k := range bigOrder {
		insert(&bigShuffled, k)
	}
	bigInsert := comparisons - before
	hb, _ := shape(bigBalanced)
	hs, _ := shape(bigShuffled)
	mb, ms := meanHit(bigBalanced, bigKeys), meanHit(bigShuffled, bigKeys)
	fmt.Printf("\n%-24s %7s %12s %14s\n", "n = 1,000,000", "height", "build cmps", "cmps per hit")
	fmt.Printf("%-24s %7d %12d %14.3f\n", "balanced build", hb, bigBuild, mb)
	fmt.Printf("%-24s %7d %12d %14.3f\n", "insert shuffled", hs, bigInsert, ms)
	fmt.Printf("insert in sorted order: not run, it would take n(n-1)/2 = %d comparisons\n", big*(big-1)/2)
	fmt.Printf("the shuffled tree is %.2fx the optimal height and %.2fx the optimal cost per hit\n", float64(hs)/float64(hb), ms/mb)

	fmt.Println("\n== (c) flatten and rebuild: the crude rebalance ==")
	fmt.Printf("%-20s %13s %12s %10s %10s %12s\n", "n = 4000", "height before", "height after", "hit before", "hit after", "comparisons")
	ok, calls := true, 0
	for _, c := range []struct {
		name string
		root *node
	}{{"sorted-insert tree", sortedTree}, {"shuffled tree", shuffledTree}} {
		h0, _ := shape(c.root)
		m0 := meanHit(c.root, keys)
		before = comparisons
		calls = 0
		rebuilt := build(inOrder(c.root), &calls) // flatten, then the same build as (a)
		spent := comparisons - before
		h1, _ := shape(rebuilt)
		_, bal := balanced(rebuilt)
		ok = ok && bal && slices.Equal(inOrder(rebuilt), keys)
		fmt.Printf("%-20s %13d %12d %10.3f %10.3f %12d\n", c.name, h0, h1, m0, meanHit(rebuilt, keys), spent)
	}
	fmt.Printf("rebuilt trees valid and balanced: %v; build calls per rebuild %d (2n+1 = %d)\n", ok, calls, 2*n+1)

	fmt.Println("\n== (d) a BST from its preorder alone ==")
	trees, nodes, mismatches, looks, bound := 1000, 0, 0, 0, 0
	for t := 0; t < trees; t++ {
		var root *node
		for want := rng.IntN(80); want > 0; {
			if insert(&root, rng.IntN(1<<30)) {
				want--
			}
		}
		var pre []int
		preorder(root, &pre)
		rebuilt := fromPreorder(pre, &looks)
		nodes += len(pre)
		bound += 2*len(pre) + 1
		if !sameShape(root, rebuilt) {
			mismatches++
		}
	}
	fmt.Printf("%d random trees, %d keys in all: %d shape mismatches\n", trees, nodes, mismatches)
	fmt.Printf("key looks %d <= 2n+1 summed over trees = %d: %v\n", looks, bound, looks <= bound)
}
```

**Output:**

```
== (a) middle-element build, every n from 1 to 2000 ==
failures over 2000 sizes: height==floor(log2 n)+1 0, balanced 0, in-order==input 0, nil links at depth >= h-1 0, calls==2n+1 0
key comparisons spent by all 2000 builds: 0

     n  height floor(log2)+1 min nil depth
     1       1             1             1
     2       2             2             1
     3       2             2             2
     4       3             3             2
     7       3             3             3
     8       4             4             3
    15       4             4             4
    16       5             5             4
  1000      10            10             9
  1023      10            10            10
  1024      11            11            10
  2000      11            11            10

n = 7, height 3:
        6
    5
        4
3
        2
    1
        0

n = 10, height 4:
        9
    8
        7
            6
5
        4
            3
    2
        1
            0

== (b) three ways to get a tree from the same sorted keys ==
n = 4000                  height   build cmps   cmps per hit
balanced build                12            0         10.979
insert in sorted order      4000      7998000       2000.500
insert shuffled               31        53866         14.466

n = 1,000,000             height   build cmps   cmps per hit
balanced build                20            0         18.951
insert shuffled               51     27487502         28.488
insert in sorted order: not run, it would take n(n-1)/2 = 499999500000 comparisons
the shuffled tree is 2.55x the optimal height and 1.50x the optimal cost per hit

== (c) flatten and rebuild: the crude rebalance ==
n = 4000             height before height after hit before  hit after  comparisons
sorted-insert tree            4000           12   2000.500     10.979            0
shuffled tree                   31           12     14.466     10.979            0
rebuilt trees valid and balanced: true; build calls per rebuild 8001 (2n+1 = 8001)

== (d) a BST from its preorder alone ==
1000 random trees, 40178 keys in all: 0 shape mismatches
key looks 78635 <= 2n+1 summed over trees = 81356: true
```

**Rust:**

```rust
use std::cmp::Ordering;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and of its IntN, so seed (1, 2)
// draws exactly the numbers the Go program draws.
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

// Same node as the rest of the lesson. A link is Option<Box<Node>>; None is nil.
struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Go needs no destructor. Rust's default drop glue recurses once per level, which
// would be a 4,000-deep recursion for the sorted-insert chain below; this Drop
// hands the children to a heap-allocated stack instead.
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

// Go counts comparisons in a package-level variable that compare() bumps. Rust has
// no mutable globals, so the counter is a `&mut u64` handed to every function that
// compares keys. build() is NOT given one: it cannot compare, by construction.

/// build makes the middle element the root and recurses on each half. It looks at
/// no key's value, only at slice indexes. `calls` counts invocations: one per node
/// plus one per empty range, so 2n+1 for n keys. The two halves are disjoint
/// shared borrows of the same slice, so no `split_at` is needed.
fn build(xs: &[i64], calls: &mut u64) -> Link {
    *calls += 1;
    if xs.is_empty() {
        return None;
    }
    let mid = xs.len() / 2;
    let left = build(&xs[..mid], calls);
    let right = build(&xs[mid + 1..], calls);
    node(xs[mid], left, right)
}

/// insert is the ordinary BST insert. `cur` is a cursor on the link where the
/// descent currently is; this loop shape compiles for insert (delete is another
/// story: example 12).
fn insert(root: &mut Link, key: i64, cmps: &mut u64) -> bool {
    let mut cur = root;
    while let Some(n) = cur {
        *cmps += 1;
        match key.cmp(&n.key) {
            Ordering::Less => cur = &mut n.left,
            Ordering::Greater => cur = &mut n.right,
            Ordering::Equal => return false,
        }
    }
    *cur = node(key, None, None);
    true
}

/// mean_hit searches for every key once and returns comparisons per search.
fn mean_hit(root: &Link, keys: &[i64], cmps: &mut u64) -> f64 {
    let before = *cmps;
    for &k in keys {
        let mut cur = root.as_deref();
        while let Some(n) = cur {
            *cmps += 1;
            match k.cmp(&n.key) {
                Ordering::Less => cur = n.left.as_deref(),
                Ordering::Greater => cur = n.right.as_deref(),
                Ordering::Equal => break,
            }
        }
    }
    (*cmps - before) as f64 / keys.len() as f64
}

/// shape does one level-order pass (no recursion, so a 4,000-deep chain is safe):
/// the height in nodes, and the depth of the shallowest nil link.
fn shape(root: &Link) -> (usize, usize) {
    let (mut height, mut min_nil) = (0, 0);
    let mut level: Vec<&Node> = root.as_deref().into_iter().collect();
    while !level.is_empty() {
        let mut next = Vec::new();
        for n in &level {
            if (n.left.is_none() || n.right.is_none()) && min_nil == 0 {
                min_nil = height + 1;
            }
            next.extend(n.left.as_deref());
            next.extend(n.right.as_deref());
        }
        level = next;
        height += 1;
    }
    (height, min_nil)
}

/// balanced: every node's two subtree heights differ by at most 1. Recursive, so
/// only used on trees whose height is logarithmic.
fn balanced(n: &Link) -> (usize, bool) {
    let Some(n) = n else { return (0, true) };
    let (hl, okl) = balanced(&n.left);
    let (hr, okr) = balanced(&n.right);
    (1 + hl.max(hr), okl && okr && hl.abs_diff(hr) <= 1)
}

/// in_order flattens with an explicit stack.
fn in_order(root: &Link) -> Vec<i64> {
    let mut out = Vec::new();
    let mut stack: Vec<&Node> = Vec::new();
    let mut cur = root.as_deref();
    while cur.is_some() || !stack.is_empty() {
        while let Some(n) = cur {
            stack.push(n);
            cur = n.left.as_deref();
        }
        let n = stack.pop().unwrap();
        out.push(n.key);
        cur = n.right.as_deref();
    }
    out
}

fn print_tree(n: &Link, depth: usize) {
    if let Some(n) = n {
        print_tree(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.key);
        print_tree(&n.left, depth + 1);
    }
}

fn ints(n: usize) -> Vec<i64> {
    (0..n as i64).collect()
}

/// Fisher-Yates by hand with IntN(i+1), the same draws as the Go program.
fn shuffled(rng: &mut Pcg, xs: &[i64]) -> Vec<i64> {
    let mut out = xs.to_vec();
    for i in (1..out.len()).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        out.swap(i, j);
    }
    out
}

/// from_preorder rebuilds a BST from its preorder alone, in one pass over an index.
/// The invariant supplies the nil markers: a key outside the open interval
/// (lo, hi) cannot belong here, so this link is nil. `None` bounds mean
/// "unbounded", so no real key can be mistaken for a sentinel.
fn from_preorder(
    pre: &[i64],
    i: &mut usize,
    lo: Option<i64>,
    hi: Option<i64>,
    looks: &mut u64,
) -> Link {
    let &k = pre.get(*i)?;
    *looks += 1;
    if lo.is_some_and(|l| k <= l) || hi.is_some_and(|h| k >= h) {
        return None;
    }
    *i += 1;
    let left = from_preorder(pre, i, lo, Some(k), looks);
    let right = from_preorder(pre, i, Some(k), hi, looks);
    node(k, left, right)
}

fn preorder(n: &Link, out: &mut Vec<i64>) {
    if let Some(n) = n {
        out.push(n.key);
        preorder(&n.left, out);
        preorder(&n.right, out);
    }
}

fn same_shape(a: &Link, b: &Link) -> bool {
    match (a, b) {
        (Some(x), Some(y)) => {
            x.key == y.key && same_shape(&x.left, &y.left) && same_shape(&x.right, &y.right)
        }
        (None, None) => true,
        _ => false,
    }
}

/// failed turns a check into a counter increment: 0 when it held, 1 when not.
fn failed(ok: bool) -> u32 {
    u32::from(!ok)
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    let mut cmps = 0u64;

    println!("== (a) middle-element build, every n from 1 to 2000 ==");
    let mut fails = [0u32; 5];
    for n in 1..=2000usize {
        let xs = ints(n);
        let mut calls = 0;
        let root = build(&xs, &mut calls);
        let (h, min_nil) = shape(&root);
        let (_, bal) = balanced(&root);
        // two formulas for the same number: ilog2 and the bit length
        let want = n.ilog2() as usize + 1;
        fails[0] += failed(h == want && h == (usize::BITS - n.leading_zeros()) as usize);
        fails[1] += failed(bal);
        fails[2] += failed(in_order(&root) == xs);
        fails[3] += failed(min_nil + 1 >= h);
        fails[4] += failed(calls == 2 * n as u64 + 1);
    }
    println!(
        "failures over 2000 sizes: height==floor(log2 n)+1 {}, balanced {}, in-order==input {}, nil links at depth >= h-1 {}, calls==2n+1 {}",
        fails[0], fails[1], fails[2], fails[3], fails[4]
    );
    println!("key comparisons spent by all 2000 builds: {cmps}\n");

    println!(
        "{:>6} {:>7} {:>13} {:>13}",
        "n", "height", "floor(log2)+1", "min nil depth"
    );
    for n in [1usize, 2, 3, 4, 7, 8, 15, 16, 1000, 1023, 1024, 2000] {
        let root = build(&ints(n), &mut 0);
        let (h, min_nil) = shape(&root);
        println!(
            "{n:>6} {h:>7} {:>13} {min_nil:>13}",
            usize::BITS - n.leading_zeros()
        );
    }
    for n in [7usize, 10] {
        let root = build(&ints(n), &mut 0);
        let (h, _) = shape(&root);
        println!("\nn = {n}, height {h}:");
        print_tree(&root, 0);
    }

    println!("\n== (b) three ways to get a tree from the same sorted keys ==");
    println!(
        "{:<24} {:>7} {:>12} {:>14}",
        "n = 4000", "height", "build cmps", "cmps per hit"
    );
    const N: usize = 4000;
    let keys = ints(N);
    let report = |name: &str, root: &Link, build_cmps: u64, cmps: &mut u64| {
        let (h, _) = shape(root);
        println!(
            "{name:<24} {h:>7} {build_cmps:>12} {:>14.3}",
            mean_hit(root, &keys, cmps)
        );
    };
    let before = cmps;
    let balanced_tree = build(&keys, &mut 0);
    report("balanced build", &balanced_tree, cmps - before, &mut cmps);
    let mut sorted_tree: Link = None;
    let before = cmps;
    for &k in &keys {
        insert(&mut sorted_tree, k, &mut cmps);
    }
    report(
        "insert in sorted order",
        &sorted_tree,
        cmps - before,
        &mut cmps,
    );
    let order = shuffled(&mut rng, &keys);
    let mut shuffled_tree: Link = None;
    let before = cmps;
    for &k in &order {
        insert(&mut shuffled_tree, k, &mut cmps);
    }
    report("insert shuffled", &shuffled_tree, cmps - before, &mut cmps);

    const BIG: usize = 1_000_000;
    let big_keys = ints(BIG);
    let big_order = shuffled(&mut rng, &big_keys);
    let before = cmps;
    let big_balanced = build(&big_keys, &mut 0);
    let big_build = cmps - before;
    let mut big_shuffled: Link = None;
    let before = cmps;
    for &k in &big_order {
        insert(&mut big_shuffled, k, &mut cmps);
    }
    let big_insert = cmps - before;
    let (hb, _) = shape(&big_balanced);
    let (hs, _) = shape(&big_shuffled);
    let mb = mean_hit(&big_balanced, &big_keys, &mut cmps);
    let ms = mean_hit(&big_shuffled, &big_keys, &mut cmps);
    println!(
        "\n{:<24} {:>7} {:>12} {:>14}",
        "n = 1,000,000", "height", "build cmps", "cmps per hit"
    );
    println!(
        "{:<24} {hb:>7} {big_build:>12} {mb:>14.3}",
        "balanced build"
    );
    println!(
        "{:<24} {hs:>7} {big_insert:>12} {ms:>14.3}",
        "insert shuffled"
    );
    println!(
        "insert in sorted order: not run, it would take n(n-1)/2 = {} comparisons",
        BIG * (BIG - 1) / 2
    );
    println!(
        "the shuffled tree is {:.2}x the optimal height and {:.2}x the optimal cost per hit",
        hs as f64 / hb as f64,
        ms / mb
    );

    println!("\n== (c) flatten and rebuild: the crude rebalance ==");
    println!(
        "{:<20} {:>13} {:>12} {:>10} {:>10} {:>12}",
        "n = 4000", "height before", "height after", "hit before", "hit after", "comparisons"
    );
    let (mut ok, mut calls) = (true, 0);
    for (name, root) in [
        ("sorted-insert tree", &sorted_tree),
        ("shuffled tree", &shuffled_tree),
    ] {
        let (h0, _) = shape(root);
        let m0 = mean_hit(root, &keys, &mut cmps);
        let before = cmps;
        calls = 0;
        let rebuilt = build(&in_order(root), &mut calls); // flatten, then the same build as (a)
        let spent = cmps - before;
        let (h1, _) = shape(&rebuilt);
        let (_, bal) = balanced(&rebuilt);
        ok = ok && bal && in_order(&rebuilt) == keys;
        println!(
            "{name:<20} {h0:>13} {h1:>12} {m0:>10.3} {:>10.3} {spent:>12}",
            mean_hit(&rebuilt, &keys, &mut cmps)
        );
    }
    println!(
        "rebuilt trees valid and balanced: {ok}; build calls per rebuild {calls} (2n+1 = {})",
        2 * N + 1
    );

    println!("\n== (d) a BST from its preorder alone ==");
    let (trees, mut nodes, mut mismatches, mut looks, mut bound) = (1000, 0, 0, 0u64, 0);
    for _ in 0..trees {
        let mut root: Link = None;
        let mut want = rng.int_n(80);
        while want > 0 {
            if insert(&mut root, rng.int_n(1 << 30) as i64, &mut cmps) {
                want -= 1;
            }
        }
        let mut pre = Vec::new();
        preorder(&root, &mut pre);
        let rebuilt = from_preorder(&pre, &mut 0, None, None, &mut looks);
        nodes += pre.len();
        bound += 2 * pre.len() as u64 + 1;
        if !same_shape(&root, &rebuilt) {
            mismatches += 1;
        }
    }
    println!("{trees} random trees, {nodes} keys in all: {mismatches} shape mismatches");
    println!(
        "key looks {looks} <= 2n+1 summed over trees = {bound}: {}",
        looks <= bound
    );
}
```

**Rust output:**

```
== (a) middle-element build, every n from 1 to 2000 ==
failures over 2000 sizes: height==floor(log2 n)+1 0, balanced 0, in-order==input 0, nil links at depth >= h-1 0, calls==2n+1 0
key comparisons spent by all 2000 builds: 0

     n  height floor(log2)+1 min nil depth
     1       1             1             1
     2       2             2             1
     3       2             2             2
     4       3             3             2
     7       3             3             3
     8       4             4             3
    15       4             4             4
    16       5             5             4
  1000      10            10             9
  1023      10            10            10
  1024      11            11            10
  2000      11            11            10

n = 7, height 3:
        6
    5
        4
3
        2
    1
        0

n = 10, height 4:
        9
    8
        7
            6
5
        4
            3
    2
        1
            0

== (b) three ways to get a tree from the same sorted keys ==
n = 4000                  height   build cmps   cmps per hit
balanced build                12            0         10.979
insert in sorted order      4000      7998000       2000.500
insert shuffled               31        53866         14.466

n = 1,000,000             height   build cmps   cmps per hit
balanced build                20            0         18.951
insert shuffled               51     27487502         28.488
insert in sorted order: not run, it would take n(n-1)/2 = 499999500000 comparisons
the shuffled tree is 2.55x the optimal height and 1.50x the optimal cost per hit

== (c) flatten and rebuild: the crude rebalance ==
n = 4000             height before height after hit before  hit after  comparisons
sorted-insert tree            4000           12   2000.500     10.979            0
shuffled tree                   31           12     14.466     10.979            0
rebuilt trees valid and balanced: true; build calls per rebuild 8001 (2n+1 = 8001)

== (d) a BST from its preorder alone ==
1000 random trees, 40178 keys in all: 0 shape mismatches
key looks 78635 <= 2n+1 summed over trees = 81356: true
```

> **Go vs Rust:** Identical output (diffed): `Pcg` ports Go's PCG-DXSM and `IntN`, so seed (1, 2) gives both programs the same shuffles and the same 1,000 random trees. Go counts comparisons in a package-level variable that `compare` bumps; Rust has no mutable globals, so a `&mut u64` is passed to every function that compares, and `build` is not given one, so "0 comparisons" holds by construction there and by measurement in Go. Bounds that may be absent are `Option<i64>` with `is_some_and` where Go passes `*int` and tests `nil`; the two halves in `build` are `&xs[..mid]` and `&xs[mid + 1..]`, disjoint shared borrows, and the height check uses `n.ilog2() + 1` and `usize::BITS - n.leading_zeros()` where Go has `bits.Len`. `impl Drop for Node` frees the 4,000-node chain with an explicit stack instead of recursing.

**Complexity:** building from a sorted slice is Θ(n) time and 0 key comparisons (2n + 1 calls for n keys), and the height is `⌊log₂ n⌋ + 1`: all five checks held for every n from 1 to 2,000. At n = 4,000 inserting the keys in sorted order costs n(n−1)/2 = 7,998,000 comparisons and a height of 4,000; shuffled insertion costs 53,866 comparisons for a height of 31, against 0 comparisons and a height of 12 for the balanced build. At n = 1,000,000 the shuffled tree is 51 levels tall against the optimal 20, and a successful search costs 28.488 comparisons against 18.951. Flatten-and-rebuild is Θ(n) time and space, spends 0 comparisons and takes both 4,000-key trees to height 12. A BST rebuilds from its preorder alone in Θ(n): 0 shape mismatches over 1,000 random trees.

---

## 11. Two-sum on a BST, with O(h) memory

`🟡 medium` · *Two iterators, one pass each*

Run a stack iterator **forward** (ascending) and a second one **backward** (descending) toward each other, like two pointers on a sorted array: if the two keys sum to less than the target, advance the ascending one; if to more, advance the descending one. That is Θ(n) time and Θ(h) memory, with no hash set (lesson 10) and no flattened copy of the keys (lesson 16's two pointers on a slice). Iterators that stop early are [lesson 17, example 15](../17-binary-trees/3-hard.md); what is new here is their **amortized accounting** (every node is pushed once and popped once, even though a single call may push a whole spine) and the *pair* of them. Everything is counted, in pushes, pops, steps and stack entries, not timed.

**Steps:**

1. Write the iterator (the left spine going forward, the right spine going backward) with counters for pushes, pops and the longest stack, and trace it call by call on a 12-key tree.
2. Walk a 100,000-key random tree both ways: total pushes against total pops, the longest stack against the height, and a histogram of how many nodes each `Next` call pushed.
3. Solve two-sum with the pair and check it against an O(n²) pair scan for every target of a 200-key tree, then compare steps and stack entries with a hash set on 100,000 keys.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

type node struct {
	key         int
	left, right *node
}

// insert is the ordinary iterative BST insert; it reports whether the key was new.
func insert(root **node, key int) bool {
	link := root
	for *link != nil {
		switch {
		case key < (*link).key:
			link = &(*link).left
		case key > (*link).key:
			link = &(*link).right
		default:
			return false
		}
	}
	*link = &node{key: key}
	return true
}

// randomTree inserts n distinct keys in [0, limit), in draw order.
func randomTree(rng *rand.Rand, n, limit int) (*node, []int) {
	var root *node
	var keys []int
	for len(keys) < n {
		if k := rng.IntN(limit); insert(&root, k) {
			keys = append(keys, k)
		}
	}
	return root, keys
}

// iter is a stack iterator over a BST. The stack holds the nodes whose turn has
// not come yet along one spine: the left spine going forward (ascending), the
// right spine going backward (descending). Pushes and pops are counted.
type iter struct {
	stack                          []*node
	forward                        bool
	pushes, pops, maxLen, lastPush int
}

func (it *iter) first(n *node) *node { // the child on the side we walk toward first
	if it.forward {
		return n.left
	}
	return n.right
}

func (it *iter) second(n *node) *node {
	if it.forward {
		return n.right
	}
	return n.left
}

func (it *iter) pushSpine(n *node) {
	it.lastPush = 0
	for ; n != nil; n = it.first(n) {
		it.stack = append(it.stack, n)
		it.pushes++
		it.lastPush++
	}
	it.maxLen = max(it.maxLen, len(it.stack))
}

func newIter(root *node, forward bool) *iter {
	it := &iter{forward: forward}
	it.pushSpine(root)
	return it
}

// next pops one node, then pushes the spine of its far child: that child's
// subtree is the part of the tree that comes next.
func (it *iter) next() (int, bool) {
	if len(it.stack) == 0 {
		return 0, false
	}
	n := it.stack[len(it.stack)-1]
	it.stack = it.stack[:len(it.stack)-1]
	it.pops++
	it.pushSpine(it.second(n))
	return n.key, true
}

func height(root *node) int {
	h := 0
	for level := []*node{root}; root != nil && len(level) > 0; h++ {
		var next []*node
		for _, n := range level {
			for _, c := range []*node{n.left, n.right} {
				if c != nil {
					next = append(next, c)
				}
			}
		}
		level = next
	}
	return h
}

func printTree(n *node, depth int) {
	if n != nil {
		printTree(n.right, depth+1)
		fmt.Println(strings.Repeat("    ", depth) + fmt.Sprint(n.key))
		printTree(n.left, depth+1)
	}
}

func stackKeys(it *iter) []int {
	out := []int{}
	for _, n := range it.stack {
		out = append(out, n.key)
	}
	return out
}

// twoSum runs an ascending and a descending iterator toward each other, like two
// pointers on a sorted array. Keys are distinct, so a < b means two different nodes.
func twoSum(root *node, target int) (a, b int, found bool, steps, high int) {
	f, g := newIter(root, true), newIter(root, false)
	a, _ = f.next()
	b, _ = g.next()
	steps, high = 2, len(f.stack)+len(g.stack)
	for a < b {
		var ok bool
		switch s := a + b; {
		case s == target:
			return a, b, true, steps, high
		case s < target:
			a, ok = f.next()
		default:
			b, ok = g.next()
		}
		steps++
		high = max(high, len(f.stack)+len(g.stack))
		if !ok {
			break
		}
	}
	return 0, 0, false, steps, high
}

// twoSumHash is the lesson-10 answer on the same tree: walk the keys once and ask
// a hash set for the partner. It needs one entry per key it has passed.
func twoSumHash(root *node, target int) (found bool, entries int) {
	seen := map[int]struct{}{}
	it := newIter(root, true)
	for k, ok := it.next(); ok; k, ok = it.next() {
		if _, hit := seen[target-k]; hit {
			return true, len(seen)
		}
		seen[k] = struct{}{}
	}
	return false, len(seen)
}

func main() {
	fmt.Println("== (a) the iterator on a 12-key tree: the stack after every Next ==")
	var small *node
	for _, k := range []int{50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25} {
		insert(&small, k)
	}
	printTree(small, 0)
	it := newIter(small, true)
	fmt.Printf("\nnew iterator pushed the left spine: %v\n", stackKeys(it))
	fmt.Printf("%5s %4s %7s  %s\n", "call", "key", "pushes", "stack after, bottom to top")
	for call := 1; ; call++ {
		k, ok := it.next()
		if !ok {
			break
		}
		fmt.Printf("%5d %4d %7d  %v\n", call, k, it.lastPush, stackKeys(it))
	}
	fmt.Printf("12 keys: %d pushes and %d pops in all\n", it.pushes, it.pops)
	var down []int
	g := newIter(small, false)
	for k, ok := g.next(); ok; k, ok = g.next() {
		down = append(down, k)
	}
	fmt.Println("descending iterator:", down)

	fmt.Println("\n== (b) a full walk of a random tree, n = 100,000 ==")
	rng := rand.New(rand.NewPCG(1, 2))
	const n = 100_000
	root, keys := randomTree(rng, n, 1<<30)
	sorted := slices.Sorted(slices.Values(keys))
	h := height(root)
	fmt.Printf("height %d\n", h)
	for _, forward := range []bool{true, false} {
		it := newIter(root, forward)
		setup := it.pushes
		var walk []int
		hist := map[int]int{}
		maxPush := 0
		for k, ok := it.next(); ok; k, ok = it.next() {
			walk = append(walk, k)
			hist[it.lastPush]++
			maxPush = max(maxPush, it.lastPush)
		}
		if !forward {
			slices.Reverse(walk)
		}
		name := "descending"
		if forward {
			name = "ascending"
		}
		fmt.Printf("%s: %d keys, equal to the sorted keys (reversed if descending): %v\n", name, len(walk), slices.Equal(walk, sorted))
		fmt.Printf("  pushes %d (%d at setup), pops %d, pushes == pops == n: %v\n", it.pushes, setup, it.pops, it.pushes == n && it.pops == n)
		fmt.Printf("  max stack length %d (height %d); most pushes in one call %d\n", it.maxLen, h, maxPush)
		if forward {
			fmt.Printf("  %6s %8s %7s\n", "pushes", "calls", "share")
			for k := 0; k <= maxPush; k++ {
				if hist[k] > 0 {
					fmt.Printf("  %6d %8d %6.2f%%\n", k, hist[k], 100*float64(hist[k])/n)
				}
			}
			fmt.Printf("  mean pushes per call %.4f\n", float64(it.pushes-setup)/n)
		}
	}

	fmt.Println("\n== (c) two-sum against brute force: 200 distinct keys in [0, 1000), every target ==")
	root, keys = randomTree(rng, 200, 1000)
	slices.Sort(keys)
	h = height(root)
	maxKey := keys[len(keys)-1]
	var foundN, missN, missFull, bad, maxSteps, maxHigh, totalSteps int
	for target := -5; target <= 2*maxKey+5; target++ {
		want := false
		for i, x := range keys {
			for _, y := range keys[i+1:] {
				want = want || x+y == target
			}
		}
		a, b, got, steps, high := twoSum(root, target)
		_, okA := slices.BinarySearch(keys, a)
		_, okB := slices.BinarySearch(keys, b)
		if got != want || got && (a+b != target || a >= b || !okA || !okB) {
			bad++
		}
		if got {
			foundN++
		} else {
			missN++
			if steps == len(keys)+1 {
				missFull++
			}
		}
		totalSteps += steps
		maxSteps = max(maxSteps, steps)
		maxHigh = max(maxHigh, high)
	}
	targets := foundN + missN
	fmt.Printf("%d targets: %d found, %d not found, %d mismatches\n", targets, foundN, missN, bad)
	fmt.Printf("steps (Next calls on both iterators): mean %.1f, max %d, with n = %d\n", float64(totalSteps)/float64(targets), maxSteps, len(keys))
	fmt.Printf("targets that were not found and cost exactly n + 1 = %d steps: %d of %d\n", len(keys)+1, missFull, missN)
	fmt.Printf("combined stack high-water %d <= 2 * height = %d: %v\n", maxHigh, 2*h, maxHigh <= 2*h)

	fmt.Println("\n== (d) memory as counts: iterators against a hash set, n = 100,000 ==")
	root, keys = randomTree(rng, n, 1<<30)
	slices.Sort(keys)
	h = height(root)
	foundN, missN, bad, maxHigh, maxSteps = 0, 0, 0, 0, 0
	maxEntries, missEntries := 0, 0
	for t := 0; t < 200; t++ {
		target := rng.IntN(1 << 31)
		a, b, got, steps, high := twoSum(root, target)
		hashGot, entries := twoSumHash(root, target)
		_, okA := slices.BinarySearch(keys, a)
		_, okB := slices.BinarySearch(keys, b)
		if got != hashGot || got && (a+b != target || a >= b || !okA || !okB) {
			bad++
		}
		if got {
			foundN++
		} else {
			missN++
			missEntries = max(missEntries, entries)
		}
		maxSteps, maxHigh, maxEntries = max(maxSteps, steps), max(maxHigh, high), max(maxEntries, entries)
	}
	fmt.Printf("200 random targets: %d found, %d not found, %d disagreements between the two methods\n", foundN, missN, bad)
	fmt.Printf("two iterators: at most %d steps, at most %d stack entries (2 * height = %d)\n", maxSteps, maxHigh, 2*h)
	fmt.Printf("hash set: at most %d entries (%d when the answer is no)\n", maxEntries, missEntries)
}
```

**Output:**

```
== (a) the iterator on a 12-key tree: the stack after every Next ==
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

new iterator pushed the left spine: [50 30 20 10]
 call  key  pushes  stack after, bottom to top
    1   10       0  [50 30 20]
    2   20       1  [50 30 25]
    3   25       0  [50 30]
    4   30       2  [50 40 35]
    5   35       0  [50 40]
    6   40       1  [50 45]
    7   45       0  [50]
    8   50       2  [70 60]
    9   60       1  [70 65]
   10   65       0  [70]
   11   70       1  [80]
   12   80       0  []
12 keys: 12 pushes and 12 pops in all
descending iterator: [80 70 65 60 50 45 40 35 30 25 20 10]

== (b) a full walk of a random tree, n = 100,000 ==
height 38
ascending: 100000 keys, equal to the sorted keys (reversed if descending): true
  pushes 100000 (11 at setup), pops 100000, pushes == pops == n: true
  max stack length 23 (height 38); most pushes in one call 16
  pushes    calls   share
       0    49807  49.81%
       1    25227  25.23%
       2    12538  12.54%
       3     6235   6.24%
       4     3111   3.11%
       5     1511   1.51%
       6      782   0.78%
       7      397   0.40%
       8      200   0.20%
       9       88   0.09%
      10       67   0.07%
      11       17   0.02%
      12       11   0.01%
      13        3   0.00%
      14        2   0.00%
      15        1   0.00%
      16        3   0.00%
  mean pushes per call 0.9999
descending: 100000 keys, equal to the sorted keys (reversed if descending): true
  pushes 100000 (11 at setup), pops 100000, pushes == pops == n: true
  max stack length 25 (height 38); most pushes in one call 15

== (c) two-sum against brute force: 200 distinct keys in [0, 1000), every target ==
2009 targets: 1894 found, 115 not found, 0 mismatches
steps (Next calls on both iterators): mean 109.9, max 201, with n = 200
targets that were not found and cost exactly n + 1 = 201 steps: 115 of 115
combined stack high-water 18 <= 2 * height = 32: true

== (d) memory as counts: iterators against a hash set, n = 100,000 ==
200 random targets: 157 found, 43 not found, 0 disagreements between the two methods
two iterators: at most 100001 steps, at most 49 stack entries (2 * height = 74)
hash set: at most 100000 entries (100000 when the answer is no)
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::HashSet;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and of its IntN, so seed (1, 2)
// draws exactly the numbers the Go program draws.
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

// Same node as the rest of the lesson. A link is Option<Box<Node>>; None is nil.
struct Node {
    key: i64,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Go needs no destructor. Rust's default drop glue recurses once per level, which
// would be a 4,000-deep recursion for the sorted-insert chain below; this Drop
// hands the children to a heap-allocated stack instead.
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

/// insert is the ordinary iterative BST insert; it reports whether the key was new.
fn insert(root: &mut Link, key: i64) -> bool {
    let mut cur = root;
    while let Some(n) = cur {
        match key.cmp(&n.key) {
            Ordering::Less => cur = &mut n.left,
            Ordering::Greater => cur = &mut n.right,
            Ordering::Equal => return false,
        }
    }
    *cur = node(key, None, None);
    true
}

/// random_tree inserts n distinct keys in [0, limit), in draw order.
fn random_tree(rng: &mut Pcg, n: usize, limit: u64) -> (Link, Vec<i64>) {
    let mut root: Link = None;
    let mut keys = Vec::new();
    while keys.len() < n {
        let k = rng.int_n(limit) as i64;
        if insert(&mut root, k) {
            keys.push(k);
        }
    }
    (root, keys)
}

/// Iter is a stack iterator over a BST. The stack holds the nodes whose turn has
/// not come yet along one spine: the left spine going forward (ascending), the
/// right spine going backward (descending). Pushes and pops are counted.
///
/// It holds `&'a Node` references, so it borrows the tree for as long as it lives.
/// Those are shared borrows, which is why two iterators can walk one tree at once.
struct Iter<'a> {
    stack: Vec<&'a Node>,
    forward: bool,
    pushes: usize,
    pops: usize,
    max_len: usize,
    last_push: usize,
}

impl<'a> Iter<'a> {
    fn new(root: &'a Link, forward: bool) -> Self {
        let mut it = Iter {
            stack: Vec::new(),
            forward,
            pushes: 0,
            pops: 0,
            max_len: 0,
            last_push: 0,
        };
        it.push_spine(root.as_deref());
        it
    }

    /// the child on the side we walk toward first
    fn first(&self, n: &'a Node) -> Option<&'a Node> {
        if self.forward {
            n.left.as_deref()
        } else {
            n.right.as_deref()
        }
    }

    fn second(&self, n: &'a Node) -> Option<&'a Node> {
        if self.forward {
            n.right.as_deref()
        } else {
            n.left.as_deref()
        }
    }

    fn push_spine(&mut self, mut n: Option<&'a Node>) {
        self.last_push = 0;
        while let Some(x) = n {
            self.stack.push(x);
            self.pushes += 1;
            self.last_push += 1;
            n = self.first(x);
        }
        self.max_len = self.max_len.max(self.stack.len());
    }
}

/// next pops one node, then pushes the spine of its far child: that child's
/// subtree is the part of the tree that comes next. Implementing `Iterator` gives
/// `for`, `collect`, `take` and the rest of the adapters for free.
impl Iterator for Iter<'_> {
    type Item = i64;

    fn next(&mut self) -> Option<i64> {
        let n = self.stack.pop()?;
        self.pops += 1;
        self.push_spine(self.second(n));
        Some(n.key)
    }
}

fn height(root: &Link) -> usize {
    let mut h = 0;
    let mut level: Vec<&Node> = root.as_deref().into_iter().collect();
    while !level.is_empty() {
        let mut next = Vec::new();
        for n in &level {
            next.extend(n.left.as_deref());
            next.extend(n.right.as_deref());
        }
        level = next;
        h += 1;
    }
    h
}

fn print_tree(n: &Link, depth: usize) {
    if let Some(n) = n {
        print_tree(&n.right, depth + 1);
        println!("{}{}", "    ".repeat(depth), n.key);
        print_tree(&n.left, depth + 1);
    }
}

/// Go prints a slice as `[1 2 3]`; Rust's `{:?}` would print `[1, 2, 3]`.
fn go_slice(xs: impl Iterator<Item = i64>) -> String {
    let parts: Vec<String> = xs.map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn stack_keys(it: &Iter) -> String {
    go_slice(it.stack.iter().map(|n| n.key))
}

/// A walk's result: the pair if there is one, the Next calls spent, and the
/// combined stack high-water.
struct Found {
    pair: Option<(i64, i64)>,
    steps: usize,
    high: usize,
}

/// two_sum runs an ascending and a descending iterator toward each other, like two
/// pointers on a sorted array. Keys are distinct, so a < b means two different nodes.
fn two_sum(root: &Link, target: i64) -> Found {
    let mut f = Iter::new(root, true);
    let mut g = Iter::new(root, false);
    let (Some(mut a), Some(mut b)) = (f.next(), g.next()) else {
        return Found {
            pair: None,
            steps: 2,
            high: 0,
        };
    };
    let mut steps = 2;
    let mut high = f.stack.len() + g.stack.len();
    while a < b {
        if a + b == target {
            return Found {
                pair: Some((a, b)),
                steps,
                high,
            };
        }
        // too small: advance the ascending one; too big: the descending one
        let (it, slot) = if a + b < target {
            (&mut f, &mut a)
        } else {
            (&mut g, &mut b)
        };
        match it.next() {
            Some(k) => *slot = k,
            None => break,
        }
        steps += 1;
        high = high.max(f.stack.len() + g.stack.len());
    }
    Found {
        pair: None,
        steps,
        high,
    }
}

/// two_sum_hash is the lesson-10 answer on the same tree: walk the keys once and
/// ask a hash set for the partner. It needs one entry per key it has passed.
fn two_sum_hash(root: &Link, target: i64) -> (bool, usize) {
    let mut seen = HashSet::new();
    for k in Iter::new(root, true) {
        if seen.contains(&(target - k)) {
            return (true, seen.len());
        }
        seen.insert(k);
    }
    (false, seen.len())
}

fn main() {
    println!("== (a) the iterator on a 12-key tree: the stack after every Next ==");
    let mut small: Link = None;
    for k in [50, 30, 70, 20, 40, 60, 80, 35, 45, 65, 10, 25] {
        insert(&mut small, k);
    }
    print_tree(&small, 0);
    let mut it = Iter::new(&small, true);
    println!("\nnew iterator pushed the left spine: {}", stack_keys(&it));
    println!(
        "{:>5} {:>4} {:>7}  stack after, bottom to top",
        "call", "key", "pushes"
    );
    let mut call = 1;
    while let Some(k) = it.next() {
        println!("{call:>5} {k:>4} {:>7}  {}", it.last_push, stack_keys(&it));
        call += 1;
    }
    println!("12 keys: {} pushes and {} pops in all", it.pushes, it.pops);
    println!(
        "descending iterator: {}",
        go_slice(Iter::new(&small, false))
    );

    println!("\n== (b) a full walk of a random tree, n = 100,000 ==");
    let mut rng = Pcg::new(1, 2);
    const N: usize = 100_000;
    let (root, keys) = random_tree(&mut rng, N, 1 << 30);
    let mut sorted = keys.clone();
    sorted.sort_unstable();
    let h = height(&root);
    println!("height {h}");
    for forward in [true, false] {
        let mut it = Iter::new(&root, forward);
        let setup = it.pushes;
        let mut walk = Vec::new();
        let mut hist: Vec<usize> = Vec::new();
        let mut max_push = 0;
        while let Some(k) = it.next() {
            walk.push(k);
            if hist.len() <= it.last_push {
                hist.resize(it.last_push + 1, 0);
            }
            hist[it.last_push] += 1;
            max_push = max_push.max(it.last_push);
        }
        if !forward {
            walk.reverse();
        }
        let name = if forward { "ascending" } else { "descending" };
        println!(
            "{name}: {} keys, equal to the sorted keys (reversed if descending): {}",
            walk.len(),
            walk == sorted
        );
        println!(
            "  pushes {} ({setup} at setup), pops {}, pushes == pops == n: {}",
            it.pushes,
            it.pops,
            it.pushes == N && it.pops == N
        );
        println!(
            "  max stack length {} (height {h}); most pushes in one call {max_push}",
            it.max_len
        );
        if forward {
            println!("  {:>6} {:>8} {:>7}", "pushes", "calls", "share");
            for (k, &calls) in hist.iter().enumerate() {
                if calls > 0 {
                    println!(
                        "  {k:>6} {calls:>8} {:>6.2}%",
                        100.0 * calls as f64 / N as f64
                    );
                }
            }
            println!(
                "  mean pushes per call {:.4}",
                (it.pushes - setup) as f64 / N as f64
            );
        }
    }

    println!(
        "\n== (c) two-sum against brute force: 200 distinct keys in [0, 1000), every target =="
    );
    let (root, mut keys) = random_tree(&mut rng, 200, 1000);
    keys.sort_unstable();
    let h = height(&root);
    let max_key = keys[keys.len() - 1];
    let (mut found_n, mut miss_n, mut miss_full, mut bad) = (0, 0, 0, 0);
    let (mut max_steps, mut max_high, mut total_steps) = (0, 0, 0);
    for target in -5..=2 * max_key + 5 {
        let want = keys
            .iter()
            .enumerate()
            .any(|(i, &x)| keys[i + 1..].iter().any(|&y| x + y == target));
        let r = two_sum(&root, target);
        let valid = r.pair.is_none_or(|(a, b)| {
            a + b == target
                && a < b
                && keys.binary_search(&a).is_ok()
                && keys.binary_search(&b).is_ok()
        });
        if r.pair.is_some() != want || !valid {
            bad += 1;
        }
        if r.pair.is_some() {
            found_n += 1;
        } else {
            miss_n += 1;
            if r.steps == keys.len() + 1 {
                miss_full += 1;
            }
        }
        total_steps += r.steps;
        max_steps = max_steps.max(r.steps);
        max_high = max_high.max(r.high);
    }
    let targets = found_n + miss_n;
    println!("{targets} targets: {found_n} found, {miss_n} not found, {bad} mismatches");
    println!(
        "steps (Next calls on both iterators): mean {:.1}, max {max_steps}, with n = {}",
        total_steps as f64 / targets as f64,
        keys.len()
    );
    println!(
        "targets that were not found and cost exactly n + 1 = {} steps: {miss_full} of {miss_n}",
        keys.len() + 1
    );
    println!(
        "combined stack high-water {max_high} <= 2 * height = {}: {}",
        2 * h,
        max_high <= 2 * h
    );

    println!("\n== (d) memory as counts: iterators against a hash set, n = 100,000 ==");
    let (root, mut keys) = random_tree(&mut rng, N, 1 << 30);
    keys.sort_unstable();
    let h = height(&root);
    let (mut found_n, mut miss_n, mut bad) = (0, 0, 0);
    let (mut max_steps, mut max_high, mut max_entries, mut miss_entries) = (0, 0, 0, 0);
    for _ in 0..200 {
        let target = rng.int_n(1 << 31) as i64;
        let r = two_sum(&root, target);
        let (hash_got, entries) = two_sum_hash(&root, target);
        let valid = r.pair.is_none_or(|(a, b)| {
            a + b == target
                && a < b
                && keys.binary_search(&a).is_ok()
                && keys.binary_search(&b).is_ok()
        });
        if r.pair.is_some() != hash_got || !valid {
            bad += 1;
        }
        if r.pair.is_some() {
            found_n += 1;
        } else {
            miss_n += 1;
            miss_entries = miss_entries.max(entries);
        }
        max_steps = max_steps.max(r.steps);
        max_high = max_high.max(r.high);
        max_entries = max_entries.max(entries);
    }
    println!(
        "200 random targets: {found_n} found, {miss_n} not found, {bad} disagreements between the two methods"
    );
    println!(
        "two iterators: at most {max_steps} steps, at most {max_high} stack entries (2 * height = {})",
        2 * h
    );
    println!("hash set: at most {max_entries} entries ({miss_entries} when the answer is no)");
}
```

**Rust output:**

```
== (a) the iterator on a 12-key tree: the stack after every Next ==
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

new iterator pushed the left spine: [50 30 20 10]
 call  key  pushes  stack after, bottom to top
    1   10       0  [50 30 20]
    2   20       1  [50 30 25]
    3   25       0  [50 30]
    4   30       2  [50 40 35]
    5   35       0  [50 40]
    6   40       1  [50 45]
    7   45       0  [50]
    8   50       2  [70 60]
    9   60       1  [70 65]
   10   65       0  [70]
   11   70       1  [80]
   12   80       0  []
12 keys: 12 pushes and 12 pops in all
descending iterator: [80 70 65 60 50 45 40 35 30 25 20 10]

== (b) a full walk of a random tree, n = 100,000 ==
height 38
ascending: 100000 keys, equal to the sorted keys (reversed if descending): true
  pushes 100000 (11 at setup), pops 100000, pushes == pops == n: true
  max stack length 23 (height 38); most pushes in one call 16
  pushes    calls   share
       0    49807  49.81%
       1    25227  25.23%
       2    12538  12.54%
       3     6235   6.24%
       4     3111   3.11%
       5     1511   1.51%
       6      782   0.78%
       7      397   0.40%
       8      200   0.20%
       9       88   0.09%
      10       67   0.07%
      11       17   0.02%
      12       11   0.01%
      13        3   0.00%
      14        2   0.00%
      15        1   0.00%
      16        3   0.00%
  mean pushes per call 0.9999
descending: 100000 keys, equal to the sorted keys (reversed if descending): true
  pushes 100000 (11 at setup), pops 100000, pushes == pops == n: true
  max stack length 25 (height 38); most pushes in one call 15

== (c) two-sum against brute force: 200 distinct keys in [0, 1000), every target ==
2009 targets: 1894 found, 115 not found, 0 mismatches
steps (Next calls on both iterators): mean 109.9, max 201, with n = 200
targets that were not found and cost exactly n + 1 = 201 steps: 115 of 115
combined stack high-water 18 <= 2 * height = 32: true

== (d) memory as counts: iterators against a hash set, n = 100,000 ==
200 random targets: 157 found, 43 not found, 0 disagreements between the two methods
two iterators: at most 100001 steps, at most 49 stack entries (2 * height = 74)
hash set: at most 100000 entries (100000 when the answer is no)
```

> **Go vs Rust:** Identical output (diffed): `Pcg` ports Go's PCG-DXSM and `IntN`, so both programs draw the same trees and targets, and `go_slice` prints `[50 30 20]` where Rust's `{:?}` would print `[50, 30, 20]`. `Iter<'a>` holds `&'a Node`, shared borrows that last as long as the iterator, so two of them walk one tree at once, and the same borrow keeps the tree from changing underneath them. Verified on rustc 1.98.1: `let mut it = Iter::new(&root, true); insert(&mut root, 5); it.next();` fails with E0502 "cannot borrow `root` as mutable because it is also borrowed as immutable", pointing at the later use of `it`. Go has no such check, so keeping an iterator valid across an edit is the caller's job. Because `Iter` implements `Iterator`, the hash-set variant is a plain `for k in Iter::new(root, true)`, where Go calls `next()` by hand.

**Complexity:** `Next` is amortized Θ(1) with Θ(h) memory: a full walk of 100,000 nodes made exactly 100,000 pushes and 100,000 pops (11 pushes at setup, the rest inside calls; 49.81% of the calls pushed nothing and the busiest pushed 16), and the stack never held more than 23 nodes in a tree of height 38. Two-sum on a BST is Θ(n) time and Θ(h) space. Over every target from −5 to 2·max + 5 on a 200-key tree (2,009 targets, 1,894 found) it agreed with the O(n²) pair scan with 0 mismatches, with a combined stack of at most 18 entries against 2h = 32. The step count is n + 1, not n: all 115 targets that were not found cost exactly 201 steps for n = 200, because the two iterators end on one key that both of them consume. On 100,000 keys the iterators held at most 49 stack entries; the hash set held up to 100,000.

---

> Next tier: [🔴 hard](3-hard.md) — delete without recursion, the drift that deletion causes, the tree
> against a sorted slice, the problem set, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
