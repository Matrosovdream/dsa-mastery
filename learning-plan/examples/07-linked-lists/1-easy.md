# Step 07 — Linked Lists · 🟢 Easy

Examples **1–6**: the node and `nil`, the order of pointer surgery, three ways to remove the head special case,
and the doubly linked ring.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic — your run should match character for character; **Sample
output:** blocks print timings or memory, which vary by machine and run.

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

## 1. The node, and `nil` as the empty list

`🟢 easy` · *two lines define the structure*

The whole structure is a struct with a pointer to itself, and `nil` is a complete, valid list: `length`
handles it in its loop condition and the recursive version has exactly one base case, `head == nil`.
The honest unit of cost for a list is the **pointer hop**: `prepend` makes none (Θ(1)), `at(i)` makes
exactly i (Θ(i)), and appending without a tail pointer walks the whole list every time, so n appends
cost Θ(n²) hops. The program counts every hop instead of timing anything, and ends with what each node
costs in memory: 16 bytes and one heap allocation.

**Steps:**

1. Define `node`, then `prepend`, `appendTail`, `lengthIter`, `lengthRec` and `at`, with a global hop counter.
2. Build lists from the empty list (`nil`) and run every function on lengths 0, 1 and 5 — no special cases.
3. Read the hop table: `at(i)` costs i, and n appends without a tail cost (n−1)(n−2)/2.

```go
package main

import (
	"fmt"
	"strings"
	"testing"
	"unsafe"
)

// Two lines define the whole structure: a value, and a pointer to the rest.
// A *node that is nil is not "an error" or "no list yet" -- it IS a list, the
// empty one. Every function below except appendTail handles it with no
// special code -- and appendTail needs its branch only because it must find
// a last node, which the empty list does not have.
type node struct {
	val  int
	next *node
}

// hops counts pointer hops: how many times we followed a `next` pointer.
// For a linked list that is the honest unit of work -- each hop is a load
// from wherever the allocator happened to put the next node.
var hops int

// prepend is Theta(1): the new node points at the old head and becomes the head.
// It works on the empty list unchanged: next = nil is exactly right.
func prepend(head *node, v int) *node {
	return &node{val: v, next: head}
}

// appendTail has no tail pointer, so it walks to the last node: Theta(n).
// The empty list is the one case that needs a branch (there is no last node).
func appendTail(head *node, v int) *node {
	n := &node{val: v}
	if head == nil {
		return n
	}
	p := head
	for p.next != nil {
		p = p.next
		hops++
	}
	p.next = n
	return head
}

// lengthIter walks once. nil gives 0 because the loop never runs.
func lengthIter(head *node) int {
	n := 0
	for p := head; p != nil; p = p.next {
		n++
	}
	return n
}

// lengthRec has exactly one base case: the empty list. Nothing else.
func lengthRec(head *node) int {
	if head == nil {
		return 0
	}
	return 1 + lengthRec(head.next)
}

// at(i) follows i pointers. Off the end, it reports false instead of panicking.
func at(head *node, i int) (int, bool) {
	p := head
	for k := 0; k < i; k++ {
		if p == nil {
			return 0, false
		}
		p = p.next
		hops++
	}
	if p == nil {
		return 0, false
	}
	return p.val, true
}

// show prints the canonical form [1 -> 2 -> 3], and [] for nil.
func show(head *node) string {
	var parts []string
	for p := head; p != nil; p = p.next {
		parts = append(parts, fmt.Sprint(p.val))
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

func build(n int) *node { // n prepends of n-1..0 give [0 -> 1 -> ... -> n-1]
	var head *node
	for v := n - 1; v >= 0; v-- {
		head = prepend(head, v)
	}
	return head
}

func main() {
	fmt.Println("1) building: prepend puts each value in FRONT, append at the BACK")
	var a, b *node // two empty lists: the zero value of a pointer
	for v := 1; v <= 5; v++ {
		a = prepend(a, v)
		b = appendTail(b, v)
	}
	fmt.Println("  5 prepends:", show(a))
	fmt.Println("  5 appends: ", show(b))

	fmt.Println()
	fmt.Println("2) length, both ways -- the empty list goes through the same code")
	for _, n := range []int{0, 1, 5} {
		l := build(n)
		fmt.Printf("  %-24s iterative=%d recursive=%d\n", show(l), lengthIter(l), lengthRec(l))
	}

	fmt.Println()
	fmt.Println("3) cost in pointer hops")
	long := build(1000)
	fmt.Printf("  %-10s %6s %6s\n", "at(i)", "value", "hops")
	for _, i := range []int{0, 1, 10, 100, 999} {
		hops = 0
		v, _ := at(long, i)
		fmt.Printf("  %-10s %6d %6d\n", fmt.Sprintf("at(%d)", i), v, hops)
	}
	v, ok := at(build(3), 5)
	fmt.Printf("  at(5) on a 3-node list = (%d, %v): off the end, no panic\n", v, ok)

	fmt.Println()
	fmt.Printf("  %-22s %10s %16s\n", "n appends, no tail", "hops", "(n-1)(n-2)/2")
	for _, n := range []int{10, 100, 1000} {
		hops = 0
		var l *node
		for v := 0; v < n; v++ {
			l = appendTail(l, v)
		}
		fmt.Printf("  %-22d %10d %16d\n", n, hops, (n-1)*(n-2)/2)
	}
	fmt.Println("  appending to a k-node list walks k-1 hops to reach the last node,")
	fmt.Println("  so n appends cost 0+0+1+...+(n-2): quadratic. Example 4 fixes it.")

	fmt.Println()
	fmt.Println("4) what a node and a list cost in memory")
	fmt.Printf("  unsafe.Sizeof(node{})  = %d bytes (an int and a pointer)\n", unsafe.Sizeof(node{}))
	fmt.Printf("  unsafe.Sizeof(&node{}) = %d bytes (the whole list, as its caller holds it)\n", unsafe.Sizeof(&node{}))
	var sink *node
	allocs := testing.AllocsPerRun(10, func() { sink = build(1000) })
	fmt.Printf("  build(1000) allocates %.0f heap objects: one per node, every time\n", allocs)
	_ = sink
}
```

**Output:**

```
1) building: prepend puts each value in FRONT, append at the BACK
  5 prepends: [5 -> 4 -> 3 -> 2 -> 1]
  5 appends:  [1 -> 2 -> 3 -> 4 -> 5]

2) length, both ways -- the empty list goes through the same code
  []                       iterative=0 recursive=0
  [0]                      iterative=1 recursive=1
  [0 -> 1 -> 2 -> 3 -> 4]  iterative=5 recursive=5

3) cost in pointer hops
  at(i)       value   hops
  at(0)           0      0
  at(1)           1      1
  at(10)         10     10
  at(100)       100    100
  at(999)       999    999
  at(5) on a 3-node list = (0, false): off the end, no panic

  n appends, no tail           hops     (n-1)(n-2)/2
  10                             36               36
  100                          4851             4851
  1000                       498501           498501
  appending to a k-node list walks k-1 hops to reach the last node,
  so n appends cost 0+0+1+...+(n-2): quadratic. Example 4 fixes it.

4) what a node and a list cost in memory
  unsafe.Sizeof(node{})  = 16 bytes (an int and a pointer)
  unsafe.Sizeof(&node{}) = 8 bytes (the whole list, as its caller holds it)
  build(1000) allocates 1000 heap objects: one per node, every time
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::mem::size_of;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Counting allocator: Rust's stand-in for testing.AllocsPerRun. Every Box::new
// goes through `alloc`, so counting calls counts nodes.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

// Two lines define the whole structure, as in Go -- but the pointer is spelled
// differently. Go's `next *node` may be nil; Rust has no null, so "maybe a next
// node" is Option, and "a node on the heap that I own" is Box:
//
//   Go:   next *node                  nil       is the empty list
//   Rust: next: Option<Box<Node>>     None      is the empty list
//
// The Box OWNS the node it points at. Every node therefore has exactly one
// owner -- the link before it -- and that one rule shapes every example in
// this lesson. (Dropping a Link drops the whole chain, recursively, one stack
// frame per node: fine for the 1000 nodes here; example 7 is about long ones.)
type Link = Option<Box<Node>>;

struct Node {
    val: i64,
    next: Link,
}

// prepend is Theta(1). `head.take()` MOVES the old list out of `head` (leaving
// None behind) and into the new node: the first sight of single ownership. Go
// copies the pointer and both copies stay valid; Rust moves it, so there is
// never a second owner. Works on the empty list unchanged: next = None.
fn prepend(head: &mut Link, val: i64) {
    *head = Some(Box::new(Node {
        val,
        next: head.take(),
    }));
}

// append_tail has no tail pointer, so it walks to the end: Theta(n).
// `cur` is a &mut Link -- a mutable reference to whichever link we are looking
// at. When it reaches a None, that None IS the end, and we overwrite it.
// Unlike Go's version, the empty list needs no branch: `head` itself is the
// None we overwrite. (Hops counted as in Go: links followed between nodes.)
fn append_tail(head: &mut Link, val: i64, hops: &mut usize) {
    let mut cur = head;
    while let Some(node) = cur {
        if node.next.is_some() {
            *hops += 1; // about to move from this node to the next one
        }
        cur = &mut node.next;
    }
    *cur = Some(Box::new(Node { val, next: None }));
}

// length_iter walks once. None gives 0 because the loop never runs.
fn length_iter(head: &Link) -> usize {
    let mut n = 0;
    let mut p = head;
    while let Some(node) = p {
        n += 1;
        p = &node.next;
    }
    n
}

// length_rec: one match, one base case -- the empty list. Nothing else.
fn length_rec(head: &Link) -> usize {
    match head {
        None => 0,
        Some(node) => 1 + length_rec(&node.next),
    }
}

// at(i) follows i links. Off the end it returns None -- Go's (0, false) as a
// type, so a caller cannot read the 0 without first checking.
fn at(head: &Link, i: usize, hops: &mut usize) -> Option<i64> {
    let mut p = head.as_deref()?;
    for _ in 0..i {
        p = p.next.as_deref()?;
        *hops += 1;
    }
    Some(p.val)
}

// show prints the canonical form [1 -> 2 -> 3], and [] for None.
fn show(head: &Link) -> String {
    let mut parts = Vec::new();
    let mut p = head;
    while let Some(node) = p {
        parts.push(node.val.to_string());
        p = &node.next;
    }
    format!("[{}]", parts.join(" -> "))
}

fn build(n: i64) -> Link {
    // n prepends of n-1..0 give [0 -> 1 -> ... -> n-1]
    let mut head = None;
    for v in (0..n).rev() {
        prepend(&mut head, v);
    }
    head
}

fn main() {
    println!("1) building: prepend puts each value in FRONT, append at the BACK");
    let (mut a, mut b): (Link, Link) = (None, None); // two empty lists
    let mut hops = 0;
    for v in 1..=5 {
        prepend(&mut a, v);
        append_tail(&mut b, v, &mut hops);
    }
    println!("  5 prepends: {}", show(&a));
    println!("  5 appends:  {}", show(&b));

    println!();
    println!("2) length, both ways -- the empty list goes through the same code");
    for n in [0, 1, 5] {
        let l = build(n);
        println!(
            "  {:<24} iterative={} recursive={}",
            show(&l),
            length_iter(&l),
            length_rec(&l)
        );
    }

    println!();
    println!("3) cost in pointer hops");
    let long = build(1000);
    println!("  {:<10} {:>6} {:>6}", "at(i)", "value", "hops");
    for i in [0, 1, 10, 100, 999] {
        let mut hops = 0;
        let v = at(&long, i, &mut hops).expect("i < 1000");
        println!("  {:<10} {:>6} {:>6}", format!("at({i})"), v, hops);
    }
    let r = at(&build(3), 5, &mut 0);
    println!("  at(5) on a 3-node list = {r:?}: off the end, no panic");

    println!();
    println!(
        "  {:<22} {:>10} {:>16}",
        "n appends, no tail", "hops", "(n-1)(n-2)/2"
    );
    for n in [10usize, 100, 1000] {
        let mut hops = 0;
        let mut l: Link = None;
        for v in 0..n as i64 {
            append_tail(&mut l, v, &mut hops);
        }
        println!("  {:<22} {:>10} {:>16}", n, hops, (n - 1) * (n - 2) / 2);
    }
    println!("  appending to a k-node list walks k-1 hops to reach the last node,");
    println!("  so n appends cost 0+0+1+...+(n-2): quadratic. Example 4 fixes it.");

    println!();
    println!("4) what a node and a list cost in memory");
    println!(
        "  size_of::<Node>() = {} bytes (an i64 and a Box)",
        size_of::<Node>()
    );
    println!(
        "  size_of::<Link>() = {} bytes (the whole list, as its caller holds it)",
        size_of::<Link>()
    );
    println!(
        "  size_of::<Box<Node>>() = {}: a Box is never null, so Option uses null for None",
        size_of::<Box<Node>>()
    );
    let before = ALLOCS.load(Relaxed);
    let l = build(1000);
    let allocs = ALLOCS.load(Relaxed) - before;
    println!("  build(1000) allocates {allocs} heap objects: one per node");
    drop(l);
}
```

**Rust output:**

```
1) building: prepend puts each value in FRONT, append at the BACK
  5 prepends: [5 -> 4 -> 3 -> 2 -> 1]
  5 appends:  [1 -> 2 -> 3 -> 4 -> 5]

2) length, both ways -- the empty list goes through the same code
  []                       iterative=0 recursive=0
  [0]                      iterative=1 recursive=1
  [0 -> 1 -> 2 -> 3 -> 4]  iterative=5 recursive=5

3) cost in pointer hops
  at(i)       value   hops
  at(0)           0      0
  at(1)           1      1
  at(10)         10     10
  at(100)       100    100
  at(999)       999    999
  at(5) on a 3-node list = None: off the end, no panic

  n appends, no tail           hops     (n-1)(n-2)/2
  10                             36               36
  100                          4851             4851
  1000                       498501           498501
  appending to a k-node list walks k-1 hops to reach the last node,
  so n appends cost 0+0+1+...+(n-2): quadratic. Example 4 fixes it.

4) what a node and a list cost in memory
  size_of::<Node>() = 16 bytes (an i64 and a Box)
  size_of::<Link>() = 8 bytes (the whole list, as its caller holds it)
  size_of::<Box<Node>>() = 8: a Box is never null, so Option uses null for None
  build(1000) allocates 1000 heap objects: one per node
```

> **Go vs Rust:** Every count is identical (36 / 4,851 / 498,501 hops, 1000 allocations), and `size_of::<Link>()` is 8 like Go's pointer: `Box` is never null, so `Option<Box<Node>>` stores `None` as the null pointer and the empty list costs exactly what `nil` does. The differences are ownership: `prepend` must `head.take()` the old list into the new node (a move, not a pointer copy), `at` returns `Option<i64>` instead of `(0, false)`, and `append_tail` needs no empty-list branch because the `&mut Link` cursor can overwrite `head` itself. There is no GC: dropping a `Link` frees the chain at once.

**Complexity:** `prepend` Θ(1) · `at(i)` Θ(i) hops · `length` Θ(n) (the recursive one also Θ(n) stack) · append without a tail Θ(n), so n appends cost (n−1)(n−2)/2 hops — 498,501 for n = 1000 · memory: 16 bytes and one heap object per node

---

## 2. Wire first, splice second

`🟢 easy` · *the order of two assignments is the whole game*

`insertAfter(p, v)` is two assignments and Θ(1) given the node: `n.next = p.next` (wire the new node to
the rest of the list), then `p.next = n` (splice it in). Swap them and Go gives no panic and no error —
`p.next = n` overwrites the only pointer to the old successor, and `n.next = p.next` then reads `n` back,
so the new node points at itself and every node after it is unreachable. The swapped version is also
Θ(1); it just leaves a one-node cycle behind. The program runs both orders on the same list, prints them
with a `showLimit` that cannot loop forever, and counts reachable nodes with a visited map.

**Steps:**

1. Build `1 -> 2 -> 3 -> 4 -> 5`, keep a pointer to every node, and insert 99 after node 2 in the correct order.
2. Do the same on a fresh list with the two assignments swapped, and print it with `showLimit(head, 12)`.
3. Compare the reachable counts: 6 of 6 versus 3 of 6, with 3, 4 and 5 lost and a cycle — then check the edges.

```go
package main

import (
	"fmt"
	"strings"
)

type node struct {
	val  int
	next *node
}

// insertAfter splices a new node in after p. Two assignments, and the ORDER
// is the whole algorithm:
//
//	n.next = p.next   // 1. wire the new node to the rest of the list FIRST
//	p.next = n        // 2. then splice it in
//
// Step 1 reads p.next while it still points at the old successor. Do step 2
// first and that pointer is gone -- overwritten -- before anyone saved it.
func insertAfter(p *node, v int) *node {
	n := &node{val: v}
	n.next = p.next
	p.next = n
	return n
}

// insertAfterSwapped is the same two lines in the other order. It compiles,
// it does not panic, and it returns normally.
func insertAfterSwapped(p *node, v int) *node {
	n := &node{val: v}
	p.next = n      // p.next is now n -- the old successor is forgotten
	n.next = p.next // ...so this reads n back: n.next = n, a 1-node cycle
	return n
}

func fromSlice(xs []int) *node {
	var head *node
	for i := len(xs) - 1; i >= 0; i-- {
		head = &node{val: xs[i], next: head}
	}
	return head
}

func nth(head *node, i int) *node {
	for ; i > 0; i-- {
		head = head.next
	}
	return head
}

// showLimit prints at most limit nodes, then "..." -- a plain show() would
// never return on a cyclic list.
func showLimit(head *node, limit int) string {
	var parts []string
	p := head
	for ; p != nil && len(parts) < limit; p = p.next {
		parts = append(parts, fmt.Sprint(p.val))
	}
	s := "[" + strings.Join(parts, " -> ")
	if p != nil {
		return s + " ...]"
	}
	return s + "]"
}

// reachable walks from head, remembering every node it has visited, and stops
// at nil or at the first node it has seen before (which means a cycle).
func reachable(head *node) (seen map[*node]bool, cycle bool) {
	seen = map[*node]bool{}
	for p := head; p != nil; p = p.next {
		if seen[p] {
			return seen, true
		}
		seen[p] = true
	}
	return seen, false
}

// all collects every node of a (non-cyclic) list, so we can later ask which
// of them can still be reached.
func all(head *node) []*node {
	var ns []*node
	for p := head; p != nil; p = p.next {
		ns = append(ns, p)
	}
	return ns
}

// report prints how many of the nodes we allocated can still be reached.
func report(head *node, allocated []*node) {
	seen, cycle := reachable(head)
	var lost []string
	for _, n := range allocated {
		if !seen[n] {
			lost = append(lost, fmt.Sprint(n.val))
		}
	}
	fmt.Printf("   reachable %d of %d nodes, lost %d %v, cycle: %s\n",
		len(seen), len(allocated), len(lost), lost, yes(cycle))
}

func main() {
	good := fromSlice([]int{1, 2, 3, 4, 5})
	allocated := all(good)
	n := insertAfter(nth(good, 1), 99) // after the node holding 2
	fmt.Println("correct order (wire, then splice):")
	fmt.Println("  ", showLimit(good, 12))
	report(good, append(allocated, n))

	bad := fromSlice([]int{1, 2, 3, 4, 5})
	allocated = all(bad)
	n = insertAfterSwapped(nth(bad, 1), 99)
	fmt.Println()
	fmt.Println("swapped order (splice, then wire) -- no panic, no error:")
	fmt.Println("  ", showLimit(bad, 12), " <- showLimit stopped after 12 nodes")
	report(bad, append(allocated, n))
	fmt.Printf("   the new node points at itself (n.next == n: %v), and the nodes after it\n", n.next == n)
	fmt.Println("   are unreachable: the GC may reclaim them, but their data is gone, and")
	fmt.Println("   a plain `for p != nil` loop over this list would now never end.")

	fmt.Println()
	fmt.Println("the correct order needs no special cases at the edges:")
	tail := fromSlice([]int{1, 2, 3})
	allocated = all(tail)
	n = insertAfter(nth(tail, 2), 4)
	fmt.Println("   after the last node:", showLimit(tail, 12))
	report(tail, append(allocated, n))
	one := fromSlice([]int{7})
	allocated = all(one)
	n = insertAfter(one, 8)
	fmt.Println("   into a 1-node list: ", showLimit(one, 12))
	report(one, append(allocated, n))
}

func yes(b bool) string {
	if b {
		return "yes"
	}
	return "no"
}
```

**Output:**

```
correct order (wire, then splice):
   [1 -> 2 -> 99 -> 3 -> 4 -> 5]
   reachable 6 of 6 nodes, lost 0 [], cycle: no

swapped order (splice, then wire) -- no panic, no error:
   [1 -> 2 -> 99 -> 99 -> 99 -> 99 -> 99 -> 99 -> 99 -> 99 -> 99 -> 99 ...]  <- showLimit stopped after 12 nodes
   reachable 3 of 6 nodes, lost 3 [3 4 5], cycle: yes
   the new node points at itself (n.next == n: true), and the nodes after it
   are unreachable: the GC may reclaim them, but their data is gone, and
   a plain `for p != nil` loop over this list would now never end.

the correct order needs no special cases at the edges:
   after the last node: [1 -> 2 -> 3 -> 4]
   reachable 4 of 4 nodes, lost 0 [], cycle: no
   into a 1-node list:  [7 -> 8]
   reachable 2 of 2 nodes, lost 0 [], cycle: no
```

**Rust:**

```rust
use std::cell::{Cell, RefCell};

type Link = Option<Box<Node>>;

struct Node {
    val: i64,
    next: Link,
}

// Bookkeeping, so the program can SEE what Go's GC hides: how many nodes were
// made, and which ones were freed. Rust has no mutable globals without a
// wrapper; a thread_local Cell/RefCell is the simplest one.
thread_local! {
    static ALLOCATED: Cell<usize> = const { Cell::new(0) };
    static FREED: RefCell<Vec<i64>> = const { RefCell::new(Vec::new()) };
}

impl Node {
    fn boxed(val: i64) -> Box<Node> {
        ALLOCATED.with(|a| a.set(a.get() + 1));
        Box::new(Node { val, next: None })
    }
}

// Drop runs when a node's owner lets go of it. In Go nothing is freed until a
// GC cycle finds the node unreachable; in Rust it happens on the very line
// that overwrites the last owner.
impl Drop for Node {
    fn drop(&mut self) {
        FREED.with(|f| f.borrow_mut().push(self.val));
    }
}

// insert_after splices a new node in after p. The same two steps as Go,
// in the same order -- but Rust forces the right order on you:
//
//   n.next = p.next;   // Go's step 1, written literally:
//   // error[E0507]: cannot move out of `p.next` which is behind a mutable reference
//
// p.next is OWNED by p, and we only borrowed p. We may not copy the Box (that
// would make two owners of node 3), only MOVE it out -- and moving out of a
// borrow is allowed only if we leave something valid behind. `take()` does
// exactly that: it moves the old successor out and leaves None in p.next.
fn insert_after(p: &mut Node, val: i64) {
    let mut n = Node::boxed(val);
    n.next = p.next.take(); // 1. wire: n now owns the rest of the list
    p.next = Some(n); //       2. splice: p now owns n
}

// insert_after_swapped is Go's swapped order. Its first line compiles:
//
//   p.next = Some(n);       // overwrites p's only link to node 3
//   n.next = p.next.take(); // Go's step 2 -- does not compile:
//   // error[E0382]: assign to part of moved value: `*n`
//
// n was moved into p.next, so there is no `n` any more to wire up, and a
// Box cannot point at itself: the cycle Go builds is unwritable here. What
// DOES happen is on the first line: the old value of p.next -- the Box of
// node 3, which owns 4, which owns 5 -- has lost its owner, so it is dropped
// right there. Same mistake, different failure: Rust frees and truncates.
fn insert_after_swapped(p: &mut Node, val: i64) {
    let n = Node::boxed(val);
    p.next = Some(n);
}

fn from_slice(xs: &[i64]) -> Link {
    let mut head = None;
    for &v in xs.iter().rev() {
        let mut n = Node::boxed(v);
        n.next = head;
        head = Some(n);
    }
    head
}

// nth_mut returns a &mut to the i-th node: the only way to "hold" a node in
// the middle of a Box chain is to borrow it from its owner.
fn nth_mut(head: &mut Link, i: usize) -> &mut Node {
    let mut p = head.as_deref_mut().expect("non-empty");
    for _ in 0..i {
        p = p.next.as_deref_mut().expect("long enough");
    }
    p
}

// show_limit prints at most limit nodes, then "...". Kept for parity with Go;
// a Box chain cannot be cyclic, so plain traversal always ends.
fn show_limit(head: &Link, limit: usize) -> String {
    let mut parts = Vec::new();
    let mut p = head.as_deref();
    while let Some(node) = p {
        if parts.len() == limit {
            break;
        }
        parts.push(node.val.to_string());
        p = node.next.as_deref();
    }
    let s = format!("[{}", parts.join(" -> "));
    if p.is_some() {
        format!("{s} ...]")
    } else {
        format!("{s}]")
    }
}

fn reachable(head: &Link) -> usize {
    let mut n = 0;
    let mut p = head.as_deref();
    while let Some(node) = p {
        n += 1;
        p = node.next.as_deref();
    }
    n
}

fn reset() {
    ALLOCATED.with(|a| a.set(0));
    FREED.with(|f| f.borrow_mut().clear());
}

// report prints how many of the nodes we allocated can still be reached,
// and which ones have already been freed.
fn report(head: &Link) {
    let allocated = ALLOCATED.with(Cell::get);
    let freed: Vec<String> = FREED.with(|f| f.borrow().iter().map(i64::to_string).collect());
    println!(
        "   reachable {} of {} nodes, freed {} [{}], cycle: impossible",
        reachable(head),
        allocated,
        freed.len(),
        freed.join(" ")
    );
}

fn main() {
    {
        reset();
        let mut good = from_slice(&[1, 2, 3, 4, 5]);
        insert_after(nth_mut(&mut good, 1), 99); // after the node holding 2
        println!("correct order (wire, then splice):");
        println!("   {}", show_limit(&good, 12));
        report(&good);
    } // `good` is dropped here, freeing its 6 nodes

    {
        reset();
        let mut bad = from_slice(&[1, 2, 3, 4, 5]);
        insert_after_swapped(nth_mut(&mut bad, 1), 99);
        println!();
        println!("swapped order (splice first) -- the only half of it that compiles:");
        println!("   {}", show_limit(&bad, 12));
        report(&bad);
        println!("   the old successor lost its only owner on the assignment, so it was");
        println!("   dropped there and then: the list is truncated, not cyclic, and the");
        println!("   data in the freed nodes is gone just the same.");
    }

    println!();
    println!("the correct order needs no special cases at the edges:");
    {
        reset();
        let mut tail = from_slice(&[1, 2, 3]);
        insert_after(nth_mut(&mut tail, 2), 4);
        println!("   after the last node: {}", show_limit(&tail, 12));
        report(&tail);
    }
    {
        reset();
        let mut one = from_slice(&[7]);
        insert_after(nth_mut(&mut one, 0), 8);
        println!("   into a 1-node list:  {}", show_limit(&one, 12));
        report(&one);
    }
}
```

**Rust output:**

```
correct order (wire, then splice):
   [1 -> 2 -> 99 -> 3 -> 4 -> 5]
   reachable 6 of 6 nodes, freed 0 [], cycle: impossible

swapped order (splice first) -- the only half of it that compiles:
   [1 -> 2 -> 99]
   reachable 3 of 6 nodes, freed 3 [3 4 5], cycle: impossible
   the old successor lost its only owner on the assignment, so it was
   dropped there and then: the list is truncated, not cyclic, and the
   data in the freed nodes is gone just the same.

the correct order needs no special cases at the edges:
   after the last node: [1 -> 2 -> 3 -> 4]
   reachable 4 of 4 nodes, freed 0 [], cycle: impossible
   into a 1-node list:  [7 -> 8]
   reachable 2 of 2 nodes, freed 0 [], cycle: impossible
```

> **Go vs Rust:** The correct order prints the same lists and counts as Go, but Rust forces it: Go's literal `n.next = p.next` is E0507 (cannot move out of a borrowed `Box`), so the wire step must be `p.next.take()`. Go's swapped order cannot be written at all — after `p.next = Some(n)` the second line is E0382, and a `Box` can never point at itself — so the same mistake fails differently: Go keeps 3 of 6 nodes reachable plus a self-loop and leaves 3 for the GC, while Rust drops nodes 3, 4, 5 on the overwriting assignment (a `Drop` counter shows `freed 3 [3 4 5]` before the list goes out of scope) and leaves a shorter, finite list.

**Complexity:** Θ(1) per insert, given the node — and the swapped order is Θ(1) too: it leaves a one-node cycle and the 3 nodes after the insertion point unreachable, with no error at any step

---

## 3. The sentinel, and the special case it deletes

`🟢 easy` · *one extra node, no head branch*

Without a sentinel, deleting the head is a different code path from deleting anywhere else (the head has
no predecessor), the empty list is another, and the function must return the new head — which callers
forget. One dummy node before everything gives every real node a predecessor, so one loop covers head,
middle, tail and empty, nothing is returned, and because the sentinel is an embedded value the zero value
`var l List` is already a working list. The search is Θ(n) either way; the sentinel costs one extra node
per list. Counters inside both functions show which path each delete took.

**Steps:**

1. Write `deleteValue(head, v) *node` with its two special branches, and `(*List).Delete(v)` with one loop from `&l.sentinel`.
2. Run both on the same six cases (head, middle, tail, missing, only element, empty) and compare the results.
3. Read the branch counters, use `var l List` with no constructor, and watch the forgotten-return-value bug.

```go
package main

import (
	"fmt"
	"strings"
	"unsafe"
)

type node struct {
	val  int
	next *node
}

// Counters, so the program can show which code path each delete took.
var (
	emptyBranch int // no sentinel: "the list is empty" special case ran
	headBranch  int // no sentinel: "the head is the one to delete" special case ran
	predIsSent  int // sentinel: the predecessor of the removed node was the sentinel
)

// deleteValue (no sentinel) removes the first node holding v. The head has no
// predecessor, so it needs its own branch -- and because the head can change,
// the function must RETURN the new head, which the caller must remember to keep.
func deleteValue(head *node, v int) *node {
	if head == nil { // special case 1: nothing to delete from
		emptyBranch++
		return nil
	}
	if head.val == v { // special case 2: the head itself goes
		headBranch++
		return head.next
	}
	for p := head; p.next != nil; p = p.next { // the general case
		if p.next.val == v {
			p.next = p.next.next
			break
		}
	}
	return head
}

// List keeps a sentinel ("dummy") node BEFORE the first real node. It is an
// embedded value, not a pointer, so the zero value `var l List` is already a
// valid empty list: sentinel.next == nil.
type List struct {
	sentinel node
}

func (l *List) PushFront(v int) {
	l.sentinel.next = &node{val: v, next: l.sentinel.next}
}

// Delete (sentinel): every real node now has a predecessor -- the first one's
// is the sentinel -- so one loop handles head, middle, tail and empty. Nothing
// is returned: the list header never moves.
func (l *List) Delete(v int) {
	for p := &l.sentinel; p.next != nil; p = p.next {
		if p.next.val == v {
			if p == &l.sentinel {
				predIsSent++ // counted only; the code below is the same either way
			}
			p.next = p.next.next
			return
		}
	}
}

func fromSlice(xs []int) *node {
	var head *node
	for i := len(xs) - 1; i >= 0; i-- {
		head = &node{val: xs[i], next: head}
	}
	return head
}

func listFrom(xs []int) *List {
	l := &List{}
	l.sentinel.next = fromSlice(xs)
	return l
}

func show(head *node) string {
	var parts []string
	for p := head; p != nil; p = p.next {
		parts = append(parts, fmt.Sprint(p.val))
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

func main() {
	cases := []struct {
		name string
		xs   []int
		v    int
	}{
		{"delete head", []int{1, 2, 3, 4, 5}, 1},
		{"delete middle", []int{1, 2, 3, 4, 5}, 3},
		{"delete tail", []int{1, 2, 3, 4, 5}, 5},
		{"missing value", []int{1, 2, 3, 4, 5}, 9},
		{"only element", []int{7}, 7},
		{"from empty", nil, 1},
	}
	fmt.Printf("%-15s %-24s %-24s %s\n", "case", "no sentinel", "sentinel", "agree")
	for _, c := range cases {
		a := deleteValue(fromSlice(c.xs), c.v)
		l := listFrom(c.xs)
		l.Delete(c.v)
		sa, sb := show(a), show(l.sentinel.next)
		fmt.Printf("%-15s %-24s %-24s %v\n", c.name, sa, sb, sa == sb)
	}

	fmt.Println()
	fmt.Println("which code path ran, over those 6 cases:")
	fmt.Printf("  no sentinel: empty-list branch %d, head branch %d -- two special cases\n", emptyBranch, headBranch)
	fmt.Printf("  sentinel:    0 special branches; the predecessor was the sentinel %d times,\n", predIsSent)
	fmt.Println("               and the same loop body handled it")

	fmt.Println()
	fmt.Println("the zero value works -- no constructor, no nil check:")
	var l List
	l.PushFront(3)
	fmt.Println("  var l List; l.PushFront(3):", show(l.sentinel.next))
	l.Delete(3)
	fmt.Println("  l.Delete(3):               ", show(l.sentinel.next))
	fmt.Printf("  the cost: unsafe.Sizeof(List{}) = %d bytes, one unused node per list\n", unsafe.Sizeof(List{}))

	fmt.Println()
	fmt.Println("the bug the sentinel deletes -- forgetting the returned head:")
	head := fromSlice([]int{1, 2, 3})
	deleteValue(head, 1) // compiles; go vet is silent; the result is thrown away
	fmt.Println("  deleteValue(head, 1) ignored:", show(head), "<- head still points at node 1")
	head = deleteValue(head, 1)
	fmt.Println("  head = deleteValue(head, 1): ", show(head))
}
```

**Output:**

```
case            no sentinel              sentinel                 agree
delete head     [2 -> 3 -> 4 -> 5]       [2 -> 3 -> 4 -> 5]       true
delete middle   [1 -> 2 -> 4 -> 5]       [1 -> 2 -> 4 -> 5]       true
delete tail     [1 -> 2 -> 3 -> 4]       [1 -> 2 -> 3 -> 4]       true
missing value   [1 -> 2 -> 3 -> 4 -> 5]  [1 -> 2 -> 3 -> 4 -> 5]  true
only element    []                       []                       true
from empty      []                       []                       true

which code path ran, over those 6 cases:
  no sentinel: empty-list branch 1, head branch 2 -- two special cases
  sentinel:    0 special branches; the predecessor was the sentinel 2 times,
               and the same loop body handled it

the zero value works -- no constructor, no nil check:
  var l List; l.PushFront(3): [3]
  l.Delete(3):                []
  the cost: unsafe.Sizeof(List{}) = 16 bytes, one unused node per list

the bug the sentinel deletes -- forgetting the returned head:
  deleteValue(head, 1) ignored: [1 -> 2 -> 3] <- head still points at node 1
  head = deleteValue(head, 1):  [2 -> 3]
```

**Rust:**

```rust
use std::mem::size_of;

type Link = Option<Box<Node>>;

// Default gives Node { val: 0, next: None } -- the sentinel's value is never read.
#[derive(Default)]
struct Node {
    val: i64,
    next: Link,
}

// Rust has no package-level mutable counters (without a wrapper), so the
// path counters travel as a &mut argument.
#[derive(Default)]
struct Paths {
    empty_branch: usize,     // no sentinel: "the list is empty" special case ran
    head_branch: usize,      // no sentinel: "the head is the one to delete" special case ran
    pred_is_sentinel: usize, // sentinel: the removed node's predecessor was the sentinel
}

// delete_value (no sentinel) removes the first node holding v. The head has
// no predecessor, so it needs its own branch, and because the head can change
// the function must RETURN the new head. In Rust it also TAKES the list by
// value -- the caller hands over ownership and gets the new list back -- and
// #[must_use] turns "forgot the return value" into a compiler warning.
#[must_use]
fn delete_value(head: Link, v: i64, paths: &mut Paths) -> Link {
    let Some(mut head) = head else {
        paths.empty_branch += 1; // special case 1: nothing to delete from
        return None;
    };
    if head.val == v {
        paths.head_branch += 1; // special case 2: the head itself goes
        return head.next; // moving a field out of a Box we own: fine
    }
    unlink_first_after(&mut head, v, &mut false); // the general case
    Some(head)
}

// unlink_first_after walks from predecessor `p`, and unlinks the first node
// after it that holds v. Both versions share it: that is the point -- the
// sentinel version is ONLY this loop.
//
// `p.next.take()` moves the doomed node out of its predecessor's link (leaving
// None for an instant), and `removed.next` -- the rest of the list -- moves
// into the hole. The removed Box is dropped (freed) at the end of the block.
fn unlink_first_after(mut p: &mut Node, v: i64, pred_was_first: &mut bool) {
    let mut first = true;
    while let Some(next) = p.next.as_ref() {
        if next.val == v {
            *pred_was_first = first;
            let removed = p.next.take().expect("just matched Some");
            p.next = removed.next;
            return;
        }
        p = p.next.as_mut().expect("just matched Some");
        first = false;
    }
}

// List keeps a sentinel node BEFORE the first real node, as an embedded value.
// Rust has no implicit zero value -- `let l: List;` is uninitialised and
// unusable -- but #[derive(Default)] gives List::default(), which is the same
// thing spelled out: a sentinel with next = None, i.e. a valid empty list.
#[derive(Default)]
struct List {
    sentinel: Node,
}

impl List {
    fn push_front(&mut self, val: i64) {
        let next = self.sentinel.next.take();
        self.sentinel.next = Some(Box::new(Node { val, next }));
    }

    // delete (sentinel): every real node has a predecessor, so one loop
    // handles head, middle, tail and empty. Nothing is returned, and `&mut
    // self` means the caller's List is the one that changed.
    fn delete(&mut self, v: i64, paths: &mut Paths) {
        let mut pred_was_sentinel = false;
        unlink_first_after(&mut self.sentinel, v, &mut pred_was_sentinel);
        if pred_was_sentinel {
            paths.pred_is_sentinel += 1; // counted only; the code was the same loop
        }
    }

    fn from_slice(xs: &[i64]) -> List {
        let mut l = List::default();
        l.sentinel.next = from_slice(xs);
        l
    }
}

fn from_slice(xs: &[i64]) -> Link {
    let mut head = None;
    for &val in xs.iter().rev() {
        head = Some(Box::new(Node { val, next: head }));
    }
    head
}

fn show(head: &Link) -> String {
    let mut parts = Vec::new();
    let mut p = head;
    while let Some(node) = p {
        parts.push(node.val.to_string());
        p = &node.next;
    }
    format!("[{}]", parts.join(" -> "))
}

fn main() {
    let cases: [(&str, &[i64], i64); 6] = [
        ("delete head", &[1, 2, 3, 4, 5], 1),
        ("delete middle", &[1, 2, 3, 4, 5], 3),
        ("delete tail", &[1, 2, 3, 4, 5], 5),
        ("missing value", &[1, 2, 3, 4, 5], 9),
        ("only element", &[7], 7),
        ("from empty", &[], 1),
    ];
    let mut paths = Paths::default();
    println!(
        "{:<15} {:<24} {:<24} agree",
        "case", "no sentinel", "sentinel"
    );
    for (name, xs, v) in cases {
        let a = delete_value(from_slice(xs), v, &mut paths);
        let mut l = List::from_slice(xs);
        l.delete(v, &mut paths);
        let (sa, sb) = (show(&a), show(&l.sentinel.next));
        println!("{name:<15} {sa:<24} {sb:<24} {}", sa == sb);
    }

    println!();
    println!("which code path ran, over those 6 cases:");
    println!(
        "  no sentinel: empty-list branch {}, head branch {} -- two special cases",
        paths.empty_branch, paths.head_branch
    );
    println!(
        "  sentinel:    0 special branches; the predecessor was the sentinel {} times,",
        paths.pred_is_sentinel
    );
    println!("               and the same loop body handled it");

    println!();
    println!("no constructor to write -- List::default() is the zero value:");
    let mut l = List::default();
    l.push_front(3);
    println!(
        "  List::default(); l.push_front(3): {}",
        show(&l.sentinel.next)
    );
    l.delete(3, &mut paths);
    println!(
        "  l.delete(3):                      {}",
        show(&l.sentinel.next)
    );
    println!(
        "  the cost: size_of::<List>() = {} bytes, one unused node per list",
        size_of::<List>()
    );

    println!();
    println!("the bug the sentinel deletes -- forgetting the returned head:");
    let head = from_slice(&[1, 2, 3]);
    // Go's bug, written in Rust:
    //
    //   delete_value(head, 1, &mut paths);
    //   // warning: unused return value of `delete_value` that must be used
    //   println!("{}", show(&head));
    //   // error[E0382]: borrow of moved value: `head`
    //
    // The list was MOVED into the call, so the old `head` no longer exists:
    // there is nothing stale left to print. The only version that compiles
    // is the one that keeps the result.
    println!("  ignoring the result: a must_use warning, and `head` is moved (E0382)");
    let head = delete_value(head, 1, &mut paths);
    println!("  let head = delete_value(head, 1): {}", show(&head));
}
```

**Rust output:**

```
case            no sentinel              sentinel                 agree
delete head     [2 -> 3 -> 4 -> 5]       [2 -> 3 -> 4 -> 5]       true
delete middle   [1 -> 2 -> 4 -> 5]       [1 -> 2 -> 4 -> 5]       true
delete tail     [1 -> 2 -> 3 -> 4]       [1 -> 2 -> 3 -> 4]       true
missing value   [1 -> 2 -> 3 -> 4 -> 5]  [1 -> 2 -> 3 -> 4 -> 5]  true
only element    []                       []                       true
from empty      []                       []                       true

which code path ran, over those 6 cases:
  no sentinel: empty-list branch 1, head branch 2 -- two special cases
  sentinel:    0 special branches; the predecessor was the sentinel 2 times,
               and the same loop body handled it

no constructor to write -- List::default() is the zero value:
  List::default(); l.push_front(3): [3]
  l.delete(3):                      []
  the cost: size_of::<List>() = 16 bytes, one unused node per list

the bug the sentinel deletes -- forgetting the returned head:
  ignoring the result: a must_use warning, and `head` is moved (E0382)
  let head = delete_value(head, 1): [2 -> 3]
```

> **Go vs Rust:** The table and the branch counters are identical (1 empty-list branch, 2 head branches, the sentinel was the predecessor 2 times), and `size_of::<List>()` is 16 like `unsafe.Sizeof(List{})`. Rust has no implicit zero value, so `#[derive(Default)]` stands in for `var l List`. The forgotten-return-value bug cannot be reproduced: `delete_value` takes the list by value, so ignoring the result is a `#[must_use]` warning and printing the old `head` afterwards is E0382 (use of a moved value) — where Go silently printed `[1 -> 2 -> 3]`.

**Complexity:** Θ(n) search either way · without a sentinel: 2 special branches (they ran 3 times over the 6 cases) and a returned head the caller can drop · with one: 0 special branches, nothing returned, zero value usable · cost: one extra node (`unsafe.Sizeof(List{})` = 16 bytes) per list

---

## 4. A tail pointer, and why `PopBack` stays Θ(n)

`🟢 easy` · *Θ(n²) → Θ(n), and the one thing a tail cannot fix*

Without a tail pointer every append walks the whole list, so n appends cost Θ(n²) hops in total. Caching
the last node (and a `size` counter) makes `PushBack` Θ(1) and the total Θ(n). But `PopBack` stays Θ(n)
even with the tail pointer: unlinking the tail means making its *predecessor* the new tail, and a singly
linked list cannot go backwards, so it walks from the head to find it. The program counts hops for all
three operations, checks the tail invariants after every single operation, and shows that draining a
list from the back is quadratic again — the sentence that justifies doubly linked lists.

**Steps:**

1. Give `List` a `head`, a `tail` and a `size`, and keep the three in sync in `PushBack`, `PopFront` and `PopBack`.
2. Count hops for n appends with and without the tail, doubling n each row, then drain a 1000-node list from each end.
3. Read the ×4-per-doubling column, the 998-hop `PopBack`, and the invariant count (0 broken).

```go
package main

import "fmt"

type node struct {
	val  int
	next *node
}

// hops counts followed `next` pointers -- the cost unit, as in example 1.
var hops int

// appendNoTail is example 1's append: walk to the last node, every time.
func appendNoTail(head *node, v int) *node {
	n := &node{val: v}
	if head == nil {
		return n
	}
	p := head
	for p.next != nil {
		p = p.next
		hops++
	}
	p.next = n
	return head
}

// List remembers its last node and its length, so neither needs a walk.
type List struct {
	head, tail *node
	size       int
}

// PushBack is Theta(1): the tail pointer IS the walk's answer, cached.
func (l *List) PushBack(v int) {
	n := &node{val: v}
	if l.tail == nil {
		l.head = n
	} else {
		l.tail.next = n
	}
	l.tail = n
	l.size++
}

// PopFront is Theta(1): the head's successor becomes the head.
func (l *List) PopFront() (int, bool) {
	if l.head == nil {
		return 0, false
	}
	n := l.head
	l.head = n.next
	if l.head == nil {
		l.tail = nil // removed the only node
	}
	l.size--
	return n.val, true
}

// PopBack is Theta(n) EVEN WITH the tail pointer. Removing the tail means
// making its predecessor the new tail -- and nothing points backwards, so
// the only way to find the predecessor is to walk from the head.
func (l *List) PopBack() (int, bool) {
	if l.head == nil {
		return 0, false
	}
	if l.head == l.tail {
		return l.PopFront()
	}
	p := l.head
	for p.next != l.tail { // stop one node short of the tail
		p = p.next
		hops++
	}
	v := l.tail.val
	p.next = nil
	l.tail = p
	l.size--
	return v, true
}

// check verifies the three invariants a tail pointer adds; ops counts calls.
var ops, broken int

func (l *List) check() {
	ops++
	count, last := 0, (*node)(nil)
	for p := l.head; p != nil; p = p.next {
		count, last = count+1, p
	}
	ok := count == l.size && l.tail == last && (l.tail == nil || l.tail.next == nil)
	if l.size == 0 {
		ok = ok && l.head == nil && l.tail == nil
	}
	if !ok {
		broken++
	}
}

func filled(n int) *List {
	l := &List{}
	for v := 0; v < n; v++ {
		l.PushBack(v)
		l.check()
	}
	return l
}

func main() {
	fmt.Println("n appends: hops without a tail pointer vs with one")
	fmt.Printf("  %6s %12s %10s %18s\n", "n", "no tail", "with tail", "x per doubling")
	prev, lo, hi := 0, 1e9, 0.0
	for _, n := range []int{100, 200, 400, 800, 1600, 3200} {
		hops = 0
		var head *node
		for v := 0; v < n; v++ {
			head = appendNoTail(head, v)
		}
		without := hops
		hops = 0
		filled(n)
		ratio := "-"
		if prev > 0 {
			r := float64(without) / float64(prev)
			lo, hi = min(lo, r), max(hi, r)
			ratio = fmt.Sprintf("%.2f", r)
		}
		fmt.Printf("  %6d %12d %10d %18s\n", n, without, hops, ratio)
		prev = without
	}
	fmt.Printf("  each doubling of n multiplied the no-tail cost by %.2f to %.2f:\n", lo, hi)
	fmt.Println("  x4 per x2 is the quadratic signature; the tail column never moves")

	const n = 1000
	fmt.Println()
	fmt.Printf("PopBack on a %d-node list, tail pointer and all:\n", n)
	l := filled(n)
	hops = 0
	v, _ := l.PopBack()
	l.check()
	fmt.Printf("  one PopBack removed %d after %d hops (walked to the second-to-last node)\n", v, hops)

	l = filled(n)
	hops = 0
	for l.size > 0 {
		l.PopBack()
		l.check()
	}
	fmt.Printf("  draining all %d by PopBack: %d hops; (n-1)(n-2)/2 = %d -- quadratic again\n",
		n, hops, (n-1)*(n-2)/2)

	l = filled(n)
	hops = 0
	for l.size > 0 {
		l.PopFront()
		l.check()
	}
	fmt.Printf("  draining all %d by PopFront: %d hops\n", n, hops)

	fmt.Println()
	fmt.Printf("invariants (size == count, tail == last node, tail.next == nil,\n")
	fmt.Printf("empty means head == tail == nil): checked after %d ops, broken %d times\n", ops, broken)
	fmt.Println()
	fmt.Println("the tail pointer fixed the back for ADDING, not for REMOVING: to unlink")
	fmt.Println("the tail you need the node before it, and a singly linked list has no")
	fmt.Println("way back. That is the one sentence that justifies a prev pointer (example 6).")
}
```

**Output:**

```
n appends: hops without a tail pointer vs with one
       n      no tail  with tail     x per doubling
     100         4851          0                  -
     200        19701          0               4.06
     400        79401          0               4.03
     800       318801          0               4.02
    1600      1277601          0               4.01
    3200      5115201          0               4.00
  each doubling of n multiplied the no-tail cost by 4.00 to 4.06:
  x4 per x2 is the quadratic signature; the tail column never moves

PopBack on a 1000-node list, tail pointer and all:
  one PopBack removed 999 after 998 hops (walked to the second-to-last node)
  draining all 1000 by PopBack: 498501 hops; (n-1)(n-2)/2 = 498501 -- quadratic again
  draining all 1000 by PopFront: 0 hops

invariants (size == count, tail == last node, tail.next == nil,
empty means head == tail == nil): checked after 11301 ops, broken 0 times

the tail pointer fixed the back for ADDING, not for REMOVING: to unlink
the tail you need the node before it, and a singly linked list has no
way back. That is the one sentence that justifies a prev pointer (example 6).
```

**Rust:**

```rust
use std::ptr;

type Link = Option<Box<Node>>;

struct Node {
    val: i64,
    next: Link,
}

// append_no_tail is example 1's append: walk to the last node, every time.
fn append_no_tail(head: &mut Link, val: i64, hops: &mut usize) {
    let mut cur = head;
    while let Some(node) = cur {
        if node.next.is_some() {
            *hops += 1;
        }
        cur = &mut node.next;
    }
    *cur = Some(Box::new(Node { val, next: None }));
}

// List remembers its last node and its length -- and here safe Rust says no.
//
// Every node is OWNED by the link before it. A tail pointer is a second
// pointer to the last node, kept beside the chain that owns it. As a
// reference it would be `tail: &'a mut Node` borrowed from `head` -- and a
// struct cannot hold a &mut into its own field (while it exists, `head` is
// frozen: no push, no pop). As `Rc<RefCell<_>>` it would work, at the price
// of a refcount and a runtime borrow flag on every node (example 6 pays it).
//
// So this is the one example in the lesson that uses `unsafe`: `tail` is a
// raw pointer, which the borrow checker does not track. That moves the proof
// from the compiler to us, written down as the invariant below and checked by
// `check()` after every operation:
//
//   INVARIANT: tail is null iff head is None; otherwise tail points at the
//   last node of the chain owned through `head`, and it was derived AFTER
//   that node's Box was put in its final place (see pop_front for why).
//
// What you would actually write: VecDeque (both ends Theta(1), no unsafe), or
// the index arena of example 13.
struct List {
    head: Link,
    tail: *mut Node,
    size: usize,
}

impl List {
    fn new() -> Self {
        List {
            head: None,
            tail: ptr::null_mut(),
            size: 0,
        }
    }

    // push_back is Theta(1): the tail pointer IS the walk's answer, cached.
    fn push_back(&mut self, val: i64) {
        let slot: &mut Link = if self.tail.is_null() {
            &mut self.head
        } else {
            // SAFETY: tail is non-null, so by the invariant it points at the
            // last node, which is alive (owned through self.head) and not
            // borrowed elsewhere: we hold &mut self, and no reference into
            // the chain outlives any method.
            unsafe { &mut (*self.tail).next }
        };
        *slot = Some(Box::new(Node { val, next: None }));
        // Derive the new tail from the Box in its final place.
        self.tail = ptr::from_mut(slot.as_deref_mut().expect("just stored"));
        self.size += 1;
    }

    // pop_front is Theta(1): the head's successor becomes the head.
    fn pop_front(&mut self) -> Option<i64> {
        let mut old = self.head.take()?;
        self.head = old.next.take();
        self.size -= 1;
        // If the list is now down to 0 or 1 nodes, the tail is (or was) the
        // node whose Box just moved into `head`. Moving a Box through a
        // function like take() re-asserts its uniqueness, and under Stacked
        // Borrows (the aliasing model Miri checks) that invalidates raw
        // pointers derived from it earlier -- so we derive the tail again
        // instead of trusting the old pointer.
        if self.size <= 1 {
            self.tail = self
                .head
                .as_deref_mut()
                .map_or(ptr::null_mut(), ptr::from_mut);
        }
        Some(old.val)
    }

    // pop_back is Theta(n) EVEN WITH the tail pointer: removing the tail means
    // making its predecessor the new tail, and nothing points backwards. No
    // unsafe needed here at all -- the walk is ordinary safe borrowing -- and
    // the tail pointer cannot help, because it points one node too far.
    fn pop_back(&mut self, hops: &mut usize) -> Option<i64> {
        if self.size <= 1 {
            return self.pop_front();
        }
        let mut p: &mut Node = self.head.as_deref_mut()?;
        // stop one node short of the tail: at the node whose next has no next
        while p.next.as_ref().is_some_and(|n| n.next.is_some()) {
            p = p.next.as_deref_mut().expect("checked above");
            *hops += 1;
        }
        let last = p.next.take().expect("size >= 2: p has a successor");
        self.tail = ptr::from_mut(p);
        self.size -= 1;
        Some(last.val)
    }

    // check verifies the invariant with safe code only: walk the chain, then
    // compare addresses (comparing raw pointers never dereferences them).
    fn check(&self, stats: &mut Stats) {
        stats.ops += 1;
        let (mut count, mut last): (usize, *const Node) = (0, ptr::null());
        let mut p = self.head.as_deref();
        while let Some(node) = p {
            count += 1;
            last = node;
            p = node.next.as_deref();
        }
        let mut ok = count == self.size && ptr::eq(last, self.tail);
        if self.size == 0 {
            ok = ok && self.head.is_none() && self.tail.is_null();
        }
        if !ok {
            stats.broken += 1;
        }
    }
}

// Dropping a Box chain recurses once per node (Box drops its Node, which
// drops its next Box, ...). An iterative Drop frees any length in O(1) stack.
impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.head.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
        }
    }
}

#[derive(Default)]
struct Stats {
    ops: usize,
    broken: usize,
}

fn filled(n: i64, stats: &mut Stats) -> List {
    let mut l = List::new();
    for v in 0..n {
        l.push_back(v);
        l.check(stats);
    }
    l
}

fn main() {
    let mut stats = Stats::default();
    println!("n appends: hops without a tail pointer vs with one");
    println!(
        "  {:>6} {:>12} {:>10} {:>18}",
        "n", "no tail", "with tail", "x per doubling"
    );
    let (mut prev, mut lo, mut hi) = (0usize, f64::MAX, 0.0f64);
    for n in [100, 200, 400, 800, 1600, 3200] {
        let mut without = 0;
        let mut head: Link = None;
        for v in 0..n {
            append_no_tail(&mut head, v, &mut without);
        }
        drop(head); // a plain chain of <= 3200 nodes: the recursive drop is fine
        let with_tail = 0; // push_back has no loop: there is nothing to count
        filled(n, &mut stats);
        let mut ratio = "-".to_string();
        if prev > 0 {
            let r = without as f64 / prev as f64;
            (lo, hi) = (lo.min(r), hi.max(r));
            ratio = format!("{r:.2}");
        }
        println!("  {n:>6} {without:>12} {with_tail:>10} {ratio:>18}");
        prev = without;
    }
    println!("  each doubling of n multiplied the no-tail cost by {lo:.2} to {hi:.2}:");
    println!("  x4 per x2 is the quadratic signature; the tail column never moves");

    const N: usize = 1000;
    println!();
    println!("PopBack on a {N}-node list, tail pointer and all:");
    let mut l = filled(N as i64, &mut stats);
    let mut hops = 0;
    let v = l.pop_back(&mut hops).expect("non-empty");
    l.check(&mut stats);
    println!("  one PopBack removed {v} after {hops} hops (walked to the second-to-last node)");

    let mut l = filled(N as i64, &mut stats);
    let mut hops = 0;
    while l.pop_back(&mut hops).is_some() {
        l.check(&mut stats);
    }
    println!(
        "  draining all {N} by PopBack: {hops} hops; (n-1)(n-2)/2 = {} -- quadratic again",
        (N - 1) * (N - 2) / 2
    );

    let mut l = filled(N as i64, &mut stats);
    let hops = 0; // pop_front has no loop
    while l.pop_front().is_some() {
        l.check(&mut stats);
    }
    println!("  draining all {N} by PopFront: {hops} hops");

    println!();
    println!("invariants (size == count, tail == address of the last node,");
    println!(
        "empty means head == None and tail null): checked after {} ops, broken {} times",
        stats.ops, stats.broken
    );
    println!();
    println!("the tail pointer fixed the back for ADDING, not for REMOVING: to unlink");
    println!("the tail you need the node before it, and a singly linked list has no");
    println!("way back. That is the one sentence that justifies a prev pointer (example 6).");
}
```

**Rust output:**

```
n appends: hops without a tail pointer vs with one
       n      no tail  with tail     x per doubling
     100         4851          0                  -
     200        19701          0               4.06
     400        79401          0               4.03
     800       318801          0               4.02
    1600      1277601          0               4.01
    3200      5115201          0               4.00
  each doubling of n multiplied the no-tail cost by 4.00 to 4.06:
  x4 per x2 is the quadratic signature; the tail column never moves

PopBack on a 1000-node list, tail pointer and all:
  one PopBack removed 999 after 998 hops (walked to the second-to-last node)
  draining all 1000 by PopBack: 498501 hops; (n-1)(n-2)/2 = 498501 -- quadratic again
  draining all 1000 by PopFront: 0 hops

invariants (size == count, tail == address of the last node,
empty means head == None and tail null): checked after 11301 ops, broken 0 times

the tail pointer fixed the back for ADDING, not for REMOVING: to unlink
the tail you need the node before it, and a singly linked list has no
way back. That is the one sentence that justifies a prev pointer (example 6).
```

> **Go vs Rust:** All counts are identical (5,115,201 vs 0 hops at n = 3200; 998 hops for one `PopBack`; 498,501 to drain; 11,301 ops checked, 0 broken). The difference is the escape hatch: a tail pointer is a second pointer to a node the chain already owns, which a safe `&mut` field cannot be, so `tail` is a raw `*mut Node` and `push_back` dereferences it in this lesson's only `unsafe` block, with the invariant in a `// SAFETY:` comment and a safe `check()` that compares addresses after every op (Miri is not installed here, so the aliasing argument in `pop_front` is read, not run). In practice you would use `VecDeque`, or the index arena of example 13.

**Complexity:** append Θ(1) with a tail, Θ(n) without (n = 3200: 5,115,201 hops vs 0) · `PopFront` Θ(1) · `PopBack` **Θ(n) even with a tail** (998 hops on 1000 nodes; draining by `PopBack` costs 498,501) · `size` Θ(1) with the counter

---

## 5. Pointer-to-pointer: one loop for head and middle

`🟢 easy` · *`pp` points at the link, not the node*

`pp := &head; for *pp != nil { ... pp = &(*pp).next }` — `pp` is not a node but the address of *whichever
link reaches the current node*: first the variable `head`, later some node's `next` field. So
`*pp = (*pp).next` unlinks the head and a middle node with the same line — no `prev`, no sentinel, no
allocation, Θ(n) time and Θ(1) space. The single trap: after unlinking, do not advance, because `*pp` is
already the next node. The program runs `removeAll` three correct ways and one buggy way against an
obviously correct slice-based answer, then counts heap allocations to confirm all three allocate nothing.

**Steps:**

1. Write `removeAll` with `prev` (a head branch), with a sentinel, and with `pp **node` — plus the version that advances after unlinking.
2. Run all four on six inputs, including all-3s, consecutive 3s, and a 3 only at the tail, and compare each result with the slice answer.
3. Read which inputs break the buggy version (exactly the ones with two 3s in a row), the if-counts, and the allocation counts.

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
	"strings"
)

type node struct {
	val  int
	next *node
}

// removeAllPrev tracks the previous node. The head has no previous node, so
// unlinking it is a different line of code: two ifs.
func removeAllPrev(head *node, v int) *node {
	var prev *node
	for cur := head; cur != nil; cur = cur.next {
		if cur.val == v {
			if prev == nil {
				head = cur.next // the head special case
			} else {
				prev.next = cur.next
			}
		} else {
			prev = cur
		}
	}
	return head
}

// removeAllSentinel gives the head a predecessor: one if, one extra node.
func removeAllSentinel(head *node, v int) *node {
	dummy := node{next: head}
	p := &dummy
	for p.next != nil {
		if p.next.val == v {
			p.next = p.next.next // unlink; p stays, its new next is unchecked
		} else {
			p = p.next
		}
	}
	return dummy.next
}

// removeAllPP: pp is not a node, it is the address of a LINK -- first &head,
// later &someNode.next. Writing *pp rewires whichever link reaches the current
// node, so the head and a middle node are unlinked by the same line. One if,
// no prev, no extra node.
func removeAllPP(head *node, v int) *node {
	pp := &head
	for *pp != nil {
		if (*pp).val == v {
			*pp = (*pp).next // unlink; do NOT advance: *pp is already the next node
		} else {
			pp = &(*pp).next // advance to the next link
		}
	}
	return head
}

// removeAllPPBuggy advances after unlinking too, so the node that slid into
// *pp is never examined. (The nil check only stops it panicking at the end.)
func removeAllPPBuggy(head *node, v int) *node {
	pp := &head
	for *pp != nil {
		if (*pp).val == v {
			*pp = (*pp).next
		}
		if *pp != nil {
			pp = &(*pp).next // BUG: runs even right after an unlink
		}
	}
	return head
}

func fromSlice(xs []int) *node {
	var head *node
	for i := len(xs) - 1; i >= 0; i-- {
		head = &node{val: xs[i], next: head}
	}
	return head
}

func show(head *node) string {
	var parts []string
	for p := head; p != nil; p = p.next {
		parts = append(parts, fmt.Sprint(p.val))
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

func length(head *node) int {
	n := 0
	for ; head != nil; head = head.next {
		n++
	}
	return n
}

// want is the obviously correct answer, computed on a slice.
func want(xs []int, v int) string {
	return show(fromSlice(slices.DeleteFunc(slices.Clone(xs), func(x int) bool { return x == v })))
}

func main() {
	const v = 3
	cases := [][]int{{}, {3}, {3, 3, 3}, {3, 1, 3, 3, 2, 3}, {1, 2}, {1, 3}}
	fns := []func(*node, int) *node{removeAllPrev, removeAllSentinel, removeAllPP, removeAllPPBuggy}

	fmt.Printf("removeAll(list, %d):\n", v)
	fmt.Printf("  %-28s %-12s %-12s %-12s %s\n", "input", "prev", "sentinel", "ptr-to-ptr", "buggy ptr-to-ptr")
	wrong, pairs, same := 0, 0, true
	for _, xs := range cases {
		hasPair := false // does the input contain v twice in a row?
		for i := 1; i < len(xs); i++ {
			hasPair = hasPair || (xs[i-1] == v && xs[i] == v)
		}
		if hasPair {
			pairs++
		}
		buggyWrong := false
		row := []string{}
		for i, f := range fns {
			got := show(f(fromSlice(xs), v))
			if got != want(xs, v) {
				if i < 3 {
					got += " WRONG?!" // never expected: a correct version disagreed
				} else {
					got += "  <- WRONG"
					wrong++
					buggyWrong = true
				}
			}
			row = append(row, got)
		}
		same = same && buggyWrong == hasPair
		fmt.Printf("  %-28s %-12s %-12s %-12s %s\n", show(fromSlice(xs)), row[0], row[1], row[2], row[3])
	}
	fmt.Printf("  the buggy version is wrong on %d of %d inputs; %d inputs have two 3s in a row;\n",
		wrong, len(cases), pairs)
	fmt.Printf("  wrong exactly on those: %v -- the node that slides into *pp is skipped\n", same)

	fmt.Println()
	fmt.Println("what each correct version needs (ifs counted in the source above):")
	fmt.Printf("  %-12s %4s  %s\n", "prev", "2", "a head special case, and a prev variable")
	fmt.Printf("  %-12s %4s  %s\n", "sentinel", "1", "an extra (dummy) node")
	fmt.Printf("  %-12s %4s  %s\n", "ptr-to-ptr", "1", "nothing extra: pp is the address of a link")

	fmt.Println()
	const n = 10000
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i % 4 // 0 1 2 3 0 1 2 3 ...: every fourth value is a 3
	}
	fmt.Printf("heap allocations while removing all 3s from a %d-node list:\n", n)
	var ms runtime.MemStats
	for i, name := range []string{"prev", "sentinel", "ptr-to-ptr"} {
		head := fromSlice(xs)
		runtime.ReadMemStats(&ms)
		before := ms.Mallocs
		head = fns[i](head, v)
		runtime.ReadMemStats(&ms)
		fmt.Printf("  %-12s %d allocations, %d nodes left\n", name, ms.Mallocs-before, length(head))
	}
}
```

**Output:**

```
removeAll(list, 3):
  input                        prev         sentinel     ptr-to-ptr   buggy ptr-to-ptr
  []                           []           []           []           []
  [3]                          []           []           []           []
  [3 -> 3 -> 3]                []           []           []           [3]  <- WRONG
  [3 -> 1 -> 3 -> 3 -> 2 -> 3] [1 -> 2]     [1 -> 2]     [1 -> 2]     [1 -> 3 -> 2]  <- WRONG
  [1 -> 2]                     [1 -> 2]     [1 -> 2]     [1 -> 2]     [1 -> 2]
  [1 -> 3]                     [1]          [1]          [1]          [1]
  the buggy version is wrong on 2 of 6 inputs; 2 inputs have two 3s in a row;
  wrong exactly on those: true -- the node that slides into *pp is skipped

what each correct version needs (ifs counted in the source above):
  prev            2  a head special case, and a prev variable
  sentinel        1  an extra (dummy) node
  ptr-to-ptr      1  nothing extra: pp is the address of a link

heap allocations while removing all 3s from a 10000-node list:
  prev         0 allocations, 7500 nodes left
  sentinel     0 allocations, 7500 nodes left
  ptr-to-ptr   0 allocations, 7500 nodes left
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::LinkedList;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Counting allocator: the stand-in for runtime.MemStats.Mallocs.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

type Link = Option<Box<Node>>;

struct Node {
    val: i64,
    next: Link,
}

// remove_all_prev: Go's version holds `prev` and `cur` at once -- two pointers
// into the same chain, one of them used to write. Safe Rust will not let a
// &mut prev and a &cur coexist, so "prev" here is the node we stand on while
// we LOOK at its successor. The head still has no predecessor, so it still
// needs its own code: a loop of its own before the general case.
fn remove_all_prev(head: &mut Link, v: i64) {
    while head.as_ref().is_some_and(|n| n.val == v) {
        *head = head.take().and_then(|n| n.next); // the head special case
    }
    let Some(mut prev) = head.as_deref_mut() else {
        return;
    };
    while let Some(cur) = prev.next.as_ref() {
        if cur.val == v {
            prev.next = prev.next.take().and_then(|n| n.next);
        } else {
            prev = prev.next.as_deref_mut().expect("checked Some");
        }
    }
}

// remove_all_sentinel gives the head a predecessor: one if, one extra node.
// The dummy takes ownership of the whole list and hands it back at the end.
fn remove_all_sentinel(head: &mut Link, v: i64) {
    let mut dummy = Node {
        val: 0,
        next: head.take(),
    };
    let mut p = &mut dummy;
    while let Some(next) = p.next.as_ref() {
        if next.val == v {
            p.next = p.next.take().and_then(|n| n.next); // unlink; p stays
        } else {
            p = p.next.as_deref_mut().expect("checked Some");
        }
    }
    *head = dummy.next;
}

// remove_all_cursor: `&mut Option<Box<Node>>` IS Go's `**node`. `cur` is a
// mutable reference to a LINK -- first `head` itself, later some node's
// `next` -- so `*cur = ...` rewires whichever link reaches the current node.
//
// The natural way to write it is the borrow checker's most famous refusal:
//
//   while let Some(node) = cur {
//       if node.val == v { *cur = node.next.take(); } else { cur = &mut node.next; }
//   }
//   // error[E0506]: cannot assign to `*cur` because it is borrowed
//
// `node` borrows *cur, and because the else-branch keeps that borrow alive
// (it becomes the new `cur`), the checker treats it as alive in the if-branch
// too. Splitting the two cases into separate match arms (the guard decides)
// compiles on rustc 1.98.1: each arm creates its own borrow of *cur, so the
// one that becomes the new `cur` no longer covers the arm that assigns *cur.
fn remove_all_cursor(head: &mut Link, v: i64) {
    let mut cur = head;
    loop {
        match cur {
            None => break,
            Some(node) if node.val == v => *cur = node.next.take(), // unlink, do NOT advance
            Some(node) => cur = &mut node.next,                     // advance to the next link
        }
    }
}

// remove_all_cursor_buggy advances after unlinking too. `if let` (not unwrap)
// keeps it from panicking at the end, so the bug shows up as a wrong answer.
fn remove_all_cursor_buggy(head: &mut Link, v: i64) {
    let mut cur = head;
    while cur.is_some() {
        if cur.as_ref().is_some_and(|n| n.val == v) {
            *cur = cur.take().and_then(|n| n.next);
        }
        if let Some(node) = cur {
            cur = &mut node.next; // BUG: runs even right after an unlink
        }
    }
}

fn from_slice(xs: &[i64]) -> Link {
    let mut head = None;
    for &val in xs.iter().rev() {
        head = Some(Box::new(Node { val, next: head }));
    }
    head
}

fn show_iter<'a>(it: impl Iterator<Item = &'a i64>) -> String {
    let parts: Vec<String> = it.map(i64::to_string).collect();
    format!("[{}]", parts.join(" -> "))
}

fn iter(head: &Link) -> impl Iterator<Item = &i64> {
    std::iter::successors(head.as_deref(), |n| n.next.as_deref()).map(|n| &n.val)
}

fn show(head: &Link) -> String {
    show_iter(iter(head))
}

// free drops a chain iteratively: dropping a Link directly recurses once per
// node, which is fine for these sizes but has no place in code that may meet
// a long list (example 7 measures where it overflows).
fn free(mut head: Link) {
    while let Some(mut n) = head {
        head = n.next.take();
    }
}

fn main() {
    const V: i64 = 3;
    let cases: [&[i64]; 6] = [&[], &[3], &[3, 3, 3], &[3, 1, 3, 3, 2, 3], &[1, 2], &[1, 3]];
    let fns: [fn(&mut Link, i64); 4] = [
        remove_all_prev,
        remove_all_sentinel,
        remove_all_cursor,
        remove_all_cursor_buggy,
    ];

    println!("remove_all(list, {V}):");
    println!(
        "  {:<28} {:<12} {:<12} {:<12} buggy cursor",
        "input", "prev", "sentinel", "cursor"
    );
    let (mut wrong, mut pairs, mut same) = (0, 0, true);
    for xs in cases {
        // the obviously correct answer, computed on the slice
        let want = show_iter(xs.iter().filter(|&&x| x != V));
        let has_pair = xs.windows(2).any(|w| w == [V, V]);
        pairs += usize::from(has_pair);
        let mut buggy_wrong = false;
        let mut row = Vec::new();
        for (i, f) in fns.iter().enumerate() {
            let mut l = from_slice(xs);
            f(&mut l, V);
            let mut got = show(&l);
            if got != want {
                if i < 3 {
                    got += " WRONG?!"; // never expected: a correct version disagreed
                } else {
                    got += "  <- WRONG";
                    wrong += 1;
                    buggy_wrong = true;
                }
            }
            row.push(got);
        }
        same = same && buggy_wrong == has_pair;
        println!(
            "  {:<28} {:<12} {:<12} {:<12} {}",
            show_iter(xs.iter()),
            row[0],
            row[1],
            row[2],
            row[3]
        );
    }
    println!(
        "  the buggy version is wrong on {wrong} of {} inputs; {pairs} inputs have two 3s in a row;",
        cases.len()
    );
    println!("  wrong exactly on those: {same} -- the node that slides into *cur is skipped");

    println!();
    println!(
        "what each correct version needs (conditionals besides the main loop, counted in the source):"
    );
    println!(
        "  {:<12} {:>4}  a head special case (its own while loop + let-else), and a prev",
        "prev", "3"
    );
    println!("  {:<12} {:>4}  an extra (dummy) node", "sentinel", "1");
    println!(
        "  {:<12} {:>4}  nothing extra: one match on a &mut to a link",
        "cursor", "1"
    );

    println!();
    const N: i64 = 10000;
    let xs: Vec<i64> = (0..N).map(|i| i % 4).collect(); // every fourth value is a 3
    println!("heap allocations while removing all 3s from a {N}-node list:");
    for (name, f) in ["prev", "sentinel", "cursor"].iter().zip(fns) {
        let mut head = from_slice(&xs);
        let before = ALLOCS.load(Relaxed);
        f(&mut head, V);
        let allocs = ALLOCS.load(Relaxed) - before;
        println!(
            "  {:<12} {} allocations, {} nodes left",
            name,
            allocs,
            iter(&head).count()
        );
        free(head);
    }

    println!();
    println!("std's own list does this filter with a stable method (since Rust 1.87):");
    let mut l: LinkedList<i64> = [3, 1, 3, 3, 2, 3].into_iter().collect();
    let before = show_iter(l.iter());
    let removed = l.extract_if(|x| *x == V).count();
    println!(
        "  {before}.extract_if(|x| *x == 3): removed {removed}, left {}",
        show_iter(l.iter())
    );
}
```

**Rust output:**

```
remove_all(list, 3):
  input                        prev         sentinel     cursor       buggy cursor
  []                           []           []           []           []
  [3]                          []           []           []           []
  [3 -> 3 -> 3]                []           []           []           [3]  <- WRONG
  [3 -> 1 -> 3 -> 3 -> 2 -> 3] [1 -> 2]     [1 -> 2]     [1 -> 2]     [1 -> 3 -> 2]  <- WRONG
  [1 -> 2]                     [1 -> 2]     [1 -> 2]     [1 -> 2]     [1 -> 2]
  [1 -> 3]                     [1]          [1]          [1]          [1]
  the buggy version is wrong on 2 of 6 inputs; 2 inputs have two 3s in a row;
  wrong exactly on those: true -- the node that slides into *cur is skipped

what each correct version needs (conditionals besides the main loop, counted in the source):
  prev            3  a head special case (its own while loop + let-else), and a prev
  sentinel        1  an extra (dummy) node
  cursor          1  nothing extra: one match on a &mut to a link

heap allocations while removing all 3s from a 10000-node list:
  prev         0 allocations, 7500 nodes left
  sentinel     0 allocations, 7500 nodes left
  cursor       0 allocations, 7500 nodes left

std's own list does this filter with a stable method (since Rust 1.87):
  [3 -> 1 -> 3 -> 3 -> 2 -> 3].extract_if(|x| *x == 3): removed 4, left [1 -> 2]
```

> **Go vs Rust:** `&mut Option<Box<Node>>` is Go's `**node`, and the results table, the buggy version's 2 wrong answers and the 0 allocations on 10,000 nodes are identical. The escape hatch is the cursor itself: the natural `while let Some(node) = cur { if .. { *cur = .. } else { cur = &mut node.next } }` is E0506, while the same logic as a three-arm `match` with a guard compiles on rustc 1.98.1, which is what is used here. The `prev` version cannot hold `prev` and `cur` at once (two references into one chain, one of them mutable), so it stands on the predecessor and needs 3 conditionals where Go needs 2 `if`s. std's `LinkedList::extract_if` (stable since 1.87, read in the source) does the same filter.

**Complexity:** Θ(n) time, Θ(1) space and 0 heap allocations for all three correct versions (measured on 10,000 nodes) · pointer-to-pointer: one `if`, no head branch, no sentinel node · the advance-after-unlink bug skips the node that slides into `*pp`, so it fails exactly on runs of the removed value

---

## 6. The doubly linked ring

`🟢 easy` · *prev buys Θ(1) removal of a node you hold*

Add a `prev` pointer and wire the sentinel into a **ring** — `root.next` is the first node, `root.prev`
the last, and the empty list is the root pointing at itself — and no link is ever `nil` while a node is in
the list. Insert and remove become four (and two) unconditional pointer writes: Θ(1), no `if` anywhere, and
the predecessor that example 4 had to walk for is simply `n.prev`. A `*dnode` is then a **stable handle**:
it stays valid across every other insertion and deletion, which a slice index does not. The program shows
the ring both ways, `MoveToFront` and `Remove` by handle, and then 1,000 seeded random operations run on
the list and on a slice side by side.

**Steps:**

1. Build `DList` with an embedded `root dnode`, and write `insertAfter`, `unlink`, `Remove`, `MoveToFront`, `PushFront`, `PushBack`.
2. Keep a handle to every node, run 1,000 PCG-random removes and inserts, and re-read every handle after every op.
3. Compare: 0 broken handles, while the 10 slice indices saved at the start no longer hold their values.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
	"unsafe"
)

// dnode has a pointer each way. The sentinel `root` closes the list into a
// RING: root.next is the first node, root.prev the last, and an empty list is
// root pointing at itself both ways. No pointer is ever nil while a node is
// in the list, so insert and remove never check for one.
type dnode struct {
	val        int
	prev, next *dnode
}

type DList struct {
	root dnode
	len  int
}

// newDList wires the empty ring. (container/list does this lazily on first
// use, so that its zero value works; here we keep it explicit.)
func newDList() *DList {
	l := &DList{}
	l.root.next, l.root.prev = &l.root, &l.root
	return l
}

// insertAfter: four pointer writes, no ifs -- the first and last node are not
// special, because their neighbour is simply the root.
func (l *DList) insertAfter(at *dnode, v int) *dnode {
	n := &dnode{val: v, prev: at, next: at.next} // 1, 2
	at.next.prev = n                             // 3
	at.next = n                                  // 4
	l.len++
	return n
}

func (l *DList) PushFront(v int) *dnode { return l.insertAfter(&l.root, v) }
func (l *DList) PushBack(v int) *dnode  { return l.insertAfter(l.root.prev, v) }

// unlink: two writes, no ifs. This is the Theta(1) delete a singly linked
// list cannot do -- the predecessor is n.prev, no walk needed.
func (l *DList) unlink(n *dnode) {
	n.prev.next = n.next
	n.next.prev = n.prev
	l.len--
}

// Remove also nils the node's own links, as container/list does, so a stale
// handle cannot be used to walk back into the list.
func (l *DList) Remove(n *dnode) {
	l.unlink(n)
	n.next, n.prev = nil, nil
}

// MoveToFront relinks an existing node: no allocation, no ifs.
func (l *DList) MoveToFront(n *dnode) {
	l.unlink(n)
	n.prev, n.next = &l.root, l.root.next
	l.root.next.prev = n
	l.root.next = n
	l.len++
}

func (l *DList) forward() []int {
	var out []int
	for p := l.root.next; p != &l.root; p = p.next {
		out = append(out, p.val)
	}
	return out
}

func (l *DList) backward() []int {
	var out []int
	for p := l.root.prev; p != &l.root; p = p.prev {
		out = append(out, p.val)
	}
	return out
}

func show(xs []int) string {
	parts := make([]string, len(xs))
	for i, x := range xs {
		parts[i] = fmt.Sprint(x)
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

func main() {
	fmt.Println("1) one ring, read both ways")
	l := newDList()
	var h [6]*dnode
	for v := 1; v <= 5; v++ {
		h[v] = l.PushBack(v)
	}
	fwd, bwd := l.forward(), l.backward()
	fmt.Println("   forward: ", show(fwd))
	fmt.Println("   backward:", show(bwd))
	slices.Reverse(bwd)
	fmt.Println("   backward reversed == forward:", slices.Equal(fwd, bwd))

	e := newDList()
	fmt.Println()
	fmt.Println("2) the empty ring is the root pointing at itself:")
	fmt.Println("   root.next == &root && root.prev == &root:", e.root.next == &e.root && e.root.prev == &e.root)

	fmt.Println()
	fmt.Println("3) MoveToFront(node 4) and Remove(node 2), by handle, no search:")
	l.MoveToFront(h[4])
	fmt.Println("   after MoveToFront(4):", show(l.forward()))
	l.Remove(h[2])
	fmt.Println("   after Remove(2):     ", show(l.forward()), " len", l.len)
	fmt.Println("   removed node's links are nil:", h[2].next == nil && h[2].prev == nil)
	fmt.Println("   ifs in insertAfter, unlink, Remove, MoveToFront (counted in the source): 0")

	fmt.Println()
	fmt.Println("4) handles stay valid; slice indices do not")
	type held struct {
		n *dnode
		v int
	}
	rng := rand.New(rand.NewPCG(1, 2))
	l = newDList()
	var live []held
	var xs []int // the same sequence, kept in a slice
	for v := 0; v < 100; v++ {
		live = append(live, held{l.PushBack(v), v})
		xs = append(xs, v)
	}
	saved := make(map[int]int) // slice index -> the value it held at the start
	for i := 0; i < 100; i += 10 {
		saved[i] = xs[i]
	}
	const ops = 1000
	next, removes, inserts, checks, broken, disagree, shortest := 100, 0, 0, 0, 0, 0, l.len
	for op := 0; op < ops; op++ {
		if rng.IntN(2) == 0 && len(live) > 0 { // remove a random live node, by handle
			k := rng.IntN(len(live))
			at := slices.Index(xs, live[k].v) // the slice must SEARCH for it
			xs = slices.Delete(xs, at, at+1)
			l.Remove(live[k].n)
			live[k] = live[len(live)-1]
			live = live[:len(live)-1]
			removes++
		} else if len(live) == 0 { // emptied out: start again at the front
			xs = slices.Insert(xs, 0, next)
			live = append(live, held{l.PushFront(next), next})
			next++
			inserts++
		} else { // insert a new value after a random live node
			k := rng.IntN(len(live))
			at := slices.Index(xs, live[k].v)
			xs = slices.Insert(xs, at+1, next)
			live = append(live, held{l.insertAfter(live[k].n, next), next})
			next++
			inserts++
		}
		shortest = min(shortest, l.len)
		for _, hd := range live { // does every handle still read its own value?
			checks++
			if hd.n.val != hd.v {
				broken++
			}
		}
		if !slices.Equal(l.forward(), xs) {
			disagree++
		}
	}
	moved := 0
	for i, v := range saved {
		if i >= len(xs) || xs[i] != v {
			moved++
		}
	}
	fmt.Printf("   %d ops (%d removes, %d inserts), length 100 -> %d (never below %d)\n",
		ops, removes, inserts, l.len, shortest)
	fmt.Printf("   every held handle re-read after every op: %d reads, %d broken\n", checks, broken)
	fmt.Printf("   list and slice disagreed after %d of %d ops\n", disagree, ops)
	fmt.Printf("   of %d saved slice indices, %d now hold a different value (or are past the end)\n", len(saved), moved)

	fmt.Println()
	fmt.Printf("5) cost: unsafe.Sizeof(dnode{}) = %d bytes (val + 2 pointers)\n", unsafe.Sizeof(dnode{}))
	allocs := testing.AllocsPerRun(10, func() {
		l := newDList()
		for v := 0; v < 1000; v++ {
			l.PushBack(v)
		}
	})
	fmt.Printf("   newDList + 1000 PushBacks: %.0f heap objects (the DList, then one per node)\n", allocs)
}
```

**Output:**

```
1) one ring, read both ways
   forward:  [1 -> 2 -> 3 -> 4 -> 5]
   backward: [5 -> 4 -> 3 -> 2 -> 1]
   backward reversed == forward: true

2) the empty ring is the root pointing at itself:
   root.next == &root && root.prev == &root: true

3) MoveToFront(node 4) and Remove(node 2), by handle, no search:
   after MoveToFront(4): [4 -> 1 -> 2 -> 3 -> 5]
   after Remove(2):      [4 -> 1 -> 3 -> 5]  len 4
   removed node's links are nil: true
   ifs in insertAfter, unlink, Remove, MoveToFront (counted in the source): 0

4) handles stay valid; slice indices do not
   1000 ops (490 removes, 510 inserts), length 100 -> 120 (never below 95)
   every held handle re-read after every op: 114554 reads, 0 broken
   list and slice disagreed after 0 of 1000 ops
   of 10 saved slice indices, 10 now hold a different value (or are past the end)

5) cost: unsafe.Sizeof(dnode{}) = 24 bytes (val + 2 pointers)
   newDList + 1000 PushBacks: 1001 heap objects (the DList, then one per node)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::{Cell, RefCell};
use std::mem::size_of;
use std::ops::{Deref, DerefMut};
use std::rc::{Rc, Weak};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Counting allocator: allocation events and bytes, the stand-in for
// testing.AllocsPerRun.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static BYTES: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so this
// program makes exactly the random choices the Go program makes.
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

    /// like rand.IntN(n): uniform in [0, n), Lemire's method with rejection.
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

// Two pointers to every node means two owners -- which a Box cannot have. The
// escape hatch is shared ownership with a runtime borrow check:
//
//   Rc<RefCell<DNode>>   a reference-counted, heap-allocated, mutable-at-
//                        runtime node: `next` links own their node (strong)
//   Weak<RefCell<DNode>> `prev` links do NOT own (a weak count, no keep-alive)
//
// Links are Options because a removed node has none, and the root must exist
// before it can point at itself. That is the price: Go's "no nil checks" turns
// into `expect()` calls on Options, each one a runtime check Go does not make.
type Handle = Rc<RefCell<DNode>>;

struct DNode {
    val: i64,
    prev: Option<Weak<RefCell<DNode>>>,
    next: Option<Handle>,
}

thread_local! {
    static FREED: Cell<usize> = const { Cell::new(0) };
}

// Count every node that is actually freed: the Rc's last strong owner went away.
impl Drop for DNode {
    fn drop(&mut self) {
        FREED.with(|f| f.set(f.get() + 1));
    }
}

fn freed() -> usize {
    FREED.with(Cell::get)
}

struct Ring {
    root: Handle,
    len: usize,
}

impl Ring {
    fn new() -> Ring {
        let root = Rc::new(RefCell::new(DNode {
            val: 0,
            prev: None,
            next: None,
        }));
        {
            let mut r = root.borrow_mut();
            r.next = Some(Rc::clone(&root)); // the root owns itself: a 1-node cycle
            r.prev = Some(Rc::downgrade(&root));
        }
        Ring { root, len: 0 }
    }

    fn next_of(n: &Handle) -> Handle {
        Rc::clone(n.borrow().next.as_ref().expect("node is in a ring"))
    }

    fn prev_of(n: &Handle) -> Handle {
        let w = n.borrow().prev.clone().expect("node is in a ring");
        w.upgrade().expect("prev is alive: its next owns us")
    }

    // insert_after: the same four link writes as Go; no if (one expect, in next_of).
    fn insert_after(&mut self, at: &Handle, val: i64) -> Handle {
        let next = Ring::next_of(at);
        let n = Rc::new(RefCell::new(DNode {
            val,
            prev: Some(Rc::downgrade(at)), // 1
            next: None,
        }));
        next.borrow_mut().prev = Some(Rc::downgrade(&n)); // 3
        n.borrow_mut().next = Some(next); //                 2
        at.borrow_mut().next = Some(Rc::clone(&n)); //       4
        self.len += 1;
        n
    }

    fn push_front(&mut self, val: i64) -> Handle {
        let root = Rc::clone(&self.root);
        self.insert_after(&root, val)
    }

    fn push_back(&mut self, val: i64) -> Handle {
        let last = Ring::prev_of(&self.root);
        self.insert_after(&last, val)
    }

    // unlink: two link writes. The node's own links are taken (left None),
    // the Rust form of Go's Remove setting them to nil.
    fn unlink(&mut self, n: &Handle) {
        let prev = Ring::prev_of(n);
        let next = n.borrow_mut().next.take().expect("node is in a ring");
        n.borrow_mut().prev = None;
        next.borrow_mut().prev = Some(Rc::downgrade(&prev));
        prev.borrow_mut().next = Some(next);
        self.len -= 1;
    }

    fn remove(&mut self, n: &Handle) {
        self.unlink(n);
    }

    fn move_to_front(&mut self, n: &Handle) {
        self.unlink(n);
        let root = Rc::clone(&self.root);
        let first = Ring::next_of(&root);
        {
            let mut b = n.borrow_mut();
            b.prev = Some(Rc::downgrade(&root));
            b.next = Some(Rc::clone(&first));
        }
        first.borrow_mut().prev = Some(Rc::downgrade(n));
        root.borrow_mut().next = Some(Rc::clone(n));
        self.len += 1;
    }

    fn forward(&self) -> Vec<i64> {
        let mut out = Vec::new();
        let mut p = Ring::next_of(&self.root);
        while !Rc::ptr_eq(&p, &self.root) {
            out.push(p.borrow().val);
            p = Ring::next_of(&p);
        }
        out
    }

    fn backward(&self) -> Vec<i64> {
        let mut out = Vec::new();
        let mut p = Ring::prev_of(&self.root);
        while !Rc::ptr_eq(&p, &self.root) {
            out.push(p.borrow().val);
            p = Ring::prev_of(&p);
        }
        out
    }
}

// DList is a Ring that breaks its own cycle when dropped. Without this, the
// strong `next` links form a loop root -> n1 -> ... -> n5 -> root: every node
// keeps a strong count of at least 1 forever, and nothing is ever freed.
// (Go's tracing GC frees unreachable cycles; reference counting cannot.)
struct DList(Ring);

impl Drop for DList {
    fn drop(&mut self) {
        // take root.next: the cycle is now a plain chain, which we free
        // iteratively (one node per loop turn, no recursion)
        let mut cur = self.0.root.borrow_mut().next.take();
        while let Some(n) = cur {
            cur = n.borrow_mut().next.take();
        }
    }
}

impl Deref for DList {
    type Target = Ring;
    fn deref(&self) -> &Ring {
        &self.0
    }
}

impl DerefMut for DList {
    fn deref_mut(&mut self) -> &mut Ring {
        &mut self.0
    }
}

fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" -> "))
}

fn main() {
    println!("1) one ring, read both ways");
    let mut l = DList(Ring::new());
    let h: Vec<Handle> = (1..=5).map(|v| l.push_back(v)).collect();
    let (fwd, mut bwd) = (l.forward(), l.backward());
    println!("   forward:  {}", show(&fwd));
    println!("   backward: {}", show(&bwd));
    bwd.reverse();
    println!("   backward reversed == forward: {}", fwd == bwd);

    let e = DList(Ring::new());
    println!();
    println!("2) the empty ring is the root pointing at itself:");
    let r = e.root.borrow();
    let self_loop = Rc::ptr_eq(r.next.as_ref().expect("set"), &e.root)
        && r.prev
            .as_ref()
            .expect("set")
            .ptr_eq(&Rc::downgrade(&e.root));
    drop(r);
    println!("   root.next == root && root.prev == root: {self_loop}");

    println!();
    println!("3) move_to_front(node 4) and remove(node 2), by handle, no search:");
    l.move_to_front(&h[3]);
    println!("   after move_to_front(4): {}", show(&l.forward()));
    l.remove(&h[1]);
    println!(
        "   after remove(2):        {}  len {}",
        show(&l.forward()),
        l.len
    );
    let b = h[1].borrow();
    println!(
        "   removed node's links are None: {}",
        b.next.is_none() && b.prev.is_none()
    );
    drop(b);
    println!(
        "   ifs in insert_after, unlink, move_to_front: 0 -- but Ring's source has 4 expect()s:"
    );
    println!(
        "   every link is an Option, and each expect() is a check Go's bare pointers do not make"
    );

    println!();
    println!("   what a handle is, in Rust:");
    println!(
        "   Rc::strong_count(node 3) = {} (its predecessor's next + our handle)",
        Rc::strong_count(&h[2])
    );
    println!(
        "   Rc::weak_count(node 3)   = {} (its successor's prev)",
        Rc::weak_count(&h[2])
    );
    let reading = h[2].borrow(); // a live shared borrow, like holding a &DNode
    println!(
        "   while a borrow() is alive, try_borrow_mut() = {:?}",
        h[2].try_borrow_mut().map(|_| ())
    );
    drop(reading);
    println!(
        "   after it ends,             try_borrow_mut() is ok: {}",
        h[2].try_borrow_mut().is_ok()
    );
    drop(h);
    drop(l);

    println!();
    println!("4) handles stay valid; Vec indices do not");
    let mut rng = Pcg::new(1, 2);
    let mut l = DList(Ring::new());
    let mut live: Vec<(Handle, i64)> = Vec::new();
    let mut xs: Vec<i64> = Vec::new(); // the same sequence, kept in a Vec
    for v in 0..100 {
        live.push((l.push_back(v), v));
        xs.push(v);
    }
    let saved: Vec<(usize, i64)> = (0..100).step_by(10).map(|i| (i, xs[i])).collect();
    const OPS: usize = 1000;
    let (mut next, mut removes, mut inserts) = (100, 0, 0);
    let (mut checks, mut broken, mut disagree, mut shortest) = (0, 0, 0, l.len);
    let pos = |xs: &[i64], v: i64| xs.iter().position(|&x| x == v).expect("present");
    for _ in 0..OPS {
        if rng.int_n(2) == 0 && !live.is_empty() {
            // remove a random live node, by handle
            let k = rng.int_n(live.len());
            let at = pos(&xs, live[k].1); // the Vec must SEARCH for it
            xs.remove(at);
            let (n, _) = live.swap_remove(k);
            l.remove(&n);
            removes += 1;
        } else if live.is_empty() {
            // emptied out: start again at the front
            xs.insert(0, next);
            live.push((l.push_front(next), next));
            next += 1;
            inserts += 1;
        } else {
            // insert a new value after a random live node
            let k = rng.int_n(live.len());
            let at = pos(&xs, live[k].1);
            xs.insert(at + 1, next);
            let n = l.insert_after(&live[k].0, next);
            live.push((n, next));
            next += 1;
            inserts += 1;
        }
        shortest = shortest.min(l.len);
        for (n, v) in &live {
            // does every handle still read its own value?
            checks += 1;
            if n.borrow().val != *v {
                broken += 1;
            }
        }
        if l.forward() != xs {
            disagree += 1;
        }
    }
    let moved = saved
        .iter()
        .filter(|&&(i, v)| xs.get(i) != Some(&v))
        .count();
    println!(
        "   {OPS} ops ({removes} removes, {inserts} inserts), length 100 -> {} (never below {shortest})",
        l.len
    );
    println!("   every held handle re-read after every op: {checks} reads, {broken} broken");
    println!("   list and Vec disagreed after {disagree} of {OPS} ops");
    println!(
        "   of {} saved Vec indices, {moved} now hold a different value (or are past the end)",
        saved.len()
    );
    drop(live);
    drop(l);

    println!();
    println!("5) cost, and the cycle");
    println!(
        "   size_of::<DNode>() = {} bytes (val + Option<Rc> + Option<Weak>)",
        size_of::<DNode>()
    );
    println!(
        "   size_of::<RefCell<DNode>>() = {} (the DNode plus the borrow flag)",
        size_of::<RefCell<DNode>>()
    );
    let (a0, b0) = (ALLOCS.load(Relaxed), BYTES.load(Relaxed));
    let mut l = DList(Ring::new());
    for v in 0..1000 {
        l.push_back(v);
    }
    let (allocs, bytes) = (ALLOCS.load(Relaxed) - a0, BYTES.load(Relaxed) - b0);
    println!(
        "   Ring::new + 1000 push_backs: {allocs} allocations, {} bytes each",
        bytes / allocs
    );
    println!(
        "   (each Rc allocation = strong count + weak count + the RefCell: {} bytes)",
        2 * size_of::<usize>() + size_of::<RefCell<DNode>>()
    );
    let f0 = freed();
    drop(l);
    println!(
        "   dropping that DList (it breaks its ring first) freed {} nodes, root included",
        freed() - f0
    );

    let mut ring = Ring::new(); // the same ring, without DList's Drop (5 nodes: it leaks)
    for v in 0..5 {
        ring.push_back(v);
    }
    let f0 = freed();
    drop(ring);
    println!(
        "   dropping a bare 5-node Ring freed {} of 6: the strong next-links form a cycle,",
        freed() - f0
    );
    println!("   every count stays >= 1, and all of it leaks (Go's GC would free it)");
}
```

**Rust output:**

```
1) one ring, read both ways
   forward:  [1 -> 2 -> 3 -> 4 -> 5]
   backward: [5 -> 4 -> 3 -> 2 -> 1]
   backward reversed == forward: true

2) the empty ring is the root pointing at itself:
   root.next == root && root.prev == root: true

3) move_to_front(node 4) and remove(node 2), by handle, no search:
   after move_to_front(4): [4 -> 1 -> 2 -> 3 -> 5]
   after remove(2):        [4 -> 1 -> 3 -> 5]  len 4
   removed node's links are None: true
   ifs in insert_after, unlink, move_to_front: 0 -- but Ring's source has 4 expect()s:
   every link is an Option, and each expect() is a check Go's bare pointers do not make

   what a handle is, in Rust:
   Rc::strong_count(node 3) = 2 (its predecessor's next + our handle)
   Rc::weak_count(node 3)   = 1 (its successor's prev)
   while a borrow() is alive, try_borrow_mut() = Err(BorrowMutError)
   after it ends,             try_borrow_mut() is ok: true

4) handles stay valid; Vec indices do not
   1000 ops (490 removes, 510 inserts), length 100 -> 120 (never below 95)
   every held handle re-read after every op: 114554 reads, 0 broken
   list and Vec disagreed after 0 of 1000 ops
   of 10 saved Vec indices, 10 now hold a different value (or are past the end)

5) cost, and the cycle
   size_of::<DNode>() = 24 bytes (val + Option<Rc> + Option<Weak>)
   size_of::<RefCell<DNode>>() = 32 (the DNode plus the borrow flag)
   Ring::new + 1000 push_backs: 1001 allocations, 48 bytes each
   (each Rc allocation = strong count + weak count + the RefCell: 48 bytes)
   dropping that DList (it breaks its ring first) freed 1001 nodes, root included
   dropping a bare 5-node Ring freed 0 of 6: the strong next-links form a cycle,
   every count stays >= 1, and all of it leaks (Go's GC would free it)
```

> **Go vs Rust:** Two pointers to every node means two owners, so the escape hatch is `Rc<RefCell<DNode>>` for `next` and `Weak` for `prev`. With the same PCG port, the random-op section is identical to Go: 490/510 ops, 114,554 handle reads, 0 broken, all 10 saved indices moved. What Go does not show: each node is a 48-byte `Rc` allocation (two counts plus a `RefCell`, against Go's 24-byte node), a held handle has `strong_count` 2, an aliasing violation becomes a runtime `Err(BorrowMutError)`, and the strong ring is a reference cycle, so a bare `Ring` frees 0 of its 6 nodes when dropped (the `DList` wrapper breaks the ring in `Drop` and frees all 1001); a tracing GC like Go's frees such cycles. The arena in example 13 is how you would actually write this in Rust.

**Complexity:** insert / remove / move at a held node Θ(1) — four pointer writes, zero branches · traversal Θ(n) either direction · 24 bytes and one heap object per node · after 1,000 random ops: 0 of 114,554 handle reads broken, while all 10 saved slice indices had moved

---

> Next tier: [🟡 medium](2-medium.md) — reverse, two pointers, Floyd's cycle detection, the merge that
> allocates nothing, and `container/list` read in the source.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
