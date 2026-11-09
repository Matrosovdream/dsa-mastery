# Step 07 — Linked Lists · 🟡 Medium

Examples **7–11**: reverse and the recursion that dies, middle and Nth-from-end with two pointers, Floyd's
cycle detection, merging with zero allocations, and Go's `container/list`.

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

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Reverse in three lines, and the recursion that dies

`🟡 medium` · *save, flip, advance — and Θ(n) stack*

Iterative reverse is three lines — save `next`, flip `cur.next` to `prev`, advance both — and the first one is
the line people forget: flip before you save and the only pointer to the rest of the list is gone, with no panic
and no error. It is Θ(n) time, Θ(1) space and exactly one link write per node. The recursive version gives the
same answer and is elegant, but it holds one stack frame per node — Θ(n) stack — so this example measures the
frame and *predicts* the length that kills it instead of crashing. `reverseBetween(m, n)` is the same three lines
run over a window, plus two reconnections, and a sentinel makes `m = 1` ordinary.

**Steps:**

1. Trace the iterative reverse on 1..5 and count the link writes; then run the version without the save.
2. Measure the recursive version's bytes per frame (two depths in one already-grown stack) and divide Go's stack limit by it.
3. Check `reverseBetween` against a slice oracle for every window, and watch the GC free 2,000,000 nodes with no recursion.

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
	"slices"
	"strings"
	"unsafe"
)

type node struct {
	val  int
	next *node
}

func buildRange(n int) *node { // 1..n
	var head *node
	for i := n; i >= 1; i-- {
		head = &node{i, head}
	}
	return head
}

func show(head *node) string { return showLimit(head, -1) }

// showLimit stops after limit nodes (limit < 0: no limit) so a broken list cannot hang us.
func showLimit(head *node, limit int) string {
	var b strings.Builder
	b.WriteString("[")
	for n, i := head, 0; n != nil; n, i = n.next, i+1 {
		if i == limit {
			b.WriteString(" ...")
			break
		}
		if i > 0 {
			b.WriteString(" -> ")
		}
		fmt.Fprint(&b, n.val)
	}
	b.WriteString("]")
	return b.String()
}

func toSlice(head *node) []int {
	var out []int
	for n := head; n != nil; n = n.next {
		out = append(out, n.val)
	}
	return out
}

func valOf(n *node) string {
	if n == nil {
		return "nil"
	}
	return fmt.Sprint(n.val)
}

// reverse is the three lines: SAVE next, FLIP the link, ADVANCE both pointers.
// It returns the number of link writes so the Θ(n) claim is counted, not asserted.
func reverse(head *node, trace bool) (*node, int) {
	var prev *node
	cur := head
	writes := 0
	for step := 1; cur != nil; step++ {
		next := cur.next // 1. SAVE: after the next line this is the only way to reach the rest
		cur.next = prev  // 2. FLIP
		writes++
		if trace {
			fmt.Printf("  step %d: next=%-3s cur=%d now points at %-3s -> prev=%d cur=%s\n",
				step, valOf(next), cur.val, valOf(prev), cur.val, valOf(next))
		}
		prev, cur = cur, next // 3. ADVANCE
	}
	return prev, writes
}

// reverseNoSave is the bug: the link is flipped before it was saved, so "advance via
// cur.next" follows the NEW link -- back to prev -- instead of forward.
func reverseNoSave(head *node) *node {
	var prev *node
	cur := head
	for cur != nil {
		cur.next = prev // the old cur.next is gone: nothing else points at node 2
		prev = cur
		cur = cur.next // == prev's old value (nil on the first step): the walk ends
	}
	return prev
}

// probeSP is called at the base case of the recursion. It records the address of one of
// its own locals: a stand-in for the stack pointer. //go:noinline keeps it a real frame.
var spAtBase uintptr
var stackInuseAtBase uint64

//go:noinline
func probeSP() {
	var x byte
	spAtBase = uintptr(unsafe.Pointer(&x)) // uintptr: the GC does not track it
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	stackInuseAtBase = ms.StackInuse
}

// reverseRec: reverse the rest, then hang this node on the end of it. h.next still
// points at what is now the LAST node of the reversed rest, so the fix-up is O(1).
// One stack frame per node -- the call is not the last thing the function does.
func reverseRec(h *node) *node {
	if h == nil || h.next == nil {
		probeSP()
		return h
	}
	newHead := reverseRec(h.next)
	h.next.next = h
	h.next = nil
	return newHead
}

// reverseBetween reverses positions m..n (1-based, inclusive) in one pass.
// The sentinel makes m == 1 an ordinary case.
func reverseBetween(head *node, m, n int) *node {
	sentinel := &node{next: head}
	before := sentinel
	for i := 1; i < m; i++ { // walk to the node BEFORE position m
		before = before.next
	}
	first := before.next // becomes the last node of the reversed segment
	var prev *node
	cur := first
	for i := m; i <= n; i++ { // the same three lines, n-m+1 times
		next := cur.next
		cur.next = prev
		prev, cur = cur, next
	}
	before.next = prev // reconnection 1: the node before m -> the old n-th node
	first.next = cur   // reconnection 2: the old m-th node -> the node after n
	return sentinel.next
}

func main() {
	fmt.Println("== 1. iterative reverse of 1..5, traced ==")
	head, writes := reverse(buildRange(5), true)
	fmt.Printf("  result %s, %d link writes for 5 nodes\n", show(head), writes)
	for _, n := range []int{10, 1000, 100000} {
		_, w := reverse(buildRange(n), false)
		fmt.Printf("  n = %-6d link writes = %d\n", n, w)
	}

	fmt.Println()
	fmt.Println("== 2. forget the SAVE ==")
	orig := buildRange(5)
	second := orig.next // keep a handle only so we can show what was lost
	bad := reverseNoSave(orig)
	fmt.Printf("  result %s (bounded walk of 12)\n", showLimit(bad, 12))
	fmt.Printf("  nodes now reachable from the result: %d of 5; %s is reachable only through\n",
		len(toSlice(bad)), show(second))
	fmt.Println("  our debug handle -- without it, those 4 nodes are garbage. No panic, no error.")

	fmt.Println()
	fmt.Println("== 3. recursive reverse: same answer, one stack frame per node ==")
	r := reverseRec(buildRange(5))
	fmt.Printf("  reverseRec(1..5) = %s\n", show(r))

	// Measure bytes per frame. A goroutine stack is contiguous but MOVES when it grows
	// (it is copied to one twice the size), so addresses are only comparable if no growth
	// happens in between. Recipe: grow the stack once with the deepest run, turn the GC off
	// (the GC is what shrinks stacks), then measure two depths in the same big stack.
	const n1, n2 = 10_000, 100_000
	big1, big2 := buildRange(n1), buildRange(n2)
	old := debug.SetGCPercent(-1)
	reverseRec(buildRange(n2)) // warm-up: grows this goroutine's stack to fit n2 frames
	big1 = reverseRec(big1)
	sp1, inuse1 := spAtBase, stackInuseAtBase
	big2 = reverseRec(big2)
	sp2, inuse2 := spAtBase, stackInuseAtBase
	debug.SetGCPercent(old)
	perFrame := float64(sp1-sp2) / float64(n2-n1)
	fmt.Printf("  base-case address, depth %d vs %d: %d bytes apart -> %.1f bytes per frame\n",
		n1, n2, sp1-sp2, perFrame)
	fmt.Printf("  MemStats.StackInuse at the base case: %.2f MiB (n=%d), %.2f MiB (n=%d)\n",
		float64(inuse1)/(1<<20), n1, float64(inuse2)/(1<<20), n2)
	if inuse1 == inuse2 { // the warm-up already grew the stack
		fmt.Println("  (identical: StackInuse counts whole power-of-two stacks, so it cannot")
		fmt.Println("  resolve a frame; the address difference can)")
	}
	// runtime/proc.go: maxstacksize = 1000000000 on 64-bit. runtime/stack.go (newstack):
	// newsize := oldsize * 2; if newsize > maxstacksize { throw("stack overflow") }.
	// Stacks are powers of two, so the biggest legal stack is 512 MiB, not 1 GB.
	const maxStack, largest = 1_000_000_000, 1 << 29 // 2^29 <= 1e9 < 2^30
	fmt.Printf("  Go max stack: %d bytes; largest power of two under it: %d MiB\n", maxStack, largest>>20)
	fmt.Printf("  PREDICTED: reverseRec dies near n = %.1f million nodes (%d MiB / %.1f B)\n",
		float64(largest)/perFrame/1e6, largest>>20, perFrame)
	fmt.Println("  (not triggered: the crash is a fatal 'goroutine stack exceeds' error, not a panic)")
	fmt.Printf("  check: %d-node result starts %d, %d-node result starts %d\n",
		n1, big1.val, n2, big2.val)

	fmt.Println()
	fmt.Println("== 4. reverseBetween: the same three lines plus two reconnections ==")
	for _, c := range [][3]int{{7, 2, 4}, {7, 1, 7}, {7, 1, 3}, {7, 5, 7}, {7, 4, 4}, {1, 1, 1}} {
		fmt.Printf("  reverseBetween(1..%d, %d, %d) = %s\n", c[0], c[1], c[2],
			show(reverseBetween(buildRange(c[0]), c[1], c[2])))
	}
	cases, mismatches := 0, 0
	const L = 8
	for m := 1; m <= L; m++ {
		for n := m; n <= L; n++ {
			want := toSlice(buildRange(L))
			slices.Reverse(want[m-1 : n]) // the slice oracle
			got := toSlice(reverseBetween(buildRange(L), m, n))
			cases++
			if !slices.Equal(got, want) {
				mismatches++
			}
		}
	}
	fmt.Printf("  every 1 <= m <= n <= %d on 1..%d: %d cases, %d mismatches\n", L, L, cases, mismatches)

	fmt.Println()
	fmt.Println("== 5. freeing a 2,000,000-node list ==")
	const big = 2_000_000
	var ms runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&ms)
	before := ms.HeapObjects
	list := buildRange(big)
	runtime.ReadMemStats(&ms)
	built := ms.HeapObjects
	fmt.Printf("  built: heap objects %d -> %d (+%d)\n", before, built, int64(built)-int64(before))
	list = nil // drop the only reference to the head
	_ = list
	runtime.GC()
	runtime.ReadMemStats(&ms)
	fmt.Printf("  head = nil; runtime.GC(): heap objects %d -> %d (-%d)\n",
		built, ms.HeapObjects, int64(built)-int64(ms.HeapObjects))
	fmt.Println("  the GC marks from roots and sweeps spans: no recursion and no loop of yours.")
}
```

**Sample output:**

```
== 1. iterative reverse of 1..5, traced ==
  step 1: next=2   cur=1 now points at nil -> prev=1 cur=2
  step 2: next=3   cur=2 now points at 1   -> prev=2 cur=3
  step 3: next=4   cur=3 now points at 2   -> prev=3 cur=4
  step 4: next=5   cur=4 now points at 3   -> prev=4 cur=5
  step 5: next=nil cur=5 now points at 4   -> prev=5 cur=nil
  result [5 -> 4 -> 3 -> 2 -> 1], 5 link writes for 5 nodes
  n = 10     link writes = 10
  n = 1000   link writes = 1000
  n = 100000 link writes = 100000

== 2. forget the SAVE ==
  result [1] (bounded walk of 12)
  nodes now reachable from the result: 1 of 5; [2 -> 3 -> 4 -> 5] is reachable only through
  our debug handle -- without it, those 4 nodes are garbage. No panic, no error.

== 3. recursive reverse: same answer, one stack frame per node ==
  reverseRec(1..5) = [5 -> 4 -> 3 -> 2 -> 1]
  base-case address, depth 10000 vs 100000: 2880000 bytes apart -> 32.0 bytes per frame
  MemStats.StackInuse at the base case: 4.22 MiB (n=10000), 4.22 MiB (n=100000)
  (identical: StackInuse counts whole power-of-two stacks, so it cannot
  resolve a frame; the address difference can)
  Go max stack: 1000000000 bytes; largest power of two under it: 512 MiB
  PREDICTED: reverseRec dies near n = 16.8 million nodes (512 MiB / 32.0 B)
  (not triggered: the crash is a fatal 'goroutine stack exceeds' error, not a panic)
  check: 10000-node result starts 10000, 100000-node result starts 100000

== 4. reverseBetween: the same three lines plus two reconnections ==
  reverseBetween(1..7, 2, 4) = [1 -> 4 -> 3 -> 2 -> 5 -> 6 -> 7]
  reverseBetween(1..7, 1, 7) = [7 -> 6 -> 5 -> 4 -> 3 -> 2 -> 1]
  reverseBetween(1..7, 1, 3) = [3 -> 2 -> 1 -> 4 -> 5 -> 6 -> 7]
  reverseBetween(1..7, 5, 7) = [1 -> 2 -> 3 -> 4 -> 7 -> 6 -> 5]
  reverseBetween(1..7, 4, 4) = [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7]
  reverseBetween(1..1, 1, 1) = [1]
  every 1 <= m <= n <= 8 on 1..8: 36 cases, 0 mismatches

== 5. freeing a 2,000,000-node list ==
  built: heap objects 177 -> 2000181 (+2000004)
  head = nil; runtime.GC(): heap objects 2000181 -> 185 (-1999996)
  the GC marks from roots and sweeps spans: no recursion and no loop of yours.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::mem;
use std::process::Command;
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering::Relaxed};
use std::thread;

// ---- counting allocator: allocations, frees, and (when TRACK is on) the stack address
// of every free. The last one is how we measure the drop recursion of a PLAIN node type:
// in a recursive drop, node k is freed from a frame k levels deep.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static FREES: AtomicUsize = AtomicUsize::new(0);
static LIVE: AtomicUsize = AtomicUsize::new(0);
static TRACK: AtomicBool = AtomicBool::new(false);
static FREE_SP_MIN: AtomicUsize = AtomicUsize::new(usize::MAX);
static FREE_SP_MAX: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        FREES.fetch_add(1, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        if TRACK.load(Relaxed) {
            let sp = stack_addr();
            FREE_SP_MIN.fetch_min(sp, Relaxed);
            FREE_SP_MAX.fetch_max(sp, Relaxed);
        }
        unsafe { System.dealloc(p, l) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// The address of a local in this (never inlined) frame: a stand-in for the stack pointer.
/// Turning a reference into a usize needs no `unsafe`; only dereferencing it would.
#[inline(never)]
fn stack_addr() -> usize {
    let x = 0u8;
    black_box(&x) as *const u8 as usize
}

struct Node {
    val: i64,
    next: Link,
}
type Link = Option<Box<Node>>; // None is the empty list (and the null pointer)

fn build_range(n: i64) -> Link {
    // 1..n
    let mut head = None;
    for v in (1..=n).rev() {
        head = Some(Box::new(Node { val: v, next: head }));
    }
    head
}

/// A list that frees itself with a LOOP. Without this, dropping a long `Link` recurses
/// once per node (section 5) -- the recursion you did not write.
struct List(Link);
impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.0.take();
        while let Some(mut n) = cur {
            cur = n.next.take(); // detach the rest first, then `n` drops with next == None
        }
    }
}

fn show(head: &Link) -> String {
    show_limit(head, usize::MAX)
}

fn show_limit(head: &Link, limit: usize) -> String {
    let mut parts = Vec::new();
    let mut cur = head;
    while let Some(n) = cur {
        if parts.len() == limit {
            return format!("[{} ...]", parts.join(" -> "));
        }
        parts.push(n.val.to_string());
        cur = &n.next;
    }
    format!("[{}]", parts.join(" -> "))
}

fn to_vec(head: &Link) -> Vec<i64> {
    let mut out = Vec::new();
    let mut cur = head;
    while let Some(n) = cur {
        out.push(n.val);
        cur = &n.next;
    }
    out
}

fn val_of(l: &Link) -> String {
    l.as_ref().map_or("nil".to_string(), |n| n.val.to_string())
}

/// The three lines -- and in Rust the first two are ONE expression:
/// `mem::replace(&mut node.next, prev)` writes `prev` into the link and hands back the old
/// value. You cannot forget the save, because the save is the return value.
fn reverse(mut cur: Link, trace: bool) -> (Link, usize) {
    let mut prev: Link = None;
    let mut writes = 0;
    let mut step = 1;
    while let Some(mut node) = cur {
        let (next_v, prev_v, me) = (val_of(&node.next), val_of(&prev), node.val);
        cur = mem::replace(&mut node.next, prev); // SAVE + FLIP
        writes += 1;
        if trace {
            println!(
                "  step {step}: next={next_v:<3} cur={me} now points at {prev_v:<3} -> prev={me} cur={}",
                val_of(&cur)
            );
        }
        prev = Some(node); // ADVANCE (cur was already advanced by the replace)
        step += 1;
    }
    (prev, writes)
}

/// The bug, as Rust lets you write it: assign the link before saving it. The assignment
/// does not orphan the rest of the list -- it DROPS it, right there (Box has one owner).
fn reverse_no_save(mut cur: Link) -> Link {
    let mut prev: Link = None;
    while let Some(mut node) = cur {
        node.next = prev; // old node.next (2 -> 3 -> 4 -> 5) is freed by this assignment
        cur = node.next.take(); // "advance via cur.next" follows the NEW link: back to prev
        prev = Some(node);
    }
    prev
}

/// Recursive reverse, head-recursive like Go's: reverse the rest first, then hang `h` on
/// its end. Go writes `h.next.next = h`, which needs two pointers to one node; here the
/// recursion returns the empty slot at the end of what it built (`&mut Link`), and
/// `Option::insert` fills it and returns the new node. Work after the call => one frame
/// per node, no tail-call elimination possible.
fn rev_rec(mut h: Box<Node>, out: &mut Link) -> &mut Link {
    match h.next.take() {
        None => {
            black_box(stack_addr_at_base());
            &mut out.insert(h).next
        }
        Some(rest) => {
            let slot = rev_rec(rest, out);
            &mut slot.insert(h).next
        }
    }
}

// rustc has no package-level mutable variable without unsafe or atomics: an atomic it is.
static SP_AT_BASE: AtomicUsize = AtomicUsize::new(0);
#[inline(never)]
fn stack_addr_at_base() -> usize {
    let sp = stack_addr();
    SP_AT_BASE.store(sp, Relaxed);
    sp
}

fn reverse_rec(head: Link) -> Link {
    let mut out = None;
    if let Some(h) = head {
        rev_rec(h, &mut out);
    }
    out
}

/// Detach everything after the first `k` nodes (the `&mut Link` cursor walk).
fn split_after(list: &mut Link, k: usize) -> Link {
    let mut cur = list;
    for _ in 0..k {
        cur = &mut cur.as_mut().unwrap().next;
    }
    cur.take()
}

/// reverseBetween(m, n), 1-based inclusive. Go keeps `first` (the old m-th node) as a
/// second pointer while the loop owns it through the chain, then writes first.next = cur.
/// Rust cannot hold that alias, so it cuts the window off first (an extra walk of n-m
/// hops) and reverses it ONTO the rest: the very first link written points at the rest,
/// which is reconnection 2 for free. Reconnection 1 is `*before = reversed`.
fn reverse_between(head: Link, m: usize, n: usize) -> Link {
    let mut sentinel = Box::new(Node { val: 0, next: head });
    let mut before = &mut sentinel.next; // the LINK before position m
    for _ in 1..m {
        before = &mut before.as_mut().unwrap().next;
    }
    let mut window = before.take();
    let rest = split_after(&mut window, n - m + 1);
    let mut prev = rest; // reverse onto the rest
    let mut cur = window;
    while let Some(mut node) = cur {
        cur = mem::replace(&mut node.next, prev);
        prev = Some(node);
    }
    *before = prev;
    sentinel.next.take()
}

fn main() {
    if std::env::args().any(|a| a == "--overflow") {
        overflow();
        return;
    }

    println!("== 1. iterative reverse of 1..5, traced ==");
    let (head, writes) = reverse(build_range(5), true);
    println!("  result {}, {writes} link writes for 5 nodes", show(&head));
    for n in [10, 1000, 100000] {
        let (r, w) = reverse(build_range(n), false);
        drop(List(r));
        println!("  n = {n:<6} link writes = {w}");
    }

    println!();
    println!("== 2. forget the SAVE ==");
    let orig = build_range(5);
    let frees0 = FREES.load(Relaxed);
    let bad = reverse_no_save(orig);
    let freed = FREES.load(Relaxed) - frees0;
    println!("  result {} (bounded walk of 12)", show_limit(&bad, 12));
    println!(
        "  nodes now reachable from the result: {} of 5; the other {freed} were FREED by the",
        to_vec(&bad).len()
    );
    println!("  assignment `node.next = prev` (dealloc count +{freed}). No debug handle can keep");
    println!("  them: a second pointer to node 2 would be a second owner. No panic, no error.");

    println!();
    println!("== 3. recursive reverse: same answer, one stack frame per node ==");
    println!(
        "  reverse_rec(1..5) = {}",
        show(&reverse_rec(build_range(5)))
    );
    // A Rust thread stack never moves, so no warm-up is needed: two depths, two base-case
    // addresses. We measure on a thread with a 64 MiB stack (memory is only committed as
    // pages are touched) so 100,000 frames are nowhere near its end.
    const N1: i64 = 10_000;
    const N2: i64 = 100_000;
    let (sp1, sp2, s1, s2) = thread::Builder::new()
        .stack_size(64 << 20)
        .spawn(|| {
            let r1 = List(reverse_rec(build_range(N1)));
            let sp1 = SP_AT_BASE.load(Relaxed);
            let r2 = List(reverse_rec(build_range(N2)));
            let sp2 = SP_AT_BASE.load(Relaxed);
            (
                sp1,
                sp2,
                r1.0.as_ref().unwrap().val,
                r2.0.as_ref().unwrap().val,
            )
        })
        .unwrap()
        .join()
        .unwrap();
    let per_frame = (sp1 - sp2) as f64 / (N2 - N1) as f64;
    println!(
        "  base-case address, depth {N1} vs {N2}: {} bytes apart -> {per_frame:.1} bytes per frame",
        sp1 - sp2
    );
    // The main thread's stack is set by the OS, not by Rust (std::thread docs); read it.
    let main_kib = main_stack_kib();
    let main_bytes = main_kib * 1024.0;
    println!("  main thread stack (ulimit -s): {main_kib} KiB; spawned threads default to 2 MiB");
    println!(
        "  PREDICTED: reverse_rec dies near n = {:.0} on the main thread, {:.0} on a 2 MiB thread",
        main_bytes / per_frame,
        (2 << 20) as f64 / per_frame
    );
    println!(
        "  (not triggered: a Rust stack overflow aborts the process -- no unwinding, no catch)"
    );
    println!("  check: {N1}-node result starts {s1}, {N2}-node result starts {s2}");

    println!();
    println!("== 4. reverseBetween: the same three lines plus two reconnections ==");
    for (len, m, n) in [
        (7, 2, 4),
        (7, 1, 7),
        (7, 1, 3),
        (7, 5, 7),
        (7, 4, 4),
        (1, 1, 1),
    ] {
        println!(
            "  reverseBetween(1..{len}, {m}, {n}) = {}",
            show(&reverse_between(build_range(len), m, n))
        );
    }
    let (mut cases, mut mismatches) = (0, 0);
    const L: usize = 8;
    for m in 1..=L {
        for n in m..=L {
            let mut want = to_vec(&build_range(L as i64));
            want[m - 1..n].reverse(); // the slice oracle
            let got = to_vec(&reverse_between(build_range(L as i64), m, n));
            cases += 1;
            if got != want {
                mismatches += 1;
            }
        }
    }
    println!("  every 1 <= m <= n <= {L} on 1..{L}: {cases} cases, {mismatches} mismatches");

    println!();
    println!("== 5. freeing a 2,000,000-node list ==");
    // (a) The recursion you did not write: the compiler-generated drop of a Link drops
    // `next` inside the drop of its Node -- one stack frame per node. Measured on the PLAIN
    // Node type: the allocator records the stack address of each free (TRACK); node k is
    // freed k levels deep, so (highest - lowest) / (n - 1) is the bytes per level.
    const PROBE: usize = 1_000;
    let per_level = thread::Builder::new()
        .stack_size(1 << 20)
        .spawn(|| {
            let l = build_range(PROBE as i64);
            FREE_SP_MIN.store(usize::MAX, Relaxed);
            FREE_SP_MAX.store(0, Relaxed);
            TRACK.store(true, Relaxed);
            drop(l); // the default, recursive drop
            TRACK.store(false, Relaxed);
            (FREE_SP_MAX.load(Relaxed) - FREE_SP_MIN.load(Relaxed)) as f64 / (PROBE - 1) as f64
        })
        .unwrap()
        .join()
        .unwrap();
    let pred_1m = (1 << 20) as f64 / per_level;
    let pred_main = main_bytes / per_level;
    println!("  default drop of a {PROBE}-node Link: {per_level:.1} bytes of stack per node");
    println!(
        "  PREDICTED crash length: {pred_1m:.0} nodes on a 1 MiB thread, {pred_main:.0} on the main thread"
    );
    let safe = (pred_1m / 2.0) as i64;
    let freed = thread::Builder::new()
        .stack_size(1 << 20)
        .spawn(move || {
            let l = build_range(safe);
            let f0 = FREES.load(Relaxed);
            drop(l);
            FREES.load(Relaxed) - f0
        })
        .unwrap()
        .join()
        .unwrap();
    println!(
        "  default drop of {safe} nodes (half the prediction) on a 1 MiB thread: {freed} frees, fine"
    );
    println!(
        "  so a 2,000,000-node Link dropped the default way is {:.1}x past the main thread's limit;",
        2_000_000.0 / pred_main
    );
    println!(
        "  `./main --overflow` drops {:.0} nodes on the main thread to show it",
        2.0 * pred_main
    );

    // (b) The fix is the same loop as reverse: detach `next`, let the node go, advance.
    const BIG: i64 = 2_000_000;
    let (a0, f0, live0) = (
        ALLOCS.load(Relaxed),
        FREES.load(Relaxed),
        LIVE.load(Relaxed),
    );
    let list = List(build_range(BIG));
    let (a1, live1) = (ALLOCS.load(Relaxed), LIVE.load(Relaxed));
    println!(
        "  built: {} allocations, {} live bytes ({} per node)",
        a1 - a0,
        live1 - live0,
        (live1 - live0) / BIG as usize
    );
    drop(list); // impl Drop for List: iterative, on the main thread
    println!(
        "  drop(List) with the iterative Drop: {} frees, live bytes back to +{}",
        FREES.load(Relaxed) - f0,
        LIVE.load(Relaxed) - live0
    );
    println!(
        "  std's LinkedList does the same: its Drop is `while pop_front_node().is_some() {{}}`."
    );
}

/// The main thread's stack size in KiB, as the shell reports it (`ulimit -s`).
fn main_stack_kib() -> f64 {
    Command::new("sh")
        .args(["-c", "ulimit -s"])
        .output()
        .ok()
        .and_then(|o| String::from_utf8(o.stdout).ok())
        .and_then(|s| s.trim().parse().ok())
        .unwrap_or(8192.0)
}

/// Only with --overflow: the default recursive drop at 2x the predicted main-thread length.
/// This aborts the process ("has overflowed its stack") and macOS writes a crash report.
fn overflow() {
    let per_level = 1.0f64.max({
        let l = build_range(1_000);
        TRACK.store(true, Relaxed);
        drop(l);
        TRACK.store(false, Relaxed);
        (FREE_SP_MAX.load(Relaxed) - FREE_SP_MIN.load(Relaxed)) as f64 / 999.0
    });
    let main_kib = main_stack_kib();
    let n = (2.0 * main_kib * 1024.0 / per_level) as i64;
    println!("dropping a {n}-node Link the default (recursive) way on the main thread...");
    drop(build_range(n));
    println!("survived (the prediction was wrong)");
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O main.rs && ./main          # ./main --overflow aborts on purpose
```

**Rust sample output:**

```
== 1. iterative reverse of 1..5, traced ==
  step 1: next=2   cur=1 now points at nil -> prev=1 cur=2
  step 2: next=3   cur=2 now points at 1   -> prev=2 cur=3
  step 3: next=4   cur=3 now points at 2   -> prev=3 cur=4
  step 4: next=5   cur=4 now points at 3   -> prev=4 cur=5
  step 5: next=nil cur=5 now points at 4   -> prev=5 cur=nil
  result [5 -> 4 -> 3 -> 2 -> 1], 5 link writes for 5 nodes
  n = 10     link writes = 10
  n = 1000   link writes = 1000
  n = 100000 link writes = 100000

== 2. forget the SAVE ==
  result [1] (bounded walk of 12)
  nodes now reachable from the result: 1 of 5; the other 4 were FREED by the
  assignment `node.next = prev` (dealloc count +4). No debug handle can keep
  them: a second pointer to node 2 would be a second owner. No panic, no error.

== 3. recursive reverse: same answer, one stack frame per node ==
  reverse_rec(1..5) = [5 -> 4 -> 3 -> 2 -> 1]
  base-case address, depth 10000 vs 100000: 5760000 bytes apart -> 64.0 bytes per frame
  main thread stack (ulimit -s): 8176 KiB; spawned threads default to 2 MiB
  PREDICTED: reverse_rec dies near n = 130816 on the main thread, 32768 on a 2 MiB thread
  (not triggered: a Rust stack overflow aborts the process -- no unwinding, no catch)
  check: 10000-node result starts 10000, 100000-node result starts 100000

== 4. reverseBetween: the same three lines plus two reconnections ==
  reverseBetween(1..7, 2, 4) = [1 -> 4 -> 3 -> 2 -> 5 -> 6 -> 7]
  reverseBetween(1..7, 1, 7) = [7 -> 6 -> 5 -> 4 -> 3 -> 2 -> 1]
  reverseBetween(1..7, 1, 3) = [3 -> 2 -> 1 -> 4 -> 5 -> 6 -> 7]
  reverseBetween(1..7, 5, 7) = [1 -> 2 -> 3 -> 4 -> 7 -> 6 -> 5]
  reverseBetween(1..7, 4, 4) = [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7]
  reverseBetween(1..1, 1, 1) = [1]
  every 1 <= m <= n <= 8 on 1..8: 36 cases, 0 mismatches

== 5. freeing a 2,000,000-node list ==
  default drop of a 1000-node Link: 32.0 bytes of stack per node
  PREDICTED crash length: 32768 nodes on a 1 MiB thread, 261632 on the main thread
  default drop of 16384 nodes (half the prediction) on a 1 MiB thread: 16384 frees, fine
  so a 2,000,000-node Link dropped the default way is 7.6x past the main thread's limit;
  `./main --overflow` drops 523264 nodes on the main thread to show it
  built: 2000000 allocations, 32000000 live bytes (16 per node)
  drop(List) with the iterative Drop: 2000000 frees, live bytes back to +0
  std's LinkedList does the same: its Drop is `while pop_front_node().is_some() {}`.
```

> **Go vs Rust:** Sections 1 and 4 print the same lines as Go. The escape hatches: `mem::replace(&mut node.next, prev)` makes save+flip one expression; the recursive reverse returns the empty `&mut Link` slot at the end of what it built, because Go's `h.next.next = h` needs two pointers to one node; and `reverse_between` cuts the window off first (n−m extra hops) because it cannot keep Go's `first` pointer into a chain it owns. Forgetting the save frees the 4 nodes on the spot (dealloc count +4), where Go leaves them to the GC.
> The recursion Go does not have is the compiler-generated drop: the plain `Option<Box<Node>>` freed its nodes 32 bytes of stack apart (read from the allocator's free addresses), which predicts a crash near 261,632 nodes on the 8,176 KiB main thread. In a separate run, 0.9× that survived and `./main --overflow` (523,264 nodes) aborted with `thread 'main' has overflowed its stack`, so `List` gets an iterative `Drop`, the same loop std's `LinkedList` uses (read in the source). Recursive reverse frames are 64 B here vs Go's 32 B. With an ~8 MiB main stack instead of Go's 512 MiB ceiling, Rust's predicted death is about 131k nodes, not 16.8 million.

**Complexity:** iterative Θ(n) time, Θ(1) space, exactly n link writes · recursive Θ(n) time **and Θ(n) stack** — 32 bytes per frame here, so Go's 512 MiB effective stack ceiling (1 GB `maxstacksize`, power-of-two stacks) is predicted to die near 16.8 million nodes · `reverseBetween` Θ(n), one pass, 36/36 windows match the oracle · the GC frees a 2,000,000-node list without walking it recursively.

---

## 8. Middle and Nth from end, in one pass

`🟡 medium` · *two pointers: different speeds, or a fixed gap*

Two pointers answer both questions without knowing the length: slow +1 / fast +2 leaves slow at the middle when
fast runs out, and a lead pointer started n nodes ahead leaves the trail pointer n from the end. Both are Θ(n)
time and Θ(1) space. Two details decide whether the code is right: on an even length the loop condition picks
*which* middle you get (the classic off-by-one), and a sentinel makes "remove the Nth from the end" handle the
head like any other node. Then the program counts hops to check what "one pass" really saves.

**Steps:**

1. Run both middle loop conditions on lengths 1..8 and see where they disagree.
2. Find and remove the Nth from the end with a gap of n, checked against a slice oracle for every length and n.
3. Count hops on a 1,000,000-node list: one pass with two pointers vs count-then-walk.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
)

type node struct {
	val  int
	next *node
}

func buildRange(n int) *node { // 1..n
	var head *node
	for i := n; i >= 1; i-- {
		head = &node{i, head}
	}
	return head
}

func toSlice(head *node) []int {
	var out []int
	for n := head; n != nil; n = n.next {
		out = append(out, n.val)
	}
	return out
}

func show(head *node) string {
	parts := []string{}
	for n := head; n != nil; n = n.next {
		parts = append(parts, fmt.Sprint(n.val))
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

// hops counts every `x = x.next`: the honest unit of work on a list.
var hops int

// middleSecond: on an even length it returns the SECOND of the two middles.
// Loop condition: fast can take two more steps from where it stands.
func middleSecond(head *node) *node {
	slow, fast := head, head
	for fast != nil && fast.next != nil {
		slow = slow.next      // +1
		fast = fast.next.next // +2
		hops += 3
	}
	return slow
}

// middleFirst: on an even length it returns the FIRST middle. The loop stops one round
// earlier -- which is also what you want when you split a list in half (lesson 07 ex 14).
func middleFirst(head *node) *node {
	slow, fast := head, head
	for fast.next != nil && fast.next.next != nil {
		slow = slow.next
		fast = fast.next.next
	}
	return slow
}

// nthFromEnd: start lead n nodes ahead, then move both until lead falls off the end.
// The gap stays n, so trail is n from the end. (0, false) if n > length or n < 1.
func nthFromEnd(head *node, n int) (int, bool) {
	if n < 1 {
		return 0, false
	}
	lead := head
	for range n {
		if lead == nil {
			return 0, false
		}
		lead = lead.next
	}
	trail := head
	for lead != nil {
		lead, trail = lead.next, trail.next
	}
	return trail.val, true
}

// removeNthFromEnd: the gap trick from a sentinel, so trail stops at the node BEFORE
// the target -- and removing the head (n == length) is not special.
func removeNthFromEnd(head *node, n int) (*node, bool) {
	s := &node{next: head}
	lead, trail := s, s
	for range n {
		if lead.next == nil {
			return head, false
		}
		lead = lead.next
		hops++
	}
	for lead.next != nil {
		lead, trail = lead.next, trail.next
		hops += 2
	}
	trail.next = trail.next.next
	return s.next, true
}

// The two-pass versions, for counting: first count, then walk.
func length(head *node) int {
	n := 0
	for x := head; x != nil; x = x.next {
		n++
		hops++
	}
	return n
}

func walk(head *node, k int) *node {
	for range k {
		head = head.next
		hops++
	}
	return head
}

func main() {
	fmt.Println("== 1. which middle? the loop condition decides ==")
	fmt.Println("  len  list                                   second middle  first middle")
	oddAgree, evenApart := 0, 0
	for n := 1; n <= 8; n++ {
		l := buildRange(n)
		sec, fst := middleSecond(l).val, middleFirst(l).val
		fmt.Printf("  %-4d %-38s %13d %13d\n", n, show(l), sec, fst)
		if n%2 == 1 && sec == fst {
			oddAgree++
		}
		if n%2 == 0 && sec == fst+1 {
			evenApart++
		}
	}
	fmt.Printf("  odd lengths where they agree: %d of 4; even lengths one node apart: %d of 4\n",
		oddAgree, evenApart)

	fmt.Println()
	fmt.Println("== 2. Nth from the end of 1..10 (lead pointer n ahead) ==")
	ten := buildRange(10)
	for _, n := range []int{1, 3, 10, 11, 0} {
		if v, ok := nthFromEnd(ten, n); ok {
			fmt.Printf("  n = %-2d -> %d\n", n, v)
		} else {
			fmt.Printf("  n = %-2d -> out of range (%d, %v)\n", n, v, ok)
		}
	}

	fmt.Println()
	fmt.Println("== 3. removeNthFromEnd with a sentinel ==")
	for _, n := range []int{1, 5, 10} {
		r, _ := removeNthFromEnd(buildRange(10), n)
		fmt.Printf("  remove n = %-2d from 1..10: %s\n", n, show(r))
	}
	cases, mismatches := 0, 0
	for L := 1; L <= 10; L++ {
		for n := 1; n <= L; n++ {
			want := slices.Delete(toSlice(buildRange(L)), L-n, L-n+1) // the slice oracle
			got, ok := removeNthFromEnd(buildRange(L), n)
			cases++
			if !ok || !slices.Equal(toSlice(got), want) {
				mismatches++
			}
		}
	}
	fmt.Printf("  every length 1..10, every valid n: %d cases, %d mismatches\n", cases, mismatches)
	three := buildRange(3)
	three, ok := removeNthFromEnd(three, 4)
	fmt.Printf("  n = 4 on a 3-node list: ok = %v, list still %s\n", ok, show(three))

	fmt.Println()
	fmt.Println("== 4. 'one pass' is fewer passes, not fewer hops (n = 1,000,000) ==")
	const N = 1_000_000
	big := buildRange(N)

	hops = 0
	m := middleSecond(big)
	onePass := hops
	hops = 0
	m2 := walk(big, length(big)/2)
	twoPass := hops
	fmt.Printf("  middle, slow+fast in one pass: %8d hops (%.2fn) -> %d\n", onePass, float64(onePass)/N, m.val)
	fmt.Printf("  middle, count then walk n/2:   %8d hops (%.2fn) -> %d\n", twoPass, float64(twoPass)/N, m2.val)
	fmt.Printf("  ratio one-pass / two-pass: %.3f\n", float64(onePass)/float64(twoPass))

	for _, n := range []int{1, N / 2} {
		hops = 0
		L := length(big)            // two-pass: count...
		walk(&node{next: big}, L-n) // ...then walk from a sentinel to the predecessor
		twoPass := hops
		hops = 0
		big, _ = removeNthFromEnd(big, n) // one pass, gap of n
		fmt.Printf("  remove n = %-6d (L = %d): gap pointers %d hops, count-then-walk %d, 2L-n = %d\n",
			n, L, hops, twoPass, 2*L-n)
	}
	fmt.Println("  so 'one pass' saves no work: the trailing pointer re-reads exactly the nodes a")
	fmt.Println("  second pass would. It buys one loop instead of two, and -- for a small gap --")
	fmt.Println("  re-reads nodes the lead touched moments ago (cache-friendly; not measured here).")
}
```

**Output:**

```
== 1. which middle? the loop condition decides ==
  len  list                                   second middle  first middle
  1    [1]                                                1             1
  2    [1 -> 2]                                           2             1
  3    [1 -> 2 -> 3]                                      2             2
  4    [1 -> 2 -> 3 -> 4]                                 3             2
  5    [1 -> 2 -> 3 -> 4 -> 5]                            3             3
  6    [1 -> 2 -> 3 -> 4 -> 5 -> 6]                       4             3
  7    [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7]                  4             4
  8    [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8]             5             4
  odd lengths where they agree: 4 of 4; even lengths one node apart: 4 of 4

== 2. Nth from the end of 1..10 (lead pointer n ahead) ==
  n = 1  -> 10
  n = 3  -> 8
  n = 10 -> 1
  n = 11 -> out of range (0, false)
  n = 0  -> out of range (0, false)

== 3. removeNthFromEnd with a sentinel ==
  remove n = 1  from 1..10: [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9]
  remove n = 5  from 1..10: [1 -> 2 -> 3 -> 4 -> 5 -> 7 -> 8 -> 9 -> 10]
  remove n = 10 from 1..10: [2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10]
  every length 1..10, every valid n: 55 cases, 0 mismatches
  n = 4 on a 3-node list: ok = false, list still [1 -> 2 -> 3]

== 4. 'one pass' is fewer passes, not fewer hops (n = 1,000,000) ==
  middle, slow+fast in one pass:  1500000 hops (1.50n) -> 500001
  middle, count then walk n/2:    1500000 hops (1.50n) -> 500001
  ratio one-pass / two-pass: 1.000
  remove n = 1      (L = 1000000): gap pointers 1999999 hops, count-then-walk 1999999, 2L-n = 1999999
  remove n = 500000 (L = 999999): gap pointers 1499998 hops, count-then-walk 1499998, 2L-n = 1499998
  so 'one pass' saves no work: the trailing pointer re-reads exactly the nodes a
  second pass would. It buys one loop instead of two, and -- for a small gap --
  re-reads nodes the lead touched moments ago (cache-friendly; not measured here).
```

**Rust:**

```rust
struct Node {
    val: i64,
    next: Link,
}
type Link = Option<Box<Node>>;

fn build_range(n: i64) -> Link {
    // 1..n
    let mut head = None;
    for v in (1..=n).rev() {
        head = Some(Box::new(Node { val: v, next: head }));
    }
    head
}

/// Owns a long list and frees it with a loop: the default drop of a 1,000,000-node Link
/// recurses once per node and would overflow the main thread's stack (example 7).
struct List(Link);
impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.0.take();
        while let Some(mut n) = cur {
            cur = n.next.take();
        }
    }
}

fn to_vec(head: &Link) -> Vec<i64> {
    let mut out = Vec::new();
    let mut cur = head;
    while let Some(n) = cur {
        out.push(n.val);
        cur = &n.next;
    }
    out
}

fn show(head: &Link) -> String {
    let parts: Vec<String> = to_vec(head).iter().map(|v| v.to_string()).collect();
    format!("[{}]", parts.join(" -> "))
}

// Rust has no mutable globals without atomics or unsafe, so the hop counter is threaded
// through as `&mut usize`: every `x = x.next` adds one.

/// Second middle on an even length. Two SHARED cursors into one list: any number of
/// `&Node`s may coexist, so this is as easy as in Go.
fn middle_second<'a>(head: &'a Link, hops: &mut usize) -> Option<&'a Node> {
    let (mut slow, mut fast) = (head.as_deref(), head.as_deref());
    while let Some(f) = fast
        && let Some(f1) = f.next.as_deref()
    {
        slow = slow.and_then(|s| s.next.as_deref()); // +1
        fast = f1.next.as_deref(); // +2
        *hops += 3;
    }
    slow
}

/// First middle on an even length: the loop stops one round earlier. Takes `&Node`, not
/// `&Link`: Go's version dereferences `fast.next` on entry and would panic on nil; here
/// the empty list cannot even be passed in.
fn middle_first(head: &Node) -> &Node {
    let (mut slow, mut fast) = (head, head);
    while let Some(f1) = fast.next.as_deref()
        && let Some(f2) = f1.next.as_deref()
    {
        slow = slow.next.as_deref().unwrap(); // slow is behind fast, so it exists
        fast = f2;
    }
    slow
}

/// Nth from the end with a gap of n -- read-only, so two shared cursors are fine.
/// `Option<i64>` replaces Go's `(int, bool)`.
fn nth_from_end(head: &Link, n: usize, hops: &mut usize) -> Option<i64> {
    if n < 1 {
        return None;
    }
    let mut lead = head.as_deref();
    for _ in 0..n {
        lead = lead?.next.as_deref(); // `?`: fell off the end => n > length => None
        *hops += 1;
    }
    let mut trail = head.as_deref();
    while let Some(l) = lead {
        lead = l.next.as_deref();
        trail = trail?.next.as_deref();
        *hops += 2;
    }
    trail.map(|t| t.val)
}

// Removing is where Rust parts ways with Go. Go's one-pass version keeps a lead pointer
// and a trail pointer (the one that unlinks) in the same chain at the same time. In Rust
// the unlinking cursor is a `&mut Link`, and a shared `&Link` into the same list while it
// lives is refused:
//
//     let mut trail: &mut Link = head;
//     let mut lead: &Link = head;
//
//     error[E0502]: cannot borrow `*head` as immutable because it is also borrowed as mutable
//
// So Rust does it in two passes: a shared walk to count, then ONE `&mut Link` cursor
// walk to the link that reaches the target. No sentinel is needed either: `head` is
// itself a `&mut Link`, so the head is just another link (example 5).

/// Remove the n-th node from the end; false (list untouched) if n is out of range.
fn remove_nth_from_end(head: &mut Link, n: usize, hops: &mut usize) -> bool {
    let mut len = 0;
    let mut c = head.as_deref();
    while let Some(node) = c {
        len += 1;
        c = node.next.as_deref();
        *hops += 1;
    }
    if n < 1 || n > len {
        return false;
    }
    let mut cur = head; // the shared walk is over, so the &mut borrow may begin
    for _ in 0..len - n {
        cur = &mut cur.as_mut().unwrap().next;
        *hops += 1;
    }
    let target = cur.take().unwrap();
    *cur = target.next; // unlink: the link now skips the target, which is freed here
    true
}

fn main() {
    let mut no_count = 0;
    println!("== 1. which middle? the loop condition decides ==");
    println!("  len  list                                   second middle  first middle");
    let (mut odd_agree, mut even_apart) = (0, 0);
    for n in 1..=8 {
        let l = build_range(n);
        let sec = middle_second(&l, &mut no_count).unwrap().val;
        let fst = middle_first(l.as_deref().unwrap()).val;
        println!("  {n:<4} {:<38} {sec:>13} {fst:>13}", show(&l));
        if n % 2 == 1 && sec == fst {
            odd_agree += 1;
        }
        if n % 2 == 0 && sec == fst + 1 {
            even_apart += 1;
        }
    }
    println!(
        "  odd lengths where they agree: {odd_agree} of 4; even lengths one node apart: {even_apart} of 4"
    );

    println!();
    println!("== 2. Nth from the end of 1..10 (lead pointer n ahead) ==");
    let ten = build_range(10);
    for n in [1, 3, 10, 11, 0] {
        match nth_from_end(&ten, n, &mut no_count) {
            Some(v) => println!("  n = {n:<2} -> {v}"),
            None => println!("  n = {n:<2} -> out of range (None)"),
        }
    }

    println!();
    println!("== 3. removeNthFromEnd (count, then one &mut cursor) ==");
    for n in [1, 5, 10] {
        let mut l = build_range(10);
        remove_nth_from_end(&mut l, n, &mut no_count);
        println!("  remove n = {n:<2} from 1..10: {}", show(&l));
    }
    let (mut cases, mut mismatches) = (0, 0);
    for len in 1..=10 {
        for n in 1..=len {
            let mut want = to_vec(&build_range(len as i64));
            want.remove(len - n); // the slice oracle
            let mut got = build_range(len as i64);
            let ok = remove_nth_from_end(&mut got, n, &mut no_count);
            cases += 1;
            if !ok || to_vec(&got) != want {
                mismatches += 1;
            }
        }
    }
    println!("  every length 1..10, every valid n: {cases} cases, {mismatches} mismatches");
    let mut three = build_range(3);
    let ok = remove_nth_from_end(&mut three, 4, &mut no_count);
    println!(
        "  n = 4 on a 3-node list: ok = {ok}, list still {}",
        show(&three)
    );

    println!();
    println!("== 4. 'one pass' is fewer passes, not fewer hops (n = 1,000,000) ==");
    const N: usize = 1_000_000;
    let mut big = List(build_range(N as i64));

    let mut one_pass = 0;
    let m = middle_second(&big.0, &mut one_pass).unwrap().val;
    // two passes: count, then walk n/2
    let mut two_pass = 0;
    let mut c = big.0.as_deref();
    let mut len = 0;
    while let Some(node) = c {
        len += 1;
        c = node.next.as_deref();
        two_pass += 1;
    }
    let mut m2 = big.0.as_deref().unwrap();
    for _ in 0..len / 2 {
        m2 = m2.next.as_deref().unwrap();
        two_pass += 1;
    }
    println!(
        "  middle, slow+fast in one pass: {one_pass:>8} hops ({:.2}n) -> {m}",
        one_pass as f64 / N as f64
    );
    println!(
        "  middle, count then walk n/2:   {two_pass:>8} hops ({:.2}n) -> {}",
        two_pass as f64 / N as f64,
        m2.val
    );
    println!(
        "  ratio one-pass / two-pass: {:.3}",
        one_pass as f64 / two_pass as f64
    );

    for n in [1, N / 2] {
        let l = to_len(&big.0);
        let mut gap = 0;
        nth_from_end(&big.0, n, &mut gap); // the gap walk Go's removal does, read-only
        let mut two = 0;
        remove_nth_from_end(&mut big.0, n, &mut two);
        println!(
            "  remove n = {n:<6} (L = {l}): gap walk (read-only) {gap} hops, count-then-cursor {two}, 2L-n = {}",
            2 * l - n
        );
    }
    println!("  so 'one pass' saves no work: Rust's forced two-pass removal costs exactly the");
    println!("  hops of the gap walk. One loop instead of two is all the one-pass version buys");
    println!("  -- and that loop needs two live cursors, one mutating: E0502 in safe Rust.");
}

fn to_len(head: &Link) -> usize {
    let mut n = 0;
    let mut c = head.as_deref();
    while let Some(node) = c {
        n += 1;
        c = node.next.as_deref();
    }
    n
}
```

**Rust output:**

```
== 1. which middle? the loop condition decides ==
  len  list                                   second middle  first middle
  1    [1]                                                1             1
  2    [1 -> 2]                                           2             1
  3    [1 -> 2 -> 3]                                      2             2
  4    [1 -> 2 -> 3 -> 4]                                 3             2
  5    [1 -> 2 -> 3 -> 4 -> 5]                            3             3
  6    [1 -> 2 -> 3 -> 4 -> 5 -> 6]                       4             3
  7    [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7]                  4             4
  8    [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8]             5             4
  odd lengths where they agree: 4 of 4; even lengths one node apart: 4 of 4

== 2. Nth from the end of 1..10 (lead pointer n ahead) ==
  n = 1  -> 10
  n = 3  -> 8
  n = 10 -> 1
  n = 11 -> out of range (None)
  n = 0  -> out of range (None)

== 3. removeNthFromEnd (count, then one &mut cursor) ==
  remove n = 1  from 1..10: [1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9]
  remove n = 5  from 1..10: [1 -> 2 -> 3 -> 4 -> 5 -> 7 -> 8 -> 9 -> 10]
  remove n = 10 from 1..10: [2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10]
  every length 1..10, every valid n: 55 cases, 0 mismatches
  n = 4 on a 3-node list: ok = false, list still [1 -> 2 -> 3]

== 4. 'one pass' is fewer passes, not fewer hops (n = 1,000,000) ==
  middle, slow+fast in one pass:  1500000 hops (1.50n) -> 500001
  middle, count then walk n/2:    1500000 hops (1.50n) -> 500001
  ratio one-pass / two-pass: 1.000
  remove n = 1      (L = 1000000): gap walk (read-only) 1999999 hops, count-then-cursor 1999999, 2L-n = 1999999
  remove n = 500000 (L = 999999): gap walk (read-only) 1499998 hops, count-then-cursor 1499998, 2L-n = 1499998
  so 'one pass' saves no work: Rust's forced two-pass removal costs exactly the
  hops of the gap walk. One loop instead of two is all the one-pass version buys
  -- and that loop needs two live cursors, one mutating: E0502 in safe Rust.
```

> **Go vs Rust:** Every count matches Go's: the two middles, 55 cases with 0 mismatches, 1,500,000 hops for each middle, and 2L − n for each removal. Only the labels differ, because `Option<i64>` replaces `(int, bool)`. The two reading algorithms port directly, since any number of shared `&Node` cursors may coexist. Removal does not: a mutating `&mut Link` trail cursor alongside a shared lead cursor in the same chain is E0502 (``cannot borrow `*head` as immutable because it is also borrowed as mutable``, error text in a comment). So the Rust version counts first, then walks one cursor to the target's link. That costs exactly the hops of Go's one-pass gap version, and needs no sentinel because `head` is already a `&mut Link`.

**Complexity:** Θ(n) time, Θ(1) space · middle: 1,500,000 hops on 1,000,000 nodes whether one pass or two (ratio 1.000) · remove Nth from end: exactly 2L − n hops either way, 55/55 cases match the oracle · the even-length middle is chosen by the loop condition (`fast != nil && fast.next != nil` → second, `fast.next != nil && fast.next.next != nil` → first).

---

## 9. Floyd's cycle detection

`🟡 medium` · *Θ(1) space, and the algebra behind phase 2*

A list whose last node points back into itself has the shape of the letter rho: a tail of L nodes, then a cycle
of C. Floyd finds it with two pointers and Θ(1) extra space. In phase 1, slow moves +1 and fast +2; once both are
inside the cycle, fast closes the gap by one node per round, so they must meet. In phase 2, one pointer goes back
to the head and both move +1; they meet exactly at the cycle's start, because slow walked t = L + k, fast walked
2t = t + mC, and so L = mC − k. One more lap gives C. The whole thing is Θ(L + C) time. A `visited` map gets the
same answers in Θ(n) space. And Floyd never needed a list: only `x → next(x)`.

**Steps:**

1. Build rho-shaped lists for eight (L, C) pairs and print every quantity in the algebra for each.
2. Measure the visited map's memory on a 1,000,000-node list against Floyd's zero extra bytes.
3. Run the same generic `floyd` on an iterated function to find its tail μ and cycle λ.

```go
package main

import (
	"fmt"
	"runtime"
)

type node struct {
	val  int
	next *node
}

// buildRho builds a "rho" shape: a tail of L nodes (indices 0..L-1) feeding a cycle of C
// nodes (indices L..L+C-1); the last node points back to node L. C == 0: acyclic.
func buildRho(L, C int) *node {
	nodes := make([]*node, L+C)
	for i := range nodes {
		nodes[i] = &node{val: i}
	}
	for i := 0; i+1 < len(nodes); i++ {
		nodes[i].next = nodes[i+1]
	}
	if C > 0 {
		nodes[L+C-1].next = nodes[L]
	}
	return nodes[0]
}

// result of Floyd's algorithm. steps counts every call of next: one pointer hop each.
type result struct {
	cycle     bool
	meet      int // phase 1: number of rounds (slow +1, fast +2) until they meet
	mu        int // phase 2: steps until they meet again = index of the cycle's start
	lambda    int // one more lap from the meeting point = cycle length
	steps     int
	fastSteps int // acyclic case: hops fast made before it hit nil
}

// floyd needs only x -> next(x): a list (T = *node, ok = false at nil) or any function
// iterated on a finite set. Θ(1) extra space: three variables of type T.
func floyd[T comparable](x0 T, next func(T) (T, bool)) result {
	var r result
	step := func(x T) (T, bool) { r.steps++; return next(x) }

	// Phase 1: fast gains one node per round, so inside the cycle it must land on slow.
	slow, fast := x0, x0
	for {
		f1, ok := step(fast)
		if !ok {
			r.fastSteps = r.steps
			return r
		}
		if fast, ok = step(f1); !ok {
			r.fastSteps = r.steps
			return r
		}
		slow, _ = step(slow)
		r.meet++
		if slow == fast {
			break
		}
	}
	r.cycle = true

	// Phase 2: slow has walked t = L + k, fast 2t = t + mC, so L + k = mC, i.e. L = mC - k:
	// walking L from the head lands where walking L from the meeting point lands.
	p, q := x0, slow
	for p != q {
		p, _ = step(p)
		q, _ = step(q)
		r.mu++
	}

	// Cycle length: one more lap from the meeting point.
	r.lambda = 1
	for x, _ := step(q); x != q; x, _ = step(x) {
		r.lambda++
	}
	return r
}

func listNext(n *node) (*node, bool) { return n.next, n.next != nil }

// visited: the obvious Θ(n)-space method. The map remembers the step of each first visit.
func visited(head *node) (mu, lambda int, cycle bool, seen map[*node]int) {
	seen = map[*node]int{}
	for i, n := 0, head; n != nil; i, n = i+1, n.next {
		if first, ok := seen[n]; ok {
			return first, i - first, true, seen
		}
		seen[n] = i
	}
	return 0, 0, false, seen
}

func main() {
	fmt.Println("== 1. Floyd on rho-shaped lists: tail L, cycle C ==")
	fmt.Println("     L    C | meet t  k=t-L  m=t/C  L==mC-k | start (=L)  length (=C) | steps   map agrees")
	cases := [][2]int{{0, 1}, {1, 1}, {0, 5}, {3, 5}, {7, 3}, {10, 10}, {100, 37}, {1000, 1}}
	good, formula := 0, 0
	for _, c := range cases {
		L, C := c[0], c[1]
		head := buildRho(L, C)
		r := floyd(head, listNext)
		k, m := r.meet-L, r.meet/C
		vmu, vlam, _, _ := visited(head)
		algebra := L == m*C-k && r.meet%C == 0
		agree := vmu == r.mu && vlam == r.lambda
		if algebra && r.mu == L && r.lambda == C && agree {
			good++
		}
		if r.steps == 3*r.meet+2*L+C {
			formula++
		}
		fmt.Printf("  %4d %4d | %6d %6d %6d  %7s | %10d  %11d | %6d  %v\n",
			L, C, r.meet, k, m, mark(algebra), r.mu, r.lambda, r.steps, agree)
	}
	fmt.Printf("  %d of %d cases: L == mC - k, start == L, length == C, visited map agrees\n", good, len(cases))
	fmt.Printf("  steps == 3*meet + 2*L + C in %d of %d cases: phase 1 is 3 hops per round\n", formula, len(cases))
	fmt.Println("  (slow 1 + fast 2), phase 2 is two pointers for L rounds, then one lap of C.")

	r := floyd(buildRho(10, 0), listNext)
	fmt.Printf("  acyclic 10-node list: cycle = %v, fast reached nil after %d hops (slow + fast)\n",
		r.cycle, r.fastSteps)

	fmt.Println()
	fmt.Println("== 2. extra memory: visited map vs Floyd, 1,000,000-node acyclic list ==")
	const N = 1_000_000
	head := buildRho(N, 0)
	var ms runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&ms)
	before := ms.HeapAlloc
	_, _, cyc, seen := visited(head)
	runtime.GC() // count only what the live map holds, not the garbage of its growth
	runtime.ReadMemStats(&ms)
	mapBytes := ms.HeapAlloc - before
	fmt.Printf("  visited map: cycle = %v, %d entries, heap +%.1f MiB = %.1f bytes per node\n",
		cyc, len(seen), float64(mapBytes)/(1<<20), float64(mapBytes)/N)
	runtime.KeepAlive(seen)
	runtime.GC()
	runtime.ReadMemStats(&ms)
	before = ms.HeapAlloc
	r = floyd(head, listNext)
	runtime.ReadMemStats(&ms)
	fmt.Printf("  Floyd:       cycle = %v, %d hops, heap %+d bytes (three pointer variables)\n",
		r.cycle, r.steps, int64(ms.HeapAlloc)-int64(before))

	fmt.Println()
	fmt.Println("== 3. no list at all: f(x) = (x*x + 1) mod 1000003, x0 = 2 ==")
	const P = 1_000_003
	f := func(x int) (int, bool) { return (x*x + 1) % P, true }
	r = floyd(2, f)
	fmt.Printf("  tail length mu = %d, cycle length lambda = %d, %d evaluations of f\n", r.mu, r.lambda, r.steps)
	// check by brute force: the sequence repeats first at index mu+lambda, back to index mu
	seq := map[int]int{}
	x, i := 2, 0
	for ; ; i++ {
		if j, ok := seq[x]; ok {
			fmt.Printf("  brute force with a map: first repeat at index %d = index %d -> mu %d, lambda %d\n",
				i, j, j, i-j)
			break
		}
		seq[x] = i
		x, _ = f(x)
	}
	fmt.Println("  the same floyd() ran on *node and on int: it only ever asks for next(x).")
}

func mark(ok bool) string {
	if ok {
		return "yes"
	}
	return "NO"
}
```

**Sample output:**

```
== 1. Floyd on rho-shaped lists: tail L, cycle C ==
     L    C | meet t  k=t-L  m=t/C  L==mC-k | start (=L)  length (=C) | steps   map agrees
     0    1 |      1      1      1      yes |          0            1 |      4  true
     1    1 |      1      0      1      yes |          1            1 |      6  true
     0    5 |      5      5      1      yes |          0            5 |     20  true
     3    5 |      5      2      1      yes |          3            5 |     26  true
     7    3 |      9      2      3      yes |          7            3 |     44  true
    10   10 |     10      0      1      yes |         10           10 |     60  true
   100   37 |    111     11      3      yes |        100           37 |    570  true
  1000    1 |   1000      0   1000      yes |       1000            1 |   5001  true
  8 of 8 cases: L == mC - k, start == L, length == C, visited map agrees
  steps == 3*meet + 2*L + C in 8 of 8 cases: phase 1 is 3 hops per round
  (slow 1 + fast 2), phase 2 is two pointers for L rounds, then one lap of C.
  acyclic 10-node list: cycle = false, fast reached nil after 14 hops (slow + fast)

== 2. extra memory: visited map vs Floyd, 1,000,000-node acyclic list ==
  visited map: cycle = false, 1000000 entries, heap +36.0 MiB = 37.8 bytes per node
  Floyd:       cycle = false, 1499999 hops, heap +0 bytes (three pointer variables)

== 3. no list at all: f(x) = (x*x + 1) mod 1000003, x0 = 2 ==
  tail length mu = 1173, cycle length lambda = 116, 6290 evaluations of f
  brute force with a map: first repeat at index 1289 = index 1173 -> mu 1173, lambda 116
  the same floyd() ran on *node and on int: it only ever asks for next(x).
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// ---- counting allocator (from the kit): the deterministic stand-in for MemStats ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static LIVE: AtomicUsize = AtomicUsize::new(0);

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
}
#[global_allocator]
static GLOBAL: Counting = Counting;

// A cycle cannot be built from `Option<Box<Node>>`: the last node would have to own a
// node that (indirectly) owns it. The two safe ways out:
//   * Rc<RefCell<Node>>: shared ownership allows the back link -- but a cycle of strong
//     Rcs is never freed (a leak, example 6), identity needs Rc::ptr_eq, and every hop is
//     a borrow() plus a refcount increment/decrement.
//   * an index arena: nodes live in one Vec and `next` is an index. Indices are Copy and
//     compare with ==, a cycle is just a number pointing backwards, and dropping the Vec
//     frees everything at once. This is exactly a "functional graph" i -> next[i].
// The arena wins on every count here, so that is what this program uses.
const NIL: u32 = u32::MAX;

struct Arena {
    next: Vec<u32>, // node i's successor, or NIL
}

impl Arena {
    /// Rho shape: a tail of L nodes (0..L-1) feeding a cycle of C nodes (L..L+C-1); the
    /// last node points back to node L. C == 0: acyclic.
    fn rho(l: usize, c: usize) -> Arena {
        let n = l + c;
        let mut next: Vec<u32> = (1..=n as u32).collect();
        next[n - 1] = if c > 0 { l as u32 } else { NIL };
        Arena { next }
    }
    fn step(&self, i: u32) -> Option<u32> {
        let j = self.next[i as usize];
        (j != NIL).then_some(j)
    }
}

/// Floyd's result; `steps` counts every call of next -- one pointer hop each.
#[derive(Default)]
struct Floyd {
    cycle: bool,
    meet: usize,   // phase 1 rounds until slow == fast
    mu: usize,     // phase 2 steps = index of the cycle's start
    lambda: usize, // one more lap = cycle length
    steps: usize,
    fast_steps: usize, // acyclic case: hops made before fast hit the end
}

/// Generic over anything Copy + ==: an arena index, or the value of an iterated function.
/// `next` returning None is the end of the list (Go's `ok == false`).
fn floyd<T: Copy + PartialEq>(x0: T, next: impl Fn(T) -> Option<T>) -> Floyd {
    let mut steps = 0;
    let mut step = |x: T| {
        steps += 1;
        next(x)
    };

    // Phase 1: fast gains one node per round, so inside the cycle it must land on slow.
    let (mut slow, mut fast) = (x0, x0);
    let mut meet = 0;
    let met = loop {
        let Some(f1) = step(fast) else { break false };
        let Some(f2) = step(f1) else { break false };
        fast = f2;
        slow = step(slow).expect("slow trails fast, so it has a next");
        meet += 1;
        if slow == fast {
            break true;
        }
    };
    if !met {
        return Floyd {
            meet,
            steps,
            fast_steps: steps,
            ..Floyd::default()
        };
    }

    // Phase 2: slow has walked t = L + k, fast 2t = t + mC, so L + k = mC, i.e. L = mC - k:
    // walking L from the head lands where walking L from the meeting point lands.
    let (mut p, mut q) = (x0, slow);
    let mut mu = 0;
    while p != q {
        p = step(p).unwrap(); // inside a cycle: next always exists
        q = step(q).unwrap();
        mu += 1;
    }

    // Cycle length: one more lap from the meeting point.
    let mut lambda = 1;
    let mut x = step(q).unwrap();
    while x != q {
        x = step(x).unwrap();
        lambda += 1;
    }
    // `step` borrowed `steps` mutably; that borrow ended at its last call above
    Floyd {
        cycle: true,
        meet,
        mu,
        lambda,
        steps,
        fast_steps: 0,
    }
}

/// visited: the obvious Θ(n)-space method -- remember the step of each first visit.
fn visited(a: &Arena, head: u32) -> (Option<(usize, usize)>, HashMap<u32, usize>) {
    let mut seen = HashMap::new();
    let (mut i, mut n) = (0, Some(head));
    while let Some(id) = n {
        if let Some(&first) = seen.get(&id) {
            return (Some((first, i - first)), seen);
        }
        seen.insert(id, i);
        n = a.step(id);
        i += 1;
    }
    (None, seen)
}

fn main() {
    println!("== 1. Floyd on rho-shaped lists: tail L, cycle C ==");
    println!(
        "     L    C | meet t  k=t-L  m=t/C  L==mC-k | start (=L)  length (=C) | steps   map agrees"
    );
    let cases = [
        (0, 1),
        (1, 1),
        (0, 5),
        (3, 5),
        (7, 3),
        (10, 10),
        (100, 37),
        (1000, 1),
    ];
    let (mut good, mut formula) = (0, 0);
    for &(l, c) in &cases {
        let a = Arena::rho(l, c);
        let r = floyd(0u32, |i| a.step(i));
        // k = t - L can be negative only if the algebra failed; keep it signed
        let (k, m) = (r.meet as i64 - l as i64, r.meet / c);
        let (v, _) = visited(&a, 0);
        let algebra = l as i64 == (m * c) as i64 - k && r.meet % c == 0;
        let agree = v == Some((r.mu, r.lambda));
        if algebra && r.mu == l && r.lambda == c && agree {
            good += 1;
        }
        if r.steps == 3 * r.meet + 2 * l + c {
            formula += 1;
        }
        println!(
            "  {l:>4} {c:>4} | {:>6} {k:>6} {m:>6}  {:>7} | {:>10}  {:>11} | {:>6}  {agree}",
            r.meet,
            if algebra { "yes" } else { "NO" },
            r.mu,
            r.lambda,
            r.steps
        );
    }
    println!(
        "  {good} of {} cases: L == mC - k, start == L, length == C, visited map agrees",
        cases.len()
    );
    println!(
        "  steps == 3*meet + 2*L + C in {formula} of {} cases: phase 1 is 3 hops per round",
        cases.len()
    );
    println!("  (slow 1 + fast 2), phase 2 is two pointers for L rounds, then one lap of C.");
    let a = Arena::rho(10, 0);
    let r = floyd(0u32, |i| a.step(i));
    println!(
        "  acyclic 10-node list: cycle = {}, fast reached nil after {} hops (slow + fast)",
        r.cycle, r.fast_steps
    );

    println!();
    println!("== 2. extra memory: visited map vs Floyd, 1,000,000-node acyclic list ==");
    const N: usize = 1_000_000;
    let a = Arena::rho(N, 0);
    let live0 = LIVE.load(Relaxed);
    let (v, seen) = visited(&a, 0);
    let map_bytes = LIVE.load(Relaxed) - live0; // what the live map holds (old tables freed)
    println!(
        "  visited map: cycle = {}, {} entries, heap +{:.1} MiB = {:.1} bytes per node",
        v.is_some(),
        seen.len(),
        map_bytes as f64 / (1 << 20) as f64,
        map_bytes as f64 / N as f64
    );
    println!(
        "  (HashMap<u32, usize>, (u32, usize) entries of {} bytes; capacity() {} for {} entries)",
        size_of::<(u32, usize)>(),
        seen.capacity(),
        seen.len()
    );
    drop(seen);
    let (a0, live0) = (ALLOCS.load(Relaxed), LIVE.load(Relaxed));
    let r = floyd(0u32, |i| a.step(i));
    println!(
        "  Floyd:       cycle = {}, {} hops, heap {:+} bytes, {} allocations (three u32 variables)",
        r.cycle,
        r.steps,
        LIVE.load(Relaxed) as i64 - live0 as i64,
        ALLOCS.load(Relaxed) - a0
    );
    println!(
        "  the arena itself: {} bytes for {N} links (4 per node; a Box list would be 16 + one allocation per node)",
        a.next.capacity() * size_of::<u32>()
    );

    println!();
    println!("== 3. no list at all: f(x) = (x*x + 1) mod 1000003, x0 = 2 ==");
    const P: u64 = 1_000_003;
    let f = |x: u64| Some((x * x + 1) % P); // x < P, so x*x < 2^40: no overflow
    let r = floyd(2u64, f);
    println!(
        "  tail length mu = {}, cycle length lambda = {}, {} evaluations of f",
        r.mu, r.lambda, r.steps
    );
    // check by brute force: the sequence repeats first at index mu+lambda, back to index mu
    let mut seq: HashMap<u64, usize> = HashMap::new();
    let mut x = 2;
    for i in 0.. {
        if let Some(&j) = seq.get(&x) {
            println!(
                "  brute force with a map: first repeat at index {i} = index {j} -> mu {j}, lambda {}",
                i - j
            );
            break;
        }
        seq.insert(x, i);
        x = f(x).unwrap();
    }
    println!("  the same floyd() ran on arena indices and on u64: it only ever asks for next(x).");
}
```

**Rust output:**

```
== 1. Floyd on rho-shaped lists: tail L, cycle C ==
     L    C | meet t  k=t-L  m=t/C  L==mC-k | start (=L)  length (=C) | steps   map agrees
     0    1 |      1      1      1      yes |          0            1 |      4  true
     1    1 |      1      0      1      yes |          1            1 |      6  true
     0    5 |      5      5      1      yes |          0            5 |     20  true
     3    5 |      5      2      1      yes |          3            5 |     26  true
     7    3 |      9      2      3      yes |          7            3 |     44  true
    10   10 |     10      0      1      yes |         10           10 |     60  true
   100   37 |    111     11      3      yes |        100           37 |    570  true
  1000    1 |   1000      0   1000      yes |       1000            1 |   5001  true
  8 of 8 cases: L == mC - k, start == L, length == C, visited map agrees
  steps == 3*meet + 2*L + C in 8 of 8 cases: phase 1 is 3 hops per round
  (slow 1 + fast 2), phase 2 is two pointers for L rounds, then one lap of C.
  acyclic 10-node list: cycle = false, fast reached nil after 14 hops (slow + fast)

== 2. extra memory: visited map vs Floyd, 1,000,000-node acyclic list ==
  visited map: cycle = false, 1000000 entries, heap +34.0 MiB = 35.7 bytes per node
  (HashMap<u32, usize>, (u32, usize) entries of 16 bytes; capacity() 1835008 for 1000000 entries)
  Floyd:       cycle = false, 1499999 hops, heap +0 bytes, 0 allocations (three u32 variables)
  the arena itself: 4000000 bytes for 1000000 links (4 per node; a Box list would be 16 + one allocation per node)

== 3. no list at all: f(x) = (x*x + 1) mod 1000003, x0 = 2 ==
  tail length mu = 1173, cycle length lambda = 116, 6290 evaluations of f
  brute force with a map: first repeat at index 1289 = index 1173 -> mu 1173, lambda 116
  the same floyd() ran on arena indices and on u64: it only ever asks for next(x).
```

> **Go vs Rust:** Every step count, μ and λ is identical to Go's. A `Box` cannot form a cycle, because the last node would own a node that owns it, so the twin uses an index arena, `next: Vec<u32>` with `u32::MAX` as nil. That arena is exactly the functional graph i → next[i], and the same generic `floyd<T: Copy + PartialEq>` runs on arena indices and on `u64`. It was chosen over `Rc<RefCell<Node>>` because a ring of strong `Rc`s is never freed (example 6), identity would need `Rc::ptr_eq`, and every hop would be a `borrow()` plus a refcount update; arena indices are `Copy` and compare with `==`.
> The visited map's cost comes from the counting allocator, so it is deterministic: 35.7 bytes per node for `HashMap<u32, usize>`, against about 38 for Go's `map[*node]int`. Floyd costs 0 allocations and +0 bytes in both languages.

**Complexity:** Θ(L + C) time, Θ(1) space · phase 2 takes exactly L steps, and the total is exactly 3·meet + 2L + C hops in all 8 cases · the visited map is Θ(n) time and Θ(n) space, about 38 bytes per node in Go (vs +0 bytes for Floyd on the same 1,000,000-node list) · on f(x) = (x² + 1) mod 1000003 from 2: μ = 1173, λ = 116.

---

## 10. Merging with zero allocations

`🟡 medium` · *the one thing lists do better on their own terms*

Merging two sorted lists relinks nodes that already exist, so it needs Θ(1) extra space and makes **zero
allocations**, where the slice merge must allocate the whole output. That is why merge sort on a list needs no
auxiliary array — and why a list merge sort is the textbook case where a list is the natural input. The price is
that merge **consumes** its inputs: run it twice on the same pair and the second run builds a cycle. Merge is
Θ(m + n); merging k lists pairwise is Θ(n log k) against Θ(nk) for folding them into one accumulator.

**Steps:**

1. Write `merge(a, b)` with a stack-allocated sentinel and a `tail` pointer, and count comparisons and heap
   allocations (`runtime.MemStats.Mallocs` deltas around the merge only).
2. Reuse it three ways: sequential vs pairwise k-way merge, and merge sort split by slow/fast pointers.
3. Observe 0 allocations vs 1, comparisons under n·log2 n — and what a second `merge(a, b)` on the same heads returns.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"runtime"
	"runtime/debug"
	"slices"
	"strings"
	"time"
)

// Merging two sorted lists is the one job a linked list does better than a
// slice on its own terms: the output is made of nodes that ALREADY EXIST, so
// the merge only rewires next pointers. No allocation, Theta(1) extra space.
// The price: merge consumes its inputs. Afterwards a and b are not lists any
// more, just pointers into the middle of the merged one.

type node struct {
	val  int
	next *node
}

func fromSlice(xs []int) *node {
	var dummy node
	tail := &dummy
	for _, x := range xs {
		tail.next = &node{val: x}
		tail = tail.next
	}
	return dummy.next
}

func toSlice(h *node) []int {
	var out []int
	for ; h != nil; h = h.next {
		out = append(out, h.val)
	}
	return out
}

// show prints at most limit nodes, so it terminates even on a cycle.
func show(h *node, limit int) string {
	var parts []string
	for ; h != nil && len(parts) < limit; h = h.next {
		parts = append(parts, fmt.Sprint(h.val))
	}
	s := strings.Join(parts, " -> ")
	if h != nil {
		s += " -> ..."
	}
	return "[" + s + "]"
}

// merge relinks the nodes of two sorted lists into one sorted list.
// dummy is a sentinel that lives in this stack frame (it does not escape), so
// "append to the output" never needs an empty-output special case. Ties take
// from a, which makes the merge stable. cmps counts element comparisons.
func merge(a, b *node, cmps *int) *node {
	var dummy node
	tail := &dummy
	for a != nil && b != nil {
		*cmps++
		if a.val <= b.val {
			tail.next, a = a, a.next
		} else {
			tail.next, b = b, b.next
		}
		tail = tail.next
	}
	// One side is exhausted: the rest of the other is already sorted and
	// already linked. One pointer write attaches all of it.
	if a != nil {
		tail.next = a
	} else {
		tail.next = b
	}
	return dummy.next
}

// mergeSlices is the array version: it MUST allocate the output, because
// there is nowhere else to put the elements.
func mergeSlices(a, b []int, cmps *int) []int {
	out := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		*cmps++
		if a[i] <= b[j] {
			out = append(out, a[i])
			i++
		} else {
			out = append(out, b[j])
			j++
		}
	}
	out = append(out, a[i:]...)
	return append(out, b[j:]...)
}

// mergeSort on a list: split with slow/fast (slow stops at the end of the
// first half), sort both halves, merge. No auxiliary array -- the merge needs
// none. Recursion depth is log2(n), so the stack is Theta(log n).
func mergeSort(h *node, cmps *int) *node {
	if h == nil || h.next == nil {
		return h
	}
	slow, fast := h, h.next
	for fast != nil && fast.next != nil {
		slow, fast = slow.next, fast.next.next
	}
	second := slow.next
	slow.next = nil // cut: h is now the first half alone
	return merge(mergeSort(h, cmps), mergeSort(second, cmps), cmps)
}

// hasCycle is Floyd's test (example 9).
func hasCycle(h *node) bool {
	slow, fast := h, h
	for fast != nil && fast.next != nil {
		slow, fast = slow.next, fast.next.next
		if slow == fast {
			return true
		}
	}
	return false
}

// mallocs reads the runtime's count of heap objects ever allocated.
// ReadMemStats itself does not allocate, so a delta around a call is the
// number of objects that call allocated.
func mallocs() uint64 {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.Mallocs
}

func randomInts(r *rand.Rand, n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = r.IntN(1_000_000)
	}
	return xs
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	fmt.Println("1. merge two sorted lists by relinking")
	a := fromSlice([]int{1, 3, 5, 7})
	b := fromSlice([]int{2, 3, 6, 8, 9, 10})
	fmt.Printf("  a = %s\n  b = %s\n", show(a, 20), show(b, 20))
	cmps := 0
	m := merge(a, b, &cmps)
	fmt.Printf("  merge(a, b) = %s\n", show(m, 20))
	fmt.Printf("  %d comparisons for %d nodes: the loop stops when a runs out,\n", cmps, 10)
	fmt.Println("  and the rest of b (8 -> 9 -> 10) is attached with one pointer write")

	fmt.Println()
	fmt.Println("2. allocations during the merge only (inputs built before measuring)")
	// Sections 2-4 count heap objects. A GC cycle that happens to run inside
	// a counted window allocates a few dozen objects of its own, so the
	// collector is off until the counting is done (a few MB of garbage).
	gcPercent := debug.SetGCPercent(-1)
	const half = 50_000
	evens, odds := make([]int, half), make([]int, half)
	for i := range half {
		evens[i], odds[i] = 2*i, 2*i+1
	}
	la, lb := fromSlice(evens), fromSlice(odds)
	c1, c2 := 0, 0
	before := mallocs()
	lm := merge(la, lb, &c1)
	listAllocs := mallocs() - before
	before = mallocs()
	sm := mergeSlices(evens, odds, &c2)
	sliceAllocs := mallocs() - before
	fmt.Printf("  %-14s %12s %12s %10s\n", "2 x 50,000", "comparisons", "allocations", "same out")
	fmt.Printf("  %-14s %12d %12d %10v\n", "list merge", c1, listAllocs, true)
	fmt.Printf("  %-14s %12d %12d %10v\n", "slice merge", c2, sliceAllocs, slices.Equal(toSlice(lm), sm))
	fmt.Println("  the slice merge's one allocation is the 100,000-element output;")
	fmt.Println("  the list merge's output is the 100,000 nodes it was given")

	fmt.Println()
	fmt.Println("3. k-way merge: k = 8 sorted lists of 1,000 (n = 8,000)")
	const k, each = 8, 1_000
	runs := make([][]int, k)
	var all []int
	for i := range runs {
		runs[i] = randomInts(r, each)
		slices.Sort(runs[i])
		all = append(all, runs[i]...)
	}
	slices.Sort(all)
	n := k * each

	// Sequential: fold every list into one growing accumulator. The
	// accumulator is re-walked by every merge, so cost ~ n*k/2.
	seq := 0
	var acc *node
	for _, run := range runs {
		acc = merge(acc, fromSlice(run), &seq)
	}
	seqOK := slices.Equal(toSlice(acc), all)

	// Pairwise (tournament): merge neighbours, halving the number of lists
	// each round. log2(k) rounds, each touching all n elements.
	pair := 0
	lists := make([]*node, k)
	for i, run := range runs { // fresh lists: the first merge consumed its inputs
		lists[i] = fromSlice(run)
	}
	for len(lists) > 1 {
		var next []*node
		for i := 0; i+1 < len(lists); i += 2 {
			next = append(next, merge(lists[i], lists[i+1], &pair))
		}
		if len(lists)%2 == 1 {
			next = append(next, lists[len(lists)-1])
		}
		lists = next
	}
	pairOK := slices.Equal(toSlice(lists[0]), all)
	fmt.Printf("  %-12s %12s %18s %8s\n", "strategy", "comparisons", "reference", "sorted")
	fmt.Printf("  %-12s %12d %18s %8v\n", "sequential", seq, fmt.Sprintf("n*k/2 = %d", n*k/2), seqOK)
	fmt.Printf("  %-12s %12d %18s %8v\n", "pairwise", pair,
		fmt.Sprintf("n*log2 k = %d", n*int(math.Log2(k))), pairOK)
	fmt.Printf("  pairwise does %.2fx fewer comparisons at k = %d; the gap grows as k/(2 log2 k)\n",
		float64(seq)/float64(pair), k)

	fmt.Println()
	fmt.Println("4. merge sort on a list: split with slow/fast, merge, no extra array")
	fmt.Printf("  %9s %12s %12s %8s %12s %8s\n", "n", "comparisons", "n*log2 n", "ratio", "allocations", "correct")
	for _, size := range []int{1_000, 100_000} {
		input := randomInts(r, size)
		want := slices.Clone(input)
		slices.Sort(want)
		h := fromSlice(input)
		c := 0
		before := mallocs()
		h = mergeSort(h, &c)
		allocs := mallocs() - before
		ideal := float64(size) * math.Log2(float64(size))
		fmt.Printf("  %9d %12d %12.0f %8.3f %12d %8v\n",
			size, c, ideal, float64(c)/ideal, allocs, slices.Equal(toSlice(h), want))
	}
	fmt.Println("  comparisons stay under n*log2 n, and the sort allocates nothing:")
	fmt.Println("  every merge relinks nodes that already exist")
	debug.SetGCPercent(gcPercent)

	fmt.Println()
	fmt.Println("5. the pitfall: merge consumes its inputs")
	a = fromSlice([]int{1, 3, 5})
	b = fromSlice([]int{6, 7})
	cmps = 0
	m = merge(a, b, &cmps)
	fmt.Printf("  first merge(a, b)  = %s\n", show(m, 12))
	fmt.Printf("  a still points at node %d, b at node %d -- both now INSIDE the merged list\n", a.val, b.val)
	// The same call again. It returns -- but look at what it returns.
	m2 := merge(a, b, &cmps)
	fmt.Printf("  second merge(a, b) = %s\n", show(m2, 12))
	fmt.Printf("  cycle: %v -- the last node now points at itself\n", hasCycle(m2))
	// Why it returns at all: once a and b reach the same node, the tuple
	// assignment `tail.next, b = b, b.next` reads b.next BEFORE it writes
	// tail.next, so b keeps advancing. Written as two statements
	// (tail.next = b; b = b.next) the second merge itself never returns
	// (checked with a step cap). Either way: any walk of m2 never ends.
	fmt.Println("  toSlice(m2), a length count, or a third merge would never return.")
	fmt.Println("  The rule: after merge(a, b), only the returned head is a list;")
	fmt.Println("  a and b are dead names.")

	fmt.Println()
	fmt.Println("6. timing: list merge sort vs slices.Sort, same 100,000 PCG values (best of 5)")
	input := randomInts(r, 100_000)
	best := func(prep func(), run func()) time.Duration {
		bestD := time.Duration(math.MaxInt64)
		for range 5 {
			prep() // untimed: rebuild the input, because sorting mutates it
			t := time.Now()
			run()
			bestD = min(bestD, time.Since(t))
		}
		return bestD
	}
	var h *node
	var xs []int
	c := 0
	tList := best(func() { h = fromSlice(input) }, func() { h = mergeSort(h, &c) })
	tSlice := best(func() { xs = slices.Clone(input) }, func() { slices.Sort(xs) })
	fmt.Printf("  %-22s %10.2f ms\n", "list merge sort", float64(tList.Microseconds())/1000)
	fmt.Printf("  %-22s %10.2f ms\n", "slices.Sort (pdqsort)", float64(tSlice.Microseconds())/1000)
	if tList > tSlice {
		fmt.Printf("  the slice sort is %.1fx faster: zero allocations does not beat contiguous memory\n",
			float64(tList)/float64(tSlice))
	} else {
		fmt.Printf("  the list sort is %.1fx faster on this run\n", float64(tSlice)/float64(tList))
	}
	fmt.Printf("  (results agree: %v)\n", slices.Equal(toSlice(h), xs))
}
```

**Sample output:**

```
1. merge two sorted lists by relinking
  a = [1 -> 3 -> 5 -> 7]
  b = [2 -> 3 -> 6 -> 8 -> 9 -> 10]
  merge(a, b) = [1 -> 2 -> 3 -> 3 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10]
  7 comparisons for 10 nodes: the loop stops when a runs out,
  and the rest of b (8 -> 9 -> 10) is attached with one pointer write

2. allocations during the merge only (inputs built before measuring)
  2 x 50,000      comparisons  allocations   same out
  list merge            99999            0       true
  slice merge           99999            1       true
  the slice merge's one allocation is the 100,000-element output;
  the list merge's output is the 100,000 nodes it was given

3. k-way merge: k = 8 sorted lists of 1,000 (n = 8,000)
  strategy      comparisons          reference   sorted
  sequential          34970      n*k/2 = 32000     true
  pairwise            23984   n*log2 k = 24000     true
  pairwise does 1.46x fewer comparisons at k = 8; the gap grows as k/(2 log2 k)

4. merge sort on a list: split with slow/fast, merge, no extra array
          n  comparisons     n*log2 n    ratio  allocations  correct
       1000         8708         9966    0.874            0     true
     100000      1535951      1660964    0.925            0     true
  comparisons stay under n*log2 n, and the sort allocates nothing:
  every merge relinks nodes that already exist

5. the pitfall: merge consumes its inputs
  first merge(a, b)  = [1 -> 3 -> 5 -> 6 -> 7]
  a still points at node 1, b at node 6 -- both now INSIDE the merged list
  second merge(a, b) = [1 -> 3 -> 5 -> 6 -> 7 -> 7 -> 7 -> 7 -> 7 -> 7 -> 7 -> 7 -> ...]
  cycle: true -- the last node now points at itself
  toSlice(m2), a length count, or a third merge would never return.
  The rule: after merge(a, b), only the returned head is a list;
  a and b are dead names.

6. timing: list merge sort vs slices.Sort, same 100,000 PCG values (best of 5)
  list merge sort              7.05 ms
  slices.Sort (pdqsort)        4.82 ms
  the slice sort is 1.5x faster: zero allocations does not beat contiguous memory
  (results agree: true)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Merging two sorted lists is the one job a linked list does better than a
// Vec on its own terms: the output is made of nodes that ALREADY EXIST, so the
// merge only moves Boxes from one Option to another. No allocation, Theta(1)
// extra space. In Go the price is that merge consumes its inputs and the old
// names still point into the merged list. In Rust merge takes both lists BY
// VALUE: the old names are moved-from, and using them again does not compile.

// ---- allocation counter: stand-in for runtime.MemStats.Mallocs ----
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
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events while running f, f's result)
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

// ---- Go's math/rand/v2 PCG-DXSM + IntN, so the inputs match Go's exactly ----
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

    /// like rand.IntN(n): Lemire's method with rejection, as Go does it.
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

// ---- the list ----
struct Node {
    val: i64,
    next: Link,
}
type Link = Option<Box<Node>>;

// The compiler-generated drop of a Link is recursive: dropping a node drops
// its `next`, one stack frame per node (example 7). 100,000 nodes is enough to
// make that dangerous on an 8 MiB stack, so every node unhooks its tail and
// frees it in a loop. Each node freed here has next == None, so its own drop
// does nothing further: the whole chain goes in one flat loop.
impl Drop for Node {
    fn drop(&mut self) {
        let mut cur = self.next.take();
        while let Some(mut n) = cur {
            cur = n.next.take();
        }
    }
}

fn from_slice(xs: &[i64]) -> Link {
    let mut head: Link = None;
    let mut tail = &mut head;
    for &x in xs {
        // insert() stores the Box and hands back &mut to it: the new tail.
        tail = &mut tail.insert(Box::new(Node { val: x, next: None })).next;
    }
    head
}

fn to_vec(mut h: &Link) -> Vec<i64> {
    let mut out = Vec::new();
    while let Some(n) = h {
        out.push(n.val);
        h = &n.next;
    }
    out
}

/// Prints at most `limit` nodes (Go's show; a Box list cannot have a cycle,
/// but the shape of the helper stays the same).
fn show(mut h: &Link, limit: usize) -> String {
    let mut parts = Vec::new();
    while let Some(n) = h {
        if parts.len() == limit {
            break;
        }
        parts.push(n.val.to_string());
        h = &n.next;
    }
    let mut s = parts.join(" -> ");
    if h.is_some() {
        s += " -> ...";
    }
    format!("[{s}]")
}

/// merge relinks the nodes of two sorted lists into one sorted list.
/// No sentinel node: `tail: &mut Link` points at the slot where the next node
/// goes -- first `head` itself, then the `next` field of the last node placed.
/// That pointer-to-a-link is the sentinel's job done without a fake node.
/// Ties take from a (stable). Both lists are taken BY VALUE: merge owns them.
fn merge(mut a: Link, mut b: Link, cmps: &mut u64) -> Link {
    let mut head: Link = None;
    let mut tail = &mut head;
    while let (Some(x), Some(y)) = (&a, &b) {
        *cmps += 1;
        let src = if x.val <= y.val { &mut a } else { &mut b };
        let mut n = src.take().unwrap(); // detach the front node...
        *src = n.next.take(); // ...its successor becomes that list's front
        tail = &mut tail.insert(n).next; // a move of a Box: no allocation
    }
    // One side is exhausted: attach the rest of the other in one move.
    *tail = if a.is_some() { a } else { b };
    head
}

/// The Vec version MUST allocate the output: there is nowhere else to put it.
fn merge_vecs(a: &[i64], b: &[i64], cmps: &mut u64) -> Vec<i64> {
    let mut out = Vec::with_capacity(a.len() + b.len());
    let (mut i, mut j) = (0, 0);
    while i < a.len() && j < b.len() {
        *cmps += 1;
        if a[i] <= b[j] {
            out.push(a[i]);
            i += 1;
        } else {
            out.push(b[j]);
            j += 1;
        }
    }
    out.extend_from_slice(&a[i..]);
    out.extend_from_slice(&b[j..]);
    out
}

/// Cut the list after its first k nodes and return the rest.
fn split_off(head: &mut Link, k: usize) -> Link {
    let mut cur = head;
    for _ in 0..k {
        cur = &mut cur.as_mut().unwrap().next;
    }
    cur.take()
}

/// merge_sort on a list. Go splits with slow/fast pointers; in Rust that is a
/// `&mut` slow cursor and a `&` fast cursor in ONE chain, which rustc 1.98.1
/// rejects:
///   error[E0502]: cannot borrow `slow.next` as mutable because it is also
///   borrowed as immutable
/// So the length travels down the recursion instead, and the cut is at
/// ceil(n/2) -- exactly where Go's slow pointer stops, so the halves, the
/// merges and the comparison counts are the same.
fn merge_sort(mut h: Link, n: usize, cmps: &mut u64) -> Link {
    if n <= 1 {
        return h;
    }
    let first = n.div_ceil(2);
    let second = split_off(&mut h, first);
    let a = merge_sort(h, first, cmps);
    let b = merge_sort(second, n - first, cmps);
    merge(a, b, cmps)
}

fn random_ints(r: &mut Pcg, n: usize) -> Vec<i64> {
    (0..n).map(|_| r.int_n(1_000_000) as i64).collect()
}

fn main() {
    let mut r = Pcg::new(1, 2);

    println!("1. merge two sorted lists by relinking");
    let a = from_slice(&[1, 3, 5, 7]);
    let b = from_slice(&[2, 3, 6, 8, 9, 10]);
    println!("  a = {}\n  b = {}", show(&a, 20), show(&b, 20));
    let mut cmps = 0;
    let m = merge(a, b, &mut cmps);
    println!("  merge(a, b) = {}", show(&m, 20));
    println!(
        "  {cmps} comparisons for {} nodes: the loop stops when a runs out,",
        to_vec(&m).len()
    );
    println!("  and the rest of b (8 -> 9 -> 10) is attached with one move");

    println!();
    println!("2. allocations during the merge only (inputs built before measuring)");
    const HALF: i64 = 50_000;
    let evens: Vec<i64> = (0..HALF).map(|i| 2 * i).collect();
    let odds: Vec<i64> = (0..HALF).map(|i| 2 * i + 1).collect();
    let (la, lb) = (from_slice(&evens), from_slice(&odds));
    let (mut c1, mut c2) = (0, 0);
    let (list_allocs, lm) = allocs_of(|| merge(la, lb, &mut c1));
    let (vec_allocs, vm) = allocs_of(|| merge_vecs(&evens, &odds, &mut c2));
    println!(
        "  {:<14} {:>12} {:>12} {:>10}",
        "2 x 50,000", "comparisons", "allocations", "same out"
    );
    println!(
        "  {:<14} {c1:>12} {list_allocs:>12} {:>10}",
        "list merge", true
    );
    println!(
        "  {:<14} {c2:>12} {vec_allocs:>12} {:>10}",
        "Vec merge",
        to_vec(&lm) == vm
    );
    println!("  the Vec merge's one allocation is the 100,000-element output;");
    println!("  the list merge's output is the 100,000 nodes it was given");

    println!();
    println!("3. k-way merge: k = 8 sorted lists of 1,000 (n = 8,000)");
    const K: usize = 8;
    const EACH: usize = 1_000;
    let runs: Vec<Vec<i64>> = (0..K)
        .map(|_| {
            let mut v = random_ints(&mut r, EACH);
            v.sort_unstable();
            v
        })
        .collect();
    let mut all: Vec<i64> = runs.concat();
    all.sort_unstable();
    let n = K * EACH;

    // Sequential: fold every list into one growing accumulator; the
    // accumulator is re-walked by every merge, so cost ~ n*k/2.
    let mut seq = 0;
    let mut acc: Link = None;
    for run in &runs {
        acc = merge(acc, from_slice(run), &mut seq);
    }
    let seq_ok = to_vec(&acc) == all;

    // Pairwise (tournament): merge neighbours, halving the number of lists
    // each round. The Vec of lists is consumed with into_iter(): each round
    // MOVES its lists into the merges, so a stale list cannot be reused.
    let mut pair = 0;
    let mut lists: Vec<Link> = runs.iter().map(|run| from_slice(run)).collect();
    while lists.len() > 1 {
        let mut next = Vec::with_capacity(lists.len().div_ceil(2));
        let mut it = lists.into_iter();
        while let Some(x) = it.next() {
            match it.next() {
                Some(y) => next.push(merge(x, y, &mut pair)),
                None => next.push(x), // odd one out goes to the next round
            }
        }
        lists = next;
    }
    let pair_ok = to_vec(&lists[0]) == all;
    println!(
        "  {:<12} {:>12} {:>18} {:>8}",
        "strategy", "comparisons", "reference", "sorted"
    );
    println!(
        "  {:<12} {seq:>12} {:>18} {seq_ok:>8}",
        "sequential",
        format!("n*k/2 = {}", n * K / 2)
    );
    println!(
        "  {:<12} {pair:>12} {:>18} {pair_ok:>8}",
        "pairwise",
        format!("n*log2 k = {}", n * K.ilog2() as usize)
    );
    println!(
        "  pairwise does {:.2}x fewer comparisons at k = {K}; the gap grows as k/(2 log2 k)",
        seq as f64 / pair as f64
    );

    println!();
    println!("4. merge sort on a list: split at ceil(n/2), merge, no extra array");
    println!(
        "  {:>9} {:>12} {:>12} {:>8} {:>12} {:>8}",
        "n", "comparisons", "n*log2 n", "ratio", "allocations", "correct"
    );
    for size in [1_000, 100_000] {
        let input = random_ints(&mut r, size);
        let mut want = input.clone();
        want.sort_unstable();
        let h = from_slice(&input);
        let mut c = 0;
        let (allocs, h) = allocs_of(|| merge_sort(h, size, &mut c));
        let ideal = size as f64 * (size as f64).log2();
        println!(
            "  {size:>9} {c:>12} {ideal:>12.0} {:>8.3} {allocs:>12} {:>8}",
            c as f64 / ideal,
            to_vec(&h) == want
        );
    }
    println!("  comparisons stay under n*log2 n, and the sort allocates nothing:");
    println!("  every merge moves Boxes that already exist");

    println!();
    println!("5. the pitfall: merge consumes its inputs");
    let a = from_slice(&[1, 3, 5]);
    let b = from_slice(&[6, 7]);
    let mut cmps = 0;
    let m = merge(a, b, &mut cmps);
    println!("  first merge(a, b)  = {}", show(&m, 12));
    // The Go program now calls merge(a, b) a second time and gets a cycle.
    // Here a and b were MOVED into the first call, and rustc 1.98.1 says:
    //   error[E0382]: use of moved value: `a`
    //   move occurs because `a` has type `Option<Box<Node>>`, which does not
    //   implement the `Copy` trait
    //   (and the same for `b`)
    // A Box list cannot hold a cycle at all: a node would have to own itself.
    println!("  second merge(a, b) = does not compile: E0382 use of moved value `a` (and `b`)");
    println!("  a and b were moved into the first call; there is no stale head to reuse,");
    println!("  and no Box chain can point back at itself. The section ends here.");

    println!();
    println!("6. timing: list merge sort vs slice sorts, same 100,000 PCG values (best of 5)");
    let input = random_ints(&mut r, 100_000);
    // Best of 5; `prep` (untimed) rebuilds the input, because sorting mutates it.
    fn best<T>(mut prep: impl FnMut() -> T, mut run: impl FnMut(T) -> T) -> (Duration, T) {
        let mut best_d = Duration::MAX;
        let mut out = None;
        for _ in 0..5 {
            let x = prep();
            let t = Instant::now();
            let y = run(x);
            best_d = best_d.min(t.elapsed());
            out = Some(y); // the previous result is dropped here, untimed
        }
        (best_d, out.unwrap())
    }
    let mut c = 0;
    let (t_list, h) = best(
        || from_slice(&input),
        |h| merge_sort(h, input.len(), &mut c),
    );
    let (t_unst, xs) = best(
        || input.clone(),
        |mut v| {
            v.sort_unstable();
            v
        },
    );
    let (t_stab, ys) = best(
        || input.clone(),
        |mut v| {
            v.sort();
            v
        },
    );
    let ms = |d: Duration| d.as_micros() as f64 / 1000.0;
    println!("  {:<30} {:>10.2} ms", "list merge sort", ms(t_list));
    println!(
        "  {:<30} {:>10.2} ms",
        "sort_unstable (ipnsort)",
        ms(t_unst)
    );
    println!(
        "  {:<30} {:>10.2} ms",
        "sort (driftsort, stable)",
        ms(t_stab)
    );
    if t_list > t_unst {
        println!(
            "  sort_unstable is {:.1}x faster: zero allocations does not beat contiguous memory",
            t_list.as_secs_f64() / t_unst.as_secs_f64()
        );
    } else {
        println!(
            "  the list sort is {:.1}x faster than sort_unstable on this run",
            t_unst.as_secs_f64() / t_list.as_secs_f64()
        );
    }
    println!("  (results agree: {})", to_vec(&h) == xs && xs == ys);
    // Allocations per call, counted once outside the timing.
    let mut v = input.clone();
    let (a_unst, ()) = allocs_of(|| v.sort_unstable());
    let mut v = input.clone();
    let (a_stab, ()) = allocs_of(|| v.sort());
    let l = from_slice(&input);
    let (a_list, _l) = allocs_of(|| merge_sort(l, input.len(), &mut c));
    println!(
        "  allocations per sort: list merge sort {a_list}, sort_unstable {a_unst}, sort {a_stab} (its scratch buffer)"
    );
}
```

**Rust sample output:**

```
1. merge two sorted lists by relinking
  a = [1 -> 3 -> 5 -> 7]
  b = [2 -> 3 -> 6 -> 8 -> 9 -> 10]
  merge(a, b) = [1 -> 2 -> 3 -> 3 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10]
  7 comparisons for 10 nodes: the loop stops when a runs out,
  and the rest of b (8 -> 9 -> 10) is attached with one move

2. allocations during the merge only (inputs built before measuring)
  2 x 50,000      comparisons  allocations   same out
  list merge            99999            0       true
  Vec merge             99999            1       true
  the Vec merge's one allocation is the 100,000-element output;
  the list merge's output is the 100,000 nodes it was given

3. k-way merge: k = 8 sorted lists of 1,000 (n = 8,000)
  strategy      comparisons          reference   sorted
  sequential          34970      n*k/2 = 32000     true
  pairwise            23984   n*log2 k = 24000     true
  pairwise does 1.46x fewer comparisons at k = 8; the gap grows as k/(2 log2 k)

4. merge sort on a list: split at ceil(n/2), merge, no extra array
          n  comparisons     n*log2 n    ratio  allocations  correct
       1000         8708         9966    0.874            0     true
     100000      1535951      1660964    0.925            0     true
  comparisons stay under n*log2 n, and the sort allocates nothing:
  every merge moves Boxes that already exist

5. the pitfall: merge consumes its inputs
  first merge(a, b)  = [1 -> 3 -> 5 -> 6 -> 7]
  second merge(a, b) = does not compile: E0382 use of moved value `a` (and `b`)
  a and b were moved into the first call; there is no stale head to reuse,
  and no Box chain can point back at itself. The section ends here.

6. timing: list merge sort vs slice sorts, same 100,000 PCG values (best of 5)
  list merge sort                      7.40 ms
  sort_unstable (ipnsort)              0.77 ms
  sort (driftsort, stable)             0.99 ms
  sort_unstable is 9.7x faster: zero allocations does not beat contiguous memory
  (results agree: true)
  allocations per sort: list merge sort 0, sort_unstable 0, sort 1 (its scratch buffer)
```

> **Go vs Rust:** Every count in sections 1–4 is identical (99,999 comparisons with 0 vs 1 allocations; 34,970 vs 23,984 for the k-way merge; 1,535,951 for the 100,000-node sort). The escape hatches: a `&mut Link` tail cursor replaces Go's stack sentinel, the merge sort passes the length down because Go's slow/fast split is a `&` and a `&mut` cursor in one chain (E0502), and `merge` takes both lists by value, so section 5's second `merge(a, b)` is E0382 and the cycle Go builds cannot exist. Timings (sample, shared machine): the two list sorts took similar time in our quiet runs (~7–8 ms), but `sort_unstable` (ipnsort) was ~6× faster than Go's `slices.Sort` (pdqsort) on the same values, so the list loses ~10× in Rust and ~1.5× in Go — the slice sort got faster, not the list sort slower.

**Complexity:** merge Θ(m + n) time, Θ(1) extra space, **0 allocations** (the slice merge: 1, its output) · merge sort Θ(n log n) time, Θ(log n) stack, 0 allocations, about 0.87–0.93 · n·log2 n comparisons here · k-way: pairwise 23,984 comparisons vs sequential 34,970 for k = 8 lists of 1,000 · the list sort was roughly 1.4–1.5× slower than `slices.Sort` on 100,000 ints in our runs.

---

## 11. `container/list`, read in the source

`🟡 medium` · *the ring-with-sentinel, before generics*

Go's `container/list` is exactly example 6's design — a doubly linked ring around a sentinel `root Element`
embedded in the `List` — and it predates generics, so every value is an `any`. Read `list.go` first (it is ~240
lines), then measure what that costs and find the holes: a value of the wrong type compiles, removing an element
from the wrong list silently does nothing, and `PushBackList` copies instead of splicing. Every operation at a held
element is Θ(1); the price is paid in memory — an `Element` plus a boxed value per int. (Example 7 of the queues
lesson measured it as a queue; this one is about its API and handle semantics.)

**Steps:**

1. Read `$(go env GOROOT)/src/container/list/list.go` and check each fact quoted in the program's comments.
2. Poke the holes on purpose: push a string into a list of ints, `Remove` and `MoveToFront` an element of another
   list, `PushBackList` 1,000 values while counting heap objects.
3. Compare heap objects, bytes and traversal time for 100,000 ints: `container/list`, a generic `GList[int]`, `[]int`.

```go
package main

import (
	"container/list"
	"fmt"
	"math"
	"runtime"
	"runtime/debug"
	"strings"
	"testing"
	"unsafe"
)

// container/list, read in the source ($(go env GOROOT)/src/container/list/list.go).
// What the source says, by function:
//
//   - type List struct { root Element; len int }: the sentinel is EMBEDDED, not
//     allocated. "internally a list l is implemented as a ring, such that
//     &l.root is both the next element of the last list element (l.Back())
//     and the previous element of the first list element (l.Front())" --
//     exactly the ring-with-sentinel of example 6.
//   - lazyInit: "if l.root.next == nil { l.Init() }", called by PushFront,
//     PushBack, PushBackList, PushFrontList -- so the zero List works.
//   - Element.Next: returns nil when e.list == nil or the next is &e.list.root;
//     that is how the ring looks nil-terminated from the outside.
//   - remove: sets e.next, e.prev and e.list to nil ("avoid memory leaks").
//   - Remove, MoveToFront, MoveToBack, InsertBefore, InsertAfter, ...: all
//     start with `if e.list != l` and silently do nothing for a foreign
//     element. Remove still returns e.Value.
//   - PushBackList: "inserts a copy of another list" -- a loop of insertValue,
//     one new Element per value. Not a splice.
//   - Element.Value is `any`: the package predates generics (Go 1.18).

// GList is the generic version of the same design (example 6), for comparison.
type gnode[T any] struct {
	val        T
	next, prev *gnode[T]
}

type GList[T any] struct {
	root gnode[T] // embedded sentinel, like container/list
	len  int
}

func (l *GList[T]) PushBack(v T) {
	if l.root.next == nil { // lazyInit, as in container/list
		l.root.next, l.root.prev = &l.root, &l.root
	}
	n := &gnode[T]{val: v, prev: l.root.prev, next: &l.root}
	n.prev.next = n
	l.root.prev = n
	l.len++
}

func show(l *list.List) string {
	var parts []string
	for e := l.Front(); e != nil; e = e.Next() {
		parts = append(parts, fmt.Sprint(e.Value))
	}
	return "[" + strings.Join(parts, " -> ") + "]"
}

func fromInts(xs ...int) *list.List {
	l := list.New()
	for _, x := range xs {
		l.PushBack(x)
	}
	return l
}

// heap reads (objects ever allocated, bytes ever allocated).
func heap() (uint64, uint64) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.Mallocs, m.TotalAlloc
}

func main() {
	fmt.Println("1. the design, as the source says")
	var l list.List // the zero value, no New(): lazyInit wires the ring on first push
	l.PushBack(1)
	two := l.PushBack(2)
	l.PushBack(3)
	fmt.Printf("  zero List after 3 PushBacks: %s, Len %d\n", show(&l), l.Len())
	fmt.Printf("  unsafe.Sizeof(list.Element{}) = %d bytes (next, prev, list + a 2-word any)\n",
		unsafe.Sizeof(list.Element{}))
	fmt.Printf("  unsafe.Sizeof(list.List{})    = %d bytes (the embedded root Element + len)\n",
		unsafe.Sizeof(list.List{}))
	fmt.Printf("  l.Remove(two) = %v, list now %s\n", l.Remove(two), show(&l))
	fmt.Printf("  after Remove: two.Next() == nil: %v, two.Prev() == nil: %v\n",
		two.Next() == nil, two.Prev() == nil)

	fmt.Println()
	fmt.Println("2. the holes")
	ints := fromInts(10, 20, 30)
	ints.PushBack("oops") // compiles: Value is any
	ints.PushBack(40)
	sum, bad := 0, 0
	for e := ints.Front(); e != nil; e = e.Next() {
		if v, ok := e.Value.(int); ok { // the plain e.Value.(int) would panic here
			sum += v
		} else {
			bad++
		}
	}
	fmt.Printf("  %s: sum of ints %d, failed assertions %d\n", show(ints), sum, bad)

	l1, l2 := fromInts(1, 2, 3), fromInts(7, 8, 9)
	e1 := l1.Front().Next() // the element holding 2, which belongs to l1
	got := l2.Remove(e1)    // wrong list: e1.list != l2, so nothing is unlinked
	fmt.Printf("  l2.Remove(element of l1) returned %v; l1 = %s (Len %d), l2 = %s (Len %d)\n",
		got, show(l1), l1.Len(), show(l2), l2.Len())
	l2.MoveToFront(e1) // also a silent no-op
	fmt.Printf("  l2.MoveToFront(element of l1): l1 = %s, l2 = %s\n", show(l1), show(l2))
	fmt.Printf("  l2.InsertAfter(99, element of l1) returned nil: %v\n", l2.InsertAfter(99, e1) == nil)
	fmt.Println("  no error, no panic: a handle into the wrong list is silently ignored")

	fmt.Println()
	fmt.Println("3. PushBackList copies; it is not a splice")
	// Sections 3-4 count heap objects. A GC cycle that happens to run inside
	// the counted window allocates a few dozen objects of its own (we saw
	// 39-40 on 100,000 pushes), so the collector is off while we count.
	gcPercent := debug.SetGCPercent(-1)
	other := list.New()
	for i := range 1_000 {
		other.PushBack(i)
	}
	dst := fromInts(-3, -2, -1)
	o0, _ := heap()
	dst.PushBackList(other)
	o1, _ := heap()
	fmt.Printf("  dst.PushBackList(other of 1,000): heap objects allocated %d\n", o1-o0)
	fmt.Printf("  dst.Len() = %d, other.Len() = %d (other is untouched)\n", dst.Len(), other.Len())
	fmt.Printf("  other.Front() is dst's 4th element: %v (they are different Elements)\n",
		other.Front() == dst.Front().Next().Next().Next())
	self := fromInts(1, 2, 3)
	self.PushBackList(self) // "The lists l and other may be the same."
	fmt.Printf("  l.PushBackList(l) on [1 -> 2 -> 3]: %s\n", show(self))

	fmt.Println()
	const n = 100_000
	fmt.Printf("4. heap objects to store %d ints (values 0..%d)\n", n, n-1)
	fmt.Printf("  %-16s %10s %16s %10s\n", "", "objects", "bytes allocated", "bytes/int")
	o0, b0 := heap()
	cl := list.New()
	for i := range n {
		cl.PushBack(i) // one Element + one 8-byte box for i >= 256
	}
	o1, b1 := heap()
	gl := &GList[int]{}
	for i := range n {
		gl.PushBack(i) // one gnode[int] (24 bytes), the int stored inline
	}
	o2, b2 := heap()
	var xs []int
	for i := range n {
		xs = append(xs, i) // amortised growth: a handful of reallocations
	}
	o3, b3 := heap()
	fmt.Printf("  %-16s %10d %16d %10.1f\n", "container/list", o1-o0, b1-b0, float64(b1-b0)/n)
	fmt.Printf("  %-16s %10d %16d %10.1f\n", "GList[int]", o2-o1, b2-b1, float64(b2-b1)/n)
	fmt.Printf("  %-16s %10d %16d %10.1f\n", "[]int (append)", o3-o2, b3-b2, float64(b3-b2)/n)
	fmt.Printf("  container/list: %d Elements + 1 List + %d boxed ints -- values 0..255 box for free\n",
		n, (o1-o0)-n-1)
	fmt.Println("  (the runtime's convT64 points small ints at a static table, src/runtime/iface.go)")
	debug.SetGCPercent(gcPercent)

	fmt.Println()
	fmt.Printf("5. sum-traversal of the same %d ints (lists freshly built, nodes in order; best of 3)\n", n)
	var sink int
	// Best of 3 benchmark runs: the machine is shared, and the minimum is the
	// run least disturbed by everything else.
	bench := func(f func() int) float64 {
		best := math.Inf(1)
		for range 3 {
			r := testing.Benchmark(func(b *testing.B) {
				for b.Loop() {
					sink += f()
				}
			})
			best = min(best, float64(r.T.Nanoseconds())/float64(r.N))
		}
		return best
	}
	tSlice := bench(func() int {
		s := 0
		for _, x := range xs {
			s += x
		}
		return s
	})
	tGen := bench(func() int {
		s := 0
		for p := gl.root.next; p != &gl.root; p = p.next {
			s += p.val
		}
		return s
	})
	tCL := bench(func() int {
		s := 0
		for e := cl.Front(); e != nil; e = e.Next() {
			s += e.Value.(int) // a type assertion on every read
		}
		return s
	})
	fmt.Printf("  %-16s %12s %10s %10s\n", "", "ns/op", "ns/int", "vs slice")
	for _, row := range []struct {
		name string
		t    float64
	}{{"[]int", tSlice}, {"GList[int]", tGen}, {"container/list", tCL}} {
		fmt.Printf("  %-16s %12.0f %10.2f %9.1fx\n", row.name, row.t, row.t/n, row.t/tSlice)
	}
	if tCL > tGen {
		fmt.Printf("  container/list is %.2fx GList's time: the box and the assertion add %.0f%%\n",
			tCL/tGen, (tCL/tGen-1)*100)
		fmt.Println("  to the pointer chase both lists pay; the bigger bill is section 4's memory")
	} else {
		fmt.Printf("  container/list matched GList on this run (%.2fx)\n", tCL/tGen)
	}
	_ = sink
}
```

**Sample output:**

```
1. the design, as the source says
  zero List after 3 PushBacks: [1 -> 2 -> 3], Len 3
  unsafe.Sizeof(list.Element{}) = 40 bytes (next, prev, list + a 2-word any)
  unsafe.Sizeof(list.List{})    = 48 bytes (the embedded root Element + len)
  l.Remove(two) = 2, list now [1 -> 3]
  after Remove: two.Next() == nil: true, two.Prev() == nil: true

2. the holes
  [10 -> 20 -> 30 -> oops -> 40]: sum of ints 100, failed assertions 1
  l2.Remove(element of l1) returned 2; l1 = [1 -> 2 -> 3] (Len 3), l2 = [7 -> 8 -> 9] (Len 3)
  l2.MoveToFront(element of l1): l1 = [1 -> 2 -> 3], l2 = [7 -> 8 -> 9]
  l2.InsertAfter(99, element of l1) returned nil: true
  no error, no panic: a handle into the wrong list is silently ignored

3. PushBackList copies; it is not a splice
  dst.PushBackList(other of 1,000): heap objects allocated 1000
  dst.Len() = 1003, other.Len() = 1000 (other is untouched)
  other.Front() is dst's 4th element: false (they are different Elements)
  l.PushBackList(l) on [1 -> 2 -> 3]: [1 -> 2 -> 3 -> 1 -> 2 -> 3]

4. heap objects to store 100000 ints (values 0..99999)
                      objects  bytes allocated  bytes/int
  container/list       199745          5598000       56.0
  GList[int]           100001          2400032       24.0
  []int (append)           28          4101376       41.0
  container/list: 100000 Elements + 1 List + 99744 boxed ints -- values 0..255 box for free
  (the runtime's convT64 points small ints at a static table, src/runtime/iface.go)

5. sum-traversal of the same 100000 ints (lists freshly built, nodes in order; best of 3)
                          ns/op     ns/int   vs slice
  []int                   26914       0.27       1.0x
  GList[int]             106316       1.06       4.0x
  container/list         124197       1.24       4.6x
  container/list is 1.17x GList's time: the box and the assertion add 17%
  to the pointer chase both lists pay; the bigger bill is section 4's memory
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::LinkedList;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// std::collections::LinkedList, read in the source
// ($(rustc --print sysroot)/lib/rustlib/src/rust/library/alloc/src/collections/linked_list.rs).
// What the source says:
//
//   - struct LinkedList<T> { head: Option<NonNull<Node<T>>>, tail: ..., len }
//     and struct Node<T> { next, prev: Option<NonNull<Node<T>>>, element: T }.
//     Doubly linked through RAW pointers (NonNull) inside unsafe code -- the
//     shape safe Rust cannot express with Box. No sentinel, no ring: the ends
//     are None.
//   - It is generic (Rust 1.0 had generics), so element: T sits inline in the
//     node. No `any`, no box, no type assertion.
//   - `append` is "O(1) time and O(1) memory": it rewires two pointers and
//     leaves `other` empty. `split_off(at)` walks from the nearer end, then cuts.
//   - The things a list is FOR -- holding a position and removing or inserting
//     there in O(1) -- are cursors (cursor_front_mut, insert_after,
//     remove_current, splice_after), all #[unstable(feature =
//     "linked_list_cursors")]. remove(at) is unstable (`linked_list_remove`),
//     retain is unstable (`linked_list_retain`). extract_if is stable since 1.87.0.
//   - Drop is a loop: `while guard.0.pop_front_node().is_some() {}` -- no
//     recursion, so a long list cannot overflow the stack when dropped.
//   - The module doc's own advice, quoted in main below.

// ---- allocation counter: stand-in for runtime.MemStats (Mallocs, TotalAlloc) ----
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

/// (allocation events, bytes requested, result) while running f.
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

/// Best of 3 ns_per_op runs: the machine is shared, and the minimum is the
/// run least disturbed by everything else.
fn best_of_3(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// ---- the hand-written list, for comparison ----
// Go's comparison list is a generic DOUBLY linked ring. In safe Rust a node
// with two owners (prev and next) needs Rc<RefCell<_>> + Weak or raw pointers;
// std's LinkedList already IS the raw-pointer version. So the hand-written one
// here is the safe singly linked Option<Box> list: 16 bytes per i64 node.
struct Node<T> {
    val: T,
    next: Option<Box<Node<T>>>,
}

struct List<T> {
    head: Option<Box<Node<T>>>,
}

impl<T> List<T> {
    fn from_iter(xs: impl IntoIterator<Item = T>) -> Self {
        let mut head = None;
        let mut tail = &mut head; // the &mut Link cursor: append without a tail field
        for val in xs {
            tail = &mut tail.insert(Box::new(Node { val, next: None })).next;
        }
        List { head }
    }
}

// Iterative Drop, like std's: the compiler-generated one would recurse once
// per node (example 7) and 100,000 nodes would put that on the stack.
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur = self.head.take();
        while let Some(mut n) = cur {
            cur = n.next.take();
        }
    }
}

fn show(l: &LinkedList<i64>) -> String {
    let parts: Vec<String> = l.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" -> "))
}

/// Remove the element at index i with STABLE API: split, pop, re-join.
/// split_off walks min(i, len - i) nodes; pop_front and append are O(1).
fn remove_at(l: &mut LinkedList<i64>, i: usize) -> Option<i64> {
    let mut rest = l.split_off(i);
    let v = rest.pop_front();
    l.append(&mut rest);
    v
}

fn main() {
    println!("1. the design, as the source says");
    // LinkedList::new() is a const fn: no allocation, ends are None.
    let (a, _, mut l) = allocs_of(LinkedList::<i64>::new);
    println!("  LinkedList::new(): {a} allocations (the empty list is three words, no sentinel)");
    let (a, bytes, ()) = allocs_of(|| {
        l.push_back(1);
        l.push_back(2);
        l.push_back(3);
    });
    println!("  after 3 push_backs: {}, len {}", show(&l), l.len());
    println!(
        "  size_of::<LinkedList<i64>>() = {} bytes (head, tail, len)",
        size_of::<LinkedList<i64>>()
    );
    println!(
        "  each push_back: {} allocation of {} bytes (next, prev, an inline i64)",
        a / 3,
        bytes / 3
    );
    // No handle to the 2 exists on stable Rust. To remove it, search for it:
    // extract_if (stable since 1.87.0) walks the list -- Theta(n), not Theta(1).
    let removed: Vec<i64> = l.extract_if(|x| *x == 2).collect();
    println!(
        "  extract_if(|x| *x == 2) = {removed:?}, list now {}",
        show(&l)
    );
    println!("  the removed value came back BY VALUE: there is no Element left to dangle");
    println!("  std's own doc, linked_list.rs (read in the source, not run):");
    println!("    \"NOTE: It is almost always better to use Vec or VecDeque because");
    println!("    array-based containers are generally faster, more memory efficient,");
    println!("    and make better use of CPU cache.\"");

    println!();
    println!("2. the holes");
    let mut ints = LinkedList::from([10, 20, 30]);
    // ints.push_back("oops");
    //   error[E0308]: mismatched types
    //   expected `i64`, found `&str`
    ints.push_back(40);
    let sum: i64 = ints.iter().sum();
    println!("  push_back(\"oops\") onto a LinkedList<i64>: does not compile (E0308)");
    println!(
        "  {}: sum {sum}, no assertions at all -- the element type is i64",
        show(&ints)
    );

    // Go's hole is a handle into the wrong list. On stable Rust there are no
    // handles: cursor_front_mut() etc. are
    //   error[E0658]: use of unstable library feature `linked_list_cursors`
    // and l.remove(0) is
    //   error[E0658]: use of unstable library feature `linked_list_remove`
    // So "remove the element you hold" becomes "remove the element at an index".
    let (mut l1, mut l2) = (LinkedList::from([1, 2, 3]), LinkedList::from([7, 8, 9]));
    let got = remove_at(&mut l1, 1);
    println!(
        "  remove_at(&mut l1, 1) returned {got:?}; l1 = {} (len {}), l2 = {} (len {})",
        show(&l1),
        l1.len(),
        show(&l2),
        l2.len()
    );
    // MoveToFront, the same way: Theta(index) to find it, then O(1) to move it.
    if let Some(v) = remove_at(&mut l2, 2) {
        l2.push_front(v);
    }
    println!("  move index 2 of l2 to the front: l2 = {}", show(&l2));
    println!("  a wrong-list mistake cannot happen silently: there is nothing to pass;");
    println!("  the cost is that every positional operation walks, Theta(min(i, n - i))");

    println!();
    println!("3. append is a splice, not a copy");
    let mut other: LinkedList<i64> = (0..1_000).collect();
    let mut dst = LinkedList::from([-3, -2, -1]);
    let (a, _, ()) = allocs_of(|| dst.append(&mut other));
    println!("  dst.append(&mut other of 1,000): allocations {a}");
    println!(
        "  dst.len() = {}, other.len() = {} (other was EMPTIED: its nodes moved)",
        dst.len(),
        other.len()
    );
    let (a, _, tail) = allocs_of(|| dst.split_off(3));
    println!(
        "  dst.split_off(3) gives the 1,000 back: allocations {a}, dst.len() = {}, tail.len() = {}",
        dst.len(),
        tail.len()
    );
    // Go's PushBackList semantics, a copy, is extend over an iterator:
    let (a, _, ()) = allocs_of(|| dst.extend(tail.iter().copied()));
    println!(
        "  dst.extend(tail.iter().copied()) -- Go's copy: allocations {a}, tail.len() = {}",
        tail.len()
    );
    // self.append(&mut self) is
    //   error[E0499]: cannot borrow `l` as mutable more than once at a time
    // so Go's l.PushBackList(l) needs an explicit clone first.
    let mut me = LinkedList::from([1, 2, 3]);
    let mut copy = me.clone();
    me.append(&mut copy);
    println!(
        "  l.append(&mut l) does not compile (E0499); clone, then append: {}",
        show(&me)
    );

    println!();
    const N: usize = 100_000;
    println!("4. heap objects to store {N} ints (values 0..{})", N - 1);
    println!(
        "  {:<18} {:>10} {:>16} {:>10}",
        "", "objects", "bytes allocated", "bytes/int"
    );
    let (o1, b1, ll) = allocs_of(|| (0..N as i64).collect::<LinkedList<i64>>());
    let (o2, b2, hl) = allocs_of(|| List::from_iter(0..N as i64));
    let (o3, b3, v) = allocs_of(|| {
        let mut v = Vec::new();
        for i in 0..N as i64 {
            v.push(i); // amortised doubling: a handful of reallocations
        }
        v
    });
    for (name, o, b) in [
        ("LinkedList<i64>", o1, b1),
        ("List<i64> (Box)", o2, b2),
        ("Vec<i64> (push)", o3, b3),
    ] {
        println!("  {name:<18} {o:>10} {b:>16} {:>10.1}", b as f64 / N as f64);
    }
    println!(
        "  LinkedList: one 24-byte node per int, the i64 inline -- no box, whatever the value"
    );

    println!();
    println!(
        "5. sum-traversal of the same {N} ints (lists freshly built, nodes in order; best of 3)"
    );
    let t_vec = best_of_3(|| {
        black_box(black_box(&v).iter().sum::<i64>());
    });
    let t_hand = best_of_3(|| {
        let mut s = 0i64;
        let mut cur = &black_box(&hl).head;
        while let Some(n) = cur {
            s += n.val;
            cur = &n.next;
        }
        black_box(s);
    });
    let t_std = best_of_3(|| {
        black_box(black_box(&ll).iter().sum::<i64>());
    });
    println!(
        "  {:<18} {:>12} {:>10} {:>10}",
        "", "ns/op", "ns/int", "vs Vec"
    );
    for (name, t) in [
        ("Vec<i64>", t_vec),
        ("List<i64> (Box)", t_hand),
        ("LinkedList<i64>", t_std),
    ] {
        println!(
            "  {name:<18} {t:>12.0} {:>10.2} {:>9.1}x",
            t / N as f64,
            t / t_vec
        );
    }
    println!(
        "  LinkedList vs the hand-written list: {:.2}x -- both are one pointer chase per int",
        t_std / t_hand
    );
}
```

**Rust sample output:**

```
1. the design, as the source says
  LinkedList::new(): 0 allocations (the empty list is three words, no sentinel)
  after 3 push_backs: [1 -> 2 -> 3], len 3
  size_of::<LinkedList<i64>>() = 24 bytes (head, tail, len)
  each push_back: 1 allocation of 24 bytes (next, prev, an inline i64)
  extract_if(|x| *x == 2) = [2], list now [1 -> 3]
  the removed value came back BY VALUE: there is no Element left to dangle
  std's own doc, linked_list.rs (read in the source, not run):
    "NOTE: It is almost always better to use Vec or VecDeque because
    array-based containers are generally faster, more memory efficient,
    and make better use of CPU cache."

2. the holes
  push_back("oops") onto a LinkedList<i64>: does not compile (E0308)
  [10 -> 20 -> 30 -> 40]: sum 100, no assertions at all -- the element type is i64
  remove_at(&mut l1, 1) returned Some(2); l1 = [1 -> 3] (len 2), l2 = [7 -> 8 -> 9] (len 3)
  move index 2 of l2 to the front: l2 = [9 -> 7 -> 8]
  a wrong-list mistake cannot happen silently: there is nothing to pass;
  the cost is that every positional operation walks, Theta(min(i, n - i))

3. append is a splice, not a copy
  dst.append(&mut other of 1,000): allocations 0
  dst.len() = 1003, other.len() = 0 (other was EMPTIED: its nodes moved)
  dst.split_off(3) gives the 1,000 back: allocations 0, dst.len() = 3, tail.len() = 1000
  dst.extend(tail.iter().copied()) -- Go's copy: allocations 1000, tail.len() = 1000
  l.append(&mut l) does not compile (E0499); clone, then append: [1 -> 2 -> 3 -> 1 -> 2 -> 3]

4. heap objects to store 100000 ints (values 0..99999)
                        objects  bytes allocated  bytes/int
  LinkedList<i64>        100000          2400000       24.0
  List<i64> (Box)        100000          1600000       16.0
  Vec<i64> (push)            16          2097120       21.0
  LinkedList: one 24-byte node per int, the i64 inline -- no box, whatever the value

5. sum-traversal of the same 100000 ints (lists freshly built, nodes in order; best of 3)
                            ns/op     ns/int     vs Vec
  Vec<i64>                   7281       0.07       1.0x
  List<i64> (Box)          100455       1.00      13.8x
  LinkedList<i64>           93721       0.94      12.9x
  LinkedList vs the hand-written list: 0.93x -- both are one pointer chase per int
```

> **Go vs Rust:** Go's three holes close at compile time (`push_back("oops")` is E0308, `l.append(&mut l)` is E0499, and there are no element handles to pass to the wrong list) — but on stable Rust the handle API does not exist at all: cursors, `remove(at)` and `retain` are unstable (`linked_list_cursors`, `linked_list_remove`, `linked_list_retain`, read in the source and confirmed as E0658), so removing a held node in Θ(1), the one job a list is for, falls back to `extract_if` or `split_off`+`pop_front`+`append`, which walk. `append` is a real splice (0 allocations, `other` emptied) where `PushBackList` makes 1,000 and leaves `other` alone, and 100,000 ints cost 100,000 objects / 2,400,000 bytes in `LinkedList` against Go's 199,745 / 5,598,000, because the `i64` sits inline instead of in a box. Per int the list traversals were close in quiet runs (~0.9–1.0 ns Rust, ~1.0–1.3 ns Go); `Vec`'s 10–14× lead (Go's slice: ~4×) is plausibly the compiler vectorising the slice sum (not measured).

**Complexity:** every operation Θ(1) at a held element · `PushBackList` Θ(m) time and exactly m allocations (one new `Element` per value; the source list is untouched) · per element: one 40-byte `Element` (48-byte size class) plus an 8-byte box for any int ≥ 256 — 199,745 heap objects and 56 bytes per int for 100,000 ints, against 100,001 objects and 24 bytes for a generic list · traversal roughly 3–5× a slice's time in our runs, and `container/list` was within about 1.0–1.2× of the generic list.

---

> Next tier: [🔴 hard](3-hard.md) — the honest list-vs-slice verdict, the arena and the unrolled list,
> palindrome in Θ(1) space, and the LRU cache.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
