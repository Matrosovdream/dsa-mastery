# Step 05 — Recursion & the Call Stack · 🟢 Easy

Examples **1–6**: termination, the shape of the stack, and when a recursion is really a loop.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Examples 1–5 are fully **deterministic** — they count and print, they never time anything. Example 6
measures stack bytes, so its numbers move a little between runs.

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

## 1. Base case and inductive step

`🟢 easy` · *Termination*

Every recursive function is two things: an input small enough to answer directly, and a step that
makes the problem **strictly smaller**. Get either wrong and it never stops — and there are exactly
three ways to get it wrong.

The third one is the interesting one: `if n == 1` looks like a perfectly good base case, and it works
for every input except 0.

**Steps:**

1. Write the correct version and confirm it stops on its own.
2. Break it three ways, each with a depth seatbelt so the program survives.
3. Compare where each one stops.

```go
package main

import "fmt"

// Every recursive function is exactly two things:
//
//   BASE CASE      an input small enough to answer without recursing
//   INDUCTIVE STEP reduce the problem to a STRICTLY SMALLER one, and trust it
//
// "Trust it" is the part people find hard. You do not trace the whole thing in
// your head; you check that the smaller call is smaller, and that the base case
// catches it. That is a proof by induction, and it is the whole technique.

func factorial(n int) int {
	if n <= 1 { // base case
		return 1
	}
	return n * factorial(n-1) // inductive step: n-1 is strictly smaller
}

// The three broken versions below return the DEPTH they reached rather than a
// factorial, so we can see how far they got. Each has a seatbelt at maxDepth --
// which is not a base case, just this demo refusing to die.
const maxDepth = 20

// correctDepth is the control: a working recursion stops on its own.
func correctDepth(n, depth int) int {
	if depth >= maxDepth {
		return depth
	}
	if n <= 1 {
		return depth // reached the real base case here
	}
	return correctDepth(n-1, depth+1)
}

func missingBase(n, depth int) int {
	if depth >= maxDepth {
		return depth
	}
	return missingBase(n-1, depth+1) // no base case at all
}

func wrongBase(n, depth int) int {
	if depth >= maxDepth {
		return depth
	}
	if n == 1 { // WRONG: starting at 0, n goes 0, -1, -2, ... and never hits 1
		return depth
	}
	return wrongBase(n-1, depth+1)
}

func neverShrinks(n, depth int) int {
	if depth >= maxDepth {
		return depth
	}
	if n <= 1 {
		return depth
	}
	return neverShrinks(n, depth+1) // the argument never gets smaller
}

func main() {
	fmt.Println("the correct version:")
	for n := 0; n <= 6; n++ {
		fmt.Printf("  factorial(%d) = %d\n", n, factorial(n))
	}

	fmt.Println()
	fmt.Printf("now the depth each version reaches, with a seatbelt at %d:\n\n", maxDepth)
	fmt.Printf("  %-34s depth %2d   %s\n", "correct, n=5", correctDepth(5, 0), "stopped on its own")
	fmt.Printf("  %-34s depth %2d   %s\n", "1. no base case, n=5", missingBase(5, 0), "HIT THE SEATBELT")
	fmt.Printf("  %-34s depth %2d   %s\n", "2. base case n==1, called at 0", wrongBase(0, 0), "HIT THE SEATBELT")
	fmt.Printf("  %-34s depth %2d   %s\n", "3. argument never shrinks, n=5", neverShrinks(5, 0), "HIT THE SEATBELT")

	fmt.Println()
	fmt.Println("the correct one stopped at depth 4 because it ARRIVED at its base case.")
	fmt.Println("The other three would have run forever; without that seatbelt they consume")
	fmt.Println("stack until the runtime dies with 'fatal error: stack overflow' (example 9).")
	fmt.Println()
	fmt.Println("case 2 is the subtle one: `if n == 1` looks like a perfectly good base case,")
	fmt.Println("and it works for every input EXCEPT 0. Test the boundary, not the middle.")

	fmt.Println()
	fmt.Println("the checklist, every single time you write a recursive function:")
	fmt.Println("  1. is there a base case?")
	fmt.Println("  2. does EVERY path to the recursive call make the argument strictly smaller?")
	fmt.Println("  3. does shrinking always ARRIVE at the base case? (n-1 from 0 does not)")
	fmt.Println()
	fmt.Println("write the base case FIRST. It is the answer you already know, and it")
	fmt.Println("tells you what 'smaller' has to mean.")
}
```

**Output:**

```
the correct version:
  factorial(0) = 1
  factorial(1) = 1
  factorial(2) = 2
  factorial(3) = 6
  factorial(4) = 24
  factorial(5) = 120
  factorial(6) = 720

now the depth each version reaches, with a seatbelt at 20:

  correct, n=5                       depth  4   stopped on its own
  1. no base case, n=5               depth 20   HIT THE SEATBELT
  2. base case n==1, called at 0     depth 20   HIT THE SEATBELT
  3. argument never shrinks, n=5     depth 20   HIT THE SEATBELT

the correct one stopped at depth 4 because it ARRIVED at its base case.
The other three would have run forever; without that seatbelt they consume
stack until the runtime dies with 'fatal error: stack overflow' (example 9).

case 2 is the subtle one: `if n == 1` looks like a perfectly good base case,
and it works for every input EXCEPT 0. Test the boundary, not the middle.

the checklist, every single time you write a recursive function:
  1. is there a base case?
  2. does EVERY path to the recursive call make the argument strictly smaller?
  3. does shrinking always ARRIVE at the base case? (n-1 from 0 does not)

write the base case FIRST. It is the answer you already know, and it
tells you what 'smaller' has to mean.
```

**Rust:**

```rust
// Every recursive function is exactly two things:
//
//   BASE CASE      an input small enough to answer without recursing
//   INDUCTIVE STEP reduce the problem to a STRICTLY SMALLER one, and trust it
//
// "Trust it" is the part people find hard. You do not trace the whole thing in
// your head; you check that the smaller call is smaller, and that the base case
// catches it. That is a proof by induction, and it is the whole technique.

fn factorial(n: i64) -> i64 {
    if n <= 1 {
        return 1; // base case
    }
    n * factorial(n - 1) // inductive step: n-1 is strictly smaller
}

// The three broken versions below return the DEPTH they reached rather than a
// factorial, so we can see how far they got. Each has a seatbelt at MAX_DEPTH --
// which is not a base case, just this demo refusing to die.
const MAX_DEPTH: usize = 20;

// correct_depth is the control: a working recursion stops on its own.
fn correct_depth(n: i64, depth: usize) -> usize {
    if depth >= MAX_DEPTH {
        return depth;
    }
    if n <= 1 {
        return depth; // reached the real base case here
    }
    correct_depth(n - 1, depth + 1)
}

// Rust-only footnote: clippy flags this function, because `n` is never read --
// it is only passed down to the next call. That IS the bug (no base case looks
// at n), so the lint is a free code review. We silence it here only because the
// broken function is the demonstration.
#[allow(clippy::only_used_in_recursion)]
fn missing_base(n: i64, depth: usize) -> usize {
    if depth >= MAX_DEPTH {
        return depth;
    }
    missing_base(n - 1, depth + 1) // no base case at all
}

fn wrong_base(n: i64, depth: usize) -> usize {
    if depth >= MAX_DEPTH {
        return depth;
    }
    if n == 1 {
        // WRONG: starting at 0, n goes 0, -1, -2, ... and never hits 1
        return depth;
    }
    wrong_base(n - 1, depth + 1)
}

fn never_shrinks(n: i64, depth: usize) -> usize {
    if depth >= MAX_DEPTH {
        return depth;
    }
    if n <= 1 {
        return depth;
    }
    never_shrinks(n, depth + 1) // the argument never gets smaller
}

fn main() {
    println!("the correct version:");
    for n in 0..=6 {
        println!("  factorial({n}) = {}", factorial(n));
    }

    println!();
    println!("now the depth each version reaches, with a seatbelt at {MAX_DEPTH}:");
    println!();
    // (label, depth reached, verdict) -- one row per version
    let rows = [
        ("correct, n=5", correct_depth(5, 0), "stopped on its own"),
        (
            "1. no base case, n=5",
            missing_base(5, 0),
            "HIT THE SEATBELT",
        ),
        (
            "2. base case n==1, called at 0",
            wrong_base(0, 0),
            "HIT THE SEATBELT",
        ),
        (
            "3. argument never shrinks, n=5",
            never_shrinks(5, 0),
            "HIT THE SEATBELT",
        ),
    ];
    for (name, depth, verdict) in rows {
        println!("  {name:<34} depth {depth:>2}   {verdict}");
    }

    println!();
    println!("the correct one stopped at depth 4 because it ARRIVED at its base case.");
    println!("The other three would have run forever; without that seatbelt they consume");
    println!("stack until the process dies with 'fatal runtime error: stack overflow,");
    println!("aborting' (example 9) -- and Rust cannot catch that: it is an abort, not a panic.");
    println!();
    println!("case 2 is the subtle one: `if n == 1` looks like a perfectly good base case,");
    println!("and it works for every input EXCEPT 0. Test the boundary, not the middle.");

    println!();
    println!("the checklist, every single time you write a recursive function:");
    println!("  1. is there a base case?");
    println!("  2. does EVERY path to the recursive call make the argument strictly smaller?");
    println!("  3. does shrinking always ARRIVE at the base case? (n-1 from 0 does not)");
    println!();
    println!("write the base case FIRST. It is the answer you already know, and it");
    println!("tells you what 'smaller' has to mean.");
}
```

**Rust output:**

```
the correct version:
  factorial(0) = 1
  factorial(1) = 1
  factorial(2) = 2
  factorial(3) = 6
  factorial(4) = 24
  factorial(5) = 120
  factorial(6) = 720

now the depth each version reaches, with a seatbelt at 20:

  correct, n=5                       depth  4   stopped on its own
  1. no base case, n=5               depth 20   HIT THE SEATBELT
  2. base case n==1, called at 0     depth 20   HIT THE SEATBELT
  3. argument never shrinks, n=5     depth 20   HIT THE SEATBELT

the correct one stopped at depth 4 because it ARRIVED at its base case.
The other three would have run forever; without that seatbelt they consume
stack until the process dies with 'fatal runtime error: stack overflow,
aborting' (example 9) -- and Rust cannot catch that: it is an abort, not a panic.

case 2 is the subtle one: `if n == 1` looks like a perfectly good base case,
and it works for every input EXCEPT 0. Test the boundary, not the middle.

the checklist, every single time you write a recursive function:
  1. is there a base case?
  2. does EVERY path to the recursive call make the argument strictly smaller?
  3. does shrinking always ARRIVE at the base case? (n-1 from 0 does not)

write the base case FIRST. It is the answer you already know, and it
tells you what 'smaller' has to mean.
```

> **Go vs Rust:** Every depth matches Go (4, 20, 20, 20); the only line that differs is the overflow message, because a Rust stack overflow prints `fatal runtime error: stack overflow, aborting` and kills the process with SIGABRT (exit 134 in a scratch test here) -- it is an abort, not a panic, so nothing can catch it. Broken case 1 is also caught by clippy, which fires `only_used_in_recursion` on `n` (checked by removing the `#[allow]`): a missing base case means nothing ever reads the argument.

**Complexity:** `factorial(n)` is Θ(n) time and **Θ(n) stack** — the depth grows with the input, which example 13 identifies as the shape you eventually have to rewrite

---

## 2. Descend, then unwind

`🟢 easy` · *Tracing the call stack*

Printing on the way in and on the way out makes the shape impossible to miss: calls go **all the way
down** doing nothing, and the work happens **on the way back up**. That's why nothing can be added
until the deepest call returns — and why every frame has to stay alive until then.

**Steps:**

1. Print on entry, indented by depth.
2. Recurse.
3. Print on exit, showing what was combined.

```go
package main

import (
	"fmt"
	"strings"
)

// A recursive call does two things people tend to conflate: it DESCENDS (each
// call suspends and starts a new one) and then it UNWINDS (each suspended call
// resumes, in reverse order, with its own saved locals).
//
// Printing on the way in and on the way out makes the shape impossible to miss.

func sum(xs []int, depth int) int {
	pad := strings.Repeat("  ", depth)
	fmt.Printf("%s-> sum(%v)\n", pad, xs)

	if len(xs) == 0 {
		fmt.Printf("%s<- 0   (base case)\n", pad)
		return 0
	}

	rest := sum(xs[1:], depth+1)
	total := xs[0] + rest

	fmt.Printf("%s<- %d  (%d + %d)\n", pad, total, xs[0], rest)
	return total
}

func main() {
	fmt.Println("sum([1 2 3 4]) -- every call printed on entry and on exit:")
	fmt.Println()
	result := sum([]int{1, 2, 3, 4}, 0)
	fmt.Println()
	fmt.Println("result:", result)

	fmt.Println()
	fmt.Println("read the shape:")
	fmt.Println("  the -> lines go all the way DOWN to the base case, doing no arithmetic.")
	fmt.Println("  the <- lines come back UP, and that is where the work happens.")
	fmt.Println()
	fmt.Println("at the deepest point, FIVE frames are alive at once: sum([1 2 3 4]),")
	fmt.Println("sum([2 3 4]), sum([3 4]), sum([4]) and sum([]). Each holds its own xs")
	fmt.Println("and its own rest. That is the O(n) space from lesson 03.")
	fmt.Println()
	fmt.Println("the work-on-the-way-back-up pattern is why this is NOT a loop in disguise:")
	fmt.Println("nothing can be added until the deepest call returns. Example 6 shows the")
	fmt.Println("rewrite that does the work on the way DOWN instead.")
}
```

**Output:**

```
sum([1 2 3 4]) -- every call printed on entry and on exit:

-> sum([1 2 3 4])
  -> sum([2 3 4])
    -> sum([3 4])
      -> sum([4])
        -> sum([])
        <- 0   (base case)
      <- 4  (4 + 0)
    <- 7  (3 + 4)
  <- 9  (2 + 7)
<- 10  (1 + 9)

result: 10

read the shape:
  the -> lines go all the way DOWN to the base case, doing no arithmetic.
  the <- lines come back UP, and that is where the work happens.

at the deepest point, FIVE frames are alive at once: sum([1 2 3 4]),
sum([2 3 4]), sum([3 4]), sum([4]) and sum([]). Each holds its own xs
and its own rest. That is the O(n) space from lesson 03.

the work-on-the-way-back-up pattern is why this is NOT a loop in disguise:
nothing can be added until the deepest call returns. Example 6 shows the
rewrite that does the work on the way DOWN instead.
```

**Rust:**

```rust
// A recursive call does two things people tend to conflate: it DESCENDS (each
// call suspends and starts a new one) and then it UNWINDS (each suspended call
// resumes, in reverse order, with its own saved locals).
//
// Printing on the way in and on the way out makes the shape impossible to miss.

// Go's %v prints a slice as `[1 2 3]`; Rust's {:?} would print `[1, 2, 3]`.
// Format by hand so the trace reads the same.
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(|v| v.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn sum(xs: &[i64], depth: usize) -> i64 {
    let pad = "  ".repeat(depth);
    println!("{pad}-> sum({})", show(xs));

    // split_first is the idiomatic "head and tail" of a slice: None means empty.
    let Some((&head, tail)) = xs.split_first() else {
        println!("{pad}<- 0   (base case)");
        return 0;
    };

    let rest = sum(tail, depth + 1);
    let total = head + rest;

    println!("{pad}<- {total}  ({head} + {rest})");
    total
}

fn main() {
    println!("sum([1 2 3 4]) -- every call printed on entry and on exit:");
    println!();
    let result = sum(&[1, 2, 3, 4], 0);
    println!();
    println!("result: {result}");

    println!();
    println!("read the shape:");
    println!("  the -> lines go all the way DOWN to the base case, doing no arithmetic.");
    println!("  the <- lines come back UP, and that is where the work happens.");
    println!();
    println!("at the deepest point, FIVE frames are alive at once: sum([1 2 3 4]),");
    println!("sum([2 3 4]), sum([3 4]), sum([4]) and sum([]). Each holds its own xs");
    println!("(a pointer and a length -- a borrowed view, no copy) and its own rest.");
    println!("That is the O(n) space from lesson 03.");
    println!();
    println!("the work-on-the-way-back-up pattern is why this is NOT a loop in disguise:");
    println!("nothing can be added until the deepest call returns. Example 6 shows the");
    println!("rewrite that does the work on the way DOWN instead.");
}
```

**Rust output:**

```
sum([1 2 3 4]) -- every call printed on entry and on exit:

-> sum([1 2 3 4])
  -> sum([2 3 4])
    -> sum([3 4])
      -> sum([4])
        -> sum([])
        <- 0   (base case)
      <- 4  (4 + 0)
    <- 7  (3 + 4)
  <- 9  (2 + 7)
<- 10  (1 + 9)

result: 10

read the shape:
  the -> lines go all the way DOWN to the base case, doing no arithmetic.
  the <- lines come back UP, and that is where the work happens.

at the deepest point, FIVE frames are alive at once: sum([1 2 3 4]),
sum([2 3 4]), sum([3 4]), sum([4]) and sum([]). Each holds its own xs
(a pointer and a length -- a borrowed view, no copy) and its own rest.
That is the O(n) space from lesson 03.

the work-on-the-way-back-up pattern is why this is NOT a loop in disguise:
nothing can be added until the deepest call returns. Example 6 shows the
rewrite that does the work on the way DOWN instead.
```

> **Go vs Rust:** The trace is identical to the Go trace except one reworded prose line (a `&[i64]` is a pointer plus a length). Go prints a slice with `%v` as `[1 2 3 4]`, but Rust `{:?}` gives `[1, 2, 3, 4]`, so the slice is formatted by hand in `show`. `split_first` replaces `xs[0]` / `xs[1:]` and turns the empty-slice base case into a pattern the compiler checks is handled.

**Complexity:** Θ(n) time, Θ(n) space — and the space is the five simultaneously-live frames you can see in the indentation

---

## 3. Every frame keeps its own locals

`🟢 easy` · *What a stack frame is for*

The reason recursion works at all: each call gets its own copy of the parameters, untouched by
everything that happens deeper down. This example proves it by reading a parameter **after** four
nested calls have come and gone.

**Steps:**

1. Record a parameter on entry.
2. Recurse.
3. Record the same parameter again after the child returns, and compare the two columns.

```go
package main

import "fmt"

// Every call gets its OWN copy of the parameters and locals. That is what makes
// recursion work at all -- and it is exactly what the stack frames from lesson
// 03 are paying for.
//
// countdown records the value of its local `n` at two moments: when the call
// starts, and again after the recursive call returns. If frames shared storage,
// the second column would be wrong.

type observation struct {
	depth      int
	onEntry    int
	afterChild int
}

func countdown(n, depth int, log *[]observation) {
	obs := observation{depth: depth, onEntry: n}

	if n > 0 {
		countdown(n-1, depth+1, log)
	}

	// n is read again AFTER the child call has finished and returned.
	obs.afterChild = n
	*log = append(*log, obs)
}

// The same idea with a slice: each frame holds a different VIEW of one array.
func shrink(xs []int, depth int) {
	fmt.Printf("  depth %d: len=%d cap=%d  %v\n", depth, len(xs), cap(xs), xs)
	if len(xs) > 0 {
		shrink(xs[1:], depth+1)
	}
}

func main() {
	var log []observation
	countdown(4, 0, &log)

	fmt.Println("each frame keeps its own n across the recursive call:")
	fmt.Println()
	fmt.Printf("  %-8s %-12s %-12s\n", "depth", "n on entry", "n after the child returned")
	for i := len(log) - 1; i >= 0; i-- { // log was appended on the way UP
		o := log[i]
		fmt.Printf("  %-8d %-12d %d\n", o.depth, o.onEntry, o.afterChild)
	}

	fmt.Println()
	fmt.Println("the two columns match at every depth. The frame at depth 0 still had")
	fmt.Println("n=4 waiting for it after four nested calls had come and gone.")

	fmt.Println()
	fmt.Println("the same thing with a slice -- five frames, five different views:")
	fmt.Println()
	shrink([]int{10, 20, 30, 40}, 0)

	fmt.Println()
	fmt.Println("note cap shrinks with len here: each xs[1:] moves the pointer forward")
	fmt.Println("into the SAME backing array (lesson 04). Recursion over a slice costs")
	fmt.Println("one 24-byte header per frame, not a copy of the data.")
	fmt.Println()
	fmt.Println("this is why recursion is O(depth) space and not O(depth x n):")
	fmt.Println("frames are cheap, but there is one per level and they all stay alive")
	fmt.Println("until the deepest one returns.")
}
```

**Output:**

```
each frame keeps its own n across the recursive call:

  depth    n on entry   n after the child returned
  0        4            4
  1        3            3
  2        2            2
  3        1            1
  4        0            0

the two columns match at every depth. The frame at depth 0 still had
n=4 waiting for it after four nested calls had come and gone.

the same thing with a slice -- five frames, five different views:

  depth 0: len=4 cap=4  [10 20 30 40]
  depth 1: len=3 cap=3  [20 30 40]
  depth 2: len=2 cap=2  [30 40]
  depth 3: len=1 cap=1  [40]
  depth 4: len=0 cap=0  []

note cap shrinks with len here: each xs[1:] moves the pointer forward
into the SAME backing array (lesson 04). Recursion over a slice costs
one 24-byte header per frame, not a copy of the data.

this is why recursion is O(depth) space and not O(depth x n):
frames are cheap, but there is one per level and they all stay alive
until the deepest one returns.
```

Note the slice version: `cap` shrinks along with `len` because each `xs[1:]` moves the pointer forward
into the **same backing array** (lesson 04). Recursing over a slice costs one 24-byte header per
frame, not a copy of the data.

**Rust:**

```rust
use std::mem::size_of;

// Every call gets its OWN copy of the parameters and locals. That is what makes
// recursion work at all -- and it is exactly what the stack frames from lesson
// 03 are paying for.
//
// countdown records the value of its local `n` at two moments: when the call
// starts, and again after the recursive call returns. If frames shared storage,
// the second column would be wrong.

struct Observation {
    depth: usize,
    on_entry: i64,
    after_child: i64,
}

// `log: &mut Vec<..>` is how Rust threads shared state through recursion: the
// borrow is passed down explicitly. A child can only change the parent's `n`
// if the parent hands it a `&mut` -- here it hands over a copy, so it cannot.
fn countdown(n: i64, depth: usize, log: &mut Vec<Observation>) {
    let mut obs = Observation {
        depth,
        on_entry: n,
        after_child: 0,
    };

    if n > 0 {
        countdown(n - 1, depth + 1, log);
    }

    // n is read again AFTER the child call has finished and returned.
    obs.after_child = n;
    log.push(obs);
}

// The same idea with a slice: each frame holds a different VIEW of one array.
// `base` is the whole array, kept only so we can print where each view starts.
// (A Rust slice has no capacity, so Go's `cap` column has no counterpart: the
// byte offset of the view's first element is the thing worth looking at.)
fn shrink(xs: &[i64], base: &[i64], depth: usize) {
    let offset = xs.as_ptr().addr() - base.as_ptr().addr();
    println!(
        "  depth {depth}: len={} offset={offset:>2} bytes  {}",
        xs.len(),
        show(xs)
    );
    if !xs.is_empty() {
        shrink(&xs[1..], base, depth + 1);
    }
}

// Go's %v prints `[1 2 3]`; Rust's {:?} prints `[1, 2, 3]`. Format by hand.
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(|v| v.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let mut log = Vec::new();
    countdown(4, 0, &mut log);

    println!("each frame keeps its own n across the recursive call:");
    println!();
    println!(
        "  {:<8} {:<12} {:<12}",
        "depth", "n on entry", "n after the child returned"
    );
    for o in log.iter().rev() {
        // log was pushed on the way UP, so walk it backwards
        println!("  {:<8} {:<12} {}", o.depth, o.on_entry, o.after_child);
    }

    println!();
    println!("the two columns match at every depth. The frame at depth 0 still had");
    println!("n=4 waiting for it after four nested calls had come and gone.");

    println!();
    println!("the same thing with a slice -- five frames, five different views:");
    println!();
    let data = [10, 20, 30, 40];
    shrink(&data, &data, 0);

    println!();
    println!("note the offset grows by one element (8 bytes) at each level: each");
    println!("&xs[1..] moves the pointer forward into the SAME array (lesson 04). Recursion");
    println!(
        "over a slice costs one {}-byte (pointer, len) header per frame, not a copy",
        size_of::<&[i64]>()
    );
    println!("of the data.");
    println!();
    println!("this is why recursion is O(depth) space and not O(depth x n):");
    println!("frames are cheap, but there is one per level and they all stay alive");
    println!("until the deepest one returns.");
}
```

**Rust output:**

```
each frame keeps its own n across the recursive call:

  depth    n on entry   n after the child returned
  0        4            4
  1        3            3
  2        2            2
  3        1            1
  4        0            0

the two columns match at every depth. The frame at depth 0 still had
n=4 waiting for it after four nested calls had come and gone.

the same thing with a slice -- five frames, five different views:

  depth 0: len=4 offset= 0 bytes  [10 20 30 40]
  depth 1: len=3 offset= 8 bytes  [20 30 40]
  depth 2: len=2 offset=16 bytes  [30 40]
  depth 3: len=1 offset=24 bytes  [40]
  depth 4: len=0 offset=32 bytes  []

note the offset grows by one element (8 bytes) at each level: each
&xs[1..] moves the pointer forward into the SAME array (lesson 04). Recursion
over a slice costs one 16-byte (pointer, len) header per frame, not a copy
of the data.

this is why recursion is O(depth) space and not O(depth x n):
frames are cheap, but there is one per level and they all stay alive
until the deepest one returns.
```

> **Go vs Rust:** The first table is identical to Go. The slice half differs because a Rust slice is a pointer plus a length with no capacity, so Go's `cap` column becomes the byte offset of each view into the same array (0, 8, 16, 24, 32), and the header is 16 bytes (`size_of::<&[i64]>()`) against Go's 24. The shared log is a `&mut Vec` passed down explicitly: a child can only touch its parent's `n` if it is handed a `&mut`.

**Complexity:** Θ(depth) frames alive at once · each frame is ~34–49 bytes for a plain function (example 9)

---

## 4. Total calls vs maximum depth

`🟢 easy` · *Time and space are different questions*

The most common analysis mistake in this topic is conflating the **size** of the recursion tree with
its **height**. One predicts time, the other predicts stack. Four shapes, same n, wildly different
answers.

**Steps:**

1. Instrument four recursion shapes with a call counter and a depth watermark.
2. Run all four at n=24.
3. Read the two middle columns as separate questions.

```go
package main

import "fmt"

// Two numbers describe any recursion, and confusing them is the most common
// analysis mistake in this whole topic:
//
//   TOTAL CALLS  = the size of the recursion tree  -> TIME
//   MAX DEPTH    = the longest root-to-leaf path   -> SPACE (stack frames)
//
// They are wildly different for different shapes.

var calls, maxDepth int

func reset() { calls, maxDepth = 0, 0 }

func enter(depth int) {
	calls++
	if depth > maxDepth {
		maxDepth = depth
	}
}

// linear: one call per level. Tree is a chain.
func linear(n, depth int) {
	enter(depth)
	if n <= 0 {
		return
	}
	linear(n-1, depth+1)
}

// halving: one call per level, but the input halves. Chain of length log n.
func halving(n, depth int) {
	enter(depth)
	if n <= 1 {
		return
	}
	halving(n/2, depth+1)
}

// binaryHalving: TWO calls per level, each on half. Merge sort's shape.
func binaryHalving(n, depth int) {
	enter(depth)
	if n <= 1 {
		return
	}
	binaryHalving(n/2, depth+1)
	binaryHalving(n/2, depth+1)
}

// binaryDecrement: two calls per level, each on n-1. Naive Fibonacci's shape.
func binaryDecrement(n, depth int) {
	enter(depth)
	if n <= 1 {
		return
	}
	binaryDecrement(n-1, depth+1)
	binaryDecrement(n-2, depth+1)
}

func main() {
	shapes := []struct {
		name string
		call string
		f    func(int, int)
		time string
		spc  string
	}{
		{"linear", "f(n-1)", linear, "Theta(n)", "Theta(n)"},
		{"halving", "f(n/2)", halving, "Theta(log n)", "Theta(log n)"},
		{"binary halving", "f(n/2) twice", binaryHalving, "Theta(n)", "Theta(log n)"},
		{"binary decrement", "f(n-1), f(n-2)", binaryDecrement, "Theta(phi^n)", "Theta(n)"},
	}

	fmt.Printf("%-18s %-16s %12s %12s   %-14s %s\n",
		"shape", "recursive call", "calls", "max depth", "time", "space")
	for _, s := range shapes {
		reset()
		s.f(24, 0)
		fmt.Printf("%-18s %-16s %12d %12d   %-14s %s\n",
			s.name, s.call, calls, maxDepth, s.time, s.spc)
	}

	fmt.Println()
	fmt.Println("n = 24 for all four. Read the two middle columns as separate questions:")
	fmt.Println()
	fmt.Println("  'binary halving' makes 31 calls but reaches depth 4. Lots of work,")
	fmt.Println("     hardly any stack. This is divide & conquer, and it is why merge")
	fmt.Println("     sort is Theta(n log n) TIME and Theta(log n) STACK (example 13).")
	fmt.Println()
	fmt.Println("  'binary decrement' makes 150,049 calls at depth 23. Enormous work,")
	fmt.Println("     modest stack -- and the work is the problem (memoize it, example 8).")
	fmt.Println()
	fmt.Println("  'linear' makes only 25 calls but reaches depth 24: the stack IS the")
	fmt.Println("     input. Cheap in time, and the one shape that overflows on real data.")

	fmt.Println()
	fmt.Println("watch the calls column explode with n for the last shape:")
	for _, n := range []int{10, 20, 30, 35} {
		reset()
		binaryDecrement(n, 0)
		fmt.Printf("  n=%2d  calls=%10d  depth=%d\n", n, calls, maxDepth)
	}
}
```

**Output:**

```
shape              recursive call          calls    max depth   time           space
linear             f(n-1)                     25           24   Theta(n)       Theta(n)
halving            f(n/2)                      5            4   Theta(log n)   Theta(log n)
binary halving     f(n/2) twice               31            4   Theta(n)       Theta(log n)
binary decrement   f(n-1), f(n-2)         150049           23   Theta(phi^n)   Theta(n)

n = 24 for all four. Read the two middle columns as separate questions:

  'binary halving' makes 31 calls but reaches depth 4. Lots of work,
     hardly any stack. This is divide & conquer, and it is why merge
     sort is Theta(n log n) TIME and Theta(log n) STACK (example 13).

  'binary decrement' makes 150,049 calls at depth 23. Enormous work,
     modest stack -- and the work is the problem (memoize it, example 8).

  'linear' makes only 25 calls but reaches depth 24: the stack IS the
     input. Cheap in time, and the one shape that overflows on real data.

watch the calls column explode with n for the last shape:
  n=10  calls=       177  depth=9
  n=20  calls=     21891  depth=19
  n=30  calls=   2692537  depth=29
  n=35  calls=  29860703  depth=34
```

Three different problems in one table: `binary halving` is 31 calls at depth 4 (fine — that's divide &
conquer). `binary decrement` is 150,049 calls at depth 23 (the *work* is the problem — memoize it,
example 8). `linear` is 25 calls at depth 24 (the *stack* is the problem — example 13).

**Rust:**

```rust
// Two numbers describe any recursion, and confusing them is the most common
// analysis mistake in this whole topic:
//
//   TOTAL CALLS  = the size of the recursion tree  -> TIME
//   MAX DEPTH    = the longest root-to-leaf path   -> SPACE (stack frames)
//
// They are wildly different for different shapes.

// Go keeps these in package-level variables. Rust has no mutable globals in
// safe code, so the counters live in a struct and a `&mut Stats` is passed
// down every call -- the borrow checker guarantees only one frame uses it at a
// time, which is exactly how recursion uses it.
#[derive(Default)]
struct Stats {
    calls: u64,
    max_depth: usize,
}

impl Stats {
    fn enter(&mut self, depth: usize) {
        self.calls += 1;
        self.max_depth = self.max_depth.max(depth);
    }
}

// linear: one call per level. Tree is a chain.
fn linear(n: i64, depth: usize, s: &mut Stats) {
    s.enter(depth);
    if n <= 0 {
        return;
    }
    linear(n - 1, depth + 1, s);
}

// halving: one call per level, but the input halves. Chain of length log n.
fn halving(n: i64, depth: usize, s: &mut Stats) {
    s.enter(depth);
    if n <= 1 {
        return;
    }
    halving(n / 2, depth + 1, s);
}

// binary_halving: TWO calls per level, each on half. Merge sort's shape.
fn binary_halving(n: i64, depth: usize, s: &mut Stats) {
    s.enter(depth);
    if n <= 1 {
        return;
    }
    binary_halving(n / 2, depth + 1, s);
    binary_halving(n / 2, depth + 1, s);
}

// binary_decrement: two calls per level, each on n-1. Naive Fibonacci's shape.
fn binary_decrement(n: i64, depth: usize, s: &mut Stats) {
    s.enter(depth);
    if n <= 1 {
        return;
    }
    binary_decrement(n - 1, depth + 1, s);
    binary_decrement(n - 2, depth + 1, s);
}

struct Shape {
    name: &'static str,
    call: &'static str,
    f: fn(i64, usize, &mut Stats),
    time: &'static str,
    space: &'static str,
}

// Run one shape from scratch and hand back its counters.
fn measure(f: fn(i64, usize, &mut Stats), n: i64) -> Stats {
    let mut s = Stats::default();
    f(n, 0, &mut s);
    s
}

fn main() {
    let shapes = [
        Shape {
            name: "linear",
            call: "f(n-1)",
            f: linear,
            time: "Theta(n)",
            space: "Theta(n)",
        },
        Shape {
            name: "halving",
            call: "f(n/2)",
            f: halving,
            time: "Theta(log n)",
            space: "Theta(log n)",
        },
        Shape {
            name: "binary halving",
            call: "f(n/2) twice",
            f: binary_halving,
            time: "Theta(n)",
            space: "Theta(log n)",
        },
        Shape {
            name: "binary decrement",
            call: "f(n-1), f(n-2)",
            f: binary_decrement,
            time: "Theta(phi^n)",
            space: "Theta(n)",
        },
    ];

    println!(
        "{:<18} {:<16} {:>12} {:>12}   {:<14} space",
        "shape", "recursive call", "calls", "max depth", "time"
    );
    for sh in &shapes {
        let s = measure(sh.f, 24);
        println!(
            "{:<18} {:<16} {:>12} {:>12}   {:<14} {}",
            sh.name, sh.call, s.calls, s.max_depth, sh.time, sh.space
        );
    }

    println!();
    println!("n = 24 for all four. Read the two middle columns as separate questions:");
    println!();
    println!("  'binary halving' makes 31 calls but reaches depth 4. Lots of work,");
    println!("     hardly any stack. This is divide & conquer, and it is why merge");
    println!("     sort is Theta(n log n) TIME and Theta(log n) STACK (example 13).");
    println!();
    println!("  'binary decrement' makes 150,049 calls at depth 23. Enormous work,");
    println!("     modest stack -- and the work is the problem (memoize it, example 8).");
    println!();
    println!("  'linear' makes only 25 calls but reaches depth 24: the stack IS the");
    println!("     input. Cheap in time, and the one shape that overflows on real data.");

    println!();
    println!("watch the calls column explode with n for the last shape:");
    for n in [10, 20, 30, 35] {
        let s = measure(binary_decrement, n);
        println!("  n={n:>2}  calls={:>10}  depth={}", s.calls, s.max_depth);
    }
}
```

**Rust output:**

```
shape              recursive call          calls    max depth   time           space
linear             f(n-1)                     25           24   Theta(n)       Theta(n)
halving            f(n/2)                      5            4   Theta(log n)   Theta(log n)
binary halving     f(n/2) twice               31            4   Theta(n)       Theta(log n)
binary decrement   f(n-1), f(n-2)         150049           23   Theta(phi^n)   Theta(n)

n = 24 for all four. Read the two middle columns as separate questions:

  'binary halving' makes 31 calls but reaches depth 4. Lots of work,
     hardly any stack. This is divide & conquer, and it is why merge
     sort is Theta(n log n) TIME and Theta(log n) STACK (example 13).

  'binary decrement' makes 150,049 calls at depth 23. Enormous work,
     modest stack -- and the work is the problem (memoize it, example 8).

  'linear' makes only 25 calls but reaches depth 24: the stack IS the
     input. Cheap in time, and the one shape that overflows on real data.

watch the calls column explode with n for the last shape:
  n=10  calls=       177  depth=9
  n=20  calls=     21891  depth=19
  n=30  calls=   2692537  depth=29
  n=35  calls=  29860703  depth=34
```

> **Go vs Rust:** Output is identical to Go: the call and depth counts are properties of the recursion tree, not of the language. The one structural difference is the counters: Go uses package-level `calls` / `maxDepth`, Rust has no mutable globals in safe code, so a `Stats` struct is threaded through every call as `&mut Stats`.

**Complexity:** calls → time, depth → space · `f(n/2)` twice is Θ(n) time and Θ(log n) space; `f(n-1),f(n-2)` is Θ(φⁿ) time and Θ(n) space

---

## 5. Linear recursion is a loop in disguise

`🟢 easy` · *The mechanical conversion*

One recursive call on an input one step smaller always converts to a loop, and the transformation is
mechanical: the base case becomes the initial value, the inductive step becomes the body.

**Steps:**

1. Write four linear recursions and their loops side by side.
2. Confirm they agree.
3. Note which one didn't need converting.

```go
package main

import "fmt"

// LINEAR recursion -- one recursive call, on a problem one step smaller -- is
// always a loop in disguise. The transformation is mechanical:
//
//   1. the base case becomes the loop's INITIAL VALUE
//   2. the inductive step becomes the loop BODY
//   3. the direction of the loop depends on WHERE the work happens:
//        work on the way DOWN  -> loop forwards
//        work on the way UP    -> loop backwards, or carry an accumulator
//
// Do this whenever the depth grows with n. It removes O(n) of stack (lesson 03)
// and the call overhead (example 14), and it cannot overflow.

// --- 1. sum: work happens on the way up ---

func sumRec(xs []int) int {
	if len(xs) == 0 {
		return 0 // base case -> the loop's initial value
	}
	return xs[0] + sumRec(xs[1:])
}

func sumIter(xs []int) int {
	total := 0 // the base case
	for _, v := range xs {
		total += v // the inductive step
	}
	return total
}

// --- 2. factorial: same shape, different operator ---

func factRec(n int) int {
	if n <= 1 {
		return 1
	}
	return n * factRec(n-1)
}

func factIter(n int) int {
	result := 1
	for i := 2; i <= n; i++ {
		result *= i
	}
	return result
}

// --- 3. reverse: the work is a swap, so the loop meets in the middle ---

func reverseRec(xs []int) {
	if len(xs) <= 1 {
		return
	}
	xs[0], xs[len(xs)-1] = xs[len(xs)-1], xs[0]
	reverseRec(xs[1 : len(xs)-1])
}

func reverseIter(xs []int) {
	for i, j := 0, len(xs)-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
	}
}

// --- 4. binary search: already halving, so the loop is flat ---

func searchRec(xs []int, target, lo, hi int) int {
	if lo > hi {
		return -1
	}
	mid := lo + (hi-lo)/2
	switch {
	case xs[mid] == target:
		return mid
	case xs[mid] < target:
		return searchRec(xs, target, mid+1, hi)
	default:
		return searchRec(xs, target, lo, mid-1)
	}
}

func searchIter(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		switch {
		case xs[mid] == target:
			return mid
		case xs[mid] < target:
			lo = mid + 1
		default:
			hi = mid - 1
		}
	}
	return -1
}

func main() {
	xs := []int{3, 1, 4, 1, 5, 9, 2, 6}
	sorted := []int{1, 3, 5, 7, 9, 11}

	fmt.Println("four linear recursions and their loops -- identical results:")
	fmt.Println()
	fmt.Printf("  %-16s recursive %v   iterative %v\n", "sum", sumRec(xs), sumIter(xs))
	fmt.Printf("  %-16s recursive %v   iterative %v\n", "factorial(10)", factRec(10), factIter(10))

	a := append([]int(nil), xs...)
	b := append([]int(nil), xs...)
	reverseRec(a)
	reverseIter(b)
	fmt.Printf("  %-16s recursive %v   iterative %v\n", "reverse", a, b)

	fmt.Printf("  %-16s recursive %v   iterative %v\n", "search(7)",
		searchRec(sorted, 7, 0, len(sorted)-1), searchIter(sorted, 7))

	fmt.Println()
	fmt.Println("the mapping, every time:")
	fmt.Println()
	fmt.Println("  recursive                          iterative")
	fmt.Println("  ---------------------------------  ---------------------------------")
	fmt.Println("  if len(xs) == 0 { return 0 }       total := 0")
	fmt.Println("  return xs[0] + sumRec(xs[1:])      for _, v := range xs { total += v }")
	fmt.Println()
	fmt.Println("  if lo > hi { return -1 }           for lo <= hi {")
	fmt.Println("  return searchRec(..., mid+1, hi)       lo = mid + 1")
	fmt.Println()
	fmt.Println("note the difference between the last two: binary search recurses on a")
	fmt.Println("HALVED input, so its depth is only log n -- there is no stack pressure to")
	fmt.Println("escape. Rewriting it as a loop is a style choice. Rewriting a Theta(n)-deep")
	fmt.Println("recursion as a loop is a correctness decision (example 15).")
	fmt.Println()
	fmt.Println("what does NOT convert this easily: recursion with TWO recursive calls.")
	fmt.Println("For those you need an explicit stack -- example 7.")
}
```

**Output:**

```
four linear recursions and their loops -- identical results:

  sum              recursive 31   iterative 31
  factorial(10)    recursive 3628800   iterative 3628800
  reverse          recursive [6 2 9 5 1 4 1 3]   iterative [6 2 9 5 1 4 1 3]
  search(7)        recursive 3   iterative 3

the mapping, every time:

  recursive                          iterative
  ---------------------------------  ---------------------------------
  if len(xs) == 0 { return 0 }       total := 0
  return xs[0] + sumRec(xs[1:])      for _, v := range xs { total += v }

  if lo > hi { return -1 }           for lo <= hi {
  return searchRec(..., mid+1, hi)       lo = mid + 1

note the difference between the last two: binary search recurses on a
HALVED input, so its depth is only log n -- there is no stack pressure to
escape. Rewriting it as a loop is a style choice. Rewriting a Theta(n)-deep
recursion as a loop is a correctness decision (example 15).

what does NOT convert this easily: recursion with TWO recursive calls.
For those you need an explicit stack -- example 7.
```

The last row is the exception that proves the rule: binary search recurses on a **halved** input, so
its depth is only log n and there's no stack pressure to escape. Rewriting it is a style choice.
Rewriting a Θ(n)-deep recursion is a correctness decision.

**Rust:**

```rust
use std::cmp::Ordering;

// LINEAR recursion -- one recursive call, on a problem one step smaller -- is
// always a loop in disguise. The transformation is mechanical:
//
//   1. the base case becomes the loop's INITIAL VALUE
//   2. the inductive step becomes the loop BODY
//   3. the direction of the loop depends on WHERE the work happens:
//        work on the way DOWN  -> loop forwards
//        work on the way UP    -> loop backwards, or carry an accumulator
//
// Do this whenever the depth grows with n. It removes O(n) of stack (lesson 03)
// and the call overhead (example 14), and it cannot overflow.

// --- 1. sum: work happens on the way up ---

// A slice pattern is the base case and the inductive step in one `match`:
// `[]` is the empty slice, `[first, rest @ ..]` splits off the head.
fn sum_rec(xs: &[i64]) -> i64 {
    match xs {
        [] => 0, // base case -> the loop's initial value
        [first, rest @ ..] => first + sum_rec(rest),
    }
}

fn sum_iter(xs: &[i64]) -> i64 {
    let mut total = 0; // the base case
    for &v in xs {
        total += v; // the inductive step
    }
    total
}

// --- 2. factorial: same shape, different operator ---

fn fact_rec(n: i64) -> i64 {
    if n <= 1 {
        return 1;
    }
    n * fact_rec(n - 1)
}

fn fact_iter(n: i64) -> i64 {
    let mut result = 1;
    for i in 2..=n {
        result *= i;
    }
    result
}

// --- 3. reverse: the work is a swap, so the loop meets in the middle ---

fn reverse_rec(xs: &mut [i64]) {
    if xs.len() <= 1 {
        return;
    }
    let last = xs.len() - 1;
    xs.swap(0, last);
    // the middle is a reborrow of the same memory: exclusive, so the swap
    // above and the recursion below can never touch the same element at once
    reverse_rec(&mut xs[1..last]);
}

fn reverse_iter(xs: &mut [i64]) {
    // Go's `i, j = i+1, j-1` walk is just "swap each element in the first half
    // with its mirror"; no second index to underflow on an empty slice.
    for i in 0..xs.len() / 2 {
        let j = xs.len() - 1 - i;
        xs.swap(i, j);
    }
}

// --- 4. binary search: already halving, so the loop is flat ---

// Indices are usize, so Go's closed range [lo, hi] with `hi = mid - 1` would
// underflow at mid == 0. Rust code uses a half-open range [lo, hi) instead: it
// is empty when lo >= hi, and no step ever subtracts from zero.
// Returns Option<usize> where Go returns -1.
fn search_rec(xs: &[i64], target: i64, lo: usize, hi: usize) -> Option<usize> {
    if lo >= hi {
        return None;
    }
    let mid = lo + (hi - lo) / 2;
    match xs[mid].cmp(&target) {
        Ordering::Equal => Some(mid),
        Ordering::Less => search_rec(xs, target, mid + 1, hi),
        Ordering::Greater => search_rec(xs, target, lo, mid),
    }
}

fn search_iter(xs: &[i64], target: i64) -> Option<usize> {
    let (mut lo, mut hi) = (0, xs.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        match xs[mid].cmp(&target) {
            Ordering::Equal => return Some(mid),
            Ordering::Less => lo = mid + 1,
            Ordering::Greater => hi = mid,
        }
    }
    None
}

// Go's %v prints `[1 2 3]`; Rust's {:?} prints `[1, 2, 3]`. Format by hand.
fn show(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(|v| v.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let xs = [3, 1, 4, 1, 5, 9, 2, 6];
    let sorted = [1, 3, 5, 7, 9, 11];

    println!("four linear recursions and their loops -- identical results:");
    println!();
    println!(
        "  {:<16} recursive {}   iterative {}",
        "sum",
        sum_rec(&xs),
        sum_iter(&xs)
    );
    println!(
        "  {:<16} recursive {}   iterative {}",
        "factorial(10)",
        fact_rec(10),
        fact_iter(10)
    );

    let mut a = xs.to_vec();
    let mut b = xs.to_vec();
    reverse_rec(&mut a);
    reverse_iter(&mut b);
    println!(
        "  {:<16} recursive {}   iterative {}",
        "reverse",
        show(&a),
        show(&b)
    );

    println!(
        "  {:<16} recursive {:?}   iterative {:?}",
        "search(7)",
        search_rec(&sorted, 7, 0, sorted.len()),
        search_iter(&sorted, 7)
    );

    // (recursive, iterative) -- the code the reader just saw, side by side
    let mapping = [
        ("[] => 0,", "let mut total = 0;"),
        (
            "[first, rest @ ..] => first + sum_rec(rest)",
            "for &v in xs { total += v; }",
        ),
        ("", ""),
        ("if lo >= hi { return None; }", "while lo < hi {"),
        (
            "Less => search_rec(.., mid + 1, hi)",
            "    Less => lo = mid + 1,",
        ),
    ];
    println!();
    println!("the mapping, every time:");
    println!();
    println!("  {:<45}  iterative", "recursive");
    println!("  {:-<45}  {:-<33}", "", "");
    for (rec, iter) in mapping {
        if rec.is_empty() {
            println!(); // a blank separator row, without trailing spaces
        } else {
            println!("  {rec:<45}  {iter}");
        }
    }
    println!();
    println!("note the difference between the last two: binary search recurses on a");
    println!("HALVED input, so its depth is only log n -- there is no stack pressure to");
    println!("escape. Rewriting it as a loop is a style choice. Rewriting a Theta(n)-deep");
    println!("recursion as a loop is a correctness decision (example 15).");
    println!();
    println!("what does NOT convert this easily: recursion with TWO recursive calls.");
    println!("For those you need an explicit stack -- example 7.");
}
```

**Rust output:**

```
four linear recursions and their loops -- identical results:

  sum              recursive 31   iterative 31
  factorial(10)    recursive 3628800   iterative 3628800
  reverse          recursive [6 2 9 5 1 4 1 3]   iterative [6 2 9 5 1 4 1 3]
  search(7)        recursive Some(3)   iterative Some(3)

the mapping, every time:

  recursive                                      iterative
  ---------------------------------------------  ---------------------------------
  [] => 0,                                       let mut total = 0;
  [first, rest @ ..] => first + sum_rec(rest)    for &v in xs { total += v; }

  if lo >= hi { return None; }                   while lo < hi {
  Less => search_rec(.., mid + 1, hi)                Less => lo = mid + 1,

note the difference between the last two: binary search recurses on a
HALVED input, so its depth is only log n -- there is no stack pressure to
escape. Rewriting it as a loop is a style choice. Rewriting a Theta(n)-deep
recursion as a loop is a correctness decision (example 15).

what does NOT convert this easily: recursion with TWO recursive calls.
For those you need an explicit stack -- example 7.
```

> **Go vs Rust:** All eight results match Go; the printed differences are Rust's types. Search returns `Option<usize>` (`Some(3)`) where Go returns `-1`, and because indices are `usize` the closed range `[lo, hi]` with `hi = mid - 1` would underflow at `mid == 0`, so both searches use a half-open `[lo, hi)` range (`lo >= hi` is the empty case). The mapping table shows the Rust code: slice patterns (`[] => ..`, `[first, rest @ ..] => ..`) are the base case and inductive step in one `match`, and `&mut xs[1..last]` is an exclusive reborrow, so the swap and the recursion cannot alias.

**Complexity:** all four pairs are identical in time · the recursive versions cost Θ(depth) stack, the loops cost Θ(1)

---

## 6. Go has no tail-call optimisation

`🟢 easy` · *Proven by measurement*

A **tail call** is a recursive call with nothing pending after it. In a language with tail-call
optimisation the frame is reused and the stack stays flat. Go does not do this — and rather than take
that on faith, this measures it.

**Steps:**

1. Write the same sum three ways: work-on-unwind, accumulator (tail), and a loop.
2. Sample stack bytes at maximum depth, in a fresh goroutine.
3. Compare how each column grows with n.

```go
package main

import (
	"fmt"
	"runtime"
)

// A TAIL CALL is a recursive call that is the very last thing a function does --
// nothing is waiting to happen after it returns.
//
//   sumUp:   return xs[0] + sumUp(xs[1:])   NOT a tail call (the + is pending)
//   sumTail: return sumTail(xs[1:], acc+..) IS a tail call (nothing pending)
//
// In a language with tail-call optimisation the second one reuses the current
// frame and runs in constant stack. Go does NOT do this. The frame is kept, the
// stack still grows linearly, and the only thing you gained is a nicer shape to
// convert into a loop by hand.
//
// This example proves it by measuring.

// sumUp does its work while unwinding: every frame must stay alive.
func sumUp(xs []int) int {
	if len(xs) == 0 {
		return 0
	}
	return xs[0] + sumUp(xs[1:])
}

// sumTail carries an accumulator, so nothing is pending after the call.
// This is the classic "tail recursive" rewrite.
func sumTail(xs []int, acc int) int {
	if len(xs) == 0 {
		return acc
	}
	return sumTail(xs[1:], acc+xs[0])
}

// sumLoop is what a TCO-capable compiler would turn sumTail into. In Go you
// write it yourself.
func sumLoop(xs []int) int {
	acc := 0
	for _, v := range xs {
		acc += v
	}
	return acc
}

func stackInUse() uint64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.StackInuse
}

// peakStack runs fn in a FRESH goroutine (which starts with a small stack) and
// reports the stack bytes in use at the moment fn calls the probe.
// Measuring on the main goroutine does not work: Go grows stacks but is slow to
// shrink them, so the first deep call hides every later measurement.
func peakStack(fn func(probe func())) uint64 {
	var peak uint64
	done := make(chan struct{})
	go func() {
		defer close(done)
		fn(func() { peak = stackInUse() })
	}()
	<-done
	runtime.GC()
	return peak
}

// Instrumented copies that call a probe at maximum depth.

func sumUpProbe(xs []int, probe func()) int {
	if len(xs) == 0 {
		probe()
		return 0
	}
	return xs[0] + sumUpProbe(xs[1:], probe)
}

func sumTailProbe(xs []int, acc int, probe func()) int {
	if len(xs) == 0 {
		probe()
		return acc
	}
	return sumTailProbe(xs[1:], acc+xs[0], probe)
}

func sumLoopProbe(xs []int, probe func()) int {
	acc := 0
	for _, v := range xs {
		acc += v
	}
	probe()
	return acc
}

func main() {
	fmt.Println("all three compute the same sum:")
	xs := []int{1, 2, 3, 4, 5}
	fmt.Printf("  sumUp   %d\n  sumTail %d\n  sumLoop %d\n",
		sumUp(xs), sumTail(xs, 0), sumLoop(xs))

	runtime.GC()
	base := stackInUse()

	fmt.Println()
	fmt.Println("stack in use at maximum depth (measured in a fresh goroutine):")
	fmt.Println()
	fmt.Printf("%10s %18s %18s %14s\n", "n", "sumUp (not tail)", "sumTail (tail)", "sumLoop")

	for _, n := range []int{1_000, 10_000, 100_000} {
		data := make([]int, n)
		for i := range data {
			data[i] = 1
		}

		up := peakStack(func(p func()) { sumUpProbe(data, p) })
		tail := peakStack(func(p func()) { sumTailProbe(data, 0, p) })
		loop := peakStack(func(p func()) { sumLoopProbe(data, p) })

		kb := func(v uint64) float64 { return float64(v-base) / 1024 }
		fmt.Printf("%10d %15.0f KB %15.0f KB %11.0f KB\n", n, kb(up), kb(tail), kb(loop))
	}

	fmt.Println()
	fmt.Println("the tail-recursive column grows just like the non-tail one. Go did not")
	fmt.Println("optimise anything away -- there is no tail-call optimisation in gc.")
	fmt.Println("Only the loop stays flat.")
	fmt.Println()
	fmt.Println("(the two recursive columns are not identical: sumTail carries an extra")
	fmt.Println(" int per frame, and the compiler lays the two frames out differently.")
	fmt.Println(" What matters is that BOTH grow with n.)")
	fmt.Println()
	fmt.Println("so what is the accumulator rewrite good for in Go? It is the step that")
	fmt.Println("makes the loop obvious: once nothing is pending after the call, the")
	fmt.Println("recursion IS a loop and you can just write it as one (example 5).")
}
```

**Sample output** (stack accounting moves between runs):

```
all three compute the same sum:
  sumUp   15
  sumTail 15
  sumLoop 15

stack in use at maximum depth (measured in a fresh goroutine):

         n   sumUp (not tail)     sumTail (tail)        sumLoop
      1000              64 KB             128 KB          64 KB
     10000             544 KB            1088 KB          96 KB
    100000            8288 KB            8288 KB         128 KB

the tail-recursive column grows just like the non-tail one. Go did not
optimise anything away -- there is no tail-call optimisation in gc.
Only the loop stays flat.

(the two recursive columns are not identical: sumTail carries an extra
 int per frame, and the compiler lays the two frames out differently.
 What matters is that BOTH grow with n.)

so what is the accumulator rewrite good for in Go? It is the step that
makes the loop obvious: once nothing is pending after the call, the
recursion IS a loop and you can just write it as one (example 5).
```

The tail-recursive column grows just like the non-tail one. Only the loop stays flat. So what *is* the
accumulator rewrite good for in Go? It's the step that makes the loop obvious: once nothing is pending
after the call, the recursion **is** a loop and you can write it as one.

**Rust:**

```rust
use std::cell::Cell;
use std::hint::black_box;
use std::thread;

// A TAIL CALL is a recursive call that is the very last thing a function does --
// nothing is waiting to happen after it returns.
//
//   sum_up:   first + sum_up(rest)         NOT a tail call (the + is pending)
//   sum_tail: sum_tail(rest, acc + first)  IS a tail call (nothing pending)
//
// Go does not eliminate tail calls: the frame is kept, the stack grows
// linearly, and the accumulator rewrite only gives you a nicer shape to turn
// into a loop by hand. Rust does not GUARANTEE elimination either -- but its
// compiler backend, LLVM, may do it as an optimisation. Whether it did is a
// fact about one build, so this example measures instead of assuming.

// The address of a fresh local is a stand-in for the stack pointer: stacks grow
// downwards, so (address at the start) - (address at the deepest call) is the
// number of stack bytes the recursion used. black_box stops the compiler from
// deleting the local.
#[inline(always)]
fn stack_marker() -> usize {
    let marker = 0u8;
    std::ptr::from_ref(black_box(&marker)).addr()
}

// All the sum functions record the marker at their deepest point in `deepest`.
// (Go keeps separate un-instrumented copies; one version is enough here, and
// the "all three agree" check below simply ignores `deepest`.)

// sum_up does its work while unwinding: every frame must stay alive.
fn sum_up(xs: &[i64], deepest: &mut usize) -> i64 {
    match xs {
        [] => {
            *deepest = stack_marker();
            0
        }
        [first, rest @ ..] => first + sum_up(rest, deepest),
    }
}

// sum_tail carries an accumulator, so nothing is pending after the call.
// This is the classic "tail recursive" rewrite.
fn sum_tail(xs: &[i64], acc: i64, deepest: &mut usize) -> i64 {
    match xs {
        [] => {
            *deepest = stack_marker();
            acc
        }
        [first, rest @ ..] => sum_tail(rest, acc + first, deepest),
    }
}

// Rust-only: a local with a destructor makes the recursive call NOT a tail
// call, even though it is the last expression. `_guard` must be dropped AFTER
// the call returns, so the frame has to stay. (Locks, files and Vecs in a
// frame all behave like this.)
struct Guard<'a>(&'a Cell<u64>);

impl Drop for Guard<'_> {
    fn drop(&mut self) {
        self.0.set(self.0.get() + 1);
    }
}

fn sum_tail_guard(xs: &[i64], acc: i64, deepest: &mut usize, drops: &Cell<u64>) -> i64 {
    let _guard = Guard(drops);
    match xs {
        [] => {
            *deepest = stack_marker();
            acc
        }
        [first, rest @ ..] => sum_tail_guard(rest, acc + first, deepest, drops),
    }
}

// sum_loop is what a TCO-capable compiler would turn sum_tail into. In Go you
// write it yourself.
fn sum_loop(xs: &[i64], deepest: &mut usize) -> i64 {
    let mut acc = 0;
    for &v in xs {
        acc += v;
    }
    *deepest = stack_marker();
    acc
}

// peak_stack runs `f` on a FRESH thread and returns the stack bytes in use at
// the moment `f` reaches its deepest point. A new thread plays the role of
// Go's fresh goroutine: its stack starts empty, so earlier measurements cannot
// hide later ones.
//
// The stack size has to be asked for up front -- a Rust thread's stack is a
// fixed-size mapping, it does not grow like a goroutine's. The main thread
// typically gets 8 MB; this one reserves 512 MB, of which only the pages the
// recursion actually touches become real memory.
fn peak_stack(f: impl FnOnce(&mut usize) -> i64 + Send) -> usize {
    thread::scope(|s| {
        thread::Builder::new()
            .stack_size(512 << 20)
            .spawn_scoped(s, || {
                let start = stack_marker();
                let mut deepest = 0;
                black_box(f(&mut deepest));
                start - deepest
            })
            .expect("spawn")
            .join()
            .expect("join")
    })
}

fn main() {
    println!("all three compute the same sum:");
    let xs = [1, 2, 3, 4, 5];
    let drops = Cell::new(0);
    println!("  sum_up         {}", sum_up(&xs, &mut 0));
    println!("  sum_tail       {}", sum_tail(&xs, 0, &mut 0));
    println!(
        "  sum_tail_guard {}   ({} destructors ran, one per frame)",
        sum_tail_guard(&xs, 0, &mut 0, &drops),
        drops.get()
    );
    println!("  sum_loop       {}", sum_loop(&xs, &mut 0));

    println!();
    println!("stack in use at maximum depth (measured on a fresh thread):");
    println!();
    println!(
        "{:>10} {:>18} {:>18} {:>20} {:>12}",
        "n", "sum_up (not tail)", "sum_tail (tail)", "tail + Drop guard", "sum_loop"
    );

    let kb = |bytes: usize| bytes as f64 / 1024.0;
    let mut last = (0, 0, 0, 0);
    let last_n = 1_000_000;
    for n in [1_000, 10_000, 100_000, last_n] {
        let data = vec![1i64; n];
        let data = black_box(data.as_slice());

        let up = peak_stack(|d| sum_up(data, d));
        let tail = peak_stack(|d| sum_tail(data, 0, d));
        let guard = peak_stack(|d| sum_tail_guard(data, 0, d, &Cell::new(0)));
        let lp = peak_stack(|d| sum_loop(data, d));

        println!(
            "{n:>10} {:>15.1} KB {:>15.1} KB {:>17.1} KB {:>9.1} KB",
            kb(up),
            kb(tail),
            kb(guard),
            kb(lp)
        );
        last = (up, tail, guard, lp);
    }

    let per_level = |bytes: usize| bytes as f64 / last_n as f64;
    println!();
    println!(
        "bytes of stack per level at n = {last_n}: sum_up {:.0}, sum_tail {:.0}, tail + Drop guard {:.0}, sum_loop {:.0}",
        per_level(last.0),
        per_level(last.1),
        per_level(last.2),
        per_level(last.3)
    );

    println!();
    println!("in THIS build (rustc -O) the tail-recursive column is flat: the optimiser");
    println!("rewrote sum_tail into a loop, so it costs the same as sum_loop. That is a");
    println!("property of the optimiser, not of the language. Rust does not promise it");
    println!("(there is no guaranteed tail-call elimination; the `become` keyword for");
    println!("it is still experimental), and the same tail-shaped code with a Drop guard");
    println!("in the frame is NOT eliminated: it grows exactly like sum_up.");
    println!();
    // Arithmetic on the measured cost, not a run: nothing is recursed 175k deep here.
    let main_stack_levels = ((8u64 << 20) as f64 / per_level(last.0) / 1000.0).round() * 1000.0;
    println!("the non-tail sum_up is not rewritten here either, so it uses stack linear in");
    println!("n -- and a stack is a fixed size. At the per-level cost above, a typical 8 MB");
    println!(
        "main-thread stack holds about {main_stack_levels:.0} levels of sum_up (arithmetic, not run); Go's"
    );
    println!("goroutine stack grows on demand instead (up to 1 GB), so it goes much deeper.");
    println!();
    println!("so what is the accumulator rewrite good for? The same thing as in Go: it is");
    println!("the step that makes the loop obvious. Once nothing is pending after the call,");
    println!("the recursion IS a loop and you can just write it as one (example 5) -- and");
    println!("then you do not depend on what the optimiser decides to do.");
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O ex06.rs && ./ex06
# for comparison: rustc --edition 2024 -C opt-level=0 ex06.rs -o ex06_O0 && ./ex06_O0
```

**Rust sample output:**

```
all three compute the same sum:
  sum_up         15
  sum_tail       15
  sum_tail_guard 15   (6 destructors ran, one per frame)
  sum_loop       15

stack in use at maximum depth (measured on a fresh thread):

         n  sum_up (not tail)    sum_tail (tail)    tail + Drop guard     sum_loop
      1000            46.9 KB             0.0 KB              46.9 KB       0.0 KB
     10000           468.8 KB             0.0 KB             468.8 KB       0.0 KB
    100000          4687.5 KB             0.0 KB            4687.5 KB       0.0 KB
   1000000         46875.0 KB             0.0 KB           46875.0 KB       0.0 KB

bytes of stack per level at n = 1000000: sum_up 48, sum_tail 0, tail + Drop guard 48, sum_loop 0

in THIS build (rustc -O) the tail-recursive column is flat: the optimiser
rewrote sum_tail into a loop, so it costs the same as sum_loop. That is a
property of the optimiser, not of the language. Rust does not promise it
(there is no guaranteed tail-call elimination; the `become` keyword for
it is still experimental), and the same tail-shaped code with a Drop guard
in the frame is NOT eliminated: it grows exactly like sum_up.

the non-tail sum_up is not rewritten here either, so it uses stack linear in
n -- and a stack is a fixed size. At the per-level cost above, a typical 8 MB
main-thread stack holds about 175000 levels of sum_up (arithmetic, not run); Go's
goroutine stack grows on demand instead (up to 1 GB), so it goes much deeper.

so what is the accumulator rewrite good for? The same thing as in Go: it is
the step that makes the loop obvious. Once nothing is pending after the call,
the recursion IS a loop and you can just write it as one (example 5) -- and
then you do not depend on what the optimiser decides to do.
```

> **Go vs Rust:** Rust contradicts the Go result here: at `rustc -O` LLVM rewrites `sum_tail` into a loop, so its column is flat (0.0 KB) instead of growing like `sum_up` (48 bytes per level). That is an optimisation, not a guarantee: built with `-C opt-level=0` the same program grows (64 / 80 / 144 bytes per level for `sum_up` / `sum_tail` / the Drop guard, `sum_loop` flat), `become` (guaranteed tail calls) is still experimental (E0658 on rustc 1.98.1), and at `-O` the Drop-guard variant keeps its self-call and its 48 bytes per level.
> There is no goroutine-style growable stack: a Rust thread's stack is a fixed size chosen at spawn, so the measuring thread reserves 512 MB, while the main thread's typical 8 MB would run out of `sum_up` stack in the low hundreds of thousands of levels (arithmetic from the measured 48 bytes, not run).

**Complexity:** all three are Θ(n) time · the two recursive versions are Θ(n) stack, the loop is Θ(1)

---

> Next tier: [🟡 medium](2-medium.md) — explicit stacks, memoization, and the ways recursion kills a process.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
