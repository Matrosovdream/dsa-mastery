# Step 03 — Complexity Analysis · 🔴 Hard

Examples **12–16**: the master theorem, quadratics hiding in single loops, and measuring complexity
for real with the lesson-02 harness.

> ⚠️ **Two shapes in this tier.** Examples 12 and 13 are `go run` programs. Examples **14, 15 and 16
> are `go test` packages** — folders with two files each, run with `go test -v`. Each says which.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The master theorem

`🔴 hard` · *Naming the three tree shapes* · `go run`

Example 11 drew three recursion trees and found three shapes. The master theorem is just those three
shapes given numbers: compare `d` (work at the top) against `log_b(a)` (how fast subproblems
multiply). It also shows *why* Karatsuba and Strassen matter — they win purely by making **fewer
recursive calls**, which moves `log_b(a)` down.

**Steps:**

1. Compute `log_b(a)` and compare it with `d`.
2. Apply it to seven classic recurrences.
3. Read the three recurrences it cannot solve, and why.

```go
package main

import (
	"fmt"
	"math"
)

// The master theorem solves T(n) = a*T(n/b) + Theta(n^d) by comparing the work
// created by the recursion (n^log_b(a)) against the work done at the top (n^d).
//
//   d <  log_b(a)  -> Case 1: the LEAVES dominate    -> Theta(n^log_b(a))
//   d == log_b(a)  -> Case 2: every level is equal   -> Theta(n^d * log n)
//   d >  log_b(a)  -> Case 3: the ROOT dominates     -> Theta(n^d)
//
// It is not a magic formula: it is the three shapes a recursion tree can have,
// which you just drew by hand in example 11.

// pow formats n^e the way a human writes it: n^0 is 1, n^1 is n, and an
// exponent that is not a whole number keeps two decimals.
func pow(e float64) string {
	const eps = 1e-9
	switch {
	case math.Abs(e) <= eps:
		return "1"
	case math.Abs(e-1) <= eps:
		return "n"
	case math.Abs(e-math.Round(e)) <= eps:
		return fmt.Sprintf("n^%d", int(math.Round(e)))
	default:
		return fmt.Sprintf("n^%.2f", e)
	}
}

func solve(a, b int, d float64) (which string, result string) {
	crit := math.Log(float64(a)) / math.Log(float64(b)) // log_b(a)

	const eps = 1e-9
	switch {
	case d < crit-eps:
		return "1 (leaves win)", fmt.Sprintf("Theta(%s)", pow(crit))
	case math.Abs(d-crit) <= eps:
		if math.Abs(d) <= eps {
			return "2 (all levels equal)", "Theta(log n)"
		}
		return "2 (all levels equal)", fmt.Sprintf("Theta(%s log n)", pow(d))
	default:
		return "3 (root wins)", fmt.Sprintf("Theta(%s)", pow(d))
	}
}

func main() {
	cases := []struct {
		name string
		a    int
		b    int
		d    float64
		note string
	}{
		{"binary search", 1, 2, 0, "T(n) = T(n/2) + O(1)"},
		{"tree traversal", 2, 2, 0, "T(n) = 2T(n/2) + O(1)"},
		{"merge sort", 2, 2, 1, "T(n) = 2T(n/2) + O(n)"},
		{"Karatsuba multiply", 3, 2, 1, "T(n) = 3T(n/2) + O(n)"},
		{"Strassen matrix", 7, 2, 2, "T(n) = 7T(n/2) + O(n^2)"},
		{"naive block matmul", 8, 2, 2, "T(n) = 8T(n/2) + O(n^2)"},
		{"one recursive call, linear work", 1, 2, 1, "T(n) = T(n/2) + O(n)"},
	}

	fmt.Printf("%-32s %-26s %8s %-22s %s\n", "algorithm", "recurrence", "log_b(a)", "case", "solution")
	for _, c := range cases {
		crit := math.Log(float64(c.a)) / math.Log(float64(c.b))
		which, result := solve(c.a, c.b, c.d)
		fmt.Printf("%-32s %-26s %8.2f %-22s %s\n", c.name, c.note, crit, which, result)
	}

	fmt.Println()
	fmt.Println("read the middle column as 'how fast the subproblems multiply'.")
	fmt.Println("Karatsuba beats naive multiplication (n^1.58 vs n^2) purely by making")
	fmt.Println("3 recursive calls instead of 4 -- log2(3)=1.58 against log2(4)=2.")
	fmt.Println("Strassen does the same to matrices: 7 calls instead of 8, n^2.81 vs n^3.")

	fmt.Println()
	fmt.Println("what the master theorem does NOT cover:")
	fmt.Println("  T(n) = 2T(n/2) + n/log n   -- f(n) sits in the gap between the cases")
	fmt.Println("  T(n) = T(n-1) + n          -- not a/b-shaped at all (see example 11)")
	fmt.Println("  T(n) = 2T(n/2) + 2^n       -- f grows too fast for case 3's regularity condition")
	fmt.Println("for those, draw the tree or expand the recurrence by hand.")
}
```

**Run it:**

```bash
go run .
```

**Output:**

```
algorithm                        recurrence                 log_b(a) case                   solution
binary search                    T(n) = T(n/2) + O(1)           0.00 2 (all levels equal)   Theta(log n)
tree traversal                   T(n) = 2T(n/2) + O(1)          1.00 1 (leaves win)         Theta(n)
merge sort                       T(n) = 2T(n/2) + O(n)          1.00 2 (all levels equal)   Theta(n log n)
Karatsuba multiply               T(n) = 3T(n/2) + O(n)          1.58 1 (leaves win)         Theta(n^1.58)
Strassen matrix                  T(n) = 7T(n/2) + O(n^2)        2.81 1 (leaves win)         Theta(n^2.81)
naive block matmul               T(n) = 8T(n/2) + O(n^2)        3.00 1 (leaves win)         Theta(n^3)
one recursive call, linear work  T(n) = T(n/2) + O(n)           0.00 3 (root wins)          Theta(n)

read the middle column as 'how fast the subproblems multiply'.
Karatsuba beats naive multiplication (n^1.58 vs n^2) purely by making
3 recursive calls instead of 4 -- log2(3)=1.58 against log2(4)=2.
Strassen does the same to matrices: 7 calls instead of 8, n^2.81 vs n^3.

what the master theorem does NOT cover:
  T(n) = 2T(n/2) + n/log n   -- f(n) sits in the gap between the cases
  T(n) = T(n-1) + n          -- not a/b-shaped at all (see example 11)
  T(n) = 2T(n/2) + 2^n       -- f grows too fast for case 3's regularity condition
for those, draw the tree or expand the recurrence by hand.
```

Compare rows 5 and 6: **7 recursive calls instead of 8** takes matrix multiplication from n³ to
n^2.81. That is the entire content of Strassen's algorithm, and the master theorem prices it in one line.

**Rust:**

```rust
// The master theorem solves T(n) = a*T(n/b) + Theta(n^d) by comparing the work
// created by the recursion (n^log_b(a)) against the work done at the top (n^d).
//
//   d <  log_b(a)  -> Case 1: the LEAVES dominate    -> Theta(n^log_b(a))
//   d == log_b(a)  -> Case 2: every level is equal   -> Theta(n^d * log n)
//   d >  log_b(a)  -> Case 3: the ROOT dominates     -> Theta(n^d)
//
// It is not a magic formula: it is the three shapes a recursion tree can have,
// which you just drew by hand in example 11.

const EPS: f64 = 1e-9;

/// The three shapes. Go returns the case as a string; in Rust the case is a
/// type, so a `match` over it is checked for completeness by the compiler.
enum Case {
    LeavesWin,
    AllLevelsEqual,
    RootWins,
}

impl Case {
    fn label(&self) -> &'static str {
        match self {
            Case::LeavesWin => "1 (leaves win)",
            Case::AllLevelsEqual => "2 (all levels equal)",
            Case::RootWins => "3 (root wins)",
        }
    }
}

/// pow formats n^e the way a human writes it: n^0 is 1, n^1 is n, and an
/// exponent that is not a whole number keeps two decimals.
fn pow(e: f64) -> String {
    if e.abs() <= EPS {
        "1".to_string()
    } else if (e - 1.0).abs() <= EPS {
        "n".to_string()
    } else if (e - e.round()).abs() <= EPS {
        format!("n^{}", e.round() as i64)
    } else {
        format!("n^{e:.2}")
    }
}

/// log_b(a): Rust has no two-argument log, so divide natural logs (the same
/// change of base the Go version uses).
fn crit(a: u32, b: u32) -> f64 {
    f64::from(a).ln() / f64::from(b).ln()
}

fn solve(a: u32, b: u32, d: f64) -> (Case, String) {
    let crit = crit(a, b);

    if d < crit - EPS {
        (Case::LeavesWin, format!("Theta({})", pow(crit)))
    } else if (d - crit).abs() <= EPS {
        if d.abs() <= EPS {
            (Case::AllLevelsEqual, "Theta(log n)".to_string())
        } else {
            (Case::AllLevelsEqual, format!("Theta({} log n)", pow(d)))
        }
    } else {
        (Case::RootWins, format!("Theta({})", pow(d)))
    }
}

struct Recurrence {
    name: &'static str,
    a: u32,
    b: u32,
    d: f64,
    note: &'static str,
}

/// T(n) = a*T(n/b) + O(n^d), with a name and the recurrence written out.
fn rec(name: &'static str, a: u32, b: u32, d: f64, note: &'static str) -> Recurrence {
    Recurrence {
        name,
        a,
        b,
        d,
        note,
    }
}

fn main() {
    let cases = [
        rec("binary search", 1, 2, 0.0, "T(n) = T(n/2) + O(1)"),
        rec("tree traversal", 2, 2, 0.0, "T(n) = 2T(n/2) + O(1)"),
        rec("merge sort", 2, 2, 1.0, "T(n) = 2T(n/2) + O(n)"),
        rec("Karatsuba multiply", 3, 2, 1.0, "T(n) = 3T(n/2) + O(n)"),
        rec("Strassen matrix", 7, 2, 2.0, "T(n) = 7T(n/2) + O(n^2)"),
        rec("naive block matmul", 8, 2, 2.0, "T(n) = 8T(n/2) + O(n^2)"),
        rec(
            "one recursive call, linear work",
            1,
            2,
            1.0,
            "T(n) = T(n/2) + O(n)",
        ),
    ];

    println!(
        "{:<32} {:<26} {:>8} {:<22} solution",
        "algorithm", "recurrence", "log_b(a)", "case"
    );
    for c in &cases {
        let (which, result) = solve(c.a, c.b, c.d);
        println!(
            "{:<32} {:<26} {:>8.2} {:<22} {}",
            c.name,
            c.note,
            crit(c.a, c.b),
            which.label(),
            result
        );
    }

    println!();
    println!("read the middle column as 'how fast the subproblems multiply'.");
    println!("Karatsuba beats naive multiplication (n^1.58 vs n^2) purely by making");
    println!("3 recursive calls instead of 4 -- log2(3)=1.58 against log2(4)=2.");
    println!("Strassen does the same to matrices: 7 calls instead of 8, n^2.81 vs n^3.");

    println!();
    println!("what the master theorem does NOT cover:");
    println!("  T(n) = 2T(n/2) + n/log n   -- f(n) sits in the gap between the cases");
    println!("  T(n) = T(n-1) + n          -- not a/b-shaped at all (see example 11)");
    println!("  T(n) = 2T(n/2) + 2^n       -- f grows too fast for case 3's regularity condition");
    println!("for those, draw the tree or expand the recurrence by hand.");
}
```

**Rust output:**

```
algorithm                        recurrence                 log_b(a) case                   solution
binary search                    T(n) = T(n/2) + O(1)           0.00 2 (all levels equal)   Theta(log n)
tree traversal                   T(n) = 2T(n/2) + O(1)          1.00 1 (leaves win)         Theta(n)
merge sort                       T(n) = 2T(n/2) + O(n)          1.00 2 (all levels equal)   Theta(n log n)
Karatsuba multiply               T(n) = 3T(n/2) + O(n)          1.58 1 (leaves win)         Theta(n^1.58)
Strassen matrix                  T(n) = 7T(n/2) + O(n^2)        2.81 1 (leaves win)         Theta(n^2.81)
naive block matmul               T(n) = 8T(n/2) + O(n^2)        3.00 1 (leaves win)         Theta(n^3)
one recursive call, linear work  T(n) = T(n/2) + O(n)           0.00 3 (root wins)          Theta(n)

read the middle column as 'how fast the subproblems multiply'.
Karatsuba beats naive multiplication (n^1.58 vs n^2) purely by making
3 recursive calls instead of 4 -- log2(3)=1.58 against log2(4)=2.
Strassen does the same to matrices: 7 calls instead of 8, n^2.81 vs n^3.

what the master theorem does NOT cover:
  T(n) = 2T(n/2) + n/log n   -- f(n) sits in the gap between the cases
  T(n) = T(n-1) + n          -- not a/b-shaped at all (see example 11)
  T(n) = 2T(n/2) + 2^n       -- f grows too fast for case 3's regularity condition
for those, draw the tree or expand the recurrence by hand.
```

> **Go vs Rust:** The three shapes become an `enum Case` instead of a string label, so every `match` over it must handle all three. The output is byte-for-byte identical to Go's, including the `{:>8.2}` / `{:<22}` column widths; neither language has a two-argument log, so both divide natural logs.

**Complexity:** the tool itself is Θ(1) · it prices any `a·T(n/b) + Θ(n^d)` recurrence, which covers most divide-and-conquer in [28 — Divide & Conquer](../../28-divide-conquer.md)

---

## 13. Quadratic hiding in a single loop

`🔴 hard` · *The O(n) operation inside the O(n) loop* · `go run`

Four functions, each a single loop — Θ(n) at a glance. Two of them are Θ(n²), because the operation
*inside* the loop is itself Θ(n). This is the most common real-world complexity bug there is, and it
never looks like one.

The diagnostic here is **work per element**: if that number grows with n, the loop is not linear.

**Steps:**

1. Count the bytes/elements copied by each loop body.
2. Divide by n and compare across three sizes — growing or bounded?
3. Confirm on the clock at n=200,000.

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

// Every function here looks like a single loop -- O(n) at a glance. Two of them
// are O(n^2), because the operation INSIDE the loop is itself O(n).
//
// bytesCopied / elemsCopied count the hidden work exactly, no clock involved.

// concatNaive: strings are immutable, so s += x allocates a new string and
// copies everything so far. Copying i bytes on iteration i totals n^2/2.
func concatNaive(n int) (result string, bytesCopied int) {
	s := ""
	for i := 0; i < n; i++ {
		bytesCopied += len(s) // the copy s += "x" performs
		s += "x"
	}
	return s, bytesCopied
}

// concatBuilder: amortized growth, exactly like append. O(n) total.
func concatBuilder(n int) (result string, bytesCopied int) {
	var b strings.Builder
	prev := 0
	for i := 0; i < n; i++ {
		before := b.Cap()
		b.WriteByte('x')
		if b.Cap() != before {
			bytesCopied += prev // only the resizes copy
		}
		prev = b.Len()
	}
	return b.String(), bytesCopied
}

// prependNaive: inserting at the front shifts everything. n^2/2 element moves.
func prependNaive(n int) (out []int, elemsCopied int) {
	var xs []int
	for i := 0; i < n; i++ {
		elemsCopied += len(xs)
		xs = append([]int{i}, xs...)
	}
	return xs, elemsCopied
}

// appendThenReverse: build at the cheap end, fix the order once. O(n).
func appendThenReverse(n int) (out []int, elemsCopied int) {
	xs := make([]int, 0, n)
	for i := 0; i < n; i++ {
		xs = append(xs, i) // preallocated: no copies at all
	}
	for i, j := 0, len(xs)-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
		elemsCopied += 2
	}
	return xs, elemsCopied
}

func timeIt(f func()) time.Duration {
	start := time.Now()
	f()
	return time.Since(start)
}

func main() {
	fmt.Println("four one-loop functions. Counting the work INSIDE the loop, as work PER")
	fmt.Println("ELEMENT -- if that column grows with n, the loop is not linear.")
	fmt.Println()
	fmt.Printf("%-22s %14s %14s %14s\n", "function", "n=2000", "n=8000", "n=32000")
	fmt.Printf("%-22s %14s %14s %14s\n", "", "copies/n", "copies/n", "copies/n")

	rows := []struct {
		name string
		f    func(int) int
	}{
		{"s += x", func(n int) int { _, c := concatNaive(n); return c }},
		{"strings.Builder", func(n int) int { _, c := concatBuilder(n); return c }},
		{"append([]T{v}, xs...)", func(n int) int { _, c := prependNaive(n); return c }},
		{"append then reverse", func(n int) int { _, c := appendThenReverse(n); return c }},
	}
	for _, r := range rows {
		perElem := func(n int) float64 { return float64(r.f(n)) / float64(n) }
		fmt.Printf("%-22s %14.2f %14.2f %14.2f\n", r.name, perElem(2000), perElem(8000), perElem(32000))
	}

	fmt.Println()
	fmt.Println("two of these loops are O(n^2), and nothing in their shape says so:")
	fmt.Println("  s += x and prepend    -> copies/n grows in step with n (it IS n/2) -> Theta(n^2)")
	fmt.Println("  Builder and append    -> copies/n stays bounded by a constant       -> Theta(n)")
	fmt.Println()
	fmt.Println("note the Builder's number wobbles (1.7 to 3.7) rather than sitting still.")
	fmt.Println("That is amortized growth: the count depends on where n falls between two")
	fmt.Println("resizes. BOUNDED is the property that matters, not CONSTANT.")

	fmt.Println()
	fmt.Println("the same thing on the clock, at n=200000:")
	const big = 200_000
	fmt.Printf("  s += x                 %12v\n", timeIt(func() { concatNaive(big) }).Round(time.Millisecond))
	fmt.Printf("  strings.Builder        %12v\n", timeIt(func() { concatBuilder(big) }).Round(time.Microsecond))
	fmt.Printf("  append([]T{v}, xs...)  %12v\n", timeIt(func() { prependNaive(big) }).Round(time.Millisecond))
	fmt.Printf("  append then reverse    %12v\n", timeIt(func() { appendThenReverse(big) }).Round(time.Microsecond))

	fmt.Println()
	fmt.Println("the rule from example 6: a call inside a loop is NESTED, so its cost")
	fmt.Println("MULTIPLIES. To analyse a loop you must know the complexity of every")
	fmt.Println("operation in its body -- including the ones that look like one line.")
	fmt.Println()
	fmt.Println("the two fixes are the same fix: work at the END of the structure,")
	fmt.Println("where amortized growth applies, and reorder once at the end if you must.")
}
```

**Run it:**

```bash
go run .
```

**Sample output** (the counts are exact; the four timings vary):

```
four one-loop functions. Counting the work INSIDE the loop, as work PER
ELEMENT -- if that column grows with n, the loop is not linear.

function                       n=2000         n=8000        n=32000
                             copies/n       copies/n       copies/n
s += x                         999.50        3999.50       15999.50
strings.Builder                  1.66           3.10           3.54
append([]T{v}, xs...)          999.50        3999.50       15999.50
append then reverse              1.00           1.00           1.00

two of these loops are O(n^2), and nothing in their shape says so:
  s += x and prepend    -> copies/n grows in step with n (it IS n/2) -> Theta(n^2)
  Builder and append    -> copies/n stays bounded by a constant       -> Theta(n)

note the Builder's number wobbles (1.7 to 3.7) rather than sitting still.
That is amortized growth: the count depends on where n falls between two
resizes. BOUNDED is the property that matters, not CONSTANT.

the same thing on the clock, at n=200000:
  s += x                        964ms
  strings.Builder               425µs
  append([]T{v}, xs...)        6.971s
  append then reverse           116µs
```

The counting table and the clock agree: **999 → 3,999 → 15,999** (work per element growing in step
with n) versus **1.66 → 3.10 → 3.54** (bounded). And the practical cost of getting it wrong is
**6.971 s versus 116 µs** — a factor of 60,000.

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Every function here looks like a single loop -- O(n) at a glance. Two of them
// are O(n^2), because the operation INSIDE the loop is itself O(n).
//
// bytes_copied / elems_copied count the hidden work exactly, no clock involved.

// concat_naive: Go's `s += "x"` allocates a new string and copies everything so
// far. Rust's String is a growable buffer, so `s += "x"` is just push_str and is
// NOT quadratic -- it is already the Builder. To get Go's behaviour you have to
// ask for a fresh String every time, which is what format!("{s}x") does.
// Copying i bytes on iteration i totals n^2/2.
fn concat_naive(n: usize) -> (String, usize) {
    let mut s = String::new();
    let mut bytes_copied = 0;
    for _ in 0..n {
        bytes_copied += s.len(); // the copy format!("{s}x") performs
        s = format!("{s}x");
    }
    (s, bytes_copied)
}

// concat_builder: amortized growth, exactly like Vec::push. O(n) total.
// Go needs strings.Builder for this; in Rust a plain String already is one.
// This counts a resize as a full copy of what was there. realloc may extend a
// block in place and copy nothing, so treat the count as an upper bound.
fn concat_builder(n: usize) -> (String, usize) {
    let mut s = String::new();
    let mut bytes_copied = 0;
    let mut prev = 0;
    for _ in 0..n {
        let before = s.capacity();
        s.push('x');
        if s.capacity() != before {
            bytes_copied += prev; // only the resizes copy
        }
        prev = s.len();
    }
    (s, bytes_copied)
}

// prepend_naive: inserting at the front shifts everything. n^2/2 element moves.
// Go builds a brand-new slice each time (append([]int{i}, xs...)); Vec::insert
// shifts the elements inside the same buffer with one memmove. The element count
// is the same, the allocation is not.
fn prepend_naive(n: usize) -> (Vec<i64>, usize) {
    let mut xs = Vec::new();
    let mut elems_copied = 0;
    for i in 0..n as i64 {
        elems_copied += xs.len();
        xs.insert(0, i);
    }
    (xs, elems_copied)
}

// push_then_reverse: build at the cheap end, fix the order once. O(n).
fn push_then_reverse(n: usize) -> (Vec<i64>, usize) {
    let mut xs = Vec::with_capacity(n);
    for i in 0..n as i64 {
        xs.push(i); // reserved: no copies at all
    }
    let mut elems_copied = 0;
    let len = xs.len();
    for i in 0..len / 2 {
        xs.swap(i, len - 1 - i);
        elems_copied += 2;
    }
    (xs, elems_copied)
}

fn time_it(f: impl FnOnce()) -> Duration {
    let start = Instant::now();
    f();
    start.elapsed()
}

type Row = (&'static str, fn(usize) -> usize);

fn main() {
    println!("four one-loop functions. Counting the work INSIDE the loop, as work PER");
    println!("ELEMENT -- if that column grows with n, the loop is not linear.");
    println!();
    println!(
        "{:<22} {:>14} {:>14} {:>14}",
        "function", "n=2000", "n=8000", "n=32000"
    );
    println!(
        "{:<22} {:>14} {:>14} {:>14}",
        "", "copies/n", "copies/n", "copies/n"
    );

    let rows: [Row; 4] = [
        ("format!(\"{s}x\")", |n| concat_naive(n).1),
        ("String::push", |n| concat_builder(n).1),
        ("Vec::insert(0, x)", |n| prepend_naive(n).1),
        ("push then reverse", |n| push_then_reverse(n).1),
    ];
    let per_elem = |f: fn(usize) -> usize, n: usize| f(n) as f64 / n as f64;
    for (name, f) in rows {
        println!(
            "{:<22} {:>14.2} {:>14.2} {:>14.2}",
            name,
            per_elem(f, 2000),
            per_elem(f, 8000),
            per_elem(f, 32000)
        );
    }

    println!();
    println!("two of these loops are O(n^2), and nothing in their shape says so:");
    println!(
        "  format! and insert(0)   -> copies/n grows in step with n (it IS n/2) -> Theta(n^2)"
    );
    println!("  String::push, push+rev  -> copies/n stays bounded by a constant       -> Theta(n)");
    println!();
    println!("the String row sits still here only because 2000, 8000 and 32000 all fall at");
    println!("the same point between two doublings. Move n by one, across a doubling:");
    let (below, above) = (per_elem(rows[1].1, 2048), per_elem(rows[1].1, 2049));
    println!("  String::push at n=2048: {below:.2} copies/n,  at n=2049: {above:.2} copies/n");
    println!("That is amortized growth: the count depends on where n falls between two");
    println!("resizes. BOUNDED is the property that matters, not CONSTANT.");

    println!();
    println!("the same thing on the clock, at n=200000:");
    const BIG: usize = 200_000;
    println!(
        "  format!(\"{{s}}x\")       {:>12.1?}",
        time_it(|| _ = black_box(concat_naive(black_box(BIG))))
    );
    println!(
        "  String::push          {:>12.1?}",
        time_it(|| _ = black_box(concat_builder(black_box(BIG))))
    );
    println!(
        "  Vec::insert(0, x)     {:>12.1?}",
        time_it(|| _ = black_box(prepend_naive(black_box(BIG))))
    );
    println!(
        "  push then reverse     {:>12.1?}",
        time_it(|| _ = black_box(push_then_reverse(black_box(BIG))))
    );

    println!();
    println!("the rule from example 6: a call inside a loop is NESTED, so its cost");
    println!("MULTIPLIES. To analyse a loop you must know the complexity of every");
    println!("operation in its body -- including the ones that look like one line.");
    println!();
    println!("the two fixes are the same fix: work at the END of the structure,");
    println!("where amortized growth applies, and reorder once at the end if you must.");
}
```

**Rust sample output:**

```
four one-loop functions. Counting the work INSIDE the loop, as work PER
ELEMENT -- if that column grows with n, the loop is not linear.

function                       n=2000         n=8000        n=32000
                             copies/n       copies/n       copies/n
format!("{s}x")                999.50        3999.50       15999.50
String::push                     1.02           1.02           1.02
Vec::insert(0, x)              999.50        3999.50       15999.50
push then reverse                1.00           1.00           1.00

two of these loops are O(n^2), and nothing in their shape says so:
  format! and insert(0)   -> copies/n grows in step with n (it IS n/2) -> Theta(n^2)
  String::push, push+rev  -> copies/n stays bounded by a constant       -> Theta(n)

the String row sits still here only because 2000, 8000 and 32000 all fall at
the same point between two doublings. Move n by one, across a doubling:
  String::push at n=2048: 1.00 copies/n,  at n=2049: 2.00 copies/n
That is amortized growth: the count depends on where n falls between two
resizes. BOUNDED is the property that matters, not CONSTANT.

the same thing on the clock, at n=200000:
  format!("{s}x")            481.1ms
  String::push                78.3µs
  Vec::insert(0, x)             2.7s
  push then reverse          284.6µs

the rule from example 6: a call inside a loop is NESTED, so its cost
MULTIPLIES. To analyse a loop you must know the complexity of every
operation in its body -- including the ones that look like one line.

the two fixes are the same fix: work at the END of the structure,
where amortized growth applies, and reorder once at the end if you must.
```

> **Go vs Rust:** Go's quadratic `s += x` has no direct Rust spelling: a `String` is a growable buffer, so `s += "x"` is already `push_str` (amortised O(1)) and the Θ(n²) form has to be written `format!("{s}x")`, while `Vec::insert(0, x)` shifts in place with a memmove instead of allocating a new slice. The two quadratic rows count exactly what Go counts (999.50 / 3999.50 / 15999.50); the bounded `String::push` row is 1.02 against the Builder's 1.66 to 3.54 because the count follows the growth policy (these counts fit doubling; Go grows large buffers by less than 2x, plausible but not measured here).
> On the clock at n=200000, three runs gave ~0.45-0.50 s and ~2.6-2.8 s for the quadratic pair (Go sample: 0.96 s, 6.97 s) and ~70-95 µs and ~290-325 µs for the fixed pair, i.e. gaps of roughly 5,000x to 9,000x; the Go sample is a different run, so do not read the cross-language ratios as a language property.

**Complexity:** `s += x` and prepend are Θ(n²) total · `strings.Builder` and preallocated append are Θ(n) · both fixes are the same fix: work at the cheap end of the structure

---

## 14. Measuring a complexity class

`🔴 hard` · *The ratio test on real timings* · `go test`

Example 4 ran the ratio test on exact operation counts. This runs it on **measured nanoseconds**,
using `testing.Benchmark` from lesson 02 — and finds the limit of the technique.

The honest result: timings separate classes that differ by a **factor of n** (log vs linear vs
quadratic) but **cannot separate O(n) from O(n log n)**. Doubling n multiplies an n log n cost by
`2·log(2n)/log(n)` — at n=65536 that's **2.125**, a 6% difference that does not survive contact with a
real CPU.

**Steps:**

1. Benchmark four functions of known complexity at four sizes each.
2. Take the **median** ratio (robust against one noisy run).
3. Classify — and note that two classes have to share a band.

**`algo.go`**

```go
package algo

import "slices"

// Four functions whose complexity we already know. The point of the test is to
// REDISCOVER it from measured timings alone.

// SumAll is Theta(n).
func SumAll(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

// Find is Theta(log n) on a sorted slice.
func Find(xs []int, target int) int {
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

// SortInto is Theta(n log n). It sorts in place so that the benchmark measures
// the sort rather than an allocation.
func SortInto(xs []int) {
	slices.Sort(xs)
}

// CountPairs is Theta(n^2).
func CountPairs(xs []int, target int) int {
	n := 0
	for i := 0; i < len(xs); i++ {
		for j := i + 1; j < len(xs); j++ {
			if xs[i]+xs[j] == target {
				n++
			}
		}
	}
	return n
}
```

**`complexity_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"testing"
)

// nsPerOp runs one benchmark and returns its cost in nanoseconds, keeping the
// fractional part that BenchmarkResult.NsPerOp truncates away.
func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// classify turns a measured doubling-ratio into a complexity class.
//
// Note what is NOT in this list: separate bands for O(n) and O(n log n).
// Doubling n multiplies an O(n) cost by 2 and an O(n log n) cost by
// 2*(log(2n)/log(n)) -- which at n=65536 is 2.125. A 6% difference does not
// survive contact with a real CPU. Timings resolve classes that differ by a
// FACTOR of n; they cannot resolve a log factor. (Operation counts can --
// see example 4, where the same two classes came out 2.00 and 2.29.)
func classify(ratio float64) string {
	switch {
	case ratio < 1.5:
		return "O(log n)"
	case ratio < 3.0:
		return "O(n) or O(n log n)"
	case ratio < 5.0:
		return "O(n^2)"
	default:
		return "worse than quadratic"
	}
}

// TestIdentifyComplexity measures each function at four sizes and reads its
// complexity class off the timings alone -- no looking at the source.
func TestIdentifyComplexity(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))

	ascending := func(n int) []int {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		return xs
	}
	shuffled := func(n int) []int {
		xs := ascending(n)
		rng.Shuffle(n, func(i, j int) { xs[i], xs[j] = xs[j], xs[i] })
		return xs
	}

	cases := []struct {
		name  string
		base  int
		build func(n int) func()
		want  string
		truth string
	}{
		{"SumAll", 4096, func(n int) func() {
			xs := ascending(n)
			return func() { SumAll(xs) }
		}, "O(n) or O(n log n)", "O(n)"},

		{"Find", 4096, func(n int) func() {
			xs := ascending(n)
			return func() { Find(xs, n-1) }
		}, "O(log n)", "O(log n)"},

		// Starts at 32768 on purpose: below ~128 KB of data the whole slice fits
		// in a fast cache level, and crossing that boundary produces a ratio
		// spike that has nothing to do with the algorithm.
		{"SortCopy", 32768, func(n int) func() {
			src := shuffled(n)
			buf := make([]int, n)
			return func() { copy(buf, src); SortInto(buf) }
		}, "O(n) or O(n log n)", "O(n log n)"},

		{"CountPairs", 256, func(n int) func() {
			xs := ascending(n)
			return func() { CountPairs(xs, -1) }
		}, "O(n^2)", "O(n^2)"},
	}

	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			sizes := []int{c.base, c.base * 2, c.base * 4, c.base * 8}

			times := make([]float64, len(sizes))
			for i, n := range sizes {
				times[i] = nsPerOp(c.build(n))
			}

			t.Logf("%-11s %14s %10s", "n", "ns/op", "ratio")
			ratios := make([]float64, 0, len(sizes)-1)
			for i, n := range sizes {
				if i == 0 {
					t.Logf("%-11d %14.1f %10s", n, times[i], "-")
					continue
				}
				r := times[i] / times[i-1]
				ratios = append(ratios, r)
				t.Logf("%-11d %14.1f %10.2f", n, times[i], r)
			}

			// The median is more robust than the mean against one noisy run.
			median := ratios[len(ratios)/2]
			got := classify(median)
			t.Logf("median ratio %.2f -> %s   (actually %s)", median, got, c.truth)

			if got != c.want {
				t.Errorf("classified as %s from a ratio of %.2f, expected %s",
					got, median, c.want)
			}
		})
	}
}
```

**Run it:**

```bash
go test -v .
```

**Sample output** (ns/op varies; the ratios are the point):

```
=== RUN   TestIdentifyComplexity/SumAll
    complexity_test.go:99: n                    ns/op      ratio
    complexity_test.go:103: 4096                 969.9          -
    complexity_test.go:108: 8192                1874.4       1.93
    complexity_test.go:108: 16384               3742.5       2.00
    complexity_test.go:108: 32768               7467.6       2.00
    complexity_test.go:114: median ratio 2.00 -> O(n) or O(n log n)   (actually O(n))
=== RUN   TestIdentifyComplexity/Find
    complexity_test.go:103: 4096                   9.0          -
    complexity_test.go:108: 8192                   9.9       1.10
    complexity_test.go:108: 16384                 10.8       1.09
    complexity_test.go:108: 32768                 11.7       1.09
    complexity_test.go:114: median ratio 1.09 -> O(log n)   (actually O(log n))
=== RUN   TestIdentifyComplexity/SortCopy
    complexity_test.go:103: 32768            1103218.8          -
    complexity_test.go:108: 65536            2612202.1       2.37
    complexity_test.go:108: 131072           5723646.0       2.19
    complexity_test.go:108: 262144          12370146.7       2.16
    complexity_test.go:114: median ratio 2.19 -> O(n) or O(n log n)   (actually O(n log n))
=== RUN   TestIdentifyComplexity/CountPairs
    complexity_test.go:103: 256                 8773.5          -
    complexity_test.go:108: 512                32953.5       3.76
    complexity_test.go:108: 1024              125676.1       3.81
    complexity_test.go:108: 2048              488628.6       3.89
    complexity_test.go:114: median ratio 3.81 -> O(n^2)   (actually O(n^2))
--- PASS: TestIdentifyComplexity (18.89s)
ok  	l03/ex14	19.389s
```

Two things worth noticing:

- **The log signal is real but too small to bet on.** `SortCopy`'s ratios (2.37, 2.19, 2.16) do sit consistently above `SumAll`'s (1.93, 2.00, 2.00) — exactly as theory predicts. It just isn't a big enough margin to assert in a test.
- **Cache cliffs produce fake ratios.** An earlier version of this example started the sort sweep at n=4096 and measured a ratio of **4.07** at n=16384 — which classifies as quadratic. At 16384 ints the data crosses 128 KB and falls out of a fast cache level. Sweeping a wide range and taking the median is the defense.

**Rust:**

```rust
// Rust keeps the code and its test in one file: the four functions come first,
// the test sits in a `#[cfg(test)] mod tests` below them. Build and run:
//   rustc --edition 2024 -O --test ex14.rs -o ex14 && ./ex14
// (`-O` matters: a debug build measures the debug build, not the algorithm.)

use std::cmp::Ordering;

// Four functions whose complexity we already know. The point of the test is to
// REDISCOVER it from measured timings alone.

/// sum_all is Theta(n).
fn sum_all(xs: &[i64]) -> i64 {
    xs.iter().sum()
}

/// find is Theta(log n) on a sorted slice. The search window is the half-open
/// range [lo, hi): with `usize` indexes `hi = mid - 1` would underflow when
/// mid is 0, and a half-open window never needs it. `None` replaces Go's -1.
fn find(xs: &[i64], target: i64) -> Option<usize> {
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

/// sort_into is Theta(n log n). It sorts in place so that the benchmark
/// measures the sort rather than an allocation. `sort_unstable` is the closest
/// relative of Go's `slices.Sort` (a pattern-defeating quicksort).
fn sort_into(xs: &mut [i64]) {
    xs.sort_unstable();
}

/// count_pairs is Theta(n^2).
fn count_pairs(xs: &[i64], target: i64) -> usize {
    let mut n = 0;
    for (i, &a) in xs.iter().enumerate() {
        for &b in &xs[i + 1..] {
            if a + b == target {
                n += 1;
            }
        }
    }
    n
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::hint::black_box;
    use std::time::{Duration, Instant};

    // Go's t.Logf shows up under `go test -v`. libtest captures every println!
    // of a passing test and prints it only with `-- --nocapture`, so to keep the
    // numbers visible with no flags this writes straight to the real stdout.
    macro_rules! log {
        ($($arg:tt)*) => {{
            use std::io::Write;
            writeln!(std::io::stdout().lock(), $($arg)*).unwrap();
        }};
    }

    // ---- Rust's stand-ins for testing.Benchmark and math/rand ----

    /// ns_per_op times `f` the way testing.Benchmark does: run it n times,
    /// and if that took less than the target, scale n up and go again.
    /// Stable Rust has no benchmark runner (`#[bench]` is nightly-only;
    /// criterion and divan are the real tools, but they are crates).
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

    /// Noise only ever ADDS time, so of three measurements the fastest is the one
    /// least disturbed by whatever else the machine was doing. Go's benchmark
    /// runs for a full second; this one runs for 200 ms, so it needs the help.
    fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
        (0..3)
            .map(|_| ns_per_op(&mut f))
            .fold(f64::INFINITY, f64::min)
    }

    /// splitmix64: a tiny seeded generator, enough to shuffle a slice.
    struct Rng(u64);
    impl Rng {
        fn next_u64(&mut self) -> u64 {
            self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
            let mut z = self.0;
            z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
            z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
            z ^ (z >> 31)
        }
        fn shuffle<T>(&mut self, xs: &mut [T]) {
            for i in (1..xs.len()).rev() {
                let j = ((self.next_u64() as u128 * (i as u128 + 1)) >> 64) as usize;
                xs.swap(i, j);
            }
        }
    }

    /// classify turns a measured doubling-ratio into a complexity class.
    ///
    /// Note what is NOT in this list: separate bands for O(n) and O(n log n).
    /// Doubling n multiplies an O(n) cost by 2 and an O(n log n) cost by
    /// 2*(log(2n)/log(n)) -- which at n=65536 is 2.125. A 6% difference does not
    /// survive contact with a real CPU. Timings resolve classes that differ by a
    /// FACTOR of n; they cannot resolve a log factor. (Operation counts can --
    /// see example 4, where the same two classes came out 2.00 and 2.29.)
    fn classify(ratio: f64) -> &'static str {
        match ratio {
            r if r < 1.5 => "O(log n)",
            r if r < 3.0 => "O(n) or O(n log n)",
            r if r < 5.0 => "O(n^2)",
            _ => "worse than quadratic",
        }
    }

    /// The middle of the SORTED ratios -- more robust than the mean against one
    /// noisy run.
    fn median(ratios: &[f64]) -> f64 {
        let mut s = ratios.to_vec();
        s.sort_by(f64::total_cmp);
        s[s.len() / 2]
    }

    fn ascending(n: usize) -> Vec<i64> {
        (0..n as i64).collect()
    }

    fn shuffled(n: usize) -> Vec<i64> {
        let mut xs = ascending(n);
        Rng(1).shuffle(&mut xs);
        xs
    }

    struct Case {
        name: &'static str,
        base: usize,
        /// Go's `func(n int) func()`: given a size, build the input and return
        /// the closure to time. Boxed because each case returns a different
        /// closure type.
        build: fn(usize) -> Box<dyn FnMut()>,
        want: &'static str,
        truth: &'static str,
    }

    /// identify_complexity measures each function at four sizes and reads its
    /// complexity class off the timings alone -- no looking at the source.
    ///
    /// libtest has no subtests (Go's t.Run), so the four cases are a loop. A
    /// failed assert would end the whole test at the first bad case, so
    /// mismatches are collected and asserted once at the end -- which is what
    /// Go's non-fatal t.Errorf gives for free.
    #[test]
    fn identify_complexity() {
        let cases = [
            Case {
                name: "SumAll",
                base: 4096,
                build: |n| {
                    let xs = ascending(n);
                    Box::new(move || _ = black_box(sum_all(black_box(&xs))))
                },
                want: "O(n) or O(n log n)",
                truth: "O(n)",
            },
            Case {
                name: "Find",
                base: 4096,
                build: |n| {
                    let xs = ascending(n);
                    let target = n as i64 - 1;
                    Box::new(move || _ = black_box(find(black_box(&xs), black_box(target))))
                },
                want: "O(log n)",
                truth: "O(log n)",
            },
            // Starts at 32768, as in the Go version, to keep the sweep off the
            // small cache-resident sizes: crossing a cache-size boundary can
            // inflate one ratio for reasons that have nothing to do with the
            // algorithm. Measured here: in four sweeps from 1024 to 524288 no
            // sort ratio went above 2.52, so unlike Go's 4.07 this guard is
            // insurance rather than something this machine needed.
            Case {
                name: "SortCopy",
                base: 32768,
                build: |n| {
                    let src = shuffled(n);
                    let mut buf = vec![0; n];
                    Box::new(move || {
                        buf.copy_from_slice(&src);
                        sort_into(black_box(&mut buf));
                    })
                },
                want: "O(n) or O(n log n)",
                truth: "O(n log n)",
            },
            Case {
                name: "CountPairs",
                base: 256,
                build: |n| {
                    let xs = ascending(n);
                    Box::new(move || _ = black_box(count_pairs(black_box(&xs), black_box(-1))))
                },
                want: "O(n^2)",
                truth: "O(n^2)",
            },
        ];

        let mut failures = Vec::new();
        for c in &cases {
            log!("=== {}", c.name);
            let sizes = [c.base, c.base * 2, c.base * 4, c.base * 8];
            let times: Vec<f64> = sizes
                .iter()
                .map(|&n| best_ns_per_op((c.build)(n)))
                .collect();

            log!("    {:<11} {:>14} {:>10}", "n", "ns/op", "ratio");
            log!("    {:<11} {:>14.1} {:>10}", sizes[0], times[0], "-");
            let mut ratios = Vec::new();
            for (i, &n) in sizes.iter().enumerate().skip(1) {
                let r = times[i] / times[i - 1];
                ratios.push(r);
                log!("    {:<11} {:>14.1} {:>10.2}", n, times[i], r);
            }

            let m = median(&ratios);
            let got = classify(m);
            log!("    median ratio {m:.2} -> {got}   (actually {})", c.truth);

            if got != c.want {
                failures.push(format!(
                    "{}: classified as {got} from a ratio of {m:.2}, expected {}",
                    c.name, c.want
                ));
            }
        }
        assert!(failures.is_empty(), "{}", failures.join("\n"));
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O --test ex14.rs -o ex14 && ./ex14
```

**Rust sample output:**

```

running 1 test
=== SumAll
    n                    ns/op      ratio
    4096                 263.0          -
    8192                 515.1       1.96
    16384               1043.9       2.03
    32768               2424.2       2.32
    median ratio 2.03 -> O(n) or O(n log n)   (actually O(n))
=== Find
    n                    ns/op      ratio
    4096                   9.1          -
    8192                  10.7       1.17
    16384                 11.7       1.10
    32768                 12.6       1.08
    median ratio 1.10 -> O(log n)   (actually O(log n))
=== SortCopy
    n                    ns/op      ratio
    32768             259224.5          -
    65536             546908.0       2.11
    131072           1183242.8       2.16
    262144           2678675.0       2.26
    median ratio 2.16 -> O(n) or O(n log n)   (actually O(n log n))
=== CountPairs
    n                    ns/op      ratio
    256                 3959.1          -
    512                14043.8       3.55
    1024               56017.9       3.99
    2048              231485.4       4.13
    median ratio 3.99 -> O(n^2)   (actually O(n^2))
test tests::identify_complexity ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 13.98s
```

> **Go vs Rust:** libtest has no subtests and no `t.Logf`: the four cases are a loop, mismatches are collected and asserted once at the end, and `log!` writes straight to stdout because a plain `println!` in a passing test is captured. Go's `median := ratios[len(ratios)/2]` indexes the unsorted ratios (the second of three); the Rust `median` sorts first. All four functions classify correctly in every run, but the Go prose's "SortCopy's ratios sit consistently above SumAll's" did not hold: over 7 runs SumAll's ratios spanned 1.86-2.44 and SortCopy's 1.81-2.48, and SortCopy's median beat SumAll's in only 6 of 7 runs (2.07-2.28 against 2.03-2.14), so the log factor is, if anything, even too small to assert on here. Go's 4.07 cache-cliff ratio did not reproduce for `sort_unstable` either: four scratch sweeps from n=1024 never went above 2.52.
> Each timing is the best of three 200 ms runs (noise only adds time), and `sort_unstable` stands in for `slices.Sort`.

**Complexity:** the measurement is Θ(sizes) benchmarks · the technique resolves Θ(log n) vs Θ(n) vs Θ(n²) reliably, and Θ(n) vs Θ(n log n) not at all

---

## 15. What amortized O(1) costs the unlucky caller

`🔴 hard` · *Tail latency* · `go test`

Example 8 counted the copies. This puts a clock on them. The amortized cost of an append is a couple
of nanoseconds; the batch containing the last resize takes **hundreds of microseconds**. The ratio of
max to median is **~8,600–14,000×**.

There is a measurement lesson here too: timing individual appends **does not work**. An append costs
~1–2 ns and the system clock resolves ~40 ns, so most samples read zero — and a median of zero makes
every ratio infinite. Batching lifts each sample clear of the clock while still isolating the resize.

**Steps:**

1. Time appends in batches of 100 rather than individually, and explain why.
2. Report p50 / p99 / p99.9 / max, with and without reserved capacity.
3. Assert that the growing slice has a tail and that reserving capacity removes it.

**`append.go`**

```go
package algo

import "time"

// BatchLatencies appends n ints and returns one timing sample per `batch`
// appends. If prealloc is set, the capacity is reserved up front and no resize
// ever happens -- same amortized cost, completely different tail.
//
// Why batches? A single append costs ~1-2 ns, and the system clock resolves
// about 40 ns. Timing them one at a time reports mostly zero, and a median of
// zero makes every ratio infinite. Batching lifts each sample well clear of the
// clock's resolution while still isolating the resize into one sample.
func BatchLatencies(n, batch int, prealloc bool) []time.Duration {
	var xs []int
	if prealloc {
		xs = make([]int, 0, n)
	}

	out := make([]time.Duration, 0, n/batch)
	for i := 0; i < n; i += batch {
		start := time.Now()
		for j := 0; j < batch; j++ {
			xs = append(xs, i+j)
		}
		out = append(out, time.Since(start))
	}
	return out
}
```

**`append_test.go`**

```go
package algo

import (
	"slices"
	"testing"
	"time"
)

func percentile(sorted []time.Duration, p float64) time.Duration {
	i := int(p / 100 * float64(len(sorted)-1))
	return sorted[i]
}

// TestAppendTailLatency measures what "amortized O(1)" costs the unlucky caller.
//
// Example 8 counted the copies. This one puts a clock on them: the amortized
// cost is a couple of nanoseconds per append, and the one batch that triggers
// the final resize takes hundreds of MICROseconds, because it moves the entire
// backing array.
func TestAppendTailLatency(t *testing.T) {
	const (
		n     = 1_000_000
		batch = 100
	)

	type result struct {
		name           string
		p50, p99, p999 time.Duration
		max            time.Duration
		total          time.Duration
		tailRatio      float64
		slowBatches    int
	}

	var results []result

	for _, c := range []struct {
		name     string
		prealloc bool
	}{
		{"growing slice", false},
		{"capacity reserved", true},
	} {
		lat := BatchLatencies(n, batch, c.prealloc)

		var total time.Duration
		for _, d := range lat {
			total += d
		}

		sorted := slices.Clone(lat)
		slices.Sort(sorted)
		p50 := percentile(sorted, 50)
		max := sorted[len(sorted)-1]

		slow := 0
		for _, d := range lat {
			if d > 10*p50 {
				slow++
			}
		}

		results = append(results, result{
			name: c.name, p50: p50,
			p99:         percentile(sorted, 99),
			p999:        percentile(sorted, 99.9),
			max:         max,
			total:       total,
			tailRatio:   float64(max) / float64(p50),
			slowBatches: slow,
		})
	}

	t.Logf("one sample = %d appends, %d samples per run\n", batch, n/batch)
	t.Logf("%-20s %10s %10s %10s %12s %10s %12s",
		"", "p50", "p99", "p99.9", "max", "max/p50", ">10x p50")
	for _, r := range results {
		t.Logf("%-20s %10v %10v %10v %12v %9.0fx %12d",
			r.name, r.p50, r.p99, r.p999, r.max, r.tailRatio, r.slowBatches)
	}
	t.Logf("")
	t.Logf("total wall time for %d appends: growing %v, reserved %v",
		n, results[0].total.Round(time.Millisecond), results[1].total.Round(time.Millisecond))

	growing, reserved := results[0], results[1]

	// The point: the growing slice has a tail, and reserving capacity removes it.
	if growing.tailRatio < 50 {
		t.Errorf("expected the growing slice to show a tail of at least 50x, got %.0fx",
			growing.tailRatio)
	}
	if reserved.tailRatio >= growing.tailRatio {
		t.Errorf("reserving capacity should shrink the tail: growing %.0fx, reserved %.0fx",
			growing.tailRatio, reserved.tailRatio)
	}
}
```

**Run it:**

```bash
go test -v .
```

**Sample output** (timings vary; the shape held across five runs):

```
    append_test.go:74: one sample = 100 appends, 10000 samples per run
    append_test.go:76:                     p50        p99      p99.9          max    max/p50     >10x p50
    append_test.go:78: growing slice              83ns      125ns   94.208µs    742.625µs      8947x           55
    append_test.go:78: capacity reserved          83ns    1.583µs    2.333µs     17.958µs       216x          488
    append_test.go:82: total wall time for 1000000 appends: growing 5ms, reserved 1ms
--- PASS: TestAppendTailLatency (0.12s)
ok  	l03/ex15	0.559s
```

Read the growing-slice row across: **p50 83 ns**, **p99 still only 125 ns**, then **p99.9 jumps to
94 µs** and the max is **742 µs**. That cliff between p99 and p99.9 is the exact shape of a tail
latency problem — invisible in an average, invisible at p99, and the thing your users complain about.

Reserving capacity cuts total time ~5× *and* collapses the tail from 8947× to 216×.

**Rust:**

```rust
// Rust keeps the code and its test in one file: batch_latencies first, the test
// in a `#[cfg(test)] mod tests` below it. Build and run:
//   rustc --edition 2024 -O --test ex15.rs -o ex15 && ./ex15

use std::hint::black_box;
use std::time::{Duration, Instant};

/// How the buffer grows when it is full.
#[derive(Clone, Copy)]
enum Growth {
    /// Plain `Vec::push`. When full, the buffer is handed to `realloc`, which
    /// is free to extend it in place (no copy) or to move it (a copy). Which one
    /// happens is up to the allocator and the address-space layout, not to Vec.
    Push,
    /// `Vec::with_capacity(n)`: reserved up front, so no resize ever happens.
    Reserved,
    /// What Go's `append` does every time it runs out of room: allocate a
    /// buffer twice the size, copy the old contents across, drop the old one.
    /// Rust never makes you do this, but it is the copy the example is about.
    Copying,
}

/// batch_latencies pushes n integers and returns one timing sample per `batch`
/// pushes. With `Growth::Reserved` the capacity is held up front and no resize
/// ever happens -- same amortized cost, completely different tail.
///
/// Why batches? A single push costs ~1 ns and the system clock resolves tens of
/// ns (on this Mac an `Instant` ticks every ~42 ns). Timing them one at a time
/// reports mostly zero, and a median of zero makes every ratio infinite.
/// Batching lifts each sample well clear of the clock's resolution while still
/// isolating the resize into one sample.
fn batch_latencies(n: usize, batch: usize, growth: Growth) -> Vec<Duration> {
    let mut xs: Vec<i64> = match growth {
        Growth::Reserved => Vec::with_capacity(n),
        Growth::Push | Growth::Copying => Vec::new(),
    };

    let mut out = Vec::with_capacity(n / batch);
    for i in (0..n).step_by(batch) {
        let start = Instant::now();
        for j in 0..batch {
            if matches!(growth, Growth::Copying) && xs.len() == xs.capacity() {
                let mut bigger = Vec::with_capacity((2 * xs.capacity()).max(4));
                bigger.extend_from_slice(&xs);
                xs = bigger; // the old buffer is dropped here
            }
            xs.push((i + j) as i64);
        }
        // `xs` is never read, so without this the optimizer is free to delete
        // the pushes (Go's compiler cannot, which is why Go needs no such line).
        black_box(&xs);
        out.push(start.elapsed());
    }
    out
}

#[cfg(test)]
mod tests {
    use super::*;

    // Go's t.Logf shows up under `go test -v`. libtest captures every println!
    // of a passing test and prints it only with `-- --nocapture`, so to keep the
    // numbers visible with no flags this writes straight to the real stdout.
    macro_rules! log {
        ($($arg:tt)*) => {{
            use std::io::Write;
            writeln!(std::io::stdout().lock(), $($arg)*).unwrap();
        }};
    }

    fn percentile(sorted: &[Duration], p: f64) -> Duration {
        sorted[(p / 100.0 * (sorted.len() - 1) as f64) as usize]
    }

    struct Summary {
        name: &'static str,
        p50: Duration,
        p99: Duration,
        p999: Duration,
        max: Duration,
        total: Duration,
        tail_ratio: f64,
        slow_batches: usize,
    }

    fn summarize(name: &'static str, lat: &[Duration]) -> Summary {
        let total: Duration = lat.iter().sum();

        let mut sorted = lat.to_vec();
        sorted.sort();
        let p50 = percentile(&sorted, 50.0);
        let max = *sorted.last().unwrap();

        let slow_batches = lat.iter().filter(|&&d| d > 10 * p50).count();

        Summary {
            name,
            p50,
            p99: percentile(&sorted, 99.0),
            p999: percentile(&sorted, 99.9),
            max,
            total,
            tail_ratio: max.as_secs_f64() / p50.as_secs_f64(),
            slow_batches,
        }
    }

    /// append_tail_latency measures what "amortized O(1)" costs the unlucky caller.
    ///
    /// Example 8 counted the copies. This one puts a clock on them: the amortized
    /// cost is a few nanoseconds per push, and the one batch that triggers a
    /// resize pays for moving the whole backing buffer -- when the resize moves it.
    #[test]
    fn append_tail_latency() {
        const N: usize = 1_000_000;
        const BATCH: usize = 100;
        // Each row is measured three times and the run with the smallest max is
        // kept. Noise only ever ADDS time: on a shared machine one descheduled
        // thread puts a huge max into a row that has no resize in it at all, and
        // that would drown the very thing being measured.
        const RUNS: usize = 3;

        let results: Vec<Summary> = [
            ("growing Vec", Growth::Push),
            ("capacity reserved", Growth::Reserved),
            ("grow by copying", Growth::Copying),
        ]
        .into_iter()
        .map(|(name, growth)| {
            (0..RUNS)
                .map(|_| summarize(name, &batch_latencies(N, BATCH, growth)))
                .min_by_key(|s| s.max)
                .unwrap()
        })
        .collect();

        log!(
            "    one sample = {BATCH} pushes, {} samples per run, best of {RUNS} runs",
            N / BATCH
        );
        log!(
            "    {:<20} {:>10} {:>10} {:>10} {:>12} {:>10} {:>12}",
            "",
            "p50",
            "p99",
            "p99.9",
            "max",
            "max/p50",
            ">10x p50"
        );
        for r in &results {
            log!(
                "    {:<20} {:>10} {:>10} {:>10} {:>12} {:>9.0}x {:>12}",
                r.name,
                format!("{:?}", r.p50),
                format!("{:?}", r.p99),
                format!("{:?}", r.p999),
                format!("{:?}", r.max),
                r.tail_ratio,
                r.slow_batches
            );
        }
        log!();
        log!(
            "    total wall time for {N} pushes: growing {:.1?}, reserved {:.1?}, copying {:.1?}",
            results[0].total,
            results[1].total,
            results[2].total
        );

        let (reserved, copying) = (&results[1], &results[2]);

        // The point: a grow that copies has a tail, and reserving capacity removes
        // it. The plain `Vec::push` row is printed but NOT asserted: whether its
        // resizes copy is decided by realloc, so its tail is not a property of
        // Vec and an assertion on it would be flaky.
        assert!(
            copying.tail_ratio >= 50.0,
            "expected the copying grow to show a tail of at least 50x, got {:.0}x",
            copying.tail_ratio
        );
        assert!(
            reserved.tail_ratio < copying.tail_ratio,
            "reserving capacity should shrink the tail: copying {:.0}x, reserved {:.0}x",
            copying.tail_ratio,
            reserved.tail_ratio
        );
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O --test ex15.rs -o ex15 && ./ex15
```

**Rust sample output:**

```

running 1 test
    one sample = 100 pushes, 10000 samples per run, best of 3 runs
                                p50        p99      p99.9          max    max/p50     >10x p50
    growing Vec                42ns      750ns      834ns      2.542µs        61x          490
    capacity reserved          42ns      709ns      833ns      1.083µs        26x          489
    grow by copying            42ns      792ns    1.125µs     322.75µs      7685x          484

    total wall time for 1000000 pushes: growing 860.4µs, reserved 862.2µs, copying 1.4ms
test tests::append_tail_latency ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

> **Go vs Rust:** Go's `append` always allocates and copies, but `Vec::push` calls `realloc`, which may extend the block in place, so the Go spike is not guaranteed here: in 22 runs of this test the plain `growing Vec` row never had a batch above 126 µs (Go sample: 742 µs), and reserving capacity did not cut its total time (0.76-1.7 ms growing vs 0.73-1.5 ms reserved, against Go's ~5x). The third row, `grow by copying` (a new buffer of twice the size plus `extend_from_slice`, which is what Go does every time), restores the Go result: max 214-698 µs, i.e. 5,100-16,600x p50, in all 22 runs, so that is the row the test asserts on.
> Between 300 and 530 batches (usually about 490) exceed 10x p50 in every row, `capacity reserved` included, so that column is not resize cost here (plausibly first-touch page faults; not measured). An earlier variant of the loop that also grew two bookkeeping `Vec`s next to the buffer did show 0.5-1.1 ms resize batches for the plain `Vec` in 16 of 18 runs outside libtest, so allocator and heap layout are part of this experiment.

**Complexity:** append is Θ(1) amortized either way · the *distribution* is what changes — `make([]T, 0, n)` converts one Θ(n) spike into nothing

---

## 16. A complexity regression gate

`🔴 hard` · *Catching Θ(n) → Θ(n²) in CI* · `go test`

The capstone, and the piece worth keeping. A correctness test **cannot** catch a complexity
regression: both implementations below return byte-identical results on every input. Only a growth
measurement catches it.

The regression is realistic — someone "simplifies" a map away in favour of `slices.Contains`. The code
gets shorter, the tests stay green, and the function becomes quadratic.

**Steps:**

1. Measure doubling ratios with `testing.Benchmark` and take the median.
2. Assert the median falls inside a **wide** band — this is a tripwire, not an instrument.
3. Prove the gate would fire on the regression, and that a correctness test would not.

**`dedupe.go`**

```go
package algo

import "slices"

// Dedupe returns the distinct values of xs, preserving first-seen order.
// The map makes the "have I seen this?" check O(1), so the whole thing is O(n).
func Dedupe(xs []int) []int {
	seen := make(map[int]struct{}, len(xs))
	out := make([]int, 0, len(xs))
	for _, v := range xs {
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		out = append(out, v)
	}
	return out
}

// DedupeSlow is the same function after a well-meaning "simplification":
// the map is gone, and the membership check is a linear scan of the output.
//
// It is shorter, it has no auxiliary structure, it passes every correctness
// test Dedupe passes -- and it is O(n^2). This is the regression the gate below
// exists to catch, because no unit test ever will.
func DedupeSlow(xs []int) []int {
	out := make([]int, 0, len(xs))
	for _, v := range xs {
		if slices.Contains(out, v) { // O(len(out)) inside an O(n) loop
			continue
		}
		out = append(out, v)
	}
	return out
}
```

**`dedupe_test.go`**

```go
package algo

import (
	"slices"
	"testing"
)

// ============================================================================
// A complexity gate. Copy this alongside the lesson-02 harness.
// ============================================================================

// measureGrowth benchmarks build(n) at each size and returns the doubling
// ratios between consecutive sizes.
func measureGrowth(sizes []int, build func(int) func()) []float64 {
	times := make([]float64, len(sizes))
	for i, n := range sizes {
		run := build(n)
		res := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				run()
			}
		})
		times[i] = float64(res.T.Nanoseconds()) / float64(res.N)
	}

	ratios := make([]float64, 0, len(sizes)-1)
	for i := 1; i < len(times); i++ {
		ratios = append(ratios, times[i]/times[i-1])
	}
	return ratios
}

func median(xs []float64) float64 {
	s := slices.Clone(xs)
	slices.Sort(s)
	return s[len(s)/2]
}

// AssertGrowth fails the test if the measured doubling ratio falls outside
// [lo, hi]. Use a WIDE band: this is a tripwire for "somebody made it
// quadratic", not a precision instrument (see example 14 -- timings cannot
// resolve a log factor, so O(n) and O(n log n) share one band).
func AssertGrowth(t *testing.T, name string, sizes []int, build func(int) func(), lo, hi float64) {
	t.Helper()

	ratios := measureGrowth(sizes, build)
	m := median(ratios)
	t.Logf("%-12s sizes %v -> ratios %.2f, median %.2f (allowed %.1f-%.1f)",
		name, sizes, ratios, m, lo, hi)

	if m < lo || m > hi {
		t.Errorf("%s: measured doubling ratio %.2f is outside [%.1f, %.1f] -- "+
			"the complexity is not what this test claims", name, m, lo, hi)
	}
}

// Linear-ish: doubling n should roughly double the time.
const (
	linearLo = 1.5
	linearHi = 3.0
)

func distinctInput(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

// --- the gate, doing its job -------------------------------------------------

func TestDedupeStaysLinear(t *testing.T) {
	AssertGrowth(t, "Dedupe", []int{8192, 16384, 32768, 65536},
		func(n int) func() {
			xs := distinctInput(n)
			return func() { Dedupe(xs) }
		}, linearLo, linearHi)
}

// --- proof that the gate would catch the regression --------------------------

// If DedupeSlow were committed under the name Dedupe, the test above would
// fail. Here we assert that it WOULD -- so this demonstration is itself a
// passing test rather than a broken build.
func TestGateWouldCatchTheRegression(t *testing.T) {
	ratios := measureGrowth([]int{1024, 2048, 4096, 8192},
		func(n int) func() {
			xs := distinctInput(n)
			return func() { DedupeSlow(xs) }
		})
	m := median(ratios)
	t.Logf("DedupeSlow  ratios %.2f, median %.2f", ratios, m)

	if m <= linearHi {
		t.Errorf("expected DedupeSlow to exceed the linear band (%.1f), got %.2f -- "+
			"the gate would have missed the regression", linearHi, m)
	}
	t.Logf("median %.2f > %.1f, so the gate rejects it. A correctness test would not:",
		m, linearHi)

	// ...because the two functions agree on every input.
	for _, n := range []int{0, 1, 5, 50, 500} {
		xs := distinctInput(n)
		xs = append(xs, xs...) // add duplicates
		if !slices.Equal(Dedupe(xs), DedupeSlow(xs)) {
			t.Fatalf("the two implementations disagree at n=%d", n)
		}
	}
	t.Logf("both implementations return identical results on every input tested.")
}
```

**Run it:**

```bash
go test -v .
```

**Sample output** (stable across runs — the ratios moved by ≤0.02):

```
=== RUN   TestDedupeStaysLinear
    dedupe_test.go:74: Dedupe       sizes [8192 16384 32768 65536] -> ratios [2.07 2.10 1.97], median 2.07 (allowed 1.5-3.0)
--- PASS: TestDedupeStaysLinear (4.78s)
=== RUN   TestGateWouldCatchTheRegression
    dedupe_test.go:93: DedupeSlow  ratios [3.84 3.91 3.97], median 3.91
    dedupe_test.go:99: median 3.91 > 3.0, so the gate rejects it. A correctness test would not:
    dedupe_test.go:110: both implementations return identical results on every input tested.
--- PASS: TestGateWouldCatchTheRegression (4.78s)
ok  	l03/ex16	9.967s
```

**2.07 against 3.91.** The two functions are indistinguishable by output and separated by a factor of
almost two in growth rate. That gap is the only signal a test can use, and it is a big one.

Keep the band wide. This exists to catch "someone made it quadratic", not to certify an exact
complexity — example 14 already showed why the latter is not possible from timings.

**Rust:**

```rust
// Rust keeps the code and its tests in one file: the two functions come first,
// the gate and its tests sit in a `#[cfg(test)] mod tests` below them. Build
// and run:
//   rustc --edition 2024 -O --test ex16.rs -o ex16 && ./ex16

use std::collections::HashSet;

/// dedupe returns the distinct values of xs, preserving first-seen order.
/// The set makes the "have I seen this?" check O(1), so the whole thing is O(n).
fn dedupe(xs: &[i64]) -> Vec<i64> {
    let mut seen = HashSet::with_capacity(xs.len());
    let mut out = Vec::with_capacity(xs.len());
    for &v in xs {
        // insert() reports whether the value was new, so one hash answers
        // "seen it?" and records it (Go's map needs a lookup and then a store).
        if seen.insert(v) {
            out.push(v);
        }
    }
    out
}

/// dedupe_slow is the same function after a well-meaning "simplification":
/// the set is gone, and the membership check is a linear scan of the output.
///
/// It is shorter, it has no auxiliary structure, it passes every correctness
/// test dedupe passes -- and it is O(n^2). This is the regression the gate below
/// exists to catch, because no unit test ever will.
fn dedupe_slow(xs: &[i64]) -> Vec<i64> {
    let mut out = Vec::with_capacity(xs.len());
    for &v in xs {
        if out.contains(&v) {
            // O(out.len()) inside an O(n) loop
            continue;
        }
        out.push(v);
    }
    out
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fmt::Display;
    use std::hint::black_box;
    use std::sync::Mutex;
    use std::time::{Duration, Instant};

    // Go's t.Logf shows up under `go test -v`. libtest captures every println!
    // of a passing test and prints it only with `-- --nocapture`, so to keep the
    // numbers visible with no flags this writes straight to the real stdout.
    macro_rules! log {
        ($($arg:tt)*) => {{
            use std::io::Write;
            writeln!(std::io::stdout().lock(), $($arg)*).unwrap();
        }};
    }

    // Go runs the tests of a package one after another. libtest runs them on
    // parallel threads, and two timing tests sharing the CPU spoil each other's
    // numbers. Every timing test holds this lock for its whole run, so they take
    // turns whatever `--test-threads` says. (Which one goes first is up to the
    // scheduler; a poisoned lock just means another test failed, so keep going.)
    static SERIAL: Mutex<()> = Mutex::new(());

    // ============================================================================
    // A complexity gate. Copy this alongside the lesson-02 harness.
    // ============================================================================

    /// ns_per_op times `f` the way testing.Benchmark does: run it n times,
    /// and if that took less than the target, scale n up and go again.
    /// Stable Rust has no benchmark runner (`#[bench]` is nightly-only;
    /// criterion and divan are the real tools, but they are crates).
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

    /// Noise only ever ADDS time, so of three measurements the fastest is the one
    /// least disturbed by whatever else the machine was doing. Go's benchmark
    /// runs for a full second; this one runs for 200 ms, so it needs the help.
    fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
        (0..3)
            .map(|_| ns_per_op(&mut f))
            .fold(f64::INFINITY, f64::min)
    }

    /// Go's `func(int) func()`: given a size, build the input and return the
    /// closure to time. Boxed because every caller returns a different closure.
    type Build = fn(usize) -> Box<dyn FnMut()>;

    /// measure_growth benchmarks build(n) at each size and returns the doubling
    /// ratios between consecutive sizes.
    fn measure_growth(sizes: &[usize], build: Build) -> Vec<f64> {
        let times: Vec<f64> = sizes.iter().map(|&n| best_ns_per_op(build(n))).collect();
        times.windows(2).map(|w| w[1] / w[0]).collect()
    }

    fn median(xs: &[f64]) -> f64 {
        let mut s = xs.to_vec();
        s.sort_by(f64::total_cmp);
        s[s.len() / 2]
    }

    /// `[a b c]`, the way Go prints a slice with %v.
    fn bracket<T: Display>(xs: &[T], fmt: impl Fn(&T) -> String) -> String {
        format!("[{}]", xs.iter().map(fmt).collect::<Vec<_>>().join(" "))
    }

    /// assert_growth fails the test if the measured doubling ratio falls outside
    /// [lo, hi]. Use a WIDE band: this is a tripwire for "somebody made it
    /// quadratic", not a precision instrument (see example 14 -- timings cannot
    /// resolve a log factor, so O(n) and O(n log n) share one band).
    ///
    /// `#[track_caller]` is Go's t.Helper(): a failure is reported at the line
    /// of the test that called this, not at the assert inside it.
    #[track_caller]
    fn assert_growth(name: &str, sizes: &[usize], build: Build, lo: f64, hi: f64) {
        let ratios = measure_growth(sizes, build);
        let m = median(&ratios);
        log!(
            "    {name:<12} sizes {} -> ratios {}, median {m:.2} (allowed {lo:.1}-{hi:.1})",
            bracket(sizes, |n| n.to_string()),
            bracket(&ratios, |r| format!("{r:.2}")),
        );

        assert!(
            (lo..=hi).contains(&m),
            "{name}: measured doubling ratio {m:.2} is outside [{lo:.1}, {hi:.1}] -- \
             the complexity is not what this test claims"
        );
    }

    // Linear-ish: doubling n should roughly double the time.
    const LINEAR_LO: f64 = 1.5;
    const LINEAR_HI: f64 = 3.0;

    fn distinct_input(n: usize) -> Vec<i64> {
        (0..n as i64).collect()
    }

    // --- the gate, doing its job -------------------------------------------------

    #[test]
    fn dedupe_stays_linear() {
        let _turn = SERIAL.lock().unwrap_or_else(|e| e.into_inner());
        assert_growth(
            "dedupe",
            &[8192, 16384, 32768, 65536],
            |n| {
                let xs = distinct_input(n);
                Box::new(move || _ = black_box(dedupe(black_box(&xs))))
            },
            LINEAR_LO,
            LINEAR_HI,
        );
    }

    // --- proof that the gate would catch the regression --------------------------

    // If dedupe_slow were committed under the name dedupe, the test above would
    // fail. Here we assert that it WOULD -- so this demonstration is itself a
    // passing test rather than a broken build.
    #[test]
    fn gate_would_catch_the_regression() {
        let _turn = SERIAL.lock().unwrap_or_else(|e| e.into_inner());
        let ratios = measure_growth(&[1024, 2048, 4096, 8192], |n| {
            let xs = distinct_input(n);
            Box::new(move || _ = black_box(dedupe_slow(black_box(&xs))))
        });
        let m = median(&ratios);
        log!(
            "    dedupe_slow  ratios {}, median {m:.2}",
            bracket(&ratios, |r| format!("{r:.2}"))
        );

        assert!(
            m > LINEAR_HI,
            "expected dedupe_slow to exceed the linear band ({LINEAR_HI:.1}), got {m:.2} -- \
             the gate would have missed the regression"
        );
        log!(
            "    median {m:.2} > {LINEAR_HI:.1}, so the gate rejects it. A correctness test would not:"
        );

        // ...because the two functions agree on every input.
        for n in [0, 1, 5, 50, 500] {
            let mut xs = distinct_input(n);
            xs.extend_from_within(..); // add duplicates
            assert_eq!(
                dedupe(&xs),
                dedupe_slow(&xs),
                "the two implementations disagree at n={n}"
            );
        }
        log!("    both implementations return identical results on every input tested.");
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O --test ex16.rs -o ex16 && ./ex16
```

**Rust sample output:**

```

running 2 tests
    dedupe_slow  ratios [3.67 3.88 3.93], median 3.88
    median 3.88 > 3.0, so the gate rejects it. A correctness test would not:
    both implementations return identical results on every input tested.
test tests::gate_would_catch_the_regression ... ok
    dedupe       sizes [8192 16384 32768 65536] -> ratios [2.06 2.13 2.17], median 2.13 (allowed 1.5-3.0)
test tests::dedupe_stays_linear ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 6.44s
```

> **Go vs Rust:** Same gate, three Rust differences: `HashSet::insert` returns whether the value was new, so one hash replaces Go's lookup-then-store; `#[track_caller]` plays `t.Helper()`; and libtest runs tests on parallel threads where `go test` runs them one after another, so a static `Mutex` makes the two timing tests take turns (which one prints first is up to the scheduler). The gap matches Go's: over 9 runs the medians were 1.84-2.19 for `dedupe` and 3.72-4.29 for `dedupe_slow`, and the two bands never overlapped.
> Go's "stable across runs, ratios moved by <=0.02" did not hold for the Rust runs (individual ratios 1.71-2.41 for `dedupe`, 3.15-4.33 for `dedupe_slow`), but the machine was shared (load average 7-9 on 10 cores), so I do not attribute that to the language. Each timing is the best of three 200 ms runs and the whole file takes about 7 s.

**Complexity:** `Dedupe` Θ(n) time, Θ(n) space · `DedupeSlow` Θ(n²) time, Θ(n) space · the gate costs `len(sizes)` benchmarks ≈ 5 s, cheap enough for CI

---

> That's the lesson. Next: [04 — Go's Memory Model for DSA](../../04-go-memory-model.md), which
> explains the constants this lesson taught you to drop — and why they decided every crossover you
> measured here.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
