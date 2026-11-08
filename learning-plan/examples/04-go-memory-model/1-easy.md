# Step 04 — Go's Memory Model · 🟢 Easy

Examples **1–6**: what things actually cost, and where they actually live.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Examples 1, 2 and 4 are **deterministic** on any 64-bit machine. Examples 3, 5 and 6 measure the
heap, so the last digits move a little between runs — the numbers that matter (8 MB pinned,
allocations 0 or 1, bytes per element) are stable.

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

## 1. What a variable costs

`🟢 easy` · *Headers vs data*

Start here, because every other surprise in this lesson follows from it. An **array is its data**; a
**slice is a 3-word header** pointing at someone else's data. That's why passing a slice is free and
passing an array is not.

**Steps:**

1. Print `unsafe.Sizeof` for every kind of variable you'll use in this plan.
2. Note which are headers (fixed size) and which are payloads (scale with content).
3. Confirm a slice header is 24 bytes whether it has 10 elements or 10 million.

```go
package main

import (
	"fmt"
	"unsafe"
)

type node struct {
	val  int
	next *node
}

func main() {
	var (
		arr10  [10]int
		arr1e6 [1_000_000]int
		sl     []int
		str    string
		m      map[int]int
		ch     chan int
		fn     func()
		iface  any
		errv   error
		ptr    *node
		nd     node
	)

	fmt.Println("what a VARIABLE of each type costs (the header, not the data):")
	fmt.Println()
	rows := []struct {
		name string
		size uintptr
		note string
	}{
		{"[10]int", unsafe.Sizeof(arr10), "an array IS its data -- 10 * 8"},
		{"[1000000]int", unsafe.Sizeof(arr1e6), "still the data. Passing this by value copies 8 MB"},
		{"[]int", unsafe.Sizeof(sl), "3 words: pointer, len, cap -- the data lives elsewhere"},
		{"string", unsafe.Sizeof(str), "2 words: pointer, len (immutable, so no cap)"},
		{"map[int]int", unsafe.Sizeof(m), "1 word: it is a pointer to a runtime struct"},
		{"chan int", unsafe.Sizeof(ch), "1 word: also a pointer"},
		{"func()", unsafe.Sizeof(fn), "1 word: pointer to a code+closure object"},
		{"any (interface{})", unsafe.Sizeof(iface), "2 words: type pointer + data pointer"},
		{"error", unsafe.Sizeof(errv), "2 words: it is an interface"},
		{"*node", unsafe.Sizeof(ptr), "1 word"},
		{"node{int, *node}", unsafe.Sizeof(nd), "8 + 8"},
	}
	for _, r := range rows {
		fmt.Printf("  %-20s %4d bytes   %s\n", r.name, r.size, r.note)
	}

	fmt.Println()
	fmt.Println("the slice header is THREE WORDS, whatever the slice holds:")
	a := make([]int, 10)
	b := make([]int, 10_000_000)
	fmt.Printf("  len 10:       header %d bytes, len %d, cap %d\n", unsafe.Sizeof(a), len(a), cap(a))
	fmt.Printf("  len 10000000: header %d bytes, len %d, cap %d\n", unsafe.Sizeof(b), len(b), cap(b))

	fmt.Println()
	fmt.Println("which is why passing a slice is O(1) and passing an array is O(n):")
	fmt.Println("  func f(xs []int)      copies 24 bytes")
	fmt.Println("  func f(xs [1000]int)  copies 8000 bytes")
	fmt.Println()
	fmt.Println("an ARRAY is a value. A SLICE is a view onto someone else's array.")
	fmt.Println("Every 'slices are weird' surprise in Go follows from that one sentence.")
}
```

**Output:**

```
what a VARIABLE of each type costs (the header, not the data):

  [10]int                80 bytes   an array IS its data -- 10 * 8
  [1000000]int         8000000 bytes   still the data. Passing this by value copies 8 MB
  []int                  24 bytes   3 words: pointer, len, cap -- the data lives elsewhere
  string                 16 bytes   2 words: pointer, len (immutable, so no cap)
  map[int]int             8 bytes   1 word: it is a pointer to a runtime struct
  chan int                8 bytes   1 word: also a pointer
  func()                  8 bytes   1 word: pointer to a code+closure object
  any (interface{})      16 bytes   2 words: type pointer + data pointer
  error                  16 bytes   2 words: it is an interface
  *node                   8 bytes   1 word
  node{int, *node}       16 bytes   8 + 8

the slice header is THREE WORDS, whatever the slice holds:
  len 10:       header 24 bytes, len 10, cap 10
  len 10000000: header 24 bytes, len 10000000, cap 10000000

which is why passing a slice is O(1) and passing an array is O(n):
  func f(xs []int)      copies 24 bytes
  func f(xs [1000]int)  copies 8000 bytes

an ARRAY is a value. A SLICE is a view onto someone else's array.
Every 'slices are weird' surprise in Go follows from that one sentence.
```

**Rust:**

```rust
use std::any::Any;
use std::collections::HashMap;
use std::mem::size_of;
use std::sync::mpsc::Sender;

// Only its size is measured, so the fields are never read.
#[allow(dead_code)]
struct Node {
    val: i64,
    next: Option<Box<Node>>,
}

// Rust's unsafe.Sizeof(v): the size of a variable, from a reference to it.
fn size_of_var<T>(_: &T) -> usize {
    size_of::<T>()
}

fn main() {
    // A closure is a struct of whatever it captures: nothing, or an i64.
    let no_capture = || 1;
    let n = 7i64;
    let captures_i64 = move || n;

    println!("what a VARIABLE of each type costs (the header, not the data):");
    println!();
    let rows: [(&str, usize, &str); 17] = [
        (
            "[i64; 10]",
            size_of::<[i64; 10]>(),
            "an array IS its data -- 10 * 8",
        ),
        (
            "[i64; 1_000_000]",
            size_of::<[i64; 1_000_000]>(),
            "still the data. Passing this by value copies 8 MB",
        ),
        (
            "Vec<i64>",
            size_of::<Vec<i64>>(),
            "3 words: pointer, capacity, len -- the data lives on the heap",
        ),
        (
            "&[i64]",
            size_of::<&[i64]>(),
            "2 words: pointer, len -- a borrowed view, so no capacity",
        ),
        (
            "&[i64; 10]",
            size_of::<&[i64; 10]>(),
            "1 word: the length is part of the type",
        ),
        (
            "String",
            size_of::<String>(),
            "3 words: it is a Vec<u8> that promises UTF-8",
        ),
        (
            "&str",
            size_of::<&str>(),
            "2 words: pointer, len (Go's string)",
        ),
        (
            "HashMap<i64, i64>",
            size_of::<HashMap<i64, i64>>(),
            "a struct, not a pointer: table bookkeeping + hasher keys inline",
        ),
        (
            "Sender<i64>",
            size_of::<Sender<i64>>(),
            "a handle to a shared channel (its layout is std's business)",
        ),
        (
            "fn()",
            size_of::<fn()>(),
            "1 word: a plain function pointer, no closure data",
        ),
        (
            "closure, captures nothing",
            size_of_var(&no_capture),
            "zero-sized: a closure is a struct of its captures",
        ),
        (
            "closure, captures an i64",
            size_of_var(&captures_i64),
            "8 bytes of capture, stored inline -- no pointer",
        ),
        (
            "Box<dyn Fn()>",
            size_of::<Box<dyn Fn()>>(),
            "2 words: data pointer + vtable pointer",
        ),
        (
            "Box<dyn Any>",
            size_of::<Box<dyn Any>>(),
            "2 words: data pointer + vtable pointer",
        ),
        (
            "Option<Box<Node>>",
            size_of::<Option<Box<Node>>>(),
            "1 word: None is the null pointer, no extra tag",
        ),
        ("Node {i64, Option<Box>}", size_of::<Node>(), "8 + 8"),
        (
            "Option<i64>",
            size_of::<Option<i64>>(),
            "an i64 has no spare value to mean None, so a tag is added",
        ),
    ];
    for (name, size, note) in rows {
        println!("  {name:<26} {size:>4} bytes   {note}");
    }

    println!();
    println!("the Vec header is THREE WORDS, whatever the Vec holds:");
    let a = vec![0i64; 10];
    let b = vec![0i64; 10_000_000];
    println!(
        "  len 10:       header {} bytes, len {}, capacity {}",
        size_of_var(&a),
        a.len(),
        a.capacity()
    );
    println!(
        "  len 10000000: header {} bytes, len {}, capacity {}",
        size_of_var(&b),
        b.len(),
        b.capacity()
    );
    println!(
        "  and a borrowed view of either, &b[..]: {} bytes",
        size_of_var(&&b[..])
    );

    println!();
    println!("which is why passing a view is O(1) and passing an array is O(n):");
    println!(
        "  fn f(xs: &[i64])      copies {} bytes",
        size_of::<&[i64]>()
    );
    println!(
        "  fn f(xs: Vec<i64>)    moves {} bytes (ownership moves; the heap data stays put)",
        size_of::<Vec<i64>>()
    );
    println!(
        "  fn f(xs: [i64; 1000]) copies {} bytes",
        size_of::<[i64; 1000]>()
    );

    println!();
    println!("an ARRAY is a value. A Vec OWNS a heap buffer. A &[T] BORROWS a view.");
    println!("Go's 3-word slice header is two of those jobs in one; Rust splits them,");
    println!("and the borrow checker keeps track of which you are holding.");
}
```

**Rust sample output:**

```
what a VARIABLE of each type costs (the header, not the data):

  [i64; 10]                    80 bytes   an array IS its data -- 10 * 8
  [i64; 1_000_000]           8000000 bytes   still the data. Passing this by value copies 8 MB
  Vec<i64>                     24 bytes   3 words: pointer, capacity, len -- the data lives on the heap
  &[i64]                       16 bytes   2 words: pointer, len -- a borrowed view, so no capacity
  &[i64; 10]                    8 bytes   1 word: the length is part of the type
  String                       24 bytes   3 words: it is a Vec<u8> that promises UTF-8
  &str                         16 bytes   2 words: pointer, len (Go's string)
  HashMap<i64, i64>            48 bytes   a struct, not a pointer: table bookkeeping + hasher keys inline
  Sender<i64>                  16 bytes   a handle to a shared channel (its layout is std's business)
  fn()                          8 bytes   1 word: a plain function pointer, no closure data
  closure, captures nothing     0 bytes   zero-sized: a closure is a struct of its captures
  closure, captures an i64      8 bytes   8 bytes of capture, stored inline -- no pointer
  Box<dyn Fn()>                16 bytes   2 words: data pointer + vtable pointer
  Box<dyn Any>                 16 bytes   2 words: data pointer + vtable pointer
  Option<Box<Node>>             8 bytes   1 word: None is the null pointer, no extra tag
  Node {i64, Option<Box>}      16 bytes   8 + 8
  Option<i64>                  16 bytes   an i64 has no spare value to mean None, so a tag is added

the Vec header is THREE WORDS, whatever the Vec holds:
  len 10:       header 24 bytes, len 10, capacity 10
  len 10000000: header 24 bytes, len 10000000, capacity 10000000
  and a borrowed view of either, &b[..]: 16 bytes

which is why passing a view is O(1) and passing an array is O(n):
  fn f(xs: &[i64])      copies 16 bytes
  fn f(xs: Vec<i64>)    moves 24 bytes (ownership moves; the heap data stays put)
  fn f(xs: [i64; 1000]) copies 8000 bytes

an ARRAY is a value. A Vec OWNS a heap buffer. A &[T] BORROWS a view.
Go's 3-word slice header is two of those jobs in one; Rust splits them,
and the borrow checker keeps track of which you are holding.
```

> **Go vs Rust:** Go has no counterpart for what Rust does here: Go's one 3-word slice header (24 B) is two types in Rust, `Vec<T>` that owns (24 B) and `&[T]` that borrows (16 B, no capacity). The Go table's `map`, `chan` and `func` are 1-word pointers to runtime objects; in Rust `HashMap` is a 48-byte struct held inline, a non-capturing closure is 0 bytes, and the nilable pointer is `Option<Box<T>>` at 8 B (while `Option<i64>` costs 16). Sizes of std types (`HashMap`, `Sender`, even `Vec`) are implementation details rather than guarantees, hence "sample".

**Complexity:** passing a slice is Θ(1) · passing an array is Θ(n) · the whole lesson is downstream of that distinction

---

## 2. Aliasing, and when append betrays you

`🟢 easy` · *Views into shared memory*

A slice is a view, so writes go through it to the shared array. Whether `append` also writes through
depends on **spare capacity** — which makes it the one Go operation whose aliasing behaviour changes
with runtime state. Case 4 below is the trap.

**Steps:**

1. Copy an array, then "copy" a slice, and compare.
2. Write through a subslice and watch the parent change.
3. Append to a view *with* spare capacity, then to one capped with `xs[a:b:c]`.

```go
package main

import "fmt"

func main() {
	fmt.Println("1. an array is a VALUE: assigning copies it")
	arrA := [5]int{1, 2, 3, 4, 5}
	arrB := arrA
	arrB[0] = 99
	fmt.Printf("   arrA = %v   arrB = %v   <- independent\n", arrA, arrB)

	fmt.Println()
	fmt.Println("2. a slice is a VIEW: assigning copies only the header")
	slA := []int{1, 2, 3, 4, 5}
	slB := slA
	slB[0] = 99
	fmt.Printf("   slA  = %v   slB  = %v   <- same backing array\n", slA, slB)

	fmt.Println()
	fmt.Println("3. a subslice is a view too -- writes go straight through")
	xs := []int{1, 2, 3, 4, 5}
	sub := xs[1:3]
	sub[0] = 99
	fmt.Printf("   xs   = %v   sub  = %v   len=%d cap=%d\n", xs, sub, len(sub), cap(sub))
	fmt.Println("   note cap(sub) = 4, not 2: the view can SEE to the end of the array.")

	fmt.Println()
	fmt.Println("4. the trap: whether append aliases depends on spare capacity")

	// Case A: spare capacity -> append writes into the shared array.
	base := []int{1, 2, 3, 4, 5}
	view := base[:2] // len 2, cap 5 -- room to spare
	fmt.Printf("   before: base = %v, view = %v (len=%d cap=%d)\n", base, view, len(view), cap(view))
	view = append(view, 99)
	fmt.Printf("   after append(view, 99): base = %v  <- base[2] was OVERWRITTEN\n", base)

	// Case B: no spare capacity -> append allocates, and the link is broken.
	base2 := []int{1, 2, 3, 4, 5}
	view2 := base2[:2:2] // the 3-index form caps it: len 2, cap 2
	fmt.Printf("\n   before: base2 = %v, view2 = %v (len=%d cap=%d)\n", base2, view2, len(view2), cap(view2))
	view2 = append(view2, 99)
	fmt.Printf("   after append(view2, 99): base2 = %v  <- untouched, append reallocated\n", base2)

	fmt.Println()
	fmt.Println("   xs[a:b:c] -- the three-index slice -- sets cap to c-a. Use it when you")
	fmt.Println("   hand a view to code you do not control: it forces append to copy.")

	fmt.Println()
	fmt.Println("5. copy() is how you actually detach")
	src := []int{1, 2, 3}
	dst := make([]int, len(src))
	copy(dst, src)
	dst[0] = 99
	fmt.Printf("   src = %v   dst = %v   <- independent (slices.Clone does this for you)\n", src, dst)

	fmt.Println()
	fmt.Println("aliasing is not a bug, it is the point: a subslice is free. But every")
	fmt.Println("in-place algorithm in this plan mutates through a view, so know which")
	fmt.Println("view you are holding.")
}
```

**Output:**

```
1. an array is a VALUE: assigning copies it
   arrA = [1 2 3 4 5]   arrB = [99 2 3 4 5]   <- independent

2. a slice is a VIEW: assigning copies only the header
   slA  = [99 2 3 4 5]   slB  = [99 2 3 4 5]   <- same backing array

3. a subslice is a view too -- writes go straight through
   xs   = [1 99 3 4 5]   sub  = [99 3]   len=2 cap=4
   note cap(sub) = 4, not 2: the view can SEE to the end of the array.

4. the trap: whether append aliases depends on spare capacity
   before: base = [1 2 3 4 5], view = [1 2] (len=2 cap=5)
   after append(view, 99): base = [1 2 99 4 5]  <- base[2] was OVERWRITTEN

   before: base2 = [1 2 3 4 5], view2 = [1 2] (len=2 cap=2)
   after append(view2, 99): base2 = [1 2 3 4 5]  <- untouched, append reallocated

   xs[a:b:c] -- the three-index slice -- sets cap to c-a. Use it when you
   hand a view to code you do not control: it forces append to copy.

5. copy() is how you actually detach
   src = [1 2 3]   dst = [99 2 3]   <- independent (slices.Clone does this for you)

aliasing is not a bug, it is the point: a subslice is free. But every
in-place algorithm in this plan mutates through a view, so know which
view you are holding.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Rust has no runtime to ask "did that allocate?", so we count. A global
// allocator is an `unsafe` trait by definition; this one only counts events
// and forwards every call to the system allocator.
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

/// How many allocation events (alloc + realloc) happened while running `f`.
fn allocs_of(f: impl FnOnce()) -> usize {
    let before = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - before
}

fn main() {
    println!("1. an array is a VALUE: assigning copies it");
    let arr_a = [1, 2, 3, 4, 5];
    let mut arr_b = arr_a; // [i32; 5] is Copy, so arr_a is still usable
    arr_b[0] = 99;
    println!("   arr_a = {arr_a:?}   arr_b = {arr_b:?}   <- independent");

    println!();
    println!("2. a slice is a VIEW: copying a &[T] copies only the header (2 words)");
    let mut vec_a = vec![1, 2, 3, 4, 5];
    let view_a: &[i32] = &vec_a;
    let view_b = view_a; // &[T] is Copy: two readers of the same data
    println!(
        "   view_a = {view_a:?}   view_b = {view_b:?}   same data: {}",
        view_a.as_ptr() == view_b.as_ptr()
    );
    // ...but a shared view is read-only:
    //     view_b[0] = 99;
    //     error[E0594]: cannot assign to `view_b[_]`, which is behind a `&` reference
    // Writing needs a `&mut` view, and there can be only ONE of those at a time:
    //     let a = &mut vec_a[..];
    //     let b = &mut vec_a[..];
    //     error[E0499]: cannot borrow `vec_a` as mutable more than once at a time
    let writer = &mut vec_a[..];
    writer[0] = 99;
    println!("   after writer[0] = 99: vec_a = {vec_a:?}   <- the write went through the view");
    // Assigning a Vec (not a view) does not copy the header: it MOVES it.
    //     let vec_b = vec_a;  println!("{vec_a:?}");
    //     error[E0382]: borrow of moved value: `vec_a`
    // Go's `slB := slA` (two live handles, both writable) has no safe Rust spelling.

    println!();
    println!("3. a subslice is a view too -- writes go straight through");
    let mut xs = vec![1, 2, 3, 4, 5];
    let sub = &mut xs[1..3];
    sub[0] = 99;
    println!("   sub = {sub:?}   len={}", sub.len());
    // `sub` is dead after that line (non-lexical lifetimes), so xs is usable again.
    // Reading xs *while sub is still live* is refused:
    //     println!("{xs:?}"); println!("{sub:?}");
    //     error[E0502]: cannot borrow `xs` as immutable because it is also borrowed as mutable
    println!("   xs  = {xs:?}");
    println!("   note there is no cap: a slice has no capacity, it sees exactly its 2 elements.");

    println!();
    println!("4. the trap that is not there: growing a Vec needs &mut, so no view can be live");

    // Case A: spare capacity -> push writes into the buffer in place.
    let mut base = vec![1, 2, 3, 4, 5];
    base.truncate(2); // len 2, capacity still 5. The 3, 4, 5 are DROPPED, not hidden.
    println!(
        "   before: base = {base:?} (len={} cap={})",
        base.len(),
        base.capacity()
    );
    let buf = base.as_ptr();
    let events = allocs_of(|| base.push(99));
    println!(
        "   after push(99): base = {base:?}  <- {events} allocation events, same buffer: {}",
        buf == base.as_ptr()
    );

    // Case B: no spare capacity -> push reallocates.
    let mut full = vec![1, 2];
    println!(
        "\n   before: full = {full:?} (len={} cap={})",
        full.len(),
        full.capacity()
    );
    let events = allocs_of(|| full.push(99));
    println!(
        "   after push(99): full = {full:?}  <- {events} allocation event, cap now {}",
        full.capacity()
    );

    println!();
    println!("   In Go the two cases behave differently for everyone holding a view. In Rust");
    println!("   the compiler rejects BOTH, because it never asks how much capacity is spare:");
    println!("       let view = &base[..2];");
    println!("       base.push(99);");
    println!(
        "       error[E0502]: cannot borrow `base` as mutable because it is also borrowed as immutable"
    );
    println!("   A view cannot outlive the buffer it points into, so a reallocation cannot");
    println!("   leave one dangling, and push-through-an-alias cannot happen.");

    println!();
    println!("   there is no xs[a:b:c] either: a &mut [T] has no push() at all.");
    //     let view = &mut xs[..2];  view.push(99);
    //     error[E0599]: no method named `push` found for mutable reference `&mut [{integer}]`
    println!("   to hand out several writable views safely, split the borrow:");
    let mut halves = [1, 2, 3, 4, 5, 6];
    let (left, right) = halves.split_at_mut(3); // two DISJOINT &mut views
    left[0] = 10;
    right[0] = 40;
    println!("   left = {left:?}   right = {right:?}");
    println!("   halves = {halves:?}");

    println!();
    println!("5. copy_from_slice() is how you copy into storage you already have");
    let src = [1, 2, 3];
    let mut dst = vec![0; src.len()];
    dst.copy_from_slice(&src);
    dst[0] = 99;
    println!("   src = {src:?}   dst = {dst:?}   <- independent (src.to_vec() does both steps)");

    println!();
    println!("aliasing is not a bug in Rust, it is a rule: any number of readers OR one");
    println!("writer. A subslice is still free, and the compiler tells you which view you");
    println!("are holding.");
}
```

**Rust output:**

```
1. an array is a VALUE: assigning copies it
   arr_a = [1, 2, 3, 4, 5]   arr_b = [99, 2, 3, 4, 5]   <- independent

2. a slice is a VIEW: copying a &[T] copies only the header (2 words)
   view_a = [1, 2, 3, 4, 5]   view_b = [1, 2, 3, 4, 5]   same data: true
   after writer[0] = 99: vec_a = [99, 2, 3, 4, 5]   <- the write went through the view

3. a subslice is a view too -- writes go straight through
   sub = [99, 3]   len=2
   xs  = [1, 99, 3, 4, 5]
   note there is no cap: a slice has no capacity, it sees exactly its 2 elements.

4. the trap that is not there: growing a Vec needs &mut, so no view can be live
   before: base = [1, 2] (len=2 cap=5)
   after push(99): base = [1, 2, 99]  <- 0 allocation events, same buffer: true

   before: full = [1, 2] (len=2 cap=2)
   after push(99): full = [1, 2, 99]  <- 1 allocation event, cap now 4

   In Go the two cases behave differently for everyone holding a view. In Rust
   the compiler rejects BOTH, because it never asks how much capacity is spare:
       let view = &base[..2];
       base.push(99);
       error[E0502]: cannot borrow `base` as mutable because it is also borrowed as immutable
   A view cannot outlive the buffer it points into, so a reallocation cannot
   leave one dangling, and push-through-an-alias cannot happen.

   there is no xs[a:b:c] either: a &mut [T] has no push() at all.
   to hand out several writable views safely, split the borrow:
   left = [10, 2, 3]   right = [40, 5, 6]
   halves = [10, 2, 3, 40, 5, 6]

5. copy_from_slice() is how you copy into storage you already have
   src = [1, 2, 3]   dst = [99, 2, 3]   <- independent (src.to_vec() does both steps)

aliasing is not a bug in Rust, it is a rule: any number of readers OR one
writer. A subslice is still free, and the compiler tells you which view you
are holding.
```

> **Go vs Rust:** Go's trap does not exist in Rust, and neither does its machinery: there is no `cap()` on a slice, no `xs[a:b:c]`, and no `append` through a view. Growing is `Vec::push`, which needs `&mut Vec`, so no view can be alive across it; the compiler rejects that program identically whether capacity is spare (len 2, cap 5) or not (len 2, cap 2), and a reallocation cannot leave a view dangling. Case A really does write in place (0 allocation events, same buffer) and case B really does reallocate (1 event, cap 2 to 4; the 4 is `Vec`'s growth policy, not a guarantee); the difference is that nobody can be watching. Lines differ from the Go output only where Rust differs, plus `{:?}` printing `[1, 2, 3]` with commas.

**Complexity:** subslicing is Θ(1) and allocates nothing — that is why it is everywhere in this plan, and why you must know which view you hold

---

## 3. Ten bytes that pin eight megabytes

`🟢 easy` · *The retained backing array*

A slice keeps its **entire** backing array reachable, not just the part you can see. Return a small
header sliced out of a large buffer and you have leaked the buffer — and the GC is powerless, because
the memory genuinely *is* reachable.

**Steps:**

1. Slice 10 bytes out of an 8 MB buffer and hold only the small slice.
2. Measure the heap. Note `cap` is still 8388608.
3. Do it again with `slices.Clone` and compare.

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
)

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

// firstTen returns a view of the first 10 bytes. The 8 MB array behind it stays
// reachable through the slice's pointer -- so it is never collected.
func firstTen(data []byte) []byte {
	return data[:10]
}

// firstTenCopied returns an independent 10-byte slice. The 8 MB array becomes
// unreachable the moment the caller drops it.
func firstTenCopied(data []byte) []byte {
	return slices.Clone(data[:10])
}

func main() {
	const size = 8 << 20 // 8 MB

	base := heapMB()
	fmt.Printf("baseline heap: %.1f MB\n\n", base)

	// --- the leak ---
	leaked := firstTen(make([]byte, size))
	afterLeak := heapMB()
	fmt.Printf("holding a %d-byte slice from firstTen():        heap %.1f MB  (+%.1f)\n",
		len(leaked), afterLeak, afterLeak-base)
	fmt.Printf("  len=%d but cap=%d -- the header still points at the whole array\n",
		len(leaked), cap(leaked))
	runtime.KeepAlive(leaked)

	// Drop it and confirm the memory comes back.
	leaked = nil
	afterDrop := heapMB()
	fmt.Printf("after dropping that 10-byte slice:               heap %.1f MB  (+%.1f)\n\n",
		afterDrop, afterDrop-base)

	// --- the fix ---
	kept := firstTenCopied(make([]byte, size))
	afterCopy := heapMB()
	fmt.Printf("holding a %d-byte slice from firstTenCopied():  heap %.1f MB  (+%.1f)\n",
		len(kept), afterCopy, afterCopy-base)
	fmt.Printf("  len=%d cap=%d -- nothing else is reachable\n", len(kept), cap(kept))
	runtime.KeepAlive(kept)

	fmt.Println()
	fmt.Println("a slice keeps its ENTIRE backing array alive, not just the part you can see.")
	fmt.Println("10 bytes pinned 8 MB. The GC cannot help: the memory is genuinely reachable.")
	fmt.Println()
	fmt.Println("where this bites in practice:")
	fmt.Println("  - returning a small header/token sliced out of a large request buffer")
	fmt.Println("  - a queue implemented as q = q[1:] (the head is never freed -> lesson 09)")
	fmt.Println("  - caching a field parsed out of a big file you have otherwise finished with")
	fmt.Println()
	fmt.Println("the fix is always the same: slices.Clone (or copy into a right-sized slice)")
	fmt.Println("when a small result must outlive a large input.")
}
```

**Sample output** (heap figures move slightly run to run):

```
baseline heap: 0.2 MB

holding a 10-byte slice from firstTen():        heap 8.2 MB  (+8.0)
  len=10 but cap=8388608 -- the header still points at the whole array
after dropping that 10-byte slice:               heap 0.2 MB  (+0.0)

holding a 10-byte slice from firstTenCopied():  heap 0.2 MB  (+0.0)
  len=10 cap=16 -- nothing else is reachable

a slice keeps its ENTIRE backing array alive, not just the part you can see.
10 bytes pinned 8 MB. The GC cannot help: the memory is genuinely reachable.

where this bites in practice:
  - returning a small header/token sliced out of a large request buffer
  - a queue implemented as q = q[1:] (the head is never freed -> lesson 09)
  - caching a field parsed out of a big file you have otherwise finished with

the fix is always the same: slices.Clone (or copy into a right-sized slice)
when a small result must outlive a large input.
```

The `cap=8388608` on a `len=10` slice is the tell. Any time a small value outlives a large buffer,
clone it.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Rust has no GC and no runtime.MemStats, so we ask the allocator instead: this
// counts the bytes currently live on the heap. A global allocator is an `unsafe`
// trait by definition; this one only does arithmetic and forwards to System.
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

// No runtime.GC() first: nothing is "waiting to be collected". Freed means freed.
fn heap_mb() -> f64 {
    LIVE.load(Relaxed) as f64 / (1 << 20) as f64
}

// first_ten returns a view of the first 10 bytes. The signature says it all: the
// returned slice BORROWS from `data`, so the 8 MB buffer must outlive it. The
// pinning is not hidden -- the compiler enforces it.
fn first_ten(data: &[u8]) -> &[u8] {
    &data[..10]
}

// first_ten_truncated is the Rust spelling of Go's leak. It takes ownership of
// the Vec, drops everything past 10 elements -- and keeps the 8 MB allocation,
// because truncate never shrinks capacity.
fn first_ten_truncated(mut data: Vec<u8>) -> Vec<u8> {
    data.truncate(10);
    data
}

// first_ten_copied returns an independent 10-byte Vec.
fn first_ten_copied(data: &[u8]) -> Vec<u8> {
    data[..10].to_vec()
}

// first_ten_shrunk keeps the original Vec but gives the spare capacity back.
fn first_ten_shrunk(mut data: Vec<u8>) -> Vec<u8> {
    data.truncate(10);
    data.shrink_to_fit();
    data
}

fn main() {
    const SIZE: usize = 8 << 20; // 8 MB

    println!("heap measured with a counting allocator (there is no GC to ask)");
    let base = heap_mb(); // taken after the first print, so stdout's buffer is in the baseline
    println!("baseline heap: {base:.1} MB");
    println!();

    // --- the view: the owner is pinned, in plain sight ---
    {
        let buf = vec![0u8; SIZE];
        let view = first_ten(&buf);
        let held = heap_mb();
        println!(
            "holding a {}-byte view from first_ten(&buf):     heap {held:.1} MB  (+{:.1})",
            view.len(),
            held - base
        );
        println!(
            "  len={} -- it borrows buf, so buf cannot be dropped while the view lives",
            view.len()
        );
        //     drop(buf);  println!("{}", view.len());
        //     error[E0505]: cannot move out of `buf` because it is borrowed
        // Nor can the view outlive the buffer it came from:
        //     let leaked = first_ten(&vec![0u8; SIZE]);
        //     error[E0716]: temporary value dropped while borrowed
    }
    let after_scope = heap_mb();
    println!(
        "after buf and the view go out of scope:           heap {after_scope:.1} MB  (+{:.1})\n",
        after_scope - base
    );

    // --- the real Rust leak: a Vec that is short but still owns the big allocation ---
    let leaked = first_ten_truncated(vec![0u8; SIZE]);
    let after_leak = heap_mb();
    println!(
        "holding a {}-byte Vec from first_ten_truncated(): heap {after_leak:.1} MB  (+{:.1})",
        leaked.len(),
        after_leak - base
    );
    println!(
        "  len={} but capacity={} -- the Vec still owns the whole allocation",
        leaked.len(),
        leaked.capacity()
    );
    drop(leaked);
    let after_drop = heap_mb();
    println!(
        "after dropping that 10-byte Vec:                  heap {after_drop:.1} MB  (+{:.1})\n",
        after_drop - base
    );

    // --- the fixes ---
    let kept = {
        let big = vec![0u8; SIZE];
        first_ten_copied(&big)
    }; // `big` is freed right here, at the closing brace
    let after_copy = heap_mb();
    println!(
        "holding a {}-byte Vec from first_ten_copied():    heap {after_copy:.1} MB  (+{:.1})",
        kept.len(),
        after_copy - base
    );
    println!(
        "  len={} capacity={} -- nothing else is alive",
        kept.len(),
        kept.capacity()
    );
    drop(kept);
    println!();

    let shrunk = first_ten_shrunk(vec![0u8; SIZE]);
    let after_shrink = heap_mb();
    println!(
        "holding a {}-byte Vec from first_ten_shrunk():    heap {after_shrink:.1} MB  (+{:.1})",
        shrunk.len(),
        after_shrink - base
    );
    println!(
        "  len={} capacity={} -- shrink_to_fit handed the spare capacity back",
        shrunk.len(),
        shrunk.capacity()
    );
    drop(shrunk);

    println!();
    println!("a Vec keeps its ENTIRE allocation until it is dropped or shrunk, not just");
    println!("the part you can see. 10 bytes of len pinned 8 MB of capacity. A view cannot");
    println!("hide this: its owner must stay named and alive, and the compiler enforces it.");
    println!();
    println!("where the Vec version bites in practice:");
    println!("  - truncate / clear / drain / retain all leave capacity behind");
    println!("  - a long-lived Vec that once held a million items and now holds ten");
    println!("  - returning a small result truncated out of a large input buffer");
    println!();
    println!("the fix is always the same: shrink_to_fit, or copy into a right-sized Vec");
    println!("(to_vec / clone) when a small result must outlive a large input.");
}
```

**Rust output:**

```
heap measured with a counting allocator (there is no GC to ask)
baseline heap: 0.0 MB

holding a 10-byte view from first_ten(&buf):     heap 8.0 MB  (+8.0)
  len=10 -- it borrows buf, so buf cannot be dropped while the view lives
after buf and the view go out of scope:           heap 0.0 MB  (+0.0)

holding a 10-byte Vec from first_ten_truncated(): heap 8.0 MB  (+8.0)
  len=10 but capacity=8388608 -- the Vec still owns the whole allocation
after dropping that 10-byte Vec:                  heap 0.0 MB  (+0.0)

holding a 10-byte Vec from first_ten_copied():    heap 0.0 MB  (+0.0)
  len=10 capacity=10 -- nothing else is alive

holding a 10-byte Vec from first_ten_shrunk():    heap 0.0 MB  (+0.0)
  len=10 capacity=10 -- shrink_to_fit handed the spare capacity back

a Vec keeps its ENTIRE allocation until it is dropped or shrunk, not just
the part you can see. 10 bytes of len pinned 8 MB of capacity. A view cannot
hide this: its owner must stay named and alive, and the compiler enforces it.

where the Vec version bites in practice:
  - truncate / clear / drain / retain all leave capacity behind
  - a long-lived Vec that once held a million items and now holds ten
  - returning a small result truncated out of a large input buffer

the fix is always the same: shrink_to_fit, or copy into a right-sized Vec
(to_vec / clone) when a small result must outlive a large input.
```

> **Go vs Rust:** Go's retention bug splits in two in Rust. A `&[u8]` view cannot hide its buffer: it borrows, so the owner has to be named and alive (dropping it, or returning a view of a temporary, is a compile error). The real Rust twin of `len=10 cap=8388608` is a `Vec` truncated but not shrunk: 8.0 MB held for 10 bytes, fixed by `shrink_to_fit` or `to_vec`. There is no GC to be "powerless": the heap column comes from a counting allocator, freed memory is freed at the closing brace or `drop`, and the numbers are exact (they were identical on repeat runs) rather than `MemStats` readings.

**Complexity:** both versions are Θ(1) time · one is Θ(1) space and the other is Θ(size of the original buffer)

---

## 4. Struct padding, and the field order that fixes it

`🟢 easy` · *Alignment*

Go never reorders your fields. Every field must land on an offset divisible by its alignment, so the
compiler inserts padding — and the same three fields cost **24 bytes or 16** depending purely on the
order you wrote them in.

**Steps:**

1. Declare the same fields in a bad order and a good one.
2. Print `Sizeof`, `Alignof` and every `Offsetof`.
3. Scale it to 10M elements, then find a struct where reordering cannot help.

```go
package main

import (
	"fmt"
	"unsafe"
)

// Every field must sit at an offset divisible by its alignment. The compiler
// inserts PADDING to make that true -- and never reorders your fields.

type Bad struct {
	flag  bool  // 1 byte, then 7 bytes of padding
	count int64 // 8
	ok    bool  // 1 byte, then 7 more of padding to round the struct up
}

type Good struct {
	count int64 // 8
	flag  bool  // 1
	ok    bool  // 1, then 6 of tail padding
}

// Edge is already ordered well, and STILL wastes 3 bytes: 13 bytes of payload
// cannot round up to a multiple of its 4-byte alignment. Some padding is
// unavoidable -- the only way past it is a different layout entirely (ex 15).
type Edge struct {
	From   int32
	To     int32
	Weight float32
	Active bool
}

func report[T any](name string, v T, fields ...string) {
	size := unsafe.Sizeof(v)
	fmt.Printf("  %-12s %3d bytes (align %d)\n", name, size, unsafe.Alignof(v))
	for _, f := range fields {
		fmt.Printf("      %s\n", f)
	}
}

func main() {
	fmt.Println("Go never reorders struct fields. You do.")
	fmt.Println()

	var bad Bad
	report("Bad", bad,
		fmt.Sprintf("flag  at offset %d (1 byte + 7 padding)", unsafe.Offsetof(bad.flag)),
		fmt.Sprintf("count at offset %d", unsafe.Offsetof(bad.count)),
		fmt.Sprintf("ok    at offset %d (1 byte + 7 padding)", unsafe.Offsetof(bad.ok)),
	)

	var good Good
	report("Good", good,
		fmt.Sprintf("count at offset %d", unsafe.Offsetof(good.count)),
		fmt.Sprintf("flag  at offset %d", unsafe.Offsetof(good.flag)),
		fmt.Sprintf("ok    at offset %d (then 6 bytes of tail padding)", unsafe.Offsetof(good.ok)),
	)

	fmt.Println()
	fmt.Printf("same three fields, %d bytes vs %d bytes -- a %.0f%% saving from field order alone.\n",
		unsafe.Sizeof(bad), unsafe.Sizeof(good),
		100*(1-float64(unsafe.Sizeof(good))/float64(unsafe.Sizeof(bad))))

	fmt.Println()
	fmt.Println("the rule: order fields LARGEST ALIGNMENT FIRST (8-byte, then 4, then 2, then 1).")

	fmt.Println()
	fmt.Println("why it matters for data structures -- scale it to a real collection:")
	const n = 10_000_000
	badMB := float64(unsafe.Sizeof(bad)) * n / (1 << 20)
	goodMB := float64(unsafe.Sizeof(good)) * n / (1 << 20)
	fmt.Printf("  %d elements of Bad:  %.0f MB\n", n, badMB)
	fmt.Printf("  %d elements of Good: %.0f MB   (%.0f MB saved, no code changed)\n",
		n, goodMB, badMB-goodMB)

	fmt.Println()
	fmt.Println("and sometimes padding is unavoidable:")
	var e Edge
	fmt.Printf("  Edge (int32,int32,float32,bool) = %d bytes for %d bytes of payload\n",
		unsafe.Sizeof(e), 4+4+4+1)
	fmt.Printf("  offsets: From=%d To=%d Weight=%d Active=%d, then %d bytes of tail padding\n",
		unsafe.Offsetof(e.From), unsafe.Offsetof(e.To),
		unsafe.Offsetof(e.Weight), unsafe.Offsetof(e.Active),
		unsafe.Sizeof(e)-(4+4+4+1))
	fmt.Println("  no reordering fixes that -- 13 bytes cannot round to a multiple of 4.")
	fmt.Println("  the way out is to stop storing structs at all (example 15: struct-of-arrays).")

	fmt.Println()
	fmt.Println("check any struct with: unsafe.Sizeof / Alignof / Offsetof,")
	fmt.Println("or `go vet -fieldalignment` (in the fieldalignment analyzer).")
}
```

**Output:**

```
Go never reorders struct fields. You do.

  Bad           24 bytes (align 8)
      flag  at offset 0 (1 byte + 7 padding)
      count at offset 8
      ok    at offset 16 (1 byte + 7 padding)
  Good          16 bytes (align 8)
      count at offset 0
      flag  at offset 8
      ok    at offset 9 (then 6 bytes of tail padding)

same three fields, 24 bytes vs 16 bytes -- a 33% saving from field order alone.

the rule: order fields LARGEST ALIGNMENT FIRST (8-byte, then 4, then 2, then 1).

why it matters for data structures -- scale it to a real collection:
  10000000 elements of Bad:  229 MB
  10000000 elements of Good: 153 MB   (76 MB saved, no code changed)

and sometimes padding is unavoidable:
  Edge (int32,int32,float32,bool) = 16 bytes for 13 bytes of payload
  offsets: From=0 To=4 Weight=8 Active=12, then 3 bytes of tail padding
  no reordering fixes that -- 13 bytes cannot round to a multiple of 4.
  the way out is to stop storing structs at all (example 15: struct-of-arrays).

check any struct with: unsafe.Sizeof / Alignof / Offsetof,
or `go vet -fieldalignment` (in the fieldalignment analyzer).
```

**Rust:**

```rust
use std::mem::{align_of, offset_of, size_of};

// Every field must sit at an offset divisible by its alignment, so padding
// appears. What differs from Go is who decides the order: by default (repr(Rust))
// the COMPILER reorders fields; #[repr(C)] pins them in declaration order, which
// is exactly Go's behaviour. The fields below are only measured, never read,
// hence the targeted allow(dead_code) on each struct.

#[allow(dead_code)]
struct Bad {
    flag: bool, // 1 byte
    count: i64, // 8
    ok: bool,   // 1 byte
}

#[allow(dead_code)]
struct Good {
    count: i64, // 8
    flag: bool, // 1
    ok: bool,   // 1
}

// The same two structs with the order frozen, as Go would lay them out.
#[allow(dead_code)]
#[repr(C)]
struct BadC {
    flag: bool,
    count: i64,
    ok: bool,
}

#[allow(dead_code)]
#[repr(C)]
struct GoodC {
    count: i64,
    flag: bool,
    ok: bool,
}

// Edge is already ordered well, and STILL wastes 3 bytes: 13 bytes of payload
// cannot round up to a multiple of its 4-byte alignment. Neither repr(Rust) nor
// repr(C) can avoid that.
#[allow(dead_code)]
struct Edge {
    from: i32,
    to: i32,
    weight: f32,
    active: bool,
}

#[allow(dead_code)]
#[repr(C)]
struct EdgeC {
    from: i32,
    to: i32,
    weight: f32,
    active: bool,
}

// repr(packed) drops the alignment requirement altogether -- Go has no spelling
// for this. Fields may then be unaligned, so the compiler forbids references to them.
#[allow(dead_code)]
#[repr(C, packed)]
struct EdgePacked {
    from: i32,
    to: i32,
    weight: f32,
    active: bool,
}

// report prints size, alignment, every field's offset (offset_of! is Rust's
// unsafe.Offsetof, and needs no unsafe) and how much of the size is padding.
macro_rules! report {
    ($ty:ty { $($field:ident: $fty:ty),+ }) => {{
        let payload = 0 $(+ size_of::<$fty>())+;
        println!(
            "  {:<8} {:>3} bytes (align {})",
            stringify!($ty),
            size_of::<$ty>(),
            align_of::<$ty>()
        );
        $(
            println!(
                "      {:<6} at offset {:>2} ({} byte{})",
                stringify!($field),
                offset_of!($ty, $field),
                size_of::<$fty>(),
                if size_of::<$fty>() == 1 { "" } else { "s" }
            );
        )+
        println!(
            "      payload {payload} bytes, padding {} bytes",
            size_of::<$ty>() - payload
        );
    }};
}

fn mb(bytes: usize) -> f64 {
    bytes as f64 / (1 << 20) as f64
}

fn main() {
    println!("Go never reorders struct fields. Rust does -- unless you say repr(C).");
    println!();

    println!("default layout, fields declared in the order shown:");
    report!(Bad {
        flag: bool,
        count: i64,
        ok: bool
    });
    report!(Good {
        count: i64,
        flag: bool,
        ok: bool
    });

    println!();
    println!("#[repr(C)] -- declaration order is kept, as in Go:");
    report!(BadC {
        flag: bool,
        count: i64,
        ok: bool
    });
    report!(GoodC {
        count: i64,
        flag: bool,
        ok: bool
    });

    println!();
    let (bad, good) = (size_of::<BadC>(), size_of::<GoodC>());
    println!(
        "same three fields under repr(C), {bad} bytes vs {good} bytes -- a {:.0}% saving from field order alone.",
        100.0 * (1.0 - good as f64 / bad as f64)
    );
    println!(
        "under the default layout Bad is {} bytes and Good is {}: here rustc already did the reordering.",
        size_of::<Bad>(),
        size_of::<Good>()
    );

    println!();
    println!("the rule under repr(C): order fields LARGEST ALIGNMENT FIRST (8, then 4, 2, 1).");
    println!(
        "the default layout is unspecified, so rustc is free to reorder -- and above, it did."
    );

    println!();
    println!("why it matters for data structures -- scale it to a real collection:");
    const N: usize = 10_000_000;
    println!(
        "  {N} elements of BadC: {:.0} MB   (the same fields in Go's order)",
        mb(size_of::<BadC>() * N)
    );
    println!(
        "  {N} elements of Bad:  {:.0} MB   (default layout, order as written)",
        mb(size_of::<Bad>() * N)
    );
    println!(
        "  {N} elements of Good: {:.0} MB   ({:.0} MB saved vs BadC, no code changed)",
        mb(size_of::<Good>() * N),
        mb(size_of::<BadC>() * N) - mb(size_of::<Good>() * N)
    );

    println!();
    println!("and sometimes padding is unavoidable:");
    report!(Edge {
        from: i32,
        to: i32,
        weight: f32,
        active: bool
    });
    println!(
        "  repr(C) gives the same {} bytes -- no reordering fixes that: 13 bytes cannot round to a multiple of 4.",
        size_of::<EdgeC>()
    );
    println!(
        "  #[repr(packed)] removes the padding ({} bytes, align {}) by giving up alignment:",
        size_of::<EdgePacked>(),
        align_of::<EdgePacked>()
    );
    println!(
        "  `&edge.weight` on a packed struct is error[E0793]: reference to field of packed struct is unaligned."
    );
    println!(
        "  the other way out is to stop storing structs at all (example 15: struct-of-arrays)."
    );

    println!();
    println!("check any struct with: size_of / align_of / offset_of!,");
    println!("or rustc's nightly -Zprint-type-sizes (not run here). Go needs a fieldalignment");
    println!("linter because it never reorders; here only repr(C) structs need that check.");
}
```

**Rust output:**

```
Go never reorders struct fields. Rust does -- unless you say repr(C).

default layout, fields declared in the order shown:
  Bad       16 bytes (align 8)
      flag   at offset  8 (1 byte)
      count  at offset  0 (8 bytes)
      ok     at offset  9 (1 byte)
      payload 10 bytes, padding 6 bytes
  Good      16 bytes (align 8)
      count  at offset  0 (8 bytes)
      flag   at offset  8 (1 byte)
      ok     at offset  9 (1 byte)
      payload 10 bytes, padding 6 bytes

#[repr(C)] -- declaration order is kept, as in Go:
  BadC      24 bytes (align 8)
      flag   at offset  0 (1 byte)
      count  at offset  8 (8 bytes)
      ok     at offset 16 (1 byte)
      payload 10 bytes, padding 14 bytes
  GoodC     16 bytes (align 8)
      count  at offset  0 (8 bytes)
      flag   at offset  8 (1 byte)
      ok     at offset  9 (1 byte)
      payload 10 bytes, padding 6 bytes

same three fields under repr(C), 24 bytes vs 16 bytes -- a 33% saving from field order alone.
under the default layout Bad is 16 bytes and Good is 16: here rustc already did the reordering.

the rule under repr(C): order fields LARGEST ALIGNMENT FIRST (8, then 4, 2, 1).
the default layout is unspecified, so rustc is free to reorder -- and above, it did.

why it matters for data structures -- scale it to a real collection:
  10000000 elements of BadC: 229 MB   (the same fields in Go's order)
  10000000 elements of Bad:  153 MB   (default layout, order as written)
  10000000 elements of Good: 153 MB   (76 MB saved vs BadC, no code changed)

and sometimes padding is unavoidable:
  Edge      16 bytes (align 4)
      from   at offset  0 (4 bytes)
      to     at offset  4 (4 bytes)
      weight at offset  8 (4 bytes)
      active at offset 12 (1 byte)
      payload 13 bytes, padding 3 bytes
  repr(C) gives the same 16 bytes -- no reordering fixes that: 13 bytes cannot round to a multiple of 4.
  #[repr(packed)] removes the padding (13 bytes, align 1) by giving up alignment:
  `&edge.weight` on a packed struct is error[E0793]: reference to field of packed struct is unaligned.
  the other way out is to stop storing structs at all (example 15: struct-of-arrays).

check any struct with: size_of / align_of / offset_of!,
or rustc's nightly -Zprint-type-sizes (not run here). Go needs a fieldalignment
linter because it never reorders; here only repr(C) structs need that check.
```

> **Go vs Rust:** The Go rule "you order the fields" does not apply by default: Rust's default layout is unspecified and rustc reorders, so `Bad` and `Good` are both 16 bytes (`Bad` puts `flag` at offset 8, after `count`). Go's layout is what `#[repr(C)]` gives: `BadC` 24 bytes, `GoodC` 16, the same 33% and 229 MB vs 153 MB as the Go output. `Edge` is 16 bytes for 13 of payload in both languages and under both `repr`s; Rust additionally offers `#[repr(packed)]` (13 bytes, align 1), which Go cannot express. Go's `unsafe.Sizeof`/`Offsetof` become `size_of` and `offset_of!`, neither of which needs `unsafe`.

**Complexity:** Θ(1) to fix, and it changes the constant on every Θ(n) collection of that struct — 76 MB at 10M elements

---

## 5. What escapes to the heap

`🟢 easy` · *Escape analysis*

The rule is **not** "pointers go on the heap". It is "does this value outlive the frame?" —
`&point{…}` used locally allocates nothing. This example measures nine cases, and three of them are
counter-intuitive enough to be worth memorizing.

**Steps:**

1. Write functions that keep a value local, return it, box it, or size it dynamically.
2. Measure each with `testing.AllocsPerRun` — using **typed sinks**, since an `any` sink would box the results and contaminate every row.
3. Compare against `go build -gcflags='-m'`, and notice where the compiler's wording misleads.

```go
package main

import (
	"fmt"
	"testing"
)

type point struct{ x, y int }

// size is a package-level var so the compiler cannot fold it to a constant.
var size = 16

// Typed sinks. Do NOT use `var sink any` here: assigning to an interface boxes
// the value, which adds an allocation of its own and contaminates every
// measurement below.
var (
	sinkInt   int
	sinkPtr   *point
	sinkIface any
)

// --- these stay on the stack ---

// stackLocal: the struct never leaves the function.
func stackLocal(a, b int) int {
	p := point{x: a, y: b}
	return p.x + p.y
}

// stackPointer: taking an address does NOT force a heap allocation. What matters
// is whether the pointer OUTLIVES the frame -- here it does not.
func stackPointer(a, b int) int {
	p := &point{x: a, y: b}
	return p.x + p.y
}

// smallFixedSlice: the size is a compile-time constant and it does not escape.
func smallFixedSlice() int {
	buf := make([]int, 16)
	for i := range buf {
		buf[i] = i
	}
	return buf[15]
}

// --- these escape to the heap ---

// escapesByReturn: the caller keeps the pointer, so it must outlive the frame.
func escapesByReturn(a, b int) *point {
	return &point{x: a, y: b}
}

// boxSmallInt: values 0..255 come from a shared runtime table, so this does NOT
// allocate. A lovely trap when you are benchmarking interface costs.
func boxSmallInt() any {
	return 42
}

// boxLargeConst: still no allocation. A CONSTANT gets one static instance that
// every interface value can point at. `-gcflags=-m` says "escapes to heap" here,
// which means "cannot live in this frame" -- not "mallocs on every call".
func boxLargeConst() any {
	return 1 << 40
}

// counter starts above 255 so its values are never the cached small ones.
var counter = 1000

// boxDynamicInt: a genuinely runtime value, outside the cache. THIS is what
// interface boxing actually costs.
func boxDynamicInt() any {
	counter++
	return counter
}

// escapesBySize: too big for a stack frame, whatever its lifetime.
func escapesBySize() int {
	buf := make([]int, 100_000)
	return len(buf)
}

// escapesByUnknownSize: the length comes from a variable the compiler cannot
// bound, so it must heap-allocate even though 16 ints would fit in a frame.
func escapesByUnknownSize() int {
	buf := make([]int, size)
	return len(buf)
}

func main() {
	fmt.Println("escape analysis decides stack vs heap. The rule is NOT 'pointers are heap':")
	fmt.Println("it is 'does this value outlive the function?'. Allocations, measured:")
	fmt.Println()

	cases := []struct {
		name string
		fn   func()
		why  string
	}{
		{"stackLocal", func() { sinkInt = stackLocal(1, 2) }, "value never leaves"},
		{"stackPointer", func() { sinkInt = stackPointer(1, 2) }, "&p taken, still does not leave"},
		{"smallFixedSlice", func() { sinkInt = smallFixedSlice() }, "make() with a constant size"},
		{"escapesByReturn", func() { sinkPtr = escapesByReturn(1, 2) }, "caller keeps the pointer"},
		{"boxSmallInt", func() { sinkIface = boxSmallInt() }, "42 comes from the runtime's cache"},
		{"boxLargeConst", func() { sinkIface = boxLargeConst() }, "a constant gets one static copy"},
		{"boxDynamicInt", func() { sinkIface = boxDynamicInt() }, "runtime value > 255: real boxing"},
		{"escapesBySize", func() { sinkInt = escapesBySize() }, "100000 ints is too big for a frame"},
		{"escapesByUnknownSize", func() { sinkInt = escapesByUnknownSize() }, "length is a variable"},
	}

	fmt.Printf("%-22s %10s   %s\n", "function", "allocs/op", "why")
	for _, c := range cases {
		allocs := testing.AllocsPerRun(100, c.fn)
		fmt.Printf("%-22s %10.0f   %s\n", c.name, allocs, c.why)
	}

	fmt.Println()
	fmt.Println("three results worth staring at:")
	fmt.Println("  1. stackPointer allocates NOTHING even though it takes an address.")
	fmt.Println("  2. smallFixedSlice and escapesByUnknownSize both make a 16-element slice;")
	fmt.Println("     only the one with a compile-time CONSTANT length stays on the stack.")
	fmt.Println("  3. boxing only costs when the value is a runtime value outside 0..255.")
	fmt.Println()
	fmt.Println("to see the compiler's own reasoning:")
	fmt.Println("  go build -gcflags='-m' -o /dev/null .")
	fmt.Println()
	fmt.Println("careful reading it, though: for make([]int, size) the compiler prints")
	fmt.Println("\"does not escape\" and the allocation still happens. 'Escapes' means")
	fmt.Println("'outlives the frame'; a variable-length make can fail to escape and still")
	fmt.Println("be heap-allocated because the frame size is not known. Trust AllocsPerRun.")

	fmt.Println()
	fmt.Println("consequences for data structures:")
	fmt.Println("  - a tree/list node returned from a constructor always escapes: 1 alloc per node,")
	fmt.Println("    which is most of why pointer structures lose to slices (examples 9 and 13)")
	fmt.Println("  - storing T in an interface boxes it unless it is a pointer or a cached small int")
	fmt.Println("  - make([]T, n) with a variable n always heap-allocates -- hoist it out of loops")
}
```

**Output:**

```
escape analysis decides stack vs heap. The rule is NOT 'pointers are heap':
it is 'does this value outlive the function?'. Allocations, measured:

function                allocs/op   why
stackLocal                      0   value never leaves
stackPointer                    0   &p taken, still does not leave
smallFixedSlice                 0   make() with a constant size
escapesByReturn                 1   caller keeps the pointer
boxSmallInt                     0   42 comes from the runtime's cache
boxLargeConst                   0   a constant gets one static copy
boxDynamicInt                   1   runtime value > 255: real boxing
escapesBySize                   1   100000 ints is too big for a frame
escapesByUnknownSize            1   length is a variable

three results worth staring at:
  1. stackPointer allocates NOTHING even though it takes an address.
  2. smallFixedSlice and escapesByUnknownSize both make a 16-element slice;
     only the one with a compile-time CONSTANT length stays on the stack.
  3. boxing only costs when the value is a runtime value outside 0..255.

to see the compiler's own reasoning:
  go build -gcflags='-m' -o /dev/null .

careful reading it, though: for make([]int, size) the compiler prints
"does not escape" and the allocation still happens. 'Escapes' means
'outlives the frame'; a variable-length make can fail to escape and still
be heap-allocated because the frame size is not known. Trust AllocsPerRun.

consequences for data structures:
  - a tree/list node returned from a constructor always escapes: 1 alloc per node,
    which is most of why pointer structures lose to slices (examples 9 and 13)
  - storing T in an interface boxes it unless it is a pointer or a cached small int
  - make([]T, n) with a variable n always heap-allocates -- hoist it out of loops
```

Three findings worth keeping:

- **`stackPointer` allocates nothing.** Taking an address is not what costs; outliving the frame is.
- **Constants never allocate.** `42` comes from the runtime's small-int cache; `1 << 40` gets one static instance. Only `boxDynamicInt` — a runtime value above 255 — pays for boxing.
- **`-gcflags=-m` says "does not escape" for `make([]int, size)`, which still allocates.** "Escapes" means "outlives the frame"; a variable-length `make` can fail to escape and still be heap-allocated because the frame size isn't known.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::any::Any;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Go measures with testing.AllocsPerRun. Rust has no such helper, so we count
// allocator calls ourselves. A global allocator is an `unsafe` trait by
// definition; this one only counts and forwards every call to the system.
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

/// Allocation events (alloc + realloc) while running `f`, and its result.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let before = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - before, r)
}

/// Like testing.AllocsPerRun: allocations per call, averaged over `runs`.
fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> usize {
    let (events, ()) = allocs_of(|| (0..runs).for_each(|_| f()));
    assert_eq!(events % runs, 0, "allocation count varied between runs");
    events / runs
}

#[derive(Clone, Copy)]
struct Point {
    x: i64,
    y: i64,
}

// Go's typed sinks become std::hint::black_box: it keeps the optimiser from
// deleting work whose result nobody reads, and it takes its argument by value
// without boxing, so it adds no allocation of its own.

// --- these stay on the stack ---

// stack_local: the struct never leaves the function.
fn stack_local(a: i64, b: i64) -> i64 {
    let p = Point { x: a, y: b };
    p.x + p.y
}

// stack_ref: taking a reference allocates nothing. A reference points INTO a
// frame; it cannot outlive it, so there is nothing to decide.
fn stack_ref(a: i64, b: i64) -> i64 {
    let p = &Point { x: a, y: b };
    p.x + p.y
}

// small_fixed_array: [i64; 16] has a compile-time size, so it is just 128 bytes of frame.
fn small_fixed_array() -> i64 {
    let mut buf = [0i64; 16];
    for (i, slot) in buf.iter_mut().enumerate() {
        *slot = i as i64;
    }
    buf[15]
}

// return_by_value: Go's `return &point{..}` must heap-allocate because the
// caller keeps the pointer. Rust returns the struct itself; it is moved into
// the caller's frame. (Returning a reference to a local does not compile:
//     fn f() -> &'static Point { let p = Point {..}; &p }
//     error[E0515]: cannot return reference to local variable `p`.)
fn return_by_value(a: i64, b: i64) -> Point {
    Point { x: a, y: b }
}

// return_box: here you asked for the heap, and you get exactly one allocation.
fn return_box(a: i64, b: i64) -> Box<Point> {
    Box::new(Point { x: a, y: b })
}

// dyn_ref: a trait object is a (data pointer, vtable pointer) pair. Borrowing a
// local as &dyn Any points at the stack: no box, no allocation.
fn dyn_ref(x: &dyn Any) -> bool {
    x.is::<i64>()
}

// --- these allocate: Box<dyn Any> is a real box, whatever it holds ---

// Go caches boxed 0..255 and gives constants a static copy. Rust has neither
// trick, so all three of these allocate.
fn box_small_int() -> Box<dyn Any> {
    Box::new(42i64)
}

fn box_large_const() -> Box<dyn Any> {
    Box::new(1i64 << 40)
}

fn box_dynamic_int(counter: &mut i64) -> Box<dyn Any> {
    *counter += 1; // no mutable globals in Rust: the counter is threaded through
    Box::new(*counter)
}

// big_array: 800 KB. Rust never moves a big local to the heap behind your back:
// it stays on the stack (the main thread has ~8 MB, typically). A 10_000_000-
// element array here would overflow the stack, not allocate.
fn big_array() -> usize {
    let buf = [0i64; 100_000];
    black_box(&buf);
    buf.len()
}

// vec_const_len: a Vec's buffer is ALWAYS on the heap, even at constant length.
// The choice between stack and heap is made by the type you write.
fn vec_const_len() -> usize {
    let v = vec![0i64; 16];
    black_box(&v).len()
}

// vec_variable_len: the length is a runtime value. The stack alternative does not
// exist: `let buf = [0i64; n];` is error[E0435]: attempt to use a non-constant value in a constant.
fn vec_variable_len(n: usize) -> usize {
    let v = vec![0i64; n];
    black_box(&v).len()
}

// vec_new_empty: an empty Vec owns no buffer yet; the first push allocates.
fn vec_new_empty() -> usize {
    let v: Vec<i64> = Vec::new();
    black_box(&v).len()
}

struct Node {
    val: i64,
    next: Option<Box<Node>>,
}

fn build_list(n: i64) -> Option<Box<Node>> {
    let mut head = None;
    for i in 0..n {
        head = Some(Box::new(Node { val: i, next: head }));
    }
    head
}

// Iterative on purpose: it also reads every field, so the list is not dead code.
fn list_sum(mut cur: &Option<Box<Node>>) -> i64 {
    let mut sum = 0;
    while let Some(node) = cur {
        sum += node.val;
        cur = &node.next;
    }
    sum
}

// One case of the table: a name, a closure that runs it once, and why.
type Case<'a> = (&'static str, Box<dyn FnMut() + 'a>, &'static str);

fn main() {
    println!("Rust has no escape analysis to consult: the heap is spelled in the type.");
    println!("Box::new, vec!, String, Rc::new and the collections are where allocation happens.");
    println!("Allocations per call, measured:");
    println!();

    // black_box(16) so the compiler cannot fold the length into a constant.
    let size = black_box(16usize);
    let mut counter = 1000i64; // starts above 255 like Go's, though Rust has no such cache

    let mut cases: Vec<Case> = vec![
        (
            "stack_local",
            Box::new(|| {
                black_box(stack_local(black_box(1), black_box(2)));
            }),
            "value never leaves",
        ),
        (
            "stack_ref",
            Box::new(|| {
                black_box(stack_ref(black_box(1), black_box(2)));
            }),
            "&p taken, still does not leave",
        ),
        (
            "small_fixed_array",
            Box::new(|| {
                black_box(small_fixed_array());
            }),
            "[i64; 16]: a constant size",
        ),
        (
            "return_by_value",
            Box::new(|| {
                black_box(return_by_value(black_box(1), black_box(2)));
            }),
            "moved into the caller's frame",
        ),
        (
            "return_box",
            Box::new(|| {
                black_box(return_box(black_box(1), black_box(2)));
            }),
            "Box::new: you asked for the heap",
        ),
        (
            "dyn_ref",
            Box::new(|| {
                let n = black_box(7i64);
                black_box(dyn_ref(&n));
            }),
            "&dyn Any of a local: no box",
        ),
        (
            "box_small_int",
            Box::new(|| {
                black_box(box_small_int());
            }),
            "42: no small-int cache",
        ),
        (
            "box_large_const",
            Box::new(|| {
                black_box(box_large_const());
            }),
            "a constant: no static copy",
        ),
        (
            "box_dynamic_int",
            Box::new(|| {
                black_box(box_dynamic_int(&mut counter));
            }),
            "a runtime value: boxed",
        ),
        (
            "big_array",
            Box::new(|| {
                black_box(big_array());
            }),
            "[i64; 100_000] on the stack",
        ),
        (
            "vec_const_len",
            Box::new(|| {
                black_box(vec_const_len());
            }),
            "a Vec is always heap",
        ),
        (
            "vec_variable_len",
            Box::new(|| {
                black_box(vec_variable_len(size));
            }),
            "length is a variable",
        ),
        (
            "vec_new_empty",
            Box::new(|| {
                black_box(vec_new_empty());
            }),
            "no buffer until the first push",
        ),
    ];

    println!("{:<20} {:>10}   why", "function", "allocs/op");
    for (name, f, why) in cases.iter_mut() {
        println!("{name:<20} {:>10}   {why}", allocs_per_run(100, f));
    }

    println!();
    println!("three results worth staring at:");
    println!("  1. stack_ref allocates nothing, and neither does return_by_value, although");
    println!("     the Point leaves its function. A value moves to its caller; only a Box,");
    println!("     which you wrote, puts it on the heap.");
    println!("  2. small_fixed_array and vec_const_len hold the same 16 integers at the same");
    println!("     constant length. The array is free and the Vec costs one allocation:");
    println!("     the TYPE decides, not an analysis.");
    println!("  3. Box<dyn Any> costs one allocation for 42, for 1 << 40 and for a runtime");
    println!("     value alike. A &dyn Any of a stack value costs none.");
    println!();
    println!("there is no `go build -gcflags=-m` to read, and nothing to misread: your own");
    println!("code allocates where you wrote Box/Vec/String/Rc/Arc. A library call can");
    println!("allocate inside, though, so trust the counting allocator for those.");

    println!();
    println!("consequences for data structures, measured at 1000 elements:");
    let (list_allocs, list) = allocs_of(|| build_list(1000));
    println!("  linked list, one Box per node:     {list_allocs:>4} allocations");
    let (pre_allocs, pre) = allocs_of(|| {
        let mut v = Vec::with_capacity(1000);
        for i in 0..1000i64 {
            v.push(i);
        }
        v
    });
    println!("  Vec::with_capacity(1000) + push:   {pre_allocs:>4} allocation");
    let (grow_allocs, grown) = allocs_of(|| {
        let mut v = Vec::new();
        for i in 0..1000i64 {
            v.push(i);
        }
        v
    });
    println!(
        "  Vec::new() + push, no capacity:    {grow_allocs:>4} allocation events (it regrows)"
    );
    println!("  - a list/tree node in a Box costs one allocation per node, a Vec of nodes one");
    println!("    in total: the gap that examples 9 and 13 are about");
    println!("  - Box<dyn Trait> costs one per value; generics and &dyn Trait cost none");
    println!(
        "  - Vec::with_capacity(n) once, outside the loop, and reuse it (clear keeps capacity)"
    );

    // Read the results so none of the measured work was dead code.
    black_box((list_sum(&list), pre.len(), grown.len()));
}
```

**Rust sample output:**

```
Rust has no escape analysis to consult: the heap is spelled in the type.
Box::new, vec!, String, Rc::new and the collections are where allocation happens.
Allocations per call, measured:

function              allocs/op   why
stack_local                   0   value never leaves
stack_ref                     0   &p taken, still does not leave
small_fixed_array             0   [i64; 16]: a constant size
return_by_value               0   moved into the caller's frame
return_box                    1   Box::new: you asked for the heap
dyn_ref                       0   &dyn Any of a local: no box
box_small_int                 1   42: no small-int cache
box_large_const               1   a constant: no static copy
box_dynamic_int               1   a runtime value: boxed
big_array                     0   [i64; 100_000] on the stack
vec_const_len                 1   a Vec is always heap
vec_variable_len              1   length is a variable
vec_new_empty                 0   no buffer until the first push

three results worth staring at:
  1. stack_ref allocates nothing, and neither does return_by_value, although
     the Point leaves its function. A value moves to its caller; only a Box,
     which you wrote, puts it on the heap.
  2. small_fixed_array and vec_const_len hold the same 16 integers at the same
     constant length. The array is free and the Vec costs one allocation:
     the TYPE decides, not an analysis.
  3. Box<dyn Any> costs one allocation for 42, for 1 << 40 and for a runtime
     value alike. A &dyn Any of a stack value costs none.

there is no `go build -gcflags=-m` to read, and nothing to misread: your own
code allocates where you wrote Box/Vec/String/Rc/Arc. A library call can
allocate inside, though, so trust the counting allocator for those.

consequences for data structures, measured at 1000 elements:
  linked list, one Box per node:     1000 allocations
  Vec::with_capacity(1000) + push:      1 allocation
  Vec::new() + push, no capacity:       9 allocation events (it regrows)
  - a list/tree node in a Box costs one allocation per node, a Vec of nodes one
    in total: the gap that examples 9 and 13 are about
  - Box<dyn Trait> costs one per value; generics and &dyn Trait cost none
  - Vec::with_capacity(n) once, outside the loop, and reuse it (clear keeps capacity)
```

> **Go vs Rust:** Escape analysis does not exist in Rust, so the question becomes "which types put it on the heap?". `return_by_value` is 0 where Go's `escapesByReturn` is 1 (the struct moves to the caller; a returned reference to a local is a compile error), `Box<dyn Any>` is 1 for `42` and `1 << 40` where Go's boxing of both is 0 (no small-int cache, no static copy), and `vec![0; 16]` is 1 where Go's constant-size `make` is 0 (a `Vec` is always heap; the stack spelling is `[i64; 16]`, which has to be constant-sized). A `Vec`'s regrowth count (9 events for 1000 pushes) is std's policy, not a guarantee, hence "sample".

**Complexity:** a stack allocation is Θ(1) and free · a heap allocation is Θ(1) plus GC work forever after — which is why "1 alloc per node" is the dominant cost of pointer structures

---

## 6. A million integers, eight ways

`🟢 easy` · *Bytes per element*

Lesson 03 taught you to drop the constant on Θ(n) space. Here is the constant. Same million
integers, same Θ(n), and a **9.4× spread** from top to bottom.

**Steps:**

1. Build each representation in isolation and measure the heap delta.
2. Divide by n to get bytes per element.
3. Check the results against `unsafe.Sizeof` — and check one common assumption.

```go
package main

import (
	"fmt"
	"runtime"
	"unsafe"
)

type node struct {
	val  int
	next *node
}

func heapAlloc() uint64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.HeapAlloc
}

// bytesPerElement builds one structure holding n ints and reports what it cost
// per element. The value is dropped after measuring, so each run starts clean.
func bytesPerElement(n int, build func(int) any) float64 {
	before := heapAlloc()
	v := build(n)
	after := heapAlloc()
	runtime.KeepAlive(v)
	return float64(after-before) / float64(n)
}

func main() {
	const n = 1_000_000

	reps := []struct {
		name string
		note string
		f    func(int) any
	}{
		{"[]int32", "4 bytes of payload, nothing else", func(n int) any {
			xs := make([]int32, n)
			for i := range xs {
				xs[i] = int32(i)
			}
			return xs
		}},
		{"[]int", "8 bytes of payload, nothing else", func(n int) any {
			xs := make([]int, n)
			for i := range xs {
				xs[i] = i
			}
			return xs
		}},
		{"[]node (values)", "16-byte struct, one contiguous block", func(n int) any {
			xs := make([]node, n)
			for i := range xs {
				xs[i].val = i
			}
			return xs
		}},
		{"[]*int", "8-byte pointer + a separate 8-byte object", func(n int) any {
			xs := make([]*int, n)
			for i := range xs {
				v := i
				xs[i] = &v
			}
			return xs
		}},
		{"linked list", "one 16-byte node per element, all separate", func(n int) any {
			var head *node
			for i := 0; i < n; i++ {
				head = &node{val: i, next: head}
			}
			return head
		}},
		{"map[int]struct{}", "hash table overhead: control bytes + spare slots", func(n int) any {
			m := make(map[int]struct{}, n)
			for i := 0; i < n; i++ {
				m[i] = struct{}{}
			}
			return m
		}},
		{"map[int]int", "a zero-size value saves nothing -- see below", func(n int) any {
			m := make(map[int]int, n)
			for i := 0; i < n; i++ {
				m[i] = i
			}
			return m
		}},
		{"map[int][4]int64", "a 32-byte value DOES cost", func(n int) any {
			m := make(map[int][4]int64, n)
			for i := 0; i < n; i++ {
				m[i] = [4]int64{int64(i)}
			}
			return m
		}},
	}

	fmt.Printf("holding %d integers, %d ways:\n\n", n, len(reps))
	fmt.Printf("%-20s %12s %12s   %s\n", "representation", "MB", "bytes/elem", "why")

	for _, r := range reps {
		bpe := bytesPerElement(n, r.f)
		fmt.Printf("%-20s %12.1f %12.1f   %s\n", r.name, bpe*float64(n)/(1<<20), bpe, r.note)
	}

	fmt.Println()
	fmt.Println("the declared sizes, for comparison:")
	var nd node
	var p *int
	fmt.Printf("  unsafe.Sizeof(node{}) = %d    unsafe.Sizeof((*int)(nil)) = %d\n",
		unsafe.Sizeof(nd), unsafe.Sizeof(p))

	fmt.Println()
	fmt.Println("read the gaps:")
	fmt.Println("  []int vs []*int       -- 2x, and the pointer version holds the SAME data")
	fmt.Println("  []node vs linked list -- identical bytes per element, completely different")
	fmt.Println("                           layout: one block vs a million separate objects")
	fmt.Println("  []int vs map          -- ~4.7x. That is what O(1) lookup costs in memory.")
	fmt.Println()
	fmt.Println("the surprise: map[int]struct{} costs the SAME as map[int]int here.")
	fmt.Println("The empty-struct 'set' trick saves nothing at this key size -- the table's")
	fmt.Println("own overhead dominates. A 32-byte value does show up (see the last row), so")
	fmt.Println("value size matters eventually; it just is not free below the noise floor.")
	fmt.Println("Measure your own key/value pair before assuming struct{} bought you anything.")
	fmt.Println()
	fmt.Println("every one of these is 'O(n) space'. Lesson 03 dropped the constant;")
	fmt.Println("this is the constant, and it decides whether your index fits in RAM.")
}
```

**Sample output** (map overhead depends on the Go version's map implementation):

```
holding 1000000 integers, 8 ways:

representation                 MB   bytes/elem   why
[]int32                       3.8          4.0   4 bytes of payload, nothing else
[]int                         7.6          8.0   8 bytes of payload, nothing else
[]node (values)              15.3         16.0   16-byte struct, one contiguous block
[]*int                       15.3         16.0   8-byte pointer + a separate 8-byte object
linked list                  15.3         16.0   one 16-byte node per element, all separate
map[int]struct{}             36.1         37.8   hash table overhead: control bytes + spare slots
map[int]int                  36.1         37.8   a zero-size value saves nothing -- see below
map[int][4]int64             96.1        100.7   a 32-byte value DOES cost

the declared sizes, for comparison:
  unsafe.Sizeof(node{}) = 16    unsafe.Sizeof((*int)(nil)) = 8

read the gaps:
  []int vs []*int       -- 2x, and the pointer version holds the SAME data
  []node vs linked list -- identical bytes per element, completely different
                           layout: one block vs a million separate objects
  []int vs map          -- ~4.7x. That is what O(1) lookup costs in memory.

the surprise: map[int]struct{} costs the SAME as map[int]int here.
The empty-struct 'set' trick saves nothing at this key size -- the table's
own overhead dominates. A 32-byte value does show up (see the last row), so
value size matters eventually; it just is not free below the noise floor.
Measure your own key/value pair before assuming struct{} bought you anything.

every one of these is 'O(n) space'. Lesson 03 dropped the constant;
this is the constant, and it decides whether your index fits in RAM.
```

The row worth arguing with: **`map[int]struct{}` costs exactly the same as `map[int]int`**. The
empty-struct "set" idiom saves nothing at this key size — the table's own overhead dominates. A
32-byte value *does* show up, so value size matters eventually. Measure your own pair before assuming.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{HashMap, HashSet};
use std::hint::black_box;
use std::mem::size_of;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Go measures with runtime.MemStats after a forced GC. Rust has neither, so we
// count the bytes currently live on the heap instead; exact, with no GC noise.
// A global allocator is an `unsafe` trait by definition; this one only does
// arithmetic and forwards every call to the system allocator.
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

struct Node {
    #[allow(dead_code)] // stored so the node is 16 bytes; only its size is measured
    val: i64,
    next: Option<Box<Node>>,
}

// A Box chain drops RECURSIVELY by default: a million nodes would overflow the
// stack when the list is freed. Go's GC never recurses, so it has no such trap.
// Unlinking one node at a time makes the drop a loop.
struct List {
    head: Option<Box<Node>>,
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur = self.head.take();
        while let Some(mut node) = cur {
            cur = node.next.take();
        }
    }
}

// bytes_per_element builds one structure holding n ints and reports what it cost
// per element. The value is dropped after measuring, so each run starts clean.
// `build` is generic, so unlike Go's `any` there is no boxing to add noise.
fn bytes_per_element<T>(n: usize, build: impl FnOnce(usize) -> T) -> f64 {
    let before = LIVE.load(Relaxed);
    let v = build(n);
    let after = LIVE.load(Relaxed);
    black_box(&v); // the Rust twin of runtime.KeepAlive
    (after - before) as f64 / n as f64
}

// One table row: name, why, and a function that measures one representation.
type Row = (&'static str, &'static str, fn(usize) -> f64);

fn main() {
    const N: usize = 1_000_000;

    // The closures capture nothing, so they coerce to plain function pointers.
    let reps: [Row; 8] = [
        ("Vec<i32>", "4 bytes of payload, nothing else", |n| {
            bytes_per_element(n, |n| (0..n as i32).collect::<Vec<_>>())
        }),
        ("Vec<i64>", "8 bytes of payload, nothing else", |n| {
            bytes_per_element(n, |n| (0..n as i64).collect::<Vec<_>>())
        }),
        (
            "Vec<Node> (values)",
            "16-byte struct, one contiguous block",
            |n| {
                bytes_per_element(n, |n| {
                    (0..n as i64)
                        .map(|val| Node { val, next: None })
                        .collect::<Vec<_>>()
                })
            },
        ),
        (
            "Vec<Box<i64>>",
            "8-byte pointer + a separate 8-byte object",
            |n| bytes_per_element(n, |n| (0..n as i64).map(Box::new).collect::<Vec<_>>()),
        ),
        (
            "linked list",
            "one 16-byte node per element, all separate",
            |n| {
                bytes_per_element(n, |n| {
                    let mut list = List { head: None };
                    for val in 0..n as i64 {
                        list.head = Some(Box::new(Node {
                            val,
                            next: list.head.take(),
                        }));
                    }
                    list
                })
            },
        ),
        (
            "HashSet<i64>",
            "hash table overhead: control bytes + spare slots",
            |n| {
                bytes_per_element(n, |n| {
                    let mut set = HashSet::with_capacity(n);
                    for i in 0..n as i64 {
                        set.insert(i);
                    }
                    set
                })
            },
        ),
        (
            "HashMap<i64, i64>",
            "the value is real payload here: stored in every bucket",
            |n| {
                bytes_per_element(n, |n| {
                    let mut map = HashMap::with_capacity(n);
                    for i in 0..n as i64 {
                        map.insert(i, i);
                    }
                    map
                })
            },
        ),
        (
            "HashMap<i64, [i64; 4]>",
            "a 32-byte value costs even more",
            |n| {
                bytes_per_element(n, |n| {
                    let mut map = HashMap::with_capacity(n);
                    for i in 0..n as i64 {
                        map.insert(i, [i, 0, 0, 0]);
                    }
                    map
                })
            },
        ),
    ];

    println!("holding {N} integers, {} ways:\n", reps.len());
    println!(
        "{:<24} {:>10} {:>12}   why",
        "representation", "MB", "bytes/elem"
    );

    let mut bpe = Vec::new();
    for (name, why, measure) in reps {
        let b = measure(N);
        bpe.push(b);
        println!(
            "{name:<24} {:>10.1} {b:>12.1}   {why}",
            b * N as f64 / (1 << 20) as f64
        );
    }
    let [_, vec_i64, vec_node, vec_box, list, set, map, map_big] = bpe[..] else {
        unreachable!("eight rows")
    };

    println!();
    println!("the declared sizes, for comparison:");
    println!(
        "  size_of::<Node>() = {}    size_of::<Box<i64>>() = {}    size_of::<Option<Box<Node>>>() = {}",
        size_of::<Node>(),
        size_of::<Box<i64>>(),
        size_of::<Option<Box<Node>>>()
    );
    println!(
        "  size_of::<(i64, ())>() = {}    size_of::<(i64, i64)>() = {}    size_of::<(i64, [i64; 4])>() = {}",
        size_of::<(i64, ())>(),
        size_of::<(i64, i64)>(),
        size_of::<(i64, [i64; 4])>()
    );

    println!();
    println!("read the gaps:");
    println!(
        "  Vec<i64> vs Vec<Box<i64>> -- {:.1}x in requested bytes, and the boxed version holds the SAME data",
        vec_box / vec_i64
    );
    println!(
        "  Vec<Node> vs linked list  -- {vec_node:.1} vs {list:.1} bytes per element: the same, but one block vs a million separate objects"
    );
    println!(
        "  Vec<i64> vs HashSet       -- {:.1}x. That is what O(1) lookup costs in memory.",
        set / vec_i64
    );
    println!();
    println!(
        "HashSet<i64> vs HashMap<i64, i64>: {set:.1} vs {map:.1} bytes per element, {:.1}x.",
        map / set
    );
    println!("A HashSet is a HashMap<T, ()>, and a () value takes zero bytes in each bucket,");
    println!("so here the empty-value 'set' trick DOES pay: it is the (i64, ()) vs (i64, i64)");
    println!("line above. A 32-byte value is bigger again ({map_big:.1} bytes per element).");
    // Where do the 18.9 come from? Ask the table how much room it really made.
    let cap = HashSet::<i64>::with_capacity(N).capacity();
    let buckets = cap * 8 / 7; // hashbrown keeps its tables at most 7/8 full
    let per_bucket = 1 + size_of::<i64>(); // one control byte + the key
    println!(
        "asking for {N} slots gives capacity {cap}, i.e. {buckets} buckets (7/8 full at most)."
    );
    println!(
        "{buckets} buckets x {per_bucket} bytes = {:.1} per element: 8.0 is the key itself,",
        (buckets * per_bucket) as f64 / N as f64
    );
    println!("the rest is control bytes and spare slots.");
    println!();
    println!("these are REQUESTED bytes: the system allocator rounds each block up and may add");
    println!("a header, which is not counted. For a million separate boxes that is extra, not");
    println!("measured here. The Vec rows are a single block, so they are exact.");
    println!();
    println!("every one of these is 'O(n) space'. Lesson 03 dropped the constant;");
    println!("this is the constant, and it decides whether your index fits in RAM.");
}
```

**Rust sample output:**

```
holding 1000000 integers, 8 ways:

representation                   MB   bytes/elem   why
Vec<i32>                        3.8          4.0   4 bytes of payload, nothing else
Vec<i64>                        7.6          8.0   8 bytes of payload, nothing else
Vec<Node> (values)             15.3         16.0   16-byte struct, one contiguous block
Vec<Box<i64>>                  15.3         16.0   8-byte pointer + a separate 8-byte object
linked list                    15.3         16.0   one 16-byte node per element, all separate
HashSet<i64>                   18.0         18.9   hash table overhead: control bytes + spare slots
HashMap<i64, i64>              34.0         35.7   the value is real payload here: stored in every bucket
HashMap<i64, [i64; 4]>         82.0         86.0   a 32-byte value costs even more

the declared sizes, for comparison:
  size_of::<Node>() = 16    size_of::<Box<i64>>() = 8    size_of::<Option<Box<Node>>>() = 8
  size_of::<(i64, ())>() = 8    size_of::<(i64, i64)>() = 16    size_of::<(i64, [i64; 4])>() = 40

read the gaps:
  Vec<i64> vs Vec<Box<i64>> -- 2.0x in requested bytes, and the boxed version holds the SAME data
  Vec<Node> vs linked list  -- 16.0 vs 16.0 bytes per element: the same, but one block vs a million separate objects
  Vec<i64> vs HashSet       -- 2.4x. That is what O(1) lookup costs in memory.

HashSet<i64> vs HashMap<i64, i64>: 18.9 vs 35.7 bytes per element, 1.9x.
A HashSet is a HashMap<T, ()>, and a () value takes zero bytes in each bucket,
so here the empty-value 'set' trick DOES pay: it is the (i64, ()) vs (i64, i64)
line above. A 32-byte value is bigger again (86.0 bytes per element).
asking for 1000000 slots gives capacity 1835008, i.e. 2097152 buckets (7/8 full at most).
2097152 buckets x 9 bytes = 18.9 per element: 8.0 is the key itself,
the rest is control bytes and spare slots.

these are REQUESTED bytes: the system allocator rounds each block up and may add
a header, which is not counted. For a million separate boxes that is extra, not
measured here. The Vec rows are a single block, so they are exact.

every one of these is 'O(n) space'. Lesson 03 dropped the constant;
this is the constant, and it decides whether your index fits in RAM.
```

> **Go vs Rust:** The contiguous rows match Go exactly (4.0, 8.0, 16.0, 16.0, 16.0 bytes per element). The maps do not: Rust's `HashSet<i64>` costs 18.9 B/element against 35.7 for `HashMap<i64, i64>`, so unlike Go's `map[int]struct{}` (same as `map[int]int`) the empty-value set does save about half, because a `()` value takes zero bytes per bucket. Numbers are bytes requested from a counting allocator (exact, no GC), not bytes the OS hands out; hash-table layout is std's, hence "sample". Rust needs a hand-written iterative `Drop` for the linked list, since the default drop of a million-node `Box` chain recurses; Go's GC has no such trap.

**Complexity:** all eight are Θ(n) space · the constants are 4, 8, 16, 16, 16, 37.8, 37.8 and 100.7 bytes, and the constant is what decides whether your index fits in RAM

---

> Next tier: [🟡 medium](2-medium.md) — where layout starts costing time.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
