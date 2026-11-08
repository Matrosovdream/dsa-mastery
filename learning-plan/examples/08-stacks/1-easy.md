# Step 08 — Stacks · 🟢 Easy

Examples **1–6**: the structure that needs no structure, and the problems that *are* stacks.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Deterministic except example 2, which benchmarks the generic wrapper.

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

## 1. A slice is already a stack

`🟢 easy` · *LIFO for free*

After two lessons of trade-offs, this is a relief. A stack only ever touches the END of the
sequence — the cheap end of a slice — so there is no structure to build.

**Steps:**

1. Push with `append`, peek at `len-1`, pop by reslicing.
2. Watch that nothing shifts and nothing is chased.
3. Then handle the one case that can panic.

```go
package main

import "fmt"

// After two lessons of trade-offs, the stack is a relief: it only ever touches
// the END of the sequence, which is the cheap end of a slice (lesson 06).
//
// So in Go a slice ALREADY IS a stack. There is no structure to build:
//
//	push    xs = append(xs, v)      Theta(1) amortized
//	peek    xs[len(xs)-1]           Theta(1)
//	pop     xs = xs[:len(xs)-1]     Theta(1)
//
// Every operation is at index len-1. Nothing shifts, nothing is chased, and the
// data stays contiguous. A stack has none of lesson 07's problems because it
// never asks for anything a slice is bad at.

func main() {
	fmt.Println("a []int used directly as a stack:")
	fmt.Println()

	var stack []int

	push := func(v int) {
		stack = append(stack, v)
		fmt.Printf("  push %-3d -> %v  (len=%d cap=%d)\n", v, stack, len(stack), cap(stack))
	}
	for _, v := range []int{1, 2, 3} {
		push(v)
	}

	fmt.Printf("\n  peek     -> %d   (the top is the LAST element, not the first)\n", stack[len(stack)-1])
	fmt.Printf("  len      -> %d\n", len(stack))

	fmt.Println()
	for len(stack) > 0 {
		top := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		fmt.Printf("  pop      -> %-3d  remaining %v\n", top, stack)
	}

	fmt.Println()
	fmt.Println("the ONE thing you must get right: popping an empty stack.")
	fmt.Println()
	fmt.Println("  stack[len(stack)-1] on an empty slice is stack[-1] -- that is a")
	fmt.Println("  PANIC, not a nil you can test. So the pop signature has to report")
	fmt.Println("  emptiness, and the idiomatic way is a second return value:")
	fmt.Println()
	fmt.Println("      func Pop() (T, bool)")
	fmt.Println()
	fmt.Println("  the same shape as a map lookup or a channel receive. Callers then")
	fmt.Println("  write `if v, ok := s.Pop(); ok {` and cannot forget the check.")

	fmt.Println()
	fmt.Println("popping empty, safely:")
	fmt.Println()
	pop := func(s []int) ([]int, int, bool) {
		if len(s) == 0 {
			return s, 0, false
		}
		return s[:len(s)-1], s[len(s)-1], true
	}
	empty := []int{}
	_, v, ok := pop(empty)
	fmt.Printf("  pop on empty -> (%d, %v)   <- no panic, and the caller knows\n", v, ok)

	fmt.Println()
	fmt.Println("why LIFO falls out of a slice for free:")
	fmt.Println()
	fmt.Println("  append writes at the tail, and the tail is where a slice has spare")
	fmt.Println("  capacity (lesson 06, example 7). Reslicing to drop the last element")
	fmt.Println("  moves no data at all -- it only changes len.")
	fmt.Println()
	fmt.Println("  compare that with a QUEUE, which needs both ends: taking from the")
	fmt.Println("  front is where the slice stops being free, and that is lesson 09's")
	fmt.Println("  entire subject.")

	fmt.Println()
	fmt.Println("one memory note, from lesson 06's example 2:")
	fmt.Println()
	fmt.Println("  popping only shrinks len. The popped value is still sitting in the")
	fmt.Println("  array's spare capacity, so if T contains a pointer it stays alive.")
	fmt.Println("  Zero the slot on the way out -- example 2 does exactly that.")
}
```

**Output:**

```
a []int used directly as a stack:

  push 1   -> [1]  (len=1 cap=1)
  push 2   -> [1 2]  (len=2 cap=2)
  push 3   -> [1 2 3]  (len=3 cap=4)

  peek     -> 3   (the top is the LAST element, not the first)
  len      -> 3

  pop      -> 3    remaining [1 2]
  pop      -> 2    remaining [1]
  pop      -> 1    remaining []

the ONE thing you must get right: popping an empty stack.

  stack[len(stack)-1] on an empty slice is stack[-1] -- that is a
  PANIC, not a nil you can test. So the pop signature has to report
  emptiness, and the idiomatic way is a second return value:

      func Pop() (T, bool)

  the same shape as a map lookup or a channel receive. Callers then
  write `if v, ok := s.Pop(); ok {` and cannot forget the check.

popping empty, safely:

  pop on empty -> (0, false)   <- no panic, and the caller knows

why LIFO falls out of a slice for free:

  append writes at the tail, and the tail is where a slice has spare
  capacity (lesson 06, example 7). Reslicing to drop the last element
  moves no data at all -- it only changes len.

  compare that with a QUEUE, which needs both ends: taking from the
  front is where the slice stops being free, and that is lesson 09's
  entire subject.

one memory note, from lesson 06's example 2:

  popping only shrinks len. The popped value is still sitting in the
  array's spare capacity, so if T contains a pointer it stays alive.
  Zero the slot on the way out -- example 2 does exactly that.
```

**Rust:**

```rust
use std::hint::black_box;
use std::panic::catch_unwind;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// After two lessons of trade-offs, the stack is a relief: it only ever touches
// the END of the sequence, which is the cheap end of a Vec (lesson 06).
//
// So in Rust a Vec<T> ALREADY IS a stack. There is no structure to build:
//
//	push    xs.push(v)         Theta(1) amortized
//	peek    xs.last()          Theta(1)   -> Option<&T>
//	pop     xs.pop()           Theta(1)   -> Option<T>
//
// Every operation is at index len-1. Nothing shifts, nothing is chased, and the
// data stays contiguous. A stack has none of lesson 07's problems because it
// never asks for anything a Vec is bad at.

/// Go's `%v` of a slice: [1 2 3]
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

// A value that counts its own death. There is no GC to ask "is it still
// reachable?", but `Drop` runs exactly when the value is destroyed.
static DROPS: AtomicUsize = AtomicUsize::new(0);
static LAST_DROPPED: AtomicUsize = AtomicUsize::new(0); // id of the Job destroyed last
struct Job {
    id: usize,
}
impl Drop for Job {
    fn drop(&mut self) {
        DROPS.fetch_add(1, Relaxed);
        LAST_DROPPED.store(self.id, Relaxed);
    }
}

fn main() {
    println!("a Vec<i64> used directly as a stack:");
    println!();

    let mut stack: Vec<i64> = Vec::new();
    for v in [1, 2, 3] {
        stack.push(v);
        println!(
            "  push {v:<3} -> {}  (len={} cap={})",
            show(&stack),
            stack.len(),
            stack.capacity()
        );
    }

    println!();
    println!("  (cap reads 4 from the first push: this std allocates room for 4 i64s at");
    println!("   once, where Go's append gave 1, 2, 4. Both are growth policies, not");
    println!("   part of the stack.)");
    println!();
    if let Some(top) = stack.last() {
        println!("  peek     -> {top}   (the top is the LAST element, not the first)");
    }
    println!("  len      -> {}", stack.len());

    println!();
    while let Some(top) = stack.pop() {
        println!("  pop      -> {top:<3}  remaining {}", show(&stack));
    }

    println!();
    println!("the ONE thing you must get right: popping an empty stack.");
    println!();
    println!("  xs[xs.len() - 1] on an empty Vec is a PANIC, not a value you can");
    println!("  test: the subtraction underflows (a debug build stops there; this -O");
    println!("  build wraps to usize::MAX and the index is out of bounds).");
    // Silence the default hook so the caught panic does not print to stderr.
    std::panic::set_hook(Box::new(|_| {}));
    let empty_vec: Vec<i64> = black_box(Vec::new());
    let caught = catch_unwind(move || empty_vec[empty_vec.len() - 1]);
    println!(
        "  checked here, not assumed: indexing an empty Vec at len-1 panicked = {}",
        caught.is_err()
    );
    println!();
    println!("  So the pop signature has to report emptiness, and Rust's type for");
    println!("  that is Option<T>. Vec already has it:");
    println!();
    println!("      fn pop(&mut self) -> Option<T>");
    println!();
    println!("  Go's second return value can be thrown away (`v, _ := s.Pop()` hands");
    println!("  you a zero value silently). The value is INSIDE the Option, so the");
    println!("  caller cannot reach it without writing `if let Some(v)`, a `match`,");
    println!("  or an `.unwrap()` that says out loud 'I assert this is non-empty'.");

    println!();
    println!("popping empty, safely:");
    println!();
    let mut empty: Vec<i64> = Vec::new();
    println!(
        "  pop on empty -> {:?}   <- no panic, and the caller knows",
        empty.pop()
    );

    println!();
    println!("why LIFO falls out of a Vec for free:");
    println!();
    println!("  push writes at the tail, and the tail is where a Vec has spare");
    println!("  capacity (lesson 06, example 7). Popping reads the last element out");
    println!("  and lowers len -- it moves no other data at all.");
    println!();
    println!("  compare that with a QUEUE, which needs both ends: taking from the");
    println!("  front is where a Vec stops being free, and that is lesson 09's");
    println!("  entire subject.");

    println!();
    println!("one memory note, from lesson 06's example 2 -- and here Rust differs:");
    println!();
    // Count the destructors of three Jobs pushed onto a stack with spare capacity.
    let mut jobs: Vec<Job> = Vec::with_capacity(4);
    for id in 1..=3 {
        jobs.push(Job { id });
    }
    let popped = jobs.pop(); // moved OUT of the Vec: the Vec no longer owns it
    println!(
        "  popped one Job, still holding it:    dropped={}  len={} cap={}",
        DROPS.load(Relaxed),
        jobs.len(),
        jobs.capacity()
    );
    drop(popped);
    println!(
        "  the popped Job goes out of scope:    dropped={} (job #{})  len={} cap={}",
        DROPS.load(Relaxed),
        LAST_DROPPED.load(Relaxed),
        jobs.len(),
        jobs.capacity()
    );
    println!();
    println!("  Go's pop only shrinks len, so the popped value would sit in the");
    println!("  spare capacity and stay alive until someone zeroes the slot. Vec::pop");
    println!("  MOVES the value out, so the slot past len owns nothing: the Job above");
    println!("  was destroyed the moment its last owner let go, with the capacity");
    println!("  untouched. There is no stale slot to zero, and safe code cannot read");
    println!("  past len to find one.");
}
```

**Rust output:**

```
a Vec<i64> used directly as a stack:

  push 1   -> [1]  (len=1 cap=4)
  push 2   -> [1 2]  (len=2 cap=4)
  push 3   -> [1 2 3]  (len=3 cap=4)

  (cap reads 4 from the first push: this std allocates room for 4 i64s at
   once, where Go's append gave 1, 2, 4. Both are growth policies, not
   part of the stack.)

  peek     -> 3   (the top is the LAST element, not the first)
  len      -> 3

  pop      -> 3    remaining [1 2]
  pop      -> 2    remaining [1]
  pop      -> 1    remaining []

the ONE thing you must get right: popping an empty stack.

  xs[xs.len() - 1] on an empty Vec is a PANIC, not a value you can
  test: the subtraction underflows (a debug build stops there; this -O
  build wraps to usize::MAX and the index is out of bounds).
  checked here, not assumed: indexing an empty Vec at len-1 panicked = true

  So the pop signature has to report emptiness, and Rust's type for
  that is Option<T>. Vec already has it:

      fn pop(&mut self) -> Option<T>

  Go's second return value can be thrown away (`v, _ := s.Pop()` hands
  you a zero value silently). The value is INSIDE the Option, so the
  caller cannot reach it without writing `if let Some(v)`, a `match`,
  or an `.unwrap()` that says out loud 'I assert this is non-empty'.

popping empty, safely:

  pop on empty -> None   <- no panic, and the caller knows

why LIFO falls out of a Vec for free:

  push writes at the tail, and the tail is where a Vec has spare
  capacity (lesson 06, example 7). Popping reads the last element out
  and lowers len -- it moves no other data at all.

  compare that with a QUEUE, which needs both ends: taking from the
  front is where a Vec stops being free, and that is lesson 09's
  entire subject.

one memory note, from lesson 06's example 2 -- and here Rust differs:

  popped one Job, still holding it:    dropped=0  len=2 cap=4
  the popped Job goes out of scope:    dropped=1 (job #3)  len=2 cap=4

  Go's pop only shrinks len, so the popped value would sit in the
  spare capacity and stay alive until someone zeroes the slot. Vec::pop
  MOVES the value out, so the slot past len owns nothing: the Job above
  was destroyed the moment its last owner let go, with the capacity
  untouched. There is no stale slot to zero, and safe code cannot read
  past len to find one.
```

> **Go vs Rust:** The pushes, peeks and pops are the Go trace line for line (a `Vec` is the stack; `push`/`last`/`pop` replace `append`, `xs[len-1]`, reslicing), except `cap`: Go grew 1, 2, 4 and this `Vec<i64>` allocates room for 4 on the first push (growth policy, not part of the stack). Go's `(T, bool)` pop becomes `Option<T>`, whose value cannot be reached without handling `None`, and indexing `xs[xs.len() - 1]` on an empty `Vec` panics too (caught and printed as `true`, no crash report). The stale-slot caveat does not carry over: `Vec::pop` moves the value out, and the `Drop` counter shows it destroyed (`dropped=1`) while `cap` stays 4, so there is nothing to zero.

**Complexity:** push Θ(1) amortized · peek and pop Θ(1) · a stack has none of lesson 07's problems because it never asks for anything a slice is bad at

---

## 2. Stack[T], and what the wrapper costs

`🟢 easy` · *The idiomatic Go container*

Generic, and whose **zero value is a usable empty stack** — no constructor to forget. The
`(T, bool)` pop is the same contract as a map lookup.

**Steps:**

1. Write the type so `var s Stack[int]` just works.
2. Zero the vacated slot on pop, and prove it.
3. Measure it against a bare slice — and do not assume the answer.

```go
package main

import (
	"fmt"
	"testing"
)

// The idiomatic Go container: generic, and whose ZERO VALUE is a usable empty
// stack. No constructor, no New(), nothing to forget.
//
//	var s Stack[int]   // ready to use
//
// That design comes straight from lesson 07's sentinel discussion: if the zero
// value works, callers cannot construct it wrong.

type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Len() int      { return len(s.items) }
func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

// Pop returns the top and whether there was one. The (T, bool) shape is the
// same contract as a map lookup, and it makes the empty case impossible to
// ignore.
func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	i := len(s.items) - 1
	v := s.items[i]
	s.items[i] = zero // release any pointer inside T (lesson 06, example 2)
	s.items = s.items[:i]
	return v, true
}

func (s *Stack[T]) Peek() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	return s.items[len(s.items)-1], true
}

// Reserve pre-sizes the stack when you know the depth (lesson 03).
func (s *Stack[T]) Reserve(n int) {
	if cap(s.items) < n {
		bigger := make([]T, len(s.items), n)
		copy(bigger, s.items)
		s.items = bigger
	}
}

type job struct{ name *string }

var sinkI int

func main() {
	fmt.Println("the zero value is a usable stack -- no constructor:")
	fmt.Println()
	var s Stack[string]
	fmt.Printf("  var s Stack[string]   IsEmpty=%v Len=%d\n", s.IsEmpty(), s.Len())

	for _, v := range []string{"a", "b", "c"} {
		s.Push(v)
	}
	top, _ := s.Peek()
	fmt.Printf("  after 3 pushes        Len=%d Peek=%q\n", s.Len(), top)

	for {
		v, ok := s.Pop()
		if !ok {
			break
		}
		fmt.Printf("  pop -> %q\n", v)
	}
	_, ok := s.Pop()
	fmt.Printf("  pop on empty -> ok=%v\n", ok)

	fmt.Println()
	fmt.Println("it works with any type, including ones a non-generic stack would box:")
	fmt.Println()
	type point struct{ x, y int }
	var ps Stack[point]
	ps.Push(point{1, 2})
	ps.Push(point{3, 4})
	p, _ := ps.Pop()
	fmt.Printf("  Stack[point]  popped %+v  -- stored as a VALUE, no interface\n", p)

	fmt.Println()
	fmt.Println("why Pop zeroes the slot it vacates:")
	fmt.Println()
	name := "still-referenced"
	var js Stack[job]
	js.Push(job{&name})
	js.Pop()
	// Look past len, into the capacity, at what the array still holds.
	held := js.items[:cap(js.items)][0].name
	fmt.Printf("  after Pop, items[:cap][0].name = %v   <- the zeroing worked\n", held)
	fmt.Println()
	fmt.Println("  without that line the popped job would stay reachable for as long")
	fmt.Println("  as the stack lives (lesson 04, example 3). For a Stack[int] nobody")
	fmt.Println("  cares; for a Stack[*Request] in a server it is a leak.")

	fmt.Println()
	fmt.Println("the cost of the wrapper, measured against a bare slice:")
	fmt.Println()
	const n = 1000
	tBare := nsPerOp(func() {
		xs := make([]int, 0, n)
		for i := 0; i < n; i++ {
			xs = append(xs, i)
		}
		for len(xs) > 0 {
			sinkI = xs[len(xs)-1]
			xs = xs[:len(xs)-1]
		}
	})
	tStack := nsPerOp(func() {
		var st Stack[int]
		st.Reserve(n)
		for i := 0; i < n; i++ {
			st.Push(i)
		}
		for {
			v, ok := st.Pop()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	fmt.Printf("  %-28s %12.0f ns\n", "bare slice push/pop x1000", tBare)
	fmt.Printf("  %-28s %12.0f ns  %.2fx\n", "Stack[int] push/pop x1000", tStack, tStack/tBare)
	fmt.Println()
	fmt.Printf("  that is about %.1f ns of overhead per operation, over 2000 ops.\n",
		(tStack-tBare)/(2*n))
	fmt.Println()
	fmt.Println("  do NOT wave that away -- the generic methods are not inlining here,")
	fmt.Println("  so each Push and Pop is a real call, plus Pop does an extra write to")
	fmt.Println("  zero the vacated slot. The wrapper is cheap, not free.")
	fmt.Println()
	fmt.Println("  whether ~2 ns matters is a question about YOUR workload, and lesson")
	fmt.Println("  03 already gave you the way to answer it: if the stack operation is")
	fmt.Println("  the innermost thing your program does, use the bare slice. If there")
	fmt.Println("  is any real work per element -- and there almost always is -- 2 ns")
	fmt.Println("  is invisible and the contract is worth having.")

	fmt.Println()
	fmt.Println("so: bare slice inside one function, where the stack is local and")
	fmt.Println("obvious and `stack = append(stack, x)` reads fine. The type when the")
	fmt.Println("stack crosses a function or package boundary, where (T, bool) and the")
	fmt.Println("pointer hygiene are doing real work.")
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}
```

**Sample output:**

```
the zero value is a usable stack -- no constructor:

  var s Stack[string]   IsEmpty=true Len=0
  after 3 pushes        Len=3 Peek="c"
  pop -> "c"
  pop -> "b"
  pop -> "a"
  pop on empty -> ok=false

it works with any type, including ones a non-generic stack would box:

  Stack[point]  popped {x:3 y:4}  -- stored as a VALUE, no interface

why Pop zeroes the slot it vacates:

  after Pop, items[:cap][0].name = <nil>   <- the zeroing worked

  without that line the popped job would stay reachable for as long
  as the stack lives (lesson 04, example 3). For a Stack[int] nobody
  cares; for a Stack[*Request] in a server it is a leak.

the cost of the wrapper, measured against a bare slice:

  bare slice push/pop x1000             671 ns
  Stack[int] push/pop x1000            4044 ns  6.03x

  that is about 1.7 ns of overhead per operation, over 2000 ops.

  do NOT wave that away -- the generic methods are not inlining here,
  so each Push and Pop is a real call, plus Pop does an extra write to
  zero the vacated slot. The wrapper is cheap, not free.

  whether ~2 ns matters is a question about YOUR workload, and lesson
  03 already gave you the way to answer it: if the stack operation is
  the innermost thing your program does, use the bare slice. If there
  is any real work per element -- and there almost always is -- 2 ns
  is invisible and the contract is worth having.

so: bare slice inside one function, where the stack is local and
obvious and `stack = append(stack, x)` reads fine. The type when the
stack crosses a function or package boundary, where (T, bool) and the
pointer hygiene are doing real work.
```

**Rust:**

```rust
use std::hint::black_box;
use std::rc::Rc;
use std::time::{Duration, Instant};

// The idiomatic Rust container: generic, and whose DEFAULT value is a usable
// empty stack. No constructor to call, nothing to forget.
//
//	let mut s: Stack<i64> = Stack::default();   // ready to use
//
// That design comes straight from lesson 07's sentinel discussion: if the
// default value works, callers cannot construct it wrong.
//
// One Rust detail: `#[derive(Default)]` would add a hidden `T: Default` bound,
// so `Stack<Job>` below (Job has no Default) would not get a default stack.
// Writing the impl by hand states what we mean: an empty Vec, for any T.

struct Stack<T> {
    items: Vec<T>,
}

impl<T> Default for Stack<T> {
    fn default() -> Self {
        Stack { items: Vec::new() }
    }
}

impl<T> Stack<T> {
    fn len(&self) -> usize {
        self.items.len()
    }
    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }

    fn push(&mut self, v: T) {
        self.items.push(v);
    }

    // Pop returns the top, if there is one. Go's (T, bool) becomes Option<T>:
    // the empty case is impossible to ignore, because the value is inside it.
    // And there is no slot to zero: Vec::pop moves the value OUT (example 1).
    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    // Peek borrows the top instead of moving it: Option<&T>.
    fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    // Reserve pre-sizes the stack when you know the depth (lesson 03).
    fn reserve(&mut self, n: usize) {
        self.items.reserve(n.saturating_sub(self.items.len()));
    }
}

struct Job {
    name: Rc<String>,
}

struct Point {
    x: i64,
    y: i64,
}

/// ns/op: stand-in for testing.Benchmark / b.Loop (the kit's version).
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

fn main() {
    println!("the default value is a usable stack -- no constructor:");
    println!();
    let mut s: Stack<String> = Stack::default();
    println!(
        "  Stack::<String>::default()   IsEmpty={} Len={}",
        s.is_empty(),
        s.len()
    );

    for v in ["a", "b", "c"] {
        s.push(v.to_string());
    }
    println!(
        "  after 3 pushes               Len={} Peek={:?}",
        s.len(),
        s.peek().expect("three pushes, so not empty")
    );

    while let Some(v) = s.pop() {
        println!("  pop -> {v:?}");
    }
    println!("  pop on empty -> {:?}", s.pop());

    println!();
    println!("it works with any type, including ones a non-generic stack would box:");
    println!();
    let mut ps: Stack<Point> = Stack::default();
    ps.push(Point { x: 1, y: 2 });
    ps.push(Point { x: 3, y: 4 });
    if let Some(p) = ps.pop() {
        println!(
            "  Stack<Point>  popped {{x:{} y:{}}}  -- stored as a VALUE, no Box<dyn>",
            p.x, p.y
        );
    }

    println!();
    println!("why Go's Pop zeroes the slot it vacates -- and why Rust's does not:");
    println!();
    let name = Rc::new(String::from("still-referenced"));
    let mut js: Stack<Job> = Stack::default();
    js.push(Job {
        name: Rc::clone(&name),
    });
    println!(
        "  after Push, owners of the name = {}   (us + the job on the stack)",
        Rc::strong_count(&name)
    );
    if let Some(job) = js.pop() {
        println!(
            "  Pop returned job {:?}; owners while the caller holds it = {}",
            job.name,
            Rc::strong_count(&name)
        );
    }
    println!(
        "  after the caller is done, owners = {}   <- len={} cap={}, nothing held past len",
        Rc::strong_count(&name),
        js.len(),
        js.items.capacity()
    );
    println!();
    println!("  Go needs the zeroing line, or the popped job stays reachable for as long");
    println!("  as the stack lives (lesson 04, example 3). Here the Vec gave up ownership");
    println!("  at the pop, so the count falls as soon as the caller lets go of the job,");
    println!("  with the capacity still allocated. For a Stack<Rc<Request>> in a server");
    println!("  that is the whole difference between a leak and no leak -- for free.");

    println!();
    println!("the cost of the wrapper, measured against a bare Vec:");
    println!();
    const N: usize = 1000;
    let t_bare = ns_per_op(|| {
        let mut xs: Vec<usize> = Vec::with_capacity(N);
        for i in 0..N {
            xs.push(black_box(i));
        }
        while let Some(v) = xs.pop() {
            black_box(v);
        }
    });
    let t_stack = ns_per_op(|| {
        let mut st: Stack<usize> = Stack::default();
        st.reserve(N);
        for i in 0..N {
            st.push(black_box(i));
        }
        while let Some(v) = st.pop() {
            black_box(v);
        }
    });
    println!("  {:<28} {:>12.0} ns", "bare Vec push/pop x1000", t_bare);
    println!(
        "  {:<28} {:>12.0} ns  {:.2}x",
        "Stack<usize> push/pop x1000",
        t_stack,
        t_stack / t_bare
    );
    println!();
    println!(
        "  that is {:+.2} ns of overhead per operation, over {} ops.",
        (t_stack - t_bare) / (2 * N) as f64,
        2 * N
    );
    println!();
    println!("  do NOT assume the Go answer here. Go's wrapper cost was real because its");
    println!("  generic methods were not inlining, plus Pop's extra zeroing write. In this");
    println!("  -O build the methods are inlined (the release assembly has no call to");
    println!("  Stack::push or Stack::pop) and Pop has no zeroing to do, so there is no");
    println!("  call and no extra write to pay for. Expect a ratio near 1.0x that wobbles");
    println!("  from run to run in both directions, and an overhead well under 0.1 ns per");
    println!("  op either way: noise, not signal.");
    println!();
    println!("  so the wrapper is a performance non-question in Rust, and the choice is");
    println!("  about the API. A bare Vec inside one function is fine and obvious. The type");
    println!("  earns its place when the stack crosses a function or module boundary:");
    println!("  it exposes push/pop/peek and NOT insert(0, x), remove(i) or indexing, so");
    println!("  a caller cannot use it as anything but a stack. (Vec's own pop already");
    println!("  returns Option<T>, so the contract is the same either way.)");
}
```

**Rust sample output:**

```
the default value is a usable stack -- no constructor:

  Stack::<String>::default()   IsEmpty=true Len=0
  after 3 pushes               Len=3 Peek="c"
  pop -> "c"
  pop -> "b"
  pop -> "a"
  pop on empty -> None

it works with any type, including ones a non-generic stack would box:

  Stack<Point>  popped {x:3 y:4}  -- stored as a VALUE, no Box<dyn>

why Go's Pop zeroes the slot it vacates -- and why Rust's does not:

  after Push, owners of the name = 2   (us + the job on the stack)
  Pop returned job "still-referenced"; owners while the caller holds it = 2
  after the caller is done, owners = 1   <- len=0 cap=4, nothing held past len

  Go needs the zeroing line, or the popped job stays reachable for as long
  as the stack lives (lesson 04, example 3). Here the Vec gave up ownership
  at the pop, so the count falls as soon as the caller lets go of the job,
  with the capacity still allocated. For a Stack<Rc<Request>> in a server
  that is the whole difference between a leak and no leak -- for free.

the cost of the wrapper, measured against a bare Vec:

  bare Vec push/pop x1000               769 ns
  Stack<usize> push/pop x1000           765 ns  0.99x

  that is -0.00 ns of overhead per operation, over 2000 ops.

  do NOT assume the Go answer here. Go's wrapper cost was real because its
  generic methods were not inlining, plus Pop's extra zeroing write. In this
  -O build the methods are inlined (the release assembly has no call to
  Stack::push or Stack::pop) and Pop has no zeroing to do, so there is no
  call and no extra write to pay for. Expect a ratio near 1.0x that wobbles
  from run to run in both directions, and an overhead well under 0.1 ns per
  op either way: noise, not signal.

  so the wrapper is a performance non-question in Rust, and the choice is
  about the API. A bare Vec inside one function is fine and obvious. The type
  earns its place when the stack crosses a function or module boundary:
  it exposes push/pop/peek and NOT insert(0, x), remove(i) or indexing, so
  a caller cannot use it as anything but a stack. (Vec's own pop already
  returns Option<T>, so the contract is the same either way.)
```

> **Go vs Rust:** The stack output matches Go line for line (`Option<T>` for `(T, bool)`, `Peek` returns `Option<&T>`). `#[derive(Default)]` would demand `T: Default`, so the default empty stack is a hand-written `impl`. The zeroing line is gone: `Vec::pop` moves the value out, and the `Rc` owner count falls back to 1 as soon as the popped job is dropped, with `cap` still 4. The benchmark flips: Go measured 6.03x for the wrapper, but across about fifteen runs here `Stack<usize>` was 0.98x to 1.18x of the bare `Vec` (usually 0.99x to 1.10x), i.e. noise, and the release assembly has no call to `Stack::push`/`pop` (inlined).

**Complexity:** every operation Θ(1) · the wrapper costs ~1.6 ns/op, which is cheap but *not* free

---

## 3. Balanced brackets

`🟢 easy` · *The giveaway phrase*

The canonical "this problem IS a stack" example. The tell is the phrase **most recently
opened** — and a counter cannot do it.

**Steps:**

1. Push openers, and require a closer to match the top.
2. Report *where* and *why*, not just a bool.
3. Find the input a counter accepts and a stack rejects.

```go
package main

import "fmt"

// Some problems do not USE a stack -- they ARE a stack. Bracket matching is the
// clearest case, and the giveaway is the phrase "most recently opened".
//
// Whenever a problem says "the most recent unmatched X", the answer is a stack,
// because that is the only structure whose defining property is "most recent".

type result struct {
	ok     bool
	reason string
	pos    int
}

var pairs = map[rune]rune{')': '(', ']': '[', '}': '{'}

// balanced reports whether every bracket is matched and correctly nested, and
// WHERE it first goes wrong -- an error message with a position is worth far
// more than a bool.
func balanced(s string) result {
	type frame struct {
		ch  rune
		pos int
	}
	var stack []frame

	for i, ch := range s {
		switch ch {
		case '(', '[', '{':
			stack = append(stack, frame{ch, i})

		case ')', ']', '}':
			if len(stack) == 0 {
				return result{false, fmt.Sprintf("closing %q with nothing open", ch), i}
			}
			top := stack[len(stack)-1]
			if top.ch != pairs[ch] {
				return result{false,
					fmt.Sprintf("closing %q but %q was opened at %d", ch, top.ch, top.pos), i}
			}
			stack = stack[:len(stack)-1]
		}
	}

	if len(stack) > 0 {
		top := stack[len(stack)-1]
		return result{false, fmt.Sprintf("%q was never closed", top.ch), top.pos}
	}
	return result{ok: true}
}

// depth returns the maximum nesting depth -- the stack's high-water mark, which
// is the same number as the recursion depth a recursive parser would need
// (lesson 05).
func depth(s string) int {
	cur, max := 0, 0
	for _, ch := range s {
		switch ch {
		case '(', '[', '{':
			cur++
			if cur > max {
				max = cur
			}
		case ')', ']', '}':
			cur--
		}
	}
	return max
}

func main() {
	fmt.Println("the stack holds every bracket that is OPEN but not yet closed.")
	fmt.Println("A closer must match the TOP -- the most recently opened one.")
	fmt.Println()

	inputs := []string{
		"()",
		"()[]{}",
		"([{}])",
		"a(b[c]{d})e",
		"",
		"(",
		")",
		"(]",
		"([)]",
		"((())",
	}

	fmt.Printf("  %-16s %-8s %-10s %s\n", "input", "ok", "at", "reason")
	for _, in := range inputs {
		r := balanced(in)
		if r.ok {
			fmt.Printf("  %-16q %-8v %-10s\n", in, true, "-")
			continue
		}
		fmt.Printf("  %-16q %-8v %-10d %s\n", in, false, r.pos, r.reason)
	}

	fmt.Println()
	fmt.Println("the third failing case is the one a counter cannot catch:")
	fmt.Println()
	fmt.Println("  \"([)]\" has two opens and two closes, perfectly balanced by COUNT.")
	fmt.Println("  It is still wrong, because the nesting interleaves. Only a stack")
	fmt.Println("  knows WHICH bracket is currently innermost.")
	fmt.Println()
	fmt.Println("  a counter is enough for ONE bracket type. The moment there are")
	fmt.Println("  several, you need the order, and order means a stack.")

	fmt.Println()
	fmt.Println("the stack's high-water mark is the nesting depth:")
	fmt.Println()
	for _, in := range []string{"()", "((()))", "(()(()))", "a(b)c"} {
		fmt.Printf("  %-12q depth %d\n", in, depth(in))
	}
	fmt.Println()
	fmt.Println("  which is exactly the recursion depth a recursive-descent parser")
	fmt.Println("  would use for the same input (lesson 05, example 16) -- because an")
	fmt.Println("  explicit stack and the call stack are the same structure.")

	fmt.Println()
	fmt.Println("the pattern to recognise, stated once:")
	fmt.Println()
	fmt.Println("  \"the most recently opened / seen / pushed X\"   -> stack")
	fmt.Println("  \"the oldest outstanding X\"                     -> queue (lesson 09)")
	fmt.Println("  \"the smallest outstanding X\"                   -> heap  (lesson 20)")
	fmt.Println()
	fmt.Println("that one line resolves most 'which structure?' questions on sight.")

	fmt.Println()
	fmt.Println("real uses of this exact algorithm: matching braces in an editor,")
	fmt.Println("validating JSON/XML nesting, checking that every opened HTML tag is")
	fmt.Println("closed, and the tokenizer stage of every parser you will ever write.")
}
```

**Output:**

```
the stack holds every bracket that is OPEN but not yet closed.
A closer must match the TOP -- the most recently opened one.

  input            ok       at         reason
  "()"             true     -         
  "()[]{}"         true     -         
  "([{}])"         true     -         
  "a(b[c]{d})e"    true     -         
  ""               true     -         
  "("              false    0          '(' was never closed
  ")"              false    0          closing ')' with nothing open
  "(]"             false    1          closing ']' but '(' was opened at 0
  "([)]"           false    2          closing ')' but '[' was opened at 1
  "((())"          false    0          '(' was never closed

the third failing case is the one a counter cannot catch:

  "([)]" has two opens and two closes, perfectly balanced by COUNT.
  It is still wrong, because the nesting interleaves. Only a stack
  knows WHICH bracket is currently innermost.

  a counter is enough for ONE bracket type. The moment there are
  several, you need the order, and order means a stack.

the stack's high-water mark is the nesting depth:

  "()"         depth 1
  "((()))"     depth 3
  "(()(()))"   depth 3
  "a(b)c"      depth 1

  which is exactly the recursion depth a recursive-descent parser
  would use for the same input (lesson 05, example 16) -- because an
  explicit stack and the call stack are the same structure.

the pattern to recognise, stated once:

  "the most recently opened / seen / pushed X"   -> stack
  "the oldest outstanding X"                     -> queue (lesson 09)
  "the smallest outstanding X"                   -> heap  (lesson 20)

that one line resolves most 'which structure?' questions on sight.

real uses of this exact algorithm: matching braces in an editor,
validating JSON/XML nesting, checking that every opened HTML tag is
closed, and the tokenizer stage of every parser you will ever write.
```

**Rust:**

```rust
// Some problems do not USE a stack -- they ARE a stack. Bracket matching is the
// clearest case, and the giveaway is the phrase "most recently opened".
//
// Whenever a problem says "the most recent unmatched X", the answer is a stack,
// because that is the only structure whose defining property is "most recent".

/// Where and why the input first goes wrong. Go returns a struct with an `ok`
/// flag; Rust returns `Result<(), Failure>`, so success carries nothing and
/// failure cannot be mistaken for success.
struct Failure {
    pos: usize,
    reason: String,
}

const PAIRS: [(char, char); 3] = [(')', '('), (']', '['), ('}', '{')];

/// The opener that a closer must match (Go: `pairs[ch]`).
fn opener(closer: char) -> Option<char> {
    PAIRS
        .iter()
        .find(|&&(c, _)| c == closer)
        .map(|&(_, open)| open)
}

/// balanced reports whether every bracket is matched and correctly nested, and
/// WHERE it first goes wrong -- an error message with a position is worth far
/// more than a bool.
fn balanced(s: &str) -> Result<(), Failure> {
    // (bracket, byte offset where it was opened). char_indices gives byte
    // offsets, like Go's `range` over a string.
    let mut stack: Vec<(char, usize)> = Vec::new();

    for (i, ch) in s.char_indices() {
        match ch {
            '(' | '[' | '{' => stack.push((ch, i)),

            ')' | ']' | '}' => {
                let Some((open, at)) = stack.pop() else {
                    return Err(Failure {
                        pos: i,
                        reason: format!("closing {ch:?} with nothing open"),
                    });
                };
                if Some(open) != opener(ch) {
                    return Err(Failure {
                        pos: i,
                        reason: format!("closing {ch:?} but {open:?} was opened at {at}"),
                    });
                }
            }
            _ => {}
        }
    }

    match stack.last() {
        Some(&(open, at)) => Err(Failure {
            pos: at,
            reason: format!("{open:?} was never closed"),
        }),
        None => Ok(()),
    }
}

/// depth returns the maximum nesting depth -- the stack's high-water mark, which
/// is the same number as the recursion depth a recursive parser would need
/// (lesson 05).
fn depth(s: &str) -> i64 {
    // i64, not usize: like Go's int, the counter may dip below zero on an
    // unmatched closer instead of underflowing
    let (mut cur, mut max) = (0i64, 0i64);
    for ch in s.chars() {
        match ch {
            '(' | '[' | '{' => {
                cur += 1;
                max = max.max(cur);
            }
            ')' | ']' | '}' => cur -= 1,
            _ => {}
        }
    }
    max
}

fn main() {
    println!("the stack holds every bracket that is OPEN but not yet closed.");
    println!("A closer must match the TOP -- the most recently opened one.");
    println!();

    let inputs = [
        "()",
        "()[]{}",
        "([{}])",
        "a(b[c]{d})e",
        "",
        "(",
        ")",
        "(]",
        "([)]",
        "((())",
    ];

    println!("  {:<16} {:<8} {:<10} reason", "input", "ok", "at");
    for input in inputs {
        let quoted = format!("{input:?}");
        match balanced(input) {
            Ok(()) => println!("  {quoted:<16} {:<8} {:<10}", true, "-"),
            Err(f) => println!("  {quoted:<16} {:<8} {:<10} {}", false, f.pos, f.reason),
        }
    }

    println!();
    println!("the third failing case is the one a counter cannot catch:");
    println!();
    println!("  \"([)]\" has two opens and two closes, perfectly balanced by COUNT.");
    println!("  It is still wrong, because the nesting interleaves. Only a stack");
    println!("  knows WHICH bracket is currently innermost.");
    println!();
    println!("  a counter is enough for ONE bracket type. The moment there are");
    println!("  several, you need the order, and order means a stack.");

    println!();
    println!("the stack's high-water mark is the nesting depth:");
    println!();
    for input in ["()", "((()))", "(()(()))", "a(b)c"] {
        println!("  {:<12} depth {}", format!("{input:?}"), depth(input));
    }
    println!();
    println!("  which is exactly the recursion depth a recursive-descent parser");
    println!("  would use for the same input (lesson 05, example 16) -- because an");
    println!("  explicit stack and the call stack are the same structure.");

    println!();
    println!("the pattern to recognise, stated once:");
    println!();
    println!("  \"the most recently opened / seen / pushed X\"   -> stack");
    println!("  \"the oldest outstanding X\"                     -> queue (lesson 09)");
    println!("  \"the smallest outstanding X\"                   -> heap  (lesson 20)");
    println!();
    println!("that one line resolves most 'which structure?' questions on sight.");

    println!();
    println!("real uses of this exact algorithm: matching braces in an editor,");
    println!("validating JSON/XML nesting, checking that every opened HTML tag is");
    println!("closed, and the tokenizer stage of every parser you will ever write.");
}
```

**Rust output:**

```
the stack holds every bracket that is OPEN but not yet closed.
A closer must match the TOP -- the most recently opened one.

  input            ok       at         reason
  "()"             true     -         
  "()[]{}"         true     -         
  "([{}])"         true     -         
  "a(b[c]{d})e"    true     -         
  ""               true     -         
  "("              false    0          '(' was never closed
  ")"              false    0          closing ')' with nothing open
  "(]"             false    1          closing ']' but '(' was opened at 0
  "([)]"           false    2          closing ')' but '[' was opened at 1
  "((())"          false    0          '(' was never closed

the third failing case is the one a counter cannot catch:

  "([)]" has two opens and two closes, perfectly balanced by COUNT.
  It is still wrong, because the nesting interleaves. Only a stack
  knows WHICH bracket is currently innermost.

  a counter is enough for ONE bracket type. The moment there are
  several, you need the order, and order means a stack.

the stack's high-water mark is the nesting depth:

  "()"         depth 1
  "((()))"     depth 3
  "(()(()))"   depth 3
  "a(b)c"      depth 1

  which is exactly the recursion depth a recursive-descent parser
  would use for the same input (lesson 05, example 16) -- because an
  explicit stack and the call stack are the same structure.

the pattern to recognise, stated once:

  "the most recently opened / seen / pushed X"   -> stack
  "the oldest outstanding X"                     -> queue (lesson 09)
  "the smallest outstanding X"                   -> heap  (lesson 20)

that one line resolves most 'which structure?' questions on sight.

real uses of this exact algorithm: matching braces in an editor,
validating JSON/XML nesting, checking that every opened HTML tag is
closed, and the tokenizer stage of every parser you will ever write.
```

> **Go vs Rust:** The output is identical to Go, character for character. Go's `result{ok, reason, pos}` becomes `Result<(), Failure>`, so success carries nothing, and `pop()` returning an `Option` replaces the `len(stack) == 0` guard (`let Some(..) = stack.pop() else { .. }`). Both languages report byte offsets (`char_indices` here, `range` over a string in Go), and the depth counter is an `i64` rather than a `usize` so it can dip below zero on an unmatched closer, as Go's `int` does.

**Complexity:** Θ(n) time, Θ(depth) space — and that depth is exactly the recursion depth a recursive parser would use

---

## 4. RPN evaluation

`🟢 easy` · *No parentheses, no precedence*

Reverse Polish notation puts the operator after its operands, so evaluating it is a stack and
nothing else — twenty lines with no grammar and no recursion.

**Steps:**

1. Push numbers; on an operator pop two, apply, push the result.
2. Trace the stack after every token.
3. Then check the operand order with `-` and `/`, not `+` and `*`.

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

// Reverse Polish notation puts the operator AFTER its operands: "3 4 +".
// No parentheses, no precedence rules, and evaluating it is a stack and
// nothing else -- twenty lines with no grammar and no recursion.
//
// That is not a coincidence. RPN is what an expression looks like once the
// structure is already resolved; example 7 converts infix to it.

type evalError struct {
	msg string
	tok int
}

func (e *evalError) Error() string { return fmt.Sprintf("token %d: %s", e.tok, e.msg) }

// evalRPN: push numbers, and on an operator pop TWO, apply, push the result.
func evalRPN(tokens []string) (float64, error) {
	var stack []float64

	for i, tok := range tokens {
		switch tok {
		case "+", "-", "*", "/":
			if len(stack) < 2 {
				return 0, &evalError{fmt.Sprintf("operator %q needs 2 operands, stack has %d", tok, len(stack)), i}
			}
			// ORDER MATTERS: the second pop is the LEFT operand.
			b := stack[len(stack)-1]
			a := stack[len(stack)-2]
			stack = stack[:len(stack)-2]

			var v float64
			switch tok {
			case "+":
				v = a + b
			case "-":
				v = a - b
			case "*":
				v = a * b
			case "/":
				if b == 0 {
					return 0, &evalError{"division by zero", i}
				}
				v = a / b
			}
			stack = append(stack, v)

		default:
			f, err := strconv.ParseFloat(tok, 64)
			if err != nil {
				return 0, &evalError{fmt.Sprintf("not a number: %q", tok), i}
			}
			stack = append(stack, f)
		}
	}

	if len(stack) != 1 {
		return 0, &evalError{fmt.Sprintf("expression left %d values on the stack, want 1", len(stack)), len(tokens)}
	}
	return stack[0], nil
}

// trace shows the stack after every token, which makes the algorithm obvious.
func trace(expr string) {
	tokens := strings.Fields(expr)
	var stack []float64
	fmt.Printf("  %-8s %s\n", "token", "stack after")
	for _, tok := range tokens {
		switch tok {
		case "+", "-", "*", "/":
			b := stack[len(stack)-1]
			a := stack[len(stack)-2]
			stack = stack[:len(stack)-2]
			var v float64
			switch tok {
			case "+":
				v = a + b
			case "-":
				v = a - b
			case "*":
				v = a * b
			case "/":
				v = a / b
			}
			stack = append(stack, v)
		default:
			f, _ := strconv.ParseFloat(tok, 64)
			stack = append(stack, f)
		}
		fmt.Printf("  %-8s %v\n", tok, stack)
	}
}

func main() {
	fmt.Println("evaluating \"3 4 + 2 *\"  (which is (3+4)*2):")
	fmt.Println()
	trace("3 4 + 2 *")

	fmt.Println()
	fmt.Println("more, including the ones that catch a wrong operand order:")
	fmt.Println()
	fmt.Printf("  %-28s %-12s %s\n", "expression", "result", "infix")
	for _, c := range []struct{ expr, infix string }{
		{"3 4 +", "3 + 4"},
		{"3 4 + 2 *", "(3 + 4) * 2"},
		{"3 4 2 * +", "3 + 4 * 2"},
		{"10 2 -", "10 - 2"},
		{"2 10 -", "2 - 10"},
		{"100 5 / 2 /", "100 / 5 / 2"},
		{"5 1 2 + 4 * + 3 -", "5 + ((1+2)*4) - 3"},
	} {
		v, err := evalRPN(strings.Fields(c.expr))
		if err != nil {
			fmt.Printf("  %-28s %-12s %s\n", c.expr, "ERR", err)
			continue
		}
		fmt.Printf("  %-28s %-12g %s\n", c.expr, v, c.infix)
	}
	fmt.Println()
	fmt.Println("  rows 4 and 5 are why the operand order matters: the SECOND pop is")
	fmt.Println("  the left operand. Swap them and subtraction and division silently")
	fmt.Println("  give the wrong sign -- and addition still looks fine, so a test")
	fmt.Println("  suite of only + and * would never notice.")

	fmt.Println()
	fmt.Println("malformed input is rejected, not crashed on:")
	fmt.Println()
	for _, expr := range []string{"3 +", "+", "3 4", "3 0 /", "3 x +", ""} {
		_, err := evalRPN(strings.Fields(expr))
		fmt.Printf("  %-12q -> %v\n", expr, err)
	}

	fmt.Println()
	fmt.Println("why RPN needs no parentheses and no precedence:")
	fmt.Println()
	fmt.Println("  the ORDER of the tokens already encodes the tree. \"3 4 2 * +\"")
	fmt.Println("  can only mean 3 + (4*2), because the * consumes the two values")
	fmt.Println("  immediately below it. There is nothing left to disambiguate.")
	fmt.Println()
	fmt.Println("  so evaluation is one linear pass with a stack, and no grammar --")
	fmt.Println("  which is exactly why compilers and virtual machines use a")
	fmt.Println("  stack-based intermediate form. Example 7 does the conversion.")
}
```

**Output:**

```
evaluating "3 4 + 2 *"  (which is (3+4)*2):

  token    stack after
  3        [3]
  4        [3 4]
  +        [7]
  2        [7 2]
  *        [14]

more, including the ones that catch a wrong operand order:

  expression                   result       infix
  3 4 +                        7            3 + 4
  3 4 + 2 *                    14           (3 + 4) * 2
  3 4 2 * +                    11           3 + 4 * 2
  10 2 -                       8            10 - 2
  2 10 -                       -8           2 - 10
  100 5 / 2 /                  10           100 / 5 / 2
  5 1 2 + 4 * + 3 -            14           5 + ((1+2)*4) - 3

  rows 4 and 5 are why the operand order matters: the SECOND pop is
  the left operand. Swap them and subtraction and division silently
  give the wrong sign -- and addition still looks fine, so a test
  suite of only + and * would never notice.

malformed input is rejected, not crashed on:

  "3 +"        -> token 1: operator "+" needs 2 operands, stack has 1
  "+"          -> token 0: operator "+" needs 2 operands, stack has 0
  "3 4"        -> token 2: expression left 2 values on the stack, want 1
  "3 0 /"      -> token 2: division by zero
  "3 x +"      -> token 1: not a number: "x"
  ""           -> token 0: expression left 0 values on the stack, want 1

why RPN needs no parentheses and no precedence:

  the ORDER of the tokens already encodes the tree. "3 4 2 * +"
  can only mean 3 + (4*2), because the * consumes the two values
  immediately below it. There is nothing left to disambiguate.

  so evaluation is one linear pass with a stack, and no grammar --
  which is exactly why compilers and virtual machines use a
  stack-based intermediate form. Example 7 does the conversion.
```

**Rust:**

```rust
use std::fmt;

// Reverse Polish notation puts the operator AFTER its operands: "3 4 +".
// No parentheses, no precedence rules, and evaluating it is a stack and
// nothing else -- twenty lines with no grammar and no recursion.
//
// That is not a coincidence. RPN is what an expression looks like once the
// structure is already resolved; example 7 converts infix to it.

struct EvalError {
    msg: String,
    tok: usize,
}

impl fmt::Display for EvalError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "token {}: {}", self.tok, self.msg)
    }
}

/// Apply one binary operator. The only failure that input can cause is division by zero.
fn apply(op: &str, a: f64, b: f64) -> Result<f64, &'static str> {
    match op {
        "+" => Ok(a + b),
        "-" => Ok(a - b),
        "*" => Ok(a * b),
        "/" if b == 0.0 => Err("division by zero"),
        "/" => Ok(a / b),
        _ => Err("unknown operator"),
    }
}

/// eval_rpn: push numbers, and on an operator pop TWO, apply, push the result.
fn eval_rpn(tokens: &[&str]) -> Result<f64, EvalError> {
    let mut stack: Vec<f64> = Vec::new();

    for (i, &tok) in tokens.iter().enumerate() {
        let err = |msg: String| EvalError { msg, tok: i };
        match tok {
            "+" | "-" | "*" | "/" => {
                let have = stack.len();
                // ORDER MATTERS: the first pop is the RIGHT operand, so the
                // second pop is the LEFT one. The tuple evaluates left to right.
                let (Some(b), Some(a)) = (stack.pop(), stack.pop()) else {
                    return Err(err(format!(
                        "operator {tok:?} needs 2 operands, stack has {have}"
                    )));
                };
                stack.push(apply(tok, a, b).map_err(|m| err(m.to_string()))?);
            }

            _ => {
                let Ok(f) = tok.parse::<f64>() else {
                    return Err(err(format!("not a number: {tok:?}")));
                };
                stack.push(f);
            }
        }
    }

    match stack.as_slice() {
        [v] => Ok(*v),
        _ => Err(EvalError {
            msg: format!(
                "expression left {} values on the stack, want 1",
                stack.len()
            ),
            tok: tokens.len(),
        }),
    }
}

/// Go's `%v` of a []float64: [3 4]
fn show(xs: &[f64]) -> String {
    let parts: Vec<String> = xs.iter().map(f64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

/// trace shows the stack after every token, which makes the algorithm obvious.
fn trace(expr: &str) {
    let mut stack: Vec<f64> = Vec::new();
    println!("  {:<8} stack after", "token");
    for tok in expr.split_whitespace() {
        match tok {
            "+" | "-" | "*" | "/" => {
                let (Some(b), Some(a)) = (stack.pop(), stack.pop()) else {
                    panic!("trace is only called on well-formed input");
                };
                stack.push(apply(tok, a, b).expect("no division by zero in the traced input"));
            }
            _ => stack.push(tok.parse().expect("trace is only called on numbers")),
        }
        println!("  {tok:<8} {}", show(&stack));
    }
}

fn main() {
    println!("evaluating \"3 4 + 2 *\"  (which is (3+4)*2):");
    println!();
    trace("3 4 + 2 *");

    println!();
    println!("more, including the ones that catch a wrong operand order:");
    println!();
    println!("  {:<28} {:<12} infix", "expression", "result");
    for (expr, infix) in [
        ("3 4 +", "3 + 4"),
        ("3 4 + 2 *", "(3 + 4) * 2"),
        ("3 4 2 * +", "3 + 4 * 2"),
        ("10 2 -", "10 - 2"),
        ("2 10 -", "2 - 10"),
        ("100 5 / 2 /", "100 / 5 / 2"),
        ("5 1 2 + 4 * + 3 -", "5 + ((1+2)*4) - 3"),
    ] {
        let tokens: Vec<&str> = expr.split_whitespace().collect();
        match eval_rpn(&tokens) {
            Ok(v) => println!("  {expr:<28} {v:<12} {infix}"),
            Err(e) => println!("  {expr:<28} {:<12} {e}", "ERR"),
        }
    }
    println!();
    println!("  rows 4 and 5 are why the operand order matters: the SECOND pop is");
    println!("  the left operand. Swap them and subtraction and division silently");
    println!("  give the wrong sign -- and addition still looks fine, so a test");
    println!("  suite of only + and * would never notice.");

    println!();
    println!("malformed input is rejected, not crashed on:");
    println!();
    for expr in ["3 +", "+", "3 4", "3 0 /", "3 x +", ""] {
        let tokens: Vec<&str> = expr.split_whitespace().collect();
        let outcome = match eval_rpn(&tokens) {
            Ok(v) => v.to_string(),
            Err(e) => e.to_string(),
        };
        println!("  {:<12} -> {outcome}", format!("{expr:?}"));
    }

    println!();
    println!("why RPN needs no parentheses and no precedence:");
    println!();
    println!("  the ORDER of the tokens already encodes the tree. \"3 4 2 * +\"");
    println!("  can only mean 3 + (4*2), because the * consumes the two values");
    println!("  immediately below it. There is nothing left to disambiguate.");
    println!();
    println!("  so evaluation is one linear pass with a stack, and no grammar --");
    println!("  which is exactly why compilers and virtual machines use a");
    println!("  stack-based intermediate form. Example 7 does the conversion.");
}
```

**Rust output:**

```
evaluating "3 4 + 2 *"  (which is (3+4)*2):

  token    stack after
  3        [3]
  4        [3 4]
  +        [7]
  2        [7 2]
  *        [14]

more, including the ones that catch a wrong operand order:

  expression                   result       infix
  3 4 +                        7            3 + 4
  3 4 + 2 *                    14           (3 + 4) * 2
  3 4 2 * +                    11           3 + 4 * 2
  10 2 -                       8            10 - 2
  2 10 -                       -8           2 - 10
  100 5 / 2 /                  10           100 / 5 / 2
  5 1 2 + 4 * + 3 -            14           5 + ((1+2)*4) - 3

  rows 4 and 5 are why the operand order matters: the SECOND pop is
  the left operand. Swap them and subtraction and division silently
  give the wrong sign -- and addition still looks fine, so a test
  suite of only + and * would never notice.

malformed input is rejected, not crashed on:

  "3 +"        -> token 1: operator "+" needs 2 operands, stack has 1
  "+"          -> token 0: operator "+" needs 2 operands, stack has 0
  "3 4"        -> token 2: expression left 2 values on the stack, want 1
  "3 0 /"      -> token 2: division by zero
  "3 x +"      -> token 1: not a number: "x"
  ""           -> token 0: expression left 0 values on the stack, want 1

why RPN needs no parentheses and no precedence:

  the ORDER of the tokens already encodes the tree. "3 4 2 * +"
  can only mean 3 + (4*2), because the * consumes the two values
  immediately below it. There is nothing left to disambiguate.

  so evaluation is one linear pass with a stack, and no grammar --
  which is exactly why compilers and virtual machines use a
  stack-based intermediate form. Example 7 does the conversion.
```

> **Go vs Rust:** The output is identical to Go, character for character (f64 `Display` prints `7` and `-8` where Go's `%g` does). The operand order is visible in the types: `let (Some(b), Some(a)) = (stack.pop(), stack.pop())` pops the right operand first because a tuple evaluates left to right, and the same pattern doubles as the "needs 2 operands" check, so there is no manual `len(stack) < 2` test (the length is read beforehand only to report it). Go's `*evalError` becomes a plain `Result<f64, EvalError>`, and the final `len(stack) != 1` test is the slice pattern `[v]`.

**Complexity:** Θ(n) time, Θ(depth) space · the token ORDER already encodes the tree, which is why compilers use a stack-based intermediate form

---

## 5. Undo/redo

`🟢 easy` · *Two stacks and one rule*

The construction is easy; the **rule** is what people get wrong. A new action must clear the
redo stack, or you can redo your way into a state that never existed.

**Steps:**

1. Store commands and their inverses, not snapshots.
2. Undo pops one stack and pushes the other.
3. Then do a new action after an undo and check what happens to redo.

```go
package main

import (
	"fmt"
	"strings"
)

// Undo/redo is two stacks and one rule, and it is worth seeing because the RULE
// is the part people get wrong.
//
//	undo stack   commands already applied, newest on top
//	redo stack   commands undone, newest on top
//
//	do(cmd)   apply, push onto undo, and CLEAR redo
//	undo()    pop undo, invert, push onto redo
//	redo()    pop redo, apply, push onto undo
//
// Clearing redo on a new action is the rule. Skip it and you can redo your way
// into a document state that never existed.

type command interface {
	apply(*strings.Builder)
	invert() command
	String() string
}

type insertCmd struct {
	at   int
	text string
}

func (c insertCmd) apply(b *strings.Builder) {
	s := b.String()
	b.Reset()
	b.WriteString(s[:c.at] + c.text + s[c.at:])
}
func (c insertCmd) invert() command { return deleteCmd{c.at, c.text} }
func (c insertCmd) String() string  { return fmt.Sprintf("insert %q at %d", c.text, c.at) }

type deleteCmd struct {
	at   int
	text string
}

func (c deleteCmd) apply(b *strings.Builder) {
	s := b.String()
	b.Reset()
	b.WriteString(s[:c.at] + s[c.at+len(c.text):])
}
func (c deleteCmd) invert() command { return insertCmd{c.at, c.text} }
func (c deleteCmd) String() string  { return fmt.Sprintf("delete %q at %d", c.text, c.at) }

type editor struct {
	buf  strings.Builder
	undo []command
	redo []command
}

func (e *editor) Do(c command) {
	c.apply(&e.buf)
	e.undo = append(e.undo, c)
	e.redo = e.redo[:0] // THE RULE: a new action invalidates the redo history
}

func (e *editor) Undo() bool {
	if len(e.undo) == 0 {
		return false
	}
	c := e.undo[len(e.undo)-1]
	e.undo = e.undo[:len(e.undo)-1]
	c.invert().apply(&e.buf)
	e.redo = append(e.redo, c)
	return true
}

func (e *editor) Redo() bool {
	if len(e.redo) == 0 {
		return false
	}
	c := e.redo[len(e.redo)-1]
	e.redo = e.redo[:len(e.redo)-1]
	c.apply(&e.buf)
	e.undo = append(e.undo, c)
	return true
}

func (e *editor) show(label string) {
	fmt.Printf("  %-22s %-14q undo=%d redo=%d\n", label, e.buf.String(), len(e.undo), len(e.redo))
}

func main() {
	var e editor
	e.show("start")

	e.Do(insertCmd{0, "hello"})
	e.show(`Do insert "hello"`)
	e.Do(insertCmd{5, " world"})
	e.show(`Do insert " world"`)
	e.Do(deleteCmd{0, "hello"})
	e.show(`Do delete "hello"`)

	fmt.Println()
	e.Undo()
	e.show("Undo")
	e.Undo()
	e.show("Undo")

	fmt.Println()
	e.Redo()
	e.show("Redo")
	e.Redo()
	e.show("Redo")
	fmt.Printf("  %-22s %v\n", "Redo again ->", e.Redo())

	fmt.Println()
	fmt.Println("now THE RULE -- a new action after an undo clears the redo stack:")
	fmt.Println()
	var f editor
	f.Do(insertCmd{0, "abc"})
	f.Do(insertCmd{3, "def"})
	f.show("two inserts")
	f.Undo()
	f.show("Undo")
	f.Do(insertCmd{3, "XYZ"})
	f.show("Do a NEW action")
	fmt.Printf("  %-22s %v   <- the old redo is gone, correctly\n", "Redo ->", f.Redo())

	fmt.Println()
	fmt.Println("  without that clear, redo would replay \"def\" on top of \"abcXYZ\"")
	fmt.Println("  and produce \"abcXYZdef\" -- a state the user never created. Editors")
	fmt.Println("  that get this wrong corrupt documents.")

	fmt.Println()
	fmt.Println("why two STACKS and not two queues or a list:")
	fmt.Println()
	fmt.Println("  undo always means 'the most recent thing I did' -- the definition")
	fmt.Println("  of LIFO. Redo means 'the most recent thing I undid'. Both are")
	fmt.Println("  most-recent-first, so both are stacks (example 3's rule).")
	fmt.Println()
	fmt.Println("  and note what each stack stores: the COMMAND, not the document.")
	fmt.Println("  Storing snapshots would be Theta(document) per step; storing the")
	fmt.Println("  command plus its inverse is Theta(edit). That is the command")
	fmt.Println("  pattern, and it is what makes undo affordable at all.")

	fmt.Println()
	fmt.Println("the one design decision left: bounding the undo stack. A long session")
	fmt.Println("grows it without limit, so real editors cap it -- which needs removal")
	fmt.Println("from the BOTTOM as well as the top, and that is a deque (lesson 09).")
}
```

**Output:**

```
  start                  ""             undo=0 redo=0
  Do insert "hello"      "hello"        undo=1 redo=0
  Do insert " world"     "hello world"  undo=2 redo=0
  Do delete "hello"      " world"       undo=3 redo=0

  Undo                   "hello world"  undo=2 redo=1
  Undo                   "hello"        undo=1 redo=2

  Redo                   "hello world"  undo=2 redo=1
  Redo                   " world"       undo=3 redo=0
  Redo again ->          false

now THE RULE -- a new action after an undo clears the redo stack:

  two inserts            "abcdef"       undo=2 redo=0
  Undo                   "abc"          undo=1 redo=1
  Do a NEW action        "abcXYZ"       undo=2 redo=0
  Redo ->                false   <- the old redo is gone, correctly

  without that clear, redo would replay "def" on top of "abcXYZ"
  and produce "abcXYZdef" -- a state the user never created. Editors
  that get this wrong corrupt documents.

why two STACKS and not two queues or a list:

  undo always means 'the most recent thing I did' -- the definition
  of LIFO. Redo means 'the most recent thing I undid'. Both are
  most-recent-first, so both are stacks (example 3's rule).

  and note what each stack stores: the COMMAND, not the document.
  Storing snapshots would be Theta(document) per step; storing the
  command plus its inverse is Theta(edit). That is the command
  pattern, and it is what makes undo affordable at all.

the one design decision left: bounding the undo stack. A long session
grows it without limit, so real editors cap it -- which needs removal
from the BOTTOM as well as the top, and that is a deque (lesson 09).
```

**Rust:**

```rust
// Undo/redo is two stacks and one rule, and it is worth seeing because the RULE
// is the part people get wrong.
//
//	undo stack   commands already applied, newest on top
//	redo stack   commands undone, newest on top
//
//	perform(cmd)  apply, push onto undo, and CLEAR redo
//	undo()        pop undo, invert, push onto redo
//	redo()        pop redo, apply, push onto undo
//
// Clearing redo on a new action is the rule. Skip it and you can redo your way
// into a document state that never existed.

/// Go uses an interface with apply/invert/String. The set of commands here is
/// closed, so Rust's natural spelling is an enum: no boxing, and a `match` that
/// the compiler checks for completeness. (`Box<dyn Command>` is the open-set
/// alternative, when other crates must add their own commands.)
#[derive(Clone, Copy)]
enum Cmd {
    Insert { at: usize, text: &'static str },
    Delete { at: usize, text: &'static str },
}

impl Cmd {
    // Go's strings.Builder cannot be edited in place, so its apply rebuilds the
    // whole string. A String can: insert_str / replace_range edit it where it is.
    fn apply(self, buf: &mut String) {
        match self {
            Cmd::Insert { at, text } => buf.insert_str(at, text),
            Cmd::Delete { at, text } => buf.replace_range(at..at + text.len(), ""),
        }
    }

    // The inverse of an insert is the delete of the same text, and vice versa.
    fn invert(self) -> Cmd {
        match self {
            Cmd::Insert { at, text } => Cmd::Delete { at, text },
            Cmd::Delete { at, text } => Cmd::Insert { at, text },
        }
    }
}

#[derive(Default)]
struct Editor {
    buf: String,
    undo_stack: Vec<Cmd>,
    redo_stack: Vec<Cmd>,
}

impl Editor {
    // apply and remember, WITHOUT touching redo. Only `perform` and the
    // deliberately wrong demo below call this directly.
    fn record(&mut self, c: Cmd) {
        c.apply(&mut self.buf);
        self.undo_stack.push(c);
    }

    // Go calls this Do; `do` is a reserved word in Rust.
    fn perform(&mut self, c: Cmd) {
        self.record(c);
        self.redo_stack.clear(); // THE RULE: a new action invalidates the redo history
    }

    fn undo(&mut self) -> bool {
        let Some(c) = self.undo_stack.pop() else {
            return false;
        };
        c.invert().apply(&mut self.buf);
        self.redo_stack.push(c);
        true
    }

    fn redo(&mut self) -> bool {
        let Some(c) = self.redo_stack.pop() else {
            return false;
        };
        c.apply(&mut self.buf);
        self.undo_stack.push(c);
        true
    }

    fn show(&self, label: &str) {
        println!(
            "  {label:<22} {:<14} undo={} redo={}",
            format!("{:?}", self.buf),
            self.undo_stack.len(),
            self.redo_stack.len()
        );
    }
}

fn main() {
    let mut e = Editor::default();
    e.show("start");

    e.perform(Cmd::Insert {
        at: 0,
        text: "hello",
    });
    e.show(r#"Do insert "hello""#);
    e.perform(Cmd::Insert {
        at: 5,
        text: " world",
    });
    e.show(r#"Do insert " world""#);
    e.perform(Cmd::Delete {
        at: 0,
        text: "hello",
    });
    e.show(r#"Do delete "hello""#);

    println!();
    e.undo();
    e.show("Undo");
    e.undo();
    e.show("Undo");

    println!();
    e.redo();
    e.show("Redo");
    e.redo();
    e.show("Redo");
    println!("  {:<22} {}", "Redo again ->", e.redo());

    println!();
    println!("now THE RULE -- a new action after an undo clears the redo stack:");
    println!();
    let mut f = Editor::default();
    f.perform(Cmd::Insert { at: 0, text: "abc" });
    f.perform(Cmd::Insert { at: 3, text: "def" });
    f.show("two inserts");
    f.undo();
    f.show("Undo");
    f.perform(Cmd::Insert { at: 3, text: "XYZ" });
    f.show("Do a NEW action");
    println!(
        "  {:<22} {}   <- the old redo is gone, correctly",
        "Redo ->",
        f.redo()
    );

    println!();
    println!("and the same steps with the clear skipped (record, not perform):");
    println!();
    let mut g = Editor::default();
    g.perform(Cmd::Insert { at: 0, text: "abc" });
    g.perform(Cmd::Insert { at: 3, text: "def" });
    g.undo();
    g.record(Cmd::Insert { at: 3, text: "XYZ" });
    g.show("NEW action, no clear");
    g.redo();
    g.show("Redo (the stale one)");
    println!();
    println!("  the stale redo replays the insert of \"def\" at position 3 on top of");
    println!("  \"abcXYZ\" and produces \"abcdefXYZ\" -- a state the user never created,");
    println!("  with an undo stack that now believes they typed all three. Editors");
    println!("  that get this wrong corrupt documents.");

    println!();
    println!("why two STACKS and not two queues or a list:");
    println!();
    println!("  undo always means 'the most recent thing I did' -- the definition");
    println!("  of LIFO. Redo means 'the most recent thing I undid'. Both are");
    println!("  most-recent-first, so both are stacks (example 3's rule).");
    println!();
    println!("  and note what each stack stores: the COMMAND, not the document.");
    println!("  Storing snapshots would be Theta(document) per step; storing the");
    println!("  command plus its inverse is Theta(edit). That is the command");
    println!("  pattern, and it is what makes undo affordable at all.");

    println!();
    println!("the one design decision left: bounding the undo stack. A long session");
    println!("grows it without limit, so real editors cap it -- which needs removal");
    println!("from the BOTTOM as well as the top, and that is a deque (lesson 09).");
}
```

**Rust output:**

```
  start                  ""             undo=0 redo=0
  Do insert "hello"      "hello"        undo=1 redo=0
  Do insert " world"     "hello world"  undo=2 redo=0
  Do delete "hello"      " world"       undo=3 redo=0

  Undo                   "hello world"  undo=2 redo=1
  Undo                   "hello"        undo=1 redo=2

  Redo                   "hello world"  undo=2 redo=1
  Redo                   " world"       undo=3 redo=0
  Redo again ->          false

now THE RULE -- a new action after an undo clears the redo stack:

  two inserts            "abcdef"       undo=2 redo=0
  Undo                   "abc"          undo=1 redo=1
  Do a NEW action        "abcXYZ"       undo=2 redo=0
  Redo ->                false   <- the old redo is gone, correctly

and the same steps with the clear skipped (record, not perform):

  NEW action, no clear   "abcXYZ"       undo=2 redo=1
  Redo (the stale one)   "abcdefXYZ"    undo=3 redo=0

  the stale redo replays the insert of "def" at position 3 on top of
  "abcXYZ" and produces "abcdefXYZ" -- a state the user never created,
  with an undo stack that now believes they typed all three. Editors
  that get this wrong corrupt documents.

why two STACKS and not two queues or a list:

  undo always means 'the most recent thing I did' -- the definition
  of LIFO. Redo means 'the most recent thing I undid'. Both are
  most-recent-first, so both are stacks (example 3's rule).

  and note what each stack stores: the COMMAND, not the document.
  Storing snapshots would be Theta(document) per step; storing the
  command plus its inverse is Theta(edit). That is the command
  pattern, and it is what makes undo affordable at all.

the one design decision left: bounding the undo stack. A long session
grows it without limit, so real editors cap it -- which needs removal
from the BOTTOM as well as the top, and that is a deque (lesson 09).
```

> **Go vs Rust:** The traces match Go line for line up to the closing paragraph. Go's `command` interface becomes an enum (a closed set, no boxing), `Do` is `perform` because `do` is reserved, and `apply` edits the `String` in place where Go's `strings.Builder` has to be rebuilt. The Go text says that without the clear, redo "would produce `abcXYZdef`"; that is wrong. Running it (here with `record`, and also in the Go program with the clear removed) gives `abcdefXYZ`, because the stale command is `insert "def" at 3`, so the demonstration is now actually run instead of asserted.

**Complexity:** all three operations Θ(1) · Θ(edit) per step instead of Θ(document), which is what makes undo affordable

---

## 6. Iterative DFS, and the one-line switch to BFS

`🟢 easy` · *The container decides the algorithm*

For a graph traversal the stack is not a workaround — it IS the algorithm. Swap it for a queue
and you get a completely different traversal.

**Steps:**

1. Write DFS recursively, then with an explicit stack.
2. Note the two details: reverse push, and mark `seen` on **pop**.
3. Then change one line and watch it become BFS.

```go
package main

import "fmt"

// Lesson 05 converted recursion to a loop plus an explicit stack, and framed it
// as a way to survive deep input. Here it is again from the other side: for a
// graph traversal the stack is not a workaround, it IS the algorithm --
// and swapping it for a queue changes the algorithm entirely.

type graph map[string][]string

var g = graph{
	"A": {"B", "C"},
	"B": {"D", "E"},
	"C": {"F"},
	"D": {},
	"E": {"F"},
	"F": {},
}

// dfsRecursive: the call stack does the remembering.
func dfsRecursive(g graph, at string, seen map[string]bool, out *[]string) {
	if seen[at] {
		return
	}
	seen[at] = true
	*out = append(*out, at)
	for _, next := range g[at] {
		dfsRecursive(g, next, seen, out)
	}
}

// dfsIterative: you do the remembering. Note the two differences from the
// recursive version, both of which matter:
//
//  1. push the neighbours in REVERSE, because a stack reverses them again
//  2. mark seen when you POP, not when you push -- a node can be pushed several
//     times before it is first visited
func dfsIterative(g graph, start string) []string {
	var out []string
	seen := map[string]bool{}
	stack := []string{start}

	for len(stack) > 0 {
		at := stack[len(stack)-1]
		stack = stack[:len(stack)-1]

		if seen[at] {
			continue
		}
		seen[at] = true
		out = append(out, at)

		ns := g[at]
		for i := len(ns) - 1; i >= 0; i-- { // reverse push
			if !seen[ns[i]] {
				stack = append(stack, ns[i])
			}
		}
	}
	return out
}

// bfs is the SAME CODE with the stack replaced by a queue. That one substitution
// turns depth-first into breadth-first -- which is lesson 09's whole point.
func bfs(g graph, start string) []string {
	var out []string
	seen := map[string]bool{start: true}
	queue := []string{start}

	for len(queue) > 0 {
		at := queue[0]    // FRONT, not back
		queue = queue[1:] // the only difference
		out = append(out, at)

		for _, next := range g[at] {
			if !seen[next] {
				seen[next] = true
				queue = append(queue, next)
			}
		}
	}
	return out
}

// dfsWithPath keeps the path on the stack alongside the node, which is how you
// answer "how did I get here?" without a parent map.
func dfsWithPath(g graph, start, target string) []string {
	type frame struct {
		at   string
		path []string
	}
	seen := map[string]bool{}
	stack := []frame{{start, []string{start}}}

	for len(stack) > 0 {
		f := stack[len(stack)-1]
		stack = stack[:len(stack)-1]

		if f.at == target {
			return f.path
		}
		if seen[f.at] {
			continue
		}
		seen[f.at] = true

		ns := g[f.at]
		for i := len(ns) - 1; i >= 0; i-- {
			next := ns[i]
			if !seen[next] {
				// COPY the path: every branch needs its own (lesson 05, example 10)
				p := make([]string, len(f.path), len(f.path)+1)
				copy(p, f.path)
				stack = append(stack, frame{next, append(p, next)})
			}
		}
	}
	return nil
}

func main() {
	fmt.Println("the graph:")
	fmt.Println()
	fmt.Println("      A")
	fmt.Println("     / \\")
	fmt.Println("    B   C")
	fmt.Println("   / \\   \\")
	fmt.Println("  D   E - F")
	fmt.Println()

	var rec []string
	dfsRecursive(g, "A", map[string]bool{}, &rec)
	fmt.Printf("  DFS recursive  %v\n", rec)
	fmt.Printf("  DFS iterative  %v\n", dfsIterative(g, "A"))
	fmt.Printf("  BFS (a queue)  %v\n", bfs(g, "A"))

	fmt.Println()
	fmt.Println("the two versions of DFS agree -- but only because of two details:")
	fmt.Println()
	fmt.Println("  1. neighbours are pushed in REVERSE order. A stack hands them back")
	fmt.Println("     reversed, so pushing forwards would visit them right-to-left.")
	fmt.Println()
	fmt.Println("  2. `seen` is set when a node is POPPED, not when it is pushed. A")
	fmt.Println("     node reachable by two paths gets pushed twice before either is")
	fmt.Println("     visited, so the check has to happen at pop time.")
	fmt.Println()
	fmt.Println("     (BFS is the opposite: it marks on PUSH, because otherwise a node")
	fmt.Println("      can enter the queue many times and the shortest-path property")
	fmt.Println("      breaks. Same structure, different rule -- lesson 22.)")

	fmt.Println()
	fmt.Println("the substitution that changes the algorithm:")
	fmt.Println()
	fmt.Println("      stack:  at := xs[len(xs)-1]; xs = xs[:len(xs)-1]   -> DEPTH first")
	fmt.Println("      queue:  at := xs[0];         xs = xs[1:]            -> BREADTH first")
	fmt.Println()
	fmt.Println("  one line. Everything else is identical. That is worth internalising:")
	fmt.Println("  DFS and BFS are the same traversal with a different container, and")
	fmt.Println("  the container is what decides the order you meet the graph in.")

	fmt.Println()
	fmt.Printf("carrying the path on the stack: A -> F is %v\n", dfsWithPath(g, "A", "F"))
	fmt.Println()
	fmt.Println("  note the path is COPIED into each frame. Sharing one slice across")
	fmt.Println("  branches is lesson 05's example 10 bug, and it produces answers")
	fmt.Println("  that look plausible and are wrong.")

	fmt.Println()
	fmt.Println("why prefer the explicit stack here at all?")
	fmt.Println()
	fmt.Println("  the recursive DFS is shorter and clearer, and for a bounded graph")
	fmt.Println("  it is the right choice. The explicit version wins when the depth")
	fmt.Println("  can grow with the input -- a chain-shaped graph is n frames deep")
	fmt.Println("  (lesson 05, example 15), and an explicit stack lives on the heap.")
}
```

**Output:**

```
the graph:

      A
     / \
    B   C
   / \   \
  D   E - F

  DFS recursive  [A B D E F C]
  DFS iterative  [A B D E F C]
  BFS (a queue)  [A B C D E F]

the two versions of DFS agree -- but only because of two details:

  1. neighbours are pushed in REVERSE order. A stack hands them back
     reversed, so pushing forwards would visit them right-to-left.

  2. `seen` is set when a node is POPPED, not when it is pushed. A
     node reachable by two paths gets pushed twice before either is
     visited, so the check has to happen at pop time.

     (BFS is the opposite: it marks on PUSH, because otherwise a node
      can enter the queue many times and the shortest-path property
      breaks. Same structure, different rule -- lesson 22.)

the substitution that changes the algorithm:

      stack:  at := xs[len(xs)-1]; xs = xs[:len(xs)-1]   -> DEPTH first
      queue:  at := xs[0];         xs = xs[1:]            -> BREADTH first

  one line. Everything else is identical. That is worth internalising:
  DFS and BFS are the same traversal with a different container, and
  the container is what decides the order you meet the graph in.

carrying the path on the stack: A -> F is [A B E F]

  note the path is COPIED into each frame. Sharing one slice across
  branches is lesson 05's example 10 bug, and it produces answers
  that look plausible and are wrong.

why prefer the explicit stack here at all?

  the recursive DFS is shorter and clearer, and for a bounded graph
  it is the right choice. The explicit version wins when the depth
  can grow with the input -- a chain-shaped graph is n frames deep
  (lesson 05, example 15), and an explicit stack lives on the heap.
```

**Rust:**

```rust
use std::collections::{HashMap, HashSet, VecDeque};

// Lesson 05 converted recursion to a loop plus an explicit stack, and framed it
// as a way to survive deep input. Here it is again from the other side: for a
// graph traversal the stack is not a workaround, it IS the algorithm --
// and swapping it for a queue changes the algorithm entirely.

type Graph = HashMap<&'static str, Vec<&'static str>>;

fn graph() -> Graph {
    HashMap::from([
        ("A", vec!["B", "C"]),
        ("B", vec!["D", "E"]),
        ("C", vec!["F"]),
        ("D", vec![]),
        ("E", vec!["F"]),
        ("F", vec![]),
    ])
}

/// A node's neighbours in listed order; a missing node has none (Go: `g[at]` is nil).
fn neighbours<'a>(g: &'a Graph, at: &str) -> &'a [&'static str] {
    g.get(at).map_or(&[], Vec::as_slice)
}

/// Go's `%v` of a []string: [A B C]
fn show(xs: &[&str]) -> String {
    format!("[{}]", xs.join(" "))
}

/// dfs_recursive: the call stack does the remembering.
fn dfs_recursive(
    g: &Graph,
    at: &'static str,
    seen: &mut HashSet<&'static str>,
    out: &mut Vec<&'static str>,
) {
    if !seen.insert(at) {
        return;
    }
    out.push(at);
    for &next in neighbours(g, at) {
        dfs_recursive(g, next, seen, out);
    }
}

/// dfs_iterative: you do the remembering. Note the two differences from the
/// recursive version, both of which matter:
///
///  1. push the neighbours in REVERSE, because a stack reverses them again
///  2. mark seen when you POP, not when you push -- a node can be pushed several
///     times before it is first visited
fn dfs_iterative(g: &Graph, start: &'static str) -> Vec<&'static str> {
    let mut out = Vec::new();
    let mut seen = HashSet::new();
    let mut stack = vec![start];

    while let Some(at) = stack.pop() {
        if !seen.insert(at) {
            continue; // already visited: this was a stale duplicate
        }
        out.push(at);

        for &next in neighbours(g, at).iter().rev() {
            // reverse push
            if !seen.contains(next) {
                stack.push(next);
            }
        }
    }
    out
}

/// bfs is the SAME CODE with the stack replaced by a queue. That one substitution
/// turns depth-first into breadth-first -- which is lesson 09's whole point.
/// (A Vec has no cheap front, so the queue is a VecDeque: `pop_front` is Go's
/// `queue[0]; queue = queue[1:]`, and it moves no data.)
fn bfs(g: &Graph, start: &'static str) -> Vec<&'static str> {
    let mut out = Vec::new();
    let mut seen = HashSet::from([start]);
    let mut queue = VecDeque::from([start]);

    while let Some(at) = queue.pop_front() {
        // FRONT, not back: the only difference
        out.push(at);

        for &next in neighbours(g, at) {
            if seen.insert(next) {
                queue.push_back(next);
            }
        }
    }
    out
}

/// Which end of the frontier a traversal takes its next node from.
#[derive(Clone, Copy)]
enum End {
    Back,
    Front,
}

/// The DFS above, with the container swapped literally: the same code, the same
/// reverse push, the same mark-on-pop, and ONE `match` that picks the end.
fn traverse(g: &Graph, start: &'static str, end: End) -> Vec<&'static str> {
    let mut out = Vec::new();
    let mut seen = HashSet::new();
    let mut frontier = VecDeque::from([start]);

    loop {
        let taken = match end {
            End::Back => frontier.pop_back(),   // a stack
            End::Front => frontier.pop_front(), // a queue
        };
        let Some(at) = taken else { break };
        if !seen.insert(at) {
            continue;
        }
        out.push(at);
        for &next in neighbours(g, at).iter().rev() {
            if !seen.contains(next) {
                frontier.push_back(next);
            }
        }
    }
    out
}

/// dfs_with_path keeps the path on the stack alongside the node, which is how you
/// answer "how did I get here?" without a parent map.
fn dfs_with_path(g: &Graph, start: &'static str, target: &str) -> Option<Vec<&'static str>> {
    let mut seen = HashSet::new();
    let mut stack: Vec<(&'static str, Vec<&'static str>)> = vec![(start, vec![start])];

    while let Some((at, path)) = stack.pop() {
        if at == target {
            return Some(path);
        }
        if !seen.insert(at) {
            continue;
        }

        for &next in neighbours(g, at).iter().rev() {
            if !seen.contains(next) {
                // CLONE the path: every branch needs its own (lesson 05, example 10)
                let mut p = path.clone();
                p.push(next);
                stack.push((next, p));
            }
        }
    }
    None
}

fn main() {
    let g = graph();

    println!("the graph:");
    println!();
    println!("      A");
    println!("     / \\");
    println!("    B   C");
    println!("   / \\   \\");
    println!("  D   E - F");
    println!();

    let mut rec = Vec::new();
    dfs_recursive(&g, "A", &mut HashSet::new(), &mut rec);
    println!("  DFS recursive  {}", show(&rec));
    println!("  DFS iterative  {}", show(&dfs_iterative(&g, "A")));
    println!("  BFS (a queue)  {}", show(&bfs(&g, "A")));

    println!();
    println!("the two versions of DFS agree -- but only because of two details:");
    println!();
    println!("  1. neighbours are pushed in REVERSE order. A stack hands them back");
    println!("     reversed, so pushing forwards would visit them right-to-left.");
    println!();
    println!("  2. `seen` is set when a node is POPPED, not when it is pushed. A");
    println!("     node reachable by two paths gets pushed twice before either is");
    println!("     visited, so the check has to happen at pop time.");
    println!();
    println!("     (BFS is the opposite: it marks on PUSH, because otherwise a node");
    println!("      can enter the queue many times and the shortest-path property");
    println!("      breaks. Same structure, different rule -- lesson 22.)");

    println!();
    println!("the substitution that changes the algorithm:");
    println!();
    println!("      stack:  let at = xs.pop()        -> DEPTH first   (Vec's cheap end)");
    println!("      queue:  let at = xs.pop_front()  -> BREADTH first (VecDeque)");
    println!();
    println!("  a Vec has no cheap front -- `xs.remove(0)` is Theta(n) -- which is why the");
    println!("  queue is a VecDeque here, and why lesson 09 exists.");
    println!();
    println!("  and taken LITERALLY, with a single `match` choosing the end and nothing");
    println!("  else changed (reverse push, mark on pop):");
    println!();
    println!(
        "      traverse(Back)   {}",
        show(&traverse(&g, "A", End::Back))
    );
    println!(
        "      traverse(Front)  {}",
        show(&traverse(&g, "A", End::Front))
    );
    println!();
    println!("  the container decides depth versus breadth: Front really is breadth-");
    println!("  first (A, then C B, then F E D). But it is not the bfs() above, because");
    println!("  the reverse push flips every level. So the container is one line of the");
    println!("  difference, not all of it: bfs() also pushes forwards and marks on push.");

    println!();
    println!(
        "carrying the path on the stack: A -> F is {}",
        dfs_with_path(&g, "A", "F").map_or_else(|| "none".to_string(), |p| show(&p))
    );
    println!();
    println!("  note the path is CLONED into each frame. Sharing one path across");
    println!("  branches is lesson 05's example 10 bug in Go; here the types forbid it:");
    println!("  a frame owns its Vec, so two frames cannot alias one buffer, and the");
    println!("  clone is the only way to give a branch a copy of the path.");

    println!();
    println!("why prefer the explicit stack here at all?");
    println!();
    println!("  the recursive DFS is shorter and clearer, and for a bounded graph");
    println!("  it is the right choice. The explicit version wins when the depth");
    println!("  can grow with the input -- a chain-shaped graph is n frames deep");
    println!("  (lesson 05, example 15), and an explicit stack lives on the heap.");
    println!("  In Rust that matters sooner than in Go: a thread's call stack has a fixed");
    println!("  size (8 MB for the main thread here), where Go's stacks grow on demand.");
}
```

**Rust output:**

```
the graph:

      A
     / \
    B   C
   / \   \
  D   E - F

  DFS recursive  [A B D E F C]
  DFS iterative  [A B D E F C]
  BFS (a queue)  [A B C D E F]

the two versions of DFS agree -- but only because of two details:

  1. neighbours are pushed in REVERSE order. A stack hands them back
     reversed, so pushing forwards would visit them right-to-left.

  2. `seen` is set when a node is POPPED, not when it is pushed. A
     node reachable by two paths gets pushed twice before either is
     visited, so the check has to happen at pop time.

     (BFS is the opposite: it marks on PUSH, because otherwise a node
      can enter the queue many times and the shortest-path property
      breaks. Same structure, different rule -- lesson 22.)

the substitution that changes the algorithm:

      stack:  let at = xs.pop()        -> DEPTH first   (Vec's cheap end)
      queue:  let at = xs.pop_front()  -> BREADTH first (VecDeque)

  a Vec has no cheap front -- `xs.remove(0)` is Theta(n) -- which is why the
  queue is a VecDeque here, and why lesson 09 exists.

  and taken LITERALLY, with a single `match` choosing the end and nothing
  else changed (reverse push, mark on pop):

      traverse(Back)   [A B D E F C]
      traverse(Front)  [A C B F E D]

  the container decides depth versus breadth: Front really is breadth-
  first (A, then C B, then F E D). But it is not the bfs() above, because
  the reverse push flips every level. So the container is one line of the
  difference, not all of it: bfs() also pushes forwards and marks on push.

carrying the path on the stack: A -> F is [A B E F]

  note the path is CLONED into each frame. Sharing one path across
  branches is lesson 05's example 10 bug in Go; here the types forbid it:
  a frame owns its Vec, so two frames cannot alias one buffer, and the
  clone is the only way to give a branch a copy of the path.

why prefer the explicit stack here at all?

  the recursive DFS is shorter and clearer, and for a bounded graph
  it is the right choice. The explicit version wins when the depth
  can grow with the input -- a chain-shaped graph is n frames deep
  (lesson 05, example 15), and an explicit stack lives on the heap.
  In Rust that matters sooner than in Go: a thread's call stack has a fixed
  size (8 MB for the main thread here), where Go's stacks grow on demand.
```

> **Go vs Rust:** The traversal orders and the path (DFS `[A B D E F C]` both ways, BFS `[A B C D E F]`, A to F `[A B E F]`) are identical to Go's. The queue is a `VecDeque`, since a `Vec` has no cheap front (`remove(0)` is Θ(n)), and the path must be `clone`d per branch, which the ownership rules enforce (the aliasing bug of lesson 05 example 10 cannot be written). One Go claim does not hold when run: "one line. Everything else is identical." Swapping only the end the frontier is read from (`traverse(Back)` / `traverse(Front)`) gives a correct breadth-first order, but `[A C B F E D]`, not the `[A B C D E F]` of `bfs()`, because DFS's reverse push flips every level; `bfs()` differs from the DFS in three ways (end, push order, mark on push), not one.

**Complexity:** Θ(V+E) either way · the difference between `xs[len-1]` and `xs[0]` is the entire difference between depth-first and breadth-first

---

> Next tier: [🟡 medium](2-medium.md).

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*