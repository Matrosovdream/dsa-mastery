# Step 09 — Queues & Deques · 🟢 Easy

Examples **1–6**: the other end of the slice, where it stops being free — and the two ways to fix it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 3–5 are deterministic apart from their closing benchmark. Examples 1 and 2 report memory,
which varies a little between runs.

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

## 1. The naive slice queue, and what it actually costs

`🟢 easy` · *the folklore is two-thirds right*

Everyone knows `q = q[1:]` leaks. Almost nobody has measured *when*, and the answer is more
interesting than the warning: in the steady state it does not leak at all.

**Steps:**

1. Run a queue in steady state — one push, one pop, a thousand live — for two million operations.
2. Then drain a large queue and hold the one-element result.
3. Then absorb a burst and drain back down.

```go
package main

import (
	"fmt"
	"runtime"
)

// Lesson 08's stack only ever touched the TAIL of a slice, which is the cheap
// end. A queue needs BOTH ends -- and the front is where a slice stops being
// free.
//
// The naive version looks perfect:
//
//	enqueue   q = append(q, v)   Theta(1) amortized
//	peek      q[0]               Theta(1)
//	dequeue   q = q[1:]          Theta(1)
//
// "Everyone knows" this leaks memory. Everyone is two-thirds right, and the
// third that is wrong is worth measuring, because it tells you exactly when a
// ring buffer is buying you something and when it is not.

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

func main() {
	q := []int{}
	for _, v := range []int{1, 2, 3} {
		q = append(q, v)
	}
	fmt.Printf("start: %v (len=%d cap=%d)\n", q, len(q), cap(q))
	q = q[1:]
	fmt.Printf("after one dequeue: %v (len=%d cap=%d)\n", q, len(q), cap(q))
	fmt.Println()
	fmt.Println("cap fell from 4 to 3 -- but no memory was freed. `q[1:]` moved the")
	fmt.Println("slice POINTER forward one element; cap is just 'how far from here to")
	fmt.Println("the end of the array'. The dequeued 1 is still in that array, and the")
	fmt.Println("array is still reachable through q.")

	fmt.Println()
	fmt.Println("so does it leak? Three workloads, three different answers.")
	fmt.Println()

	// ---- A: steady state -------------------------------------------------
	base := heapMB()
	a := make([]int, 0, 1000)
	for i := 0; i < 1000; i++ {
		a = append(a, i)
	}
	maxCap := 0
	for i := 0; i < 2_000_000; i++ {
		a = append(a, i)
		a = a[1:]
		if cap(a) > maxCap {
			maxCap = cap(a)
		}
	}
	fmt.Println("A. STEADY STATE -- one in, one out, 2,000,000 times, 1000 live:")
	fmt.Printf("   final len=%d, largest cap ever seen=%d, heap grew %.1f MB\n",
		len(a), maxCap, heapMB()-base)
	fmt.Println("   -> NO LEAK. This is the part the folklore gets wrong.")
	fmt.Println()
	fmt.Println("   when append runs out of room it allocates a new array and copies")
	fmt.Println("   only the LIVE elements. The abandoned head goes with the old array,")
	fmt.Println("   and the GC takes it. Memory stays around 1.5x the live size forever.")
	runtime.KeepAlive(a)

	// ---- B: drain and hold ----------------------------------------------
	fmt.Println()
	base2 := heapMB()
	b := make([]int, 10_000_000)
	afterAlloc := heapMB() - base2
	for i := 0; i < 9_999_999; i++ {
		b = b[1:]
	}
	afterDrain := heapMB() - base2
	fmt.Println("B. DRAIN AND HOLD -- build 10,000,000, dequeue all but one, keep it:")
	fmt.Printf("   len=%d cap=%d, heap after building %.1f MB, after draining %.1f MB\n",
		len(b), cap(b), afterAlloc, afterDrain)
	fmt.Println("   -> LEAK. 76 MB held alive by a ONE-ELEMENT slice.")
	fmt.Println()
	fmt.Println("   append never ran, so the array was never replaced. The header says")
	fmt.Println("   cap=1 and the truth is 80 MB. This is lesson 04's example 3 wearing")
	fmt.Println("   a queue costume.")
	runtime.KeepAlive(b)

	// ---- C: burst then drain --------------------------------------------
	fmt.Println()
	base3 := heapMB()
	c := []int{}
	for i := 0; i < 5_000_000; i++ {
		c = append(c, i)
	}
	peakCap := cap(c)
	peak := heapMB() - base3
	for len(c) > 1000 {
		c = c[1:]
	}
	after := heapMB() - base3
	fmt.Println("C. BURST THEN DRAIN -- 5,000,000 arrive, then drain back to 1000:")
	fmt.Printf("   peak cap=%d (%.1f MB) -> len=%d cap=%d, heap still %.1f MB\n",
		peakCap, peak, len(c), cap(c), after)
	fmt.Println("   -> LEAK, and this is the one that bites real servers.")
	fmt.Println()
	fmt.Println("   the queue is small again and the memory is not. A slice never")
	fmt.Println("   shrinks (lesson 06, example 8), so the array stays at the")
	fmt.Println("   HIGH-WATER MARK until every element is dequeued and append")
	fmt.Println("   happens to reallocate.")
	runtime.KeepAlive(c)

	fmt.Println()
	fmt.Println("what that actually tells you:")
	fmt.Println()
	fmt.Println("  the naive slice queue is fine for a bounded, steady workload, and")
	fmt.Println("  it is a memory bug for a bursty one -- which is most real queues.")
	fmt.Println("  A request queue, a work queue, a BFS frontier: all bursty.")
	fmt.Println()
	fmt.Println("  and even in case A you are paying a periodic Theta(n) copy as the")
	fmt.Println("  live window marches through memory and append hauls it back to the")
	fmt.Println("  start. Amortized Theta(1), with a tail (lesson 03, example 15).")

	fmt.Println()
	fmt.Println("the structural mismatch, in one sentence:")
	fmt.Println()
	fmt.Println("  a slice grows at one end and is indexed from a fixed start; a queue")
	fmt.Println("  consumes at one end and produces at the other, so its live region")
	fmt.Println("  MOVES. Those two models do not fit.")
	fmt.Println()
	fmt.Println("two ways to reconcile them, and the rest of this lesson is both:")
	fmt.Println()
	fmt.Println("  1. keep a HEAD INDEX and compact on your own schedule (example 2)")
	fmt.Println("  2. make the array CIRCULAR so the live region wraps instead of")
	fmt.Println("     marching, and never moves at all (example 3)")
}
```

**Sample output:**

```
start: [1 2 3] (len=3 cap=4)
after one dequeue: [2 3] (len=2 cap=3)

cap fell from 4 to 3 -- but no memory was freed. `q[1:]` moved the
slice POINTER forward one element; cap is just 'how far from here to
the end of the array'. The dequeued 1 is still in that array, and the
array is still reachable through q.

so does it leak? Three workloads, three different answers.

A. STEADY STATE -- one in, one out, 2,000,000 times, 1000 live:
   final len=1000, largest cap ever seen=1535, heap grew 0.0 MB
   -> NO LEAK. This is the part the folklore gets wrong.

   when append runs out of room it allocates a new array and copies
   only the LIVE elements. The abandoned head goes with the old array,
   and the GC takes it. Memory stays around 1.5x the live size forever.

B. DRAIN AND HOLD -- build 10,000,000, dequeue all but one, keep it:
   len=1 cap=1, heap after building 76.3 MB, after draining 76.3 MB
   -> LEAK. 76 MB held alive by a ONE-ELEMENT slice.

   append never ran, so the array was never replaced. The header says
   cap=1 and the truth is 80 MB. This is lesson 04's example 3 wearing
   a queue costume.

C. BURST THEN DRAIN -- 5,000,000 arrive, then drain back to 1000:
   peak cap=5045248 (38.5 MB) -> len=1000 cap=46248, heap still 38.5 MB
   -> LEAK, and this is the one that bites real servers.

   the queue is small again and the memory is not. A slice never
   shrinks (lesson 06, example 8), so the array stays at the
   HIGH-WATER MARK until every element is dequeued and append
   happens to reallocate.

what that actually tells you:

  the naive slice queue is fine for a bounded, steady workload, and
  it is a memory bug for a bursty one -- which is most real queues.
  A request queue, a work queue, a BFS frontier: all bursty.

  and even in case A you are paying a periodic Theta(n) copy as the
  live window marches through memory and append hauls it back to the
  start. Amortized Theta(1), with a tail (lesson 03, example 15).

the structural mismatch, in one sentence:

  a slice grows at one end and is indexed from a fixed start; a queue
  consumes at one end and produces at the other, so its live region
  MOVES. Those two models do not fit.

two ways to reconcile them, and the rest of this lesson is both:

  1. keep a HEAD INDEX and compact on your own schedule (example 2)
  2. make the array CIRCULAR so the live region wraps instead of
     marching, and never moves at all (example 3)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::VecDeque;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::Instant;

// Lesson 08's stack only ever touched the TAIL of a Vec, which is the cheap
// end. A queue needs BOTH ends -- and the front is where a Vec stops being
// free.
//
// Go's naive queue is `q = q[1:]`: dequeue by moving the slice pointer forward.
// Rust cannot write that on an owned Vec -- there is no operation that moves
// an owned buffer's start. What it offers instead:
//
//	enqueue   q.push(v)       Theta(1) amortized
//	peek      q[0]
//	dequeue   q.remove(0)     Theta(n): shifts every other element left
//	batch     q.drain(..k)    one shift for k dequeues
//	view      &q[1..]         a BORROWED slice: Go's reslice, but read-only
//
// So the naive Rust queue is not "leaky and fast", it is "tidy and slow": it
// pays in time where Go's pays in memory. The leak SHAPES still exist (a
// borrowed view keeps its owner alive; a Vec never shrinks), and this program
// measures each one.
//
// What the borrow checker does with Go's pattern (checked, not run here):
//
//	let view = &q[1..];
//	q.push(4);          // error[E0502]: cannot borrow `q` as mutable
//	println!("{view:?}");   //   because it is also borrowed as immutable
//
//	let view = &b[1..];
//	drop(b);            // error[E0505]: cannot move out of `b` because it is borrowed
//	println!("{view:?}");

// ---- the kit's allocation counter: stand-in for runtime.MemStats ----
// No GC means no runtime.GC() before every reading: this counts bytes
// currently live on the heap, exactly, every run.
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
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

fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

fn mb(bytes: usize) -> f64 {
    bytes as f64 / (1 << 20) as f64
}

/// Go's `%v` of a slice: [1 2 3]
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let mut q: Vec<i64> = Vec::new();
    for v in [1, 2, 3] {
        q.push(v);
    }
    println!("start: {} (len={} cap={})", show(&q), q.len(), q.capacity());
    // Go's `q = q[1:]` has a Rust spelling, but only as a borrowed view:
    let view = &q[1..];
    println!(
        "view &q[1..]: {} (len={}; a &[T] is pointer + length, there is no cap)",
        show(view),
        view.len()
    );
    // ...and the dequeue Rust lets you write on the owner is remove(0).
    let front = q.remove(0);
    println!(
        "after one dequeue: {} (len={} cap={})   removed {front}",
        show(&q),
        q.len(),
        q.capacity()
    );
    println!();
    println!("cap stayed at 4 and no pointer moved: remove(0) shifted the 2 and the 3");
    println!("left by one slot, so the buffer's front is always the queue's front and");
    println!("nothing is abandoned. The price is the shift -- every element, every time.");
    println!("The view is the other model: it starts one element in, and the borrow");
    println!("checker refuses to push to q while the view is still used (E0502).");

    println!();
    println!("so does it leak? Three workloads, three different answers.");
    println!();

    // ---- A: steady state -------------------------------------------------
    let base = live_bytes();
    let mut a: Vec<i64> = Vec::with_capacity(1000);
    for i in 0..1000 {
        a.push(i);
    }
    let mut max_cap = 0;
    let mut cap_changes = 0; // how many times the buffer was replaced
    let mut shifted: u64 = 0;
    let t = Instant::now();
    for i in 0..2_000_000 {
        let before = a.capacity();
        a.push(i);
        if a.capacity() != before {
            cap_changes += 1;
        }
        shifted += (a.len() - 1) as u64; // what remove(0) is about to move
        black_box(a.remove(0));
        max_cap = max_cap.max(a.capacity());
    }
    let remove_ms = t.elapsed().as_secs_f64() * 1e3;
    let a_bytes = live_bytes() - base;
    println!("A. STEADY STATE -- one in, one out, 2,000,000 times, 1000 live:");
    println!(
        "   final len={}, largest cap ever seen={}, heap grew {:.1} MB ({} bytes)",
        a.len(),
        max_cap,
        mb(a_bytes),
        a_bytes
    );
    println!("   -> NO LEAK: nothing is ever abandoned behind the front.");
    println!();
    println!(
        "   the buffer was replaced {cap_changes} time(s): once, when the 1001st element did not fit."
    );
    println!("   After that it stayed put: Vec capacity is a high-water mark, here 2x the");
    println!("   live size. What it paid instead was the shift:");
    println!(
        "   {shifted} elements moved by remove(0) in {remove_ms:.0} ms -- 1000 of them per dequeue."
    );
    black_box(&a);

    // The std answer on the same workload: a VecDeque is a ring buffer
    // (example 3) that grows on demand, so nothing is ever shifted.
    let base_d = live_bytes();
    let mut d: VecDeque<i64> = VecDeque::with_capacity(1000);
    for i in 0..1000 {
        d.push_back(i);
    }
    let mut d_max_cap = 0;
    let t = Instant::now();
    for i in 0..2_000_000 {
        d.push_back(i);
        black_box(d.pop_front());
        d_max_cap = d_max_cap.max(d.capacity());
    }
    let deque_ms = t.elapsed().as_secs_f64() * 1e3;
    println!();
    println!(
        "   same workload on std's VecDeque: len={}, largest cap={}, heap {} bytes, {:.0} ms ({:.0}x faster)",
        d.len(),
        d_max_cap,
        live_bytes() - base_d,
        deque_ms,
        remove_ms / deque_ms
    );
    black_box(&d);
    drop((a, d));

    // ---- B: drain and hold ----------------------------------------------
    println!();
    // B1: Go's mechanism. A borrowed view, advanced 9,999,999 times.
    let base2 = live_bytes();
    let b: Vec<i64> = vec![0; 10_000_000];
    let after_alloc = live_bytes() - base2;
    let mut view: &[i64] = &b;
    for _ in 0..9_999_999 {
        view = &view[1..];
    }
    let view = black_box(view);
    let after_drain = live_bytes() - base2;
    println!("B. DRAIN AND HOLD -- build 10,000,000, dequeue all but one, keep it:");
    println!(
        "   as a view: len={} (no cap), heap after building {:.1} MB, after draining {:.1} MB",
        view.len(),
        mb(after_alloc),
        mb(after_drain)
    );
    println!("   -> the same 76 MB held by a ONE-ELEMENT slice -- but now it is a *borrow*.");
    println!();
    println!("   The view cannot outlive `b`: the compiler rejects `drop(b)` while the");
    println!("   view is still used (E0505, see the top of the file), so the 80 MB cannot");
    println!("   be orphaned behind a tiny handle. Copy the element out, and the owner can go:");
    let kept = view[0];
    drop(b);
    println!(
        "   kept={kept}, heap after drop(b): {:.1} MB",
        mb(live_bytes() - base2)
    );

    // B2: the owned queue. drain(..k) dequeues k elements in one shift.
    println!();
    let base2 = live_bytes();
    let mut b: Vec<i64> = vec![0; 10_000_000];
    b.drain(..9_999_999);
    println!(
        "   as an owned Vec after drain(..9_999_999): len={} cap={}, heap {:.1} MB",
        b.len(),
        b.capacity(),
        mb(live_bytes() - base2)
    );
    println!("   -> LEAK-SHAPED too, but the header does not lie: cap says 10,000,000.");
    b.shrink_to_fit();
    println!(
        "   after shrink_to_fit(): cap={}, heap {:.1} MB",
        b.capacity(),
        mb(live_bytes() - base2)
    );
    drop(b);

    // ---- C: burst then drain --------------------------------------------
    println!();
    let base3 = live_bytes();
    let mut c: Vec<i64> = Vec::new();
    for i in 0..5_000_000 {
        c.push(i);
    }
    let peak_cap = c.capacity();
    let peak = live_bytes() - base3;
    let excess = c.len() - 1000;
    c.drain(..excess);
    let after = live_bytes() - base3;
    println!("C. BURST THEN DRAIN -- 5,000,000 arrive, then drain back to 1000:");
    println!(
        "   peak cap={} ({:.1} MB) -> len={} cap={}, heap still {:.1} MB",
        peak_cap,
        mb(peak),
        c.len(),
        c.capacity(),
        mb(after)
    );
    println!("   -> LEAK, and this is the one that bites real servers.");
    println!();
    println!(
        "   (Vec doubles, so the peak is {:.2}x the 5,000,000 elements held; the",
        peak_cap as f64 / 5e6
    );
    println!("   Go output above peaked at cap=5045248: its growth is gentler at this size.)");
    println!();
    println!("   the queue is small again and the memory is not. A Vec never shrinks");
    println!("   on its own (lesson 06, example 8): capacity stays at the HIGH-WATER");
    println!("   MARK until you ask. Here you can ask:");
    c.shrink_to_fit();
    println!(
        "   after shrink_to_fit(): len={} cap={}, heap {:.3} MB ({} bytes)",
        c.len(),
        c.capacity(),
        mb(live_bytes() - base3),
        live_bytes() - base3
    );
    black_box(&c);
    drop(c);

    println!();
    println!("what that actually tells you:");
    println!();
    println!("  Go's naive queue is a memory bug for bursty work and cheap for steady");
    println!("  work. Rust's naive queue, Vec::remove(0), is the opposite trade: no");
    println!("  abandoned prefix, but Theta(n) per dequeue -- fine for ten elements,");
    println!("  quadratic for a request queue, a work queue or a BFS frontier.");
    println!();
    println!("  and what a Vec shares with a Go slice is cases B and C: the buffer's");
    println!("  size is set by its PEAK, not by its live count.");

    println!();
    println!("the structural mismatch, in one sentence:");
    println!();
    println!("  a Vec grows at one end and is indexed from a fixed start; a queue");
    println!("  consumes at one end and produces at the other, so its live region");
    println!("  MOVES. Those two models do not fit.");
    println!();
    println!("two ways to reconcile them, and the rest of this lesson is both:");
    println!();
    println!("  1. keep a HEAD INDEX and compact on your own schedule (example 2)");
    println!("  2. make the array CIRCULAR so the live region wraps instead of");
    println!("     marching, and never moves at all (example 3)");
}
```

**Rust sample output:**

```
start: [1 2 3] (len=3 cap=4)
view &q[1..]: [2 3] (len=2; a &[T] is pointer + length, there is no cap)
after one dequeue: [2 3] (len=2 cap=4)   removed 1

cap stayed at 4 and no pointer moved: remove(0) shifted the 2 and the 3
left by one slot, so the buffer's front is always the queue's front and
nothing is abandoned. The price is the shift -- every element, every time.
The view is the other model: it starts one element in, and the borrow
checker refuses to push to q while the view is still used (E0502).

so does it leak? Three workloads, three different answers.

A. STEADY STATE -- one in, one out, 2,000,000 times, 1000 live:
   final len=1000, largest cap ever seen=2000, heap grew 0.0 MB (16000 bytes)
   -> NO LEAK: nothing is ever abandoned behind the front.

   the buffer was replaced 1 time(s): once, when the 1001st element did not fit.
   After that it stayed put: Vec capacity is a high-water mark, here 2x the
   live size. What it paid instead was the shift:
   2000000000 elements moved by remove(0) in 154 ms -- 1000 of them per dequeue.

   same workload on std's VecDeque: len=1000, largest cap=2000, heap 16000 bytes, 6 ms (27x faster)

B. DRAIN AND HOLD -- build 10,000,000, dequeue all but one, keep it:
   as a view: len=1 (no cap), heap after building 76.3 MB, after draining 76.3 MB
   -> the same 76 MB held by a ONE-ELEMENT slice -- but now it is a *borrow*.

   The view cannot outlive `b`: the compiler rejects `drop(b)` while the
   view is still used (E0505, see the top of the file), so the 80 MB cannot
   be orphaned behind a tiny handle. Copy the element out, and the owner can go:
   kept=0, heap after drop(b): 0.0 MB

   as an owned Vec after drain(..9_999_999): len=1 cap=10000000, heap 76.3 MB
   -> LEAK-SHAPED too, but the header does not lie: cap says 10,000,000.
   after shrink_to_fit(): cap=1, heap 0.0 MB

C. BURST THEN DRAIN -- 5,000,000 arrive, then drain back to 1000:
   peak cap=8388608 (64.0 MB) -> len=1000 cap=8388608, heap still 64.0 MB
   -> LEAK, and this is the one that bites real servers.

   (Vec doubles, so the peak is 1.68x the 5,000,000 elements held; the
   Go output above peaked at cap=5045248: its growth is gentler at this size.)

   the queue is small again and the memory is not. A Vec never shrinks
   on its own (lesson 06, example 8): capacity stays at the HIGH-WATER
   MARK until you ask. Here you can ask:
   after shrink_to_fit(): len=1000 cap=1000, heap 0.008 MB (8000 bytes)

what that actually tells you:

  Go's naive queue is a memory bug for bursty work and cheap for steady
  work. Rust's naive queue, Vec::remove(0), is the opposite trade: no
  abandoned prefix, but Theta(n) per dequeue -- fine for ten elements,
  quadratic for a request queue, a work queue or a BFS frontier.

  and what a Vec shares with a Go slice is cases B and C: the buffer's
  size is set by its PEAK, not by its live count.

the structural mismatch, in one sentence:

  a Vec grows at one end and is indexed from a fixed start; a queue
  consumes at one end and produces at the other, so its live region
  MOVES. Those two models do not fit.

two ways to reconcile them, and the rest of this lesson is both:

  1. keep a HEAD INDEX and compact on your own schedule (example 2)
  2. make the array CIRCULAR so the live region wraps instead of
     marching, and never moves at all (example 3)
```

> **Go vs Rust:** Rust has no `q = q[1:]` on an owned `Vec`; the owned naive queue is `remove(0)`, which never abandons a prefix and so does not leak in case A, but it shifted 2,000,000,000 elements (1000 per dequeue) to get there. Cases B and C do reproduce: a length-1 view or `Vec` still holds 76.3 MB, and a `Vec` keeps its doubling peak (cap 8388608, versus Go's 5045248) until `shrink_to_fit`. There is no GC, so memory is read exactly from a counting allocator rather than `runtime.ReadMemStats` after `runtime.GC()`.

**Complexity:** `append` Θ(1) amortized, `q[1:]` Θ(1) · **space is the problem, not time** — the live
region marches forward and the abandoned prefix is still reachable through the backing array

---

## 2. Fix one: a head index and scheduled compaction

`🟢 easy` · *bounded memory, on a dial you choose*

Keep the elements where they are and remember where the live region starts. The dead prefix is then
*yours to reclaim on your own schedule* — which turns a leak into a tuning parameter.

**Steps:**

1. Replace the reslice with an index, and zero each vacated slot.
2. Compact when the dead prefix crosses a threshold.
3. Sweep the threshold and watch the copies/waste trade appear.

```go
package main

import (
	"fmt"
	"testing"
)

// Fix 1: stop reslicing. Keep a HEAD INDEX into a slice you own, and copy the
// live region back to the front when the wasted prefix gets too big.
//
// This keeps the mental model of a slice -- contiguous, indexable, printable --
// and puts YOU in charge of when the Theta(n) copy happens, instead of leaving
// it to append.

type Queue struct {
	buf  []int
	head int // index of the front element; buf[:head] is dead space

	compactions int
	copied      int
}

func (q *Queue) Len() int { return len(q.buf) - q.head }

func (q *Queue) Enqueue(v int) { q.buf = append(q.buf, v) }

// Dequeue advances the head and compacts when more than half the buffer is
// dead. The RATIO is the whole design decision -- see below.
func (q *Queue) Dequeue() (int, bool) {
	if q.head == len(q.buf) {
		return 0, false
	}
	v := q.buf[q.head]
	q.buf[q.head] = 0 // release any pointer (lesson 06, example 2)
	q.head++

	if q.head > len(q.buf)/2 {
		q.compact()
	}
	return v, true
}

func (q *Queue) compact() {
	live := len(q.buf) - q.head
	copy(q.buf, q.buf[q.head:]) // move the live region back to the front
	clear(q.buf[live:])         // and zero what is left behind
	q.buf = q.buf[:live]
	q.head = 0

	q.compactions++
	q.copied += live
}

func (q *Queue) Peek() (int, bool) {
	if q.head == len(q.buf) {
		return 0, false
	}
	return q.buf[q.head], true
}

// live returns the elements currently in the queue.
func (q *Queue) live() []int { return q.buf[q.head:] }

// --- a version with the compaction threshold as a knob ---

// tunable exposes the threshold: compact once the dead prefix exceeds
// deadPct of the buffer. deadPct == 0 means compact on EVERY dequeue.
type tunable struct {
	buf         []int
	head        int
	deadPct     int
	compactions int
	copied      int
}

func (q *tunable) Enqueue(v int) { q.buf = append(q.buf, v) }
func (q *tunable) Dequeue() (int, bool) {
	if q.head == len(q.buf) {
		return 0, false
	}
	v := q.buf[q.head]
	q.head++
	if q.head*100 > len(q.buf)*q.deadPct {
		live := len(q.buf) - q.head
		copy(q.buf, q.buf[q.head:])
		q.buf = q.buf[:live]
		q.head = 0
		q.compactions++
		q.copied += live
	}
	return v, true
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
	var q Queue
	fmt.Println("a head index instead of reslicing:")
	fmt.Println()
	for _, v := range []int{1, 2, 3, 4} {
		q.Enqueue(v)
	}
	fmt.Printf("  enqueue 1..4   buf=%v head=%d live=%v\n", q.buf, q.head, q.live())
	for i := 0; i < 3; i++ {
		v, _ := q.Dequeue()
		fmt.Printf("  dequeue -> %d   buf=%v head=%d live=%v\n", v, q.buf, q.head, q.live())
	}
	fmt.Println()
	fmt.Println("  the first two dequeues just advance head and zero the vacated slot.")
	fmt.Println("  The THIRD crosses the halfway mark (head=3 > 4/2), so it compacts:")
	fmt.Println("  the live tail is copied back to index 0 and the buffer shortened,")
	fmt.Println("  which is why the last line reads buf=[4] head=0.")
	fmt.Println()
	fmt.Println("  nothing is abandoned, so nothing leaks -- and the queue is still a")
	fmt.Println("  plain contiguous slice you can print, range over and index.")

	fmt.Println()
	fmt.Println("the amortized argument, counted:")
	fmt.Println()
	fmt.Printf("  %12s %14s %16s %18s\n", "operations", "compactions", "elements copied", "copies/operation")
	for _, n := range []int{1_000, 10_000, 100_000, 1_000_000} {
		var r Queue
		for i := 0; i < 1000; i++ { // a standing backlog of 1000
			r.Enqueue(i)
		}
		for i := 0; i < n; i++ {
			r.Enqueue(i)
			r.Dequeue()
		}
		fmt.Printf("  %12d %14d %16d %18.2f\n",
			n, r.compactions, r.copied, float64(r.copied)/float64(2*n))
	}
	fmt.Println()
	fmt.Println("  copies per operation settles at a constant, so this is amortized")
	fmt.Println("  Theta(1) by exactly the aggregate argument from lesson 03: each")
	fmt.Println("  compaction can only happen after enough dequeues to make half the")
	fmt.Println("  buffer dead, and those dequeues pay for it.")

	fmt.Println()
	fmt.Println("the threshold is a real knob, and it is the same trade as lesson 06's")
	fmt.Println("shrink hysteresis -- compact eagerly and you copy constantly:")
	fmt.Println()
	fmt.Printf("  %-28s %14s %18s %18s\n", "dead space tolerated", "compactions", "elements copied", "copies/operation")
	for _, pct := range []int{0, 25, 50, 75} {
		t := &tunable{deadPct: pct}
		for i := 0; i < 1000; i++ {
			t.Enqueue(i)
		}
		for i := 0; i < 100_000; i++ {
			t.Enqueue(i)
			t.Dequeue()
		}
		label := fmt.Sprintf("%d%%", pct)
		if pct == 0 {
			label = "0% (compact every dequeue)"
		}
		fmt.Printf("  %-28s %14d %18d %18.2f\n",
			label, t.compactions, t.copied, float64(t.copied)/200_000)
	}
	fmt.Println()
	fmt.Println("  compacting on every dequeue copies the whole live region every")
	fmt.Println("  time -- 1000 elements per operation here, and Theta(n) per")
	fmt.Println("  operation in general, which makes the queue quadratic.")
	fmt.Println()
	fmt.Println("  tolerating more dead space copies less and wastes more. It is the")
	fmt.Println("  same dial as lesson 06's shrink hysteresis, and 50% is the same")
	fmt.Println("  default for the same reason: it is the point where the copies you")
	fmt.Println("  pay for are covered by the dequeues that made them necessary.")

	fmt.Println()
	fmt.Println("versus the naive version, on a bursty workload:")
	fmt.Println()
	const burst = 200_000
	tNaive := nsPerOp(func() {
		s := []int{}
		for i := 0; i < burst; i++ {
			s = append(s, i)
		}
		for len(s) > 0 {
			sinkI = s[0]
			s = s[1:]
		}
	})
	tHead := nsPerOp(func() {
		var r Queue
		for i := 0; i < burst; i++ {
			r.Enqueue(i)
		}
		for {
			v, ok := r.Dequeue()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	fmt.Printf("  %-30s %14.0f ns\n", "naive q = q[1:]", tNaive)
	fmt.Printf("  %-30s %14.0f ns  %.2fx\n", "head index + compaction", tHead, tHead/tNaive)
	fmt.Println()
	fmt.Println("  slower, and that is the honest result: draining once is exactly the")
	fmt.Println("  case the naive version handles well, and compaction is pure extra")
	fmt.Println("  work. What you bought is BOUNDED MEMORY on the cases that leak")
	fmt.Println("  (example 1, B and C), not speed.")
	fmt.Println()
	fmt.Println("  for speed AND bounded memory with no copying at all, the live region")
	fmt.Println("  has to stop moving -- which is the ring buffer, example 3.")
}
```

**Sample output:**

```
a head index instead of reslicing:

  enqueue 1..4   buf=[1 2 3 4] head=0 live=[1 2 3 4]
  dequeue -> 1   buf=[0 2 3 4] head=1 live=[2 3 4]
  dequeue -> 2   buf=[0 0 3 4] head=2 live=[3 4]
  dequeue -> 3   buf=[4] head=0 live=[4]

  the first two dequeues just advance head and zero the vacated slot.
  The THIRD crosses the halfway mark (head=3 > 4/2), so it compacts:
  the live tail is copied back to index 0 and the buffer shortened,
  which is why the last line reads buf=[4] head=0.

  nothing is abandoned, so nothing leaks -- and the queue is still a
  plain contiguous slice you can print, range over and index.

the amortized argument, counted:

    operations    compactions  elements copied   copies/operation
          1000              0                0               0.00
         10000              9             9000               0.45
        100000             99            99000               0.49
       1000000            999           999000               0.50

  copies per operation settles at a constant, so this is amortized
  Theta(1) by exactly the aggregate argument from lesson 03: each
  compaction can only happen after enough dequeues to make half the
  buffer dead, and those dequeues pay for it.

the threshold is a real knob, and it is the same trade as lesson 06's
shrink hysteresis -- compact eagerly and you copy constantly:

  dead space tolerated            compactions    elements copied   copies/operation
  0% (compact every dequeue)           100000          100000000             500.00
  25%                                     299             299000               1.50
  50%                                      99              99000               0.49
  75%                                      33              33000               0.17

  compacting on every dequeue copies the whole live region every
  time -- 1000 elements per operation here, and Theta(n) per
  operation in general, which makes the queue quadratic.

  tolerating more dead space copies less and wastes more. It is the
  same dial as lesson 06's shrink hysteresis, and 50% is the same
  default for the same reason: it is the point where the copies you
  pay for are covered by the dequeues that made them necessary.

versus the naive version, on a bursty workload:

  naive q = q[1:]                        484158 ns
  head index + compaction               1123820 ns  2.32x

  slower, and that is the honest result: draining once is exactly the
  case the naive version handles well, and compaction is pure extra
  work. What you bought is BOUNDED MEMORY on the cases that leak
  (example 1, B and C), not speed.

  for speed AND bounded memory with no copying at all, the live region
  has to stop moving -- which is the ring buffer, example 3.
```

**Rust:**

```rust
use std::collections::VecDeque;
use std::hint::black_box;
use std::time::{Duration, Instant};

// Fix 1: stop reslicing. Keep a HEAD INDEX into a Vec you own, and copy the
// live region back to the front when the wasted prefix gets too big.
//
// This keeps the mental model of a slice -- contiguous, indexable, printable --
// and puts YOU in charge of when the Theta(n) copy happens, instead of leaving
// it to push (Go) or to remove(0) (Rust, where it happens on every dequeue).

struct Queue {
    buf: Vec<i64>,
    head: usize, // index of the front element; buf[..head] is dead space

    compactions: usize,
    copied: usize,
}

impl Queue {
    fn new() -> Self {
        Queue {
            buf: Vec::new(),
            head: 0,
            compactions: 0,
            copied: 0,
        }
    }

    fn enqueue(&mut self, v: i64) {
        self.buf.push(v);
    }

    /// Dequeue advances the head and compacts when more than half the buffer
    /// is dead. The RATIO is the whole design decision -- see below.
    fn dequeue(&mut self) -> Option<i64> {
        // `get` is the Option-returning index: None when the queue is empty.
        let v = *self.buf.get(self.head)?;
        // Go zeroes the vacated slot so the GC can free whatever it pointed at.
        // For an i64 there is nothing to release; the write is kept only so the
        // printed buffer matches Go's. A generic Queue<T> cannot do this: it
        // would store Option<T> and `take()` the slot, which moves the value out.
        self.buf[self.head] = 0;
        self.head += 1;

        if self.head > self.buf.len() / 2 {
            self.compact();
        }
        Some(v)
    }

    fn compact(&mut self) {
        let live = self.buf.len() - self.head;
        // drain(..head) removes the dead prefix and shifts the live tail down:
        // one memmove, and the Vec's length shrinks with it, so there is no
        // separate "zero the leftovers" step as in Go.
        self.buf.drain(..self.head);
        self.head = 0;

        self.compactions += 1;
        self.copied += live;
    }

    /// the elements currently in the queue
    fn live(&self) -> &[i64] {
        &self.buf[self.head..]
    }
}

// --- a version with the compaction threshold as a knob ---

/// Compact once the dead prefix exceeds dead_pct of the buffer.
/// dead_pct == 0 means compact on EVERY dequeue.
struct Tunable {
    buf: Vec<i64>,
    head: usize,
    dead_pct: usize,
    compactions: usize,
    copied: usize,
}

impl Tunable {
    fn new(dead_pct: usize) -> Self {
        Tunable {
            buf: Vec::new(),
            head: 0,
            dead_pct,
            compactions: 0,
            copied: 0,
        }
    }

    fn enqueue(&mut self, v: i64) {
        self.buf.push(v);
    }

    fn dequeue(&mut self) -> Option<i64> {
        let v = *self.buf.get(self.head)?;
        self.head += 1;
        if self.head * 100 > self.buf.len() * self.dead_pct {
            let live = self.buf.len() - self.head;
            self.buf.drain(..self.head);
            self.head = 0;
            self.compactions += 1;
            self.copied += live;
        }
        Some(v)
    }
}

/// Go's `%v` of a slice: [1 2 3]
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
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
    let mut q = Queue::new();
    println!("a head index instead of reslicing:");
    println!();
    for v in [1, 2, 3, 4] {
        q.enqueue(v);
    }
    println!(
        "  enqueue 1..4   buf={} head={} live={}",
        show(&q.buf),
        q.head,
        show(q.live())
    );
    for _ in 0..3 {
        let v = q.dequeue().unwrap();
        println!(
            "  dequeue -> {v}   buf={} head={} live={}",
            show(&q.buf),
            q.head,
            show(q.live())
        );
    }
    println!();
    println!("  the first two dequeues just advance head and zero the vacated slot.");
    println!("  The THIRD crosses the halfway mark (head=3 > 4/2), so it compacts:");
    println!("  the live tail is copied back to index 0 and the buffer shortened,");
    println!("  which is why the last line reads buf=[4] head=0.");
    println!();
    println!("  nothing is abandoned, so nothing leaks -- and the queue is still a");
    println!("  plain contiguous slice you can print, iterate and index.");

    println!();
    println!("the amortized argument, counted:");
    println!();
    println!(
        "  {:>12} {:>14} {:>16} {:>18}",
        "operations", "compactions", "elements copied", "copies/operation"
    );
    for n in [1_000, 10_000, 100_000, 1_000_000] {
        let mut r = Queue::new();
        for i in 0..1000 {
            // a standing backlog of 1000
            r.enqueue(i);
        }
        for i in 0..n {
            r.enqueue(i);
            r.dequeue();
        }
        println!(
            "  {:>12} {:>14} {:>16} {:>18.2}",
            n,
            r.compactions,
            r.copied,
            r.copied as f64 / (2 * n) as f64
        );
    }
    println!();
    println!("  copies per operation settles at a constant, so this is amortized");
    println!("  Theta(1) by exactly the aggregate argument from lesson 03: each");
    println!("  compaction can only happen after enough dequeues to make half the");
    println!("  buffer dead, and those dequeues pay for it.");

    println!();
    println!("the threshold is a real knob, and it is the same trade as lesson 06's");
    println!("shrink hysteresis -- compact eagerly and you copy constantly:");
    println!();
    println!(
        "  {:<28} {:>14} {:>18} {:>18}",
        "dead space tolerated", "compactions", "elements copied", "copies/operation"
    );
    for pct in [0, 25, 50, 75] {
        let mut t = Tunable::new(pct);
        for i in 0..1000 {
            t.enqueue(i);
        }
        for i in 0..100_000 {
            t.enqueue(i);
            t.dequeue();
        }
        let label = if pct == 0 {
            "0% (compact every dequeue)".to_string()
        } else {
            format!("{pct}%")
        };
        println!(
            "  {:<28} {:>14} {:>18} {:>18.2}",
            label,
            t.compactions,
            t.copied,
            t.copied as f64 / 200_000.0
        );
    }
    println!();
    println!("  compacting on every dequeue copies the whole live region every");
    println!("  time -- 1000 elements per operation here, and Theta(n) per");
    println!("  operation in general, which makes the queue quadratic.");
    println!();
    println!("  tolerating more dead space copies less and wastes more. It is the");
    println!("  same dial as lesson 06's shrink hysteresis, and 50% is the same");
    println!("  default for the same reason: it is the point where the copies you");
    println!("  pay for are covered by the dequeues that made them necessary.");
    println!();
    println!("  (the 0% row does the same shifting as Vec::remove(0): Rust's naive");
    println!("  queue is this table's worst row, built into the standard library.)");

    println!();
    println!("versus the naive versions, on a bursty workload:");
    println!();
    const BURST: i64 = 200_000;
    // Go's `s = s[1:]` as a Rust view: split_first peels the front off a
    // borrowed slice. It can only drain, never push, so it is the fair
    // comparison only for "fill once, drain once".
    let t_view = ns_per_op(|| {
        let mut v = Vec::new();
        for i in 0..BURST {
            v.push(i);
        }
        let mut s: &[i64] = &v;
        while let Some((&first, rest)) = s.split_first() {
            black_box(first);
            s = rest;
        }
    });
    // The naive queue Rust lets you write on an owned Vec.
    let t_remove = ns_per_op(|| {
        let mut v = Vec::new();
        for i in 0..BURST {
            v.push(i);
        }
        while !v.is_empty() {
            black_box(v.remove(0));
        }
    });
    let t_head = ns_per_op(|| {
        let mut r = Queue::new();
        for i in 0..BURST {
            r.enqueue(i);
        }
        while let Some(v) = r.dequeue() {
            black_box(v);
        }
    });
    let t_deque = ns_per_op(|| {
        let mut d = VecDeque::new();
        for i in 0..BURST {
            d.push_back(i);
        }
        while let Some(v) = d.pop_front() {
            black_box(v);
        }
    });
    println!("  {:<30} {:>14.0} ns", "naive &q[1..] (view)", t_view);
    println!(
        "  {:<30} {:>14.0} ns  {:.2}x",
        "naive q.remove(0)",
        t_remove,
        t_remove / t_view
    );
    println!(
        "  {:<30} {:>14.0} ns  {:.2}x",
        "head index + compaction",
        t_head,
        t_head / t_view
    );
    println!(
        "  {:<30} {:>14.0} ns  {:.2}x",
        "std VecDeque",
        t_deque,
        t_deque / t_view
    );
    println!();
    // one untimed pass to count what the compaction actually copied
    let mut r = Queue::new();
    for i in 0..BURST {
        r.enqueue(i);
    }
    while r.dequeue().is_some() {}
    println!(
        "  the head index compacted {} times and copied {} elements in total, for {} dequeues:",
        r.compactions, r.copied, BURST
    );
    println!("  each compaction copies roughly half of what the last one did, so the copying");
    println!("  is a geometric series that sums to about the size of the burst.");
    println!();
    println!(
        "  remove(0) is the catastrophe: {:.0}x the view, because it shifts the whole",
        t_remove / t_view
    );
    println!(
        "  live region on every dequeue -- n(n-1)/2 = {} element moves in total,",
        BURST * (BURST - 1) / 2
    );
    println!("  quadratic. That is about four orders of magnitude here, and it is the");
    println!("  price of the naive queue Rust lets you write.");
    println!();
    println!("  the three queues that do not shift -- view, head index, VecDeque --");
    println!("  stay within a small factor of each other. Across repeated runs on a");
    println!("  shared machine their ratios to the view ranged from about 0.9x to 3x");
    println!("  and the head index and VecDeque swapped places, so read nothing into");
    println!("  the order. What they buy over the view is BOUNDED MEMORY and the");
    println!("  ability to push while draining (example 1, B and C); a drain-once");
    println!("  burst is the one case where the view needs neither, and compaction");
    println!("  or wrap-around is extra work.");
    println!();
    println!("  for speed AND bounded memory with no copying at all, the live region");
    println!("  has to stop moving -- which is the ring buffer, example 3.");
}
```

**Rust sample output:**

```
a head index instead of reslicing:

  enqueue 1..4   buf=[1 2 3 4] head=0 live=[1 2 3 4]
  dequeue -> 1   buf=[0 2 3 4] head=1 live=[2 3 4]
  dequeue -> 2   buf=[0 0 3 4] head=2 live=[3 4]
  dequeue -> 3   buf=[4] head=0 live=[4]

  the first two dequeues just advance head and zero the vacated slot.
  The THIRD crosses the halfway mark (head=3 > 4/2), so it compacts:
  the live tail is copied back to index 0 and the buffer shortened,
  which is why the last line reads buf=[4] head=0.

  nothing is abandoned, so nothing leaks -- and the queue is still a
  plain contiguous slice you can print, iterate and index.

the amortized argument, counted:

    operations    compactions  elements copied   copies/operation
          1000              0                0               0.00
         10000              9             9000               0.45
        100000             99            99000               0.49
       1000000            999           999000               0.50

  copies per operation settles at a constant, so this is amortized
  Theta(1) by exactly the aggregate argument from lesson 03: each
  compaction can only happen after enough dequeues to make half the
  buffer dead, and those dequeues pay for it.

the threshold is a real knob, and it is the same trade as lesson 06's
shrink hysteresis -- compact eagerly and you copy constantly:

  dead space tolerated            compactions    elements copied   copies/operation
  0% (compact every dequeue)           100000          100000000             500.00
  25%                                     299             299000               1.50
  50%                                      99              99000               0.49
  75%                                      33              33000               0.17

  compacting on every dequeue copies the whole live region every
  time -- 1000 elements per operation here, and Theta(n) per
  operation in general, which makes the queue quadratic.

  tolerating more dead space copies less and wastes more. It is the
  same dial as lesson 06's shrink hysteresis, and 50% is the same
  default for the same reason: it is the point where the copies you
  pay for are covered by the dequeues that made them necessary.

  (the 0% row does the same shifting as Vec::remove(0): Rust's naive
  queue is this table's worst row, built into the standard library.)

versus the naive versions, on a bursty workload:

  naive &q[1..] (view)                   228533 ns
  naive q.remove(0)                  2818534833 ns  12333.19x
  head index + compaction                582164 ns  2.55x
  std VecDeque                           446078 ns  1.95x

  the head index compacted 17 times and copied 199977 elements in total, for 200000 dequeues:
  each compaction copies roughly half of what the last one did, so the copying
  is a geometric series that sums to about the size of the burst.

  remove(0) is the catastrophe: 12333x the view, because it shifts the whole
  live region on every dequeue -- n(n-1)/2 = 19999900000 element moves in total,
  quadratic. That is about four orders of magnitude here, and it is the
  price of the naive queue Rust lets you write.

  the three queues that do not shift -- view, head index, VecDeque --
  stay within a small factor of each other. Across repeated runs on a
  shared machine their ratios to the view ranged from about 0.9x to 3x
  and the head index and VecDeque swapped places, so read nothing into
  the order. What they buy over the view is BOUNDED MEMORY and the
  ability to push while draining (example 1, B and C); a drain-once
  burst is the one case where the view needs neither, and compaction
  or wrap-around is extra work.

  for speed AND bounded memory with no copying at all, the live region
  has to stop moving -- which is the ring buffer, example 3.
```

> **Go vs Rust:** The head-index tables are identical to Go's (0.00 / 0.45 / 0.49 / 0.50 copies per operation, and 500.00 / 1.50 / 0.49 / 0.17 across thresholds), since they are pure integer logic; `drain(..head)` stands in for Go's `copy` + `clear` + reslice. The benchmark does not carry over: Rust's owned naive queue is `Vec::remove(0)`, about four orders of magnitude slower than draining a borrowed view (seconds versus a fraction of a millisecond), and the head index was 0.9x to 3x of the view across my runs where Go measured 2.32x.

**Complexity:** amortized Θ(1) per operation at the 50% threshold — **0.49 copies per operation
measured** · compacting on every dequeue is Θ(n) per operation and makes the queue quadratic (500
copies/op measured)

---

## 3. Fix two: the ring buffer

`🟢 easy` · *stop moving the data*

The live region still travels — but around a circle, so it never runs off the end and never has to
be copied back. Three fields, Θ(1) worst case, zero allocations. This is the structure inside every
bounded work queue you have ever used.

**Steps:**

1. Build it with `head`, `count` and a derived `tail`, and watch the wrap happen.
2. Meet `Full` as a real state that the caller must be told about.
3. Benchmark it — and find out why the capacity must be a power of two.

```go
package main

import (
	"fmt"
	"testing"
)

// Fix 2, and the one that actually solves the problem: stop moving the data.
//
// A RING BUFFER is a fixed slice plus two indices that WRAP. The live region
// still travels, but it travels around a circle instead of marching off the end,
// so it never needs to be copied back and the array never grows.
//
//	head  where the next Dequeue takes from
//	tail  where the next Enqueue writes
//	count how many are live -- needed because head == tail is ambiguous
//	      (it means both "empty" and "full")
//
// Every operation is Theta(1) WORST CASE, not amortized. No copying, no
// reallocation, no GC pressure, and a hard memory bound you chose.

type Ring struct {
	buf         []int
	head, count int
}

func NewRing(capacity int) *Ring {
	return &Ring{buf: make([]int, capacity)}
}

func (r *Ring) Len() int   { return r.count }
func (r *Ring) Cap() int   { return len(r.buf) }
func (r *Ring) Full() bool { return r.count == len(r.buf) }

// tail is derived rather than stored -- one less thing to keep in sync.
func (r *Ring) tail() int { return (r.head + r.count) % len(r.buf) }

func (r *Ring) Enqueue(v int) bool {
	if r.Full() {
		return false // the caller decides what to do -- example 9
	}
	r.buf[r.tail()] = v
	r.count++
	return true
}

func (r *Ring) Dequeue() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	v := r.buf[r.head]
	r.buf[r.head] = 0 // release any pointer inside T
	r.head = (r.head + 1) % len(r.buf)
	r.count--
	return v, true
}

func (r *Ring) Peek() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	return r.buf[r.head], true
}

// At indexes logically: At(0) is the front. The modulo is the only difference
// from a slice.
func (r *Ring) At(i int) (int, bool) {
	if i < 0 || i >= r.count {
		return 0, false
	}
	return r.buf[(r.head+i)%len(r.buf)], true
}

// render draws the physical array with markers, so the wrap is visible.
func (r *Ring) render() string {
	s := "["
	for i := range r.buf {
		mark := " "
		switch {
		case r.count > 0 && i == r.head:
			mark = "H"
		case r.count > 0 && i == r.tail():
			mark = "t"
		}
		live := false
		for j := 0; j < r.count; j++ {
			if (r.head+j)%len(r.buf) == i {
				live = true
				break
			}
		}
		if live {
			s += fmt.Sprintf("%s%d ", mark, r.buf[i])
		} else {
			s += fmt.Sprintf("%s. ", mark)
		}
	}
	return s + "]"
}

// RingPow2 is the same structure with one constraint: the capacity is rounded
// up to a power of two. That turns every `% cap` into `& (cap-1)` -- a bitwise
// AND instead of an integer division. Example 3's benchmark shows what that
// single change is worth.
type RingPow2 struct {
	buf         []int
	mask        int
	head, count int
}

func NewRingPow2(capacity int) *RingPow2 {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &RingPow2{buf: make([]int, n), mask: n - 1}
}

func (r *RingPow2) Enqueue(v int) bool {
	if r.count == len(r.buf) {
		return false
	}
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
	return true
}

func (r *RingPow2) Dequeue() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	v := r.buf[r.head]
	r.buf[r.head] = 0
	r.head = (r.head + 1) & r.mask
	r.count--
	return v, true
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
	r := NewRing(5)
	fmt.Println("H marks head, t marks the next write slot, . is a free slot:")
	fmt.Println()
	fmt.Printf("  %-22s %s\n", "empty", r.render())

	for _, v := range []int{1, 2, 3} {
		r.Enqueue(v)
		fmt.Printf("  %-22s %s\n", fmt.Sprintf("enqueue %d", v), r.render())
	}
	for i := 0; i < 2; i++ {
		v, _ := r.Dequeue()
		fmt.Printf("  %-22s %s\n", fmt.Sprintf("dequeue -> %d", v), r.render())
	}
	for _, v := range []int{4, 5, 6} {
		r.Enqueue(v)
		fmt.Printf("  %-22s %s\n", fmt.Sprintf("enqueue %d", v), r.render())
	}

	fmt.Println()
	fmt.Println("  look at the last two lines: the tail ran off the end of the array")
	fmt.Println("  and reappeared at index 0. That is the whole idea -- the live")
	fmt.Println("  region WRAPS instead of marching, so it never has to be copied.")

	fmt.Println()
	fmt.Printf("  logical order is still available: ")
	for i := 0; i < r.Len(); i++ {
		v, _ := r.At(i)
		fmt.Printf("%d ", v)
	}
	fmt.Printf("  (At(i) = buf[(head+i)%%cap])\n")

	fmt.Println()
	fmt.Println("full is a real state, and the caller has to be told:")
	fmt.Println()
	fmt.Printf("  Len=%d Cap=%d Full=%v -> Enqueue(7) = %v\n",
		r.Len(), r.Cap(), r.Full(), r.Enqueue(7))
	fmt.Printf("  Len=%d Cap=%d Full=%v -> Enqueue(8) = %v   <- refused\n",
		r.Len(), r.Cap(), r.Full(), r.Enqueue(8))
	fmt.Printf("  physical array is now %s -- not one free slot\n", r.render())
	fmt.Println()
	fmt.Println("  that `bool` is the price of a bounded structure, and it is a")
	fmt.Println("  FEATURE: a queue that cannot refuse work has no backpressure, and")
	fmt.Println("  a queue with no backpressure is a memory leak with extra steps.")
	fmt.Println("  Example 9 is entirely about what to do with that false.")

	fmt.Println()
	fmt.Println("why `count` and not just head and tail:")
	fmt.Println()
	fmt.Println("  with only two indices, head == tail means the ring is either")
	fmt.Println("  completely EMPTY or completely FULL, and nothing distinguishes")
	fmt.Println("  them. The three classic fixes:")
	fmt.Println()
	fmt.Println("    keep a count            <- what this does. Clearest.")
	fmt.Printf("    %-24s%s\n", "waste one slot", "full is (tail+1)%cap == head")
	fmt.Printf("    %-24s%s\n", "keep monotonic counters", "head/tail only grow; index with %cap")
	fmt.Println()
	fmt.Println("  the third is what Go's runtime uses for channels, because two")
	fmt.Println("  ever-increasing integers are easier to update atomically.")

	fmt.Println()
	fmt.Println("the benchmark, and it does not say what you expect:")
	fmt.Println()
	const ops = 1_000_000
	tNaive := nsPerOp(func() {
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
	tRing := nsPerOp(func() {
		rr := NewRing(2000)
		for i := 0; i < 1000; i++ {
			rr.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			rr.Enqueue(i)
			v, _ := rr.Dequeue()
			sinkI = v
		}
	})
	tMask := nsPerOp(func() {
		rr := NewRingPow2(2000)
		for i := 0; i < 1000; i++ {
			rr.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			rr.Enqueue(i)
			v, _ := rr.Dequeue()
			sinkI = v
		}
	})
	fmt.Printf("  %-34s %10.2f ns/op\n", "naive q = q[1:]", tNaive/ops)
	fmt.Printf("  %-34s %10.2f ns/op   %.2fx\n", "ring, capacity 2000 (uses %)", tRing/ops, tNaive/tRing)
	fmt.Printf("  %-34s %10.2f ns/op   %.2fx\n", "ring, capacity 2048 (uses &)", tMask/ops, tNaive/tMask)

	fmt.Println()
	fmt.Println("  BOTH rings lose. I expected the ring to win and it does not, so")
	fmt.Println("  here is what the two numbers actually say.")
	fmt.Println()
	fmt.Printf("  `%% 2000` is an integer division -- one of the slowest instructions\n")
	fmt.Println("  the CPU has -- and there are two per operation. Rounding the")
	fmt.Printf("  capacity up to 2048 turns each %% into an &, and that ONE LINE in\n")
	fmt.Printf("  the constructor recovers %.2f ns/op, most of the gap. This is why\n", tRing/ops-tMask/ops)
	fmt.Println("  every real ring buffer -- Go's own channel buffers, LMAX Disruptor,")
	fmt.Println("  io_uring -- is power-of-two sized. It is not optional.")
	fmt.Println()
	fmt.Printf("  the %.2f ns/op that remains is the method calls: two non-inlined\n", tMask/ops-tNaive/ops)
	fmt.Println("  calls per operation, which is exactly the cost lesson 08 measured")
	fmt.Println("  for the generic Stack wrapper. `q = q[1:]` is a pointer increment")
	fmt.Println("  the compiler can see through; Enqueue/Dequeue are not.")
	fmt.Println()
	fmt.Println("  so the naive slice queue is the FASTEST queue in this lesson. That")
	fmt.Println("  has been true in all three examples now, and it is the honest")
	fmt.Println("  frame: its problem was never speed.")

	fmt.Println()
	fmt.Println("its problem is this:")
	fmt.Println()
	allocNaive := testing.AllocsPerRun(5, func() {
		q := make([]int, 0, 1000)
		for i := 0; i < 100_000; i++ {
			q = append(q, i)
			q = q[1:]
		}
		sinkI = len(q)
	})
	rr := NewRingPow2(2048)
	allocRing := testing.AllocsPerRun(5, func() {
		for i := 0; i < 100_000; i++ {
			rr.Enqueue(i)
			v, _ := rr.Dequeue()
			sinkI = v
		}
	})
	fmt.Printf("  %-34s %10.0f allocations per 100,000 ops\n", "naive q = q[1:]", allocNaive)
	fmt.Printf("  %-34s %10.0f allocations per 100,000 ops\n", "ring buffer", allocRing)

	fmt.Println()
	fmt.Println("  ZERO against 98,996. The ring never allocates after the")
	fmt.Println("  constructor, its memory is bounded by a number YOU chose, and")
	fmt.Println("  every operation is Theta(1) WORST CASE -- not amortized, so no")
	fmt.Println("  single enqueue can ever stall on a 4 MB copy the way append can.")
	fmt.Println()
	fmt.Println("  that combination -- bounded, predictable, allocation-free -- is why")
	fmt.Println("  every audio buffer, network buffer, log ring and bounded work queue")
	fmt.Println("  in production is this structure. The cost is the fixed capacity,")
	fmt.Println("  which example 4 removes and example 9 turns into a feature.")
}
```

**Sample output:**

```
H marks head, t marks the next write slot, . is a free slot:

  empty                  [ .  .  .  .  . ]
  enqueue 1              [H1 t.  .  .  . ]
  enqueue 2              [H1  2 t.  .  . ]
  enqueue 3              [H1  2  3 t.  . ]
  dequeue -> 1           [ . H2  3 t.  . ]
  dequeue -> 2           [ .  . H3 t.  . ]
  enqueue 4              [ .  . H3  4 t. ]
  enqueue 5              [t.  . H3  4  5 ]
  enqueue 6              [ 6 t. H3  4  5 ]

  look at the last two lines: the tail ran off the end of the array
  and reappeared at index 0. That is the whole idea -- the live
  region WRAPS instead of marching, so it never has to be copied.

  logical order is still available: 3 4 5 6   (At(i) = buf[(head+i)%cap])

full is a real state, and the caller has to be told:

  Len=4 Cap=5 Full=false -> Enqueue(7) = true
  Len=5 Cap=5 Full=true -> Enqueue(8) = false   <- refused
  physical array is now [ 6  7 H3  4  5 ] -- not one free slot

  that `bool` is the price of a bounded structure, and it is a
  FEATURE: a queue that cannot refuse work has no backpressure, and
  a queue with no backpressure is a memory leak with extra steps.
  Example 9 is entirely about what to do with that false.

why `count` and not just head and tail:

  with only two indices, head == tail means the ring is either
  completely EMPTY or completely FULL, and nothing distinguishes
  them. The three classic fixes:

    keep a count            <- what this does. Clearest.
    waste one slot          full is (tail+1)%cap == head
    keep monotonic counters head/tail only grow; index with %cap

  the third is what Go's runtime uses for channels, because two
  ever-increasing integers are easier to update atomically.

the benchmark, and it does not say what you expect:

  naive q = q[1:]                          1.76 ns/op
  ring, capacity 2000 (uses %)             4.60 ns/op   0.38x
  ring, capacity 2048 (uses &)             2.21 ns/op   0.80x

  BOTH rings lose. I expected the ring to win and it does not, so
  here is what the two numbers actually say.

  `% 2000` is an integer division -- one of the slowest instructions
  the CPU has -- and there are two per operation. Rounding the
  capacity up to 2048 turns each % into an &, and that ONE LINE in
  the constructor recovers 2.39 ns/op, most of the gap. This is why
  every real ring buffer -- Go's own channel buffers, LMAX Disruptor,
  io_uring -- is power-of-two sized. It is not optional.

  the 0.44 ns/op that remains is the method calls: two non-inlined
  calls per operation, which is exactly the cost lesson 08 measured
  for the generic Stack wrapper. `q = q[1:]` is a pointer increment
  the compiler can see through; Enqueue/Dequeue are not.

  so the naive slice queue is the FASTEST queue in this lesson. That
  has been true in all three examples now, and it is the honest
  frame: its problem was never speed.

its problem is this:

  naive q = q[1:]                         98996 allocations per 100,000 ops
  ring buffer                                 0 allocations per 100,000 ops

  ZERO against 98,996. The ring never allocates after the
  constructor, its memory is bounded by a number YOU chose, and
  every operation is Theta(1) WORST CASE -- not amortized, so no
  single enqueue can ever stall on a 4 MB copy the way append can.

  that combination -- bounded, predictable, allocation-free -- is why
  every audio buffer, network buffer, log ring and bounded work queue
  in production is this structure. The cost is the fixed capacity,
  which example 4 removes and example 9 turns into a feature.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::VecDeque;
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
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

// Fix 2, and the one that actually solves the problem: stop moving the data.
//
// A RING BUFFER is a fixed array plus indices that WRAP. The live region
// still travels, but it travels around a circle instead of marching off the end,
// so it never needs to be copied back and the array never grows.
//
//	head  where the next dequeue takes from
//	tail  where the next enqueue writes
//	count how many are live -- needed because head == tail is ambiguous
//	      (it means both "empty" and "full")
//
// Every operation is Theta(1) WORST CASE, not amortized. No copying, no
// reallocation, and a hard memory bound you chose.

struct Ring {
    buf: Vec<i64>,
    head: usize,
    count: usize,
}

impl Ring {
    fn new(capacity: usize) -> Self {
        Ring {
            buf: vec![0; capacity],
            head: 0,
            count: 0,
        }
    }

    fn len(&self) -> usize {
        self.count
    }
    fn cap(&self) -> usize {
        self.buf.len()
    }
    fn full(&self) -> bool {
        self.count == self.buf.len()
    }

    /// tail is derived rather than stored -- one less thing to keep in sync.
    fn tail(&self) -> usize {
        (self.head + self.count) % self.buf.len()
    }

    /// A full ring refuses the value and hands it BACK (Err(v)): in Rust the
    /// refusal carries ownership home, so a non-Copy element is not lost.
    /// The caller decides what to do -- example 9.
    fn enqueue(&mut self, v: i64) -> Result<(), i64> {
        if self.full() {
            return Err(v);
        }
        let t = self.tail();
        self.buf[t] = v;
        self.count += 1;
        Ok(())
    }

    fn dequeue(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        // Go zeroes the slot so the GC can free what it pointed at. For an i64
        // there is nothing to release; kept so both versions do the same work.
        // A generic Ring<T> would store Option<T> and take() the slot.
        self.buf[self.head] = 0;
        self.head = (self.head + 1) % self.buf.len();
        self.count -= 1;
        Some(v)
    }

    /// Indexes logically: at(0) is the front (and so doubles as peek). The
    /// modulo is the only difference from a slice.
    fn at(&self, i: usize) -> Option<i64> {
        if i >= self.count {
            return None;
        }
        Some(self.buf[(self.head + i) % self.buf.len()])
    }

    /// render draws the physical array with markers, so the wrap is visible.
    fn render(&self) -> String {
        let mut s = String::from("[");
        for i in 0..self.buf.len() {
            let mark = if self.count > 0 && i == self.head {
                "H"
            } else if self.count > 0 && i == self.tail() {
                "t"
            } else {
                " "
            };
            let live = (0..self.count).any(|j| (self.head + j) % self.buf.len() == i);
            if live {
                s += &format!("{mark}{} ", self.buf[i]);
            } else {
                s += &format!("{mark}. ");
            }
        }
        s + "]"
    }
}

/// RingPow2 is the same structure with one constraint: the capacity is rounded
/// up to a power of two. That turns every `% cap` into `& (cap-1)` -- a bitwise
/// AND instead of an integer division. The benchmark shows what that single
/// change is worth.
struct RingPow2 {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
}

impl RingPow2 {
    fn new(capacity: usize) -> Self {
        // Go loops `n <<= 1`; Rust has it built in.
        let n = capacity.next_power_of_two();
        RingPow2 {
            buf: vec![0; n],
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }

    fn enqueue(&mut self, v: i64) -> Result<(), i64> {
        if self.count == self.buf.len() {
            return Err(v);
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
        Ok(())
    }

    fn dequeue(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        self.buf[self.head] = 0;
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
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
    let mut r = Ring::new(5);
    println!("H marks head, t marks the next write slot, . is a free slot:");
    println!();
    println!("  {:<22} {}", "empty", r.render());

    for v in [1, 2, 3] {
        r.enqueue(v).unwrap();
        println!("  {:<22} {}", format!("enqueue {v}"), r.render());
    }
    for _ in 0..2 {
        let v = r.dequeue().unwrap();
        println!("  {:<22} {}", format!("dequeue -> {v}"), r.render());
    }
    for v in [4, 5, 6] {
        r.enqueue(v).unwrap();
        println!("  {:<22} {}", format!("enqueue {v}"), r.render());
    }

    println!();
    println!("  look at the last two lines: the tail ran off the end of the array");
    println!("  and reappeared at index 0. That is the whole idea -- the live");
    println!("  region WRAPS instead of marching, so it never has to be copied.");

    println!();
    print!("  logical order is still available: ");
    for i in 0..r.len() {
        print!("{} ", r.at(i).unwrap());
    }
    println!("  (at(i) = buf[(head+i)%cap])");

    println!();
    println!("full is a real state, and the caller has to be told:");
    println!();
    let (len, cap, full) = (r.len(), r.cap(), r.full());
    let ok = r.enqueue(7);
    println!("  Len={len} Cap={cap} Full={full} -> enqueue(7) = {ok:?}");
    let (len, cap, full) = (r.len(), r.cap(), r.full());
    let refused = r.enqueue(8);
    println!(
        "  Len={len} Cap={cap} Full={full} -> enqueue(8) = {refused:?}   <- refused, value handed back"
    );
    println!(
        "  physical array is now {} -- not one free slot",
        r.render()
    );
    println!();
    println!("  that Err is the price of a bounded structure, and it is a");
    println!("  FEATURE: a queue that cannot refuse work has no backpressure, and");
    println!("  a queue with no backpressure is a memory leak with extra steps.");
    println!("  Example 9 is entirely about what to do with that Err.");

    println!();
    println!("why `count` and not just head and tail:");
    println!();
    println!("  with only two indices, head == tail means the ring is either");
    println!("  completely EMPTY or completely FULL, and nothing distinguishes");
    println!("  them. The three classic fixes:");
    println!();
    println!("    keep a count            <- what this does. Clearest.");
    println!("    waste one slot          full is (tail+1)%cap == head");
    println!("    keep monotonic counters head/tail only grow; index with %cap");
    println!();
    println!("  the third is popular in lock-free rings, because two ever-increasing");
    println!("  integers are easier to update atomically (this program is single-");
    println!("  threaded and does not measure that).");

    println!();
    println!("the benchmark:");
    println!();
    const OPS: usize = 1_000_000;
    // black_box on the capacity: with a literal 2000 LLVM sees the capacity all
    // the way through the constructor, turns `% 2000` into a multiply (umulh in
    // the assembly) and the mask into a constant, and the benchmark measures
    // constant folding. Real capacities come from configuration; this keeps it so.
    let t_naive = ns_per_op(|| {
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
    let t_ring = ns_per_op(|| {
        let mut rr = Ring::new(black_box(2000));
        for i in 0..1000 {
            let _ = rr.enqueue(i);
        }
        for i in 0..OPS as i64 {
            let _ = rr.enqueue(i);
            black_box(rr.dequeue());
        }
    });
    let t_mask = ns_per_op(|| {
        let mut rr = RingPow2::new(black_box(2000));
        for i in 0..1000 {
            let _ = rr.enqueue(i);
        }
        for i in 0..OPS as i64 {
            let _ = rr.enqueue(i);
            black_box(rr.dequeue());
        }
    });
    let t_deque = ns_per_op(|| {
        let mut d: VecDeque<i64> = VecDeque::with_capacity(black_box(2000));
        for i in 0..1000 {
            d.push_back(i);
        }
        for i in 0..OPS as i64 {
            d.push_back(i);
            black_box(d.pop_front());
        }
    });
    let ops = OPS as f64;
    println!(
        "  {:<34} {:>10.2} ns/op",
        "naive q.remove(0)",
        t_naive / ops
    );
    println!(
        "  {:<34} {:>10.2} ns/op   {:.2}x",
        "ring, capacity 2000 (uses %)",
        t_ring / ops,
        t_naive / t_ring
    );
    println!(
        "  {:<34} {:>10.2} ns/op   {:.2}x",
        "ring, capacity 2048 (uses &)",
        t_mask / ops,
        t_naive / t_mask
    );
    println!(
        "  {:<34} {:>10.2} ns/op   {:.2}x",
        "std VecDeque, capacity 2000",
        t_deque / ops,
        t_naive / t_deque
    );

    println!();
    println!(
        "  (std's VecDeque::with_capacity(2000).capacity() = {})",
        VecDeque::<i64>::with_capacity(2000).capacity()
    );
    println!();
    println!("  BOTH rings win, by about two orders of magnitude over the naive queue.");
    println!("  remove(0) shifts the whole 1000-element backlog on every dequeue; the");
    println!("  ring moves nothing, so its cost does not depend on how full it is.");
    println!();
    println!("  the `% 2000` vs `& 2047` gap is the surprise: the two rings were within");
    println!("  about 0.15 ns/op of each other in every run (one CPU, an Apple M4).");
    println!("  Reading the generated assembly (rustc 1.98, `-O --emit asm`): in dequeue,");
    println!("  `% cap` was compiled to compare-and-select, because the bounds check on");
    println!("  buf[head] already proves head < cap; a real `udiv` is left only in");
    println!("  enqueue. So one division per operation, not two. Plausible mechanism for");
    println!("  the small gap, not measured: nothing in the head-to-head chain waits on");
    println!("  that division. A CPU with a slower divide may show a bigger gap.");
    println!();
    println!("  and std's VecDeque is not power-of-two sized: asked for 2000, it holds");
    println!("  exactly 2000. It is slower than these hand-rolled rings, but it can also");
    println!("  GROW, which they refuse to do; this program does not measure where its");
    println!("  extra cost per operation comes from.");
    println!();
    println!("  so in Rust the naive queue is the SLOWEST queue in this lesson, not the");
    println!("  fastest: Go's `q = q[1:]` was a pointer increment, remove(0) moves the");
    println!("  whole backlog.");
    println!();
    println!("its problem is this:");
    println!();
    // The same workload for all three: construct, 1000 standing, then
    // 100,000 x (enqueue, dequeue). Construction is counted for every one.
    let mut shifted: u64 = 0;
    let (alloc_naive, ()) = allocs_of(|| {
        let mut q: Vec<i64> = Vec::with_capacity(1000);
        for i in 0..1000 {
            q.push(i);
        }
        for i in 0..100_000 {
            q.push(i);
            shifted += (q.len() - 1) as u64; // what remove(0) is about to move
            black_box(q.remove(0));
        }
    });
    let (alloc_ring, ()) = allocs_of(|| {
        let mut rr = RingPow2::new(1000);
        for i in 0..1000 {
            let _ = rr.enqueue(i);
        }
        for i in 0..100_000 {
            let _ = rr.enqueue(i);
            black_box(rr.dequeue());
        }
    });
    let (alloc_deque, ()) = allocs_of(|| {
        let mut d: VecDeque<i64> = VecDeque::with_capacity(2000);
        for i in 0..1000 {
            d.push_back(i);
        }
        for i in 0..100_000 {
            d.push_back(i);
            black_box(d.pop_front());
        }
    });
    println!(
        "  {:<34} {:>10} allocation events, {} elements shifted",
        "naive q.remove(0)", alloc_naive, shifted
    );
    println!(
        "  {:<34} {:>10} allocation events",
        "ring buffer", alloc_ring
    );
    println!(
        "  {:<34} {:>10} allocation events",
        "std VecDeque", alloc_deque
    );
    println!();
    println!("  the naive queue allocated twice -- the constructor, and one growth when");
    println!("  the 1001st element arrived -- so Go's 98,996 does not carry over. My");
    println!("  reading of that count: `q = q[1:]` shrinks the slice's capacity until");
    println!("  append must replace the array, and an owned Vec cannot be resliced.");
    println!("  What the naive Rust queue pays is the shift: 100,000,000 elements moved");
    println!("  for 100,000 operations, 1000 per dequeue, counted in the loop.");
    println!();
    println!("  the ring and the VecDeque each allocate once, in the constructor. The");
    println!("  ring's memory is bounded by a number YOU chose and every operation is");
    println!("  Theta(1) WORST CASE -- not amortized, so no single enqueue can stall on");
    println!("  a copy the way Vec::push or VecDeque::push_back can: the ring refuses");
    println!("  instead of growing.");
    println!();
    println!("  that combination -- bounded, predictable, allocation-free after the");
    println!("  constructor -- is why audio buffers, network buffers, log rings and");
    println!("  bounded work queues are built this way. The cost is the fixed capacity,");
    println!("  which example 4 removes and example 9 turns into a feature.");
}
```

**Rust sample output:**

```
H marks head, t marks the next write slot, . is a free slot:

  empty                  [ .  .  .  .  . ]
  enqueue 1              [H1 t.  .  .  . ]
  enqueue 2              [H1  2 t.  .  . ]
  enqueue 3              [H1  2  3 t.  . ]
  dequeue -> 1           [ . H2  3 t.  . ]
  dequeue -> 2           [ .  . H3 t.  . ]
  enqueue 4              [ .  . H3  4 t. ]
  enqueue 5              [t.  . H3  4  5 ]
  enqueue 6              [ 6 t. H3  4  5 ]

  look at the last two lines: the tail ran off the end of the array
  and reappeared at index 0. That is the whole idea -- the live
  region WRAPS instead of marching, so it never has to be copied.

  logical order is still available: 3 4 5 6   (at(i) = buf[(head+i)%cap])

full is a real state, and the caller has to be told:

  Len=4 Cap=5 Full=false -> enqueue(7) = Ok(())
  Len=5 Cap=5 Full=true -> enqueue(8) = Err(8)   <- refused, value handed back
  physical array is now [ 6  7 H3  4  5 ] -- not one free slot

  that Err is the price of a bounded structure, and it is a
  FEATURE: a queue that cannot refuse work has no backpressure, and
  a queue with no backpressure is a memory leak with extra steps.
  Example 9 is entirely about what to do with that Err.

why `count` and not just head and tail:

  with only two indices, head == tail means the ring is either
  completely EMPTY or completely FULL, and nothing distinguishes
  them. The three classic fixes:

    keep a count            <- what this does. Clearest.
    waste one slot          full is (tail+1)%cap == head
    keep monotonic counters head/tail only grow; index with %cap

  the third is popular in lock-free rings, because two ever-increasing
  integers are easier to update atomically (this program is single-
  threaded and does not measure that).

the benchmark:

  naive q.remove(0)                       73.86 ns/op
  ring, capacity 2000 (uses %)             0.82 ns/op   89.74x
  ring, capacity 2048 (uses &)             0.70 ns/op   105.14x
  std VecDeque, capacity 2000              2.75 ns/op   26.83x

  (std's VecDeque::with_capacity(2000).capacity() = 2000)

  BOTH rings win, by about two orders of magnitude over the naive queue.
  remove(0) shifts the whole 1000-element backlog on every dequeue; the
  ring moves nothing, so its cost does not depend on how full it is.

  the `% 2000` vs `& 2047` gap is the surprise: the two rings were within
  about 0.15 ns/op of each other in every run (one CPU, an Apple M4).
  Reading the generated assembly (rustc 1.98, `-O --emit asm`): in dequeue,
  `% cap` was compiled to compare-and-select, because the bounds check on
  buf[head] already proves head < cap; a real `udiv` is left only in
  enqueue. So one division per operation, not two. Plausible mechanism for
  the small gap, not measured: nothing in the head-to-head chain waits on
  that division. A CPU with a slower divide may show a bigger gap.

  and std's VecDeque is not power-of-two sized: asked for 2000, it holds
  exactly 2000. It is slower than these hand-rolled rings, but it can also
  GROW, which they refuse to do; this program does not measure where its
  extra cost per operation comes from.

  so in Rust the naive queue is the SLOWEST queue in this lesson, not the
  fastest: Go's `q = q[1:]` was a pointer increment, remove(0) moves the
  whole backlog.

its problem is this:

  naive q.remove(0)                           2 allocation events, 100000000 elements shifted
  ring buffer                                 1 allocation events
  std VecDeque                                1 allocation events

  the naive queue allocated twice -- the constructor, and one growth when
  the 1001st element arrived -- so Go's 98,996 does not carry over. My
  reading of that count: `q = q[1:]` shrinks the slice's capacity until
  append must replace the array, and an owned Vec cannot be resliced.
  What the naive Rust queue pays is the shift: 100,000,000 elements moved
  for 100,000 operations, 1000 per dequeue, counted in the loop.

  the ring and the VecDeque each allocate once, in the constructor. The
  ring's memory is bounded by a number YOU chose and every operation is
  Theta(1) WORST CASE -- not amortized, so no single enqueue can stall on
  a copy the way Vec::push or VecDeque::push_back can: the ring refuses
  instead of growing.

  that combination -- bounded, predictable, allocation-free after the
  constructor -- is why audio buffers, network buffers, log rings and
  bounded work queues are built this way. The cost is the fixed capacity,
  which example 4 removes and example 9 turns into a feature.
```

> **Go vs Rust:** The ring and its printed wrap trace are identical to Go's; the only change is that a refused enqueue returns `Err(8)`, handing the value back, instead of `false`. The benchmark flips: Go's naive `q = q[1:]` beat both rings, but Rust's naive queue is `Vec::remove(0)`, 70 to 110 ns/op across my runs against under 1 ns/op for the rings, and the `%` ring stayed within about 0.15 ns/op of the `&` ring (assembly shows one of the two `%` became compare-and-select). `VecDeque`, documented as a growable ring buffer, took about 2.7 ns/op.

**Complexity:** every operation Θ(1) **worst case**, not amortized · **0 allocations** in the steady
state · a non-power-of-two capacity costs 2.39 ns/op in integer division alone

---

## 4. Growing a ring, and the bug that hides until it doesn't

`🟢 easy` · *a wrapped region is two pieces, never one*

Doubling a ring is four lines, one of which everybody gets wrong — and the wrong version passes every
test you write by hand, because those tests all start with `head == 0`.

**Steps:**

1. Write the one-copy grow, then rotate the queue so `head != 0` and watch it corrupt.
2. Add the rotation sweep that catches it.
3. Confirm the amortized bound, and notice what you gave up by growing at all.

```go
package main

import (
	"fmt"
	"testing"
)

// The ring buffer's one weakness is the fixed capacity. Growing it is not hard,
// but it is the one place where "it's just a slice" stops being true:
//
//	the live region may be WRAPPED, so you cannot copy it in one call.
//
// Getting this wrong is the single most common ring-buffer bug, so this example
// writes the wrong version first and lets the test catch it.

type Growable struct {
	buf         []int
	mask        int
	head, count int
	grows       int
}

func NewGrowable(capacity int) *Growable {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Growable{buf: make([]int, n), mask: n - 1}
}

func (g *Growable) Len() int { return g.count }
func (g *Growable) Cap() int { return len(g.buf) }

func (g *Growable) Enqueue(v int) {
	if g.count == len(g.buf) {
		g.grow()
	}
	g.buf[(g.head+g.count)&g.mask] = v
	g.count++
}

func (g *Growable) Dequeue() (int, bool) {
	if g.count == 0 {
		return 0, false
	}
	v := g.buf[g.head]
	g.buf[g.head] = 0
	g.head = (g.head + 1) & g.mask
	g.count--
	return v, true
}

// grow doubles the array and UNWRAPS the live region back to index 0.
// Doubling keeps the capacity a power of two, so the mask stays valid.
//
// Two copies, because the live region is at most two contiguous pieces:
//
//	[ 6 7 . H3 4 5 ]   ->   piece A = buf[head:]  = 3 4 5
//	                        piece B = buf[:tail]  = 6 7
func (g *Growable) grow() {
	next := make([]int, len(g.buf)*2)
	n := copy(next, g.buf[g.head:])
	copy(next[n:], g.buf[:g.head])
	g.buf = next
	g.mask = len(next) - 1
	g.head = 0
	g.grows++
}

// growWrong is the bug: one copy, which only works when head == 0.
type GrowWrong struct {
	buf         []int
	mask        int
	head, count int
}

func NewGrowWrong(capacity int) *GrowWrong {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &GrowWrong{buf: make([]int, n), mask: n - 1}
}

func (g *GrowWrong) Enqueue(v int) {
	if g.count == len(g.buf) {
		next := make([]int, len(g.buf)*2)
		copy(next, g.buf) // WRONG: copies the PHYSICAL array, not the logical queue
		g.buf = next
		g.mask = len(next) - 1
		// head is left alone, which is the other half of the bug
	}
	g.buf[(g.head+g.count)&g.mask] = v
	g.count++
}

func (g *GrowWrong) Dequeue() (int, bool) {
	if g.count == 0 {
		return 0, false
	}
	v := g.buf[g.head]
	g.head = (g.head + 1) & g.mask
	g.count--
	return v, true
}

func drainG(g *Growable) []int {
	out := make([]int, 0, g.Len())
	for {
		v, ok := g.Dequeue()
		if !ok {
			return out
		}
		out = append(out, v)
	}
}

func drainW(g *GrowWrong) []int {
	out := make([]int, 0, g.count)
	for {
		v, ok := g.Dequeue()
		if !ok {
			return out
		}
		out = append(out, v)
	}
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
	fmt.Println("growing a ring is not `copy(next, buf)`, because the live region")
	fmt.Println("may be WRAPPED. Both queues below hold 1..4 in order, but one has")
	fmt.Println("been rotated so that head != 0:")
	fmt.Println()

	// Rotate both so head == 2, then fill to capacity and force a grow.
	build := func(enq func(int), deq func() (int, bool)) {
		for i := 0; i < 2; i++ { // push and pop twice to advance head
			enq(-1)
			deq()
		}
		for i := 1; i <= 4; i++ {
			enq(i)
		}
	}

	good := NewGrowable(4)
	build(good.Enqueue, good.Dequeue)
	bad := NewGrowWrong(4)
	build(bad.Enqueue, bad.Dequeue)

	fmt.Printf("  before grow, physical array: %v  head=%d\n", good.buf, good.head)
	fmt.Println("  logical order is 1 2 3 4 -- it wraps around the end.")
	fmt.Println()

	good.Enqueue(5)
	bad.Enqueue(5)

	fmt.Printf("  %-32s %v\n", "correct grow (two copies):", drainG(good))
	fmt.Printf("  %-32s %v\n", "one-copy grow:", drainW(bad))
	fmt.Println()
	fmt.Println("  the broken version returns the elements in the wrong order and")
	fmt.Println("  invents a zero. It does not panic. It does not fail on the first")
	fmt.Println("  test you write, because with head == 0 it is CORRECT -- and head")
	fmt.Println("  is 0 in every example you write by hand.")
	fmt.Println()
	fmt.Println("  the two-line fix, and the reason it is two lines:")
	fmt.Println()
	fmt.Println("      n := copy(next, g.buf[g.head:])   // the piece before the wrap")
	fmt.Println("      copy(next[n:], g.buf[:g.head])    // the piece after it")
	fmt.Println()
	fmt.Println("  a wrapped live region is at most TWO contiguous pieces, never")
	fmt.Println("  more. That fact is worth memorising -- it is also how you write")
	fmt.Println("  Slice(), io.Reader and io.Writer for a ring.")

	fmt.Println()
	fmt.Println("a model test finds it immediately, because it rotates:")
	fmt.Println()
	fails := 0
	firstFail := ""
	for rot := 0; rot < 8; rot++ {
		w := NewGrowWrong(4)
		for i := 0; i < rot; i++ {
			w.Enqueue(-1)
			w.Dequeue()
		}
		for i := 1; i <= 5; i++ { // 5 elements into a cap-4 ring forces a grow
			w.Enqueue(i)
		}
		got := drainW(w)
		want := []int{1, 2, 3, 4, 5}
		ok := len(got) == len(want)
		for i := range want {
			if ok && got[i] != want[i] {
				ok = false
			}
		}
		if !ok {
			fails++
			if firstFail == "" {
				firstFail = fmt.Sprintf("rotation %d -> %v", rot, got)
			}
		}
	}
	fmt.Printf("  8 starting rotations, same 5 enqueues: %d fail\n", fails)
	fmt.Printf("  first failure: %s\n", firstFail)
	fmt.Println()
	fmt.Println("  rotation 0 passes. That is the whole trap: the state that breaks")
	fmt.Println("  a ring buffer is a state you never reach by hand, so the test has")
	fmt.Println("  to reach it FOR you. Add a rotate loop to every ring test you")
	fmt.Println("  write -- it costs three lines and catches the only real bug.")

	fmt.Println()
	fmt.Println("growth is amortized Theta(1), the same argument as lesson 06:")
	fmt.Println()
	fmt.Printf("  %10s %10s %10s %12s\n", "enqueued", "capacity", "grows", "copies/enq")
	for _, n := range []int{1000, 10_000, 100_000, 1_000_000} {
		g := NewGrowable(4)
		for i := 0; i < n; i++ {
			g.Enqueue(i)
		}
		copies := 0
		for c := 4; c < g.Cap(); c *= 2 {
			copies += c
		}
		fmt.Printf("  %10d %10d %10d %12.2f\n", n, g.Cap(), g.grows, float64(copies)/float64(n))
	}
	fmt.Println()
	fmt.Println("  always under 2.0, never converging to a single number. Doubling")
	fmt.Println("  makes the total copies n + n/2 + n/4 + ... < 2n, so the BOUND is 2")
	fmt.Println("  copies per enqueue -- and where you land inside it depends on how")
	fmt.Println("  far past the last power of two you stopped. 10,000 is just over")
	fmt.Println("  8192 so it pays for a 16384-element array it barely uses (1.64);")
	fmt.Println("  1,000,000 is just under 1048576 and gets 1.05. Same algorithm,")
	fmt.Println("  1.6x apart, decided entirely by n.")
	fmt.Println()
	fmt.Println("  that still beats Go's own append (4.02 copies, lesson 03) for the")
	fmt.Println("  reason lesson 06 found: append's factor tapers to 1.25x, and a ring")
	fmt.Println("  cannot taper -- the mask requires a power of two, so it must double.")
	fmt.Println("  A constraint that looked like a limitation is buying you copies.")

	fmt.Println()
	fmt.Println("what it costs against the fixed ring, once it has stopped growing:")
	fmt.Println()
	const ops = 1_000_000
	tFixed := nsPerOp(func() {
		r := NewGrowable(2048)
		for i := 0; i < 1000; i++ {
			r.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			r.Enqueue(i)
			v, _ := r.Dequeue()
			sinkI = v
		}
	})
	tGrown := nsPerOp(func() {
		r := NewGrowable(4)
		for i := 0; i < 1000; i++ {
			r.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			r.Enqueue(i)
			v, _ := r.Dequeue()
			sinkI = v
		}
	})
	fmt.Printf("  %-34s %10.2f ns/op\n", "preallocated to 2048", tFixed/ops)
	fmt.Printf("  %-34s %10.2f ns/op\n", "grown from 4", tGrown/ops)
	fmt.Println()
	fmt.Println("  identical in the steady state -- growth is a startup cost, not a")
	fmt.Println("  per-operation one. But you have now given up the two properties")
	fmt.Println("  that made the ring worth having: BOUNDED memory and WORST-CASE")
	fmt.Println("  Theta(1). A growable ring is a faster naive queue, not a")
	fmt.Println("  different kind of thing.")
	fmt.Println()
	fmt.Println("  pick deliberately: growable when the queue is an implementation")
	fmt.Println("  detail and the producer is trusted; FIXED when the queue is a")
	fmt.Println("  boundary between a producer and a consumer you do not control.")
	fmt.Println("  Example 9 is about the second case.")
}
```

**Sample output:**

```
growing a ring is not `copy(next, buf)`, because the live region
may be WRAPPED. Both queues below hold 1..4 in order, but one has
been rotated so that head != 0:

  before grow, physical array: [3 4 1 2]  head=2
  logical order is 1 2 3 4 -- it wraps around the end.

  correct grow (two copies):       [1 2 3 4 5]
  one-copy grow:                   [1 2 0 0 5]

  the broken version returns the elements in the wrong order and
  invents a zero. It does not panic. It does not fail on the first
  test you write, because with head == 0 it is CORRECT -- and head
  is 0 in every example you write by hand.

  the two-line fix, and the reason it is two lines:

      n := copy(next, g.buf[g.head:])   // the piece before the wrap
      copy(next[n:], g.buf[:g.head])    // the piece after it

  a wrapped live region is at most TWO contiguous pieces, never
  more. That fact is worth memorising -- it is also how you write
  Slice(), io.Reader and io.Writer for a ring.

a model test finds it immediately, because it rotates:

  8 starting rotations, same 5 enqueues: 6 fail
  first failure: rotation 1 -> [1 2 3 0 5]

  rotation 0 passes. That is the whole trap: the state that breaks
  a ring buffer is a state you never reach by hand, so the test has
  to reach it FOR you. Add a rotate loop to every ring test you
  write -- it costs three lines and catches the only real bug.

growth is amortized Theta(1), the same argument as lesson 06:

    enqueued   capacity      grows   copies/enq
        1000       1024          8         1.02
       10000      16384         12         1.64
      100000     131072         15         1.31
     1000000    1048576         18         1.05

  always under 2.0, never converging to a single number. Doubling
  makes the total copies n + n/2 + n/4 + ... < 2n, so the BOUND is 2
  copies per enqueue -- and where you land inside it depends on how
  far past the last power of two you stopped. 10,000 is just over
  8192 so it pays for a 16384-element array it barely uses (1.64);
  1,000,000 is just under 1048576 and gets 1.05. Same algorithm,
  1.6x apart, decided entirely by n.

  that still beats Go's own append (4.02 copies, lesson 03) for the
  reason lesson 06 found: append's factor tapers to 1.25x, and a ring
  cannot taper -- the mask requires a power of two, so it must double.
  A constraint that looked like a limitation is buying you copies.

what it costs against the fixed ring, once it has stopped growing:

  preallocated to 2048                     2.17 ns/op
  grown from 4                             2.17 ns/op

  identical in the steady state -- growth is a startup cost, not a
  per-operation one. But you have now given up the two properties
  that made the ring worth having: BOUNDED memory and WORST-CASE
  Theta(1). A growable ring is a faster naive queue, not a
  different kind of thing.

  pick deliberately: growable when the queue is an implementation
  detail and the producer is trusted; FIXED when the queue is a
  boundary between a producer and a consumer you do not control.
  Example 9 is about the second case.
```

**Rust:**

```rust
use std::fmt::Display;
use std::hint::black_box;
use std::time::{Duration, Instant};

// The ring buffer's one weakness is the fixed capacity. Growing it is not hard,
// but it is the one place where "it's just a slice" stops being true:
//
//	the live region may be WRAPPED, so you cannot copy it in one call.
//
// Getting this wrong is the single most common ring-buffer bug, so this example
// writes the wrong version first and lets the test catch it.

/// Both queues speak this trait, so the rotate-and-fill helper and the drain
/// are written once (Go passes method values; a trait is the Rust spelling).
trait Fifo {
    fn enqueue(&mut self, v: i64);
    fn dequeue(&mut self) -> Option<i64>;

    /// dequeue everything, front to back
    fn drain_all(&mut self) -> Vec<i64> {
        let mut out = Vec::new();
        while let Some(v) = self.dequeue() {
            out.push(v);
        }
        out
    }
}

struct Growable {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
    grows: usize,
}

impl Growable {
    fn new(capacity: usize) -> Self {
        // Go loops `n <<= 1`; Rust has it built in.
        let n = capacity.next_power_of_two();
        Growable {
            buf: vec![0; n],
            mask: n - 1,
            head: 0,
            count: 0,
            grows: 0,
        }
    }

    fn cap(&self) -> usize {
        self.buf.len()
    }

    // grow doubles the array and UNWRAPS the live region back to index 0.
    // Doubling keeps the capacity a power of two, so the mask stays valid.
    //
    // Two copies, because the live region is at most two contiguous pieces:
    //
    //	[ 6 7 . H3 4 5 ]   ->   piece A = buf[head..]  = 3 4 5
    //	                        piece B = buf[..tail]  = 6 7
    fn grow(&mut self) {
        let doubled = self.buf.len() * 2;
        let mut next = Vec::with_capacity(doubled);
        // extend_from_slice APPENDS, so unlike Go's copy there is no `n` to carry.
        next.extend_from_slice(&self.buf[self.head..]);
        next.extend_from_slice(&self.buf[..self.head]);
        next.resize(doubled, 0);
        self.buf = next;
        self.mask = doubled - 1;
        self.head = 0;
        self.grows += 1;
    }
}

impl Fifo for Growable {
    fn enqueue(&mut self, v: i64) {
        if self.count == self.buf.len() {
            self.grow();
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
    }

    fn dequeue(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        // Go zeroes the slot so the GC can free what it pointed at. For an i64
        // there is nothing to release; kept so both versions do the same work.
        self.buf[self.head] = 0;
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
    }
}

/// GrowWrong is the bug: one copy, which only works when head == 0.
///
/// Bounds checks stay on, so in Rust the bug cannot read outside the array:
/// it returns the wrong VALUES, not garbage memory. The borrow checker has
/// nothing to say about it either -- it checks who may touch the elements, not
/// whether the right ones were copied.
struct GrowWrong {
    buf: Vec<i64>,
    mask: usize,
    head: usize,
    count: usize,
}

impl GrowWrong {
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        GrowWrong {
            buf: vec![0; n],
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }
}

impl Fifo for GrowWrong {
    fn enqueue(&mut self, v: i64) {
        if self.count == self.buf.len() {
            let mut next = vec![0; self.buf.len() * 2];
            // WRONG: copies the PHYSICAL array, not the logical queue
            next[..self.buf.len()].copy_from_slice(&self.buf);
            self.buf = next;
            self.mask = self.buf.len() - 1;
            // head is left alone, which is the other half of the bug
        }
        self.buf[(self.head + self.count) & self.mask] = v;
        self.count += 1;
    }

    fn dequeue(&mut self) -> Option<i64> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head];
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        Some(v)
    }
}

/// Go's `%v` of a slice: [1 2 3]
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(ToString::to_string).collect();
    format!("[{}]", parts.join(" "))
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

/// Rust's own Vec::push, counted the way the ring is counted: every time the
/// capacity changes, the elements already present would have had to move.
/// (An UPPER bound: the allocator may extend a block in place and move nothing.)
fn vec_push_copies(n: usize) -> (usize, f64) {
    let mut v: Vec<i64> = Vec::new();
    let mut cap = v.capacity();
    let mut copies = 0;
    for i in 0..n {
        v.push(i as i64);
        if v.capacity() != cap {
            copies += v.len() - 1;
            cap = v.capacity();
        }
    }
    (v.capacity(), copies as f64 / n as f64)
}

/// Rotate so that head == 2, then fill to capacity.
fn build<Q: Fifo>(q: &mut Q) {
    for _ in 0..2 {
        // push and pop twice to advance head
        q.enqueue(-1);
        q.dequeue();
    }
    for i in 1..=4 {
        q.enqueue(i);
    }
}

fn main() {
    println!("growing a ring is not `copy(next, buf)`, because the live region");
    println!("may be WRAPPED. Both queues below hold 1..4 in order, but one has");
    println!("been rotated so that head != 0:");
    println!();

    // Rotate both so head == 2, then fill to capacity and force a grow.
    let mut good = Growable::new(4);
    build(&mut good);
    let mut bad = GrowWrong::new(4);
    build(&mut bad);

    println!(
        "  before grow, physical array: {}  head={}",
        show(&good.buf),
        good.head
    );
    println!("  logical order is 1 2 3 4 -- it wraps around the end.");
    println!();

    good.enqueue(5);
    bad.enqueue(5);

    println!(
        "  {:<32} {}",
        "correct grow (two copies):",
        show(&good.drain_all())
    );
    println!("  {:<32} {}", "one-copy grow:", show(&bad.drain_all()));
    println!();
    println!("  the broken version returns the elements in the wrong order and");
    println!("  invents a zero. It does not panic. It does not fail on the first");
    println!("  test you write, because with head == 0 it is CORRECT -- and head");
    println!("  is 0 in every example you write by hand.");
    println!();
    println!("  the fix, and the reason it is two copies:");
    println!();
    println!(
        "      next.extend_from_slice(&self.buf[self.head..]);   // the piece before the wrap"
    );
    println!("      next.extend_from_slice(&self.buf[..self.head]);   // the piece after it");
    println!();
    println!("  a wrapped live region is at most TWO contiguous pieces, never");
    println!("  more. That fact is worth memorising -- it is also how you write");
    println!("  as_slices() (which is what VecDeque offers), io::Read and io::Write");
    println!("  for a ring.");

    println!();
    println!("a model test finds it immediately, because it rotates:");
    println!();
    let mut fails = 0;
    let mut first_fail = String::new();
    for rot in 0..8 {
        let mut w = GrowWrong::new(4);
        for _ in 0..rot {
            w.enqueue(-1);
            w.dequeue();
        }
        for i in 1..=5 {
            // 5 elements into a cap-4 ring forces a grow
            w.enqueue(i);
        }
        let got = w.drain_all();
        if got != [1, 2, 3, 4, 5] {
            fails += 1;
            if first_fail.is_empty() {
                first_fail = format!("rotation {rot} -> {}", show(&got));
            }
        }
    }
    println!("  8 starting rotations, same 5 enqueues: {fails} fail");
    println!("  first failure: {first_fail}");
    println!();
    println!("  rotation 0 passes. That is the whole trap: the state that breaks");
    println!("  a ring buffer is a state you never reach by hand, so the test has");
    println!("  to reach it FOR you. Add a rotate loop to every ring test you");
    println!("  write -- it costs three lines and catches the only real bug.");

    println!();
    println!("growth is amortized Theta(1), the same argument as lesson 06:");
    println!();
    println!(
        "  {:>10} {:>10} {:>10} {:>12}",
        "enqueued", "capacity", "grows", "copies/enq"
    );
    for n in [1000, 10_000, 100_000, 1_000_000] {
        let mut g = Growable::new(4);
        for i in 0..n {
            g.enqueue(i);
        }
        let mut copies = 0;
        let mut c = 4;
        while c < g.cap() {
            copies += c;
            c *= 2;
        }
        println!(
            "  {:>10} {:>10} {:>10} {:>12.2}",
            n,
            g.cap(),
            g.grows,
            copies as f64 / n as f64
        );
    }
    println!();
    println!("  always under 2.0, never converging to a single number. Doubling");
    println!("  makes the total copies n + n/2 + n/4 + ... < 2n, so the BOUND is 2");
    println!("  copies per enqueue -- and where you land inside it depends on how");
    println!("  far past the last power of two you stopped. 10,000 is just over");
    println!("  8192 so it pays for a 16384-element array it barely uses (1.64);");
    println!("  1,000,000 is just under 1048576 and gets 1.05. Same algorithm,");
    println!("  1.6x apart, decided entirely by n.");

    println!();
    println!("and Rust's own Vec::push, counted the same way (capacity() changes):");
    println!();
    println!(
        "  {:>10} {:>10} {:>12}",
        "pushed", "capacity", "copies/push"
    );
    for n in [1000, 10_000, 100_000, 1_000_000] {
        let (cap, per) = vec_push_copies(n);
        println!("  {:>10} {:>10} {:>12.2}", n, cap, per);
    }
    println!();
    println!("  the same capacities and the same copies/push as the ring: Vec also");
    println!("  doubles. So the Go comparison -- the ring beating `append` because");
    println!("  append's factor tapers to 1.25x -- does not exist here. In Rust, on");
    println!("  this count, the ring's power-of-two constraint neither costs nor");
    println!("  buys any copies compared with Vec's growth. (These count elements");
    println!("  that WOULD move; realloc may extend a block in place and move fewer,");
    println!("  so read them as an upper bound. This program does not count real");
    println!("  moves.)");

    println!();
    println!("what it costs against the fixed ring, once it has stopped growing:");
    println!();
    const OPS: i64 = 1_000_000;
    // black_box on the capacity: otherwise LLVM sees it through the constructor.
    let t_fixed = ns_per_op(|| {
        let mut r = Growable::new(black_box(2048));
        for i in 0..1000 {
            r.enqueue(i);
        }
        for i in 0..OPS {
            r.enqueue(i);
            black_box(r.dequeue());
        }
    });
    let t_grown = ns_per_op(|| {
        let mut r = Growable::new(black_box(4));
        for i in 0..1000 {
            r.enqueue(i);
        }
        for i in 0..OPS {
            r.enqueue(i);
            black_box(r.dequeue());
        }
    });
    println!(
        "  {:<34} {:>10.2} ns/op",
        "preallocated to 2048",
        t_fixed / OPS as f64
    );
    println!(
        "  {:<34} {:>10.2} ns/op",
        "grown from 4",
        t_grown / OPS as f64
    );
    println!();
    println!("  the two stayed within noise of each other (3 to 4 ns/op each, and");
    println!("  neither was consistently faster across my runs). Growth is a startup");
    println!("  cost, not a per-operation one: the grown ring is built from 4 and");
    println!("  doubles 8 times on the way to 1024, then runs the million operations");
    println!("  at a fixed size, same as the preallocated one. But you have now");
    println!("  given up the two properties that made the ring worth having: BOUNDED");
    println!("  memory and WORST-CASE Theta(1). A growable ring is an ordinary");
    println!("  unbounded queue with a cheap dequeue, not a different kind of thing.");
    println!();
    println!("  pick deliberately: growable when the queue is an implementation");
    println!("  detail and the producer is trusted; FIXED when the queue is a");
    println!("  boundary between a producer and a consumer you do not control.");
    println!("  Example 9 is about the second case.");
}
```

**Rust sample output:**

```
growing a ring is not `copy(next, buf)`, because the live region
may be WRAPPED. Both queues below hold 1..4 in order, but one has
been rotated so that head != 0:

  before grow, physical array: [3 4 1 2]  head=2
  logical order is 1 2 3 4 -- it wraps around the end.

  correct grow (two copies):       [1 2 3 4 5]
  one-copy grow:                   [1 2 0 0 5]

  the broken version returns the elements in the wrong order and
  invents a zero. It does not panic. It does not fail on the first
  test you write, because with head == 0 it is CORRECT -- and head
  is 0 in every example you write by hand.

  the fix, and the reason it is two copies:

      next.extend_from_slice(&self.buf[self.head..]);   // the piece before the wrap
      next.extend_from_slice(&self.buf[..self.head]);   // the piece after it

  a wrapped live region is at most TWO contiguous pieces, never
  more. That fact is worth memorising -- it is also how you write
  as_slices() (which is what VecDeque offers), io::Read and io::Write
  for a ring.

a model test finds it immediately, because it rotates:

  8 starting rotations, same 5 enqueues: 6 fail
  first failure: rotation 1 -> [1 2 3 0 5]

  rotation 0 passes. That is the whole trap: the state that breaks
  a ring buffer is a state you never reach by hand, so the test has
  to reach it FOR you. Add a rotate loop to every ring test you
  write -- it costs three lines and catches the only real bug.

growth is amortized Theta(1), the same argument as lesson 06:

    enqueued   capacity      grows   copies/enq
        1000       1024          8         1.02
       10000      16384         12         1.64
      100000     131072         15         1.31
     1000000    1048576         18         1.05

  always under 2.0, never converging to a single number. Doubling
  makes the total copies n + n/2 + n/4 + ... < 2n, so the BOUND is 2
  copies per enqueue -- and where you land inside it depends on how
  far past the last power of two you stopped. 10,000 is just over
  8192 so it pays for a 16384-element array it barely uses (1.64);
  1,000,000 is just under 1048576 and gets 1.05. Same algorithm,
  1.6x apart, decided entirely by n.

and Rust's own Vec::push, counted the same way (capacity() changes):

      pushed   capacity  copies/push
        1000       1024         1.02
       10000      16384         1.64
      100000     131072         1.31
     1000000    1048576         1.05

  the same capacities and the same copies/push as the ring: Vec also
  doubles. So the Go comparison -- the ring beating `append` because
  append's factor tapers to 1.25x -- does not exist here. In Rust, on
  this count, the ring's power-of-two constraint neither costs nor
  buys any copies compared with Vec's growth. (These count elements
  that WOULD move; realloc may extend a block in place and move fewer,
  so read them as an upper bound. This program does not count real
  moves.)

what it costs against the fixed ring, once it has stopped growing:

  preallocated to 2048                     3.67 ns/op
  grown from 4                             3.68 ns/op

  the two stayed within noise of each other (3 to 4 ns/op each, and
  neither was consistently faster across my runs). Growth is a startup
  cost, not a per-operation one: the grown ring is built from 4 and
  doubles 8 times on the way to 1024, then runs the million operations
  at a fixed size, same as the preallocated one. But you have now
  given up the two properties that made the ring worth having: BOUNDED
  memory and WORST-CASE Theta(1). A growable ring is an ordinary
  unbounded queue with a cheap dequeue, not a different kind of thing.

  pick deliberately: growable when the queue is an implementation
  detail and the producer is trusted; FIXED when the queue is a
  boundary between a producer and a consumer you do not control.
  Example 9 is about the second case.
```

> **Go vs Rust:** The wrapped-grow demo, the 6-of-8 rotation sweep and the copies-per-enqueue table are pure integer logic and identical to Go's; `extend_from_slice` appends, so there is no `n` to carry between the two pieces, and the one-copy bug is still a wrong-values bug (indexing is bounds-checked), not memory corruption. One Go claim does not carry over: Go's ring beat `append`'s 4.02 copies, but Rust's `Vec::push` doubles too and, counted through `capacity()`, shows the same 1.02 / 1.64 / 1.31 / 1.05 copies per push as the ring (an upper bound, since `realloc` may extend in place). Steady-state cost was 3 to 4 ns/op for both the preallocated and the grown ring, within noise of each other as in Go (2.17 each).

**Complexity:** growth amortized Θ(1), bounded by **2 copies per enqueue** (measured 1.02–1.64
depending only on where `n` falls relative to a power of two) · **rotation 0 passes and 6 of 8 fail** —
the sweep is three lines and catches the only real bug

---

## 5. `Deque[T]` — opening the other end

`🟢 easy` · *`(head-1) & mask`, and why `%` will not do*

One extra line makes the ring run backwards too. That line is `head = (head-1) & mask`, and it works
only because the capacity is a power of two — Go's `%` returns `-1` for `-1 % 8`.

**Steps:**

1. Add `PushFront` and `PopBack`, and watch `head` wrap backwards past zero.
2. See that a deque subsumes both a stack and a queue.
3. Delete the slot-zeroing line and measure what it was doing.

```go
package main

import (
	"fmt"
	"runtime"
	"testing"
)

// A DEQUE (double-ended queue) is a ring buffer that also runs backwards.
// That is the entire generalisation:
//
//	PushBack   write at (head+count)&mask, count++
//	PushFront  head = (head-1)&mask, write at head, count++
//
// The `(head-1)&mask` is the trick. In Go, `-1 & mask` on an int gives mask,
// because Go's & is bitwise on two's complement -- so decrementing past zero
// wraps to the end of the array with no branch. `%` would NOT do this: in Go,
// -1 % 8 is -1, not 7. That is a second reason powers of two are not optional.

type Deque[T any] struct {
	buf         []T
	mask        int
	head, count int
}

func NewDeque[T any](capacity int) *Deque[T] {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Deque[T]{buf: make([]T, n), mask: n - 1}
}

func (d *Deque[T]) Len() int { return d.count }
func (d *Deque[T]) Cap() int { return len(d.buf) }

func (d *Deque[T]) grow() {
	next := make([]T, len(d.buf)*2)
	n := copy(next, d.buf[d.head:])
	copy(next[n:], d.buf[:d.head])
	d.buf, d.mask, d.head = next, len(next)-1, 0
}

func (d *Deque[T]) PushBack(v T) {
	if d.count == len(d.buf) {
		d.grow()
	}
	d.buf[(d.head+d.count)&d.mask] = v
	d.count++
}

func (d *Deque[T]) PushFront(v T) {
	if d.count == len(d.buf) {
		d.grow()
	}
	d.head = (d.head - 1) & d.mask // wraps to the end when head is 0
	d.buf[d.head] = v
	d.count++
}

func (d *Deque[T]) PopFront() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	v := d.buf[d.head]
	d.buf[d.head] = zero // MUST: T may contain pointers
	d.head = (d.head + 1) & d.mask
	d.count--
	return v, true
}

func (d *Deque[T]) PopBack() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	i := (d.head + d.count - 1) & d.mask
	v := d.buf[i]
	d.buf[i] = zero
	d.count--
	return v, true
}

func (d *Deque[T]) At(i int) T { return d.buf[(d.head+i)&d.mask] }

func (d *Deque[T]) Front() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	return d.buf[d.head], true
}

func (d *Deque[T]) Back() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	return d.buf[(d.head+d.count-1)&d.mask], true
}

func (d *Deque[T]) slice() []T {
	out := make([]T, d.count)
	for i := 0; i < d.count; i++ {
		out[i] = d.At(i)
	}
	return out
}

// A leaky deque: same code with the zeroing removed. Example 1's lesson,
// applied to a structure that looks like it cannot leak.
type Leaky[T any] struct {
	buf         []T
	mask        int
	head, count int
}

func NewLeaky[T any](capacity int) *Leaky[T] {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Leaky[T]{buf: make([]T, n), mask: n - 1}
}
func (d *Leaky[T]) PushBack(v T) {
	d.buf[(d.head+d.count)&d.mask] = v
	d.count++
}
func (d *Leaky[T]) PopFront() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	v := d.buf[d.head]
	// no d.buf[d.head] = zero
	d.head = (d.head + 1) & d.mask
	d.count--
	return v, true
}

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
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
	d := NewDeque[string](4)
	fmt.Println("one structure, four operations, and it subsumes both of the last")
	fmt.Println("two lessons:")
	fmt.Println()
	for _, op := range []struct {
		name string
		fn   func()
	}{
		{`PushBack("b")`, func() { d.PushBack("b") }},
		{`PushBack("c")`, func() { d.PushBack("c") }},
		{`PushFront("a")`, func() { d.PushFront("a") }},
		{`PushFront("z")`, func() { d.PushFront("z") }},
	} {
		op.fn()
		fmt.Printf("  %-16s %v   head=%d physical=%v\n", op.name, d.slice(), d.head, d.buf)
	}
	f, _ := d.PopFront()
	b, _ := d.PopBack()
	fmt.Printf("  %-16s -> %q and %q, leaving %v\n", "PopFront/PopBack", f, b, d.slice())

	fmt.Println()
	fmt.Println("  watch the physical array on the PushFront lines: head went 0 -> 3,")
	fmt.Println("  wrapping backwards past zero. That is `(head-1) & mask`, one AND")
	fmt.Println("  with no branch and no bounds problem.")
	fmt.Println()
	fmt.Println("  it does NOT work with %: in Go, -1 % 8 == -1, not 7, because Go's")
	fmt.Println("  remainder takes the sign of the dividend. A ring written with % has")
	fmt.Println("  to add cap first: ((head-1+cap) % cap). Powers of two make the")
	fmt.Println("  problem disappear rather than be worked around.")

	fmt.Println()
	fmt.Println("use only one end and you have already built lesson 08 and this one:")
	fmt.Println()
	fmt.Printf("  %-34s %s\n", "PushBack + PopBack", "a stack (lesson 08)")
	fmt.Printf("  %-34s %s\n", "PushBack + PopFront", "a queue (this lesson)")
	fmt.Printf("  %-34s %s\n", "PushFront + PopFront", "a stack, growing the other way")
	fmt.Printf("  %-34s %s\n", "all four", "a deque -- examples 10, 13, 15")
	fmt.Println()
	fmt.Println("  which is why Go has no queue type in the standard library and no")
	fmt.Println("  deque type either: a slice is a stack, and everything else is")
	fmt.Println("  thirty lines you write once.")

	fmt.Println()
	fmt.Println("the zeroing on Pop is not optional. Same deque, one line removed:")
	fmt.Println()
	type payload struct{ data []byte }
	const n = 20_000

	base := heapMB()
	good := NewDeque[*payload](1 << 16)
	for i := 0; i < n; i++ {
		good.PushBack(&payload{data: make([]byte, 1024)})
		good.PopFront()
	}
	goodMB := heapMB() - base
	runtime.KeepAlive(good)

	base = heapMB()
	leaky := NewLeaky[*payload](1 << 16)
	for i := 0; i < n; i++ {
		leaky.PushBack(&payload{data: make([]byte, 1024)})
		leaky.PopFront()
	}
	leakyMB := heapMB() - base
	runtime.KeepAlive(leaky)

	fmt.Printf("  %-40s %8.1f MB held\n", "Pop writes the zero value", goodMB)
	fmt.Printf("  %-40s %8.1f MB held\n", "Pop leaves the slot alone", leakyMB)
	fmt.Println()
	fmt.Printf("  both deques report Len()==0. The leaky one is still holding %.0f\n", leakyMB)
	fmt.Println("  MB, because every slot it ever wrote is still a live pointer as")
	fmt.Println("  far as the GC is concerned. The array is bounded; what the array")
	fmt.Println("  POINTS AT is not.")
	fmt.Println()
	fmt.Println("  this is example 1's leak wearing a different hat, and it is worse:")
	fmt.Println("  there is no growing slice to notice, no capacity climbing, nothing")
	fmt.Println("  to see in a heap profile except a large []*payload that is")
	fmt.Println("  'obviously' empty.")

	fmt.Println()
	fmt.Println("what it costs against a slice, on the operation slices are bad at:")
	fmt.Println()
	const m = 20_000
	tSlice := nsPerOp(func() {
		var s []int
		for i := 0; i < m; i++ {
			s = append([]int{i}, s...) // the classic front-insert
		}
		sinkI = len(s)
	})
	tDeque := nsPerOp(func() {
		dd := NewDeque[int](8)
		for i := 0; i < m; i++ {
			dd.PushFront(i)
		}
		sinkI = dd.Len()
	})
	fmt.Printf("  %-34s %12.0f ns for %d front-inserts\n", "append([]int{v}, s...)", tSlice, m)
	fmt.Printf("  %-34s %12.0f ns   %.0fx faster\n", "deque PushFront", tDeque, tSlice/tDeque)
	fmt.Println()
	fmt.Println("  Theta(n^2) against Theta(n), which is the same gap lesson 06's gap")
	fmt.Println("  buffer opened -- but a deque needs no gap, no rebalancing and no")
	fmt.Println("  special cases. When you need both ends, this is the structure.")
	fmt.Println()
	fmt.Println("  the rest of the lesson uses it four times: BFS (6), sliding-window")
	fmt.Println("  maximum (10), 0-1 BFS (13) and work stealing (15).")
}
```

**Sample output:**

```
one structure, four operations, and it subsumes both of the last
two lessons:

  PushBack("b")    [b]   head=0 physical=[b   ]
  PushBack("c")    [b c]   head=0 physical=[b c  ]
  PushFront("a")   [a b c]   head=3 physical=[b c  a]
  PushFront("z")   [z a b c]   head=2 physical=[b c z a]
  PopFront/PopBack -> "z" and "c", leaving [a b]

  watch the physical array on the PushFront lines: head went 0 -> 3,
  wrapping backwards past zero. That is `(head-1) & mask`, one AND
  with no branch and no bounds problem.

  it does NOT work with %: in Go, -1 % 8 == -1, not 7, because Go's
  remainder takes the sign of the dividend. A ring written with % has
  to add cap first: ((head-1+cap) % cap). Powers of two make the
  problem disappear rather than be worked around.

use only one end and you have already built lesson 08 and this one:

  PushBack + PopBack                 a stack (lesson 08)
  PushBack + PopFront                a queue (this lesson)
  PushFront + PopFront               a stack, growing the other way
  all four                           a deque -- examples 10, 13, 15

  which is why Go has no queue type in the standard library and no
  deque type either: a slice is a stack, and everything else is
  thirty lines you write once.

the zeroing on Pop is not optional. Same deque, one line removed:

  Pop writes the zero value                     0.5 MB held
  Pop leaves the slot alone                    20.5 MB held

  both deques report Len()==0. The leaky one is still holding 20
  MB, because every slot it ever wrote is still a live pointer as
  far as the GC is concerned. The array is bounded; what the array
  POINTS AT is not.

  this is example 1's leak wearing a different hat, and it is worse:
  there is no growing slice to notice, no capacity climbing, nothing
  to see in a heap profile except a large []*payload that is
  'obviously' empty.

what it costs against a slice, on the operation slices are bad at:

  append([]int{v}, s...)                 94152788 ns for 20000 front-inserts
  deque PushFront                           59587 ns   1580x faster

  Theta(n^2) against Theta(n), which is the same gap lesson 06's gap
  buffer opened -- but a deque needs no gap, no rebalancing and no
  special cases. When you need both ends, this is the structure.

  the rest of the lesson uses it four times: BFS (6), sliding-window
  maximum (10), 0-1 BFS (13) and work stealing (15).
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::VecDeque;
use std::fmt::Display;
use std::hint::black_box;
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- the kit's allocation counter: stand-in for runtime.ReadMemStats ----
// Go reads HeapAlloc after runtime.GC(); Rust has no GC, so the allocator itself
// counts the bytes currently live -- exact and deterministic.
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
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

/// bytes currently live on the heap (a deterministic MemStats.HeapAlloc).
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

// A DEQUE (double-ended queue) is a ring buffer that also runs backwards.
// That is the entire generalisation:
//
//	push_back   write at (head+count)&mask, count++
//	push_front  head = head.wrapping_sub(1)&mask, write at head, count++
//
// The `wrapping_sub(1) & mask` is the trick. Decrementing past zero wraps to
// the top of the number line, and ANDing with the mask keeps only the low bits,
// which is the last slot of the array -- no branch. It works because the
// capacity is a power of two. `%` would NOT do this -- see main for why.
//
// Rust difference: the slots are Option<T>, not T. Go fills a slot with the zero
// value of T; Rust has no universal zero value (not every T has a Default), and
// a Vec<T> cannot hold "holes". None is the hole.

struct Deque<T> {
    buf: Vec<Option<T>>,
    mask: usize,
    head: usize,
    count: usize,
}

impl<T> Deque<T> {
    fn new(capacity: usize) -> Self {
        // Go loops `n <<= 1`; Rust has it built in.
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
    fn cap(&self) -> usize {
        self.buf.len()
    }

    // Two pieces, as in example 4, but T may not be Copy, so they are MOVED
    // out of the old array with drain/append instead of copied.
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

    fn push_front(&mut self, v: T) {
        if self.count == self.buf.len() {
            self.grow();
        }
        // head is a usize: a plain `head - 1` at 0 is an overflow (a panic in a
        // debug build). wrapping_sub says "wrap" explicitly, in every build.
        self.head = self.head.wrapping_sub(1) & self.mask; // wraps to the end when head is 0
        self.buf[self.head] = Some(v);
        self.count += 1;
    }

    fn pop_front(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        // take() MOVES the value out and leaves None behind. This is the whole
        // of Go's `v := buf[head]; buf[head] = zero`: in Rust the read and the
        // clearing are one operation, so there is no second line to forget.
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

    // Go's Front/Back/Cap-style accessors are omitted: nothing below calls them
    // and clippy -D warnings rejects dead code.

    fn slice(&self) -> Vec<&T> {
        (0..self.count).filter_map(|i| self.at(i)).collect()
    }
}

impl<T: Display> Deque<T> {
    /// the physical array as Go's %v prints it: an empty slot is an empty string
    fn physical(&self) -> String {
        let parts: Vec<String> = self
            .buf
            .iter()
            .map(|s| s.as_ref().map_or_else(String::new, ToString::to_string))
            .collect();
        format!("[{}]", parts.join(" "))
    }
}

// A leaky deque: same code with the clearing removed. Example 1's lesson,
// applied to a structure that looks like it cannot leak.
//
// Rust will not let you write Go's bug by forgetting a line: moving a value out
// of a Vec slot (`let v = self.buf[i];`) is a compile error (E0507), and the only
// move that leaves the slot Some is take(). To keep the slot full you must COPY
// the handle on purpose, with .clone() -- so that is what this does.
struct Leaky<T> {
    buf: Vec<Option<T>>,
    mask: usize,
    head: usize,
    count: usize,
}

impl<T: Clone> Leaky<T> {
    fn new(capacity: usize) -> Self {
        let n = capacity.next_power_of_two();
        Leaky {
            buf: std::iter::repeat_with(|| None).take(n).collect(),
            mask: n - 1,
            head: 0,
            count: 0,
        }
    }

    fn push_back(&mut self, v: T) {
        let i = (self.head + self.count) & self.mask;
        self.buf[i] = Some(v);
        self.count += 1;
    }

    fn pop_front(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        let v = self.buf[self.head].clone();
        // no .take(): the slot keeps its own handle
        self.head = (self.head + 1) & self.mask;
        self.count -= 1;
        v
    }
}

/// Go's `%v` of a slice: [1 2 3]
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(ToString::to_string).collect();
    format!("[{}]", parts.join(" "))
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

/// The payload Go's `*payload` points at. Rc is a shared pointer: cloning it
/// copies the pointer, not the 1 KB, which is exactly what Go's `v := buf[head]` does.
struct Payload {
    data: Vec<u8>,
}

fn payload() -> Rc<Payload> {
    Rc::new(Payload {
        data: vec![0; 1024],
    })
}

fn main() {
    let mut d: Deque<&str> = Deque::new(4);
    println!("one structure, four operations, and it subsumes both of the last");
    println!("two lessons:");
    println!();
    for (name, front, v) in [
        (r#"push_back("b")"#, false, "b"),
        (r#"push_back("c")"#, false, "c"),
        (r#"push_front("a")"#, true, "a"),
        (r#"push_front("z")"#, true, "z"),
    ] {
        if front {
            d.push_front(v);
        } else {
            d.push_back(v);
        }
        println!(
            "  {:<16} {}   head={} physical={}",
            name,
            show(&d.slice()),
            d.head,
            d.physical()
        );
    }
    let f = d.pop_front().unwrap();
    let b = d.pop_back().unwrap();
    println!(
        "  {:<16} -> {:?} and {:?}, leaving {}",
        "pop_front/pop_back",
        f,
        b,
        show(&d.slice())
    );

    println!();
    println!("  watch the physical array on the push_front lines: head went 0 -> 3,");
    println!("  wrapping backwards past zero. That is `wrapping_sub(1) & mask`, one");
    println!("  AND with no branch and no bounds problem. head is a usize, so a plain");
    println!("  `head - 1` at 0 would be an overflow (a panic in a debug build);");
    println!("  wrapping_sub makes the wrap explicit.");
    println!();
    println!("  it does NOT work with %. Signed first -- the remainder takes the sign");
    println!("  of the dividend in Rust too, as in Go:");
    println!();
    let minus_one = black_box(-1i64);
    println!("      (-1i64) % 8              = {:>2}", minus_one % 8);
    println!(
        "      (-1i64).rem_euclid(8)    = {:>2}   (Rust's other remainder)",
        minus_one.rem_euclid(8)
    );
    println!("      (-1i64) & 7              = {:>2}", minus_one & 7);
    println!();
    println!("  and with the unsigned wrap, which is what the code above uses:");
    println!();
    let zero = black_box(0usize);
    let (cap8, cap5) = (black_box(8usize), black_box(5usize));
    println!(
        "      cap 8: 0usize.wrapping_sub(1) % 8 = {}   (right: 8 divides 2^64)",
        zero.wrapping_sub(1) % cap8
    );
    println!(
        "      cap 5: 0usize.wrapping_sub(1) % 5 = {}   (WRONG, wanted 4: 5 does not divide 2^64)",
        zero.wrapping_sub(1) % cap5
    );
    println!(
        "      cap 5: (0 + 5 - 1) % 5            = {}   (add cap first: right for any cap)",
        (zero + cap5 - 1) % cap5
    );
    println!();
    println!("  so a ring written with % has to add cap first: ((head+cap-1) % cap).");
    println!("  Powers of two make the problem disappear rather than be worked around.");

    println!();
    println!("use only one end and you have already built lesson 08 and this one:");
    println!();
    for (ops, meaning) in [
        ("push_back + pop_back", "a stack (lesson 08)"),
        ("push_back + pop_front", "a queue (this lesson)"),
        ("push_front + pop_front", "a stack, growing the other way"),
        ("all four", "a deque -- examples 10, 13, 15"),
    ] {
        println!("  {:<34} {}", ops, meaning);
    }
    println!();
    println!("  a Vec is already a stack, and std ships the rest: VecDeque, which its");
    println!("  documentation calls a growable ring buffer -- which is what the");
    println!("  code above is, minus the polish.");

    println!();
    println!("the clearing on pop is not optional. Go removes one line; in Rust you");
    println!("have to copy the handle on purpose to get the same bug:");
    println!();
    const N: usize = 20_000;
    let mb = |bytes: usize| bytes as f64 / (1 << 20) as f64;

    // Each block measures while its deque is still alive (Go's runtime.KeepAlive).
    let good_held = {
        let base = live_bytes();
        let mut good: Deque<Rc<Payload>> = Deque::new(1 << 16);
        for _ in 0..N {
            good.push_back(payload());
            if let Some(p) = good.pop_front() {
                black_box(p.data.len());
            }
        }
        black_box(good.len());
        live_bytes() - base
    };
    let (leaky_held, leaky_len) = {
        let base = live_bytes();
        let mut leaky: Leaky<Rc<Payload>> = Leaky::new(1 << 16);
        for _ in 0..N {
            leaky.push_back(payload());
            if let Some(p) = leaky.pop_front() {
                black_box(p.data.len());
            }
        }
        (live_bytes() - base, leaky.count)
    };
    let std_held = {
        let base = live_bytes();
        let mut dq: VecDeque<Rc<Payload>> = VecDeque::with_capacity(1 << 16);
        for _ in 0..N {
            dq.push_back(payload());
            if let Some(p) = dq.pop_front() {
                black_box(p.data.len());
            }
        }
        black_box(dq.len());
        live_bytes() - base
    };

    println!(
        "  {:<40} {:>8.1} MB held",
        "pop_front moves the value out (take)",
        mb(good_held)
    );
    println!(
        "  {:<40} {:>8.1} MB held",
        "pop_front clones, leaves the slot full",
        mb(leaky_held)
    );
    println!(
        "  {:<40} {:>8.1} MB held",
        "std VecDeque (not in the Go example)",
        mb(std_held)
    );
    println!();
    println!(
        "  both hand-written deques report len()==0 (the leaky one: count = {}).",
        leaky_len
    );
    println!(
        "  The leaky one is still holding {:.0} MB, because every slot it ever",
        mb(leaky_held)
    );
    println!("  wrote still owns a strong Rc handle to its payload, so none of the");
    println!("  20,000 payloads reaches a reference count of zero. The array is");
    println!("  bounded; what the array POINTS AT is not.");
    println!();
    println!("  in Go a forgotten line is enough; in Rust the safe pop that moves out");
    println!("  (take, or what VecDeque does) cannot leave a live value behind, and");
    println!("  holding on needs a deliberate .clone(). The failure is the same: a");
    println!(
        "  structure that is 'obviously' empty and owns {:.0} MB. There is no GC in",
        mb(leaky_held)
    );
    println!("  this story; the counting allocator reads the held bytes directly.");

    println!();
    println!("what it costs against a slice, on the operation slices are bad at:");
    println!();
    const M: i64 = 20_000;
    let t_insert = ns_per_op(|| {
        let mut s: Vec<i64> = Vec::new();
        for i in 0..M {
            s.insert(0, i); // the idiomatic front-insert: shifts everything right IN PLACE
        }
        black_box(s.len());
    });
    let t_concat = ns_per_op(|| {
        let mut s: Vec<i64> = Vec::new();
        for i in 0..M {
            // Go's `append([]int{i}, s...)`: a NEW array every time, old one dropped
            s = [&[i][..], &s[..]].concat();
        }
        black_box(s.len());
    });
    let t_deque = ns_per_op(|| {
        let mut dd: Deque<i64> = Deque::new(8);
        for i in 0..M {
            dd.push_front(i);
        }
        black_box(dd.len());
    });
    let t_std = ns_per_op(|| {
        let mut dd: VecDeque<i64> = VecDeque::new();
        for i in 0..M {
            dd.push_front(i);
        }
        black_box(dd.len());
    });
    println!(
        "  {:<34} {:>12.0} ns for {} front-inserts",
        "s.insert(0, v)", t_insert, M
    );
    println!(
        "  {:<34} {:>12.0} ns for {} front-inserts",
        "[&[v], &s].concat() (Go's shape)", t_concat, M
    );
    println!(
        "  {:<34} {:>12.0} ns   {:.0}x faster than insert(0), {:.0}x than concat",
        "deque push_front",
        t_deque,
        t_insert / t_deque,
        t_concat / t_deque
    );
    println!(
        "  {:<34} {:>12.0} ns   {:.0}x faster than insert(0), {:.0}x than concat",
        "std VecDeque push_front",
        t_std,
        t_insert / t_std,
        t_concat / t_std
    );

    // element moves are a property of the algorithm, so count them instead of timing them
    let shifted: u64 = (0..M as u64).sum();
    let mut probe: Deque<i64> = Deque::new(8);
    for i in 0..M {
        probe.push_front(i);
    }
    let mut grow_moves = 0;
    let mut c = 8;
    while c < probe.cap() {
        grow_moves += c;
        c *= 2;
    }
    println!();
    println!("  the two slice versions are different algorithms with the same");
    println!("  shape of cost: insert(0) shifts the Vec in place (no allocation),");
    println!("  concat allocates a fresh Vec and copies into it every time (Go's");
    println!("  append does the latter). Both move the whole backlog on every");
    println!("  front-insert -- {shifted} element moves for {M} inserts,");
    println!("  Theta(n^2). The deque moves elements only when it grows: {grow_moves}");
    println!(
        "  for the same {M} inserts (capacity 8 -> {}), Theta(n).",
        probe.cap()
    );
    println!();
    println!("  across my runs the ratios stayed in the same ballpark and the absolute");
    println!("  ns did not: the deque 300x to 400x faster than insert(0) (450x to 700x");
    println!("  than concat), std's VecDeque 1000x to 1350x than insert(0). std's is");
    println!("  about 3x faster than the hand-rolled one; plausible reason, not");
    println!("  measured: this one stores Option<i64> slots, twice the width of a");
    println!("  bare i64.");
    println!();
    println!("  a deque needs no gap, no rebalancing and no special cases. When you");
    println!("  need both ends, this is the structure.");
    println!();
    println!("  the rest of the lesson uses it four times: BFS (6), sliding-window");
    println!("  maximum (10), 0-1 BFS (13) and work stealing (15).");
}
```

**Rust sample output:**

```
one structure, four operations, and it subsumes both of the last
two lessons:

  push_back("b")   [b]   head=0 physical=[b   ]
  push_back("c")   [b c]   head=0 physical=[b c  ]
  push_front("a")  [a b c]   head=3 physical=[b c  a]
  push_front("z")  [z a b c]   head=2 physical=[b c z a]
  pop_front/pop_back -> "z" and "c", leaving [a b]

  watch the physical array on the push_front lines: head went 0 -> 3,
  wrapping backwards past zero. That is `wrapping_sub(1) & mask`, one
  AND with no branch and no bounds problem. head is a usize, so a plain
  `head - 1` at 0 would be an overflow (a panic in a debug build);
  wrapping_sub makes the wrap explicit.

  it does NOT work with %. Signed first -- the remainder takes the sign
  of the dividend in Rust too, as in Go:

      (-1i64) % 8              = -1
      (-1i64).rem_euclid(8)    =  7   (Rust's other remainder)
      (-1i64) & 7              =  7

  and with the unsigned wrap, which is what the code above uses:

      cap 8: 0usize.wrapping_sub(1) % 8 = 7   (right: 8 divides 2^64)
      cap 5: 0usize.wrapping_sub(1) % 5 = 0   (WRONG, wanted 4: 5 does not divide 2^64)
      cap 5: (0 + 5 - 1) % 5            = 4   (add cap first: right for any cap)

  so a ring written with % has to add cap first: ((head+cap-1) % cap).
  Powers of two make the problem disappear rather than be worked around.

use only one end and you have already built lesson 08 and this one:

  push_back + pop_back               a stack (lesson 08)
  push_back + pop_front              a queue (this lesson)
  push_front + pop_front             a stack, growing the other way
  all four                           a deque -- examples 10, 13, 15

  a Vec is already a stack, and std ships the rest: VecDeque, which its
  documentation calls a growable ring buffer -- which is what the
  code above is, minus the polish.

the clearing on pop is not optional. Go removes one line; in Rust you
have to copy the handle on purpose to get the same bug:

  pop_front moves the value out (take)          0.5 MB held
  pop_front clones, leaves the slot full       20.8 MB held
  std VecDeque (not in the Go example)          0.5 MB held

  both hand-written deques report len()==0 (the leaky one: count = 0).
  The leaky one is still holding 21 MB, because every slot it ever
  wrote still owns a strong Rc handle to its payload, so none of the
  20,000 payloads reaches a reference count of zero. The array is
  bounded; what the array POINTS AT is not.

  in Go a forgotten line is enough; in Rust the safe pop that moves out
  (take, or what VecDeque does) cannot leave a live value behind, and
  holding on needs a deliberate .clone(). The failure is the same: a
  structure that is 'obviously' empty and owns 21 MB. There is no GC in
  this story; the counting allocator reads the held bytes directly.

what it costs against a slice, on the operation slices are bad at:

  s.insert(0, v)                         24805383 ns for 20000 front-inserts
  [&[v], &s].concat() (Go's shape)       34774410 ns for 20000 front-inserts
  deque push_front                          68252 ns   363x faster than insert(0), 509x than concat
  std VecDeque push_front                   24430 ns   1015x faster than insert(0), 1423x than concat

  the two slice versions are different algorithms with the same
  shape of cost: insert(0) shifts the Vec in place (no allocation),
  concat allocates a fresh Vec and copies into it every time (Go's
  append does the latter). Both move the whole backlog on every
  front-insert -- 199990000 element moves for 20000 inserts,
  Theta(n^2). The deque moves elements only when it grows: 32760
  for the same 20000 inserts (capacity 8 -> 32768), Theta(n).

  across my runs the ratios stayed in the same ballpark and the absolute
  ns did not: the deque 300x to 400x faster than insert(0) (450x to 700x
  than concat), std's VecDeque 1000x to 1350x than insert(0). std's is
  about 3x faster than the hand-rolled one; plausible reason, not
  measured: this one stores Option<i64> slots, twice the width of a
  bare i64.

  a deque needs no gap, no rebalancing and no special cases. When you
  need both ends, this is the structure.

  the rest of the lesson uses it four times: BFS (6), sliding-window
  maximum (10), 0-1 BFS (13) and work stealing (15).
```

> **Go vs Rust:** The wrap trace (`head` 0 -> 3 -> 2) is identical to Go's, but `usize` forces `wrapping_sub(1)` where Go's `int` goes negative, and the program shows that `wrapping_sub(1) % cap` is wrong for cap 5 (0, not 4). Go's leak cannot be written by forgetting a line: `Option::take` reads and clears the slot in one step, so the leaky deque had to `.clone()` an `Rc` on purpose, and then held 20.8 MB against 0.5 MB, read exactly from the counting allocator since there is no GC. The front-insert baseline is a different algorithm from Go's: `append([]int{v}, s...)` allocates a new slice per call, whereas Rust's idiomatic `Vec::insert(0, v)` shifts in place, so both were measured (`insert(0)` was the faster of the two, 1.4x to 1.7x across my runs); the deque was roughly 300x to 400x faster than `insert(0)` and 450x to 700x faster than the allocating form, where Go measured 1580x.

**Complexity:** all four operations Θ(1) · front-insertion **1580× faster** than `append([]T{v}, s...)`
at n=20,000 · dropping the zeroing line held **20.5 MB** in a deque reporting `Len() == 0`

---

## 6. BFS, and why the container *is* the algorithm

`🟢 easy` · *one word apart from DFS*

Swap `PopFront` for `PopBack` and breadth-first search becomes depth-first search. Nothing else
changes — which means the shortest-path guarantee belongs to the queue, not to the traversal.

**Steps:**

1. Run the same twenty lines with each end and compare the visit orders.
2. Reconstruct shortest paths from the distances the queue produced.
3. Then measure the classic bug: marking visited on *dequeue*.

```go
package main

import (
	"fmt"
	"math/rand"
)

// Here is why the queue matters: BFS and DFS are THE SAME ALGORITHM with a
// different container. Swap the queue for a stack and the traversal order
// changes completely -- and so does what the result means.
//
// BFS with a queue finds SHORTEST PATHS on an unweighted graph. DFS with a
// stack finds *a* path. That difference is entirely the container's doing.

type graph [][]int

func (g graph) neighbors(v int) []int { return g[v] }

// bfs returns dist[v] = number of edges from src, and prev[v] for path
// reconstruction. -1 means unreachable.
func bfs(g graph, src int) (dist, prev []int) {
	dist = make([]int, len(g))
	prev = make([]int, len(g))
	for i := range dist {
		dist[i], prev[i] = -1, -1
	}

	q := NewDeque[int](8)
	dist[src] = 0
	q.PushBack(src)

	for q.Len() > 0 {
		v, _ := q.PopFront()
		for _, w := range g.neighbors(v) {
			if dist[w] != -1 { // MARK ON ENQUEUE -- see below
				continue
			}
			dist[w] = dist[v] + 1
			prev[w] = v
			q.PushBack(w)
		}
	}
	return dist, prev
}

// dfs is byte-for-byte bfs with PopFront changed to PopBack.
func dfs(g graph, src int) []int {
	order := []int{}
	seen := make([]bool, len(g))
	q := NewDeque[int](8)
	seen[src] = true
	q.PushBack(src)
	for q.Len() > 0 {
		v, _ := q.PopBack() // <- the only difference
		order = append(order, v)
		for _, w := range g.neighbors(v) {
			if seen[w] {
				continue
			}
			seen[w] = true
			q.PushBack(w)
		}
	}
	return order
}

func bfsOrder(g graph, src int) []int {
	order := []int{}
	seen := make([]bool, len(g))
	q := NewDeque[int](8)
	seen[src] = true
	q.PushBack(src)
	for q.Len() > 0 {
		v, _ := q.PopFront()
		order = append(order, v)
		for _, w := range g.neighbors(v) {
			if seen[w] {
				continue
			}
			seen[w] = true
			q.PushBack(w)
		}
	}
	return order
}

// bfsMarkOnDequeue is the classic bug: the visited check happens when a vertex
// comes OUT of the queue rather than when it goes in. The distance travels
// WITH the queue entry, so the answers stay correct -- it just queues a vertex
// once per incoming edge instead of once.
type entry struct{ v, d int }

func bfsMarkOnDequeue(g graph, src int) (dist []int, pushes int) {
	dist = make([]int, len(g))
	for i := range dist {
		dist[i] = -1
	}
	q := NewDeque[entry](8)
	q.PushBack(entry{src, 0})
	pushes = 1
	for q.Len() > 0 {
		e, _ := q.PopFront()
		if dist[e.v] != -1 {
			continue // already settled -- this vertex was queued more than once
		}
		dist[e.v] = e.d
		for _, w := range g.neighbors(e.v) {
			if dist[w] != -1 {
				continue
			}
			q.PushBack(entry{w, e.d + 1})
			pushes++
		}
	}
	return dist, pushes
}

// bfsSharedDist is the same bug PLUS the shortcut of keeping the tentative
// distance in an array instead of in the queue entry. Now it is not merely
// slow -- a later, longer tentative distance overwrites an earlier shorter one
// before the vertex is ever popped.
func bfsSharedDist(g graph, src int) []int {
	dist := make([]int, len(g))
	for i := range dist {
		dist[i] = -1
	}
	d := make([]int, len(g))
	q := NewDeque[int](8)
	q.PushBack(src)
	for q.Len() > 0 {
		v, _ := q.PopFront()
		if dist[v] != -1 {
			continue
		}
		dist[v] = d[v]
		for _, w := range g.neighbors(v) {
			if dist[w] != -1 {
				continue
			}
			d[w] = d[v] + 1
			q.PushBack(w)
		}
	}
	return dist
}

func bfsPushes(g graph, src int) int {
	seen := make([]bool, len(g))
	q := NewDeque[int](8)
	seen[src] = true
	q.PushBack(src)
	pushes := 1
	for q.Len() > 0 {
		v, _ := q.PopFront()
		for _, w := range g.neighbors(v) {
			if seen[w] {
				continue
			}
			seen[w] = true
			q.PushBack(w)
			pushes++
		}
	}
	return pushes
}

func path(prev []int, dst int) []int {
	var p []int
	for v := dst; v != -1; v = prev[v] {
		p = append(p, v)
	}
	for i, j := 0, len(p)-1; i < j; i, j = i+1, j-1 {
		p[i], p[j] = p[j], p[i]
	}
	return p
}

// randomGraph builds a connected undirected graph with n vertices.
func randomGraph(n, extra int, seed int64) graph {
	rng := rand.New(rand.NewSource(seed))
	g := make(graph, n)
	add := func(a, b int) {
		g[a] = append(g[a], b)
		g[b] = append(g[b], a)
	}
	for v := 1; v < n; v++ { // spanning tree, so it is connected
		add(v, rng.Intn(v))
	}
	for i := 0; i < extra; i++ {
		a, b := rng.Intn(n), rng.Intn(n)
		if a != b {
			add(a, b)
		}
	}
	return g
}

func main() {
	//        0
	//      / | \
	//     1  2  3
	//    /|     |
	//   4 5     6
	//           |
	//           7
	g := graph{
		0: {1, 2, 3},
		1: {0, 4, 5},
		2: {0},
		3: {0, 6},
		4: {1},
		5: {1},
		6: {3, 7},
		7: {6},
	}

	fmt.Println("the same twenty lines, one word different:")
	fmt.Println()
	fmt.Printf("  PopFront (queue) -> %v\n", bfsOrder(g, 0))
	fmt.Printf("  PopBack  (stack) -> %v\n", dfs(g, 0))
	fmt.Println()
	fmt.Println("  BFS visits 1, 2, 3 before any of their children: it explores in")
	fmt.Println("  RINGS of increasing distance. DFS dives to 7 before it has even")
	fmt.Println("  looked at 2. Neither is a special algorithm -- the container is")
	fmt.Println("  doing all the work.")

	dist, prev := bfs(g, 0)
	fmt.Println()
	fmt.Println("and because BFS explores in rings, dist[] is a SHORTEST path:")
	fmt.Println()
	fmt.Printf("  %-8s %-10s %s\n", "vertex", "distance", "path from 0")
	for v := range g {
		fmt.Printf("  %-8d %-10d %v\n", v, dist[v], path(prev, v))
	}
	fmt.Println()
	fmt.Println("  the queue is what guarantees this. Vertices come out in")
	fmt.Println("  non-decreasing distance order, so the FIRST time you reach a")
	fmt.Println("  vertex is necessarily by a shortest path. A stack gives you no")
	fmt.Println("  such promise -- DFS reached 7 in 4 steps down one branch, but")
	fmt.Println("  had the edges been listed differently it might have found a")
	fmt.Println("  longer route first and never revisited.")
	fmt.Println()
	fmt.Println("  this is Dijkstra with all weights equal to 1, and the queue is")
	fmt.Println("  standing in for the priority queue you would otherwise need")
	fmt.Println("  (lesson 20). Example 13 does the same trick for weights 0 and 1.")

	fmt.Println()
	fmt.Println("the bug that does not look like a bug -- marking on DEQUEUE:")
	fmt.Println()
	fmt.Printf("  %10s %8s %14s %14s %10s\n", "vertices", "edges", "mark on enq", "mark on deq", "ratio")
	wrongTotal, wrongN := 0, 0
	for _, n := range []int{1000, 10_000, 100_000} {
		gg := randomGraph(n, n*4, 7)
		edges := 0
		for _, adj := range gg {
			edges += len(adj)
		}
		good := bfsPushes(gg, 0)
		dd, bad := bfsMarkOnDequeue(gg, 0)
		ref, _ := bfs(gg, 0)
		for i := range ref { // correctness is proved, not assumed
			if ref[i] != dd[i] {
				panic("distances differ")
			}
		}
		shared := bfsSharedDist(gg, 0)
		for i := range ref {
			if ref[i] != shared[i] {
				wrongTotal++
			}
		}
		wrongN = n
		fmt.Printf("  %10d %8d %14d %14d %9.1fx\n", n, edges/2, good, bad, float64(bad)/float64(good))
	}
	fmt.Println()
	fmt.Println("  the distances are IDENTICAL -- verified against the correct BFS")
	fmt.Println("  above, not assumed. The broken version is correct and slow, which")
	fmt.Println("  is the worst kind of wrong: it queues a vertex once per incoming")
	fmt.Println("  edge instead of once, so the queue holds Theta(E) entries instead")
	fmt.Println("  of Theta(V), and nothing ever tells you.")
	fmt.Println()
	fmt.Println("  mark the vertex the moment you PUSH it, never when you pop it.")

	fmt.Println()
	fmt.Println("and it stays correct only because the distance rides in the QUEUE")
	fmt.Println("ENTRY. Move it to an array -- which looks like an obvious saving --")
	fmt.Println("and a later, longer tentative distance overwrites an earlier one:")
	fmt.Println()
	fmt.Printf("  vertices with the WRONG distance across the three graphs: %d\n", wrongTotal)
	fmt.Printf("  (largest graph had %d vertices)\n", wrongN)
	fmt.Println()
	fmt.Println("  I wrote that version first and asserted it was 'correct but slow'.")
	fmt.Println("  The check three lines above panicked on the first graph. Both")
	fmt.Println("  variants were plausible; only the measurement separated them.")
	fmt.Println()
	fmt.Println("  (the one algorithm where you genuinely must settle on POP is")
	fmt.Println("  Dijkstra with a priority queue, because a shorter path can still")
	fmt.Println("  arrive later. On an unweighted graph it cannot -- which is exactly")
	fmt.Println("  what the queue buys you.)")
}
```

> Reuses `Deque[T]` from example 5 — copy that type into the same folder as `deque.go`.

**Sample output:**

```
the same twenty lines, one word different:

  PopFront (queue) -> [0 1 2 3 4 5 6 7]
  PopBack  (stack) -> [0 3 6 7 2 1 5 4]

  BFS visits 1, 2, 3 before any of their children: it explores in
  RINGS of increasing distance. DFS dives to 7 before it has even
  looked at 2. Neither is a special algorithm -- the container is
  doing all the work.

and because BFS explores in rings, dist[] is a SHORTEST path:

  vertex   distance   path from 0
  0        0          [0]
  1        1          [0 1]
  2        1          [0 2]
  3        1          [0 3]
  4        2          [0 1 4]
  5        2          [0 1 5]
  6        2          [0 3 6]
  7        3          [0 3 6 7]

  the queue is what guarantees this. Vertices come out in
  non-decreasing distance order, so the FIRST time you reach a
  vertex is necessarily by a shortest path. A stack gives you no
  such promise -- DFS reached 7 in 4 steps down one branch, but
  had the edges been listed differently it might have found a
  longer route first and never revisited.

  this is Dijkstra with all weights equal to 1, and the queue is
  standing in for the priority queue you would otherwise need
  (lesson 20). Example 13 does the same trick for weights 0 and 1.

the bug that does not look like a bug -- marking on DEQUEUE:

    vertices    edges    mark on enq    mark on deq      ratio
        1000     4994           1000           4995       5.0x
       10000    49994          10000          49995       5.0x
      100000   499999         100000         500000       5.0x

  the distances are IDENTICAL -- verified against the correct BFS
  above, not assumed. The broken version is correct and slow, which
  is the worst kind of wrong: it queues a vertex once per incoming
  edge instead of once, so the queue holds Theta(E) entries instead
  of Theta(V), and nothing ever tells you.

  mark the vertex the moment you PUSH it, never when you pop it.

and it stays correct only because the distance rides in the QUEUE
ENTRY. Move it to an array -- which looks like an obvious saving --
and a later, longer tentative distance overwrites an earlier one:

  vertices with the WRONG distance across the three graphs: 95797
  (largest graph had 100000 vertices)

  I wrote that version first and asserted it was 'correct but slow'.
  The check three lines above panicked on the first graph. Both
  variants were plausible; only the measurement separated them.

  (the one algorithm where you genuinely must settle on POP is
  Dijkstra with a priority queue, because a shorter path can still
  arrive later. On an unweighted graph it cannot -- which is exactly
  what the queue buys you.)
```

**Rust:**

```rust
use std::fmt::Display;

// ---- RNG: stand-in for math/rand (the kit's splitmix64; seeded, so runs are reproducible) ----
// It is NOT Go's generator, so the random graphs below are different graphs from
// the Go run's, and every number that depends on the graph differs from Go's.
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

// Here is why the queue matters: BFS and DFS are THE SAME ALGORITHM with a
// different container. Swap the queue for a stack and the traversal order
// changes completely -- and so does what the result means.
//
// BFS with a queue finds SHORTEST PATHS on an unweighted graph. DFS with a
// stack finds *a* path. That difference is entirely the container's doing.

// ---- Deque<T> from example 5, trimmed to the methods BFS calls ----
// (clippy -D warnings rejects unused methods; std's VecDeque has all of these
// under the names push_back / pop_front / pop_back.)
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
}

// The graph is an adjacency list: g[v] lists v's neighbours. Go uses -1 for
// "no distance yet" / "no predecessor"; Rust has a type for that, Option<usize>.
type Dist = Vec<Option<usize>>;

/// bfs returns dist[v] = number of edges from src, and prev[v] for path
/// reconstruction. None means unreachable.
fn bfs(g: &[Vec<usize>], src: usize) -> (Dist, Dist) {
    let mut dist: Dist = vec![None; g.len()];
    let mut prev: Dist = vec![None; g.len()];

    let mut q = Deque::new(8);
    dist[src] = Some(0);
    q.push_back(src);

    // Go: `for q.Len() > 0 { v, _ := q.PopFront() ... }`. The Rust loop is the
    // Option itself: it ends when the pop returns None.
    while let Some(v) = q.pop_front() {
        for &w in &g[v] {
            if dist[w].is_some() {
                // MARK ON ENQUEUE -- see below
                continue;
            }
            // dist[v] is Some here; map adds one without unwrapping it
            dist[w] = dist[v].map(|d| d + 1);
            prev[w] = Some(v);
            q.push_back(w);
        }
    }
    (dist, prev)
}

/// dfs is byte-for-byte bfs with pop_front changed to pop_back.
fn dfs(g: &[Vec<usize>], src: usize) -> Vec<usize> {
    let mut order = Vec::new();
    let mut seen = vec![false; g.len()];
    let mut q = Deque::new(8);
    seen[src] = true;
    q.push_back(src);
    while let Some(v) = q.pop_back() {
        // <- the only difference
        order.push(v);
        for &w in &g[v] {
            if seen[w] {
                continue;
            }
            seen[w] = true;
            q.push_back(w);
        }
    }
    order
}

fn bfs_order(g: &[Vec<usize>], src: usize) -> Vec<usize> {
    let mut order = Vec::new();
    let mut seen = vec![false; g.len()];
    let mut q = Deque::new(8);
    seen[src] = true;
    q.push_back(src);
    while let Some(v) = q.pop_front() {
        order.push(v);
        for &w in &g[v] {
            if seen[w] {
                continue;
            }
            seen[w] = true;
            q.push_back(w);
        }
    }
    order
}

/// bfs_mark_on_dequeue is the classic bug: the visited check happens when a
/// vertex comes OUT of the queue rather than when it goes in. The distance
/// travels WITH the queue entry, so the answers stay correct -- it just queues
/// a vertex once per incoming edge instead of once.
struct Entry {
    v: usize,
    d: usize,
}

fn bfs_mark_on_dequeue(g: &[Vec<usize>], src: usize) -> (Dist, usize) {
    let mut dist: Dist = vec![None; g.len()];
    let mut q = Deque::new(8);
    q.push_back(Entry { v: src, d: 0 });
    let mut pushes = 1;
    while let Some(e) = q.pop_front() {
        if dist[e.v].is_some() {
            continue; // already settled -- this vertex was queued more than once
        }
        dist[e.v] = Some(e.d);
        for &w in &g[e.v] {
            if dist[w].is_some() {
                continue;
            }
            q.push_back(Entry { v: w, d: e.d + 1 });
            pushes += 1;
        }
    }
    (dist, pushes)
}

/// bfs_shared_dist is the same bug PLUS the shortcut of keeping the tentative
/// distance in an array instead of in the queue entry. Now it is not merely
/// slow -- a later, longer tentative distance overwrites an earlier shorter one
/// before the vertex is ever popped.
fn bfs_shared_dist(g: &[Vec<usize>], src: usize) -> Dist {
    let mut dist: Dist = vec![None; g.len()];
    let mut d = vec![0; g.len()];
    let mut q = Deque::new(8);
    q.push_back(src);
    while let Some(v) = q.pop_front() {
        if dist[v].is_some() {
            continue;
        }
        dist[v] = Some(d[v]);
        for &w in &g[v] {
            if dist[w].is_some() {
                continue;
            }
            d[w] = d[v] + 1;
            q.push_back(w);
        }
    }
    dist
}

fn bfs_pushes(g: &[Vec<usize>], src: usize) -> usize {
    let mut seen = vec![false; g.len()];
    let mut q = Deque::new(8);
    seen[src] = true;
    q.push_back(src);
    let mut pushes = 1;
    while let Some(v) = q.pop_front() {
        for &w in &g[v] {
            if seen[w] {
                continue;
            }
            seen[w] = true;
            q.push_back(w);
            pushes += 1;
        }
    }
    pushes
}

fn path(prev: &[Option<usize>], dst: usize) -> Vec<usize> {
    let mut p = Vec::new();
    let mut cur = Some(dst);
    while let Some(v) = cur {
        p.push(v);
        cur = prev[v];
    }
    p.reverse();
    p
}

fn add_edge(g: &mut [Vec<usize>], a: usize, b: usize) {
    g[a].push(b);
    g[b].push(a);
}

/// random_graph builds a connected undirected graph with n vertices.
fn random_graph(n: usize, extra: usize, seed: u64) -> Vec<Vec<usize>> {
    let mut rng = Rng::new(seed);
    let mut g = vec![Vec::new(); n];
    for v in 1..n {
        // spanning tree, so it is connected
        let parent = rng.below(v as u64) as usize;
        add_edge(&mut g, v, parent);
    }
    for _ in 0..extra {
        let (a, b) = (rng.below(n as u64) as usize, rng.below(n as u64) as usize);
        if a != b {
            add_edge(&mut g, a, b);
        }
    }
    g
}

/// Go's `%v` of a slice: [1 2 3]
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(ToString::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    //        0
    //      / | \
    //     1  2  3
    //    /|     |
    //   4 5     6
    //           |
    //           7
    let g = vec![
        vec![1, 2, 3],
        vec![0, 4, 5],
        vec![0],
        vec![0, 6],
        vec![1],
        vec![1],
        vec![3, 7],
        vec![6],
    ];

    println!("the same twenty lines, one word different:");
    println!();
    println!("  pop_front (queue) -> {}", show(&bfs_order(&g, 0)));
    println!("  pop_back  (stack) -> {}", show(&dfs(&g, 0)));
    println!();
    println!("  BFS visits 1, 2, 3 before any of their children: it explores in");
    println!("  RINGS of increasing distance. DFS dives to 7 before it has even");
    println!("  looked at 2. Neither is a special algorithm -- the container is");
    println!("  doing all the work.");

    let (dist, prev) = bfs(&g, 0);
    println!();
    println!("and because BFS explores in rings, dist[] is a SHORTEST path:");
    println!();
    println!("  {:<8} {:<10} path from 0", "vertex", "distance");
    for (v, d) in dist.iter().enumerate() {
        // the graph is connected, so every distance is Some
        let d = d.expect("connected graph");
        println!("  {:<8} {:<10} {}", v, d, show(&path(&prev, v)));
    }
    println!();
    println!("  the queue is what guarantees this. Vertices come out in");
    println!("  non-decreasing distance order, so the FIRST time you reach a");
    println!("  vertex is necessarily by a shortest path. A stack gives you no");
    println!("  such promise -- DFS reached 7 in 4 steps down one branch, but");
    println!("  had the edges been listed differently it might have found a");
    println!("  longer route first and never revisited.");
    println!();
    println!("  this is Dijkstra with all weights equal to 1, and the queue is");
    println!("  standing in for the priority queue you would otherwise need");
    println!("  (lesson 20). Example 13 does the same trick for weights 0 and 1.");

    println!();
    println!("the bug that does not look like a bug -- marking on DEQUEUE:");
    println!();
    println!(
        "  {:>10} {:>8} {:>14} {:>14} {:>10}",
        "vertices", "edges", "mark on enq", "mark on deq", "ratio"
    );
    let mut wrong_by_graph = Vec::new();
    let mut wrong_n = 0;
    for n in [1000, 10_000, 100_000] {
        let gg = random_graph(n, n * 4, 7);
        let edges: usize = gg.iter().map(Vec::len).sum();
        let good = bfs_pushes(&gg, 0);
        let (dd, bad) = bfs_mark_on_dequeue(&gg, 0);
        let (reference, _) = bfs(&gg, 0);
        // correctness is proved, not assumed
        assert!(reference == dd, "distances differ");
        // and the push count is exact, not just 'about E': every undirected
        // edge pushes once (from the endpoint that settles first), plus the source
        assert_eq!(bad, edges / 2 + 1);
        let shared = bfs_shared_dist(&gg, 0);
        let wrong = reference
            .iter()
            .zip(&shared)
            .filter(|(a, b)| a != b)
            .count();
        wrong_by_graph.push(wrong);
        wrong_n = n;
        println!(
            "  {:>10} {:>8} {:>14} {:>14} {:>9.1}x",
            n,
            edges / 2,
            good,
            bad,
            bad as f64 / good as f64
        );
    }
    println!();
    println!("  the distances are IDENTICAL -- verified against the correct BFS");
    println!("  above (the program asserts it), not assumed. The broken version is");
    println!("  correct and slow, which is the worst kind of wrong: it queues one");
    println!("  entry per EDGE instead of one per vertex, so the queue holds");
    println!("  Theta(E) entries instead of Theta(V), and nothing ever tells you.");
    println!("  The push count is exactly edges + 1 in every row (also asserted):");
    println!("  each edge pushes once, from whichever endpoint settles first.");
    println!();
    println!("  mark the vertex the moment you PUSH it, never when you pop it.");

    println!();
    println!("and it stays correct only because the distance rides in the QUEUE");
    println!("ENTRY. Move it to an array -- which looks like an obvious saving --");
    println!("and a later, longer tentative distance overwrites an earlier one:");
    println!();
    println!(
        "  vertices with the WRONG distance across the three graphs: {}",
        wrong_by_graph.iter().sum::<usize>()
    );
    println!("  (largest graph had {wrong_n} vertices)");
    println!(
        "  per graph: {}",
        wrong_by_graph
            .iter()
            .zip([1000, 10_000, 100_000])
            .map(|(w, n)| format!("{w} of {n}"))
            .collect::<Vec<_>>()
            .join(", ")
    );
    println!();
    println!("  the check above, applied to this variant, would have panicked on the");
    println!(
        "  first graph: {} of its 1000 vertices already have the wrong distance.",
        wrong_by_graph[0]
    );
    println!("  Both variants are plausible; only the measurement separates them.");
    println!();
    println!("  (the one algorithm where you genuinely must settle on POP is");
    println!("  Dijkstra with a priority queue, because a shorter path can still");
    println!("  arrive later. On an unweighted graph it cannot -- which is exactly");
    println!("  what the queue buys you.)");
}
```

**Rust sample output:**

```
the same twenty lines, one word different:

  pop_front (queue) -> [0 1 2 3 4 5 6 7]
  pop_back  (stack) -> [0 3 6 7 2 1 5 4]

  BFS visits 1, 2, 3 before any of their children: it explores in
  RINGS of increasing distance. DFS dives to 7 before it has even
  looked at 2. Neither is a special algorithm -- the container is
  doing all the work.

and because BFS explores in rings, dist[] is a SHORTEST path:

  vertex   distance   path from 0
  0        0          [0]
  1        1          [0 1]
  2        1          [0 2]
  3        1          [0 3]
  4        2          [0 1 4]
  5        2          [0 1 5]
  6        2          [0 3 6]
  7        3          [0 3 6 7]

  the queue is what guarantees this. Vertices come out in
  non-decreasing distance order, so the FIRST time you reach a
  vertex is necessarily by a shortest path. A stack gives you no
  such promise -- DFS reached 7 in 4 steps down one branch, but
  had the edges been listed differently it might have found a
  longer route first and never revisited.

  this is Dijkstra with all weights equal to 1, and the queue is
  standing in for the priority queue you would otherwise need
  (lesson 20). Example 13 does the same trick for weights 0 and 1.

the bug that does not look like a bug -- marking on DEQUEUE:

    vertices    edges    mark on enq    mark on deq      ratio
        1000     4990           1000           4991       5.0x
       10000    49994          10000          49995       5.0x
      100000   499995         100000         499996       5.0x

  the distances are IDENTICAL -- verified against the correct BFS
  above (the program asserts it), not assumed. The broken version is
  correct and slow, which is the worst kind of wrong: it queues one
  entry per EDGE instead of one per vertex, so the queue holds
  Theta(E) entries instead of Theta(V), and nothing ever tells you.
  The push count is exactly edges + 1 in every row (also asserted):
  each edge pushes once, from whichever endpoint settles first.

  mark the vertex the moment you PUSH it, never when you pop it.

and it stays correct only because the distance rides in the QUEUE
ENTRY. Move it to an array -- which looks like an obvious saving --
and a later, longer tentative distance overwrites an earlier one:

  vertices with the WRONG distance across the three graphs: 95586
  (largest graph had 100000 vertices)
  per graph: 862 of 1000, 8527 of 10000, 86197 of 100000

  the check above, applied to this variant, would have panicked on the
  first graph: 862 of its 1000 vertices already have the wrong distance.
  Both variants are plausible; only the measurement separates them.

  (the one algorithm where you genuinely must settle on POP is
  Dijkstra with a priority queue, because a shorter path can still
  arrive later. On an unweighted graph it cannot -- which is exactly
  what the queue buys you.)
```

> **Go vs Rust:** Traversal orders, distances and paths are identical to Go's; `Option<usize>` replaces the `-1` sentinel and `while let Some(v) = q.pop_front()` replaces `for q.Len() > 0`. The random graphs differ, because the kit's splitmix64 is not Go's `math/rand` (4990 edges at n=1000 against Go's 4994), so the table digits and the wrong-distance count (95,586 here, 95,797 in Go) are Rust's own; the structure carries over: mark-on-dequeue pushed exactly edges + 1 in every row (asserted), and the shared-array variant was wrong on 862 of the first graph's 1000 vertices.

**Complexity:** Θ(V + E) with marking on enqueue · marking on dequeue is still **correct** and queues
**5× more** entries — Θ(E) instead of Θ(V) · moving the tentative distance out of the queue entry into
an array makes it *wrong*, on **95,797** vertices here

---

> Next tier: [🟡 medium](2-medium.md) — what the standard library gives you, what a channel really is,
> and what to do when the queue is full.
