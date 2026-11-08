# Step 09 — Queues & Deques · 🟡 Medium

Examples **7–11**: what the standard library actually gives you, what a buffered channel really is,
what to do when the queue is full, and the deque algorithm that earns its keep.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Example 7's `container/ring` trace is deterministic; everything else here reports timings or memory,
sampled on an Apple M4 with Go 1.26.3.

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

## 7. `container/list` and `container/ring` — what they are for

`🟡 medium` · *neither one is a ring buffer*

Go ships two containers that look like queues. Knowing exactly what each one *is* saves you from both
mistakes: reaching for them when you want a queue, and dismissing `container/ring` when it is
genuinely the right answer.

**Steps:**

1. Benchmark `container/list` on the queue workload and count its heap objects.
2. Meet `container/ring` — a circular *linked list*, not a buffer.
3. Find the operation it does that nothing else does in Θ(1).

```go
package main

import (
	"container/list"
	"container/ring"
	"fmt"
	"runtime"
	"testing"
)

// Go ships two containers that look like a queue, and neither one is.
// Knowing precisely what they ARE saves you from both mistakes:
// reaching for them when you want a queue, and dismissing them when
// container/ring is exactly right.

type sizedRing struct {
	buf         []int
	mask        int
	head, count int
}

func newSizedRing(n int) *sizedRing {
	c := 1
	for c < n {
		c <<= 1
	}
	return &sizedRing{buf: make([]int, c), mask: c - 1}
}
func (r *sizedRing) push(v int) {
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
}
func (r *sizedRing) pop() int {
	v := r.buf[r.head]
	r.head = (r.head + 1) & r.mask
	r.count--
	return v
}

var sinkI int
var sinkA any

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func heapObjects() int64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return int64(ms.HeapObjects)
}

func main() {
	fmt.Println("container/list IS a queue, in the sense that PushBack and Front/")
	fmt.Println("Remove are both Theta(1). Lesson 07 already measured what that")
	fmt.Println("costs; here it is again on the queue workload specifically:")
	fmt.Println()

	const ops = 200_000
	tRing := nsPerOp(func() {
		r := newSizedRing(2048)
		for i := 0; i < 1000; i++ {
			r.push(i)
		}
		for i := 0; i < ops; i++ {
			r.push(i)
			sinkI = r.pop()
		}
	})
	tList := nsPerOp(func() {
		l := list.New()
		for i := 0; i < 1000; i++ {
			l.PushBack(i)
		}
		for i := 0; i < ops; i++ {
			l.PushBack(i)
			sinkA = l.Remove(l.Front())
		}
	})
	tSlice := nsPerOp(func() {
		q := make([]int, 0, 1000)
		for i := 0; i < 1000; i++ {
			q = append(q, i)
		}
		for i := 0; i < ops; i++ {
			q = append(q, i)
			sinkI = q[0]
			q = q[1:]
		}
	})
	fmt.Printf("  %-30s %10.2f ns/op\n", "ring buffer", tRing/ops)
	fmt.Printf("  %-30s %10.2f ns/op   %.1fx\n", "slice q = q[1:]", tSlice/ops, tSlice/tRing)
	fmt.Printf("  %-30s %10.2f ns/op   %.1fx\n", "container/list", tList/ops, tList/tRing)

	base := heapObjects()
	l := list.New()
	for i := 0; i < 100_000; i++ {
		l.PushBack(i)
	}
	listObjs := heapObjects() - base
	runtime.KeepAlive(l)

	base = heapObjects()
	r := newSizedRing(1 << 17)
	for i := 0; i < 100_000; i++ {
		r.push(i)
	}
	ringObjs := heapObjects() - base
	runtime.KeepAlive(r)

	fmt.Println()
	fmt.Printf("  heap objects for 100,000 ints: list %d, ring %d\n", listObjs, ringObjs)
	fmt.Println()
	fmt.Println("  the ring is ONE object -- the backing array. The list is one")
	fmt.Println("  Element per value plus one boxed `any` per int, and every one of")
	fmt.Println("  them is a pointer the GC must trace on every cycle. That is")
	fmt.Println("  lesson 04's finding and lesson 07's finding arriving together.")
	fmt.Println()
	fmt.Println("  use container/list for a queue only when you also need the thing")
	fmt.Println("  a list uniquely gives you: Theta(1) removal from the MIDDLE by a")
	fmt.Println("  handle you are already holding. That is an LRU cache (lesson 07,")
	fmt.Println("  example 16), not a queue.")

	fmt.Println()
	fmt.Println("container/ring is the one that surprises people. It is NOT a ring")
	fmt.Println("BUFFER -- there is no head, no tail, no Push and no Pop:")
	fmt.Println()
	rr := ring.New(5)
	for i := 0; i < 5; i++ {
		rr.Value = i + 1
		rr = rr.Next()
	}
	fmt.Printf("  ring.New(5), filled 1..5, then Do: ")
	rr.Do(func(v any) { fmt.Printf("%v ", v) })
	fmt.Println()
	fmt.Printf("  Move(2) then Value:                %v\n", rr.Move(2).Value)
	fmt.Printf("  Prev().Value:                      %v\n", rr.Prev().Value)
	fmt.Println()
	fmt.Println("  it is a CIRCULAR DOUBLY LINKED LIST of fixed length -- every")
	fmt.Println("  element is a node with Next and Prev, and the last one points back")
	fmt.Println("  at the first. Lesson 07's sentinel ring, with the sentinel removed")
	fmt.Println("  and the length frozen.")
	fmt.Println()

	base = heapObjects()
	cr := ring.New(100_000)
	for i := 0; i < 100_000; i++ {
		cr.Value = i
		cr = cr.Next()
	}
	crObjs := heapObjects() - base
	runtime.KeepAlive(cr)
	fmt.Printf("  heap objects for a 100,000-element container/ring: %d\n", crObjs)
	fmt.Println()
	fmt.Printf("  heap objects for container/list, again, for comparison:   %d\n", listObjs)
	fmt.Println()
	fmt.Println("  the same. I expected container/ring to do better -- it allocates")
	fmt.Println("  a fixed number of nodes up front rather than one per push -- but")
	fmt.Println("  it is a linked structure storing `any`, so it pays for a node AND")
	fmt.Println("  a box per element exactly like container/list does.")
	fmt.Println()
	fmt.Println("  a ring BUFFER of 100,000 ints is one array and ONE object. The")
	fmt.Println("  word 'ring' in the package name is about topology, not layout.")

	fmt.Println()
	fmt.Println("what container/ring is actually FOR -- rotation and splicing:")
	fmt.Println()
	a := ring.New(3)
	for i, v := range []string{"a", "b", "c"} {
		_ = i
		a.Value = v
		a = a.Next()
	}
	b := ring.New(2)
	for _, v := range []string{"X", "Y"} {
		b.Value = v
		b = b.Next()
	}
	show := func(label string, r *ring.Ring) {
		fmt.Printf("  %-22s len=%d  ", label, r.Len())
		r.Do(func(v any) { fmt.Printf("%v ", v) })
		fmt.Println()
	}
	show("ring a", a)
	show("ring b", b)
	a.Link(b)
	show("a.Link(b)", a)
	rem := a.Unlink(2)
	show("after a.Unlink(2)", a)
	show("  the removed part", rem)
	fmt.Println()
	fmt.Println("  Link and Unlink splice whole rings in Theta(1) -- no copying, no")
	fmt.Println("  reallocation, no bounds. That is genuinely hard to do any other")
	fmt.Println("  way, and it is why the package exists: round-robin schedulers,")
	fmt.Println("  token rings, cyclic buffers of connections you rotate through.")

	fmt.Println()
	fmt.Println("the summary, and it is short:")
	fmt.Println()
	fmt.Printf("  %-20s %s\n", "your own ring", "a queue. Use this.")
	fmt.Printf("  %-20s %s\n", "container/list", "Theta(1) removal by handle. LRU, not queues.")
	fmt.Printf("  %-20s %s\n", "container/ring", "rotation and Theta(1) splicing. Not a buffer.")
	fmt.Printf("  %-20s %s\n", "buffered channel", "a queue with a lock. Example 8.")
	fmt.Println()
	fmt.Println("  both container packages predate generics and store `any`, which")
	fmt.Println("  is why both box. Neither will ever be made generic -- the Go team")
	fmt.Println("  has said so -- so treat them as legacy for anything hot.")
}
```

**Sample output:**

```
container/list IS a queue, in the sense that PushBack and Front/
Remove are both Theta(1). Lesson 07 already measured what that
costs; here it is again on the queue workload specifically:

  ring buffer                          2.40 ns/op
  slice q = q[1:]                      1.76 ns/op   0.7x
  container/list                      20.73 ns/op   8.6x

  heap objects for 100,000 ints: list 149869, ring 2

  the ring is ONE object -- the backing array. The list is one
  Element per value plus one boxed `any` per int, and every one of
  them is a pointer the GC must trace on every cycle. That is
  lesson 04's finding and lesson 07's finding arriving together.

  use container/list for a queue only when you also need the thing
  a list uniquely gives you: Theta(1) removal from the MIDDLE by a
  handle you are already holding. That is an LRU cache (lesson 07,
  example 16), not a queue.

container/ring is the one that surprises people. It is NOT a ring
BUFFER -- there is no head, no tail, no Push and no Pop:

  ring.New(5), filled 1..5, then Do: 1 2 3 4 5 
  Move(2) then Value:                3
  Prev().Value:                      5

  it is a CIRCULAR DOUBLY LINKED LIST of fixed length -- every
  element is a node with Next and Prev, and the last one points back
  at the first. Lesson 07's sentinel ring, with the sentinel removed
  and the length frozen.

  heap objects for a 100,000-element container/ring: 149867

  heap objects for container/list, again, for comparison:   149869

  the same. I expected container/ring to do better -- it allocates
  a fixed number of nodes up front rather than one per push -- but
  it is a linked structure storing `any`, so it pays for a node AND
  a box per element exactly like container/list does.

  a ring BUFFER of 100,000 ints is one array and ONE object. The
  word 'ring' in the package name is about topology, not layout.

what container/ring is actually FOR -- rotation and splicing:

  ring a                 len=3  a b c 
  ring b                 len=2  X Y 
  a.Link(b)              len=5  a X Y b c 
  after a.Unlink(2)      len=3  a b c 
    the removed part     len=2  X Y 

  Link and Unlink splice whole rings in Theta(1) -- no copying, no
  reallocation, no bounds. That is genuinely hard to do any other
  way, and it is why the package exists: round-robin schedulers,
  token rings, cyclic buffers of connections you rotate through.

the summary, and it is short:

  your own ring        a queue. Use this.
  container/list       Theta(1) removal by handle. LRU, not queues.
  container/ring       rotation and Theta(1) splicing. Not a buffer.
  buffered channel     a queue with a lock. Example 8.

  both container packages predate generics and store `any`, which
  is why both box. Neither will ever be made generic -- the Go team
  has said so -- so treat them as legacy for anything hot.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{LinkedList, VecDeque};
use std::fmt::Display;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Go ships two containers that look like a queue, and neither one is.
// Rust's std ships one of them and not the other:
//
//	Go                Rust std
//	container/list    LinkedList<T>   doubly linked, generic (no boxing)
//	container/ring    -- nothing --   no circular list in std
//	(your own ring)   VecDeque<T>     the real ring buffer, already written
//
// So this example measures LinkedList against the ring, and builds
// container/ring's topology by hand to see what it is FOR. In Rust that is an
// ARENA: nodes live in one Vec and `next`/`prev` are indices, because a ring of
// owning pointers is a cycle and safe Rust will not give you one for free
// (Rc cycles leak, and an Rc chain of 100,000 nodes overflows the stack when
// it is dropped).

// ---- the kit's allocation counter: stand-in for runtime.MemStats.HeapObjects ----
// Go forces runtime.GC() and reads the number of live heap objects; Rust has no
// GC, so the allocator counts live BLOCKS (and bytes) directly. A realloc moves
// or resizes a block but does not create one.
struct Counting;
static BLOCKS: AtomicUsize = AtomicUsize::new(0); // blocks currently live
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        BLOCKS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        BLOCKS.fetch_sub(1, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (live heap blocks, live bytes) -- a deterministic MemStats.HeapObjects/HeapAlloc.
fn heap() -> (usize, usize) {
    (BLOCKS.load(Relaxed), LIVE.load(Relaxed))
}

/// The same power-of-two ring as example 3's `RingPow2`, under example 7's Go
/// name. push/pop return Result/Option, as in examples 3 and 5, instead of
/// trusting the caller not to overfill it.
struct SizedRing {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
}

impl SizedRing {
    fn new(n: usize) -> Self {
        // Go loops `c <<= 1`; Rust has it built in.
        let c = n.next_power_of_two();
        SizedRing {
            buf: vec![0; c],
            mask: c - 1,
            head: 0,
            count: 0,
        }
    }
    fn push(&mut self, v: i64) -> Result<(), i64> {
        if self.count == self.buf.len() {
            return Err(v);
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
        Ok(())
    }
    fn pop(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
    }
}

// ---- container/ring's topology, as an arena ----
//
// A circular doubly linked list of FIXED length: every node has next and prev,
// the last points back at the first. A handle is a node's index (Go's *ring.Ring).
// Go's Ring has a `Value any`; here the node is generic, so T sits inline in the
// node -- no box.

type Ring = usize;

struct Node<T> {
    value: T,
    next: Ring,
    prev: Ring,
}

struct Rings<T> {
    nodes: Vec<Node<T>>,
}

impl<T> Rings<T> {
    fn new() -> Self {
        Rings { nodes: Vec::new() }
    }

    /// ring.New(n) and the loop that fills Value, in one step: a Rust node
    /// cannot exist without a value, so there is no "empty ring of n nils".
    /// Returns the first node, or None for an empty ring.
    fn make(&mut self, values: impl IntoIterator<Item = T>) -> Option<Ring> {
        let start = self.nodes.len();
        self.nodes
            .extend(values.into_iter().enumerate().map(|(k, value)| Node {
                value,
                next: start + k + 1,
                prev: (start + k).wrapping_sub(1),
            }));
        let end = self.nodes.len().checked_sub(1)?;
        if end < start {
            return None;
        }
        self.nodes[end].next = start; // the last points back at the first...
        self.nodes[start].prev = end; // ...and the first at the last
        Some(start)
    }

    fn next(&self, r: Ring) -> Ring {
        self.nodes[r].next
    }
    fn prev(&self, r: Ring) -> Ring {
        self.nodes[r].prev
    }

    /// Go's Move(n): n may be negative. `move` is a Rust keyword.
    fn step(&self, mut r: Ring, n: isize) -> Ring {
        for _ in 0..n.unsigned_abs() {
            r = if n < 0 { self.prev(r) } else { self.next(r) };
        }
        r
    }

    /// Go's Len(): Theta(n), it walks the whole ring.
    fn len(&self, r: Ring) -> usize {
        self.values(r).count()
    }

    /// Go's Do(f): one lap, starting at r.
    fn values(&self, r: Ring) -> impl Iterator<Item = &T> {
        let mut at = Some(r);
        std::iter::from_fn(move || {
            let cur = at?;
            let nx = self.next(cur);
            at = (nx != r).then_some(nx);
            Some(&self.nodes[cur].value)
        })
    }

    /// Go's Link, line for line. Connects ring `s` in after `r` and returns
    /// what used to follow `r`. If r and s are already in one ring it cuts
    /// instead (that is how Unlink works).
    fn link(&mut self, r: Ring, s: Ring) -> Ring {
        let n = self.next(r);
        let p = self.prev(s);
        self.nodes[r].next = s;
        self.nodes[s].prev = r;
        self.nodes[n].prev = p;
        self.nodes[p].next = n;
        n
    }

    /// Go's Unlink(n): remove the n nodes after r; returns the removed ring.
    /// Go returns nil for n <= 0; an unsigned n makes only 0 left.
    fn unlink(&mut self, r: Ring, n: usize) -> Option<Ring> {
        if n == 0 {
            return None;
        }
        let s = self.step(r, n as isize + 1);
        Some(self.link(r, s))
    }
}

impl<T: Display> Rings<T> {
    /// Go's `show` closure: label, length, then one lap of values.
    fn show(&self, label: &str, r: Ring) {
        let vals: Vec<String> = self.values(r).map(ToString::to_string).collect();
        println!(
            "  {:<22} len={}  {}",
            label,
            self.len(r),
            vals.iter().map(|v| format!("{v} ")).collect::<String>()
        );
    }
}

/// ns per call of `run`, measured for about 200 ms (kit: stand-in for testing.Benchmark).
fn ns_per_op(mut run: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            run();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

fn main() {
    println!("LinkedList IS a queue, in the sense that push_back and pop_front");
    println!("are both Theta(1). Lesson 07 already measured what that costs;");
    println!("here it is again on the queue workload specifically:");
    println!();

    const OPS: usize = 200_000;
    // black_box on the capacity, as in example 3: a literal 2048 would let LLVM
    // fold the mask into a constant and the benchmark would time that instead.
    let t_ring = ns_per_op(|| {
        let mut r = SizedRing::new(black_box(2048));
        for i in 0..1000 {
            let _ = r.push(i);
        }
        for i in 0..OPS as i64 {
            let _ = r.push(i);
            black_box(r.pop());
        }
    });
    // Go's `q = q[1:]` cannot be written on an owned Vec (example 1), so there
    // are two Rust spellings, and they are DIFFERENT ALGORITHMS:
    //   remove(0)   the naive queue of examples 1-3: shifts the backlog, Theta(n)
    //   head index  never moves data, abandons the prefix: Go's cost shape
    let t_remove = ns_per_op(|| {
        let mut q: Vec<i64> = Vec::with_capacity(1000);
        for i in 0..1000 {
            q.push(i);
        }
        for i in 0..OPS as i64 {
            q.push(i);
            black_box(q[0]);
            q.remove(0);
        }
    });
    let t_head = ns_per_op(|| {
        let mut q: Vec<i64> = Vec::with_capacity(1000);
        for i in 0..1000 {
            q.push(i);
        }
        // the head index is the loop counter: dequeue number `head` reads q[head]
        for (head, i) in (0..OPS as i64).enumerate() {
            q.push(i);
            black_box(q[head]);
        }
    });
    let t_list = ns_per_op(|| {
        let mut l: LinkedList<i64> = LinkedList::new();
        for i in 0..1000 {
            l.push_back(i);
        }
        for i in 0..OPS as i64 {
            l.push_back(i);
            black_box(l.pop_front());
        }
    });
    let t_deque = ns_per_op(|| {
        let mut d: VecDeque<i64> = VecDeque::with_capacity(black_box(2048));
        for i in 0..1000 {
            d.push_back(i);
        }
        for i in 0..OPS as i64 {
            d.push_back(i);
            black_box(d.pop_front());
        }
    });
    let ops = OPS as f64;
    println!("  {:<38} {:>10.2} ns/op", "ring buffer", t_ring / ops);
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "Vec remove(0), shifts (NOT Go's)",
        t_remove / ops,
        t_remove / t_ring
    );
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "Vec + head index (Go's q = q[1:])",
        t_head / ops,
        t_head / t_ring
    );
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "LinkedList",
        t_list / ops,
        t_list / t_ring
    );
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "std VecDeque (not in the Go example)",
        t_deque / ops,
        t_deque / t_ring
    );

    println!();
    println!("  Go's `q = q[1:]` has two Rust spellings and they are DIFFERENT");
    println!("  ALGORITHMS: remove(0) shifts the 1000-element backlog on every dequeue");
    println!("  (Theta(n)); the head index moves nothing and abandons the prefix, which");
    println!("  is Go's cost shape. In Go the slice beat the ring (0.7x). Here it did");
    println!("  not: across five runs the head-index queue was 1.5x to 2.2x SLOWER than");
    println!("  the ring, remove(0) 90x to 120x slower, LinkedList 20x to 28x and");
    println!("  VecDeque 3.3x to 4.3x. Plausible reason for the head index, not");
    println!("  measured: it never reuses a slot, so it writes into a Vec that grows to");
    println!("  201,000 elements (1.6 MB, doubling as it goes) while the ring stays in");
    println!("  a 16 KB array.");

    // Live heap blocks while the structure is alive: Go's HeapObjects after GC.
    const N: usize = 100_000;
    let (b0, y0) = heap();
    let mut l: LinkedList<i64> = LinkedList::new();
    for i in 0..N as i64 {
        l.push_back(i);
    }
    let (b1, y1) = heap();
    let (list_blocks, list_bytes) = (b1 - b0, y1 - y0);
    black_box(&l);
    drop(l);

    let (b0, y0) = heap();
    let mut r = SizedRing::new(1 << 17);
    for i in 0..N as i64 {
        let _ = r.push(i);
    }
    let (b1, y1) = heap();
    let (ring_blocks, ring_bytes) = (b1 - b0, y1 - y0);
    black_box(&r);
    drop(r);

    println!();
    println!(
        "  heap blocks for 100,000 i64: LinkedList {list_blocks}, ring {ring_blocks}   ({list_bytes} B vs {ring_bytes} B)"
    );
    println!();
    println!("  the ring is ONE block -- the backing array. The list is one block per");
    println!(
        "  value: a node holding the i64 and two pointers, {} bytes",
        list_bytes / N
    );
    println!("  each. There is no box on top of that (Go's list stores `any`, so each");
    println!("  int is boxed too: 149,869 objects there for the same 100,000 values),");
    println!("  and there is no GC to trace the blocks. What a Rust list still pays");
    println!("  for is one malloc and one free per element (counted above). That is");
    println!("  where I would look for the 20x, though I did not time them apart.");
    println!();
    println!("  use LinkedList for a queue only when you also need the thing a list");
    println!("  uniquely gives you: Theta(1) removal from the MIDDLE by a handle you");
    println!("  are already holding. Std's LinkedList cannot do even that on stable");
    println!("  Rust -- its cursors (`cursor_front_mut`, `remove_current`) are behind");
    println!("  the unstable `linked_list_cursors` feature (error[E0658]) -- so the");
    println!("  LRU cache cannot be built on it the way container/list builds one.");

    println!();
    println!("container/ring is the one that surprises people, and Rust has no");
    println!("version of it at all. It is NOT a ring BUFFER -- there is no head, no");
    println!("tail, no Push and no Pop. Built by hand, as an arena:");
    println!();
    let mut rings: Rings<i64> = Rings::new();
    let rr = rings.make(1..=5).expect("5 values make a non-empty ring");
    print!("  ring of 1..=5, then one lap: ");
    for v in rings.values(rr) {
        print!("{v} ");
    }
    println!();
    println!(
        "  step(2) then value:                {}",
        rings.nodes[rings.step(rr, 2)].value
    );
    println!(
        "  prev().value:                      {}",
        rings.nodes[rings.prev(rr)].value
    );
    println!();
    println!("  it is a CIRCULAR DOUBLY LINKED LIST of fixed length -- every");
    println!("  element is a node with next and prev, and the last one points back");
    println!("  at the first. Lesson 07's sentinel ring, with the sentinel removed");
    println!("  and the length frozen.");
    println!();

    let (b0, y0) = heap();
    let mut big: Rings<i64> = Rings::new();
    let cr = big.make(0..N as i64).expect("non-empty");
    let (b1, y1) = heap();
    black_box((&big, cr));
    println!(
        "  heap blocks for a 100,000-node arena ring: {}   ({} B, {} B per node)",
        b1 - b0,
        y1 - y0,
        (y1 - y0) / N
    );
    println!();
    println!(
        "  heap blocks for LinkedList, again, for comparison:   {}",
        list_blocks
    );
    println!();
    println!("  NOT the same, and that is the Rust finding. Go's container/ring pays");
    println!("  for a node AND a box per element, exactly like container/list, because");
    println!("  it is a linked structure of heap objects storing `any`. The arena ring");
    println!("  has the same topology and the same node (value + next + prev) but its");
    println!("  nodes are slots of one Vec, so the whole ring is one block. That is a");
    println!("  choice the pointer-based Go package cannot make, not a property of");
    println!("  rings. The price is that every ring built in one arena shares its");
    println!("  lifetime, and a handle is an index the compiler cannot check.");
    println!();
    println!("  a ring BUFFER of 100,000 i64 is one array too, but it is 8 bytes per");
    println!("  element, not 24, and its next slot is arithmetic, not a field. The");
    println!("  word 'ring' in the package name is about topology, not layout.");
    drop(big);

    println!();
    println!("what container/ring is actually FOR -- rotation and splicing:");
    println!();
    let mut s: Rings<&str> = Rings::new();
    let a = s.make(["a", "b", "c"]).expect("non-empty");
    let b = s.make(["X", "Y"]).expect("non-empty");
    s.show("ring a", a);
    s.show("ring b", b);
    s.link(a, b);
    s.show("a.link(b)", a);
    let rem = s.unlink(a, 2).expect("n > 0");
    s.show("after a.unlink(2)", a);
    s.show("  the removed part", rem);
    println!();
    println!("  link splices whole rings in Theta(1) -- four index writes, no copying,");
    println!("  no reallocation, no bounds. unlink is link plus a walk of n+1 steps to");
    println!("  find the cut, so it is Theta(n) in the nodes removed (Go's Unlink does");
    println!("  the same Move(n+1) walk); the splice itself is still four writes. That");
    println!("  is genuinely hard to do any other way, and it is why the package");
    println!("  exists: round-robin schedulers, token rings, cyclic buffers of");
    println!("  connections you rotate through.");
    println!();

    // The nearest thing std has: splice at the ENDS only.
    let mut la: LinkedList<&str> = ["a", "b", "c"].into();
    let mut lb: LinkedList<&str> = ["X", "Y"].into();
    la.append(&mut lb);
    let joined = format!("{la:?}");
    let tail = la.split_off(3);
    println!("  std's nearest: LinkedList::append and split_off, at the ENDS only:");
    println!(
        "    la.append(&mut lb)   {joined}   (lb is left empty: len {})",
        lb.len()
    );
    println!("    la.split_off(3)      {la:?} and {tail:?}");
    println!("  append is documented O(1); split_off must walk to the cut, so it is");
    println!("  not. Neither can splice into the MIDDLE, which is what link does.");

    println!();
    println!("the summary, and it is short:");
    println!();
    for (what, verdict) in [
        ("your own ring", "a queue. Use this (or VecDeque)."),
        (
            "LinkedList",
            "Theta(1) append of whole lists. Not queues, not LRU.",
        ),
        (
            "arena ring (no std)",
            "rotation and Theta(1) splicing. Not a buffer.",
        ),
        (
            "sync_channel",
            "a bounded queue you can block on. Example 8.",
        ),
    ] {
        println!("  {what:<22} {verdict}");
    }
    println!();
    println!("  Go's two container packages predate generics, store `any` and so box.");
    println!("  Rust's LinkedList<T> is generic and does not: the 24-byte node above is");
    println!("  all it costs per element. That removes the box and the GC tracing the");
    println!("  Go prose is about; it does not remove the per-node allocation, and the");
    println!("  list was still 20x to 28x slower than the ring on this queue.");
}
```

**Rust sample output:**

```
LinkedList IS a queue, in the sense that push_back and pop_front
are both Theta(1). Lesson 07 already measured what that costs;
here it is again on the queue workload specifically:

  ring buffer                                  0.70 ns/op
  Vec remove(0), shifts (NOT Go's)            74.24 ns/op   106.7x
  Vec + head index (Go's q = q[1:])            1.14 ns/op   1.6x
  LinkedList                                  15.34 ns/op   22.1x
  std VecDeque (not in the Go example)         2.71 ns/op   3.9x

  Go's `q = q[1:]` has two Rust spellings and they are DIFFERENT
  ALGORITHMS: remove(0) shifts the 1000-element backlog on every dequeue
  (Theta(n)); the head index moves nothing and abandons the prefix, which
  is Go's cost shape. In Go the slice beat the ring (0.7x). Here it did
  not: across five runs the head-index queue was 1.5x to 2.2x SLOWER than
  the ring, remove(0) 90x to 120x slower, LinkedList 20x to 28x and
  VecDeque 3.3x to 4.3x. Plausible reason for the head index, not
  measured: it never reuses a slot, so it writes into a Vec that grows to
  201,000 elements (1.6 MB, doubling as it goes) while the ring stays in
  a 16 KB array.

  heap blocks for 100,000 i64: LinkedList 100000, ring 1   (2400000 B vs 1048576 B)

  the ring is ONE block -- the backing array. The list is one block per
  value: a node holding the i64 and two pointers, 24 bytes
  each. There is no box on top of that (Go's list stores `any`, so each
  int is boxed too: 149,869 objects there for the same 100,000 values),
  and there is no GC to trace the blocks. What a Rust list still pays
  for is one malloc and one free per element (counted above). That is
  where I would look for the 20x, though I did not time them apart.

  use LinkedList for a queue only when you also need the thing a list
  uniquely gives you: Theta(1) removal from the MIDDLE by a handle you
  are already holding. Std's LinkedList cannot do even that on stable
  Rust -- its cursors (`cursor_front_mut`, `remove_current`) are behind
  the unstable `linked_list_cursors` feature (error[E0658]) -- so the
  LRU cache cannot be built on it the way container/list builds one.

container/ring is the one that surprises people, and Rust has no
version of it at all. It is NOT a ring BUFFER -- there is no head, no
tail, no Push and no Pop. Built by hand, as an arena:

  ring of 1..=5, then one lap: 1 2 3 4 5 
  step(2) then value:                3
  prev().value:                      5

  it is a CIRCULAR DOUBLY LINKED LIST of fixed length -- every
  element is a node with next and prev, and the last one points back
  at the first. Lesson 07's sentinel ring, with the sentinel removed
  and the length frozen.

  heap blocks for a 100,000-node arena ring: 1   (2400000 B, 24 B per node)

  heap blocks for LinkedList, again, for comparison:   100000

  NOT the same, and that is the Rust finding. Go's container/ring pays
  for a node AND a box per element, exactly like container/list, because
  it is a linked structure of heap objects storing `any`. The arena ring
  has the same topology and the same node (value + next + prev) but its
  nodes are slots of one Vec, so the whole ring is one block. That is a
  choice the pointer-based Go package cannot make, not a property of
  rings. The price is that every ring built in one arena shares its
  lifetime, and a handle is an index the compiler cannot check.

  a ring BUFFER of 100,000 i64 is one array too, but it is 8 bytes per
  element, not 24, and its next slot is arithmetic, not a field. The
  word 'ring' in the package name is about topology, not layout.

what container/ring is actually FOR -- rotation and splicing:

  ring a                 len=3  a b c 
  ring b                 len=2  X Y 
  a.link(b)              len=5  a X Y b c 
  after a.unlink(2)      len=3  a b c 
    the removed part     len=2  X Y 

  link splices whole rings in Theta(1) -- four index writes, no copying,
  no reallocation, no bounds. unlink is link plus a walk of n+1 steps to
  find the cut, so it is Theta(n) in the nodes removed (Go's Unlink does
  the same Move(n+1) walk); the splice itself is still four writes. That
  is genuinely hard to do any other way, and it is why the package
  exists: round-robin schedulers, token rings, cyclic buffers of
  connections you rotate through.

  std's nearest: LinkedList::append and split_off, at the ENDS only:
    la.append(&mut lb)   ["a", "b", "c", "X", "Y"]   (lb is left empty: len 0)
    la.split_off(3)      ["a", "b", "c"] and ["X", "Y"]
  append is documented O(1); split_off must walk to the cut, so it is
  not. Neither can splice into the MIDDLE, which is what link does.

the summary, and it is short:

  your own ring          a queue. Use this (or VecDeque).
  LinkedList             Theta(1) append of whole lists. Not queues, not LRU.
  arena ring (no std)    rotation and Theta(1) splicing. Not a buffer.
  sync_channel           a bounded queue you can block on. Example 8.

  Go's two container packages predate generics, store `any` and so box.
  Rust's LinkedList<T> is generic and does not: the 24-byte node above is
  all it costs per element. That removes the box and the GC tracing the
  Go prose is about; it does not remove the per-node allocation, and the
  list was still 20x to 28x slower than the ring on this queue.
```

> **Go vs Rust:** Rust's std has `LinkedList` but no `container/ring`, so the ring's topology is rebuilt as an arena (nodes in one `Vec`, `next`/`prev` as indices) and `Link`/`Unlink` are ported line for line: that ring is 1 heap block for 100,000 nodes against `LinkedList`'s 100,000 (generic 24-byte nodes, no `any` box), so Go's finding that `container/ring` costs the same as `container/list` does not carry over. `q = q[1:]` has no owned-`Vec` spelling: the head-index form (Go's cost shape) was 1.5x to 2.2x slower than the ring where Go's slice was faster, and `remove(0)` is a different algorithm (Θ(n), 90x to 120x slower). Std's handle-based middle removal (`LinkedList` cursors) is unstable (E0658 on rustc 1.98.1), and the heap-object count comes from a counting allocator instead of `runtime.GC()` + `MemStats`.

**Complexity:** `container/list` **8.6×** slower with **149,869 heap objects** to the ring's **1** ·
`container/ring` costs the same **149,867** — the word "ring" describes its topology, not its layout ·
`Link`/`Unlink` splice whole rings in Θ(1), which is what the package is actually for

---

## 8. A buffered channel is a ring buffer with a mutex

`🟡 medium` · *the question is never "channel or ring?"*

Go's runtime `hchan` holds a circular array, a send index, a receive index, a count and a lock — the
same three fields as example 3, plus the lock. So the real question is whether you need the lock.

**Steps:**

1. Watch `len`/`cap`/`select`-with-`default` behave exactly like example 3's ring.
2. Measure the ring, the ring behind a `sync.Mutex`, and the channel.
3. Then do the thing a ring buffer cannot do at all.

```go
package main

import (
	"fmt"
	"sync"
	"testing"
	"time"
)

// A buffered channel is a ring buffer with a mutex bolted to it. That is not a
// metaphor -- Go's runtime type `hchan` literally holds:
//
//	buf      unsafe.Pointer  // the circular array
//	sendx    uint            // tail
//	recvx    uint            // head
//	qcount   uint            // count -- same three fields as example 3
//	lock     mutex
//
// So the question is never "channel or ring buffer?" It is "do I need the
// mutex?" -- and if the queue lives inside one goroutine, you are paying for a
// lock that protects you from nobody.

type ring struct {
	buf         []int
	mask        int
	head, count int
}

func newRing(n int) *ring {
	c := 1
	for c < n {
		c <<= 1
	}
	return &ring{buf: make([]int, c), mask: c - 1}
}
func (r *ring) push(v int) {
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
}
func (r *ring) pop() int {
	v := r.buf[r.head]
	r.head = (r.head + 1) & r.mask
	r.count--
	return v
}

type lockedRing struct {
	mu sync.Mutex
	r  *ring
}

func (l *lockedRing) push(v int) {
	l.mu.Lock()
	l.r.push(v)
	l.mu.Unlock()
}
func (l *lockedRing) pop() int {
	l.mu.Lock()
	v := l.r.pop()
	l.mu.Unlock()
	return v
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("a buffered channel behaves exactly like example 3's ring, down to")
	fmt.Println("Len and Cap:")
	fmt.Println()
	ch := make(chan int, 3)
	fmt.Printf("  make(chan int, 3)   len=%d cap=%d\n", len(ch), cap(ch))
	for i := 1; i <= 3; i++ {
		ch <- i
		fmt.Printf("  ch <- %d             len=%d cap=%d\n", i, len(ch), cap(ch))
	}
	select {
	case ch <- 4:
		fmt.Println("  ch <- 4             accepted")
	default:
		fmt.Printf("  ch <- 4             REFUSED (len=%d == cap)\n", len(ch))
	}
	v := <-ch
	fmt.Printf("  <-ch                -> %d, len=%d\n", v, len(ch))

	fmt.Println()
	fmt.Println("  `select` with a `default` is TryEnqueue -- the non-blocking form")
	fmt.Println("  that returns example 3's `bool` instead of blocking. Without the")
	fmt.Println("  default, a full channel blocks the sender, which IS backpressure")
	fmt.Println("  and is the whole reason to choose a channel (example 9).")

	fmt.Println()
	fmt.Println("what the mutex costs when nobody is contending for it:")
	fmt.Println()
	const ops = 500_000
	tRing := nsPerOp(func() {
		r := newRing(2048)
		for i := 0; i < 1000; i++ {
			r.push(i)
		}
		for i := 0; i < ops; i++ {
			r.push(i)
			sinkI = r.pop()
		}
	})
	tLocked := nsPerOp(func() {
		l := &lockedRing{r: newRing(2048)}
		for i := 0; i < 1000; i++ {
			l.push(i)
		}
		for i := 0; i < ops; i++ {
			l.push(i)
			sinkI = l.pop()
		}
	})
	tChan := nsPerOp(func() {
		c := make(chan int, 2048)
		for i := 0; i < 1000; i++ {
			c <- i
		}
		for i := 0; i < ops; i++ {
			c <- i
			sinkI = <-c
		}
	})
	fmt.Printf("  %-32s %10.2f ns/op\n", "plain ring buffer", tRing/ops)
	fmt.Printf("  %-32s %10.2f ns/op   %.1fx\n", "ring + sync.Mutex", tLocked/ops, tLocked/tRing)
	fmt.Printf("  %-32s %10.2f ns/op   %.1fx\n", "buffered channel", tChan/ops, tChan/tRing)
	fmt.Println()
	fmt.Println("  all three single-goroutine, all three uncontended. The mutex is")
	fmt.Println("  not free and the channel is not the mutex -- a channel send does")
	fmt.Println("  more than lock/copy/unlock: it also checks for a waiting receiver")
	fmt.Println("  and may hand the value over directly, which is what makes it a")
	fmt.Println("  synchronisation primitive rather than a container.")
	fmt.Println()
	fmt.Println("  if your queue never crosses a goroutine boundary, that is the")
	fmt.Println("  price of a feature you are not using.")

	fmt.Println()
	fmt.Println("now the case a ring buffer cannot do at all:")
	fmt.Println()
	jobs := make(chan int, 64)
	results := make(chan int, 64)
	var wg sync.WaitGroup
	for w := 0; w < 4; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs { // ranges until the channel is CLOSED and drained
				results <- j * j
			}
		}()
	}
	go func() {
		for i := 1; i <= 10; i++ {
			jobs <- i
		}
		close(jobs) // the only correct way to say "no more work"
	}()
	go func() { wg.Wait(); close(results) }()

	sum := 0
	n := 0
	for r := range results {
		sum += r
		n++
	}
	fmt.Printf("  4 workers, 10 jobs -> %d results, sum of squares = %d\n", n, sum)
	fmt.Println()
	fmt.Println("  the ring buffer has no answer for any of this: blocking when full,")
	fmt.Println("  blocking when empty, waking exactly one waiter, `close` as an")
	fmt.Println("  end-of-stream signal, `range` that terminates. Those are the")
	fmt.Println("  channel's actual product; the ring buffer inside it is incidental.")

	fmt.Println()
	fmt.Println("and the case where a channel is the wrong tool even between")
	fmt.Println("goroutines -- you cannot look at it:")
	fmt.Println()
	c := make(chan int, 8)
	for i := 0; i < 5; i++ {
		c <- i
	}
	fmt.Printf("  len(ch)=%d cap(ch)=%d -- and that is ALL you can ask.\n", len(c), cap(c))
	fmt.Println()
	fmt.Println("  no peek at the front, no scan, no removal from the middle, no")
	fmt.Println("  priority, no iterating without consuming. len(ch) is also stale")
	fmt.Println("  the instant you read it, so it is a metric, never a decision.")
	fmt.Println()
	fmt.Println("  need any of those and you need a real structure behind a mutex,")
	fmt.Println("  with a channel or a sync.Cond used only to signal.")

	fmt.Println()
	fmt.Println("the deadlock a channel makes easy, since it is the other half:")
	fmt.Println()
	done := make(chan struct{})
	small := make(chan int, 2)
	go func() {
		for i := 0; i < 5; i++ { // 5 sends into a 2-slot channel, no receiver
			small <- i
		}
		close(done)
	}()
	select {
	case <-done:
		fmt.Println("  sender finished -- unexpected")
	case <-time.After(50 * time.Millisecond):
		fmt.Printf("  sender is blocked after filling %d/%d slots, still waiting\n",
			len(small), cap(small))
	}
	fmt.Println()
	fmt.Println("  that block is not a bug -- it is backpressure, and it is correct.")
	fmt.Println("  It becomes a deadlock only when nothing will ever receive. The")
	fmt.Println("  distinction is the entire subject of example 9.")
	fmt.Println()
	fmt.Println("  the rule: a channel when the queue CROSSES a goroutine boundary,")
	fmt.Println("  a ring buffer when it does not. Do not use a channel as a data")
	fmt.Println("  structure, and do not hand-roll goroutine handoff with a mutex.")
}
```

**Sample output:**

```
a buffered channel behaves exactly like example 3's ring, down to
Len and Cap:

  make(chan int, 3)   len=0 cap=3
  ch <- 1             len=1 cap=3
  ch <- 2             len=2 cap=3
  ch <- 3             len=3 cap=3
  ch <- 4             REFUSED (len=3 == cap)
  <-ch                -> 1, len=2

  `select` with a `default` is TryEnqueue -- the non-blocking form
  that returns example 3's `bool` instead of blocking. Without the
  default, a full channel blocks the sender, which IS backpressure
  and is the whole reason to choose a channel (example 9).

what the mutex costs when nobody is contending for it:

  plain ring buffer                      2.35 ns/op
  ring + sync.Mutex                      5.73 ns/op   2.4x
  buffered channel                      17.13 ns/op   7.3x

  all three single-goroutine, all three uncontended. The mutex is
  not free and the channel is not the mutex -- a channel send does
  more than lock/copy/unlock: it also checks for a waiting receiver
  and may hand the value over directly, which is what makes it a
  synchronisation primitive rather than a container.

  if your queue never crosses a goroutine boundary, that is the
  price of a feature you are not using.

now the case a ring buffer cannot do at all:

  4 workers, 10 jobs -> 10 results, sum of squares = 385

  the ring buffer has no answer for any of this: blocking when full,
  blocking when empty, waking exactly one waiter, `close` as an
  end-of-stream signal, `range` that terminates. Those are the
  channel's actual product; the ring buffer inside it is incidental.

and the case where a channel is the wrong tool even between
goroutines -- you cannot look at it:

  len(ch)=5 cap(ch)=8 -- and that is ALL you can ask.

  no peek at the front, no scan, no removal from the middle, no
  priority, no iterating without consuming. len(ch) is also stale
  the instant you read it, so it is a metric, never a decision.

  need any of those and you need a real structure behind a mutex,
  with a channel or a sync.Cond used only to signal.

the deadlock a channel makes easy, since it is the other half:

  sender is blocked after filling 2/2 slots, still waiting

  that block is not a bug -- it is backpressure, and it is correct.
  It becomes a deadlock only when nothing will ever receive. The
  distinction is the entire subject of example 9.

  the rule: a channel when the queue CROSSES a goroutine boundary,
  a ring buffer when it does not. Do not use a channel as a data
  structure, and do not hand-roll goroutine handoff with a mutex.
```

**Rust:**

```rust
use std::hint::black_box;
use std::sync::Mutex;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::sync::mpsc::{RecvTimeoutError, TrySendError, channel, sync_channel};
use std::thread;
use std::time::{Duration, Instant};

// A buffered channel is a ring buffer with a mutex bolted to it -- in Go. Go's
// runtime type `hchan` literally holds:
//
//	buf      unsafe.Pointer  // the circular array
//	sendx    uint            // tail
//	recvx    uint            // head
//	qcount   uint            // count -- same three fields as example 3
//	lock     mutex
//
// Rust's bounded channel is std::sync::mpsc::sync_channel(n). I read its
// source in this toolchain (library/std/src/sync/mpmc/array.rs, which mpsc
// wraps): for n > 0 it is Dmitry Vyukov's bounded MPMC queue -- a ring of slots
// with an atomic "stamp" each, and atomic head and tail, no lock on the fast
// path. A Mutex appears only inside the list of parked threads (SyncWaker).
// So "a ring with a mutex" is Go's design; Rust's is a ring with atomics. The
// benchmark below measures the real thing instead of assuming either.
//
// What Rust's channel does NOT have, compared with Go's:
//
//	len(ch), cap(ch)   no len() or capacity() on SyncSender / Receiver (E0599)
//	many receivers     mpsc is multi-PRODUCER, single-consumer: Receiver is not
//	                   Clone. Go's four workers sharing one `jobs` channel need a
//	                   Mutex<Receiver> here -- a lock on top of the channel.
//	close(ch)          there is no close: drop every Sender and the channel is
//	                   closed (the receiver's loop ends once it is drained).
//	select/default     `try_send` is the non-blocking form (it returns Full(v)).

/// The same power-of-two ring as example 7's `SizedRing`, under example 8's Go
/// name. push/pop return Result/Option instead of trusting the caller.
struct Ring {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
}

impl Ring {
    fn new(n: usize) -> Self {
        // Go loops `c <<= 1`; Rust has it built in.
        let c = n.next_power_of_two();
        Ring {
            buf: vec![0; c],
            mask: c - 1,
            head: 0,
            count: 0,
        }
    }
    fn push(&mut self, v: i64) -> Result<(), i64> {
        if self.count == self.buf.len() {
            return Err(v);
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
        Ok(())
    }
    fn pop(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
    }
}

/// Go's `lockedRing { mu sync.Mutex; r *ring }`. In Rust the Mutex OWNS the
/// data: the ring cannot be touched without taking the lock, and the guard
/// releases it when it goes out of scope (Go's Lock/Unlock pair, minus the
/// way to forget the Unlock).
struct LockedRing {
    r: Mutex<Ring>,
}

impl LockedRing {
    fn push(&self, v: i64) {
        let _ = self.r.lock().unwrap().push(v);
    }
    fn pop(&self) -> Option<i64> {
        self.r.lock().unwrap().pop()
    }
}

/// ns per call of `run`, measured for about 200 ms (kit: stand-in for testing.Benchmark).
fn ns_per_op(mut run: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            run();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

fn main() {
    println!("a buffered channel behaves like example 3's ring -- except that Rust");
    println!("will not tell you Len and Cap. sync_channel has neither, so this");
    println!("program keeps the count itself:");
    println!();
    let (tx, rx) = sync_channel::<i64>(3);
    let mut queued = 0;
    println!("  sync_channel(3)     queued={queued} cap=3");
    for i in 1..=3 {
        tx.send(i).unwrap();
        queued += 1;
        println!("  send({i})             queued={queued} cap=3");
    }
    // Go: select { case ch <- 4: ... default: ... }. Rust: try_send.
    match tx.try_send(4) {
        Ok(()) => println!("  try_send(4)         accepted"),
        Err(TrySendError::Full(v)) => println!(
            "  try_send({v})         REFUSED, Full({v}) handed back (queued={queued} == cap)"
        ),
        Err(TrySendError::Disconnected(v)) => println!("  try_send({v})         disconnected"),
    }
    let v = rx.recv().unwrap();
    queued -= 1;
    println!("  recv()              -> {v}, queued={queued}");

    println!();
    println!("  `try_send` is TryEnqueue -- the non-blocking form that returns");
    println!("  example 3's refusal instead of blocking, and like example 3's Err it");
    println!("  hands the value back inside the error. Plain `send` on a full channel");
    println!("  blocks the sender, which IS backpressure and is the whole reason to");
    println!("  choose a channel (example 9).");

    println!();
    println!("what the lock costs when nobody is contending for it:");
    println!();
    const OPS: usize = 500_000;
    // black_box on the capacity, as in example 3: a literal 2048 would let LLVM
    // fold the mask into a constant and the benchmark would time that instead.
    let t_ring = ns_per_op(|| {
        let mut r = Ring::new(black_box(2048));
        for i in 0..1000 {
            let _ = r.push(i);
        }
        for i in 0..OPS as i64 {
            let _ = r.push(i);
            black_box(r.pop());
        }
    });
    let t_locked = ns_per_op(|| {
        let l = LockedRing {
            r: Mutex::new(Ring::new(black_box(2048))),
        };
        for i in 0..1000 {
            l.push(i);
        }
        for i in 0..OPS as i64 {
            l.push(i);
            black_box(l.pop());
        }
    });
    let t_chan = ns_per_op(|| {
        let (tx, rx) = sync_channel::<i64>(black_box(2048));
        for i in 0..1000 {
            tx.send(i).unwrap();
        }
        for i in 0..OPS as i64 {
            tx.send(i).unwrap();
            black_box(rx.recv().unwrap());
        }
    });
    // Not in the Go example: the unbounded flavour, for comparison.
    let t_unbounded = ns_per_op(|| {
        let (tx, rx) = channel::<i64>();
        for i in 0..1000 {
            tx.send(i).unwrap();
        }
        for i in 0..OPS as i64 {
            tx.send(i).unwrap();
            black_box(rx.recv().unwrap());
        }
    });
    let ops = OPS as f64;
    println!("  {:<38} {:>10.2} ns/op", "plain ring buffer", t_ring / ops);
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "ring + Mutex",
        t_locked / ops,
        t_locked / t_ring
    );
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "sync_channel(2048)",
        t_chan / ops,
        t_chan / t_ring
    );
    println!(
        "  {:<38} {:>10.2} ns/op   {:.1}x",
        "channel() unbounded (not in Go)",
        t_unbounded / ops,
        t_unbounded / t_ring
    );
    println!();
    println!("  all single-threaded, all uncontended. The lock is not free: ring +");
    println!("  Mutex was 12x to 13x the plain ring across eleven runs, which is");
    println!("  roughly 4 ns per lock/unlock pair, two pairs per operation. Go");
    println!("  measured 2.4x. The channel is not the Mutex either, but here it is");
    println!("  close to it: sync_channel was 1.1x to 1.4x the ring + Mutex (13x to");
    println!("  18x the plain ring; Go: 7.3x), and the unbounded channel() was about");
    println!("  the same as sync_channel (within 25% in every run, usually a little");
    println!("  cheaper).");
    println!();
    println!("  two Rust facts sit behind those numbers. std's bounded channel is not");
    println!("  a ring behind a mutex (see the top of the file: atomics per slot, read");
    println!("  in the source, not measured here), and std's Mutex on macOS is a");
    println!("  pthread mutex, where Linux, Windows and the BSDs get a futex (read in");
    println!("  sys/sync/mutex). Plausible reason for 12x against Go's 2.4x, not");
    println!("  measured: this platform's uncontended lock is simply dearer.");
    println!();
    println!("  if your queue never crosses a thread boundary, that is the price of a");
    println!("  feature you are not using -- and it is bigger here than in Go.");

    println!();
    println!("now the case a ring buffer cannot do at all:");
    println!();
    let (jobs_tx, jobs_rx) = sync_channel::<i64>(64);
    let (results_tx, results_rx) = sync_channel::<i64>(64);
    // mpsc has ONE consumer. Go hands the same `jobs` channel to four workers;
    // here the single Receiver is shared by wrapping it in a Mutex.
    let jobs_rx = Mutex::new(jobs_rx);
    let (mut sum, mut n) = (0, 0);
    thread::scope(|s| {
        for _ in 0..4 {
            let results_tx = results_tx.clone();
            let jobs_rx = &jobs_rx;
            s.spawn(move || {
                loop {
                    // NOT `while let Ok(j) = jobs_rx.lock().unwrap().recv()`: a
                    // temporary in a `while let` scrutinee lives for the whole
                    // body, so the lock would be held while the job runs and the
                    // four workers would run one at a time. A `let` statement
                    // drops the guard at its semicolon.
                    let job = jobs_rx.lock().unwrap().recv();
                    let Ok(j) = job else { break }; // Err: closed and drained
                    results_tx.send(j * j).unwrap();
                }
            }); // this worker's results_tx clone is dropped here
        }
        s.spawn(move || {
            for i in 1..=10 {
                jobs_tx.send(i).unwrap();
            }
            // jobs_tx is dropped here: that IS Go's close(jobs). No call needed,
            // and none to forget.
        });
        // Go needs `go func() { wg.Wait(); close(results) }()` to end the range
        // below. Here the main thread drops its own sender, and the channel
        // closes by itself when the last worker's clone goes.
        drop(results_tx);
        for r in results_rx.iter() {
            sum += r;
            n += 1;
        }
    });
    println!("  4 workers, 10 jobs -> {n} results, sum of squares = {sum}");
    println!();
    println!("  the ring buffer has no answer for any of this: blocking when full,");
    println!("  blocking when empty, waking a waiter, an end-of-stream signal, and a");
    println!("  loop that terminates. Those are the channel's actual product.");
    println!();
    println!("  Rust moves two of Go's pieces. The end-of-stream signal is not a call:");
    println!("  it is dropping the last Sender, so there is no close() to forget (or to");
    println!("  call twice -- closing a closed channel panics in Go). And the pool had");
    println!("  to wrap the Receiver in a Mutex: mpsc is multi-producer, SINGLE-");
    println!("  consumer, and Receiver is not Clone. A multi-consumer channel exists in");
    println!("  std (std::sync::mpmc) but is unstable on this compiler (E0658), so");
    println!("  four workers sharing a queue take a lock you write yourself.");

    println!();
    println!("and the case where a channel is the wrong tool even between");
    println!("threads -- you cannot look at it:");
    println!();
    let (tx, rx) = sync_channel::<i64>(8);
    for i in 0..5 {
        tx.send(i).unwrap();
    }
    println!("  5 values sent into sync_channel(8). Go lets you ask len and cap and");
    println!("  nothing else; Rust does not even give you those. The only way to see");
    println!("  inside is to take what is there:");
    println!();
    let taken: Vec<i64> = rx.try_iter().collect();
    println!("  try_iter().collect() -> {taken:?}");
    println!("  try_recv() afterwards -> {:?}", rx.try_recv());
    println!();
    println!("  no peek at the front, no scan, no removal from the middle, no");
    println!("  priority, no iterating without consuming. And since a count of a");
    println!("  channel other threads are using is stale the instant you read it, the");
    println!("  missing len() is less of a loss than it looks.");
    println!();
    println!("  need any of those and you need a real structure behind a mutex, with");
    println!("  a Condvar used only to signal. That is example 9's queue.");

    println!();
    println!("the deadlock a channel makes easy, since it is the other half:");
    println!();
    let (tx, rx) = sync_channel::<i64>(2);
    let sent = AtomicUsize::new(0);
    let (done_tx, done_rx) = channel::<Option<i64>>();
    thread::scope(|s| {
        s.spawn(|| {
            let mut failed_at = None;
            for i in 0..5 {
                // 5 sends into a 2-slot channel, and nobody receiving
                if tx.send(i).is_err() {
                    failed_at = Some(i);
                    break;
                }
                sent.fetch_add(1, Relaxed);
            }
            let _ = done_tx.send(failed_at);
        });
        match done_rx.recv_timeout(Duration::from_millis(50)) {
            Ok(_) => println!("  sender finished -- unexpected"),
            Err(RecvTimeoutError::Timeout) => println!(
                "  sender is blocked after filling {}/2 slots, still waiting",
                sent.load(Relaxed)
            ),
            Err(RecvTimeoutError::Disconnected) => println!("  sender died -- unexpected"),
        }
        // Go leaks that goroutine when main returns. Here the scope would wait
        // for the thread forever, so something has to end it: dropping the
        // Receiver disconnects the channel and wakes the blocked send with Err.
        drop(rx);
        let failed_at = done_rx
            .recv()
            .unwrap()
            .expect("the blocked send fails once the receiver is gone");
        println!("  after drop(rx): the blocked send returned Err at item {failed_at},");
        println!("  the thread exited and the scope joined it");
    });
    println!();
    println!("  that block is not a bug -- it is backpressure, and it is correct. It");
    println!("  becomes a deadlock only when nothing will ever receive. Rust can see");
    println!("  one version of that: dropping the Receiver woke the blocked send with");
    println!("  Err, which Go has no equivalent of (there the goroutine just stays");
    println!("  blocked until the program exits). It cannot see the other version: a");
    println!("  Receiver that is alive and never calls recv still blocks the sender");
    println!("  for ever. The distinction is the entire subject of example 9.");
    println!();
    println!("  the rule: a channel when the queue CROSSES a thread boundary, a ring");
    println!("  buffer when it does not. Do not use a channel as a data structure,");
    println!("  and do not hand-roll thread handoff with a mutex and no Condvar.");
}
```

**Rust sample output:**

```
a buffered channel behaves like example 3's ring -- except that Rust
will not tell you Len and Cap. sync_channel has neither, so this
program keeps the count itself:

  sync_channel(3)     queued=0 cap=3
  send(1)             queued=1 cap=3
  send(2)             queued=2 cap=3
  send(3)             queued=3 cap=3
  try_send(4)         REFUSED, Full(4) handed back (queued=3 == cap)
  recv()              -> 1, queued=2

  `try_send` is TryEnqueue -- the non-blocking form that returns
  example 3's refusal instead of blocking, and like example 3's Err it
  hands the value back inside the error. Plain `send` on a full channel
  blocks the sender, which IS backpressure and is the whole reason to
  choose a channel (example 9).

what the lock costs when nobody is contending for it:

  plain ring buffer                            0.69 ns/op
  ring + Mutex                                 8.63 ns/op   12.4x
  sync_channel(2048)                          11.19 ns/op   16.1x
  channel() unbounded (not in Go)              9.39 ns/op   13.5x

  all single-threaded, all uncontended. The lock is not free: ring +
  Mutex was 12x to 13x the plain ring across eleven runs, which is
  roughly 4 ns per lock/unlock pair, two pairs per operation. Go
  measured 2.4x. The channel is not the Mutex either, but here it is
  close to it: sync_channel was 1.1x to 1.4x the ring + Mutex (13x to
  18x the plain ring; Go: 7.3x), and the unbounded channel() was about
  the same as sync_channel (within 25% in every run, usually a little
  cheaper).

  two Rust facts sit behind those numbers. std's bounded channel is not
  a ring behind a mutex (see the top of the file: atomics per slot, read
  in the source, not measured here), and std's Mutex on macOS is a
  pthread mutex, where Linux, Windows and the BSDs get a futex (read in
  sys/sync/mutex). Plausible reason for 12x against Go's 2.4x, not
  measured: this platform's uncontended lock is simply dearer.

  if your queue never crosses a thread boundary, that is the price of a
  feature you are not using -- and it is bigger here than in Go.

now the case a ring buffer cannot do at all:

  4 workers, 10 jobs -> 10 results, sum of squares = 385

  the ring buffer has no answer for any of this: blocking when full,
  blocking when empty, waking a waiter, an end-of-stream signal, and a
  loop that terminates. Those are the channel's actual product.

  Rust moves two of Go's pieces. The end-of-stream signal is not a call:
  it is dropping the last Sender, so there is no close() to forget (or to
  call twice -- closing a closed channel panics in Go). And the pool had
  to wrap the Receiver in a Mutex: mpsc is multi-producer, SINGLE-
  consumer, and Receiver is not Clone. A multi-consumer channel exists in
  std (std::sync::mpmc) but is unstable on this compiler (E0658), so
  four workers sharing a queue take a lock you write yourself.

and the case where a channel is the wrong tool even between
threads -- you cannot look at it:

  5 values sent into sync_channel(8). Go lets you ask len and cap and
  nothing else; Rust does not even give you those. The only way to see
  inside is to take what is there:

  try_iter().collect() -> [0, 1, 2, 3, 4]
  try_recv() afterwards -> Err(Empty)

  no peek at the front, no scan, no removal from the middle, no
  priority, no iterating without consuming. And since a count of a
  channel other threads are using is stale the instant you read it, the
  missing len() is less of a loss than it looks.

  need any of those and you need a real structure behind a mutex, with
  a Condvar used only to signal. That is example 9's queue.

the deadlock a channel makes easy, since it is the other half:

  sender is blocked after filling 2/2 slots, still waiting
  after drop(rx): the blocked send returned Err at item 2,
  the thread exited and the scope joined it

  that block is not a bug -- it is backpressure, and it is correct. It
  becomes a deadlock only when nothing will ever receive. Rust can see
  one version of that: dropping the Receiver woke the blocked send with
  Err, which Go has no equivalent of (there the goroutine just stays
  blocked until the program exits). It cannot see the other version: a
  Receiver that is alive and never calls recv still blocks the sender
  for ever. The distinction is the entire subject of example 9.

  the rule: a channel when the queue CROSSES a thread boundary, a ring
  buffer when it does not. Do not use a channel as a data structure,
  and do not hand-roll thread handoff with a mutex and no Condvar.
```

> **Go vs Rust:** `sync_channel` has no `len()` or `capacity()` (E0599), so the demo counts by hand; and it is not a ring behind a mutex but Vyukov's lock-free bounded queue (read in std's source), while std's `Mutex` on macOS is a pthread mutex, so the ratios differ from Go's: ring + `Mutex` was 12x to 13x the plain ring (Go 2.4x) and `sync_channel` 13x to 18x (Go 7.3x). `mpsc` is single-consumer, so Go's four workers sharing one `jobs` channel need a `Mutex<Receiver>`; `close` is dropping the last `Sender`, and `drop(rx)` wakes a blocked `send` with `Err`, so the blocked sender can be joined instead of leaking like Go's goroutine.

**Complexity:** ring **2.35 ns/op** · ring + `sync.Mutex` **5.73 ns/op** (2.4×) · buffered channel
**17.13 ns/op** (7.3×) — all single-goroutine and uncontended · what the channel sells is blocking,
`close` and `range`, and you cannot inspect it beyond `len` and `cap`

---

## 9. Bounded queues: the four things that can give

`🟡 medium` · *refusing to choose is choosing OOM*

Example 3's `Enqueue` returned `false`. This is what you do with it. If the producer outruns the
consumer for long enough, something must give — and there are exactly four candidates.

**Steps:**

1. Implement block, drop-newest, drop-oldest and error behind one ring.
2. Run the same too-fast producer through each and account for every item.
3. Then sweep the capacity and measure what it actually buys.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Example 3 ended with `Enqueue` returning false. This example is about what
// you do with that false, because it is the single most consequential design
// decision in any system with a queue in it:
//
//	producer faster than consumer, for long enough
//	  -> the queue grows without bound, or something gives
//
// There are exactly four things that can give, and you must pick one. Refusing
// to pick means picking "unbounded", which means picking OOM.

type policy int

const (
	block policy = iota
	dropNewest
	dropOldest
	errorOut
)

func (p policy) String() string {
	return [...]string{"block (backpressure)", "drop newest", "drop oldest", "return an error"}[p]
}

// bounded is a ring buffer with an overflow policy and full accounting, so we
// can measure what each policy actually costs.
type bounded struct {
	mu      sync.Mutex
	notFull *sync.Cond
	buf     []item
	mask    int
	head    int
	count   int
	pol     policy

	accepted, dropped, errors, blockedNS int64

	// per-item queueing delay: how long each value SAT in the queue.
	waitSumNS, waitMaxNS int64
	waitN                int64
}

// item carries its own enqueue timestamp, which is the only way to measure
// queueing delay -- the producer's block time measures something else entirely.
type item struct {
	v int
	t time.Time
}

func newBounded(capacity int, pol policy) *bounded {
	n := 1
	for n < capacity {
		n <<= 1
	}
	b := &bounded{buf: make([]item, n), mask: n - 1, pol: pol}
	b.notFull = sync.NewCond(&b.mu)
	return b
}

func (b *bounded) Enqueue(v int) (ok bool) {
	b.mu.Lock()
	defer b.mu.Unlock()

	if b.count == len(b.buf) {
		switch b.pol {
		case block:
			start := time.Now()
			for b.count == len(b.buf) {
				b.notFull.Wait() // the producer is now going as slow as the consumer
			}
			b.blockedNS += time.Since(start).Nanoseconds()
		case dropNewest:
			b.dropped++
			return false // the value we were handed is discarded
		case dropOldest:
			b.head = (b.head + 1) & b.mask // evict the front
			b.count--
			b.dropped++
		case errorOut:
			b.errors++
			return false // the CALLER decides -- retry, shed, log, page someone
		}
	}
	b.buf[(b.head+b.count)&b.mask] = item{v, time.Now()}
	b.count++
	b.accepted++
	return true
}

func (b *bounded) Dequeue() (int, bool) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if b.count == 0 {
		return 0, false
	}
	it := b.buf[b.head]
	b.head = (b.head + 1) & b.mask
	b.count--
	w := time.Since(it.t).Nanoseconds()
	b.waitSumNS += w
	b.waitN++
	if w > b.waitMaxNS {
		b.waitMaxNS = w
	}
	b.notFull.Signal()
	return it.v, true
}

// run drives one producer and one consumer where the producer is 4x faster,
// and reports what the policy did about it.
func run(pol policy, capacity, items int) *bounded {
	b := newBounded(capacity, pol)
	var wg sync.WaitGroup
	wg.Add(2)

	go func() { // producer: sends `items` values as fast as it can
		defer wg.Done()
		for i := 0; i < items; i++ {
			b.Enqueue(i)
		}
	}()
	go func() { // consumer: deliberately 4x slower
		defer wg.Done()
		got := 0
		for got < items {
			if _, ok := b.Dequeue(); ok {
				got++
				for j := 0; j < 400; j++ { // simulated work
					_ = j * j
				}
				continue
			}
			b.mu.Lock()
			done := b.accepted+b.dropped+b.errors >= int64(items) && b.count == 0
			b.mu.Unlock()
			if done {
				return
			}
			time.Sleep(time.Microsecond)
		}
	}()
	wg.Wait()
	return b
}

func main() {
	fmt.Println("the four policies, and the fact that there is no fifth:")
	fmt.Println()
	fmt.Printf("  %-22s %-38s %s\n", "policy", "what it costs", "when it is right")
	for _, r := range [][3]string{
		{"block", "producer latency; risk of deadlock", "the producer can wait"},
		{"drop newest", "the newest data", "metrics, telemetry, logs"},
		{"drop oldest", "the oldest data", "live video, sensors, prices"},
		{"return an error", "caller complexity", "an API boundary"},
	} {
		fmt.Printf("  %-22s %-38s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  every one of them LOSES something. That is not a flaw in the")
	fmt.Println("  design -- it is arithmetic. If the producer is faster than the")
	fmt.Println("  consumer for long enough, the only alternative to losing data is")
	fmt.Println("  losing the process.")

	fmt.Println()
	fmt.Println("the same producer (4x faster than the consumer), 20,000 items,")
	fmt.Println("capacity 64, under each policy:")
	fmt.Println()
	fmt.Printf("  %-22s %10s %10s %10s %14s\n", "policy", "accepted", "dropped", "errors", "producer blocked")
	for _, pol := range []policy{block, dropNewest, dropOldest, errorOut} {
		b := run(pol, 64, 20_000)
		blocked := fmt.Sprintf("%.1f ms", float64(b.blockedNS)/1e6)
		if b.blockedNS == 0 {
			blocked = "-"
		}
		fmt.Printf("  %-22s %10d %10d %10d %14s\n",
			pol, b.accepted, b.dropped, b.errors, blocked)
	}
	fmt.Println()
	fmt.Println("  `block` is the only row that accepted everything, and it paid for")
	fmt.Println("  that in producer latency -- which is exactly the point. Blocking")
	fmt.Println("  is not a failure mode, it is the mechanism by which a slow")
	fmt.Println("  consumer TELLS the producer to slow down. That is backpressure,")
	fmt.Println("  and a system without it does not have a queue, it has a countdown.")

	fmt.Println()
	fmt.Println("capacity is a latency dial, not a throughput dial:")
	fmt.Println()
	fmt.Printf("  %10s %16s %14s %14s\n", "capacity", "producer blocked", "mean wait", "worst wait")
	for _, c := range []int{8, 64, 512, 4096} {
		b := run(block, c, 20_000)
		fmt.Printf("  %10d %13.1f ms %11.1f us %11.1f us\n", c,
			float64(b.blockedNS)/1e6,
			float64(b.waitSumNS)/float64(b.waitN)/1e3,
			float64(b.waitMaxNS)/1e3)
	}
	fmt.Println()
	fmt.Println("  read those two right-hand columns, not the left one. The producer")
	fmt.Println("  blocks LESS as the buffer grows -- that is the buffer doing its")
	fmt.Println("  job -- but every item then waits behind more items, and the mean")
	fmt.Println("  and worst queueing delays climb with the capacity.")
	fmt.Println()
	fmt.Println("  a bigger buffer does not make the consumer faster -- accepted is")
	fmt.Println("  20,000 in every row and throughput is set by the consumer alone.")
	fmt.Println("  What a bigger buffer buys is a longer BURST absorbed before the")
	fmt.Println("  producer feels anything, paid for in latency and memory: 160x")
	fmt.Println("  more mean queueing delay to save 10 ms of producer blocking.")
	fmt.Println()
	fmt.Println("  `bufferbloat` is this exact mistake, made in routers, and it made")
	fmt.Println("  the internet slower for a decade.")
	fmt.Println()
	fmt.Println("  size the queue for the BURST you must absorb, then stop. If you")
	fmt.Println("  find yourself raising it repeatedly, the consumer is too slow and")
	fmt.Println("  no capacity will fix that.")

	fmt.Println()
	fmt.Println("drop-oldest is the one people forget, and it is often the right")
	fmt.Println("answer -- when the data is a SAMPLE of a continuous quantity:")
	fmt.Println()
	newest := newBounded(4, dropNewest)
	oldest := newBounded(4, dropOldest)
	for i := 1; i <= 8; i++ {
		newest.Enqueue(i)
		oldest.Enqueue(i)
	}
	drain := func(b *bounded) []int {
		var out []int
		for {
			v, ok := b.Dequeue()
			if !ok {
				return out
			}
			out = append(out, v)
		}
	}
	fmt.Printf("  pushed 1..8 into a 4-slot queue\n")
	fmt.Printf("  %-14s %v   <- stuck in the past\n", "drop newest:", drain(newest))
	fmt.Printf("  %-14s %v   <- the four most recent\n", "drop oldest:", drain(oldest))
	fmt.Println()
	fmt.Println("  for a video frame, a sensor reading or a price tick, the stale")
	fmt.Println("  value is worthless and the fresh one is the whole product --")
	fmt.Println("  drop-oldest is correct and drop-newest is a bug. For an audit log")
	fmt.Println("  or a payment, both are unacceptable and you must block or error.")
	fmt.Println()
	fmt.Println("  the policy is a property of the DATA, not of the queue. Decide it")
	fmt.Println("  where the data is defined, and make the type say which one it is.")
}
```

**Sample output:**

```
the four policies, and the fact that there is no fifth:

  policy                 what it costs                          when it is right
  block                  producer latency; risk of deadlock     the producer can wait
  drop newest            the newest data                        metrics, telemetry, logs
  drop oldest            the oldest data                        live video, sensors, prices
  return an error        caller complexity                      an API boundary

  every one of them LOSES something. That is not a flaw in the
  design -- it is arithmetic. If the producer is faster than the
  consumer for long enough, the only alternative to losing data is
  losing the process.

the same producer (4x faster than the consumer), 20,000 items,
capacity 64, under each policy:

  policy                   accepted    dropped     errors producer blocked
  block (backpressure)        20000          0          0         3.8 ms
  drop newest                    82      19918          0              -
  drop oldest                 20000      19348          0              -
  return an error               218          0      19782              -

  `block` is the only row that accepted everything, and it paid for
  that in producer latency -- which is exactly the point. Blocking
  is not a failure mode, it is the mechanism by which a slow
  consumer TELLS the producer to slow down. That is backpressure,
  and a system without it does not have a queue, it has a countdown.

capacity is a latency dial, not a throughput dial:

    capacity producer blocked      mean wait     worst wait
           8          19.5 ms         7.0 us       600.4 us
          64           3.5 ms         8.7 us        53.0 us
         512           2.7 ms        93.1 us       209.0 us
        4096           2.5 ms       881.4 us      1221.5 us

  read those two right-hand columns, not the left one. The producer
  blocks LESS as the buffer grows -- that is the buffer doing its
  job -- but every item then waits behind more items, and the mean
  and worst queueing delays climb with the capacity.

  a bigger buffer does not make the consumer faster -- accepted is
  20,000 in every row and throughput is set by the consumer alone.
  What a bigger buffer buys is a longer BURST absorbed before the
  producer feels anything, paid for in latency and memory: 160x
  more mean queueing delay to save 10 ms of producer blocking.

  `bufferbloat` is this exact mistake, made in routers, and it made
  the internet slower for a decade.

  size the queue for the BURST you must absorb, then stop. If you
  find yourself raising it repeatedly, the consumer is too slow and
  no capacity will fix that.

drop-oldest is the one people forget, and it is often the right
answer -- when the data is a SAMPLE of a continuous quantity:

  pushed 1..8 into a 4-slot queue
  drop newest:   [1 2 3 4]   <- stuck in the past
  drop oldest:   [5 6 7 8]   <- the four most recent

  for a video frame, a sensor reading or a price tick, the stale
  value is worthless and the fresh one is the whole product --
  drop-oldest is correct and drop-newest is a bug. For an audit log
  or a payment, both are unacceptable and you must block or error.

  the policy is a property of the DATA, not of the queue. Decide it
  where the data is defined, and make the type say which one it is.
```

**Rust:**

```rust
use std::fmt;
use std::hint::black_box;
use std::sync::mpsc::{TrySendError, sync_channel};
use std::sync::{Condvar, Mutex};
use std::thread;
use std::time::Instant;

// Example 3 ended with `enqueue` returning Err. This example is about what
// you do with that Err, because it is the single most consequential design
// decision in any system with a queue in it:
//
//	producer faster than consumer, for long enough
//	  -> the queue grows without bound, or something gives
//
// There are exactly four things that can give, and you must pick one. Refusing
// to pick means picking "unbounded", which means picking OOM.
//
// Where this differs from the Go program (the policies and the accounting are
// the same):
//
//	- the Mutex OWNS the queue state, so no field can be touched unlocked;
//	- the consumer WAITS on a second Condvar (not_empty) instead of polling
//	  with time.Sleep(time.Microsecond), and the producer announces the end
//	  with close() -- so the consumer ends exactly when the queue is closed
//	  and empty, and every item is accounted for;
//	- the rows add a `delivered` column and ASSERT that delivered + lost equals
//	  the items sent -- "account for every item" is checked, not eyeballed;
//	- the Go consumer's simulated work is a fixed 400 iterations; here the
//	  work size is chosen so that the producer is ~4x faster than the
//	  consumer, and the program MEASURES the actual ratio before it uses it.

#[derive(Clone, Copy, PartialEq)]
enum Policy {
    Block,
    DropNewest,
    DropOldest,
    ErrorOut,
}

impl fmt::Display for Policy {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // `pad` honours the width in `{:<22}`; `write_str` would ignore it.
        f.pad(match self {
            Policy::Block => "block (backpressure)",
            Policy::DropNewest => "drop newest",
            Policy::DropOldest => "drop oldest",
            Policy::ErrorOut => "return an error",
        })
    }
}

/// An item carries its own enqueue timestamp, which is the only way to measure
/// queueing delay -- the producer's block time measures something else entirely.
#[derive(Clone, Copy)]
struct Item {
    v: i64,
    t: Instant,
}

/// Everything the mutex protects: a ring buffer plus full accounting, so we can
/// measure what each policy actually costs.
struct Inner {
    buf: Vec<Option<Item>>,
    mask: usize,
    head: usize,
    count: usize,
    closed: bool,

    accepted: u64,
    dropped: u64,
    errors: u64,
    blocked_ns: u64,

    // per-item queueing delay: how long each value SAT in the queue.
    wait_sum_ns: u64,
    wait_max_ns: u64,
    delivered: u64,
}

/// A bounded ring buffer with an overflow policy.
struct Bounded {
    pol: Policy,
    inner: Mutex<Inner>,
    not_full: Condvar,
    not_empty: Condvar,
}

impl Inner {
    /// Takes the front item and does the delay accounting. Caller holds the lock.
    fn pop(&mut self) -> Option<i64> {
        // take() moves the item out and leaves None: Go's `it := buf[head]`
        // plus the clearing, in one operation.
        let it = self.buf[self.head].take()?;
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        let w = it.t.elapsed().as_nanos() as u64;
        self.wait_sum_ns += w;
        self.wait_max_ns = self.wait_max_ns.max(w);
        self.delivered += 1;
        Some(it.v)
    }
}

impl Bounded {
    fn new(capacity: usize, pol: Policy) -> Self {
        // Go loops `n <<= 1`; Rust has it built in.
        let n = capacity.next_power_of_two();
        Bounded {
            pol,
            inner: Mutex::new(Inner {
                buf: vec![None; n],
                mask: n - 1,
                head: 0,
                count: 0,
                closed: false,
                accepted: 0,
                dropped: 0,
                errors: 0,
                blocked_ns: 0,
                wait_sum_ns: 0,
                wait_max_ns: 0,
                delivered: 0,
            }),
            not_full: Condvar::new(),
            not_empty: Condvar::new(),
        }
    }

    /// Err(v) hands the value back when it was refused (drop newest, error):
    /// what to do with it is the CALLER's decision -- retry, shed, log, page
    /// someone. `let _ =` at a call site is "shed".
    fn enqueue(&self, v: i64) -> Result<(), i64> {
        let mut g = self.inner.lock().unwrap();
        if g.count == g.buf.len() {
            match self.pol {
                Policy::Block => {
                    let start = Instant::now();
                    while g.count == g.buf.len() {
                        // wait() releases the lock while parked and takes it
                        // back before returning: the producer is now going as
                        // slow as the consumer.
                        g = self.not_full.wait(g).unwrap();
                    }
                    g.blocked_ns += start.elapsed().as_nanos() as u64;
                }
                Policy::DropNewest => {
                    g.dropped += 1;
                    return Err(v); // the value we were handed is discarded
                }
                Policy::DropOldest => {
                    let head = g.head;
                    g.buf[head] = None; // evict the front
                    g.head = (head + 1) & g.mask;
                    g.count -= 1;
                    g.dropped += 1;
                }
                Policy::ErrorOut => {
                    g.errors += 1;
                    return Err(v);
                }
            }
        }
        let slot = (g.head + g.count) & g.mask;
        g.buf[slot] = Some(Item {
            v,
            t: Instant::now(),
        });
        g.count += 1;
        g.accepted += 1;
        drop(g);
        self.not_empty.notify_one();
        Ok(())
    }

    /// Non-blocking, like Go's Dequeue: None when empty.
    fn dequeue(&self) -> Option<i64> {
        let mut g = self.inner.lock().unwrap();
        let v = g.pop();
        drop(g);
        if v.is_some() {
            self.not_full.notify_one();
        }
        v
    }

    /// Blocks until there is an item; None only once the queue is closed AND
    /// drained.
    fn dequeue_wait(&self) -> Option<i64> {
        let mut g = self.inner.lock().unwrap();
        loop {
            if let Some(v) = g.pop() {
                drop(g);
                self.not_full.notify_one();
                return Some(v);
            }
            if g.closed {
                return None;
            }
            g = self.not_empty.wait(g).unwrap();
        }
    }

    /// The producer is done (Go: the consumer polled for it).
    fn close(&self) {
        self.inner.lock().unwrap().closed = true;
        self.not_empty.notify_all();
    }
}

/// The consumer's simulated work per item. Go loops 400 times; 250 here, which
/// on this machine made the producer about 4x faster than the consumer (main
/// measures and prints the real ratio every run).
const WORK: u64 = 250;
fn work() {
    for j in 0..WORK {
        black_box(j * j);
    }
}

struct Report {
    s: Inner,
    run_ms: f64,
}

/// run drives one producer and one consumer where the producer is ~4x faster,
/// and reports what the policy did about it.
fn run(pol: Policy, capacity: usize, items: i64) -> Report {
    let b = Bounded::new(capacity, pol);
    let t = Instant::now();
    thread::scope(|s| {
        s.spawn(|| {
            // producer: sends `items` values as fast as it can
            for i in 0..items {
                let _ = b.enqueue(i); // shed whatever the policy refuses
            }
            b.close();
        });
        s.spawn(|| {
            // consumer: deliberately slower
            while b.dequeue_wait().is_some() {
                work();
            }
        });
    });
    Report {
        run_ms: t.elapsed().as_secs_f64() * 1e3,
        s: b.inner.into_inner().unwrap(),
    }
}

/// Go's `%v` of a slice: [1 2 3]
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    println!("the four policies, and the fact that there is no fifth:");
    println!();
    println!(
        "  {:<22} {:<38} when it is right",
        "policy", "what it costs"
    );
    for r in [
        (
            "block",
            "producer latency; risk of deadlock",
            "the producer can wait",
        ),
        ("drop newest", "the newest data", "metrics, telemetry, logs"),
        (
            "drop oldest",
            "the oldest data",
            "live video, sensors, prices",
        ),
        ("return an error", "caller complexity", "an API boundary"),
    ] {
        println!("  {:<22} {:<38} {}", r.0, r.1, r.2);
    }
    println!();
    println!("  every one of them LOSES something. That is not a flaw in the");
    println!("  design -- it is arithmetic. If the producer is faster than the");
    println!("  consumer for long enough, the only alternative to losing data is");
    println!("  losing the process.");

    // How much faster is the producer, really? Measure each side alone, on the
    // calling thread, best of 5: a queue big enough that nothing is refused,
    // 20,000 items in, then 20,000 items out with the simulated work.
    const ITEMS: i64 = 20_000;
    let (mut prod_ns, mut cons_ns) = (f64::MAX, f64::MAX);
    for _ in 0..5 {
        let b = Bounded::new(32_768, Policy::Block);
        let t = Instant::now();
        for i in 0..ITEMS {
            let _ = b.enqueue(i);
        }
        prod_ns = prod_ns.min(t.elapsed().as_nanos() as f64 / ITEMS as f64);
        let t = Instant::now();
        while let Some(v) = b.dequeue() {
            black_box(v);
            work();
        }
        cons_ns = cons_ns.min(t.elapsed().as_nanos() as f64 / ITEMS as f64);
    }

    println!();
    println!("the same producer, 20,000 items, capacity 64, under each policy. First,");
    println!("how much faster is it? Each side alone, no contention, best of 5:");
    println!();
    println!(
        "  producer: {prod_ns:>6.1} ns/item   consumer: {cons_ns:>6.1} ns/item (dequeue + work)"
    );
    println!("  -> the producer is {:.1}x faster", cons_ns / prod_ns);
    println!();
    println!(
        "  {:<22} {:>10} {:>10} {:>10} {:>10} {:>16}",
        "policy", "accepted", "dropped", "errors", "delivered", "producer blocked"
    );
    for pol in [
        Policy::Block,
        Policy::DropNewest,
        Policy::DropOldest,
        Policy::ErrorOut,
    ] {
        let r = run(pol, 64, ITEMS);
        let s = &r.s;
        let blocked = if s.blocked_ns == 0 {
            "-".to_string()
        } else {
            format!("{:.1} ms", s.blocked_ns as f64 / 1e6)
        };
        println!(
            "  {:<22} {:>10} {:>10} {:>10} {:>10} {:>16}",
            pol, s.accepted, s.dropped, s.errors, s.delivered, blocked
        );
        // Every item is either delivered or lost, and lost means dropped
        // (at the door or evicted) or refused with an error.
        assert_eq!(
            s.delivered + s.dropped + s.errors,
            ITEMS as u64,
            "{pol}: an item went missing"
        );
    }
    println!();
    println!("  every row accounts for all 20,000 items -- the program asserts that");
    println!("  delivered + dropped + errors = 20,000 -- and `accepted` is not");
    println!("  `delivered`: it counts enqueues that succeeded, so under drop oldest it");
    println!("  is 20,000 while most of those were evicted before anyone read");
    println!("  them.");
    println!();
    println!("  `block` is the only row that delivered everything, and it paid for");
    println!("  that in producer latency -- which is exactly the point. Blocking");
    println!("  is not a failure mode, it is the mechanism by which a slow");
    println!("  consumer TELLS the producer to slow down. That is backpressure,");
    println!("  and a system without it does not have a queue, it has a countdown.");
    println!();
    println!("  the three lossy rows are decided by scheduling, not by the policy, and");
    println!("  they change on every run. Run at its measured speed for the whole of");
    println!("  the producer's 20,000 x 21 ns = 0.43 ms, the consumer would serve about");
    println!("  4,800 items; it delivered between 65 and 2,000 across all my runs (and");
    println!("  4,458 once, under drop oldest). Plausible mechanism, not measured: a");
    println!("  producer that never blocks keeps re-taking the mutex, and the consumer");
    println!("  has to win that lock and a CPU against it. The only stable facts in");
    println!("  those rows are the accounting and which end of the queue lost.");
    println!();
    println!("capacity is a latency dial, not a throughput dial:");
    println!();
    println!(
        "  {:>10} {:>16} {:>14} {:>14} {:>10}",
        "capacity", "producer blocked", "mean wait", "worst wait", "run time"
    );
    for c in [8, 64, 512, 4096] {
        let r = run(Policy::Block, c, ITEMS);
        let s = &r.s;
        println!(
            "  {:>10} {:>13.1} ms {:>11.1} us {:>11.1} us {:>7.1} ms",
            c,
            s.blocked_ns as f64 / 1e6,
            s.wait_sum_ns as f64 / s.delivered as f64 / 1e3,
            s.wait_max_ns as f64 / 1e3,
            r.run_ms
        );
    }
    println!();
    println!("  the absolute numbers moved by two to three times between my runs with");
    println!("  how busy the machine was; the orderings below did not. Read the right-");
    println!("  hand columns, not the left one. The producer blocks LESS as the buffer");
    println!("  grows -- that is the buffer doing its job -- but the big drop is from 8");
    println!("  slots to 64 (2x to 6x less blocked time); from 64 up it is a few ms,");
    println!("  noisy and not monotone as Go's table was. Meanwhile every item waits");
    println!("  behind more items: mean wait rose with every step in capacity, roughly");
    println!("  in proportion (3x to 13x for each 8x in slots, about 150x to 500x for");
    println!("  the full 512x), and the worst wait climbs with it from 64 slots up.");
    println!();
    println!("  Go's text says a bigger buffer does not touch throughput. The run time");
    println!("  column does not support that for SMALL buffers: 8 slots took 1.7x to 5x");
    println!("  as long as 4096 slots across all my runs, with the same consumer and");
    println!("  20,000 items delivered in every row. (Plausible mechanism, not");
    println!("  measured: with 8 slots the two threads hand off through the Condvar");
    println!("  over and over.) From 64 slots up the run time stayed within 2x, so");
    println!("  there the dial is mostly latency. Even then the run took 2x to 3.6x the");
    println!("  1.8 ms the consumer alone would need (20,000 x ~90 ns): two threads");
    println!("  sharing one mutex cost more than the single-threaded rates above.");
    println!();
    println!("  What a bigger buffer buys is a longer BURST absorbed before the");
    println!("  producer feels anything, paid for in latency and memory.");
    println!();
    println!("  `bufferbloat` is this exact mistake, made in routers, and it made");
    println!("  the internet slower for a decade.");
    println!();
    println!("  size the queue for the BURST you must absorb, then stop. If you");
    println!("  find yourself raising it repeatedly, the consumer is too slow and");
    println!("  no capacity will fix that.");

    println!();
    println!("drop-oldest is the one people forget, and it is often the right");
    println!("answer -- when the data is a SAMPLE of a continuous quantity:");
    println!();
    let newest = Bounded::new(4, Policy::DropNewest);
    let oldest = Bounded::new(4, Policy::DropOldest);
    for i in 1..=8 {
        let _ = newest.enqueue(i);
        let _ = oldest.enqueue(i);
    }
    let drain = |b: &Bounded| -> Vec<i64> { std::iter::from_fn(|| b.dequeue()).collect() };
    println!("  pushed 1..8 into a 4-slot queue");
    println!(
        "  {:<14} {}   <- stuck in the past",
        "drop newest:",
        show(&drain(&newest))
    );
    println!(
        "  {:<14} {}   <- the four most recent",
        "drop oldest:",
        show(&drain(&oldest))
    );
    println!();
    println!("  for a video frame, a sensor reading or a price tick, the stale");
    println!("  value is worthless and the fresh one is the whole product --");
    println!("  drop-oldest is correct and drop-newest is a bug. For an audit log");
    println!("  or a payment, both are unacceptable and you must block or error.");
    println!();
    println!("  the policy is a property of the DATA, not of the queue. Decide it");
    println!("  where the data is defined, and make the type say which one it is.");

    println!();
    println!("and what std's own bounded channel gives you for free:");
    println!();
    let (tx, rx) = sync_channel::<i64>(4);
    let (mut kept, mut refused) = (Vec::new(), Vec::new());
    for i in 1..=8 {
        match tx.try_send(i) {
            Ok(()) => kept.push(i),
            Err(TrySendError::Full(v) | TrySendError::Disconnected(v)) => refused.push(v),
        }
    }
    let got: Vec<i64> = rx.try_iter().collect();
    println!(
        "  sync_channel(4), try_send 1..8: kept {}, refused with Full(v): {}, received {}",
        show(&kept),
        show(&refused),
        show(&got)
    );
    println!();
    println!("  std's sync_channel is two of the four policies: send blocks (block),");
    println!("  try_send refuses with Full(v) (return an error -- and drop the v and it");
    println!("  is drop newest). It cannot be drop-oldest: only the Receiver can reach the");
    println!("  front of the queue and a Sender never holds it. That policy needs your");
    println!("  own queue, like Bounded above.");
}
```

**Rust sample output:**

```
the four policies, and the fact that there is no fifth:

  policy                 what it costs                          when it is right
  block                  producer latency; risk of deadlock     the producer can wait
  drop newest            the newest data                        metrics, telemetry, logs
  drop oldest            the oldest data                        live video, sensors, prices
  return an error        caller complexity                      an API boundary

  every one of them LOSES something. That is not a flaw in the
  design -- it is arithmetic. If the producer is faster than the
  consumer for long enough, the only alternative to losing data is
  losing the process.

the same producer, 20,000 items, capacity 64, under each policy. First,
how much faster is it? Each side alone, no contention, best of 5:

  producer:   21.3 ns/item   consumer:   89.1 ns/item (dequeue + work)
  -> the producer is 4.2x faster

  policy                   accepted    dropped     errors  delivered producer blocked
  block (backpressure)        20000          0          0      20000           3.6 ms
  drop newest                   237      19763          0        237                -
  drop oldest                 20000      19249          0        751                -
  return an error              1069          0      18931       1069                -

  every row accounts for all 20,000 items -- the program asserts that
  delivered + dropped + errors = 20,000 -- and `accepted` is not
  `delivered`: it counts enqueues that succeeded, so under drop oldest it
  is 20,000 while most of those were evicted before anyone read
  them.

  `block` is the only row that delivered everything, and it paid for
  that in producer latency -- which is exactly the point. Blocking
  is not a failure mode, it is the mechanism by which a slow
  consumer TELLS the producer to slow down. That is backpressure,
  and a system without it does not have a queue, it has a countdown.

  the three lossy rows are decided by scheduling, not by the policy, and
  they change on every run. Run at its measured speed for the whole of
  the producer's 20,000 x 21 ns = 0.43 ms, the consumer would serve about
  4,800 items; it delivered between 65 and 2,000 across all my runs (and
  4,458 once, under drop oldest). Plausible mechanism, not measured: a
  producer that never blocks keeps re-taking the mutex, and the consumer
  has to win that lock and a CPU against it. The only stable facts in
  those rows are the accounting and which end of the queue lost.

capacity is a latency dial, not a throughput dial:

    capacity producer blocked      mean wait     worst wait   run time
           8           7.6 ms         2.1 us        31.1 us     9.4 ms
          64           3.8 ms        10.8 us        51.3 us     6.0 ms
         512           3.2 ms       125.0 us       162.5 us     5.2 ms
        4096           2.5 ms       812.9 us      1090.6 us     4.3 ms

  the absolute numbers moved by two to three times between my runs with
  how busy the machine was; the orderings below did not. Read the right-
  hand columns, not the left one. The producer blocks LESS as the buffer
  grows -- that is the buffer doing its job -- but the big drop is from 8
  slots to 64 (2x to 6x less blocked time); from 64 up it is a few ms,
  noisy and not monotone as Go's table was. Meanwhile every item waits
  behind more items: mean wait rose with every step in capacity, roughly
  in proportion (3x to 13x for each 8x in slots, about 150x to 500x for
  the full 512x), and the worst wait climbs with it from 64 slots up.

  Go's text says a bigger buffer does not touch throughput. The run time
  column does not support that for SMALL buffers: 8 slots took 1.7x to 5x
  as long as 4096 slots across all my runs, with the same consumer and
  20,000 items delivered in every row. (Plausible mechanism, not
  measured: with 8 slots the two threads hand off through the Condvar
  over and over.) From 64 slots up the run time stayed within 2x, so
  there the dial is mostly latency. Even then the run took 2x to 3.6x the
  1.8 ms the consumer alone would need (20,000 x ~90 ns): two threads
  sharing one mutex cost more than the single-threaded rates above.

  What a bigger buffer buys is a longer BURST absorbed before the
  producer feels anything, paid for in latency and memory.

  `bufferbloat` is this exact mistake, made in routers, and it made
  the internet slower for a decade.

  size the queue for the BURST you must absorb, then stop. If you
  find yourself raising it repeatedly, the consumer is too slow and
  no capacity will fix that.

drop-oldest is the one people forget, and it is often the right
answer -- when the data is a SAMPLE of a continuous quantity:

  pushed 1..8 into a 4-slot queue
  drop newest:   [1 2 3 4]   <- stuck in the past
  drop oldest:   [5 6 7 8]   <- the four most recent

  for a video frame, a sensor reading or a price tick, the stale
  value is worthless and the fresh one is the whole product --
  drop-oldest is correct and drop-newest is a bug. For an audit log
  or a payment, both are unacceptable and you must block or error.

  the policy is a property of the DATA, not of the queue. Decide it
  where the data is defined, and make the type say which one it is.

and what std's own bounded channel gives you for free:

  sync_channel(4), try_send 1..8: kept [1 2 3 4], refused with Full(v): [5 6 7 8], received [1 2 3 4]

  std's sync_channel is two of the four policies: send blocks (block),
  try_send refuses with Full(v) (return an error -- and drop the v and it
  is drop newest). It cannot be drop-oldest: only the Receiver can reach the
  front of the queue and a Sender never holds it. That policy needs your
  own queue, like Bounded above.
```

> **Go vs Rust:** Go's four policies, the drop-oldest vs drop-newest result (`[1 2 3 4]` against `[5 6 7 8]`) and the accounting carry over; the Mutex now owns the ring, the consumer waits on a second `Condvar` and ends on an explicit `close()` instead of polling with `time.Sleep`, and the program asserts that delivered + dropped + errors equals the 20,000 items. Rust contradicts "throughput is set by the consumer alone" for small buffers (8 slots took 1.7x to 5x as long as 4096 across my runs), and the lossy rows' counts are scheduling noise (65 to 2,000 delivered; Go's 82 and 218 are the same kind of number). std's `sync_channel` covers only block and return-an-error: `try_send` hands the refused value back as `Full(v)`, and drop-oldest needs your own queue because only the `Receiver` can reach the front.

**Complexity:** every policy loses something — that is arithmetic, not a design flaw · capacity is a
**latency** dial, not a throughput dial: 8 → 4096 cuts producer blocking from 19.5 ms to 2.5 ms and
raises mean queueing delay from **7.0 µs to 881.4 µs** (126×). That is bufferbloat, measured

---

## 10. The monotonic deque: sliding-window maximum in Θ(n)

`🟡 medium` · *expire from one end, evict from the other*

Lesson 08 built a monotonic *stack*. Opening the second end lets you expire elements that have fallen
out of a window — which turns an Θ(nk) problem into an Θ(n) one with no heap and no allocations.

**Steps:**

1. Trace the deque of indices and watch the two rules fire.
2. Count the pushes and pops to make the aggregate argument concrete.
3. Prove it against brute force *on inputs with ties*, then benchmark it.

```go
package main

import (
	"fmt"
	"math/rand"
	"testing"
)

// The MONOTONIC DEQUE is what a deque is for. Lesson 08 built a monotonic
// STACK; this is the same idea with the second end opened up, and the extra end
// is what lets you EXPIRE elements that have fallen out of a window.
//
// Problem: sliding window maximum. Given a and k, report max(a[i:i+k]) for
// every window.
//
//	brute force  Theta(n*k)
//	heap         Theta(n log k)   (lesson 20)
//	deque        Theta(n)         -- and no allocations after the first
//
// The invariant: the deque holds INDICES whose values are strictly decreasing.
// Everything follows from defending that one sentence.

func maxSlidingBrute(a []int, k int) []int {
	if k <= 0 || len(a) < k {
		return nil
	}
	out := make([]int, 0, len(a)-k+1)
	for i := 0; i+k <= len(a); i++ {
		m := a[i]
		for j := i + 1; j < i+k; j++ {
			if a[j] > m {
				m = a[j]
			}
		}
		out = append(out, m)
	}
	return out
}

func maxSliding(a []int, k int) []int {
	if k <= 0 || len(a) < k {
		return nil
	}
	out := make([]int, 0, len(a)-k+1)
	d := NewDeque[int](k + 1) // holds INDICES, values strictly decreasing

	for i, v := range a {
		// 1. EXPIRE from the front: the index at the front has left the window.
		if f, ok := d.Front(); ok && f <= i-k {
			d.PopFront()
		}
		// 2. MAINTAIN from the back: anything smaller than v can never be the
		//    maximum of any future window, because v is both larger and newer.
		for {
			b, ok := d.Back()
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
		}
		d.PushBack(i)
		// 3. The front is now the maximum of the current window, by induction.
		if i >= k-1 {
			out = append(out, a[d.At(0)])
		}
	}
	return out
}

// counted is the same algorithm with the pushes and pops counted, to make the
// aggregate argument concrete.
func maxSlidingCounted(a []int, k int) (pushes, pops int) {
	d := NewDeque[int](k + 1)
	for i, v := range a {
		if f, ok := d.Front(); ok && f <= i-k {
			d.PopFront()
			pops++
		}
		for {
			b, ok := d.Back()
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
			pops++
		}
		d.PushBack(i)
		pushes++
	}
	return pushes, pops
}

func equal(x, y []int) bool {
	if len(x) != len(y) {
		return false
	}
	for i := range x {
		if x[i] != y[i] {
			return false
		}
	}
	return true
}

var sink []int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	a := []int{1, 3, -1, -3, 5, 3, 6, 7}
	k := 3

	fmt.Printf("a = %v, k = %d\n", a, k)
	fmt.Println()
	fmt.Println("the deque holds INDICES whose values strictly decrease. Watch it:")
	fmt.Println()
	fmt.Printf("  %3s %5s  %-22s %-22s %s\n", "i", "a[i]", "deque (indices)", "their values", "window max")

	d := NewDeque[int](k + 1)
	for i, v := range a {
		if f, ok := d.Front(); ok && f <= i-k {
			d.PopFront()
		}
		for {
			b, ok := d.Back()
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
		}
		d.PushBack(i)

		idx := make([]int, d.Len())
		vals := make([]int, d.Len())
		for j := 0; j < d.Len(); j++ {
			idx[j] = d.At(j)
			vals[j] = a[d.At(j)]
		}
		out := ""
		if i >= k-1 {
			out = fmt.Sprintf("%d", a[d.At(0)])
		}
		fmt.Printf("  %3d %5d  %-22s %-22s %s\n", i, v,
			fmt.Sprint(idx), fmt.Sprint(vals), out)
	}
	fmt.Println()
	fmt.Println("  two things happen to an index, and only two:")
	fmt.Println()
	fmt.Println("    it EXPIRES off the front  -- it left the window (i-k)")
	fmt.Println("    it is EVICTED off the back -- a larger, newer value arrived,")
	fmt.Println("                                  so it can never win again")
	fmt.Println()
	fmt.Println("  the second rule is the one that needs proof, and the proof is one")
	fmt.Println("  sentence: if a[j] <= a[i] and j < i, then every window containing")
	fmt.Println("  j either contains i too -- in which case j is not the maximum --")
	fmt.Println("  or has already passed. Either way j is dead. Delete it.")

	fmt.Println()
	fmt.Println("Theta(n) by the aggregate method, exactly like lesson 08's")
	fmt.Println("monotonic stack:")
	fmt.Println()
	fmt.Printf("  %10s %8s %10s %10s %14s\n", "n", "k", "pushes", "pops", "ops/element")
	rng := rand.New(rand.NewSource(1))
	for _, n := range []int{1000, 10_000, 100_000, 1_000_000} {
		arr := make([]int, n)
		for i := range arr {
			arr[i] = rng.Intn(1_000_000)
		}
		pu, po := maxSlidingCounted(arr, 100)
		fmt.Printf("  %10d %8d %10d %10d %14.2f\n", n, 100, pu, po, float64(pu+po)/float64(n))
	}
	fmt.Println()
	fmt.Println("  the inner `for` loop looks like it makes this quadratic. It does")
	fmt.Println("  not: each index is pushed exactly once and popped at most once, so")
	fmt.Println("  the total work over the whole run is at most 2n regardless of what")
	fmt.Println("  any single step does. Never read a nested loop as O(n^2) without")
	fmt.Println("  asking what it is a loop OVER.")
	fmt.Println()
	fmt.Println("  note ops/element is independent of k -- which is the whole point.")

	fmt.Println()
	fmt.Println("proved against brute force before it is benchmarked, because a fast")
	fmt.Println("wrong answer is worth nothing:")
	fmt.Println()
	fails := 0
	cases := 0
	for trial := 0; trial < 2000; trial++ {
		n := 1 + rng.Intn(30)
		arr := make([]int, n)
		for i := range arr {
			arr[i] = rng.Intn(21) - 10 // small range -> lots of ties
		}
		for kk := 1; kk <= n; kk++ {
			cases++
			if !equal(maxSliding(arr, kk), maxSlidingBrute(arr, kk)) {
				fails++
			}
		}
	}
	fmt.Printf("  %d random (array, k) pairs, ties and negatives included: %d mismatches\n", cases, fails)
	fmt.Println()
	fmt.Println("  the small value range matters. With `a[b] >= v` instead of")
	fmt.Println("  `a[b] > v` in the eviction test the answers are still CORRECT --")
	fmt.Println("  equal elements are interchangeable -- but the deque gets shorter.")
	fmt.Println("  Random values from a large range almost never tie, so a test that")
	fmt.Println("  uses them exercises neither branch. Force the collisions.")

	fmt.Println()
	fmt.Println("and now the benchmark, on the input that actually distinguishes")
	fmt.Println("them -- lesson 08's warning applies here too:")
	fmt.Println()
	fmt.Printf("  %8s %10s %14s %14s %10s\n", "n", "k", "brute (ns)", "deque (ns)", "speedup")
	for _, tc := range []struct{ n, k int }{
		{100_000, 4}, {100_000, 64}, {100_000, 1000}, {100_000, 10_000},
	} {
		arr := make([]int, tc.n)
		for i := range arr {
			arr[i] = rng.Intn(1_000_000)
		}
		tb := nsPerOp(func() { sink = maxSlidingBrute(arr, tc.k) })
		td := nsPerOp(func() { sink = maxSliding(arr, tc.k) })
		fmt.Printf("  %8d %10d %14.0f %14.0f %9.2fx\n", tc.n, tc.k, tb, td, tb/td)
	}
	fmt.Println()
	fmt.Println("  the deque's time is FLAT in k -- under 0.9 ms whether the window")
	fmt.Println("  is 4 wide or 10,000 -- while brute force is linear in it.")
	fmt.Println()
	fmt.Println("  and at k=4 brute force WINS, by nearly 2x. That is not noise: a")
	fmt.Println("  4-element scan is four array reads with no branches, against two")
	fmt.Println("  non-inlined deque method calls plus the bookkeeping. Theta(n) beats")
	fmt.Println("  Theta(nk) only once k is large enough to pay for the constant.")
	fmt.Println()
	fmt.Println("  the crossover here is somewhere between k=4 and k=64. If your")
	fmt.Println("  window is small and FIXED, write the loop. The deque earns its")
	fmt.Println("  complexity when k is large, or when k is a parameter you do not")
	fmt.Println("  control -- because then the crossover is not yours to reason about.")
	fmt.Println()
	fmt.Println("  the same skeleton -- expire from one end, evict from the other --")
	fmt.Println("  solves minimum-in-window, 'first negative in every window',")
	fmt.Println("  'longest subarray with max-min <= limit' (two deques), and the")
	fmt.Println("  constrained-window DP in lesson 30. Learn the invariant, not the")
	fmt.Println("  problem.")
}
```

> Reuses `Deque[T]` from example 5 — copy that type into the same folder as `deque.go`.

**Sample output:**

```
a = [1 3 -1 -3 5 3 6 7], k = 3

the deque holds INDICES whose values strictly decrease. Watch it:

    i  a[i]  deque (indices)        their values           window max
    0     1  [0]                    [1]                    
    1     3  [1]                    [3]                    
    2    -1  [1 2]                  [3 -1]                 3
    3    -3  [1 2 3]                [3 -1 -3]              3
    4     5  [4]                    [5]                    5
    5     3  [4 5]                  [5 3]                  5
    6     6  [6]                    [6]                    6
    7     7  [7]                    [7]                    7

  two things happen to an index, and only two:

    it EXPIRES off the front  -- it left the window (i-k)
    it is EVICTED off the back -- a larger, newer value arrived,
                                  so it can never win again

  the second rule is the one that needs proof, and the proof is one
  sentence: if a[j] <= a[i] and j < i, then every window containing
  j either contains i too -- in which case j is not the maximum --
  or has already passed. Either way j is dead. Delete it.

Theta(n) by the aggregate method, exactly like lesson 08's
monotonic stack:

           n        k     pushes       pops    ops/element
        1000      100       1000        995           2.00
       10000      100      10000       9996           2.00
      100000      100     100000      99993           2.00
     1000000      100    1000000     999998           2.00

  the inner `for` loop looks like it makes this quadratic. It does
  not: each index is pushed exactly once and popped at most once, so
  the total work over the whole run is at most 2n regardless of what
  any single step does. Never read a nested loop as O(n^2) without
  asking what it is a loop OVER.

  note ops/element is independent of k -- which is the whole point.

proved against brute force before it is benchmarked, because a fast
wrong answer is worth nothing:

  30796 random (array, k) pairs, ties and negatives included: 0 mismatches

  the small value range matters. With `a[b] >= v` instead of
  `a[b] > v` in the eviction test the answers are still CORRECT --
  equal elements are interchangeable -- but the deque gets shorter.
  Random values from a large range almost never tie, so a test that
  uses them exercises neither branch. Force the collisions.

and now the benchmark, on the input that actually distinguishes
them -- lesson 08's warning applies here too:

         n          k     brute (ns)     deque (ns)    speedup
    100000          4         443778         854594      0.52x
    100000         64        3996921         878460      4.55x
    100000       1000       31388885         856741     36.64x
    100000      10000      237704533         910696    261.01x

  the deque's time is FLAT in k -- under 0.9 ms whether the window
  is 4 wide or 10,000 -- while brute force is linear in it.

  and at k=4 brute force WINS, by nearly 2x. That is not noise: a
  4-element scan is four array reads with no branches, against two
  non-inlined deque method calls plus the bookkeeping. Theta(n) beats
  Theta(nk) only once k is large enough to pay for the constant.

  the crossover here is somewhere between k=4 and k=64. If your
  window is small and FIXED, write the loop. The deque earns its
  complexity when k is large, or when k is a parameter you do not
  control -- because then the crossover is not yours to reason about.

  the same skeleton -- expire from one end, evict from the other --
  solves minimum-in-window, 'first negative in every window',
  'longest subarray with max-min <= limit' (two deques), and the
  constrained-window DP in lesson 30. Learn the invariant, not the
  problem.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::fmt::Display;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- the kit's allocation counter: stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)

unsafe impl GlobalAlloc for Counting {
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

/// allocation events while running `f`
fn allocs_of<R>(f: impl FnOnce() -> R) -> usize {
    let a0 = ALLOCS.load(Relaxed);
    black_box(f());
    ALLOCS.load(Relaxed) - a0
}

// ---- RNG: stand-in for math/rand (the kit's splitmix64; seeded, so runs are reproducible) ----
// It is NOT Go's generator, so every number below that comes from random data
// (the pop counts, the number of random cases, the timings' inputs) is Rust's own.
struct Rng(u64);
impl Rng {
    fn new(seed: u64) -> Self {
        Rng(seed)
    }
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// ---- ns/op: stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut run: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            run();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// The MONOTONIC DEQUE is what a deque is for. Lesson 08 built a monotonic
// STACK; this is the same idea with the second end opened up, and the extra end
// is what lets you EXPIRE elements that have fallen out of a window.
//
// Problem: sliding window maximum. Given a and k, report max(a[i..i+k]) for
// every window.
//
//	brute force  Theta(n*k)
//	heap         Theta(n log k)   (lesson 20)
//	deque        Theta(n)         -- and no allocations after the first
//
// The invariant: the deque holds INDICES whose values are strictly decreasing.
// Everything follows from defending that one sentence.

// ---- Deque<T> from example 5, trimmed to the methods this program calls ----
// (clippy -D warnings rejects unused methods; std's VecDeque has all of these
// under the names push_back / pop_front / pop_back / front / back.)
struct Deque<T> {
    buf: Vec<Option<T>>,
    mask: usize,
    head: usize,
    count: usize,
}

impl<T> Deque<T> {
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        Deque {
            buf: std::iter::repeat_with(|| None).take(n).collect(),
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }

    fn len(&self) -> usize {
        self.count
    }

    fn grow(&mut self) {
        let doubled = self.buf.len() * 2;
        let mut old = std::mem::take(&mut self.buf);
        let mut next = Vec::with_capacity(doubled);
        next.extend(old.drain(self.head..)); // the piece before the wrap
        next.append(&mut old); // the piece after it
        next.resize_with(doubled, || None);
        self.buf = next;
        self.mask = doubled - 1;
        self.head = 0;
    }

    fn push_back(&mut self, v: T) {
        if self.count == self.buf.len() {
            self.grow();
        }
        let i = (self.head + self.count) & self.mask;
        self.buf[i] = Some(v);
        self.count += 1;
    }

    fn pop_front(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head].take();
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        v
    }

    fn pop_back(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        let i = (self.head + self.count - 1) & self.mask;
        self.count -= 1;
        self.buf[i].take()
    }

    /// Indexes logically; None when i is not a live position.
    fn at(&self, i: usize) -> Option<&T> {
        if i >= self.count {
            return None;
        }
        self.buf[(self.head + i) & self.mask].as_ref()
    }

    fn front(&self) -> Option<&T> {
        self.at(0)
    }

    fn back(&self) -> Option<&T> {
        self.at(self.count.checked_sub(1)?)
    }
}

// Go returns a nil slice for k <= 0 or k > len(a); there are no windows, and an
// empty Vec says exactly that (windows(0) would panic, so k == 0 is checked).
//
// Brute force, written the way Go's is: scan each window with a plain loop.
fn max_sliding_brute(a: &[i64], k: usize) -> Vec<i64> {
    if k == 0 || a.len() < k {
        return Vec::new();
    }
    let mut out = Vec::with_capacity(a.len() - k + 1);
    for i in 0..=a.len() - k {
        let w = &a[i..i + k];
        let mut m = w[0];
        for &x in &w[1..] {
            if x > m {
                m = x;
            }
        }
        out.push(m);
    }
    out
}

// Not in the Go program: the same Theta(n*k) algorithm, spelled the way an
// iterator-minded Rust programmer would. Same work, different machine code --
// the benchmark below shows how much that matters.
fn max_sliding_brute_iter(a: &[i64], k: usize) -> Vec<i64> {
    if k == 0 || a.len() < k {
        return Vec::new();
    }
    a.windows(k)
        .map(|w| *w.iter().max().expect("a window has k >= 1 elements"))
        .collect()
}

fn max_sliding(a: &[i64], k: usize) -> Vec<i64> {
    if k == 0 || a.len() < k {
        return Vec::new();
    }
    let mut out = Vec::with_capacity(a.len() - k + 1);
    let mut d: Deque<usize> = Deque::new(k + 1); // holds INDICES, values strictly decreasing

    for (i, &v) in a.iter().enumerate() {
        // 1. EXPIRE from the front: the index at the front has left the window.
        //    (Go writes `f <= i-k`; with usize indexes i-k would underflow for
        //    i < k, so the same test is `f + k <= i`.)
        if d.front().is_some_and(|&f| f + k <= i) {
            d.pop_front();
        }
        // 2. MAINTAIN from the back: anything smaller than v can never be the
        //    maximum of any future window, because v is both larger and newer.
        while d.back().is_some_and(|&b| a[b] <= v) {
            d.pop_back();
        }
        d.push_back(i);
        // 3. The front is now the maximum of the current window, by induction.
        if i + 1 >= k
            && let Some(&f) = d.front()
        {
            out.push(a[f]);
        }
    }
    out
}

/// What the counted run saw: the aggregate argument, made concrete.
struct Counts {
    pushes: usize,
    pops: usize,
    ties: usize, // times the back of the deque held a value EQUAL to the new one
    peak: usize, // longest the deque ever got
    out: Vec<i64>,
}

// counted is the same algorithm with the pushes and pops counted. Go's version
// returns only (pushes, pops); the tie counter, the peak length, the output and
// the `keep_ties` switch are additions that back the claims made about ties below.
// keep_ties = false is exactly max_sliding (break on `a[b] > v`); true breaks
// on `a[b] >= v`, i.e. it keeps an equal element instead of evicting it.
fn max_sliding_counted(a: &[i64], k: usize, keep_ties: bool) -> Counts {
    let mut c = Counts {
        pushes: 0,
        pops: 0,
        ties: 0,
        peak: 0,
        out: Vec::new(),
    };
    let mut d: Deque<usize> = Deque::new(k + 1);
    for (i, &v) in a.iter().enumerate() {
        if d.front().is_some_and(|&f| f + k <= i) {
            d.pop_front();
            c.pops += 1;
        }
        while let Some(&b) = d.back() {
            let tie = a[b] == v;
            c.ties += usize::from(tie);
            if a[b] > v || (tie && keep_ties) {
                break;
            }
            d.pop_back();
            c.pops += 1;
        }
        d.push_back(i);
        c.pushes += 1;
        c.peak = c.peak.max(d.len());
        if i + 1 >= k
            && let Some(&f) = d.front()
        {
            c.out.push(a[f]);
        }
    }
    c
}

/// Go's `%v` of a slice: [1 2 3]
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let a: Vec<i64> = vec![1, 3, -1, -3, 5, 3, 6, 7];
    let k = 3;

    println!("a = {}, k = {k}", show(&a));
    println!();
    println!("the deque holds INDICES whose values strictly decrease. Watch it:");
    println!();
    println!(
        "  {:>3} {:>5}  {:<22} {:<22} window max",
        "i", "a[i]", "deque (indices)", "their values"
    );

    let mut d: Deque<usize> = Deque::new(k + 1);
    for (i, &v) in a.iter().enumerate() {
        if d.front().is_some_and(|&f| f + k <= i) {
            d.pop_front();
        }
        while d.back().is_some_and(|&b| a[b] <= v) {
            d.pop_back();
        }
        d.push_back(i);

        let idx: Vec<usize> = (0..d.len()).filter_map(|j| d.at(j).copied()).collect();
        let vals: Vec<i64> = idx.iter().map(|&j| a[j]).collect();
        let out = match d.front() {
            Some(&f) if i + 1 >= k => a[f].to_string(),
            _ => String::new(),
        };
        println!(
            "  {:>3} {:>5}  {:<22} {:<22} {}",
            i,
            v,
            show(&idx),
            show(&vals),
            out
        );
    }
    println!();
    println!("  two things happen to an index, and only two:");
    println!();
    println!("    it EXPIRES off the front  -- it left the window (i-k)");
    println!("    it is EVICTED off the back -- a larger, newer value arrived,");
    println!("                                  so it can never win again");
    println!();
    println!("  the second rule is the one that needs proof, and the proof is one");
    println!("  sentence: if a[j] <= a[i] and j < i, then every window containing");
    println!("  j either contains i too -- in which case j is not the maximum --");
    println!("  or has already passed. Either way j is dead. Delete it.");

    println!();
    println!("Theta(n) by the aggregate method, exactly like lesson 08's");
    println!("monotonic stack:");
    println!();
    println!(
        "  {:>10} {:>8} {:>10} {:>10} {:>14}",
        "n", "k", "pushes", "pops", "ops/element"
    );
    let mut rng = Rng::new(1);
    let mut large_range_ties = 0;
    for n in [1000usize, 10_000, 100_000, 1_000_000] {
        let arr: Vec<i64> = (0..n).map(|_| rng.below(1_000_000) as i64).collect();
        let c = max_sliding_counted(&arr, 100, false);
        large_range_ties += c.ties;
        println!(
            "  {:>10} {:>8} {:>10} {:>10} {:>14.2}",
            n,
            100,
            c.pushes,
            c.pops,
            (c.pushes + c.pops) as f64 / n as f64
        );
    }
    println!();
    println!("  the inner `while` loop looks like it makes this quadratic. It does");
    println!("  not: each index is pushed exactly once and popped at most once, so");
    println!("  the total work over the whole run is at most 2n regardless of what");
    println!("  any single step does. Never read a nested loop as O(n^2) without");
    println!("  asking what it is a loop OVER.");
    println!();
    println!("  note ops/element is independent of k -- which is the whole point.");

    println!();
    println!("proved against brute force before it is benchmarked, because a fast");
    println!("wrong answer is worth nothing:");
    println!();
    let mut fails = 0;
    let mut cases = 0;
    let mut small_range_ties = 0;
    let (mut keep_fails, mut peak_strict, mut peak_keep) = (0, 0, 0);
    for _ in 0..2000 {
        let n = 1 + rng.below(30) as usize;
        // small range -> lots of ties
        let arr: Vec<i64> = (0..n).map(|_| rng.below(21) as i64 - 10).collect();
        for kk in 1..=n {
            cases += 1;
            let brute = max_sliding_brute(&arr, kk);
            assert_eq!(
                brute,
                max_sliding_brute_iter(&arr, kk),
                "the two brutes agree"
            );
            if max_sliding(&arr, kk) != brute {
                fails += 1;
            }
            let strict = max_sliding_counted(&arr, kk, false);
            let keep = max_sliding_counted(&arr, kk, true);
            small_range_ties += strict.ties;
            peak_strict += strict.peak;
            peak_keep += keep.peak;
            if keep.out != brute {
                keep_fails += 1;
            }
        }
    }
    println!("  {cases} random (array, k) pairs, ties and negatives included: {fails} mismatches");
    println!();
    println!("  the small value range matters. Break on `a[b] >= v` instead of");
    println!("  `a[b] > v` -- keep an equal element instead of evicting it -- and the");
    println!("  answers are still CORRECT (equal elements are interchangeable:");
    println!("  {keep_fails} mismatches in the same pairs), but the deque gets LONGER, because");
    println!("  it is no longer strictly decreasing: the peak lengths, summed over the");
    println!(
        "  pairs, come to {peak_strict} when ties are evicted and {peak_keep} when they are kept."
    );
    println!("  Random values from a large range almost never tie, so a test that");
    println!("  uses them exercises neither branch. Force the collisions. Counted: the");
    println!(
        "  tie branch ran {small_range_ties} times in the pairs above and {large_range_ties} times in the"
    );
    println!("  four large-range runs of the table before them.");
    println!();
    println!("and now the benchmark, on the input that actually distinguishes");
    println!("them -- lesson 08's warning applies here too:");
    println!();
    println!(
        "  {:>8} {:>10} {:>14} {:>14} {:>14} {:>9} {:>9}",
        "n", "k", "brute (ns)", "brute iter (ns)", "deque (ns)", "speedup", "vs iter"
    );
    for (n, k) in [
        (100_000usize, 4usize),
        (100_000, 64),
        (100_000, 1000),
        (100_000, 10_000),
    ] {
        let arr: Vec<i64> = (0..n).map(|_| rng.below(1_000_000) as i64).collect();
        let tb = ns_per_op(|| {
            black_box(max_sliding_brute(black_box(&arr), black_box(k)));
        });
        let ti = ns_per_op(|| {
            black_box(max_sliding_brute_iter(black_box(&arr), black_box(k)));
        });
        let td = ns_per_op(|| {
            black_box(max_sliding(black_box(&arr), black_box(k)));
        });
        println!(
            "  {:>8} {:>10} {:>14.0} {:>14.0} {:>14.0} {:>8.2}x {:>8.2}x",
            n,
            k,
            tb,
            ti,
            td,
            tb / td,
            ti / td
        );
    }
    println!();
    // the header comment claims "no allocations after the first"; count them
    for n in [1_000usize, 100_000] {
        let arr: Vec<i64> = (0..n).map(|_| rng.below(1_000_000) as i64).collect();
        let brute = allocs_of(|| max_sliding_brute(&arr, 64));
        let deque = allocs_of(|| max_sliding(&arr, 64));
        println!("  allocation events per call, n = {n:>6}, k = 64: brute {brute}, deque {deque}");
    }
    println!();
    println!("  the deque's time is FLAT in k: it moved between roughly 0.7 and 2.3 ms");
    println!("  across my runs with how loaded the machine was, with no trend in k,");
    println!("  while brute force is linear in it. The allocation lines above are the");
    println!("  header's \"no allocations after the first\": two events per call at any");
    println!("  n -- the output Vec and the deque's buffer (k+1 slots, rounded up to a");
    println!("  power of two) -- and neither grows.");
    println!();
    println!("  and at k=4 brute force WINS, by MORE than it did in Go. Across my runs");
    println!("  the plain loop beat the deque by 3.3x to 6x (Go: 1.9x) and the iterator");
    println!("  spelling by 5x to 10x. That is not noise. Theta(n) beats Theta(nk) only");
    println!("  once k is large enough to pay for the constant -- and how large depends");
    println!("  on how fast the brute scan is.");
    println!();
    println!("  the two brute forces do the same Theta(n*k) work, yet at k=10,000 the");
    println!("  iterator spelling took 3.4x to 4.4x as long as the loop. I read the");
    println!("  assembly of both: the loop is auto-vectorised (cmgt.2d, two i64 lanes");
    println!("  per compare) and the iterator version is scalar (csel). That is the one");
    println!("  mechanism I checked; why LLVM vectorises one and not the other, I did");
    println!("  not look into.");
    println!();
    println!("  so the crossover is not one number:");
    println!();
    println!("    against the loop            between k=64 (brute wins, 0.6x to 0.9x)");
    println!("                                and k=1000 (deque 7.6x to 19x faster)");
    println!("    against windows().max()     between k=4 and k=64 (deque 2.8x to 3.5x");
    println!("                                faster at 64), where Go's was");
    println!();
    println!("  If your window is small and FIXED, write the loop. The deque earns its");
    println!("  complexity when k is large, or when k is a parameter you do not");
    println!("  control -- because then the crossover is not yours to reason about,");
    println!("  and in Rust it moves with how you spell the loop you are comparing to.");
    println!();
    println!("  the same skeleton -- expire from one end, evict from the other --");
    println!("  solves minimum-in-window, 'first negative in every window',");
    println!("  'longest subarray with max-min <= limit' (two deques), and the");
    println!("  constrained-window DP in lesson 30. Learn the invariant, not the");
    println!("  problem.");
}
```

**Rust sample output:**

```
a = [1 3 -1 -3 5 3 6 7], k = 3

the deque holds INDICES whose values strictly decrease. Watch it:

    i  a[i]  deque (indices)        their values           window max
    0     1  [0]                    [1]                    
    1     3  [1]                    [3]                    
    2    -1  [1 2]                  [3 -1]                 3
    3    -3  [1 2 3]                [3 -1 -3]              3
    4     5  [4]                    [5]                    5
    5     3  [4 5]                  [5 3]                  5
    6     6  [6]                    [6]                    6
    7     7  [7]                    [7]                    7

  two things happen to an index, and only two:

    it EXPIRES off the front  -- it left the window (i-k)
    it is EVICTED off the back -- a larger, newer value arrived,
                                  so it can never win again

  the second rule is the one that needs proof, and the proof is one
  sentence: if a[j] <= a[i] and j < i, then every window containing
  j either contains i too -- in which case j is not the maximum --
  or has already passed. Either way j is dead. Delete it.

Theta(n) by the aggregate method, exactly like lesson 08's
monotonic stack:

           n        k     pushes       pops    ops/element
        1000      100       1000        997           2.00
       10000      100      10000       9989           2.00
      100000      100     100000      99992           2.00
     1000000      100    1000000     999994           2.00

  the inner `while` loop looks like it makes this quadratic. It does
  not: each index is pushed exactly once and popped at most once, so
  the total work over the whole run is at most 2n regardless of what
  any single step does. Never read a nested loop as O(n^2) without
  asking what it is a loop OVER.

  note ops/element is independent of k -- which is the whole point.

proved against brute force before it is benchmarked, because a fast
wrong answer is worth nothing:

  31488 random (array, k) pairs, ties and negatives included: 0 mismatches

  the small value range matters. Break on `a[b] >= v` instead of
  `a[b] > v` -- keep an equal element instead of evicting it -- and the
  answers are still CORRECT (equal elements are interchangeable:
  0 mismatches in the same pairs), but the deque gets LONGER, because
  it is no longer strictly decreasing: the peak lengths, summed over the
  pairs, come to 123771 when ties are evicted and 140639 when they are kept.
  Random values from a large range almost never tie, so a test that
  uses them exercises neither branch. Force the collisions. Counted: the
  tie branch ran 62807 times in the pairs above and 5 times in the
  four large-range runs of the table before them.

and now the benchmark, on the input that actually distinguishes
them -- lesson 08's warning applies here too:

         n          k     brute (ns) brute iter (ns)     deque (ns)   speedup   vs iter
    100000          4         183400         124899         799828     0.23x     0.16x
    100000         64         784953        3553794        1034407     0.76x     3.44x
    100000       1000       14501208       55715260         970682    14.94x    57.40x
    100000      10000      129248500      563103500        1329417    97.22x   423.57x

  allocation events per call, n =   1000, k = 64: brute 1, deque 2
  allocation events per call, n = 100000, k = 64: brute 1, deque 2

  the deque's time is FLAT in k: it moved between roughly 0.7 and 2.3 ms
  across my runs with how loaded the machine was, with no trend in k,
  while brute force is linear in it. The allocation lines above are the
  header's "no allocations after the first": two events per call at any
  n -- the output Vec and the deque's buffer (k+1 slots, rounded up to a
  power of two) -- and neither grows.

  and at k=4 brute force WINS, by MORE than it did in Go. Across my runs
  the plain loop beat the deque by 3.3x to 6x (Go: 1.9x) and the iterator
  spelling by 5x to 10x. That is not noise. Theta(n) beats Theta(nk) only
  once k is large enough to pay for the constant -- and how large depends
  on how fast the brute scan is.

  the two brute forces do the same Theta(n*k) work, yet at k=10,000 the
  iterator spelling took 3.4x to 4.4x as long as the loop. I read the
  assembly of both: the loop is auto-vectorised (cmgt.2d, two i64 lanes
  per compare) and the iterator version is scalar (csel). That is the one
  mechanism I checked; why LLVM vectorises one and not the other, I did
  not look into.

  so the crossover is not one number:

    against the loop            between k=64 (brute wins, 0.6x to 0.9x)
                                and k=1000 (deque 7.6x to 19x faster)
    against windows().max()     between k=4 and k=64 (deque 2.8x to 3.5x
                                faster at 64), where Go's was

  If your window is small and FIXED, write the loop. The deque earns its
  complexity when k is large, or when k is a parameter you do not
  control -- because then the crossover is not yours to reason about,
  and in Rust it moves with how you spell the loop you are comparing to.

  the same skeleton -- expire from one end, evict from the other --
  solves minimum-in-window, 'first negative in every window',
  'longest subarray with max-min <= limit' (two deques), and the
  constrained-window DP in lesson 30. Learn the invariant, not the
  problem.
```

> **Go vs Rust:** The trace and everything printed before the first random array are identical to Go's; `f + k <= i` replaces `f <= i-k` because a `usize` subtraction would underflow, and the random arrays differ (kit splitmix64, not `math/rand`: pops 997 / 9989 / 99992 / 999994 against Go's 995 / 9996 / 99993 / 999998, 31,488 random pairs against 30,796). Two Go claims do not carry over: the `a[b] >= v` variant makes the deque *longer*, not shorter (summed peak length 140,639 against 123,771, measured here), and the k=4 and crossover conclusions depend on how the brute force is spelled, since Rust's explicit loop auto-vectorises and `windows().max()` does not (about 4x slower at k=10,000), which moves the crossover against the loop to k=64-1000 (against `windows().max()` it stays near Go's 4-64).

**Complexity:** Θ(n), **exactly 2.00 operations per element** regardless of k · the deque's runtime is
*flat* in k (0.85 ms at k=4, 0.91 ms at k=10,000) while brute force is linear in it — but **at k=4 brute
force wins** — the deque runs at 0.52× its speed, and the crossover is between 4 and 64

---

## 11. Six queues, three workloads, three different winners

`🟡 medium` · *and the harness turned out to be part of the experiment*

Build them all, then let the measurement pick. The three workloads are chosen because they
**disagree** — a benchmark that only runs the steady state will tell you the naive queue is best, and
it will be lying by omission.

**Steps:**

1. Put all six behind one `Queue` interface so the call overhead is identical.
2. Run steady state, burst-then-drain, and memory-held-after-drain.
3. Then check whether the harness itself changed the answer.

```go
package main

import (
	"container/list"
	"fmt"
	"runtime"
	"testing"
)

// Six queues, three workloads, one table. Lesson 07 ended this way and the
// habit is the point: build them all, then let the measurement pick.
//
// The three workloads are chosen because they DISAGREE. A benchmark that only
// runs the steady state will tell you the naive slice queue is best, and it
// will be lying by omission.

type Queue interface {
	Push(int)
	Pop() (int, bool)
}

// 1. naive: q = q[1:]
type Naive struct{ q []int }

func (n *Naive) Push(v int) { n.q = append(n.q, v) }
func (n *Naive) Pop() (int, bool) {
	if len(n.q) == 0 {
		return 0, false
	}
	v := n.q[0]
	n.q = n.q[1:]
	return v, true
}

// 2. head index, compacting once half the buffer is dead
type Compact struct {
	buf  []int
	head int
}

func (c *Compact) Push(v int) { c.buf = append(c.buf, v) }
func (c *Compact) Pop() (int, bool) {
	if c.head == len(c.buf) {
		return 0, false
	}
	v := c.buf[c.head]
	c.head++
	if c.head > len(c.buf)/2 {
		live := len(c.buf) - c.head
		copy(c.buf, c.buf[c.head:])
		c.buf = c.buf[:live]
		c.head = 0
	}
	return v, true
}

// 3. fixed ring, power of two. Push REFUSES when full.
type Ring struct {
	buf         []int
	mask        int
	head, count int
	refused     int
}

func NewRing(capacity int) *Ring {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Ring{buf: make([]int, n), mask: n - 1}
}
func (r *Ring) Push(v int) {
	if r.count == len(r.buf) {
		r.refused++
		return
	}
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
}
func (r *Ring) Pop() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	v := r.buf[r.head]
	r.head = (r.head + 1) & r.mask
	r.count--
	return v, true
}

// 4. growable ring
type GrowRing struct {
	buf         []int
	mask        int
	head, count int
}

func NewGrowRing(capacity int) *GrowRing {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &GrowRing{buf: make([]int, n), mask: n - 1}
}
func (g *GrowRing) Push(v int) {
	if g.count == len(g.buf) {
		next := make([]int, len(g.buf)*2)
		k := copy(next, g.buf[g.head:])
		copy(next[k:], g.buf[:g.head])
		g.buf, g.mask, g.head = next, len(next)-1, 0
	}
	g.buf[(g.head+g.count)&g.mask] = v
	g.count++
}
func (g *GrowRing) Pop() (int, bool) {
	if g.count == 0 {
		return 0, false
	}
	v := g.buf[g.head]
	g.head = (g.head + 1) & g.mask
	g.count--
	return v, true
}

// 5. container/list
type List struct{ l *list.List }

func NewList() *List       { return &List{l: list.New()} }
func (l *List) Push(v int) { l.l.PushBack(v) }
func (l *List) Pop() (int, bool) {
	e := l.l.Front()
	if e == nil {
		return 0, false
	}
	return l.l.Remove(e).(int), true
}

// 6. buffered channel, non-blocking on both ends so it fits the interface
type Chan struct{ c chan int }

func NewChan(capacity int) *Chan { return &Chan{c: make(chan int, capacity)} }
func (c *Chan) Push(v int) {
	select {
	case c.c <- v:
	default:
	}
}
func (c *Chan) Pop() (int, bool) {
	select {
	case v := <-c.c:
		return v, true
	default:
		return 0, false
	}
}

type impl struct {
	name    string
	bounded bool
	new     func(capacity int) Queue
}

var impls = []impl{
	{"naive q = q[1:]", false, func(c int) Queue { return &Naive{q: make([]int, 0, c)} }},
	{"head index + compact", false, func(c int) Queue { return &Compact{buf: make([]int, 0, c)} }},
	{"ring (fixed)", true, func(c int) Queue { return NewRing(c) }},
	{"ring (growable)", false, func(c int) Queue { return NewGrowRing(c) }},
	{"container/list", false, func(int) Queue { return NewList() }},
	{"buffered channel", true, func(c int) Queue { return NewChan(c) }},
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

func main() {
	const steady = 500_000
	const burst = 2_000_000

	fmt.Println("workload A -- STEADY STATE: 1000 live, one push and one pop each,")
	fmt.Println("500,000 times. A work queue keeping up with its producer.")
	fmt.Println()
	fmt.Printf("  %-24s %14s\n", "implementation", "ns/op")
	steadyNS := map[string]float64{}
	for _, im := range impls {
		t := nsPerOp(func() {
			q := im.new(4096)
			for i := 0; i < 1000; i++ {
				q.Push(i)
			}
			for i := 0; i < steady; i++ {
				q.Push(i)
				v, _ := q.Pop()
				sinkI = v
			}
		})
		steadyNS[im.name] = t / steady
		fmt.Printf("  %-24s %14.2f\n", im.name, t/steady)
	}
	fmt.Println()
	fmt.Println("  all six behind the same `Queue` interface, so the call overhead is")
	fmt.Println("  identical and the differences are the structures themselves.")

	fmt.Println()
	fmt.Println("workload B -- BURST THEN DRAIN: push 2,000,000, then pop them all,")
	fmt.Println("starting from capacity 1024. A BFS frontier, a batch import, a")
	fmt.Println("backlog after an outage.")
	fmt.Println()
	fmt.Printf("  %-24s %14s %12s %14s\n", "implementation", "ns/op", "peak MB", "pushes lost")
	for _, im := range impls {
		base := heapMB()
		q := im.new(1024)
		for i := 0; i < burst; i++ {
			q.Push(i)
		}
		peak := heapMB() - base
		got := 0
		for {
			if _, ok := q.Pop(); !ok {
				break
			}
			got++
		}
		lost := burst - got
		t := nsPerOp(func() {
			q2 := im.new(1024)
			for i := 0; i < burst; i++ {
				q2.Push(i)
			}
			for i := 0; i < burst; i++ {
				v, _ := q2.Pop()
				sinkI = v
			}
		})
		fmt.Printf("  %-24s %14.2f %12.1f %14d\n", im.name, t/(2*burst), peak, lost)
	}
	fmt.Println()
	fmt.Println("  read the last column first. The two BOUNDED implementations threw")
	fmt.Println("  away 99.9% of the input, silently, and were fast about it. That is")
	fmt.Println("  not a benchmark loss -- it is the policy working exactly as")
	fmt.Println("  designed (example 9), on a workload where the policy is wrong.")
	fmt.Println()
	fmt.Println("  a bounded queue is not a slower unbounded queue. It is a different")
	fmt.Println("  contract, and the ns/op column cannot see the difference.")

	fmt.Println()
	fmt.Println("workload C -- WHAT IS STILL HELD after draining to empty:")
	fmt.Println()
	fmt.Printf("  %-24s %18s\n", "implementation", "MB held at len 0")
	for _, im := range impls {
		if im.bounded {
			continue
		}
		base := heapMB()
		q := im.new(1024)
		for i := 0; i < burst; i++ {
			q.Push(i)
		}
		for {
			if _, ok := q.Pop(); !ok {
				break
			}
		}
		held := heapMB() - base
		runtime.KeepAlive(q)
		fmt.Printf("  %-24s %18.1f\n", im.name, held)
	}
	fmt.Println()
	fmt.Println("  this is example 1's case B, and it reverses the ranking again:")
	fmt.Println("  container/list is the ONLY structure here that gives its memory")
	fmt.Println("  back on drain, because every node it dropped is genuinely garbage.")
	fmt.Println("  The three array-backed queues are all still holding their peak.")
	fmt.Println()
	fmt.Println("  the slowest, most allocation-hungry implementation wins the")
	fmt.Println("  memory-return column outright. Three workloads, three different")
	fmt.Println("  winners -- which is the entire reason to run all three.")

	fmt.Println()
	fmt.Println("the harness is part of the experiment, and this table proves it:")
	fmt.Println()
	naiveInline := nsPerOp(func() {
		q := make([]int, 0, 4096)
		for i := 0; i < 1000; i++ {
			q = append(q, i)
		}
		for i := 0; i < steady; i++ {
			q = append(q, i)
			sinkI = q[0]
			q = q[1:]
		}
	})
	ringInline := nsPerOp(func() {
		buf, mask := make([]int, 4096), 4095
		head, count := 0, 0
		for i := 0; i < 1000; i++ {
			buf[(head+count)&mask] = i
			count++
		}
		for i := 0; i < steady; i++ {
			buf[(head+count)&mask] = i
			count++
			sinkI = buf[head]
			head = (head + 1) & mask
			count--
		}
	})
	allocInline := testing.AllocsPerRun(3, func() {
		q := make([]int, 0, 4096)
		for i := 0; i < 1000; i++ {
			q = append(q, i)
		}
		for i := 0; i < 200_000; i++ {
			q = append(q, i)
			sinkI = q[0]
			q = q[1:]
		}
	})
	allocIface := testing.AllocsPerRun(3, func() {
		var q Queue = &Naive{q: make([]int, 0, 4096)}
		for i := 0; i < 1000; i++ {
			q.Push(i)
		}
		for i := 0; i < 200_000; i++ {
			q.Push(i)
			v, _ := q.Pop()
			sinkI = v
		}
	})
	fmt.Printf("  %-28s %12s %12s %10s\n", "", "inline", "interface", "added")
	fmt.Printf("  %-28s %10.2f ns %10.2f ns %8.2f ns\n", "naive q = q[1:]",
		naiveInline/steady, steadyNS["naive q = q[1:]"],
		steadyNS["naive q = q[1:]"]-naiveInline/steady)
	fmt.Printf("  %-28s %10.2f ns %10.2f ns %8.2f ns\n", "ring (fixed)",
		ringInline/steady, steadyNS["ring (fixed)"],
		steadyNS["ring (fixed)"]-ringInline/steady)
	fmt.Println()
	fmt.Printf("  allocations, naive, 200,000 ops: inline %.0f, interface %.0f\n",
		allocInline, allocIface)
	fmt.Println()
	fmt.Println("  the allocation counts are the SAME, so the gap is not allocation.")
	fmt.Println("  It is the two indirect calls per operation: an interface method")
	fmt.Println("  cannot be inlined, so the slice header moves between the struct")
	fmt.Println("  and registers on every push and every pop instead of staying put.")
	fmt.Println()
	fmt.Println("  that is why examples 1 to 3, which wrote the loops inline, kept")
	fmt.Println("  finding the naive queue fastest and this table does not. Both are")
	fmt.Println("  correct measurements of different programs.")
	fmt.Println()
	fmt.Println("  the practical reading: within THIS table the comparison is fair,")
	fmt.Println("  because all six pay the same tax. Across tables it is not. Never")
	fmt.Println("  quote a ns/op figure without the harness it was measured in.")

	fmt.Println()
	fmt.Println("the verdict, and it is not 'use the fastest one':")
	fmt.Println()
	fmt.Printf("  %-22s %-34s %s\n", "implementation", "use it when", "never when")
	for _, r := range [][3]string{
		{"naive q = q[1:]", "short-lived, scoped, fully drained", "it outlives the function"},
		{"head index + compact", "you want one file, no surprises", "you need worst-case Theta(1)"},
		{"ring (fixed)", "there is a real capacity limit", "bursts must not be refused"},
		{"ring (growable)", "hot path, size unknown", "you needed a memory bound"},
		{"container/list", "you remove by handle too", "it is only a queue"},
		{"buffered channel", "it crosses a goroutine", "it does not"},
	} {
		fmt.Printf("  %-22s %-34s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Printf("  the spread on workload A is %.1fx between best and worst, which is\n",
		steadyNS["container/list"]/steadyNS["ring (fixed)"])
	fmt.Println("  real but small next to the difference between a queue that bounds")
	fmt.Println("  its memory and one that does not. Speed was the least interesting")
	fmt.Println("  axis in this lesson, and measuring it was the only way to find")
	fmt.Println("  that out.")
}
```

**Sample output:**

```
workload A -- STEADY STATE: 1000 live, one push and one pop each,
500,000 times. A work queue keeping up with its producer.

  implementation                    ns/op
  naive q = q[1:]                    7.99
  head index + compact               2.17
  ring (fixed)                       3.55
  ring (growable)                    2.44
  container/list                    21.12
  buffered channel                  19.23

  all six behind the same `Queue` interface, so the call overhead is
  identical and the differences are the structures themselves.

workload B -- BURST THEN DRAIN: push 2,000,000, then pop them all,
starting from capacity 1024. A BFS frontier, a batch import, a
backlog after an outage.

  implementation                    ns/op      peak MB    pushes lost
  naive q = q[1:]                    5.04         16.9              0
  head index + compact               4.20         16.9              0
  ring (fixed)                       1.71          0.0        1998976
  ring (growable)                    3.60         16.0              0
  container/list                    17.80        106.8              0
  buffered channel                   2.93          0.0        1998976

  read the last column first. The two BOUNDED implementations threw
  away 99.9% of the input, silently, and were fast about it. That is
  not a benchmark loss -- it is the policy working exactly as
  designed (example 9), on a workload where the policy is wrong.

  a bounded queue is not a slower unbounded queue. It is a different
  contract, and the ns/op column cannot see the difference.

workload C -- WHAT IS STILL HELD after draining to empty:

  implementation             MB held at len 0
  naive q = q[1:]                        16.9
  head index + compact                   16.9
  ring (growable)                        16.0
  container/list                         -0.0

  this is example 1's case B, and it reverses the ranking again:
  container/list is the ONLY structure here that gives its memory
  back on drain, because every node it dropped is genuinely garbage.
  The three array-backed queues are all still holding their peak.

  the slowest, most allocation-hungry implementation wins the
  memory-return column outright. Three workloads, three different
  winners -- which is the entire reason to run all three.

the harness is part of the experiment, and this table proves it:

                                     inline    interface      added
  naive q = q[1:]                    1.82 ns       7.99 ns     6.17 ns
  ring (fixed)                       0.55 ns       3.55 ns     3.00 ns

  allocations, naive, 200,000 ops: inline 368, interface 369

  the allocation counts are the SAME, so the gap is not allocation.
  It is the two indirect calls per operation: an interface method
  cannot be inlined, so the slice header moves between the struct
  and registers on every push and every pop instead of staying put.

  that is why examples 1 to 3, which wrote the loops inline, kept
  finding the naive queue fastest and this table does not. Both are
  correct measurements of different programs.

  the practical reading: within THIS table the comparison is fair,
  because all six pay the same tax. Across tables it is not. Never
  quote a ns/op figure without the harness it was measured in.

the verdict, and it is not 'use the fastest one':

  implementation         use it when                        never when
  naive q = q[1:]        short-lived, scoped, fully drained it outlives the function
  head index + compact   you want one file, no surprises    you need worst-case Theta(1)
  ring (fixed)           there is a real capacity limit     bursts must not be refused
  ring (growable)        hot path, size unknown             you needed a memory bound
  container/list         you remove by handle too           it is only a queue
  buffered channel       it crosses a goroutine             it does not

  the spread on workload A is 5.9x between best and worst, which is
  real but small next to the difference between a queue that bounds
  its memory and one that does not. Speed was the least interesting
  axis in this lesson, and measuring it was the only way to find
  that out.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{LinkedList, VecDeque};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::sync::mpsc::{Receiver, SyncSender, sync_channel};
use std::time::{Duration, Instant};

// ---- the kit's allocation counter: stand-in for runtime.MemStats / testing.AllocsPerRun ----
// Go reads HeapAlloc after runtime.GC(); Rust has no GC, so the allocator itself
// counts the bytes currently live (exact, deterministic) and the allocation events.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// allocation events while running `f`
fn allocs_of(f: impl FnOnce()) -> usize {
    let a0 = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - a0
}
/// bytes currently live on the heap, in MB (Go's `1<<20` MB)
fn live_mb() -> f64 {
    LIVE.load(Relaxed) as f64 / (1 << 20) as f64
}

// ---- ns/op: stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut run: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            run();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// Six queues, three workloads, one table. Lesson 07 ended this way and the
// habit is the point: build them all, then let the measurement pick.
//
// The three workloads are chosen because they DISAGREE. A benchmark that only
// runs the steady state will tell you the naive queue is best, and it will be
// lying by omission.
//
// THE NAIVE QUEUE IS A DIFFERENT ALGORITHM HERE. Go's is `q = q[1:]`, which
// moves the slice's start and never copies; Rust cannot write that on an owned
// Vec (see example 1). The naive queue Rust lets you write is `Vec::remove(0)`,
// which shifts every other element left on EVERY pop: Theta(n) per dequeue
// instead of Theta(1). Its row is therefore a result about remove(0), not about
// Go's reslicing -- and it is the row most likely to change a conclusion.

trait Queue {
    fn push(&mut self, v: i64);
    fn pop(&mut self) -> Option<i64>;
}

// 1. naive: the owned Vec's own dequeue, remove(0)
struct Naive {
    q: Vec<i64>,
}

impl Queue for Naive {
    fn push(&mut self, v: i64) {
        self.q.push(v);
    }
    fn pop(&mut self) -> Option<i64> {
        if self.q.is_empty() {
            None
        } else {
            Some(self.q.remove(0))
        }
    }
}

// 2. head index, compacting once half the buffer is dead
struct Compact {
    buf: Vec<i64>,
    head: usize,
}

impl Queue for Compact {
    fn push(&mut self, v: i64) {
        self.buf.push(v);
    }
    fn pop(&mut self) -> Option<i64> {
        let v = *self.buf.get(self.head)?;
        self.head += 1;
        if self.head > self.buf.len() / 2 {
            // one memmove; the Vec's length shrinks with it, so there is no
            // separate "reslice to the live length" step as in Go
            self.buf.drain(..self.head);
            self.head = 0;
        }
        Some(v)
    }
}

// 3. fixed ring, power of two. Push REFUSES when full.
struct Ring {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
}

impl Ring {
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        Ring {
            buf: vec![0; n],
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }
}

impl Queue for Ring {
    // A full ring refuses and drops the value. Example 3's ring returned
    // Err(v) to hand it back; the interface here has no return value (Go's
    // doesn't either), so the refusal is silent and the harness counts the loss.
    fn push(&mut self, v: i64) {
        if self.count == self.buf.len() {
            return;
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
    }
    fn pop(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
    }
}

// 4. growable ring
struct GrowRing {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
}

impl GrowRing {
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        GrowRing {
            buf: vec![0; n],
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }
}

impl Queue for GrowRing {
    fn push(&mut self, v: i64) {
        if self.count == self.buf.len() {
            // two pieces, as in example 4: the run from head to the end of the
            // array, then the run that wrapped around to the start
            let mut next = vec![0; self.buf.len() * 2];
            let k = self.buf.len() - self.head;
            next[..k].copy_from_slice(&self.buf[self.head..]);
            next[k..k + self.head].copy_from_slice(&self.buf[..self.head]);
            self.mask = next.len() - 1;
            self.buf = next;
            self.head = 0;
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
    }
    fn pop(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
    }
}

// 5. std's doubly linked list (Go: container/list)
impl Queue for LinkedList<i64> {
    fn push(&mut self, v: i64) {
        self.push_back(v);
    }
    fn pop(&mut self) -> Option<i64> {
        self.pop_front()
    }
}

// 6. bounded channel, non-blocking on both ends so it fits the interface
//    (Go: a buffered chan with select/default). Nothing here crosses a thread;
//    that is the point of the "never when" column. The queue owns both ends.
struct Chan {
    tx: SyncSender<i64>,
    rx: Receiver<i64>,
}

impl Chan {
    fn new(capacity: usize) -> Self {
        let (tx, rx) = sync_channel(capacity);
        Chan { tx, rx }
    }
}

impl Queue for Chan {
    fn push(&mut self, v: i64) {
        // try_send never blocks; Err(Full(v)) hands the value back and we drop
        // it -- example 9's drop-newest policy. Disconnected cannot happen: this
        // struct holds the receiver.
        let _ = self.tx.try_send(v);
    }
    fn pop(&mut self) -> Option<i64> {
        self.rx.try_recv().ok()
    }
}

// 7. NOT in the Go program: std's growable ring, for reference
impl Queue for VecDeque<i64> {
    fn push(&mut self, v: i64) {
        self.push_back(v);
    }
    fn pop(&mut self) -> Option<i64> {
        self.pop_front()
    }
}

struct Impl {
    name: &'static str,
    bounded: bool,
    new: fn(usize) -> Box<dyn Queue>,
}

static IMPLS: [Impl; 7] = [
    Impl {
        name: "naive Vec::remove(0)",
        bounded: false,
        new: |c| {
            Box::new(Naive {
                q: Vec::with_capacity(c),
            })
        },
    },
    Impl {
        name: "head index + compact",
        bounded: false,
        new: |c| {
            Box::new(Compact {
                buf: Vec::with_capacity(c),
                head: 0,
            })
        },
    },
    Impl {
        name: "ring (fixed)",
        bounded: true,
        new: |c| Box::new(Ring::new(c)),
    },
    Impl {
        name: "ring (growable)",
        bounded: false,
        new: |c| Box::new(GrowRing::new(c)),
    },
    Impl {
        name: "LinkedList",
        bounded: false,
        new: |_| Box::new(LinkedList::new()),
    },
    Impl {
        name: "sync_channel",
        bounded: true,
        new: |c| Box::new(Chan::new(c)),
    },
    Impl {
        name: "VecDeque (std, not in Go)",
        bounded: false,
        new: |c| Box::new(VecDeque::with_capacity(c)),
    },
];
// The first six rows are the Go program's six; the seventh is a reference row.
const SIX: usize = 6;

const STEADY: i64 = 500_000;
// Go bursts 2,000,000. That is 16 MB of i64 and nothing for five of the queues,
// but remove(0) shifts the whole backlog on every pop: 2,000,000 pops of a
// 2,000,000-element queue is ~16 TB of memmove. Scaled to 200,000 (10x down) so
// the naive row finishes in seconds; the capacity-1024 bound stays.
const BURST: i64 = 200_000;

/// The steady-state loop, written ONCE. `Q: ?Sized` lets the same body run
/// behind a vtable (`Q = dyn Queue`, Go's interface) or monomorphised
/// (`Q = Ring`, a direct, inlinable call).
fn run_steady<Q: Queue + ?Sized>(q: &mut Q) {
    for i in 0..1000 {
        q.push(i);
    }
    for i in 0..STEADY {
        q.push(i);
        black_box(q.pop());
    }
}

fn main() {
    println!("two things differ from the Go program, so read them first:");
    println!();
    println!("  1. Go's naive queue is `q = q[1:]` (move the start, never copy). Rust");
    println!("     cannot write that on an owned Vec, so row 1 is `Vec::remove(0)`,");
    println!("     which shifts the whole backlog on EVERY pop. That is a different");
    println!("     algorithm, so its row is not comparable with Go's naive row. The");
    println!("     other five queues are the same algorithms as Go's.");
    println!("  2. The burst is {BURST}, not Go's 2,000,000: remove(0) is Theta(n^2) over");
    println!("     a burst, and 2,000,000 would be ~16 TB of memmove. Every queue gets");
    println!("     the same scaled burst. The capacity of 1024 is unchanged.");
    println!();
    println!("workload A -- STEADY STATE: 1000 live, one push and one pop each,");
    println!("500,000 times. A work queue keeping up with its producer.");
    println!();
    println!("  {:<26} {:>14}", "implementation", "ns/op");
    let mut steady_ns = Vec::new();
    for im in &IMPLS {
        let t = ns_per_op(|| {
            let mut q = (im.new)(4096);
            run_steady(&mut *q);
        });
        steady_ns.push(t / STEADY as f64);
        println!("  {:<26} {:>14.2}", im.name, t / STEADY as f64);
    }
    println!();
    println!("  all behind the same `dyn Queue` vtable, so the call overhead is");
    println!("  identical and the differences are the structures themselves.");
    println!();
    println!("  the first six rows are Go's six; VecDeque is std's growable ring, added");
    println!("  for reference. Across my runs (a few of them while the machine was");
    println!("  loaded by other processes) head index, ring (growable) and VecDeque");
    println!("  sat within about 1 ns of each other on the quiet runs and swapped");
    println!("  places, so this workload does not pick one of them. Against the best of");
    println!("  them: ring (fixed) 1.3x to 2.3x, sync_channel 2.6x to 3.9x, LinkedList");
    println!("  6.8x to 12x, and Vec::remove(0) 19x to 38x, because every pop shifts");
    println!("  the 1000 live elements.");

    println!();
    println!("workload B -- BURST THEN DRAIN: push {BURST}, then pop them all,");
    println!("starting from capacity 1024. A BFS frontier, a batch import, a");
    println!("backlog after an outage.");
    println!();
    println!(
        "  {:<26} {:>14} {:>12} {:>14}",
        "implementation", "ns/op", "peak MB", "pushes lost"
    );
    let mut burst_ns = Vec::new();
    let mut lost_of = Vec::new(); // pushes lost, per queue
    for im in &IMPLS {
        let base = live_mb();
        let mut q = (im.new)(1024);
        for i in 0..BURST {
            q.push(i);
        }
        let peak = live_mb() - base;
        // drain, checking FIFO order on the way: a bounded queue keeps the
        // OLDEST values (it drops the newest), so even it must pop 0, 1, 2...
        let mut got = 0;
        while let Some(v) = q.pop() {
            assert_eq!(v, got, "{} broke FIFO order", im.name);
            got += 1;
        }
        let lost = BURST - got;
        drop(q);
        let t = ns_per_op(|| {
            let mut q2 = (im.new)(1024);
            for i in 0..BURST {
                q2.push(i);
            }
            for _ in 0..BURST {
                black_box(q2.pop());
            }
        });
        burst_ns.push(t / (2 * BURST) as f64);
        lost_of.push(lost);
        println!(
            "  {:<26} {:>14.2} {:>12.1} {:>14}",
            im.name,
            t / (2 * BURST) as f64,
            peak,
            lost
        );
    }
    println!();
    assert_eq!(
        lost_of[2], lost_of[5],
        "both bounded queues keep exactly 1024"
    );
    println!(
        "  read the last column first. The two BOUNDED implementations threw away {:.1}%",
        100.0 * lost_of[2] as f64 / BURST as f64
    );
    println!(
        "  of the input ({} of {BURST}), silently, and were fast about it. That is",
        lost_of[2]
    );
    println!("  not a benchmark loss -- it is the policy working exactly as");
    println!("  designed (example 9), on a workload where the policy is wrong.");
    println!();
    println!("  a bounded queue is not a slower unbounded queue. It is a different");
    println!("  contract, and the ns/op column cannot see the difference.");
    println!();
    println!(
        "  the naive row is the other one to read: {:.0}x the head index's cost per op,",
        burst_ns[0] / burst_ns[1]
    );
    println!("  and it grows with the burst, because every pop shifts what is left.");
    println!("  Go's naive queue stayed in the same few-ns range as the array queues");
    println!("  on this workload (5.04 ns/op in the Go run); that is the reslice, which");
    println!("  Rust cannot write on an owned Vec. Among the queues that lost nothing,");
    println!("  head index was fastest in all my runs but one, ahead of VecDeque by up");
    println!("  to roughly 35% (in the exception VecDeque led by 2%), and LinkedList");
    println!("  took 9x to 15x the head index.");

    println!();
    println!("workload C -- WHAT IS STILL HELD after draining to empty:");
    println!();
    println!("  {:<26} {:>18}", "implementation", "MB held at len 0");
    for im in IMPLS.iter().filter(|im| !im.bounded) {
        let base = live_mb();
        let mut q = (im.new)(1024);
        for i in 0..BURST {
            q.push(i);
        }
        while q.pop().is_some() {}
        let held = live_mb() - base;
        black_box(&q); // the queue is empty but still alive while we read the counter
        println!("  {:<26} {:>18.1}", im.name, held);
    }
    println!();
    println!("  this is example 1's case B, and it reverses the ranking again:");
    println!("  LinkedList is the ONLY structure here that gives its memory back on");
    println!("  drain, because it frees every node as it pops it -- there is no GC to");
    println!("  wait for. The array-backed queues are all still holding their peak,");
    println!("  2.0 MB: 262,144 slots of 8 bytes, the doubling from 1024 that had");
    println!("  room for 200,000 elements. VecDeque is no different.");
    println!();
    println!("  the slowest of the Theta(1) queues wins the memory-return column");
    println!("  outright (7x to 12x the best on workload A, 9x to 15x on B). The");
    println!("  ranking differs per workload, which is the entire reason to run all");
    println!("  three. (Rust's naive queue holds the same 2.0 MB as the other arrays:");
    println!("  remove(0) never abandons a prefix as Go's reslice does, so it has no");
    println!("  leak to show in a table like this -- see example 1.)");

    println!();
    println!("the harness is part of the experiment, and this table proves it:");
    println!();
    // The same steady-state loop three ways: written out by hand, as the generic
    // run_steady (static dispatch), and behind the vtable (workload A's figure).
    let naive_inline = ns_per_op(|| {
        let mut q: Vec<i64> = Vec::with_capacity(4096);
        for i in 0..1000 {
            q.push(i);
        }
        for i in 0..STEADY {
            q.push(i);
            black_box(q.remove(0));
        }
    }) / STEADY as f64;
    let naive_generic = ns_per_op(|| {
        run_steady(&mut Naive {
            q: Vec::with_capacity(4096),
        });
    }) / STEADY as f64;
    let compact_inline = ns_per_op(|| {
        let mut buf: Vec<i64> = Vec::with_capacity(4096);
        let mut head = 0;
        for i in 0..1000 {
            buf.push(i);
        }
        for i in 0..STEADY {
            buf.push(i);
            black_box(buf[head]);
            head += 1;
            if head > buf.len() / 2 {
                buf.drain(..head);
                head = 0;
            }
        }
    }) / STEADY as f64;
    let compact_generic = ns_per_op(|| {
        run_steady(&mut Compact {
            buf: Vec::with_capacity(4096),
            head: 0,
        });
    }) / STEADY as f64;
    let ring_inline = ns_per_op(|| {
        let (mut buf, mask) = (vec![0i64; 4096], 4095);
        let (mut head, mut count) = (0, 0);
        for i in 0..1000 {
            buf[(head + count) & mask] = i;
            count += 1;
        }
        for i in 0..STEADY {
            buf[(head + count) & mask] = i;
            count += 1;
            black_box(buf[head]);
            head = (head + 1) & mask;
            count -= 1;
        }
    }) / STEADY as f64;
    let ring_generic = ns_per_op(|| {
        run_steady(&mut Ring::new(4096));
    }) / STEADY as f64;

    println!(
        "  {:<26} {:>10} {:>10} {:>10} {:>10}",
        "", "inline", "generic", "dyn", "added"
    );
    for (name, inline, generic, dynamic) in [
        (
            "naive Vec::remove(0)",
            naive_inline,
            naive_generic,
            steady_ns[0],
        ),
        (
            "head index + compact",
            compact_inline,
            compact_generic,
            steady_ns[1],
        ),
        ("ring (fixed)", ring_inline, ring_generic, steady_ns[2]),
    ] {
        println!(
            "  {:<26} {:>7.2} ns {:>7.2} ns {:>7.2} ns {:>7.2} ns",
            name,
            inline,
            generic,
            dynamic,
            dynamic - inline
        );
    }
    println!();
    let inline_allocs = allocs_of(|| {
        let mut q: Vec<i64> = Vec::with_capacity(4096);
        for i in 0..1000 {
            q.push(i);
        }
        for i in 0..200_000 {
            q.push(i);
            black_box(q.remove(0));
        }
    });
    let dyn_allocs = allocs_of(|| {
        let mut q = (IMPLS[0].new)(4096);
        for i in 0..1000 {
            q.push(i);
        }
        for i in 0..200_000 {
            q.push(i);
            black_box(q.pop());
        }
    });
    println!("  allocations, naive, 200,000 ops: inline {inline_allocs}, dyn {dyn_allocs}");
    println!();
    println!("  the interface tax is real and it is not allocation: dyn adds ONE");
    println!("  allocation per run (the Box<dyn Queue> itself: 1 against 2 for 200,000");
    println!("  operations), not one per operation.");
    println!();
    println!("  the third column is the one Go has no counterpart for. `generic` is the");
    println!("  same loop, the same push and pop, behind a type parameter instead of a");
    println!("  vtable: for the ring it ran at the speed of the hand-inlined loop");
    println!("  (0.5 to 1.2 ns across my runs, the high end under load), and `dyn`");
    println!("  cost 3 to 6.6 ns more (3 to 3.2 on the quiet runs). So the tax is the");
    println!("  vtable call, not the trait. Plausible mechanism, not measured here: a");
    println!("  call through a vtable cannot be inlined, so head and count are loaded");
    println!("  and stored through &mut self on every call instead of staying in");
    println!("  registers.");
    println!();
    println!("  head index + compact: dyn added about 0.7 to 3 ns. naive Vec::remove(0):");
    println!("  nothing that survives a rerun -- inline, generic and dyn all ran at 75");
    println!("  to 240 ns (the top under load) and the `added` column changed sign");
    println!("  from run to run. A 1000-element memmove hides a vtable call. Go's");
    println!("  naive queue paid the largest absolute tax (1.82 -> 7.99 ns, +6.17)");
    println!("  because its operation is nearly free; Rust's is not.");
    println!();
    println!("  that is also why example 3, which timed the ring inline (under 1 ns/op),");
    println!("  and this table, where the same ring behind `dyn Queue` costs 3.6 to 7");
    println!("  ns, do not contradict each other: both are correct measurements of");
    println!("  different programs.");
    println!();
    println!("  the practical reading: within THIS table the comparison is fair,");
    println!("  because every row pays the same vtable tax. Across tables it is not.");
    println!("  Never quote a ns/op figure without the harness it was measured in.");

    println!();
    println!("the verdict, and it is not 'use the fastest one':");
    println!();
    println!(
        "  {:<22} {:<34} never when",
        "implementation", "use it when"
    );
    for (name, when, never) in [
        (
            "naive Vec::remove(0)",
            "a handful of elements, ever",
            "the backlog can reach thousands",
        ),
        (
            "head index + compact",
            "you want one file, no surprises",
            "you need worst-case Theta(1)",
        ),
        (
            "ring (fixed)",
            "there is a real capacity limit",
            "bursts must not be refused",
        ),
        (
            "ring (growable)",
            "hot path, size unknown",
            "you needed a memory bound",
        ),
        (
            "LinkedList",
            "you splice whole queues (append)",
            "it is only a queue",
        ),
        ("sync_channel", "it crosses a thread", "it does not"),
    ] {
        println!("  {name:<22} {when:<34} {never}");
    }
    println!();
    {
        let best = steady_ns[..SIX].iter().cloned().fold(f64::MAX, f64::min);
        let worst = steady_ns[..SIX].iter().cloned().fold(0.0, f64::max);
        let worst_o1 = steady_ns[1..SIX].iter().cloned().fold(0.0, f64::max);
        println!(
            "  the spread on workload A is {:.1}x between best and worst of the six, but only",
            worst / best
        );
        println!(
            "  {:.1}x between best and worst of the five that are Theta(1) per operation.",
            worst_o1 / best
        );
    }
    // The one thing a linked list does that an array queue cannot: splice a
    // whole queue onto another by relinking two pointers. Hand all BURST
    // elements back and forth 1000 times; every call moves every element.
    let mut a: LinkedList<i64> = (0..BURST).collect();
    let mut b: LinkedList<i64> = LinkedList::new();
    let t = Instant::now();
    for _ in 0..1000 {
        b.append(black_box(&mut a));
        a.append(black_box(&mut b));
    }
    let per_append = t.elapsed().as_nanos() / 2000;
    black_box((&a, &b));
    println!();
    println!(
        "  LinkedList::append moving all {BURST} elements: {per_append} ns per call (mean of 2000)."
    );
    println!("  That is 1 to 2 ns across my runs with all 200,000 elements moving:");
    println!("  it relinks two pointers, which is the one thing a slice cannot do. It");
    println!("  has no handle to hold on to either: its cursor API (`cursor_front_mut`,");
    println!("  `remove_current`) is unstable on this compiler (E0658, checked by");
    println!("  compiling it), so Go's \"you remove by handle too\" row does not exist");
    println!("  for std's list.");
    println!();
    println!("  In Go the spread was \"real but small next to the difference between a");
    println!("  queue that bounds its memory and one that does not\", and speed \"the");
    println!("  least interesting axis\". The second half survives only without the");
    println!("  naive row: remove(0) is 19x to 38x off the best on workload A and");
    println!("  3,000x to 7,000x off the head index on B, a bigger swing than anything");
    println!("  else in these tables. Among queues that are Theta(1) per operation the");
    println!("  spread is the modest one Go saw, and what was lost and what is held");
    println!("  matter more. So speed is the least interesting axis only inside that");
    println!("  group, and the first thing a measurement can tell you is whether you");
    println!("  are in it.");
}
```

**Rust sample output:**

```
two things differ from the Go program, so read them first:

  1. Go's naive queue is `q = q[1:]` (move the start, never copy). Rust
     cannot write that on an owned Vec, so row 1 is `Vec::remove(0)`,
     which shifts the whole backlog on EVERY pop. That is a different
     algorithm, so its row is not comparable with Go's naive row. The
     other five queues are the same algorithms as Go's.
  2. The burst is 200000, not Go's 2,000,000: remove(0) is Theta(n^2) over
     a burst, and 2,000,000 would be ~16 TB of memmove. Every queue gets
     the same scaled burst. The capacity of 1024 is unchanged.

workload A -- STEADY STATE: 1000 live, one push and one pop each,
500,000 times. A work queue keeping up with its producer.

  implementation                      ns/op
  naive Vec::remove(0)                94.71
  head index + compact                 2.86
  ring (fixed)                         4.11
  ring (growable)                      3.14
  LinkedList                          19.52
  sync_channel                         7.43
  VecDeque (std, not in Go)            3.13

  all behind the same `dyn Queue` vtable, so the call overhead is
  identical and the differences are the structures themselves.

  the first six rows are Go's six; VecDeque is std's growable ring, added
  for reference. Across my runs (a few of them while the machine was
  loaded by other processes) head index, ring (growable) and VecDeque
  sat within about 1 ns of each other on the quiet runs and swapped
  places, so this workload does not pick one of them. Against the best of
  them: ring (fixed) 1.3x to 2.3x, sync_channel 2.6x to 3.9x, LinkedList
  6.8x to 12x, and Vec::remove(0) 19x to 38x, because every pop shifts
  the 1000 live elements.

workload B -- BURST THEN DRAIN: push 200000, then pop them all,
starting from capacity 1024. A BFS frontier, a batch import, a
backlog after an outage.

  implementation                      ns/op      peak MB    pushes lost
  naive Vec::remove(0)              7025.40          2.0              0
  head index + compact                 1.72          2.0              0
  ring (fixed)                         1.06          0.0         198976
  ring (growable)                      2.28          2.0              0
  LinkedList                          19.13          4.6              0
  sync_channel                         2.55          0.0         198976
  VecDeque (std, not in Go)            2.14          2.0              0

  read the last column first. The two BOUNDED implementations threw away 99.5%
  of the input (198976 of 200000), silently, and were fast about it. That is
  not a benchmark loss -- it is the policy working exactly as
  designed (example 9), on a workload where the policy is wrong.

  a bounded queue is not a slower unbounded queue. It is a different
  contract, and the ns/op column cannot see the difference.

  the naive row is the other one to read: 4091x the head index's cost per op,
  and it grows with the burst, because every pop shifts what is left.
  Go's naive queue stayed in the same few-ns range as the array queues
  on this workload (5.04 ns/op in the Go run); that is the reslice, which
  Rust cannot write on an owned Vec. Among the queues that lost nothing,
  head index was fastest in all my runs but one, ahead of VecDeque by up
  to roughly 35% (in the exception VecDeque led by 2%), and LinkedList
  took 9x to 15x the head index.

workload C -- WHAT IS STILL HELD after draining to empty:

  implementation               MB held at len 0
  naive Vec::remove(0)                      2.0
  head index + compact                      2.0
  ring (growable)                           2.0
  LinkedList                                0.0
  VecDeque (std, not in Go)                 2.0

  this is example 1's case B, and it reverses the ranking again:
  LinkedList is the ONLY structure here that gives its memory back on
  drain, because it frees every node as it pops it -- there is no GC to
  wait for. The array-backed queues are all still holding their peak,
  2.0 MB: 262,144 slots of 8 bytes, the doubling from 1024 that had
  room for 200,000 elements. VecDeque is no different.

  the slowest of the Theta(1) queues wins the memory-return column
  outright (7x to 12x the best on workload A, 9x to 15x on B). The
  ranking differs per workload, which is the entire reason to run all
  three. (Rust's naive queue holds the same 2.0 MB as the other arrays:
  remove(0) never abandons a prefix as Go's reslice does, so it has no
  leak to show in a table like this -- see example 1.)

the harness is part of the experiment, and this table proves it:

                                 inline    generic        dyn      added
  naive Vec::remove(0)         86.46 ns   85.90 ns   94.71 ns    8.26 ns
  head index + compact          1.06 ns    1.03 ns    2.86 ns    1.80 ns
  ring (fixed)                  0.57 ns    0.63 ns    4.11 ns    3.54 ns

  allocations, naive, 200,000 ops: inline 1, dyn 2

  the interface tax is real and it is not allocation: dyn adds ONE
  allocation per run (the Box<dyn Queue> itself: 1 against 2 for 200,000
  operations), not one per operation.

  the third column is the one Go has no counterpart for. `generic` is the
  same loop, the same push and pop, behind a type parameter instead of a
  vtable: for the ring it ran at the speed of the hand-inlined loop
  (0.5 to 1.2 ns across my runs, the high end under load), and `dyn`
  cost 3 to 6.6 ns more (3 to 3.2 on the quiet runs). So the tax is the
  vtable call, not the trait. Plausible mechanism, not measured here: a
  call through a vtable cannot be inlined, so head and count are loaded
  and stored through &mut self on every call instead of staying in
  registers.

  head index + compact: dyn added about 0.7 to 3 ns. naive Vec::remove(0):
  nothing that survives a rerun -- inline, generic and dyn all ran at 75
  to 240 ns (the top under load) and the `added` column changed sign
  from run to run. A 1000-element memmove hides a vtable call. Go's
  naive queue paid the largest absolute tax (1.82 -> 7.99 ns, +6.17)
  because its operation is nearly free; Rust's is not.

  that is also why example 3, which timed the ring inline (under 1 ns/op),
  and this table, where the same ring behind `dyn Queue` costs 3.6 to 7
  ns, do not contradict each other: both are correct measurements of
  different programs.

  the practical reading: within THIS table the comparison is fair,
  because every row pays the same vtable tax. Across tables it is not.
  Never quote a ns/op figure without the harness it was measured in.

the verdict, and it is not 'use the fastest one':

  implementation         use it when                        never when
  naive Vec::remove(0)   a handful of elements, ever        the backlog can reach thousands
  head index + compact   you want one file, no surprises    you need worst-case Theta(1)
  ring (fixed)           there is a real capacity limit     bursts must not be refused
  ring (growable)        hot path, size unknown             you needed a memory bound
  LinkedList             you splice whole queues (append)   it is only a queue
  sync_channel           it crosses a thread                it does not

  the spread on workload A is 33.1x between best and worst of the six, but only
  6.8x between best and worst of the five that are Theta(1) per operation.

  LinkedList::append moving all 200000 elements: 1 ns per call (mean of 2000).
  That is 1 to 2 ns across my runs with all 200,000 elements moving:
  it relinks two pointers, which is the one thing a slice cannot do. It
  has no handle to hold on to either: its cursor API (`cursor_front_mut`,
  `remove_current`) is unstable on this compiler (E0658, checked by
  compiling it), so Go's "you remove by handle too" row does not exist
  for std's list.

  In Go the spread was "real but small next to the difference between a
  queue that bounds its memory and one that does not", and speed "the
  least interesting axis". The second half survives only without the
  naive row: remove(0) is 19x to 38x off the best on workload A and
  3,000x to 7,000x off the head index on B, a bigger swing than anything
  else in these tables. Among queues that are Theta(1) per operation the
  spread is the modest one Go saw, and what was lost and what is held
  matter more. So speed is the least interesting axis only inside that
  group, and the first thing a measurement can tell you is whether you
  are in it.
```

> **Go vs Rust:** Go's naive `q = q[1:]` has no owned-`Vec` spelling, so row 1 is `Vec::remove(0)`, a different algorithm that shifts the backlog on every pop (19x to 38x the best queue on workload A and 3,000x to 7,000x the head index on B across my runs), and the burst is scaled from 2,000,000 to 200,000 because at Go's size it would be ~16 TB of memmove, so the MB columns are not comparable with Go's 16.9 / 106.8 (`LinkedList` is 24 bytes per element here, 4.6 MB for 200,000, against about 56 in Go's `container/list`). The other five rows are Go's algorithms with FIFO order asserted, and the outline carries over: the bounded ring and `sync_channel` (`try_send` / `try_recv`, no threads involved) lose 198,976 of 200,000 where Go lost 1,998,976 of 2,000,000, and the list alone returns its memory on drain. The interface tax carries over for the ring (0.5 ns inline, 3.6 ns behind `dyn Queue` on quiet runs; Go 0.55 and 3.55) but a monomorphised generic runs at the inline speed, the naive row shows no tax because its memmove hides the vtable call, and std's `LinkedList` has no stable handle API, so Go's "remove by handle" row has no equivalent.

**Complexity:** three workloads, three winners — head-index on speed, the unbounded queues on the
burst (the bounded ones silently discarded **1,998,976** pushes), and `container/list` alone returns
its memory on drain · the same naive queue measures **1.82 ns/op** inline and **7.99 ns/op** behind an
interface, with *identical* allocation counts

---

> Next tier: [🔴 hard](3-hard.md) — BFS variations, 0-1 BFS, a queue that knows its own minimum,
> work stealing, and the capstone.
