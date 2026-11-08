# Step 05 — Recursion & the Call Stack · 🟡 Medium

Examples **7–11**: explicit stacks, memoization, the stack limit, and the bug that ruins backtracking in Go.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 7, 8, 10 and 11 are **deterministic**. Example 9 measures stack bytes and spawns a child
process, so its numbers and traceback vary.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

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

**Rust:**

```rust
// When a recursion makes MORE THAN ONE recursive call, you cannot turn it into
// a plain loop. But you can always turn it into a loop plus an explicit stack --
// because that is exactly what the call stack was doing for you.
//
// The transformation:
//   1. make a stack of the things the recursive parameter held
//   2. loop while the stack is non-empty: pop, do the visit, push the children
//   3. push children in REVERSE order, because a stack reverses them again

// Go's *node (nil or a pointer) is Option<Box<Node>>: "no child" is a type, not nil.
struct Node {
    val: i32,
    left: Option<Box<Node>>,
    right: Option<Box<Node>>,
}

// The default drop of a Box chain is itself recursive: dropping a node drops its
// children, one frame per level. A million-deep chain would overflow the stack
// while being freed, so tear it down with an explicit stack too (example 9
// measures what the thread's stack can hold).
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

fn leaf(v: i32) -> Option<Box<Node>> {
    Some(Box::new(Node {
        val: v,
        left: None,
        right: None,
    }))
}

// --- recursive: the call stack does the bookkeeping ---

// `Option<&Node>` borrows the tree: no clone, no ownership change, and the
// borrow checker guarantees the tree outlives every reference we hand around.
fn preorder_rec(n: Option<&Node>, out: &mut Vec<i32>) {
    let Some(n) = n else { return };
    out.push(n.val); // visit
    preorder_rec(n.left.as_deref(), out);
    preorder_rec(n.right.as_deref(), out);
}

// --- iterative: you do the bookkeeping ---

// The explicit stack holds `&Node` -- 8-byte borrows into the tree, not copies.
fn preorder_iter(root: &Node) -> Vec<i32> {
    let mut out = Vec::new();
    let mut stack = vec![root];

    while let Some(n) = stack.pop() {
        out.push(n.val); // visit

        // Push RIGHT first so that LEFT is popped first -- a stack reverses.
        if let Some(r) = n.right.as_deref() {
            stack.push(r);
        }
        if let Some(l) = n.left.as_deref() {
            stack.push(l);
        }
    }
    out
}

// --- in-order is the harder one: the visit happens BETWEEN the two calls ---

fn inorder_rec(n: Option<&Node>, out: &mut Vec<i32>) {
    let Some(n) = n else { return };
    inorder_rec(n.left.as_deref(), out);
    out.push(n.val); // visit is in the middle
    inorder_rec(n.right.as_deref(), out);
}

// inorder_iter needs a "walk left as far as possible, then visit, then go right"
// loop, because the visit happens after the left subtree is finished.
fn inorder_iter(root: &Node) -> Vec<i32> {
    let mut out = Vec::new();
    let mut stack: Vec<&Node> = Vec::new();
    let mut cur = Some(root);

    while cur.is_some() || !stack.is_empty() {
        while let Some(n) = cur {
            // descend left, remembering the way back
            stack.push(n);
            cur = n.left.as_deref();
        }
        // the left subtree is done (the loop condition guarantees a pop succeeds)
        let Some(n) = stack.pop() else { break };

        out.push(n.val); // visit
        cur = n.right.as_deref(); // and now the right subtree
    }
    out
}

// max_stack_depth walks with an explicit stack of (node, depth) pairs; the
// largest depth it sees is the recursion depth the recursive version would
// have used.
fn max_stack_depth(root: &Node) -> usize {
    let mut stack = vec![(root, 1)];
    let mut best = 0;
    while let Some((n, depth)) = stack.pop() {
        best = best.max(depth);
        for child in [n.right.as_deref(), n.left.as_deref()]
            .into_iter()
            .flatten()
        {
            stack.push((child, depth + 1));
        }
    }
    best
}

fn build() -> Node {
    //         1
    //       /   \
    //      2     3
    //     / \   /
    //    4   5 6
    Node {
        val: 1,
        left: Some(Box::new(Node {
            val: 2,
            left: leaf(4),
            right: leaf(5),
        })),
        right: Some(Box::new(Node {
            val: 3,
            left: leaf(6),
            right: None,
        })),
    }
}

// Go's %v of a []int is "[1 2 4 5 3 6]" (spaces, no commas); {:?} would give commas.
fn go_list(xs: &[i32]) -> String {
    let items: Vec<String> = xs.iter().map(i32::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    let root = build();

    let mut rec = Vec::new();
    preorder_rec(Some(&root), &mut rec);
    let iter = preorder_iter(&root);

    println!("pre-order (visit, then left, then right):");
    println!("  recursive  {}", go_list(&rec));
    println!("  iterative  {}", go_list(&iter));

    let mut rec_in = Vec::new();
    inorder_rec(Some(&root), &mut rec_in);
    let iter_in = inorder_iter(&root);

    println!();
    println!("in-order (left, then visit, then right):");
    println!("  recursive  {}", go_list(&rec_in));
    println!("  iterative  {}", go_list(&iter_in));

    println!();
    println!(
        "max depth reached: {} (that is how many frames the recursion needed)",
        max_stack_depth(&root)
    );

    println!();
    println!("why pre-order is easy and in-order is not:");
    println!();
    println!("  pre-order visits BEFORE both calls, so a node is finished the moment");
    println!("  you pop it -- push children and move on.");
    println!();
    println!("  in-order visits BETWEEN the calls, so a popped node is only HALF done:");
    println!("  its left subtree must complete first. The explicit version encodes that");
    println!("  as 'descend left pushing everything, then pop-visit-go-right'.");
    println!();
    println!("that extra bookkeeping is precisely what the call stack was doing for free.");
    println!("Converting is worth it when depth can grow with n -- a degenerate tree is");
    println!("a linked list, and n frames deep (example 15).");
    println!();
    println!("note the explicit stack lives on the HEAP: a Vec grows by reallocation");
    println!("(lesson 03) instead of by frame, so it can reach millions of entries");
    println!("without ever touching the thread's fixed stack (example 9). Checked here,");
    println!("on the main thread, on a degenerate tree that leans left 1,000,000 deep:");
    println!();

    let mut chain = Node {
        val: 0,
        left: None,
        right: None,
    };
    for v in 1..1_000_000 {
        chain = Node {
            val: v,
            left: Some(Box::new(chain)),
            right: None,
        };
    }
    let visited = inorder_iter(&chain);
    println!(
        "  in-order, explicit stack: {} values visited, first {}, last {}",
        visited.len(),
        visited[0],
        visited[visited.len() - 1]
    );
    println!("  max depth, explicit stack: {}", max_stack_depth(&chain));
}
```

**Rust output:**

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

note the explicit stack lives on the HEAP: a Vec grows by reallocation
(lesson 03) instead of by frame, so it can reach millions of entries
without ever touching the thread's fixed stack (example 9). Checked here,
on the main thread, on a degenerate tree that leans left 1,000,000 deep:

  in-order, explicit stack: 1000000 values visited, first 0, last 999999
  max depth, explicit stack: 1000000
```

> **Go vs Rust:** Go's `*node` becomes `Option<Box<Node>>`, so "no child" is a type rather than `nil`, and the explicit stack is a `Vec<&Node>` of borrows into the tree (the compiler proves the tree outlives them). The four traversal lines and the max depth match the Go output exactly (`[1 2 4 5 3 6]` is hand-formatted because `{:?}` prints commas). The 1,000,000-deep check at the end is new: without the hand-written iterative `Drop`, freeing that chain recurses once per level and overflows the main thread's stack (verified by deleting the `Drop` impl), a trap Go's GC does not have.

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

**Rust:**

```rust
use std::collections::HashMap;

// A recursion whose tree contains the SAME subproblem more than once is doing
// redundant work. Recording each answer the first time turns an exponential
// tree into a linear one -- and that single idea is the whole of dynamic
// programming (lessons 30-32).
//
// Three versions of the same function, in the order you should always write them:
//   1. plain recursion   -- correct, obvious, exponential
//   2. + memo            -- correct, obvious, linear. Change: 3 lines.
//   3. tabulation        -- same recurrence, bottom-up, no recursion at all
//
// Rust has no mutable package-level `var calls int`: the counter is threaded
// through every call as `&mut u64`, exactly like the memo.

// --- 1. the honest recursion ---

fn fib(n: u64, calls: &mut u64) -> u64 {
    *calls += 1;
    if n < 2 {
        return n;
    }
    fib(n - 1, calls) + fib(n - 2, calls)
}

// --- 2. the same function, plus a memo ---

fn fib_memo(n: u64, memo: &mut HashMap<u64, u64>, calls: &mut u64) -> u64 {
    *calls += 1;
    if n < 2 {
        return n;
    }
    if let Some(&v) = memo.get(&n) {
        // added
        return v; // added
    } // added
    let v = fib_memo(n - 1, memo, calls) + fib_memo(n - 2, memo, calls);
    memo.insert(n, v); // added
    v
}

// --- 3. bottom-up: fill the table in dependency order ---

fn fib_table(n: u64) -> u64 {
    if n < 2 {
        return n;
    }
    let (mut prev, mut cur) = (0, 1);
    for _ in 2..=n {
        (prev, cur) = (cur, prev + cur);
    }
    cur
}

// count_target answers "how many times does computing fib(n) invoke fib(target)?"
// -- the redundancy, measured rather than asserted.
fn count_target(n: u64, target: u64, hits: &mut u64) -> u64 {
    if n == target {
        *hits += 1;
    }
    if n < 2 {
        return n;
    }
    count_target(n - 1, target, hits) + count_target(n - 2, target, hits)
}

// --- the same three stages on a two-dimensional problem ---

// grid_paths counts routes from (0,0) to (r,c) moving only down or right.
fn grid_paths(r: u64, c: u64, calls: &mut u64) -> u64 {
    *calls += 1;
    if r == 0 || c == 0 {
        return 1;
    }
    grid_paths(r - 1, c, calls) + grid_paths(r, c - 1, calls)
}

// The key is a tuple: Go's [2]int array key. Tuples of integers are Hash + Eq.
fn grid_paths_memo(r: u64, c: u64, memo: &mut HashMap<(u64, u64), u64>, calls: &mut u64) -> u64 {
    *calls += 1;
    if r == 0 || c == 0 {
        return 1;
    }
    if let Some(&v) = memo.get(&(r, c)) {
        return v;
    }
    let v = grid_paths_memo(r - 1, c, memo, calls) + grid_paths_memo(r, c - 1, memo, calls);
    memo.insert((r, c), v);
    v
}

fn main() {
    println!("fib(n): calls made by each version");
    println!();
    println!(
        "{:>6} {:>16} {:>14} {:>14}",
        "n", "plain", "memoized", "fib(n)"
    );
    for n in [10, 20, 30, 35] {
        let mut calls = 0;
        let got = fib(n, &mut calls);
        let plain = calls;

        calls = 0;
        fib_memo(n, &mut HashMap::new(), &mut calls);
        let memo = calls;

        println!("{n:>6} {plain:>16} {memo:>14} {got:>14}");
    }
    println!(
        "\n  tabulated fib(35) = {}, in 34 loop iterations and O(1) space",
        fib_table(35)
    );

    println!();
    println!("grid paths on an r x c grid:");
    println!();
    println!(
        "{:>10} {:>16} {:>14} {:>14}",
        "grid", "plain", "memoized", "paths"
    );
    for n in [4, 8, 12, 14] {
        let mut calls = 0;
        let got = grid_paths(n, n, &mut calls);
        let plain = calls;

        calls = 0;
        grid_paths_memo(n, n, &mut HashMap::new(), &mut calls);
        let memo = calls;

        println!("{n:>8}x{n:<2} {plain:>16} {memo:>14} {got:>14}");
    }

    println!();
    println!("the memo works because the recursion tree REVISITS subproblems.");
    println!("fib(5) asks for fib(3) twice. Deeper down it gets absurd:");
    println!();
    for target in [20, 15, 10, 5] {
        let mut hits = 0;
        count_target(30, target, &mut hits);
        println!("  computing fib(30) invokes fib({target:>2}) {hits:>10} times");
    }
    println!();
    println!("With a memo each distinct subproblem is computed exactly once, so the");
    println!("call count collapses to O(number of distinct subproblems).");
    println!();
    println!("  fib:        n distinct subproblems     -> Theta(n)");
    println!("  gridPaths:  r*c distinct subproblems   -> Theta(r*c)");
    println!();
    println!("that is the entire trick, and it has a name for each direction:");
    println!("  MEMOIZATION  top-down: recurse, and cache on the way back");
    println!("  TABULATION   bottom-up: order the subproblems and fill a table");
    println!();
    println!("memoization is easier to derive (it is your recursion plus 3 lines).");
    println!("Tabulation removes the recursion entirely, so it cannot overflow the");
    println!("stack and it often needs less space -- fib_table keeps two ints, not n.");
    println!();
    println!("when memoizing does NOT help: a recursion tree with no repeated");
    println!("subproblems. Merge sort's halves never overlap, so there is nothing");
    println!("to cache -- that is divide & conquer, not DP (example 13).");
}
```

**Rust output:**

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
stack and it often needs less space -- fib_table keeps two ints, not n.

when memoizing does NOT help: a recursion tree with no repeated
subproblems. Merge sort's halves never overlap, so there is nothing
to cache -- that is divide & conquer, not DP (example 13).
```

> **Go vs Rust:** Every number in the three tables is identical to the Go output, because call counts belong to the algorithm (29,860,703 plain calls for `fib(35)`, 69 memoized). The only differing line is the prose naming `fib_table`. Go's package-level `calls` counter has no Rust twin: it becomes a `&mut u64` parameter threaded through every function, exactly like the memo, and the `[2]int` map key becomes a `(u64, u64)` tuple.

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

**Rust:**

```rust
use std::hint::black_box;
use std::process::Command;
use std::thread;

// How deep can a Rust recursion actually go, and what happens at the end?
//
// Rust does not grow stacks. Every thread gets ONE fixed block when it is
// created: the main thread's comes from the OS (`ulimit -s`, usually 8 MB), and
// a thread you spawn gets 2 MiB unless you ask for another size with
// `thread::Builder::stack_size`. Go's goroutines start at ~8 KB and GROW by
// copying, up to 1 GB -- nothing like that exists here.
//
// The important part is the same as in Go: running out is NOT a panic. It is a
// fatal runtime error, and neither catch_unwind nor Drop can run. The process
// is aborted.
//
// Aborting is an effect on the machine (macOS writes a crash report for every
// aborted process), so a plain run of this program NEVER crashes anything: it
// measures the frame size and PREDICTS where each stack would overflow. The real
// overflow, in child processes, only happens with `--overflow`.

// The main thread's stack on the machine this was written on (`ulimit -s` printed
// 8176 KiB). Std cannot read it, so the prediction below assumes it.
const MAIN_STACK: usize = 8176 << 10;
const THREAD_STACK: usize = 4 << 20; // the child's thread, the same 4 MiB as Go's child
const PROGRESS_EVERY: u64 = 4096; // the crash run prints a line every this many frames

// `#[inline(never)]` and `black_box` stop LLVM from turning `1 + deep(n - 1)`
// into a loop, which at -O it happily would. Without them this would not be
// recursion at all by the time it runs.
#[inline(never)]
fn deep(n: u64) -> u64 {
    if n == 0 {
        return 0;
    }
    1 + black_box(deep(n - 1))
}

// The address of a local variable, as a number. Taking a reference and casting
// it to a pointer is safe; only dereferencing a raw pointer needs `unsafe`.
fn addr_of_local(marker: &u8) -> usize {
    marker as *const u8 as usize
}

#[inline(never)]
fn report(depth: u64, used: usize) {
    println!("depth {depth} stack-used {used}");
}

// descend recurses from `depth` to `limit` and returns how many bytes of stack
// lie between `top` (a local in the caller) and a local in the deepest frame.
// The stack grows DOWN, so top > here. With `progress` it also prints a line
// every PROGRESS_EVERY frames, so a parent process can see how far a run got
// before dying. One function serves the measurement AND the crash run, so both
// have the same frame.
#[inline(never)]
fn descend(depth: u64, limit: u64, top: usize, progress: bool) -> usize {
    let marker = 0u8;
    let here = addr_of_local(black_box(&marker));
    if progress && depth.is_multiple_of(PROGRESS_EVERY) {
        report(depth, top - here);
    }
    if depth == limit {
        return top - here;
    }
    black_box(descend(depth + 1, limit, top, progress))
}

// bytes_per_frame measures in a FRESH thread, whose stack we size ourselves:
// 256 MiB is address space, not memory -- pages are only touched as the
// recursion reaches them.
fn bytes_per_frame(depth: u64) -> f64 {
    let used = thread::Builder::new()
        .stack_size(256 << 20)
        .spawn(move || {
            let marker = 0u8;
            let top = addr_of_local(black_box(&marker));
            descend(0, depth, top, false)
        })
        .expect("spawn")
        .join()
        .expect("join");
    used as f64 / depth as f64
}

// The deepest frame the stack can hold, and the last progress line that run
// would print before the overflow.
fn predicted_frames(stack: usize, per_frame: f64) -> u64 {
    (stack as f64 / per_frame) as u64
}

fn predicted_last_progress(stack: usize, per_frame: f64) -> u64 {
    predicted_frames(stack, per_frame) / PROGRESS_EVERY * PROGRESS_EVERY
}

// ---------- everything below this line only runs with --overflow ----------

// The child branch: recurse forever on a thread with a deliberately small stack
// (4 MiB), or on the main thread.
struct Guard;
impl Drop for Guard {
    fn drop(&mut self) {
        println!("guard dropped");
    }
}

fn run_forever() {
    let _guard = Guard; // would print if unwinding or normal exit ever ran Drop
    let marker = 0u8;
    let top = addr_of_local(black_box(&marker));
    match std::panic::catch_unwind(|| descend(0, u64::MAX, top, true)) {
        Ok(used) => println!("returned {used}"),
        Err(_) => println!("recovered"), // this is Rust's recover()
    }
}

// This function ABORTS THE PROCESS. It is only reached through `--child`, which
// only `--overflow` passes.
fn child_overflow(mode: &str) {
    if mode == "main" {
        run_forever();
    } else {
        thread::Builder::new()
            .stack_size(THREAD_STACK)
            .spawn(run_forever)
            .expect("spawn")
            .join()
            .expect("join");
    }
}

// Re-run this very executable as a child, in the given mode, and show what the
// parent can observe: the prediction against the last progress line, the first
// lines of stderr, the exit status, and whether any recovery / cleanup code ran.
fn run_child(mode: &str, predicted: u64) {
    let exe = std::env::current_exe().expect("current_exe");
    let out = Command::new(exe)
        .args(["--child", mode])
        .output()
        .expect("spawn child");

    let stdout = String::from_utf8_lossy(&out.stdout);
    let stderr = String::from_utf8_lossy(&out.stderr);
    let last = stdout.lines().rfind(|l| l.starts_with("depth"));
    // "depth D stack-used U"  ->  (D, U)
    let parsed = last.and_then(|l| {
        let mut f = l.split_whitespace();
        Some((
            f.nth(1)?.parse::<u64>().ok()?,
            f.nth(1)?.parse::<u64>().ok()?,
        ))
    });
    match parsed {
        Some((d, u)) => {
            println!("  predicted last progress line: depth {predicted}");
            println!(
                "  observed  last progress line: depth {d}, {u} bytes of stack used (~{:.0} bytes/frame)",
                u as f64 / d as f64
            );
            println!("  prediction matched: {}", d == predicted);
        }
        None => println!("  no progress line"),
    }
    let lines: Vec<&str> = stderr.trim().lines().collect();
    for ln in lines.iter().take(4) {
        println!("  {ln}");
    }
    if lines.len() > 4 {
        println!("  ... ({} more lines of stderr)", lines.len() - 4);
    }
    println!("  child exit status: {}", out.status);
    let ran = stdout
        .lines()
        .any(|l| l == "recovered" || l == "guard dropped");
    println!("  catch_unwind or Drop ran in the child: {ran}");
}

fn overflow_demo(per_frame: f64) {
    println!("--overflow: two child processes that really overflow (each is aborted,");
    println!("and macOS writes a crash report for each one).");
    for (mode, what, stack) in [
        ("thread", "a 4 MiB thread", THREAD_STACK),
        ("main", "the main thread", MAIN_STACK),
    ] {
        println!();
        println!("child on {what}, recursing forever:");
        println!();
        run_child(mode, predicted_last_progress(stack, per_frame));
    }
}

fn main() {
    let args: Vec<String> = std::env::args().skip(1).collect();
    // Internal: the crash itself. Never reached without an explicit argument.
    if args.first().map(String::as_str) == Some("--child") {
        child_overflow(args.get(1).map_or("thread", String::as_str));
        return;
    }

    println!("Rust stacks are a fixed size chosen when the thread is created, so deep");
    println!("recursion works -- right up until the block is full.");
    println!();

    println!("{:>14} {:>18}", "depth", "bytes per frame");
    for d in [10_000, 100_000, 1_000_000] {
        println!("{d:>14} {:>18.0}", bytes_per_frame(d));
    }

    println!();
    let result = thread::Builder::new()
        .stack_size(256 << 20)
        .spawn(|| deep(1_000_000))
        .expect("spawn")
        .join()
        .expect("join");
    println!(
        "deep(1000000) returned {result} on a 256 MiB thread, so a million frames is fine there."
    );

    let per_frame = bytes_per_frame(1_000_000);
    println!();
    println!("where each stack would overflow, predicted from ~{per_frame:.0} bytes/frame:");
    println!();
    println!(
        "{:<36} {:>12} {:>20}",
        "stack", "max frames", "last progress line"
    );
    for (name, bytes) in [
        ("a default spawned thread (2 MiB)", 2 << 20),
        ("a 4 MiB thread", THREAD_STACK),
        ("the main thread (8176 KiB here)", MAIN_STACK),
    ] {
        println!(
            "{name:<36} {:>12} {:>20}",
            predicted_frames(bytes, per_frame),
            predicted_last_progress(bytes, per_frame)
        );
    }
    println!();
    println!(
        "(\"last progress line\": the crash run prints one line every {PROGRESS_EVERY} frames,"
    );
    println!("so the last line it manages to print is the largest multiple of {PROGRESS_EVERY}");
    println!("below the overflow.)");

    if args.first().map(String::as_str) == Some("--overflow") {
        println!();
        overflow_demo(per_frame);
    } else {
        println!();
        println!("what happens at that depth is not run by default, because it ABORTS the");
        println!("process (and macOS writes a crash report for every abort). To watch it, in");
        println!("two child processes: run this program with the argument --overflow.");
    }

    println!();
    println!("what to expect there, and the three things to take from it:");
    println!();
    println!("  1. it is a \"fatal runtime error\", not a panic. Neither catch_unwind nor a");
    println!("     Drop guard runs -- you cannot catch this, and destructors do not run.");
    println!("  2. the process dies from a signal (SIGABRT), with every thread in it. In a");
    println!("     server, one deeply-nested request takes down every other in-flight");
    println!("     request with it.");
    println!("  3. the limit is per THREAD and fixed when the thread is created: the table");
    println!("     above gives a different depth for each stack, and a thread pool or a");
    println!("     timeout does not protect you.");
    println!();
    println!("the defence is never a bigger limit (stack_size is one argument away, and");
    println!("only moves the cliff). It is either bounding the depth (reject input deeper");
    println!("than N) or removing the recursion (examples 7, 15).");
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O ex09.rs -o ex09
./ex09              # measures frame size and PREDICTS the overflow depths; crashes nothing
./ex09 --overflow   # really overflows two child processes (each aborts; macOS writes a crash report per abort)
```

**Rust sample output:**

```
Rust stacks are a fixed size chosen when the thread is created, so deep
recursion works -- right up until the block is full.

         depth    bytes per frame
         10000                 96
        100000                 96
       1000000                 96

deep(1000000) returned 1000000 on a 256 MiB thread, so a million frames is fine there.

where each stack would overflow, predicted from ~96 bytes/frame:

stack                                  max frames   last progress line
a default spawned thread (2 MiB)            21845                20480
a 4 MiB thread                              43690                40960
the main thread (8176 KiB here)             87210                86016

("last progress line": the crash run prints one line every 4096 frames,
so the last line it manages to print is the largest multiple of 4096
below the overflow.)

what happens at that depth is not run by default, because it ABORTS the
process (and macOS writes a crash report for every abort). To watch it, in
two child processes: run this program with the argument --overflow.

what to expect there, and the three things to take from it:

  1. it is a "fatal runtime error", not a panic. Neither catch_unwind nor a
     Drop guard runs -- you cannot catch this, and destructors do not run.
  2. the process dies from a signal (SIGABRT), with every thread in it. In a
     server, one deeply-nested request takes down every other in-flight
     request with it.
  3. the limit is per THREAD and fixed when the thread is created: the table
     above gives a different depth for each stack, and a thread pool or a
     timeout does not protect you.

the defence is never a bigger limit (stack_size is one argument away, and
only moves the cliff). It is either bounding the depth (reject input deeper
than N) or removing the recursion (examples 7, 15).
```

> **Go vs Rust:** Go's stack grows by copying up to 1 GB; Rust's is one fixed block per thread, so the table is measured by stack-address difference in a thread given 256 MiB: 96 bytes/frame at all three depths, where Go's MemStats deltas read 52, 42, 34 (different functions and compilers, so not a like-for-like comparison). The overflow is predicted, not run, by default because an abort makes macOS write a crash report. A separate `--overflow` run (not the output above) checked the prediction: the child on a 4 MiB thread printed its last progress line at depth 40,960 and the main-thread child at 86,016, exactly the predicted values (max frames 43,690 and 87,210), after which both died with SIGABRT and two lines on stderr rather than Go's exit status 2 and traceback. Neither `catch_unwind` (the nearest thing to `recover`) nor a `Drop` guard ran in either child, as in Go. Frame size is a compiler and target detail, hence "sample".

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

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::RefCell;
use std::collections::HashSet;
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// THE bug of recursive algorithms in Go is lesson 04's append-aliasing trap, and
// recursion is where it hides best. The pattern: build up a partial answer in a
// slice, and record it when you reach a leaf. If you record the slice ITSELF,
// every recorded answer is a view into one shared backing array, and later
// branches overwrite earlier results.
//
// Rust cannot write that bug by accident. A `&[T]` borrows from its `Vec`, and
// while it is alive nobody can `push` to the Vec. This is what the Go program's
// shape becomes if you try it (compiled for this example, rustc 1.98):
//
//     fn subsets_view<'a>(nums: &[i32], i: usize, cur: &'a mut Vec<i32>,
//                         out: &mut Vec<&'a [i32]>) {
//         if i == nums.len() {
//             out.push(&cur[..]); // record a view of the working buffer
//             return;
//         }
//         subsets_view(nums, i + 1, cur, out);
//         cur.push(nums[i]);
//         subsets_view(nums, i + 1, cur, out);
//     }
//
//     error[E0499]: cannot borrow `*cur` as mutable more than once at a time
//       |
//     6 |     subsets_view(nums, i + 1, cur, out);
//       |     -----------------------------------
//       |     |                         |
//       |     |                         first mutable borrow occurs here
//       |     argument requires that `*cur` is borrowed for `'a`
//     7 |     cur.push(nums[i]);
//       |     ^^^ second mutable borrow occurs here
//
// So there is no broken Rust version to run. To still SEE the Go bug, the first
// version below builds Go's slice header by hand: a pointer to a shared backing
// array (Rc<RefCell<..>>), a len and a cap, with Go's append rule. It is a
// model of Go, not something idiomatic Rust ever needs.

#[derive(Clone)]
struct GoSlice {
    arr: Rc<RefCell<Vec<i32>>>, // the backing array, shared by every copy of the header
    len: usize,
    cap: usize,
}

impl GoSlice {
    // make([]int, 0, cap)
    fn with_cap(cap: usize) -> Self {
        GoSlice {
            arr: Rc::new(RefCell::new(vec![0; cap])),
            len: 0,
            cap,
        }
    }

    // append(s, x): spare capacity means write IN PLACE into the shared array;
    // otherwise allocate a bigger array (small slices double), copy, then write.
    fn append(&self, x: i32) -> GoSlice {
        if self.len < self.cap {
            self.arr.borrow_mut()[self.len] = x;
            GoSlice {
                arr: Rc::clone(&self.arr),
                len: self.len + 1,
                cap: self.cap,
            }
        } else {
            let cap = (self.cap * 2).max(1);
            let mut v = self.arr.borrow()[..self.len].to_vec();
            v.resize(cap, 0);
            v[self.len] = x;
            GoSlice {
                arr: Rc::new(RefCell::new(v)),
                len: self.len + 1,
                cap,
            }
        }
    }

    // s[:len(s):len(s)] -- same array, but no spare capacity left to write into.
    fn clip(&self) -> GoSlice {
        GoSlice {
            cap: self.len,
            ..self.clone()
        }
    }

    // What the slice shows RIGHT NOW: the first `len` elements of the shared array.
    fn contents(&self) -> Vec<i32> {
        self.arr.borrow()[..self.len].to_vec()
    }
}

// --- the broken version, as Go wrote it (modelled) ---

fn subsets_bad(nums: &[i32], i: usize, cur: &GoSlice, out: &mut Vec<GoSlice>, clip: bool) {
    if i == nums.len() {
        // BUG: copying the header copies the pointer, not the data
        out.push(cur.clone());
        return;
    }
    // `clip` is Go's defence 3: cur[:len(cur):len(cur)] when passing down
    let cur = if clip { cur.clip() } else { cur.clone() };
    subsets_bad(nums, i + 1, &cur, out, clip); // exclude nums[i]
    subsets_bad(nums, i + 1, &cur.append(nums[i]), out, clip); // include nums[i]
}

// --- fix 1: ownership. Each call OWNS its Vec; the leaf moves it into the answer ---

fn subsets_owned(nums: &[i32], i: usize, cur: Vec<i32>, out: &mut Vec<Vec<i32>>) {
    if i == nums.len() {
        out.push(cur); // a move: no copy, and nobody else can hold `cur`
        return;
    }
    let mut with = cur.clone(); // the price: one copy per branch
    with.push(nums[i]);
    subsets_owned(nums, i + 1, cur, out); // exclude nums[i]
    subsets_owned(nums, i + 1, with, out); // include nums[i]
}

// --- fix 2: the classic choose / explore / un-choose ---
//
// One Vec, mutated in place and restored on the way out. This is the shape
// used by every backtracking algorithm in lesson 29.
fn subsets_backtrack(nums: &[i32], i: usize, cur: &mut Vec<i32>, out: &mut Vec<Vec<i32>>) {
    if i == nums.len() {
        out.push(cur.clone()); // still must copy at the leaf
        return;
    }
    // don't choose nums[i]
    subsets_backtrack(nums, i + 1, cur, out);

    // choose
    cur.push(nums[i]);
    subsets_backtrack(nums, i + 1, cur, out);
    cur.pop(); // un-choose
}

// Go's %v of a [][]int: [[] [1] [1 2]] -- spaces, no commas.
fn go_list(xs: &[i32]) -> String {
    let items: Vec<String> = xs.iter().map(i32::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn go_lists(xss: &[Vec<i32>]) -> String {
    let items: Vec<String> = xss.iter().map(|xs| go_list(xs)).collect();
    format!("[{}]", items.join(" "))
}

fn count_distinct(xss: &[Vec<i32>]) -> usize {
    xss.iter().collect::<HashSet<_>>().len()
}

// The four runs, each starting from an empty working buffer of capacity `cap`.
fn run_model(nums: &[i32], cap: usize, clip: bool) -> Vec<Vec<i32>> {
    let mut out = Vec::new();
    subsets_bad(nums, 0, &GoSlice::with_cap(cap), &mut out, clip);
    // The answers are read only NOW, after the whole recursion has finished.
    out.iter().map(GoSlice::contents).collect()
}

fn run_owned(nums: &[i32], cap: usize) -> Vec<Vec<i32>> {
    let mut out = Vec::new();
    subsets_owned(nums, 0, Vec::with_capacity(cap), &mut out);
    out
}

fn run_backtrack(nums: &[i32], cap: usize) -> Vec<Vec<i32>> {
    let mut out = Vec::new();
    subsets_backtrack(nums, 0, &mut Vec::with_capacity(cap), &mut out);
    out
}

// ---- allocation counter (the shared kit): replaces Go's AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    // SAFETY: every method forwards unchanged to the system allocator; we only count.
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

fn main() {
    let nums = [1, 2, 3];

    println!("all subsets of [1 2 3] -- there should be 8, all distinct:");
    println!();
    let bad = run_model(&nums, 3, false);
    let owned = run_owned(&nums, 3);
    let bt = run_backtrack(&nums, 3);
    println!("  {:<24} {}", "Go slice (modelled)", go_lists(&bad));
    println!("  {:<24} {}", "owned Vec, moved", go_lists(&owned));
    println!("  {:<24} {}", "backtracking", go_lists(&bt));

    println!();
    println!(
        "distinct results: go-slice={}  owned={}  backtrack={}  (want 8)",
        count_distinct(&bad),
        count_distinct(&owned),
        count_distinct(&bt)
    );

    println!();
    println!("why the first one is wrong -- and why it cannot be written in Rust:");
    println!();
    println!("  The first row is a MODEL of Go's slice header: (shared array, len, cap).");
    println!("  `append` on a header with spare capacity writes into the shared array,");
    println!("  so both branches hold headers over the SAME array. Whatever the last");
    println!("  branch writes is what every previously-recorded slice now shows.");
    println!();
    println!("  Real Rust has no such header to copy. `&[T]` borrows its Vec, and while");
    println!("  a borrow is alive the Vec cannot be pushed to: the program that records");
    println!("  `&cur[..]` and keeps pushing is a compile error (E0499, see the comment");
    println!("  at the top of the source), however much capacity there is.");

    println!();
    println!("what makes the bug appear: spare capacity (distinct results out of 8)");
    println!();
    println!(
        "  {:<46} {:>9}",
        "modelled Go slice, cap 0",
        count_distinct(&run_model(&nums, 0, false))
    );
    println!(
        "  {:<46} {:>9}",
        "modelled Go slice, cap 3",
        count_distinct(&run_model(&nums, 3, false))
    );
    println!(
        "  {:<46} {:>9}",
        "modelled Go slice, cap 3, [:len:len]",
        count_distinct(&run_model(&nums, 3, true))
    );
    let same = run_owned(&nums, 0) == run_owned(&nums, 3)
        && run_backtrack(&nums, 0) == run_backtrack(&nums, 3);
    println!(
        "  {:<46} {:>9}",
        "real Vec, cap 0 vs cap 3 (owned, backtrack)",
        if same { "identical" } else { "DIFFERENT" }
    );

    println!();
    println!("what the two real fixes cost: allocation events for all subsets of 10 items");
    println!();
    let ten: Vec<i32> = (1..=10).collect();
    let (a_owned, r1) = allocs_of(|| run_owned(&ten, 10));
    let (a_bt, r2) = allocs_of(|| run_backtrack(&ten, 10));
    println!("  {:<40} {:>8}", "owned Vec, clone per branch", a_owned);
    println!("  {:<40} {:>8}", "backtracking, clone at the leaf", a_bt);
    println!("  ({} subsets each, same answers: {})", r1.len(), r1 == r2);

    println!();
    println!("three defences, in order of preference:");
    println!();
    println!("  1. let the type system do it: `&mut Vec` + push/pop (backtracking), or an");
    println!("     owned Vec per call that you MOVE into the answer. Neither can alias.");
    println!("  2. choose / explore / un-choose: one buffer, mutated and restored. The");
    println!("     leaf still needs `cur.clone()`, because the buffer is reused.");
    println!("  3. (Go only) cur[:len(cur):len(cur)] forces append to copy, as the");
    println!("     [:len:len] row above shows. Rust has nothing to clip: a Vec's spare");
    println!("     capacity is never shared with anyone else.");

    println!();
    println!("the rule: a Vec you hand to a recursive call is either BORROWED mutably");
    println!("(restore it before returning) or MOVED (nobody else can see it).");
}
```

**Rust sample output:**

```
all subsets of [1 2 3] -- there should be 8, all distinct:

  Go slice (modelled)      [[] [1] [1] [1 2] [1] [1 2] [1 2] [1 2 3]]
  owned Vec, moved         [[] [3] [2] [2 3] [1] [1 3] [1 2] [1 2 3]]
  backtracking             [[] [3] [2] [2 3] [1] [1 3] [1 2] [1 2 3]]

distinct results: go-slice=4  owned=8  backtrack=8  (want 8)

why the first one is wrong -- and why it cannot be written in Rust:

  The first row is a MODEL of Go's slice header: (shared array, len, cap).
  `append` on a header with spare capacity writes into the shared array,
  so both branches hold headers over the SAME array. Whatever the last
  branch writes is what every previously-recorded slice now shows.

  Real Rust has no such header to copy. `&[T]` borrows its Vec, and while
  a borrow is alive the Vec cannot be pushed to: the program that records
  `&cur[..]` and keeps pushing is a compile error (E0499, see the comment
  at the top of the source), however much capacity there is.

what makes the bug appear: spare capacity (distinct results out of 8)

  modelled Go slice, cap 0                               8
  modelled Go slice, cap 3                               4
  modelled Go slice, cap 3, [:len:len]                   8
  real Vec, cap 0 vs cap 3 (owned, backtrack)    identical

what the two real fixes cost: allocation events for all subsets of 10 items

  owned Vec, clone per branch                  2046
  backtracking, clone at the leaf              1033
  (1024 subsets each, same answers: true)

three defences, in order of preference:

  1. let the type system do it: `&mut Vec` + push/pop (backtracking), or an
     owned Vec per call that you MOVE into the answer. Neither can alias.
  2. choose / explore / un-choose: one buffer, mutated and restored. The
     leaf still needs `cur.clone()`, because the buffer is reused.
  3. (Go only) cur[:len(cur):len(cur)] forces append to copy, as the
     [:len:len] row above shows. Rust has nothing to clip: a Vec's spare
     capacity is never shared with anyone else.

the rule: a Vec you hand to a recursive call is either BORROWED mutably
(restore it before returning) or MOVED (nobody else can see it).
```

> **Go vs Rust:** The Go bug cannot be written in Rust: recording `&cur[..]` while pushing is error E0499 (real rustc 1.98 output is in the source comment), whatever the capacity. The first row is therefore a hand-built model of Go's slice header (`Rc<RefCell<Vec>>` plus len, cap and Go's append rule); it reproduces Go's row and its 4 distinct results exactly, and its cap 0 versus cap 3 rows (8 versus 4) test that model, not Go itself. The real fixes are an owned `Vec` moved into the answer, and push/pop backtracking; both match Go's clone and backtracking rows and are unaffected by capacity. The allocation counts (2046 versus 1033 for 10 items) come from a counting allocator and depend on std's growth policy, hence "sample".

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

**Rust:**

```rust
use std::collections::HashMap;
use std::hint::black_box;
use std::thread;

// MUTUAL RECURSION: two or more functions that call each other. The base
// case and shrinking argument rules are exactly the same -- you just have to
// check them across the whole cycle rather than in one function.

// --- 1. the textbook pair ---

fn is_even(n: u64) -> bool {
    if n == 0 {
        return true;
    }
    is_odd(n - 1)
}

fn is_odd(n: u64) -> bool {
    if n == 0 {
        return false;
    }
    is_even(n - 1)
}

// The same pair with the call kept OUT of tail position, so the optimiser cannot
// turn it into a loop (`black_box` hides the result from it). `top` is the
// address of a local in the caller; the deepest frame reports how far down the
// stack it is. Same trick as example 9.
fn addr_of_local(marker: &u8) -> usize {
    marker as *const u8 as usize
}

#[inline(never)]
fn is_even_deep(n: u64, top: usize) -> (bool, usize) {
    let marker = 0u8;
    if n == 0 {
        return (true, top - addr_of_local(black_box(&marker)));
    }
    let (odd, used) = black_box(is_odd_deep(n - 1, top));
    (odd, used)
}

#[inline(never)]
fn is_odd_deep(n: u64, top: usize) -> (bool, usize) {
    let marker = 0u8;
    if n == 0 {
        return (false, top - addr_of_local(black_box(&marker)));
    }
    let (even, used) = black_box(is_even_deep(n - 1, top));
    (even, used)
}

// --- 2. where it is actually the right tool: mutually recursive DATA ---
//
// A value is either a scalar, a list of values, or a named group of values.
// The data definition refers to itself through two shapes, so the code does too.
//
// Go needs a `kind string` tag next to three fields, only one of which is
// meaningful. In Rust the tag IS the type: an enum whose variants own exactly
// the data they need, and `match` must cover every variant. `Vec` and `HashMap`
// already put their elements on the heap, so no `Box` is needed for the cycle.

// The scalar's payload is only ever counted, never read -- as in the Go program.
#[allow(dead_code)]
enum Value {
    Scalar(i64),
    List(Vec<Value>),
    Group(HashMap<String, Value>),
}

fn count_scalars(v: &Value) -> usize {
    match v {
        Value::Scalar(_) => 1,
        Value::List(vs) => count_list(vs),
        Value::Group(g) => count_group(g),
    }
}

fn count_list(vs: &[Value]) -> usize {
    vs.iter().map(count_scalars).sum() // back into the other function
}

fn count_group(g: &HashMap<String, Value>) -> usize {
    g.values().map(count_scalars).sum()
}

fn depth_of(v: &Value) -> usize {
    match v {
        Value::Scalar(_) => 1,
        Value::List(vs) => 1 + vs.iter().map(depth_of).max().unwrap_or(0),
        Value::Group(g) => 1 + g.values().map(depth_of).max().unwrap_or(0),
    }
}

fn scalar(n: i64) -> Value {
    Value::Scalar(n)
}

fn list(vs: Vec<Value>) -> Value {
    Value::List(vs)
}

fn group<const N: usize>(entries: [(&str, Value); N]) -> Value {
    Value::Group(
        entries
            .map(|(k, v)| (k.to_string(), v))
            .into_iter()
            .collect(),
    )
}

fn main() {
    println!("1. the textbook pair -- is_even and is_odd call each other:");
    println!();
    for n in [0, 1, 6, 7] {
        println!("   {n}: even={:<5} odd={}", is_even(n), is_odd(n));
    }
    println!();
    println!("   correct, and a terrible way to test parity: it is Theta(n) time for");
    println!("   something `n % 2 == 0` answers in one instruction. What it costs in STACK");
    println!("   is the optimiser's decision:");
    println!();

    // Run only with `--tail-calls`. Real recursion would need a frame per call:
    // 10^9 calls is at least 8 GB of return addresses, and a build WITHOUT -O keeps
    // them all and aborts the process with a stack overflow (macOS then writes a
    // crash report). So the plain pair is not pushed this deep by default.
    if std::env::args().nth(1).as_deref() == Some("--tail-calls") {
        let n = black_box(1_000_000_000);
        println!("   is_even({n}) on the main thread: {}", is_even(n));
        println!("   -> returned: the calls in tail position cost no stack at -O");
    }

    let n = 1_000_000;
    let (parity, used) = thread::Builder::new()
        .stack_size(256 << 20)
        .spawn(move || {
            let marker = 0u8;
            is_even_deep(n, addr_of_local(black_box(&marker)))
        })
        .expect("spawn")
        .join()
        .expect("join");
    println!(
        "   call kept out of tail position, n={n}: {parity}, {used} bytes of stack ({} bytes/frame)",
        used / n as usize
    );
    println!("   -> Theta(n) stack, once the optimiser cannot remove the recursion");
    println!();
    println!("   The plain pair above is another matter: whether its tail calls cost stack");
    println!("   depends on the build. `--tail-calls` runs is_even(1_000_000_000) on the");
    println!("   main thread to find out; it returns at -O but would abort a build without");
    println!("   -O, which is why it is not run by default.");
    println!();
    println!("   Rust makes no promise either way (no guaranteed tail calls), so do not");
    println!("   write code that needs them. Mutual recursion is not the point here --");
    println!("   the DATA below is.");

    println!();
    println!("2. mutually recursive data needs mutually recursive code:");
    println!();

    let doc = group([
        ("id", scalar(1)),
        ("tags", list(vec![scalar(10), scalar(20), scalar(30)])),
        (
            "meta",
            group([
                ("version", scalar(2)),
                (
                    "nested",
                    list(vec![scalar(40), list(vec![scalar(50), scalar(60)])]),
                ),
            ]),
        ),
    ]);

    println!("   scalars in the document: {}", count_scalars(&doc));
    println!("   maximum nesting depth:   {}", depth_of(&doc));
    println!();
    println!("   a value contains lists and groups; lists and groups contain values.");
    println!("   The type definition is mutually recursive, so count_scalars/count_list/");
    println!("   count_group are too. This is what JSON, XML, ASTs and file systems");
    println!("   all look like -- and why recursion is the natural tool for them.");

    println!();
    println!("3. a language wrinkle -- Rust's is the opposite of Go's:");
    println!();
    println!("   `fn` items can call each other in any order, even when they are declared");
    println!("   INSIDE a function body, below the line that uses them. That is the");
    println!("   even/odd below: called first, defined last, and that is fine.");
    println!();
    println!("   LOCAL closures cannot be mutually recursive at all. This does not compile:");
    println!(
        "       let even = |n: u64| ... odd(n - 1) ...;   // error[E0425]: cannot find function `odd`"
    );
    println!("       let odd = |n: u64| ... even(n - 1) ...;");
    println!("   and Go's `var odd func(int) bool` forward declaration has no direct");
    println!("   equivalent: a closure cannot name itself or one defined after it (the");
    println!("   workaround is to hand a `&dyn Fn` in as an argument).");
    println!("   The price of the nested `fn` is that it cannot capture variables");
    println!("   (error[E0434]: can't capture dynamic environment in a fn item); pass the");
    println!("   state as a parameter, as example 8 did.");

    // The nested items, used BEFORE they are declared.
    println!();
    println!(
        "   the nested-fn version works: even(6)={} odd(6)={}",
        even(6),
        odd(6)
    );

    fn even(n: u64) -> bool {
        if n == 0 { true } else { odd(n - 1) }
    }
    fn odd(n: u64) -> bool {
        if n == 0 { false } else { even(n - 1) }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O ex11.rs -o ex11
./ex11
./ex11 --tail-calls   # also run is_even(1_000_000_000) on the main thread (returns at -O; a build without -O would abort)
```

**Rust output:**

```
1. the textbook pair -- is_even and is_odd call each other:

   0: even=true  odd=false
   1: even=false odd=true
   6: even=true  odd=false
   7: even=false odd=true

   correct, and a terrible way to test parity: it is Theta(n) time for
   something `n % 2 == 0` answers in one instruction. What it costs in STACK
   is the optimiser's decision:

   call kept out of tail position, n=1000000: true, 48000032 bytes of stack (48 bytes/frame)
   -> Theta(n) stack, once the optimiser cannot remove the recursion

   The plain pair above is another matter: whether its tail calls cost stack
   depends on the build. `--tail-calls` runs is_even(1_000_000_000) on the
   main thread to find out; it returns at -O but would abort a build without
   -O, which is why it is not run by default.

   Rust makes no promise either way (no guaranteed tail calls), so do not
   write code that needs them. Mutual recursion is not the point here --
   the DATA below is.

2. mutually recursive data needs mutually recursive code:

   scalars in the document: 8
   maximum nesting depth:   5

   a value contains lists and groups; lists and groups contain values.
   The type definition is mutually recursive, so count_scalars/count_list/
   count_group are too. This is what JSON, XML, ASTs and file systems
   all look like -- and why recursion is the natural tool for them.

3. a language wrinkle -- Rust's is the opposite of Go's:

   `fn` items can call each other in any order, even when they are declared
   INSIDE a function body, below the line that uses them. That is the
   even/odd below: called first, defined last, and that is fine.

   LOCAL closures cannot be mutually recursive at all. This does not compile:
       let even = |n: u64| ... odd(n - 1) ...;   // error[E0425]: cannot find function `odd`
       let odd = |n: u64| ... even(n - 1) ...;
   and Go's `var odd func(int) bool` forward declaration has no direct
   equivalent: a closure cannot name itself or one defined after it (the
   workaround is to hand a `&dyn Fn` in as an argument).
   The price of the nested `fn` is that it cannot capture variables
   (error[E0434]: can't capture dynamic environment in a fn item); pass the
   state as a parameter, as example 8 did.

   the nested-fn version works: even(6)=true odd(6)=false
```

> **Go vs Rust:** Sections 1 (table) and 2 (8 scalars, depth 5) match Go's output; the prose differs because Rust's facts do. A `kind string` tag plus three mostly-unused fields becomes an `enum` whose `match` the compiler checks for exhaustiveness. Go's "Theta(n) stack" for `isEven` does not hold at `-O`: the separate `--tail-calls` run (not the output above) called `is_even(1_000_000_000)` on the main thread and it returned, so the optimiser removed the tail calls, whereas the same pair with the call kept out of tail position used 48 bytes per level (48,000,032 bytes for n = 1,000,000, in the output above). A build without `-O` aborts with a stack overflow at that call, which is why it is gated behind the flag. Go's closure-per-frame remark was not ported: Rust's local mutual recursion is a nested `fn`, an ordinary function, and its closure wrinkle is the opposite (E0425, E0434, shown in the output).

**Complexity:** `isEven(n)` is Θ(n) time and Θ(n) stack for something `n%2` answers in one instruction · the document walk is Θ(nodes) time and Θ(nesting depth) stack

---

> Next tier: [🔴 hard](3-hard.md) — backtracking, depth as a design constraint, and a parser.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
