# Step 08 — Stacks · 🔴 Hard

Examples **12–16**: the monotonic stack, and knowing when not to use it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

> ⚠️ **Example 16 is a `go test` package** — three files. Examples 12–15 are `go run` programs.
> Sample output from an Apple M4, Go 1.26.3.

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

## 12. The monotonic stack

`🔴 hard` · *Θ(n²) → Θ(n), by the aggregate argument*

The technique this lesson exists to teach. Keep the stack sorted; before pushing, pop everything
that violates the order — and **each pop is an answer**, because the popped element has just found
what it was waiting for.

**Steps:**

1. Write next-greater-element with a stack of indices, and trace it.
2. Verify against brute force.
3. Then benchmark on **random** and on **descending** input, and explain why only one shows the difference.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// The MONOTONIC STACK is the technique this lesson exists to teach. It turns a
// whole family of Theta(n^2) "look at every pair" problems into Theta(n), and
// the argument for why is the aggregate method from lesson 03.
//
// The invariant: the stack is kept sorted (here, strictly decreasing). Before
// pushing a new element you pop everything that violates it -- and each pop is
// an ANSWER, because the popped element has just found the thing it was
// waiting for.
//
// The canonical problem: for each element, find the next element to its right
// that is greater than it.

var comparisons, pops int

// nextGreaterBrute: look right until you find one. Theta(n^2).
func nextGreaterBrute(xs []int) []int {
	out := make([]int, len(xs))
	for i := range xs {
		out[i] = -1
		for j := i + 1; j < len(xs); j++ {
			comparisons++
			if xs[j] > xs[i] {
				out[i] = xs[j]
				break
			}
		}
	}
	return out
}

// nextGreaterMono: one pass, a stack of INDICES still waiting for an answer.
//
// The stack holds indices whose answer is not yet known, and it is always
// decreasing in value -- because the moment a bigger value arrives, everything
// smaller below it has found its answer and leaves.
func nextGreaterMono(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var stack []int // indices, values strictly decreasing from bottom to top

	for i, v := range xs {
		// Everything on the stack smaller than v has just found its answer.
		for len(stack) > 0 {
			comparisons++
			if xs[stack[len(stack)-1]] >= v {
				break
			}
			top := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			pops++
			out[top] = v
		}
		stack = append(stack, i)
	}
	// Whatever is left never found anything bigger; they keep -1.
	return out
}

// The same shape, three variations -- change the comparison and the direction
// and you have a different problem.

func nextSmallerMono(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var stack []int
	for i, v := range xs {
		for len(stack) > 0 && xs[stack[len(stack)-1]] > v {
			out[stack[len(stack)-1]] = v
			stack = stack[:len(stack)-1]
		}
		stack = append(stack, i)
	}
	return out
}

func prevGreaterMono(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var stack []int
	for i := len(xs) - 1; i >= 0; i-- { // walk RIGHT to LEFT
		v := xs[i]
		for len(stack) > 0 && xs[stack[len(stack)-1]] < v {
			out[stack[len(stack)-1]] = v
			stack = stack[:len(stack)-1]
		}
		stack = append(stack, i)
	}
	return out
}

func trace(xs []int) {
	fmt.Printf("  %-6s %-8s %-22s %s\n", "i", "value", "stack (indices)", "answers set")
	var stack []int
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	for i, v := range xs {
		var set []string
		for len(stack) > 0 && xs[stack[len(stack)-1]] < v {
			top := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			out[top] = v
			set = append(set, fmt.Sprintf("out[%d]=%d", top, v))
		}
		stack = append(stack, i)
		s := "-"
		if len(set) > 0 {
			s = fmt.Sprint(set)
		}
		fmt.Printf("  %-6d %-8d %-22s %s\n", i, v, fmt.Sprint(stack), s)
	}
}

var sinkS []int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	xs := []int{2, 1, 2, 4, 3}
	fmt.Printf("next greater element for %v, traced:\n\n", xs)
	trace(xs)
	fmt.Println()
	fmt.Println("  read the stack column: it is ALWAYS decreasing in value. When 4")
	fmt.Println("  arrives at i=3 it pops 2, 1 and 2 -- three answers in one step --")
	fmt.Println("  because every one of them was waiting for exactly this.")

	fmt.Println()
	fmt.Println("brute force and monotonic agree on every input:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	for i := 0; i < 6; i++ {
		in := make([]int, rng.IntN(7)+1)
		for j := range in {
			in[j] = rng.IntN(10)
		}
		a := nextGreaterBrute(slices.Clone(in))
		b := nextGreaterMono(slices.Clone(in))
		fmt.Printf("  %-24v -> %-26v agree: %v\n", in, b, slices.Equal(a, b))
	}

	fmt.Println()
	fmt.Println("WHY it is Theta(n), which is the part worth understanding:")
	fmt.Println()
	fmt.Println("  the inner loop can pop many elements in one step, so the code")
	fmt.Println("  LOOKS quadratic. It is not, by the aggregate argument:")
	fmt.Println()
	fmt.Println("      each index is PUSHED exactly once")
	fmt.Println("      each index is POPPED at most once")
	fmt.Println("      => total inner-loop work over the whole run is at most n")
	fmt.Println()
	fmt.Println("  that is the same reasoning lesson 03 used for append, and lesson")
	fmt.Println("  09's two-stack queue: bound the TOTAL, not the single step.")

	fmt.Println()
	fmt.Println("counted, so it is not just an assertion:")
	fmt.Println()
	fmt.Println("  first on RANDOM data:")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s %14s %12s\n", "n", "brute comparisons", "mono comparisons", "mono pops", "ratio")
	for _, n := range []int{100, 400, 1600, 6400} {
		in := make([]int, n)
		for i := range in {
			in[i] = rng.IntN(1 << 20)
		}
		comparisons, pops = 0, 0
		nextGreaterBrute(in)
		bc := comparisons

		comparisons, pops = 0, 0
		nextGreaterMono(in)
		mc, mp := comparisons, pops

		fmt.Printf("  %10d %18d %18d %14d %11.1fx\n", n, bc, mc, mp, float64(bc)/float64(mc))
	}
	fmt.Println()
	fmt.Println("  the ratio is a small CONSTANT, not growing -- on random data the")
	fmt.Println("  brute force is nearly linear too, because the next greater element")
	fmt.Println("  is usually only a step or two away. Random input does not exercise")
	fmt.Println("  the worst case at all, which is exactly the trap lesson 02 warned")
	fmt.Println("  about: a benchmark on convenient data proves nothing.")

	fmt.Println()
	fmt.Println("  now on DESCENDING data, where nothing ever finds a greater element:")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s %14s %12s\n", "n", "brute comparisons", "mono comparisons", "mono pops", "ratio")
	for _, n := range []int{100, 400, 1600, 6400} {
		in := make([]int, n)
		for i := range in {
			in[i] = n - i // strictly decreasing
		}
		comparisons, pops = 0, 0
		nextGreaterBrute(in)
		bc := comparisons

		comparisons, pops = 0, 0
		nextGreaterMono(in)
		mc, mp := comparisons, pops

		fmt.Printf("  %10d %18d %18d %14d %11.0fx\n", n, bc, mc, mp, float64(bc)/float64(mc))
	}
	fmt.Println()
	fmt.Println("  THERE it is: 4x the input costs the brute force ~16x the")
	fmt.Println("  comparisons (Theta(n^2)) while the monotonic version stays exactly")
	fmt.Println("  linear -- and note mono pops is 0, because nothing is ever popped.")

	fmt.Println()
	fmt.Println("on the clock, on the adversarial input:")
	fmt.Println()
	for _, n := range []int{1000, 4000, 16000} {
		in := make([]int, n)
		for i := range in {
			in[i] = n - i
		}
		tb := nsPerOp(func() { sinkS = nextGreaterBrute(in) })
		tm := nsPerOp(func() { sinkS = nextGreaterMono(in) })
		fmt.Printf("  n=%-8d brute %12.0f ns   mono %10.0f ns   %8.0fx\n", n, tb, tm, tb/tm)
	}

	fmt.Println()
	fmt.Println("four problems, one shape -- change the comparison or the direction:")
	fmt.Println()
	demo := []int{2, 1, 2, 4, 3}
	fmt.Printf("  input          %v\n", demo)
	fmt.Printf("  next greater   %v\n", nextGreaterMono(demo))
	fmt.Printf("  next smaller   %v\n", nextSmallerMono(demo))
	fmt.Printf("  prev greater   %v   (walk right-to-left)\n", prevGreaterMono(demo))
	fmt.Println()
	fmt.Println("  that is the whole family. Examples 13, 14 and 15 are all this")
	fmt.Println("  technique with a different thing stored on the stack.")
}
```

**Sample output:**

```
next greater element for [2 1 2 4 3], traced:

  i      value    stack (indices)        answers set
  0      2        [0]                    -
  1      1        [0 1]                  -
  2      2        [0 2]                  [out[1]=2]
  3      4        [3]                    [out[2]=4 out[0]=4]
  4      3        [3 4]                  -

  read the stack column: it is ALWAYS decreasing in value. When 4
  arrives at i=3 it pops 2, 1 and 2 -- three answers in one step --
  because every one of them was waiting for exactly this.

brute force and monotonic agree on every input:

  [6                        7                        7                        2                        0                        4                       ] -> [7                          -1                         -1                         4                          4                          -1                        ] agree: true
  [1                        1                        6                        4                       ] -> [6                          6                          -1                         -1                        ] agree: true
  [1                        3                        7                        7                        5                        0                        3                       ] -> [3                          7                          -1                         -1                         -1                         3                          -1                        ] agree: true
  [6                       ] -> [-1                        ] agree: true
  [0                        3                       ] -> [3                          -1                        ] agree: true
  [7                        8                        9                        4                        5                       ] -> [8                          9                          -1                         5                          -1                        ] agree: true

WHY it is Theta(n), which is the part worth understanding:

  the inner loop can pop many elements in one step, so the code
  LOOKS quadratic. It is not, by the aggregate argument:

      each index is PUSHED exactly once
      each index is POPPED at most once
      => total inner-loop work over the whole run is at most n

  that is the same reasoning lesson 03 used for append, and lesson
  09's two-stack queue: bound the TOTAL, not the single step.

counted, so it is not just an assertion:

  first on RANDOM data:

           n  brute comparisons   mono comparisons      mono pops        ratio
         100                487                188             92         2.6x
         400               2449                790            391         3.1x
        1600              10762               3180           1590         3.4x
        6400              49790              12784           6391         3.9x

  the ratio is a small CONSTANT, not growing -- on random data the
  brute force is nearly linear too, because the next greater element
  is usually only a step or two away. Random input does not exercise
  the worst case at all, which is exactly the trap lesson 02 warned
  about: a benchmark on convenient data proves nothing.

  now on DESCENDING data, where nothing ever finds a greater element:

           n  brute comparisons   mono comparisons      mono pops        ratio
         100               4950                 99              0          50x
         400              79800                399              0         200x
        1600            1279200               1599              0         800x
        6400           20476800               6399              0        3200x

  THERE it is: 4x the input costs the brute force ~16x the
  comparisons (Theta(n^2)) while the monotonic version stays exactly
  linear -- and note mono pops is 0, because nothing is ever popped.

on the clock, on the adversarial input:

  n=1000     brute       413271 ns   mono       3309 ns        125x
  n=4000     brute      6617596 ns   mono      11056 ns        599x
  n=16000    brute    109116400 ns   mono      50486 ns       2161x

four problems, one shape -- change the comparison or the direction:

  input          [2 1 2 4 3]
  next greater   [4 2 4 -1 -1]
  next smaller   [1 -1 -1 3 -1]
  prev greater   [-1 2 -1 -1 4]   (walk right-to-left)

  that is the whole family. Examples 13, 14 and 15 are all this
  technique with a different thing stored on the stack.
```

**Rust:**

```rust
use std::fmt::Display;
use std::hint::black_box;
use std::time::{Duration, Instant};

// The MONOTONIC STACK is the technique this lesson exists to teach. It turns a
// whole family of Theta(n^2) "look at every pair" problems into Theta(n), and
// the argument for why is the aggregate method from lesson 03.
//
// The invariant: the stack is kept sorted (here, strictly decreasing). Before
// pushing a new element you pop everything that violates it -- and each pop is
// an ANSWER, because the popped element has just found the thing it was
// waiting for.
//
// The canonical problem: for each element, find the next element to its right
// that is greater than it.

// Go counts into package-level `var comparisons, pops int`. Rust has no mutable
// globals, so the counters travel in a struct passed as `&mut Counts`; callers
// that do not care pass a throwaway `&mut Counts::default()`.
#[derive(Default)]
struct Counts {
    comparisons: u64,
    pops: u64,
}

// Go answers with `-1` for "nothing greater to the right". Rust has a type for
// that: `None`. The `-1` only appears when we print.
type Answers = Vec<Option<i64>>;

// next_greater_brute: look right until you find one. Theta(n^2).
fn next_greater_brute(xs: &[i64], c: &mut Counts) -> Answers {
    let mut out = vec![None; xs.len()];
    for (i, &x) in xs.iter().enumerate() {
        for &y in &xs[i + 1..] {
            c.comparisons += 1;
            if y > x {
                out[i] = Some(y);
                break;
            }
        }
    }
    out
}

// next_greater_mono: one pass, a stack of INDICES still waiting for an answer.
//
// The stack holds indices whose answer is not yet known, and it is always
// decreasing in value -- because the moment a bigger value arrives, everything
// smaller below it has found its answer and leaves.
fn next_greater_mono(xs: &[i64], c: &mut Counts) -> Answers {
    let mut out = vec![None; xs.len()];
    let mut stack: Vec<usize> = Vec::new(); // indices, values strictly decreasing from bottom to top

    for (i, &v) in xs.iter().enumerate() {
        // Everything on the stack smaller than v has just found its answer.
        // `stack.last()` is an Option, so Go's `len(stack) > 0 && ...` guard
        // is the `while let` pattern itself.
        while let Some(&top) = stack.last() {
            c.comparisons += 1;
            if xs[top] >= v {
                break;
            }
            stack.pop();
            c.pops += 1;
            out[top] = Some(v);
        }
        stack.push(i);
    }
    // Whatever is left never found anything bigger; they keep None.
    out
}

// The same shape, three variations -- change the comparison and the direction
// and you have a different problem.

fn next_smaller_mono(xs: &[i64]) -> Answers {
    let mut out = vec![None; xs.len()];
    let mut stack: Vec<usize> = Vec::new();
    for (i, &v) in xs.iter().enumerate() {
        // `pop_if` (stable since Rust 1.86) is "peek, test, pop" in one call:
        // it pops the top only if the closure says yes, so there is no
        // `last().unwrap()` followed by a separate `pop()`.
        while let Some(top) = stack.pop_if(|t| xs[*t] > v) {
            out[top] = Some(v);
        }
        stack.push(i);
    }
    out
}

fn prev_greater_mono(xs: &[i64]) -> Answers {
    let mut out = vec![None; xs.len()];
    let mut stack: Vec<usize> = Vec::new();
    for i in (0..xs.len()).rev() {
        // walk RIGHT to LEFT
        let v = xs[i];
        while let Some(top) = stack.pop_if(|t| xs[*t] < v) {
            out[top] = Some(v);
        }
        stack.push(i);
    }
    out
}

// Go prints a slice as [2 1 2 4 3]; Rust's {:?} prints [2, 1, 2, 4, 3]. Format
// by hand so the output reads the same in both languages.
fn go_slice<T: Display>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

// Answers print their None as -1, as Go's slice does.
fn go_answers(xs: &[Option<i64>]) -> String {
    let items: Vec<i64> = xs.iter().map(|a| a.unwrap_or(-1)).collect();
    go_slice(&items)
}

// Go's `%-24v` on a slice applies the width to EACH ELEMENT, not to the whole
// "[...]" -- which is why the "agree" lines below are so wide. Rust's
// `{:<24?}` does the same thing, so to get the same bytes we pad each element
// ourselves. (Pad the finished string instead and the columns line up.)
fn go_padded_each<T: Display>(xs: &[T], width: usize) -> String {
    let items: Vec<String> = xs.iter().map(|x| format!("{x:<width$}")).collect();
    format!("[{}]", items.join(" "))
}

fn trace(xs: &[i64]) {
    println!(
        "  {:<6} {:<8} {:<22} answers set",
        "i", "value", "stack (indices)"
    );
    let mut stack: Vec<usize> = Vec::new();
    for (i, &v) in xs.iter().enumerate() {
        let mut set: Vec<String> = Vec::new();
        while let Some(top) = stack.pop_if(|t| xs[*t] < v) {
            set.push(format!("out[{top}]={v}"));
        }
        stack.push(i);
        let s = if set.is_empty() {
            "-".to_string()
        } else {
            go_slice(&set)
        };
        println!("  {:<6} {:<8} {:<22} {}", i, v, go_slice(&stack), s);
    }
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN, so
// that the same seed (1, 2) gives the same inputs as the Go program.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
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

// ns/op: stand-in for testing.Benchmark / b.Loop.
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
    let xs = [2, 1, 2, 4, 3];
    println!("next greater element for {}, traced:\n", go_slice(&xs));
    trace(&xs);
    println!();
    println!("  read the stack column: it is ALWAYS decreasing in value. When 4");
    println!("  arrives at i=3 it pops 2, 1 and 2 -- three answers in one step --");
    println!("  because every one of them was waiting for exactly this.");

    println!();
    println!("brute force and monotonic agree on every input:");
    println!();
    let mut rng = Pcg::new(1, 2);
    for _ in 0..6 {
        let len = rng.int_n(7) as usize + 1;
        let input: Vec<i64> = (0..len).map(|_| rng.int_n(10) as i64).collect();
        let a = next_greater_brute(&input, &mut Counts::default());
        let b = next_greater_mono(&input, &mut Counts::default());
        let b_shown: Vec<i64> = b.iter().map(|x| x.unwrap_or(-1)).collect();
        println!(
            "  {} -> {} agree: {}",
            go_padded_each(&input, 24),
            go_padded_each(&b_shown, 26),
            a == b
        );
    }

    println!();
    println!("WHY it is Theta(n), which is the part worth understanding:");
    println!();
    println!("  the inner loop can pop many elements in one step, so the code");
    println!("  LOOKS quadratic. It is not, by the aggregate argument:");
    println!();
    println!("      each index is PUSHED exactly once");
    println!("      each index is POPPED at most once");
    println!("      => total inner-loop work over the whole run is at most n");
    println!();
    println!("  that is the same reasoning lesson 03 used for append, and lesson");
    println!("  09's two-stack queue: bound the TOTAL, not the single step.");

    println!();
    println!("counted, so it is not just an assertion:");
    println!();
    println!("  first on RANDOM data:");
    println!();
    println!(
        "  {:>10} {:>18} {:>18} {:>14} {:>12}",
        "n", "brute comparisons", "mono comparisons", "mono pops", "ratio"
    );
    for n in [100, 400, 1600, 6400] {
        let input: Vec<i64> = (0..n).map(|_| rng.int_n(1 << 20) as i64).collect();
        let mut brute = Counts::default();
        next_greater_brute(&input, &mut brute);
        let mut mono = Counts::default();
        next_greater_mono(&input, &mut mono);
        println!(
            "  {:>10} {:>18} {:>18} {:>14} {:>11.1}x",
            n,
            brute.comparisons,
            mono.comparisons,
            mono.pops,
            brute.comparisons as f64 / mono.comparisons as f64
        );
    }
    println!();
    println!("  the ratio is a small CONSTANT, not growing -- on random data the");
    println!("  brute force is nearly linear too, because the next greater element");
    println!("  is usually only a step or two away. Random input does not exercise");
    println!("  the worst case at all, which is exactly the trap lesson 02 warned");
    println!("  about: a benchmark on convenient data proves nothing.");

    println!();
    println!("  now on DESCENDING data, where nothing ever finds a greater element:");
    println!();
    println!(
        "  {:>10} {:>18} {:>18} {:>14} {:>12}",
        "n", "brute comparisons", "mono comparisons", "mono pops", "ratio"
    );
    for n in [100i64, 400, 1600, 6400] {
        let input: Vec<i64> = (0..n).map(|i| n - i).collect(); // strictly decreasing
        let mut brute = Counts::default();
        next_greater_brute(&input, &mut brute);
        let mut mono = Counts::default();
        next_greater_mono(&input, &mut mono);
        println!(
            "  {:>10} {:>18} {:>18} {:>14} {:>11.0}x",
            n,
            brute.comparisons,
            mono.comparisons,
            mono.pops,
            brute.comparisons as f64 / mono.comparisons as f64
        );
    }
    println!();
    println!("  THERE it is: 4x the input costs the brute force ~16x the");
    println!("  comparisons (Theta(n^2)) while the monotonic version stays exactly");
    println!("  linear -- and note mono pops is 0, because nothing is ever popped.");

    println!();
    println!("on the clock, on the adversarial input:");
    println!();
    for n in [1000i64, 4000, 16000] {
        let input: Vec<i64> = (0..n).map(|i| n - i).collect();
        let tb = ns_per_op(|| {
            black_box(next_greater_brute(
                black_box(&input),
                &mut Counts::default(),
            ));
        });
        let tm = ns_per_op(|| {
            black_box(next_greater_mono(black_box(&input), &mut Counts::default()));
        });
        println!(
            "  n={:<8} brute {:>12.0} ns   mono {:>10.0} ns   {:>8.0}x",
            n,
            tb,
            tm,
            tb / tm
        );
    }

    println!();
    println!("four problems, one shape -- change the comparison or the direction:");
    println!();
    let demo = [2, 1, 2, 4, 3];
    println!("  input          {}", go_slice(&demo));
    println!(
        "  next greater   {}",
        go_answers(&next_greater_mono(&demo, &mut Counts::default()))
    );
    println!("  next smaller   {}", go_answers(&next_smaller_mono(&demo)));
    println!(
        "  prev greater   {}   (walk right-to-left)",
        go_answers(&prev_greater_mono(&demo))
    );
    println!();
    println!("  that is the whole family. Examples 13, 14 and 15 are all this");
    println!("  technique with a different thing stored on the stack.");
}
```

**Rust sample output:**

```
next greater element for [2 1 2 4 3], traced:

  i      value    stack (indices)        answers set
  0      2        [0]                    -
  1      1        [0 1]                  -
  2      2        [0 2]                  [out[1]=2]
  3      4        [3]                    [out[2]=4 out[0]=4]
  4      3        [3 4]                  -

  read the stack column: it is ALWAYS decreasing in value. When 4
  arrives at i=3 it pops 2, 1 and 2 -- three answers in one step --
  because every one of them was waiting for exactly this.

brute force and monotonic agree on every input:

  [6                        7                        7                        2                        0                        4                       ] -> [7                          -1                         -1                         4                          4                          -1                        ] agree: true
  [1                        1                        6                        4                       ] -> [6                          6                          -1                         -1                        ] agree: true
  [1                        3                        7                        7                        5                        0                        3                       ] -> [3                          7                          -1                         -1                         -1                         3                          -1                        ] agree: true
  [6                       ] -> [-1                        ] agree: true
  [0                        3                       ] -> [3                          -1                        ] agree: true
  [7                        8                        9                        4                        5                       ] -> [8                          9                          -1                         5                          -1                        ] agree: true

WHY it is Theta(n), which is the part worth understanding:

  the inner loop can pop many elements in one step, so the code
  LOOKS quadratic. It is not, by the aggregate argument:

      each index is PUSHED exactly once
      each index is POPPED at most once
      => total inner-loop work over the whole run is at most n

  that is the same reasoning lesson 03 used for append, and lesson
  09's two-stack queue: bound the TOTAL, not the single step.

counted, so it is not just an assertion:

  first on RANDOM data:

           n  brute comparisons   mono comparisons      mono pops        ratio
         100                487                188             92         2.6x
         400               2449                790            391         3.1x
        1600              10762               3180           1590         3.4x
        6400              49790              12784           6391         3.9x

  the ratio is a small CONSTANT, not growing -- on random data the
  brute force is nearly linear too, because the next greater element
  is usually only a step or two away. Random input does not exercise
  the worst case at all, which is exactly the trap lesson 02 warned
  about: a benchmark on convenient data proves nothing.

  now on DESCENDING data, where nothing ever finds a greater element:

           n  brute comparisons   mono comparisons      mono pops        ratio
         100               4950                 99              0          50x
         400              79800                399              0         200x
        1600            1279200               1599              0         800x
        6400           20476800               6399              0        3200x

  THERE it is: 4x the input costs the brute force ~16x the
  comparisons (Theta(n^2)) while the monotonic version stays exactly
  linear -- and note mono pops is 0, because nothing is ever popped.

on the clock, on the adversarial input:

  n=1000     brute        89214 ns   mono       1253 ns         71x
  n=4000     brute      1342444 ns   mono       4025 ns        333x
  n=16000    brute     20407670 ns   mono      15171 ns       1345x

four problems, one shape -- change the comparison or the direction:

  input          [2 1 2 4 3]
  next greater   [4 2 4 -1 -1]
  next smaller   [1 -1 -1 3 -1]
  prev greater   [-1 2 -1 -1 4]   (walk right-to-left)

  that is the whole family. Examples 13, 14 and 15 are all this
  technique with a different thing stored on the stack.
```

> **Go vs Rust:** Every trace line and counted table is identical to Go's (diffed against a live run of the Go program); only the three timing rows differ. Go's `%-24v` pads each element of a slice, not the whole `[...]`, and Rust's `{:<24?}` does the same, so `go_padded_each` pads element by element to reproduce those wide "agree" lines. `-1` is `None` until it is printed, `Vec::pop_if` replaces peek-then-pop, and the package-level counters become a `&mut Counts`.
> The speedup column is not Go's: at n=16000 the Rust brute force took 20.5 to 25.7 ms across 7 runs against Go's 112 to 114 ms on this machine, and mono took 16 to 26 µs against 52 to 89 µs, so the ratio was about 890x to 1360x where Go showed 2161x. In the assembly of a standalone copy of the brute-force function, Rust's inner loop holds the comparison counter in a register and stores it once after the loop; Go increments a package-level `int` per comparison, a plausible reason for its slower loop (Go's assembly was not inspected).

**Complexity:** Θ(n) — each index is pushed once and popped at most once, so total inner-loop work is bounded by n

---

## 13. The same shape, four more problems

`🔴 hard` · *Recognising it is the skill*

Example 12's fifteen lines with a different thing on the stack. Recognising that these *are* the
same problem is most of the value.

**Steps:**

1. Daily temperatures — store indices, because the answer is a distance.
2. Stock span — the mirror, looking left.
3. Remove-k-digits — a greedy variant where the stack holds the answer.
4. Verify all of them against brute force with a **small** value range, so ties happen.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// Example 12's technique, applied. These are the same fifteen lines with a
// different thing stored on the stack -- and recognising that they ARE the same
// problem is most of the value.
//
// The tell, every time: "for each element, find the nearest one to its
// left/right that is bigger/smaller".

// dailyTemperatures: for each day, how many days until a warmer one?
// Store INDICES so the answer can be a distance rather than a value.
func dailyTemperatures(t []int) []int {
	out := make([]int, len(t))
	var stack []int // indices, temperatures decreasing

	for i, temp := range t {
		for len(stack) > 0 && t[stack[len(stack)-1]] < temp {
			j := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			out[j] = i - j // the DISTANCE, which is why we stored indices
		}
		stack = append(stack, i)
	}
	return out
}

// stockSpan: for each day, how many consecutive days up to and including today
// had a price <= today's? The mirror of the above: look LEFT, and pop while
// the top is smaller.
func stockSpan(prices []int) []int {
	out := make([]int, len(prices))
	var stack []int

	for i, p := range prices {
		for len(stack) > 0 && prices[stack[len(stack)-1]] <= p {
			stack = stack[:len(stack)-1]
		}
		if len(stack) == 0 {
			out[i] = i + 1 // nothing bigger to the left: the span is everything
		} else {
			out[i] = i - stack[len(stack)-1]
		}
		stack = append(stack, i)
	}
	return out
}

// removeKDigits: delete k digits to leave the smallest possible number.
// Greedy + monotonic stack: a digit should go if a SMALLER one follows it.
func removeKDigits(num string, k int) string {
	var stack []byte
	for i := 0; i < len(num); i++ {
		c := num[i]
		for k > 0 && len(stack) > 0 && stack[len(stack)-1] > c {
			stack = stack[:len(stack)-1]
			k--
		}
		stack = append(stack, c)
	}
	stack = stack[:len(stack)-k] // any budget left: drop from the (largest) tail

	i := 0
	for i < len(stack)-1 && stack[i] == '0' { // strip leading zeroes
		i++
	}
	if len(stack) == 0 {
		return "0"
	}
	return string(stack[i:])
}

// --- brute-force oracles, so none of this is taken on faith ---

func dailyTemperaturesBrute(t []int) []int {
	out := make([]int, len(t))
	for i := range t {
		for j := i + 1; j < len(t); j++ {
			if t[j] > t[i] {
				out[i] = j - i
				break
			}
		}
	}
	return out
}

func stockSpanBrute(p []int) []int {
	out := make([]int, len(p))
	for i := range p {
		n := 1
		for j := i - 1; j >= 0 && p[j] <= p[i]; j-- {
			n++
		}
		out[i] = n
	}
	return out
}

func main() {
	temps := []int{73, 74, 75, 71, 69, 72, 76, 73}
	fmt.Println("daily temperatures -- days until a warmer one:")
	fmt.Println()
	fmt.Printf("  temps  %v\n", temps)
	fmt.Printf("  answer %v\n", dailyTemperatures(temps))
	fmt.Println()
	fmt.Println("  the 0s are days with no warmer day ahead -- they are still on the")
	fmt.Println("  stack when the loop ends, and the zero value is already correct.")

	prices := []int{100, 80, 60, 70, 60, 75, 85}
	fmt.Println()
	fmt.Println("stock span -- consecutive days up to today with price <= today's:")
	fmt.Println()
	fmt.Printf("  prices %v\n", prices)
	fmt.Printf("  span   %v\n", stockSpan(prices))

	fmt.Println()
	fmt.Println("both verified against brute force on random input:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	okAll := true
	for i := 0; i < 2000; i++ {
		in := make([]int, rng.IntN(12))
		for j := range in {
			in[j] = rng.IntN(8) // a small range, so ties happen constantly
		}
		if !slices.Equal(dailyTemperatures(in), dailyTemperaturesBrute(in)) {
			fmt.Printf("  MISMATCH dailyTemperatures on %v\n", in)
			okAll = false
			break
		}
		if !slices.Equal(stockSpan(in), stockSpanBrute(in)) {
			fmt.Printf("  MISMATCH stockSpan on %v\n", in)
			okAll = false
			break
		}
	}
	if okAll {
		fmt.Println("  2000 random inputs, both algorithms agree with brute force.")
		fmt.Println("  (a small value range is deliberate -- TIES are where the strict")
		fmt.Println("   vs non-strict comparison goes wrong, and random big numbers")
		fmt.Println("   would almost never produce one.)")
	}

	fmt.Println()
	fmt.Println("a greedy variant -- remove k digits to leave the smallest number:")
	fmt.Println()
	fmt.Printf("  %-14s %-6s %s\n", "input", "k", "result")
	for _, c := range []struct {
		num string
		k   int
	}{
		{"1432219", 3},
		{"10200", 1},
		{"10", 2},
		{"112", 1},
		{"9876543210", 5},
	} {
		fmt.Printf("  %-14s %-6d %s\n", c.num, c.k, removeKDigits(c.num, c.k))
	}
	fmt.Println()
	fmt.Println("  the stack holds the answer being built, kept non-decreasing. A")
	fmt.Println("  digit is removed exactly when a smaller one follows it, which is")
	fmt.Println("  the greedy choice -- and the stack is what makes 'the previous")
	fmt.Println("  digit' available in Theta(1).")

	fmt.Println()
	fmt.Println("recognising the pattern, which is the actual skill:")
	fmt.Println()
	fmt.Printf("  %-42s %s\n", "problem says", "store on the stack")
	for _, r := range [][2]string{
		{"days until a warmer temperature", "indices (you need a distance)"},
		{"how far back prices stayed lower", "indices, popping while <="},
		{"next greater / smaller element", "indices or values"},
		{"smallest number after k removals", "the answer so far, kept sorted"},
		{"largest rectangle (example 14)", "indices of increasing heights"},
		{"water trapped (example 15)", "indices of decreasing heights"},
	} {
		fmt.Printf("  %-42s %s\n", r[0], r[1])
	}
	fmt.Println()
	fmt.Println("  when the answer for an element depends on the nearest larger or")
	fmt.Println("  smaller thing beside it, it is a monotonic stack. Every one of")
	fmt.Println("  these is Theta(n) by the same aggregate argument.")
}
```

**Output:**

```
daily temperatures -- days until a warmer one:

  temps  [73 74 75 71 69 72 76 73]
  answer [1 1 4 2 1 1 0 0]

  the 0s are days with no warmer day ahead -- they are still on the
  stack when the loop ends, and the zero value is already correct.

stock span -- consecutive days up to today with price <= today's:

  prices [100 80 60 70 60 75 85]
  span   [1 1 1 2 1 4 6]

both verified against brute force on random input:

  2000 random inputs, both algorithms agree with brute force.
  (a small value range is deliberate -- TIES are where the strict
   vs non-strict comparison goes wrong, and random big numbers
   would almost never produce one.)

a greedy variant -- remove k digits to leave the smallest number:

  input          k      result
  1432219        3      1219
  10200          1      200
  10             2      0
  112            1      11
  9876543210     5      43210

  the stack holds the answer being built, kept non-decreasing. A
  digit is removed exactly when a smaller one follows it, which is
  the greedy choice -- and the stack is what makes 'the previous
  digit' available in Theta(1).

recognising the pattern, which is the actual skill:

  problem says                               store on the stack
  days until a warmer temperature            indices (you need a distance)
  how far back prices stayed lower           indices, popping while <=
  next greater / smaller element             indices or values
  smallest number after k removals           the answer so far, kept sorted
  largest rectangle (example 14)             indices of increasing heights
  water trapped (example 15)                 indices of decreasing heights

  when the answer for an element depends on the nearest larger or
  smaller thing beside it, it is a monotonic stack. Every one of
  these is Theta(n) by the same aggregate argument.
```

**Rust:**

```rust
use std::fmt::Display;

// Example 12's technique, applied. These are the same fifteen lines with a
// different thing stored on the stack -- and recognising that they ARE the same
// problem is most of the value.
//
// The tell, every time: "for each element, find the nearest one to its
// left/right that is bigger/smaller".

// daily_temperatures: for each day, how many days until a warmer one?
// Store INDICES so the answer can be a distance rather than a value.
//
// The answer is a count of days, never negative, so it is a `usize` and "no
// warmer day" is 0 (a distance of 0 days is not a possible real answer).
fn daily_temperatures(t: &[i64]) -> Vec<usize> {
    let mut out = vec![0; t.len()];
    let mut stack: Vec<usize> = Vec::new(); // indices, temperatures decreasing

    for (i, &temp) in t.iter().enumerate() {
        // `pop_if` pops the top only while the closure says it is colder.
        while let Some(j) = stack.pop_if(|j| t[*j] < temp) {
            out[j] = i - j; // the DISTANCE, which is why we stored indices
        }
        stack.push(i);
    }
    out
}

// stock_span: for each day, how many consecutive days up to and including today
// had a price <= today's? The mirror of the above: look LEFT, and pop while
// the top is smaller.
fn stock_span(prices: &[i64]) -> Vec<usize> {
    let mut out = vec![0; prices.len()];
    let mut stack: Vec<usize> = Vec::new();

    for (i, &p) in prices.iter().enumerate() {
        while stack.last().is_some_and(|&j| prices[j] <= p) {
            stack.pop();
        }
        out[i] = match stack.last() {
            Some(&j) => i - j,
            None => i + 1, // nothing bigger to the left: the span is everything
        };
        stack.push(i);
    }
    out
}

// remove_k_digits: delete k digits to leave the smallest possible number.
// Greedy + monotonic stack: a digit should go if a SMALLER one follows it.
//
// Go indexes the string's bytes; digits are ASCII, so here we walk `chars()`
// and the stack is a Vec<char>. (Go panics on `k > len(num)`; this version
// answers "0", because `saturating_sub` cannot underflow.)
fn remove_k_digits(num: &str, mut k: usize) -> String {
    let mut stack: Vec<char> = Vec::new();
    for c in num.chars() {
        while k > 0 && stack.last().is_some_and(|&top| top > c) {
            stack.pop();
            k -= 1;
        }
        stack.push(c);
    }
    // any budget left: drop from the (largest) tail
    stack.truncate(stack.len().saturating_sub(k));

    // strip leading zeroes; if nothing but zeroes (or nothing) is left, it is "0"
    match stack.iter().position(|&c| c != '0') {
        Some(first) => stack[first..].iter().collect(),
        None => "0".to_string(),
    }
}

// --- brute-force oracles, so none of this is taken on faith ---

fn daily_temperatures_brute(t: &[i64]) -> Vec<usize> {
    let mut out = vec![0; t.len()];
    for (i, &x) in t.iter().enumerate() {
        // position() stops at the first warmer day, like Go's `break`
        if let Some(d) = t[i + 1..].iter().position(|&y| y > x) {
            out[i] = d + 1;
        }
    }
    out
}

fn stock_span_brute(p: &[i64]) -> Vec<usize> {
    (0..p.len())
        .map(|i| 1 + p[..i].iter().rev().take_while(|&&x| x <= p[i]).count())
        .collect()
}

// Go prints a slice as [1 1 4 2]; Rust's {:?} prints [1, 1, 4, 2]. Format by
// hand so the output reads the same in both languages.
fn go_slice<T: Display>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN, so
// that the same seed (1, 2) gives the same 2000 inputs as the Go program.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
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

fn main() {
    let temps = [73, 74, 75, 71, 69, 72, 76, 73];
    println!("daily temperatures -- days until a warmer one:");
    println!();
    println!("  temps  {}", go_slice(&temps));
    println!("  answer {}", go_slice(&daily_temperatures(&temps)));
    println!();
    println!("  the 0s are days with no warmer day ahead -- they are still on the");
    println!("  stack when the loop ends, and the zero value is already correct.");

    let prices = [100, 80, 60, 70, 60, 75, 85];
    println!();
    println!("stock span -- consecutive days up to today with price <= today's:");
    println!();
    println!("  prices {}", go_slice(&prices));
    println!("  span   {}", go_slice(&stock_span(&prices)));

    println!();
    println!("both verified against brute force on random input:");
    println!();
    let mut rng = Pcg::new(1, 2);
    let mut ok_all = true;
    for _ in 0..2000 {
        let len = rng.int_n(12) as usize;
        // a small range, so ties happen constantly
        let input: Vec<i64> = (0..len).map(|_| rng.int_n(8) as i64).collect();
        if daily_temperatures(&input) != daily_temperatures_brute(&input) {
            println!("  MISMATCH daily_temperatures on {}", go_slice(&input));
            ok_all = false;
            break;
        }
        if stock_span(&input) != stock_span_brute(&input) {
            println!("  MISMATCH stock_span on {}", go_slice(&input));
            ok_all = false;
            break;
        }
    }
    if ok_all {
        println!("  2000 random inputs, both algorithms agree with brute force.");
        println!("  (a small value range is deliberate -- TIES are where the strict");
        println!("   vs non-strict comparison goes wrong, and random big numbers");
        println!("   would almost never produce one.)");
    }

    println!();
    println!("a greedy variant -- remove k digits to leave the smallest number:");
    println!();
    println!("  {:<14} {:<6} result", "input", "k");
    for (num, k) in [
        ("1432219", 3),
        ("10200", 1),
        ("10", 2),
        ("112", 1),
        ("9876543210", 5),
    ] {
        println!("  {:<14} {:<6} {}", num, k, remove_k_digits(num, k));
    }
    println!();
    println!("  the stack holds the answer being built, kept non-decreasing. A");
    println!("  digit is removed exactly when a smaller one follows it, which is");
    println!("  the greedy choice -- and the stack is what makes 'the previous");
    println!("  digit' available in Theta(1).");

    println!();
    println!("recognising the pattern, which is the actual skill:");
    println!();
    println!("  {:<42} store on the stack", "problem says");
    for (problem, stores) in [
        (
            "days until a warmer temperature",
            "indices (you need a distance)",
        ),
        (
            "how far back prices stayed lower",
            "indices, popping while <=",
        ),
        ("next greater / smaller element", "indices or values"),
        (
            "smallest number after k removals",
            "the answer so far, kept sorted",
        ),
        (
            "largest rectangle (example 14)",
            "indices of increasing heights",
        ),
        (
            "water trapped (example 15)",
            "indices of decreasing heights",
        ),
    ] {
        println!("  {problem:<42} {stores}");
    }
    println!();
    println!("  when the answer for an element depends on the nearest larger or");
    println!("  smaller thing beside it, it is a monotonic stack. Every one of");
    println!("  these is Theta(n) by the same aggregate argument.");
}
```

**Rust output:**

```
daily temperatures -- days until a warmer one:

  temps  [73 74 75 71 69 72 76 73]
  answer [1 1 4 2 1 1 0 0]

  the 0s are days with no warmer day ahead -- they are still on the
  stack when the loop ends, and the zero value is already correct.

stock span -- consecutive days up to today with price <= today's:

  prices [100 80 60 70 60 75 85]
  span   [1 1 1 2 1 4 6]

both verified against brute force on random input:

  2000 random inputs, both algorithms agree with brute force.
  (a small value range is deliberate -- TIES are where the strict
   vs non-strict comparison goes wrong, and random big numbers
   would almost never produce one.)

a greedy variant -- remove k digits to leave the smallest number:

  input          k      result
  1432219        3      1219
  10200          1      200
  10             2      0
  112            1      11
  9876543210     5      43210

  the stack holds the answer being built, kept non-decreasing. A
  digit is removed exactly when a smaller one follows it, which is
  the greedy choice -- and the stack is what makes 'the previous
  digit' available in Theta(1).

recognising the pattern, which is the actual skill:

  problem says                               store on the stack
  days until a warmer temperature            indices (you need a distance)
  how far back prices stayed lower           indices, popping while <=
  next greater / smaller element             indices or values
  smallest number after k removals           the answer so far, kept sorted
  largest rectangle (example 14)             indices of increasing heights
  water trapped (example 15)                 indices of decreasing heights

  when the answer for an element depends on the nearest larger or
  smaller thing beside it, it is a monotonic stack. Every one of
  these is Theta(n) by the same aggregate argument.
```

> **Go vs Rust:** Output is identical to Go's, byte for byte (also diffed against a live run of the Go program): `Pcg` is a port of Go's `math/rand/v2` PCG-DXSM, so seed (1, 2) feeds both programs the same 2000 inputs. The "ties matter" claim reproduces in Rust: changing `<=` to `<` in `stock_span`, or `<` to `<=` in `daily_temperatures`, makes the check print `MISMATCH` (both on `[4 0 2 0 4 1 6 1 0]`). Distances and spans are `usize` (so `i - j` cannot go negative), the stack top is peeked with `last().is_some_and(..)` or popped with `pop_if`, and Go's `stack[:len(stack)-k]` becomes `truncate(len.saturating_sub(k))`, which answers `"0"` where Go would panic when `k > len(num)`.

**Complexity:** all Θ(n) by the same argument · the tell is "the nearest larger/smaller thing beside it"

---

## 14. Largest rectangle in a histogram

`🔴 hard` · *The payoff*

The hardest of the classics, and the one that justifies the technique. The reframing: every
maximal rectangle is capped in height by exactly one bar, so the question becomes
nearest-smaller-on-each-side — which example 12 already solved.

**Steps:**

1. Keep an increasing stack; a shorter bar arriving finishes the top's rectangle.
2. Use a virtual zero bar at the end to drain the stack with no special case.
3. Then extend it to the 2-D maximal rectangle, one histogram per row.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// The hardest of the classic monotonic-stack problems, and the one that pays
// off the technique: the largest rectangle that fits under a histogram.
//
// The reframing that solves it: every maximal rectangle is limited in HEIGHT by
// exactly one bar. So for each bar, ask "how far left and right can I extend
// while staying at least this tall?" -- and that is next-smaller-to-the-left
// and next-smaller-to-the-right, which is example 12.
//
// A monotonic INCREASING stack computes both in one pass.

var brutePairs int

// brute: try every bar as the limiting height. Theta(n^2).
func largestBrute(h []int) int {
	best := 0
	for i := range h {
		minH := h[i]
		for j := i; j < len(h); j++ {
			brutePairs++
			if h[j] < minH {
				minH = h[j]
			}
			if a := minH * (j - i + 1); a > best {
				best = a
			}
		}
	}
	return best
}

// mono: one pass, a stack of indices with INCREASING heights.
//
// When a bar shorter than the top arrives, the top can extend no further right
// -- so its rectangle is finished, and we can compute it. The left edge is
// whatever is below it on the stack, because everything between them was
// taller (that is the invariant).
func largestMono(h []int) int {
	best := 0
	var stack []int // indices, heights increasing bottom to top

	for i := 0; i <= len(h); i++ {
		// A virtual bar of height 0 at the end flushes the stack, which removes
		// the "handle the leftovers" special case entirely.
		cur := 0
		if i < len(h) {
			cur = h[i]
		}

		for len(stack) > 0 && h[stack[len(stack)-1]] >= cur {
			top := stack[len(stack)-1]
			stack = stack[:len(stack)-1]

			height := h[top]
			// left edge: the element now on top is the nearest SHORTER bar to
			// the left. If the stack is empty, the bar extends to index 0.
			left := -1
			if len(stack) > 0 {
				left = stack[len(stack)-1]
			}
			width := i - left - 1

			if a := height * width; a > best {
				best = a
			}
		}
		stack = append(stack, i)
	}
	return best
}

// maximalRectangle: the 2-D version. Treat each row as the base of a histogram
// whose bar heights are the run of 1s above -- then it is the same function.
func maximalRectangle(grid [][]int) int {
	if len(grid) == 0 {
		return 0
	}
	heights := make([]int, len(grid[0]))
	best := 0
	for _, row := range grid {
		for c, v := range row {
			if v == 1 {
				heights[c]++
			} else {
				heights[c] = 0
			}
		}
		if a := largestMono(heights); a > best {
			best = a
		}
	}
	return best
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

func draw(h []int) {
	max := 0
	for _, v := range h {
		if v > max {
			max = v
		}
	}
	for level := max; level > 0; level-- {
		fmt.Print("  ")
		for _, v := range h {
			if v >= level {
				fmt.Print("##")
			} else {
				fmt.Print("  ")
			}
		}
		fmt.Println()
	}
	fmt.Print("  ")
	for range h {
		fmt.Print("--")
	}
	fmt.Println()
	fmt.Print("  ")
	for _, v := range h {
		fmt.Printf("%-2d", v)
	}
	fmt.Println()
}

func main() {
	h := []int{2, 1, 5, 6, 2, 3}
	fmt.Println("largest rectangle under this histogram:")
	fmt.Println()
	draw(h)
	fmt.Printf("\n  answer: %d  (the 5 and 6 together: height 5, width 2)\n", largestMono(h))

	fmt.Println()
	fmt.Println("the reframing that makes it Theta(n):")
	fmt.Println()
	fmt.Println("  every maximal rectangle is capped in height by ONE bar. So for")
	fmt.Println("  each bar, the question is how far it can extend before hitting")
	fmt.Println("  something shorter -- on both sides.")
	fmt.Println()
	fmt.Println("  that is 'nearest smaller to the left' and 'nearest smaller to the")
	fmt.Println("  right', which example 12 already solved in one pass. Keeping the")
	fmt.Println("  stack INCREASING means the element below any bar is precisely its")
	fmt.Println("  left boundary, and the bar that evicts it is its right one.")

	fmt.Println()
	fmt.Println("verified against brute force on 5000 random histograms:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	bad := 0
	for i := 0; i < 5000; i++ {
		in := make([]int, rng.IntN(10))
		for j := range in {
			in[j] = rng.IntN(6) // small range: plateaus and ties everywhere
		}
		if largestBrute(in) != largestMono(in) {
			fmt.Printf("  MISMATCH on %v: brute %d, mono %d\n", in, largestBrute(in), largestMono(in))
			bad++
			break
		}
	}
	if bad == 0 {
		fmt.Println("  5000 random histograms, including empty, flat and all-equal:")
		fmt.Println("  every one agrees.")
	}
	for _, edge := range [][]int{{}, {0}, {5}, {3, 3, 3}, {1, 2, 3, 4}, {4, 3, 2, 1}} {
		fmt.Printf("  %-16s -> brute %-6d mono %-6d\n", fmt.Sprint(edge), largestBrute(edge), largestMono(edge))
	}

	fmt.Println()
	fmt.Println("the cost, on the input that actually hurts brute force:")
	fmt.Println()
	fmt.Printf("  %10s %18s %14s %12s\n", "n", "brute ns", "mono ns", "speedup")
	for _, n := range []int{200, 800, 3200} {
		in := make([]int, n)
		for i := range in {
			in[i] = 1 + i%3 // lots of plateaus, so brute never breaks early
		}
		tb := nsPerOp(func() { sinkI = largestBrute(in) })
		tm := nsPerOp(func() { sinkI = largestMono(in) })
		fmt.Printf("  %10d %18.0f %14.0f %11.0fx\n", n, tb, tm, tb/tm)
	}

	fmt.Println()
	fmt.Println("the 2-D version is the same function, called once per row:")
	fmt.Println()
	grid := [][]int{
		{1, 0, 1, 0, 0},
		{1, 0, 1, 1, 1},
		{1, 1, 1, 1, 1},
		{1, 0, 0, 1, 0},
	}
	for _, row := range grid {
		fmt.Print("  ")
		for _, v := range row {
			if v == 1 {
				fmt.Print("# ")
			} else {
				fmt.Print(". ")
			}
		}
		fmt.Println()
	}
	fmt.Printf("\n  largest all-1s rectangle: %d\n", maximalRectangle(grid))
	fmt.Println()
	fmt.Println("  each row becomes a histogram whose bars are the run of 1s above")
	fmt.Println("  it. Then it is Theta(rows x cols) total, because each row is one")
	fmt.Println("  linear pass. Reducing a hard 2-D problem to a solved 1-D one is")
	fmt.Println("  the move worth remembering -- it recurs throughout lesson 31.")

	fmt.Println()
	fmt.Println("the two details that make the code short:")
	fmt.Println()
	fmt.Println("  1. the VIRTUAL zero bar at i == len(h). Without it you need a")
	fmt.Println("     second loop to drain the stack at the end; with it there is")
	fmt.Println("     one loop and no special case -- the same trick as lesson 07's")
	fmt.Println("     sentinel.")
	fmt.Println()
	fmt.Println("  2. width = i - left - 1, where `left` is the element BELOW the")
	fmt.Println("     popped one. That is correct even when equal heights are")
	fmt.Println("     involved: an earlier equal bar computes a too-small rectangle,")
	fmt.Println("     but the LAST one of the run computes the full width.")
}
```

**Sample output:**

```
largest rectangle under this histogram:

        ##    
      ####    
      ####    
      ####  ##
  ##  ########
  ############
  ------------
  2 1 5 6 2 3 

  answer: 10  (the 5 and 6 together: height 5, width 2)

the reframing that makes it Theta(n):

  every maximal rectangle is capped in height by ONE bar. So for
  each bar, the question is how far it can extend before hitting
  something shorter -- on both sides.

  that is 'nearest smaller to the left' and 'nearest smaller to the
  right', which example 12 already solved in one pass. Keeping the
  stack INCREASING means the element below any bar is precisely its
  left boundary, and the bar that evicts it is its right one.

verified against brute force on 5000 random histograms:

  5000 random histograms, including empty, flat and all-equal:
  every one agrees.
  []               -> brute 0      mono 0     
  [0]              -> brute 0      mono 0     
  [5]              -> brute 5      mono 5     
  [3 3 3]          -> brute 9      mono 9     
  [1 2 3 4]        -> brute 6      mono 6     
  [4 3 2 1]        -> brute 6      mono 6     

the cost, on the input that actually hurts brute force:

           n           brute ns        mono ns      speedup
         200              12244            282          43x
         800             172656           1054         164x
        3200            2696118           4209         641x

the 2-D version is the same function, called once per row:

  # . # . . 
  # . # # # 
  # # # # # 
  # . . # . 

  largest all-1s rectangle: 6

  each row becomes a histogram whose bars are the run of 1s above
  it. Then it is Theta(rows x cols) total, because each row is one
  linear pass. Reducing a hard 2-D problem to a solved 1-D one is
  the move worth remembering -- it recurs throughout lesson 31.

the two details that make the code short:

  1. the VIRTUAL zero bar at i == len(h). Without it you need a
     second loop to drain the stack at the end; with it there is
     one loop and no special case -- the same trick as lesson 07's
     sentinel.

  2. width = i - left - 1, where `left` is the element BELOW the
     popped one. That is correct even when equal heights are
     involved: an earlier equal bar computes a too-small rectangle,
     but the LAST one of the run computes the full width.
```

**Rust:**

```rust
use std::fmt::Display;
use std::hint::black_box;
use std::time::{Duration, Instant};

// The hardest of the classic monotonic-stack problems, and the one that pays
// off the technique: the largest rectangle that fits under a histogram.
//
// The reframing that solves it: every maximal rectangle is limited in HEIGHT by
// exactly one bar. So for each bar, ask "how far left and right can I extend
// while staying at least this tall?" -- and that is next-smaller-to-the-left
// and next-smaller-to-the-right, which is example 12.
//
// A monotonic INCREASING stack computes both in one pass.

// largest_brute: try every bar as the limiting height. Theta(n^2).
// (The Go version also bumps a package-level `brutePairs` counter that the
// program never prints; Rust has no mutable globals, and an unread counter
// would only be dead code, so it is dropped.)
fn largest_brute(h: &[i64]) -> i64 {
    let mut best = 0;
    for (i, &start) in h.iter().enumerate() {
        let mut min_h = start;
        for (w, &hj) in h[i..].iter().enumerate() {
            min_h = min_h.min(hj);
            best = best.max(min_h * (w as i64 + 1)); // width = j - i + 1
        }
    }
    best
}

// largest_mono: one pass, a stack of indices with INCREASING heights.
//
// When a bar shorter than the top arrives, the top can extend no further right
// -- so its rectangle is finished, and we can compute it. The left edge is
// whatever is below it on the stack, because everything between them was
// taller (that is the invariant).
fn largest_mono(h: &[i64]) -> i64 {
    let mut best = 0;
    let mut stack: Vec<usize> = Vec::new(); // indices, heights increasing bottom to top

    for i in 0..=h.len() {
        // A virtual bar of height 0 at the end flushes the stack, which removes
        // the "handle the leftovers" special case entirely. `get(i)` is None
        // exactly at i == h.len(), so Go's `if i < len(h)` is `unwrap_or(0)`.
        let cur = h.get(i).copied().unwrap_or(0);

        // pop_if: pop the top only if it is at least as tall as the new bar.
        while let Some(top) = stack.pop_if(|t| h[*t] >= cur) {
            let height = h[top];
            // left edge: the element now on top is the nearest SHORTER bar to
            // the left. If the stack is empty, the bar extends to index 0.
            // (Go uses `left := -1` as the "none" marker; Rust matches on the
            // Option, so `i - left - 1` needs no signed arithmetic.)
            let width = match stack.last() {
                Some(&left) => i - left - 1,
                None => i,
            };
            best = best.max(height * width as i64);
        }
        // The virtual bar is pushed too, exactly as in Go; nothing reads it.
        stack.push(i);
    }
    best
}

// maximal_rectangle: the 2-D version. Treat each row as the base of a histogram
// whose bar heights are the run of 1s above -- then it is the same function.
fn maximal_rectangle(grid: &[Vec<u8>]) -> i64 {
    let Some(first) = grid.first() else {
        return 0;
    };
    let mut heights = vec![0i64; first.len()];
    let mut best = 0;
    for row in grid {
        for (height, &cell) in heights.iter_mut().zip(row) {
            *height = if cell == 1 { *height + 1 } else { 0 };
        }
        best = best.max(largest_mono(&heights));
    }
    best
}

// Go prints a slice as [3 3 3]; Rust's {:?} prints [3, 3, 3]. Format by hand
// so the output reads the same in both languages.
fn go_slice<T: Display>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn draw(h: &[i64]) {
    let max = h.iter().copied().max().unwrap_or(0);
    for level in (1..=max).rev() {
        let row: String = h
            .iter()
            .map(|&v| if v >= level { "##" } else { "  " })
            .collect();
        println!("  {row}");
    }
    println!("  {}", "--".repeat(h.len()));
    let labels: String = h.iter().map(|v| format!("{v:<2}")).collect();
    println!("  {labels}");
}

// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN, so
// that the same seed (1, 2) gives the same 5000 histograms as the Go program.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
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

// ns/op: stand-in for testing.Benchmark / b.Loop.
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
    let h = [2, 1, 5, 6, 2, 3];
    println!("largest rectangle under this histogram:");
    println!();
    draw(&h);
    println!(
        "\n  answer: {}  (the 5 and 6 together: height 5, width 2)",
        largest_mono(&h)
    );

    println!();
    println!("the reframing that makes it Theta(n):");
    println!();
    println!("  every maximal rectangle is capped in height by ONE bar. So for");
    println!("  each bar, the question is how far it can extend before hitting");
    println!("  something shorter -- on both sides.");
    println!();
    println!("  that is 'nearest smaller to the left' and 'nearest smaller to the");
    println!("  right', which example 12 already solved in one pass. Keeping the");
    println!("  stack INCREASING means the element below any bar is precisely its");
    println!("  left boundary, and the bar that evicts it is its right one.");

    println!();
    println!("verified against brute force on 5000 random histograms:");
    println!();
    let mut rng = Pcg::new(1, 2);
    let mut bad = 0;
    for _ in 0..5000 {
        let len = rng.int_n(10) as usize;
        // small range: plateaus and ties everywhere
        let input: Vec<i64> = (0..len).map(|_| rng.int_n(6) as i64).collect();
        if largest_brute(&input) != largest_mono(&input) {
            println!(
                "  MISMATCH on {}: brute {}, mono {}",
                go_slice(&input),
                largest_brute(&input),
                largest_mono(&input)
            );
            bad += 1;
            break;
        }
    }
    if bad == 0 {
        println!("  5000 random histograms, including empty, flat and all-equal:");
        println!("  every one agrees.");
    }
    let edges: [&[i64]; 6] = [&[], &[0], &[5], &[3, 3, 3], &[1, 2, 3, 4], &[4, 3, 2, 1]];
    for edge in edges {
        println!(
            "  {:<16} -> brute {:<6} mono {:<6}",
            go_slice(edge),
            largest_brute(edge),
            largest_mono(edge)
        );
    }

    println!();
    println!("the cost, on the input that actually hurts brute force:");
    println!();
    println!(
        "  {:>10} {:>18} {:>14} {:>12}",
        "n", "brute ns", "mono ns", "speedup"
    );
    for n in [200i64, 800, 3200] {
        // lots of plateaus, so brute never breaks early
        let input: Vec<i64> = (0..n).map(|i| 1 + i % 3).collect();
        let tb = ns_per_op(|| {
            black_box(largest_brute(black_box(&input)));
        });
        let tm = ns_per_op(|| {
            black_box(largest_mono(black_box(&input)));
        });
        println!("  {:>10} {:>18.0} {:>14.0} {:>11.0}x", n, tb, tm, tb / tm);
    }

    println!();
    println!("the 2-D version is the same function, called once per row:");
    println!();
    let grid: Vec<Vec<u8>> = vec![
        vec![1, 0, 1, 0, 0],
        vec![1, 0, 1, 1, 1],
        vec![1, 1, 1, 1, 1],
        vec![1, 0, 0, 1, 0],
    ];
    for row in &grid {
        let cells: String = row
            .iter()
            .map(|&v| if v == 1 { "# " } else { ". " })
            .collect();
        println!("  {cells}");
    }
    println!("\n  largest all-1s rectangle: {}", maximal_rectangle(&grid));
    println!();
    println!("  each row becomes a histogram whose bars are the run of 1s above");
    println!("  it. Then it is Theta(rows x cols) total, because each row is one");
    println!("  linear pass. Reducing a hard 2-D problem to a solved 1-D one is");
    println!("  the move worth remembering -- it recurs throughout lesson 31.");

    println!();
    println!("the two details that make the code short:");
    println!();
    println!("  1. the VIRTUAL zero bar at i == len(h). Without it you need a");
    println!("     second loop to drain the stack at the end; with it there is");
    println!("     one loop and no special case -- the same trick as lesson 07's");
    println!("     sentinel.");
    println!();
    println!("  2. width = i - left - 1, where `left` is the element BELOW the");
    println!("     popped one. That is correct even when equal heights are");
    println!("     involved: an earlier equal bar computes a too-small rectangle,");
    println!("     but the LAST one of the run computes the full width.");
}
```

**Rust sample output:**

```
largest rectangle under this histogram:

        ##    
      ####    
      ####    
      ####  ##
  ##  ########
  ############
  ------------
  2 1 5 6 2 3 

  answer: 10  (the 5 and 6 together: height 5, width 2)

the reframing that makes it Theta(n):

  every maximal rectangle is capped in height by ONE bar. So for
  each bar, the question is how far it can extend before hitting
  something shorter -- on both sides.

  that is 'nearest smaller to the left' and 'nearest smaller to the
  right', which example 12 already solved in one pass. Keeping the
  stack INCREASING means the element below any bar is precisely its
  left boundary, and the bar that evicts it is its right one.

verified against brute force on 5000 random histograms:

  5000 random histograms, including empty, flat and all-equal:
  every one agrees.
  []               -> brute 0      mono 0     
  [0]              -> brute 0      mono 0     
  [5]              -> brute 5      mono 5     
  [3 3 3]          -> brute 9      mono 9     
  [1 2 3 4]        -> brute 6      mono 6     
  [4 3 2 1]        -> brute 6      mono 6     

the cost, on the input that actually hurts brute force:

           n           brute ns        mono ns      speedup
         200              11341            337          34x
         800             181324           1200         151x
        3200            2774081           4713         589x

the 2-D version is the same function, called once per row:

  # . # . . 
  # . # # # 
  # # # # # 
  # . . # . 

  largest all-1s rectangle: 6

  each row becomes a histogram whose bars are the run of 1s above
  it. Then it is Theta(rows x cols) total, because each row is one
  linear pass. Reducing a hard 2-D problem to a solved 1-D one is
  the move worth remembering -- it recurs throughout lesson 31.

the two details that make the code short:

  1. the VIRTUAL zero bar at i == len(h). Without it you need a
     second loop to drain the stack at the end; with it there is
     one loop and no special case -- the same trick as lesson 07's
     sentinel.

  2. width = i - left - 1, where `left` is the element BELOW the
     popped one. That is correct even when equal heights are
     involved: an earlier equal bar computes a too-small rectangle,
     but the LAST one of the run computes the full width.
```

> **Go vs Rust:** Everything except the timing table is identical to Go's (diffed against a live run of the Go program). Unlike example 12, Rust has no speed advantage here: the brute force took 11.3 to 11.5 µs, 175 to 177 µs and 2.77 to 3.17 ms over 6 runs against Go's 12.0 to 12.7 µs, 173 to 249 µs and 2.85 to 3.13 ms over 3 live runs, and the monotonic version was slower at n=200 (315 to 358 ns against 274 to 290), so the speedups (33-36x, 134-148x, 589-677x) sit close to Go's (43-44x, 135-225x, 627-687x). This loop has no per-iteration counter, which fits the counter explanation given for example 12 but does not prove it. The virtual zero bar is `h.get(i).copied().unwrap_or(0)`, Go's `left := -1` is a `match` on `stack.last()`, and Go's never-printed `brutePairs` counter is dropped.

**Complexity:** Θ(n) for the histogram, Θ(rows × cols) for the grid · reducing a hard 2-D problem to a solved 1-D one is the move worth remembering

---

## 15. Trapping rain water, four ways

`🔴 hard` · *When *not* to use the stack*

Three correct Θ(n) solutions with different space, and the honest conclusion is not "use the
clever one".

**Steps:**

1. Precompute both running maxima into arrays.
2. Then fill horizontally with a monotonic stack.
3. Then do it with two pointers in Θ(1) space, and compare all four.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// Trapping rain water: given a histogram, how much water sits in the dips?
//
// Worth doing last because it has THREE correct solutions with the same
// Theta(n) time and different space, and comparing them shows what the
// monotonic stack is really for -- and where a simpler idea beats it.
//
// The insight all three share: the water above column i is
//
//	min(tallest to its left, tallest to its right) - height[i]
//
// The three approaches differ only in how they get those two maxima.

// --- 1. precompute both maxima into arrays. Theta(n) time, Theta(n) space. ---

func trapArrays(h []int) int {
	if len(h) == 0 {
		return 0
	}
	n := len(h)
	leftMax := make([]int, n)
	rightMax := make([]int, n)

	leftMax[0] = h[0]
	for i := 1; i < n; i++ {
		leftMax[i] = max(leftMax[i-1], h[i])
	}
	rightMax[n-1] = h[n-1]
	for i := n - 2; i >= 0; i-- {
		rightMax[i] = max(rightMax[i+1], h[i])
	}

	total := 0
	for i := 0; i < n; i++ {
		total += min(leftMax[i], rightMax[i]) - h[i]
	}
	return total
}

// --- 2. the monotonic stack. Fills water HORIZONTALLY, layer by layer. ---
//
// The stack holds indices of decreasing heights. When a taller bar arrives, the
// bar it evicts is the FLOOR of a puddle: the new bar is the right wall, and
// whatever is below on the stack is the left wall.
func trapStack(h []int) int {
	total := 0
	var stack []int // indices, heights decreasing

	for i, cur := range h {
		for len(stack) > 0 && h[stack[len(stack)-1]] < cur {
			floor := stack[len(stack)-1]
			stack = stack[:len(stack)-1]

			if len(stack) == 0 {
				break // no left wall: the water runs off
			}
			left := stack[len(stack)-1]
			width := i - left - 1
			depth := min(h[left], cur) - h[floor]
			total += width * depth
		}
		stack = append(stack, i)
	}
	return total
}

// --- 3. two pointers. Theta(n) time, Theta(1) SPACE. ---
//
// Walk in from both ends. Whichever side is SHORTER is the side whose answer is
// already determined -- because the taller side guarantees a wall at least that
// high exists beyond it. So that side can be resolved and advanced.
func trapTwoPointer(h []int) int {
	if len(h) == 0 {
		return 0
	}
	l, r := 0, len(h)-1
	leftMax, rightMax := h[l], h[r]
	total := 0

	for l < r {
		if leftMax <= rightMax {
			l++
			leftMax = max(leftMax, h[l])
			total += leftMax - h[l]
		} else {
			r--
			rightMax = max(rightMax, h[r])
			total += rightMax - h[r]
		}
	}
	return total
}

// --- brute force, to verify all three ---

func trapBrute(h []int) int {
	total := 0
	for i := range h {
		l, r := 0, 0
		for j := 0; j <= i; j++ {
			l = max(l, h[j])
		}
		for j := i; j < len(h); j++ {
			r = max(r, h[j])
		}
		total += min(l, r) - h[i]
	}
	return total
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

func draw(h []int) {
	maxH := 0
	for _, v := range h {
		maxH = max(maxH, v)
	}
	n := len(h)
	leftMax := make([]int, n)
	rightMax := make([]int, n)
	leftMax[0] = h[0]
	for i := 1; i < n; i++ {
		leftMax[i] = max(leftMax[i-1], h[i])
	}
	rightMax[n-1] = h[n-1]
	for i := n - 2; i >= 0; i-- {
		rightMax[i] = max(rightMax[i+1], h[i])
	}

	for level := maxH; level > 0; level-- {
		fmt.Print("  ")
		for i, v := range h {
			switch {
			case v >= level:
				fmt.Print("#")
			case min(leftMax[i], rightMax[i]) >= level:
				fmt.Print("~")
			default:
				fmt.Print(" ")
			}
		}
		fmt.Println()
	}
}

func main() {
	h := []int{0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1}
	fmt.Println("# is wall, ~ is trapped water:")
	fmt.Println()
	draw(h)
	fmt.Printf("\n  heights %v\n", h)
	fmt.Printf("  trapped %d units\n", trapStack(h))

	fmt.Println()
	fmt.Println("all four approaches agree, on 5000 random inputs:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	bad := 0
	for i := 0; i < 5000; i++ {
		in := make([]int, rng.IntN(12))
		for j := range in {
			in[j] = rng.IntN(5)
		}
		a, b, c, d := trapBrute(in), trapArrays(in), trapStack(in), trapTwoPointer(in)
		if a != b || b != c || c != d {
			fmt.Printf("  MISMATCH on %v: brute=%d arrays=%d stack=%d two-ptr=%d\n", in, a, b, c, d)
			bad++
			break
		}
	}
	if bad == 0 {
		fmt.Println("  brute, arrays, stack and two-pointer all agree everywhere.")
	}

	fmt.Println()
	fmt.Println("so which one? They are all Theta(n) time:")
	fmt.Println()
	fmt.Printf("  %-24s %-14s %-14s %s\n", "approach", "time", "space", "note")
	for _, r := range [][4]string{
		{"brute force", "Theta(n^2)", "Theta(1)", "recomputes both maxima per column"},
		{"precomputed arrays", "Theta(n)", "Theta(n)", "clearest to read and to prove"},
		{"monotonic stack", "Theta(n)", "Theta(n)", "fills horizontally, layer by layer"},
		{"two pointers", "Theta(n)", "Theta(1)", "fewest moving parts, hardest to see"},
	} {
		fmt.Printf("  %-24s %-14s %-14s %s\n", r[0], r[1], r[2], r[3])
	}

	fmt.Println()
	const n = 20000
	in := make([]int, n)
	for i := range in {
		in[i] = rng.IntN(1000)
	}
	tArr := nsPerOp(func() { sinkI = trapArrays(in) })
	tStk := nsPerOp(func() { sinkI = trapStack(in) })
	tTwo := nsPerOp(func() { sinkI = trapTwoPointer(in) })
	fmt.Printf("  at n=%d:  arrays %.0f ns   stack %.0f ns   two-pointer %.0f ns\n",
		n, tArr, tStk, tTwo)
	fmt.Printf("  two-pointer is %.1fx FASTER than the arrays version, and allocates nothing.\n", tArr/tTwo)

	fmt.Println()
	fmt.Println("the honest conclusion, which is not 'use the clever one':")
	fmt.Println()
	fmt.Println("  the TWO-POINTER version wins -- Theta(1) space, no allocation,")
	fmt.Println("  fewest branches. The monotonic stack is not the best tool here.")
	fmt.Println()
	fmt.Println("  that is worth sitting with. Examples 12-14 were problems where the")
	fmt.Println("  stack is genuinely the answer, because each element needs the")
	fmt.Println("  nearest larger/smaller NEIGHBOUR and nothing simpler will produce")
	fmt.Println("  it. Here every column only needs two RUNNING MAXIMA, which a pair")
	fmt.Println("  of pointers tracks for free.")
	fmt.Println()
	fmt.Println("  the giveaway: if the answer depends on a specific nearby ELEMENT,")
	fmt.Println("  reach for the stack. If it depends only on a running AGGREGATE,")
	fmt.Println("  two pointers or a prefix scan will be simpler (lesson 16).")
}
```

**Sample output:**

```
# is wall, ~ is trapped water:

         #    
     #~~~##~# 
   #~##~######

  heights [0 1 0 2 1 0 1 3 2 1 2 1]
  trapped 6 units

all four approaches agree, on 5000 random inputs:

  brute, arrays, stack and two-pointer all agree everywhere.

so which one? They are all Theta(n) time:

  approach                 time           space          note
  brute force              Theta(n^2)     Theta(1)       recomputes both maxima per column
  precomputed arrays       Theta(n)       Theta(n)       clearest to read and to prove
  monotonic stack          Theta(n)       Theta(n)       fills horizontally, layer by layer
  two pointers             Theta(n)       Theta(1)       fewest moving parts, hardest to see

  at n=20000:  arrays 68514 ns   stack 39551 ns   two-pointer 10727 ns
  two-pointer is 6.4x FASTER than the arrays version, and allocates nothing.

the honest conclusion, which is not 'use the clever one':

  the TWO-POINTER version wins -- Theta(1) space, no allocation,
  fewest branches. The monotonic stack is not the best tool here.

  that is worth sitting with. Examples 12-14 were problems where the
  stack is genuinely the answer, because each element needs the
  nearest larger/smaller NEIGHBOUR and nothing simpler will produce
  it. Here every column only needs two RUNNING MAXIMA, which a pair
  of pointers tracks for free.

  the giveaway: if the answer depends on a specific nearby ELEMENT,
  reach for the stack. If it depends only on a running AGGREGATE,
  two pointers or a prefix scan will be simpler (lesson 16).
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Trapping rain water: given a histogram, how much water sits in the dips?
//
// Worth doing last because it has THREE correct solutions with the same
// Theta(n) time and different space, and comparing them shows what the
// monotonic stack is really for -- and where a simpler idea beats it.
//
// The insight all three share: the water above column i is
//
//     min(tallest to its left, tallest to its right) - height[i]
//
// The three approaches differ only in how they get those two maxima.
//
// Heights are `usize`: a height is never negative, and the water above a column
// (a `min` of two maxima minus the column) never is either, so plain unsigned
// subtraction is safe wherever it appears below.

// --- 1. precompute both maxima into arrays. Theta(n) time, Theta(n) space. ---

/// Fills the two running maxima in place: `left_max[i]` is the tallest bar in
/// h[..=i], `right_max[i]` the tallest in h[i..]. (Go writes this loop out twice,
/// in trapArrays and in draw; Rust shares it.)
fn fill_running_maxima(h: &[usize], left_max: &mut [usize], right_max: &mut [usize]) {
    let mut m = 0;
    for (slot, &x) in left_max.iter_mut().zip(h) {
        m = m.max(x);
        *slot = m;
    }
    let mut m = 0;
    for (slot, &x) in right_max.iter_mut().zip(h).rev() {
        m = m.max(x);
        *slot = m;
    }
}

/// The two maxima as fresh vectors: two allocations of n elements each,
/// like Go's two `make([]int, n)`.
fn running_maxima(h: &[usize]) -> (Vec<usize>, Vec<usize>) {
    let (mut left_max, mut right_max) = (vec![0; h.len()], vec![0; h.len()]);
    fill_running_maxima(h, &mut left_max, &mut right_max);
    (left_max, right_max)
}

/// min(leftMax, rightMax) - height, summed over every column.
fn water_between(h: &[usize], left_max: &[usize], right_max: &[usize]) -> usize {
    h.iter()
        .zip(left_max)
        .zip(right_max)
        .map(|((&x, &l), &r)| l.min(r) - x)
        .sum()
}

fn trap_arrays(h: &[usize]) -> usize {
    // No `if h.is_empty() { return 0 }`: zipping empty slices sums to 0.
    // (Go needs the guard because it reads h[0] before the loop.)
    let (left_max, right_max) = running_maxima(h);
    water_between(h, &left_max, &right_max)
}

/// Rust addition: the same algorithm with the caller's two buffers instead of
/// two fresh allocations. It exists to ask how much of the arrays version's cost
/// is the allocation, and how much is the passes over the data.
fn trap_arrays_reusing(h: &[usize], left_max: &mut [usize], right_max: &mut [usize]) -> usize {
    fill_running_maxima(h, left_max, right_max);
    water_between(h, left_max, right_max)
}

// --- 2. the monotonic stack. Fills water HORIZONTALLY, layer by layer. ---
//
// The stack holds indices of decreasing heights. When a taller bar arrives, the
// bar it evicts is the FLOOR of a puddle: the new bar is the right wall, and
// whatever is below on the stack is the left wall.
fn trap_stack(h: &[usize]) -> usize {
    let mut total = 0;
    let mut stack: Vec<usize> = Vec::new(); // indices, heights decreasing

    for (i, &cur) in h.iter().enumerate() {
        while let Some(&floor) = stack.last() {
            if h[floor] >= cur {
                break;
            }
            stack.pop();

            // `let ... else` is Go's `if len(stack) == 0 { break }` fused with
            // the read of the left wall: no left wall, the water runs off.
            let Some(&left) = stack.last() else { break };
            let width = i - left - 1;
            let depth = h[left].min(cur) - h[floor];
            total += width * depth;
        }
        stack.push(i);
    }
    total
}

// --- 3. two pointers. Theta(n) time, Theta(1) SPACE. ---
//
// Walk in from both ends. Whichever side is SHORTER is the side whose answer is
// already determined -- because the taller side guarantees a wall at least that
// high exists beyond it. So that side can be resolved and advanced.
fn trap_two_pointer(h: &[usize]) -> usize {
    if h.is_empty() {
        return 0;
    }
    let (mut l, mut r) = (0, h.len() - 1);
    let (mut left_max, mut right_max) = (h[l], h[r]);
    let mut total = 0;

    while l < r {
        if left_max <= right_max {
            l += 1;
            left_max = left_max.max(h[l]);
            total += left_max - h[l];
        } else {
            r -= 1; // cannot underflow: l < r means r >= 1
            right_max = right_max.max(h[r]);
            total += right_max - h[r];
        }
    }
    total
}

// --- brute force, to verify all three ---

fn trap_brute(h: &[usize]) -> usize {
    (0..h.len())
        .map(|i| {
            let l = h[..=i].iter().copied().max().unwrap_or(0);
            let r = h[i..].iter().copied().max().unwrap_or(0);
            l.min(r) - h[i]
        })
        .sum()
}

// ---------------------------------------------------------------------------
// Measuring: stand-ins for what Go's testing package gives for free.
// ---------------------------------------------------------------------------

// Counts allocation events and bytes requested: a deterministic stand-in for
// `-benchmem`. (`unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every
// method counts and then forwards to the system allocator unchanged.)
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
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested) during one call of `f`.
fn allocs_of(f: impl FnOnce()) -> (usize, usize) {
    let (a0, b0) = (ALLOCS.load(Relaxed), BYTES.load(Relaxed));
    f();
    (ALLOCS.load(Relaxed) - a0, BYTES.load(Relaxed) - b0)
}

/// ns/op the way testing.Benchmark does it: run `f` n times, and if that took
/// less than the target, scale n up and go again.
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

/// Noise only ever ADDS time, so the fastest of three measurements is the one
/// least disturbed by whatever else the machine was doing.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

/// A port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN, so
/// the 5000 random inputs below are the very ones the Go program checks.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> usize {
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

/// `[0 1 0 2]`: how Go's %v prints a slice (Rust's {:?} would add commas).
fn go_list(xs: &[usize]) -> String {
    let items: Vec<String> = xs.iter().map(usize::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn draw(h: &[usize]) {
    let max_h = h.iter().copied().max().unwrap_or(0);
    let (left_max, right_max) = running_maxima(h);

    for level in (1..=max_h).rev() {
        let row: String = h
            .iter()
            .enumerate()
            .map(|(i, &v)| {
                if v >= level {
                    '#'
                } else if left_max[i].min(right_max[i]) >= level {
                    '~'
                } else {
                    ' '
                }
            })
            .collect();
        println!("  {row}");
    }
}

fn main() {
    let h = [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1];
    println!("# is wall, ~ is trapped water:");
    println!();
    draw(&h);
    println!("\n  heights {}", go_list(&h));
    println!("  trapped {} units", trap_stack(&h));

    println!();
    println!("all four approaches agree, on 5000 random inputs:");
    println!();
    let mut rng = Pcg::new(1, 2);
    let mut agree = true;
    for _ in 0..5000 {
        let len = rng.int_n(12);
        let input: Vec<usize> = (0..len).map(|_| rng.int_n(5)).collect();
        let (a, b, c, d) = (
            trap_brute(&input),
            trap_arrays(&input),
            trap_stack(&input),
            trap_two_pointer(&input),
        );
        if !(a == b && b == c && c == d) {
            println!(
                "  MISMATCH on {}: brute={a} arrays={b} stack={c} two-ptr={d}",
                go_list(&input)
            );
            agree = false;
            break;
        }
    }
    if agree {
        println!("  brute, arrays, stack and two-pointer all agree everywhere.");
    }

    println!();
    println!("so which one? They are all Theta(n) time:");
    println!();
    println!("  {:<24} {:<14} {:<14} note", "approach", "time", "space");
    for (approach, time, space, note) in [
        (
            "brute force",
            "Theta(n^2)",
            "Theta(1)",
            "recomputes both maxima per column",
        ),
        (
            "precomputed arrays",
            "Theta(n)",
            "Theta(n)",
            "clearest to read and to prove",
        ),
        (
            "monotonic stack",
            "Theta(n)",
            "Theta(n)",
            "fills horizontally, layer by layer",
        ),
        (
            "two pointers",
            "Theta(n)",
            "Theta(1)",
            "fewest moving parts, hardest to see",
        ),
    ] {
        println!("  {approach:<24} {time:<14} {space:<14} {note}");
    }

    println!();
    const N: usize = 20000;
    let input: Vec<usize> = (0..N).map(|_| rng.int_n(1000)).collect();
    let t_arr = best_ns_per_op(|| _ = black_box(trap_arrays(black_box(&input))));
    let t_stk = best_ns_per_op(|| _ = black_box(trap_stack(black_box(&input))));
    let t_two = best_ns_per_op(|| _ = black_box(trap_two_pointer(black_box(&input))));
    println!("  at n={N}:  arrays {t_arr:.0} ns   stack {t_stk:.0} ns   two-pointer {t_two:.0} ns");
    println!(
        "  time relative to two-pointer:  arrays {:.1}x   stack {:.1}x",
        t_arr / t_two,
        t_stk / t_two
    );

    // Rust additions. The counting allocator turns Go's "allocates nothing" from
    // something to read off -benchmem into numbers this program can print, and
    // the reusing variant separates allocation from the passes over the data.
    let (a_arr, b_arr) = allocs_of(|| _ = black_box(trap_arrays(black_box(&input))));
    let (a_stk, b_stk) = allocs_of(|| _ = black_box(trap_stack(black_box(&input))));
    let (a_two, b_two) = allocs_of(|| _ = black_box(trap_two_pointer(black_box(&input))));
    let descending: Vec<usize> = (0..N).rev().collect();
    let (a_dsc, b_dsc) = allocs_of(|| _ = black_box(trap_stack(black_box(&descending))));
    println!();
    println!("  allocations per call (events / bytes requested):");
    println!(
        "    arrays {a_arr} / {b_arr}   stack {a_stk} / {b_stk}   two-pointer {a_two} / {b_two}"
    );
    println!("    stack on a strictly descending histogram of the same length: {a_dsc} / {b_dsc}");

    let (mut left_buf, mut right_buf) = (vec![0; N], vec![0; N]);
    let t_reuse = best_ns_per_op(|| {
        _ = black_box(trap_arrays_reusing(
            black_box(&input),
            &mut left_buf,
            &mut right_buf,
        ))
    });
    println!("  arrays with both buffers reused (no allocation): {t_reuse:.0} ns");

    println!();
    println!("the honest conclusion, which is not 'use the clever one':");
    println!();
    println!("  the TWO-POINTER version wins -- Theta(1) space, no allocation (0 events");
    println!("  counted above), one pass, and the lowest time of the three. The");
    println!("  monotonic stack is not the best tool here: it was the SLOWEST of the");
    println!("  three, behind even the arrays version that allocates two n-element");
    println!("  vectors on every call. (A plausible reason: its inner loop branches on");
    println!("  the data, and random heights make those branches unpredictable. Not");
    println!("  measured.) Its Theta(n) space is a worst case: this input needed {a_stk}");
    println!("  allocation events, while the descending one above needed {a_dsc}.");
    println!();
    println!("  that is worth sitting with. Examples 12-14 were problems where the");
    println!("  stack is genuinely the answer, because each element needs the");
    println!("  nearest larger/smaller NEIGHBOUR and nothing simpler will produce");
    println!("  it. Here every column only needs two RUNNING MAXIMA, which a pair");
    println!("  of pointers tracks for free.");
    println!();
    println!("  the giveaway: if the answer depends on a specific nearby ELEMENT,");
    println!("  reach for the stack. If it depends only on a running AGGREGATE,");
    println!("  two pointers or a prefix scan will be simpler (lesson 16).");
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O ex15.rs -o ex15 && ./ex15
```

**Rust sample output:**

```
# is wall, ~ is trapped water:

         #    
     #~~~##~# 
   #~##~######

  heights [0 1 0 2 1 0 1 3 2 1 2 1]
  trapped 6 units

all four approaches agree, on 5000 random inputs:

  brute, arrays, stack and two-pointer all agree everywhere.

so which one? They are all Theta(n) time:

  approach                 time           space          note
  brute force              Theta(n^2)     Theta(1)       recomputes both maxima per column
  precomputed arrays       Theta(n)       Theta(n)       clearest to read and to prove
  monotonic stack          Theta(n)       Theta(n)       fills horizontally, layer by layer
  two pointers             Theta(n)       Theta(1)       fewest moving parts, hardest to see

  at n=20000:  arrays 26879 ns   stack 41469 ns   two-pointer 10437 ns
  time relative to two-pointer:  arrays 2.6x   stack 4.0x

  allocations per call (events / bytes requested):
    arrays 2 / 320000   stack 5 / 992   two-pointer 0 / 0
    stack on a strictly descending histogram of the same length: 14 / 524256
  arrays with both buffers reused (no allocation): 24548 ns

the honest conclusion, which is not 'use the clever one':

  the TWO-POINTER version wins -- Theta(1) space, no allocation (0 events
  counted above), one pass, and the lowest time of the three. The
  monotonic stack is not the best tool here: it was the SLOWEST of the
  three, behind even the arrays version that allocates two n-element
  vectors on every call. (A plausible reason: its inner loop branches on
  the data, and random heights make those branches unpredictable. Not
  measured.) Its Theta(n) space is a worst case: this input needed 5
  allocation events, while the descending one above needed 14.

  that is worth sitting with. Examples 12-14 were problems where the
  stack is genuinely the answer, because each element needs the
  nearest larger/smaller NEIGHBOUR and nothing simpler will produce
  it. Here every column only needs two RUNNING MAXIMA, which a pair
  of pointers tracks for free.

  the giveaway: if the answer depends on a specific nearby ELEMENT,
  reach for the stack. If it depends only on a running AGGREGATE,
  two pointers or a prefix scan will be simpler (lesson 16).
```

> **Go vs Rust:** The drawing, the 6 units, the agreement check (a port of Go's PCG feeds both programs the same 5000 inputs) and the table are identical to Go's; only the timing lines are a sample. Two-pointer costs about the same in both languages (10.3-11.8 µs in Rust), but the arrays version is 27-35 µs in Rust against about 81-83 µs for Go on this machine, so its gap to two-pointer shrinks from roughly 7x to 2.4-3.2x, and the stack, which beat the arrays version in Go's sample, was the slowest of the three in all six Rust runs (39-49 µs). Reusing the two buffers only moves the arrays version from about 27 to 24-26 µs, so allocation is not what it pays for; why Go's arrays version is so much slower was not measured.

**Complexity:** arrays and stack Θ(n) space · **two pointers Θ(1) space and fastest** · the giveaway: a specific nearby *element* wants a stack, a running *aggregate* wants two pointers

---

## 16. Capstone: Stack[T] and the algorithms

`🔴 hard` · *Build it · Prove it · Measure it` · `go test*

The structure is nearly trivial, so most of the risk lives in the **algorithms** built on it — and
a monotonic stack is exactly the kind of index arithmetic that looks right and is off by one.
So those get brute-force oracles of their own.

**Steps:**

1. Package `Stack[T]` with `Reserve` and `Clear`.
2. Prove the structure: table, slot-release invariant, a 50,000-operation model oracle.
3. Prove the algorithms: 20,000 random inputs each against brute force, plus edge cases.
4. Assert the amortized claim directly: total pops ≤ n.

**`stack.go`**

```go
package stack

// Stack[T] is example 2's type, finished: generic, zero-value-ready, with the
// (T, bool) contract and the pointer hygiene.
//
// INVARIANT, true before and after every exported method:
//
//	items[len(items):cap(items)] holds only zero values
//	  -- so a popped element is never kept alive by the stack
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Len() int      { return len(s.items) }
func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	i := len(s.items) - 1
	v := s.items[i]
	s.items[i] = zero
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

// Reserve makes room for n more pushes without reallocating.
func (s *Stack[T]) Reserve(n int) {
	if need := len(s.items) + n; need > cap(s.items) {
		bigger := make([]T, len(s.items), need)
		copy(bigger, s.items)
		s.items = bigger
	}
}

// Clear empties the stack, zeroing every slot so nothing stays reachable,
// and keeps the capacity for reuse (lesson 06, example 10).
func (s *Stack[T]) Clear() {
	clear(s.items)
	s.items = s.items[:0]
}

// ============================================================================
// The monotonic-stack algorithms from examples 12-14, as library functions.
// ============================================================================

// NextGreater returns, for each i, the value of the next element to the right
// that is strictly greater -- or -1 if there is none. Theta(n).
func NextGreater(xs []int) []int {
	out := make([]int, len(xs))
	for i := range out {
		out[i] = -1
	}
	var st Stack[int] // indices, values strictly decreasing
	for i, v := range xs {
		for {
			top, ok := st.Peek()
			if !ok || xs[top] >= v {
				break
			}
			st.Pop()
			out[top] = v
		}
		st.Push(i)
	}
	return out
}

// LargestRectangle returns the area of the largest rectangle under the
// histogram. Theta(n).
func LargestRectangle(h []int) int {
	best := 0
	var st Stack[int] // indices, heights increasing
	for i := 0; i <= len(h); i++ {
		cur := 0 // a virtual zero bar at the end flushes the stack
		if i < len(h) {
			cur = h[i]
		}
		for {
			top, ok := st.Peek()
			if !ok || h[top] < cur {
				break
			}
			st.Pop()
			left := -1
			if l, ok := st.Peek(); ok {
				left = l
			}
			if a := h[top] * (i - left - 1); a > best {
				best = a
			}
		}
		st.Push(i)
	}
	return best
}
```

**`stack_test.go`** — pass 2: prove it

```go
package stack

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// PASS 2: PROVE IT. Same four layers as lessons 06 and 07.
//
// The structure here is nearly trivial, so most of the risk is in the
// ALGORITHMS built on it -- and those get brute-force oracles, because a
// monotonic stack is exactly the kind of index arithmetic that looks right
// and is off by one.
// ============================================================================

// --- 1. the table -----------------------------------------------------------

func TestStackBasics(t *testing.T) {
	var s Stack[int]

	if _, ok := s.Pop(); ok {
		t.Error("Pop on an empty stack reported ok=true")
	}
	if _, ok := s.Peek(); ok {
		t.Error("Peek on an empty stack reported ok=true")
	}
	if !s.IsEmpty() || s.Len() != 0 {
		t.Error("the zero value is not empty")
	}

	for i := 1; i <= 3; i++ {
		s.Push(i)
	}
	if v, _ := s.Peek(); v != 3 {
		t.Errorf("Peek = %d, want 3", v)
	}
	if s.Len() != 3 {
		t.Errorf("Len = %d, want 3", s.Len())
	}
	// Peek must not remove.
	if v, _ := s.Peek(); v != 3 || s.Len() != 3 {
		t.Error("Peek changed the stack")
	}

	for i := 3; i >= 1; i-- {
		v, ok := s.Pop()
		if !ok || v != i {
			t.Fatalf("Pop = (%d, %v), want (%d, true)", v, ok, i)
		}
	}
	if !s.IsEmpty() {
		t.Error("stack not empty after popping everything")
	}
}

// --- 2. the invariant -------------------------------------------------------

// Popping must zero the vacated slot, or a removed element stays reachable
// through the backing array (lesson 06, example 2).
func TestPopReleasesTheSlot(t *testing.T) {
	type box struct{ p *int }
	var s Stack[box]

	n := 42
	s.Push(box{&n})
	s.Push(box{&n})
	s.Pop()
	s.Pop()

	full := s.items[:cap(s.items)]
	for i := 0; i < len(full); i++ {
		if full[i].p != nil {
			t.Errorf("slot %d past the length still holds %v -- Pop did not zero it", i, full[i].p)
		}
	}
}

func TestClearReleasesEverything(t *testing.T) {
	type box struct{ p *int }
	var s Stack[box]
	n := 7
	for i := 0; i < 5; i++ {
		s.Push(box{&n})
	}
	c := cap(s.items)
	s.Clear()

	if s.Len() != 0 {
		t.Errorf("Len = %d after Clear, want 0", s.Len())
	}
	if cap(s.items) != c {
		t.Errorf("Clear changed the capacity (%d -> %d); it should keep it for reuse", c, cap(s.items))
	}
	for i, b := range s.items[:cap(s.items)] {
		if b.p != nil {
			t.Errorf("slot %d still holds a pointer after Clear", i)
		}
	}
}

// --- 3. the model oracle ----------------------------------------------------

func TestStackAgainstModel(t *testing.T) {
	rng := rand.New(rand.NewPCG(42, 7))
	var s Stack[int]
	var model []int

	const trials = 50_000
	for i := 0; i < trials; i++ {
		switch rng.IntN(3) {
		case 0, 1:
			v := rng.IntN(1000)
			s.Push(v)
			model = append(model, v)
		case 2:
			got, gotOK := s.Pop()
			if len(model) == 0 {
				if gotOK {
					t.Fatalf("trial %d: Pop reported ok on an empty stack", i)
				}
				break
			}
			want := model[len(model)-1]
			model = model[:len(model)-1]
			if !gotOK || got != want {
				t.Fatalf("trial %d: Pop = (%d,%v), model says %d", i, got, gotOK, want)
			}
		}
		if s.Len() != len(model) {
			t.Fatalf("trial %d: Len = %d, model has %d", i, s.Len(), len(model))
		}
		if !slices.Equal(s.items, model) {
			t.Fatalf("trial %d: contents diverged", i)
		}
	}
	t.Logf("%d random operations, final depth %d", trials, s.Len())
}

// --- 4. brute-force oracles for the ALGORITHMS ------------------------------

func nextGreaterBrute(xs []int) []int {
	out := make([]int, len(xs))
	for i := range xs {
		out[i] = -1
		for j := i + 1; j < len(xs); j++ {
			if xs[j] > xs[i] {
				out[i] = xs[j]
				break
			}
		}
	}
	return out
}

func largestRectangleBrute(h []int) int {
	best := 0
	for i := range h {
		minH := h[i]
		for j := i; j < len(h); j++ {
			minH = min(minH, h[j])
			best = max(best, minH*(j-i+1))
		}
	}
	return best
}

func TestNextGreaterAgainstBrute(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))
	for i := 0; i < 20_000; i++ {
		// SMALL values: ties are where a strict/non-strict comparison goes
		// wrong, and a wide range would almost never produce one.
		xs := make([]int, rng.IntN(10))
		for j := range xs {
			xs[j] = rng.IntN(5)
		}
		want := nextGreaterBrute(xs)
		got := NextGreater(xs)
		if !slices.Equal(got, want) {
			t.Fatalf("NextGreater(%v)\n  got  %v\n  want %v", xs, got, want)
		}
	}
}

func TestLargestRectangleAgainstBrute(t *testing.T) {
	rng := rand.New(rand.NewPCG(3, 4))
	for i := 0; i < 20_000; i++ {
		h := make([]int, rng.IntN(10))
		for j := range h {
			h[j] = rng.IntN(6)
		}
		if got, want := LargestRectangle(h), largestRectangleBrute(h); got != want {
			t.Fatalf("LargestRectangle(%v) = %d, want %d", h, got, want)
		}
	}
}

func TestAlgorithmEdgeCases(t *testing.T) {
	for _, tt := range []struct {
		name string
		in   []int
	}{
		{"nil", nil},
		{"empty", []int{}},
		{"single", []int{5}},
		{"all equal", []int{3, 3, 3, 3}},
		{"ascending", []int{1, 2, 3, 4}},
		{"descending", []int{4, 3, 2, 1}},
		{"zeros", []int{0, 0, 0}},
		{"one zero", []int{2, 0, 2}},
	} {
		t.Run(tt.name, func(t *testing.T) {
			if got, want := NextGreater(tt.in), nextGreaterBrute(tt.in); !slices.Equal(got, want) {
				t.Errorf("NextGreater = %v, want %v", got, want)
			}
			if got, want := LargestRectangle(tt.in), largestRectangleBrute(tt.in); got != want {
				t.Errorf("LargestRectangle = %d, want %d", got, want)
			}
		})
	}
}

// --- the amortized claim, asserted -----------------------------------------

var sinkI int

func TestPushIsAmortizedConstant(t *testing.T) {
	var s Stack[int]
	s.Reserve(1000)
	allocs := testing.AllocsPerRun(50, func() {
		s.Clear()
		for i := 0; i < 1000; i++ {
			s.Push(i)
		}
		sinkI = s.Len()
	})
	if allocs != 0 {
		t.Errorf("1000 pushes into reserved capacity allocated %.0f times, want 0", allocs)
	}
	t.Logf("1000 pushes after Reserve: %.0f allocations", allocs)
}

// The monotonic-stack claim: total inner-loop work over a whole run is at most
// n, because each index is pushed once and popped at most once. Assert it.
func TestMonotonicIsLinear(t *testing.T) {
	for _, n := range []int{100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = n - i // descending: nothing is ever popped
		}
		pops := countPops(xs)
		if pops > n {
			t.Errorf("n=%d: %d pops, which exceeds n -- the amortized bound is broken", n, pops)
		}

		asc := make([]int, n)
		for i := range asc {
			asc[i] = i // ascending: everything is popped, exactly once
		}
		pops = countPops(asc)
		if pops > n {
			t.Errorf("n=%d ascending: %d pops > n", n, pops)
		}
		t.Logf("n=%-6d descending: 0 pops   ascending: %d pops (both <= n)", n, pops)
	}
}

func countPops(xs []int) int {
	pops := 0
	var st Stack[int]
	for i, v := range xs {
		for {
			top, ok := st.Peek()
			if !ok || xs[top] >= v {
				break
			}
			st.Pop()
			pops++
		}
		st.Push(i)
	}
	return pops
}

func ExampleNextGreater() {
	fmt.Println(NextGreater([]int{2, 1, 2, 4, 3}))
	// Output: [4 2 4 -1 -1]
}
```

**`bench_test.go`** — pass 3: measure it

```go
package stack

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// ============================================================================
// PASS 3: MEASURE IT.
// ============================================================================

func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// Push/Pop must be flat in n -- that is what Theta(1) amortized means.
func BenchmarkPushPop(b *testing.B) {
	for _, n := range []int{100, 10_000, 1_000_000} {
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			var s Stack[int]
			s.Reserve(n)
			for b.Loop() {
				s.Clear()
				for i := 0; i < n; i++ {
					s.Push(i)
				}
				for !s.IsEmpty() {
					s.Pop()
				}
			}
		})
	}
}

// The monotonic claim: linear, on the input that makes brute force quadratic.
func BenchmarkNextGreater(b *testing.B) {
	for _, n := range []int{1_000, 4_000, 16_000, 64_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = n - i // descending: brute force's worst case
		}
		b.Run(fmt.Sprintf("mono/n=%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkS = NextGreater(xs)
			}
		})
	}
}

func BenchmarkLargestRectangle(b *testing.B) {
	rng := rand.New(rand.NewPCG(1, 2))
	for _, n := range []int{1_000, 4_000, 16_000} {
		h := make([]int, n)
		for i := range h {
			h[i] = rng.IntN(1000)
		}
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				sinkI = LargestRectangle(h)
			}
		})
	}
}

var sinkS []int
```

**Run it:**

```bash
go test -v .
go test -bench=. -benchmem -run='^$' .
```

**Sample output:**

```
--- go test -v ---
--- PASS: TestStackBasics (0.00s)
--- PASS: TestPopReleasesTheSlot (0.00s)
--- PASS: TestClearReleasesEverything (0.00s)
    stack_test.go:139: 50000 random operations, final depth 16731
--- PASS: TestStackAgainstModel (0.11s)
--- PASS: TestNextGreaterAgainstBrute (0.00s)
--- PASS: TestLargestRectangleAgainstBrute (0.00s)
--- PASS: TestAlgorithmEdgeCases (0.00s)
    --- PASS: TestAlgorithmEdgeCases/nil (0.00s)
    --- PASS: TestAlgorithmEdgeCases/empty (0.00s)
    --- PASS: TestAlgorithmEdgeCases/single (0.00s)
    --- PASS: TestAlgorithmEdgeCases/all_equal (0.00s)
    --- PASS: TestAlgorithmEdgeCases/ascending (0.00s)
    --- PASS: TestAlgorithmEdgeCases/descending (0.00s)
    --- PASS: TestAlgorithmEdgeCases/zeros (0.00s)
    --- PASS: TestAlgorithmEdgeCases/one_zero (0.00s)
    stack_test.go:242: 1000 pushes after Reserve: 0 allocations
--- PASS: TestPushIsAmortizedConstant (0.00s)
    stack_test.go:266: n=100    descending: 0 pops   ascending: 99 pops (both <= n)
    stack_test.go:266: n=1000   descending: 0 pops   ascending: 999 pops (both <= n)
    stack_test.go:266: n=10000  descending: 0 pops   ascending: 9999 pops (both <= n)
--- PASS: TestMonotonicIsLinear (0.00s)
--- PASS: ExampleNextGreater (0.00s)
PASS
ok  	l08/ex16	0.238s

--- go test -bench=. -benchmem -run='^$' ---
goos: darwin
goarch: arm64
pkg: l08/ex16
cpu: Apple M4
BenchmarkEmptyControl-10        	672748970	         1.794 ns/op	       0 B/op	       0 allocs/op
BenchmarkPushPop/n=100-10       	 3287128	       364.2 ns/op	       0 B/op	       0 allocs/op
BenchmarkPushPop/n=10000-10     	   32924	     36401 ns/op	       0 B/op	       0 allocs/op
BenchmarkPushPop/n=1000000-10   	     314	   3800719 ns/op	       0 B/op	       0 allocs/op
BenchmarkNextGreater/mono/n=1000-10         	  276115	      4872 ns/op	   33400 B/op	      13 allocs/op
BenchmarkNextGreater/mono/n=4000-10         	   44610	     24987 ns/op	  161018 B/op	      17 allocs/op
BenchmarkNextGreater/mono/n=16000-10        	   18421	     64878 ns/op	  619772 B/op	      21 allocs/op
BenchmarkNextGreater/mono/n=64000-10        	    4206	    291830 ns/op	 3028248 B/op	      27 allocs/op
BenchmarkLargestRectangle/n=1000-10         	  318907	      3773 ns/op	     248 B/op	       5 allocs/op
BenchmarkLargestRectangle/n=4000-10         	   78596	     15248 ns/op	     248 B/op	       5 allocs/op
BenchmarkLargestRectangle/n=16000-10        	   18186	     63865 ns/op	     504 B/op	       6 allocs/op
PASS
ok  	l08/ex16	13.379s
```

**Rust:**

```rust
// Go splits this capstone over three files (stack.go, stack_test.go,
// bench_test.go). Rust keeps the code and its tests in one file: the structure
// and the algorithms come first, then `#[cfg(test)]` modules for the test
// machinery, pass 2 (prove it) and pass 3 (measure it). Build and run:
//   rustc --edition 2024 -O --test ex16.rs -o ex16 && ./ex16

// =============================================================================
// stack.go -> Stack<T>
// =============================================================================

// Stack<T> is example 2's type, finished: generic, default-value-ready, with
// Option instead of Go's (T, bool).
//
// INVARIANT, true before and after every method:
//
//   a popped element is never kept alive by the stack
//
// Go has to WORK for this: its Pop zeroes the vacated slot, because a slice's
// backing array would otherwise keep the removed pointer reachable. In Rust the
// invariant is free. `Vec::pop` MOVES the value out and the slot past the length
// is uninitialised memory that safe code cannot read, so there is nothing to
// zero (example 1). The tests below still check it, with a reference count.
struct Stack<T> {
    items: Vec<T>,
}

// `#[derive(Default)]` would add a hidden `T: Default` bound; writing it by hand
// says what we mean: an empty Vec, for any T (example 2).
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

    /// Go returns `(T, bool)`; the Option is the same information in one type.
    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    /// Borrows the top instead of copying it out: `Option<&T>`.
    fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    /// Makes room for n more pushes without reallocating.
    fn reserve(&mut self, n: usize) {
        self.items.reserve(n);
    }

    /// Empties the stack and keeps the capacity for reuse (lesson 06, example 10).
    /// Go zeroes every slot by hand first; `Vec::clear` DROPS every element.
    fn clear(&mut self) {
        self.items.clear();
    }
}

// =============================================================================
// The monotonic-stack algorithms from examples 12-14, as library functions.
// =============================================================================

/// Returns, for each i, the next element to the right that is strictly greater,
/// or `None` if there is none (Go answers `-1`). Theta(n).
fn next_greater(xs: &[i64]) -> Vec<Option<i64>> {
    let mut out = vec![None; xs.len()];
    let mut st: Stack<usize> = Stack::default(); // indices, values strictly decreasing
    for (i, &v) in xs.iter().enumerate() {
        while let Some(&top) = st.peek() {
            if xs[top] >= v {
                break;
            }
            st.pop();
            out[top] = Some(v);
        }
        st.push(i);
    }
    out
}

/// Returns the area of the largest rectangle under the histogram. Theta(n).
fn largest_rectangle(h: &[i64]) -> i64 {
    let mut best = 0;
    let mut st: Stack<usize> = Stack::default(); // indices, heights increasing
    for i in 0..=h.len() {
        // a virtual zero bar at the end flushes the stack
        let cur = h.get(i).copied().unwrap_or(0);
        while let Some(&top) = st.peek() {
            if h[top] < cur {
                break;
            }
            st.pop();
            // The bar BELOW the popped one is the left edge. Go starts from
            // `left := -1` for "none"; here None means the rectangle reaches
            // column 0, so its width is just i.
            let width = st.peek().map_or(i, |&left| i - left - 1);
            best = best.max(h[top] * width as i64);
        }
        st.push(i);
    }
    best
}

// =============================================================================
// Shared test machinery: what `testing`, `testing.AllocsPerRun` and
// `math/rand/v2` give Go for free. Nothing here is about the Stack.
// =============================================================================

#[cfg(test)]
mod harness {
    use std::alloc::{GlobalAlloc, Layout, System};
    use std::cell::Cell;
    use std::io::Write;

    // ---- allocation counter: stand-in for testing.AllocsPerRun and -benchmem ----
    //
    // Counts per THREAD, not globally: libtest runs the tests in parallel, and a
    // global counter would also count whatever the other tests allocate while this
    // one is measured. (Go's AllocsPerRun avoids the same problem by pinning
    // GOMAXPROCS to 1.) A `const` thread-local with no destructor is safe to touch
    // from inside an allocator.
    struct Counting;
    thread_local! {
        static EVENTS: Cell<usize> = const { Cell::new(0) };
        static BYTES: Cell<usize> = const { Cell::new(0) };
    }

    unsafe impl GlobalAlloc for Counting {
        // `unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every method only
        // counts and then forwards to the system allocator unchanged.
        unsafe fn alloc(&self, l: Layout) -> *mut u8 {
            EVENTS.with(|c| c.set(c.get() + 1));
            BYTES.with(|c| c.set(c.get() + l.size()));
            unsafe { System.alloc(l) }
        }
        unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
            unsafe { System.dealloc(p, l) }
        }
        unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
            EVENTS.with(|c| c.set(c.get() + 1));
            BYTES.with(|c| c.set(c.get() + new));
            unsafe { System.realloc(p, l, new) }
        }
    }
    #[global_allocator]
    static GLOBAL: Counting = Counting;

    /// (allocation events, bytes requested) on this thread while running `f`.
    pub fn allocs_of(f: impl FnOnce()) -> (usize, usize) {
        let (e0, b0) = (EVENTS.with(Cell::get), BYTES.with(Cell::get));
        f();
        (EVENTS.with(Cell::get) - e0, BYTES.with(Cell::get) - b0)
    }

    /// Like testing.AllocsPerRun: one warm-up call, then the average over `runs`.
    pub fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> usize {
        f();
        let (events, _) = allocs_of(|| {
            for _ in 0..runs {
                f();
            }
        });
        events / runs
    }

    // go test -v shows t.Logf lines. libtest swallows the println! of a PASSING
    // test unless you pass --show-output, so this writes to the real stdout and
    // the numbers are visible with a plain `./ex16`. (libtest runs tests on
    // parallel threads, so a log line can land between two `test ... ok` lines.)
    pub fn log(line: &str) {
        let _ = writeln!(std::io::stdout(), "{line}");
    }

    /// A port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN.
    /// With the same generator and seeds, every random test below sees exactly
    /// the inputs Go's test sees.
    pub struct Pcg {
        hi: u64,
        lo: u64,
    }

    impl Pcg {
        pub fn new(seed1: u64, seed2: u64) -> Self {
            Pcg {
                hi: seed1,
                lo: seed2,
            }
        }

        fn next_u64(&mut self) -> u64 {
            const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
            const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
            // state = state * MUL + INC, with explicit 128-bit wraparound
            let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
                .wrapping_mul(MUL)
                .wrapping_add(INC);
            self.hi = (state >> 64) as u64;
            self.lo = state as u64;

            // DXSM "double xorshift multiply"
            let mut hi = self.hi;
            hi ^= hi >> 32;
            hi = hi.wrapping_mul(0xda942042e4dd58b5);
            hi ^= hi >> 48;
            hi.wrapping_mul(self.lo | 1)
        }

        /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
        pub fn int_n(&mut self, n: usize) -> usize {
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
}

// =============================================================================
// stack_test.go -> PASS 2: PROVE IT. Same four layers as lessons 06 and 07.
//
// The structure here is nearly trivial, so most of the risk is in the
// ALGORITHMS built on it -- and those get brute-force oracles, because a
// monotonic stack is exactly the kind of index arithmetic that looks right
// and is off by one.
// =============================================================================

#[cfg(test)]
mod tests {
    use super::*;
    use crate::harness::{Pcg, allocs_per_run, log};
    use std::hint::black_box;
    use std::rc::Rc;

    // --- 1. the table ---------------------------------------------------------

    #[test]
    fn stack_basics() {
        let mut s: Stack<i64> = Stack::default();

        assert_eq!(s.pop(), None, "Pop on an empty stack returned a value");
        assert_eq!(s.peek(), None, "Peek on an empty stack returned a value");
        assert!(s.is_empty(), "the default value is not empty");
        assert_eq!(s.len(), 0, "the default value is not empty");

        for i in 1..=3 {
            s.push(i);
        }
        assert_eq!(s.peek(), Some(&3), "Peek");
        assert_eq!(s.len(), 3, "Len");
        // Peek must not remove.
        assert_eq!(s.peek(), Some(&3), "second Peek");
        assert_eq!(s.len(), 3, "Peek changed the stack");

        for i in (1..=3).rev() {
            assert_eq!(s.pop(), Some(i), "Pop");
        }
        assert!(s.is_empty(), "stack not empty after popping everything");
    }

    // --- 2. the invariant -----------------------------------------------------

    // Go's test pushes pointers, pops them, and then reads the spare capacity of
    // the backing array to check every slot was zeroed. Safe Rust cannot read
    // that memory at all, so the question changes to the one the invariant is FOR:
    // does the stack still own anything it gave back? An Rc's strong count says
    // exactly how many owners are alive.
    #[test]
    fn pop_releases_the_slot() {
        let token = Rc::new(());
        let mut s: Stack<Rc<()>> = Stack::default();

        s.push(Rc::clone(&token));
        s.push(Rc::clone(&token));
        assert_eq!(Rc::strong_count(&token), 1 + 2);

        s.pop();
        s.pop();
        assert_eq!(
            Rc::strong_count(&token),
            1,
            "a popped element is still owned by the stack's spare capacity"
        );
    }

    #[test]
    fn clear_releases_everything() {
        let token = Rc::new(());
        let mut s: Stack<Rc<()>> = Stack::default();
        for _ in 0..5 {
            s.push(Rc::clone(&token));
        }
        let cap = s.items.capacity();
        s.clear();

        assert_eq!(s.len(), 0, "Len after Clear");
        assert_eq!(
            s.items.capacity(),
            cap,
            "Clear changed the capacity; it should keep it for reuse"
        );
        assert_eq!(
            Rc::strong_count(&token),
            1,
            "Clear left elements alive in the stack"
        );
    }

    // --- 3. the model oracle --------------------------------------------------

    #[test]
    fn stack_against_model() {
        let mut rng = Pcg::new(42, 7);
        let mut s: Stack<i64> = Stack::default();
        let mut model: Vec<i64> = Vec::new();

        const TRIALS: usize = 50_000;
        for i in 0..TRIALS {
            match rng.int_n(3) {
                0 | 1 => {
                    let v = rng.int_n(1000) as i64;
                    s.push(v);
                    model.push(v);
                }
                _ => {
                    // One assert covers "ok on an empty stack", the value, and
                    // the missing value, because Option<T> carries all three.
                    assert_eq!(s.pop(), model.pop(), "trial {i}: Pop");
                }
            }
            assert_eq!(s.len(), model.len(), "trial {i}: Len");
            assert_eq!(s.items, model, "trial {i}: contents diverged");
        }
        log(&format!(
            "{TRIALS} random operations, final depth {}",
            s.len()
        ));
    }

    // --- 4. brute-force oracles for the ALGORITHMS ----------------------------

    fn next_greater_brute(xs: &[i64]) -> Vec<Option<i64>> {
        (0..xs.len())
            .map(|i| xs[i + 1..].iter().copied().find(|&y| y > xs[i]))
            .collect()
    }

    fn largest_rectangle_brute(h: &[i64]) -> i64 {
        let mut best = 0;
        for i in 0..h.len() {
            let mut min_h = h[i];
            for (j, &x) in h.iter().enumerate().skip(i) {
                min_h = min_h.min(x);
                best = best.max(min_h * (j - i + 1) as i64);
            }
        }
        best
    }

    #[test]
    fn next_greater_against_brute() {
        let mut rng = Pcg::new(1, 2);
        for _ in 0..20_000 {
            // SMALL values: ties are where a strict/non-strict comparison goes
            // wrong, and a wide range would almost never produce one.
            let len = rng.int_n(10);
            let xs: Vec<i64> = (0..len).map(|_| rng.int_n(5) as i64).collect();
            assert_eq!(
                next_greater(&xs),
                next_greater_brute(&xs),
                "next_greater({xs:?})"
            );
        }
    }

    #[test]
    fn largest_rectangle_against_brute() {
        let mut rng = Pcg::new(3, 4);
        for _ in 0..20_000 {
            let len = rng.int_n(10);
            let h: Vec<i64> = (0..len).map(|_| rng.int_n(6) as i64).collect();
            assert_eq!(
                largest_rectangle(&h),
                largest_rectangle_brute(&h),
                "largest_rectangle({h:?})"
            );
        }
    }

    // libtest has no subtests (Go's t.Run). A macro turns each row of the table
    // into its own #[test], so every case still gets its own line in the output
    // and can be run alone: `./ex16 algorithm_edge_cases::one_zero`. Go's table
    // also has a "nil" row, separate from "empty"; a Rust slice cannot be nil,
    // so there is only `empty`.
    mod algorithm_edge_cases {
        use super::*;

        macro_rules! cases {
            ($($name:ident: $input:expr;)*) => {$(
                #[test]
                fn $name() {
                    let xs: &[i64] = &$input;
                    assert_eq!(next_greater(xs), next_greater_brute(xs), "next_greater");
                    assert_eq!(
                        largest_rectangle(xs),
                        largest_rectangle_brute(xs),
                        "largest_rectangle"
                    );
                }
            )*};
        }

        cases! {
            empty: [];
            single: [5];
            all_equal: [3, 3, 3, 3];
            ascending: [1, 2, 3, 4];
            descending: [4, 3, 2, 1];
            zeros: [0, 0, 0];
            one_zero: [2, 0, 2];
        }
    }

    // --- the amortized claim, asserted ----------------------------------------

    #[test]
    fn push_is_amortized_constant() {
        let mut s: Stack<i64> = Stack::default();
        s.reserve(1000);
        let allocs = allocs_per_run(50, || {
            s.clear();
            for i in 0..1000 {
                s.push(i);
            }
            black_box(s.len());
        });
        assert_eq!(
            allocs, 0,
            "1000 pushes into reserved capacity allocated {allocs} times, want 0"
        );
        log(&format!("1000 pushes after Reserve: {allocs} allocations"));

        // Rust addition: the other half of the claim. WITHOUT Reserve, a fresh
        // stack still allocates only O(log n) times, because Vec's capacity
        // grows geometrically: 4, 8, 16, ... 1024 is nine allocation events.
        let (events, _) = crate::harness::allocs_of(|| {
            let mut fresh: Stack<i64> = Stack::default();
            for i in 0..1000 {
                fresh.push(i);
            }
            black_box(&fresh);
        });
        assert!(
            events <= 12,
            "1000 pushes onto a fresh stack allocated {events} times -- growth is not geometric"
        );
        log(&format!(
            "1000 pushes onto a fresh stack: {events} allocations"
        ));
    }

    // The monotonic-stack claim: total inner-loop work over a whole run is at
    // most n, because each index is pushed once and popped at most once. Assert it.
    #[test]
    fn monotonic_is_linear() {
        for n in [100usize, 1000, 10_000] {
            let descending: Vec<i64> = (1..=n as i64).rev().collect(); // nothing is ever popped
            let pops = count_pops(&descending);
            assert!(
                pops <= n,
                "n={n}: {pops} pops, which exceeds n -- the amortized bound is broken"
            );

            let ascending: Vec<i64> = (0..n as i64).collect(); // everything is popped, exactly once
            let pops = count_pops(&ascending);
            assert!(pops <= n, "n={n} ascending: {pops} pops > n");
            log(&format!(
                "n={n:<6} descending: 0 pops   ascending: {pops} pops (both <= n)"
            ));
        }
    }

    fn count_pops(xs: &[i64]) -> usize {
        let mut pops = 0;
        let mut st: Stack<usize> = Stack::default();
        for (i, &v) in xs.iter().enumerate() {
            while let Some(&top) = st.peek() {
                if xs[top] >= v {
                    break;
                }
                st.pop();
                pops += 1;
            }
            st.push(i);
        }
        pops
    }

    // Go's ExampleNextGreater is a test whose printed output must match its
    // `// Output:` comment, and godoc shows it as documentation. Rust's twin is a
    // doctest inside a `///` comment, which `cargo test --doc` or `rustdoc --test`
    // runs -- but plain `rustc --test` does not, so here it is an ordinary test.
    #[test]
    fn example_next_greater() {
        assert_eq!(
            next_greater(&[2, 1, 2, 4, 3]),
            [Some(4), Some(2), Some(4), None, None]
        );
    }
}

// =============================================================================
// bench_test.go -> PASS 3: MEASURE IT.
//
// Stable Rust has no benchmark runner (#[bench] is nightly-only; criterion and
// divan are the real tools, but they are crates). So the benchmarks are ONE
// #[test] that prints a table, built from `ns_per_op` below, which times `f`
// the way testing.Benchmark does: run it n times, and if that took less than
// the target, scale n up and go again.
// =============================================================================

#[cfg(test)]
mod bench {
    use super::*;
    use crate::harness::{Pcg, allocs_of, log};
    use std::hint::black_box;
    use std::time::{Duration, Instant};

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

    /// One row of `go test -bench -benchmem`: ns/op, B/op, allocs/op.
    /// B/op is bytes REQUESTED from the allocator (Go rounds up to size classes).
    /// Noise only ever ADDS time, so ns/op is the fastest of three measurements:
    /// the one least disturbed by whatever else the machine was doing.
    fn run(name: &str, mut f: impl FnMut()) -> f64 {
        let (allocs, bytes) = allocs_of(&mut f);
        let ns = (0..3)
            .map(|_| ns_per_op(&mut f))
            .fold(f64::INFINITY, f64::min);
        log(&format!(
            "{name:<28} {ns:>12.1} ns/op {bytes:>10} B/op {allocs:>4} allocs/op"
        ));
        ns
    }

    #[test]
    fn benchmarks() {
        // Always first. No number below this floor means anything (lesson 02).
        // Rust needs black_box even here: an EMPTY closure folds away, the whole
        // timing loop is deleted, and the "floor" would print as 0.0.
        run("EmptyControl", || black_box(()));

        // Push/Pop must be flat in n -- that is what Theta(1) amortized means.
        // black_box(n) hides the constant from the optimizer.
        let mut per_pair = Vec::new();
        for n in [100, 10_000, 1_000_000] {
            let n = black_box(n);
            let mut s: Stack<i64> = Stack::default();
            s.reserve(n);
            let ns = run(&format!("PushPop/n={n}"), || {
                s.clear();
                for i in 0..n {
                    s.push(i as i64);
                }
                while let Some(v) = s.pop() {
                    black_box(v);
                }
            });
            per_pair.push(ns / n as f64);
        }
        log(&format!(
            "  ns per push+pop pair: {}",
            per_pair
                .iter()
                .map(|p| format!("{p:.2}"))
                .collect::<Vec<_>>()
                .join(", ")
        ));

        // The monotonic claim: linear, on the input that makes brute force quadratic.
        for n in [1_000, 4_000, 16_000, 64_000] {
            // descending: brute force's worst case
            let xs: Vec<i64> = (1..=n as i64).rev().collect();
            run(&format!("NextGreater/mono/n={n}"), || {
                black_box(next_greater(black_box(&xs)));
            });
        }

        let mut rng = Pcg::new(1, 2);
        for n in [1_000, 4_000, 16_000] {
            let h: Vec<i64> = (0..n).map(|_| rng.int_n(1000) as i64).collect();
            run(&format!("LargestRectangle/n={n}"), || {
                black_box(largest_rectangle(black_box(&h)));
            });
        }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O --test ex16.rs -o ex16 && ./ex16
```

**Rust sample output:**

```

running 17 tests
test tests::algorithm_edge_cases::empty ... ok
test tests::algorithm_edge_cases::all_equal ... ok
test tests::algorithm_edge_cases::ascending ... ok
test tests::algorithm_edge_cases::descending ... ok
test tests::algorithm_edge_cases::one_zero ... ok
test tests::algorithm_edge_cases::single ... ok
test tests::algorithm_edge_cases::zeros ... ok
test tests::clear_releases_everything ... ok
test tests::example_next_greater ... ok
test tests::largest_rectangle_against_brute ... ok
test tests::next_greater_against_brute ... ok
n=100    descending: 0 pops   ascending: 99 pops (both <= n)
n=1000   descending: 0 pops   ascending: 999 pops (both <= n)
n=10000  descending: 0 pops   ascending: 9999 pops (both <= n)
1000 pushes after Reserve: 0 allocations
1000 pushes onto a fresh stack: 9 allocations
test tests::pop_releases_the_slot ... ok
test tests::stack_basics ... ok
test tests::monotonic_is_linear ... ok
test tests::push_is_amortized_constant ... ok
50000 random operations, final depth 16731
test tests::stack_against_model ... ok
EmptyControl                          0.3 ns/op          0 B/op    0 allocs/op
PushPop/n=100                       171.8 ns/op          0 B/op    0 allocs/op
PushPop/n=10000                   16846.4 ns/op          0 B/op    0 allocs/op
PushPop/n=1000000               2117766.3 ns/op          0 B/op    0 allocs/op
  ns per push+pop pair: 1.72, 1.68, 2.12
NextGreater/mono/n=1000            1430.4 ns/op      32352 B/op   10 allocs/op
NextGreater/mono/n=4000            4288.0 ns/op     129504 B/op   12 allocs/op
NextGreater/mono/n=16000          15607.1 ns/op     518112 B/op   14 allocs/op
NextGreater/mono/n=64000          57139.8 ns/op    2072544 B/op   16 allocs/op
LargestRectangle/n=1000            2197.2 ns/op        224 B/op    3 allocs/op
LargestRectangle/n=4000            8661.7 ns/op        224 B/op    3 allocs/op
LargestRectangle/n=16000          58203.3 ns/op        480 B/op    4 allocs/op
test bench::benchmarks ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 10.93s
```

> **Go vs Rust:** Every line that is not a timing is identical to Go's (a port of Go's PCG feeds both programs the same random inputs): final depth 16731, 99/999/9999 pops, 0 allocations after `reserve`; libtest's `test ... ok` lines replace `--- PASS`. Rust cannot read `items[len:cap]`, so the slot-release tests count owners with `Rc` (the invariant holds by construction, because `Vec::pop` moves the value out), `Option` replaces `-1` and `(T, bool)`, and Go's `ExampleNextGreater` has no `rustc --test` form (it would be a doctest), so it is a plain assert. Timings are the best of three, and across six runs on a shared machine push+pop cost 1.5-3.1 ns per pair from n=100 to n=1,000,000 (Go: 3.6-3.8), `NextGreater` was about 3-6x faster than Go's sample at every n, and its allocation count grows 10, 12, 14, 16 against Go's 13, 17, 21, 27 for the same four sizes. The PushPop ns/op also moved by about 2x in side experiments when unrelated code in the binary changed (installing the counting allocator behind the B/op column, or deleting the EmptyControl row), so read it as 'a couple of ns per pair', not a precise figure.

**Complexity:** `Push`/`Pop`/`Peek` Θ(1) amortized · `NextGreater` and `LargestRectangle` Θ(n) · the tests assert both

---

> That closes the stack. Next: [09 — Queues & Deques](../../09-queues-deques.md) —
> the other end of the slice, where it stops being free.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*