# Step 07 — Linked Lists · 🔴 Hard

Examples **12–16**: when a list beats a slice (rarely), the arena of indices, palindrome in Θ(1) space with the
input restored, the unrolled list, and the LRU cache capstone.

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

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. When a list beats a slice (rarely), measured

`🔴 hard` · *the honest verdict*

"Insertion is O(1)" is true, and it is almost never the reason to use a list: every node is a heap
allocation and every hop is a likely cache miss, while a slice is one block the prefetcher streams. So
instead of arguing, this example measures four workloads on the same 100,000 ints, a doubly linked ring
(example 6's design) against a `[]int`: traversal (on a list built in order, and on an *aged* list whose
links jump around memory), building, memory, and the one job a list was designed for: deleting an element
you already hold. On paper traversal and building are Θ(n) on both, and delete-by-handle is Θ(1) on the
list against Θ(n) on the slice. The constants decide the rest, and the program prints its verdict from
its own ratios.

**Steps:**

1. Build the slice, a fresh list (linked in allocation order) and an aged list (nodes allocated in order,
   Fisher–Yates shuffled with PCG(1, 2), then linked in the shuffled order); check that all three sum to
   the same total.
2. Time traversal and building with `testing.Benchmark`. Deletion mutates its input, so each of its 20
   rounds starts from a freshly built list and a fresh copy of the slice, made outside the timer, and the
   program keeps the best round.
3. Read the table: the list wins only the two delete-by-handle rows, and it wins those by a wide margin.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"strings"
	"testing"
	"time"
)

// "Insertion is O(1)" is true. This program measures whether it matters.
// Same n, same values, a doubly linked ring (example 6's design) against a
// plain []int, on four workloads: traversal, building, memory, and the one
// job a list was designed for -- deleting an element you already hold.

const (
	n      = 100_000 // elements in every structure
	k      = 1_000   // deletions in the delete-by-handle workload
	rounds = 20      // timed rounds of the deletion workload
)

type dnode struct {
	val        int
	prev, next *dnode
}

// DList is example 6's ring around an embedded sentinel: no nil checks.
type DList struct {
	root dnode
	len  int
}

func newDList() *DList {
	l := &DList{}
	l.root.next, l.root.prev = &l.root, &l.root
	return l
}

func (l *DList) insertAfter(at, x *dnode) *dnode {
	x.prev, x.next = at, at.next // wire first
	at.next.prev = x             // splice second
	at.next = x
	l.len++
	return x
}

func (l *DList) PushBack(v int) *dnode { return l.insertAfter(l.root.prev, &dnode{val: v}) }

// Remove unlinks a node we hold: four pointer writes, no search, no shifting.
func (l *DList) Remove(x *dnode) {
	x.prev.next = x.next
	x.next.prev = x.prev
	x.prev, x.next = nil, nil
	l.len--
}

func (l *DList) Sum() int {
	s := 0
	for e := l.root.next; e != &l.root; e = e.next {
		s += e.val
	}
	return s
}

func (l *DList) Values() []int {
	out := make([]int, 0, l.len)
	for e := l.root.next; e != &l.root; e = e.next {
		out = append(out, e.val)
	}
	return out
}

func sumSlice(xs []int) int {
	s := 0
	for _, v := range xs {
		s += v
	}
	return s
}

// buildFresh links 0..n-1 in allocation order: neighbours in the list are
// (mostly) neighbours in memory. It also returns every node as a handle.
func buildFresh() (*DList, []*dnode) {
	l := newDList()
	handles := make([]*dnode, n)
	for i := range n {
		handles[i] = l.PushBack(i)
	}
	return l, handles
}

// buildAged allocates the nodes in order but links them in a shuffled order,
// which is what a long-lived list looks like after months of inserts and
// deletes: every hop jumps to an unrelated address.
func buildAged(rng *rand.Rand) *DList {
	nodes := make([]*dnode, n)
	for i := range nodes {
		nodes[i] = &dnode{val: i}
	}
	shuffle(rng, nodes)
	l := newDList()
	for _, x := range nodes {
		l.insertAfter(l.root.prev, x)
	}
	return l
}

// Fisher-Yates by hand, so the Rust twin can reproduce the exact permutation.
func shuffle[T any](rng *rand.Rand, xs []T) {
	for i := len(xs) - 1; i > 0; i-- {
		j := rng.IntN(i + 1)
		xs[i], xs[j] = xs[j], xs[i]
	}
}

// Typed sinks keep results alive. (Storing into an `any` would box them --
// one more heap allocation that would show up in the memory rows.)
var (
	sinkInt   int
	sinkSlice []int
	sinkList  *DList
)

func nsPerOp(f func()) float64 {
	r := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			f()
		}
	})
	return float64(r.T.Nanoseconds()) / float64(r.N)
}

// heapCost reports heap objects and bytes allocated while f runs.
func heapCost(f func()) (objects, bytes uint64) {
	var before, after runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&before)
	f()
	runtime.ReadMemStats(&after)
	return after.Mallocs - before.Mallocs, after.TotalAlloc - before.TotalAlloc
}

type row struct {
	name        string
	slice, list float64 // ns per op on each side
}

func main() {
	pristine := make([]int, n)
	for i := range pristine {
		pristine[i] = i
	}
	fmt.Printf("n = %d ints, k = %d deletions\n\n", n, k)

	// ---- 2: building. Measured FIRST, on an otherwise empty heap: every GC
	// that building triggers must mark whatever lists are alive, so building
	// a slice next to 200,000 live list nodes would charge the slice for them.
	buildList := nsPerOp(func() {
		l := newDList()
		for i := range n {
			l.PushBack(i)
		}
		sinkList = l
	}) / n
	sinkList = nil
	buildEmpty := nsPerOp(func() {
		var s []int
		for i := range n {
			s = append(s, i)
		}
		sinkSlice = s
	}) / n
	buildPre := nsPerOp(func() {
		s := make([]int, 0, n)
		for i := range n {
			s = append(s, i)
		}
		sinkSlice = s
	}) / n
	sinkSlice = nil

	// ---- the structures under test ----
	rng := rand.New(rand.NewPCG(1, 2))
	fresh, _ := buildFresh()
	aged := buildAged(rng) // first use of the RNG: the aged link order

	// k distinct deletion targets: the first k of a second shuffle.
	perm := slices.Clone(pristine)
	shuffle(rng, perm)
	targets := perm[:k]

	// ---- deterministic checks: every structure holds the same data ----
	want := n * (n - 1) / 2
	sums := []int{sumSlice(pristine), fresh.Sum(), aged.Sum()}
	agree := sums[0] == want && sums[1] == want && sums[2] == want
	fmt.Printf("checksums: slice %d, fresh list %d, aged list %d -> agree: %s\n",
		sums[0], sums[1], sums[2], yes(agree))

	// The slice has no stable handle: the index of a value moves every time
	// something before it is deleted. To give the slice its best case we
	// precompute, outside the timer, the index each target will have at the
	// moment it is deleted ("index known"). The honest case must search.
	idx := make([]int, k)
	expected := slices.Clone(pristine)
	for j, t := range targets {
		i := slices.Index(expected, t)
		idx[j] = i
		expected = slices.Delete(expected, i, i+1)
	}

	// ---- 1: traversal (reads only, so b.Loop over the same input is fine) ----
	sliceSum := nsPerOp(func() { sinkInt = sumSlice(pristine) }) / n
	rows := []row{
		{"traverse, fresh list", sliceSum, nsPerOp(func() { sinkInt = fresh.Sum() }) / n},
		{"traverse, aged list", sliceSum, nsPerOp(func() { sinkInt = aged.Sum() }) / n},
		{"build: PushBack vs append (from empty)", buildEmpty, buildList},
		{"build: PushBack vs append (prealloc)", buildPre, buildList},
	}

	// ---- 4: delete by handle. This MUTATES, so every round starts from a
	// freshly built list and a fresh copy of the slice, built outside the timer.
	// We keep the best round of each: the machine is shared, and the minimum
	// is the run with the least interference.
	best := func(cur *time.Duration, d time.Duration) {
		if *cur == 0 || d < *cur {
			*cur = d
		}
	}
	var listT, knownT, searchT time.Duration
	identical := true
	for r := range rounds {
		l, handles := buildFresh()
		runtime.GC() // nothing below allocates, so no GC can land inside a timer
		t0 := time.Now()
		for _, t := range targets {
			l.Remove(handles[t])
		}
		best(&listT, time.Since(t0))

		s1 := slices.Clone(pristine)
		runtime.GC()
		t0 = time.Now()
		for _, i := range idx {
			s1 = slices.Delete(s1, i, i+1)
		}
		best(&knownT, time.Since(t0))

		s2 := slices.Clone(pristine)
		t0 = time.Now()
		for _, t := range targets {
			i := slices.Index(s2, t)
			s2 = slices.Delete(s2, i, i+1)
		}
		best(&searchT, time.Since(t0))

		if r == 0 {
			identical = slices.Equal(l.Values(), expected) &&
				slices.Equal(s1, expected) && slices.Equal(s2, expected)
		}
	}
	per := func(d time.Duration) float64 { return float64(d.Nanoseconds()) / k }
	rows = append(rows,
		row{name: "delete: handle vs Delete (index known)",
			slice: per(knownT), list: per(listT)},
		row{name: "delete: handle vs Index + Delete",
			slice: per(searchT), list: per(listT)})

	fmt.Printf("after %d deletions: %d left in each; list, slice(known), slice(search) identical: %s\n\n",
		k, len(expected), yes(identical))

	// ---- the one table ----
	fmt.Println("ns per element (traverse, build) or per deletion (delete, best of", rounds, "rounds):")
	fmt.Printf("%-40s %10s %10s  %s\n", "workload", "slice", "list", "winner")
	fmt.Println(strings.Repeat("-", 78))
	var wins []string
	lo, hi, sliceWins := 0.0, 0.0, 0
	for _, r := range rows {
		var w string
		if r.list < r.slice {
			w = fmt.Sprintf("list  %.0fx", r.slice/r.list)
			wins = append(wins, fmt.Sprintf("  %s: list %.0fx faster", r.name, r.slice/r.list))
		} else {
			ratio := r.list / r.slice
			w = fmt.Sprintf("slice %.1fx", ratio)
			if sliceWins == 0 || ratio < lo {
				lo = ratio
			}
			hi = max(hi, ratio)
			sliceWins++
		}
		fmt.Printf("%-40s %10.2f %10.2f  %s\n", r.name, r.slice, r.list, w)
	}

	// ---- 3: memory (counts, not timings) ----
	fmt.Println()
	fmt.Printf("memory to hold %d ints (runtime.MemStats delta while building):\n", n)
	type mem struct {
		name string
		f    func()
	}
	for _, m := range []mem{
		{"[]int, preallocated", func() {
			s := make([]int, 0, n)
			for i := range n {
				s = append(s, i)
			}
			sinkSlice = s
		}},
		{"[]int, append from empty", func() {
			var s []int
			for i := range n {
				s = append(s, i)
			}
			sinkSlice = s
		}},
		{"DList, PushBack", func() {
			l := newDList()
			for i := range n {
				l.PushBack(i)
			}
			sinkList = l
		}},
	} {
		obj, b := heapCost(m.f)
		fmt.Printf("  %-26s %7d heap objects %9d bytes  (%.1f bytes per element)\n",
			m.name, obj, b, float64(b)/n)
	}

	// ---- the verdict, written by the numbers above ----
	fmt.Println()
	fmt.Printf("verdict: the list wins %d of %d rows\n", len(wins), len(rows))
	for _, w := range wins {
		fmt.Println(w)
	}
	if sliceWins > 0 {
		fmt.Printf("and the slice wins the other %d, by %.1fx to %.1fx.\n", sliceWins, lo, hi)
	}
}

func yes(b bool) string {
	if b {
		return "yes"
	}
	return "NO"
}
```

**Sample output:**

```
n = 100000 ints, k = 1000 deletions

checksums: slice 4999950000, fresh list 4999950000, aged list 4999950000 -> agree: yes
after 1000 deletions: 99000 left in each; list, slice(known), slice(search) identical: yes

ns per element (traverse, build) or per deletion (delete, best of 20 rounds):
workload                                      slice       list  winner
------------------------------------------------------------------------------
traverse, fresh list                           0.28       0.86  slice 3.1x
traverse, aged list                            0.28       6.27  slice 22.4x
build: PushBack vs append (from empty)         2.25      13.72  slice 6.1x
build: PushBack vs append (prealloc)           0.53      13.72  slice 25.6x
delete: handle vs Delete (index known)      5776.71       3.79  list  1523x
delete: handle vs Index + Delete           18708.96       3.79  list  4934x

memory to hold 100000 ints (runtime.MemStats delta while building):
  []int, preallocated              1 heap objects    802816 bytes  (8.0 bytes per element)
  []int, append from empty        25 heap objects   4101312 bytes  (41.0 bytes per element)
  DList, PushBack             100001 heap objects   2400032 bytes  (24.0 bytes per element)

verdict: the list wins 2 of 6 rows
  delete: handle vs Delete (index known): list 1523x faster
  delete: handle vs Index + Delete: list 4934x faster
and the slice wins the other 4, by 3.1x to 25.6x.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::RefCell;
use std::collections::LinkedList;
use std::hint::black_box;
use std::rc::{Rc, Weak};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// "Insertion is O(1)" is true. This program measures whether it matters, on
// the same n and the same values as the Go version. Rust has no single list
// that does everything Go's pointer ring does, so each workload uses the list
// that safe Rust would actually reach for:
//   traversal + building: a singly linked Option<Box<Node>> list, and std's
//     LinkedList<i64> (doubly linked) as extra rows;
//   delete by handle:     a doubly linked ring of Rc<RefCell<DNode>> with Weak
//     prev links (example 6's design), because a handle you keep while the
//     list also points at the node means two owners -- shared ownership.
//     LinkedList cannot do this row on stable Rust: removing a node in the
//     middle needs its cursors, which are unstable (linked_list_cursors).

const N: usize = 100_000; // elements in every structure
const K: usize = 1_000; // deletions in the delete-by-handle workload
const ROUNDS: usize = 20; // timed rounds of the deletion workload

// ---- allocation counter (no MemStats in Rust: count at the allocator) ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes requested

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

/// (allocation events, bytes requested) while running `f`; the result is
/// returned so that dropping it is not counted inside.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

// ---- ns/op: stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// ---- Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN, so the
// aged order and the deletion targets are exactly Go's.
#[derive(Clone)]
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

// Fisher-Yates by hand, the same swaps as the Go version.
fn shuffle<T>(rng: &mut Pcg, xs: &mut [T]) {
    for i in (1..xs.len()).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        xs.swap(i, j);
    }
}

// ---- the singly linked Box list ----
struct Node {
    val: i64,
    next: Link,
}
type Link = Option<Box<Node>>;

struct List {
    head: Link,
}

impl List {
    fn new() -> Self {
        List { head: None }
    }
    fn push_front(&mut self, val: i64) {
        let next = self.head.take();
        self.head = Some(Box::new(Node { val, next }));
    }
    fn sum(&self) -> i64 {
        let mut s = 0;
        let mut cur = self.head.as_deref();
        while let Some(node) = cur {
            s += node.val;
            cur = node.next.as_deref();
        }
        s
    }
}

// The compiler-generated drop of a Box chain is recursive: dropping a node
// drops its `next` first, one stack frame per node. 100,000 nodes would be
// 100,000 frames, so free the chain in a loop instead (example 7).
impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.head.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
        }
    }
}

// Linked in allocation order: n-1 pushed first, so 0..n-1 reads in order.
fn build_box_list() -> List {
    let mut l = List::new();
    for i in (0..N as i64).rev() {
        l.push_front(i);
    }
    l
}

// Aged: boxes allocated in order into a Vec, shuffled, then linked in the
// shuffled order. Moving a Box moves only the pointer, so relinking is free.
fn build_aged_box_list(rng: &mut Pcg) -> List {
    let mut boxes: Vec<Box<Node>> = (0..N as i64)
        .map(|val| Box::new(Node { val, next: None }))
        .collect();
    shuffle(rng, &mut boxes);
    let mut l = List::new();
    for mut b in boxes.into_iter().rev() {
        b.next = l.head.take();
        l.head = Some(b);
    }
    l
}

// Aged LinkedList: one single-element list per value, shuffled, then glued
// with `append`, which is Θ(1) and moves the node without reallocating it.
fn build_aged_linked_list(rng: &mut Pcg) -> LinkedList<i64> {
    let mut singles: Vec<LinkedList<i64>> = (0..N as i64).map(|v| LinkedList::from([v])).collect();
    shuffle(rng, &mut singles);
    let mut l = LinkedList::new();
    for mut s in singles {
        l.append(&mut s);
    }
    l
}

// ---- the doubly linked ring with handles: Rc for next, Weak for prev ----
type DRef = Rc<RefCell<DNode>>;

struct DNode {
    val: i64,
    next: Option<DRef>,
    prev: Weak<RefCell<DNode>>,
}

struct DList {
    root: DRef,
}

impl DList {
    fn new() -> Self {
        let root = Rc::new(RefCell::new(DNode {
            val: 0,
            next: None,
            prev: Weak::new(),
        }));
        root.borrow_mut().next = Some(Rc::clone(&root));
        root.borrow_mut().prev = Rc::downgrade(&root);
        DList { root }
    }

    fn push_back(&mut self, val: i64) -> DRef {
        let last = self.root.borrow().prev.upgrade().expect("ring is closed");
        let x = Rc::new(RefCell::new(DNode {
            val,
            next: Some(Rc::clone(&self.root)), // wire first
            prev: Rc::downgrade(&last),
        }));
        self.root.borrow_mut().prev = Rc::downgrade(&x); // splice second
        last.borrow_mut().next = Some(Rc::clone(&x));
        x
    }

    // Unlink a node we hold: the same four link writes as Go, plus refcount
    // updates and a runtime RefCell borrow check on every node touched.
    fn remove(&mut self, x: &DRef) {
        let (prev, next) = {
            let mut node = x.borrow_mut();
            let prev = node.prev.upgrade().expect("linked node has a prev");
            node.prev = Weak::new();
            (prev, node.next.take().expect("linked node has a next"))
        };
        next.borrow_mut().prev = Rc::downgrade(&prev);
        prev.borrow_mut().next = Some(next);
    }

    fn values(&self) -> Vec<i64> {
        let mut out = Vec::new();
        let mut cur = self.root.borrow().next.clone().expect("ring is closed");
        while !Rc::ptr_eq(&cur, &self.root) {
            out.push(cur.borrow().val);
            let next = cur.borrow().next.clone().expect("ring is closed");
            cur = next;
        }
        out
    }
}

// The strong `next` links form a cycle through the sentinel, so without this
// nothing would ever be freed. Break the ring, one link at a time.
impl Drop for DList {
    fn drop(&mut self) {
        let mut cur = self.root.borrow_mut().next.take();
        while let Some(node) = cur {
            cur = node.borrow_mut().next.take();
        }
    }
}

fn build_ring() -> (DList, Vec<DRef>) {
    let mut l = DList::new();
    let handles = (0..N as i64).map(|v| l.push_back(v)).collect();
    (l, handles)
}

struct Row {
    name: &'static str,
    slice: f64, // ns per op on each side
    list: f64,
}

fn yes(b: bool) -> &'static str {
    if b { "yes" } else { "NO" }
}

fn main() {
    let pristine: Vec<i64> = (0..N as i64).collect();
    println!("n = {N} ints, k = {K} deletions\n");

    // ---- 2: building. Each closure builds AND drops its structure, so the
    // list rows include n frees (Rust's counterpart of Go's GC work).
    let build_box = ns_per_op(|| {
        black_box(build_box_list());
    }) / N as f64;
    let build_ll = ns_per_op(|| {
        let mut l = LinkedList::new();
        for i in 0..N as i64 {
            l.push_back(i);
        }
        black_box(l);
    }) / N as f64;
    let build_empty = ns_per_op(|| {
        let mut v = Vec::new();
        for i in 0..N as i64 {
            v.push(i);
        }
        black_box(v);
    }) / N as f64;
    let build_pre = ns_per_op(|| {
        let mut v = Vec::with_capacity(N);
        for i in 0..N as i64 {
            v.push(i);
        }
        black_box(v);
    }) / N as f64;

    // ---- the structures under test ----
    let mut rng = Pcg::new(1, 2);
    let mut rng_same = rng.clone(); // same seed, same permutation for LinkedList
    let fresh = build_box_list();
    let aged = build_aged_box_list(&mut rng); // first use of the RNG: the aged order
    let fresh_ll: LinkedList<i64> = (0..N as i64).collect();
    let aged_ll = build_aged_linked_list(&mut rng_same);

    // k distinct deletion targets: the first k of a second shuffle.
    let mut perm: Vec<usize> = (0..N).collect();
    shuffle(&mut rng, &mut perm);
    let targets = &perm[..K];

    // ---- deterministic checks: every structure holds the same data ----
    let want = (N as i64) * (N as i64 - 1) / 2;
    let slice_total: i64 = pristine.iter().sum();
    let sums = [
        slice_total,
        fresh.sum(),
        aged.sum(),
        fresh_ll.iter().sum(),
        aged_ll.iter().sum(),
    ];
    println!(
        "checksums: slice {}, fresh list {}, aged list {}, LinkedList fresh {} / aged {} -> agree: {}",
        sums[0],
        sums[1],
        sums[2],
        sums[3],
        sums[4],
        yes(sums.iter().all(|&s| s == want))
    );

    // The Vec has no stable handle: an index moves every time something before
    // it is removed. Best case for the Vec: precompute, outside the timer, the
    // index each target will have when it is removed ("index known").
    let mut idx = Vec::with_capacity(K);
    let mut expected = pristine.clone();
    for &t in targets {
        let i = expected
            .iter()
            .position(|&v| v == t as i64)
            .expect("target present");
        idx.push(i);
        expected.remove(i);
    }

    // ---- 1: traversal (reads only, so timing the same input is fine) ----
    let slice_sum = ns_per_op(|| {
        black_box(black_box(&pristine).iter().sum::<i64>());
    }) / N as f64;
    let trav = |f: &dyn Fn() -> i64| {
        ns_per_op(|| {
            black_box(f());
        }) / N as f64
    };
    let mut rows = vec![
        Row {
            name: "traverse, fresh Box list",
            slice: slice_sum,
            list: trav(&|| black_box(&fresh).sum()),
        },
        Row {
            name: "traverse, aged Box list",
            slice: slice_sum,
            list: trav(&|| black_box(&aged).sum()),
        },
        Row {
            name: "traverse, fresh LinkedList",
            slice: slice_sum,
            list: trav(&|| black_box(&fresh_ll).iter().sum()),
        },
        Row {
            name: "traverse, aged LinkedList",
            slice: slice_sum,
            list: trav(&|| black_box(&aged_ll).iter().sum()),
        },
        Row {
            name: "build: Box list vs push (from empty)",
            slice: build_empty,
            list: build_box,
        },
        Row {
            name: "build: Box list vs push (prealloc)",
            slice: build_pre,
            list: build_box,
        },
        Row {
            name: "build: LinkedList vs push (from empty)",
            slice: build_empty,
            list: build_ll,
        },
    ];

    // ---- 4: delete by handle. This MUTATES, so every round starts from a
    // freshly built ring and a fresh clone of the Vec, made outside the timer.
    // Keep the best round of each (the machine is shared).
    let mut best = [Duration::MAX; 3];
    let mut identical = true;
    for r in 0..ROUNDS {
        let (mut l, handles) = build_ring();
        let t0 = Instant::now();
        for &t in targets {
            l.remove(&handles[t]);
        }
        best[0] = best[0].min(t0.elapsed());

        let mut v1 = pristine.clone();
        let t0 = Instant::now();
        for &i in &idx {
            v1.remove(i);
        }
        best[1] = best[1].min(t0.elapsed());

        let mut v2 = pristine.clone();
        let t0 = Instant::now();
        for &t in targets {
            let i = v2
                .iter()
                .position(|&v| v == t as i64)
                .expect("target present");
            v2.remove(i);
        }
        best[2] = best[2].min(t0.elapsed());

        if r == 0 {
            identical = l.values() == expected && v1 == expected && v2 == expected;
        }
    }
    let per = |d: Duration| d.as_nanos() as f64 / K as f64;
    rows.push(Row {
        name: "delete: Rc handle vs remove (index known)",
        slice: per(best[1]),
        list: per(best[0]),
    });
    rows.push(Row {
        name: "delete: Rc handle vs position + remove",
        slice: per(best[2]),
        list: per(best[0]),
    });

    println!(
        "after {K} deletions: {} left in each; ring, Vec(known), Vec(search) identical: {}\n",
        expected.len(),
        yes(identical)
    );

    // ---- the one table ----
    println!("ns per element (traverse, build) or per deletion (delete, best of {ROUNDS} rounds):");
    println!("{:<42} {:>10} {:>10}  winner", "workload", "slice", "list");
    println!("{}", "-".repeat(80));
    let mut wins = Vec::new();
    let (mut lo, mut hi, mut slice_wins) = (0.0f64, 0.0f64, 0);
    for r in &rows {
        let w = if r.list < r.slice {
            wins.push(format!(
                "  {}: list {:.0}x faster",
                r.name,
                r.slice / r.list
            ));
            format!("list  {:.0}x", r.slice / r.list)
        } else {
            let ratio = r.list / r.slice;
            if slice_wins == 0 || ratio < lo {
                lo = ratio;
            }
            hi = hi.max(ratio);
            slice_wins += 1;
            format!("slice {ratio:.1}x")
        };
        println!("{:<42} {:>10.2} {:>10.2}  {w}", r.name, r.slice, r.list);
    }

    // ---- 3: memory, from the counting allocator: exact, no GC noise ----
    println!();
    println!("memory to hold {N} ints (allocator calls while building):");
    let mem = |name: &str, (count, bytes): (usize, usize)| {
        println!(
            "  {name:<26} {count:>7} allocations {bytes:>9} bytes  ({:.1} bytes per element)",
            bytes as f64 / N as f64
        );
    };
    let (c, b, v) = allocs_of(|| {
        let mut v = Vec::with_capacity(N);
        v.extend(0..N as i64);
        v
    });
    drop(v);
    mem("Vec, with_capacity", (c, b));
    let (c, b, v) = allocs_of(|| {
        let mut v = Vec::new();
        for i in 0..N as i64 {
            v.push(i);
        }
        v
    });
    drop(v);
    mem("Vec, push from empty", (c, b));
    let (c, b, l) = allocs_of(build_box_list);
    drop(l);
    mem("Box list, push_front", (c, b));
    let (c, b, l) = allocs_of(|| (0..N as i64).collect::<LinkedList<i64>>());
    drop(l);
    mem("LinkedList, push_back", (c, b));
    let (c, b, l) = allocs_of(|| {
        let mut l = DList::new();
        for v in 0..N as i64 {
            l.push_back(v);
        }
        l
    });
    drop(l);
    mem("Rc ring (+ sentinel)", (c, b));
    println!(
        "  (size_of: Box node {} bytes; Rc ring node {} bytes of payload + 2 counts = {} per allocation)",
        size_of::<Node>(),
        size_of::<RefCell<DNode>>(),
        size_of::<RefCell<DNode>>() + 2 * size_of::<usize>()
    );

    // ---- the verdict, written by the numbers above ----
    println!();
    println!(
        "verdict: the list wins {} of {} rows",
        wins.len(),
        rows.len()
    );
    for w in &wins {
        println!("{w}");
    }
    if slice_wins > 0 {
        println!("and the slice wins the other {slice_wins}, by {lo:.1}x to {hi:.1}x.");
    }
}
```

**Rust sample output:**

```
n = 100000 ints, k = 1000 deletions

checksums: slice 4999950000, fresh list 4999950000, aged list 4999950000, LinkedList fresh 4999950000 / aged 4999950000 -> agree: yes
after 1000 deletions: 99000 left in each; ring, Vec(known), Vec(search) identical: yes

ns per element (traverse, build) or per deletion (delete, best of 20 rounds):
workload                                        slice       list  winner
--------------------------------------------------------------------------------
traverse, fresh Box list                         0.07       1.03  slice 13.8x
traverse, aged Box list                          0.07       5.64  slice 76.0x
traverse, fresh LinkedList                       0.07       0.95  slice 12.8x
traverse, aged LinkedList                        0.07       7.18  slice 96.7x
build: Box list vs push (from empty)             0.53      53.54  slice 101.0x
build: Box list vs push (prealloc)               0.36      53.54  slice 148.5x
build: LinkedList vs push (from empty)           0.53      49.42  slice 93.2x
delete: Rc handle vs remove (index known)     5935.21      11.75  list  505x
delete: Rc handle vs position + remove       18921.42      11.75  list  1610x

memory to hold 100000 ints (allocator calls while building):
  Vec, with_capacity               1 allocations    800000 bytes  (8.0 bytes per element)
  Vec, push from empty            16 allocations   2097120 bytes  (21.0 bytes per element)
  Box list, push_front        100000 allocations   1600000 bytes  (16.0 bytes per element)
  LinkedList, push_back       100000 allocations   2400000 bytes  (24.0 bytes per element)
  Rc ring (+ sentinel)        100001 allocations   4800048 bytes  (48.0 bytes per element)
  (size_of: Box node 16 bytes; Rc ring node 32 bytes of payload + 2 counts = 48 per allocation)

verdict: the list wins 2 of 9 rows
  delete: Rc handle vs remove (index known): list 505x faster
  delete: Rc handle vs position + remove: list 1610x faster
and the slice wins the other 7, by 12.8x to 148.5x.
```

> **Go vs Rust:** Escape hatches: traversal and building use an owned `Option<Box<Node>>` chain (with a hand-written iterative `Drop`, because the generated one recurses once per node), and delete-by-handle needs `Rc<RefCell<_>>` + `Weak`, because a handle the caller keeps *and* the list's own link are two owners of one node; std's `LinkedList` cannot do that row on stable Rust (its cursors are unstable). Same verdict as Go: the list wins only the two delete rows, by two to three orders of magnitude in both languages, although the Rc ring removes more slowly than Go's pointer ring (9–26 ns against 3.5–9 ns per removal across runs here; plausibly the refcount and `RefCell` flag updates, not profiled). The slice margins are much wider in Rust mainly because `Vec` sums in under 0.1 ns/element against Go's ~0.28 (consistent with LLVM vectorising the integer sum, asm not inspected); the lists walk at about the same ~1 ns (fresh) and ~6–9 ns (aged) per node in both, and the Rust build rows include dropping the structure (n `free` calls), Rust's counterpart of Go's GC work.

**Complexity:** traversal Θ(n) on both, building Θ(n) on both, delete at a held node Θ(1) on the list against Θ(n) on the slice · memory: 1 heap object (8 bytes per element) against 100,001 objects (24 bytes per element). Over three runs here: the slice traversed 3.0–3.5× faster than a fresh list and 22–26× faster than an aged one, and built 6–10× faster even when growing from empty. The list won only delete-by-handle: 650–1,640× faster than `slices.Delete` with the index already known, and 2,100–5,300× faster when the slice first had to `slices.Index` for the value.

---

## 13. The arena: links without pointers

`🔴 hard` · *one allocation, int32 links, nothing for the GC to trace*

Keep every node in one slice and link by `int32` index, with a free list threaded through the freed slots
so they are reused first. You keep everything example 6 bought you (Θ(1) insert, remove and move-to-front
at a held position, stable handles) and gain one allocation instead of n, contiguous memory, 4-byte links
and **no pointers at all**, so the garbage collector never has to trace the structure. What you give up:
the type system no longer checks the link (an `int32` is just a number), and a slot is never freed back to
the runtime on its own. Worse, a stale handle silently reads whatever element reused its slot, which is why
each slot carries a generation counter and each handle remembers the generation it was issued with.
Every list operation stays Θ(1), and the stale-handle check is one extra comparison.

**Steps:**

1. Build `Arena{nodes []anode; free int32}` with the sentinel at index 0 and `Handle{idx int32; gen uint32}`,
   then drive it and example 6's pointer ring with the same 100,000 PCG ops (insert after, remove,
   move-to-front) and compare the contents.
2. Remove 1,000 elements, insert 1,000 new ones into the freed slots, and look up the 1,000 old handles
   with and without the generation check.
3. With 200,000 nodes alive, one structure at a time, time a `runtime.GC()` and a full traversal for the
   pointer ring (fresh and aged) and the arena (fresh and after churn).

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"time"
	"unsafe"
)

// An arena list keeps every node in ONE slice and links them by int32 index
// instead of by pointer. Index 0 is the sentinel, so the ring has no nil
// checks (example 6), and 0 doubles as "no next free slot". Freed slots go on
// a free list threaded through their own `next` field and are reused first.

type anode struct {
	val        int64
	prev, next int32  // 4-byte links: indices into Arena.nodes
	gen        uint32 // bumped every time the slot is freed
}

// Handle is what callers hold. idx alone would silently follow a reused
// slot; gen lets the arena notice that the slot has changed hands.
type Handle struct {
	idx int32
	gen uint32
}

type Arena struct {
	nodes []anode // nodes[0] is the sentinel
	free  int32   // head of the free list, 0 = empty
	len   int
}

func NewArena(capHint int) *Arena {
	return &Arena{nodes: make([]anode, 1, capHint+1)} // sentinel: prev = next = 0
}

func (a *Arena) alloc(v int64) int32 {
	if i := a.free; i != 0 { // reuse a freed slot first
		a.free = a.nodes[i].next
		a.nodes[i].val = v
		return i
	}
	a.nodes = append(a.nodes, anode{val: v}) // the only allocation, amortized
	return int32(len(a.nodes) - 1)
}

func (a *Arena) link(at, i int32) { // wire first, splice second
	ns := a.nodes
	ns[i].prev, ns[i].next = at, ns[at].next
	ns[ns[at].next].prev = i
	ns[at].next = i
}

func (a *Arena) unlink(i int32) {
	ns := a.nodes
	ns[ns[i].prev].next = ns[i].next
	ns[ns[i].next].prev = ns[i].prev
}

func (a *Arena) valid(h Handle) bool {
	return h.idx > 0 && int(h.idx) < len(a.nodes) && a.nodes[h.idx].gen == h.gen
}

func (a *Arena) InsertAfter(at Handle, v int64) Handle {
	i := a.alloc(v)
	a.link(at.idx, i)
	a.len++
	return Handle{i, a.nodes[i].gen}
}

func (a *Arena) PushBack(v int64) Handle {
	return a.InsertAfter(Handle{a.nodes[0].prev, 0}, v)
}

func (a *Arena) Remove(h Handle) bool {
	if !a.valid(h) {
		return false
	}
	a.unlink(h.idx)
	a.nodes[h.idx].gen++ // every outstanding handle to this slot is now stale
	a.nodes[h.idx].next = a.free
	a.free = h.idx
	a.len--
	return true
}

func (a *Arena) MoveToFront(h Handle) {
	a.unlink(h.idx)
	a.link(0, h.idx)
}

func (a *Arena) Get(h Handle) (int64, bool) {
	if !a.valid(h) {
		return 0, false
	}
	return a.nodes[h.idx].val, true
}

func (a *Arena) GetUnchecked(h Handle) int64 { return a.nodes[h.idx].val } // no check

func (a *Arena) Values() []int64 {
	out := make([]int64, 0, a.len)
	for i := a.nodes[0].next; i != 0; i = a.nodes[i].next {
		out = append(out, a.nodes[i].val)
	}
	return out
}

func (a *Arena) Sum() int64 {
	var s int64
	for i := a.nodes[0].next; i != 0; i = a.nodes[i].next {
		s += a.nodes[i].val
	}
	return s
}

// ---- example 6's pointer ring, for comparison ----

type dnode struct {
	val        int64
	prev, next *dnode
}

type DList struct{ root dnode }

func NewDList() *DList {
	l := &DList{}
	l.root.next, l.root.prev = &l.root, &l.root
	return l
}

func (l *DList) insertAfter(at, x *dnode) *dnode {
	x.prev, x.next = at, at.next
	at.next.prev = x
	at.next = x
	return x
}

func (l *DList) InsertAfter(at *dnode, v int64) *dnode { return l.insertAfter(at, &dnode{val: v}) }
func (l *DList) PushBack(v int64) *dnode               { return l.InsertAfter(l.root.prev, v) }

func (l *DList) Remove(x *dnode) {
	x.prev.next, x.next.prev = x.next, x.prev
	x.prev, x.next = nil, nil
}

func (l *DList) MoveToFront(x *dnode) {
	x.prev.next, x.next.prev = x.next, x.prev
	l.insertAfter(&l.root, x)
}

func (l *DList) Values() []int64 {
	var out []int64
	for e := l.root.next; e != &l.root; e = e.next {
		out = append(out, e.val)
	}
	return out
}

func (l *DList) Sum() int64 {
	var s int64
	for e := l.root.next; e != &l.root; e = e.next {
		s += e.val
	}
	return s
}

const big = 200_000 // nodes in the GC and traversal section

// ---- builders for section 3/4: each returns ONLY the structure ----

func ptrAged(rng *rand.Rand) *DList {
	nodes := make([]*dnode, big)
	for i := range nodes {
		nodes[i] = &dnode{val: int64(i)}
	}
	for i := len(nodes) - 1; i > 0; i-- { // Fisher-Yates by hand
		j := rng.IntN(i + 1)
		nodes[i], nodes[j] = nodes[j], nodes[i]
	}
	l := NewDList()
	for _, x := range nodes {
		l.insertAfter(l.root.prev, x)
	}
	return l
}

// arenaChurned: build, then remove a random element and insert a new one
// after another random element, big times. The size stays at big, slots are
// reused through the free list, and the link order ends up scattered across
// the slice -- an arena can age too, it just ages inside one block.
func arenaChurned(rng *rand.Rand) (*Arena, int64) {
	a := NewArena(big)
	hs := make([]Handle, big)
	sum := int64(0)
	for i := range big {
		hs[i] = a.PushBack(int64(i))
		sum += int64(i)
	}
	for r := range big {
		j := rng.IntN(len(hs))
		v, _ := a.Get(hs[j])
		a.Remove(hs[j])
		sum -= v
		at := hs[rng.IntN(len(hs))]
		if at == hs[j] { // the one we just removed: insert at the front instead
			at = Handle{}
		}
		nv := int64(big + r)
		hs[j] = a.InsertAfter(at, nv)
		sum += nv
	}
	return a, sum
}

var (
	liveP *DList // the one structure kept alive while measuring GC
	liveA *Arena
	sink  int64
)

// mallocs counts heap allocations so far (exact, unlike the live-object
// count, which drifts by a few runtime-internal objects between GCs).
func mallocs() uint64 {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.Mallocs
}

// best returns the fastest of reps runs of f. The machine is shared, so the
// minimum -- the run with the least interference -- is the stable number.
func best(reps int, f func()) time.Duration {
	m := time.Duration(1<<63 - 1)
	for range reps {
		t0 := time.Now()
		f()
		m = min(m, time.Since(t0))
	}
	return m
}

func gcMicros() float64 {
	runtime.GC()
	return float64(best(15, runtime.GC).Nanoseconds()) / 1e3
}

func nsPerElem(sum func() int64) float64 {
	return float64(best(30, func() { sink = sum() }).Nanoseconds()) / big
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	// ---- 1. correctness: one op sequence, applied to both structures ----
	const start, ops = 1_000, 100_000
	a := NewArena(start)
	p := NewDList()
	var ah []Handle
	var ph []*dnode
	next := int64(0)
	for range start {
		ah = append(ah, a.PushBack(next))
		ph = append(ph, p.PushBack(next))
		next++
	}
	peak := start
	var nIns, nRem, nMove int
	for range ops {
		op := rng.IntN(3)
		if len(ah) == 0 {
			op = 0
		}
		switch op {
		case 0: // insert after a random live element (after the sentinel if none)
			nIns++
			atA, atP := Handle{}, &p.root
			if len(ah) > 0 {
				j := rng.IntN(len(ah))
				atA, atP = ah[j], ph[j]
			}
			ah = append(ah, a.InsertAfter(atA, next))
			ph = append(ph, p.InsertAfter(atP, next))
			next++
		case 1: // remove a random live element, by handle
			nRem++
			j := rng.IntN(len(ah))
			a.Remove(ah[j])
			p.Remove(ph[j])
			last := len(ah) - 1
			ah[j], ph[j] = ah[last], ph[last]
			ah, ph = ah[:last], ph[:last]
		case 2: // move a random live element to the front
			nMove++
			j := rng.IntN(len(ah))
			a.MoveToFront(ah[j])
			p.MoveToFront(ph[j])
		}
		peak = max(peak, len(ah))
	}
	fmt.Println("1. correctness: the same", ops, "PCG ops on the arena and on the pointer ring")
	fmt.Printf("   inserts %d, removes %d, move-to-fronts %d, live at the end %d\n", nIns, nRem, nMove, a.len)
	fmt.Printf("   contents identical: %s\n", yes(slices.Equal(a.Values(), p.Values())))
	fmt.Printf("   slots allocated %d, peak live %d -> every freed slot was reused: %s\n",
		len(a.nodes)-1, peak, yes(len(a.nodes)-1 == peak))
	fmt.Printf("   (without the free list: %d slots, one per element ever inserted)\n", start+nIns)

	// ---- 2. stale handles ----
	const k = 1_000
	b := NewArena(10_000)
	hs := make([]Handle, 10_000)
	for i := range hs {
		hs[i] = b.PushBack(int64(i))
	}
	old := make([]Handle, k)
	for i := range k { // remove k elements at PCG-random positions
		j := rng.IntN(len(hs))
		old[i] = hs[j]
		b.Remove(hs[j])
		hs[j] = hs[len(hs)-1]
		hs = hs[:len(hs)-1]
	}
	for i := range k { // k new elements, values that no old handle ever had
		b.PushBack(int64(1_000_000 + i))
	}
	detected, wrong := 0, 0
	for _, h := range old {
		if _, ok := b.Get(h); !ok {
			detected++
		}
		if v := b.GetUnchecked(h); v >= 1_000_000 { // a value that was never this handle's
			wrong++
		}
	}
	fmt.Println()
	fmt.Printf("2. stale handles: removed %d, inserted %d new; slots still %d, so all reused freed slots\n",
		k, k, len(b.nodes)-1)
	fmt.Printf("   with the generation check:    %d of %d old handles detected as stale\n", detected, k)
	fmt.Printf("   without it (index only):      %d of %d silently return a NEW element's value\n", wrong, k)

	// ---- 3 + 4. GC cost and traversal, one structure alive at a time ----
	fmt.Println()
	fmt.Printf("3+4. %d nodes; each structure measured while it is the only one alive\n", big)
	fmt.Printf("     node size: dnode %d bytes (two 8-byte pointers), anode %d bytes (two 4-byte links + gen)\n",
		unsafe.Sizeof(dnode{}), unsafe.Sizeof(anode{}))
	baseGC := gcMicros()
	fmt.Println()
	fmt.Printf("   %-22s %13s %10s %15s\n", "structure", "allocations", "GC µs", "traverse ns/el")
	fmt.Println("   (heap allocations to build it; fastest of 15 GCs and of 30 traversals)")
	fmt.Printf("   %-22s %13d %10.0f %15s\n", "(nothing live)", 0, baseGC, "-")

	type result struct{ gc, trav float64 }
	want := int64(big) * int64(big-1) / 2
	// build stores the structure in liveP/liveA and returns its Sum and the
	// expected checksum; everything else it allocated is garbage by then.
	measure := func(name string, build func() (func() int64, int64)) result {
		liveP, liveA = nil, nil
		runtime.GC() // collect the previous structure first
		base := mallocs()
		sum, want := build()
		objs := mallocs() - base
		gc := gcMicros()
		got := sum()
		trav := nsPerElem(sum)
		ok := ""
		if got != want {
			ok = "  CHECKSUM MISMATCH"
		}
		fmt.Printf("   %-22s %13d %10.0f %15.2f%s\n", name, objs, gc, trav, ok)
		return result{gc, trav}
	}

	pf := measure("pointer ring, fresh", func() (func() int64, int64) {
		liveP = NewDList()
		for i := range big {
			liveP.PushBack(int64(i))
		}
		return liveP.Sum, want
	})
	pa := measure("pointer ring, aged", func() (func() int64, int64) {
		liveP = ptrAged(rng)
		return liveP.Sum, want
	})
	af := measure("arena, fresh", func() (func() int64, int64) {
		liveA = NewArena(big)
		for i := range big {
			liveA.PushBack(int64(i))
		}
		return liveA.Sum, want
	})
	ac := measure("arena, after churn", func() (func() int64, int64) {
		var churnSum int64
		liveA, churnSum = arenaChurned(rng)
		return liveA.Sum, churnSum
	})
	liveP, liveA = nil, nil
	fmt.Println()
	fmt.Printf("GC: a collection with the fresh pointer ring alive took %.1fx as long as with the arena alive\n", pf.gc/af.gc)
	fmt.Printf("    (and %.1fx the empty-heap baseline; the arena: %.1fx).\n", pf.gc/baseGC, af.gc/baseGC)
	fmt.Printf("traversal: arena fresh vs pointer fresh %.2fx the time; arena churned vs pointer aged %.2fx;\n",
		af.trav/pf.trav, ac.trav/pa.trav)
	fmt.Printf("           churn made the arena %.1fx slower to walk, aging made the pointer ring %.1fx slower.\n",
		ac.trav/af.trav, pa.trav/pf.trav)
}

func yes(b bool) string {
	if b {
		return "yes"
	}
	return "NO"
}
```

**Sample output:**

```
1. correctness: the same 100000 PCG ops on the arena and on the pointer ring
   inserts 33383, removes 33164, move-to-fronts 33453, live at the end 1219
   contents identical: yes
   slots allocated 1304, peak live 1304 -> every freed slot was reused: yes
   (without the free list: 34383 slots, one per element ever inserted)

2. stale handles: removed 1000, inserted 1000 new; slots still 10000, so all reused freed slots
   with the generation check:    1000 of 1000 old handles detected as stale
   without it (index only):      1000 of 1000 silently return a NEW element's value

3+4. 200000 nodes; each structure measured while it is the only one alive
     node size: dnode 24 bytes (two 8-byte pointers), anode 24 bytes (two 4-byte links + gen)

   structure                allocations      GC µs  traverse ns/el
   (heap allocations to build it; fastest of 15 GCs and of 30 traversals)
   (nothing live)                     0         83               -
   pointer ring, fresh           200002       3157            0.87
   pointer ring, aged            200010       4436            8.21
   arena, fresh                       3         75            0.57
   arena, after churn                 4         82            5.68

GC: a collection with the fresh pointer ring alive took 42.3x as long as with the arena alive
    (and 37.9x the empty-heap baseline; the arena: 0.9x).
traversal: arena fresh vs pointer fresh 0.65x the time; arena churned vs pointer aged 0.69x;
           churn made the arena 10.0x slower to walk, aging made the pointer ring 9.5x slower.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::RefCell;
use std::hint::black_box;
use std::rc::{Rc, Weak};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// An arena list keeps every node in ONE Vec and links them by u32 index
// instead of by pointer. Index 0 is the sentinel, so the ring has no
// special cases (example 6), and 0 doubles as "no next free slot". Freed
// slots go on a free list threaded through their own `next` field.
//
// In Rust this is not a trick, it is THE way to write a doubly linked list
// in safe code: the Vec is the single owner of every node, a link is a plain
// number, and a Handle is a Copy struct -- the borrow checker has nothing to
// object to. No Rc, no RefCell, no Weak, no unsafe.

#[derive(Clone, Copy)]
struct ANode {
    val: i64,
    prev: u32, // 4-byte links: indices into Arena::nodes
    next: u32,
    // bumped every time the slot is freed. (Go calls it `gen`; in edition
    // 2024 `gen` is a reserved keyword, so the twin spells it out.)
    generation: u32,
}

// What callers hold. idx alone would silently follow a reused slot; gen lets
// the arena notice that the slot has changed hands.
#[derive(Clone, Copy, PartialEq, Eq)]
struct Handle {
    idx: u32,
    generation: u32,
}

const ROOT: Handle = Handle {
    idx: 0,
    generation: 0,
};

struct Arena {
    nodes: Vec<ANode>, // nodes[0] is the sentinel
    free: u32,         // head of the free list, 0 = empty
    len: usize,
}

impl Arena {
    fn with_capacity(cap: usize) -> Self {
        let mut nodes = Vec::with_capacity(cap + 1);
        nodes.push(ANode {
            val: 0,
            prev: 0,
            next: 0,
            generation: 0,
        });
        Arena {
            nodes,
            free: 0,
            len: 0,
        }
    }

    fn alloc(&mut self, val: i64) -> u32 {
        if self.free != 0 {
            // reuse a freed slot first
            let i = self.free;
            self.free = self.nodes[i as usize].next;
            self.nodes[i as usize].val = val;
            return i;
        }
        self.nodes.push(ANode {
            val,
            prev: 0,
            next: 0,
            generation: 0,
        }); // the only allocation, amortized
        (self.nodes.len() - 1) as u32
    }

    fn link(&mut self, at: u32, i: u32) {
        // wire first, splice second
        let next = self.nodes[at as usize].next;
        self.nodes[i as usize].prev = at;
        self.nodes[i as usize].next = next;
        self.nodes[next as usize].prev = i;
        self.nodes[at as usize].next = i;
    }

    fn unlink(&mut self, i: u32) {
        let ANode { prev, next, .. } = self.nodes[i as usize];
        self.nodes[prev as usize].next = next;
        self.nodes[next as usize].prev = prev;
    }

    fn valid(&self, h: Handle) -> bool {
        h.idx > 0
            && self
                .nodes
                .get(h.idx as usize)
                .is_some_and(|n| n.generation == h.generation)
    }

    fn insert_after(&mut self, at: Handle, val: i64) -> Handle {
        let i = self.alloc(val);
        self.link(at.idx, i);
        self.len += 1;
        Handle {
            idx: i,
            generation: self.nodes[i as usize].generation,
        }
    }

    fn push_back(&mut self, val: i64) -> Handle {
        let last = Handle {
            idx: self.nodes[0].prev,
            generation: 0,
        };
        self.insert_after(last, val)
    }

    fn remove(&mut self, h: Handle) -> bool {
        if !self.valid(h) {
            return false;
        }
        self.unlink(h.idx);
        let n = &mut self.nodes[h.idx as usize];
        n.generation = n.generation.wrapping_add(1); // every outstanding handle to this slot is now stale
        n.next = self.free;
        self.free = h.idx;
        self.len -= 1;
        true
    }

    fn move_to_front(&mut self, h: Handle) {
        self.unlink(h.idx);
        self.link(0, h.idx);
    }

    fn get(&self, h: Handle) -> Option<i64> {
        self.valid(h).then(|| self.nodes[h.idx as usize].val)
    }

    // What you get without a generation: still memory-safe (it is an index
    // into a Vec), but it reads whoever lives in the slot now.
    fn get_ignoring_gen(&self, h: Handle) -> i64 {
        self.nodes[h.idx as usize].val
    }

    fn iter(&self) -> impl Iterator<Item = i64> + '_ {
        let mut i = self.nodes[0].next;
        std::iter::from_fn(move || {
            (i != 0).then(|| {
                let n = &self.nodes[i as usize];
                i = n.next;
                n.val
            })
        })
    }

    fn sum(&self) -> i64 {
        let mut s = 0;
        let mut i = self.nodes[0].next;
        while i != 0 {
            let n = &self.nodes[i as usize];
            s += n.val;
            i = n.next;
        }
        s
    }
}

// ---- example 6's ring, the safe-Rust pointer way: Rc next, Weak prev ----
type DRef = Rc<RefCell<DNode>>;

struct DNode {
    val: i64,
    next: Option<DRef>,
    prev: Weak<RefCell<DNode>>,
}

struct DList {
    root: DRef,
}

impl DList {
    fn new() -> Self {
        let root = Rc::new(RefCell::new(DNode {
            val: 0,
            next: None,
            prev: Weak::new(),
        }));
        root.borrow_mut().next = Some(Rc::clone(&root));
        root.borrow_mut().prev = Rc::downgrade(&root);
        DList { root }
    }

    fn link_after(&self, at: &DRef, x: DRef) -> DRef {
        let next = at.borrow_mut().next.take().expect("ring is closed");
        {
            let mut n = x.borrow_mut();
            n.prev = Rc::downgrade(at); // wire first
            n.next = Some(Rc::clone(&next));
        }
        next.borrow_mut().prev = Rc::downgrade(&x); // splice second
        at.borrow_mut().next = Some(Rc::clone(&x));
        x
    }

    fn insert_after(&self, at: &DRef, val: i64) -> DRef {
        let x = Rc::new(RefCell::new(DNode {
            val,
            next: None,
            prev: Weak::new(),
        }));
        self.link_after(at, x)
    }

    fn push_back(&self, val: i64) -> DRef {
        let last = self.root.borrow().prev.upgrade().expect("ring is closed");
        self.insert_after(&last, val)
    }

    fn unlink(&self, x: &DRef) {
        let (prev, next) = {
            let mut n = x.borrow_mut();
            let prev = n.prev.upgrade().expect("linked node has a prev");
            n.prev = Weak::new();
            (prev, n.next.take().expect("linked node has a next"))
        };
        next.borrow_mut().prev = Rc::downgrade(&prev);
        prev.borrow_mut().next = Some(next);
    }

    fn move_to_front(&self, x: &DRef) {
        self.unlink(x);
        let root = Rc::clone(&self.root);
        self.link_after(&root, Rc::clone(x));
    }

    // The safe walk clones an Rc per hop: a refcount increment and decrement
    // plus a RefCell borrow on every node.
    fn fold(&self, mut f: impl FnMut(i64)) {
        let mut cur = self.root.borrow().next.clone().expect("ring is closed");
        while !Rc::ptr_eq(&cur, &self.root) {
            let next = {
                let n = cur.borrow();
                f(n.val);
                n.next.clone().expect("ring is closed")
            };
            cur = next;
        }
    }

    fn values(&self) -> Vec<i64> {
        let mut out = Vec::new();
        self.fold(|v| out.push(v));
        out
    }

    fn sum(&self) -> i64 {
        let mut s = 0;
        self.fold(|v| s += v);
        s
    }
}

// The strong `next` links form a cycle through the sentinel: break it, or
// nothing is ever freed.
impl Drop for DList {
    fn drop(&mut self) {
        let mut cur = self.root.borrow_mut().next.take();
        while let Some(node) = cur {
            cur = node.borrow_mut().next.take();
        }
    }
}

// ---- Pcg: port of Go's math/rand/v2 PCG-DXSM + IntN: same op sequence ----
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

// ---- allocation counter: there is no GC, so count what Rust does pay ----
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

fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

const BIG: usize = 200_000; // nodes in the drop and traversal section

// ---- builders for section 3/4 ----

fn ptr_aged(rng: &mut Pcg) -> DList {
    let mut nodes: Vec<DRef> = (0..BIG as i64)
        .map(|val| {
            Rc::new(RefCell::new(DNode {
                val,
                next: None,
                prev: Weak::new(),
            }))
        })
        .collect();
    for i in (1..nodes.len()).rev() {
        // Fisher-Yates by hand, Go's swaps
        nodes.swap(i, rng.int_n(i + 1));
    }
    let l = DList::new();
    for x in nodes {
        let last = l.root.borrow().prev.upgrade().expect("ring is closed");
        l.link_after(&last, x);
    }
    l
}

// Build, then remove a random element and insert a new one after another
// random element, BIG times: size stays BIG, slots are reused, and the link
// order ends up scattered across the Vec.
fn arena_churned(rng: &mut Pcg) -> (Arena, i64) {
    let mut a = Arena::with_capacity(BIG);
    let mut hs: Vec<Handle> = (0..BIG as i64).map(|v| a.push_back(v)).collect();
    let mut sum: i64 = (0..BIG as i64).sum();
    for r in 0..BIG {
        let j = rng.int_n(hs.len());
        sum -= a.get(hs[j]).expect("live handle");
        a.remove(hs[j]);
        let mut at = hs[rng.int_n(hs.len())];
        if at == hs[j] {
            at = ROOT; // the one we just removed: insert at the front instead
        }
        let nv = (BIG + r) as i64;
        hs[j] = a.insert_after(at, nv);
        sum += nv;
    }
    (a, sum)
}

/// fastest of `reps` runs: the machine is shared, the minimum is the stable number.
fn best(reps: usize, mut f: impl FnMut() -> Duration) -> Duration {
    (0..reps).map(|_| f()).min().expect("reps > 0")
}

fn yes(b: bool) -> &'static str {
    if b { "yes" } else { "NO" }
}

struct Measured {
    drop: f64,
    trav: f64,
}

fn main() {
    let mut rng = Pcg::new(1, 2);

    // ---- 1. correctness: one op sequence, applied to both structures ----
    const START: usize = 1_000;
    const OPS: usize = 100_000;
    let mut a = Arena::with_capacity(START);
    let p = DList::new();
    let mut ah: Vec<Handle> = Vec::new();
    let mut ph: Vec<DRef> = Vec::new();
    let mut next: i64 = 0;
    for _ in 0..START {
        ah.push(a.push_back(next));
        ph.push(p.push_back(next));
        next += 1;
    }
    let mut peak = START;
    let (mut n_ins, mut n_rem, mut n_move) = (0, 0, 0);
    for _ in 0..OPS {
        let op = rng.int_n(3); // drawn even when empty, as in Go
        let op = if ah.is_empty() { 0 } else { op };
        match op {
            0 => {
                // insert after a random live element (after the sentinel if none)
                n_ins += 1;
                let (at_a, at_p) = if ah.is_empty() {
                    (ROOT, Rc::clone(&p.root))
                } else {
                    let j = rng.int_n(ah.len());
                    (ah[j], Rc::clone(&ph[j]))
                };
                ah.push(a.insert_after(at_a, next));
                ph.push(p.insert_after(&at_p, next));
                next += 1;
            }
            1 => {
                // remove a random live element, by handle
                n_rem += 1;
                let j = rng.int_n(ah.len());
                a.remove(ah.swap_remove(j));
                p.unlink(&ph.swap_remove(j));
            }
            _ => {
                // move a random live element to the front
                n_move += 1;
                let j = rng.int_n(ah.len());
                a.move_to_front(ah[j]);
                p.move_to_front(&ph[j]);
            }
        }
        peak = peak.max(ah.len());
    }
    println!("1. correctness: the same {OPS} PCG ops on the arena and on the pointer ring");
    println!(
        "   inserts {n_ins}, removes {n_rem}, move-to-fronts {n_move}, live at the end {}",
        a.len
    );
    println!(
        "   contents identical: {}",
        yes(a.iter().eq(p.values().into_iter()))
    );
    println!(
        "   slots allocated {}, peak live {peak} -> every freed slot was reused: {}",
        a.nodes.len() - 1,
        yes(a.nodes.len() - 1 == peak)
    );
    println!(
        "   (without the free list: {} slots, one per element ever inserted)",
        START + n_ins
    );

    // ---- 2. stale handles ----
    const K: usize = 1_000;
    let mut b = Arena::with_capacity(10_000);
    let mut hs: Vec<Handle> = (0..10_000).map(|v| b.push_back(v)).collect();
    let mut old = Vec::with_capacity(K);
    for _ in 0..K {
        // remove K elements at PCG-random positions
        let j = rng.int_n(hs.len());
        let h = hs.swap_remove(j);
        b.remove(h);
        old.push(h);
    }
    for i in 0..K {
        // K new elements, values that no old handle ever had
        b.push_back(1_000_000 + i as i64);
    }
    let detected = old.iter().filter(|&&h| b.get(h).is_none()).count();
    let wrong = old
        .iter()
        .filter(|&&h| b.get_ignoring_gen(h) >= 1_000_000)
        .count();
    println!();
    println!(
        "2. stale handles: removed {K}, inserted {K} new; slots still {}, so all reused freed slots",
        b.nodes.len() - 1
    );
    println!("   with the generation check:    {detected} of {K} old handles detected as stale");
    println!(
        "   without it (index only):      {wrong} of {K} silently return a NEW element's value"
    );

    // ---- 3 + 4. no GC: what Rust pays instead is n allocations and n frees ----
    println!();
    println!("3+4. {BIG} nodes; each structure built, walked and dropped on its own");
    println!(
        "     node size: ANode {} bytes (two 4-byte links + generation); Rc ring node {} bytes per allocation",
        size_of::<ANode>(),
        size_of::<RefCell<DNode>>() + 2 * size_of::<usize>()
    );
    println!();
    println!(
        "   {:<26} {:>11} {:>10} {:>9} {:>15}",
        "structure", "allocations", "bytes", "drop µs", "traverse ns/el"
    );
    println!("   (allocator calls to build it; fastest of 5 drops and of 30 traversals)");

    let want = (BIG as i64) * (BIG as i64 - 1) / 2;
    // build() returns the structure and its expected checksum; the drop is
    // timed on a fresh build each rep, since dropping consumes it.
    fn measure<T>(
        name: &str,
        mut build: impl FnMut() -> (T, i64),
        sum: impl Fn(&T) -> i64,
    ) -> Measured {
        let (count, bytes, (s, want)) = allocs_of(&mut build);
        let got = sum(&s);
        let trav = best(30, || {
            let t = Instant::now();
            black_box(sum(black_box(&s)));
            t.elapsed()
        })
        .as_nanos() as f64
            / BIG as f64;
        drop(s);
        let dropt = best(5, || {
            let (s, _) = build();
            let t = Instant::now();
            drop(black_box(s));
            t.elapsed()
        })
        .as_nanos() as f64
            / 1e3;
        let ok = if got == want {
            ""
        } else {
            "  CHECKSUM MISMATCH"
        };
        println!("   {name:<26} {count:>11} {bytes:>10} {dropt:>9.0} {trav:>15.2}{ok}");
        Measured { drop: dropt, trav }
    }

    let pf = measure(
        "Rc ring, fresh",
        || {
            let l = DList::new();
            for v in 0..BIG as i64 {
                l.push_back(v);
            }
            (l, want)
        },
        DList::sum,
    );
    // The aged builder consumes the RNG, so it runs ONCE with the shared RNG
    // (Go's sequence) and the drop reps re-shuffle with their own seed.
    let mut first = true;
    let mut rng_drop = Pcg::new(3, 4);
    let pa = measure(
        "Rc ring, aged",
        || {
            let r = if first { &mut rng } else { &mut rng_drop };
            first = false;
            (ptr_aged(r), want)
        },
        DList::sum,
    );
    let af = measure(
        "arena, fresh",
        || {
            let mut a = Arena::with_capacity(BIG);
            for v in 0..BIG as i64 {
                a.push_back(v);
            }
            (a, want)
        },
        Arena::sum,
    );
    measure(
        "arena, grown from empty",
        || {
            let mut a = Arena::with_capacity(0);
            for v in 0..BIG as i64 {
                a.push_back(v);
            }
            (a, want)
        },
        Arena::sum,
    );
    let mut first = true;
    let mut rng_drop = Pcg::new(3, 4);
    let ac = measure(
        "arena, after churn",
        || {
            let r = if first { &mut rng } else { &mut rng_drop };
            first = false;
            arena_churned(r)
        },
        Arena::sum,
    );

    println!("   (aged and churned also count the temporary Vec they were built through)");
    println!();
    println!(
        "drop: freeing the fresh Rc ring took {:.0}x as long as freeing the arena (n frees vs 1).",
        pf.drop / af.drop
    );
    println!(
        "traversal: arena fresh vs Rc fresh {:.2}x the time; arena churned vs Rc aged {:.2}x;",
        af.trav / pf.trav,
        ac.trav / pa.trav
    );
    println!(
        "           churn made the arena {:.1}x slower to walk, aging made the Rc ring {:.1}x slower.",
        ac.trav / af.trav,
        pa.trav / pf.trav
    );
}
```

**Rust sample output:**

```
1. correctness: the same 100000 PCG ops on the arena and on the pointer ring
   inserts 33383, removes 33164, move-to-fronts 33453, live at the end 1219
   contents identical: yes
   slots allocated 1304, peak live 1304 -> every freed slot was reused: yes
   (without the free list: 34383 slots, one per element ever inserted)

2. stale handles: removed 1000, inserted 1000 new; slots still 10000, so all reused freed slots
   with the generation check:    1000 of 1000 old handles detected as stale
   without it (index only):      1000 of 1000 silently return a NEW element's value

3+4. 200000 nodes; each structure built, walked and dropped on its own
     node size: ANode 24 bytes (two 4-byte links + generation); Rc ring node 48 bytes per allocation

   structure                  allocations      bytes   drop µs  traverse ns/el
   (allocator calls to build it; fastest of 5 drops and of 30 traversals)
   Rc ring, fresh                  200001    9600048      5137            1.47
   Rc ring, aged                   200002   11200048     12650            9.60
   arena, fresh                         1    4800024        15            2.12
   arena, grown from empty             18   12582840        30            2.13
   arena, after churn                   2    6400024        41            5.86
   (aged and churned also count the temporary Vec they were built through)

drop: freeing the fresh Rc ring took 350x as long as freeing the arena (n frees vs 1).
traversal: arena fresh vs Rc fresh 1.44x the time; arena churned vs Rc aged 0.61x;
           churn made the arena 2.8x slower to walk, aging made the Rc ring 6.5x slower.
```

> **Go vs Rust:** No escape hatch needed: the arena is how Rust programs actually write doubly linked lists and graphs, since the `Vec` is the single owner of every node, links are `u32`s and a `Handle` is a `Copy` struct, so there is no `Rc`, no `RefCell` and no `unsafe` (generational indices like these are what crates such as `slotmap` provide; not used here). Only the comparison ring needs `Rc<RefCell<_>>` + `Weak` plus a ring-breaking `Drop`. Sections 1–2 print the same counts as Go. Rust has no GC, so section 3 shows what it pays instead: 200,001 allocations against 1, and a drop that takes two to three orders of magnitude longer for the Rc ring.
> One reversal: in Rust the fresh arena walked *slower* than the fresh Rc ring (about 2.1 against 1.5 ns/node over three runs), where Go's arena walked faster (about 0.6 against 0.9). The cause is not established: the Rust loop has an integer multiply-add on its dependent index chain where Go uses shifts, but a side test with 32-byte nodes, which removes the multiply, still ran at 1.6 ns.

**Complexity:** insert, remove and move-to-front at a handle Θ(1), as in example 6 · stale-handle check Θ(1) · one amortized allocation instead of n · GC work that does not grow with n, because `anode` holds no pointers. Deterministic: after 100,000 mixed ops the arena and the pointer ring hold identical contents in 1,304 slots (the peak live count, against 34,383 without a free list), and the generation check catches 1,000 of 1,000 stale handles that would otherwise all read a new element's value. Note that with the `gen` field `anode` is 24 bytes, the same as a pointer `dnode`: the 4-byte links do not save memory here. Over seven runs: a GC with the 200,000-node pointer ring alive took tens of times as long as with the arena alive (22–56×), and with the arena alive a GC cost about what it costs on an empty heap (0.6–1.3×). The arena also walked faster than the pointer ring (roughly 0.5–0.75× the time fresh, 0.7–0.8× aged), but it *does* age: churn made its traversal about 10× slower, about as much as aging slowed the pointer ring (7–10×).

---

## 14. Palindrome in Θ(1) space, and putting it back

`🔴 hard` · *composing middle + reverse, and restoring the input*

`isPalindrome` in Θ(1) extra space is example 8's first middle, plus example 7's reverse of the second
half, plus a two-pointer comparison. It saves memory by **mutating its input**, so a correct version must
reverse the half back and reattach it before returning. Forgetting that one line still gives the right
answer, and it silently changes the caller's list. `reorder` (L0, Ln, L1, Ln−1, …) is the same split and
reverse followed by a merge that alternates between the halves. The algorithms are short. The work is in
the composition and the edge cases (empty, one node, even against odd length), so the program checks
every list up to length 12 against a slice oracle, and it checks that the *same nodes* come back, not just
the same values. Θ(n) time, Θ(1) space, zero allocations.

**Steps:**

1. Run `isPalindrome` on all 8,191 lists of length 0..12 over {0, 1}, comparing the answer with a slice
   oracle and the list afterwards with a snapshot of its node pointers and values.
2. Delete the restore line and watch what `[1 -> 2 -> 2 -> 1]` turns into. Then check `reorder` the same
   exhaustive way, including that it allocates nothing.
3. On a 1,000,000-node palindrome, count hops, allocations and bytes for the in-place version and for the
   copy-into-a-slice version, and time both.

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
	"strings"
	"time"
)

// isPalindrome in Θ(1) extra space is three earlier examples glued together:
// example 8's first middle, example 7's reverse, and a two-pointer compare.
// It works by REVERSING HALF OF ITS INPUT -- so a correct version must put the
// list back exactly as it found it. That restore is the hard part.

type node struct {
	val  int
	next *node
}

var hops int // every `x = x.next` below counts one hop

func fromSlice(vs []int) *node {
	var head *node
	for i := len(vs) - 1; i >= 0; i-- {
		head = &node{vs[i], head}
	}
	return head
}

func show(head *node) string {
	var b strings.Builder
	b.WriteString("[")
	for x := head; x != nil; x = x.next {
		if x != head {
			b.WriteString(" -> ")
		}
		fmt.Fprint(&b, x.val)
	}
	b.WriteString("]")
	return b.String()
}

// snapshot records the nodes themselves, not just the values: "restored"
// means the same nodes, in the same order, holding the same values.
type snap struct {
	ptrs []*node
	vals []int
}

func take(head *node) snap {
	var s snap
	for x := head; x != nil; x = x.next {
		s.ptrs = append(s.ptrs, x)
		s.vals = append(s.vals, x.val)
	}
	return s
}

func (s snap) same(t snap) bool { return slices.Equal(s.ptrs, t.ptrs) && slices.Equal(s.vals, t.vals) }

// firstMiddle returns node (n-1)/2: the middle of an odd list, the FIRST of
// the two middles of an even one. The loop condition decides which.
func firstMiddle(head *node) *node {
	slow, fast := head, head
	for fast.next != nil && fast.next.next != nil {
		slow = slow.next
		fast = fast.next.next
		hops += 3
	}
	return slow
}

func reverse(head *node) *node {
	var prev *node
	for cur := head; cur != nil; {
		next := cur.next // save
		cur.next = prev  // flip
		prev, cur = cur, next
		hops++
	}
	return prev
}

func isPalindrome(head *node) bool {
	if head == nil || head.next == nil {
		return true
	}
	mid := firstMiddle(head)
	second := reverse(mid.next) // mid.next still points at the old first node of the half
	ok := true
	for p, q := head, second; q != nil; p, q = p.next, q.next {
		hops += 2
		if p.val != q.val {
			ok = false
			break
		}
	}
	mid.next = reverse(second) // PUT IT BACK
	return ok
}

// The same function without the last line before return.
func isPalindromeNoRestore(head *node) bool {
	if head == nil || head.next == nil {
		return true
	}
	mid := firstMiddle(head)
	second := reverse(mid.next)
	for p, q := head, second; q != nil; p, q = p.next, q.next {
		if p.val != q.val {
			return false
		}
	}
	return true
}

// The Θ(n)-space version: copy into a slice, compare from both ends.
func isPalindromeCopy(head *node) bool {
	n := 0
	for x := head; x != nil; x = x.next {
		n++
		hops++
	}
	vals := make([]int, 0, n)
	for x := head; x != nil; x = x.next {
		vals = append(vals, x.val)
		hops++
	}
	for i, j := 0, n-1; i < j; i, j = i+1, j-1 {
		if vals[i] != vals[j] {
			return false
		}
	}
	return true
}

// reorder: L0, Ln, L1, Ln-1, ... Same split and reverse, then a merge that
// alternates between the halves. Every output node already exists.
func reorder(head *node) {
	if head == nil || head.next == nil {
		return
	}
	mid := firstMiddle(head)
	second := reverse(mid.next)
	mid.next = nil // cut: the first half ends at mid
	for p := head; second != nil; {
		n1, n2 := p.next, second.next
		p.next = second
		second.next = n1
		p, second = n1, n2
	}
}

func oraclePal(vs []int) bool {
	for i, j := 0, len(vs)-1; i < j; i, j = i+1, j-1 {
		if vs[i] != vs[j] {
			return false
		}
	}
	return true
}

func oracleReorder(vs []int) []int {
	var out []int
	for i, j := 0, len(vs)-1; i <= j; i, j = i+1, j-1 {
		out = append(out, vs[i])
		if i != j {
			out = append(out, vs[j])
		}
	}
	return out
}

func mallocs() uint64 {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.Mallocs
}

func best(reps int, f func()) time.Duration {
	m := time.Duration(1<<63 - 1)
	for range reps {
		t0 := time.Now()
		f()
		m = min(m, time.Since(t0))
	}
	return m
}

func main() {
	// ---- 1. exhaustive: every list of length 0..12 over {0, 1} ----
	lists, wrong, modified, pals := 0, 0, 0, 0
	brokenByNoRestore := 0
	for n := 0; n <= 12; n++ {
		for bits := 0; bits < 1<<n; bits++ {
			vs := make([]int, n)
			for i := range vs {
				vs[i] = bits >> i & 1
			}
			head := fromSlice(vs)
			before := take(head)
			got := isPalindrome(head)
			lists++
			if got != oraclePal(vs) {
				wrong++
			}
			if got {
				pals++
			}
			if !take(head).same(before) {
				modified++
			}
			head2 := fromSlice(vs)
			before2 := take(head2)
			isPalindromeNoRestore(head2)
			if !take(head2).same(before2) {
				brokenByNoRestore++
			}
		}
	}
	fmt.Println("1. isPalindrome, every list of length 0..12 over {0,1}:")
	fmt.Printf("   %d lists (%d palindromes), %d wrong answers, %d lists left modified\n",
		lists, pals, wrong, modified)

	// ---- 2. the bug when you skip the restore ----
	head := fromSlice([]int{1, 2, 2, 1})
	all := take(head)
	fmt.Println()
	fmt.Println("2. the same function without `mid.next = reverse(second)`:")
	fmt.Printf("   before: %s\n", show(head))
	fmt.Printf("   answer: %v\n", isPalindromeNoRestore(head))
	after := take(head)
	fmt.Printf("   after:  %s   <- what the head reaches now (%d of %d nodes)\n",
		show(head), len(after.ptrs), len(all.ptrs))
	for i, x := range all.ptrs {
		if !slices.Contains(after.ptrs, x) {
			fmt.Printf("   node %d (val %d) is unreachable from the head; it now points to %s\n",
				i, x.val, show(x))
		}
	}
	fmt.Printf("   over all %d lists, the no-restore version left %d of them modified\n",
		lists, brokenByNoRestore)

	// ---- 3. reorder, checked against a slice oracle ----
	mismatches, allocs := 0, uint64(0)
	for n := 0; n <= 12; n++ {
		vs := make([]int, n)
		for i := range vs {
			vs[i] = i
		}
		head := fromSlice(vs)
		before := take(head)
		m0 := mallocs()
		reorder(head)
		allocs += mallocs() - m0
		got := take(head)
		if !slices.Equal(got.vals, oracleReorder(vs)) {
			mismatches++
		}
		slices.SortFunc(got.ptrs, func(a, b *node) int { return a.val - b.val })
		if !slices.Equal(got.ptrs, before.ptrs) { // the same nodes, rearranged
			mismatches++
		}
		if n == 7 {
			fmt.Println()
			fmt.Printf("3. reorder %s\n        -> %s\n", show(fromSlice(vs)), show(head))
		}
	}
	fmt.Printf("   lengths 0..12: %d mismatches against the oracle, %d allocations inside reorder\n",
		mismatches, allocs)

	// ---- 4. cost on a 1,000,000-node palindrome ----
	const n = 1_000_000
	vs := make([]int, n)
	for i := range vs {
		vs[i] = min(i, n-1-i)
	}
	big := fromSlice(vs)
	fmt.Println()
	fmt.Printf("4. a %d-node palindrome\n", n)

	before := take(big)
	// cost runs f once and reports its hops, heap allocations and bytes.
	cost := func(f func(*node) bool) (bool, int, uint64, uint64) {
		var m0, m1 runtime.MemStats
		runtime.GC()
		runtime.ReadMemStats(&m0)
		hops = 0
		ok := f(big)
		runtime.ReadMemStats(&m1)
		return ok, hops, m1.Mallocs - m0.Mallocs, m1.TotalAlloc - m0.TotalAlloc
	}
	fmt.Printf("   %-26s %6s %12s %6s %12s\n", "version", "answer", "hops", "allocs", "bytes")
	for _, v := range []struct {
		name string
		f    func(*node) bool
	}{{"in place (reverse + undo)", isPalindrome}, {"copy into a slice", isPalindromeCopy}} {
		ok, h, allocs, bytes := cost(v.f)
		fmt.Printf("   %-26s %6v %12d %6d %12d   (%.2f hops per node)\n",
			v.name, ok, h, allocs, bytes, float64(h)/n)
	}

	tIn := best(10, func() { isPalindrome(big) })
	tCp := best(10, func() { isPalindromeCopy(big) })
	fmt.Printf("   time (best of 10): in place %.2f ms, copy %.2f ms -> in place takes %.2fx the time\n",
		float64(tIn.Microseconds())/1e3, float64(tCp.Microseconds())/1e3, float64(tIn)/float64(tCp))
	fmt.Printf("   after 11 in-place calls the list is still the same %d nodes in order: %v\n",
		n, take(big).same(before))
}
```

**Sample output:**

```
1. isPalindrome, every list of length 0..12 over {0,1}:
   8191 lists (253 palindromes), 0 wrong answers, 0 lists left modified

2. the same function without `mid.next = reverse(second)`:
   before: [1 -> 2 -> 2 -> 1]
   answer: true
   after:  [1 -> 2 -> 2]   <- what the head reaches now (3 of 4 nodes)
   node 3 (val 1) is unreachable from the head; it now points to [1 -> 2]
   over all 8191 lists, the no-restore version left 8176 of them modified

3. reorder [0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 6]
        -> [0 -> 6 -> 1 -> 5 -> 2 -> 4 -> 3]
   lengths 0..12: 0 mismatches against the oracle, 0 allocations inside reorder

4. a 1000000-node palindrome
   version                    answer         hops allocs        bytes
   in place (reverse + undo)    true      3499997      0            0   (3.50 hops per node)
   copy into a slice            true      2000000      1      8003584   (2.00 hops per node)
   time (best of 10): in place 2.29 ms, copy 2.64 ms -> in place takes 0.86x the time
   after 11 in-place calls the list is still the same 1000000 nodes in order: true
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::mem;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// isPalindrome in Θ(1) extra space: count, walk to the first middle, reverse
// the second half, compare, reverse it back and reattach it. In Rust the
// split is explicit: `mid.next.take()` hands us the second half as an OWNED
// list, so reversing it is an ordinary `reverse(second)` call -- and if the
// function forgets to give it back, the half is not dangling, it is DROPPED
// (freed) when the function returns. The counting allocator makes that visible.

struct Node {
    val: i64,
    next: Link,
}
type Link = Option<Box<Node>>;

// ---- counting allocator: allocations, frees, bytes ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static FREES: AtomicUsize = AtomicUsize::new(0);
static TOTAL: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        FREES.fetch_add(1, Relaxed);
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

/// (allocations, frees, bytes requested) while running `f`.
fn heap_of<R>(f: impl FnOnce() -> R) -> (usize, usize, usize, R) {
    let (a, d, t) = (
        ALLOCS.load(Relaxed),
        FREES.load(Relaxed),
        TOTAL.load(Relaxed),
    );
    let r = f();
    let (a2, d2, t2) = (
        ALLOCS.load(Relaxed),
        FREES.load(Relaxed),
        TOTAL.load(Relaxed),
    );
    (a2 - a, d2 - d, t2 - t, r)
}

fn from_slice(vs: &[i64]) -> Link {
    vs.iter()
        .rev()
        .fold(None, |next, &val| Some(Box::new(Node { val, next })))
}

fn iter(list: &Link) -> impl Iterator<Item = &Node> {
    std::iter::successors(list.as_deref(), |n| n.next.as_deref())
}

fn show(list: &Link) -> String {
    let parts: Vec<String> = iter(list).map(|n| n.val.to_string()).collect();
    format!("[{}]", parts.join(" -> "))
}

// "Restored" means the same nodes (addresses), in the same order, with the
// same values. Comparing addresses is safe; only dereferencing would not be.
fn snapshot(list: &Link) -> (Vec<*const Node>, Vec<i64>) {
    iter(list).map(|n| (n as *const Node, n.val)).unzip()
}

// Every `x = x.next` below counts one hop; the counter is threaded through
// as `&mut u64` because Rust has no mutable globals in safe code.
fn length(list: &Link, hops: &mut u64) -> usize {
    let mut n = 0;
    let mut cur = list.as_deref();
    while let Some(node) = cur {
        n += 1;
        *hops += 1;
        cur = node.next.as_deref();
    }
    n
}

// Save, flip, advance -- in one `mem::replace`.
fn reverse(mut cur: Link, hops: &mut u64) -> Link {
    let mut prev = None;
    while let Some(mut node) = cur {
        cur = mem::replace(&mut node.next, prev);
        prev = Some(node);
        *hops += 1;
    }
    prev
}

// Go finds the middle with slow/fast pointers. In Rust we need a MUTABLE
// cursor at the middle (to take its `next`), and a second cursor into the
// same chain while the first is mutable is exactly what the borrow checker
// refuses. So: count first, then walk (n-1)/2 steps with one &mut cursor.
fn split_after_first_middle(list: &mut Link, hops: &mut u64) -> Option<Link> {
    let n = length(list, hops);
    if n < 2 {
        return None;
    }
    let mut mid = list.as_mut().expect("n >= 2");
    for _ in 0..(n - 1) / 2 {
        mid = mid.next.as_mut().expect("inside the list");
        *hops += 1;
    }
    Some(mid.next.take()) // two owned lists from here on
}

fn halves_equal(first: &Link, second: &Link, hops: &mut u64) -> bool {
    let (mut p, mut q) = (first.as_deref(), second.as_deref());
    while let (Some(a), Some(b)) = (p, q) {
        *hops += 2;
        if a.val != b.val {
            return false;
        }
        p = a.next.as_deref();
        q = b.next.as_deref();
    }
    true
}

// The append-through-a-cursor walk: `cur` always points at the link that
// reaches the current node, so at the end it IS the empty tail link.
fn reattach(list: &mut Link, half: Link, hops: &mut u64) {
    let mut cur = list;
    while let Some(node) = cur {
        cur = &mut node.next;
        *hops += 1;
    }
    *cur = half;
}

fn is_palindrome(list: &mut Link, hops: &mut u64) -> bool {
    let Some(second) = split_after_first_middle(list, hops) else {
        return true;
    };
    let second = reverse(second, hops);
    let ok = halves_equal(list, &second, hops);
    let back = reverse(second, hops);
    reattach(list, back, hops); // PUT IT BACK
    ok
}

// The same function without the two lines before `ok`. It still compiles:
// `second` is simply dropped at the closing brace.
fn is_palindrome_no_restore(list: &mut Link, hops: &mut u64) -> bool {
    let Some(second) = split_after_first_middle(list, hops) else {
        return true;
    };
    let second = reverse(second, hops);
    halves_equal(list, &second, hops)
}

// The Θ(n)-space version: copy into a Vec, compare from both ends.
fn is_palindrome_copy(list: &Link, hops: &mut u64) -> bool {
    let n = length(list, hops);
    let mut vals = Vec::with_capacity(n);
    for node in iter(list) {
        vals.push(node.val);
        *hops += 1;
    }
    vals.iter().take(n / 2).eq(vals.iter().rev().take(n / 2))
}

// reorder: L0, Ln, L1, Ln-1, ... Split, reverse the second half, then move
// nodes alternately into the output. `Option::insert` puts a node into the
// tail link and returns a &mut to it, so the tail cursor advances in one line.
fn reorder(list: &mut Link, hops: &mut u64) {
    let Some(second) = split_after_first_middle(list, hops) else {
        return;
    };
    let mut b = reverse(second, hops);
    let mut a = list.take();
    let mut tail = list;
    while let Some(mut x) = a {
        a = x.next.take();
        tail = &mut tail.insert(x).next;
        if let Some(mut y) = b {
            b = y.next.take();
            tail = &mut tail.insert(y).next;
        }
    }
}

fn oracle_pal(vs: &[i64]) -> bool {
    vs.iter().eq(vs.iter().rev())
}

fn oracle_reorder(vs: &[i64]) -> Vec<i64> {
    let mut out = Vec::new();
    let (mut i, mut j) = (0, vs.len() as isize - 1);
    while i <= j {
        out.push(vs[i as usize]);
        if i != j {
            out.push(vs[j as usize]);
        }
        i += 1;
        j -= 1;
    }
    out
}

// A 1,000,000-node chain must not be freed by the compiler-generated
// recursive drop (one stack frame per node): free it in a loop.
struct List(Link);

impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.0.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
        }
    }
}

fn best(reps: usize, mut f: impl FnMut()) -> Duration {
    (0..reps)
        .map(|_| {
            let t = Instant::now();
            f();
            t.elapsed()
        })
        .min()
        .expect("reps > 0")
}

fn main() {
    let mut dummy = 0u64;

    // ---- 1. exhaustive: every list of length 0..12 over {0, 1} ----
    let (mut lists, mut wrong, mut modified, mut pals) = (0, 0, 0, 0);
    let (mut heap_allocs, mut heap_frees) = (0, 0);
    let mut broken_by_no_restore = 0;
    for n in 0..=12 {
        for bits in 0..1u32 << n {
            let vs: Vec<i64> = (0..n).map(|i| i64::from(bits >> i & 1)).collect();
            let mut list = from_slice(&vs);
            let before = snapshot(&list);
            let (a, f, _, got) = heap_of(|| is_palindrome(&mut list, &mut dummy));
            heap_allocs += a;
            heap_frees += f;
            lists += 1;
            wrong += usize::from(got != oracle_pal(&vs));
            pals += usize::from(got);
            modified += usize::from(snapshot(&list) != before);

            let mut list2 = from_slice(&vs);
            let before2 = snapshot(&list2);
            is_palindrome_no_restore(&mut list2, &mut dummy);
            broken_by_no_restore += usize::from(snapshot(&list2) != before2);
        }
    }
    println!("1. isPalindrome, every list of length 0..12 over {{0,1}}:");
    println!(
        "   {lists} lists ({pals} palindromes), {wrong} wrong answers, {modified} lists left modified"
    );
    println!("   inside all those calls: {heap_allocs} allocations, {heap_frees} frees");

    // ---- 2. the bug when you skip the restore ----
    let mut list = from_slice(&[1, 2, 2, 1]);
    let all = snapshot(&list).0.len();
    println!();
    println!("2. the same function without `reverse` + `reattach` at the end:");
    println!("   before: {}", show(&list));
    let (_, frees, _, answer) = heap_of(|| is_palindrome_no_restore(&mut list, &mut dummy));
    println!("   answer: {answer}");
    println!(
        "   after:  {}   <- what the head reaches now ({} of {all} nodes)",
        show(&list),
        snapshot(&list).0.len()
    );
    println!(
        "   nothing is dangling: the {frees} missing nodes were FREED when the function returned"
    );
    println!(
        "   over all {lists} lists, the no-restore version left {broken_by_no_restore} of them modified"
    );

    // ---- 3. reorder, checked against a slice oracle ----
    let (mut mismatches, mut allocs) = (0, 0);
    for n in 0..=12i64 {
        let vs: Vec<i64> = (0..n).collect();
        let mut list = from_slice(&vs);
        let (before_ptrs, _) = snapshot(&list);
        let (a, _, _, ()) = heap_of(|| reorder(&mut list, &mut dummy));
        allocs += a;
        let (ptrs, vals) = snapshot(&list);
        mismatches += usize::from(vals != oracle_reorder(&vs));
        // the same nodes, rearranged: sort the addresses back by value
        let mut pairs: Vec<(i64, *const Node)> = vals.into_iter().zip(ptrs).collect();
        pairs.sort_by_key(|&(v, _)| v);
        let sorted: Vec<*const Node> = pairs.into_iter().map(|(_, p)| p).collect();
        mismatches += usize::from(sorted != before_ptrs);
        if n == 7 {
            println!();
            println!(
                "3. reorder {}\n        -> {}",
                show(&from_slice(&vs)),
                show(&list)
            );
        }
    }
    println!(
        "   lengths 0..12: {mismatches} mismatches against the oracle, {allocs} allocations inside reorder"
    );

    // ---- 4. cost on a 1,000,000-node palindrome ----
    const N: usize = 1_000_000;
    let vs: Vec<i64> = (0..N).map(|i| i.min(N - 1 - i) as i64).collect();
    let mut big = List(from_slice(&vs));
    let before = snapshot(&big.0);
    println!();
    println!("4. a {N}-node palindrome");
    println!(
        "   {:<26} {:>6} {:>12} {:>6} {:>6} {:>12}",
        "version", "answer", "hops", "allocs", "frees", "bytes"
    );
    let row = |name: &str, (a, f, b, (ok, h)): (usize, usize, usize, (bool, u64))| {
        println!(
            "   {name:<26} {ok:>6} {h:>12} {a:>6} {f:>6} {b:>12}   ({:.2} hops per node)",
            h as f64 / N as f64
        );
    };
    row(
        "in place (reverse + undo)",
        heap_of(|| {
            let mut h = 0;
            (is_palindrome(&mut big.0, &mut h), h)
        }),
    );
    row(
        "copy into a Vec",
        heap_of(|| {
            let mut h = 0;
            (is_palindrome_copy(&big.0, &mut h), h)
        }),
    );

    let t_in = best(10, || {
        std::hint::black_box(is_palindrome(&mut big.0, &mut dummy));
    });
    let t_cp = best(10, || {
        std::hint::black_box(is_palindrome_copy(&big.0, &mut dummy));
    });
    println!(
        "   time (best of 10): in place {:.2} ms, copy {:.2} ms -> in place takes {:.2}x the time",
        t_in.as_secs_f64() * 1e3,
        t_cp.as_secs_f64() * 1e3,
        t_in.as_secs_f64() / t_cp.as_secs_f64()
    );
    println!(
        "   after 11 in-place calls the list is still the same {N} nodes in order: {}",
        snapshot(&big.0) == before
    );
}
```

**Rust sample output:**

```
1. isPalindrome, every list of length 0..12 over {0,1}:
   8191 lists (253 palindromes), 0 wrong answers, 0 lists left modified
   inside all those calls: 0 allocations, 0 frees

2. the same function without `reverse` + `reattach` at the end:
   before: [1 -> 2 -> 2 -> 1]
   answer: true
   after:  [1 -> 2]   <- what the head reaches now (2 of 4 nodes)
   nothing is dangling: the 2 missing nodes were FREED when the function returned
   over all 8191 lists, the no-restore version left 8188 of them modified

3. reorder [0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 6]
        -> [0 -> 6 -> 1 -> 5 -> 2 -> 4 -> 3]
   lengths 0..12: 0 mismatches against the oracle, 0 allocations inside reorder

4. a 1000000-node palindrome
   version                    answer         hops allocs  frees        bytes
   in place (reverse + undo)    true      3999999      0      0            0   (4.00 hops per node)
   copy into a Vec              true      2000000      1      1      8000000   (2.00 hops per node)
   time (best of 10): in place 2.38 ms, copy 2.30 ms -> in place takes 1.03x the time
   after 11 in-place calls the list is still the same 1000000 nodes in order: true
```

> **Go vs Rust:** Escape hatches: `Option::take` (`mid.next.take()` splits the list into two *owned* halves), `mem::replace` for the reverse, and the `&mut Option<Box<_>>` cursor to reattach. Go's slow/fast middle needs a second cursor into a chain that one `&mut` cursor already holds, which the borrow checker refuses, so the twin counts first and walks to the tail again to reattach: 4.00 hops per node against Go's 3.50. Sections 1 and 3 print Go's counts (8,191 / 253 / 0 / 0, plus 0 allocations and 0 frees inside every call).
> The bug is different in kind. In Go the skipped restore leaves `[1 -> 2 -> 2]` and an unreachable node for the GC; in Rust the dropped half is *freed* at the closing brace, so the list becomes `[1 -> 2]` and lists of length 2 and 3 are damaged too (8,188 modified against Go's 8,176). The signature `fn is_palindrome(list: &mut Link) -> bool` does not catch the bug, but the free count and the snapshot do. On 1,000,000 nodes neither Rust version won consistently: the ratio stayed near 1× (0.89–1.11× across six runs) and changed sides between runs, unlike Go, where the in-place version was faster every time. Rust's copy also requests exactly 8,000,000 bytes where Go's allocator rounded the request up to 8,003,584.

**Complexity:** Θ(n) time, Θ(1) extra space, 0 allocations, input restored. The copy-into-a-slice version is also Θ(n) time but Θ(n) space (one allocation of n words). Deterministic: 8,191 lists, 0 wrong answers, 0 left modified. Without the restore, 8,176 of them (every list of length 4 or more) come back changed, and `[1 -> 2 -> 2 -> 1]` reads `[1 -> 2 -> 2]` with its last node cut off. `reorder` has 0 mismatches for lengths 0..12 and makes 0 allocations. On 1,000,000 nodes the in-place version takes 3.5 hops per node against the copy's 2.0, and it was still faster in every run (0.76–0.87× the copy's time): the copy pays for an 8 MB allocation on every call (plausible mechanism, not measured).

---

## 15. The unrolled list

`🔴 hard` · *an array in each node: one cache miss buys k elements*

A plain list pays one hop, often a cache miss, for every element. Put k = 32 values in each node and one hop
delivers 32 of them. You get most of a slice's locality back and keep cheap insertion inside a node: shift at
most k values, and split a full node in half (the same idea as a B-tree, lesson 19). Traversal is Θ(n) with
n/k hops, `At(i)` is Θ(i/k), and `InsertAt` is Θ(i/k + k). The program checks the structure against a slice,
counts its allocations, and then times it, and the timing shows the limit of the idea. Unrolling makes the
*shift* about 2,000× smaller. It does nothing about the *search*: on random positions, that walk costs more
than the slice's memmove.

**Steps:**

1. Write `chunk{n, vals [32]int, next}` with `Append` (keep a tail pointer), `InsertAt` (split a full chunk:
   its upper half moves into a new chunk), `At` and `Sum`.
2. Mirror 100,000 random `InsertAt`s on a `[]int` oracle, counting chunk hops and values moved on both sides.
3. Compare allocations for 1,000,000 ints, then time traversal (slice, fresh list, aged list, two unrolled
   lists) and random `InsertAt` against `slices.Insert`.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"testing"
	"time"
)

// K values per node. 32 ints = 256 bytes = four 64-byte cache lines: one hop
// (one likely cache miss) now delivers 32 elements instead of 1.
const K = 32

type chunk struct {
	n    int    // how many of vals are in use (vals[:n])
	vals [K]int // inline array: part of the node, not a separate allocation
	next *chunk
}

// Unrolled is a singly linked list of chunks. Every chunk except possibly the
// last is non-empty; chunks may be partly full after a split.
type Unrolled struct {
	head, tail *chunk
	length     int
	chunks     int
	hops       int // chunk-to-chunk moves made by InsertAt (the search cost)
	shifted    int // values moved inside chunks by InsertAt (the shift cost)
}

// Append is Θ(1): fill the tail chunk, start a new one when it is full.
func (u *Unrolled) Append(v int) {
	if u.tail == nil || u.tail.n == K {
		c := &chunk{}
		if u.tail == nil {
			u.head = c
		} else {
			u.tail.next = c
		}
		u.tail = c
		u.chunks++
	}
	u.tail.vals[u.tail.n] = v
	u.tail.n++
	u.length++
}

// InsertAt puts v at position i (0 <= i <= length). Θ(i/K) to find the chunk,
// then Θ(K) to shift inside it; a full chunk is split in half first.
func (u *Unrolled) InsertAt(i, v int) {
	if i == u.length {
		u.Append(v)
		return
	}
	c := u.head
	for i > c.n || (i == c.n && c.n == K) { // i == c.n on a non-full chunk: append to it
		i -= c.n
		c = c.next
		u.hops++
	}
	if c.n == K { // split: the upper half moves into a new chunk after c
		nc := &chunk{n: K - K/2, next: c.next}
		copy(nc.vals[:], c.vals[K/2:])
		c.n = K / 2
		c.next = nc
		if u.tail == c {
			u.tail = nc
		}
		u.chunks++
		u.shifted += K - K/2
		if i > c.n {
			i -= c.n
			c = nc
		}
	}
	copy(c.vals[i+1:c.n+1], c.vals[i:c.n]) // the shift: at most K-1 values
	u.shifted += c.n - i
	c.vals[i] = v
	c.n++
	u.length++
}

// At is Θ(i/K): skip whole chunks, then index into one.
func (u *Unrolled) At(i int) int {
	c := u.head
	for i >= c.n {
		i -= c.n
		c = c.next
	}
	return c.vals[i]
}

func (u *Unrolled) Sum() (s int) {
	for c := u.head; c != nil; c = c.next {
		for _, v := range c.vals[:c.n] {
			s += v
		}
	}
	return s
}

func (u *Unrolled) Slice() []int {
	out := make([]int, 0, u.length)
	for c := u.head; c != nil; c = c.next {
		out = append(out, c.vals[:c.n]...)
	}
	return out
}

func (u *Unrolled) fill() float64 { return float64(u.length) / float64(u.chunks*K) * 100 }

// The plain list, for comparison: one value and one allocation per node.
type node struct {
	val  int
	next *node
}

// plainList links values in order: one allocation per node, in list order.
func plainList(vals []int) (head *node) {
	for i := len(vals) - 1; i >= 0; i-- {
		head = &node{vals[i], head}
	}
	return head
}

// agedList allocates the nodes in order but links them in a PCG-shuffled
// order, like a list that has lived through churn: each hop jumps in memory.
func agedList(vals []int) (head *node) {
	nodes := make([]*node, len(vals))
	for i := range nodes {
		nodes[i] = &node{}
	}
	rng := rand.New(rand.NewPCG(3, 4))
	for i := len(nodes) - 1; i > 0; i-- { // Fisher-Yates, by hand
		j := rng.IntN(i + 1)
		nodes[i], nodes[j] = nodes[j], nodes[i]
	}
	for i := len(nodes) - 1; i >= 0; i-- {
		nodes[i].val, nodes[i].next = vals[i], head
		head = nodes[i]
	}
	return head
}

func listSum(h *node) (s int) {
	for ; h != nil; h = h.next {
		s += h.val
	}
	return s
}

func mallocs(f func()) (objs, bytes uint64) {
	var a, b runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&a)
	f()
	runtime.ReadMemStats(&b)
	return b.Mallocs - a.Mallocs, b.TotalAlloc - a.TotalAlloc
}

var sink int

func main() {
	// ---- 1. correctness: random inserts mirrored on a slice oracle ----
	const n1 = 100_000
	rng := rand.New(rand.NewPCG(1, 2))
	var u Unrolled
	var oracle []int
	oracleShift := 0
	for v := range n1 {
		i := rng.IntN(len(oracle) + 1)
		u.InsertAt(i, v)
		oracleShift += len(oracle) - i
		oracle = slices.Insert(oracle, i, v)
	}
	same := slices.Equal(u.Slice(), oracle)
	atOK := 0
	for range 1000 {
		i := rng.IntN(n1)
		if u.At(i) == oracle[i] {
			atOK++
		}
	}
	fmt.Println("1. correctness: 100000 InsertAt at PCG-random positions, mirrored on a []int")
	fmt.Printf("   contents identical: %v   At(i) agrees on %d/1000 random positions\n", same, atOK)
	fmt.Printf("   chunks: %d   average fill: %.1f%% of %d slots\n", u.chunks, u.fill(), K)
	fmt.Printf("   unrolled: %10d chunk hops + %10d values moved (%.1f + %.1f per insert)\n",
		u.hops, u.shifted, float64(u.hops)/n1, float64(u.shifted)/n1)
	fmt.Printf("   slice:    %10d chunk hops + %10d values moved (%.1f per insert)\n",
		0, oracleShift, float64(oracleShift)/n1)
	fmt.Printf("   -> the slice moved %.0fx more values; the unrolled list pays in hops instead\n",
		float64(oracleShift)/float64(u.shifted))

	// ---- 2. node counts: allocations to hold 1,000,000 ints ----
	const n2 = 1_000_000
	vals := make([]int, n2)
	for i := range vals {
		vals[i] = i
	}
	var big Unrolled
	uo, ub := mallocs(func() {
		for _, v := range vals {
			big.Append(v)
		}
	})
	var plain *node
	po, pb := mallocs(func() { plain = plainList(vals) })
	var s []int
	so, sb := mallocs(func() {
		for _, v := range vals {
			s = append(s, v)
		}
	})
	fmt.Println("\n2. allocations to build 1000000 ints (MemStats Mallocs / TotalAlloc)")
	fmt.Printf("   %-26s %10s %14s\n", "", "objects", "bytes")
	fmt.Printf("   %-26s %10d %14d   (%d chunks = ceil(n/32))\n", "unrolled, Append", uo, ub, big.chunks)
	fmt.Printf("   %-26s %10d %14d   (one per node)\n", "plain list, prepend", po, pb)
	fmt.Printf("   %-26s %10d %14d   (append growth steps)\n", "slice, append", so, sb)
	fmt.Printf("   -> %.0fx fewer objects than the plain list\n", float64(po)/float64(uo))
	runtime.KeepAlive(plain)

	// ---- 3. timing: traversal at n = 100,000, and InsertAt ----
	fresh := Unrolled{}
	for _, v := range oracle {
		fresh.Append(v)
	}
	listFresh, listAged := plainList(oracle), agedList(oracle)
	want := listSum(listFresh)
	if fresh.Sum() != want || u.Sum() != want || listSum(listAged) != want {
		panic("checksums differ")
	}
	perElem := func(f func() int) float64 {
		r := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				sink += f()
			}
		})
		return float64(r.NsPerOp()) / n1
	}
	slSum := func() (t int) {
		for _, v := range oracle {
			t += v
		}
		return t
	}
	rows := []struct {
		name string
		f    func() int
	}{
		{"slice", slSum},
		{"plain list, fresh", func() int { return listSum(listFresh) }},
		{"plain list, aged", func() int { return listSum(listAged) }},
		{"unrolled, fresh (100% full)", fresh.Sum},
		{fmt.Sprintf("unrolled, random (%.0f%% full)", u.fill()), u.Sum},
	}
	fmt.Println("\n3. timing (checksums of all five traversals agree)")
	fmt.Printf("   %-34s %10s %8s\n", "traversal sum, n = 100000", "ns/elem", "x slice")
	var base float64
	ratio := map[string]float64{}
	for i, r := range rows {
		ns := perElem(r.f)
		if i == 0 {
			base = ns
		}
		ratio[r.name] = ns / base
		fmt.Printf("   %-34s %10.3f %8.1f\n", r.name, ns, ns/base)
	}

	// InsertAt: 10,000 random inserts into a fresh 100,000-element copy, timed
	// without the rebuild (the insert mutates, so every round starts pristine).
	const m = 10_000
	pos := make([]int, m)
	for k := range pos {
		pos[k] = rng.IntN(n1 + k + 1)
	}
	var tSlice, tUnr time.Duration
	const rounds = 5
	for range rounds {
		sl := slices.Clone(oracle)
		t0 := time.Now()
		for k, p := range pos {
			sl = slices.Insert(sl, p, k)
		}
		tSlice += time.Since(t0)
		var w Unrolled
		for _, v := range oracle {
			w.Append(v)
		}
		t0 = time.Now()
		for k, p := range pos {
			w.InsertAt(p, k)
		}
		tUnr += time.Since(t0)
		if !slices.Equal(w.Slice(), sl) {
			panic("insert results differ")
		}
	}
	nsS := float64(tSlice.Nanoseconds()) / (rounds * m)
	nsU := float64(tUnr.Nanoseconds()) / (rounds * m)
	fmt.Printf("   %-34s %10s %8s\n", "InsertAt random, n = 100000..110000", "ns/insert", "x slice")
	fmt.Printf("   %-34s %10.0f %8.1f\n", "slices.Insert", nsS, 1.0)
	fmt.Printf("   %-34s %10.0f %8.1f\n", "unrolled InsertAt", nsU, nsU/nsS)

	fmt.Printf("\nverdict: unrolled traversal takes %.1fx a slice's time (plain list: %.1fx fresh, %.1fx aged);\n",
		ratio["unrolled, fresh (100% full)"], ratio["plain list, fresh"], ratio["plain list, aged"])
	hopsPer, movesPer := float64(u.hops)/n1, float64(oracleShift)/n1
	if nsU > nsS {
		fmt.Printf("InsertAt is %.1fx SLOWER than slices.Insert. Its shift is tiny (%.1f values per\n",
			nsU/nsS, float64(u.shifted)/n1)
		fmt.Printf("insert), so its time is the walk: ~%.0f dependent hops cost more than one memmove\n", hopsPer)
		fmt.Printf("of ~%.0f contiguous values. Both are Θ(n) per insert:\n", movesPer)
		fmt.Println("unrolling shrinks the shift, not the search.")
	} else {
		fmt.Printf("InsertAt is %.1fx faster than slices.Insert: ~%.0f hops beat a memmove of ~%.0f values.\n",
			nsS/nsU, hopsPer, movesPer)
	}
}
```

**Sample output:**

```
1. correctness: 100000 InsertAt at PCG-random positions, mirrored on a []int
   contents identical: true   At(i) agrees on 1000/1000 random positions
   chunks: 4455   average fill: 70.1% of 32 slots
   unrolled:  111361418 chunk hops +    1151685 values moved (1113.6 + 11.5 per insert)
   slice:             0 chunk hops + 2500561616 values moved (25005.6 per insert)
   -> the slice moved 2171x more values; the unrolled list pays in hops instead

2. allocations to build 1000000 ints (MemStats Mallocs / TotalAlloc)
                                 objects          bytes
   unrolled, Append                31250        9000000   (31250 chunks = ceil(n/32))
   plain list, prepend           1000000       16000000   (one per node)
   slice, append                      44       41683400   (append growth steps)
   -> 32x fewer objects than the plain list

3. timing (checksums of all five traversals agree)
   traversal sum, n = 100000             ns/elem  x slice
   slice                                   0.277      1.0
   plain list, fresh                       1.040      3.8
   plain list, aged                        5.887     21.3
   unrolled, fresh (100% full)             0.293      1.1
   unrolled, random (70% full)             0.641      2.3
   InsertAt random, n = 100000..110000  ns/insert  x slice
   slices.Insert                            6355      1.0
   unrolled InsertAt                       15861      2.5

verdict: unrolled traversal takes 1.1x a slice's time (plain list: 3.8x fresh, 21.3x aged);
InsertAt is 2.5x SLOWER than slices.Insert. Its shift is tiny (11.5 values per
insert), so its time is the walk: ~1114 dependent hops cost more than one memmove
of ~25006 contiguous values. Both are Θ(n) per insert:
unrolling shrinks the shift, not the search.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::iter::successors;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- counting allocator: stand-in for runtime.MemStats Mallocs / TotalAlloc ----
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

// ---- ns/op: stand-in for testing.Benchmark ----
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(300);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN: same seed, same stream as Go.
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

// K values per node: 32 i64 = 256 bytes = four cache lines per hop.
const K: usize = 32;

// The inline array needs a concrete Copy type ([i64; 32] is zero-filled on
// creation); MaybeUninit would avoid that, at the price of unsafe.
struct Chunk {
    len: usize,
    vals: [i64; K],
    next: Option<Box<Chunk>>,
}

impl Chunk {
    fn new() -> Box<Chunk> {
        Box::new(Chunk {
            len: 0,
            vals: [0; K],
            next: None,
        })
    }
}

// No tail pointer: in safe Rust the head owns the whole chain, so a second
// `&mut` to the last chunk cannot be stored next to it. `extend` walks to the
// end ONCE and then appends many values through one `&mut` cursor.
#[derive(Default)]
struct Unrolled {
    head: Option<Box<Chunk>>,
    len: usize,
    chunks: usize,
    hops: usize,    // chunk-to-chunk moves made by insert_at (the search cost)
    shifted: usize, // values moved inside chunks by insert_at (the shift cost)
}

impl Unrolled {
    fn extend(&mut self, vals: &[i64]) {
        if vals.is_empty() {
            return;
        }
        if self.head.is_none() {
            self.head = Some(Chunk::new());
            self.chunks += 1;
        }
        let mut c = self.head.as_mut().unwrap();
        while c.next.is_some() {
            c = c.next.as_mut().unwrap();
        }
        for &v in vals {
            if c.len == K {
                c.next = Some(Chunk::new());
                c = c.next.as_mut().unwrap();
                self.chunks += 1;
            }
            c.vals[c.len] = v;
            c.len += 1;
        }
        self.len += vals.len();
    }

    /// Put v at position i (i <= len): Θ(i/K) walk, then a shift of at most K-1.
    fn insert_at(&mut self, mut i: usize, v: i64) {
        assert!(i <= self.len, "insert_at past the end");
        if i == self.len {
            return self.extend(&[v]);
        }
        let mut c = self.head.as_mut().unwrap();
        // i == c.len on a non-full chunk: append to it
        while i > c.len || (i == c.len && c.len == K) {
            i -= c.len;
            c = c.next.as_mut().unwrap(); // reborrow moves forward: no E0506 here
            self.hops += 1;
        }
        if c.len == K {
            // split: the upper half moves into a new chunk after c
            let mut nc = Chunk::new();
            nc.vals[..K - K / 2].copy_from_slice(&c.vals[K / 2..]);
            nc.len = K - K / 2;
            nc.next = c.next.take();
            c.len = K / 2;
            c.next = Some(nc);
            self.chunks += 1;
            self.shifted += K - K / 2;
            if i > c.len {
                i -= c.len;
                c = c.next.as_mut().unwrap();
            }
        }
        let n = c.len;
        c.vals.copy_within(i..n, i + 1); // the shift: a memmove of at most K-1 values
        self.shifted += n - i;
        c.vals[i] = v;
        c.len += 1;
        self.len += 1;
    }

    fn chunk_slices(&self) -> impl Iterator<Item = &[i64]> {
        successors(self.head.as_deref(), |c| c.next.as_deref()).map(|c| &c.vals[..c.len])
    }

    /// Θ(i/K): skip whole chunks, then index into one. None past the end.
    fn get(&self, mut i: usize) -> Option<i64> {
        for s in self.chunk_slices() {
            if i < s.len() {
                return Some(s[i]);
            }
            i -= s.len();
        }
        None
    }

    fn sum(&self) -> i64 {
        self.chunk_slices().map(|s| s.iter().sum::<i64>()).sum()
    }

    fn to_vec(&self) -> Vec<i64> {
        self.chunk_slices().flatten().copied().collect()
    }

    fn fill(&self) -> f64 {
        self.len as f64 / (self.chunks * K) as f64 * 100.0
    }
}

// Iterative Drop: the default drop recurses once per Box in the chain.
impl Drop for Unrolled {
    fn drop(&mut self) {
        let mut cur = self.head.take();
        while let Some(mut c) = cur {
            cur = c.next.take();
        }
    }
}

// The plain list, for comparison: one value and one allocation per node.
struct Node {
    val: i64,
    next: Option<Box<Node>>,
}

struct List(Option<Box<Node>>);

impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.0.take();
        while let Some(mut n) = cur {
            cur = n.next.take(); // iterative: 1,000,000 recursive drops would overflow
        }
    }
}

impl List {
    /// one allocation per node, in list order
    fn plain(vals: &[i64]) -> List {
        let mut head = None;
        for &val in vals.iter().rev() {
            head = Some(Box::new(Node { val, next: head }));
        }
        List(head)
    }

    /// nodes allocated in order, linked in a PCG-shuffled order: each hop jumps.
    /// Ownership makes the relink easy: the boxes are MOVED from the Vec into the chain.
    fn aged(vals: &[i64]) -> List {
        let mut nodes: Vec<Box<Node>> = (0..vals.len())
            .map(|_| Box::new(Node { val: 0, next: None }))
            .collect();
        let mut rng = Pcg::new(3, 4);
        for i in (1..nodes.len()).rev() {
            nodes.swap(i, rng.int_n(i + 1)); // Fisher-Yates, by hand
        }
        let mut head = None;
        for (mut b, &v) in nodes.into_iter().zip(vals).rev() {
            b.val = v;
            b.next = head;
            head = Some(b);
        }
        List(head)
    }

    fn sum(&self) -> i64 {
        successors(self.0.as_deref(), |n| n.next.as_deref())
            .map(|n| n.val)
            .sum()
    }
}

fn main() {
    // ---- 1. correctness: random inserts mirrored on a Vec oracle ----
    const N1: usize = 100_000;
    let mut rng = Pcg::new(1, 2);
    let mut u = Unrolled::default();
    let mut oracle: Vec<i64> = Vec::new();
    let mut oracle_shift = 0usize;
    for v in 0..N1 as i64 {
        let i = rng.int_n(oracle.len() + 1);
        u.insert_at(i, v);
        oracle_shift += oracle.len() - i;
        oracle.insert(i, v);
    }
    let same = u.to_vec() == oracle;
    let at_ok = (0..1000)
        .filter(|_| {
            let i = rng.int_n(N1);
            u.get(i) == Some(oracle[i])
        })
        .count();
    println!("1. correctness: 100000 insert_at at PCG-random positions, mirrored on a Vec<i64>");
    println!("   contents identical: {same}   get(i) agrees on {at_ok}/1000 random positions");
    println!(
        "   chunks: {}   average fill: {:.1}% of {K} slots",
        u.chunks,
        u.fill()
    );
    let n1 = N1 as f64;
    println!(
        "   unrolled: {:>10} chunk hops + {:>10} values moved ({:.1} + {:.1} per insert)",
        u.hops,
        u.shifted,
        u.hops as f64 / n1,
        u.shifted as f64 / n1
    );
    println!(
        "   Vec:      {:>10} chunk hops + {oracle_shift:>10} values moved ({:.1} per insert)",
        0,
        oracle_shift as f64 / n1
    );
    println!(
        "   -> the Vec moved {:.0}x more values; the unrolled list pays in hops instead",
        oracle_shift as f64 / u.shifted as f64
    );

    // ---- 2. allocations to hold 1,000,000 ints ----
    let vals: Vec<i64> = (0..1_000_000).collect();
    let (uo, ub, big) = allocs_of(|| {
        let mut b = Unrolled::default();
        b.extend(&vals);
        b
    });
    let (po, pb, plain) = allocs_of(|| List::plain(&vals));
    let (so, sb, s) = allocs_of(|| {
        let mut s = Vec::new();
        for &v in &vals {
            s.push(v);
        }
        s
    });
    println!(
        "\n2. allocations to build 1000000 ints (counting allocator: events / bytes requested)"
    );
    println!("   {:<26} {:>10} {:>14}", "", "allocs", "bytes");
    println!(
        "   {:<26} {uo:>10} {ub:>14}   ({} chunks = ceil(n/32), size_of::<Chunk>() = {})",
        "unrolled, extend",
        big.chunks,
        size_of::<Chunk>()
    );
    println!(
        "   {:<26} {po:>10} {pb:>14}   (one per node, size_of::<Node>() = {})",
        "plain list, prepend",
        size_of::<Node>()
    );
    println!(
        "   {:<26} {so:>10} {sb:>14}   (push growth steps)",
        "Vec, push"
    );
    println!(
        "   -> {:.0}x fewer allocations than the plain list",
        po as f64 / uo as f64
    );
    drop((big, plain, s));

    // ---- 3. timing: traversal at n = 100,000, and insert_at ----
    let mut fresh = Unrolled::default();
    fresh.extend(&oracle);
    let (list_fresh, list_aged) = (List::plain(&oracle), List::aged(&oracle));
    let want = list_fresh.sum();
    assert!(fresh.sum() == want && u.sum() == want && list_aged.sum() == want);
    let fill_name = format!("unrolled, random ({:.0}% full)", u.fill());
    let rows: [(&str, &dyn Fn() -> i64); 5] = [
        ("Vec", &|| black_box(&oracle).iter().sum()),
        ("plain list, fresh", &|| black_box(&list_fresh).sum()),
        ("plain list, aged", &|| black_box(&list_aged).sum()),
        ("unrolled, fresh (100% full)", &|| black_box(&fresh).sum()),
        (&fill_name, &|| black_box(&u).sum()),
    ];
    println!("\n3. timing (checksums of all five traversals agree)");
    println!(
        "   {:<34} {:>10} {:>8}",
        "traversal sum, n = 100000", "ns/elem", "x Vec"
    );
    let mut ratio = [0.0; 5];
    let mut base = 0.0;
    for (k, (name, f)) in rows.iter().enumerate() {
        let ns = ns_per_op(|| {
            black_box(f());
        }) / n1;
        if k == 0 {
            base = ns;
        }
        ratio[k] = ns / base;
        println!("   {name:<34} {ns:>10.3} {:>8.1}", ns / base);
    }

    // insert_at: 10,000 random inserts into a fresh 100,000-element copy, timed
    // without the rebuild (the insert mutates, so every round starts pristine).
    const M: usize = 10_000;
    let pos: Vec<usize> = (0..M).map(|k| rng.int_n(N1 + k + 1)).collect();
    let (mut t_vec, mut t_unr) = (Duration::ZERO, Duration::ZERO);
    const ROUNDS: u32 = 5;
    for _ in 0..ROUNDS {
        let mut sl = oracle.clone();
        let t0 = Instant::now();
        for (k, &p) in pos.iter().enumerate() {
            sl.insert(p, k as i64);
        }
        t_vec += t0.elapsed();
        let mut w = Unrolled::default();
        w.extend(&oracle);
        let t0 = Instant::now();
        for (k, &p) in pos.iter().enumerate() {
            w.insert_at(p, k as i64);
        }
        t_unr += t0.elapsed();
        assert!(w.to_vec() == sl, "insert results differ");
    }
    let per = f64::from(ROUNDS) * M as f64;
    let (ns_v, ns_u) = (t_vec.as_nanos() as f64 / per, t_unr.as_nanos() as f64 / per);
    println!(
        "   {:<34} {:>10} {:>8}",
        "insert_at random, n = 100000..110000", "ns/insert", "x Vec"
    );
    println!("   {:<34} {ns_v:>10.0} {:>8.1}", "Vec::insert", 1.0);
    println!(
        "   {:<34} {ns_u:>10.0} {:>8.1}",
        "unrolled insert_at",
        ns_u / ns_v
    );

    println!(
        "\nverdict: unrolled traversal takes {:.1}x a Vec's time (plain list: {:.1}x fresh, {:.1}x aged);",
        ratio[3], ratio[1], ratio[2]
    );
    let (hops_per, moves_per) = (u.hops as f64 / n1, oracle_shift as f64 / n1);
    if ns_u > ns_v {
        println!(
            "insert_at is {:.1}x SLOWER than Vec::insert. Its shift is tiny ({:.1} values per",
            ns_u / ns_v,
            u.shifted as f64 / n1
        );
        println!(
            "insert), so its time is the walk: ~{hops_per:.0} dependent hops cost more than one memmove"
        );
        println!("of ~{moves_per:.0} contiguous values. Both are Θ(n) per insert:");
        println!("unrolling shrinks the shift, not the search.");
    } else {
        println!(
            "insert_at is {:.1}x faster than Vec::insert: ~{hops_per:.0} hops beat a memmove of ~{moves_per:.0} values.",
            ns_v / ns_u
        );
    }
}
```

**Rust sample output:**

```
1. correctness: 100000 insert_at at PCG-random positions, mirrored on a Vec<i64>
   contents identical: true   get(i) agrees on 1000/1000 random positions
   chunks: 4455   average fill: 70.1% of 32 slots
   unrolled:  111361418 chunk hops +    1151685 values moved (1113.6 + 11.5 per insert)
   Vec:               0 chunk hops + 2500561616 values moved (25005.6 per insert)
   -> the Vec moved 2171x more values; the unrolled list pays in hops instead

2. allocations to build 1000000 ints (counting allocator: events / bytes requested)
                                  allocs          bytes
   unrolled, extend                31250        8500000   (31250 chunks = ceil(n/32), size_of::<Chunk>() = 272)
   plain list, prepend           1000000       16000000   (one per node, size_of::<Node>() = 16)
   Vec, push                          19       16777184   (push growth steps)
   -> 32x fewer allocations than the plain list

3. timing (checksums of all five traversals agree)
   traversal sum, n = 100000             ns/elem    x Vec
   Vec                                     0.072      1.0
   plain list, fresh                       1.016     14.1
   plain list, aged                        5.534     76.6
   unrolled, fresh (100% full)             0.082      1.1
   unrolled, random (70% full)             0.246      3.4
   insert_at random, n = 100000..110000  ns/insert    x Vec
   Vec::insert                              6614      1.0
   unrolled insert_at                      10850      1.6

verdict: unrolled traversal takes 1.1x a Vec's time (plain list: 14.1x fresh, 76.6x aged);
insert_at is 1.6x SLOWER than Vec::insert. Its shift is tiny (11.5 values per
insert), so its time is the walk: ~1114 dependent hops cost more than one memmove
of ~25006 contiguous values. Both are Θ(n) per insert:
unrolling shrinks the shift, not the search.
```

> **Go vs Rust:** Sections 1 and 2 match Go count for count: 4,455 chunks, 70.1% fill, 111,361,418 hops, and 31,250 / 1,000,000 allocations. The exceptions are bytes and growth steps. The counting allocator sees the 272 bytes requested where Go's `TotalAlloc` sees the 288-byte size class (8,500,000 vs 9,000,000), and `Vec` doubles in 19 steps where Go's `append` grew in 38. There is no tail pointer, because the head owns the chain and a stored `&mut` to the last chunk would alias it. `extend` walks once, then appends through one `&mut` cursor. `Drop` is iterative.
> In timing, `Vec` and the fresh unrolled sum both take ~0.08 ns/elem, about 3× faster than Go's ~0.3. A plausible cause is LLVM vectorizing the contiguous loops (not measured). The list rows cost about what they cost in Go (~1 and ~6 ns/elem), so the plain-list ratios grow to ~12–14× (fresh) and ~70–79× (aged). `insert_at` is still slower than `Vec::insert`, by 1.5–1.6× against Go's ~2.5×.

**Complexity:** traversal Θ(n) with n/32 hops · `At(i)` Θ(i/32) · `InsertAt` Θ(i/32 + 32) · 31,250 objects for 1,000,000 ints instead of 1,000,000. Measured (deterministic): 100,000 random inserts leave 4,455 chunks at 70.1% fill, and the slice moves 2,171× more values than the unrolled list. Measured (timing, five runs): fresh unrolled traversal takes 0.7–1.1× a slice's time against 2.4–4.2× (fresh) and 15–23× (aged) for a plain list. The unrolled list built by random inserts takes 1.4–2.9×. Random `InsertAt` is about 2.5× *slower* than `slices.Insert` at n = 100,000 (2.4–2.8× across runs), because the ~1,114-hop walk costs more than the memmove.

---

## 16. Capstone: an LRU cache, built, proved, measured

`🔴 hard` · *the structure a doubly linked list exists for*

An LRU cache needs Θ(1) lookup by key, which takes a map, and Θ(1) "this entry is now the newest" and "drop
the oldest", which take a doubly linked list of nodes you hold. A map has no order, and a slice has no Θ(1)
move, so you need both: a map from key to node, plus example 6's sentinel ring ordered by recency. `Get` and
`Put` are Θ(1), and so is eviction, because the oldest entry is always `root.prev`. The cache is then *proved*
against a brute-force oracle with invariant checks after every operation, and a deliberately broken variant
shows what the oracle catches that the invariants cannot. Last, it is *measured* against the obvious
alternative, a map of timestamps, which must scan Θ(cap) entries to evict.

**Steps:**

1. Build `LRU[K, V]` from `map[K]*entry` and a sentinel ring, where `Put` at capacity evicts `root.prev` and
   reuses that node for the new key.
2. Run 200,000 PCG operations on both the cache and a slice oracle ordered by recency, comparing every result
   and every eviction. Then run the same operations against a bug, a `Get` that forgets to move the entry.
3. Time evicting `Put`s at cap 256, 1,024 and 4,096 against the timestamp scan, and watch which column grows
   with cap.

```go
package main

import (
	"flag"
	"fmt"
	"math"
	"math/rand/v2"
	"runtime"
	"slices"
	"strings"
	"testing"
)

// An LRU cache needs Θ(1) lookup by key (a map) AND Θ(1) "this one is now
// newest" / "drop the oldest" (a doubly linked list); neither has both. So: a
// map from key to node, plus a sentinel ring (example 6) ordered by recency.
// root.next is the most recently used entry, root.prev the least.
type entry[K comparable, V any] struct {
	key        K // kept in the node so eviction can delete it from the map
	val        V
	prev, next *entry[K, V]
}

type LRU[K comparable, V any] struct {
	cap     int
	m       map[K]*entry[K, V]
	root    entry[K, V] // sentinel: never holds data, so no nil checks anywhere
	noTouch bool        // a deliberate BUG for section 1: Get forgets to move to front
}

func NewLRU[K comparable, V any](capacity int) *LRU[K, V] {
	c := &LRU[K, V]{cap: capacity, m: make(map[K]*entry[K, V], capacity)}
	c.root.next, c.root.prev = &c.root, &c.root
	return c
}

func (c *LRU[K, V]) unlink(e *entry[K, V]) { e.prev.next, e.next.prev = e.next, e.prev }

func (c *LRU[K, V]) pushFront(e *entry[K, V]) {
	e.prev, e.next = &c.root, c.root.next // wire the new node first...
	c.root.next.prev, c.root.next = e, e  // ...then splice it in
}

// Get is Θ(1): one map lookup, then unlink + pushFront (eight pointer writes).
func (c *LRU[K, V]) Get(k K) (V, bool) {
	e, ok := c.m[k]
	if !ok {
		var zero V
		return zero, false // a miss changes nothing, including the order
	}
	if !c.noTouch {
		c.unlink(e)
		c.pushFront(e)
	}
	return e.val, true
}

// Put is Θ(1). An existing key is updated and moved to the front, and never
// evicts. A new key at capacity evicts root.prev, and the evicted node is
// recycled for the new key, so a full cache allocates no nodes.
func (c *LRU[K, V]) Put(k K, v V) (evicted K, did bool) {
	if e, ok := c.m[k]; ok {
		e.val = v
		c.unlink(e)
		c.pushFront(e)
		return evicted, false
	}
	var e *entry[K, V]
	if len(c.m) == c.cap {
		e = c.root.prev
		c.unlink(e)
		delete(c.m, e.key)
		evicted, did = e.key, true
	} else {
		e = &entry[K, V]{}
	}
	e.key, e.val = k, v
	c.pushFront(e)
	c.m[k] = e
	return evicted, did
}

// Keys lists the entries from most to least recently used.
func (c *LRU[K, V]) Keys() []K {
	var ks []K
	for e := c.root.next; e != &c.root; e = e.next {
		ks = append(ks, e.key)
	}
	return ks
}

// ok checks every invariant: forward length == backward length == len(map) <=
// cap, every link mirrored (e.next.prev == e), every ring node is the map's node
// for its key. Distinct keys + equal counts = map and ring hold the same entries.
func (c *LRU[K, V]) ok() bool {
	fwd, bwd := 0, 0
	for e := c.root.next; e != &c.root; e = e.next {
		if fwd++; fwd > c.cap || e.next.prev != e || c.m[e.key] != e {
			return false
		}
	}
	for e := c.root.prev; e != &c.root && bwd <= c.cap; e = e.prev {
		bwd++
	}
	return fwd == len(c.m) && bwd == len(c.m)
}

// The oracle: obviously correct, obviously slow. A slice ordered by recency
// (index 0 = most recent), linear search, shift to move.
type kv struct{ k, v int }

type oracle struct {
	cap int
	kv  []kv
}

func (o *oracle) touch(i int) kv {
	p := o.kv[i]
	copy(o.kv[1:i+1], o.kv[:i])
	o.kv[0] = p
	return p
}

func (o *oracle) get(k int) (int, bool) {
	for i := range o.kv {
		if o.kv[i].k == k {
			return o.touch(i).v, true
		}
	}
	return 0, false
}

func (o *oracle) put(k, v int) (evicted int, did bool) {
	for i := range o.kv {
		if o.kv[i].k == k {
			o.kv[i].v = v
			o.touch(i)
			return 0, false
		}
	}
	if len(o.kv) == o.cap {
		evicted, did = o.kv[len(o.kv)-1].k, true
		o.kv = o.kv[:len(o.kv)-1]
	}
	o.kv = slices.Insert(o.kv, 0, kv{k, v})
	return evicted, did
}

// prove runs 200,000 PCG ops (60% Get, 40% Put, keys in [0, 2*cap)) on the
// cache and the oracle. Every Get result and every eviction is compared, the
// invariants are checked after every op, and the full order every 1,000 ops.
func prove(c *LRU[int, int], stopAtFirst bool) {
	const ops = 200_000
	rng := rand.New(rand.NewPCG(1, 2))
	o := &oracle{cap: c.cap}
	gets, hits, evictions, mismatches, invFails := 0, 0, 0, 0, 0
	for op := 1; op <= ops; op++ {
		k := rng.IntN(2 * c.cap)
		var bad string
		if rng.IntN(10) < 6 {
			v, ok := c.Get(k)
			ov, ook := o.get(k)
			if gets++; ok {
				hits++
			}
			if v != ov || ok != ook {
				bad = fmt.Sprintf("Get(%d) = (%d, %v), oracle (%d, %v)", k, v, ok, ov, ook)
			}
		} else {
			ev, did := c.Put(k, op)
			oev, odid := o.put(k, op)
			if did {
				evictions++
			}
			if ev != oev || did != odid {
				bad = fmt.Sprintf("Put(%d) evicted (%d, %v), oracle evicted (%d, %v)", k, ev, did, oev, odid)
			}
		}
		if op%1000 == 0 && bad == "" && !slices.Equal(c.Keys(), keysOf(o.kv)) {
			bad = "recency order differs from the oracle"
		}
		if bad != "" {
			mismatches++
			if stopAtFirst {
				fmt.Printf("   first mismatch at op %d: %s\n", op, bad)
				fmt.Printf("   (invariant failures up to that op: %d: the structure is consistent, the POLICY is wrong)\n", invFails)
				return
			}
		}
		if !c.ok() {
			invFails++
		}
	}
	fmt.Printf("   %d ops: %d hits, %d misses, %d evictions\n", ops, hits, gets-hits, evictions)
	fmt.Printf("   %d mismatches, %d invariant failures\n", mismatches, invFails)
}

func keysOf(kvs []kv) (ks []int) {
	for _, p := range kvs {
		ks = append(ks, p.k)
	}
	return ks
}

func show(ks []int) string {
	parts := make([]string, len(ks))
	for i, k := range ks {
		parts[i] = fmt.Sprint(k)
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

// The alternative design: a map whose entries carry a timestamp. Get is Θ(1),
// but eviction must SCAN the whole map for the oldest timestamp: Θ(cap).
type tsEntry struct{ v, t int }

type tsCache struct {
	cap   int
	clock int
	m     map[int]tsEntry
}

func (c *tsCache) Put(k, v int) {
	c.clock++
	if _, ok := c.m[k]; !ok && len(c.m) == c.cap {
		oldK, oldT := 0, math.MaxInt
		for kk, e := range c.m {
			if e.t < oldT {
				oldK, oldT = kk, e.t
			}
		}
		delete(c.m, oldK)
	}
	c.m[k] = tsEntry{v, c.clock}
}

// bench: best ns/op of three 300 ms runs (least disturbed on a busy machine).
func bench(f func(b *testing.B)) (ns float64, allocs int64) {
	ns = math.Inf(1)
	for range 3 {
		r := testing.Benchmark(f)
		ns, allocs = min(ns, float64(r.T.Nanoseconds())/float64(r.N)), r.AllocsPerOp()
	}
	return ns, allocs
}

var keep any // the measured caches are stored here so they stay live

func heapObjects(f func()) (objs, bytes uint64) {
	var a, b runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&a)
	f()
	runtime.ReadMemStats(&b)
	return b.Mallocs - a.Mallocs, b.TotalAlloc - a.TotalAlloc
}

func main() {
	testing.Init()
	flag.Set("test.benchtime", "300ms")
	fmt.Println("1. prove: cap 64, against the brute-force oracle")
	prove(NewLRU[int, int](64), false)
	fmt.Println("   the same run with a bug (Get does not move the entry to the front):")
	buggy := NewLRU[int, int](64)
	buggy.noTouch = true
	prove(buggy, true)

	fmt.Println("\n   edge cases:")
	one := NewLRU[int, int](1)
	one.Put(1, 10)
	ev, did := one.Put(2, 20)
	_, hit1 := one.Get(1)
	v2, _ := one.Get(2)
	fmt.Printf("   cap 1: Put(1) Put(2) evicts %d (%v); Get(1) hit=%v; Get(2) = %d\n", ev, did, hit1, v2)
	c3 := NewLRU[int, int](3)
	for k := 1; k <= 3; k++ {
		c3.Put(k, k*10)
	}
	before := show(c3.Keys())
	_, did = c3.Put(1, 11)
	fmt.Printf("   cap 3 full %s: Put(1, 11) evicts? %v -> %s\n", before, did, show(c3.Keys()))
	before = show(c3.Keys())
	_, hit := c3.Get(9)
	fmt.Printf("   Get(9) on %s: hit=%v -> %s (a miss changes nothing)\n", before, hit, show(c3.Keys()))

	fmt.Println("\n2. measure: all-distinct Puts, so every Put evicts (cache pre-filled)")
	fmt.Printf("   %-6s %12s %12s %9s   %s\n", "cap", "LRU ns/op", "scan ns/op", "scan/LRU", "allocs/op LRU/scan")
	var lru, scan []float64
	caps := []int{256, 1024, 4096}
	for _, cp := range caps {
		l, la := bench(func(b *testing.B) {
			c := NewLRU[int, int](cp)
			for k := range cp {
				c.Put(k, k)
			}
			next := cp
			for b.Loop() {
				c.Put(next, next)
				next++
			}
		})
		s, sa := bench(func(b *testing.B) {
			c := &tsCache{cap: cp, m: make(map[int]tsEntry, cp)}
			for k := range cp {
				c.Put(k, k)
			}
			next := cp
			for b.Loop() {
				c.Put(next, next)
				next++
			}
		})
		fmt.Printf("   %-6d %12.1f %12.1f %8.0fx   %d / %d\n", cp, l, s, s/l, la, sa)
		lru, scan = append(lru, l), append(scan, s)
	}
	fmt.Printf("   cap grew %dx: the scan's cost grew %.1fx, the LRU's %.1fx\n",
		caps[2]/caps[0], scan[2]/scan[0], lru[2]/lru[0])

	g, ga := bench(func(b *testing.B) {
		c := NewLRU[int, int](4096)
		for k := range 4096 {
			c.Put(k, k)
		}
		k := 0
		for b.Loop() {
			c.Get(k & 4095) // always a hit, always a move-to-front
			k++
		}
	})
	fmt.Printf("   LRU Get (hit, cap 4096): %.1f ns/op, %d allocs/op\n", g, ga)

	const n = 4096
	lo, lb := heapObjects(func() {
		c := NewLRU[int, int](n)
		for k := range n {
			c.Put(k, k)
		}
		keep = c
	})
	so, sb := heapObjects(func() {
		c := &tsCache{cap: n, m: make(map[int]tsEntry, n)}
		for k := range n {
			c.Put(k, k)
		}
		keep = c
	})
	fmt.Printf("   memory, %d entries: LRU %.3f objects and %.0f bytes per entry; scan map %.3f and %.0f\n",
		n, float64(lo)/n, float64(lb)/n, float64(so)/n, float64(sb)/n)
}
```

**Sample output:**

```
1. prove: cap 64, against the brute-force oracle
   200000 ops: 59771 hits, 60154 misses, 39773 evictions
   0 mismatches, 0 invariant failures
   the same run with a bug (Get does not move the entry to the front):
   first mismatch at op 227: Put(17) evicted (16, true), oracle evicted (70, true)
   (invariant failures up to that op: 0: the structure is consistent, the POLICY is wrong)

   edge cases:
   cap 1: Put(1) Put(2) evicts 1 (true); Get(1) hit=false; Get(2) = 20
   cap 3 full [3 -> 2 -> 1]: Put(1, 11) evicts? false -> [1 -> 3 -> 2]
   Get(9) on [1 -> 3 -> 2]: hit=false -> [1 -> 3 -> 2] (a miss changes nothing)

2. measure: all-distinct Puts, so every Put evicts (cache pre-filled)
   cap       LRU ns/op   scan ns/op  scan/LRU   allocs/op LRU/scan
   256            37.2       1501.7       40x   0 / 0
   1024           41.0       5956.8      145x   0 / 0
   4096           45.5      24917.0      547x   0 / 0
   cap grew 16x: the scan's cost grew 16.6x, the LRU's 1.2x
   LRU Get (hit, cap 4096): 8.6 ns/op, 0 allocs/op
   memory, 4096 entries: LRU 1.005 objects and 68 bytes per entry; scan map 0.005 and 53
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::hash::Hash;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- counting allocator: stand-in for runtime.MemStats and AllocsPerOp ----
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

/// best ns/op of three 300 ms runs (the least disturbed one on a busy machine)
fn bench(mut f: impl FnMut()) -> f64 {
    let mut best = f64::INFINITY;
    for _ in 0..3 {
        let mut n: u64 = 1;
        loop {
            let t = Instant::now();
            for _ in 0..n {
                f();
            }
            let el = t.elapsed();
            if el >= Duration::from_millis(300) {
                best = best.min(el.as_nanos() as f64 / n as f64);
                break;
            }
            let per = el.as_nanos().max(1) as f64 / n as f64;
            n = ((360e6 / per) as u64).clamp(n + 1, n * 100);
        }
    }
    best
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN: same seed, same op stream.
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

// The LRU: a HashMap from key to node INDEX, plus a doubly linked ring kept in
// an arena (example 13). Go's design, `map[K]*entry` with prev/next pointers,
// would be `HashMap<K, Rc<RefCell<Node>>>` with `Weak` back links here: a
// refcount and a runtime borrow check on every touch. With indices there is one
// owner (the Vecs), so no Rc, no RefCell, no unsafe, and nothing to Drop by hand.
#[derive(Clone, Copy)]
struct Links {
    prev: u32,
    next: u32,
}

struct Lru<K, V> {
    cap: usize,
    map: HashMap<K, u32>,
    links: Vec<Links>, // links[0] is the sentinel: next = most recent, prev = least
    slots: Vec<(K, V)>, // node i (i >= 1) holds slots[i - 1]; the sentinel holds no data
    no_touch: bool,    // a deliberate BUG for section 1: get forgets to move to front
}

impl<K: Hash + Eq + Clone, V> Lru<K, V> {
    fn new(cap: usize) -> Self {
        let mut links = Vec::with_capacity(cap + 1);
        links.push(Links { prev: 0, next: 0 }); // an empty ring: the sentinel alone
        Lru {
            cap,
            map: HashMap::with_capacity(cap),
            links,
            slots: Vec::with_capacity(cap),
            no_touch: false,
        }
    }

    fn link(&mut self, i: u32) -> &mut Links {
        &mut self.links[i as usize]
    }

    fn unlink(&mut self, i: u32) {
        let Links { prev, next } = self.links[i as usize];
        self.link(prev).next = next;
        self.link(next).prev = prev;
    }

    fn push_front(&mut self, i: u32) {
        let first = self.links[0].next;
        *self.link(i) = Links {
            prev: 0,
            next: first,
        }; // wire the new node first...
        self.link(first).prev = i; // ...then splice it in
        self.link(0).next = i;
    }

    /// Θ(1): one hash lookup, then unlink + push_front. A miss changes nothing.
    fn get(&mut self, k: &K) -> Option<&V> {
        let i = *self.map.get(k)?;
        if !self.no_touch {
            self.unlink(i);
            self.push_front(i);
        }
        Some(&self.slots[i as usize - 1].1)
    }

    /// Θ(1). An existing key is updated and moved, never evicts. A new key at
    /// capacity evicts the sentinel's prev and REUSES its slot: no allocation.
    fn put(&mut self, k: K, v: V) -> Option<K> {
        if let Some(&i) = self.map.get(&k) {
            self.slots[i as usize - 1].1 = v;
            self.unlink(i);
            self.push_front(i);
            return None;
        }
        let (i, evicted) = if self.map.len() == self.cap {
            let i = self.links[0].prev;
            self.unlink(i);
            let (old, _) = std::mem::replace(&mut self.slots[i as usize - 1], (k.clone(), v));
            self.map.remove(&old);
            (i, Some(old))
        } else {
            self.slots.push((k.clone(), v));
            self.links.push(Links { prev: 0, next: 0 });
            (self.slots.len() as u32, None)
        };
        self.push_front(i);
        self.map.insert(k, i);
        evicted
    }

    /// keys from most to least recently used
    fn keys(&self) -> Vec<K> {
        let mut ks = Vec::with_capacity(self.map.len());
        let mut i = self.links[0].next;
        while i != 0 {
            ks.push(self.slots[i as usize - 1].0.clone());
            i = self.links[i as usize].next;
        }
        ks
    }

    /// every invariant: forward length == backward length == map.len() <= cap,
    /// every link mirrored, every ring node is the map's node for its key.
    fn ok(&self) -> bool {
        let (mut fwd, mut bwd) = (0, 0);
        let mut i = self.links[0].next;
        while i != 0 {
            fwd += 1;
            let next = self.links[i as usize].next;
            let key = &self.slots[i as usize - 1].0;
            if fwd > self.cap
                || self.links[next as usize].prev != i
                || self.map.get(key) != Some(&i)
            {
                return false;
            }
            i = next;
        }
        let mut i = self.links[0].prev;
        while i != 0 && bwd <= self.cap {
            bwd += 1;
            i = self.links[i as usize].prev;
        }
        fwd == self.map.len() && bwd == self.map.len()
    }
}

// The oracle: a Vec ordered by recency (index 0 = most recent), linear search,
// and rotate_right(1) on the prefix to move an entry to the front.
struct Oracle {
    cap: usize,
    kv: Vec<(i64, i64)>,
}

impl Oracle {
    fn get(&mut self, k: i64) -> Option<i64> {
        let i = self.kv.iter().position(|p| p.0 == k)?;
        self.kv[..=i].rotate_right(1);
        Some(self.kv[0].1)
    }
    fn put(&mut self, k: i64, v: i64) -> Option<i64> {
        if let Some(i) = self.kv.iter().position(|p| p.0 == k) {
            self.kv[i].1 = v;
            self.kv[..=i].rotate_right(1);
            return None;
        }
        let evicted = if self.kv.len() == self.cap {
            self.kv.pop().map(|p| p.0)
        } else {
            None
        };
        self.kv.insert(0, (k, v));
        evicted
    }
}

/// 200,000 PCG ops (60% get, 40% put, keys in [0, 2*cap)) on cache and oracle:
/// every get result and every eviction compared, invariants after every op,
/// the full order every 1,000 ops.
fn prove(c: &mut Lru<i64, i64>, stop_at_first: bool) {
    const OPS: i64 = 200_000;
    let mut rng = Pcg::new(1, 2);
    let mut o = Oracle {
        cap: c.cap,
        kv: Vec::new(),
    };
    let (mut gets, mut hits, mut evictions, mut mismatches, mut inv_fails) = (0, 0, 0, 0, 0);
    for op in 1..=OPS {
        let k = rng.int_n(2 * c.cap as u64) as i64;
        let mut bad = None;
        if rng.int_n(10) < 6 {
            let got = c.get(&k).copied();
            let want = o.get(k);
            gets += 1;
            hits += usize::from(got.is_some());
            if got != want {
                bad = Some(format!("get({k}) = {got:?}, oracle {want:?}"));
            }
        } else {
            let ev = c.put(k, op);
            let want = o.put(k, op);
            evictions += usize::from(ev.is_some());
            if ev != want {
                bad = Some(format!("put({k}) evicted {ev:?}, oracle evicted {want:?}"));
            }
        }
        let keys_ok = || c.keys().iter().eq(o.kv.iter().map(|p| &p.0));
        if op % 1000 == 0 && bad.is_none() && !keys_ok() {
            bad = Some("recency order differs from the oracle".to_string());
        }
        if let Some(bad) = bad {
            mismatches += 1;
            if stop_at_first {
                println!("   first mismatch at op {op}: {bad}");
                println!(
                    "   (invariant failures up to that op: {inv_fails}: the structure is consistent, the POLICY is wrong)"
                );
                return;
            }
        }
        if !c.ok() {
            inv_fails += 1;
        }
    }
    println!(
        "   {OPS} ops: {hits} hits, {} misses, {evictions} evictions",
        gets - hits
    );
    println!("   {mismatches} mismatches, {inv_fails} invariant failures");
}

fn show(ks: &[i64]) -> String {
    let parts: Vec<String> = ks.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" -> "))
}

// The alternative: a map whose entries carry a timestamp. get is Θ(1), but
// eviction must SCAN the whole map for the oldest timestamp: Θ(cap).
struct TsCache {
    cap: usize,
    clock: u64,
    m: HashMap<i64, (i64, u64)>, // key -> (value, last-use time)
}

impl TsCache {
    fn new(cap: usize) -> Self {
        TsCache {
            cap,
            clock: 0,
            m: HashMap::with_capacity(cap),
        }
    }
    fn put(&mut self, k: i64, v: i64) {
        self.clock += 1;
        if !self.m.contains_key(&k) && self.m.len() == self.cap {
            let (&oldest, _) = self.m.iter().min_by_key(|(_, e)| e.1).unwrap();
            self.m.remove(&oldest);
        }
        self.m.insert(k, (v, self.clock));
    }
}

fn main() {
    println!("1. prove: cap 64, against the brute-force oracle");
    prove(&mut Lru::new(64), false);
    println!("   the same run with a bug (get does not move the entry to the front):");
    let mut buggy = Lru::new(64);
    buggy.no_touch = true;
    prove(&mut buggy, true);

    println!("\n   edge cases:");
    let mut one = Lru::new(1);
    one.put(1, 10);
    let ev = one.put(2, 20);
    let hit1 = one.get(&1).is_some();
    let v2 = one.get(&2).copied();
    println!("   cap 1: put(1) put(2) evicts {ev:?}; get(1) hit={hit1}; get(2) = {v2:?}");
    let mut c3 = Lru::new(3);
    for k in 1..=3 {
        c3.put(k, k * 10);
    }
    let before = show(&c3.keys());
    let ev = c3.put(1, 11);
    println!(
        "   cap 3 full {before}: put(1, 11) evicts? {ev:?} -> {}",
        show(&c3.keys())
    );
    let before = show(&c3.keys());
    let hit = c3.get(&9).is_some();
    println!(
        "   get(9) on {before}: hit={hit} -> {} (a miss changes nothing)",
        show(&c3.keys())
    );

    println!("\n2. measure: all-distinct puts, so every put evicts (cache pre-filled)");
    println!(
        "   {:<6} {:>12} {:>12} {:>9}   allocs/op LRU/scan",
        "cap", "LRU ns/op", "scan ns/op", "scan/LRU"
    );
    let (mut lru, mut scan) = (Vec::new(), Vec::new());
    let caps = [256usize, 1024, 4096];
    for &cp in &caps {
        let mut c = Lru::new(cp);
        let mut t = TsCache::new(cp);
        for k in 0..cp as i64 {
            c.put(k, k);
            t.put(k, k);
        }
        let mut next = cp as i64;
        let l = bench(|| {
            black_box(c.put(black_box(next), next));
            next += 1;
        });
        let s = bench(|| {
            t.put(black_box(next), next);
            next += 1;
        });
        // allocations per evicting put, over 10,000 more (after the warm-up above)
        let (la, _, ()) = allocs_of(|| {
            for _ in 0..10_000 {
                c.put(next, next);
                next += 1;
            }
        });
        let (sa, _, ()) = allocs_of(|| {
            for _ in 0..10_000 {
                t.put(next, next);
                next += 1;
            }
        });
        println!(
            "   {cp:<6} {l:>12.1} {s:>12.1} {:>8.0}x   {} / {}",
            s / l,
            la / 10_000,
            sa / 10_000
        );
        lru.push(l);
        scan.push(s);
    }
    println!(
        "   cap grew {}x: the scan's cost grew {:.1}x, the LRU's {:.1}x",
        caps[2] / caps[0],
        scan[2] / scan[0],
        lru[2] / lru[0]
    );

    let mut c = Lru::new(4096);
    for k in 0..4096 {
        c.put(k, k);
    }
    let mut k = 0i64;
    let g = bench(|| {
        black_box(c.get(&black_box(k & 4095))); // always a hit, always a move-to-front
        k += 1;
    });
    let (ga, _, ()) = allocs_of(|| {
        for k in 0..10_000 {
            black_box(c.get(&(k & 4095)));
        }
    });
    println!(
        "   LRU get (hit, cap 4096): {g:.1} ns/op, {} allocs/op",
        ga / 10_000
    );

    const N: i64 = 4096;
    let (lo, lb, _keep_lru) = allocs_of(|| {
        let mut c = Lru::new(N as usize);
        for k in 0..N {
            c.put(k, k);
        }
        c
    });
    let (so, sb, _keep_scan) = allocs_of(|| {
        let mut t = TsCache::new(N as usize);
        for k in 0..N {
            t.put(k, k);
        }
        t
    });
    let n = N as f64;
    println!(
        "   memory, {N} entries: LRU {lo} allocations in all and {:.0} bytes per entry; scan map {so} and {:.0}",
        lb as f64 / n,
        sb as f64 / n
    );
}
```

**Rust sample output:**

```
1. prove: cap 64, against the brute-force oracle
   200000 ops: 59771 hits, 60154 misses, 39773 evictions
   0 mismatches, 0 invariant failures
   the same run with a bug (get does not move the entry to the front):
   first mismatch at op 227: put(17) evicted Some(16), oracle evicted Some(70)
   (invariant failures up to that op: 0: the structure is consistent, the POLICY is wrong)

   edge cases:
   cap 1: put(1) put(2) evicts Some(1); get(1) hit=false; get(2) = Some(20)
   cap 3 full [3 -> 2 -> 1]: put(1, 11) evicts? None -> [1 -> 3 -> 2]
   get(9) on [1 -> 3 -> 2]: hit=false -> [1 -> 3 -> 2] (a miss changes nothing)

2. measure: all-distinct puts, so every put evicts (cache pre-filled)
   cap       LRU ns/op   scan ns/op  scan/LRU   allocs/op LRU/scan
   256            43.1        435.5       10x   0 / 0
   1024           42.9       1138.2       27x   0 / 0
   4096           41.7       3742.6       90x   0 / 0
   cap grew 16x: the scan's cost grew 8.6x, the LRU's 1.0x
   LRU get (hit, cap 4096): 10.7 ns/op, 0 allocs/op
   memory, 4096 entries: LRU 3 allocations in all and 58 bytes per entry; scan map 1 and 50
```

> **Go vs Rust:** Section 1 is identical to Go: 59,771 / 60,154 / 39,773, and the bug is caught at op 227. The PCG port feeds the same operation stream; results just print as `Option` (`Some(16)`) where Go prints `(16, true)`.
> Go's `map[K]*entry` with prev/next pointers becomes `HashMap<K, u32>` plus an index arena. The arena is a `Vec<Links>` of `u32` prev/next with the sentinel at 0, and the evicted slot is reused. It avoids the `HashMap<K, Rc<RefCell<Node>>>` + `Weak` design, which pays a refcount and a runtime borrow check on every touch. There is no `unsafe` and no hand-written `Drop`, and the whole cache is 3 allocations (58 bytes per entry) against Go's ~1 heap object (68 bytes) per entry.
> Timing (four runs): Rust's evicting `put` costs ~40–80 ns with the default SipHash, against Go's ~36–45 ns with its runtime hash. Rust's full-map scan, however, is ~4–6× faster than Go's (3.7–6.6 µs vs ~24 µs at cap 4,096). So the scan/LRU gap at 4,096 is roughly 70–90× instead of Go's ~540×, though the LRU still wins at every cap. Why the `HashMap` scan is faster was not measured.

**Complexity:** `Get` / `Put` / evict Θ(1) for the LRU · the timestamp map is Θ(1) `Get` but **Θ(cap) evict** · memory is one map slot plus one node per key. Measured (deterministic): 200,000 operations give 59,771 hits, 60,154 misses, 39,773 evictions, 0 mismatches and 0 invariant failures. The no-touch bug is caught at op 227 by the oracle alone, with every invariant still holding. Measured (timing): an evicting `Put` costs ~36–45 ns at every cap, while the scan grows ~16× from cap 256 to cap 4,096, reaching ~540× slower at 4,096. Both make 0 allocations per operation at steady state. In Go the LRU costs ~1 heap object and ~68 bytes per entry.

---

> Next lesson: [08 — Stacks](../../08-stacks.md) — the structure that only ever touches the cheap end.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
